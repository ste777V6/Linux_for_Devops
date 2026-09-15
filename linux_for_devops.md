# Linux_for_Devops

> A field guide for diagnosing, securing, and automating Linux systems — organized by mission.

---

## Mission 1: System Health & Monitoring
*Where am I, and is anything on fire?*

### First Look at the System

| Command | Key Flags | What It Does / Key Output |
|---|---|---|
| `uptime` | *(none)* | Server uptime + load averages. Load > core count = overloaded. Check user count for unexpected sessions. |
| `free -h` | `-h` `-m` `-s N` | RAM usage. Ignore `free` column. Check `available` — if <10% of total, trouble. High swap = out of memory. |
| `df -h` | `-h` `-T` `-i` | Disk usage. Use% >90% = danger. 100% = apps crash. Check `/` and `/var` first. |
| `top` | `P` `M` `1` `k` `q` | Live process monitor. Watch %CPU, %MEM, RES. `wa` in header >10% = disk bottleneck. |
| `htop` | `F5` `F9` `/` `F6` | Colorful `top`. Green=user, Red=kernel, Blue=low-pri. Install: `apt install htop` |
| `vmstat 1 5` | `interval count` | CPU/mem/IO snapshot. `wa`=disk, `si/so`=swap, `r`>cores=overloaded, `b`>0=IO blocked. |
| `iostat -x 1 3` | `-x` (extended) | Disk I/O detail. `await` <10ms=good, >50ms=slow. `%util`=100%=saturated. |

> **Tip:** Golden rule — high `wa` = disk problem, high `us` = app problem, high `sy` = kernel problem. If `si/so` are active, add RAM.

### History & Logs

| Command | Key Flags | What It Does / Key Output |
|---|---|---|
| `dmesg -T` | `-T` `\| grep -i` | Kernel messages. Look for: OOM killer, I/O errors, segfaults, network link changes. |
| `journalctl` | `-u svc` `-f` `-p err` `-b` | Systemd master log. `-p err -b` = all errors since boot. `-f` = follow live. `-u` = filter by service. |
| `sar` | `-u` `-r` `-n DEV` `-d` | Historical stats (DVR for server). Install `sysstat`. Records every 10min. Rewind to find 3AM crashes. |

### Services & Processes

| Command | Key Flags | What It Does / Key Output |
|---|---|---|
| `systemctl` | `status` `restart` `enable` | Control services. `list-units --failed` = find all broken. `daemon-reload` after editing `.service` files. |
| `ps aux` | `--sort=-%mem` `-%cpu` | All processes. `a`=all users, `u`=details, `x`=daemons. STAT: S=sleep, R=run, Z=zombie, D=IO wait. |

> **Tip:** Day 1 combo — `systemctl list-units --failed` + `journalctl -p err -b` = instant picture of everything broken.

---

## Mission 2: Logs, Files & Network
*The app is slow — find the problem.*

### Log Analysis & File Operations

| Command | Key Flags | What It Does / Key Output |
|---|---|---|
| `grep` | `-i` `-c` `-n` `-A3` `-B1` `-r` | Search text. `-c`=count, `-n`=line nums, `-A/B`=context, `-r`=recursive. Use `\|` for OR patterns. |
| `sed` | `-n 's/old/new/'` `-i` | Stream edit. `-i`=in-place (dry run first!). `s///g`=global replace. `/pat/d`=delete lines. |
| `awk` | `'{print $1}'` `-F':'` | Column extraction. `$0`=whole line. `\| sort \| uniq -c \| sort -rn` = frequency analysis. |
| `find` | `-name` `-type` `-mtime` `-size` | Find files. `-mtime +30`=older 30d. `-size +50M`=big. ALWAYS `-print` before `-delete`! |
| `du -sh` | `-s` `-h` `\| sort -rh` | Directory sizes. Pipe to `sort -rh \| head -10` for biggest dirs. |
| `xargs` | `-I {}` `-n1` `-P4` | Pipe to commands. `find \| xargs wc -l`. `-I {}` for placeholder. `-P4` for parallel. |

> **Tip:** `awk | sort | uniq -c | sort -rn` — the most powerful one-liner. Turns any log into a frequency table.
> **Tip:** `find . -type f -name "*.log" | xargs cat | grep "ERROR" | sort -k4 | uniq -f4 > errors.log
find . -type f -name` - Find all the logs file .lpg from the current directory with "ERROR" and sort them by the 4th column on the log (separated by space), remove duplicates (from column 4)

### Network Diagnostics

