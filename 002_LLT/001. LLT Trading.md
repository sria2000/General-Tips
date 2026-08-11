# Low-Latency Trading Server: FPGA + Solarflare Host Configuration Guide

Reference architecture and host-side configuration for a dual-NIC low-latency trading server on RHEL, using:
- **FPGA card** → inbound market data (feed handling, hardware timestamping)
- **Solarflare card** → outbound order entry (OpenOnload kernel-bypass)
- **2x onboard/add-in NICs** → active-active management bond (OS, monitoring, SSH)

---

## Architecture Summary

```
                         NUMA Node 0
   ┌───────────────────────────────────────────────────────┐
   │  Market data ──▶ [FPGA card] ──▶ [Isolated CPU cores]  │
   │                                        │                │
   │                                        ▼                │
   │                              [Isolated CPU cores] ──▶   │
   │                                [Solarflare card] ──▶ Orders
   │                                                         │
   │            PTP hardware timestamp source (shared)       │
   └───────────────────────────────────────────────────────┘

                         Housekeeping (non-isolated cores)
   ┌───────────────────────────────────────────────────────┐
   │  [bond0: 2x mgmt NIC, LACP active-active]               │
   │  [OS / SSH / CheckMK / auditd / general IRQs]           │
   └───────────────────────────────────────────────────────┘
```

**Key principle:** the FPGA card, the Solarflare card, and the CPU cores running the trading application must all sit on the **same NUMA node**. Everything not latency-critical (OS, monitoring, management bond) is deliberately kept on the other node/non-isolated cores.

---

## 1. Identify NUMA Topology and PCIe Slot Mapping

Before touching any config, confirm which NUMA node each PCIe slot belongs to — this determines where you physically install the FPGA and Solarflare cards.

```bash
# Show NUMA node layout and core ranges
lscpu | grep -i numa

# Show which NUMA node each PCIe device is attached to
for dev in $(lspci | grep -iE 'mellanox|solarflare|xilinx' | cut -d' ' -f1); do
  echo "PCI $dev -> NUMA node: $(cat /sys/bus/pci/devices/0000:$dev/numa_node)"
done

# Full hardware/NUMA map
numactl --hardware
```

Install both the FPGA and Solarflare cards into PCIe slots physically wired to the **same** NUMA node. This is a BIOS/motherboard-layout fact, not something software can fix after the fact — check your server's technical manual for slot-to-socket wiring before racking the hardware.

---

## 2. BIOS/UEFI Settings

Set these in the server's BIOS before OS installation:

- **Hyper-Threading / SMT:** Disabled (removes sibling-thread cache contention on latency-critical cores)
- **Power management / C-states:** Disabled (prevents CPU sleep-state wake latency spikes)
- **P-states / SpeedStep / Turbo:** Set to maximum performance / disable frequency scaling
- **NUMA:** Enabled (do not use "node interleaving" — you want strict NUMA locality, not interleaved memory)
- **SR-IOV:** Enabled if you plan to virtualize any NIC access
- **VT-d / IOMMU:** Enabled if using SR-IOV or PCIe passthrough

---

## 3. GRUB Kernel Parameters

Edit `/etc/default/grub`:

```bash
GRUB_CMDLINE_LINUX="isolcpus=4-7,12-15 nohz_full=4-7,12-15 rcu_nocbs=4-7,12-15 intel_pstate=disable processor.max_cstate=1 intel_idle.max_cstate=0 default_hugepagesz=1G hugepagesz=1G hugepages=16 intel_iommu=on iommu=pt ipv6.disable=1"
```

Adjust `4-7,12-15` to match your actual isolated core ranges from `lscpu`/`numactl --hardware` — these should be cores physically on the same NUMA node as your FPGA/Solarflare cards.

Rebuild grub config:

```bash
grub2-mkconfig -o /boot/grub2/grub.cfg   # BIOS boot
# or
grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg   # UEFI boot

reboot
```

Verify after reboot:

```bash
cat /sys/devices/system/cpu/isolated
cat /proc/cmdline
```

---

## 4. Hugepages Verification

```bash
grep Huge /proc/meminfo
```

