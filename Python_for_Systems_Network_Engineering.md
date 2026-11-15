# Python for Systems, Networking & Infrastructure Engineering

**IT 101 → Expert | Hands-on GitHub training guide**

> Scope: authorized systems and networks only. Start with read-only automation, keep credentials out of code, and test changes in your isolated Proxmox cyber range before touching production.

## Table of contents
1. [What Python engineers automate](#1-what-python-engineers-automate)
2. [Lab topology and prerequisites](#2-lab-topology-and-prerequisites)
3. [Install Python and build a project](#3-install-python-and-build-a-project)
4. [Python fundamentals](#4-python-fundamentals)
5. [Files, JSON, CSV and YAML](#5-files-json-csv-and-yaml)
6. [Operating-system and process automation](#6-operating-system-and-process-automation)
7. [IP addresses, subnetting and DNS](#7-ip-addresses-subnetting-and-dns)
8. [TCP connectivity and service checks](#8-tcp-connectivity-and-service-checks)
9. [SSH automation for Linux and network devices](#9-ssh-automation-for-linux-and-network-devices)
10. [Network-device automation with Netmiko](#10-network-device-automation-with-netmiko)
11. [REST APIs, pfSense and Proxmox](#11-rest-apis-pfsense-and-proxmox)
12. [Windows and Active Directory integration](#12-windows-and-active-directory-integration)
13. [Logging, exceptions, retries and tests](#13-logging-exceptions-retries-and-tests)
14. [Monitoring and scheduled execution](#14-monitoring-and-scheduled-execution)
15. [Security and production engineering practices](#15-security-and-production-engineering-practices)
16. [Hands-on labs](#16-hands-on-labs)
17. [Capstone project](#17-capstone-project)
18. [Troubleshooting and command reference](#18-troubleshooting-and-command-reference)
19. [Completion checklist and further learning](#19-completion-checklist-and-further-learning)

---

## 1. What Python engineers automate

| Discipline | Examples | Typical libraries |
|---|---|---|
| Systems engineering | Inventory, service checks, disk usage, log parsing | `pathlib`, `subprocess`, `psutil` |
| Networking | IP/subnet validation, DNS, TCP checks | `ipaddress`, `socket`, `dnspython` |
| Network engineering | SSH show commands, configuration backups | `netmiko`, `paramiko` |
| Infrastructure | Proxmox API, VM inventory, health checks | `requests`, `proxmoxer` |
| Windows administration | WMI/CIM, WinRM, PowerShell orchestration | `pywinrm`, PowerShell |
| Reliability | Monitoring, alerts, reports, automated tests | `logging`, `pytest`, `schedule` |

**Key distinction:** Python is a programming language; Bash and PowerShell are also shells and scripting environments. Choose the tool that best matches the job. For example, use PowerShell cmdlets to manage Active Directory and Python to combine reports from Linux, Windows, and network APIs.

## 2. Lab topology and prerequisites

Example isolated training environment (replace addresses with yours):

```text
Proxmox VE 9.x (management network: 192.168.1.0/24)
  |
  +-- pfSense VM: 10.10.10.1 / 10.10.20.1 / 10.10.30.1
  |     +-- vmbr10: Engineering workstation (Ubuntu or Kali) 10.10.10.10
  |     +-- vmbr20: Ubuntu target 10.10.20.10; Windows Server DC01 10.10.20.20
  |     +-- vmbr30: Monitoring VM 10.10.30.10
  +-- virtual networking and controlled firewall rules
```

**Requirements:** Python 3.11+ recommended, a code editor (VS Code optional), terminal access, a standard non-administrator user, and permission to administer the lab. The exercise subnet addresses are examples, not automatic defaults.

**Safety:** Avoid broad scanning, unapproved changes, and connecting intentionally vulnerable targets to production. Use pfSense to permit only the flows required for each lab.

## 3. Install Python and build a project

### Windows 11

Install Python from [python.org](https://www.python.org/downloads/) or your organization's approved package manager. In PowerShell:

```powershell
py -V
py -m pip --version
mkdir python-infra-lab
cd python-infra-lab
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
```

If activation is restricted by policy, run `.\.venv\Scripts\python.exe` directly rather than weakening system-wide execution policy.

### Ubuntu / Kali

```bash
sudo apt update
sudo apt install -y python3 python3-venv python3-pip
mkdir -p ~/python-infra-lab && cd ~/python-infra-lab
python3 -m venv .venv
source .venv/bin/activate
python --version
python -m pip --version
```

### Recommended repository structure

```text
python-infra-lab/
├── README.md
├── requirements.txt
├── .gitignore
├── inventory/
│   └── devices.csv
├── scripts/
│   ├── host_inventory.py
│   ├── subnet_tool.py
│   ├── tcp_check.py
│   └── ssh_readonly.py
├── tests/
│   └── test_subnets.py
└── output/              # generated reports; do not commit secrets
```

Create `.gitignore`:

```gitignore
.venv/
__pycache__/
.pytest_cache/
.env
output/
*.log
```

## 4. Python fundamentals

Create `hello.py`:

```python
hostname = "ubuntu01"
address = "10.10.20.10"
ports = [22, 80, 443]

print(f"Checking {hostname} at {address}")
for port in ports:
    print(f"Candidate TCP port: {port}")
```

Run: `python hello.py`. Expected: a host line followed by three port lines.

### Functions, dictionaries, conditions

```python
def summarize_host(host: dict) -> str:
    status = "configured" if host.get("ip") else "missing IP"
    return f"{host['name']}: {status}"

hosts = [
    {"name": "dc01", "ip": "10.10.20.20"},
    {"name": "app01", "ip": ""},
]
for host in hosts:
    print(summarize_host(host))
```

**Exercise:** Add a `role` field and print only hosts whose role is `server`.

## 5. Files, JSON, CSV and YAML

### CSV inventory

Create `inventory/devices.csv`:

```csv
name,ip,role
ubuntu01,10.10.20.10,linux
DC01,10.10.20.20,windows
monitor01,10.10.30.10,monitoring
```

Read and validate it:

```python
import csv
import ipaddress
from pathlib import Path

with Path("inventory/devices.csv").open(newline="", encoding="utf-8") as file:
    for row in csv.DictReader(file):
        address = ipaddress.ip_address(row["ip"])
        print(row["name"], address, row["role"])
```

### JSON report

```python
import json
from pathlib import Path

report = {"host": "ubuntu01", "healthy": True, "services": ["ssh", "nginx"]}
Path("output").mkdir(exist_ok=True)
Path("output/report.json").write_text(json.dumps(report, indent=2), encoding="utf-8")
```

YAML is common for configuration files. Install `PyYAML` if needed, then use `yaml.safe_load()` for untrusted configuration rather than unsafe object loading.

## 6. Operating-system and process automation

### Cross-platform system inventory

Install: `python -m pip install psutil`.

```python
import platform
import shutil
import psutil

print("Host:", platform.node())
print("OS:", platform.platform())
print("CPU logical cores:", psutil.cpu_count())
print("RAM used %:", psutil.virtual_memory().percent)
print("Root disk used %:", psutil.disk_usage("C:\\" if platform.system() == "Windows" else "/").percent)
print("Python:", platform.python_version())
```

### Running approved OS commands

```python
import subprocess

result = subprocess.run(
    ["hostname"], capture_output=True, text=True, check=True, timeout=10
)
print(result.stdout.strip())
```

**Why list arguments?** Avoid `shell=True` for input-derived commands; it increases command-injection risk. Catch `subprocess.CalledProcessError` and `subprocess.TimeoutExpired` in production scripts.

## 7. IP addresses, subnetting and DNS

### Subnet calculator (`scripts/subnet_tool.py`)

```python
import argparse
import ipaddress

parser = argparse.ArgumentParser(description="Inspect an IPv4 or IPv6 subnet")
parser.add_argument("cidr", help="Example: 10.10.20.0/24")
args = parser.parse_args()

network = ipaddress.ip_network(args.cidr, strict=False)
print("Network:", network.network_address)
print("Prefix:", network.prefixlen)
print("Netmask:", network.netmask)
print("Total addresses:", network.num_addresses)
if network.version == 4:
    print("Broadcast:", network.broadcast_address)
```

Run:

```bash
python scripts/subnet_tool.py 10.10.20.0/24
```

Expected network: `10.10.20.0`, prefix: `24`, netmask: `255.255.255.0`, total addresses: `256`. Usable host counts depend on address family and special subnet rules.

### DNS resolution

```python
import socket

name = "example.com"
try:
    answers = socket.getaddrinfo(name, None, proto=socket.IPPROTO_TCP)
    for address in sorted({item[4][0] for item in answers}):
        print(address)
except socket.gaierror as exc:
    print(f"DNS lookup failed: {exc}")
```

For querying specific DNS record types, use `dnspython` (`python -m pip install dnspython`) and `dns.resolver.resolve("example.com", "MX")`.

## 8. TCP connectivity and service checks

Create `scripts/tcp_check.py`:

```python
import argparse
import socket
import time

parser = argparse.ArgumentParser(description="Check one authorized TCP endpoint")
parser.add_argument("host")
parser.add_argument("port", type=int)
parser.add_argument("--timeout", type=float, default=3)
args = parser.parse_args()

start = time.monotonic()
try:
    with socket.create_connection((args.host, args.port), timeout=args.timeout):
        elapsed = (time.monotonic() - start) * 1000
        print(f"OPEN {args.host}:{args.port} ({elapsed:.1f} ms)")
except (OSError, TimeoutError) as exc:
    print(f"UNREACHABLE {args.host}:{args.port} ({exc})")
    raise SystemExit(1)
```

Run only against your lab:

```bash
python scripts/tcp_check.py 10.10.20.10 22
```

**Interpretation:** A successful TCP handshake confirms reachability of that port, not successful SSH authentication or application health. A timeout may indicate filtering, routing, or host availability issues.

## 9. SSH automation for Linux and network devices

Install: `python -m pip install paramiko`.

**Secure default:** Prefer key-based authentication, verify server host keys, and never automatically accept unknown keys in production.

```python
import os
import paramiko

host = os.environ.get("LAB_SSH_HOST", "10.10.20.10")
username = os.environ.get("LAB_SSH_USER", "student")

client = paramiko.SSHClient()
client.load_system_host_keys()
client.set_missing_host_key_policy(paramiko.RejectPolicy())
try:
    client.connect(hostname=host, username=username, timeout=10, look_for_keys=True)
    stdin, stdout, stderr = client.exec_command("hostname && uptime", timeout=10)
    print(stdout.read().decode())
    errors = stderr.read().decode()
    if errors:
        print("stderr:", errors)
finally:
    client.close()
```

Before running, verify and store the server's host key using an approved process. If your lab has a nonstandard key location, configure it explicitly. Never embed a password or private key in a Git repository.

## 10. Network-device automation with Netmiko

Install: `python -m pip install netmiko`.

Example **read-only** Cisco IOS command collection (use a lab switch or emulator):

```python
import getpass
from netmiko import ConnectHandler

password = getpass.getpass("Lab switch password: ")
connection = ConnectHandler(
    device_type="cisco_ios",
    host="10.10.10.50",
    username="labadmin",
    password=password,
    conn_timeout=10,
)
try:
    print(connection.send_command("show version"))
    print(connection.send_command("show vlan brief"))
finally:
    connection.disconnect()
```

**Note:** Netmiko is convenient for training but device SSH host-key verification settings need deliberate review; enforce trusted host keys for production. Avoid using configuration-mode commands until you have a change plan, backup, and rollback.

**Practice:** Save sanitized `show interfaces status`, `show ip interface brief`, and `show vlan brief` output to timestamped text files. Remove credentials and sensitive network details before sharing.

## 11. REST APIs, pfSense and Proxmox

An API lets software communicate with an infrastructure platform using structured requests. Common methods: `GET` (read), `POST` (create/action), `PUT` (replace), `PATCH` (partial update), `DELETE` (remove).

Install: `python -m pip install requests`.

### Generic authenticated API request

```python
import os
import requests

url = os.environ["LAB_API_URL"]
token = os.environ["LAB_API_TOKEN"]
response = requests.get(
    url,
    headers={"Authorization": f"Bearer {token}"},
    timeout=10,
)
response.raise_for_status()
print(response.json())
```

### Proxmox VE read-only VM inventory

Proxmox uses API tokens in the form `PVEAPIToken=USER@REALM!TOKENID=SECRET` (the header syntax differs from generic Bearer tokens).

```python
import os
import requests

host = os.environ["PVE_HOST"]  # e.g. pve01.lab.example.test
user_token = os.environ["PVE_TOKEN_ID"]  # e.g. auditor@pve!inventory
secret = os.environ["PVE_TOKEN_SECRET"]
url = f"https://{host}:8006/api2/json/cluster/resources"
headers = {"Authorization": f"PVEAPIToken={user_token}={secret}"}

# Use a valid certificate or your organization's trusted CA bundle.
response = requests.get(url, headers=headers, params={"type": "vm"}, timeout=15)
response.raise_for_status()
for item in response.json()["data"]:
    print(item.get("vmid"), item.get("name"), item.get("status"), item.get("node"))
```

Use a dedicated **least-privilege** API token. Do not disable TLS verification to make certificate errors disappear. Review the current Proxmox API Viewer for your installed release: `https://YOUR-PROXMOX:8006/pve-docs/api-viewer/index.html`.

### pfSense automation

pfSense API support varies by edition, release, and installed packages. Do **not** assume every pfSense installation includes the same REST API. Confirm supported APIs in the official documentation for your exact version. For beginner labs, use pfSense's web UI and export a configuration backup rather than writing firewall rules through an unverified API.

## 12. Windows and Active Directory integration

For Windows-specific administration, PowerShell remains the primary supported tool. Python can launch approved PowerShell scripts locally or use a properly configured remoting interface.

Example **local Windows** service inventory:

```python
import json
import subprocess

command = [
    "powershell.exe", "-NoProfile", "-Command",
    "Get-Service | Select-Object -First 5 Name,Status | ConvertTo-Json"
]
result = subprocess.run(command, capture_output=True, text=True, check=True, timeout=30)
print(json.loads(result.stdout))
```

For Active Directory, use an authorized admin workstation with RSAT and the `ActiveDirectory` PowerShell module. Start with read-only commands such as `Get-ADUser` and `Get-ADComputer`. Avoid placing privileged domain credentials in scripts.

## 13. Logging, exceptions, retries and tests

### Structured logs

```python
import logging

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")
logging.info("Starting inventory collection")
try:
    value = int("42")
    logging.info("Parsed value=%d", value)
except ValueError:
    logging.exception("Invalid numeric value")
```

### Retry only safe operations

```python
import time
import requests

for attempt in range(3):
    try:
        response = requests.get("https://example.com", timeout=5)
        response.raise_for_status()
        break
    except requests.RequestException:
        if attempt == 2:
            raise
        time.sleep(2 ** attempt)
```

Only retry requests that are safe to repeat. Do not blindly retry VM creation, account changes, or other state-changing actions.

### Unit test

Install: `python -m pip install pytest`.

```python
# tests/test_subnets.py
import ipaddress

def test_lab_host_is_in_target_network():
    network = ipaddress.ip_network("10.10.20.0/24")
    assert ipaddress.ip_address("10.10.20.10") in network
```

Run: `python -m pytest -q`.

## 14. Monitoring and scheduled execution

Good monitoring tracks a baseline, thresholds, timestamps, failures, and trends—not just whether a host replies to ping.

### Example metrics
- CPU and RAM utilization
- Disk capacity and filesystem errors
- SSH/HTTPS service reachability
- DNS lookup time
- Proxmox VM status and backup success
- Network interface errors and packet loss

### Scheduling

Linux cron example (adjust paths and timezone):

```cron
*/15 * * * * /home/student/python-infra-lab/.venv/bin/python /home/student/python-infra-lab/scripts/host_inventory.py >> /home/student/python-infra-lab/inventory.log 2>&1
```

On Windows, use Task Scheduler with the virtual environment's `python.exe` and an explicit working directory. Use dedicated service identities and restricted file permissions for automation.

## 15. Security and production engineering practices

1. **Inventory and scope:** List only devices you own or are authorized to manage.
2. **Secrets:** Use environment variables or a managed secret store; never commit `.env`, passwords, SSH keys, or API tokens.
3. **Least privilege:** Use read-only credentials for discovery and separate credentials for approved changes.
4. **TLS and SSH trust:** Verify certificates and host keys. Avoid `verify=False` and unknown-host-key auto-acceptance.
5. **Idempotency:** Design configuration automation so reruns do not produce unintended changes.
6. **Change control:** Document expected impact, maintenance window, backup, validation, and rollback.
7. **Error handling:** Use timeouts, structured logs, bounded retries, and clear nonzero exit codes.
8. **Testing:** Use unit tests, a staging environment, and small pilot deployments.
9. **Dependencies:** Pin tested package versions in deployed environments and review vulnerability advisories.
10. **Privacy:** Sanitize reports before committing or sharing them.

## 16. Hands-on labs

### Lab 1 — Python installation
- Install Python and create a virtual environment.
- Run `hello.py` and confirm the version.
- **Evidence:** Screenshot of version and output.

### Lab 2 — Build a CSV inventory
- Add at least three authorized VMs to `devices.csv`.
- Validate IP addresses using `ipaddress`.
- **Success:** Invalid addresses cause a clear error.

### Lab 3 — Subnet engineering
- Calculate `10.10.10.0/24`, `10.10.20.0/24`, and `10.10.30.0/24`.
- Explain why they do not overlap.
- **Success:** Correct network/prefix/netmask results.

### Lab 4 — TCP troubleshooting
- Test SSH on Ubuntu and HTTPS on an authorized lab web server.
- Stop the SSH service temporarily from the VM console, retest, and restore it.
- **Success:** Script distinguishes successful connection from failure.

### Lab 5 — Linux health inventory
- Collect hostname, OS, RAM usage, and disk usage.
- Write a JSON report to `output/`.
- **Success:** Report parses with `python -m json.tool output/report.json`.

### Lab 6 — Read-only SSH automation
- Establish trusted SSH keys and execute `hostname && uptime`.
- Test behavior when the host is unavailable.
- **Success:** No hard-coded passwords or disabled host-key checks.

### Lab 7 — Cisco switch inventory
- Use a Cisco lab switch or emulator and collect `show version` / `show vlan brief`.
- **Success:** Output is saved with device name and timestamp.

### Lab 8 — Proxmox API inventory
- Create a read-only API token and fetch VM state.
- Compare the output with `qm list` and the web UI.
- **Success:** No credentials appear in terminal logs or Git history.

### Lab 9 — Windows service report
- Execute a read-only PowerShell query from Python on Windows.
- Export and parse JSON.
- **Success:** The script handles an unavailable command gracefully.

### Lab 10 — Automated monitoring
- Schedule an approved endpoint check every 15 minutes.
- Record timestamps, results, and failures.
- **Success:** At least four scheduled checks and a readable report.

## 17. Capstone project

**Build a Python Infrastructure Health Dashboard (CLI version).**

Inputs: CSV device inventory, Proxmox API read-only token, approved SSH keys, and configured service ports.

Outputs: `output/health.json`, `output/health.csv`, a concise terminal summary, and a troubleshooting runbook.

Recommended phases:

1. Validate inventory and detect duplicate IPs.
2. Check only explicitly listed TCP endpoints.
3. Gather Linux health over SSH and Windows health through approved PowerShell methods.
4. Query Proxmox for VM state.
5. Record timestamps and errors without exposing secrets.
6. Compare results with thresholds and label `healthy`, `warning`, or `unreachable`.
7. Add unit tests and demonstrate a failed-service scenario.
8. Document recovery and run the project through a scheduler.

**Acceptance criteria:** repeatable setup, no embedded secrets, verified host identities, clear error messages, reproducible results, and a README explaining how to run it.

## 18. Troubleshooting and command reference

| Symptom | Likely cause | Action |
|---|---|---|
| `python` not found | Missing installation or PATH | Try `py -V` on Windows or `python3 -V` on Linux |
| `ModuleNotFoundError` | Wrong environment | Activate `.venv`, then `python -m pip install PACKAGE` |
| `Permission denied` | Insufficient OS rights | Check user permissions; don't default to admin/root |
| `Connection refused` | Port closed/service down | Verify listener and firewall |
| `Connection timed out` | Routing/filtering/host issue | Check IP, gateway, pfSense rules, and host status |
| SSH host-key failure | Unknown or changed identity | Verify fingerprint out of band before trusting |
| TLS certificate failure | CA or hostname mismatch | Fix certificate trust; do not disable verification |
| API `401` | Missing/invalid authentication | Check token ID, secret, expiry |
| API `403` | Insufficient permissions | Review least-privilege role permissions |
| API `404` | Wrong path/unsupported endpoint | Check exact product version/API docs |
| JSON parsing error | Response not JSON | Inspect status/content type and handle error response |
| Scheduled script fails | Different working directory/env | Use absolute paths and explicit interpreter |

Useful commands:

```bash
python --version
python -m pip list
python -m pip freeze > requirements.txt
python -m pytest -q
python -m compileall scripts
```

## 19. Completion checklist and further learning

- [ ] Install Python and use virtual environments on Windows and Linux.
- [ ] Explain lists, dictionaries, functions, loops, exceptions, and modules.
- [ ] Read CSV and write JSON reports.
- [ ] Validate subnets and troubleshoot DNS/TCP connections.
- [ ] Automate read-only SSH and network-device inventory.
- [ ] Query a Proxmox API using least-privilege credentials.
- [ ] Integrate Python with PowerShell for Windows administration.
- [ ] Add logging, timeouts, tests, and scheduling.
- [ ] Demonstrate safe failure handling and recovery.
- [ ] Complete the infrastructure health capstone.

### Official documentation

- [Python tutorial](https://docs.python.org/3/tutorial/)
- [Python standard library](https://docs.python.org/3/library/)
- [Python `ipaddress`](https://docs.python.org/3/library/ipaddress.html)
- [Python `socket`](https://docs.python.org/3/library/socket.html)
- [Python `subprocess`](https://docs.python.org/3/library/subprocess.html)
- [Python `venv`](https://docs.python.org/3/library/venv.html)
- [Proxmox VE API](https://pve.proxmox.com/wiki/Proxmox_VE_API)
- [Paramiko documentation](https://docs.paramiko.org/)
- [Netmiko documentation](https://ktbyers.github.io/netmiko/)
- [pytest documentation](https://docs.pytest.org/)
- [Microsoft PowerShell documentation](https://learn.microsoft.com/powershell/)

---

**Suggested GitHub path:** `scripting/python-systems-network-engineering/README.md`

**Next modules:** Python network automation with Nornir and Ansible; SNMP and Syslog monitoring; Infrastructure as Code; CI/CD for network configuration validation.
