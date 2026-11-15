# Disk Partitioning

> **Hands-on lab:** Learn to inspect, create, shrink, format, mount, and troubleshoot partitions on Windows and Linux. Practice on a **disposable VM disk** before touching a physical computer.

## Table of contents
1. [Learning outcomes](#1-learning-outcomes)
2. [What is a partition?](#2-what-is-a-partition)
3. [Types of partitions and file systems](#3-types-of-partitions-and-file-systems)
4. [Safety checklist](#4-safety-checklist)
5. [Lab A — Create a practice disk in a VM](#5-lab-a--create-a-practice-disk-in-a-vm)
6. [Lab B — Partition a new disk in Windows](#6-lab-b--partition-a-new-disk-in-windows)
7. [Lab C — Shrink an existing Windows volume](#7-lab-c--shrink-an-existing-windows-volume)
8. [Lab D — Inspect disks with PowerShell](#8-lab-d--inspect-disks-with-powershell)
9. [Lab E — Partition a new Linux disk](#9-lab-e--partition-a-new-linux-disk)
10. [Lab F — Persistent Linux mounting](#10-lab-f--persistent-linux-mounting)
11. [Advanced concepts](#11-advanced-concepts)
12. [Troubleshooting](#12-troubleshooting)
13. [Practice assignments and completion checklist](#13-practice-assignments-and-completion-checklist)

## 1. Learning outcomes

After this module, you should be able to:
- Explain the difference between a **physical disk**, **partition**, **volume**, **file system**, and **drive letter**.
- Explain **GPT vs. MBR**, **NTFS vs. exFAT vs. ext4**, and **EFI / recovery partitions**.
- Add a virtual disk to a VM, initialize it, partition it, and format it.
- Safely shrink a Windows data or system volume **after backing up**.
- Inspect Linux disks, create an ext4 file system, and mount it by UUID.
- Recognize dangerous operations and avoid modifying the wrong disk.

## 2. What is a partition?

A **disk** is a storage device (SSD, HDD, or virtual disk). A **partition** is a defined region of that disk. A **file system** organizes files within a partition. A **volume** is a storage unit presented to the operating system; it often maps to a partition but can span multiple disks. Windows often gives volumes drive letters such as `C:` or `D:`; Linux usually attaches file systems to folders such as `/mnt/labdata`.

Example (illustrative, not to scale):

```text
Physical SSD: 500 GB
+----------------+--------------------+----------------+----------+
| EFI / Recovery | C: Windows 300 GB  | D: Data 150 GB | Free     |
+----------------+--------------------+----------------+----------+
```

**Important:** A second partition on the same physical drive is **not a backup**. If the drive fails, both partitions may be lost.

### Vocabulary notes

| Term | Explanation | Example |
|---|---|---|
| Disk | Physical or virtual storage device | SSD 0, Virtual Disk 1 |
| Partition | Region of a disk | Partition 2 |
| File system | How data is organized | NTFS, ext4 |
| Volume | OS-managed storage area | `D:` |
| Unallocated space | Disk capacity not assigned to a partition | 30 GB unallocated |
| Format | Create a file system on a volume | Format as NTFS |
| Mount | Make a file system accessible | `/mnt/labdata` |
| EFI System Partition | UEFI boot files | Usually small, no drive letter |
| Recovery partition | OS recovery tools | Windows RE |

## 3. Types of partitions and file systems

### GPT vs. MBR

| Feature | GPT (GUID Partition Table) | MBR (Master Boot Record) |
|---|---|---|
| Typical modern use | Preferred for UEFI systems | Legacy compatibility |
| Disk-size limits | Supports very large disks | Common 2 TiB addressing limit with 512-byte sectors |
| Partition layout | Usually supports many partitions | Four primary entries (or extended/logical scheme) |
| Recommendation | Choose for new modern disks | Use only if required for older systems |

**Do not convert your boot disk between MBR and GPT just for practice.** Conversion can require special preparation and may make a computer unbootable.

### Common file systems

| File system | Typical use | Notes |
|---|---|---|
| NTFS | Windows internal disks | Permissions, journaling |
| exFAT | USB drives shared across operating systems | Broad compatibility; fewer advanced features |
| FAT32 | EFI and some removable media | Individual files limited to 4 GiB minus 1 byte |
| ext4 | Linux disks | Common journaling Linux file system |
| XFS | Linux servers | Common for large-scale Linux workloads |
| ZFS | Servers / Proxmox | Storage pooling, checksums, snapshots; requires planning |

### Partition roles

- **System / EFI partition:** Contains boot support files; do not delete.
- **OS partition:** Usually `C:` on Windows or `/` on Linux.
- **Data partition:** Stores documents, applications, or lab files.
- **Recovery partition:** Used for recovery operations; do not delete.
- **Swap partition or swap file:** Supports Linux virtual memory; not required as a separate partition for every setup.

## 4. Safety checklist

Before modifying a **real** disk:

1. **Back up important files** to a different disk or trusted backup destination; verify you can restore them.
2. If Windows uses **BitLocker / Device Encryption**, confirm your recovery key is accessible through your approved recovery process. Storage changes can trigger recovery.
3. Keep the computer plugged into reliable power.
4. Open **Disk Management** and identify the correct disk by **capacity, model, and existing volumes**.
5. Never delete or format an unknown partition, particularly **EFI**, **Recovery**, **MSR**, or an OS partition.
6. Read each dialog carefully. **Formatting destroys existing file-system data** on the selected volume.
7. For first practice, create a **separate blank virtual disk** in a VM; do not practice on `C:`.

> **STOP:** If the disk contains files you need, or its identity is uncertain, do not continue until you have confirmed the backup and selected disk.

## 5. Lab A — Create a practice disk in a VM

**Goal:** Add a separate, blank 20 GB disk to a Windows VM. This isolates the exercise from its OS disk.

### VirtualBox

1. Shut down the Windows VM fully (not saved state).
2. In **VirtualBox Manager**, select the VM → **Settings** → **Storage**.
3. Select the SATA controller, then choose **Add Hard Disk**.
4. Choose **Create** and select a VDI virtual disk.
5. Set **20 GB** and **dynamically allocated**.
6. Name it `partition-practice.vdi` and attach it.
7. Start the VM and log in to Windows.

### Proxmox VE

1. Shut down your Windows VM.
2. In the Proxmox web interface, select the VM → **Hardware** → **Add** → **Hard Disk**.
3. Choose the intended lab storage, bus/device (for example **SCSI** with VirtIO SCSI controller), and **20 GiB** size.
4. Confirm the new disk and start the VM.
5. Log in to Windows.

**Verify:** Windows Disk Management should show a new disk of approximately 20 GB as **Unknown / Not Initialized** or **Unallocated**. If you do not see it, check the VM's hardware configuration before proceeding.

## 6. Lab B — Partition a new disk in Windows

**Goal:** Create two NTFS volumes, `LAB-DATA` and `LAB-BACKUP`, on the **new, empty VM disk**. These are exercise names; the second volume is not an actual backup.

### Step 1 — Open Disk Management

1. In the Windows VM, press **Windows + X**.
2. Select **Disk Management** (or press **Windows + R**, type `diskmgmt.msc`, press Enter).
3. Locate the **new 20 GB disk** by its capacity. **Do not select Disk 0 just because it is first.**

### Step 2 — Initialize the blank disk

1. If prompted, select the new disk only.
2. Choose **GPT**.
3. Click **OK**.
4. Confirm that the new disk shows **Unallocated** space.

**Why GPT?** It is the standard partition-table choice for modern Windows systems.

### Step 3 — Create the first volume

1. Right-click the **Unallocated** area of the new disk.
2. Select **New Simple Volume** → **Next**.
3. Enter **10240 MB** as the volume size (roughly 10 GiB).
4. Choose an available drive letter, such as **D:**. Your actual letter may differ.
5. Choose **NTFS**, **Default allocation unit size**, **Quick Format**.
6. Enter the label **LAB-DATA**.
7. Click **Next → Finish**.

**Expected result:** A healthy NTFS volume named `LAB-DATA` appears in Disk Management and File Explorer.

### Step 4 — Create the second volume

1. Right-click the remaining **Unallocated** area of the same practice disk.
2. Select **New Simple Volume**.
3. Accept the remaining available size.
4. Assign another available letter, such as **E:**.
5. Choose **NTFS** and label it **LAB-BACKUP**.
6. Finish the wizard.

**Verify:** Open File Explorer → **This PC**. You should see both new volumes. Create a file named `test.txt` on each volume, open it, and confirm it saves correctly.

### What happened?

You divided one virtual disk into two partitions, created NTFS file systems, and mounted them using Windows drive letters. The partitions share the **same virtual disk**, so neither protects against that disk being lost.

## 7. Lab C — Shrink an existing Windows volume

**Goal:** Learn how Windows can reduce the size of an existing NTFS volume to create unallocated space.

**Safest setup:** Use a disposable Windows VM with a recent snapshot **and a separate backup**. Do not start on your daily-use PC.

1. Verify the VM backup and snapshot exist. Check that the volume has ample free space.
2. Open **Disk Management** (`diskmgmt.msc`).
3. Right-click the **test NTFS volume** you intend to reduce.
4. Select **Shrink Volume**.
5. Wait for Windows to calculate the maximum shrink amount.
6. Enter a conservative shrink amount, for example **2048 MB** (approximately 2 GiB), provided Windows permits it.
7. Click **Shrink**.
8. Confirm that **Unallocated** space now appears next to the reduced partition.
9. Right-click the unallocated space → **New Simple Volume**, then format it as NTFS with a clear label such as `SHRINK-LAB`.

**Why might Windows refuse to shrink much?** Unmovable files, file-system conditions, or layout constraints may limit the available shrink size. Do not delete system files or disable recovery features just to force a shrink.

**Note:** Shrinking a partition changes its boundary. It does not automatically create a new drive; you must create a volume in the resulting unallocated space.

## 8. Lab D — Inspect disks with PowerShell

Open **PowerShell as Administrator** inside your Windows VM.

### Step 1 — List disks

```powershell
Get-Disk | Format-Table Number, FriendlyName, PartitionStyle, Size, OperationalStatus
```

**Explanation:** Shows disk numbers, names, partition styles, sizes, and status. **Disk numbers can change**; verify before any write operation.

### Step 2 — List partitions

```powershell
Get-Partition | Format-Table DiskNumber, PartitionNumber, DriveLetter, Size, Type
```

**Explanation:** Shows how disks are divided into partitions.

### Step 3 — List volumes

```powershell
Get-Volume | Format-Table DriveLetter, FileSystemLabel, FileSystem, Size, SizeRemaining
```

**Explanation:** Shows volume letters, file systems, total sizes, and free space.

### Step 4 — Inspect one known disk

```powershell
Get-Partition -DiskNumber 1
```

Replace `1` **only after** identifying your actual practice disk from `Get-Disk`.

**Assignment:** Record the number, size, partition style, and volume labels of your practice disk. Avoid automated disk-writing commands until you are comfortable identifying the target.

## 9. Lab E — Partition a new Linux disk

**Goal:** Add a second blank disk to an Ubuntu VM, create one GPT partition, format it as ext4, and mount it.

**Prerequisite:** Add a new **blank 10 GiB virtual disk** in VirtualBox or Proxmox using the same method as Lab A. Work only in the disposable Ubuntu VM.

### Step 1 — Identify disks

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL
```

**Explanation:** `lsblk` lists block devices and mounted partitions. Identify the new **10 GiB disk** by size and absence of partitions. Your disk may be `/dev/sdb`, `/dev/vdb`, or `/dev/nvme1n1`—**never assume its name**.

For the commands below, the example device is `/dev/sdb`. **Replace it with the verified blank practice disk.** If you cannot confidently identify it, stop.

### Step 2 — Create a GPT partition table and partition

**WARNING: The following commands destroy the existing partition layout on the specified disk. Run only against the confirmed blank practice disk.**

```bash
sudo parted /dev/sdb --script mklabel gpt
sudo parted /dev/sdb --script mkpart primary ext4 1MiB 100%
```

**Explanation:** `mklabel gpt` creates a new GPT partition table. `mkpart` creates one partition occupying almost all available space, with alignment space at the beginning.

### Step 3 — Inspect the result

```bash
lsblk -f
```

You should see a new partition, typically `/dev/sdb1`. NVMe names may instead end in `p1`.

### Step 4 — Format the new partition

**WARNING: Formatting overwrites the file-system metadata on the selected partition.**

```bash
sudo mkfs.ext4 -L LABDATA /dev/sdb1
```

**Explanation:** Creates an ext4 file system with label `LABDATA` on the new partition. Replace `/dev/sdb1` with the **actual new partition** you verified.

### Step 5 — Mount the file system

```bash
sudo mkdir -p /mnt/labdata
sudo mount /dev/sdb1 /mnt/labdata
findmnt /mnt/labdata
df -h /mnt/labdata
```

**Explanation:** Creates a mount directory, attaches the file system, and verifies it is accessible.

### Step 6 — Test writing

```bash
echo 'Partition lab successful' | sudo tee /mnt/labdata/test.txt
cat /mnt/labdata/test.txt
```

Expected text: `Partition lab successful`.

## 10. Lab F — Persistent Linux mounting

A manual `mount` may not survive reboot. Linux can mount a file system automatically using `/etc/fstab`.

### Step 1 — Find the UUID

```bash
sudo blkid /dev/sdb1
```

Find the `UUID="..."` value. **Use your actual UUID**, not the sample below.

### Step 2 — Back up fstab

```bash
sudo cp /etc/fstab /etc/fstab.bak
```

### Step 3 — Edit fstab

```bash
sudo nano /etc/fstab
```

Add a line, replacing `YOUR-ACTUAL-UUID` with the UUID from `blkid`:

```text
UUID=YOUR-ACTUAL-UUID /mnt/labdata ext4 defaults,nofail 0 2
```

**Explanation:** The UUID identifies the file system more reliably than a device name. `nofail` allows boot to continue if the lab disk is unavailable.

### Step 4 — Validate before reboot

```bash
sudo systemctl daemon-reload
sudo findmnt --verify
sudo mount -a
findmnt /mnt/labdata
```

If validation reports an error, fix `/etc/fstab` **before rebooting**. You can restore the backup with `sudo cp /etc/fstab.bak /etc/fstab` if necessary.

### Step 5 — Reboot and verify

```bash
sudo reboot
```

After login:

```bash
findmnt /mnt/labdata
cat /mnt/labdata/test.txt
```

If both succeed, the partition is persistently mounted.

## 11. Advanced concepts

### UEFI, BIOS, and boot partitions

- **UEFI** firmware commonly boots from an **EFI System Partition** on a GPT disk.
- **Legacy BIOS** systems may use MBR boot arrangements.
- **Windows Recovery Environment** often has a dedicated recovery partition.
- Do not delete, resize, or move boot and recovery partitions as a beginner.

### Basic partitions vs. LVM and storage pools

- **Windows basic disk:** Partitions and volumes on a disk.
- **Linux LVM:** Separates physical storage, volume groups, and logical volumes; useful for flexible server storage.
- **Storage Spaces / ZFS:** Higher-level storage management with different redundancy and resiliency features.
- **RAID is not backup.** Redundancy may help with a disk failure but does not protect against accidental deletion, malware, or many other loss scenarios.

### Partitioning in Proxmox

The **Proxmox host's** partition layout is different from the **guest VM's** partition layout. A 32 GiB virtual disk appears inside a VM as its own disk. Partitioning that virtual disk does **not** partition the Proxmox host's physical disk directly. The host stores the VM's virtual-disk data on configured storage.

**Practice:** Create an Ubuntu VM with a 32 GiB OS disk plus a separate 10 GiB data disk. In Ubuntu, partition and format **only the second disk**.

### Resizing virtual disks

Increasing the size of a virtual disk in VirtualBox or Proxmox does **not necessarily enlarge the guest partition or file system automatically**. Resizing can involve three separate operations:
1. Increase the virtual disk capacity in the hypervisor.
2. Extend the guest partition into the new free space.
3. Grow the file system.

The exact commands depend on whether the guest uses NTFS, ext4, XFS, LVM, BitLocker, or another layout. **Do not run generic resize commands on an unfamiliar disk layout.**

## 12. Troubleshooting

| Problem | Likely cause | Safe first action |
|---|---|---|
| New disk not visible in Windows | VM disk not attached or detected | Verify VM hardware; refresh Disk Management |
| Disk says “Not Initialized” | No partition table | Confirm it is the **new blank lab disk**, then initialize GPT |
| Cannot shrink much | Unmovable files / layout limitations | Check free space and backups; do not force |
| Drive letter missing | No letter assigned | Use Disk Management to assign an available letter to a **data volume** |
| Linux disk not visible | VM disk attachment or controller issue | Recheck `lsblk` and VM hardware |
| `mount` says wrong fs type | Wrong device or missing/corrupt file system | Check `lsblk -f` and `blkid`; do not reformat blindly |
| Linux does not boot after fstab edit | Invalid mount entry | Use recovery access and restore/fix `/etc/fstab` |
| Windows asks for BitLocker key | Protection recovery triggered | Use your saved recovery key; do not attempt bypass |
| “Format disk?” prompt on existing disk | Unrecognized or damaged file system | Cancel; investigate before writing anything |

## 13. Practice assignments and completion checklist

### Assignment 1 — Vocabulary

Explain in your own words: disk, partition, volume, file system, unallocated space, GPT, MBR, EFI, mount point.

### Assignment 2 — Windows hands-on

- [ ] Create a separate blank 20 GB virtual disk.
- [ ] Identify it correctly in Disk Management.
- [ ] Initialize it as GPT.
- [ ] Create `LAB-DATA` and `LAB-BACKUP` NTFS volumes.
- [ ] Write and reopen a test file on each volume.
- [ ] Use PowerShell to list the disk, partitions, and volumes.
- [ ] Explain why two partitions on one disk are not a backup.

### Assignment 3 — Linux hands-on

- [ ] Add a blank 10 GiB disk to Ubuntu VM.
- [ ] Identify its correct device name with `lsblk`.
- [ ] Create a GPT partition table and one partition on that disk.
- [ ] Format the new partition as ext4.
- [ ] Mount it at `/mnt/labdata`.
- [ ] Create and read `test.txt`.
- [ ] Configure `/etc/fstab` using the actual UUID.
- [ ] Reboot and verify the mount persists.

### Assignment 4 — Explain the design

Write a short note answering:
1. Why should you back up before shrinking a partition?
2. What is the difference between **formatting** and **partitioning**?
3. Why is it risky to delete an EFI or recovery partition?
4. Why does Linux prefer UUIDs in `/etc/fstab`?
5. How is expanding a VM disk different from expanding its file system?

### Final checkpoint

You have completed this module when you can safely identify the correct disk, create and format a new partition, verify read/write access, explain the risk of modifying a boot partition, and recover from a simple mount-configuration mistake.

---

**Next modules:** Disk imaging and backups · Windows installation and dual boot · Linux LVM · Proxmox storage pools · RAID and ZFS · Disaster recovery.

**Responsible-use reminder:** All destructive disk commands in this guide are intended only for disposable lab disks that you have positively identified. Back up real systems and follow your organization's change-control process.
