# Build a Cybersecurity Range with Proxmox VE 9.x

A beginner-to-advanced, hands-on guide for a **self-contained, authorized training environment**. Do not attach deliberately vulnerable targets to a production LAN or expose them to the public internet.

## 1. Objectives

By the end you can deploy isolated virtual networks, configure a virtual firewall, install Kali and target VMs, verify traffic policies, take snapshots, and reset your lab safely.

## 2. Hardware and prerequisites

- Proxmox VE 9.x host; 8+ CPU threads, 32–64 GB RAM and 500 GB+ SSD recommended for several VMs.
- Administrative access to the Proxmox web UI and console.
- Ubuntu Server ISO, Kali Linux ISO, optional Windows 11 evaluation installation media, and OPNsense installation media downloaded from official sites.
- A dedicated, authorized practice environment and a written list of permitted targets.

**Resource tip:** Start with Kali, OPNsense and one Ubuntu target if memory is limited. Avoid overcommitting memory on a busy host.

## 3. Reference architecture and IP plan

```text
                 Your trusted LAN
                       |
               vmbr0 (existing)
                       |
               Proxmox host UI

    No physical uplinks on the lab bridges

 Kali01 -------- vmbr10 -------- OPNsense -------- vmbr20 -------- Ubuntu01
10.10.10.10/24  attacker         |             target          10.10.20.10/24
 GW .10.1       segment          |
                               vmbr30 -------- Monitor01
                               monitoring      10.10.30.10/24

OPNsense interfaces:
  ATTACKER: 10.10.10.1/24
  TARGET:   10.10.20.1/24
  MONITOR:  10.10.30.1/24
```

| Network | CIDR | Gateway | Purpose |
|---|---|---|---|
| Management | Your existing LAN subnet | Existing router | Proxmox administration only |
| Attacker | `10.10.10.0/24` | `10.10.10.1` | Kali workstation |
| Target | `10.10.20.0/24` | `10.10.20.1` | Training targets |
| Monitoring | `10.10.30.0/24` | `10.10.30.1` | Logging and detection |

**Important:** These are **separate virtual bridges**, not 802.1Q VLAN tags. They work without a managed physical switch. If you later use VLANs on a trunk, document the physical switch configuration separately.

## 4. Preserve management access

1. Sign in to `https://<proxmox-management-ip>:8006`.
2. Select the node → **System → Network**.
3. Record the current `vmbr0`, management address, gateway, and physical NIC mapping.
4. Use a local console or out-of-band access for any risky network changes. Do **not** move your management NIC into a lab bridge.
5. Back up the host network configuration before editing:

```bash
sudo cp /etc/network/interfaces /root/interfaces.before-cyber-range
ip -br addr
ip route
```

## 5. Create isolated Proxmox bridges

In the web UI, open **Node → System → Network → Create → Linux Bridge**. Create each bridge as follows:

| Name | Bridge ports | Host IPv4/CIDR | Host gateway | VLAN aware |
|---|---|---|---|---|
| `vmbr10` | blank | blank | blank | No |
| `vmbr20` | blank | blank | blank | No |
| `vmbr30` | blank | blank | blank | No |

Select **Apply Configuration** only after confirming `vmbr0` remains unchanged. A typical `/etc/network/interfaces` addition is:

```text
# Cyber-range only: no physical NIC, no host IP, no gateway
auto vmbr10
iface vmbr10 inet manual
    bridge-ports none
    bridge-stp off
    bridge-fd 0

auto vmbr20
iface vmbr20 inet manual
    bridge-ports none
    bridge-stp off
    bridge-fd 0

auto vmbr30
iface vmbr30 inet manual
    bridge-ports none
    bridge-stp off
    bridge-fd 0
```

**Do not replace the existing file with this snippet**; append only the intended bridge stanzas using the UI or careful editing. Verify:

```bash
ip -br link show vmbr10 vmbr20 vmbr30
bridge link
```

## 6. Create the firewall/router VM

