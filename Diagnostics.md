# Linux Server Diagnostics Cheat Sheet

A quick reference for diagnosing unexpected server reboots, PM2 issues, system failures, resource problems, and package updates on Ubuntu/Debian-based systems.

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

### Check unattended upgrade activity

```bash
grep -R "unattended" /var/log/apt/
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

---

# Common Root Causes

| Symptom | Likely Cause |
|---------|--------------|
| Hypervisor initiated shutdown | VPS provider rebooted the server |
| `guest-shutdown called` | Host requested a graceful shutdown |
| `Kernel panic` | Kernel crash |
| `OOM Killer` | System ran out of memory |
| Empty `pm2 list` after reboot | PM2 startup or `pm2 save` not configured |
| `dump.pm2` missing | No saved PM2 process list |
| `systemctl --failed` shows services | Services failed to start after boot |

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
