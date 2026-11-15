# Linux 101 — Commands from Beginner to Expert

> Hands-on training for Ubuntu, Debian, and Kali Linux. Run examples in your own VM. Commands marked **CAUTION** can change or remove data.

## How to use this guide

1. Open Terminal (`Ctrl+Alt+T` on many Linux desktops).
2. Work inside your home directory and create a dedicated practice folder.
3. Type commands exactly, inspect the output, then modify one option at a time.
4. Keep a lab notebook with the command, result, and what it means.
5. Avoid `sudo` unless a lesson requires it. Do not run destructive commands on production machines.

**Initial lab setup**

```bash
whoami
pwd
mkdir -p "$HOME/IT-Labs/linux101"
cd "$HOME/IT-Labs/linux101"
```

Expected: `whoami` prints your username; `pwd` prints your starting directory; `cd` moves you into the lab folder. Use `pwd` again to verify.

## Level 1 — Novice: navigate, create, and read

### 1. `pwd`, `ls`, and `cd` — where am I?

```bash
pwd                 # print working directory
ls                  # list names
ls -lah             # include hidden files, permissions, human sizes
cd ~                # go to your home folder
cd ~/IT-Labs/linux101
cd ..               # move up one level
cd -                # return to previous directory
```

**Understand:** `.` means current directory; `..` means parent; `~` means your home; `/` is the filesystem root. Hidden files begin with a dot. **Practice:** move between home and lab and confirm with `pwd`.

### 2. `mkdir`, `touch`, `echo`, `cat`, `nano`

```bash
cd ~/IT-Labs/linux101
mkdir -p projects/networking
touch projects/networking/notes.txt
echo 'My first Linux note' > projects/networking/notes.txt
cat projects/networking/notes.txt
nano projects/networking/notes.txt
```

**Understand:** `>` replaces file contents; `>>` appends. In `nano`, save with `Ctrl+O`, Enter; exit with `Ctrl+X`. **Practice:** add three lines and display them with `cat`.

### 3. `cp`, `mv`, `rm`, `rmdir` — manage files

```bash
cd ~/IT-Labs/linux101
cp projects/networking/notes.txt notes-backup.txt
mv notes-backup.txt notes-renamed.txt
mkdir empty-folder
rmdir empty-folder
rm -i notes-renamed.txt
```

**Understand:** `cp` copies, `mv` renames or moves, `rmdir` only removes empty directories, `rm -i` asks before deleting. **CAUTION:** Linux deletion generally bypasses the trash; avoid `rm -rf` in beginner labs. **Practice:** copy and rename a harmless practice file.

### 4. `man`, `--help`, `history`, `clear`

```bash
ls --help
man ls
history | tail -10
clear
```

**Understand:** `man` is a manual page; press `q` to quit. `history` lists previously entered commands. **Practice:** find what `ls -t` means.

## Level 2 — Beginner: inspect and search

### 5. `file`, `stat`, `wc`, `head`, `tail`, `less`

```bash
cd ~/IT-Labs/linux101
printf 'alpha\nbeta\ngamma\n' > sample.txt
file sample.txt
stat sample.txt
wc -l sample.txt
head -n 2 sample.txt
tail -n 1 sample.txt
less sample.txt
```

**Expected:** `wc -l` reports 3 lines; `head` shows alpha/beta; `tail` shows gamma. Exit `less` with `q`.

### 6. `grep`, `find`, pipes, redirection

```bash
grep -n 'beta' sample.txt
find . -type f -name '*.txt'
cat sample.txt | grep 'a'
ls -lah > listing.txt
printf 'four\n' >> sample.txt
```

**Understand:** `|` passes one command's output to another; `>` overwrites; `>>` appends. **Practice:** find all `.txt` files in the lab and search for `alpha`.

### 7. Permissions: `ls -l`, `chmod`, `chown`

```bash
printf '#!/bin/sh\necho Hello\n' > hello.sh
ls -l hello.sh
chmod u+x hello.sh
./hello.sh
id
```

