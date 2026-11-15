# Bash Scripting 101 → Advanced: Hands-On IT Automation

> GitHub path: `scripting/bash/README.md`  
> Platform: Ubuntu, Debian, Kali, Proxmox VE (Linux shell)  
> Prerequisites: Linux terminal basics; a test VM and a standard user account with `sudo` when necessary.

## Table of contents

1. [What Bash is](#1-what-bash-is)
2. [Create and run a script](#2-create-and-run-a-script)
3. [Variables, arguments, and input](#3-variables-arguments-and-input)
4. [Conditions and comparisons](#4-conditions-and-comparisons)
5. [Loops and arrays](#5-loops-and-arrays)
6. [Functions and exit codes](#6-functions-and-exit-codes)
7. [Files, pipes, and text processing](#7-files-pipes-and-text-processing)
8. [Safe scripting and debugging](#8-safe-scripting-and-debugging)
9. [System administration automation](#9-system-administration-automation)
10. [Proxmox and SSH automation](#10-proxmox-and-ssh-automation)
11. [Scheduling with cron and systemd](#11-scheduling-with-cron-and-systemd)
12. [Labs and capstone](#12-labs-and-capstone)
13. [Troubleshooting and checklist](#13-troubleshooting-and-checklist)


## Lab safety and conventions

Use a disposable VM or test directory. Read a script before executing it, avoid running downloaded scripts blindly, and do not store passwords or API tokens in source files. Commands shown with example paths, usernames, and hostnames must be adapted to your environment. Test with nonproduction data first.

## Learning objectives

By the end, you should be able to read, write, test, debug, and schedule reusable administration scripts; explain exit codes and errors; and document scripts in GitHub.

## 1. What Bash is

Bash (Bourne Again SHell) interprets shell commands and scripts. It is common on Linux servers and Proxmox hosts. Scripts are text files containing commands that can be repeated consistently. Bash is not PowerShell: Bash pipelines primarily pass text streams; PowerShell pipelines pass .NET objects.

Check your shell:

```bash
printf 'Shell: %s\n' "$SHELL"
bash --version | head -n 1
command -v bash
```

Expected: a path such as `/usr/bin/bash` and a Bash version.

## 2. Create and run a script

```bash
mkdir -p ~/it-labs/bash && cd ~/it-labs/bash
nano hello.sh
```

Paste:

```bash
#!/usr/bin/env bash
printf 'Hello, %s!\n' "${USER:-student}"
date '+%Y-%m-%d %H:%M:%S'
```

Save (`Ctrl+O`, Enter, `Ctrl+X` in nano). Run:

```bash
bash hello.sh
chmod +x hello.sh
./hello.sh
```

`#!` is the shebang; `chmod +x` permits direct execution. Expected: greeting and current date. If you see `Permission denied`, check execute permission and directory mount options.

## 3. Variables, arguments, and input

```bash
#!/usr/bin/env bash
name="${1:-Student}"
role='System Administrator'
printf 'Name: %s | Role: %s\n' "$name" "$role"
printf 'Arguments supplied: %s\n' "$#"
```

Save as `args.sh`, then run `bash args.sh Alex`. `$1` is the first positional argument; `${1:-Student}` provides a default; `$#` counts arguments. **Always quote** variable expansions unless you deliberately need splitting/globbing.

Read interactive input:

```bash
read -r -p 'Enter a hostname: ' host
printf 'You entered: %s\n' "$host"
```

For secrets, avoid echoing input (`read -rs`) and prefer a secure credential store rather than saving secrets in scripts.

## 4. Conditions and comparisons

```bash
#!/usr/bin/env bash
file="${1:-/etc/hosts}"
if [[ -f "$file" ]]; then
  printf 'File exists: %s\n' "$file"
else
  printf 'Missing file: %s\n' "$file" >&2
  exit 1
fi
```

`[[ -f ... ]]` checks a regular file; `-d` checks directories; `-z` checks an empty string. Numeric tests include `-eq`, `-gt`, and `-lt`.

```bash
count=3
if (( count >= 2 )); then echo 'Enough nodes'; fi
case "${1:-status}" in
  start) echo 'Start requested' ;;
  stop) echo 'Stop requested' ;;
  status) echo 'Status requested' ;;
  *) echo 'Usage: start|stop|status' >&2; exit 2 ;;
esac
```

## 5. Loops and arrays

```bash
servers=(web01 db01 dc01)
for server in "${servers[@]}"; do
  printf 'Checking %s\n' "$server"
done

n=1
while (( n <= 3 )); do
  echo "Attempt $n"
  ((n+=1))
done
```

Use `"${array[@]}"` to preserve each element. For reading a file safely:

```bash
while IFS= read -r line || [[ -n "$line" ]]; do
  printf 'Line: %s\n' "$line"
done < /etc/hosts
```

## 6. Functions and exit codes

```bash
#!/usr/bin/env bash
check_command() {
  local cmd="$1"
  if command -v "$cmd" >/dev/null 2>&1; then
    printf 'OK: %s\n' "$cmd"
  else
    printf 'MISSING: %s\n' "$cmd" >&2
    return 1
  fi
}
check_command ssh
printf 'Previous exit code: %s\n' "$?"
```

Exit code `0` means success; nonzero indicates failure. `$?` is the last command's exit code; capture it immediately if needed. `local` limits a variable to the function.

## 7. Files, pipes, and text processing

```bash
mkdir -p ~/it-labs/bash/output
printf 'alpha\nbeta\nalpha\n' > ~/it-labs/bash/output/names.txt
sort ~/it-labs/bash/output/names.txt | uniq -c
wc -l ~/it-labs/bash/output/names.txt
grep -n 'alpha' ~/it-labs/bash/output/names.txt
```

`>` overwrites; `>>` appends; `2>` redirects stderr; `|` pipes stdout. Useful tools: `grep` (filter), `awk` (fields), `sed` (transform), `cut` (columns), `find` (paths), `jq` (JSON; install separately).

```bash
awk -F: '{print $1}' /etc/passwd | head
find ~/it-labs/bash -type f -name '*.sh' -print
```

Avoid `for f in $(ls ...)`: filenames can contain spaces/newlines. Use `find ... -print0` with a null-delimited reader for complex filenames.

## 8. Safe scripting and debugging

A reliable template:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
trap 'printf "Error at line %s (exit %s)\n" "$LINENO" "$?" >&2' ERR

main() {
  local target="${1:-}"
  if [[ -z "$target" ]]; then
    echo 'Usage: script.sh TARGET' >&2
    return 2
  fi
  printf 'Target: %s\n' "$target"
}
main "$@"
```

`-e` exits on many failures (with exceptions), `-u` rejects unset variables, and `pipefail` detects failed pipeline stages. These options are helpful but do **not** replace explicit error handling. Debug with `bash -n script.sh` (syntax), `bash -x script.sh` (trace; may reveal secrets), and ShellCheck (`shellcheck script.sh` if installed). Do not use `eval` with untrusted input.

## 9. System administration automation

Create `health-report.sh`:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
out="${1:-$HOME/it-labs/bash/output/health.txt}"
mkdir -p "$(dirname "$out")"
{
  echo '=== HOST ==='; hostname
  echo '=== TIME ==='; date -Is
  echo '=== UPTIME ==='; uptime
  echo '=== DISKS ==='; df -h
  echo '=== MEMORY ==='; free -h
  echo '=== NETWORK ==='; ip -br addr
  echo '=== FAILED SERVICES ==='; systemctl --failed --no-pager || true
} > "$out"
printf 'Report saved: %s\n' "$out"
```

Run `bash health-report.sh` and `cat ~/it-labs/bash/output/health.txt`. Expected: a timestamped inventory of your lab machine. Note: service output varies by distro and permissions.

Back up a test directory:

```bash
src="$HOME/it-labs/bash/output"
dst="$HOME/it-labs/bash/backup-$(date +%Y%m%d-%H%M%S).tar.gz"
tar -czf "$dst" -C "$(dirname "$src")" "$(basename "$src")"
tar -tzf "$dst" | head
```

Verify archives and restores before relying on them. For production use, add retention, off-host storage, monitoring, and encryption as needed.

## 10. Proxmox and SSH automation

On an authorized Proxmox host, inspect VMs (read-only):

```bash
pveversion
qm list
pct list
pvesm status
```

From a Windows WSL or Linux administration machine:

```bash
ssh admin@192.168.56.10 'hostname; uptime'
```

Use your actual Proxmox account, SSH port, and address. Do not enable root SSH merely for convenience. For a multi-host report:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
hosts=(192.168.56.10 192.168.56.11)
for host in "${hosts[@]}"; do
  echo "=== $host ==="
  if ! ssh -o BatchMode=yes -o ConnectTimeout=5 "admin@$host" 'hostname; uptime'; then
    printf 'Unable to reach %s\n' "$host" >&2
  fi
done
```

Requires configured SSH keys, correct host-key verification, and a permitted account. Never suppress host-key checking in production.

## 11. Scheduling with cron and systemd

Edit your own cron jobs:

```bash
crontab -e
```

Example entry (replace `student` with your username and verify path):

```cron
15 7 * * * /usr/bin/bash /home/student/it-labs/bash/health-report.sh >> /home/student/it-labs/bash/cron.log 2>&1
```

This runs daily at 07:15 according to the host's local time. Cron has a limited environment; use absolute paths and set needed environment explicitly. Verify with `crontab -l`, inspect the log, and check `journalctl` for cron service errors. Systemd timers are an alternative for managed schedules, dependencies, and logging.

## 12. Labs and capstone

**Lab A — Hello and parameters:** create `hello.sh` and `args.sh`; run each with and without arguments; explain `$1` and `$#`.

**Lab B — Validation:** modify the file-check script to check a directory and return exit code 2 on incorrect usage. Verify using `echo $?`.

**Lab C — Inventory:** create a list of three lab hosts and print them with a `for` loop. Add an `ssh` availability check without disabling host verification.

**Lab D — Logging:** generate a health report; append a timestamp and exit code to a separate log; rotate or archive the log.

**Lab E — Backup and restore:** archive a test folder, delete a disposable copy, restore it to a new folder, and compare checksums with `sha256sum`.

**Capstone — Lab health collector:** accept a destination directory argument; gather hostname, disk, memory, and network status; record failures; output one report per host; schedule it; document how to verify and roll back. Never delete original data during testing.

## 13. Troubleshooting and checklist

| Symptom | Likely cause | Fix |
|---|---|---|
| `Permission denied` | Script not executable | `chmod +x file.sh`, or `bash file.sh` |
| `bad interpreter` | Windows CRLF line endings | Convert with `dos2unix file.sh` or editor |
| `command not found` | Wrong PATH or package missing | `command -v tool`, install/adjust PATH |
| `unbound variable` | `set -u` and missing input | Validate arguments; use `${var:-}` where appropriate |
| Cron runs differently | Different environment | Use absolute paths and capture stderr |
| SSH fails | Auth, firewall, DNS, host key | `ssh -v`, verify reachability and fingerprint |

**Completion checklist:**

- [ ] Can write and execute a Bash script.
- [ ] Can use variables, arguments, conditions, loops, arrays, and functions.
- [ ] Can interpret exit codes and log errors.
- [ ] Can generate a system health report.
- [ ] Can safely use SSH in a script.
- [ ] Can schedule and verify a job.
- [ ] Can restore a test backup.

## Further reading

- [GNU Bash Reference Manual](https://www.gnu.org/software/bash/manual/)
- [ShellCheck](https://www.shellcheck.net/)
- [Proxmox VE Administration Guide](https://pve.proxmox.com/pve-docs/)
