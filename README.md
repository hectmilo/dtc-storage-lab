# DTC Storage Lab

A hands-on lab documenting software RAID and LVM operations in Linux — the kind of work a data center technician does when a drive fails on the floor. Built on an Ubuntu Server VM using loopback-backed virtual disks so the full failure-and-recovery cycle can be run safely and repeatedly.

---

## Skills Demonstrated

| Skill | Tool |
|---|---|
| Software RAID assembly (RAID 1 and RAID 5) | `mdadm` |
| Live array monitoring and degraded-state diagnosis | `/proc/mdstat`, `mdadm --detail` |
| Hot-disk failure simulation and replacement | `mdadm --fail`, `mdadm --remove`, `mdadm --add` |
| Rebuild time measurement and data integrity verification | `watch`, `cat` |
| LVM volume group and logical volume management | `pvcreate`, `vgcreate`, `lvcreate` |
| Online filesystem resize without downtime | `lvextend`, `resize2fs` |
| Point-in-time snapshots and rollback | `lvcreate -s`, `lvconvert --merge` |

---

## Lab Structure

```
dtc-storage-lab/
├── setup/
│   └── 01-create-disks.md       # Loopback virtual disks, environment setup
├── raid/
│   ├── 02-raid1.md              # RAID 1 (mirror) build and inspection
│   ├── 03-raid5.md              # RAID 5 (stripe + parity) build and inspection
│   └── 04-failure-rebuild.md    # Disk failure, replacement, and rebuild
├── lvm/
│   └── 05-lvm-on-raid.md        # PV, VG, LV, resize, snapshot, and restore
└── logs/
    └── incident-log.md          # Timestamped recovery record from the failure drill
```

---

## How to Reproduce

**Requirements:** Ubuntu Server VM (22.04+), `mdadm`, `lvm2`, ~3 GB free disk space.

```bash
sudo apt install -y mdadm lvm2
```

Follow each phase doc in order. Every command is explained before it runs. The `logs/incident-log.md` file records what actually happened during the failure drill — rebuild time, degraded state output, and post-recovery verification.

---

## Key Takeaways

- RAID 5 stayed online and served data throughout the entire disk failure — the array ran degraded until the rebuild completed.
- The rebuild window (time between failure and restored redundancy) is the highest-risk period. A second failure during this window causes data loss.
- LVM sits above RAID and is unaware of it — volume resizes and snapshots work identically whether the underlying device is RAID, a single disk, or a cloud block device.
- Snapshots cost no space at creation time; they only consume space as the source volume changes after the snapshot is taken.
