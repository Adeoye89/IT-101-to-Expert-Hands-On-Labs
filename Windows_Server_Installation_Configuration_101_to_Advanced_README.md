# Windows Server Installation & Configuration — IT 101 to Advanced

> Hands-on guide for Windows Server 2022/2025 in an isolated Proxmox VE cyber range. Practice only on systems you own or administer.

## 1. Learning outcomes

By the end you can deploy Windows Server, configure IP/DNS, install AD DS, join clients, manage users/groups/OUs, configure GPO, DHCP, SMB shares, firewall, updates, backup, and troubleshoot common failures.

## 2. Lab architecture

| System | Example IP | Role |
|---|---|---|
| pfSense | `10.10.20.1/24` | Target-lab gateway/firewall |
| DC01 | `10.10.20.10/24` | Windows Server domain controller, DNS, optional DHCP |
| PC01 | `10.10.20.20/24` | Windows 11 Pro client |
| FS01 (optional) | `10.10.20.30/24` | Dedicated file server |

- AD domain: `corp.example.test` (lab-only example; for production, use a subdomain of a domain you control).
- Proxmox bridge: `vmbr20`, **no physical bridge port**.
- Keep lab systems isolated from your production/home network. Use pfSense rules to restrict access. Never expose RDP, SMB, LDAP, or AD management ports directly to the internet.
- **DHCP warning:** use **either pfSense or Windows DHCP** for a given subnet, not both. This guide shows Windows DHCP as an optional exercise.

## 3. Prerequisites

- Proxmox VE 9.x and a Windows Server 2022/2025 ISO obtained from Microsoft.
- For Windows on Proxmox, also upload the current signed VirtIO driver ISO from the Fedora/Red Hat VirtIO project.
- Suggested VM: 2–4 vCPUs, 4–8 GB RAM, 80 GB system disk, one NIC on `vmbr20`.
- Windows Server license/evaluation terms apply. Use a valid license for production.
- Proxmox snapshot before role changes, plus a separate backup. **Snapshots are not backups.**

## 4. Create the Windows Server VM in Proxmox

1. Open Proxmox web UI (`https://<host-ip>:8006`) and select **Create VM**.
2. Name it `DC01`; select the Windows Server ISO.
3. Choose a supported machine/firmware configuration. For a modern Windows Server VM, `q35` and OVMF (UEFI) are reasonable choices; retain the EFI disk if prompted.
4. Add an 80 GB disk. `SCSI` with `VirtIO SCSI single` offers good performance, but Windows setup may need the VirtIO storage driver. A SATA disk can simplify a first lab install.
5. Allocate CPU/RAM and select `vmbr20` for the NIC. VirtIO NICs may need drivers; E1000 is a temporary fallback for installation.
6. Add the VirtIO ISO as a second CD/DVD drive under **Hardware**.
7. Start the VM and open **Console**.
8. If Windows setup does not see the VirtIO disk, click **Load driver**, browse the VirtIO ISO for the matching storage driver and OS architecture, then continue.
9. Install **Windows Server with Desktop Experience** for the GUI labs. Set a strong local Administrator password.
10. After login, install missing VirtIO drivers from the ISO; reboot and verify Device Manager has no unexpected missing devices.

**Verify:** Server Manager opens; `Get-ComputerInfo | Select-Object WindowsProductName,WindowsVersion` returns the OS details.

## 5. Rename the server

**GUI:** Server Manager → **Local Server** → click computer name → **Change** → `DC01` → restart.

**PowerShell (Administrator):**

```powershell
Rename-Computer -NewName 'DC01' -Restart
```

Verify after restart:

```powershell
hostname
```

## 6. Configure a static IP address

**GUI:** Server Manager → Local Server → Ethernet → adapter properties → Internet Protocol Version 4 (TCP/IPv4) → Properties.

Use:
- IP `10.10.20.10`
- Mask `255.255.255.0`
- Gateway `10.10.20.1`
- Preferred DNS: initially use an available trusted lab resolver during setup; **after promoting DC01 and installing DNS, configure DC01 to use its own AD DNS service** (`10.10.20.10` or loopback, according to your design).

