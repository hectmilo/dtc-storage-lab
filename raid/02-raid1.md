# Phase 2 — RAID 1 (Mirror)

**Learning goal:** Understand what mirroring means: every write goes to both disks simultaneously. One disk dies, zero data loss, no rebuild required to keep running.

---

## Background

RAID 1 writes identical data to every member disk. The kernel handles this transparently — the filesystem above sees a single device. If one disk fails, the array enters **degraded mode** and keeps serving data from the survivor.

**Trade-off:** Usable capacity equals one disk, not the sum of all disks.

---

## Commands

### Load device variables

```bash
source /opt/raid-lab/devices.env
echo "Using: $DISK1 and $DISK2"
```

### Create the array

```bash
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 $DISK1 $DISK2
```

- `/dev/md0` — the new array device (`md` = multiple devices)
- `--level=1` — RAID 1, the mirror
- `--raid-devices=2` — two members

Type `y` when prompted.

### Check the initial sync

```bash
cat /proc/mdstat
```

`/proc/mdstat` is the kernel's live RAID status file. Look for `[UU]` — both members Up. A short sync may run on a fresh array; it completes in seconds at this size.

### Inspect the array

```bash
sudo mdadm --detail /dev/md0
```

Read every line. This is the same output you'd examine when an alert fires on a production server. Note the **State**, member disk roles, and UUID.

### Create a filesystem and mount it

```bash
sudo mkfs.ext4 /dev/md0
sudo mkdir -p /mnt/raid1
sudo mount /dev/md0 /mnt/raid1
echo "RAID 1 test - $(date)" | sudo tee /mnt/raid1/testfile.txt
```

The filesystem sits on `/dev/md0` — the array device. The mirroring is invisible above this layer.

### Check usable capacity

```bash
df -h /mnt/raid1
```

Expected: ~470 MB — one disk's worth, not two. This is the RAID 1 capacity trade-off.

---

## Expected State After This Phase

- `/dev/md0` active, state `clean`, `[UU]`
- ext4 filesystem mounted at `/mnt/raid1`
- `testfile.txt` readable on the mount
- `df -h` shows ~470 MB usable