**Expected:** prints `Hello`. `r`=read, `w`=write, `x`=execute; `u`=owner, `g`=group, `o`=others. `chown user:group file` changes ownership and may require `sudo`; do not alter system-file ownership in this lab. **Practice:** remove execute permission using `chmod u-x hello.sh`.

### 8. Users and system identity

```bash
whoami
id
hostnamectl
uname -a
cat /etc/os-release
```

**Understand:** `id` shows user/group identifiers; `hostnamectl` shows host details; `/etc/os-release` identifies the distribution. **Practice:** write down your username, Linux version, and hostname.

## Level 3 — Intermediate: packages, processes, networking

### 9. Package management (Ubuntu/Debian/Kali)

```bash
sudo apt update
apt list --installed | head
sudo apt install -y curl
curl --version
sudo apt upgrade
```

**Understand:** `apt update` refreshes package metadata; `apt install` installs software; `apt upgrade` updates installed packages. Review proposed changes before accepting upgrades on important systems. **Practice:** install `tree` and run `tree ~/IT-Labs/linux101`.

### 10. Processes and services

```bash
ps aux | head
top
systemctl --failed
systemctl status ssh
journalctl -b -n 30 --no-pager
```

**Understand:** `ps` lists processes, `top` monitors live activity (press `q`), `systemctl` manages systemd services, `journalctl` reads logs. SSH may not be installed, so `systemctl status ssh` may say the unit is missing. **Practice:** identify a running service and its status.

### 11. IP, ping, DNS, routes, ports

```bash
ip -br addr
ip route
ping -c 4 1.1.1.1
getent hosts example.com
ss -tuln
```

**Understand:** `ip` shows interfaces and routing; `ping` tests ICMP reachability; `getent` checks name resolution; `ss` lists listening sockets. A failed ping does not always mean the host is down: ICMP can be blocked. **Practice:** record the VM IP, gateway, and DNS resolution result.

### 12. HTTP, downloads, SSH, and file copy

```bash
curl -I https://example.com
curl --help | head
ssh username@192.168.56.10
scp sample.txt username@192.168.56.10:~/
```

Replace example username/IP with an SSH-enabled VM you own. `curl -I` retrieves HTTP headers; `ssh` starts an encrypted remote shell; `scp` securely copies a file. **Practice:** connect from Kali to your Ubuntu lab VM after enabling SSH on Ubuntu (`sudo apt install openssh-server`). Keep lab networking isolated.

### 13. Storage and disk space

```bash
lsblk -f
df -h
du -sh ~/IT-Labs
findmnt /
```

**Understand:** `lsblk` lists block devices, `df` shows filesystem space, `du` shows directory usage, `findmnt` displays mounts. **CAUTION:** commands such as `mkfs`, `fdisk`, and `parted` can destroy data when used on the wrong disk; practice them only on an empty disposable virtual disk with a verified backup.

## Level 4 — Advanced: automation, scheduling, troubleshooting

### 14. Environment variables and shell scripting

```bash
echo "$HOME"
export LAB_NAME=linux101
printf '#!/usr/bin/env bash\nset -euo pipefail\necho "Lab: $LAB_NAME"\ndate\n' > report.sh
chmod u+x report.sh
./report.sh
```

**Understand:** `$HOME` is a variable; `export` makes it available to child processes; `set -euo pipefail` enables stricter script error behavior. **Practice:** modify the script to print `hostname` and `uptime`.

### 15. Archives and backups

```bash
cd ~/IT-Labs
tar -czf linux101-backup.tar.gz linux101
tar -tzf linux101-backup.tar.gz | head
sha256sum linux101-backup.tar.gz
```

**Understand:** `tar -czf` creates a compressed archive; `tar -tzf` lists its contents without extracting; `sha256sum` computes a checksum. **Practice:** copy the archive to a second safe location and compare checksums.

### 16. Scheduling with cron

```bash
crontab -l
crontab -e
```

Add this line to your personal crontab to write a timestamp once daily at 9 AM (local system time):