PowerShell (identify the interface first):

```powershell
Get-NetAdapter
Get-NetIPConfiguration
# Replace 'Ethernet' with your actual interface alias.
New-NetIPAddress -InterfaceAlias 'Ethernet' -IPAddress '10.10.20.10' -PrefixLength 24 -DefaultGateway '10.10.20.1'
# Once AD DNS is installed and healthy:
Set-DnsClientServerAddress -InterfaceAlias 'Ethernet' -ServerAddresses '10.10.20.10'
```

If an old DHCP address exists, remove or adjust it before creating the static address; avoid duplicate IPs. Work from the Proxmox console because changing IP settings can disconnect RDP.

Verify:

```powershell
ipconfig /all
Test-Connection 10.10.20.1 -Count 2
```

## 7. Install Active Directory Domain Services and DNS

**GUI:** Server Manager → **Manage → Add Roles and Features** → Role-based installation → select DC01 → **Active Directory Domain Services** → Add Features → Install. Then click the flag notification → **Promote this server to a domain controller** → **Add a new forest** → `corp.example.test` → choose a strong DSRM password → review DNS/options → Install → reboot.

**PowerShell alternative:**

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Install-ADDSForest -DomainName 'corp.example.test' -DomainNetbiosName 'CORP' -InstallDNS
```

The wizard prompts for DSRM password and confirmation; promotion restarts the machine. Run this only on a new lab server intended to become a DC.

Verify after reboot:

```powershell
Get-ADDomain
Get-ADForest
Get-Service NTDS,DNS
Resolve-DnsName -Type SRV _ldap._tcp.dc._msdcs.corp.example.test
```

**Important:** Clients must use the AD DNS server (`10.10.20.10`) to reliably locate the domain controller. Configure DNS forwarders in DNS Manager if controlled external resolution is needed.

## 8. Create organizational units, users, and groups

**GUI:** Server Manager → Tools → **Active Directory Users and Computers** → right-click domain → New → Organizational Unit. Create `LabUsers`, `LabComputers`, and `LabGroups`.

**PowerShell:**

```powershell
Import-Module ActiveDirectory
$base = 'DC=corp,DC=example,DC=test'
New-ADOrganizationalUnit -Name 'LabUsers' -Path $base
New-ADOrganizationalUnit -Name 'LabComputers' -Path $base
New-ADOrganizationalUnit -Name 'LabGroups' -Path $base
$pw = Read-Host 'Temporary password for lab user' -AsSecureString
New-ADUser -Name 'Alex Student' -SamAccountName 'astudent' -UserPrincipalName 'astudent@corp.example.test' -Path "OU=LabUsers,$base" -AccountPassword $pw -Enabled $true -ChangePasswordAtLogon $true
New-ADGroup -Name 'LabHelpdesk' -GroupScope Global -GroupCategory Security -Path "OU=LabGroups,$base"
Add-ADGroupMember -Identity 'LabHelpdesk' -Members 'astudent'
Get-ADUser astudent
Get-ADGroupMember LabHelpdesk
```

Avoid using Domain Admin accounts for ordinary work. Use separate admin and standard user identities.

## 9. Join Windows 11 Pro to the domain

1. Attach PC01 to `vmbr20`.
2. Set PC01 IP `10.10.20.20/24`, gateway `10.10.20.1`, **DNS `10.10.20.10`**.
3. Confirm `nslookup corp.example.test` and `Test-NetConnection 10.10.20.10 -Port 53` work.
4. Windows 11 Pro/Enterprise: **Settings → System → About → Domain or workgroup** (or `sysdm.cpl` → Computer Name → Change) → Domain → `corp.example.test`.
5. Supply authorized domain credentials and restart.
6. Sign in as `CORP\astudent` (after password change if required).

PowerShell alternative from an elevated prompt on PC01:

```powershell
Add-Computer -DomainName 'corp.example.test' -Credential 'CORP\Administrator' -Restart
```

Check:

```powershell
whoami
nltest /dsgetdc:corp.example.test
gpresult /r
```

## 10. Group Policy basics

1. Open **Server Manager → Tools → Group Policy Management**.
2. Right-click `LabComputers` OU → **Create a GPO in this domain, and Link it here**.
3. Name it `LAB - Workstation Baseline`.
4. Edit a low-risk setting, such as a workstation-specific screen lock policy, then close the editor.
5. Move the PC01 computer account into `LabComputers` OU using AD Users and Computers.
6. On PC01 run:

```powershell
gpupdate /force
gpresult /h C:\Windows\Temp\gpo-report.html
```

Open the HTML report and verify the policy applies. Do not edit Default Domain Policy for routine lab experiments.

## 11. Optional Windows DHCP role

**Only do this if pfSense DHCP is disabled on `10.10.20.0/24`.** A duplicate DHCP service can break the lab.

**GUI:** Server Manager → Add Roles and Features → DHCP Server → Install → complete post-install configuration/authorization → Tools → DHCP → IPv4 → New Scope.

Example scope:
- Range `10.10.20.100`–`10.10.20.199`
- Subnet mask `255.255.255.0`
- Router option 003 `10.10.20.1`
- DNS option 006 `10.10.20.10`
- DNS domain option 015 `corp.example.test`

PowerShell (run on DC01):

```powershell
Install-WindowsFeature DHCP -IncludeManagementTools
Add-DhcpServerInDC -DnsName 'DC01.corp.example.test' -IPAddress '10.10.20.10'
Add-DhcpServerv4Scope -Name 'LabTargetSubnet' -StartRange 10.10.20.100 -EndRange 10.10.20.199 -SubnetMask 255.255.255.0
Set-DhcpServerv4OptionValue -ScopeId 10.10.20.0 -Router 10.10.20.1 -DnsServer 10.10.20.10 -DnsDomain 'corp.example.test'
Get-DhcpServerv4Scope
```

Do not use the example scope if those addresses are already assigned. Confirm leases from a test client with `ipconfig /renew` and `ipconfig /all`.

## 12. Create an SMB file share

For production, prefer a **separate member server FS01** instead of putting file services on a domain controller.

On FS01 (or DC01 for a limited lab exercise), create a folder and share:

```powershell
New-Item -ItemType Directory -Path 'C:\LabShare' -Force
New-SmbShare -Name 'LabShare' -Path 'C:\LabShare' -ChangeAccess 'CORP\LabHelpdesk' -ReadAccess 'CORP\Domain Users'
Get-SmbShare -Name LabShare
```

**Critical:** SMB share permissions and NTFS permissions both apply. Configure NTFS ACLs using **Properties → Security** to grant only the intended groups; do not assume the share command alone grants filesystem access.

From PC01, open `\\FS01\LabShare` (or `\\DC01\LabShare` in the single-server lab). Test read/write as an authorized user and verify an unauthorized user is denied.

## 13. Windows Firewall and remote administration

- Keep Windows Defender Firewall enabled for Domain, Private, and Public profiles.
- Allow only required traffic from approved lab subnets; do not broadly open all ports.
- If enabling RDP for lab administration: **Settings → System → Remote Desktop** (on supported editions) and restrict access through pfSense and Windows Firewall. Prefer an admin jump host or VPN for remote access.
- Confirm services:

```powershell
Get-NetFirewallProfile
Get-Service WinRM,TermService
Test-NetConnection 10.10.20.10 -Port 53
```

Do not expose TCP 3389 or 445 to the internet.

## 14. Updates, security, and time synchronization

1. Apply approved Windows Updates and reboot during a maintenance window.
2. Confirm the domain controller and clients have correct time; Kerberos is time-sensitive.
3. Enable Microsoft Defender, auditing, and least-privilege administration.
4. Use strong unique passwords, MFA where supported, and restrict privileged logons.
5. Keep domain controller roles separate from general-purpose application workloads in production.

```powershell
w32tm /query /status
Get-MpComputerStatus | Select-Object AMServiceEnabled,AntivirusEnabled,RealTimeProtectionEnabled
Get-WinEvent -LogName System -MaxEvents 10
```

## 15. Backups and recovery

- Use Proxmox backups for the VM and **Windows-aware system-state backups** for AD recovery planning.
- For a domain controller, avoid treating arbitrary VM snapshot rollback as a substitute for supported AD restore procedures.
- Document the DSRM password in a secure password vault.
- Test restore on an isolated recovery network; verify DNS, AD DS, authentication, and GPO afterward.
- If using Windows Server Backup, install its feature:

```powershell
Install-WindowsFeature Windows-Server-Backup
```

Configure a supported backup destination and recovery plan through Windows Server Backup; never overwrite the only backup during tests.

## 16. Troubleshooting reference

| Symptom | Likely cause | Checks |
|---|---|---|
| Client cannot join domain | Wrong DNS, unreachable DC, time mismatch | `ipconfig /all`, `nslookup`, `nltest`, `w32tm` |
| `Access denied` to share | Share/NTFS ACL mismatch | `Get-SmbShareAccess`, folder Security tab |
| No DHCP lease | DHCP disabled, scope exhausted, competing DHCP | DHCP console, `ipconfig /all`, pfSense DHCP settings |
| GPO not applied | Wrong OU, filtering, replication, DNS | `gpresult /h`, `gpupdate /force` |
| DNS resolution fails | Wrong resolver or missing records | `Resolve-DnsName`, DNS Manager |
| VM cannot reach gateway | Wrong bridge, IP, pfSense rule | Proxmox VM Hardware, `ipconfig`, pfSense logs |
| RDP unavailable | Firewall, service, routing, policy | `Test-NetConnection -Port 3389`, service status |

Useful DC diagnostics:

```powershell
dcdiag /v
repadmin /replsummary
Get-ADDomainController -Filter *
Get-DnsServerZone
```

`repadmin` replication results are most useful with multiple DCs; a single-DC lab has no peer replication to evaluate.

## 17. Hands-on labs

1. **Install:** Deploy DC01 on isolated `vmbr20`, install VirtIO drivers, record a baseline snapshot.
2. **Network:** Assign static IP and verify gateway/DNS connectivity.
3. **Directory:** Promote DC01, create two users, two OUs, and a security group.
4. **Client:** Join PC01, log in as a domain user, and verify DC discovery.
5. **Policy:** Create an OU-linked GPO and generate a `gpresult` report.
6. **DHCP:** Enable one DHCP service only; confirm PC01 receives correct gateway and AD DNS.
7. **Files:** Configure a least-privilege SMB share and test access from PC01.
8. **Incident:** Break PC01 DNS deliberately, diagnose the domain join failure, then restore DNS.
9. **Recovery:** Run a backup, document restore steps, and test in an isolated copy.

## 18. Final acceptance checklist

- [ ] DC01 boots with correct VirtIO drivers and time.
- [ ] Static IP, subnet mask, gateway, and AD DNS are correct.
- [ ] AD DS and DNS are healthy.
- [ ] PC01 joins the domain and can authenticate.
- [ ] Users, groups, OUs, and GPO function as expected.
- [ ] Exactly one DHCP server serves the target subnet.
- [ ] SMB access follows least privilege.
- [ ] Lab is isolated from production and the public internet.
- [ ] Backup and restore procedures are documented and tested.

## 19. Official references

- [Microsoft Learn — Windows Server](https://learn.microsoft.com/en-us/windows-server/)
- [Microsoft Learn — Active Directory Domain Services](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/)
- [Microsoft Learn — DNS in Windows Server](https://learn.microsoft.com/en-us/windows-server/networking/dns/)
- [Microsoft Learn — DHCP in Windows Server](https://learn.microsoft.com/en-us/windows-server/networking/technologies/dhcp/)
- [Proxmox VE documentation](https://pve.proxmox.com/pve-docs/)

**Suggested GitHub path:** `servers/windows-server/README.md`
