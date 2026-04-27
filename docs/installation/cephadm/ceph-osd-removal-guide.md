# File Name: `ceph-osd-removal-guide.md`

---

# Ceph OSD Removal and Disk Cleanup Guide

## Table of Contents
1. [Prerequisites](#1-prerequisites)
2. [Step 1: Mark OSD as Out](#2-step-1-mark-osd-as-out)
3. [Step 2: Stop the OSD Daemon](#3-step-2-stop-the-osd-daemon)
4. [Step 3: Remove OSD from Cluster](#4-step-3-remove-osd-from-cluster)
5. [Step 4: Clean Up Authentication and CRUSH Map](#5-step-4-clean-up-authentication-and-crush-map)
6. [Step 5: Zap and Wipe the Disk](#6-step-5-zap-and-wipe-the-disk)
7. [Step 6: Verify Removal](#7-step-6-verify-removal)

---

## 1. Prerequisites
Ensure you have root access or sudo privileges on the Ceph admin node. Replace `<osd_id>` with the actual OSD ID (e.g., `11`) and `<device_path>` with the actual disk path (e.g., `/dev/vdd`).

#### List all OSD_ID
```
ceph osd tree
```
#### Find out the specefic OSD Disk DEVICE_PATH
```
lsblk
```
#### Verify to HOST_NAME
```
hostname
```

## 2. Step 1: Mark OSD as Out
This step stops data from being written to the OSD and triggers data rebalancing to other OSDs in the cluster.

```bash
ceph osd out [Only_OSD_ID]
```
> Example: ceph osd out 12

**Explanation:**
*   `ceph osd out`: Marks the specified OSD as "out" of the cluster.
*   `$OSD_ID`: The ID of the OSD to be removed (e.g., `12`).
*   **Note:** Wait for the cluster to finish rebalancing (`ceph -s` should show `HEALTH_OK` or no active backfilling) before proceeding if possible, though forced removal is also supported.

## 3. Step 2: Stop the OSD Daemon
Stop the running OSD service to ensure it is not actively processing requests.

```bash
ceph orch daemon stop [OSD.OSD_ID]
```
> Example: ceph orch daemon stop osd.12

**Explanation:**
*   `ceph orch daemon stop`: Stops the specific daemon managed by cephadm.
*   `osd.$OSD_ID`: Specifies the OSD daemon to stop.

## 4. Step 3: Remove OSD from Cluster
This command removes the OSD from the Ceph orchestration and the cluster map.

```bash
ceph orch osd rm [Only_OSD_ID]
```
> Example: ceph orch osd rm 12

**Explanation:**
*   `ceph orch osd rm`: Initiates the removal process via the orchestrator.
*   `$OSD_ID`: The ID of the OSD to remove.
*   **Note:** If the OSD is stuck, you may need to use `--force` (use with caution):
    ```bash
    ceph orch osd rm $OSD_ID --force
    ```

## 5. Step 4: Clean Up Authentication and CRUSH Map
If the orchestrator removal does not fully clean up the authentication keys or CRUSH map entries, perform these steps manually.

```bash
ceph osd crush remove [OSD.OSD_ID]
```
```
ceph auth del [OSD.OSD_ID]
```
```
ceph osd rm [Only_OSD_ID]
```
> Replace to your actual OSD ID

**Explanation:**
*   `ceph osd crush remove`: Removes the OSD from the CRUSH hierarchy.
*   `ceph auth del`: Deletes the authentication credentials for the OSD.
*   `ceph osd rm`: Permanently removes the OSD ID from the cluster map.

## 6. Step 5: Zap and Wipe the Disk
This step wipes the partition table and any remaining Ceph metadata from the physical disk, making it available for reuse.

```bash
ceph orch device zap $HOST_NAME $DEVICE_PATH --force
```
> Example: ceph orch device zap ceph1 /dev/vdd --force

**Explanation:**
*   `ceph orch device zap`: Wipes the device signature.
*   `$HOST_NAME`: The hostname where the disk is located (e.g., `ceph1`).
*   `$DEVICE_PATH`: The path to the disk (e.g., `/dev/vdd`).
*   `--force`: Forces the wipe even if the device is perceived as in use.

**Alternative Manual Zap (if orchestrator fails):**
```bash
ceph-volume lvm zap $DEVICE_PATH --destroy
```
> Example: ceph-volume lvm zap /dev/vdd --destroy

## 7. Step 6: Verify Removal
Confirm that the OSD is no longer present in the cluster and the disk is clean.

```bash
ceph osd tree
```
```
lsblk -f $DEVICE_PATH
```
```
ceph orch device ls --refresh
```

**Explanation:**
*   `ceph osd tree`: Checks if the OSD ID is absent from the tree.
*   `lsblk -f`: Verifies that the disk has no filesystem or LVM signatures.
*   `ceph orch device ls --refresh`: Confirms the device is listed as "Available" for new OSD deployment.

---

### Quick Copy-Paste Block (For OSD 11 on /dev/vdd)


#### 1. Mark Out
```
ceph osd out 11
```
#### 2. Stop Daemon
```
ceph orch daemon stop osd.11
```
#### 3. Remove via Orchestrator
```
ceph orch osd rm 11
```
#### 4. Manual Cleanup (If needed)
```
ceph osd crush remove osd.11
ceph auth del osd.11
ceph osd rm 11
```
#### 5. Zap Disk
```
ceph orch device zap ceph1 /dev/vdd --force
```
#### 6. Verify
```
ceph osd tree
lsblk -f /dev/vdd
```

---
---

The error `(16) Device or resource busy` indicates that the Logical Volume (LV) associated with the OSD is still active, mounted, or locked by the system (often due to `dm-crypt` encryption layers or lingering LVM locks). The `ceph-volume zap` command inside the container cannot wipe a device that the host kernel considers "in use."

Here is the step-by-step guide to forcefully clean this specific disk.

# File Name: `ceph-osd-force-zap-fix.md`

---

# Fixing "Device or Resource Busy" During Ceph OSD Zap

## Table of Contents
1. [Diagnosis: Identify Active Handles](#1-diagnosis-identify-active-handles)
2. [Step 1: Deactivate LVM and Crypto Layers](#2-step-1-deactivate-lvm-and-crypto-layers)
3. [Step 2: Kill Remaining Processes](#3-step-2-kill-remaining-processes)
4. [Step 3: Force Remove LVM Metadata](#4-step-3-force-remove-lvm-metadata)
5. [Step 4: Wipe Disk Signatures](#5-step-4-wipe-disk-signatures)
6. [Step 5: Verify and Re-add](#6-step-5-verify-and-re-add)

---

## 1. Diagnosis: Identify Active Handles
Before forcing changes, identify what is holding the device `/dev/vdd` or its LVM partitions busy.

**Commands:**
```bash
lsblk -f /dev/vdd
dmsetup ls | grep vdd
lvs | grep vdd
fuser -v /dev/vdd
```

**Explanation:**
*   `lsblk -f`: Shows filesystems and LVM structures on the disk.
*   `dmsetup ls`: Lists device-mapper entries (critical for encrypted OSDs).
*   `lvs`: Lists logical volumes.
*   `fuser`: Identifies processes using the device.

## 2. Step 1: Deactivate LVM and Crypto Layers
If the OSD was encrypted (`dmcrypt`), you must close the crypto layer and deactivate the LVM volume group before zapping.

**Commands:**

#### 1. Find the LV path from previous output (e.g., /dev/ceph-xxxx/osd-block-xxxx)
#### 2. Deactivate the Logical Volume
```
lvchange -an /dev/ceph-4e0b8bbd-ade6-43ca-b258-fabd3573be8d/osd-block-fc90d120-400d-4f38-b005-d9026403695b
```
#### 3. If encrypted, close the LUKS mapper (replace <mapper_name> with actual name from dmsetup ls)
```
cryptsetup luksClose /dev/mapper/ceph-4e0b8bbd-ade6-43ca-b258-fabd3573be8d-osd-block-fc90d120-400d-4f38-b005-d9026403695b
```
#### 4. Remove the device-mapper entry if it persists
```
dmsetup remove /dev/mapper/ceph-4e0b8bbd-ade6-43ca-b258-fabd3573be8d-osd-block-fc90d120-400d-4f38-b005-d9026403695b
```
**If not work above command, Then try the bellow command**
```
dmsetup remove -f ceph--4e0b8bbd--ade6--43ca--b258--fabd3573be8d-osd--block--fc90d120--400d--4f38--b005--d9026403695b
```
**Explanation:**
*   `lvchange -an`: Deactivates the LV so it is no longer accessible.
*   `cryptsetup luksClose`: Closes the encrypted container.
*   `dmsetup remove`: Forces removal of the device-mapper node if it's stuck.

## 3. Step 2: Kill Remaining Processes
If any process (like `ceph-osd` or `podman`) is still holding the file descriptor, kill it.

**Commands:**

#### Kill any process using the device
```
fuser -km /dev/vdd
```
#### Ensure no ceph-osd daemon is running for this ID
```
ps aux | grep osd.11
```
```
kill -9 <PID>
```

**Explanation:**
*   `fuser -km`: Kills all processes accessing the file/device.
*   `kill -9`: Forcefully terminates the specific OSD process if it didn't stop gracefully.

## 4. Step 3: Force Remove LVM Metadata
Now that the device is inactive, remove the LVM metadata signatures directly from the host (not inside the container).

**Commands:**

#### Wipe the LVM signature from the physical volume
```
pvremove -ff /dev/vdd
```
#### If the above fails because it's part of a VG, remove the VG first
```
vgremove -f ceph-4e0b8bbd-ade6-43ca-b258-fabd3573be8d
```
#### Then retry pvremove
```
pvremove -ff /dev/vdd
```

**Explanation:**
*   `pvremove -ff`: Forces the removal of LVM physical volume labels. The second `-f` ignores confirmation prompts.
*   `vgremove -f`: Removes the Volume Group if it still exists.

## 5. Step 4: Wipe Disk Signatures
Use `wipefs` to clear all remaining filesystem, RAID, and partition table signatures. This is often more effective than `ceph-volume zap` for stuck disks.

**Commands:**

#### Wipe all signatures
```
wipefs -a /dev/vdd
```
#### Zero out the first 10MB to ensure clean slate (optional but recommended)
```
dd if=/dev/zero of=/dev/vdd bs=1M count=10 status=progress
```
#### Inform the kernel of partition table changes
```
partprobe /dev/vdd
```

**Explanation:**
*   `wipefs -a`: Removes all magic strings (signatures) from the device.
*   `dd if=/dev/zero`: Overwrites the beginning of the disk to destroy any residual boot sectors or partition tables.
*   `partprobe`: Updates the kernel's view of the disk.

## 6. Step 5: Verify and Re-add
Confirm the disk is clean and ready for Ceph to reuse.

**Commands:**

#### Verify disk is clean
```
lsblk -f /dev/vdd
udevadm settle
```
#### Refresh Ceph device list
```
ceph orch device ls --refresh
```
#### The disk should now appear as "Available" in Ceph


**Explanation:**
*   `lsblk -f`: Should show no FSTYPE or MOUNTPOINT for `/dev/vdd`.
*   `udevadm settle`: Ensures udev rules have processed the changes.
*   `ceph orch device ls --refresh`: Forces Ceph to rescan hosts for available storage.

---

### Quick Copy-Paste Block (Force Clean /dev/vdd)

```bash
# 1. Deactivate LVM/Crypto (Adjust LV name if different)
lvchange -an /dev/ceph-4e0b8bbd-ade6-43ca-b258-fabd3573be8d/osd-block-fc90d120-400d-4f38-b005-d9026403695b 2>/dev/null
dmsetup remove /dev/mapper/ceph-4e0b8bbd-ade6-43ca-b258-fabd3573be8d-osd-block-fc90d120-400d-4f38-b005-d9026403695b 2>/dev/null

# 2. Kill processes
fuser -km /dev/vdd

# 3. Remove LVM/PG signatures
vgremove -f ceph-4e0b8bbd-ade6-43ca-b258-fabd3573be8d 2>/dev/null
pvremove -ff /dev/vdd 2>/dev/null

# 4. Wipe disk
wipefs -a /dev/vdd
dd if=/dev/zero of=/dev/vdd bs=1M count=10 status=progress

# 5. Refresh Ceph
udevadm settle
ceph orch device ls --refresh
```

---
---


The error `Can't open /dev/vdd exclusively` and `Device or resource busy` confirms that the kernel still has an active handle on the block device. This is common with encrypted OSDs where the `dm-crypt` layer or a lingering `ceph-osd` process holds the device lock, preventing `wipefs` or `pvremove` from writing to it.

Here is the aggressive cleanup guide to force-release the device.

# File Name: `ceph-disk-force-unlock.md`

---

# Force Unlock and Wipe Busy Ceph Disk

## Table of Contents
1. [Step 1: Identify and Kill Holding Processes](#1-step-1-identify-and-kill-holding-processes)
2. [Step 2: Remove Device-Mapper (Crypto/LVM) Links](#2-step-2-remove-device-mapper-cryptolvm-links)
3. [Step 3: Force Wipe Signatures](#3-step-3-force-wipe-signatures)
4. [Step 4: Verify Clean State](#4-step-4-verify-clean-state)

---

## 1. Step 1: Identify and Kill Holding Processes
Find exactly which process is holding `/dev/vdd` open and kill it.

**Commands:**
```bash
# Find processes using the disk
fuser -v /dev/vdd

# Kill all processes using the disk forcefully
fuser -km /dev/vdd

# Check for any remaining ceph-osd processes specifically
ps aux | grep "[o]sd.*vdd"
kill -9 <PID>
```

**Explanation:**
*   `fuser -v`: Verbose mode shows which user/process is accessing the file.
*   `fuser -km`: Kills (`-k`) all processes accessing the file (`-m`).
*   `kill -9`: Sends SIGKILL to any stubborn OSD daemon processes that didn't terminate gracefully.

## 2. Step 2: Remove Device-Mapper (Crypto/LVM) Links
If the disk was encrypted, the device-mapper table must be removed. Even if `lvchange` failed earlier, `dmsetup` can often force removal.

**Commands:**
```bash
# List all device-mapper entries related to ceph
dmsetup ls | grep ceph

# Remove the specific mapper entry (replace with actual name from above)
dmsetup remove --force /dev/mapper/ceph-4e0b8bbd-ade6-43ca-b258-fabd3573be8d-osd-block-fc90d120-400d-4f38-b005-d9026403695b

# If that fails, try removing by ID
dmsetup remove --force <mapper-name>

# Verify no dm entries remain for this disk
dmsetup ls | grep vdd
```

**Explanation:**
*   `dmsetup remove --force`: Forces the removal of the device-mapper node, breaking the link between the encrypted container and the physical disk.
*   **Note:** If `dmsetup` says "device is busy," repeat Step 1 (`fuser -km`) on the *mapper* device path (e.g., `/dev/mapper/...`).

## 3. Step 3: Force Wipe Signatures
Once the kernel locks are released, use `dd` to overwrite the partition table and signatures. `dd` is lower-level than `wipefs` and often succeeds when `wipefs` fails due to minor locking issues.

**Commands:**
```bash
# Overwrite the first 100MB with zeros (destroys partition table, LVM headers, and crypto headers)
dd if=/dev/zero of=/dev/vdd bs=1M count=100 status=progress

# Sync changes to disk
sync

# Inform kernel of partition table changes
partprobe /dev/vdd
```

**Explanation:**
*   `dd if=/dev/zero`: Writes zeros directly to the block device, bypassing filesystem layers.
*   `bs=1M count=100`: Writes 100MB of zeros, which is sufficient to destroy all standard headers (GPT, MBR, LVM, LUKS).
*   `partprobe`: Forces the kernel to re-read the partition table (now empty).

## 4. Step 4: Verify Clean State
Confirm the disk is no longer busy and has no signatures.

**Commands:**
```bash
# Check for remaining signatures
wipefs /dev/vdd

# Check block structure
lsblk -f /dev/vdd

# Refresh Ceph inventory
ceph orch device ls --refresh
```

**Explanation:**
*   `wipefs /dev/vdd`: Should return **no output** if clean. If it lists types, repeat Step 3.
*   `lsblk -f`: Should show no FSTYPE or MOUNTPOINT.
*   `ceph orch device ls --refresh`: Ceph should now list `/dev/vdd` as `Available`.

---

### Quick Copy-Paste Block (Aggressive Clean)

```bash
# 1. Kill holders
fuser -km /dev/vdd
sleep 2

# 2. Force remove DM mappings (Ignore errors if none exist)
for dev in $(dmsetup ls | grep ceph | awk '{print $1}'); do
    dmsetup remove --force $dev 2>/dev/null
done
sleep 2

# 3. Zero out headers
dd if=/dev/zero of=/dev/vdd bs=1M count=100 status=progress
sync
partprobe /dev/vdd

# 4. Verify
wipefs /dev/vdd
lsblk -f /dev/vdd
```

---
---