Create VM `FW01` with approximately 2 vCPUs, 4 GB RAM, and 20 GB disk. Install OPNsense from official media. In **VM → Hardware → Add → Network Device**, create three NICs:

- `net0` on `vmbr10`
- `net1` on `vmbr20`
- `net2` on `vmbr30`

Do not attach `FW01` to `vmbr0` or any external uplink during the initial build. From the OPNsense console, map the virtual NICs to the correct interfaces by checking their MAC addresses, then assign static interface IPs `.1/24` as in the plan.

In the OPNsense web UI, configure firewall rules deliberately. Because firewall platforms differ in default policies, **verify the actual effective rules instead of assuming isolation**. For an initial safe state, block traffic from all lab interfaces to the management LAN and any non-lab destinations. Permit only the specific Kali-to-target ports needed for an exercise, and explicitly allow target-to-monitor logging traffic where required. Avoid broad allow-any rules. Consider using an isolated management workstation VM to reach the firewall UI from `vmbr10`.

**Critical validation:** An interface without a WAN NIC is not by itself a complete security guarantee. Test firewall policy, guest routing, and Proxmox NIC attachments before using vulnerable targets.

## 7. Create Kali01

Create a Kali Linux VM with 2 vCPUs, 4 GB RAM, 40 GB disk and one NIC on `vmbr10`. Install Kali using the ISO. In the guest, identify the interface:

```bash
ip -br addr
ip route
```

Configure `10.10.10.10/24` with gateway `10.10.10.1` using the guest's network manager or installer settings. Verify:

```bash
ping -c 3 10.10.10.1
```

## 8. Create Ubuntu01 target

Create an Ubuntu Server VM with 2 vCPUs, 2–4 GB RAM, 25 GB disk and one NIC on `vmbr20`. Assign `10.10.20.10/24` and gateway `10.10.20.1`. If using Netplan, first identify the actual NIC name (`ens18` is only an example):

```bash
ip -br addr
sudo nano /etc/netplan/99-cyber-range.yaml
```

Example:

```yaml
network:
  version: 2
  ethernets:
    ens18:
      dhcp4: false
      addresses: [10.10.20.10/24]
      routes:
        - to: default
          via: 10.10.20.1
```

Apply from the VM console so a network mistake will not strand an SSH session:

```bash
sudo netplan try
ip route
ping -c 3 10.10.20.1
```

**DNS:** Offline exercises do not require public DNS. For multi-VM naming, configure a lab-only resolver on OPNsense or use `/etc/hosts` for a few hosts.

## 9. Optional Windows and monitoring VMs

- `WIN11-01`: 2–4 vCPUs, 4–8 GB RAM, 64 GB disk, on `vmbr20`; use evaluation media and applicable licensing. Address `10.10.20.20/24`.
- `MONITOR01`: Ubuntu Server, 2 vCPUs, 4 GB RAM, on `vmbr30`; address `10.10.30.10/24`.
- Configure host firewalls, logging, and time synchronization as appropriate. If using a SIEM, check its actual resource requirements before installing.

## 10. Install deliberately vulnerable practice apps safely

Use only intentionally vulnerable software and systems that you are authorized to test, such as OWASP Juice Shop or purpose-built training images. Keep them on `vmbr20`. Prefer offline package/image transfer through trusted removable media or a tightly controlled, temporary staging process. Do not enable a general-purpose WAN route for vulnerable targets.

Before introducing any vulnerable app:

1. Confirm its NIC is connected only to `vmbr20`.
2. Confirm no Proxmox port forwarding or upstream NAT exposes it.
3. Take a clean snapshot.
4. Record its IP, services, credentials and reset method in your lab inventory (store secrets securely).

## 11. Test connectivity and isolation

Run from Kali:

```bash
ip route
ping -c 3 10.10.10.1
ping -c 3 10.10.20.10
```

The second ping may be blocked intentionally; **a failed ping does not prove complete isolation**. Test the specific permitted application port instead:

```bash
nc -vz -w 3 10.10.20.10 22
```

Run from the Ubuntu target:

