# Phase 4 — Disk Failure and Rebuild

**Learning goal:** Walk the full DCT failure-recovery workflow: detect a degraded array, remove the failed disk, add a replacement, and verify data integrity after the rebuild.

---

## Background

RAID 5 can survive one disk failure. When a failure occurs the array enters **degraded mode** — it keeps serving data, but there is no fault tolerance until the rebuild completes. The time between failure and a completed rebuild is the highest-risk window: a second failure during this period causes data loss.

On a real server this is an automated alert. In this lab we trigger it manually with `--fail`.

---

## The Recovery Sequence

```
Failure detected → confirm degraded state → remove bad disk
→ attach replacement → watch rebuild → verify data intact
```

---

## Commands

### Confirm healthy baseline

```bash
sudo mdadm --detail /dev/md1
```

Note `State : clean` and all three members active before introducing the failure.

### Mark a disk as failed

```bash
source /opt/raid-lab/devices.env
sudo mdadm /dev/md1 --fail $DISK3
```

`--fail` tells the kernel this disk has gone bad. On production hardware this happens automatically when a drive stops responding to I/O.

### Read the degraded state

```bash
cat /proc/mdstat
sudo mdadm --detail /dev/md1
```

`/proc/mdstat` will show `[2/3]` and a `_` in the status flags — e.g. `[_UU]`. In `--detail`, the failed disk will show state `faulty`. This is the information you'd read to confirm which physical drive to pull.

The array is still online and serving data. RAID 5 is doing its job.

### Remove the failed disk

```bash
sudo mdadm /dev/md1 --remove $DISK3
```

On physical hardware this is the hot-swap. The array now runs on two disks with no redundancy.

### Create the replacement disk

```bash
sudo dd if=/dev/zero of=/opt/raid-lab/disk6.img bs=1M count=500
DISK6=$(sudo losetup --find --show /opt/raid-lab/disk6.img)
echo "DISK6=$DISK6" | sudo tee -a /opt/raid-lab/devices.env
```

This simulates pulling a new drive from the parts shelf and sliding it into the bay.

### Add the replacement and time the rebuild

```bash
# Note start time
date

sudo mdadm /dev/md1 --add $DISK6

# Watch the rebuild live
watch -n1 cat /proc/mdstat
```

You'll see a progress bar with a time estimate:

```
[=>................]  recovery =  8.2% (42112/511936) finish=1.2min speed=6420K/sec
```

Note the finish time when `[UUU]` reappears. The delta is your **rebuild time** — log it in `logs/incident-log.md`.

### Verify data survived

```bash
cat /mnt/raid5/testfile.txt
ls -lh /mnt/raid5/bigfile.bin
sudo mdadm --detail /dev/md1
```

`testfile.txt` should read back correctly. `--detail` should show `State : clean` with the new disk as an active member.

---

## Expected State After This Phase

- `/dev/md1` back to `clean`, `[UUU]`, three active members
- `testfile.txt` intact
- `bigfile.bin` present and correct size
- Rebuild time recorded in `logs/incident-log.md`
