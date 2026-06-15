# Incident Log — RAID 5 Disk Failure and Recovery

This log records the failure drill performed in Phase 4. Fill in your actual values where indicated.

---

## Incident Summary

| Field | Value |
|---|---|
| Date | <!-- e.g. 2026-06-14 --> |
| Array | `/dev/md1` (RAID 5, 3 × 500 MB loop devices) |
| Failed disk | <!-- e.g. /dev/loop6 (disk3.img) --> |
| Replacement disk | <!-- e.g. /dev/loop9 (disk6.img) --> |
| Rebuild start | <!-- timestamp from `date` before mdadm --add --> |
| Rebuild finish | <!-- timestamp when /proc/mdstat showed [UUU] --> |
| Rebuild duration | <!-- e.g. 1m 42s --> |
| Data integrity | <!-- PASS / FAIL + notes --> |

---

## Timeline

### Failure introduced

```
# Paste output of: sudo mdadm /dev/md1 --fail $DISK3
```

### Degraded state confirmed

```
# Paste output of: cat /proc/mdstat  (while degraded)
```

```
# Paste output of: sudo mdadm --detail /dev/md1  (while degraded)
```

### Failed disk removed

```
# Paste output of: sudo mdadm /dev/md1 --remove $DISK3
```

### Replacement added — rebuild started

```
# Paste output of: sudo mdadm /dev/md1 --add $DISK6
```

### Rebuild progress (sample)

```
# Paste a snapshot of /proc/mdstat during rebuild
# e.g.:
# md1 : active raid5 loop9[3] loop8[1] loop7[0]
#       1023488 blocks super 1.2 level 5, 512k chunk
#       [3/3] [UUU]
#             resync=DELAYED
#       [=>................]  recovery = 14.3% finish=1.1min speed=7200K/sec
```

### Rebuild complete

```
# Paste final /proc/mdstat showing [UUU] with no recovery line
```

### Data integrity verification

```
# Paste output of:
#   cat /mnt/raid5/testfile.txt
#   ls -lh /mnt/raid5/bigfile.bin
#   sudo mdadm --detail /dev/md1
```

---

## Notes

<!-- Anything that surprised you, went differently than expected, or took longer than the estimate -->