```bash
ip route
ss -lnt
```

From a lab VM, test that the Proxmox management IP and trusted LAN devices are **not reachable**. Repeat after every change to firewall rules, NIC attachments, or routing. Never scan devices outside the written lab scope.

## 12. Add a VLAN-based variant (optional)

If you have a managed switch and need to extend lab networks to physical ports, use VLAN IDs 110 (attacker), 120 (target), and 130 (monitor). On a dedicated physical trunk, configure the switch to allow only those IDs; on Proxmox, use a VLAN-aware bridge with the appropriate physical NIC and assign VM NIC VLAN tags. **Do not reuse the management bridge or allow lab VLANs onto untrusted ports without a reviewed design.** For a single-host range, isolated bridges are simpler and safer.

## 13. Snapshots, backups, and resets

In Proxmox select **VM → Snapshots → Take Snapshot** after installing and configuring each clean VM. Snapshots are useful for quick lab resets but **are not a substitute for backups**. Use **VM → Backup** or scheduled Proxmox backup jobs to separate storage, and test at least one restore.

Suggested snapshot names:

```text
clean-os
configured-network
before-exercise-01
```

Restore a snapshot only when you understand that newer guest changes and data may be discarded.

## 14. Common problems

| Symptom | Check | Fix |
|---|---|---|
| VM cannot ping its gateway | VM bridge, interface address, firewall VM NIC mapping | Correct bridge/NIC/IP; verify link state |
| Kali cannot reach target | OPNsense rules and route, target host firewall | Permit only required traffic |
| Lab can reach home LAN | Extra NIC, NAT, routing, permissive firewall | Disconnect unintended uplink and block management destinations |
| Ubuntu IP disappears | Incorrect Netplan interface name | Inspect `ip -br link`; correct YAML; use `netplan try` |
| Proxmox UI inaccessible | Management bridge modified | Recover from local console using backed-up config |
| VM performance poor | RAM pressure, slow storage, CPU contention | Reduce concurrent VMs; inspect host resources |
| DNS fails | No lab resolver or incorrect DNS address | Configure internal DNS or lab hosts entries |

## 15. Practical exercises

1. Build three isolated bridges and document their purpose.
2. Deploy OPNsense and prove that management traffic is blocked.
3. Deploy Kali and Ubuntu, configure static IPs, and verify gateway connectivity.
4. Permit one specific Kali-to-Ubuntu TCP service; prove other access is denied.
5. Add a monitoring VM and forward selected logs to it.
6. Snapshot, change a target configuration, and restore the snapshot.
7. Write a one-page incident report showing the issue, evidence, corrective action, and retest.

## 16. Acceptance checklist

- [ ] Management bridge unchanged and accessible.
- [ ] Lab bridges have no physical ports and no host gateway.
- [ ] Firewall VM has only the intended lab NICs.
- [ ] Every VM uses its designated subnet and bridge.
- [ ] Management LAN and non-lab destinations are inaccessible from lab VMs.
- [ ] Authorized traffic between lab segments behaves as specified.
- [ ] Vulnerable targets have no public exposure.
- [ ] Clean snapshots and separate backups exist.
- [ ] Restore test completed.
- [ ] IP plan, firewall rules, and lab scope documented.

## 17. Suggested repository structure

```text
IT-101-to-Expert/
└── cybersecurity/
    └── cyber-range/
        ├── README.md
        ├── diagrams/
        ├── configs/
        └── lab-notes/
```

## 18. References

- Proxmox VE documentation: https://pve.proxmox.com/pve-docs/
- OPNsense documentation: https://docs.opnsense.org/
- Ubuntu Server documentation: https://documentation.ubuntu.com/server/
- Kali Linux documentation: https://www.kali.org/docs/
- OWASP Juice Shop: https://owasp.org/www-project-juice-shop/

> **Safety:** Practice only against systems you own or are explicitly authorized to test. Keep intentionally vulnerable machines isolated, document all permitted access, and verify segmentation before starting an exercise.