Mount a hugetlbfs mountpoint if your application expects one:

```bash
mkdir -p /mnt/huge
mount -t hugetlbfs nodev /mnt/huge
echo "nodev /mnt/huge hugetlbfs pagesize=1G 0 0" >> /etc/fstab
```

---

## 5. tuned Profile

```bash
dnf install -y tuned tuned-profiles-realtime
tuned-adm profile latency-performance

# Or build a custom profile layering on top of it
mkdir -p /etc/tuned/trading-latency
cat <<'EOF' > /etc/tuned/trading-latency/tuned.conf
[main]
include=latency-performance

[cpu]
force_latency=1
governor=performance
energy_perf_bias=performance
min_perf_pct=100

[vm]
transparent_hugepages=never
EOF

tuned-adm profile trading-latency
tuned-adm active
```

---

## 6. Persistent NIC Naming (udev)

Don't rely on `ens33`/`eno1` auto-naming for a multi-NIC box — pin names explicitly so a reboot or card reseat never silently swaps roles.

```bash
# Find MAC addresses
ip link show

# /etc/udev/rules.d/70-persistent-net.rules
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="<fpga-mac>", NAME="fpga0"
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="<solarflare-mac>", NAME="sfc0"
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="<mgmt-nic1-mac>", NAME="mgmt0"
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="<mgmt-nic2-mac>", NAME="mgmt1"
```

```bash
udevadm control --reload-rules
udevadm trigger
```

---

## 7. Solarflare OpenOnload Setup

```bash
# Install the OpenOnload package (obtained from AMD/Solarflare support portal)
rpm -ivh openonload-<version>.rpm

# Load the kernel modules
onload_tool reload

# Confirm the driver and NIC are recognised
sfboot --version
ethtool -i sfc0

# Set the interrupt moderation and ring buffer sizes for lowest latency
ethtool -C sfc0 rx-usecs 0 tx-usecs 0
ethtool -G sfc0 rx 4096 tx 4096

# Run the order-entry application under OpenOnload acceleration
onload --profile=latency ./order_engine
```

Key OpenOnload environment tuning (`/etc/sysconfig/onload` or exported before launch):

```bash
export EF_POLL_USEC=100000      # spin-poll instead of blocking, trades CPU for latency
export EF_INT_DRIVEN=0          # disable interrupt-driven mode, pure polling
export EF_RXQ_SIZE=4096
export EF_TXQ_SIZE=4096
```

---

## 8. FPGA Card Setup

