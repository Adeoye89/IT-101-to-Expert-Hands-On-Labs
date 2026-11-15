# IT 101 — Network Switches & Configuration (Novice → Advanced)

> Hands-on GitHub README | Cisco IOS-style managed-switch labs, vendor-neutral concepts, Proxmox integration, verification and troubleshooting.
>
> **Lab only:** Examples use an isolated practice switch or simulator (Cisco Packet Tracer, GNS3, EVE-NG, or a spare managed switch). Cisco commands differ across models/IOS releases. Never paste a full configuration into a production switch without confirming interface names, management access, change approval and rollback.

## Table of contents
1. [What a switch does](#1-what-a-switch-does)
2. [Types of switches](#2-types-of-switches)
3. [Equipment and lab topology](#3-equipment-and-lab-topology)
4. [First login and command modes](#4-first-login-and-command-modes)
5. [Baseline switch setup](#5-baseline-switch-setup)
6. [Inspect ports, MAC addresses and links](#6-inspect-ports-mac-addresses-and-links)
7. [Create VLANs and access ports](#7-create-vlans-and-access-ports)
8. [Configure a trunk between switches](#8-configure-a-trunk-between-switches)
9. [Configure management IP and SSH](#9-configure-management-ip-and-ssh)
10. [Connect Proxmox to VLANs](#10-connect-proxmox-to-vlans)
11. [Layer 3 routing and DHCP concepts](#11-layer-3-routing-and-dhcp-concepts)
12. [Spanning Tree and edge-port protection](#12-spanning-tree-and-edge-port-protection)
13. [Port security and unused ports](#13-port-security-and-unused-ports)
14. [Backup, restore and safe changes](#14-backup-restore-and-safe-changes)
15. [Troubleshooting playbook](#15-troubleshooting-playbook)
16. [Exercises and assessment](#16-exercises-and-assessment)
17. [Command reference](#17-command-reference)

---

## 1. What a switch does

A **network switch** connects devices on a local area network (LAN). A typical Ethernet switch learns which **MAC address** appears on which port, then forwards frames to the appropriate port. A router moves packets **between IP networks**; a Layer 2 switch primarily forwards frames **within a VLAN**.

- **MAC address:** Layer 2 hardware identifier, e.g. `02:11:22:33:44:55`.
- **IP address:** Layer 3 logical address, e.g. `192.168.10.21`.
- **Port/interface:** Physical or logical switch connection, e.g. `GigabitEthernet0/1`.
- **VLAN:** Separate logical Layer 2 broadcast domain on shared switching hardware.
- **Access port:** Carries one access VLAN, usually untagged to an endpoint.
- **Trunk port:** Carries multiple VLANs, usually with IEEE 802.1Q tags.
- **SVI:** Switch virtual interface such as `interface vlan 99`; often used for management or inter-VLAN routing on a Layer 3 switch.

**Example:** A PC on VLAN 10 and a server on VLAN 20 normally need a router or Layer 3 switch with permitted routing/firewall rules to communicate.

## 2. Types of switches

| Type | Purpose | Example use |
|---|---|---|
| Unmanaged | Plug-and-play; little/no configuration | Small home LAN |
| Smart / web-managed | VLANs, basic monitoring via web UI | Small business |
| Managed Layer 2 | VLANs, trunks, STP, security, CLI | Office access switch |
| Layer 3 | Layer 2 plus IP routing | Distribution/core |
| PoE | Delivers power over Ethernet | IP phones, cameras, Wi-Fi APs |
| Stackable | Several switches managed as a logical unit | Larger access layer |
| Virtual switch | Software switching inside a hypervisor | Proxmox Linux bridge |

**Important distinction:** PoE and stackable describe capabilities; they are not mutually exclusive with Layer 2 or Layer 3.

## 3. Equipment and lab topology

Use Packet Tracer or two spare Cisco IOS-style switches (SW1 and SW2), a router or firewall (optional), two PCs, and optionally Proxmox. Sample addressing is documentation-only and must be adjusted to your environment.

```text
 PC-A (VLAN 10) --- Gi0/2  SW1  Gi0/1 ===== Gi0/1  SW2  Gi0/2 --- PC-B (VLAN 10)
                            |    802.1Q trunk
                         Gi0/3
                            |
                     Proxmox host (optional)

VLAN 10  USERS       192.168.10.0/24
VLAN 20  SERVERS     192.168.20.0/24
VLAN 99  MANAGEMENT  192.168.99.0/24
SW1 management: 192.168.99.11/24
SW2 management: 192.168.99.12/24
Router/gateway on VLAN 99 (if provided): 192.168.99.1
```

**Preflight:** Record actual model, IOS version, port names, existing config, VLAN list, uplinks, management address, gateway, and console access. Do not use a production uplink for initial trunk practice.

## 4. First login and command modes

1. Attach a console cable or open the switch's console in Packet Tracer.
2. If physical, use an appropriate terminal application and the device's documented serial settings (often 9600 baud, 8 data bits, no parity, 1 stop bit).
3. Press Enter to obtain a prompt.
4. Learn the modes:

```text
Switch>                 User EXEC
Switch> enable
Switch#                 Privileged EXEC
Switch# configure terminal
Switch(config)#          Global configuration
Switch(config)# interface gigabitEthernet0/2
Switch(config-if)#       Interface configuration
Switch(config-if)# end
Switch#                 Back to privileged EXEC
```

**Check before changing anything:**

```cisco
show version
show running-config
show interfaces status
show vlan brief
```

Expected: switch model/OS information, current configuration, port state and VLAN membership. Some commands vary by platform.

## 5. Baseline switch setup

**Goal:** Set a hostname and protect privileged access on a lab switch.

```cisco
enable
configure terminal
hostname SW1
no ip domain-lookup
enable secret YOUR_STRONG_LAB_SECRET
service password-encryption
banner motd #Authorized lab access only#
end
copy running-config startup-config
```

**What these mean:** `hostname` labels the device; `no ip domain-lookup` prevents mistyped commands triggering DNS lookups; `enable secret` protects privileged mode; `service password-encryption` only obfuscates some stored legacy passwords, **not** a substitute for modern password hashing or SSH. `copy running-config startup-config` saves changes across reboot.

**Verify:**

```cisco
show running-config | include hostname
show startup-config
```

**Caution:** Do not configure remote login until you have a working management network and console fallback. Never store real passwords in GitHub.

## 6. Inspect ports, MAC addresses and links

```cisco
show interfaces status
show interfaces gigabitEthernet0/2
show mac address-table
show mac address-table dynamic
show interfaces counters errors
```

**Exercise:** Connect PC-A to Gi0/2, send a ping to a reachable peer in the same VLAN, and check whether the PC's MAC appears on Gi0/2. MAC learning requires frames from that endpoint; an empty table can be normal on an idle port.

**Interpretation:** `connected` generally means physical link is up; an `err-disabled` interface requires investigation (e.g., BPDU guard or port security). CRC/input errors may indicate cabling, optics, or duplex problems.

## 7. Create VLANs and access ports

**Goal:** Put PC-A on VLAN 10 and a lab server on VLAN 20.

```cisco
configure terminal
vlan 10
 name USERS
vlan 20
 name SERVERS
vlan 99
 name MANAGEMENT
exit
interface gigabitEthernet0/2
 description PC-A_USERS
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
exit
interface gigabitEthernet0/4
 description LAB_SERVER
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
end
copy running-config startup-config
```

`spanning-tree portfast` is for ports connected to **end devices**, not switch-to-switch links. Check vendor guidance before using it on a hypervisor carrying bridged VMs.

**Verify:**

```cisco
show vlan brief
show interfaces gigabitEthernet0/2 switchport
show interfaces gigabitEthernet0/4 switchport
```

**Expected:** Gi0/2 in access VLAN 10, Gi0/4 in access VLAN 20. Devices in different VLANs will not communicate until Layer 3 routing is configured.

## 8. Configure a trunk between switches

**Goal:** Carry VLANs 10, 20 and 99 over the SW1–SW2 link.

**On SW1, after confirming Gi0/1 is the lab inter-switch link:**

```cisco
configure terminal
interface gigabitEthernet0/1
 description TRUNK_TO_SW2
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
end
```

**On SW2:**

```cisco
configure terminal
vlan 10
 name USERS
vlan 20
 name SERVERS
vlan 99
 name MANAGEMENT
exit
interface gigabitEthernet0/1
 description TRUNK_TO_SW1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
end
```

Some older platforms require `switchport trunk encapsulation dot1q` before `switchport mode trunk`; many modern models support only 802.1Q and do not have that command.

**Verify both ends:**

```cisco
show interfaces trunk
show interfaces gigabitEthernet0/1 switchport
show spanning-tree vlan 10
```

**Expected:** Gi0/1 trunking and VLAN 10/20/99 allowed and active. Configure PC-B's SW2 port as access VLAN 10; give PC-A and PC-B compatible addresses in `192.168.10.0/24` and test ping. If they cannot communicate, check trunk allowed VLANs, STP state, host firewall and endpoint IPs.

**Native VLAN note:** Avoid using VLAN 1 for management where feasible; ensure native VLAN settings match on both ends. Use an unused native VLAN when supported by your network design and security policy. Do not casually change a production native VLAN.

## 9. Configure management IP and SSH

**Prerequisite:** VLAN 99 must be carried to the administrator's network; the management SVI must be up (typically requires at least one active forwarding port in the VLAN), and routing must exist if connecting from another subnet.

**SW1 example:**

```cisco
configure terminal
interface vlan 99
 description MANAGEMENT_SVI
 ip address 192.168.99.11 255.255.255.0
 no shutdown
exit
ip default-gateway 192.168.99.1
ip domain-name lab.example.test
username labadmin privilege 15 secret YOUR_UNIQUE_STRONG_PASSWORD
crypto key generate rsa modulus 2048
ip ssh version 2
line vty 0 4
 login local
 transport input ssh
end
copy running-config startup-config
```

For a **Layer 2 switch**, `ip default-gateway` specifies the next hop for management traffic outside the local subnet. A **Layer 3 switch with `ip routing` enabled** normally uses a default route instead. RSA key generation syntax and SSH algorithms vary by device/version; prefer supported modern algorithms and keys.

**Verify on switch:**

```cisco
show ip interface brief
show ip ssh
show running-config | section line vty
```

**From a management workstation on the reachable VLAN:**

```bash
ssh labadmin@192.168.99.11
```

Verify the SSH host-key fingerprint through your trusted inventory or console before accepting it. If unreachable, test physical path, VLAN tagging, SVI status, gateway and access controls. Restrict SSH management access using a management ACL or firewall in real deployments.

## 10. Connect Proxmox to VLANs

**Option A — Single VLAN access port:** Use when the Proxmox NIC only needs one untagged VLAN. Assign the switch port to that access VLAN. A Proxmox Linux bridge connected to the NIC can carry the untagged network.

**Option B — VLAN-aware trunk:** Use when Proxmox VMs need multiple VLANs. On a dedicated lab switch port (example Gi0/3):

```cisco
configure terminal
interface gigabitEthernet0/3
 description PROXMOX_LAB_UPLINK
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
end
```

**Proxmox `/etc/network/interfaces` conceptual example** (adapt the NIC name, subnet and uplink; changing management networking can disconnect you):

```ini
auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual

auto vmbr0
iface vmbr0 inet manual
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 10 20 99

auto vmbr0.99
iface vmbr0.99 inet static
    address 192.168.99.20/24
    gateway 192.168.99.1
```

**Critical:** This is an **illustration**, not a drop-in replacement for your existing Proxmox network file. Verify interface naming, gateway uniqueness, bridge/VLAN behavior and console access before applying. Do not replace a remote host's working management network without an out-of-band recovery path.

**VM configuration:** In Proxmox → VM → Hardware → Network Device → Bridge `vmbr0` → VLAN Tag `10` (or `20`). Proxmox then tags that VM's traffic on the VLAN-aware bridge. Keep guest NIC configuration untagged unless deliberately using guest-side VLAN interfaces.

**Check on Proxmox:**

```bash
ip -br link
ip -br addr
bridge vlan show
ip route
```

**Check on switch:** `show interfaces trunk` and `show mac address-table dynamic`.

## 11. Layer 3 routing and DHCP concepts

A VLAN alone does not provide DHCP or internet access. Each VLAN needs an appropriate **gateway** and, if using automatic addressing, a **DHCP service** or DHCP relay. Common designs:

- **Router-on-a-stick:** A router/firewall interface has one 802.1Q subinterface per VLAN.
- **Layer 3 switch:** SVIs with `ip routing` enabled provide inter-VLAN routing.
- **Firewall routing:** Firewall interfaces/VLANs enforce security policies between networks.

**Layer 3 switch lab only (platform must support routing):**

```cisco
configure terminal
ip routing
interface vlan 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown
interface vlan 20
 ip address 192.168.20.1 255.255.255.0
 no shutdown
end
show ip route
```

**Security note:** Enabling routing can connect networks that were previously isolated. Apply ACLs/firewall rules appropriate to your lab; do not enable this in production without a reviewed policy. If a DHCP server is on another subnet, a supported L3 gateway may need `ip helper-address <DHCP-server-IP>` on the client VLAN SVI.

## 12. Spanning Tree and edge-port protection

**STP/RSTP** prevents switching loops that can cause broadcast storms. Never disable STP casually.

```cisco
show spanning-tree
show spanning-tree vlan 10
```

For a **true endpoint port** only, protect against an accidental switch connection:

```cisco
configure terminal
interface gigabitEthernet0/2
 spanning-tree portfast
 spanning-tree bpduguard enable
end
```

If the port receives a BPDU, BPDU Guard may shut it down. Investigate and remove the cause before recovery. Do **not** apply edge-port assumptions to inter-switch trunks.

## 13. Port security and unused ports

Some Cisco access switches support limiting learned MAC addresses on an endpoint port:

```cisco
configure terminal
interface gigabitEthernet0/2
 switchport mode access
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
end
show port-security interface gigabitEthernet0/2
```

**Caution:** MAC limits may break phones with PCs behind them, hypervisor hosts, and multi-MAC devices. Avoid port-security MAC limits on VM trunks without careful planning.

**Unused port example:**

```cisco
configure terminal
interface gigabitEthernet0/24
 description UNUSED_DISABLED
 shutdown
end
```

Do not disable a port until you confirm it is unused and not an uplink or management path.

## 14. Backup, restore and safe changes

Before a change:

```cisco
show running-config
show startup-config
show interfaces status
show vlan brief
show interfaces trunk
```

Capture the output in your change record, **redacting secrets**. In a lab, save a known-good configuration:

```cisco
copy running-config startup-config
```

For a safe lab rollback of **unsaved** changes, rebooting may restore startup config, but it causes downtime and can affect management. For a targeted change, explicitly reverse the affected commands instead. Do not use `write erase` or `reload` as routine troubleshooting on production equipment.

**Common artifacts to keep:** device model/serial (private inventory), OS version, sanitized `show running-config`, port map, VLAN table, IP plan, cable diagram, change log, backup location, validation screenshots and rollback steps.

## 15. Troubleshooting playbook

| Symptom | Checks | Typical cause |
|---|---|---|
| Port down | `show interfaces status`; cable/optics | Bad cable, disabled port, peer off |
| VLAN host cannot reach same-VLAN peer | `show vlan brief`; `show interfaces trunk`; host IP | Wrong access VLAN or missing trunk VLAN |
| VLANs cannot reach each other | `show ip route`; gateway; ACLs | No L3 gateway or routing policy |
| SSH timeout | SVI up? VLAN 99 path? route? ACL? | Unreachable management subnet |
| SSH refused | `show ip ssh`; VTY config; keys | SSH not enabled/configured |
| DHCP failure | DHCP server, relay, VLAN, scope | No DHCP service or helper |
| Proxmox VM isolated | VM VLAN tag; bridge VLAN-aware; switch trunk | Tag mismatch or missing VLAN |
| Port err-disabled | Logs, `show interfaces status` | BPDU guard or security violation |
| Slow/intermittent traffic | Counters, duplex, loops, STP | Physical errors or L2 instability |

**Troubleshoot from lowest layer upward:** (1) cable/link, (2) VLAN and MAC learning, (3) IP/subnet/gateway, (4) routing/ACL, (5) DNS/app.

**Host tests:**

Windows PowerShell:
```powershell
ipconfig /all
ping 192.168.10.1
Test-NetConnection 192.168.99.11 -Port 22
```

Linux:
```bash
ip -br addr
ip route
ping -c 4 192.168.10.1
ssh -v labadmin@192.168.99.11
```

## 16. Exercises and assessment

### Lab A — First switch (novice)
1. Open console; run `show version`, `show interfaces status` and `show vlan brief`.
2. Set hostname `SW1`; save config.
3. Connect two PCs in the same default VLAN; give compatible static IPs.
4. Ping between PCs; inspect the MAC table.

**Pass:** Both PCs communicate and MAC entries appear on expected ports.

### Lab B — VLAN isolation (beginner)
1. Create VLANs 10 and 20.
2. Assign PC-A to VLAN 10 and PC-C to VLAN 20.
3. Give PCs IPs in separate /24 subnets.
4. Confirm they cannot communicate without a router/L3 gateway.

**Pass:** VLAN membership is correct and L2 isolation is observed.

### Lab C — Trunk (intermediate)
1. Connect SW1 Gi0/1 to SW2 Gi0/1.
2. Configure both ends as trunks allowing 10,20,99.
3. Place PC-B on SW2 access VLAN 10.
4. Ping PC-A on SW1 VLAN 10 from PC-B.

**Pass:** Same-VLAN traffic crosses trunk; `show interfaces trunk` is correct.

### Lab D — SSH management (intermediate)
1. Configure management VLAN 99 and SW1 SVI `192.168.99.11/24`.
2. Create local SSH user and RSA keys.
3. Ensure the admin PC can reach VLAN 99.
4. SSH to SW1; confirm Telnet is disabled on VTY lines.

**Pass:** SSH works; Telnet remote login is not allowed.

### Lab E — Proxmox integration (advanced)
1. Configure a dedicated lab trunk to Proxmox.
2. Configure a VLAN-aware Linux bridge with console recovery access.
3. Attach an Ubuntu VM to VLAN 10 and a server VM to VLAN 20.
4. Verify tags, switch MAC learning, gateway access and intended isolation.

**Pass:** Each VM reaches its own VLAN gateway; cross-VLAN traffic follows your explicit routing/firewall policy.

### Completion checklist
- [ ] Explain MAC vs IP, switch vs router and access vs trunk.
- [ ] Identify port status and learned MAC addresses.
- [ ] Create and verify VLANs.
- [ ] Configure and validate a trunk.
- [ ] Configure a management SVI and SSH.
- [ ] Explain how Proxmox VLAN tagging works.
- [ ] Identify and resolve a deliberate VLAN mismatch.
- [ ] Back up a sanitized configuration and document rollback.

## 17. Command reference

| Command | Purpose |
|---|---|
| `enable` | Privileged EXEC mode |
| `configure terminal` | Enter configuration mode |
| `show running-config` | View active configuration |
| `show startup-config` | View saved startup configuration |
| `show version` | Device/software details |
| `show interfaces status` | Port state and speed |
| `show mac address-table` | Learned MAC-to-port mappings |
| `show vlan brief` | VLANs and access membership |
| `show interfaces trunk` | Trunk ports and allowed VLANs |
| `show ip interface brief` | Interface IP/status summary |
| `show spanning-tree` | STP state/topology |
| `show logging` | Device log messages |
| `show port-security` | Port-security status (if supported) |
| `copy running-config startup-config` | Save configuration |

### Suggested GitHub location

`networking/switches/README.md`

### Official references
- [Cisco Networking documentation](https://www.cisco.com/c/en/us/support/switches/index.html)
- [Proxmox VE documentation](https://pve.proxmox.com/pve-docs/)
- [IEEE 802.1Q overview (IEEE standards)](https://standards.ieee.org/)

> **Next lesson:** Routers, inter-VLAN routing, ACLs, NAT and firewall policies. The switch lab becomes the foundation for your Windows Server, Kali and Proxmox network.
