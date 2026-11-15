# Servers 101 to Expert — Complete Hands-On Deployment and Administration

> **Goal:** Plan, build, configure, secure, operate, troubleshoot, back up, and recover a small enterprise server environment. Designed for a GitHub `README.md` and an isolated, authorized home lab.

## Table of contents
1. [What a server is](#1-what-a-server-is)
2. [Planning and hardware](#2-planning-and-hardware)
3. [Lab architecture and addressing](#3-lab-architecture-and-addressing)
4. [Install Proxmox VE](#4-install-proxmox-ve)
5. [Build an Ubuntu Server VM](#5-build-an-ubuntu-server-vm)
6. [Build a Windows Server VM](#6-build-a-windows-server-vm)
7. [Networking, DNS and DHCP](#7-networking-dns-and-dhcp)
8. [Active Directory and domain join](#8-active-directory-and-domain-join)
9. [Linux services and web hosting](#9-linux-services-and-web-hosting)
10. [Storage, shares and permissions](#10-storage-shares-and-permissions)
11. [SSH, RDP and remote management](#11-ssh-rdp-and-remote-management)
12. [Hardening and updates](#12-hardening-and-updates)
13. [Monitoring and logs](#13-monitoring-and-logs)
14. [Backups and restore drills](#14-backups-and-restore-drills)
15. [Automation](#15-automation)
16. [Troubleshooting playbook](#16-troubleshooting-playbook)
17. [Operations and change management](#17-operations-and-change-management)
18. [Capstone projects](#18-capstone-projects)
19. [Final skills checklist](#19-final-skills-checklist)

---

## 1. What a server is

A **server** is a computer or program that supplies a service to clients over a network. A single machine can run several services, but separating critical roles improves security and reliability.

| Role | Typical function | Example software | Common ports |
|---|---|---|---|
| Domain controller | Identity and authentication | Active Directory Domain Services | DNS 53, Kerberos 88, LDAP 389 |
| DNS server | Names to IP addresses | Windows DNS, BIND | TCP/UDP 53 |
| DHCP server | Assigns IP configuration | Windows DHCP, Kea | UDP 67/68 |
| Web server | Websites and APIs | Nginx, Apache, IIS | TCP 80/443 |
| File server | Shared folders | SMB, Samba, NFS | SMB TCP 445 |
| Database server | Structured application data | PostgreSQL, SQL Server | Product-specific |
| Hypervisor | Runs virtual machines | Proxmox VE | Management HTTPS 8006 |
| Backup server | Stores recovery copies | Proxmox Backup Server | Product-specific |

**Terms:** CPU = compute; RAM = working memory; NIC = network interface; RAID = disk redundancy, **not a backup**; hypervisor = VM host; VM = virtual machine; container = isolated application environment sharing a kernel; UPS = battery-backed power supply.

**Exercise:** Identify the server role behind visiting a website, signing into a company PC, and opening a shared folder.

## 2. Planning and hardware

Before installation, document: intended users, workloads, OS licensing, CPU/RAM, disk IOPS and capacity, NIC speed, backup retention, recovery objectives, security boundaries, and maintenance window.

**Suggested learning host:** 64-bit CPU with virtualization support, 32 GB RAM preferred (16 GB for a smaller lab), SSD storage, wired Ethernet, and reliable backups. Windows Server evaluation/licensing terms must be checked before use.

**BIOS/UEFI checklist:** Enable Intel VT-x/AMD-V; enable virtualization-related IOMMU only if needed; confirm boot order; update firmware according to vendor guidance; document BIOS settings. For a physical Proxmox host, installation can erase the target disk—back up first.

**Storage choices:**
- **Directory:** straightforward file-backed images; commonly supports qcow2 and ISO images.
- **LVM-thin:** efficient block-backed VM disks and snapshots; no qcow2 file path to browse.
- **ZFS:** checksums, snapshots and optional redundancy; needs sufficient RAM and careful disk planning.
- **NFS/iSCSI:** shared network storage; requires a dependable network and storage server.

**Capacity rule:** Estimate provisioned virtual disks, expected *actual* usage, snapshot growth, backups, and at least 20% operational headroom. Do not confuse thin provisioning with unlimited physical space.

## 3. Lab architecture and addressing

Use an isolated lab VLAN or a Proxmox-only bridge. Do not accidentally deploy a second DHCP server onto a home or corporate production LAN.

```text
                  Trusted admin laptop
                          |
                  Router / firewall
                          |
                  Managed switch
                    VLAN 10 (lab)
                          |
                     Proxmox host
                      vmbr0 bridge
              +-----------+-----------+
              |           |           |
          DC01          LNX01        WIN11-01
       Windows Server  Ubuntu Server  Windows client
       AD/DNS/DHCP     SSH/Nginx      Domain joined
```

| System | Example IP | Role |
|---|---|---|
| Gateway | 192.168.56.1 | Lab router |
| Proxmox | 192.168.56.5 | Hypervisor management |
| DC01 | 192.168.56.10 | AD DS / DNS / optional DHCP |
| LNX01 | 192.168.56.20 | Linux / web / SSH |
| WIN11-01 | DHCP or 192.168.56.30 | Client |
| Lab domain | `corp.example.test` | Documentation-only sample domain |

**Network prerequisite:** The example gateway `192.168.56.1` exists only if you configure one. Without a router, same-subnet VM communication works but internet access and inter-subnet routing do not. If using VLAN 10, both switch and Proxmox tagging must agree; an untagged isolated bridge is also valid.

## 4. Install Proxmox VE

1. Download the current Proxmox VE installer ISO from the official site and verify its checksum.
2. Create bootable installation media using a trusted imaging tool.
3. Boot a **dedicated lab machine**; select the intended disk carefully.
4. Choose filesystem, country, time zone, administrator password, email, hostname, management IP, gateway, and DNS.
5. After reboot, open `https://192.168.56.5:8006` from a machine that can reach the management network.
6. Sign in using the configured administrative realm and credentials; follow your environment's subscription/repository instructions.

**On the Proxmox shell:**

```bash
pveversion -v
ip -br addr
ip route
pvesm status
lsblk -f
```

Expected: version details, management interface, route (if configured), available storages, and disks.

**Create VM via GUI:** Datacenter → node → **Create VM** → General (VM ID/name) → OS (ISO) → System (UEFI/BIOS as appropriate) → Disks (storage/size) → CPU → Memory → Network (bridge/VLAN) → Confirm. For guest agents, install the agent *inside* the guest and enable the VM's QEMU Guest Agent option.

**CLI inspection:**

```bash
qm list
qm config 101
pct list
```

Replace `101` with an existing VM ID. Do not copy commands that modify an existing VM without checking its configuration first.

## 5. Build an Ubuntu Server VM

**Example VM:** `LNX01`, 2 vCPU, 4 GB RAM, 30 GB disk, VirtIO NIC. Mount a supported Ubuntu Server ISO and install with OpenSSH Server selected. Choose a non-root administrator user.

**Discover the network:**

```bash
ip -br addr
ip route
resolvectl status
hostnamectl
```

**Configure a static IP with Netplan** (adjust the actual interface name and YAML indentation):

```yaml
# /etc/netplan/99-lab.yaml
network:
  version: 2
  ethernets:
    ens18:
      dhcp4: false
      addresses: [192.168.56.20/24]
      routes:
        - to: default
          via: 192.168.56.1
      nameservers:
        addresses: [192.168.56.10]
        search: [corp.example.test]
```

> If the lab has no gateway yet, omit the default route. Configure DNS only after DC01 DNS is operational; use a working resolver temporarily for package installation. When working remotely, use `sudo netplan try` first to allow automatic rollback.

```bash
sudo netplan try
ip -br addr
ping -c 3 192.168.56.10
```

**Initial packages:**

```bash
sudo apt update
sudo apt install -y openssh-server qemu-guest-agent curl dnsutils
sudo systemctl enable --now ssh qemu-guest-agent
systemctl status ssh --no-pager
```

Expected: `ssh` active; guest agent active when supported by the hypervisor.

## 6. Build a Windows Server VM

1. Upload a licensed/evaluation Windows Server ISO and the matching VirtIO driver ISO to Proxmox.
2. Create `DC01`: 2–4 vCPU, 4–8 GB RAM, 60+ GB disk (adjust for actual workload).
3. Choose UEFI or BIOS deliberately; add an EFI disk when using OVMF.
4. If Windows setup cannot see a VirtIO disk, use **Load driver** and browse the correct storage driver on the VirtIO ISO, matching Windows release and architecture.
5. Complete setup, set the local Administrator password, install VirtIO network/storage drivers and QEMU Guest Agent.
6. Rename the computer to `DC01`, reboot, and assign the static IP `192.168.56.10/24` with the lab gateway if available.

**PowerShell as Administrator:**

```powershell
Get-ComputerInfo | Select-Object CsName, WindowsProductName
Get-NetAdapter
Get-NetIPConfiguration
Get-Service QEMU-GA -ErrorAction SilentlyContinue
```

**Set a static IP** (confirm adapter name, IP availability, and gateway first):

```powershell
Get-NetAdapter
# Example only; run after verifying your adapter and topology:
New-NetIPAddress -InterfaceAlias 'Ethernet' -IPAddress 192.168.56.10 -PrefixLength 24 -DefaultGateway 192.168.56.1
Set-DnsClientServerAddress -InterfaceAlias 'Ethernet' -ServerAddresses 192.168.56.10
```

If an IP or gateway is already configured, use `Get-NetIPAddress` and `Remove-NetIPAddress` / `Set-NetIPInterface` as appropriate instead of blindly adding duplicates. Do not configure an unreachable gateway.

## 7. Networking, DNS and DHCP

**Know the path:** guest NIC → Proxmox bridge → physical NIC → switch port/VLAN → router/firewall → destination. A VM can be powered on while disconnected at any point along that path.

**Linux checks:**

```bash
ip -br link
ip route
ping -c 3 192.168.56.1
dig @192.168.56.10 dc01.corp.example.test
sudo ss -lntup
```

**Windows checks:**

```powershell
ipconfig /all
Test-NetConnection 192.168.56.20 -Port 22
Resolve-DnsName dc01.corp.example.test -Server 192.168.56.10
route print
```

**Proxmox VLANs:** On a VLAN-aware Linux bridge, a VM NIC may use `bridge=vmbr0,tag=10`; the physical switch uplink must carry VLAN 10. Avoid tagging a VM *and* tagging the same traffic inside the guest unless deliberately implementing nested VLANs.

**DHCP:** In an isolated subnet, install/configure Windows DHCP *only if no other DHCP server serves that subnet*. Create a scope such as `192.168.56.100–192.168.56.199`, mask `/24`, router `192.168.56.1` (only if present), DNS `192.168.56.10`, and DNS suffix `corp.example.test`. Authorize Windows DHCP in AD when required. Test with a client `ipconfig /renew` and `ipconfig /all`.

## 8. Active Directory and domain join

**On DC01, PowerShell as Administrator:**

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Install-ADDSForest -DomainName 'corp.example.test' -DomainNetbiosName 'CORP' -InstallDNS
```

The second command prompts for a Directory Services Restore Mode password and normally reboots. Use this only on a new, intended lab DC.

**Verify after reboot:**

```powershell
Get-ADDomain
Get-ADForest
Get-Service NTDS,DNS
Resolve-DnsName -Type SRV _ldap._tcp.dc._msdcs.corp.example.test
```

**Create a test OU and user:**

```powershell
New-ADOrganizationalUnit -Name 'LabUsers' -Path 'DC=corp,DC=example,DC=test'
$Password = Read-Host 'Temporary password' -AsSecureString
New-ADUser -Name 'Lab Student' -SamAccountName 'labstudent' `
  -Path 'OU=LabUsers,DC=corp,DC=example,DC=test' `
  -AccountPassword $Password -Enabled $true -ChangePasswordAtLogon $true
Get-ADUser labstudent
```

**Join Windows client:** Set the client's DNS to `192.168.56.10` first. Then Settings → System → About → Domain or workgroup (UI varies), or run as Administrator:

```powershell
Add-Computer -DomainName 'corp.example.test' -Credential 'CORP\Administrator' -Restart
```

**Verify:** `whoami`, `gpresult /r`, `nltest /dsgetdc:corp.example.test`.

**Warning:** Never join cloned production domain controllers or restore a DC by simply copying its disk into an active production domain. Use supported AD recovery procedures.

## 9. Linux services and web hosting

**Install a simple web server:**

```bash
sudo apt update
sudo apt install -y nginx
sudo systemctl enable --now nginx
systemctl is-active nginx
curl -I http://127.0.0.1
```

Expected: `active` and an HTTP response. Create a test landing page:

```bash
echo '<h1>LNX01 Lab Server</h1>' | sudo tee /var/www/html/index.html
curl http://127.0.0.1
```

From Windows: browse to `http://192.168.56.20` or test `Test-NetConnection 192.168.56.20 -Port 80`.

**Understand systemd:**

```bash
systemctl status nginx --no-pager
journalctl -u nginx -n 30 --no-pager
sudo nginx -t
sudo systemctl reload nginx
```

For a public-facing site, use HTTPS certificates, DNS ownership, reverse-proxy rules, patching, and a firewall; don't expose a practice site directly to the internet.

## 10. Storage, shares and permissions

**Linux:**

```bash
lsblk -f
df -hT
sudo du -sh /var/log
findmnt
```

Before adding disks, identify devices by stable identifiers (`/dev/disk/by-id/`) and verify they are not in use. Formatting a disk destroys data. For production, document filesystem, mount options, encryption, backups, and capacity alerts.

**Windows SMB file share (lab):** Create `C:\LabShare` in File Explorer, then configure **both share permissions and NTFS permissions**. Prefer domain groups rather than individual users; use least privilege. Test from a domain-joined client using `\\DC01\LabShare` (or a dedicated file server, preferred for production).

**PowerShell inspection:**

```powershell
Get-Volume
Get-SmbShare
Get-SmbShareAccess -Name 'LabShare'
Get-Acl 'C:\LabShare' | Format-List
```

**Permissions concept:** Effective access depends on both SMB share permissions and NTFS ACLs. A permissive share does not override restrictive NTFS permissions.

## 11. SSH, RDP and remote management

**SSH to Ubuntu from Windows:**

```powershell
ssh student@192.168.56.20
```

Verify the SSH host-key fingerprint through a trusted channel before accepting it. Create an Ed25519 key on your workstation:

```powershell
ssh-keygen -t ed25519 -C 'lab-admin'
```

Copy the `.pub` contents to the Linux account's `~/.ssh/authorized_keys`; secure the directory (`chmod 700 ~/.ssh`) and file (`chmod 600 ~/.ssh/authorized_keys`). Test key authentication before considering password-authentication changes.

**RDP:** Enable Remote Desktop on the intended Windows server, restrict allowed accounts, and permit TCP 3389 **only from trusted management networks**. Never expose RDP directly to the internet. Use a VPN or a secured management gateway.

**Proxmox management:** Restrict HTTPS port 8006 and SSH port 22 to administrators; use strong authentication, MFA where supported, and role-based access. Do not expose the Proxmox management interface publicly.

## 12. Hardening and updates

**Baseline checklist:** unique administrator accounts; MFA for management; least privilege; disable unused services; host firewalls; security updates; NTP/time sync; endpoint protection where appropriate; encryption; logging; backups; and documented emergency access.

**Ubuntu:**

```bash
sudo apt update && sudo apt upgrade
sudo ufw status verbose
sudo ss -lntup
sudo systemctl --failed
```

Before enabling UFW on a remote machine, explicitly allow SSH from the trusted management subnet and verify a second session can connect:

```bash
sudo ufw allow from 192.168.56.0/24 to any port 22 proto tcp
sudo ufw allow from 192.168.56.0/24 to any port 80 proto tcp
sudo ufw enable
```

**Windows:**

```powershell
Get-NetFirewallProfile
Get-HotFix | Select-Object -First 10
Get-LocalGroupMember Administrators
Get-Service | Where-Object Status -eq 'Running' | Select-Object -First 20
```

For enterprise systems, follow an approved baseline and staged patch process. Avoid disabling firewalls, antivirus, SELinux/AppArmor, or authentication controls as a troubleshooting shortcut.

## 13. Monitoring and logs

Track **availability, CPU, memory, disk free space, I/O latency, backup success, certificate expiration, authentication failures, and application response**.

**Linux:**

```bash
uptime
free -h
df -h
journalctl -p warning -b --no-pager | tail -n 30
sudo ss -tulpn
```

**Windows:**

```powershell
Get-Process | Sort-Object CPU -Descending | Select-Object -First 10
Get-WinEvent -LogName System -MaxEvents 20 | Select-Object TimeCreated,Id,LevelDisplayName,Message
Get-Counter '\Processor(_Total)\% Processor Time' -SampleInterval 1 -MaxSamples 3
```

**Proxmox:** use node and VM Summary panels, Tasks, Syslog/Journal, storage status, and alerts. Central monitoring options include Zabbix, Prometheus/Grafana, and syslog collectors. Define alert thresholds and test delivery rather than assuming alerts work.

## 14. Backups and restore drills

**Key terms:** RPO = acceptable amount of data loss measured in time; RTO = acceptable recovery duration. A snapshot is not an independent backup. Keep copies outside the primary host; protect them from deletion/ransomware.

**Proxmox GUI backup:** Datacenter → Backup → Add job; choose storage, schedule, selection, mode, retention and notifications. Confirm the destination supports backups and has enough free space. For small labs, use an external backup target; for more advanced environments, consider Proxmox Backup Server.

**Manual CLI example:**

```bash
vzdump 101 --mode snapshot --storage BACKUP_STORAGE --compress zstd
```

Replace `101` and `BACKUP_STORAGE` with verified values; snapshot mode depends on guest/storage support. Check task output and resulting archive.

**Restore test:**
1. Select a non-production test VM ID and isolated network.
2. Restore the backup through the Proxmox GUI or supported CLI.
3. Boot the restored VM; validate OS, applications, services, data, and permissions.
4. Record recovery time and the timestamp of recovered data.
5. Delete test artifacts only after confirming they are not needed.

**Windows AD-specific note:** Domain controllers require application-aware/system-state backup and supported AD recovery methods; generic VM disk restore is not a substitute for an AD disaster recovery plan.

## 15. Automation

**Bash health script — `server-health.sh`:**

```bash
#!/usr/bin/env bash
set -euo pipefail
printf 'Host: %s\n' "$(hostname)"
printf 'Time: %s\n' "$(date -Is)"
uptime
free -h
df -h
systemctl --failed --no-pager || true
```

Run: `bash server-health.sh`. Expected: hostname, time, uptime, memory, disk, and failed service status.

**PowerShell health script — `Get-ServerHealth.ps1`:**

```powershell
$ErrorActionPreference = 'Stop'
[pscustomobject]@{
    Computer = $env:COMPUTERNAME
    Time = Get-Date
    OS = (Get-CimInstance Win32_OperatingSystem).Caption
    UptimeDays = [math]::Round(((Get-Date) - (Get-CimInstance Win32_OperatingSystem).LastBootUpTime).TotalDays, 2)
    FreeC_GB = [math]::Round((Get-PSDrive C).Free / 1GB, 2)
} | Format-List
```

Run in PowerShell: `./Get-ServerHealth.ps1` (or `./` with the correct path on Windows PowerShell). Expected: system inventory and capacity data. Prefer signed scripts and approved execution policies in managed environments; do not globally disable execution-policy protections to run untrusted code.

**Next step:** schedule scripts with systemd timers/cron or Windows Task Scheduler; write structured logs; handle missing data; alert on thresholds; store scripts in Git; review changes before deployment.

## 16. Troubleshooting playbook

Follow the same sequence every time: **define symptom → identify scope → collect evidence → form hypothesis → test one change → validate → document**.

| Symptom | First checks | Likely next action |
|---|---|---|
| VM will not boot | Proxmox task log, boot order, disk controller, firmware | Match UEFI/BIOS and storage driver; preserve original disk |
| Cannot SSH | `ip route`, `ss -lnt`, firewall, `Test-NetConnection` | Verify listener, IP and ACL |
| Client cannot join domain | Client DNS, time sync, AD SRV records, DC reachability | Fix DNS/time before retrying |
| Website unavailable | `systemctl status nginx`, `curl localhost`, firewall | Separate service failure from network failure |
| Slow VM | CPU ready/usage, memory pressure, storage latency | Identify actual bottleneck; avoid blind resource increases |
| No IP from DHCP | VLAN, DHCP scope, server authorization, relay | Ensure client and DHCP service can exchange broadcasts/relay |
| Disk full | `df -h`, snapshots, backup retention, logs | Safely remove approved data; expand capacity if needed |
| Backup fails | Target reachability, permissions, free space, job logs | Fix root cause and rerun/restore-test |

**Quick network ladder:**

```text
1. Is the NIC/link up?
2. Is the IP/mask correct?
3. Can the host reach another IP on its subnet?
4. Is a gateway configured and reachable?
5. Does DNS resolve the target?
6. Is the target TCP port reachable?
7. Is the service listening and authenticating?
8. Does the application actually work?
```

**Safe packet capture (authorized lab):** `sudo tcpdump -ni ens18 'port 53 or port 22' -c 30`. Captures may contain sensitive metadata; restrict access and retention.

## 17. Operations and change management

A server is not finished when installation completes. Maintain an inventory with owner, purpose, OS, IP, VLAN, software, license, backup job, recovery procedure, patch window, and lifecycle dates.

**Standard change record:** objective; current state; risk; approved maintenance window; backup verified; exact implementation steps; validation criteria; rollback procedure; change owner; and results.

**Daily:** inspect alerts and failed backups. **Weekly:** review disk growth and failed services. **Monthly:** patch after testing, audit privileged access, test restores on a rotating basis. **Quarterly:** review permissions, capacity, certificates, firmware advisories, and recovery plans.

**Security rule:** Never test disruptive changes against an employer or third-party network without authorization. Use the isolated lab for exercises.

## 18. Capstone projects

### Lab A — First Linux server (beginner)
1. Create LNX01 on Proxmox.
2. Install Ubuntu Server and SSH.
3. Assign the planned IP.
4. Install Nginx and publish the test page.
5. Connect from Windows over SSH.
6. Capture proof: `ip -br addr`, `systemctl is-active nginx`, browser screenshot.

**Pass criteria:** SSH and HTTP both work from the authorized client, and the VM restarts cleanly.

### Lab B — Windows domain (intermediate)
1. Create DC01 and a Windows client.
2. Configure DC01 static IP, DNS and AD DS.
3. Create an OU, user and security group.
4. Join the client to the domain.
5. Sign in as a lab user and run `gpresult /r`.
6. Document DNS and authentication troubleshooting steps.

**Pass criteria:** Domain logon succeeds and client resolves AD DNS records.

### Lab C — VLAN separation (advanced)
1. Create a second lab VLAN on a managed switch and Proxmox bridge.
2. Place a test VM in the second VLAN.
3. Confirm same-VLAN reachability.
4. Confirm inter-VLAN traffic is blocked until an approved router/firewall rule permits it.
5. Document allowed management traffic and denied traffic.

**Pass criteria:** Packet paths match the documented network policy.

### Lab D — Failure and recovery (advanced)
1. Back up LNX01 to storage outside its primary VM disk.
2. Record baseline service and file contents.
3. Simulate a reversible failure in the **test VM** (for example, stop Nginx).
4. Diagnose using systemd and logs.
5. Restore the service; then separately restore a VM backup to an isolated test VM ID.
6. Record measured RTO and achieved RPO.

**Pass criteria:** Services and test data are verified after recovery, without affecting the original VM.

### Lab E — Operations handover (expert foundations)
Produce a one-page runbook for each server containing: diagram, IP/VLAN, dependencies, patching steps, monitoring checks, backup location, restore steps, escalation contacts (lab placeholders), and rollback procedures.

**Pass criteria:** Another student can reproduce your setup and recovery using only your documentation.

## 19. Final skills checklist

- [ ] Explain physical servers, hypervisors, VMs, containers and service roles.
- [ ] Size CPU, RAM, storage, network and backup requirements.
- [ ] Install Proxmox and create Linux/Windows VMs.
- [ ] Configure IP, gateway, DNS, DHCP and VLANs safely.
- [ ] Deploy and validate SSH, web services and Active Directory.
- [ ] Explain NTFS, SMB and Linux filesystem permissions.
- [ ] Apply least privilege, updates, firewall rules and secure remote access.
- [ ] Investigate logs, service failures, network paths and storage alerts.
- [ ] Automate health checks with Bash and PowerShell.
- [ ] Create, verify and restore backups in an isolated test.
- [ ] Document a complete server build, change and rollback plan.

---

## Reference documentation

- [Proxmox VE Administration Guide](https://pve.proxmox.com/pve-docs/)
- [Ubuntu Server documentation](https://documentation.ubuntu.com/server/)
- [Microsoft Windows Server documentation](https://learn.microsoft.com/en-us/windows-server/)
- [Microsoft Active Directory Domain Services](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/)
- [OpenSSH documentation](https://www.openssh.com/manual.html)
- [Nginx documentation](https://nginx.org/en/docs/)

> **Version note:** UI names and commands can vary by release. Confirm product-specific instructions against the version installed in your lab. Example IP addresses and domain names are placeholders, not live endpoints.
