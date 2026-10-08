# Linux Essentials for DevOps

> **TL;DR:** Linux is the substrate of every container, VM, and CI runner. You must read permissions, kill the right process, find a runaway port, and tail the right log — under pressure.

## Filesystem & Permissions

```bash
# Octal permission cheat sheet
# r=4 w=2 x=1
# 755 = rwxr-xr-x  (typical dir / executable)
# 644 = rw-r--r--  (typical file)
# 600 = rw-------  (ssh keys, secrets)

chmod 600 ~/.ssh/id_ed25519       # private key — must be 600
chmod -R 755 /var/www/html        # web root
chown -R www-data:www-data /var/www/html

# Setuid / setgid / sticky (advanced)
# 4xxx = setuid (run as file owner)  -> chmod 4755 /usr/bin/passwd
# 2xxx = setgid (inherit group)
# 1xxx = sticky bit                  -> chmod 1777 /tmp
```

ACL extras (when octal isn't enough):

```bash
setfacl -m u:deploy:rwx /var/log/app
getfacl /var/log/app
```

## Processes & Signals

```bash
ps aux | grep nginx                # classic
ps -ef --forest                    # process tree
pstree -p                          # same, pretty

top                                # interactive
htop                               # nicer interactive
top -p $(pgrep -d, -f myapp)       # filter to one app

# Background, foreground, disown
long-job &                         # background, attached to shell
disown %1                          # detach from shell (survives logout)
nohup long-job > out.log 2>&1 &    # detach + redirect

jobs                               # list shell jobs
fg %1                              # bring to foreground
```

### Signals — the ones you must know

| Signal | Number | Use |
|--------|--------|-----|
| SIGHUP | 1 | Reload config (most daemons) |
| SIGINT | 2 | Ctrl+C |
| SIGKILL | 9 | Cannot be caught — last resort |
| SIGTERM | 15 | Polite stop (default `kill`) |
| SIGUSR1/2 | 10/12 | App-defined (e.g., logrotate) |

```bash
kill -TERM 1234                    # ask politely
kill -KILL 1234                    # force (kernel-level kill)
kill -HUP $(pidof nginx)           # reload nginx config
pkill -f "java -jar myapp.jar"     # by command pattern
```

**Gotcha:** SIGKILL skips userspace cleanup — open file handles flush, but app shutdown hooks do **not** run. Containers with `docker stop` send SIGTERM then SIGKILL after `--time=10`.

## systemd — the Init System You'll Actually See

```bash
systemctl status nginx
systemctl start|stop|restart|reload nginx
systemctl enable --now nginx       # start now and on boot
systemctl daemon-reload            # after editing unit files
systemctl list-units --failed      # what's broken?

journalctl -u nginx -f             # tail unit logs
journalctl -u nginx --since "10 min ago"
journalctl -p err -b               # errors since last boot
```

A minimal service unit:

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My App
After=network.target

[Service]
Type=simple
User=myapp
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/server
Restart=on-failure
RestartSec=5
Environment=NODE_ENV=production
EnvironmentFile=-/etc/myapp/env    # leading - = optional

[Install]
WantedBy=multi-user.target
```

## Package Managers

```bash
# Debian / Ubuntu
apt update && apt upgrade -y
apt install -y nginx
apt list --installed | grep ssl
dpkg -l | grep openssl
apt-cache policy openssl           # which version is pinned?

# RHEL / CentOS / Amazon Linux
dnf install -y nginx               # newer; yum is legacy alias
rpm -qa | grep ssl
dnf list installed
dnf history                        # what was installed when, by whom

# Alpine (your container friend)
apk add --no-cache curl bash
apk del curl
```

## SSH — Beyond `ssh user@host`

```bash
# Key gen
ssh-keygen -t ed25519 -C "kushal@laptop" -f ~/.ssh/id_ed25519
ssh-copy-id user@server            # appends to ~/.ssh/authorized_keys

# Config file is a force multiplier
# ~/.ssh/config
Host prod-bastion
    HostName bastion.example.com
    User deploy
    IdentityFile ~/.ssh/id_ed25519_prod
    ForwardAgent yes

Host prod-*
    ProxyJump prod-bastion
    User deploy

# Then: ssh prod-db1  (proxies via bastion)

# Tunnels
ssh -L 5432:db.internal:5432 bastion   # local fwd: localhost:5432 -> db
ssh -R 8080:localhost:3000 jumphost    # reverse fwd
ssh -D 1080 jumphost                   # SOCKS proxy

# rsync — your scp replacement
rsync -avzP --delete src/ user@host:/dst/   # archive, verbose, compress, progress
```

## Disk, Memory, Network — Diagnostics

```bash
# Disk
df -h                              # mounted FS usage
du -sh /var/log/* | sort -h        # what's eating /var/log?
ncdu /var                          # interactive du
lsblk                              # block devices

# Memory
free -h
vmstat 1 5                         # 5 samples 1s apart
cat /proc/meminfo

# Network
ss -tlnp                           # listening TCP, with PIDs (replaces netstat)
ss -tan state established
ip addr                            # replaces ifconfig
ip route                           # replaces route -n
dig +short example.com             # DNS
dig @8.8.8.8 example.com           # query specific server
mtr example.com                    # traceroute + ping combined
tcpdump -i eth0 -nn port 443 -c 20 # capture 20 TLS packets
```

## The "Server Is Slow" Playbook (USE method)

For each resource, check **U**tilization, **S**aturation, **E**rrors.

```bash
uptime                             # load avg: anything > CPU count is high
top                                # %CPU, %MEM, top consumers
vmstat 1 5                         # cpu/mem/IO at glance; high "wa" = disk
iostat -x 1 5                      # per-disk utilization (need sysstat pkg)
dmesg -T | tail -50                # kernel errors (OOM, disk)
journalctl -p err --since "1h ago"
ss -s                              # socket summary
```

Memorize: **load avg > #CPUs** = CPU-bound; high `%wa` in top = IO-bound; high `si/so` in vmstat = swapping → memory pressure.

## /proc and /sys — The Truth

```bash
cat /proc/cpuinfo | grep ^processor | wc -l   # cpu count
cat /proc/loadavg
cat /proc/<pid>/status             # all signals, memory, threads of a pid
cat /proc/<pid>/limits             # ulimits seen by the process
ls -l /proc/<pid>/fd/              # open file descriptors
```

## Interview Questions

**Q: A process is using 100% CPU. How do you find which thread inside it?**
A: `top -H -p <pid>` shows threads. `ps -T -p <pid>`. Convert TID to hex and `jstack <pid>` for the JVM equivalent.

**Q: What's the difference between SIGTERM and SIGKILL?**
A: SIGTERM (15) is catchable — the app can clean up. SIGKILL (9) is delivered by the kernel, not catchable, process dies immediately. Use SIGTERM first; SIGKILL only if it hangs.

**Q: How do you make a service start on boot?**
A: `systemctl enable myapp.service` (creates symlink in `multi-user.target.wants`). `--now` also starts it immediately.

**Q: You can't SSH into a new server. Common causes?**
A: (1) Security group / firewall blocks 22, (2) `sshd` not running, (3) key not in `authorized_keys` or wrong perms (need 600 on key, 700 on `~/.ssh`), (4) `PermitRootLogin no` and you're trying root, (5) `AllowUsers` whitelist excludes you.

**Q: What's a zombie process?**
A: A child that has exited but whose parent hasn't called `wait()` to reap it. Shows as `Z` in `ps`. Fix the parent; if PID 1 in a container, your entrypoint isn't reaping (use `tini` or `--init`).

**Q: Difference between hard link and symlink?**
A: Hard link = second name for the same inode (same FS only, can't link dirs). Symlink = a file pointing to a path (can cross FS, can be broken).

## Common Pitfalls

- Forgetting `systemctl daemon-reload` after editing a `.service` file — your edits are ignored.
- `chmod 777` to "make it work" — opens the door for any local user to write to it.
- `kill -9` as first response — leaves stale lock files, sockets, half-flushed buffers.
- Editing `/etc/fstab` and rebooting without `mount -a` first — unbootable system.
- Running everything as root inside containers — defeats namespace isolation.

## Related

- [02-bash-and-shell-scripting.md](02-bash-and-shell-scripting.md)
- [22-networking-fundamentals.md](22-networking-fundamentals.md)
- [20-incident-response.md](20-incident-response.md)
- [../serious-prep/](../serious-prep/)
