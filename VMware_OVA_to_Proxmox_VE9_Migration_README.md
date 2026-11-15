# VMware OVA → Proxmox VE 9.x: Hands-On Migration Runbook

> **Audience:** IT administrators, homelab operators, and virtualization engineers. **Scope:** offline OVA/OVF/VMDK migrations to Proxmox VE 9.x, including storage, firmware, disks, networks/VLANs, scripts, guest configuration, validation, and rollback. Commands assume a Proxmox node shell as `root`, unless labeled otherwise.

> **Safety:** Use only VM images you own or are authorized to migrate. Never boot source and migrated production copies on the same network simultaneously. Keep source exports and verified backups until sign-off. Test first with a noncritical VM. For domain controllers, clustered databases, encrypted appliances, and vendor-supported appliances, follow their application-specific migration procedures rather than assuming a generic disk copy is sufficient.

## Table of contents

1. [Key concepts and artifacts](#1-key-concepts-and-artifacts)
2. [Preflight and migration worksheet](#2-preflight-and-migration-worksheet)
3. [Storage preparation](#3-storage-preparation)
4. [Network bridges and VLAN mapping](#4-network-bridges-and-vlan-mapping)
5. [Transfer, inspect, and extract OVA](#5-transfer-inspect-and-extract-ova)
6. [Path A: OVF-assisted import](#6-path-a-ovf-assisted-import)
7. [Path B: manual disk import](#7-path-b-manual-disk-import)
8. [Windows and Linux guest adjustments](#8-windows-and-linux-guest-adjustments)
9. [First boot and validation](#9-first-boot-and-validation)
10. [Troubleshooting matrix](#10-troubleshooting-matrix)
11. [Automation scripts](#11-automation-scripts)
12. [Rollback, backups, and acceptance](#12-rollback-backups-and-acceptance)
13. [Hands-on labs and reference links](#13-hands-on-labs-and-reference-links)

## 1. Key concepts and artifacts

**OVA** is usually a TAR archive containing an **OVF** hardware description, one or more **VMDK** virtual disks, and optional **MF** checksums, **CERT** signatures, **NVRAM** firmware variables, and other vendor files. The OVA is **not** itself a bootable disk. Proxmox VE uses QEMU/KVM and its own VM configuration (`/etc/pve/qemu-server/<VMID>.conf`). An OVF import may translate only some hardware attributes; always inspect and correct the resulting VM.

| VMware item | Meaning | Proxmox handling |
|---|---|---|
| `.ova` | Packed appliance | Extract safely, then import OVF or VMDK |
| `.ovf` | Virtual hardware and disk references | `qm importovf` parses a subset; verify result |
| `.vmdk` | Disk descriptor/data; sometimes sparse or split | `qm disk import` or `qm importovf` |
| `*-flat.vmdk` | Raw extent referenced by a descriptor | Keep with matching descriptor; import descriptor, not flat extent alone |
| `.mf` | Hash manifest | Validate when possible; does not configure the VM |
| `.cert` | Optional signature | Verify using your organization's trust process |
| `.nvram` | VMware UEFI variable store | Usually do **not** transplant; create Proxmox EFI disk |
| `.vmx` | VMware configuration (outside OVA often) | Reference for firmware/NIC/controller settings; not directly a Proxmox config |
| `.vmsd`, `.vmsn` | Snapshot metadata/state | Consolidate VMware snapshots before exporting |
| `.log`, `.lck` | VMware logs/locks | Not imported |
| `.iso` | Mounted install/media image | Reattach only if needed |
| vTPM / encrypted VM | Hardware-backed trust/secrets | **Not automatically portable**; plan BitLocker recovery and vendor-specific procedure |

**Disk types:** `raw` is a direct block image; `qcow2` supports copy-on-write features on file storage; LVM-thin and ZFS volumes generally use raw-style block volumes managed by the backend. Do not assume `--format qcow2` works on `local-lvm`.

**Hypervisor hardware differences:** VMware's PVSCSI/LSI controllers and VMXNET3 NICs differ from Proxmox VirtIO SCSI and VirtIO NICs. BIOS vs UEFI, Secure Boot, boot order, disk bus, MAC address, and interface names are common reasons an imported VM fails to boot or network.

## 2. Preflight and migration worksheet

### Step 1 — Inventory the source VM

Record the following **before export**:

```text
VM name:                    app01
Source hypervisor/version:  VMware ESXi / Workstation: ______
Source OS and version:      ______________________________
Guest architecture:        x86_64 / other
Firmware:                  BIOS / UEFI
Secure Boot:               on / off
vTPM / BitLocker:          yes / no (recovery key secured?)
CPU / RAM:                 2 vCPU / 4096 MiB
Disks, order and sizes:    Disk 0 (OS) 40 GiB; Disk 1 (data) 100 GiB
Disk controllers:          LSI / PVSCSI / SATA / NVMe
VMware NIC / MAC:          VMXNET3 / __:__:__:__:__:__
Port group / VLAN:         Servers / 20
IP / prefix / gateway:     192.168.20.50/24 / 192.168.20.1
DNS / hostname:            ______________________________
Applications / services:   ______________________________
Application backups:       date / restore test / owner
Downtime window:           ______________________________
Rollback decision time:    ______________________________
Proxmox node / VMID:       pve01 / 120
Proxmox storage:           local-lvm
Proxmox bridge / tag:      vmbr0 / 20
```

### Step 2 — Prepare source guest

1. Make an application-consistent backup and verify restore procedures.
2. Consolidate/delete VMware snapshots **on the source** before export (do not delete production data).
3. Record BIOS/UEFI and Secure Boot status, all NICs, MACs, IP settings, and disk order.
4. For Windows, save BitLocker recovery keys; suspend BitLocker only under your organization's change plan. Consider installing the VirtIO driver package while still on VMware.
5. For Linux, record `lsblk -f`, `/etc/fstab`, `ip -br a`, `ip route`, and boot mode (`test -d /sys/firmware/efi && echo UEFI || echo BIOS`).
6. Shut down the guest cleanly before making the final offline export.

### Step 3 — Verify Proxmox VE 9.x host

```bash
pveversion
qm list
pvesm status
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS
df -h
ip -br addr
ip route
```

**Explanation:** `pveversion` identifies Proxmox; `qm list` finds occupied VMIDs; `pvesm status` shows storage IDs, capacity and state; `df` checks file-system staging space; `ip` checks bridges and routes. **Do not confuse free space on `local` with free space on `local-lvm`.**

Plan space for the original OVA, extracted VMDKs, imported destination disks, and backups. Thick provisioning, conversion, and multiple disks may require substantially more space than the compressed OVA size. Keep the extraction area off a small root filesystem when possible.

## 3. Storage preparation

### Step 1 — Understand Proxmox storage choices

| Storage | Typical purpose | Notes |
|---|---|---|
| `local` (directory, often `/var/lib/vz`) | ISO, backups, optional disk images | Check allowed **content types**; root partition may be small |
| `local-lvm` (LVM-thin) | VM disks | No ordinary filesystem directory for uploading VMDKs |
| ZFS pool | VM disks, snapshots, checksums | Check pool capacity and ZFS health |
| Directory on dedicated SSD | Staging and VM disks | Can support raw/qcow2 images when configured for Disk image |
| NFS/Ceph/shared storage | Shared VM disks or backups | Validate permissions, latency, capacity and supported content |

Check storage details:

```bash
pvesm status
pvesm config local
pvesm config local-lvm
```

The target storage must allow **Disk image** content (`images`). The **staging directory** only needs to be writable and large enough; it need not be a configured Proxmox disk-image storage.

### Step 2 — Create a dedicated staging directory (example)

```bash
mkdir -p /mnt/migration-staging/app01
# Confirm this path is on the intended, sufficiently large filesystem:
df -h /mnt/migration-staging/app01
```

If `/mnt/migration-staging` is not a separate mounted volume, it may still reside on `/`; check before copying a large file. Do not blindly format or mount disks with existing data.

### Step 3 — Optional directory-backed VM storage (GUI)

1. Mount an already prepared filesystem, e.g. `/mnt/vm-storage`, persistently using its UUID in `/etc/fstab`.
2. In Proxmox: **Datacenter → Storage → Add → Directory**.
3. ID: `vm-ssd`; Directory: `/mnt/vm-storage`; Content: **Disk image** (and any other content intentionally needed).
4. Verify with `pvesm status` and `pvesm config vm-ssd`.

**Do not create a second filesystem on top of `local-lvm` to upload VMDKs.** Stage them on a normal mounted filesystem, then import to the storage backend.

## 4. Network bridges and VLAN mapping

### Step 1 — Translate VMware port groups

| VMware source | Proxmox target | Meaning |
|---|---|---|
| Port group `Servers`, VLAN 20 | `vmbr0`, VM NIC `tag=20` | Tagged guest traffic on VLAN-aware bridge |
| Port group `Clients`, VLAN 30 | `vmbr0`, VM NIC `tag=30` | Separate client VLAN |
| Isolated lab port group | `vmbr1`, no physical uplink | Lab-only traffic |
| Untagged port group | Bridge with untagged access | Depends on upstream switch/native VLAN |

### Step 2 — Check host bridge and physical trunk

```bash
ip -br link
bridge vlan show
cat /etc/network/interfaces
```

**Example only** — a VLAN-aware bridge using physical NIC `enp3s0`:

```text
auto enp3s0
iface enp3s0 inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.168.10.10/24
    gateway 192.168.10.1
    bridge-ports enp3s0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094
```

This assumes the Proxmox management network is untagged on the upstream port. If management uses a tagged VLAN, the host configuration differs. **Do not paste this over your production network configuration.** Changing the bridge remotely can disconnect your Proxmox host. Have console/out-of-band access and use the Proxmox GUI's **Apply Configuration** or `ifreload -a` only after reviewing the exact change.

**Upstream switch:** VLAN 20 and 30 must be permitted on the trunk connected to `enp3s0`; otherwise the bridge can be configured correctly while packets still fail to reach the network. Do not create VLAN 20 on a physical port configured only as access VLAN 10.

### Step 3 — Attach the imported VM to a VLAN

```bash
qm set 120 --net0 virtio,bridge=vmbr0,tag=20
qm config 120 | grep '^net'
```

**Explanation:** `net0` is the first virtual NIC; `virtio` is the guest-facing adapter model; `bridge=vmbr0` connects to the host bridge; `tag=20` places that NIC in VLAN 20. Add `firewall=1` only when intentionally using and configuring Proxmox firewall rules.

For an isolated lab, create a bridge with `bridge-ports none` and **no physical uplink**; use DHCP inside that lab or static addresses. Never expose an unpatched migrated guest to a production VLAN during first-boot testing.

## 5. Transfer, inspect, and extract OVA

### Step 1 — Transfer from your workstation

On a Linux/macOS shell, or Windows PowerShell with OpenSSH:

```bash
scp ./app01.ova root@192.168.10.10:/mnt/migration-staging/app01/
```

Replace the IP and path with your actual Proxmox node. For large transfers, `rsync -avP` over SSH is useful when available.

### Step 2 — Validate the file before extraction

```bash
cd /mnt/migration-staging/app01
ls -lh app01.ova
sha256sum app01.ova
file app01.ova
tar -tf app01.ova | head -50
```

Compare the SHA-256 digest against a **trusted publisher-provided** checksum when available. Inspect archive paths for unexpected absolute paths, `..`, or symlinks. Extract only trusted OVA files into a dedicated empty directory; archive extraction can overwrite files or traverse directories if maliciously crafted.

### Step 3 — Extract trusted OVA

```bash
mkdir -p extracted
tar -xvf app01.ova -C extracted
find extracted -maxdepth 2 -type f -printf '%p %s bytes\n'
```

Expected files might include:

```text
app01.ovf
app01-disk1.vmdk
app01-disk2.vmdk
app01.mf
app01.nvram
```

### Step 4 — Inspect metadata and disk format

```bash
find extracted -type f \( -iname '*.ovf' -o -iname '*.vmdk' \)
qemu-img info 'extracted/app01-disk1.vmdk'
grep -Ei 'firmware|VirtualSystemType|ResourceType|HostResource|Network' extracted/*.ovf | head -60
```

Use the actual filenames shown by `find`. `qemu-img info` should identify VMDK format, virtual size, and any backing-file information. **If a VMDK uses a descriptor plus `-flat.vmdk` extent, keep both together and reference the descriptor.** A missing extent, chain, or backing image is a stop condition; recover the full export before importing.

Optional: inspect `.mf` checksum entries and verify against the listed files using the matching hashing algorithm. A manifest check catches accidental corruption but does not establish that an untrusted OVA is safe.

## 6. Path A: OVF-assisted import

**Use when:** You have a valid OVF and its referenced VMDKs, and want Proxmox to translate supported VM settings. The result still needs review.

### Step 1 — Choose VMID and target storage

```bash
qm list
pvesm status
```

Assume `120` is unused and `local-lvm` is available.

### Step 2 — Preview the OVF import

```bash
qm importovf 120 /mnt/migration-staging/app01/extracted/app01.ovf local-lvm --dryrun 1
```

Review the parsed settings. If the OVF parser fails, use **Path B** instead. If the command reports that VMID 120 already exists, choose a new unused ID; never overwrite an existing VM.

### Step 3 — Import

```bash
qm importovf 120 /mnt/migration-staging/app01/extracted/app01.ovf local-lvm
qm config 120
```

**Inspect:** disk count and order, CPU, RAM, firmware, SCSI controller, NICs, boot order, and any CD/DVD entries. Do not assume OVF firmware or NIC settings were preserved.

### Step 4 — Configure firmware and NICs

For a **BIOS** guest, normally retain `seabios`. For a **UEFI** guest, first confirm the original firmware and EFI System Partition, then use an EFI disk on compatible storage:

```bash
# ONLY for source guests known to use UEFI:
qm set 120 --bios ovmf
qm set 120 --efidisk0 local-lvm:1,efitype=4m,pre-enrolled-keys=0
```

`pre-enrolled-keys=0` is appropriate only if you do not need Secure Boot. For Windows 11/Secure Boot guests, evaluate keys, TPM state, BitLocker, and boot requirements separately. Never blindly flip BIOS to UEFI or vice versa.

```bash
qm set 120 --net0 virtio,bridge=vmbr0,tag=20
qm config 120
```

For Windows guests without preinstalled VirtIO NIC drivers, start with `e1000` for networking or install VirtIO drivers via console before changing NIC model. For boot disks, keep the imported compatible controller for first boot if needed.

## 7. Path B: manual disk import

**Use when:** OVF import fails, disk order must be controlled, or you only have VMDKs.

### Step 1 — Create an empty VM shell

```bash
qm create 121 --name app01-manual --memory 4096 --cores 2 \
  --ostype l26 --scsihw virtio-scsi-single \
  --net0 virtio,bridge=vmbr0,tag=20
```

For Windows, select the appropriate Windows OS type in the GUI (or check `qm help create`) rather than copying `--ostype l26`. `qm create` here makes **no disk**.

### Step 2 — Import each VMDK

```bash
qm disk import 121 /mnt/migration-staging/app01/extracted/app01-disk1.vmdk local-lvm
qm disk import 121 /mnt/migration-staging/app01/extracted/app01-disk2.vmdk local-lvm
qm config 121
```

The imported disks typically appear as `unused0`, `unused1`. **Do not assume the resulting volume names.** Read the `unused` entries in `qm config` and copy their exact storage-volume IDs.

### Step 3 — Attach disks using the actual imported volume IDs

Example only (replace IDs with those reported by your VM):

```bash
qm set 121 --scsi0 'local-lvm:vm-121-disk-0'
qm set 121 --scsi1 'local-lvm:vm-121-disk-1'
qm set 121 --boot order=scsi0
qm config 121
```

If the guest lacks VirtIO storage drivers, first attach the OS disk on a compatible **SATA** bus and set `--boot order=sata0`, then install VirtIO drivers and switch only after confirming the driver is loaded. For Windows guests, a controller mismatch can produce `INACCESSIBLE_BOOT_DEVICE`.

### Step 4 — BIOS or UEFI

For a UEFI guest:

```bash
qm set 121 --bios ovmf
qm set 121 --efidisk0 local-lvm:1,efitype=4m,pre-enrolled-keys=0
```

Do not add this for a known BIOS guest. Verify EFI partition and boot entry from guest recovery tools if firmware starts but does not find an OS.

### Step 5 — Disk conversion alternative

To inspect/convert a VMDK **on a filesystem with adequate space**:

```bash
qemu-img info app01-disk1.vmdk
qemu-img convert -p -f vmdk -O qcow2 app01-disk1.vmdk app01-disk1.qcow2
qemu-img info app01-disk1.qcow2
```

Then import the qcow2 file using `qm disk import`. Conversion is optional: `qm disk import` can import supported VMDK files directly. Avoid manual `qemu-img` conversion when the source is an incomplete snapshot chain or backing-file set.

## 8. Windows and Linux guest adjustments

### Windows guest: before and after import

1. Record static IPv4/DNS, disk layout, OS edition, activation status, and installed services.
2. Secure BitLocker recovery material and assess vTPM dependence.
3. Install VirtIO drivers from the trusted Fedora/Proxmox Windows VirtIO ISO; a Windows guest may need the VirtIO storage driver **before** switching its boot controller.
4. At first Proxmox boot, use the console and a compatible disk/NIC model if drivers are missing.
5. Install **QEMU Guest Agent** after the guest is stable, then enable it in Proxmox VM **Options**.
6. Reconfigure static IP on the new adapter; remove stale hidden adapters only after confirming the correct settings.
7. Validate Windows activation, applications, domain connectivity, time sync, event logs, and scheduled tasks.
8. Remove VMware Tools when no longer needed, following vendor guidance.

Useful Windows commands (PowerShell, inside guest):

```powershell
Get-NetAdapter
Get-NetIPConfiguration
Get-Disk
Get-Volume
ipconfig /all
Test-NetConnection 192.168.20.1 -Port 53
Get-Service QEMU-GA -ErrorAction SilentlyContinue
```

### Linux guest: before and after import

```bash
lsblk -f
findmnt /
cat /etc/fstab
ip -br addr
ip route
systemctl --failed
journalctl -b -p err --no-pager | tail -80
```

1. Prefer `UUID=` or filesystem labels in `/etc/fstab`; `/dev/sdX` naming can change.
2. Check `/etc/netplan/*.yaml`, NetworkManager profiles, or `/etc/network/interfaces`. VMware's `ens192` may become `ens18` or another name. Update the configuration for the new interface.
3. Check initramfs for needed storage drivers if a Linux guest fails to find root.
4. Install and enable `qemu-guest-agent` when supported:

```bash
sudo apt update
sudo apt install -y qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```

5. Remove `open-vm-tools` only after confirming no dependency on it and the VM is operating correctly.
6. Validate SSH, DNS, routes, mounted data disks, service startup, firewall, and application logs.

## 9. First boot and validation

### Step 1 — Boot isolated

- Leave the original VMware VM **powered off**.
- Prefer a temporary isolated bridge/VLAN for initial boot.
- Start the migrated VM from the Proxmox **Console** and observe firmware/boot messages.
- Check that OS partitions, data disks, and application files are present.

```bash
qm start 120
qm status 120
qm config 120
```

### Step 2 — Validate networking

Inside Linux:

```bash
ip -br a
ip route
ping -c 4 192.168.20.1
getent hosts example.com
```

Inside Windows:

```powershell
ipconfig /all
ping 192.168.20.1
Resolve-DnsName example.com
Test-NetConnection 192.168.20.1 -Port 53
```

Replace all addresses with your actual lab/production values. A successful ping is only one test; validate actual application ports and services too.

### Step 3 — Validate disk and application integrity

- Compare expected disk count, partition sizes, and filesystem UUIDs.
- Confirm databases mount and start cleanly; check transaction/application logs.
- Verify services, scheduled jobs, firewall rules, user login, and monitoring.
- For AD domain controllers, use Microsoft-supported replication/backup validation and **never** casually boot duplicated DCs onto the same domain network.
- Confirm QEMU guest agent, backups, and a test restore.

### Step 4 — Proxmox backup example

```bash
# Replace backup storage with a configured storage that permits VZDump backup files:
vzdump 120 --mode snapshot --storage YOUR_BACKUP_STORAGE --compress zstd
```

Check that the storage supports backups and the VM/disk configuration supports the requested backup mode. Validate restoration on an isolated test VM, not just successful archive creation.

## 10. Troubleshooting matrix

| Symptom | Likely cause | Checks / fix |
|---|---|---|
| `qm importovf` rejects OVF | Unsupported/malformed OVF metadata | Check XML and VMDK references; switch to manual import |
| `No space left on device` | Staging/root/target storage full | `df -h`, `pvesm status`; move staging; free space safely |
| `storage does not support images` | Wrong target content type | Choose VM disk storage or enable Disk image on supported storage |
| VMDK missing / descriptor error | Missing `-flat.vmdk`, split extents or snapshot chain | Restore all referenced files; inspect descriptor; re-export |
| `No bootable device` | Wrong boot disk, firmware, EFI boot entry | `qm config`, boot order, BIOS vs UEFI, EFI System Partition |
| Windows blue screen `INACCESSIBLE_BOOT_DEVICE` | Storage driver/controller mismatch | Reattach boot disk via SATA, install VirtIO driver, then switch |
| Windows activation warning | Virtual hardware changed | Follow legitimate activation/licensing process |
| Linux emergency mode | Changed device names, invalid `/etc/fstab` | Boot console/recovery; inspect UUIDs and mounts |
| NIC missing in Windows | VirtIO driver absent | Use compatible E1000 temporarily; install VirtIO driver |
| Linux has no IP | Interface renamed / DHCP absent | `ip -br a`, netplan/NetworkManager, DHCP/static config |
| VLAN network unreachable | Wrong VM tag, bridge, switch trunk | `qm config`, `bridge vlan show`, upstream allowed VLANs |
| Duplicate IP or hostname | Source and destination simultaneously online | Isolate one copy; cut over only after sign-off |
| High disk latency | Slow target storage, thin pool near full | Check `pvesm status`, ZFS/LVM health, guest I/O |
| Wrong data disk mounted | Disk order/bus changed | Match by filesystem UUID/label; inspect `lsblk -f` |
| QEMU guest agent offline | Not installed, service stopped, VM option off | Check guest service and Proxmox Options |
| OVA extraction fails | Corrupt/incomplete archive | Re-check checksum, `tar -tf`, re-transfer/export |
| Secure Boot/BitLocker recovery | EFI keys or TPM state changed | Use approved recovery process and guest-specific migration plan |
| OVF imported with missing NIC | Metadata not fully translated | Explicitly add `qm set ... --net0 ...` |

## 11. Automation scripts

### Script A — Safe preflight inspection (read-only)

Save as `inspect-ova.sh`, then run `bash inspect-ova.sh /path/to/app01.ova`. It does **not** extract, import, or change any VM.

```bash
#!/usr/bin/env bash
set -euo pipefail
OVA="${1:?Usage: bash inspect-ova.sh /path/to/file.ova}"
[[ -f "$OVA" ]] || { echo "File not found: $OVA" >&2; exit 1; }
echo '=== File ==='
ls -lh -- "$OVA"
file -- "$OVA"
echo '=== SHA256 ==='
sha256sum -- "$OVA"
echo '=== Archive entries (review before extraction) ==='
tar -tf "$OVA"
echo '=== Proxmox version / storage / existing VMs ==='
pveversion
pvesm status
qm list
echo '=== Staging filesystem ==='
df -h -- "$(dirname "$OVA")"
```

### Script B — Guarded OVF import helper

This script intentionally **does not** change firmware, attach network interfaces, or boot the VM. Review the resulting configuration manually. Use only on a trusted, already-extracted OVF with its VMDKs.

Save as `import-ovf.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
if (( $# != 3 )); then
  echo 'Usage: bash import-ovf.sh <unused-vmid> </absolute/path/file.ovf> <storage-id>' >&2
  exit 2
fi
VMID="$1"; OVF="$2"; STORAGE="$3"
[[ "$VMID" =~ ^[0-9]+$ ]] || { echo 'VMID must be numeric' >&2; exit 2; }
[[ -f "$OVF" ]] || { echo 'OVF not found' >&2; exit 2; }
if qm status "$VMID" >/dev/null 2>&1; then
  echo "VMID $VMID already exists; refusing to overwrite" >&2
  exit 1
fi
pvesm status | awk 'NR>1 {print $1}' | grep -Fxq "$STORAGE" || {
  echo "Storage $STORAGE not found" >&2; exit 1;
}
echo '=== Dry run ==='
qm importovf "$VMID" "$OVF" "$STORAGE" --dryrun 1
read -r -p 'Type IMPORT to create the VM: ' ANSWER
[[ "$ANSWER" == 'IMPORT' ]] || { echo 'Cancelled'; exit 1; }
qm importovf "$VMID" "$OVF" "$STORAGE"
echo '=== Imported configuration: REVIEW BEFORE BOOT ==='
qm config "$VMID"
```

**Script limitations:** Storage existence does not guarantee enough capacity or that it accepts disk images; check `pvesm config STORAGE`, free space, firmware, disk mapping, and networking first. Import is not an atomic rollback transaction. If it fails partway through, inspect created volumes/VM state before any cleanup. Never automate `qm destroy` on an uncertain VMID.

### Script C — Post-import read-only report

Save as `check-vm.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
VMID="${1:?Usage: bash check-vm.sh <vmid>}"
qm status "$VMID"
qm config "$VMID"
echo '=== Snapshots ==='
qm listsnapshot "$VMID" || true
echo '=== Host storage ==='
pvesm status
echo '=== Reminder: validate from INSIDE guest ==='
echo 'OS boot, IP/DNS, services, data disks, application tests, backup and restore.'
```

## 12. Rollback, backups, and acceptance

**Rollback plan:**

1. Define a cutover window and explicit go/no-go owner.
2. Before cutover, keep the source VMware VM powered off and retain the original OVA plus application backups.
3. If Proxmox validation fails, power off and isolate the imported VM; do not let two copies run with the same IP/identity.
4. Assess whether any writes occurred on the migrated VM. If yes, plan data reconciliation; blindly returning to the older VMware disk can lose changes.
5. Re-enable the original VMware VM only after confirming network identity, application data consistency, and ownership of production traffic.
6. Document root cause and reschedule migration.

**Acceptance checklist:**

- [ ] Original OVA checksum recorded and export retained
- [ ] VMware VM inventory and firmware documented
- [ ] Target storage capacity and disk count verified
- [ ] Proxmox VM configuration reviewed (`qm config`)
- [ ] Boot disk and firmware correct
- [ ] All data disks mounted and applications validated
- [ ] Network bridge, VLAN, IP, DNS and gateway correct
- [ ] No source/destination identity collision
- [ ] Guest drivers and QEMU guest agent healthy
- [ ] Backups completed and restore tested
- [ ] Monitoring, time synchronization, and security controls checked
- [ ] Application owner signed off; rollback window closed

## 13. Hands-on labs and reference links

**Lab 1 — Linux VM:** Export a disposable Ubuntu VM to OVA, inspect OVF/VMDK, import to Proxmox, fix a renamed NIC, validate SSH and a sample data disk.

**Lab 2 — Windows VM:** Migrate a nonproduction Windows guest, practice booting on SATA then moving to VirtIO after installing drivers, reassign static IP, and test RDP.

**Lab 3 — VLAN mapping:** Build two isolated VLANs on a lab-capable managed switch and VLAN-aware bridge. Verify each VM reaches only the intended network. Do not change a remote production switch without out-of-band access.

**Lab 4 — Failure recovery:** Intentionally attach a disposable VM's disk on an incorrect bus, observe boot failure, and recover by restoring its original controller/boot order. Never do this on production VMs.

### Official references

- Proxmox VE documentation: https://pve.proxmox.com/pve-docs/
- Proxmox `qm` command reference: https://pve.proxmox.com/pve-docs/qm.1.html
- Proxmox migration guide: https://pve.proxmox.com/wiki/Migrate_to_Proxmox_VE
- Proxmox network configuration: https://pve.proxmox.com/wiki/Network_Configuration
- Windows VirtIO drivers: https://pve.proxmox.com/wiki/Windows_VirtIO_Drivers
- QEMU disk image documentation: https://www.qemu.org/docs/master/tools/qemu-img.html

---

**Recommended repository path:** `virtualization/vmware-ova-to-proxmox-ve9/README.md`  
**Version:** 1.0 · 2026-11-15  
**Reminder:** This is an educational runbook, not a substitute for an application-specific production change plan. Test commands on your actual Proxmox VE 9.x build and verify storage/network names before execution.