| Command | Key Flags | What It Does / Key Output |
|---|---|---|
| `curl` | `-I` `-s` `-o /dev/null` `-w` | HTTP client. `-w` shows timing (DNS, connect, total). `-I`=headers only. `-s`=silent. |
| `ss -tuln` | `-t` `-u` `-l` `-n` `-p` | Socket stats. `0.0.0.0:80`=all interfaces. `127.0.0.1`=localhost only. `-p` shows process. |
| `netstat -tuln` | *(same as `ss`)* | Older version of `ss`. Install: `apt install net-tools`. Know both. |
| `dig +short` | `@8.8.8.8` `+trace` | DNS lookup. Empty = DNS broken. Check `/etc/resolv.conf`. Half of issues are DNS. |
| `ip addr`/`route` | `addr show` `route show` | IP config. `state UP/DOWN` for interfaces. `default via` = gateway. |
| `ifconfig` | *(deprecated)* | Legacy IP config. Same as `ip addr`. Still in many docs. |
| `mtr` | `--report` `-c 10` | Traceroute + ping. Loss% >5% = problem hop. Avg jump = slow link. |
| `tcpdump` | `-i eth0 port X` `-c N` | Packet capture. `-c`=limit. `-w`=save for Wireshark. |
| `nmap` | `-p ports` `-p-` `-sV` | Port scanner. open/closed/filtered. `-sV`=version detection. |

> **Tip:** `curl -w 'DNS:%{time_namelookup} Connect:%{time_connect} Total:%{time_total}'` — pinpoints where slowness is.

---

## Mission 3: Security, Users & Processes
*Lock it down.*

### User & Permission Management

| Command | Key Flags | What It Does / Key Output |
|---|---|---|
| `id` | `id username` | Show user/group IDs. Root = uid=0. Groups determine file access. |
| `usermod -aG` | `-aG grp user` `-L` `-U` | Modify user. ALWAYS use `-aG` (append). Without `-a` = REPLACE all groups (disaster!). |
| `groupmod` | `-n newname old` | Rename group. Members stay intact. |
| `visudo` | *(safe editor)* | Edit `/etc/sudoers` with syntax checking. NEVER use vim/nano on sudoers directly. |
| `chmod` | `600` `640` `755` `777` | Change perms. r=4 w=2 x=1. 640=owner rw, group r. 777=NEVER in production. |
| `chown` | `user:group file` | Change ownership. Example: `chown root:sre-team config.env` |
| `umask` | `022` `027` | Default perms for new files. Umask removes perms. 027 = new files get 640. |

> **Tip:** Permission cheat — 600=private, 640=team-read, 755=public-exec, 777=NEVER. Always `chmod` BEFORE sharing configs.

### SSH & Firewall

| Command | Key Flags | What It Does / Key Output |
|---|---|---|
| `sshd_config` | `PermitRootLogin` etc. | Check `/etc/ssh/sshd_config`. Set `PermitRootLogin` + `PasswordAuth` to `no`. Add SSH key FIRST! |
| `iptables` | `-L` `-n` `-v` `-A` `-j` | Firewall rules. `-j ACCEPT/DROP/REJECT`. Rules are top-to-bottom — ORDER MATTERS. |

### Process Control

| Command | Key Flags | What It Does / Key Output |
|---|---|---|
| `kill` | `-15` `-9` `-1` | SIGTERM(-15)=graceful. SIGKILL(-9)=force, last resort. SIGHUP(-1)=reload config. |
| `pkill`/`pgrep` | `-la` `-f` `-u user` | Kill/find by name. `-f` matches full command line. `-u` filters by user. |
| `nice`/`renice` | `-n VAL` (-20 to 19) | Process priority. `nice`=at start, `renice -p PID`=while running. Lower = higher priority. |
| `lsof` | `-p PID` `-i :port` `/path` | Open files/ports. `lsof -i :80` = what uses port 80. Solves "port in use" instantly. |
| `strace` | `-p PID` `-e trace` `-c` `-f` | `-c` = summary table (hidden gem). Shows call counts, errors, time. Reveals hidden failures. |

> **Tip:** `strace -c` shows EXACTLY where a process spends time. The "errors" column reveals failures the app doesn't log.

---

## Mission 4: Deploy, Backup & Automate
*Never do this manually again.*

### File Operations & Deployment

| Command | Key Flags | What It Does / Key Output |
|---|---|---|
| `tar` | `-czf` `-xzf` `-tzf` `-C` | Archive/compress. `czf`=Create+gZip+File. `-C`=extract to dir. `-v`=verbose. |
| `rsync` | `-avz` `--dry-run` `--delete` | Smart sync. `-a`=preserves all. `--dry-run` FIRST! Trailing slash: `/src/` vs `/src` matters! |
| `tmux` | `new -s` `attach -t` | Terminal multiplexer. SSH dies? tmux survives. `Ctrl+B D`=detach. `%`=split vertical. |

> **Tip:** rsync trailing slash — `/src/` copies CONTENTS. `/src` (no slash) copies the DIRECTORY itself. This trips up everyone.

### Automation