```cron
0 9 * * * /usr/bin/date >> /home/YOUR_USERNAME/IT-Labs/linux101/daily.log
```

Replace `YOUR_USERNAME` with your real username. Verify later with `cat ~/IT-Labs/linux101/daily.log`. Cron jobs use a limited environment and might not run if the VM is powered off. Remove the entry after the exercise.

### 17. Logs and basic troubleshooting

```bash
uptime
free -h
df -h
systemctl --failed
journalctl -p err -b --no-pager | tail -30
```

**Troubleshooting sequence:** (1) confirm symptom; (2) check service status; (3) inspect logs; (4) verify IP/DNS/routes if network-related; (5) make one reversible change; (6) test again and document outcome.

## Level 5 — Expert foundations: operations and security

### 18. Secure SSH basics

```bash
ssh-keygen -t ed25519 -C 'linux101-lab'
ssh-copy-id username@192.168.56.10
ssh username@192.168.56.10
```

These create a local SSH key pair, install the public key on an authorized server, and connect. Protect your private key; never commit it to GitHub. Test key-based access before changing server SSH authentication settings. `ssh-copy-id` may need installation on some systems.

### 19. Safe command-line habits

- Inspect the current directory with `pwd` before moving/deleting files.
- Quote paths containing spaces: `cat "my notes.txt"`.
- Use `--help` or `man` before unfamiliar commands.
- Prefer least privilege: use `sudo` only when needed.
- Keep snapshots/backups before patching or storage changes.
- Never paste unreviewed internet scripts directly into a privileged shell.
- Keep Kali security testing inside your authorized lab.

### 20. Capstone lab: system health report

Create a script:

```bash
cd ~/IT-Labs/linux101
cat > health-report.sh <<'SCRIPT'
#!/usr/bin/env bash
set -euo pipefail
printf '=== System Health ===\n'
date
printf '\nHost: '; hostname
printf '\nUptime:\n'; uptime
printf '\nMemory:\n'; free -h
printf '\nDisk:\n'; df -h /
printf '\nNetwork:\n'; ip -br addr
SCRIPT
chmod u+x health-report.sh
./health-report.sh | tee health-report.txt
```

**Expected:** a terminal report and `health-report.txt` containing date, hostname, uptime, memory, disk, and network information. **Practice:** add a section that prints the default route using `ip route`.

## Quick reference

| Goal | Command |
|---|---|
| Current directory | `pwd` |
| List files | `ls -lah` |
| Change directory | `cd /path` |
| Create directory | `mkdir -p folder` |
| Create file | `touch file.txt` |
| Read file | `cat file.txt` |
| Edit file | `nano file.txt` |
| Copy file | `cp a.txt b.txt` |
| Rename file | `mv a.txt b.txt` |
| Delete with prompt | `rm -i file.txt` |
| Search text | `grep -n 'text' file.txt` |
| Find files | `find . -name '*.txt'` |
| User identity | `id` |
| IP addresses | `ip -br addr` |
| Routing | `ip route` |
| Test connectivity | `ping -c 4 1.1.1.1` |
| DNS lookup | `getent hosts example.com` |
| Listening ports | `ss -tuln` |
| Disk space | `df -h` |
| Filesystems | `lsblk -f` |
| Running processes | `ps aux` |
| Service status | `systemctl status SERVICE` |
| Logs | `journalctl -b` |
| Update package lists | `sudo apt update` |
| Install package | `sudo apt install PACKAGE` |
| SSH login | `ssh user@host` |
| Create archive | `tar -czf backup.tar.gz folder` |

## Completion checklist

- [ ] Navigate and manage files without using a GUI
- [ ] Explain permissions and execute a personal script
- [ ] Install a package and inspect a service
- [ ] Identify VM IP, gateway, DNS, and listening ports
- [ ] Connect over SSH to an authorized VM
- [ ] Create and verify a backup archive
- [ ] Write and run a system health report

**Suggested repository location:** `linux-commands/README.md`.
