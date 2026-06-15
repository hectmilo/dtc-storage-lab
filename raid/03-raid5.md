# Phase 3 — RAID 5 (Stripe + Parity)

**Learning goal:** Understand striping with distributed parity. RAID 5 spreads both data and checksums across all disks. One disk can die with no data loss. Usable capacity is (N−1) × disk size.

---

## Background

RAID 5 writes data in **stripes** across all member disks. For each stripe, one disk holds **parity** — a checksum computed from the other disks' data. If one disk fails, the controller recalculates the missing data from the survivors + parity.

Unlike RAID 1, parity is distributed — no single disk is the dedicated parity disk. This avoids a write bottleneck and means any single disk can be lost.

**Compared to RAID 1:**
- More usable space (2 of 3 disks, vs 1 of 2)
- Rebuild is CPU-intensive (must recalculate parity), not just a copy

**Minimum disks:** 3

---

## Commands

### Load device variables

```bash
source /opt/raid-lab/devices.env
echo "Using: $DISK3 $DISK4 $DISK5"
```

### Create the array

```bash
sudo mdadm --create /dev/md1 --level=5 --raid-devices=3 $DISK3 $DISK4 $DISK5
```

- `/dev/md1` — second array, separate from the mirror
- `--level=5` — RAID 5
- `--raid-devices=3` — three members

Type `y` when prompted.

### Watch the initial sync

```bash
watch -n1 cat /proc/mdstat
```

RAID 5 calculates parity during the initial sync. You'll see a progress bar and speed estimate. The `chunk` size (default 512k) is how large each stripe unit is — data cycles across disks in 512 KB blocks.

Hit `Ctrl+C` once the status shows `[UUU]`.

### Inspect the array

```bash
sudo mdadm --detail /dev/md1
```

Compare **Array Size** vs **Used Dev Size**. Array size is roughly double one disk — the (N−1) capacity formula at work.

### Filesystem, mount, and test data

```bash
sudo mkfs.ext4 /dev/md1
sudo mkdir -p /mnt/raid5
sudo mount /dev/md1 /mnt/raid5
echo "RAID 5 test - $(date)" | sudo tee /mnt/raid5/testfile.txt
sudo dd if=/dev/urandom of=/mnt/raid5/bigfile.bin bs=1M count=50
```

`bigfile.bin` gives the Phase 4 rebuild real data to work with and verify.

### Compare capacity side-by-side

```bash
df -h /mnt/raid1 /mnt/raid5
```

- `raid1` ≈ 470 MB (1 disk)
- `raid5` ≈ 940 MB (2 disks worth, parity on the 3rd)

This single output captures the core RAID 1 vs RAID 5 capacity trade-off.

---

## Expected State After This Phase

- `/dev/md1` active, state `clean`, `[UUU]`
- ext4 filesystem mounted at `/mnt/raid5`
- `testfile.txt` and `bigfile.bin` present on the mount
- `df -h` shows ~940 MB usable on `raid5` vs ~470 MB on `raid1`