| Command | Key Flags | What It Does / Key Output |
|---|---|---|
| `crontab` | `-e` `-l` | Schedule jobs. `0 2 * * *`=2AM daily. `*/15`=every 15min. ALWAYS add `>> log 2>&1` for logging. |
| `logrotate` | `daily` `rotate N` `compress` | Auto-manage log files. `-f`=force now. Config in `/etc/logrotate.d/`. Prevents disks filling up. |

#### Cron Quick Reference

| Expression | Meaning |
|---|---|
| `0 2 * * *` | Every day at 2:00 AM |
| `*/15 * * * *` | Every 15 minutes |
| `0 9 * * 1-5` | 9 AM, Monday through Friday |
| `0 0 1 * *` | Midnight on 1st of every month |

### Disk & Storage

| Command | Key Flags | What It Does / Key Output |
|---|---|---|
| `lsblk` | *(none)* | List block devices. NAME, SIZE, TYPE (disk/part/lvm), MOUNTPOINT (empty=not mounted). |
| `fdisk -l` | `-l` (list) | List all partitions. Use `fdisk` for <2TB, `parted` for larger. |
| `mount`/`umount` | `mount dev /path` | Make partitions accessible. Add to `/etc/fstab` for persistence across reboots. |

### Deep System Tools

| Command | Key Flags | What It Does / Key Output |
|---|---|---|
| `chroot` | `/mnt/recovery /bin/bash` | Change root dir. System recovery tool. Docker uses this concept under the hood. |
| `ldd` | `/usr/bin/nginx` | Show shared libraries. "not found" = missing dependency causing app startup failure. |
| `udevadm` | `info --query=all` | Device event manager. Mostly bare-metal/embedded DevOps. |

> **Tip:** When an app says "error loading shared libraries," run `ldd` on the binary. It shows exactly which library is missing.

---

## Complete Command Index
*All 50 commands at a glance — sorted by mission*

| # | Command | Category | # | Command | Category |
|---|---|---|---|---|---|
| 1 | `uptime` | System | 26 | `tcpdump` | Network |
| 2 | `free` | System | 27 | `nmap` | Network |
| 3 | `df` | System | 28 | `id` | Users |
| 4 | `top` | System | 29 | `usermod` | Users |
| 5 | `htop` | System | 30 | `groupmod` | Users |
| 6 | `vmstat` | System | 31 | `visudo` | Users |
| 7 | `iostat` | System | 32 | `chmod`/`chown` | Perms |
| 8 | `dmesg` | History | 33 | `umask` | Perms |
| 9 | `journalctl` | History | 34 | `sshd` | SSH |
| 10 | `sar` | History | 35 | `iptables` | Firewall |
| 11 | `systemctl` | Services | 36 | `kill` | Process |
| 12 | `ps aux` | Services | 37 | `pkill`/`pgrep` | Process |
| 13 | `grep` | Logs | 38 | `nice`/`renice` | Process |
| 14 | `sed` | Logs | 39 | `lsof` | Process |
| 15 | `awk` | Logs | 40 | `strace` | Process |
| 16 | `find` | Files | 41 | `tar` | Deploy |
| 17 | `du` | Files | 42 | `rsync` | Deploy |
| 18 | `xargs` | Files | 43 | `tmux` | Deploy |
| 19 | `curl`/`wget` | Network | 44 | `crontab` | Automate |
| 20 | `ss` | Network | 45 | `logrotate` | Automate |
| 21 | `netstat` | Network | 46 | `lsblk` | Storage |
| 22 | `dig` | Network | 47 | `fdisk` | Storage |
| 23 | `ip addr` | Network | 48 | `mount` | Storage |
| 24 | `ifconfig` | Network | 49 | `chroot` | Deep |
| 25 | `mtr` | Network | 50 | `ldd` | Deep |

---

## Emergency Quick Reference

| Situation | Run This |
|---|---|
| Server slow | `top` → check `wa%` → `iostat -x 1 3` → `free -h` |
| Disk full | `df -h` → `du -sh /* \| sort -rh \| head` → `find / -size +100M` |
| App crashed | `systemctl status app` → `journalctl -u app -n 50` → `dmesg \| tail` |
| Can't connect | `dig host +short` → `ss -tuln` → `iptables -L -n` → `curl -I url` |
| Port in use | `lsof -i :PORT` → `kill PID` (or `kill -9 PID`) |
| Out of memory | `free -h` → `ps aux --sort=-%mem \| head` → `dmesg \| grep oom` |
| Who did what | `journalctl -p err -b` → `last` → `grep -i error /var/log/syslog` |

---

*Original source: **50 Linux Commands for DevOps** by Piyush Sachdeva*
*YouTube & Instagram: [@Techtutorialswithpiyush](https://www.youtube.com/@Techtutorialswithpiyush) · LinkedIn: [piyush-sachdeva](https://www.linkedin.com/in/piyush-sachdeva)*
