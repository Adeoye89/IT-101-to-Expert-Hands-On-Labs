# Tailscale 101 — Hands-On Training (Beginner to Advanced)

> **Goal:** Connect your own computers and Proxmox lab securely using Tailscale. Follow the labs in order. Commands are shown for Ubuntu/Debian; Tailscale supports other operating systems with different installation steps.

## Contents
1. [What Tailscale is](#1-what-tailscale-is)
2. [Concepts and security model](#2-concepts-and-security-model)
3. [Create a tailnet](#3-create-a-tailnet)
4. [Install on Windows](#4-install-on-windows)
5. [Install on Ubuntu or Kali](#5-install-on-ubuntu-or-kali)
6. [Connect two devices](#6-connect-two-devices)
7. [MagicDNS and SSH](#7-magicdns-and-ssh)
8. [Proxmox remote management](#8-proxmox-remote-management)
9. [Subnet routing](#9-subnet-routing)
10. [Access controls and device security](#10-access-controls-and-device-security)
11. [Exit nodes](#11-exit-nodes)
12. [Troubleshooting and capstone](#12-troubleshooting-and-capstone)

## 1. What Tailscale is
**Tailscale** creates a private network (a *tailnet*) among authorized devices using WireGuard-based encrypted tunnels. It often works without manually forwarding router ports. Its coordination service helps devices discover one another; encrypted traffic may connect directly or traverse relays depending on network conditions.

**Example:** Your Windows laptop away from home connects to your Proxmox host at its Tailscale address, rather than exposing port 8006 to the public internet.

**Do not confuse:** Tailscale is not a replacement for OS patching, strong authentication, backups, or access controls. Only enroll devices you trust.

## 2. Concepts and security model
| Term | Meaning |
|---|---|
| Tailnet | Your private Tailscale network |
| Node | An enrolled device |
| Tailscale IP | Private address assigned to a node, often in the 100.64.0.0/10 range |
| MagicDNS | DNS names for nodes within the tailnet |
| Subnet router | Authorized node forwarding traffic to a LAN subnet |
| Exit node | Node used as an internet egress point by selected clients |
| Grants/ACLs | Rules governing which identities/devices may access resources |
| Device approval | Optional requirement to approve new devices |
| Key expiry | Device authentication lifecycle setting |

**Security:** Use MFA on your identity provider, review device inventory, limit access by role, and remove lost or unused devices promptly. Never share reusable authentication keys in README files.

## 3. Create a tailnet
1. Visit https://tailscale.com/ and sign up/sign in with a supported identity provider.
2. Open the **Admin console** at https://login.tailscale.com/admin .
3. Locate the **Machines/Devices** area (labels may vary with UI changes).
4. Review the tailnet name, authentication settings, and access policy.
5. If available for your plan, enable device approval and review security settings.

**Verify:** You can access the admin console and see your tailnet, even if it has zero enrolled devices.

## 4. Install on Windows
1. Download the Windows installer from https://tailscale.com/download .
2. Run the installer, open Tailscale, and choose **Log in**.
3. Authenticate using the same identity/tailnet from Lab 3.
4. In the admin console, confirm your Windows device appears.
5. Open PowerShell and run (if Tailscale is on your PATH):
```powershell
tailscale status
tailscale ip -4
```
`status` shows devices/connectivity; `ip -4` shows the local device's Tailscale IPv4 address. If the command is not recognized, use the Tailscale application UI or its installed CLI path.

**Practice:** Record the Windows device name and Tailscale IP in your private lab notes (not a public repository).

## 5. Install on Ubuntu or Kali
On a trusted Ubuntu or Kali lab machine with outbound internet access, follow the **official distribution-specific instructions** at https://tailscale.com/download/linux . For Ubuntu/Debian, Tailscale publishes an installation script; review the script and official instructions before executing any downloaded script.

**Example (Ubuntu/Debian, after reviewing the official installer):**
```bash
curl -fsSL https://tailscale.com/install.sh -o /tmp/tailscale-install.sh
less /tmp/tailscale-install.sh
sudo sh /tmp/tailscale-install.sh
```
Then authenticate:
```bash
sudo tailscale up
```
Follow the browser sign-in URL printed by the command. Do not post the authentication URL or keys publicly.

**Verify:**
```bash
tailscale version
tailscale status
tailscale ip -4
systemctl status tailscaled --no-pager
```
If the machine has no browser, open the displayed sign-in URL on another trusted device.

**Note:** Avoid joining deliberately vulnerable VMs or offensive testing targets to your normal personal tailnet. Use trusted admin devices; preserve lab isolation.

## 6. Connect two devices
1. Ensure Windows and Ubuntu are both authenticated to the **same tailnet**.
2. On Ubuntu run `tailscale ip -4` and note the address privately.
3. On Windows PowerShell run:
```powershell
tailscale status
tailscale ping <ubuntu-tailscale-ip>
```
Replace `<ubuntu-tailscale-ip>` with the address shown by Ubuntu. `tailscale ping` tests Tailscale connectivity; it is different from normal ICMP ping.
4. If Ubuntu runs SSH, test from Windows:
```powershell
ssh your-ubuntu-user@<ubuntu-tailscale-ip>
```
This uses ordinary OpenSSH and requires the SSH service, credentials, firewall allowance, and tailnet access permission. If you use Tailscale SSH instead, follow its separate setup and policy instructions.

**Expected:** Devices appear in `tailscale status`; a successful ping indicates tailnet reachability. If SSH fails despite ping success, inspect the SSH service and permissions.

## 7. MagicDNS and SSH
1. In Tailscale Admin → **DNS**, find and enable **MagicDNS** if desired.
2. Check your device's MagicDNS hostname in the admin console.
3. From another tailnet device, test the hostname:
```bash
tailscale ping ubuntu-lab01
```
Replace with your actual node name. Depending on naming and DNS settings, use the full MagicDNS name if the short name does not resolve.
4. For ordinary SSH, use `ssh user@hostname` after verifying OpenSSH is installed and access policies allow it.

**Tailscale SSH** is an optional feature that can manage SSH access to supported Linux nodes using Tailscale identity and policy. It is distinct from installing `openssh-server`. Read https://tailscale.com/kb/1193/tailscale-ssh before enabling it, and use least-privilege policy.

## 8. Proxmox remote management
**Recommended architecture:** Enroll the **trusted Proxmox host** or a dedicated trusted management jump box; do not expose Proxmox port 8006 on your home router. Understand that installing extra software on a hypervisor adds maintenance and security considerations.

### Option A: Tailscale on the Proxmox host
1. Confirm you have console or local LAN access in case remote connectivity breaks.
2. Back up configuration and confirm current management connectivity.
3. From Proxmox node → **Shell**, use the official Debian/Linux Tailscale installation instructions (Proxmox is Debian-based; check version compatibility).
4. Authenticate with `sudo tailscale up` and approve the node as required.
5. Run:
```bash
tailscale ip -4
tailscale status
```
6. From a trusted remote Windows device in the same tailnet, browse to:
```text
https://<proxmox-tailscale-ip>:8006
```
Replace the placeholder with the Proxmox host's actual Tailscale IP. Verify the server identity before proceeding past any certificate warning.
7. Log in with Proxmox credentials and MFA where configured. Keep Proxmox firewall and tailnet access rules restrictive.

### Option B: Dedicated management jump box
Instead of installing Tailscale directly on the host, enroll a hardened Linux management VM on the LAN, restrict its access, and use it to reach Proxmox. For direct remote access to LAN IPs, configure a subnet router as described below; do not accidentally expose the entire LAN to every tailnet member.

## 9. Subnet routing
A **subnet router** allows authorized tailnet devices to reach a network behind a Tailscale node, even when those LAN devices do not run Tailscale. This is powerful and must be scoped carefully.

**Example:** Trusted Linux router on `192.168.56.0/24` lab subnet. Do **not** use this to bridge a vulnerable security lab to your normal personal tailnet.

### Steps on a dedicated trusted Linux router
1. Install Tailscale on the router and confirm the intended subnet is actually reachable from it.
2. Enable IPv4 forwarding:
```bash
sudo sysctl -w net.ipv4.ip_forward=1
```
For persistence, create a dedicated file such as `/etc/sysctl.d/99-tailscale-forward.conf` containing `net.ipv4.ip_forward = 1`, then apply with `sudo sysctl --system`.
3. Advertise only the specific subnet:
```bash
sudo tailscale set --advertise-routes=192.168.56.0/24
```
4. In the Tailscale admin console, **approve** the advertised route if your policy requires it.
5. Add appropriate tailnet access rules to restrict which users/devices can reach which hosts/ports. Confirm Linux firewall/routing also permits the intended traffic.
6. On Linux clients that need the route, accept routes as appropriate:
```bash
sudo tailscale set --accept-routes=true
```
Windows and other clients may handle accepted routes differently.
7. Test a single approved service, such as SSH to a lab server, and confirm unrelated networks are inaccessible.

**Routing caveats:** The subnet router must have a path to the target network. Address overlap, host firewall rules, source NAT behavior, and return routes can affect connectivity. Refer to https://tailscale.com/kb/1019/subnets .

## 10. Access controls and device security
1. Open Admin → **Access controls** (or policy editor).
2. Review the default policy; avoid assuming a new tailnet is restricted by default.
3. Define only necessary access (e.g., admin laptop to Proxmox TCP 8006 and SSH TCP 22).
4. Validate policy with built-in tests or policy preview where available before applying.
5. Review device approval, key expiry, MFA, and admin roles.
6. Remove a retired device in the admin console and verify access stops.

**Important:** Tailscale access policy syntax evolves; use the current grants/ACL documentation and validate JSON/huJSON before saving: https://tailscale.com/kb/1337/policy-syntax . Avoid copying an overly permissive `*:*` rule into a production environment.

## 11. Exit nodes
An **exit node** routes a client's general internet traffic through another authorized tailnet device. It is different from a subnet router, which routes specific subnets.

1. On a trusted Linux node, review official instructions and enable IP forwarding as required.
2. Advertise exit-node capability:
```bash
sudo tailscale set --advertise-exit-node=true
```
3. Approve the exit node in the admin console if required.
4. On a trusted client, select the exit node in the Tailscale UI (or follow current CLI instructions).
5. Confirm external egress IP and DNS behavior; then disable exit-node selection when finished.

**Caution:** Exit nodes can affect privacy, throughput, DNS, and access to local LAN devices. Do not route traffic through a device you do not trust.

## 12. Troubleshooting and capstone
### Diagnostics
**Linux:**
```bash
tailscale status
tailscale ip -4
tailscale netcheck
systemctl status tailscaled --no-pager
journalctl -u tailscaled -n 50 --no-pager
```
**Windows PowerShell:**
```powershell
tailscale status
tailscale netcheck
```
`netcheck` provides information about connectivity/NAT conditions; relay use does not necessarily mean a connection is broken.

| Symptom | Check |
|---|---|
| Device not shown | Signed into correct tailnet, daemon running, approval/key status |
| `tailscale ping` fails | Access policy, device online, firewall, sign-in |
| Ping works but SSH fails | SSH daemon, port 22, account, firewall, access policy |
| Proxmox UI unreachable | Port 8006, host Tailscale IP, pveproxy, firewall |
| Subnet unreachable | Route advertisement/approval, accept-routes, forwarding, return path |
| MagicDNS fails | DNS settings, correct node name, resolver configuration |

### Capstone checklist
- [ ] Explain tailnet, node, Tailscale IP, MagicDNS, subnet router, exit node.
- [ ] Install Tailscale on Windows and Ubuntu.
- [ ] Confirm both devices in the admin console.
- [ ] Test `tailscale ping` and approved SSH access.
- [ ] Enable and test MagicDNS.
- [ ] Access Proxmox management securely over Tailscale from a trusted device.
- [ ] Restrict management access with tailnet policies and host firewall.
- [ ] Document and test device removal/revocation.
- [ ] Optional: advertise one approved subnet and test limited access.
- [ ] Optional: test an exit node and disable it afterward.

## Official references
- https://tailscale.com/download
- https://tailscale.com/kb/
- https://tailscale.com/kb/1019/subnets
- https://tailscale.com/kb/1193/tailscale-ssh
- https://tailscale.com/kb/1337/policy-syntax
- https://pve.proxmox.com/pve-docs/
