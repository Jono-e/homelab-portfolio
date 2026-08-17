[linux-lab-README.md](https://github.com/user-attachments/files/31137358/linux-lab-README.md)
# 🐧 Linux Sysadmin Lab

A structured, hands-on Linux server administration lab built on Ubuntu Server 26.04 LTS running in VirtualBox. Covers the core skills required for Linux support and sysadmin roles, with real troubleshooting tickets worked through from diagnosis to resolution.

---

## Environment

- **OS**: Ubuntu Server 26.04 LTS (headless, SSH-only)
- **Hypervisor**: VirtualBox 7.x on Windows 11
- **Access**: SSH via key-based authentication from Windows host
- **Networking**: Bridged adapter (VM accessible on home network)

---

## Modules Completed

### Module 1: Users, Groups & Permissions

Built and managed a multi-user environment simulating a real team setup.

**What I did:**
- Created users (`jdoe`, `asmith`) and groups (`developers`)
- Configured a shared team directory (`/srv/teamshare`) with group-based permissions (`chmod 770`, `chown root:developers`)
- Diagnosed and reproduced a silent permission bug using `usermod -G` (without `-a`), which silently strips a user's existing group memberships

**Key finding — Ticket #1:**
> "User can't access shared folder despite being added to the group"

Root cause: `usermod -G developers asmith` replaces the user's entire supplementary group list instead of appending. The correct command is `usermod -aG`. This produced no error, no warning — the bug was only visible by comparing `groups asmith` before and after. A great example of a silent failure that causes real-world support tickets.

**Commands used:** `adduser`, `groupadd`, `usermod -aG`, `groups`, `chmod`, `chown`, `ls -ld`, `cat /etc/passwd`

---

### Module 2: Package Management & Services (Nginx)

Installed and managed Nginx as a real network service, then diagnosed two distinct failure modes.

**What I did:**
- Updated package index and installed Nginx via `apt`
- Verified service health using `systemctl status` — reading Active state, ExecStartPre results, CGroup process tree, and embedded logs
- Stopped/started the service and observed state transitions

**Key finding — Ticket #2: Config syntax error**
> "Site is down after a config change"

Introduced a deliberate syntax error (missing semicolon) in `/etc/nginx/sites-available/default`. Diagnosed via:
1. `systemctl restart nginx` — failed with exit code
2. `systemctl status nginx` — `ExecStartPre` showed `status=1/FAILURE` (config test failed before the process even started)
3. `journalctl -xeu nginx` — revealed the exact file and error type
4. `nginx -t` — confirmed the syntax error with line reference

Fix: corrected the config, ran `nginx -t` to validate before restarting. Key lesson: always validate config before reloading a service.

**Key finding — Ticket #3: Port conflict**
> "Nginx won't start after reboot"

A rogue Python process was occupying port 80. Diagnosed via:
1. `ExecStart` failed (not `ExecStartPre`) — this meant the config was fine, the process itself couldn't bind
2. `bind() to 0.0.0.0:80 failed (98: Address already in use)` in journalctl
3. `ss -tlnp | grep :80` — identified `python3` (PID) as the offending process
4. `kill <PID>` — freed the port, Nginx restarted cleanly

Key lesson: **timeout vs refused** — a connection timeout indicates a firewall dropping packets; connection refused indicates nothing listening on the port or the process actively rejecting it. Knowing which one you're seeing immediately narrows the diagnosis.

**Commands used:** `apt update`, `apt install`, `systemctl status/start/stop/restart/enable`, `journalctl -xeu`, `nginx -t`, `ss -tlnp`, `curl`

---

### Module 3: systemd Deep Dive & Custom Services

Wrote a custom systemd service from scratch and tested self-healing/auto-restart behavior.

**What I did:**
- Wrote a bash heartbeat script (`/usr/local/bin/heartbeat.sh`) that logs a timestamped entry to `/var/log/heartbeat.log` every 5 seconds
- Created a full systemd unit file (`/etc/systemd/system/heartbeat.service`) with:
  - `After=network.target` — ordering dependency
  - `Restart=always` / `RestartSec=3` — auto-restart on crash
  - `WantedBy=multi-user.target` — boots with the system
- Ran `systemctl daemon-reload` (required after any unit file change)
- Enabled the service (`systemctl enable`) and confirmed it survived a reboot

**Self-healing test:**
Killed the main process PID directly (`kill <PID>`) to simulate a crash. Within 3 seconds, `systemctl status heartbeat` showed a new PID and the line:
```
heartbeat.service: Scheduled restart job, restart counter is at 1.
```
This confirmed systemd detected the crash and automatically restarted the service — without any manual intervention.

**Key distinction — start vs enable:**
`systemctl start` = run now. `systemctl enable` = run on every boot. They are independent switches. A service can be started but not enabled (won't survive reboot) or enabled but not started (won't run until next boot).

**Commands used:** `systemctl daemon-reload`, `systemctl enable`, `systemctl status`, `kill`, `journalctl -u`

---

### Module 4: Networking & Firewall (UFW)

Configured and managed the UFW firewall, and diagnosed a real port-blocking scenario.

**What I did:**
- Inspected network config with `ip a`, `ip route`, `cat /etc/resolv.conf`
- Enabled UFW with a default-deny incoming policy
- Allowed SSH **before** enabling UFW (critical — forgetting this locks you out of a remote server)
- Diagnosed a firewall-blocked port scenario

**Key finding — Ticket #4: Firewall blocking port**
> "Can't reach the internal admin panel"

A test service running on port 8080 was unreachable. Browser showed a **timeout** (not "connection refused"). This distinction is important:
- **Timeout** = firewall silently dropping packets (the server never responds)
- **Connection refused** = nothing listening on that port (the OS actively rejects)

Confirmed with `ss -tlnp | grep :8080` — the service was running fine. Root cause: UFW's default-deny policy was blocking port 8080. Fix: `ufw allow 8080/tcp`.

**Commands used:** `ip a`, `ip route`, `ufw enable/status/allow/delete`, `ss -tlnp`

---

### Module 5: Logs & Troubleshooting

Diagnosed a disk space incident from a vague "server is acting weird" report.

**What I did:**
- Learned log locations: `/var/log/syslog`, `/var/log/auth.log`, `/var/log/nginx/`, journald
- Used `journalctl` filtering: `--since`, `-p err`, `-f` (follow mode)

**Key finding — Ticket #5: Full disk**
> "Server is slow, things seem to be failing randomly"

No obvious error clues in logs. First check: `df -h` — root filesystem at 93% used. Then `du -sh /var/*` — found a 5.1GB file (`/var/junkfile.img`) dwarfing everything else.

Key lesson: a full disk rarely throws an obvious "disk full" error in logs. It causes indirect failures (services can't write PID files, logs fail to rotate, installs break) that look like unrelated issues. Checking disk space early in any "something's wrong" investigation is a cheap, high-value first step.

**Bonus — Log rotation (logrotate):**
Set up log rotation for the custom heartbeat log (`/var/log/heartbeat.log`), which was growing unbounded. Configured `/etc/logrotate.d/heartbeat` with:
- `daily` rotation, `rotate 7` (7 days history), `compress`, `copytruncate`
- Fixed a `su root syslog` directive requirement caused by `/var/log` being group-writable by the `syslog` group (standard Ubuntu default)

**Commands used:** `df -h`, `du -sh`, `journalctl`, `logrotate -f`, `rm`

---

### Module 6: Shell Scripting & Automation (Cron)

Built a practical disk monitoring script and scheduled it with cron.

**Script: `/usr/local/bin/disk_check.sh`**
```bash
#!/bin/bash
THRESHOLD=80
USAGE=$(df / --output=pcent | tail -1 | tr -d '% ')
if [ "$USAGE" -ge "$THRESHOLD" ]; then
    echo "$(date): WARNING - Disk usage at ${USAGE}% (threshold: ${THRESHOLD}%)" >> /var/log/disk_check.log
else
    echo "$(date): OK - Disk usage at ${USAGE}%" >> /var/log/disk_check.log
fi
```

**What I did:**
- Wrote and tested the script — confirmed both the OK and WARNING branches with threshold manipulation
- Scheduled it with cron: `*/15 * * * * /usr/local/bin/disk_check.sh`
- Verified cron executed it automatically by watching for new timestamped log entries without manually triggering the script

**Commands used:** `chmod +x`, `crontab -e`, `crontab -l`, `cat /var/log/disk_check.log`

---

## Interview Talking Points

| Scenario | What I'd say |
|---|---|
| "Tell me about a permissions issue you solved" | Module 1 — silent group membership stripping via `usermod -G` without `-a` |
| "Tell me about a service outage you diagnosed" | Module 2 — config syntax error caught by `ExecStartPre` failure pattern and `nginx -t` |
| "Have you worked with systemd?" | Module 3 — wrote a custom unit file with auto-restart, tested crash recovery via restart counter |
| "Tell me about a networking/firewall issue" | Module 4 — timeout vs refused distinction, UFW misconfiguration |
| "Have you troubleshot performance issues?" | Module 5 — disk space incident, indirect failures from a full filesystem |
| "Can you write bash scripts?" | Module 6 — disk monitoring script with threshold logic, scheduled via cron |

---

## Skills Demonstrated

`Linux` `Ubuntu Server` `systemd` `Nginx` `UFW` `Bash scripting` `Cron` `Package management (apt)` `User/group management` `File permissions` `Log analysis` `journalctl` `SSH` `VirtualBox`
