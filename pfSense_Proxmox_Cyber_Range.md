# pfSense on Proxmox VE 9.x — Cyber Range Firewall Lab

> Hands-on IT 101 → Advanced lab. Use only on systems you own or are authorized to test. Keep intentionally vulnerable targets isolated from production networks.

## 1. Goal and topology

Build a pfSense virtual firewall that routes between three **internal-only** Proxmox Linux bridges. The Proxmox management bridge remains separate.

```text
Home/management LAN -- vmbr0 -- Proxmox VE 9.x management ONLY
                                 |
                 pfSense VM (no physical WAN in phase 1)
                   |             |             |
                vmbr10        vmbr20        vmbr30
             10.10.10.1     10.10.20.1    10.10.30.1
                Kali         Targets       Monitoring
             .10.10         .20.10         .30.10
```

| Network | Bridge | Gateway | Example VM | Purpose |
|---|---|---|---|---|
| Management | `vmbr0` | Existing LAN router | Proxmox host | Host administration only |
| Attacker | `vmbr10` | `10.10.10.1/24` | Kali `10.10.10.10` | Authorized lab tests |
| Targets | `vmbr20` | `10.10.20.1/24` | Ubuntu `10.10.20.10` | Isolated practice targets |
| Monitoring | `vmbr30` | `10.10.30.1/24` | Collector `10.10.30.10` | Logs and telemetry |

**Important pfSense design detail:** pfSense requires a WAN assignment during its standard initial setup, even if the cyber range has no real WAN uplink. For an offline lab, create a **fourth internal bridge** `vmbr99`, attach pfSense's WAN interface to it, and leave that bridge disconnected from physical interfaces. Assign no upstream gateway or DHCP on that WAN. Do **not** connect the pfSense WAN to `vmbr0` unless you intentionally redesign and secure the lab.

## 2. Prerequisites

