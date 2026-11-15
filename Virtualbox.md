# IT 101 → Expert: Hands-On Training README

**Audience:** Complete beginners through aspiring systems administrators, network engineers, and security practitioners.  
**Format:** Explanation → prerequisites → exact steps → expected result → troubleshooting → practice → completion check.  
**Lab safety:** Work on your own devices, a home lab, or systems you are explicitly authorized to administer. Use an isolated virtual network for vulnerable security labs. Never expose intentionally vulnerable machines to the internet.

## How to use this manual
1. Complete the levels in order; do not skip the practice assignments.
2. Keep a lab journal (`lab-notes.md`) recording commands, screenshots, IP addresses, errors, and fixes. Never put passwords or secrets in notes.
3. For commands, type only what is inside the code block; replace placeholders such as `<YOUR-IP>` with your own value.
4. Take a VM snapshot before major changes. Practice restoring it.
5. Mark a lesson complete only after you can explain *why* the steps worked and repeat them without the instructions.


### Quick navigation
- [Level 1 — Novice: IT 101](#level-1--novice-it-101)
- [Hands-on VM Bootcamp — VirtualBox, Proxmox, Ubuntu, Windows & Kali](#hands-on-vm-bootcamp--virtualbox-proxmox-ubuntu-windows--kali)
- [Level 2 — Beginner: Operating Systems & Active Directory](#level-2--beginner-operating-systems--active-directory)
- [Level 3 — Intermediate: Virtualization, Remote Access, Data, Operations](#level-3--intermediate-virtualization-remote-access-data-operations)
- [Level 4 — Advanced: Automation, Observability, Security](#level-4--advanced-automation-observability-security)
- [Level 5 — Expert: Enterprise Lab & Defensive Security](#level-5--expert-enterprise-lab--defensive-security)

### Learning path
| Level | Focus | Suggested time | Evidence of completion |
|---|---|---|---|
| 1 — Novice | Computers, terminal, IP, ping, DNS, DHCP, switches, routers | 2–4 weeks | Diagnose your own home network |
| 2 — Beginner | Linux, Windows Server, Active Directory, permissions | 3–5 weeks | Build a working domain lab |
| 3 — Intermediate | Proxmox, VMs, VLAN concepts, Tailscale, databases, patching, backups | 4–6 weeks | Operate a small virtual infrastructure |
| 4 — Advanced | Automation, monitoring, security hardening, incident response | 4–8 weeks | Automate and secure the lab |
| 5 — Expert | GOAD, adversary emulation concepts, Adaptix C2 awareness, architecture | 6–10 weeks | Document a defended enterprise-style lab |

---

# LEVEL 1 — NOVICE: IT 101

## Lesson 1.1 — Learn the parts of a computer
**What you are learning:** CPU executes instructions; RAM holds active working data; SSD/HDD stores files; an operating system manages hardware and applications; a NIC (network interface card) connects to a network.

**Do this on Windows 10/11:**
1. Press **Windows + I** → **System** → **About**.
2. Write down your Windows edition, processor, and installed RAM.
3. Press **Ctrl + Shift + Esc** to open Task Manager.
4. Open **Performance**. Select **CPU**, **Memory**, **Disk**, and **Wi-Fi/Ethernet** one by one.
5. Open a browser, then watch CPU and memory usage change.

**Expected:** You can identify the computer's CPU, RAM, storage, OS, and active network interface.  
**Troubleshooting:** If Task Manager is compact, select **More details**.  
**Practice:** Explain why more RAM does not necessarily mean more disk storage.  
**Check:** [ ] I can identify five major computer components.

## Lesson 1.2 — Learn files, folders, and the command line
**Why:** IT administrators often use commands to troubleshoot and automate faster than clicking through menus.

**Windows practice:**
1. Press **Windows + R**, type `cmd`, press Enter.
2. Run the following, one line at a time:

```bat
whoami
hostname
cd %USERPROFILE%
mkdir IT-Lab
cd IT-Lab
echo My first IT note > notes.txt
type notes.txt
dir
```

**What each does:** `whoami` identifies your logged-in account; `hostname` displays the device name; `cd` changes directory; `mkdir` creates a folder; `echo ... >` writes a file; `type` reads it; `dir` lists files.

**Expected:** You have an `IT-Lab` folder containing `notes.txt`.  
**Troubleshooting:** If `mkdir IT-Lab` says the folder exists, continue with `cd IT-Lab`.  
**Practice:** Create another text file named `inventory.txt`.  
**Check:** [ ] I can create, navigate, and read files from Command Prompt.

## Lesson 1.3 — Find and understand your IP address
**What is an IP?** A device address used for communication. An IPv4 address might be `192.168.1.105`. A subnet mask identifies the local subnet; a default gateway routes traffic to other networks; DNS translates names to IP addresses; DHCP supplies configuration automatically.

**Windows practice:**
1. Press **Windows + R**, enter `cmd`.
2. Type:

```bat
ipconfig /all
```

3. Find your active **Wi-Fi** or **Ethernet** adapter (not a disconnected adapter).
4. Record the **IPv4 Address**, **Subnet Mask**, **Default Gateway**, **DNS Servers**, and **DHCP Enabled** fields.
5. Do **not** publish your public IP, credentials, or network inventory online.

**Example (your values will differ):**

```text
IPv4 Address: 192.168.1.105
Subnet Mask: 255.255.255.0
Default Gateway: 192.168.1.1
DNS Servers: 192.168.1.1
DHCP Enabled: Yes
```

**Expected:** You can distinguish your own IP from the gateway IP.  
**Troubleshooting:** If your IPv4 address starts with `169.254`, your device may not have obtained a DHCP lease.  
**Practice:** Identify whether your address is private or public.  
**Check:** [ ] I can explain all five fields.

## Lesson 1.4 — Ping and trace network connectivity
**What is ping?** A tool that sends ICMP echo requests and checks for replies. A failed ping is *not* proof a host is offline; firewalls may block ICMP.

**Steps:**
1. Open Command Prompt.
2. Test the local TCP/IP stack:

```bat
ping 127.0.0.1
```

3. Ping your actual default gateway, replacing the sample:

```bat
ping 192.168.1.1
```

4. Test an external IP and a hostname:

```bat
ping 1.1.1.1
ping example.com
tracert example.com
```

**Interpretation:** Loopback success confirms local IP processing; gateway success suggests local network reachability; external IP success suggests some external routing; hostname success also requires name resolution. `tracert` shows responding hops along a route, not necessarily every hop.

**Troubleshooting decision tree:** Gateway fails → check Wi-Fi/cable, adapter, and gateway address. External IP fails but gateway works → investigate router/ISP/firewall. IP works but hostname fails → investigate DNS.  
**Practice:** Turn off Wi-Fi briefly, repeat `ping`, then reconnect.  
**Check:** [ ] I can interpret reply, timeout, and name-resolution errors.

## Lesson 1.5 — DNS and DHCP
**DNS:** Translates a name such as `example.com` to an IP address. **DHCP:** Assigns an IP, subnet mask, gateway, and often DNS settings.

**DNS steps:**
1. Open Command Prompt.
2. Run:

```bat
nslookup example.com
ipconfig /displaydns
```

3. Compare the IP answer to what you saw with `ping example.com` (answers may vary due to load balancing).

**DHCP steps (on your own PC only):**
1. Run `ipconfig /all` and note whether DHCP is enabled.
2. Open **Settings → Network & internet → Wi-Fi or Ethernet → your connection → IP assignment**.
3. Inspect the setting; leave it on **Automatic (DHCP)** unless you understand your network's static IP requirements.
4. If your lab connection is disrupted and you have local access, try:

```bat
ipconfig /renew
```

**Warning:** Do not run `ipconfig /release` on a remote session: it may disconnect you.  
**Expected:** You can resolve a name and explain where automatic IP settings come from.  
**Practice:** Explain what breaks if DNS fails while IP routing still works.  
**Check:** [ ] I know the difference between DNS and DHCP.

## Lesson 1.6 — Switch vs. router vs. firewall
**Switch:** Connects devices on a local network and forwards Ethernet frames by MAC address. **Router:** Connects different IP networks and chooses next hops. **Firewall:** Allows or blocks traffic according to rules. **Access point:** Provides Wi-Fi access to a wired network.

**Home network inspection (read-only):**
1. Identify your modem/ONT, router, Wi-Fi access point, and any switches.
2. Draw: `Internet → modem/ONT → router/firewall → switch/Wi-Fi → laptop` (your layout may combine devices).
3. On Windows run:

```bat
arp -a
route print
```

4. Identify the default route (`0.0.0.0` destination on IPv4) and the local ARP entries.

**Expected:** You can sketch your own topology.  
**Troubleshooting:** ARP entries only show neighbors recently learned; an empty table is not automatically an error.  
**Practice:** Explain why two PCs on the same LAN may communicate without their traffic crossing the internet.  
**Check:** [ ] I can distinguish a switch from a router.

### Level 1 checkpoint
[ ] Explain IP, subnet, gateway, DNS, DHCP, ping, switch, router, and firewall without notes.  
[ ] Create a one-page network diagram.  
[ ] Diagnose one simulated failure and write the evidence.

---

# HANDS-ON VM BOOTCAMP — VIRTUALBOX, PROXMOX, UBUNTU, WINDOWS & KALI

> **When to take this bootcamp:** After Level 1 and before the Active Directory exercises in Level 2. The existing Level 2 and Level 3 lessons build on these skills.

## VM 101 — What is a virtual machine?

A **virtual machine (VM)** is an independent software-defined computer running a **guest operating system** on physical **host** hardware. A **hypervisor** assigns virtual CPUs (vCPUs), RAM, virtual disks and network adapters to each guest. A VM uses real host resources; allocating 4 GB RAM to a running VM leaves less RAM for the host.

**System VMs** run complete operating systems (Ubuntu, Windows, Kali); **process VMs** provide an execution environment for a program (for example, the Java Virtual Machine). This course uses system VMs.

| Virtualization type | How it works | Common examples | When to use |
|---|---|---|---|
| Type 1 / bare-metal | Hypervisor runs directly on hardware or as the foundational virtualization layer | Proxmox VE (KVM), VMware ESXi, Hyper-V | Dedicated server / multi-VM lab |
| Type 2 / hosted | Virtualization application runs on an existing desktop OS | Oracle VirtualBox, VMware Workstation | First VM on a laptop or desktop |
| Container | Shares the host kernel rather than running an independent guest kernel | LXC, Docker | Lightweight Linux services; not a full Windows VM |

**Other terms:** ISO = bootable installer image; **snapshot** = point-in-time rollback state (not a backup); **clone** = duplicate VM; **template** = prepared reusable VM image; **virtual bridge** = software switch connecting virtual network interfaces.

### Four-VM target topology

```text
               Physical host: VirtualBox or Proxmox VE
                              |
                  Isolated virtual lab network
                      192.168.56.0/24
                              |
         +--------------------+--------------------+
         |                    |                    |
 Ubuntu-Lab01          Windows-Client01        WinServer-DC01
 192.168.56.10          192.168.56.20          192.168.56.30
 SSH/Linux              Domain client           AD DS / DNS
                              |
                         Kali-Lab01
                       192.168.56.40
                    Authorized security lab
```

These are **example static addresses**, not automatically assigned addresses. Set up the same subnet on each guest, avoid duplicate addresses, and add a gateway only if you intentionally configure a router for this isolated network. **Do not connect deliberately vulnerable systems to your physical LAN or the internet.** For updates, temporarily use NAT and return to isolated mode before security exercises.

## VM Lab 1 — Check your Windows host

1. Press `Ctrl + Shift + Esc` → **Performance** → **CPU**. Check **Virtualization: Enabled**.
2. Check **Memory** and free disk space in **File Explorer → This PC**. A single Linux VM can work with 8 GB host RAM; for four simultaneous VMs, 32 GB or more is preferable.
3. If virtualization is disabled, restart into BIOS/UEFI and enable **Intel VT-x** or **AMD SVM/AMD-V** (menu varies by computer). Save and restart.
4. On Windows, open PowerShell and record host details:

```powershell
Get-ComputerInfo | Select-Object CsName, OsName, CsTotalPhysicalMemory
```

**Check:** Task Manager shows virtualization enabled. **If VirtualBox reports unavailable virtualization**, verify BIOS/UEFI settings and review possible conflicts with Hyper-V/VBS; don't disable security features without understanding the tradeoffs.

## VM Lab 2 — Create Ubuntu Desktop in VirtualBox (step by step)

**Official downloads:** [VirtualBox](https://www.virtualbox.org/wiki/Downloads) · [Ubuntu Desktop](https://ubuntu.com/download/desktop).

1. Download and install the appropriate VirtualBox installer for your host. Accept required network-driver installation prompts; network connectivity may briefly reset.
2. Download the Ubuntu Desktop **ISO** from the official Ubuntu site. An ISO is the installer, not an installed VM.
3. Open **VirtualBox → New**. Name: `Ubuntu-Lab01`.
4. Choose the ISO. If you want to see every Ubuntu installation screen, **skip/disable unattended installation** (wording varies by VirtualBox version).
5. Set **RAM: 4096 MB**, **CPU: 2 vCPU**, **disk: 30 GB dynamically allocated** (increase disk if the installer recommends more). Don't allocate most of your host's memory or cores.
6. Open **Settings → Network → Adapter 1**. Select **NAT** for initial installation and package updates.
7. Select the VM → **Start**. In the Ubuntu installer choose language, keyboard, **Install Ubuntu**, user account and time zone.
8. When the installer offers to erase a disk, verify that the installer is running **inside the VM** and that the target is its **virtual disk**. Never run a disk-erasing installer on your real host by mistake.
9. Finish, reboot, and eject the virtual optical disk if prompted (**Devices → Optical Drives**).
10. Log in and open Terminal with `Ctrl + Alt + T`.

```bash
ip -br address         # Identify the VM's network interface and IP
ip route               # Find routes and default gateway
ping -c 4 1.1.1.1      # Test IP reachability
ping -c 4 example.com  # Test name resolution too
sudo apt update        # Refresh package metadata
sudo apt upgrade -y    # Install available upgrades
sudo apt install -y curl git openssh-server
sudo systemctl status ssh --no-pager
```

**Expected:** `ip -br address` shows a non-loopback interface with an IP; `apt update` succeeds when internet access is available; SSH shows `active (running)` after installation. Some networks block ping even when internet works.

**Troubleshoot:** If no IP, check **Settings → Network → Adapter 1 → Enable Network Adapter**, and verify the guest's interface is up. If IP ping works but domain ping fails, inspect DNS. If the desktop is slow, reduce guest RAM/CPU load or use Ubuntu Server.

### Make your first snapshot

1. Shut down Ubuntu.
2. In VirtualBox, select the VM → **Snapshots** → **Take**.
3. Name it `Ubuntu-Clean-Install` and add a description.
4. After a future practice change, restore the snapshot to learn rollback. **Snapshots do not replace backups.**

**Assignment:** Record the VM's name, CPU, RAM, disk, IP, SSH service status and snapshot name in `lab-notes.md` (never save passwords).

## VM Lab 3 — Understand virtual networking

| Mode | Can reach internet? | Can communicate with physical LAN? | Typical use |
|---|---|---|---|
| NAT | Usually yes, outbound | Outbound through host; unsolicited inbound requires forwarding | Initial OS setup and updates |
| Bridged | Depends on physical network | Usually appears as a separate LAN device | Trusted LAN integration; not vulnerable labs |
| Host-only | Not by default | Host and other host-only VMs only | Local isolated exercises |
| Internal Network | Not by default | Only VMs on same named internal network | Stronger VM-only isolation |
| NAT Network | Usually yes, outbound | VMs can talk to one another on same NAT network | Multi-VM lab with updates |

**VirtualBox isolated-network practice:**

1. Power off the VM.
2. Open **Settings → Network → Adapter 1** and select **Host-only Adapter** or **Internal Network**.
3. Attach all lab VMs to the same chosen host-only network or internal-network name.
4. For a host-only adapter, check the host-only DHCP settings in VirtualBox. You may use DHCP or configure static addresses; do not mix addresses carelessly.
5. Boot Ubuntu and run `ip -br address`. If you need to use the example static topology, configure static IPs in each guest using the appropriate OS network settings.
6. Ping a second VM's **actual** address; don't assume the example addresses were assigned.

**Security note:** Host-only includes the host as a network participant. Internal Network is VM-to-VM only. Use Internal Network when you want to reduce accidental host exposure; keep the host OS patched regardless.

## VM Lab 4 — Spin up Kali Linux in VirtualBox

**Official downloads:** [Kali Linux](https://www.kali.org/get-kali/) and [Kali virtual machines](https://www.kali.org/docs/virtualization/).

**Why Kali?** It is a Debian-based distribution preconfigured for security learning and authorized assessments. Kali is **not** a vulnerable target; it is the workstation from which you inspect your own test systems.

1. From the official Kali website choose **Virtual Machines → VirtualBox** and download the image appropriate for your architecture. Verify checksums/signatures using Kali's published instructions when possible.
2. Extract the archive with an archive utility (for example 7-Zip).
3. In VirtualBox select **Machine → Add** (or **Tools → Add**, depending on version) and open the extracted `.vbox` file. If you instead downloaded a Kali installer ISO, use **New** and install from ISO as in Lab 2.
4. Name/identify it as `Kali-Lab01`. Check settings before starting: **2 vCPU, 4 GB RAM, approximately 40 GB virtual disk** (the prebuilt image's disk size may already be fixed).
5. For first boot and updates use **NAT**. Start the VM. Check the official image's documented credentials (official prebuilt images commonly use `kali` / `kali`) and **change the password immediately**:

```bash
passwd
sudo apt update
sudo apt full-upgrade -y
sudo reboot
```

6. Shut down Kali. Switch **Adapter 1** to the **same isolated host-only/internal network** used by Ubuntu. Ensure both VMs have addresses in the same subnet.
7. Start Kali, open Terminal and inspect networking:

```bash
ip -br addr
ip route
ping -c 4 192.168.56.10    # Replace with your Ubuntu VM's actual lab IP
```

8. On Ubuntu, verify SSH is running with `sudo systemctl status ssh`. From Kali, scan **only your own lab Ubuntu VM**:

```bash
nmap -sV 192.168.56.10      # Replace with your Ubuntu VM's actual lab IP
```

**Interpretation:** `22/tcp open ssh` suggests an SSH service is reachable; `closed` means no service accepted a connection; `filtered` may indicate a firewall or packet filtering. Scan only machines you own or have explicit permission to test.

9. Shut down Kali and create a snapshot named `Kali-Updated-Baseline`.

**Kali tools to study next:** Nmap (service discovery), Wireshark/tcpdump (packets), Burp Suite (web testing), and defensive log analysis. Use intentionally vulnerable practice targets only on an isolated network.

**Troubleshooting:** If ping fails, verify both guests are on the same named virtual network, confirm IP/subnet assignments and check firewall ICMP rules. If Nmap finds no SSH port, check Ubuntu's SSH service and firewall. If updates fail after isolation, temporarily switch to NAT, update, then return to isolation.

## VM Lab 5 — Install Proxmox VE and create a VM

**Official resources:** [Proxmox downloads](https://www.proxmox.com/en/downloads) · [Proxmox documentation](https://pve.proxmox.com/pve-docs/).

> **Destructive-install warning:** Installing Proxmox VE directly on a computer can erase the selected physical drive. Use a spare dedicated machine and back up all data before beginning. Do not install over your daily-use Windows drive.

1. Download the Proxmox VE installer ISO from the official website.
2. Create a bootable USB drive from the ISO with a reputable USB imaging tool.
3. Boot the **spare server** from the USB and select **Install Proxmox VE (Graphical)**.
4. Select the correct target disk; review the destructive-install warning. Choose country/time zone, admin password, email, hostname and a reachable static management IP/gateway/DNS appropriate for your network.
5. Finish installation and reboot. From another machine on the management network, browse to `https://YOUR-PROXMOX-IP:8006` (a certificate warning may appear for a self-signed certificate; verify the IP and certificate context).
6. Log in as `root` using the **Linux PAM** realm.
7. Select the node → **local** storage → **ISO Images** → **Upload**, and upload the Ubuntu Server ISO.
8. Click **Create VM** and enter:

| Proxmox option | Starter value |
|---|---|
| VM ID / name | `100` / `Ubuntu-Server01` |
| OS media | Ubuntu Server ISO |
| CPU | 2 cores |
| Memory | 4096 MiB |
| Disk | 32 GiB |
| Network | `vmbr0` only for trusted networking; use a dedicated isolated bridge for security labs |
| NIC model | VirtIO |

9. Click **Finish → Start → Console** and complete the Ubuntu Server installer. Enable OpenSSH during setup if desired.
10. Inside Ubuntu, install the guest agent:

```bash
sudo apt update
sudo apt install -y qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```

11. Shut down the guest, enable **Options → QEMU Guest Agent** in Proxmox, then boot the guest again. Check the VM summary for guest details.
12. Create a backup using the Proxmox backup interface to storage with sufficient space; verify you can locate the backup and understand the restore workflow.

**Isolated Proxmox bridge (concept):** Create a separate Linux bridge (for example `vmbr1`) with **no physical bridge port**, and attach Kali and lab targets to it. Do not alter the management bridge until you understand the impact; a misconfiguration can disconnect your server. Provide IPs/DHCP to guests through a lab router or guest static configuration. For internet access, explicitly add a controlled NAT/router path instead of bridging vulnerable guests to the LAN.

## VM Lab 6 — Add Windows Server and Windows client

1. Download official Windows Server evaluation media from [Microsoft Evaluation Center](https://www.microsoft.com/en-us/evalcenter/) and Windows client installation media from [Microsoft](https://www.microsoft.com/software-download/). Check edition, evaluation limits and licensing.
2. In VirtualBox or Proxmox, create `WinServer-DC01` (start with **2 vCPU, 4–8 GB RAM, 60 GB disk**) and install Windows Server. A GUI/Desktop Experience edition is easiest for beginners.
3. Create `Windows-Client01` with resources matching the chosen Windows edition; Windows 11 typically needs at least **4 GB RAM, 64 GB disk, UEFI, Secure Boot and TPM 2.0**, plus a suitable processor. Configure the hypervisor accordingly.
4. Connect both VMs to the **same isolated lab network** as Ubuntu and Kali. Configure distinct IP addresses. In this example, server `192.168.56.30` and client `192.168.56.20` with subnet mask `255.255.255.0`.
5. From Windows PowerShell on each guest, inspect network settings:

```powershell
ipconfig /all
Test-NetConnection 192.168.56.30
```

6. Follow **Lesson 2.3** to install Active Directory Domain Services and DNS on the server. Point the client DNS setting to the domain controller's lab IP before joining the domain (Lesson 2.5). Avoid assigning a public DNS server as the primary DNS server for an AD client.
7. Once the domain works, record your topology and take snapshots/backups before changing Group Policy.

## VM Lab 7 — Troubleshooting decision tree

| Symptom | First checks | Next action |
|---|---|---|
| VM won't start | Host RAM/CPU free? virtualization enabled? | Review hypervisor error, reduce allocation |
| Installer can't see disk | Virtual disk attached? correct storage controller? | Check VM hardware settings; for Windows consider storage driver |
| Guest has no IP | Adapter enabled? same virtual network? DHCP available? | Configure guest static IP or working lab DHCP |
| Can ping IP, not domain | DNS server and resolver settings | Verify DNS resolution and DNS service |
| Kali cannot reach Ubuntu | Same isolated network and subnet? | Check IPs, firewall and guest network interface |
| SSH connection refused | SSH installed and running? port 22 listening? | Start SSH and inspect guest firewall |
| Windows won't join domain | Client DNS points to DC? time synchronized? | Check DNS SRV records, credentials and DC reachability |
| Proxmox UI unreachable | Correct management IP, port 8006, cabling? | Check management bridge, service and firewall locally |

## VM Bootcamp completion checklist

- [ ] Explain host vs. guest, Type 1 vs. Type 2, VM vs. container.
- [ ] Create Ubuntu in VirtualBox and install an operating system from ISO.
- [ ] Identify Ubuntu's IP and test internet/DNS.
- [ ] Install and verify SSH.
- [ ] Create and restore a snapshot in a disposable test VM.
- [ ] Import and update Kali Linux.
- [ ] Put Kali and Ubuntu on an isolated virtual network.
- [ ] Run an authorized Nmap service scan against your own Ubuntu VM.
- [ ] Create an Ubuntu Server VM in Proxmox on dedicated hardware (if available).
- [ ] Create Windows Server and Windows client VMs when hardware permits.
- [ ] Document VM settings, network addresses, expected results and errors in `lab-notes.md`.

---

# LEVEL 2 — BEGINNER: OPERATING SYSTEMS & ACTIVE DIRECTORY

## Lesson 2.1 — Install Linux in a virtual machine
**Why:** Linux is common on servers, appliances, and cloud infrastructure. A VM is a computer simulated by software.

**Prerequisites:** A PC with virtualization enabled, at least 8 GB RAM (16 GB recommended), 30 GB free disk space, and a supported hypervisor such as VirtualBox or Hyper-V. Download Ubuntu Server ISO from its official website and verify the download as instructed there.

**Steps (VirtualBox example):**
1. Install VirtualBox from its official site.
2. Select **New** → name `ubuntu-lab` → choose the Ubuntu ISO.
3. Allocate **2 vCPUs**, **2–4 GB RAM**, and **25 GB virtual disk** if your PC supports it.
4. For simple internet access, choose **NAT** networking.
5. Start the VM; follow Ubuntu Server's installer, create a non-root user, and install OpenSSH only if you need it.
6. Log in at the VM console and run:

```bash
whoami
hostname
ip -br addr
ip route
sudo apt update
```

**Expected:** A working Ubuntu login and an IP address.  
**Troubleshooting:** No 64-bit virtualization option? Check BIOS/UEFI virtualization support and conflicts with other hypervisors.  
**Practice:** Create a directory with `mkdir ~/lab`, then `ls -la ~/lab`.  
**Check:** [ ] I can install a Linux VM and inspect its IP.

## Lesson 2.2 — Linux accounts, files, services
**Why:** A server admin must understand permissions and services.

**Steps inside your Ubuntu VM:**

```bash
pwd
mkdir -p ~/lab
printf 'hello\n' > ~/lab/readme.txt
cat ~/lab/readme.txt
ls -l ~/lab/readme.txt
chmod 600 ~/lab/readme.txt
ls -l ~/lab/readme.txt
systemctl --failed
```

**Explanation:** `chmod 600` gives the owner read/write access and removes access for group/others. `systemctl --failed` lists failed systemd units.

**Expected:** The file permission string starts with `-rw-------`.  
**Troubleshooting:** Use `sudo` only for administrative actions; do not routinely log in as root.  
**Practice:** Explain the difference between file owner, group, and others.  
**Check:** [ ] I can read permissions and inspect service health.

## Lesson 2.3 — Build a Windows Server + Active Directory lab
**What is Active Directory (AD DS)?** A directory service for managing domain users, computers, groups, and authentication. A **domain controller (DC)** hosts AD DS. **DNS** is essential to domain discovery.

**Prerequisites:** Windows Server evaluation ISO obtained from Microsoft; enough host RAM/storage; a virtualization platform; an **isolated lab network**; one Windows Server VM (`DC1`) and one Windows client VM (`CLIENT1`). The client edition must support domain joining (e.g., Windows Pro/Enterprise). Use the lab-only DNS domain `corp.example.test` and lab subnet `10.50.10.0/24`. Avoid connecting this lab DC to a production LAN.

**Part A — Prepare the server:**
1. Install Windows Server in a VM named `DC1`.
2. Connect it to your **isolated lab virtual network**.
3. Open **Server Manager → Local Server** and rename the server `DC1`; restart.
4. Configure its lab adapter with static IP `10.50.10.10`, mask `255.255.255.0`, DNS `10.50.10.10`. Leave the gateway blank for a fully isolated single-subnet lab.
5. Verify in PowerShell:

```powershell
Get-NetIPAddress -AddressFamily IPv4
Get-DnsClientServerAddress -AddressFamily IPv4
```

**Part B — Install AD DS:**
1. Open **Server Manager → Manage → Add Roles and Features**.
2. Choose **Role-based or feature-based installation**, select `DC1`.
3. Check **Active Directory Domain Services** → **Add Features** → **Install**.
4. After installation select **Promote this server to a domain controller**.
5. Select **Add a new forest** → root domain `corp.example.test`.
6. Set a strong Directory Services Restore Mode password and store it securely.
7. Accept appropriate defaults for a new isolated lab; review prerequisites and select **Install**. The VM restarts.
8. Log in with the domain administrator account and open **Tools → Active Directory Users and Computers**.

**Expected:** The domain `corp.example.test` appears in AD Users and Computers.  
**Troubleshooting:** Promotion failures often involve DNS, IP configuration, unsupported OS prerequisites, or resource limits. Review the prerequisite report rather than blindly retrying.  
**Practice:** Draw the relationship among DC, domain, DNS, user, and computer.  
**Check:** [ ] I can promote a lab DC.

## Lesson 2.4 — Create users, groups, and organizational units (OUs)
**Why:** OUs organize directory objects; groups simplify access control; users represent identities.

**Steps:**
1. On `DC1`, open **Server Manager → Tools → Active Directory Users and Computers**.
2. Right-click `corp.example.test` → **New → Organizational Unit** → name it `LabUsers`.
3. Right-click `LabUsers` → **New → User** → create `Alex Student`, sign-in name `alex.student`.
4. Assign a strong temporary password; select **User must change password at next logon** if appropriate for your lab.
5. Right-click `LabUsers` → **New → Group** → name it `IT-Lab-Readers`; choose **Global** and **Security**.
6. Open the group's **Members** tab → **Add** → enter `alex.student` → **Check Names** → **OK**.
7. In PowerShell (AD module installed), verify:

```powershell
Get-ADUser alex.student
Get-ADGroupMember IT-Lab-Readers
```

**Expected:** Alex exists and appears as a group member.  
**Troubleshooting:** If `Get-ADUser` is missing, use PowerShell on the DC or install RSAT AD tools on a supported management workstation.  
**Practice:** Create a second group for lab administrators, but do **not** give ordinary learners Domain Admin privileges.  
**Check:** [ ] I can create OUs, users, and groups.

## Lesson 2.5 — Join a Windows client to the domain
**Steps:**
1. Install a Windows Pro/Enterprise VM named `CLIENT1` on the **same isolated virtual network** as `DC1`.
2. Give it IP `10.50.10.20/24` and set **DNS server** to `10.50.10.10`; leave gateway blank in the isolated lab.
3. In Command Prompt run:

```bat
ping 10.50.10.10
nslookup corp.example.test 10.50.10.10
```

4. Open **Settings → System → About → Domain or workgroup** (or **Advanced system settings → Computer Name → Change**, depending on Windows version).
5. Choose **Domain** and enter `corp.example.test`.
6. Provide authorized domain credentials when prompted, then restart.
7. On the sign-in screen choose **Other user** and sign in as `CORP\alex.student` using the password you assigned.
8. On `DC1`, confirm `CLIENT1` appears in AD Users and Computers (often under **Computers**).

**Expected:** The domain user can log on to the domain-joined client.  
**Troubleshooting:** DNS must point to the DC, not a public resolver; verify clock synchronization and network connectivity.  
**Practice:** Explain why `ping` success alone does not prove AD is healthy.  
**Check:** [ ] I can join and log in to a domain client.

## Lesson 2.6 — Group Policy basics
**What is Group Policy?** Centralized configuration for Windows users and computers.

**Safe demonstration:**
1. On `DC1`, open **Group Policy Management**.
2. Create a new OU `LabComputers` and move `CLIENT1` into it (only your lab client).
3. Right-click `LabComputers` → **Create a GPO in this domain, and Link it here** → name `Lab-Desktop-Policy`.
4. Edit the GPO → **User Configuration → Policies → Administrative Templates → Desktop → Desktop → Desktop Wallpaper** only if you have a valid shared wallpaper path; otherwise choose a harmless, reversible policy such as a supported Control Panel setting and document it.
5. On `CLIENT1`, run:

```bat
gpupdate /force
gpresult /r
```

6. Verify whether the GPO applies. **Note:** User Configuration applies to user objects; a GPO linked only to a computer OU normally will not apply its user settings without loopback processing. For this exercise, configure a **Computer Configuration** setting, or link a user-setting GPO to the OU containing `alex.student`.

**Expected:** You can see your lab GPO in the appropriate `gpresult` section.  
**Troubleshooting:** Check whether the GPO is linked to the correct OU and whether settings are user-side or computer-side.  
**Practice:** Document one GPO setting, its scope, and how to reverse it.  
**Check:** [ ] I understand GPO scope and verification.

### Level 2 checkpoint
[ ] Install a Linux VM. [ ] Promote a Windows DC. [ ] Create a user and security group. [ ] Join a Windows client. [ ] Verify a GPO.  
**Deliverable:** Screenshot or notes showing the domain, user/group, and successful client logon (without exposing passwords).

---

# LEVEL 3 — INTERMEDIATE: VIRTUALIZATION, REMOTE ACCESS, DATA, OPERATIONS

## Lesson 3.1 — Install Proxmox VE on a dedicated lab machine
**What is Proxmox?** A virtualization platform that runs VMs (KVM) and Linux containers (LXC) and provides web-based management.

**Important:** The Proxmox installer can erase the target disk. Use a **dedicated spare PC/server** with data backed up, not your daily-use Windows disk. Check the official Proxmox installation guide for supported hardware and current release requirements.

**Steps:**
1. Download the official **Proxmox VE ISO** and verify its checksum using the official instructions.
2. Write the ISO to a USB drive with a trusted imaging utility.
3. Connect the spare server by Ethernet; boot from USB.
4. Choose **Install Proxmox VE**, select only the dedicated target disk, and review the erase warning.
5. Set country/time zone, a strong administrative password, and email.
6. Configure a **static management IP** on your lab LAN (example `192.168.50.10/24`), correct gateway and DNS for *your* LAN; reserve the address on your router if appropriate.
7. Finish installation, reboot, and from another machine browse to `https://192.168.50.10:8006` (replace with your address).
8. Review the browser certificate warning: a fresh lab install may use a self-signed certificate. Verify the host and certificate before proceeding; deploy a trusted certificate for long-term use.
9. Sign in, inspect **Datacenter**, **Node**, **Storage**, and **Network**. Apply supported updates and configure backups.

**Expected:** You can reach the Proxmox web console.  
**Troubleshooting:** If unreachable, check the physical Ethernet link, management IP, subnet, gateway, and whether port 8006 is reachable on your trusted network.  
**Practice:** Create a small Ubuntu VM and snapshot it before changing its configuration.  
**Check:** [ ] I can install and log in to Proxmox safely.

## Lesson 3.2 — Virtual networks, bridges, and VLANs
**Definitions:** A **bridge** connects virtual NICs to a virtual or physical Layer-2 network. A **VLAN** separates Layer-2 broadcast domains using tags (802.1Q). An **access port** carries one untagged VLAN; a **trunk** carries tagged VLANs between VLAN-aware devices.

**Safe Proxmox practice:**
1. In Proxmox, open **Node → System → Network**.
2. Identify the existing management bridge (often `vmbr0`). **Do not modify it while relying on remote access.**
3. Create a **Linux Bridge** named `vmbr1` with **no physical bridge port**, no IP address, and no gateway; apply carefully.
4. Create two Ubuntu VMs; attach each virtual NIC to `vmbr1`.
5. In the VM consoles, temporarily assign addresses on the same subnet, e.g. `10.60.1.11/24` and `10.60.1.12/24` using your distribution's supported network configuration tool.
6. Ping one VM from the other. Observe that they can communicate locally but have no internet route by default.
7. Restore the VMs' prior networking when done.

**Expected:** VM-to-VM traffic works on an isolated bridge.  
**Troubleshooting:** Verify each VM NIC is on `vmbr1`, both addresses are in the same subnet, and host firewalls allow the test.  
**Practice:** Draw a separate VLAN design on paper before touching a managed switch.  
**Check:** [ ] I can explain bridge vs. VLAN vs. router.

## Lesson 3.3 — Tailscale remote connectivity
**What is Tailscale?** A private overlay network using WireGuard-based connectivity and identity/access controls. It can help you reach lab machines remotely without directly exposing management ports to the internet.

**Steps:**
1. Create a Tailscale account using its official website; enable MFA on the identity provider.
2. Install Tailscale on your Windows laptop using the official installer.
3. Sign in and confirm the device appears in the Tailscale admin console.
4. Install Tailscale on your Ubuntu lab VM using the official OS-specific instructions (review the installation script before executing it).
5. Authenticate the VM to the **same tailnet**.
6. On the Ubuntu VM run:

```bash
tailscale status
tailscale ip -4
```

7. From Windows, ping the VM's **Tailscale IP** or MagicDNS name if enabled:

```powershell
ping <TAILSCALE-IP>
```

8. Configure least-privilege access rules in the admin console; disable unnecessary sharing and remove stale devices.

**Expected:** Both devices show in the tailnet and can communicate if policy/firewall allows.  
**Troubleshooting:** Check device approval, ACL/grants, host firewall, login status, and whether ping is permitted.  
**Practice:** Explain why remote access does not automatically mean remote admin permissions.  
**Check:** [ ] I can connect two authorized lab devices through Tailscale.

## Lesson 3.4 — Databases: install and query PostgreSQL
**What is a database?** A system for storing and retrieving structured data. A **table** has rows and columns; **SQL** queries the data. PostgreSQL is an open-source relational database.

**Steps (Ubuntu VM):**
1. Update package metadata and install PostgreSQL:

```bash
sudo apt update
sudo apt install postgresql -y
sudo systemctl status postgresql
```

2. Open the local PostgreSQL shell as its service account:

```bash
sudo -u postgres psql
```

3. Run these SQL statements **inside** `psql`:

```sql
CREATE DATABASE it_training;
\connect it_training
CREATE TABLE devices (
  id SERIAL PRIMARY KEY,
  hostname TEXT NOT NULL,
  ip_address INET NOT NULL
);
INSERT INTO devices (hostname, ip_address)
VALUES ('client1', '10.50.10.20'), ('dc1', '10.50.10.10');
SELECT * FROM devices;
UPDATE devices SET hostname = 'client01' WHERE id = 1;
SELECT * FROM devices ORDER BY id;
\quit
```

**Expected:** You see two rows and the first hostname changes.  
**Troubleshooting:** SQL commands end with semicolons; `\connect` and `\quit` are `psql` commands and do not use semicolons.  
**Practice:** Add a third device; query only its hostname.  
**Check:** [ ] I can create a database/table and run CRUD basics.

## Lesson 3.5 — Patch management
**What is a patch?** An update that fixes bugs or security issues. Patch management includes inventory, testing, approval, deployment, verification, and rollback.

**Windows steps (your own test PC/VM):**
1. Open **Settings → Windows Update**.
2. Select **Check for updates**; read pending update details.
3. Take a VM snapshot or verify a current backup before major changes.
4. Install approved updates and restart when requested.
5. Open **Update history** and record installed KB numbers and dates.

**Ubuntu steps:**

```bash
sudo apt update
apt list --upgradable
sudo apt upgrade
sudo reboot
```

After reboot:

```bash
uname -r
systemctl --failed
```

**Expected:** Updates complete and services remain healthy.  
**Troubleshooting:** If a service fails, review logs (`journalctl -u <service>`) and your snapshot/backup plan; avoid indiscriminate rollbacks on production systems.  
**Practice:** Write a change record: asset, reason, maintenance window, backup, result, rollback plan.  
**Check:** [ ] I can patch and verify a test server.

## Lesson 3.6 — Backups and restoration
**Why:** Snapshots are useful but **not** a substitute for independent backups. A backup is proven only when restoration works.

**Proxmox example:**
1. In the web console, select a test VM → **Backup → Backup now**.
2. Choose an appropriate backup storage target and mode; confirm free space and retention policy.
3. Wait for **Task OK** and inspect the resulting backup record.
4. Restore the backup **to a new VM ID**, on a safe test network, to avoid overwriting your working VM.
5. Boot the restored VM and check files, login, and network configuration.
6. Document backup date, storage location, restore time, and verification.

**Expected:** A separate restored VM starts and contains the expected data.  
**Troubleshooting:** Check backup storage permissions, capacity, and VM disk compatibility.  
**Practice:** Explain the 3-2-1 backup principle and why offsite/offline copies matter.  
**Check:** [ ] I have completed a test restore.

### Level 3 checkpoint
[ ] Proxmox VM created. [ ] Isolated bridge tested. [ ] Tailscale connected. [ ] SQL table queried. [ ] OS patched. [ ] Backup restored.

---

# LEVEL 4 — ADVANCED: AUTOMATION, OBSERVABILITY, SECURITY

## Lesson 4.1 — PowerShell automation for inventory
**Why:** Repeating the same manual check across many machines is slow and error-prone.

**Steps (Windows PowerShell):**
1. Open PowerShell as your normal user.
2. Run:

```powershell
Get-ComputerInfo | Select-Object CsName, WindowsProductName, OsVersion
Get-NetIPAddress -AddressFamily IPv4 |
  Where-Object { $_.IPAddress -notlike '169.254*' } |
  Select-Object InterfaceAlias, IPAddress, PrefixLength
```

3. Create an inventory file in your Documents folder:

```powershell
$folder = Join-Path $HOME 'Documents\IT-Lab'
New-Item -ItemType Directory -Path $folder -Force | Out-Null
Get-Service |
  Select-Object Name, Status, StartType |
  Export-Csv (Join-Path $folder 'services.csv') -NoTypeInformation
```

4. Open `services.csv` in a spreadsheet application.

**Expected:** A CSV containing service names, status, and startup types.  
**Troubleshooting:** If Documents is redirected to OneDrive, use an existing writable folder path.  
**Practice:** Add a timestamp to the export filename.  
**Check:** [ ] I can produce repeatable inventory output.

## Lesson 4.2 — Python automation: check your own lab hosts
**Why:** Python is useful for scripts, APIs, reporting, and data handling.

**Steps:**
1. Install Python from its official source and verify `python --version` (or `py --version` on Windows).
2. Save this as `check_hosts.py`:

```python
import platform
import subprocess

hosts = ["127.0.0.1", "10.50.10.10", "10.50.10.20"]  # your lab only
flag = "-n" if platform.system() == "Windows" else "-c"

for host in hosts:
    result = subprocess.run(
        ["ping", flag, "1", host],
        capture_output=True, text=True, timeout=5, check=False
    )
    print(f"{host}: {'responded' if result.returncode == 0 else 'no ping reply'}")
```

3. In a terminal in the file's directory run `python check_hosts.py` (or `py check_hosts.py`).
4. Compare results with manual `ping`.

**Expected:** A simple report for three **authorized** lab addresses.  
**Troubleshooting:** A firewall may block ping; treat 'no ping reply' as inconclusive rather than proof the host is down.  
**Practice:** Save output to a dated text file.  
**Check:** [ ] I can run and explain a basic automation script.

## Lesson 4.3 — Logs and monitoring
**Why:** Monitoring detects problems; logs explain what happened.

**Windows steps:**
1. Open **Event Viewer** → **Windows Logs → System**.
2. Filter for **Error** and **Warning** in the last 24 hours.
3. Pick one event and record time, source, event ID, and message.
4. In PowerShell run:

```powershell
Get-WinEvent -LogName System -MaxEvents 10 |
  Select-Object TimeCreated, Id, ProviderName, LevelDisplayName
```

**Linux steps:**

```bash
systemctl --failed
journalctl -p warning -n 30 --no-pager
df -h
free -h
```

**Expected:** You can identify a log event and determine whether it needs action.  
**Troubleshooting:** Some logs require elevated permissions. An error entry is not always an ongoing incident.  
**Practice:** Define three monitoring signals: disk capacity, service status, and failed sign-ins.  
**Check:** [ ] I can find and summarize useful system events.

## Lesson 4.4 — Firewall rules and least privilege
**What is least privilege?** Give users and services only the access needed for their task.

**Windows lab demonstration:**
1. Open **Windows Defender Firewall with Advanced Security**.
2. Inspect **Inbound Rules**; do not change domain-wide settings.
3. On an isolated lab VM, create a narrowly scoped inbound rule for a service you intentionally host, specifying protocol/port, source subnet, and appropriate network profile.
4. Test access from an authorized lab client; verify traffic from outside the permitted subnet is not allowed (if you have a separate test segment).
5. Disable or remove the rule after the exercise.

**Expected:** You can explain inbound vs. outbound and why source scoping matters.  
**Troubleshooting:** The application must actually listen on the port; firewall allowance alone does not start a service.  
**Practice:** Document a rule's purpose, owner, source, destination, port, and expiration/review date.  
**Check:** [ ] I can describe and review a minimal firewall rule.

## Lesson 4.5 — Vulnerability management and incident response
**Definitions:** A **vulnerability** is a weakness; **risk** considers likelihood and impact; **incident response** is the process of preparing, detecting, analyzing, containing, eradicating, recovering, and learning.

**Safe tabletop exercise:**
1. Imagine your test Ubuntu VM has an outdated package.
2. Record the asset owner, OS/version, exposure, and business impact.
3. Check the vendor's advisory and available update.
4. Prioritize based on exposure and severity; schedule a test patch.
5. Patch and verify the package/service still works.
6. Write an incident-style timeline: detection, triage, remediation, verification, lessons learned.

**Expected:** A one-page risk and remediation record.  
**Troubleshooting:** A scanner finding alone is not proof of exploitation; verify version, configuration, and applicability.  
**Practice:** Define recovery time objective (RTO) and recovery point objective (RPO).  
**Check:** [ ] I can explain a safe response workflow.

### Level 4 checkpoint
[ ] PowerShell inventory exported. [ ] Python check script works. [ ] Logs reviewed. [ ] Firewall rule documented. [ ] Vulnerability remediation recorded.

---

# LEVEL 5 — EXPERT: ENTERPRISE LAB & DEFENSIVE SECURITY

## Lesson 5.1 — Plan an enterprise-style home lab
**Goal:** Design an environment that resembles a small company without touching real company systems.

**Example architecture:**

```text
                 [Your regular home network]
                            |
                    [Lab hypervisor]
                            |
                 [ISOLATED LAB NETWORK]
                   10.50.10.0/24
                   /      |       \
       DC1 / DNS      CLIENT1    UBUNTU-DB
       10.50.10.10    .20        .30
```

**Steps:**
1. Write an asset list (hostname, role, OS, IP, owner, backup policy).
2. Define one isolated lab subnet and avoid overlapping your home network's addressing.
3. Create an admin account separate from your ordinary test user.
4. Record required ports and services (DNS, Kerberos, LDAP, SMB, SSH, database) without opening them to the internet.
5. Create a backup plan and a recovery test plan.
6. Create a change log for every major configuration modification.

**Expected:** A network diagram, IP plan, and access-control matrix.  
**Practice:** Explain the failure impact if `DC1` is unavailable.  
**Check:** [ ] I can document an enterprise-style environment.

## Lesson 5.2 — GOAD (Game of Active Directory): safe learning orientation
**What is GOAD?** A deliberately vulnerable Active Directory lab for studying attack paths and defensive controls. It is **not** a production-ready domain. Use only in a fully isolated environment with explicit authorization.

**Preparation and observation steps:**
1. Read the official GOAD project's documentation and identify its supported hypervisors, resource requirements, and provisioning method for the version you choose.
2. Provision a **dedicated isolated virtual network** with no inbound access from the internet and no routes to production systems.
3. Create a lab snapshot/backup before deployment.
4. Deploy the environment using the **current official instructions**, checking every VM is attached to the intended isolated network.
5. Inventory domain controllers, member servers, clients, and their lab IPs.
6. Observe Windows security logs for authentication activity; document account/group memberships and excessive permissions.
7. Practice **defensive** remediation in a cloned copy: remove unnecessary group privileges, improve password policy, review service accounts, and compare before/after audit logs.
8. Revert to the original snapshot when finished.

**Expected:** A diagram and defensive findings report.  
**Troubleshooting:** If provisioning fails, check official version-specific dependencies, RAM, virtualization support, and network isolation before retrying.  
**Practice:** Write three examples of privilege misconfiguration and the principle that prevents each.  
**Check:** [ ] I can operate a vulnerable training domain without exposing other networks.

## Lesson 5.3 — Adaptix C2: understand the concept and detect misuse
**What is Adaptix C2?** A command-and-control framework associated with adversary emulation and post-compromise operations. It is dual-use and should be studied only in an explicitly authorized, isolated environment. This course focuses on recognizing C2 behavior, logging, and defensive investigation—not deploying agents or controlling other machines.

**Defensive observation exercise:**
1. Review the framework's publicly documented architecture at a conceptual level: operator interface, server, agents, and communications.
2. Draw a diagram of how a hypothetical endpoint might communicate with an external command server.
3. In your lab, enable Windows event logging and a network traffic capture tool on **your own** test VM.
4. Generate harmless normal traffic, such as opening `https://example.com`, and record destination, protocol, and timing.
5. Compare this baseline to *hypothetical* C2 indicators: unexpected periodic outbound connections, unfamiliar processes making network requests, unusual DNS activity, or suspicious parent-child processes.
6. Write a triage checklist: verify host ownership, preserve logs, isolate if necessary, investigate process ancestry, assess scope, and escalate.

**Expected:** A defensive detection and triage worksheet.  
**Troubleshooting:** Periodic network traffic alone is not malicious; legitimate updaters and monitoring agents also communicate regularly.  
**Practice:** Explain what evidence is needed before declaring an incident.  
**Check:** [ ] I understand C2 concepts and defensive telemetry.

## Lesson 5.4 — Final capstone: build, operate, and recover a small IT environment
**Project:** Create a documented lab with one Windows domain controller, one Windows client, one Linux database VM, and an optional Proxmox host and Tailscale access.

**Step-by-step deliverables:**
1. Draw the network and IP addressing plan.
2. Install VMs on an isolated lab network.
3. Configure AD DS/DNS and create an ordinary user and a security group.
4. Join the client to the domain and verify user login.
5. Install PostgreSQL, create a `devices` table, and query it.
6. Use PowerShell to export an inventory CSV.
7. Install approved patches and verify services.
8. Back up at least one VM, then restore it to a separate test VM.
9. Capture one Windows event and one Linux log, and write a troubleshooting note.
10. Review firewall rules and account privileges.
11. Write a two-page operations runbook covering login, updates, backups, recovery, and incident escalation.
12. Present evidence: screenshots with secrets hidden, command outputs, diagram, and a short reflection on what failed and how you fixed it.

**Pass criteria:** You can rebuild core services, explain network dependencies, demonstrate a successful restore, and justify the security controls you chose.

---

# APPENDIX A — QUICK COMMAND REFERENCE

| Task | Windows | Linux |
|---|---|---|
| Show IP | `ipconfig /all` | `ip -br addr` |
| Show route | `route print` | `ip route` |
| Test connectivity | `ping 1.1.1.1` | `ping -c 4 1.1.1.1` |
| Resolve DNS | `nslookup example.com` | `getent hosts example.com` |
| Show hostname | `hostname` | `hostname` |
| Show user | `whoami` | `whoami` |
| List files | `dir` | `ls -la` |
| Check services | `Get-Service` (PowerShell) | `systemctl --failed` |
| Show recent logs | `Get-WinEvent -LogName System -MaxEvents 10` | `journalctl -n 30` |

# APPENDIX B — OPTIONAL TOPICS FOR CONTINUED EXPERT STUDY

- IPv6, subnetting and CIDR, TCP/UDP, ports, NAT, VPNs, routing protocols, STP, LACP, Wi-Fi security
- Windows Server DNS/DHCP roles, Group Policy troubleshooting, AD replication, certificate services, PKI
- Linux SSH hardening, Bash, cron/systemd timers, storage/LVM, permissions, SELinux/AppArmor
- Proxmox clustering, high availability, Ceph, ZFS, backup server, disaster recovery
- SQL joins, indexes, transactions, role-based database access, database backup/restore
- Containers, Docker, Kubernetes fundamentals, infrastructure as code, Git, Ansible, CI/CD
- Cloud IAM, networking, security groups, logging, cost controls, shared responsibility
- SIEM, endpoint detection, secure configuration baselines, threat modeling, threat hunting
- Network troubleshooting with Wireshark, `tcpdump`, and packet-flow diagrams
- Zero trust, identity governance, secrets management, business continuity, and risk assessment
- **GaroteNet:** The name may refer to a specific product or project; verify the exact spelling and official documentation before attempting installation. If you mean a particular tool, add its official link to your lab notes.

# APPENDIX C — REUSABLE LESSON NOTES TEMPLATE

```markdown
## Lesson title:
Date:
Goal:
What it means (in my words):
My lab machine / OS:
Network/IP used:
Commands or clicks performed:
Expected result:
Actual result:
Error messages:
How I fixed the issue:
Security precautions:
What I learned:
[ ] I can repeat the task without help
```

**Next step:** Begin with Level 1, Lesson 1.1. When all Level 1 checks are complete, move to Level 2. Do not use vulnerable lab tooling outside isolated, authorized systems.
