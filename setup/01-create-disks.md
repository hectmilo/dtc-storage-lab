# Phase 1 — Environment Setup

**Learning goal:** Understand how Linux represents block devices, and how to simulate real disks safely using loopback files.

---

## Background

The `loop` driver in Linux lets you attach any regular file as a block device. The kernel treats `/dev/loop*` exactly like a physical drive — you can partition it, format it, and build RAID arrays on it. This is the same mechanism used inside VMs and container runtimes.

We create five virtual disks:
- `disk1.img` + `disk2.img` → RAID 1 (Phase 2)
- `disk3.img` + `disk4.img` + `disk5.img` → RAID 5 (Phase 3)
- `disk6.img` → replacement drive for the failure drill (Phase 4)

---

## Commands

### Install tools

```bash
sudo apt update && sudo apt install -y mdadm lvm2
```

`mdadm` manages software RAID arrays. `lvm2` is the Logical Volume Manager.

### Create a working directory

```bash
sudo mkdir -p /opt/raid-lab
```

### Create five 500 MB disk image files

```bash
for i in 1 2 3 4 5; do
    sudo dd if=/dev/zero of=/opt/raid-lab/disk${i}.img bs=1M count=500
done
```

`dd` copies raw bytes. `if=/dev/zero` is an infinite source of zero bytes. Each file is 500 MB.

### Check which loop devices are already in use

```bash
losetup -l
```

Ubuntu uses loop devices for snap packages. This shows what's taken before attaching ours.

### Attach each image file as a loop device

```bash
DISK1=$(sudo losetup --find --show /opt/raid-lab/disk1.img)
DISK2=$(sudo losetup --find --show /opt/raid-lab/disk2.img)
DISK3=$(sudo losetup --find --show /opt/raid-lab/disk3.img)
DISK4=$(sudo losetup --find --show /opt/raid-lab/disk4.img)
DISK5=$(sudo losetup --find --show /opt/raid-lab/disk5.img)

printf "DISK1=%s\nDISK2=%s\nDISK3=%s\nDISK4=%s\nDISK5=%s\n" \
  "$DISK1" "$DISK2" "$DISK3" "$DISK4" "$DISK5" \
  | sudo tee /opt/raid-lab/devices.env
```

`--find` picks the next free loop device. `--show` prints the name it chose. We save all paths to `devices.env` so they survive a terminal restart.

### Verify the kernel sees them

```bash
lsblk $DISK1 $DISK2 $DISK3 $DISK4 $DISK5
```

Expected output: five block devices, each 500M, no partitions or filesystems.

---

## Expected State After This Phase

- Five files under `/opt/raid-lab/`: `disk1.img` through `disk5.img`
- Five loop devices attached and visible in `lsblk`
- `/opt/raid-lab/devices.env` written with device paths
