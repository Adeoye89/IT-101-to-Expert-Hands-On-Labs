# Proxmox VE 101 — Hands-On Training (Beginner to Advanced)

> **Goal:** Build and operate a safe, small virtualization lab. Work through each lab in order. Commands marked **HOST** run in the Proxmox shell; commands marked **GUEST** run inside a VM. Examples use `pve1.lab.example` and documentation addresses; replace them with your own.

## Contents
1. [Concepts and VM types](#1-concepts-and-vm-types)
2. [Hardware and safety](#2-hardware-and-safety)
3. [Install Proxmox VE](#3-install-proxmox-ve)
4. [Web interface tour](#4-web-interface-tour)
5. [Create an Ubuntu VM](#5-create-an-ubuntu-vm)
6. [Create a Windows Server VM](#6-create-a-windows-server-vm)
7. [Create a Kali VM](#7-create-a-kali-vm)
8. [Network bridges and isolated lab](#8-network-bridges-and-isolated-lab)
9. [Storage, snapshots, backups](#9-storage-snapshots-backups)
10. [Containers and templates](#10-containers-and-templates)
11. [CLI and troubleshooting](#11-cli-and-troubleshooting)
12. [Advanced practice and checklist](#12-advanced-practice-and-checklist)

## 1. Concepts and VM types
**Proxmox VE** is a Debian-based virtualization platform with KVM virtual machines, LXC containers, a web interface, and management tools.

- **Host/node:** the physical computer running Proxmox.
- **VM:** an independent virtual computer with its own kernel, virtual CPU, RAM, disks, and NICs.
- **LXC container:** isolated Linux userspace sharing the Proxmox host's kernel; lighter than a VM, but not equivalent isolation.
- **Hypervisor:** software that manages virtualized hardware. KVM is integrated into Linux.
- **ISO:** installation media image; **template:** reusable starting image; **clone:** copied VM; **snapshot:** rollback point; **backup:** independently stored recoverable copy.
- **Type 1 vs Type 2:** Proxmox is a bare-metal-oriented platform; VirtualBox is commonly hosted inside a desktop OS.

**Example lab:** `pve1` hosts Ubuntu (VM 100), Windows Server (VM 110), Kali (VM 120). Allocate conservatively: 2 vCPU, 2–4 GiB RAM, 30–50 GiB disk per VM, adjusting for guest requirements. Running all simultaneously requires sufficient host resources.

## 2. Hardware and safety
**Recommended:** spare x86-64 computer with hardware virtualization (Intel VT-x/AMD-V), 16–32 GiB RAM, SSD, wired Ethernet, and a separate backup drive. Check the supported hardware and release requirements in current Proxmox documentation.

**Important:** The Proxmox installer can erase the selected disk. Back up all files and disconnect disks you do not want to touch. Use a **dedicated spare machine** rather than your daily-use Windows laptop. Record your router's subnet, gateway, DNS, and a free/static management address; avoid IP conflicts.

**Preflight checklist:**
- [ ] Backup verified and correct installation drive identified.
- [ ] Virtualization enabled in BIOS/UEFI.
- [ ] Management Ethernet connected and network information recorded.
- [ ] Installer ISO downloaded from the official site: https://www.proxmox.com/en/downloads

## 3. Install Proxmox VE
1. Download the current Proxmox VE ISO from the official site.
2. Use a USB imaging application to write the ISO to a USB drive (**this overwrites the USB**).
3. Boot the spare machine from USB using its boot menu.
4. Select the graphical installer; accept the license.
5. **Carefully select the intended target disk.** Confirm you are not erasing your main computer's drive.
6. Set country/time zone, a strong root password, and administrator email.
7. Configure management networking. **Illustrative only:** hostname `pve1.lab.example`, IP `192.168.1.50/24`, gateway `192.168.1.1`, DNS `192.168.1.1`. Use real values for your LAN.
8. Review the summary, install, remove USB, and reboot.
9. On another device that can reach the management subnet, open `https://192.168.1.50:8006` (substitute your real IP). The initial self-signed certificate may trigger a warning: verify you reached your server before proceeding.
10. Sign in as `root`, realm **Linux PAM**, using the password you set.

**Verify (HOST):** open **Node → Shell** and run:
```bash
hostnamectl
pveversion
ip -br addr
ip route
```
Expected: Proxmox version, configured hostname/IP, and default route. If no web access, confirm power, Ethernet, IP address, firewall, and port 8006 reachability. Do not expose the management UI directly to the public internet.

## 4. Web interface tour
- **Datacenter:** settings across nodes, permissions, storage, backups.
- **Node (pve1):** CPU/memory status, shell, disks, network, updates.
- **local:** commonly holds ISO images, backups, templates (depends on storage configuration).
- **local-lvm:** commonly stores VM disks; not generally ISO images.
- **Create VM / Create CT:** start a virtual machine or container.

**Practice:** Click your node → **Summary**; write down CPU, memory, storage usage. Go to **Updates**, review pending updates and repository notices. Apply updates during a maintenance window after backups; subscription and repository choices depend on your installation.

## 5. Create an Ubuntu VM
### A. Upload installation media
1. Download Ubuntu Server ISO from https://ubuntu.com/download/server .
2. In Proxmox select node → `local` storage → **ISO Images → Upload**.
3. Select the ISO; wait for upload completion.

### B. VM wizard
1. Click **Create VM**.
2. **General:** VM ID `100`, name `ubuntu-lab01`.
3. **OS:** select Ubuntu Server ISO; Linux guest.
4. **System:** use defaults for the first lab.
5. **Disks:** `scsi0`, ~32 GiB, suitable storage (for example `local-lvm`), SCSI controller **VirtIO SCSI single** where offered.
6. **CPU:** 2 cores. **Memory:** 4096 MiB (or 2048 MiB for a lighter server setup).
7. **Network:** bridge `vmbr0`, model **VirtIO**.
8. Confirm, click **Finish**. Select VM 100 → **Start → Console**.
9. Follow Ubuntu installer: language, network via DHCP, username, disk (the VM's virtual disk only), optional OpenSSH, install, reboot. Detach ISO if the installer reappears.

**GUEST verification:**
```bash
ip -br addr
ip route
ping -c 4 1.1.1.1
getent hosts example.com
sudo apt update
```
The first commands inspect IP/route, ping tests connectivity, `getent` tests DNS, and `apt update` refreshes package metadata.

### C. Guest agent
In Ubuntu (**GUEST**):
```bash
sudo apt install -y qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
systemctl status qemu-guest-agent --no-pager
```
In Proxmox: shut down VM → **Options → QEMU Guest Agent → Enabled** → start VM. Agent functionality depends on guest and settings; verify its status in the VM.

## 6. Create a Windows Server VM
1. Obtain a legitimate Windows Server evaluation ISO from Microsoft and the appropriate **VirtIO driver ISO** from the Fedora VirtIO project. Check licensing/evaluation terms.
2. Upload both ISOs to Proxmox ISO storage.
3. **Create VM:** ID `110`, name `win-server-lab`, Windows OS type/version matching your ISO, ~2–4 cores, 4–8 GiB RAM, 60+ GiB disk as hardware permits. Attach Windows installation ISO and the VirtIO driver ISO as a second CD/DVD drive.
4. For Windows guests, select compatible firmware/machine settings. If using VirtIO disk and the installer shows no disks, select **Load driver**, browse the VirtIO ISO to the matching storage driver for your Windows version, and load it.
5. Install Windows to the **VM virtual disk**, create an administrator password, and sign in.
6. Install the matching VirtIO guest drivers and QEMU guest agent from the trusted driver ISO; enable the guest agent in Proxmox after shutdown.
7. Set an appropriate VM name and apply Windows updates. Use a private lab network for Active Directory experiments; don't install AD DS until DNS and network planning are understood.

**Verify (PowerShell in Windows GUEST):**
```powershell
Get-ComputerInfo | Select-Object CsName,WindowsProductName
Get-NetIPConfiguration
Test-NetConnection 1.1.1.1
```

## 7. Create a Kali VM
1. Download the Kali Installer ISO from https://www.kali.org/get-kali/ (choose an installer compatible with your architecture).
2. Upload ISO to `local` → ISO Images.
3. Create VM `120`, name `kali-lab`, Linux guest, 2 vCPU, 4 GiB RAM, ~40 GiB disk, VirtIO NIC.
4. Attach to the **isolated lab bridge** from Section 8 for security exercises; you may temporarily use a controlled outbound connection to update packages.
5. Start → Console; follow Kali's installer, create your own credentials, reboot.
6. Inside Kali (**GUEST**):
```bash
ip -br addr
sudo apt update
sudo apt full-upgrade -y
```
7. Shut down and take a clean snapshot before lab changes.

**Safety:** Run security testing only against machines you own or have explicit authorization to test. Keep intentionally vulnerable targets and offensive tools on an isolated network.

## 8. Network bridges and isolated lab
**`vmbr0`** often bridges the physical NIC to your LAN. A second **Linux bridge without physical ports**, e.g. `vmbr1`, can create an isolated virtual switch.

### GUI: create isolated bridge
1. Node → **System → Network → Create → Linux Bridge**.
2. Name `vmbr1`; leave **Bridge ports** empty.
3. Do not add a gateway or physical NIC. For an isolated Layer-2 network, no host IP is required.
4. Apply configuration cautiously (network changes can disconnect management access). Prefer local console access for risky changes.
5. Shut down each lab VM → **Hardware → Network Device → Edit → Bridge: vmbr1**.
6. Restart VMs. Without DHCP on `vmbr1`, configure static guest addresses such as:
   - Ubuntu `192.168.56.10/24`
   - Windows `192.168.56.20/24`
   - Kali `192.168.56.30/24`
   - No default gateway on the isolated adapter.
7. Ping between VMs; check guest firewall rules if ping is blocked. This network has no internet by default, which is intentional.

**Concept:** A bridge acts like a virtual switch. A gateway/router is needed for traffic between subnets; adding one changes isolation. Avoid bridging security-lab targets directly to a production LAN.

## 9. Storage, snapshots, backups
### Inspect storage
In web UI: **Datacenter → Storage**; inspect content types, capacity, and storage IDs. **HOST:**
```bash
pvesm status
lsblk -f
```
`pvesm status` lists configured Proxmox storage; `lsblk -f` shows block devices/filesystems. Never format a device unless you positively identify it and have a backup.

### Snapshot (rollback, not backup)
1. VM 100 → **Snapshots → Take Snapshot**.
2. Name `clean-ubuntu`; enter a description.
3. After a safe test change, restore snapshot if needed. Restoring can discard later changes; check whether the VM/storage supports snapshots.

### Backup and restore
1. Select VM → **Backup → Backup now**.
2. Choose backup storage with enough capacity, mode `Snapshot` if supported, and compression.
3. Run and verify task log. Prefer backups stored on a separate disk/system from the host.
4. To restore, select backup under storage **Backups → Restore**; choose a **new VM ID** for a test restore so you don't overwrite the original.
5. Boot restored VM and validate login, files, and networking. Avoid duplicate IP/hostname conflicts when both copies run.

**HOST CLI examples:**
```bash
qm list
qm status 100
vzdump 100 --mode snapshot --storage local --compress zstd
```
`vzdump` works only if the selected storage supports backups and has adequate space; replace `local` as necessary.

## 10. Containers and templates
1. Node → `local` → **CT Templates → Templates**; download a trusted Debian or Ubuntu template.
2. Click **Create CT**; set ID `200`, hostname `ct-lab`, password or SSH key, disk ~8–16 GiB, 1–2 cores, 1–2 GiB RAM.
3. Select template and appropriate bridge/network settings; use an unprivileged container unless a documented need requires otherwise.
4. Finish, start, open Console, and run:
```bash
cat /etc/os-release
ip -br addr
```
**Difference:** An LXC container shares the host kernel, so don't treat it as a full VM for kernel testing or high-risk untrusted workloads.

**Template workflow:** Install/patch a VM, remove machine-specific secrets, configure guest initialization (such as cloud-init when appropriate), shut down, and convert to template. Clone the template to create new VMs. Never bake credentials into shared templates.

## 11. CLI and troubleshooting
**HOST commands:**
```bash
pveversion             # show Proxmox version
qm list                # list QEMU VMs
qm config 100          # show VM 100 configuration
qm status 100          # show power status
pct list               # list LXC containers
pvesm status           # storage health/capacity
ip -br addr            # host IP addresses
ip route               # host routes
journalctl -u pveproxy -n 50 --no-pager  # recent web proxy logs
```
**Start/shutdown VM (HOST):**
```bash
qm start 100
qm shutdown 100
```
Use `qm stop 100` only when a graceful shutdown fails and you accept the risk of unsaved data loss.

| Symptom | What to check |
|---|---|
| Web UI unreachable | Host IP, Ethernet, port 8006, `systemctl status pveproxy`, firewall |
| VM no internet | Bridge selection, guest IP/gateway/DNS, host/LAN connectivity |
| Installer cannot see disk | Guest storage driver, VirtIO ISO, disk controller |
| VM fails to start | Available RAM, storage space, task log, VM configuration |
| Slow VM | CPU/RAM pressure, disk I/O, guest drivers, excessive simultaneous VMs |
| Snapshot unavailable | Storage format and snapshot support |

## 12. Advanced practice and checklist
**Advanced topics to research after mastering the basics:** storage replication and ZFS, VLAN-aware bridges, Proxmox firewall, RBAC, clustering/quorum, HA, PCI passthrough, API tokens, automation, monitoring, and disaster recovery. Never build a cluster or change management networking without understanding quorum and rollback procedures.

### Capstone
- [ ] Install Proxmox on a spare host and log into port 8006.
- [ ] Create Ubuntu VM 100 and verify networking.
- [ ] Create Windows Server VM 110 and install drivers.
- [ ] Create Kali VM 120 on an isolated bridge.
- [ ] Explain NAT, bridge, guest IP, subnet, and default gateway.
- [ ] Create and restore a snapshot.
- [ ] Back up a VM and **test restore to a different ID**.
- [ ] Create an unprivileged LXC container.
- [ ] Use `qm list`, `qm status`, `pvesm status`, and review logs.

### Next module
Proceed to **Tailscale 101** to connect trusted administrative devices securely without publishing Proxmox management ports on the internet.

## Official references
- https://pve.proxmox.com/pve-docs/
- https://www.proxmox.com/en/downloads
- https://ubuntu.com/download/server
- https://www.kali.org/get-kali/
