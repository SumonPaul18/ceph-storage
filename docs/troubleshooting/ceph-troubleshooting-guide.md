**File Name:** `ceph-troubleshooting-guide.md`

# Ceph Cluster Troubleshooting & Maintenance Guide
### Practical Handbook for Diagnosing Health Warnings, OSD Crashes, and Daemon Failures in Cephadm Environments

---

## Table of Contents

1. [Introduction to Ceph Health Monitoring](#1-introduction-to-ceph-health-monitoring)
2. [Initial Health Assessment & Diagnosis](#2-initial-health-assessment--diagnosis)
3. [Deep Dive: Analyzing OSD Crashes](#3-deep-dive-analyzing-osd-crashes)
4. [Resolving Failed Daemons & Stale Services](#4-resolving-failed-daemons--stale-services)
5. [Hardware Verification & Disk Management](#5-hardware-verification--disk-management)
6. [Maintenance, Cleanup & Verification](#6-maintenance-cleanup--verification)
7. [Best Practices for Lab & Production Stability](#7-best-practices-for-lab--production-stability)

---

## 1. Introduction to Ceph Health Monitoring

In a modern Cloud Infrastructure environment, particularly when managing a private cloud lab or production storage cluster using **Ceph**, maintaining `HEALTH_OK` is critical. However, warnings (`HEALTH_WARN`) are common during hardware failures, network glitches, or configuration mismatches.

This guide focuses on a real-world scenario where a Ceph cluster reports two specific warnings:
1.  **Failed cephadm daemon(s):** Indicates a service managed by the orchestrator is not running correctly.
2.  **Daemons have recently crashed:** Indicates a process (usually an OSD or Manager) terminated unexpectedly, often due to hardware I/O errors or software assertions.

We will use **Cephadm** (the modern deployment tool for Ceph) and standard Linux system administration tools to diagnose, troubleshoot, and resolve these issues. The focus is on practical, hands-on commands that you can execute directly on your Ceph manager node.

> **Note:** This guide assumes you are running Ceph Quincy (v17), Reef (v18), or Squid (v19) with `cephadm` as the orchestrator.

---

## 2. Initial Health Assessment & Diagnosis

Before making any changes, we must understand the current state of the cluster. The primary tool for this is `ceph -s`.

### 2.1 Checking Cluster Status

Run the following command on your Ceph manager node (e.g., `ceph1`):

```bash
ceph -s
```

**Typical Output with Warnings:**
```text
  cluster:
    id:     81bde713-1aa7-11f1-92f0-bc2411c67839
    health: HEALTH_WARN
            1 failed cephadm daemon(s)
            1 daemons have recently crashed

  services:
    mon: 4 daemons, quorum ceph1,ceph2,ceph3,ceph4
    mgr: ceph2.btupjx(active), standbys: ceph1.gmpclf
    osd: 11 osds: 11 up, 11 in
    rgw: 3 daemons active

  data:
    pools:   10 pools, 289 pgs
    objects: 13.48k objects, 51 GiB
    usage:   157 GiB used, 2.7 TiB / 2.8 TiB avail
    pgs:     289 active+clean
```

**Interpretation:**
*   **HEALTH_WARN:** The cluster is still serving data (PGs are `active+clean`), but there are underlying issues.
*   **1 failed cephadm daemon(s):** A specific service container is down or in an error state.
*   **1 daemons have recently crashed:** A process died and generated a crash dump.

### 2.2 Identifying the Failed Daemon

To find which daemon is failing, we list all orchestrated services.

**Command:**
```bash
ceph orch ps
```

**What to Look For:**
Scan the `STATUS` column for `error`, `failed`, or `stopped`.

**Example Output Analysis:**
```text
NAME                       HOST   STATUS         ...
osd.11                     ceph1  error          ...
crash.ceph1                ceph1  running        ...
mgr.ceph1                  ceph1  running        ...
```
*In our scenario, `osd.11` is in an `error` state.*

### 2.3 Identifying the Crashed Daemon

Ceph stores crash dumps for analysis. We need to list new crashes.

**Command:**
```bash
ceph crash ls-new
```

**Output:**
```text
ID                                                                ENTITY  NEW
2026-04-26T04:54:51.035037Z_3ca345d4-dd8a-4f2e-a26f-ed63147dacb0  osd.13   *
```
*Here, `osd.13` is the entity that crashed.*

---

## 3. Deep Dive: Analyzing OSD Crashes

An OSD crash is serious. It usually indicates a hardware failure (disk death) or a severe kernel issue. We must analyze the crash dump to confirm the root cause.

### 3.1 Inspecting the Crash Dump

Use the crash ID from the previous step to get detailed information.

**Command:**
```bash
ceph crash info <CRASH_ID>
```

**Example:**
```bash
ceph crash info 2026-04-26T04:54:51.035037Z_3ca345d4-dd8a-4f2e-a26f-ed63147dacb0
```

**Key Fields to Analyze in JSON Output:**
1.  **`assert_msg`**: Look for phrases like "Unexpected IO error" or "hardware issue".
2.  **`io_error`**: If `true`, it confirms a disk I/O failure.
3.  **`io_error_devname`**: The device name (e.g., `dm-4`, `sdb`) that failed.
4.  **`process_name`**: Confirms it was `ceph-osd`.

**Sample Analysis:**
```json
{
    "assert_msg": "... Unexpected IO error. This may suggest a hardware issue ...",
    "io_error": true,
    "io_error_devname": "dm-4",
    "entity_name": "osd.13"
}
```
**Conclusion:** OSD.13 crashed because the underlying block device `dm-4` failed to respond to an I/O request. This is a **hardware-level failure**.

### 3.2 Verifying OSD Existence in Cluster Map

Sometimes, a crashed OSD is already removed from the cluster map by automated processes or previous admin actions. We must check if the OSD still exists logically.

**Command:**
```bash
ceph osd tree | grep <OSD_ID>
```

**Example:**
```bash
ceph osd tree | grep 13
```

**Scenario A: OSD Exists**
If it appears in the tree, it needs to be taken out and removed safely.

**Scenario B: OSD Does Not Exist**
If the command returns nothing, the OSD has already been removed from the CRUSH map and OSD map. The crash record is now just historical data.

*In our case, `osd.13` did not exist in the tree, meaning the logical removal happened, but the crash record remained.*

---

## 4. Resolving Failed Daemons & Stale Services

A "failed daemon" in `ceph orch ps` often means `cephadm` is trying to manage a service that is broken or stale. In our case, `osd.11` was in an `error` state.

### 4.1 Diagnosing the Failed Service

Check the logs of the specific daemon to understand why it is failing.

**Command:**
```bash
cephadm logs --name <DAEMON_NAME>
```

**Example:**
```bash
cephadm logs --name osd.11
```

**Common Error Patterns:**
*   `Failed to activate via raw: did not find any matching OSD to activate`: The underlying disk is missing.
*   `Permission denied`: File ownership issues.
*   `Address already in use`: Port conflict.

**Analysis of Our Case:**
The logs showed:
`--> Failed to activate via raw: did not find any matching OSD to activate`
And references to `/dev/dm-4`.

This confirms that `osd.11` is a **stale service**. The OSD was likely removed from the cluster, but the systemd service/container definition remains on the host `ceph1`, trying to start and failing because the disk (`dm-4`) is gone.

### 4.2 Removing Stale Services

If an OSD is no longer in the cluster (`ceph osd tree` does not show it), but `ceph orch ps` shows it as `error`, you must clean up the local service.

#### Step 1: Stop the Systemd Service Manually

Since `ceph orch` might not recognize it as a valid service to stop, use `systemctl`.

**Find the Service Name:**
Cephadm services follow the pattern: `ceph-<FSID>@<DAEMON_TYPE>.<ID>.service`

**Command:**
```bash
systemctl list-units | grep osd.11
```

**Stop and Disable:**
```bash
systemctl stop ceph-81bde713-1aa7-11f1-92f0-bc2411c67839@osd.11
systemctl disable ceph-81bde713-1aa7-11f1-92f0-bc2411c67839@osd.11
```

#### Step 2: Force Remove from Ceph Orchestrator (If applicable)

If the daemon still appears in `ceph orch ps` after stopping the service, try forcing its removal from the orchestrator's view.

**Command:**
```bash
ceph orch daemon rm osd.11 --force
```

*Note: If `ceph orch daemon rm` says "Daemon not found", it means `cephadm` has already dropped it from its internal state, but the `ps` output might be cached. Wait 2-5 minutes for the cache to refresh.*

---

## 5. Hardware Verification & Disk Management

Both `osd.11` and `osd.13` were associated with device `dm-4`. This indicates a physical disk or LVM volume issue on host `ceph1`.

### 5.1 Identifying the Physical Disk

`dm-4` is a Device Mapper entry (likely LVM). We need to find the physical disk behind it.

**Command:**
```bash
lsblk | grep dm-4
```

**Output Example:**
```text
dm-4      253:4    0   30G  0 lvm
```

To see the parent disk:
```bash
lsblk
```
Look for the tree structure. `dm-4` will be indented under a physical disk like `sdb` or `nvme0n1`.

### 5.2 Checking Kernel Logs for Hardware Errors

If the disk is failing, the Linux kernel will log I/O errors.

**Command:**
```bash
dmesg -T | grep -i "error" | tail -n 20
```

**Or specifically for the disk:**
```bash
journalctl -k --since "1 hour ago" | grep -i "sdX"
```
*(Replace `sdX` with your actual disk identifier)*

**Critical Signs:**
*   `I/O error, dev sdX, sector ...`
*   `Buffer I/O error on dev dm-4`
*   `Device offlined`

If you see these, the disk is physically dead or disconnected. **Do not attempt to reuse it.**

### 5.3 Cleaning Up LVM Metadata

If you remove a disk that was part of an LVM group, stale metadata can cause issues for future OSD deployments.

**Commands:**
```bash
vgscan
lvscan
```

If you see "inactive" volumes related to the failed OSD, you can remove them manually (use with caution):

```bash
lvremove /dev/ceph-<VG-ID>/osd-block-<OSD-ID>
vgremove ceph-<VG-ID>
```

*Note: Only do this if you are sure the OSD is permanently removed and the data is lost.*

---

## 6. Maintenance, Cleanup & Verification

Once the root causes (hardware failure and stale services) are addressed, we must clear the warnings from the Ceph cluster status.

### 6.1 Archiving Crash Dumps

Ceph keeps crash dumps until they are "archived". This is a safety feature to ensure admins review them. To clear the `HEALTH_WARN` about crashes:

**Command:**
```bash
ceph crash archive-all
```

**Or archive a specific ID:**
```bash
ceph crash archive <CRASH_ID>
```

### 6.2 Verifying Cluster Health

After archiving crashes and stopping stale services, wait for the manager to update the status (usually 1-2 minutes).

**Command:**
```bash
ceph -s
```

**Expected Output:**
```text
  cluster:
    id:     ...
    health: HEALTH_OK
```

If `HEALTH_OK` is achieved, the issue is resolved.

### 6.3 Re-balancing Data (If OSD was Removed)

If you permanently removed an OSD (like OSD.13), the data that was on it has been replicated to other OSDs. The cluster might be slightly unbalanced.

**Check Balance:**
```bash
ceph osd df
```

**Trigger Re-balance (Optional):**
Ceph automatically re-balances, but you can tune the backfill speed if needed. Usually, no manual action is required unless the cluster is heavily loaded.

---

## 7. Best Practices for Lab & Production Stability

### 7.1 Regular Health Checks

Implement a daily cron job or monitoring alert for `ceph health`.

**Example Check:**
```bash
ceph health detail
```
This provides more context than `ceph -s`.

### 7.2 Hardware Monitoring

Since Ceph is software-defined storage, it relies entirely on underlying hardware health.
*   Use **SMART** tools to monitor disk health:
    ```bash
    smartctl -a /dev/sdX
    ```
*   Monitor kernel logs for I/O errors.
*   In a virtualized lab (like Proxmox or VMware), ensure virtual disks are backed by reliable physical storage.

### 7.3 Managing Cephadm Services

*   **Avoid Manual Systemctl Changes:** Always prefer `ceph orch` commands to start/stop/remove services. Manual `systemctl` changes can desynchronize `cephadm`'s state.
*   **Use `ceph orch ps` Frequently:** This is your source of truth for what `cephadm` is managing.
*   **Clean Up Stale Devices:** If you remove a disk from a server, ensure you run `ceph orch osd rm <ID> --zap` to cleanly wipe the device before physically removing it.

### 7.4 Handling "Ghost" OSDs

If an OSD disappears from the cluster but remains in `ceph orch ps`:
1.  Check `ceph osd tree`.
2.  If not in tree, stop the systemd service manually.
3.  Run `ceph orch daemon ls` to refresh state.
4.  If it persists, restart the `ceph-mgr` module:
    ```bash
    ceph mgr module disable orch
    ceph mgr module enable orch
    ```
    *(Use this as a last resort)*

### 7.5 Documentation & Logging

Keep a log of all OSD replacements and crashes.
*   Date of failure.
*   OSD ID.
*   Serial Number of the failed disk.
*   Root cause (e.g., "I/O Error", "Kernel Panic").

This helps in identifying patterns, such as a bad batch of disks or a faulty SATA controller.

---

## Appendix: Quick Reference Commands

| Task | Command |
| :--- | :--- |
| Check Cluster Health | `ceph -s` |
| List All Daemons | `ceph orch ps` |
| List New Crashes | `ceph crash ls-new` |
| Inspect Crash Details | `ceph crash info <ID>` |
| Archive All Crashes | `ceph crash archive-all` |
| Remove OSD (Safe) | `ceph orch osd rm <ID> --zap` |
| Stop Stale Service | `systemctl stop ceph-<FSID>@<DAEMON>` |
| Check Disk Health | `smartctl -a /dev/sdX` |
| View Kernel Logs | `dmesg -T \| grep -i error` |
| List Block Devices | `lsblk` |

---

## Conclusion

Troubleshooting Ceph health warnings requires a systematic approach:
1.  **Identify** the warning source (`ceph -s`).
2.  **Diagnose** the specific daemon (`ceph orch ps`, `ceph crash ls-new`).
3.  **Analyze** the root cause (Logs, Crash Info, Hardware checks).
4.  **Resolve** the issue (Remove stale services, replace hardware).
5.  **Clean Up** (Archive crashes, verify health).

By following this guide, you can maintain a healthy, stable Ceph cluster in both lab and production environments. Remember, most OSD crashes are hardware-related. Always prioritize checking physical disk health before assuming software bugs.