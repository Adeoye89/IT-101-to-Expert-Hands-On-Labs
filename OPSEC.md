# OPSEC 101 to Advanced — Operations Security for IT and Cybersecurity

> GitHub-ready hands-on guide | Audience: beginner to advanced | Scope: authorized systems and defensive operations

## Learning objectives

By the end, you can explain the five-step OPSEC process, identify critical information, assess threats and vulnerabilities, build an exposure inventory, implement safeguards on Windows/Linux/Proxmox/pfSense, and verify controls without disrupting production.

## 1. What is OPSEC?

**Operations Security (OPSEC)** is a disciplined process for preventing sensitive information about your operations from being collected and exploited. It covers people, processes, physical environments, and technology. OPSEC complements cybersecurity: a patched server can still be exposed through a publicly posted network diagram, leaked credential, or overly broad access permission.

**Example:** A screenshot posted in a public repository shows the Proxmox host's public IP, usernames, and internal VLAN map. An OPSEC review identifies those disclosures and establishes screenshot-redaction and repository-review procedures.

### Five steps

1. **Identify critical information:** What must an unauthorized party not learn? Examples: secrets, access methods, internal architecture, client data, recovery codes.
2. **Analyze threats:** Who might seek it, and what capabilities or motives might they have? Avoid assuming every visitor is malicious.
3. **Analyze vulnerabilities:** Where could information leak? Public repositories, logs, shared drives, screenshots, backups, misconfigured services.
4. **Assess risk:** Evaluate likelihood and impact; prioritize the highest-risk combinations.
5. **Apply countermeasures:** Reduce exposure, strengthen controls, assign owners, and verify effectiveness.

## 2. Build a safe practice lab

Use your existing Proxmox VE 9.x environment. Keep management separate from deliberately vulnerable VMs. Example **documentation-only** lab addresses:

| Segment | Example subnet | Purpose |
|---|---|---|
| Management | 10.10.10.0/24 | Proxmox and administrative access |
| Servers | 10.10.20.0/24 | Windows Server and Ubuntu |
| Workstations | 10.10.30.0/24 | Windows client |
| Security exercises | 10.10.40.0/24 | Isolated training targets |
| Monitoring | 10.10.50.0/24 | Logging and alerts |

These are illustrative; adapt them to avoid overlap with your actual network. Prefer isolated bridges for vulnerable systems, with no internet or production LAN access. Do not expose Proxmox or pfSense management to the public internet.

## 3. Critical-information inventory

Create `critical-information.csv`:

```csv
Asset,Information,Exposure,Impact,Owner,Control
Proxmox,Admin credentials,Password manager,High,Lab admin,MFA and least privilege
GitHub,Infrastructure screenshots,Public repository,Medium,Repository owner,Redaction review
pfSense,Firewall configuration,Encrypted backup,High,Network admin,Restricted backup access
Windows Server,Domain recovery credentials,Offline vault,High,Domain admin,Protected recovery process
```

**Exercise:** Identify five more items, including backup keys, API tokens, SSH private keys, and client records. Never store actual secrets in this inventory.

## 4. Risk assessment worksheet

Use a simple 1–5 likelihood and impact scale; multiply for a *relative* priority score, not a precise probability.

| Finding | Likelihood | Impact | Score | First action |
|---|---:|---:|---:|---|
| Publicly shared password | 5 | 5 | 25 | Revoke/rotate immediately |
| Proxmox GUI reachable from internet | 4 | 5 | 20 | Restrict ingress |
| Unredacted internal diagram | 3 | 3 | 9 | Remove and review |
| Unused service enabled | 3 | 3 | 9 | Disable after dependency check |

**Verification:** Every high-priority finding should have an owner, due date, change record, and retest result.

## 5. Windows OPSEC checks

Run PowerShell on systems you administer:

```powershell
# Review listening TCP ports
Get-NetTCPConnection -State Listen | Sort-Object LocalPort |
  Select-Object LocalAddress,LocalPort,OwningProcess

# Review enabled local accounts
Get-LocalUser | Select-Object Name,Enabled,LastLogon

# Review firewall profiles
Get-NetFirewallProfile | Select-Object Name,Enabled,DefaultInboundAction

# Review RDP setting (1 = disabled)
Get-ItemProperty 'HKLM:\System\CurrentControlSet\Control\Terminal Server' |
  Select-Object fDenyTSConnections
```

**Interpretation:** Listening does not necessarily mean internet-exposed. Check host firewall, upstream pfSense rules, and NAT. Review findings before making changes.

**GUI:** Windows Security → Firewall & network protection; Computer Management → Local Users and Groups (availability varies by edition); Event Viewer → Windows Logs → Security.

## 6. Linux OPSEC checks

```bash
# Listening ports and processes
sudo ss -lntup

# Network addresses and routes
ip -br addr
ip route

# Accounts with interactive shells (review carefully)
getent passwd | grep -E '/(bash|zsh|sh)$'

# SSH daemon effective settings (requires OpenSSH server)
sudo sshd -T | grep -E '^(permitrootlogin|passwordauthentication|pubkeyauthentication) '

# Recent SSH service messages (service name varies)
sudo journalctl -u ssh --since '24 hours ago' --no-pager
```

Do not disable SSH password authentication until a tested alternative administrative path exists. Protect `~/.ssh/id_ed25519`; share only the `.pub` key.

## 7. Proxmox VE management safeguards

