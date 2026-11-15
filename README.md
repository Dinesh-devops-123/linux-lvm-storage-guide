# linux-lvm-storage-guide
LVM


# Linux Disk Partitioning & LVM Setup Guide

A step-by-step guide for partitioning a block device using `fdisk`, configuring Logical Volume Management (PV, VG, LV), formatting with XFS, persisting mounts in `/etc/fstab`, and extending storage dynamically.

---

## 1. Partition the Disk Using `fdisk`

First, inspect attached block devices to confirm the target disk name:
```bash
lsblk
```

Open the target disk in `fdisk` (example: `/dev/nvme1n1`):
```bash
fdisk /dev/nvme1n1
```

Inside the `fdisk` interactive prompt:
1. Type `m` to view the help menu.
2. Type `n` to create a new partition.
   * **Partition number:** `1` (or press Enter for default)
   * **First sector:** Press Enter for default
   * **Last sector:** `+10G` (allocates 10 GB)
3. Type `p` to print and verify the new 10 GB partition.
4. Type `t` to change the partition type to **Linux LVM**:
   * Enter `8e` (for MBR/DOS) or `30` / `L` to select `Linux LVM` (for GPT).
5. Type `p` again to confirm the partition type shows **Linux LVM**.
6. Type `w` to write changes to disk and exit back to the shell.

Verify the new partition appears under the block device:
```bash
lsblk
```

---

## 2. Create Physical Volume (PV), Volume Group (VG), and Logical Volume (LV)

### Step 1: Initialize the Physical Volume
```bash
pvcreate /dev/nvme1n1p1
pvs
```

### Step 2: Create a Volume Group
```bash
vgcreate <vg_name> /dev/nvme1n1p1
vgs
```

### Step 3: Create a Logical Volume
If you try to allocate the exact partition size (`-L 10G`), LVM may report slightly fewer extents due to metadata overhead:
> `Volume group "vg_appvg" has insufficient free space (2559 extents): 2560 required.`

Use `-l 100%FREE` to allocate all available extents in the Volume Group:
```bash
lvcreate -l 100%FREE -n <lv_name> <vg_name>
lvs
```

---

## 3. Format and Mount the Filesystem

### Format with XFS
```bash
mkfs.xfs /dev/<vg_name>/<lv_name>
```

### Create a Mount Point and Mount
```bash
mkdir -p /<app_name>
mount /dev/<vg_name>/<lv_name> /<app_name>
```

Verify the mounted filesystem:
```bash
df -h
```

---

## 4. Persist the Mount Across Reboots (`/etc/fstab`)

Always take a backup of `/etc/fstab` before editing it:
```bash
cp -p /etc/fstab /etc/fstab_bkp
```

Open `/etc/fstab`:
```bash
vi /etc/fstab
```

Add the following entry at the end of the file:
```text
/dev/<vg_name>/<lv_name>   /<app_name>   xfs   defaults   0 0
```

Validate `/etc/fstab` syntax without rebooting (if no errors appear, the entry is valid):
```bash
mount -a
df -h /<app_name>
```

---

## 5. Extend the Volume Group and Logical Volume

### Step 1: Create a Second Partition (`+8G`)
Open `fdisk` on the disk again:
```bash
fdisk /dev/nvme1n1
```
1. Type `n` to add a new partition (`nvme1n1p2`) with size `+8G`.
2. Type `t` and set its type to **Linux LVM** (`8e` / `Linux LVM`).
3. Type `w` to save and exit.

### Step 2: Create PV and Extend the Existing VG
```bash
pvcreate /dev/nvme1n1p2
vgextend <vg_name> /dev/nvme1n1p2
vgs
```

### Step 3: Extend the Logical Volume and Resize Filesystem Online
Check current usage:
```bash
df -h /<app_name>
```

Extend the Logical Volume by `+5G` and automatically resize the XFS filesystem using `-r`:
```bash
lvextend -L +5G /dev/<vg_name>/<lv_name> -r
```

Confirm the updated filesystem size:
```bash
df -h /<app_name>
```
