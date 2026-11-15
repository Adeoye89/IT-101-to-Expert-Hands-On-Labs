# IT 101 — SSH and Telnet: Hands-On Training

**Level:** Novice → Advanced  
**Lab systems:** Windows 11, Ubuntu Server/Desktop, Kali Linux, Proxmox VE 9.x  
**Goal:** Install, configure, use, secure, and troubleshoot remote terminal access.

> **Safety:** Only connect to systems you own or are authorized to administer. SSH encrypts traffic; Telnet does not. Never use Telnet to transmit real credentials on an untrusted or production network. Take VM snapshots before changing network or SSH configuration, and keep console access available.

## Table of contents
1. [Core concepts](#1-core-concepts)
2. [Build a safe lab](#2-build-a-safe-lab)
3. [Install and use SSH on Ubuntu](#3-install-and-use-ssh-on-ubuntu)
4. [Connect from Windows and Kali](#4-connect-from-windows-and-kali)
5. [Understand SSH keys](#5-understand-ssh-keys)
6. [Configure SSH safely](#6-configure-ssh-safely)
7. [Copy files with SCP and SFTP](#7-copy-files-with-scp-and-sftp)
8. [Run remote commands and use SSH config](#8-run-remote-commands-and-use-ssh-config)
9. [SSH port forwarding](#9-ssh-port-forwarding)
10. [Proxmox SSH administration](#10-proxmox-ssh-administration)
11. [Telnet basics and TCP testing](#11-telnet-basics-and-tcp-testing)
12. [Optional isolated Telnet demonstration](#12-optional-isolated-telnet-demonstration)
13. [Troubleshooting](#13-troubleshooting)
14. [Security checklist](#14-security-checklist)
15. [Practical assessments](#15-practical-assessments)

---

## 1. Core concepts

**SSH (Secure Shell)** is a cryptographic protocol used for remote login, executing commands, and transferring files. **Telnet** is an older remote-terminal protocol that sends data in plaintext. Both typically use TCP.

| Term | Meaning | Example |
|---|---|---|
| Client | Device initiating a connection | Windows laptop |
| Server | Device accepting a connection | Ubuntu VM |
| IP address | Network address | `192.168.56.10` |
| TCP port | Identifies a service | SSH `22`, Telnet `23` |
| Authentication | Verifying who you are | Password or SSH key |
| Host key | Verifies the server's identity | ED25519 host key fingerprint |
| Public/private key pair | Credentials for key-based authentication | `id_ed25519.pub` / `id_ed25519` |
| Firewall | Controls permitted traffic | Allow TCP 22 from admin subnet |

**How SSH works:** (1) client reaches the server on TCP 22; (2) client and server negotiate encryption and establish a protected channel; (3) the client checks the server host key; (4) the user authenticates; (5) commands and responses travel encrypted.

**How Telnet differs:** A Telnet login may look similar, but usernames, passwords, and commands can be observed by devices that can capture the traffic.

## 2. Build a safe lab

Suggested layout:

```text
Windows 11 (SSH client)      Kali VM (SSH client)
           \                 /
            \               /
         Host-only / isolated virtual network
                      |
          Ubuntu VM (SSH server)
          192.168.56.10/24 (example)
```

1. Create an Ubuntu VM and optionally a Kali VM in VirtualBox or Proxmox.
2. Attach them to the same **host-only** VirtualBox network or isolated Proxmox bridge. Ensure your Windows host can reach the lab if you want to connect from Windows.
3. Assign IP addresses using a lab DHCP service or a documented static configuration. Do not assume the sample address is already assigned.
4. On Ubuntu, run `ip -br addr` and record its actual IP.
5. From Windows PowerShell, run `ping <UBUNTU_IP>`; if ping is blocked, use a TCP connectivity test instead.

**Checkpoint:** The client and Ubuntu must have a working IP path before SSH can connect.

## 3. Install and use SSH on Ubuntu

### Task A — Install the server

Open the Ubuntu VM console and run one command at a time:

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
sudo systemctl status ssh --no-pager
```

**Explanation:** `apt update` refreshes package metadata; `apt install` installs the daemon; `systemctl enable --now` starts it and enables it at boot; `status` checks whether it is running.

Expected status includes `active (running)`. Press `q` to exit a paged status view if necessary.

### Task B — Confirm SSH is listening

```bash
sudo ss -tlnp | grep ':22'
```

Expected: a `LISTEN` entry on TCP port 22. If there is no output, check the service and configuration.

### Task C — Check firewall rules

```bash
sudo ufw status verbose
```

If UFW is **active** and your lab subnet is `192.168.56.0/24`, permit SSH from that subnet:

```bash
sudo ufw allow from 192.168.56.0/24 to any port 22 proto tcp
```

Do not enable or change a firewall on a remote-only system until you have confirmed SSH access or have console access.

### Task D — Verify the address and username

```bash
ip -br addr
whoami
hostname
```

Write down the IP, username, and hostname.

## 4. Connect from Windows and Kali

### Windows PowerShell

1. Open **Start → PowerShell**.
2. Check whether the client is installed:

```powershell
ssh -V
```

3. If the command is missing, install the **OpenSSH Client** optional Windows feature in **Settings → System → Optional features** (the exact Settings path varies by Windows version), or use the Windows capability-management tools.
4. Connect:

```powershell
ssh student@192.168.56.10
```

Replace the username and IP with your own. On first connection, **independently verify the host-key fingerprint** before accepting it. On the Ubuntu console, view the ED25519 fingerprint:

```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

5. Enter the account password when prompted; it will not echo.
6. Verify your remote session:

```bash
whoami
hostname
pwd
```

7. Disconnect with `exit`.

### Kali Linux client

```bash
ssh -V
ssh student@192.168.56.10
```

Most Kali images include the SSH client. Install it with `sudo apt install openssh-client` if needed.

### Test the TCP port without logging in

Windows:

```powershell
Test-NetConnection 192.168.56.10 -Port 22
```

Linux:

```bash
nc -vz 192.168.56.10 22
```

**Interpretation:** TCP success proves a service accepted a TCP connection, not that login credentials or host-key verification are correct.

## 5. Understand SSH keys

SSH key authentication uses a **private key** retained by the client and a **public key** installed on the server. Never share your private key.

### Task A — Generate a client key

On your Windows, Kali, or Ubuntu **client**:

```bash
ssh-keygen -t ed25519 -C "it-lab-client"
```

Press Enter to accept the default location (unless a key already exists), and choose a strong passphrase. On Windows PowerShell, the same `ssh-keygen` command works if OpenSSH Client is installed.

Typical files:

```text
~/.ssh/id_ed25519       PRIVATE — keep secret
~/.ssh/id_ed25519.pub   PUBLIC — safe to install on server
```

### Task B — Install the public key

On Linux clients with `ssh-copy-id`:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub student@192.168.56.10
```

On Windows without `ssh-copy-id`, display the **public** key:

```powershell
Get-Content "$HOME\.ssh\id_ed25519.pub"
```

At the Ubuntu server console, as the target user, create the SSH directory:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
```

Paste the public key as **one complete line**, save, and run:

```bash
chmod 600 ~/.ssh/authorized_keys
```

### Task C — Verify key login

```bash
ssh -i ~/.ssh/id_ed25519 student@192.168.56.10
```

A passphrase prompt for the **private key** is normal. You should not need the Ubuntu account password if key authentication succeeds.

## 6. Configure SSH safely

**Never disable password authentication until you have confirmed key login in a second session and have console recovery access.**

### Inspect active settings

```bash
sudo sshd -T | grep -E '^(port|passwordauthentication|pubkeyauthentication|permitrootlogin) '
```

### Create a drop-in configuration

On Ubuntu, inspect existing files first because earlier settings and drop-in ordering can affect the effective result:

```bash
sudo ls -l /etc/ssh/sshd_config.d/
sudo nano /etc/ssh/sshd_config.d/01-it-lab.conf
```

Example policy **after verifying key access**:

```text
PubkeyAuthentication yes
PasswordAuthentication no
PermitRootLogin no
```

Validate syntax and effective settings:

```bash
sudo sshd -t
sudo sshd -T | grep -E '^(passwordauthentication|pubkeyauthentication|permitrootlogin) '
```

If valid, reload:

```bash
sudo systemctl reload ssh
```

**Test a new connection before closing the existing one.** If the effective settings are not as expected, inspect other configuration files and `Match` blocks; Ubuntu's OpenSSH drop-ins may be processed before the main configuration.

## 7. Copy files with SCP and SFTP

### SCP: upload a file

From the client:

```bash
scp ./notes.txt student@192.168.56.10:/home/student/
```

Download it back:

```bash
scp student@192.168.56.10:/home/student/notes.txt ./notes-copy.txt
```

Use `-r` for a directory:

```bash
scp -r ./lab-notes student@192.168.56.10:/home/student/
```

**Note:** Modern OpenSSH `scp` generally uses the SFTP protocol under the hood.

### SFTP: interactive file transfer

```bash
sftp student@192.168.56.10
```

At the `sftp>` prompt:

```text
pwd
lpwd
ls
put notes.txt
get notes.txt downloaded-notes.txt
bye
```

`pwd` is the remote directory; `lpwd` is your local directory. `put` uploads and `get` downloads.

## 8. Run remote commands and use SSH config

### Execute one command remotely

```bash
ssh student@192.168.56.10 'hostname && uptime'
```

### Simplify repeated connections

Edit `~/.ssh/config` on your client (on Windows: `$HOME\.ssh\config`):

```sshconfig
Host ubuntu-lab
    HostName 192.168.56.10
    User student
    IdentityFile ~/.ssh/id_ed25519
```

Now connect with:

```bash
ssh ubuntu-lab
```

**Checkpoint:** Confirm `ssh ubuntu-lab` connects to the intended machine.

## 9. SSH port forwarding

**Local port forwarding** securely carries a connection to a remote service over SSH. Example: an HTTP app listening only on the Ubuntu server's loopback port 8080.

On Ubuntu, start a temporary test web server in a dedicated lab folder:

```bash
mkdir -p ~/web-lab
printf 'Hello from Ubuntu lab\n' > ~/web-lab/index.html
cd ~/web-lab
python3 -m http.server 8080 --bind 127.0.0.1
```

Leave that terminal open. On your client, run:

```bash
ssh -N -L 8080:127.0.0.1:8080 student@192.168.56.10
```

In a **second client terminal**:

```bash
curl http://127.0.0.1:8080
```

Expected: `Hello from Ubuntu lab`. Stop the SSH tunnel and Python server with `Ctrl+C` in their respective terminals.

**Explanation:** `-L` maps client port 8080 to server loopback port 8080 through the encrypted SSH connection; `-N` does not start a remote shell. This example binds the tunnel to local loopback by default.

## 10. Proxmox SSH administration

Proxmox VE is Debian-based and commonly runs an SSH server. Prefer a dedicated administrative account and least-privilege access; avoid exposing the host's SSH port to the public internet.

From a trusted management client:

```bash
ssh root@192.168.56.20
```

Use your **actual** Proxmox management IP and only if root SSH login is permitted in your controlled lab. In production, follow your organization's account and authentication policy.

Once connected:

```bash
pveversion
qm list
pct list
ip -br addr
ip route
```

| Command | Meaning |
|---|---|
| `pveversion` | Displays Proxmox version information |
| `qm list` | Lists QEMU/KVM virtual machines |
| `pct list` | Lists LXC containers |
| `ip -br addr` | Summarizes interfaces and IP addresses |
| `ip route` | Displays routes |

**Exercise:** Identify a VM's ID using `qm list`, then view its configuration using `qm config <VMID>`. Do not alter the Proxmox network bridge over an SSH-only connection without console recovery access.

## 11. Telnet basics and TCP testing

Telnet was historically used for interactive remote terminal sessions over TCP 23. **Do not use Telnet for routine remote administration.** It lacks transport encryption.

### Use a Telnet client to test a non-sensitive TCP service

On Ubuntu/Kali, install the client:

```bash
sudo apt update
sudo apt install -y telnet
```

For a local HTTP server running on port 8080 (from Lesson 9), connect:

```bash
telnet 127.0.0.1 8080
```

Type these lines, then press Enter twice:

```text
GET / HTTP/1.0
Host: localhost

```

Expected: an HTTP response with headers and the lab page body. This is **using a Telnet client as a raw TCP tool**, not logging in to a Telnet server. For routine port tests, `nc`, `curl`, or `Test-NetConnection` are generally better.

Windows can install the optional **Telnet Client** feature, but PowerShell's `Test-NetConnection` is preferable for connectivity testing.

## 12. Optional isolated Telnet demonstration

For historical protocol education only, use a disposable VM on an **internal-only** virtual network, without a bridge to your home/business LAN and without internet access. A Telnet daemon may not be installed by default and its package/service setup varies by distribution; consult that distribution's documentation rather than copying unverified daemon commands.

Suggested observation lab:

1. Use two disposable VMs on the same isolated internal network.
2. Configure a temporary Telnet service only inside that network with throwaway credentials.
3. Capture traffic on that isolated interface using Wireshark or `tcpdump`.
4. Observe that Telnet application data is not encrypted; compare with an SSH session.
5. Stop and remove the Telnet service after the exercise.

**Never use real passwords for this demonstration.**

## 13. Troubleshooting

| Symptom | Likely cause | Verification / fix |
|---|---|---|
| `Connection refused` | SSH service stopped or no listener | On server: `systemctl status ssh`; `ss -tlnp` |
| `Connection timed out` | Routing/firewall/network path issue | `ip route`, `ping`, `Test-NetConnection -Port 22` |
| `Permission denied (publickey)` | Wrong key/user or permissions | Check username, `authorized_keys`, client `ssh -v` |
| Host key changed warning | Reinstall, new server, or possible interception | Verify new fingerprint out of band before updating `known_hosts` |
| `No route to host` | No valid network route or network filtering | Check interfaces, subnet, gateways, bridges/VLANs |
| SFTP permission denied | Wrong destination ownership or path | `ls -ld` target folder, verify permissions |
| SSH works locally, not remotely | Firewall, VLAN, NAT or bridge issue | Compare server and client TCP reachability |

Useful diagnostic commands:

```bash
ssh -v student@192.168.56.10
sudo journalctl -u ssh --since '15 minutes ago' --no-pager
sudo sshd -t
```

`ssh -v` shows client connection and authentication steps. Avoid posting verbose logs publicly without reviewing them for identifying details.

### Recovering from a bad SSH configuration

1. Use the VM console or Proxmox console, **not** the broken SSH connection.
2. Inspect `/etc/ssh/sshd_config` and `/etc/ssh/sshd_config.d/`.
3. Correct or remove the faulty lab drop-in.
4. Run `sudo sshd -t`.
5. Run `sudo systemctl reload ssh`.
6. Test a fresh connection.

## 14. Security checklist

- [ ] Use SSH rather than Telnet for administration.
- [ ] Verify SSH server host-key fingerprints on first connection.
- [ ] Use strong passwords or passphrase-protected SSH keys.
- [ ] Never share private SSH keys.
- [ ] Limit SSH access to trusted networks or approved VPN paths (e.g., Tailscale).
- [ ] Apply operating system and OpenSSH security updates.
- [ ] Disable unnecessary direct root SSH access.
- [ ] Test key-based access before disabling passwords.
- [ ] Keep console recovery access for hypervisors and servers.
- [ ] Review logs and remove unused accounts/keys.
- [ ] Never expose a Telnet login service to the public internet.

## 15. Practical assessments

### Novice — First SSH session

- [ ] Identify the client and server IPs.
- [ ] Install and start OpenSSH Server on Ubuntu.
- [ ] Connect from Windows PowerShell.
- [ ] Run `whoami`, `hostname`, and `pwd` remotely.
- [ ] Disconnect cleanly.

### Beginner — Secure authentication

- [ ] Generate an ED25519 SSH key pair.
- [ ] Install the public key on Ubuntu.
- [ ] Verify key-based login.
- [ ] Inspect effective SSH settings with `sshd -T`.

### Intermediate — Files and remote commands

- [ ] Upload and download a file with SCP.
- [ ] Transfer a file with SFTP.
- [ ] Create a named host in SSH config.
- [ ] Run a remote command without an interactive shell.

### Advanced — Networking and diagnosis

- [ ] Create an SSH local port forward to a lab-only web service.
- [ ] Test TCP port 22 from Windows.
- [ ] Troubleshoot a deliberately stopped SSH service using VM console.
- [ ] Explain why Telnet is inappropriate for sensitive logins.
- [ ] Inspect Proxmox host status using SSH on an authorized lab host.

---

## Command quick reference

```text
ssh user@host                  Remote SSH login
ssh -p 2222 user@host          Login using a nondefault SSH port
ssh -i KEY user@host           Specify client private key
ssh -v user@host               Verbose connection diagnostics
ssh-keygen -t ed25519          Generate key pair
ssh-copy-id user@host          Install public key (Linux clients)
scp file user@host:/path/      Upload file
sftp user@host                Interactive secure file transfer
ssh user@host 'hostname'       Run a remote command
ssh -N -L 8080:127.0.0.1:8080 user@host  Local port forwarding
systemctl status ssh           Check SSH daemon on Ubuntu
ss -tlnp                       Inspect listening TCP ports
telnet host port               Raw TCP connection / legacy Telnet client
```

## Further reading

- OpenSSH manual pages: https://www.openssh.com/manual.html
- Ubuntu OpenSSH server documentation: https://documentation.ubuntu.com/server/how-to/security/openssh-server/
- Proxmox VE documentation: https://pve.proxmox.com/pve-docs/
- Tailscale SSH documentation: https://tailscale.com/kb/1193/tailscale-ssh

**Suggested GitHub path:** `remote-access/ssh-telnet/README.md`
