# Linux Server Diagnostics Cheat Sheet

A quick reference for diagnosing unexpected server reboots, PM2 issues, system failures, resource problems, and package updates on Ubuntu/Debian-based systems.

---

## Table of Contents

1. [Server Status](#1-server-status)
2. [Previous Boot Logs](#2-previous-boot-logs)
3. [Determine Who Initiated the Reboot](#3-determine-who-initiated-the-reboot)
4. [PM2 Diagnostics](#4-pm2-diagnostics)
5. [PM2 Auto Startup](#5-pm2-auto-startup)
6. [System Services](#6-system-services)
7. [Memory & OOM Diagnostics](#7-memory--oom-diagnostics)
8. [Disk Usage](#8-disk-usage)
9. [Package Updates](#9-package-updates)
10. [Live Monitoring](#10-live-monitoring)
11. [Hypervisor Detection](#11-hypervisor-detection)
12. [Shell History](#12-shell-history)
13. [Swap Diagnostics](#13-swap-diagnostics)
14. [CPU & Load Diagnostics](#14-cpu--load-diagnostics)
15. [Network Diagnostics](#15-network-diagnostics)
16. [Disk I/O & Health](#16-disk-io--health)
17. [Cron & Scheduled Task Diagnostics](#17-cron--scheduled-task-diagnostics)
18. [Boot Time Diagnostics](#18-boot-time-diagnostics)
19. [Security & Auth Diagnostics](#19-security--auth-diagnostics)
20. [Nginx & Certbot Diagnostics](#20-nginx--certbot-diagnostics)
21. [Quick Health Check Script](#21-quick-health-check-script)
22. [Common Root Causes](#common-root-causes)
23. [Recommended PM2 Setup](#recommended-pm2-setup)

---

# 1. Server Status

### Check reboot history

```bash
last reboot
```

### Check current uptime

```bash
uptime
```

### Check when the current boot started

```bash
who -b
```

### Check current system date/time and timezone (rules out log timestamp confusion)

```bash
timedatectl
```

---

# 2. Previous Boot Logs

### View the last 200 lines before the previous shutdown

```bash
journalctl -b -1 -n 200
```

### View the last 100 shutdown log lines

```bash
journalctl -b -1 | tail -100
```

### Search for reboot/shutdown events

```bash
journalctl -b -1 | grep -Ei "reboot|shutdown|poweroff|halt"
```

### Search for kernel panic, OOM, or crashes

```bash
journalctl -b -1 | grep -Ei "panic|oom|segfault|BUG|Call Trace"
```

### View kernel logs from the previous boot

```bash
journalctl -k -b -1
```

### List all available boots (useful if the server has crashed more than once)

```bash
journalctl --list-boots
```

### View logs from a specific boot by offset (e.g., two boots ago)

```bash
journalctl -b -2 -n 200
```

---

# 3. Determine Who Initiated the Reboot

### Search for reboot commands or shutdown requests

```bash
journalctl -b -1 | grep -Ei "systemctl reboot|shutdown|guest-shutdown"
```

### Check if the hypervisor initiated the shutdown

```bash
journalctl -b -1 | grep -i hypervisor
```

### Check authentication logs around a specific time

Replace the timestamp as needed.

```bash
grep "Jul  7 10:4" /var/log/auth.log
```

### Search rotated authentication logs

```bash
zgrep "Jul  7 10:4" /var/log/auth.log*
```

### View recent logins

```bash
last
```

### View recently created or modified sudoers/user activity (rules out manual admin action)

```bash
last -x | grep -Ei "shutdown|runlevel"
```

---

# 4. PM2 Diagnostics

### Verify PM2 is installed

```bash
which pm2
pm2 -v
```

### View running PM2 applications

```bash
pm2 list
```

### View the PM2 daemon log

```bash
tail -100 ~/.pm2/pm2.log
```

### Inspect the PM2 directory

```bash
ls -la ~/.pm2
```

### Check if a saved process list exists

```bash
ls -l ~/.pm2/dump.pm2
```

### Restore saved applications

```bash
pm2 resurrect
```

### Save the current running applications

```bash
pm2 save
```

### View per-app error and output logs directly

```bash
pm2 logs [app-name] --lines 200 --err
pm2 logs [app-name] --lines 200 --out
```

### Check per-process memory/CPU usage and restart counts

```bash
pm2 monit
pm2 describe [app-name]
```

A high **restart count** in `pm2 list` alongside OOM entries in `dmesg` usually means the app itself is being killed for using too much memory — see [Section 7](#7-memory--oom-diagnostics) and [Section 13](#13-swap-diagnostics).

### Flush old PM2 logs (they can silently fill up disk on low-storage VPS)

```bash
pm2 flush
```

---

# 5. PM2 Auto Startup

### Generate the startup service

```bash
pm2 startup
```

### Check the PM2 systemd service

```bash
systemctl status pm2-root
```

### List PM2-related services

```bash
systemctl list-unit-files | grep pm2
```

### Confirm PM2 startup service is enabled to run on boot

```bash
systemctl is-enabled pm2-root
```

---

# 6. System Services

### View failed services

```bash
systemctl --failed
```

### Check the status of common services

```bash
systemctl status nginx
systemctl status ssh
```

### List services that restarted recently (possible instability indicator)

```bash
systemctl list-units --type=service --state=running | grep -v "1)"
```

### Check a specific service's recent logs

```bash
journalctl -u [service-name] -n 100 --no-pager
```

---

# 7. Memory & OOM Diagnostics

### View memory usage

```bash
free -h
```

### Search for Out-of-Memory (OOM) killer events

```bash
dmesg | grep -i oom
```

```bash
journalctl -k | grep -i oom
```

### Get full context around an OOM kill (which process, how much memory it wanted)

```bash
dmesg -T | grep -i -A 5 -B 5 "killed process"
```

### Identify the top memory-consuming processes right now

```bash
ps aux --sort=-%mem | head -15
```

### Identify the single largest memory consumer (fast one-liner)

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head -5
```

### Check available memory over time (spot slow leaks vs sudden spikes)

```bash
vmstat 2 10
```

### Check per-process memory in detail (RSS vs VSZ)

```bash
smem -r -k 2>/dev/null || ps aux --sort=-rss | head -15
```

> **Note:** `smem` gives more accurate shared-memory-aware numbers but may need `sudo apt install smem` first. The `ps aux --sort=-rss` fallback works everywhere.

---

# 8. Disk Usage

### View filesystem usage

```bash
df -h
```

### Check log directory sizes

```bash
du -sh /var/log/*
```

### Find the largest files/directories on disk (helps when df shows near-full disk)

```bash
du -ahx / 2>/dev/null | sort -rh | head -20
```

### Check inode usage (disk can be "full" on inodes even with free space)

```bash
df -ih
```

---

# 9. Package Updates

### Check if package updates requested a reboot

```bash
grep -i reboot /var/log/apt/history.log
```

### Search for reboot-required messages

```bash
grep -R "Reboot required" /var/log/apt/
```

### Check if a reboot is currently pending

```bash
[ -f /var/run/reboot-required ] && cat /var/run/reboot-required
```

### Check unattended upgrade activity

```bash
grep -R "unattended" /var/log/apt/
```

### List packages upgraded around a specific date (correlate with a crash)

```bash
grep "2026-07-07" /var/log/dpkg.log
```

---

# 10. Live Monitoring

### Follow the system journal

```bash
journalctl -f
```

### Follow syslog

```bash
tail -f /var/log/syslog
```

### Follow the PM2 daemon log

```bash
tail -f ~/.pm2/pm2.log
```

### Live process/resource monitor (interactive)

```bash
htop 2>/dev/null || top
```

---

# 11. Hypervisor Detection

Determine whether the VPS provider or virtualization platform initiated the shutdown.

```bash
journalctl -b -1 | grep -Ei "guest-shutdown|hypervisor|qemu-ga"
```

Typical output indicating a provider-initiated reboot:

```
guest-shutdown called
System is powering down (hypervisor initiated shutdown)
```

### Identify the virtualization platform you're running on

```bash
systemd-detect-virt
```

---

# 12. Shell History

### Search for PM2 commands

```bash
history | grep pm2
```

### Search for reboot commands

```bash
history | grep reboot
```

### Search for shutdown commands

```bash
history | grep shutdown
```

### View the last 100 executed commands

```bash
history | tail -100
```

### Search all users' bash history for reboot-related commands (root-only)

```bash
sudo find /home /root -name ".bash_history" -exec grep -Hi "reboot\|shutdown" {} \;
```

---

# 13. Swap Diagnostics

Swap issues are one of the most common causes of unexpected kills on small (1-2GB RAM) VPS instances, especially during `npm run build` / `bun run build`.

### Check whether swap is configured and active

```bash
swapon --show
free -h
```

### Check swap usage in detail

```bash
cat /proc/swaps
```

### Check current swappiness value

```bash
cat /proc/sys/vm/swappiness
```

### Check if swap is defined in fstab (persists across reboot)

```bash
grep -i swap /etc/fstab
```

### Confirm if a build/process died from lack of swap + low RAM

```bash
dmesg -T | grep -i "killed process" && free -h
```

If `swapon --show` returns nothing and `free -h` shows 0 in the Swap row, the server has **no swap configured** — this is the most common root cause of builds being OOM-killed on 1-2GB RAM servers. See the deployment guide's swap setup steps to fix this.

---

# 14. CPU & Load Diagnostics

### Check current load average (1, 5, 15 min)

```bash
uptime
```

A load average consistently higher than the number of CPU cores indicates the server is CPU-bound.

### Check number of CPU cores available

```bash
nproc
```

### Identify top CPU-consuming processes

```bash
ps aux --sort=-%cpu | head -15
```

### Check for CPU throttling (common on burstable/shared VPS plans)

```bash
cat /proc/cpuinfo | grep -i mhz
```

---

# 15. Network Diagnostics

### Check listening ports and which process owns them

```bash
sudo ss -tulpn
```

### Check active connections to the server

```bash
sudo ss -antp | grep ESTAB
```

### Test DNS resolution

```bash
nslookup [your-domain]
```

### Test connectivity to a specific port (e.g., confirm Nginx/PM2 app is reachable)

```bash
curl -I http://localhost:3000
```

### Check for dropped packets or interface errors

```bash
ip -s link
```

---

# 16. Disk I/O & Health

### Check real-time disk I/O per process (helps spot a process hammering disk)

```bash
sudo iotop -o 2>/dev/null || echo "install with: sudo apt install iotop"
```

### Check overall disk I/O stats

```bash
iostat -x 2 5 2>/dev/null || echo "install with: sudo apt install sysstat"
```

### Check disk health/SMART status (physical/dedicated servers; not always available on VPS)

```bash
sudo smartctl -H /dev/sda 2>/dev/null || echo "install with: sudo apt install smartmontools"
```

---

# 17. Cron & Scheduled Task Diagnostics

Scheduled jobs (backups, log rotation, certbot renewal) are a common hidden cause of memory spikes or reboots at a consistent time of day.

### List current user's cron jobs

```bash
crontab -l
```

### List system-wide cron jobs

```bash
cat /etc/crontab
ls /etc/cron.d/
```

### Check systemd timers (modern alternative to cron)

```bash
systemctl list-timers --all
```

### Correlate a crash time with a scheduled job

```bash
grep "07:00" /var/log/syslog
```

---

# 18. Boot Time Diagnostics

### Check how long the last boot took, and what took the longest

```bash
systemd-analyze
systemd-analyze blame
```

### Check the critical-path chain of slow startup services

```bash
systemd-analyze critical-chain
```

---

# 19. Security & Auth Diagnostics

### Check for repeated failed SSH login attempts (possible brute-force triggering fail2ban/reboots)

```bash
grep -i "failed password" /var/log/auth.log | tail -30
```

### Check fail2ban status (if installed)

```bash
sudo fail2ban-client status 2>/dev/null || echo "fail2ban not installed"
```

### Check currently banned IPs

```bash
sudo fail2ban-client status sshd 2>/dev/null
```

### Check firewall rules (UFW)

```bash
sudo ufw status verbose
```

---

# 20. Nginx & Certbot Diagnostics

### Test Nginx config validity

```bash
sudo nginx -t
```

### Check Nginx error log

```bash
sudo tail -100 /var/log/nginx/error.log
```

### Check Nginx access log for a spike in traffic (possible resource exhaustion cause)

```bash
sudo tail -100 /var/log/nginx/access.log
```

### Check SSL certificate expiry and renewal status

```bash
sudo certbot certificates
```

### Dry-run a renewal to check for config issues without actually renewing

```bash
sudo certbot renew --dry-run
```

---

# 21. Quick Health Check Script

Run this one-liner block for a fast overall snapshot when you first SSH in after an incident:

```bash
echo "=== UPTIME ===" && uptime && \
echo "=== MEMORY ===" && free -h && \
echo "=== SWAP ===" && swapon --show && \
echo "=== DISK ===" && df -h && \
echo "=== FAILED SERVICES ===" && systemctl --failed && \
echo "=== RECENT OOM ===" && dmesg -T | grep -i "killed process" | tail -5 && \
echo "=== PM2 LIST ===" && pm2 list
```

---

# Common Root Causes

| Symptom | Likely Cause |
|---------|--------------|
| Hypervisor initiated shutdown | VPS provider rebooted the server |
| `guest-shutdown called` | Host requested a graceful shutdown |
| `Kernel panic` | Kernel crash |
| `OOM Killer` | System ran out of memory |
| No output from `swapon --show` + OOM entries | No swap configured on a low-RAM server |
| Empty `pm2 list` after reboot | PM2 startup or `pm2 save` not configured |
| `dump.pm2` missing | No saved PM2 process list |
| `systemctl --failed` shows services | Services failed to start after boot |
| High restart count in `pm2 list` | App is crashing/being OOM-killed repeatedly |
| Disk full but `df -h` shows space | Inode exhaustion — check `df -ih` |
| Crash at a consistent time of day | A cron job or systemd timer is the trigger |
| Repeated `Failed password` entries | Brute-force attempt, possibly triggering fail2ban actions |

---

# Recommended PM2 Setup

After deploying your applications:

```bash
pm2 save
pm2 startup
```

Execute the command printed by `pm2 startup`, then save again:

```bash
pm2 save
```

This ensures your PM2-managed applications are automatically restored after a server reboot.

## Recommended Baseline for Low-RAM VPS (1-2GB)

To prevent most of the issues this cheat sheet diagnoses, do this once during initial server setup:

```bash
# 1. Configure swap (see deployment guide)
# 2. Enable PM2 startup + save
pm2 save
pm2 startup
pm2 save

# 3. Confirm no reboot is pending from unattended upgrades
[ -f /var/run/reboot-required ] && echo "Reboot needed" || echo "No reboot pending"

# 4. Confirm swap and PM2 are both active
swapon --show && pm2 list
```
