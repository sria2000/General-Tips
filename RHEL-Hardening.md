# RHEL OS Hardening Cheat Sheet

A practical, command-level reference for hardening RHEL 7/8/9 systems in an enterprise or regulated environment. Aligned loosely with CIS Benchmarks / DISA STIG controls. Always validate against your organisation's actual baseline and change management process before applying to production.

---

## Table of Contents

1. [Secure Build Standards](#1-secure-build-standards)
2. [Partitioning & Filesystem Hardening](#2-partitioning--filesystem-hardening)
3. [Service Minimisation](#3-service-minimisation)
4. [Kernel & Network Hardening (sysctl)](#4-kernel--network-hardening-sysctl)
5. [SSH Hardening](#5-ssh-hardening)
6. [Password Policy & PAM](#6-password-policy--pam)
7. [Account Lockout (pam_faillock)](#7-account-lockout-pam_faillock)
8. [sudo Hardening](#8-sudo-hardening)
9. [Centralised Identity (IdM / AD / SSSD)](#9-centralised-identity-idm--ad--sssd)
10. [Auditd Configuration](#10-auditd-configuration)
11. [File Integrity Monitoring (AIDE)](#11-file-integrity-monitoring-aide)
12. [Logging & SIEM Forwarding](#12-logging--siem-forwarding)
13. [Patch Management](#13-patch-management)
14. [Automated Compliance Scanning (OpenSCAP)](#14-automated-compliance-scanning-openscap)
15. [Legal Banner / MOTD](#15-legal-banner--motd)
16. [Quick Reference Checklist](#16-quick-reference-checklist)

---

## 1. Secure Build Standards

- Build from a **hardened golden image** (Kickstart + post-install scripts, or Packer/Ansible) rather than hardening ad hoc after install.
- Pin to a documented baseline: **CIS Benchmark** (most common in banking) or **DISA STIG** (common in defence-adjacent environments). Know which one applies before you start.
- Document and formally sign off any deviation from the baseline — auditors will ask for justified exceptions, not blanket compliance claims.
- Version-control your Kickstart files, Ansible playbooks, and sysctl/audit rule files in Git. This is also your evidence trail for audits.

```bash
# Example minimal kickstart hardening snippet
rootpw --iscrypted $6$...
selinux --enforcing
firewall --enabled
authselect select sssd with-faillock with-pamaccess
```

---

## 2. Partitioning & Filesystem Hardening

Separate partitions limit blast radius (e.g. a filled `/tmp` or `/var/log` can't crash the whole root filesystem) and allow strict mount options.

| Partition | Recommended mount options |
|---|---|
| `/tmp` | `nodev,nosuid,noexec` |
| `/var` | `nodev` |
| `/var/tmp` | `nodev,nosuid,noexec` |
| `/var/log` | `nodev,nosuid,noexec` |
| `/var/log/audit` | `nodev,nosuid,noexec` |
| `/home` | `nodev,nosuid` |

```bash
# /etc/fstab example line
/dev/mapper/vg-tmp  /tmp  xfs  defaults,nodev,nosuid,noexec  0 0
```

Disable unused/legacy filesystem modules:

```bash
cat <<'EOF' > /etc/modprobe.d/cis-hardening.conf
install cramfs /bin/true
install freevxfs /bin/true
install jffs2 /bin/true
install hfs /bin/true
install hfsplus /bin/true
install squashfs /bin/true
install udf /bin/true
install usb-storage /bin/true
EOF
```

---

## 3. Service Minimisation

```bash
# List enabled services and review each one
systemctl list-unit-files --state=enabled

# Disable anything not explicitly required
systemctl disable --now <service>

# Common services to disable unless explicitly needed
systemctl disable --now avahi-daemon cups nfs-server rpcbind telnet.socket rsh.socket
```

- Remove legacy/clear-text protocols entirely where possible: telnet, rsh, FTP, TFTP.
- SSH is the only remote administration protocol permitted in most banking baselines.
- Remove compilers and unnecessary dev toolchains from production servers where not required.

```bash
rpm -qa | grep -E 'telnet-server|rsh-server|ypserv|tftp-server'
dnf remove -y telnet-server rsh-server ypserv tftp-server
```

---

## 4. Kernel & Network Hardening (sysctl)

Create `/etc/sysctl.d/99-hardening.conf`:

```ini
# IP forwarding — disable unless this is a router/gateway
net.ipv4.ip_forward = 0
net.ipv6.conf.all.forwarding = 0

# ICMP redirects — disable, prevents MITM route injection
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

# Source routing — disable
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0

# SYN flood protection
net.ipv4.tcp_syncookies = 1

# Ignore broadcast ICMP (Smurf attack mitigation)
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Log martian packets
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1

# Reverse path filtering
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# ASLR — full randomisation
kernel.randomize_va_space = 2

# Restrict kernel pointer exposure
kernel.kptr_restrict = 2

# Restrict dmesg access to root
kernel.dmesg_restrict = 1

# Restrict ptrace scope (limits process injection/debugging attacks)
kernel.yama.ptrace_scope = 1

# Disable core dumps for setuid programs
fs.suid_dumpable = 0
```

Apply:

```bash
sysctl -p /etc/sysctl.d/99-hardening.conf
```

---

## 5. SSH Hardening

Edit `/etc/ssh/sshd_config`:

```ini
Protocol 2
PermitRootLogin prohibit-password
PasswordAuthentication no
PermitEmptyPasswords no
X11Forwarding no
MaxAuthTries 4
ClientAliveInterval 300
ClientAliveCountMax 0
LoginGraceTime 60
AllowTcpForwarding no
Banner /etc/issue.net
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
```

Always validate syntax before restarting, especially on a remote session:

```bash
sshd -t
systemctl restart sshd
```

> **Tip:** Keep an existing console/out-of-band session open when changing `sshd_config` remotely. A typo can lock you out with no auto-revert (unlike `netplan try` for network config).

---

## 6. Password Policy & PAM

`/etc/security/pwquality.conf`:

```ini
minlen = 14
minclass = 4
maxrepeat = 3
maxsequence = 3
dcredit = -1
ucredit = -1
lcredit = -1
ocredit = -1
enforce_for_root
```

`/etc/login.defs`:

```ini
PASS_MAX_DAYS   90
PASS_MIN_DAYS   1
PASS_WARN_AGE   14
```

Password history (prevent reuse) via `/etc/pam.d/system-auth`:

```ini
password    requisite     pam_pwquality.so try_first_pass local_users_only retry=3
password    sufficient    pam_unix.so sha512 shadow use_authtok remember=5
```

---

## 7. Account Lockout (pam_faillock)

`/etc/pam.d/system-auth` and `/etc/pam.d/password-auth` (RHEL 8/9 use `authselect`, don't hand-edit if using it — create a custom profile instead):

```bash
authselect create-profile hardening -b sssd --symlink-meta
authselect select custom/hardening with-faillock
```

`/etc/security/faillock.conf`:

```ini
deny = 5
unlock_time = 900
fail_interval = 900
even_deny_root
```

Check and manage lockouts:

```bash
faillock --user <username>
faillock --user <username> --reset
```

---

## 8. sudo Hardening

Never edit `/etc/sudoers` directly — always use `visudo`.

```bash
visudo
```

Principles:

- No shared root logins. Every admin uses a named account + `sudo`.
- Scope grants to specific commands, not `ALL=(ALL) ALL`, wherever practical.
- Log everything, forward to SIEM.

```ini
# /etc/sudoers.d/logging
Defaults logfile="/var/log/sudo.log"
Defaults log_input, log_output
Defaults iolog_dir="/var/log/sudo-io"
Defaults requiretty
Defaults passwd_timeout=1
Defaults timestamp_timeout=5
```

Example scoped grant instead of full root:

```ini
# /etc/sudoers.d/dba-team
%dba_team  ALL=(postgres) /usr/bin/systemctl restart postgresql, /usr/bin/systemctl status postgresql
```

Audit existing grants periodically:

```bash
sudo -l -U <username>
```

---

## 9. Centralised Identity (IdM / AD / SSSD)

In an enterprise estate, local accounts should be the exception, not the rule.

- **Red Hat IdM (FreeIPA)** — native RHEL solution for centralised identity, HBAC (host-based access control), and centrally managed sudo rules.
- **SSSD** integrates RHEL with AD or LDAP for authentication, with group membership driving role-based access.

```bash
# Join to IdM domain
ipa-client-install --domain=example.com --realm=EXAMPLE.COM --mkhomedir

# Or join to AD via SSSD/realmd
realm join example.com -U administrator
```

Centralising sudo/HBAC rules in IdM means access changes are made once, centrally, and are auditable from a single console — critical for access review cycles in a regulated environment.

---

## 10. Auditd Configuration

Ensure `auditd` is enabled and running:

```bash
systemctl enable --now auditd
```

Key rules in `/etc/audit/rules.d/hardening.rules`:

```ini
# Identity changes
-w /etc/passwd -p wa -k identity
-w /etc/group -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/gshadow -p wa -k identity

# Privilege escalation
-w /etc/sudoers -p wa -k privilege_escalation
-w /etc/sudoers.d/ -p wa -k privilege_escalation

# Login/logout events
-w /var/log/faillog -p wa -k logins
-w /var/log/lastlog -p wa -k logins
-w /var/run/faillock -p wa -k logins

# SSH config changes
-w /etc/ssh/sshd_config -p wa -k sshd_config

# All root command execution
-a always,exit -F arch=b64 -S execve -F euid=0 -k root_actions

# Kernel module load/unload
-w /sbin/insmod -p x -k module_change
-w /sbin/rmmod -p x -k module_change
-w /sbin/modprobe -p x -k module_change

# Mandatory access control changes
-w /etc/selinux/ -p wa -k mac_policy

# Make the audit config itself immutable (must be last line, requires reboot to change)
-e 2
```

Load and query:

```bash
augenrules --load
ausearch -k identity
ausearch -k privilege_escalation --start today
aureport --summary
```

---

## 11. File Integrity Monitoring (AIDE)

```bash
dnf install -y aide
aide --init
mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz

# Run a check
aide --check

# Schedule daily via cron
echo "0 5 * * * root /usr/sbin/aide --check" > /etc/cron.d/aide-check
```

Forward AIDE output/alerts to your SIEM so unauthorised file changes surface immediately rather than at the next manual review.

---

## 12. Logging & SIEM Forwarding

Never rely on local logs alone for compliance — a compromised host can tamper with its own logs. Forward everything centrally.

`/etc/rsyslog.d/siem-forward.conf`:

```ini
*.* @@siem.internal.example.com:514
```

Ensure at minimum these are forwarded:

- `/var/log/secure` (auth events)
- `/var/log/sudo.log`
- audit logs (`auditd` can write directly to a remote log collector via `audisp-remote` plugin)
- AIDE integrity check output

---

## 13. Patch Management

- Use a controlled internal repo (Red Hat Satellite, Foreman, or a local mirror) — don't pull directly from public Red Hat repos in production.
- Test patches in a staging tier before production rollout, tied to a formal change management ticket.
- Track and enforce SLA-based patch timelines (example, adjust to your policy):
  - Critical CVEs: patched within 15 days
  - High: 30 days
  - Medium: 90 days

```bash
# List available security updates
dnf updateinfo list security

# Apply security-only updates
dnf upgrade --security -y

# Check for a specific CVE
dnf updateinfo info CVE-2024-XXXXX
```

---

## 14. Automated Compliance Scanning (OpenSCAP)

```bash
dnf install -y openscap-scanner scap-security-guide

# List available profiles for RHEL 9
oscap info /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# Run a CIS-aligned evaluation
oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_cis \
  --results /root/scap-results.xml \
  --report /root/scap-report.html \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# Auto-remediate where safe to do so (review before running in prod)
oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_cis \
  --remediate \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
```

The generated HTML report is a ready-made artefact for audit evidence — pass/fail per control, mapped directly to the benchmark.

---

## 15. Legal Banner / MOTD

`/etc/issue.net` (shown pre-authentication):

```
***************************************************************************
                            NOTICE TO USERS

This system is for authorised use only. By accessing this system you
consent to monitoring and recording of all activities. Unauthorised
access or use is strictly prohibited and may be subject to criminal
and/or civil penalties.
***************************************************************************
```

Reference it in `sshd_config` (`Banner /etc/issue.net`) — this is frequently a specific, named control in banking security policy documents, not just good practice.

---

## 16. Quick Reference Checklist

- [ ] Golden image / Kickstart baseline documented and version-controlled
- [ ] Separate partitions with `nodev,nosuid,noexec` where applicable
- [ ] Unused filesystem modules disabled
- [ ] Unnecessary services disabled/removed
- [ ] sysctl hardening applied (`/etc/sysctl.d/99-hardening.conf`)
- [ ] SSH hardened (no root password login, key-only, ciphers restricted)
- [ ] Password policy enforced (`pwquality`, `login.defs`, history/reuse)
- [ ] Account lockout configured (`faillock`)
- [ ] sudo scoped per role, logged, no shared root
- [ ] Centralised identity in place (IdM/AD/SSSD) — no unmanaged local accounts
- [ ] `auditd` running with rules for identity, privilege escalation, SSH, kernel modules
- [ ] Audit config marked immutable (`-e 2`) in production
- [ ] AIDE (or equivalent FIM) initialised and scheduled
- [ ] All logs forwarded to central SIEM
- [ ] Patch SLAs defined and tracked against CVE severity
- [ ] OpenSCAP scan run and report retained as audit evidence
- [ ] Legal banner configured pre-auth (SSH + console)
- [ ] All deviations from baseline documented and formally approved

---

*Reference: CIS Benchmarks for RHEL, DISA STIG for RHEL, Red Hat SCAP Security Guide. Always validate exact control numbers/values against the specific benchmark version your organisation mandates before applying to production systems.*
