# Troubleshooting Guide: Resolving "Down" OSD on a Running Node

This guide provides a professional, step-by-step approach to diagnosing and fixing a Ceph OSD that appears **DOWN** even though the host machine is physically **UP**.

---

## Table of Contents
1.  [Understanding the Root Causes](#1-understanding-the-root-causes)
2.  [Phase 1: Service Level Investigation](#2-phase-1-service-level-investigation)
3.  [Phase 2: Hardware & Storage Validation](#3-phase-2-hardware--storage-validation)
4.  [Phase 3: Network & Communication Check](#4-phase-3-network--communication-check)
5.  [Phase 4: Log Analysis & Crash Recovery](#5-phase-4-log-analysis--crash-recovery)
6.  [Real-World Scenarios & Common Issues](#6-real-world-scenarios--common-issues)

---

## 1. Understanding the Root Causes
In your specific case (OSD.13 on ceph4), the node is alive, but the OSD is down. This usually happens due to:
*   **Process Crash:** The `ceph-osd` daemon stopped due to a bug or OOM (Out of Memory).
*   **Disk Failure:** The underlying drive is experiencing I/O errors or has gone read-only.
*   **Authentication Issues:** The OSD's keyring is mismatched or corrupted.
*   **XFS/BlueStore Corruption:** Filesystem level errors preventing the OSD from booting.

---

## 2. Phase 1: Service Level Investigation
First, log into `ceph4` and check if the daemon is actually running.

### Step 1: Check Status
```bash
ssh ceph4
systemctl status ceph-osd@13
```

### Step 2: Attempt Restart
If it is `inactive` or `failed`, try to bring it back up:
```bash
systemctl restart ceph-osd@13
```
*Wait 30 seconds and check `ceph osd tree` again.*

---

## 3. Phase 2: Hardware & Storage Validation
If the service fails to start or crashes immediately, the problem is likely the physical disk or the mount point.

### Step 3: Check Disk Visibility
```bash
lsblk
```
Look for the disk associated with OSD.13. If the device (e.g., `/dev/sdb`) is missing from the list, the drive has physically failed or disconnected.

### Step 4: Check for Kernel Errors
Check the kernel log for "I/O errors" or "SATA link down":
```bash
dmesg | grep -iE "sd|ata|scsi" | tail -n 20
```

---

## 4. Phase 3: Network & Communication Check
Ceph OSDs communicate over a **Public Network** (client traffic) and a **Cluster Network** (replication).

### Step 5: Verify Ports
Ensure the OSD can bind to its ports (usually 6800+):
```bash
netstat -tulpn | grep ceph-osd
```

### Step 6: Firewall Check
If you recently updated the node, verify the firewall isn't blocking Ceph:
```bash
ufw status  # or 'firewall-cmd --list-all'
```

---

## 5. Phase 4: Log Analysis & Crash Recovery
The logs will tell you exactly why the OSD won't stay "UP".

### Step 7: Read OSD Logs
```bash
tail -n 50 /var/log/ceph/ceph-osd.13.log
```
*   **Error "Permission Denied":** Check ownership of `/var/lib/ceph/osd/`.
*   **Error "BlueStore::mount failed":** The database is corrupted.
*   **Error "Heartbeat timeout":** Network is too slow or congested.

### Step 8: Fix "Reweight 0" Issue
In your output, `osd.13` has a `REWEIGHT` of **0**. Once the OSD is "up", you must tell Ceph to use it again:
```bash
ceph osd status
ceph osd reweight 13 1.0
```

---

## 6. Real-World Scenarios & Common Issues

| Scenario | Typical Cause | Practical Solution |
| :--- | :--- | :--- |
| **OSD flapping (Up/Down)** | High network latency or disk saturation. | Check `iostat -x` for disk bottlenecks. |
| **Out of Memory (OOM)** | Node RAM is too low for the number of OSDs. | Add Swap or increase RAM; check `osd_memory_target`. |
| **XFS Corruption** | Sudden power loss on ceph4. | Run `xfs_repair` on the partition (if using FileStore). |
| **Clock Skew** | Node time is not synced with the Monitor. | Restart `chronyd` or `ntp` service. |
| **Full Disk** | The OSD reached the `full_ratio` (95%). | Add more disks or delete unnecessary data to trigger rebalancing. |

---
**Note:** Since your `osd.13` is significantly larger (7.27TB) than your other OSDs (~0.29TB), it handles a massive amount of the cluster's data. If this OSD stays down, your cluster health will degrade quickly. Priority should be given to checking the health of this specific high-capacity drive.