Exact steps are vendor-specific (Xilinx/AMD Alveo, Exegy, Solarflare's own FPGA line, etc.) — the general host-side pattern is:

```bash
# Install vendor driver/runtime (example: Xilinx XRT)
dnf install -y xrt

# Verify the card is detected
xbutil examine

# Load the feed-handler bitstream/shell onto the card
xbutil program --device <bdf> --image feed_handler.xclbin

# Confirm the interface is up and bound to the correct NUMA node
ethtool -i fpga0
cat /sys/class/net/fpga0/device/numa_node
```

If the FPGA vendor provides its own low-latency socket library (common — many FPGA feed handlers expose a userspace API rather than a standard socket), your feed handler application will link against that library directly rather than using standard Linux sockets. Confirm with the vendor's SDK docs which model applies to your card.

---

## 9. IRQ Affinity — Keep Isolated Cores Interrupt-Free

Disable `irqbalance` globally so it stops auto-migrating IRQs onto isolated cores:

```bash
systemctl disable --now irqbalance
```

Pin trading NIC IRQs to the isolated cores (matching the cores where the application actually runs), and everything else to housekeeping cores:

```bash
# Find IRQs for a given NIC
grep sfc0 /proc/interrupts
grep fpga0 /proc/interrupts

# Example: pin an IRQ to core 5 (bitmask, core 5 = 0x20)
echo 20 > /proc/irq/<irq_number>/smp_affinity

# Or using the core list form (RHEL 8/9)
echo 5 > /proc/irq/<irq_number>/smp_affinity_list
```

Script this at boot via a systemd oneshot service so it survives reboots — example `/etc/systemd/system/irq-pin.service`:

```ini
[Unit]
Description=Pin trading NIC IRQs to isolated cores
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/pin-irqs.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
systemctl enable --now irq-pin.service
```

---

## 10. CPU Pinning for the Applications

```bash
# Pin the feed handler process to its isolated cores
taskset -c 4-5 ./feed_handler

# Pin the order entry process to its isolated cores
taskset -c 6-7 ./order_engine

# Confirm NUMA-local memory allocation as well as CPU pinning
numactl --cpunodebind=0 --membind=0 ./feed_handler
```

---

## 11. Active-Active Management Bond (LACP)

Requires coordination with the network team to configure the matching port-channel/LAG on the switch side.

```bash
nmcli connection add type bond con-name bond0 ifname bond0 mode 802.3ad
nmcli connection modify bond0 +bond.options "lacp_rate=fast,miimon=100,xmit_hash_policy=layer3+4"

nmcli connection add type ethernet ifname mgmt0 master bond0
nmcli connection add type ethernet ifname mgmt1 master bond0

nmcli connection modify bond0 ipv4.addresses 192.168.10.10/24 ipv4.gateway 192.168.10.1 ipv4.method manual
nmcli connection modify bond0 ipv4.dns "192.168.10.1"

nmcli connection up bond0
```

Verify:

```bash
cat /proc/net/bonding/bond0
```

---

## 12. PTP Hardware Timestamping (linuxptp)

Both the FPGA and Solarflare NICs should reference the same PTP grandmaster clock for consistent, comparable timestamps across the inbound and outbound paths.

```bash
dnf install -y linuxptp

# Run ptp4l against the trading NIC's hardware clock (example: sfc0)
ptp4l -i sfc0 -m -H &

# Sync the system clock to the NIC's PTP hardware clock
phc2sys -s sfc0 -c CLOCK_REALTIME -w &
```

Persist as systemd services (`/etc/systemd/system/ptp4l.service`, `phc2sys.service`) rather than running ad hoc — both should start at boot before the trading applications do.

Check synchronisation status:

```bash
pmc -u -b 0 'GET CURRENT_DATA_SET'
```

---

## 13. Network Stack Tuning (sysctl)

```ini
# /etc/sysctl.d/98-trading-network.conf

# Increase socket buffer sizes for burst market data handling
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.core.rmem_default = 67108864
net.core.wmem_default = 67108864

# Increase backlog to absorb bursts without drops
net.core.netdev_max_backlog = 250000

# Disable IPv6 (unused in this stack — see notes)
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1

# Busy polling — reduces latency at the cost of CPU, appropriate on isolated cores
net.core.busy_poll = 50
net.core.busy_read = 50
```

```bash
sysctl -p /etc/sysctl.d/98-trading-network.conf
```

---

## 14. Validation Checklist

```bash
# Confirm isolated cores took effect
cat /sys/devices/system/cpu/isolated

# Confirm hugepages are allocated
grep Huge /proc/meminfo

# Confirm tuned profile is active
tuned-adm active

# Confirm NIC-to-NUMA alignment
cat /sys/class/net/fpga0/device/numa_node
cat /sys/class/net/sfc0/device/numa_node

# Confirm IRQs are NOT landing on isolated cores
grep -E 'sfc0|fpga0' /proc/interrupts

# Confirm bond is active-active and both links are up
cat /proc/net/bonding/bond0

# Confirm PTP sync status
pmc -u -b 0 'GET CURRENT_DATA_SET'

# Confirm IPv6 is disabled
sysctl net.ipv6.conf.all.disable_ipv6
```

---

## Notes / Honesty Flags for Interview Use

- Exact FPGA driver/bitstream steps are vendor-specific — the pattern above (load driver → program bitstream → confirm NUMA binding) is generic; substitute the actual vendor SDK commands for whichever FPGA platform the client uses.
- OpenOnload licensing and exact tuning parameters vary by Solarflare card generation (X2/X3 series) — confirm against the vendor's current tuning guide before applying blindly in production.
- This document assumes a single-application-per-NUMA-node design. Multi-strategy or multi-tenant boxes need additional cgroup/cpuset isolation on top of this.
