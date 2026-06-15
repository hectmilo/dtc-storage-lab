# Phase 5 — LVM on Top of RAID

**Learning goal:** Understand that LVM is a flexible allocation layer that sits above RAID. RAID handles redundancy; LVM handles resizing, snapshots, and named volumes. Together they are the standard production pattern.

---

## Background

**The stack:**
```
filesystem (ext4)
      ↑
logical volume (lv_data)    ← LVM layer: flexible allocation
      ↑
volume group (vg_lab)       ← LVM layer: storage pool
      ↑
physical volume (/dev/md1)  ← LVM layer: registers the block device
      ↑
RAID 5 array (/dev/md1)     ← RAID layer: redundancy
      ↑
loop devices / disks
```

LVM is unaware of RAID. It sees `/dev/md1` as a plain block device. This means LVM features — resizing, snapshots — work identically whether the underlying device is RAID, a single disk, or a cloud block volume.

**Three LVM terms:**
- **PV (Physical Volume)** — a block device registered with LVM
- **VG (Volume Group)** — a storage pool built from one or more PVs
- **LV (Logical Volume)** — a named slice of a VG, formatted and mounted like a partition

---

## Commands

### Unmount the RAID 5 filesystem

```bash
sudo umount /mnt/raid5
```

We replace the raw ext4 on `/dev/md1` with LVM. The previous filesystem is overwritten.

### Create the Physical Volume

```bash
sudo pvcreate /dev/md1
sudo pvs
```

Writes an LVM header to the RAID device. Type `y` to confirm overwriting the existing ext4 signature.

### Create the Volume Group

```bash
sudo vgcreate vg_lab /dev/md1
sudo vgs
```

Creates a storage pool named `vg_lab` backed by the PV. `vgs` shows total size and free space.

### Create a Logical Volume

```bash
sudo lvcreate -n lv_data -L 500M vg_lab
sudo lvs
```

Carves a 500 MB named volume out of the VG. We leave headroom for the snapshot.

### Format and mount

```bash
sudo mkfs.ext4 /dev/vg_lab/lv_data
sudo mkdir -p /mnt/lvm
sudo mount /dev/vg_lab/lv_data /mnt/lvm
echo "LVM test - $(date)" | sudo tee /mnt/lvm/testfile.txt
df -h /mnt/lvm
```

The device path `/dev/vg_lab/lv_data` is a symlink LVM creates automatically: VG name / LV name.

### Resize the LV online

```bash
sudo lvextend -L +150M /dev/vg_lab/lv_data
sudo resize2fs /dev/vg_lab/lv_data
df -h /mnt/lvm
```

- `lvextend` grows the block device
- `resize2fs` tells ext4 to use the new space

Both run while the filesystem is **mounted and in use** — no downtime, no unmount required. The `df -h` output should show ~150 MB more usable space than before.

### Take a snapshot

```bash
echo "important data before snapshot" | sudo tee /mnt/lvm/important.txt

sudo lvcreate -n lv_snap -L 200M -s /dev/vg_lab/lv_data
sudo lvs
```

A snapshot is a frozen, point-in-time copy of the LV. It costs no space at creation — space is consumed only as the source volume changes after the snapshot. The `-L 200M` reserves space to track those changes.

On a real server you'd snapshot before a risky upgrade or database migration.

### Demonstrate a restore

```bash
# Simulate data loss after the snapshot
sudo rm /mnt/lvm/important.txt
ls /mnt/lvm/

# Restore from snapshot
sudo umount /mnt/lvm
sudo lvconvert --merge /dev/vg_lab/lv_snap
sudo mount /dev/vg_lab/lv_data /mnt/lvm
cat /mnt/lvm/important.txt
```

`--merge` rolls `lv_data` back to the snapshot state and removes the snapshot LV automatically. The deleted file is restored.

---

## Expected State After This Phase

- `vg_lab` VG present, backed by `/dev/md1`
- `lv_data` LV ~650 MB (500 + 150 MB resize), mounted at `/mnt/lvm`
- Snapshot successfully merged, `important.txt` restored
- `lvs` shows only `lv_data` (snapshot consumed by merge)