- Proxmox VE 9.x installed and reachable through its existing management network.
- Hardware virtualization enabled, about 2 vCPUs, 4 GB RAM, 32 GB disk for pfSense; additional resources for guests.
- A current pfSense CE installer image obtained from the [official pfSense download page](https://www.pfsense.org/download/). Verify the image checksum when supplied.
- Local console access to Proxmox before modifying network settings.
- A written rollback plan and current backups of existing workloads.

## 3. Create the internal Proxmox bridges

In the web UI: **Node → System → Network → Create → Linux Bridge**.

Create `vmbr10`, `vmbr20`, `vmbr30`, and `vmbr99`. For each, leave **Bridge ports** empty, do not set a gateway, and do not set an IP address on the Proxmox host. Apply configuration only after confirming `vmbr0` is unchanged.

Example `/etc/network/interfaces` snippets (append carefully; do not overwrite your existing configuration):

```ini
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

auto vmbr99
iface vmbr99 inet manual
    bridge-ports none
    bridge-stp off
    bridge-fd 0
```

Inspect before applying:

```bash
cat /etc/network/interfaces
ip -br link
```

Use the Proxmox GUI's **Apply Configuration** or the platform-supported `ifreload -a` from a local console. Do not interrupt a production management connection.

### Bridges versus VLANs

These four bridges are **separate virtual switches**. No 802.1Q VLAN tags are needed for this first lab. A later VLAN-aware design can use one trunk bridge with tagged VM interfaces, but it requires correct physical switch and pfSense VLAN configuration; do not mix the two models accidentally.

## 4. Create the pfSense VM

1. Upload the installer ISO to **local → ISO Images → Upload**.
2. Select **Create VM**, VM ID `105`, name `pfsense-fw01`.
3. Select the installer ISO. Use **Other** as the OS type if appropriate.
4. Choose a compatible machine/firmware combination, a 32 GB disk, 2 CPU cores, and 4096 MB RAM.
5. Start with a **VirtIO** NIC attached to `vmbr99` (WAN placeholder).
6. In **VM → Hardware → Add → Network Device**, add three more VirtIO NICs: `vmbr10`, `vmbr20`, `vmbr30`.
7. Record each adapter's MAC address and corresponding Proxmox bridge before booting.
8. Start the VM and install pfSense from the console; detach the ISO after installation.

Example inventory:

| Proxmox adapter | Bridge | Intended pfSense assignment |
|---|---|---|
| net0 | vmbr99 | WAN (offline placeholder) |
| net1 | vmbr10 | LAN / Kali |
| net2 | vmbr20 | OPT1 / Targets |
| net3 | vmbr30 | OPT2 / Monitoring |

**Verify the actual interface mapping by MAC address**: FreeBSD/pfSense interface names may vary (for example `vtnet0`, `vtnet1`). Do not assume numbering without checking.

## 5. Assign pfSense interfaces

From the pfSense VM console, choose the interface-assignment option. Assign WAN to the NIC on `vmbr99`, LAN to `vmbr10`, OPT1 to `vmbr20`, and OPT2 to `vmbr30`.

Set static interface addresses:

- LAN: `10.10.10.1/24`
- OPT1: `10.10.20.1/24`
- OPT2: `10.10.30.1/24`
- WAN: disconnected placeholder; no upstream default gateway

If the installer requires a temporary WAN configuration, use an isolated, non-overlapping test subnet only on `vmbr99`, with no gateway and no physical uplink. Do not allow it to become an alternate route into the home LAN.

To administer pfSense, attach a **temporary trusted admin VM** to `vmbr10` with IP `10.10.10.50/24`. Browse to `https://10.10.10.1`, verify the connection is to your pfSense VM, log in with the installer-provided credentials, and immediately change the administrator password. Follow the setup wizard without adding an upstream WAN gateway.

## 6. Configure guest IPs

Use static addresses initially to make troubleshooting easier.

### Kali on vmbr10

- IP: `10.10.10.10/24`
- Gateway: `10.10.10.1`

### Ubuntu target on vmbr20

Check interface name:

```bash
ip -br addr
```

Example `/etc/netplan/01-lab.yaml` (replace `ens18` with the actual interface):

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

Apply from the VM console:

```bash
sudo netplan try
ip route
```

### Monitoring VM on vmbr30

- IP: `10.10.30.10/24`
- Gateway: `10.10.30.1`

Optional: configure DHCP pools in pfSense under **Services → DHCP Server** (or **Services → DHCP**, depending on version), such as `.100–.199` on each lab subnet. Avoid handing out addresses that overlap your static IPs.

## 7. Firewall rules: deny first, permit narrowly

pfSense rules are evaluated on the **ingress interface**. Default LAN rules may permit broad outbound access; review and remove or narrow these before treating the range as isolated.

Start with no cross-subnet access, then add explicit rules for authorized tests:

| Interface where packet enters | Source | Destination | Service | Action |
|---|---|---|---|---|
| LAN/Kali | `10.10.10.10` | `10.10.20.10` | ICMP, then selected training ports | Pass |
| LAN/Kali | Lab subnet | Management subnet | Any | Block |
| OPT1/Targets | `10.10.20.10` | `10.10.30.10` | Specific log receiver port | Pass if required |
| OPT1/Targets | Target subnet | Other networks | Any | Block by default |
| OPT2/Monitoring | Monitoring subnet | Management subnet | Any | Block |

Place specific blocks above broad allows where necessary. Remember that an implicit deny at the end of a rule set blocks unmatched traffic, but only after all existing pass rules are considered. Check floating rules, interface groups, NAT, and IPv6 as well. Do not enable permissive any-to-any rules to make a lab test work.

### pfSense web UI

Navigate to **Firewall → Rules → [interface] → Add**, choose action, protocol, source, destination, and destination port; save and **Apply Changes**. Give each rule a descriptive name and enable logging for important denies.

### Internet access and updates

Initially **do not connect the lab to the internet**. If updates are needed, prefer offline package transfer or a temporary, explicitly controlled egress design with outbound allowlisting and logging. Re-isolate vulnerable VMs before exercises. Never port-forward vulnerable lab services from a public interface.

## 8. Verify connectivity and isolation

On Kali:

```bash
ip -br addr
ip route
ping -c 4 10.10.10.1
ping -c 4 10.10.20.10
```

On Ubuntu:

```bash
ip -br addr
ip route
ping -c 4 10.10.20.1
```

From the pfSense web UI inspect **Status → System Logs → Firewall** (menu wording may vary). Use **Diagnostics → Ping** to test from the firewall itself.

### Acceptance checklist

- [ ] Proxmox management remains reachable only from authorized administration networks.
- [ ] `vmbr10`, `vmbr20`, `vmbr30`, `vmbr99` have no physical bridge ports.
- [ ] pfSense interfaces match the intended bridges and subnets.
- [ ] Kali can reach its pfSense gateway.
- [ ] Target can reach its pfSense gateway.
- [ ] Kali reaches the designated target **only** when the relevant rule is enabled.
- [ ] Target cannot reach the management LAN or public internet.
- [ ] IPv6 does not bypass isolation policies.
- [ ] Firewall deny events are visible in logs.
- [ ] Clean VM snapshots and a pfSense configuration backup exist.

## 9. Common problems

| Symptom | Check | Fix |
|---|---|---|
| Cannot open pfSense GUI | Admin VM bridge, IP, HTTPS, interface mapping | Use Proxmox console, verify LAN IP and NIC mapping |
| Ping gateway fails | Wrong VM bridge, subnet mask, guest NIC down | Compare VM Hardware NIC and `ip -br addr` |
| Cross-subnet ping fails | pfSense rules, gateway, ICMP filtering | Check ingress rules and firewall logs |
| Unexpected internet access | WAN uplink, NAT, broad pass rule, IPv6 | Disconnect uplink; inspect rules and routes |
| DHCP not working | Service enabled on correct interface, pool, guest settings | Check DHCP leases and guest renewal |
| Web UI disappears after changes | Interface assignment or lockout | Recover via pfSense VM console; restore saved config |
| VM performance poor | CPU, RAM, disk contention, NIC model | Check Proxmox resources and VirtIO configuration |

## 10. pfSense backups and recovery

1. In pfSense, open **Diagnostics → Backup & Restore** (menu labels vary by release).
2. Export the configuration XML and store it securely; it can contain sensitive secrets.
3. Take a Proxmox backup of the firewall VM before major changes.
4. Snapshot lab guests before exercises, but do not treat snapshots as a substitute for independent backups.
5. Test restoring the pfSense configuration to a disposable VM before depending on it for recovery.

## 11. Hands-on exercises

1. **Build:** Create four internal bridges and verify they have no physical uplinks.
2. **Install:** Deploy pfSense, map NICs by MAC, configure three lab gateways.
3. **Restrict:** Prove that Kali cannot reach the target before the allow rule.
4. **Permit:** Add one narrowly scoped ICMP rule and prove the target becomes reachable.
5. **Observe:** Turn on rule logging and identify a denied connection.
6. **Recover:** Export pfSense configuration, change a harmless rule, and restore the saved config.
7. **Document:** Record VM IDs, addresses, bridge names, rules, screenshots, and test evidence.

## 12. Next steps

- Add an isolated Windows Server Active Directory environment to `vmbr20`.
- Add a SIEM/logging collector to `vmbr30` and configure syslog forwarding.
- Learn the difference between pfSense interface-based segmentation and 802.1Q VLAN trunks.
- Build a temporary, restricted update path only after the offline range passes its isolation checks.

## References

- [pfSense Documentation — Netgate](https://docs.netgate.com/pfsense/en/latest/)
- [pfSense Virtualization](https://docs.netgate.com/pfsense/en/latest/recipes/virtualize-proxmox-ve.html)
- [Proxmox VE Administration Guide](https://pve.proxmox.com/pve-docs/)