1. Open the Proxmox web UI on a trusted management network.
2. Review **Datacenter → Permissions → Users**, **Groups**, and **Roles**; use least privilege.
3. Configure supported second-factor authentication for administrator accounts.
4. Review **Node → System → Network** for unexpected management paths.
5. Review **Datacenter/Node → Firewall** rules; test from an alternate console before enforcing restrictions.
6. Keep backups encrypted or access-restricted; test restore operations.
7. Review API tokens and remove unused credentials.

Read-only CLI examples:

```bash
pveversion
ip -br addr
ip route
ss -lntup
qm list
pct list
```

**Success criteria:** Only approved administration devices can reach the management UI; nonmanagement lab segments cannot.

## 8. pfSense OPSEC safeguards

- Use **Firewall → Rules** to restrict management access to trusted administrative sources.
- Review **Firewall → NAT** for accidental port forwards.
- Review **System → User Manager** and enable supported MFA for administrative accounts.
- Export configuration backups to a protected location; treat them as sensitive.
- Review **Status → System Logs → Firewall** for blocked and allowed traffic patterns.
- Ensure intentionally vulnerable target networks have no uncontrolled WAN access.

**Verification:** From an authorized workstation, test required access. From an isolated target VM, confirm management access is blocked. A successful ping alone is not sufficient evidence of application access.

## 9. SSH, keys, and remote access

On Windows:

```powershell
ssh-keygen -t ed25519 -C 'lab-admin'
ssh -v student@10.10.20.10
```

On Linux, verify public-key file permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Use a VPN such as Tailscale for approved remote access rather than opening SSH/RDP to the entire internet. Follow your organization's policy for key storage and device approval.

## 10. Repository and screenshot hygiene

Before publishing a GitHub README or screenshot:

- Remove passwords, private keys, tokens, recovery codes, session cookies, and client data.
- Replace real public IPs, domain names, and identifiable hostnames with examples.
- Review terminal history and browser tabs visible in screenshots.
- Inspect `.env`, configuration exports, and automation scripts.
- If a secret was committed, **revoke and rotate it**; deleting a commit alone does not make it safe.

Example `.gitignore` additions:

```gitignore
.env
.env.*
*.pem
*.key
id_ed25519
*.p12
*.pfx
```

`.gitignore` prevents some accidental additions but is **not** a secret scanner and does not remove files already tracked.

## 11. Logs and privacy

Log administrative changes, authentication events, and firewall decisions, but minimize sensitive contents. Establish retention limits and access controls. Do not put passwords or authentication tokens into troubleshooting tickets or screenshots.

## 12. Incident response when OPSEC fails

1. Confirm what was exposed and when.
2. Contain exposure: restrict access or remove the public artifact.
3. Revoke compromised tokens/credentials and rotate secrets.
4. Review logs for suspicious use; preserve relevant evidence.
5. Restore safe access and verify controls.
6. Document the incident, notify appropriate owners, and improve procedures.

For a leaked SSH private key, create a replacement key pair, deploy the new public key, and remove the old key from `authorized_keys` on affected systems.

## 13. Practical labs

### Lab 1 — OPSEC inventory
Create a ten-item critical-information inventory for your Proxmox range. **Pass:** Every item has an owner and safeguard.

### Lab 2 — Windows exposure review
Run the read-only PowerShell commands above. Explain three listening ports and whether each is reachable from another VLAN. **Pass:** Findings are validated against firewall rules.

### Lab 3 — Linux SSH review
Inspect SSH configuration, key permissions, and recent logs. **Pass:** You identify approved authentication methods without losing remote access.

### Lab 4 — Proxmox access review
Review users, roles, API tokens, and management paths. **Pass:** An untrusted target VM cannot access the management interface.

### Lab 5 — pfSense rule audit
Review rules and NAT; document source, destination, port, action, and business justification. **Pass:** No unexplained broad management allow rules remain.

### Lab 6 — GitHub publication review
Create a sample diagram using fictitious addresses, scan a sample README for sensitive details, and verify screenshots are redacted. **Pass:** No real credentials or sensitive identifiers appear.

### Lab 7 — Tabletop incident
Pretend a lab API token was accidentally committed. Write a timeline, containment plan, rotation steps, and validation checklist. **Pass:** The response addresses both public exposure and potential reuse.

## 14. Reusable OPSEC review checklist

- [ ] Critical information identified and classified
- [ ] Threats and vulnerabilities documented
- [ ] Risk owners and priorities assigned
- [ ] MFA and least-privilege access reviewed
- [ ] Management networks segmented
- [ ] Firewall/NAT exposure verified
- [ ] SSH keys and secrets protected
- [ ] Logs retained securely
- [ ] Backups protected and tested
- [ ] Public documentation and screenshots sanitized
- [ ] Incident response procedure rehearsed
- [ ] Controls retested after changes

## 15. Further reading

- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [CISA Cybersecurity Resources](https://www.cisa.gov/resources-tools)
- [MITRE ATT&CK](https://attack.mitre.org/)
- [Proxmox VE Documentation](https://pve.proxmox.com/pve-docs/)
- [pfSense Documentation](https://docs.netgate.com/pfsense/en/latest/)
- [Microsoft Windows Security Documentation](https://learn.microsoft.com/windows/security/)

## Suggested repository location

`cybersecurity/opsec/README.md`

**Reminder:** OPSEC is a continuous review process, not a one-time configuration task. Perform tests only on systems you own or are authorized to administer.
