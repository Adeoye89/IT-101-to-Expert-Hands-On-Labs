# Network Troubleshooting 101 → Advanced

> **GitHub training module:** `networking/network-troubleshooting/README.md`  
> **Lab platforms:** Windows 10/11, Windows Server, Ubuntu, Kali, Proxmox VE 9.x, Cisco IOS-style switches  
> **Goal:** Find the failing network layer, collect evidence, fix the root cause, and verify service restoration.

## Contents

1. [Safety and lab topology](#1-safety-and-lab-topology)
2. [The troubleshooting method](#2-the-troubleshooting-method)
3. [Essential tools and command reference](#3-essential-tools-and-command-reference)
4. [Physical and link-layer troubleshooting](#4-physical-and-link-layer-troubleshooting)
5. [IP addressing and DHCP](#5-ip-addressing-and-dhcp)
6. [Gateway, routing, and NAT](#6-gateway-routing-and-nat)
7. [DNS troubleshooting](#7-dns-troubleshooting)
8. [Ports, firewalls, and SSH](#8-ports-firewalls-and-ssh)
9. [Switches, VLANs, and trunks](#9-switches-vlans-and-trunks)
10. [Proxmox virtual networking](#10-proxmox-virtual-networking)
11. [Performance, MTU, and packet loss](#11-performance-mtu-and-packet-loss)
12. [Packet capture with tcpdump and Wireshark](#12-packet-capture-with-tcpdump-and-wireshark)
13. [Windows and Linux troubleshooting scripts](#13-windows-and-linux-troubleshooting-scripts)
14. [Ten hands-on labs](#14-ten-hands-on-labs)
15. [Symptom-to-cause quick reference](#15-symptom-to-cause-quick-reference)
16. [Incident documentation and final checklist](#16-incident-documentation-and-final-checklist)

---

## 1. Safety and lab topology

Practice on equipment you own or administer with permission. Avoid scanning unknown networks. Before changing remote firewalls, VLANs, bridges, or IP addresses, ensure console access and a rollback plan; a mistake can disconnect your management session.

Example isolated training network:

```text
                       [Internet]
                           |
                    [Router / Firewall]
                     192.168.10.1/24
                           |
                  [Managed switch trunk]
                           |
                  [Proxmox host: vmbr0]
                           |
              +------------+------------+
              |                         |
          VLAN 10                   VLAN 20
       192.168.10.0/24           192.168.20.0/24
       Windows client            Ubuntu server
       192.168.10.50             192.168.20.20
```

**Assumptions:** The router has a configured interface and policy for each VLAN if inter-VLAN traffic is expected. A switch trunk carries tagged VLANs to Proxmox; access ports carry one untagged VLAN. The IPs above are examples, not addresses you should copy blindly.

Record: source IP, destination IP/hostname, expected port/protocol, VLAN, gateway, DNS server, timestamp, recent changes, and whether the failure affects one host or many.

## 2. The troubleshooting method

1. **Define the symptom:** What fails? For whom? Since when? Is it intermittent?
2. **Establish scope:** One application, one VM, one VLAN, or the entire site?
3. **Check from near to far:** Link → local address → same-subnet neighbor → gateway → routed destination → DNS → service.
4. **Test a hypothesis:** Change only one variable at a time and capture before/after output.
5. **Fix the cause:** Not merely a symptom. Document the exact change.
6. **Verify:** Test the original application, not just ping; confirm other services still work.
7. **Prevent recurrence:** Update diagrams, alerts, configuration backups, or procedures.

### OSI-inspired diagnostic map

| Layer | Typical failure | First evidence |
|---|---|---|
| 1 Physical | Bad cable, unplugged NIC, no carrier | Link LEDs, `ethtool`, interface status |
| 2 Data link | Wrong VLAN, MAC learning, STP | MAC table, VLAN/trunk status, ARP/neighbor |
| 3 Network | Wrong subnet, gateway, routing | `ipconfig`, `ip route`, `tracert` |
| 4 Transport | Port blocked, no listener | `Test-NetConnection`, `ss`, `nc` |
| 7 Application | DNS, TLS, web service, SSH auth | `nslookup`, `curl`, application logs |

**Remember:** Ping uses ICMP. A blocked ICMP echo does not prove the destination service is unavailable.

## 3. Essential tools and command reference

| Question | Windows PowerShell / CMD | Linux / Proxmox |
|---|---|---|
| What IP and DNS do I have? | `ipconfig /all` or `Get-NetIPConfiguration` | `ip -br addr`, `resolvectl status` |
| Is the interface up? | `Get-NetAdapter` | `ip -br link` |
| What route will traffic use? | `route print` or `Get-NetRoute` | `ip route get 1.1.1.1` |
| Can I reach the gateway? | `ping 192.168.10.1` | `ping -c 4 192.168.10.1` |
| Where does traffic stop? | `tracert 1.1.1.1` | `tracepath 1.1.1.1` or `traceroute` |
| Does DNS resolve? | `Resolve-DnsName example.com` | `dig example.com` |
| Is TCP 22 reachable? | `Test-NetConnection HOST -Port 22` | `nc -vz HOST 22` |
| What ports are listening? | `Get-NetTCPConnection -State Listen` | `sudo ss -lntup` |
| What neighbors are known? | `arp -a` | `ip neigh` |
| Capture packets | Wireshark or `pktmon` | `sudo tcpdump -ni INTERFACE` |

Install optional Ubuntu utilities with:

```bash
sudo apt update
sudo apt install -y dnsutils traceroute iputils-tracepath tcpdump ethtool netcat-openbsd
```

Commands vary by distribution and installed package. Run read-only diagnostics first.

## 4. Physical and link-layer troubleshooting

### Windows

```powershell
Get-NetAdapter | Format-Table Name,Status,LinkSpeed,MacAddress
Get-NetAdapterStatistics -Name "Ethernet"
```

Expected: `Status` is `Up`, and the negotiated link speed matches the equipment. `Disconnected` means no active link; investigate cable, NIC, switch port, or virtual adapter.

### Linux

```bash
ip -br link
sudo ethtool eno1
sudo ethtool -S eno1 | head -40
```

Replace `eno1` with the actual interface. Look for `Link detected: yes`, negotiated speed, increasing drops/errors, and duplex mismatches. Not every virtual NIC exposes meaningful `ethtool` data.

### Cisco IOS-style switch

```text
show interfaces status
show interfaces gigabitEthernet0/1
show interfaces counters errors
show mac address-table interface gigabitEthernet0/1
show logging
```

**Interpretation:** A down port can indicate cable or shutdown; CRC/input errors can suggest media problems; no learned MAC can indicate no traffic or wrong port. Syntax varies across models.

**Do not** repeatedly bounce production switch ports without checking what depends on them.

## 5. IP addressing and DHCP

### Verify the host configuration

Windows:

```powershell
ipconfig /all
Get-NetIPAddress -AddressFamily IPv4
Get-NetIPConfiguration
```

Linux:

```bash
ip -4 addr
ip route
resolvectl status
```

Check IP, prefix/mask, default gateway, DNS server, and DHCP lease source. For a `/24` subnet, `192.168.10.50` and `192.168.10.1` are in the same subnet; `192.168.20.20` is not.

**APIPA clue:** `169.254.x.x` on a Windows DHCP client commonly indicates no usable DHCP lease. Investigate DHCP server availability, VLAN placement, relay (`ip helper-address`), and exhausted scopes.

### DHCP lease renewal (changes connectivity)

Windows:

```powershell
ipconfig /release
ipconfig /renew
ipconfig /all
```

Run this at a local console or with an alternate management path: releasing an address can terminate a remote session.

Ubuntu using NetworkManager (if installed):

```bash
nmcli device status
sudo nmcli device reapply ens18
```

Ubuntu using systemd-networkd may instead require `networkctl` or a controlled service restart. **Do not assume all Ubuntu hosts use the same network manager.**

### Check for duplicate IPs

```powershell
arp -a
```

```bash
ip neigh show
```

A changing MAC for a stable IP may indicate a duplicate address, VRRP/HA behavior, or legitimate device replacement. Confirm using switch MAC tables and DHCP logs before concluding.

## 6. Gateway, routing, and NAT

Use the following progression:

```text
1. Ping local loopback (127.0.0.1)
2. Inspect local interface IP
3. Ping same-subnet known host
4. Ping default gateway
5. Test routed IP destination
6. Test destination by hostname
7. Test the actual application port
```

Windows:

```powershell
route print
tracert -d 1.1.1.1
Test-NetConnection 1.1.1.1 -InformationLevel Detailed
```

Linux:

```bash
ip route
ip route get 1.1.1.1
tracepath 1.1.1.1
```

If local VLAN communication works but another VLAN fails, inspect router interfaces, ACL/firewall policy, return routes, and host firewalls. For internet-bound private IPs, check whether the edge firewall performs source NAT. A missing return route can produce one-way communication.

**Traceroute caution:** Routers may deprioritize or block TTL-expired responses. Asterisks at one hop do not necessarily mean traffic stops there.

## 7. DNS troubleshooting

Separate **name resolution** from **IP reachability**.

Windows:

```powershell
Resolve-DnsName example.com
Resolve-DnsName example.com -Server 1.1.1.1
ipconfig /displaydns
ipconfig /flushdns
```

Linux:

```bash
resolvectl status
dig example.com
dig @1.1.1.1 example.com
dig +short example.com
getent hosts example.com
```

`dig` queries DNS; `getent` reflects system name-service resolution, which may also use local hosts files or other configured sources.

### Active Directory DNS

For a domain controller, verify domain-specific SRV records against **your internal AD DNS server**, not a public resolver:

```powershell
Resolve-DnsName _ldap._tcp.dc._msdcs.corp.example.test -Type SRV -Server 192.168.10.10
```

If domain logon fails but internet browsing works, inspect client DNS settings first. Domain-joined Windows clients generally need AD-aware DNS servers to locate domain controllers.

## 8. Ports, firewalls, and SSH

Windows client test:

```powershell
Test-NetConnection 192.168.20.20 -Port 22
Test-NetConnection 192.168.20.20 -Port 443
```

Linux client test:

```bash
nc -vz -w 3 192.168.20.20 22
curl -I --connect-timeout 5 https://example.com
```

On the destination Linux server:

```bash
sudo systemctl status ssh
sudo ss -lntp | grep ':22'
sudo ufw status verbose
```

On Windows server:

```powershell
Get-Service sshd
Get-NetTCPConnection -State Listen -LocalPort 22
Get-NetFirewallRule -DisplayGroup "OpenSSH Server" -ErrorAction SilentlyContinue
```

**Interpretation:**

- **Connection refused:** Host reachable, but nothing accepts the connection on that port or a firewall actively rejects it.
- **Timed out:** Packets or replies may be filtered, lost, or misrouted; do not assume one cause.
- **SSH permission denied:** Transport works; inspect account, authentication method, keys, and server logs.
- **TLS/certificate error:** TCP may work while the application-layer trust or hostname validation fails.

Use `ssh -vvv user@host` for verbose client logs. Do not disable host-key checking to make an unexpected fingerprint warning disappear; verify the server identity first.

## 9. Switches, VLANs, and trunks

### Access port vs trunk

- **Access port:** Typically connects an endpoint to one untagged VLAN.
- **Trunk port:** Carries multiple VLANs using 802.1Q tags; its allowed VLAN list must match the intended design.
- **Native VLAN:** Often untagged on an 802.1Q trunk; mismatches can cause unexpected connectivity or security issues.

Cisco IOS-style read-only checks:

```text
show vlan brief
show interfaces trunk
show interfaces gigabitEthernet0/2 switchport
show spanning-tree vlan 20
show mac address-table vlan 20
show ip interface brief
```

**Example lab configuration (do not apply to a live uplink without approval):**

```text
configure terminal
vlan 20
 name SERVERS
exit
interface gigabitEthernet0/2
 description LAB-UBUNTU
 switchport mode access
 switchport access vlan 20
 no shutdown
end
show vlan brief
```

A switch management SVI or routed interface is needed only where appropriate for the switch's role; creating a VLAN alone does **not** provide inter-VLAN routing.

### VLAN troubleshooting sequence

1. Confirm endpoint's assigned VLAN.
2. Confirm switch access-port membership.
3. Confirm VLAN exists on intermediate switches.
4. Confirm trunks allow the VLAN.
5. Confirm STP is not blocking the relevant path.
6. Confirm gateway interface, DHCP relay, ACL, and return route.
7. Re-test the original application.

## 10. Proxmox virtual networking

### Inspect without changing anything

On the Proxmox host:

```bash
ip -br addr
ip -br link
ip route
bridge link
bridge vlan show
cat /etc/network/interfaces
qm list
qm config 120
```

Replace VMID `120` with your VM's actual ID. Check that the VM's `net0` is attached to the intended bridge, has the intended VLAN tag, and uses a supported guest NIC model.

Example VM network line:

```text
net0: virtio=BC:24:11:AA:BB:CC,bridge=vmbr0,tag=20
```

This means the VM's virtual NIC connects to `vmbr0` and Proxmox applies VLAN 20 tagging. The example MAC is illustrative.

### Example VLAN-aware bridge

```text
auto vmbr0
iface vmbr0 inet manual
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094
```

**Do not paste this blindly.** Existing host management addressing, bridge names, bonding, and physical interfaces differ. An incorrect bridge change can lock you out of Proxmox. Back up `/etc/network/interfaces` and use local/iKVM console access before modifying it. Proxmox may use `ifupdown2` for applying network changes; prefer the supported GUI workflow and validate pending changes before committing.

### VM-level checks

```bash
qm config 120 | grep -E '^(net|boot|machine|ostype)'
```

Inside the guest:

```bash
ip -br addr
ip route
ping -c 4 192.168.20.1
```

If the guest has no IP: check virtual NIC presence, guest driver, DHCP, VLAN tag, switch trunk, and whether the bridge uplink is physically active.

## 11. Performance, MTU, and packet loss

### Latency and loss

Windows:

```powershell
ping -n 20 192.168.20.20
pathping 192.168.20.20
```

Linux:

```bash
ping -c 20 192.168.20.20
ip -s link show dev ens18
```

Compare a local gateway, an internal server, and a remote destination. High latency to one destination is not automatically a local switch issue. Test repeatedly during the reported incident window.

### MTU checks

Windows (IPv4, do-not-fragment, payload of 1472 bytes for an Ethernet MTU of 1500):

```powershell
ping 192.168.20.20 -f -l 1472
```

Linux:

```bash
ping -M do -s 1472 -c 3 192.168.20.20
```

These are diagnostic examples; VPNs, tunnels, IPv6, and encapsulation can reduce usable payload size. Firewalls may block ICMP, making the result inconclusive.

### Throughput with iperf3 (authorized endpoints only)

Server:

```bash
iperf3 -s
```

Client:

```bash
iperf3 -c 192.168.20.20 -t 15
```

Install `iperf3` if needed. A throughput test consumes bandwidth; schedule appropriately and do not run it against hosts without permission.

## 12. Packet capture with tcpdump and Wireshark

**Capture only authorized traffic.** Packet captures may contain credentials, session tokens, internal addresses, or other sensitive data. Protect and delete captures according to your organization's rules.

Linux examples:

```bash
sudo tcpdump -ni ens18 'host 192.168.20.20 and port 22'
sudo tcpdump -ni ens18 'port 53'
sudo tcpdump -ni ens18 -c 100 -w /tmp/network-lab.pcap
```

- `-n`: Do not resolve IPs to hostnames.
- `-i ens18`: Capture on the named interface.
- `-c 100`: Stop after 100 packets.
- `-w`: Write a packet-capture file.

Wireshark display filters:

```text
dns
tcp.port == 22
ip.addr == 192.168.20.20
tcp.analysis.retransmission
arp
```

### Reading a TCP handshake

```text
Client  → Server : SYN
Server  → Client : SYN, ACK
Client  → Server : ACK
```

If you see repeated SYN packets but no reply, inspect path, firewall, server availability, and return routing. If a RST comes back, the destination or an intermediary may be rejecting the connection. Capture from both ends when possible.

## 13. Windows and Linux troubleshooting scripts

These scripts collect diagnostics; they do not change network settings.

### Bash: `network-check.sh`

```bash
#!/usr/bin/env bash
set -u
TARGET="${1:-192.168.10.1}"
LOG="network-check-$(date +%Y%m%d-%H%M%S).log"
{
  echo "=== Timestamp ==="; date -Is
  echo "=== Host ==="; hostname
  echo "=== Interfaces ==="; ip -br addr
  echo "=== Routes ==="; ip route
  echo "=== DNS ==="; resolvectl status 2>&1 || cat /etc/resolv.conf
  echo "=== Ping ${TARGET} ==="; ping -c 4 -W 2 "$TARGET" 2>&1
  echo "=== SSH port ${TARGET}:22 ==="; timeout 5 bash -c "</dev/tcp/$TARGET/22" 2>&1 && echo OPEN || echo CLOSED_OR_FILTERED
} | tee "$LOG"
echo "Saved: $LOG"
```

Run:

```bash
chmod +x network-check.sh
./network-check.sh 192.168.20.20
```

The `/dev/tcp` test requires Bash; a failure cannot distinguish all firewall and listener scenarios. The target argument should be a trusted hostname or IP; do not pass untrusted shell metacharacters.

### PowerShell: `Network-Check.ps1`

```powershell
param(
    [string]$Target = '192.168.10.1',
    [int]$Port = 22
)
$Stamp = Get-Date -Format 'yyyyMMdd-HHmmss'
$Log = "network-check-$Stamp.txt"
$Output = @(
    "=== Timestamp ==="
    (Get-Date).ToString('o')
    "=== Host ==="
    $env:COMPUTERNAME
    "=== IP configuration ==="
    (Get-NetIPConfiguration | Out-String)
    "=== Routes ==="
    (Get-NetRoute -AddressFamily IPv4 | Out-String)
    "=== Ping ==="
    (Test-Connection -ComputerName $Target -Count 4 | Out-String)
    "=== TCP port ==="
    (Test-NetConnection -ComputerName $Target -Port $Port | Out-String)
)
$Output | Set-Content -Path $Log -Encoding utf8
Get-Content $Log
Write-Host "Saved: $Log"
```

Run from PowerShell in the folder containing the file:

```powershell
.\Network-Check.ps1 -Target 192.168.20.20 -Port 22
```

If local script execution is restricted, follow your organization's approved execution policy; do not bypass security controls. Note that some PowerShell cmdlets differ across editions and platforms; this script targets Windows PowerShell/PowerShell on Windows.

## 14. Ten hands-on labs

For each lab, write down **symptom → evidence → hypothesis → fix → verification**.

### Lab 1 — Identify the local network

**Task:** On Windows, run `ipconfig /all`, `Get-NetAdapter`, and `route print`. On Ubuntu, run `ip -br addr`, `ip route`, and `resolvectl status`.

**Success:** Record interface, IPv4 address, prefix, gateway, DNS server, and MAC address. Explain each field.

### Lab 2 — Diagnose an unreachable gateway

**Task:** Verify interface link, local IP/subnet, neighbor table, and gateway reachability. Use a deliberately disconnected lab NIC if available.

**Success:** Identify whether failure occurs at link, addressing, VLAN, or gateway stage; restore connectivity.

### Lab 3 — DHCP failure

**Task:** In an isolated VM VLAN, disable the test DHCP scope or remove the DHCP relay temporarily using the lab console.

**Success:** Observe the lease failure, inspect DHCP logs and VLAN, restore service, renew lease, and document the assigned address. Avoid disrupting shared networks.

### Lab 4 — DNS vs IP connectivity

**Task:** Test an IP destination and then its hostname. Temporarily assign an invalid DNS server to a disposable VM, not your management host.

**Success:** Show that IP connectivity can work while name resolution fails; restore DNS and verify.

### Lab 5 — SSH port unreachable

**Task:** From Windows, use `Test-NetConnection SERVER -Port 22`. On Ubuntu, inspect `systemctl status ssh` and `ss -lntp`.

**Success:** Distinguish service stopped, firewall blocked, wrong IP, and bad credentials.

### Lab 6 — VLAN mismatch

**Task:** Put a disposable VM in VLAN 20 while its expected gateway exists only in VLAN 10. Inspect Proxmox `qm config`, switch trunk, and `bridge vlan show`.

**Success:** Explain why a valid IP alone is insufficient; restore correct tagging and verify gateway reachability.

### Lab 7 — Inter-VLAN routing

**Task:** With authorized lab router/firewall, test VLAN 10 → VLAN 20 TCP access. Inspect gateway interfaces, routes, ACLs, and return path.

**Success:** Show a working allowed service and a blocked service consistent with policy.

### Lab 8 — Packet capture of DNS and SSH

**Task:** Capture your own VM's DNS query and SSH handshake using `tcpdump`; open the `.pcap` in Wireshark.

**Success:** Identify DNS query/response and TCP SYN/SYN-ACK/ACK without exposing secrets.

### Lab 9 — MTU and performance

**Task:** Run a short authorized `iperf3` test and compare with ping latency/loss. Try an MTU diagnostic on your lab path.

**Success:** Distinguish throughput, latency, loss, and MTU as separate measurements.

### Lab 10 — Full incident simulation

**Scenario:** A Windows client can access the gateway but cannot SSH to an Ubuntu VM on a different VLAN.

**Workflow:**

1. Confirm source and destination IP/subnet/VLAN.
2. Test gateway reachability from both ends.
3. Inspect Proxmox NIC bridge/tag and physical switch trunk.
4. Inspect inter-VLAN routing and firewall rules.
5. Check Ubuntu `sshd` service and TCP listener.
6. Test TCP port 22 from Windows.
7. Capture packets on the destination if still failing.
8. Restore the intended configuration and test login.

**Success:** Produce a one-page incident report with commands, evidence, cause, correction, and post-fix checks.

## 15. Symptom-to-cause quick reference

| Symptom | First checks | Common causes |
|---|---|---|
| `169.254.x.x` address | DHCP lease, VLAN, relay | No DHCP response, scope exhausted |
| IP works, hostname fails | DNS servers, resolver, records | Wrong DNS, missing record, search suffix |
| Gateway reachable, internet fails | Default route, NAT, firewall | Missing NAT, upstream outage, ACL |
| Same VLAN works, other VLAN fails | Router, ACL, return route | Missing L3 interface or policy |
| VM has no network | Guest NIC, `qm config`, bridge | Driver, wrong bridge/tag, disabled NIC |
| SSH times out | TCP 22 test, firewall, route | Filtering, packet loss, wrong destination |
| SSH says permission denied | User, keys, server logs | Authentication problem |
| Intermittent packet loss | Errors, duplex, congestion | Cable, overloaded link, Wi-Fi interference |
| Slow file transfers | `iperf3`, NIC errors, MTU | Congestion, loss, disk bottleneck |
| DNS works, website fails | TCP 443, TLS, proxy, app logs | Service down, firewall, certificate |

## 16. Incident documentation and final checklist

### Incident report template

```markdown
# Network Incident Report
- Incident ID:
- Start/end time and timezone:
- Impacted users/services:
- Source host/IP/VLAN:
- Destination host/IP/port:
- Expected behavior:
- Observed behavior:
- Recent changes:
- Evidence and command output:
- Root cause (confirmed vs suspected):
- Corrective action:
- Rollback performed or available:
- Verification of original service:
- Follow-up / prevention:
```

### Skills checklist

- [ ] Identify interface link state and IP configuration on Windows and Linux.
- [ ] Explain subnet masks, default gateways, and DNS roles.
- [ ] Diagnose DHCP failures without disrupting production.
- [ ] Differentiate ICMP reachability from TCP service availability.
- [ ] Trace a route and interpret incomplete traceroute responses.
- [ ] Inspect switch access VLANs, trunks, and MAC tables.
- [ ] Inspect Proxmox bridges and per-VM VLAN tags.
- [ ] Identify SSH listener, firewall, and authentication failures.
- [ ] Capture and interpret a DNS query and TCP handshake.
- [ ] Measure latency, packet loss, and throughput responsibly.
- [ ] Run and explain the Bash and PowerShell diagnostic scripts.
- [ ] Produce an evidence-based incident report.

## Further reading

- [Microsoft Learn: Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [Microsoft Learn: Get-NetIPConfiguration](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-netipconfiguration)
- [Proxmox VE Administration Guide](https://pve.proxmox.com/pve-docs/pve-admin-guide.html)
- [Wireshark User's Guide](https://www.wireshark.org/docs/wsug_html_chunked/)
- [tcpdump manual](https://www.tcpdump.org/manpages/tcpdump.1.html)

---

**Next suggested module:** Network monitoring and observability — SNMP, syslog, NetFlow, Zabbix, Grafana, alerting, and baseline performance.
