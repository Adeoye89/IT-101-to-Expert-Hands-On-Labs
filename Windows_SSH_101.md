# Windows SSH 101 — Hands-On Training README

**Level:** Beginner → Intermediate  
**Platform:** Windows 10/11, Ubuntu, Kali Linux, Proxmox VE  
**Goal:** Use Windows as an SSH client, configure key authentication, troubleshoot connections, and optionally enable Windows as an SSH server.

> **Security:** Only connect to systems you own or are authorized to administer. Confirm server host-key fingerprints through a trusted channel before accepting them. Never share a private SSH key or expose SSH directly to the public internet without appropriate protections.

## Table of Contents

1. [What SSH does](#1-what-ssh-does)
2. [Lab topology and prerequisites](#2-lab-topology-and-prerequisites)
3. [Check and install the Windows SSH client](#3-check-and-install-the-windows-ssh-client)
4. [Prepare the remote Linux or Proxmox server](#4-prepare-the-remote-linux-or-proxmox-server)
5. [Connect from Windows](#5-connect-from-windows)
6. [Use the remote shell](#6-use-the-remote-shell)
7. [Create and install SSH keys](#7-create-and-install-ssh-keys)
8. [Use SSH config and alternate ports](#8-use-ssh-config-and-alternate-ports)
9. [Copy files with SCP and SFTP](#9-copy-files-with-scp-and-sftp)
10. [Enable SSH access into Windows](#10-enable-ssh-access-into-windows)
11. [Troubleshooting and diagnostics](#11-troubleshooting-and-diagnostics)
12. [Proxmox practice lab](#12-proxmox-practice-lab)
13. [Exercises and completion checklist](#13-exercises-and-completion-checklist)

---

## 1. What SSH does

SSH (Secure Shell) creates an encrypted connection between an **SSH client** and an **SSH server**. A Windows PowerShell or Terminal window can act as the client. Ubuntu, Kali, and Proxmox can act as servers if their SSH service is enabled.

| Item | Meaning |
|---|---|
| SSH | Encrypted remote command-line access |
| TCP port 22 | Default SSH service port; may be changed |
| Username | Account that exists on the **remote** machine |
| Host key | Identifies the remote SSH server |
| User key pair | Public and private keys used to authenticate a user |
| SCP / SFTP | Secure file-transfer methods over SSH |

**SSH vs Telnet:** SSH encrypts traffic; traditional Telnet does not. Prefer SSH for administration.

## 2. Lab topology and prerequisites

Example isolated home lab (substitute your own addresses):

```text
Windows 11 PC (SSH client)  192.168.1.50
          |
       LAN / VPN
          |
Ubuntu VM (SSH server)     192.168.1.100
Kali VM (SSH server)       192.168.1.110
Proxmox host               192.168.1.120
```

Prerequisites:

- Administrator access to install Windows optional features if needed.
- A valid account on the remote system.
- Working IP routing between client and server, plus a firewall rule allowing the SSH port.
- SSH server installed and running on the destination.
- For remote internet access, prefer a private VPN such as Tailscale rather than forwarding TCP 22 from the internet.

## 3. Check and install the Windows SSH client

**Step 1 — Open PowerShell.** Press **Windows + X** and choose **Terminal** or **Windows PowerShell**.

**Step 2 — Check the installed SSH version:**

```powershell
ssh -V
```

Expected: a line beginning with `OpenSSH_for_Windows_...` (exact version varies).

**Step 3 — If `ssh` is missing,** use **Settings → Optional features** (location varies by Windows release) and install **OpenSSH Client**, or open PowerShell **as Administrator** and run:

```powershell
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH.Client*'
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
```

Open a new terminal and rerun `ssh -V`.

## 4. Prepare the remote Linux or Proxmox server

### Ubuntu / Debian

At the VM's local console:

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
sudo systemctl status ssh --no-pager
ip -br addr
```

Expected: `ssh` is **active (running)** and the network interface shows an IP address.

If using UFW and SSH access is approved:

```bash
sudo ufw status
sudo ufw allow OpenSSH
```

If UFW is disabled, the allow rule does not by itself enable the firewall. Review firewall policy before making changes.

### Kali Linux

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
ip -br addr
```

### Proxmox VE

Proxmox commonly runs an SSH service by default. Verify from the Proxmox local console:

```bash
systemctl status ssh --no-pager
ip -br addr
```

Check Proxmox firewall rules and your network's access policy before connecting. Prefer a restricted administrator account where suitable; direct root SSH access depends on host policy.

## 5. Connect from Windows

In Windows PowerShell:

```powershell
ssh student@192.168.1.100
```

Replace `student` with the **remote Ubuntu username** and `192.168.1.100` with its IP address.

Other examples:

```powershell
ssh kali@192.168.1.110
ssh root@192.168.1.120
ssh -p 2222 student@192.168.1.100
```

The Proxmox example assumes root SSH login is permitted by your server's security policy.

**First connection:** SSH may show a host-key fingerprint and ask whether to trust the host. **Compare the fingerprint with a trusted console or administrator record first**. Only if it matches, type `yes`.

**Password prompt:** Type the remote account password. Characters do not appear as you type; this is expected.

**Success indicator:** The prompt changes to the remote machine's shell, such as `student@ubuntu:~$`.

## 6. Use the remote shell

Once connected to Linux:

```bash
whoami       # Which remote account am I using?
hostname     # Remote host name
pwd          # Current remote directory
ls -la       # List files including hidden files
uptime       # Time since last boot and load averages
exit         # End the SSH session
```

You can also run one command without opening a persistent shell:

```powershell
ssh student@192.168.1.100 hostname
```

**Expected:** Windows displays the remote host name, then returns to the local PowerShell prompt.

## 7. Create and install SSH keys

### 7.1 Generate a key pair on Windows

```powershell
ssh-keygen -t ed25519 -C "windows-lab"
```

Accept the default location, normally:

```text
C:\Users\YourUsername\.ssh\id_ed25519      (PRIVATE — do not share)
C:\Users\YourUsername\.ssh\id_ed25519.pub  (PUBLIC — may be copied)
```

Set a strong passphrase when prompted.

### 7.2 View and copy the public key

```powershell
Get-Content "$env:USERPROFILE\.ssh\id_ed25519.pub"
```

Copy the **entire single line** beginning with `ssh-ed25519`.

### 7.3 Install the public key on Ubuntu

Log in through the Ubuntu console or an existing authorized SSH session, **as the account you want to access**:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
```

Paste the Windows public key as **one complete line**, save, then run:

```bash
chmod 600 ~/.ssh/authorized_keys
```

If you edited files as root for another account, ensure the target user owns `~/.ssh` and `authorized_keys`.

### 7.4 Test the key

From Windows:

```powershell
ssh -i "$env:USERPROFILE\.ssh\id_ed25519" student@192.168.1.100
```

**Expected:** You authenticate using the key (possibly entering the key's passphrase) instead of the remote account password.

> Do **not** disable password authentication until key-based access is tested successfully in a second terminal and you have console recovery access.

## 8. Use SSH config and alternate ports

Create or edit the file `C:\Users\YourUsername\.ssh\config`:

```sshconfig
Host ubuntu-lab
    HostName 192.168.1.100
    User student
    Port 22
    IdentityFile ~/.ssh/id_ed25519

Host proxmox-lab
    HostName 192.168.1.120
    User root
    Port 22
```

Then connect using a short alias:

```powershell
ssh ubuntu-lab
ssh proxmox-lab
```

For a nondefault SSH port:

```powershell
ssh -p 2222 student@192.168.1.100
```

**Remember:** Changing the SSH port alone is not a substitute for secure authentication and firewall restrictions.

## 9. Copy files with SCP and SFTP

### Copy a file from Windows to Ubuntu

```powershell
scp "$env:USERPROFILE\Documents\notes.txt" student@192.168.1.100:/home/student/
```

### Copy a file from Ubuntu to Windows

```powershell
scp student@192.168.1.100:/home/student/report.txt "$env:USERPROFILE\Downloads\"
```

### Start an interactive SFTP session

```powershell
sftp student@192.168.1.100
```

At the `sftp>` prompt:

```text
pwd
lpwd
ls
put notes.txt
get report.txt
bye
```

**Note:** `put notes.txt` uses the current local SFTP directory. Use `lcd` to change the local directory if needed.

## 10. Enable SSH access into Windows

This section reverses the direction: **another computer connects to Windows**. Perform it only when required and permitted.

**Step 1 — Open PowerShell as Administrator:**

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
Get-Service sshd
```

Expected service status: `Running`.

**Step 2 — Check the inbound firewall rule:**

```powershell
Get-NetFirewallRule -Name 'OpenSSH-Server-In-TCP' -ErrorAction SilentlyContinue
```

The Windows OpenSSH installer often creates this rule automatically. If absent, create an appropriately **scoped** rule using Windows Defender Firewall or `New-NetFirewallRule`, restricting remote addresses to your trusted lab/VPN subnet.

Example for a trusted **192.168.1.0/24** lab only:

```powershell
New-NetFirewallRule -Name 'Lab-OpenSSH-In' -DisplayName 'Lab OpenSSH Inbound' `
  -Enabled True -Direction Inbound -Protocol TCP -Action Allow `
  -LocalPort 22 -RemoteAddress 192.168.1.0/24
```

If a broad pre-existing SSH allow rule exists, review and narrow it; adding a narrow rule does **not** override a broader allow rule.

**Step 3 — Find the Windows IP:**

```powershell
ipconfig
```

**Step 4 — From an authorized Linux computer:**

```bash
ssh WindowsUsername@192.168.1.50
```

For Windows account naming and key authorization, local vs domain accounts and administrator accounts can require different setup. Refer to Microsoft OpenSSH documentation before deploying key authentication to Windows Server or enterprise PCs.

## 11. Troubleshooting and diagnostics

### Check TCP reachability from Windows

```powershell
Test-NetConnection 192.168.1.100 -Port 22
```

`TcpTestSucceeded : True` means the TCP port is reachable; it does **not** prove login will work.

### Collect verbose SSH output

```powershell
ssh -vvv student@192.168.1.100
```

| Symptom | Possible cause | Verification / next action |
|---|---|---|
| `ssh` not recognized | OpenSSH Client missing or PATH issue | Install client, open new terminal, run `ssh -V` |
| Connection timed out | Routing, firewall, VPN, wrong IP | `Test-NetConnection`, check route and rules |
| Connection refused | Server not listening on target port | Check `systemctl status ssh` and `ss -tlnp` |
| Permission denied | Wrong account, password, key, or permissions | Confirm username, public key, `.ssh` permissions |
| Host-key mismatch | Server rebuilt, IP reused, or possible interception | **Stop**; verify new fingerprint out of band before changing known_hosts |
| Key ignored | Wrong key file, server policy, permissions | `ssh -i ... -vvv`, inspect server logs |
| Works on LAN, not remotely | VPN/routing/firewall restrictions | Prefer private VPN and test route to server |

Linux server-side checks:

```bash
sudo systemctl status ssh --no-pager
sudo ss -tlnp | grep ':22'
sudo journalctl -u ssh -n 50 --no-pager
```

On some distributions the service name is `sshd` instead of `ssh`.

### Safe host-key recovery

If a server was intentionally rebuilt, first verify its **new** fingerprint using the local console. Only then remove the outdated known-hosts entry:

```powershell
ssh-keygen -R 192.168.1.100
```

Reconnect and verify the fingerprint again. **Never blindly remove a mismatched host key.**

## 12. Proxmox practice lab

**Objective:** Manage your authorized Proxmox host from Windows.

1. On Proxmox console, find the management IP with `ip -br addr`.
2. Verify the SSH service is running and your access is allowed.
3. From Windows, connect:

```powershell
ssh root@192.168.1.120
```

4. Once connected, run these **read-only** inventory commands:

```bash
pveversion
hostname
qm list
pct list
pvesm status
ip -br addr
```

5. Record the Proxmox version, VM IDs, container IDs, available storage, and interface addresses.
6. Type `exit` to disconnect.

**Expected:** Commands return Proxmox inventory information without changing VM state.

## 13. Exercises and completion checklist

### Hands-on exercises

- **Lab A:** Connect from Windows to Ubuntu and run `whoami`, `hostname`, and `uptime`.
- **Lab B:** Generate an Ed25519 key, install its public key on Ubuntu, and authenticate successfully.
- **Lab C:** Configure `~/.ssh/config` on Windows and connect using `ssh ubuntu-lab`.
- **Lab D:** Upload a test file with SCP and download it again; verify the contents match.
- **Lab E:** Connect to Proxmox over SSH and collect read-only inventory commands.
- **Lab F (optional):** Enable Windows OpenSSH Server on a trusted lab network, restrict firewall access, and test inbound SSH.

### Completion checklist

- [ ] I can explain the SSH client/server model and default TCP port.
- [ ] I can check or install the Windows OpenSSH client.
- [ ] I can identify a remote server's IP address and verify the SSH service.
- [ ] I can confirm a host-key fingerprint before first login.
- [ ] I can connect to Ubuntu/Kali/Proxmox using PowerShell.
- [ ] I can create and safely store SSH keys.
- [ ] I can configure an SSH alias.
- [ ] I can transfer files with SCP or SFTP.
- [ ] I can diagnose a timeout, refusal, and authentication failure.
- [ ] I can use SSH for read-only Proxmox administration.

## Official references

- [Microsoft Learn — OpenSSH for Windows overview](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-overview)
- [Microsoft Learn — Get started with OpenSSH for Windows](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse)
- [OpenSSH manual pages](https://www.openssh.com/manual.html)
- [Ubuntu Server — OpenSSH server](https://documentation.ubuntu.com/server/how-to/security/openssh-server/)
- [Proxmox VE documentation](https://pve.proxmox.com/pve-docs/)

---

**Suggested GitHub location:** `remote-access/windows-ssh/README.md`  
**Next lesson:** SSH port forwarding, jump hosts, Tailscale SSH, and Proxmox administration over a private VPN.
