# 🛠️ Ceph OSD Engineering: LVM & Dmcrypt Practical Handbook

This guide provides a step-by-step narrative for managing Ceph Object Storage Daemons (OSDs) with LVM and Encryption (dm-crypt). It covers environment preparation, safe removal of existing OSDs, troubleshooting "busy" device errors, and the manual creation/activation of encrypted storage.

---

## 📋 Table of Contents

1. [Phase 1: Environment Readiness & Prerequisites](#1-phase-1-environment-readiness--prerequisites)
2. [Phase 2: Safe OSD Removal & Disk Cleanup](#2-phase-2-safe-osd-removal--disk-cleanup)
3. [Phase 3: Troubleshooting "Device Busy" Errors](#3-phase-3-troubleshooting-device-busy-errors)
4. [Phase 4: Manual Encrypted OSD Creation](#4-phase-4-manual-encrypted-osd-creation)
5. [Phase 5: Activation, Validation & Maintenance](#5-phase-5-activation-validation--maintenance)

---

## 1. Phase 1: Environment Readiness & Prerequisites

Before manipulating storage, ensure the host has the necessary binaries, correct paths, and authentication keys. This prevents common "command not found" or "permission denied" errors.

### 1.1 Install Ceph OSD Binaries
`ceph-volume` requires specific binaries to format and mount disks. Ensure they are installed.

```bash
apt-get update && apt-get install ceph-osd -y
```
> **Explanation:** Updates package lists and installs the Ceph OSD daemon and helper tools.
> *   `apt-get update`: Refreshes local package index.
> *   `install ceph-osd`: Installs the core OSD binary (`ceph-osd`) and tools like `ceph-bluestore-tool`.
> *   `-y`: Automatically confirms installation prompts.

### 1.2 Verify Binary Existence
Confirm that the system can locate the executables.

```bash
which ceph-osd
which ceph-bluestore-tool
```
> **Explanation:** Checks if the binaries are in the system PATH.
> *   **Expected Output:** `/usr/bin/ceph-osd` and `/usr/bin/ceph-bluestore-tool`.
> *   **Troubleshooting:** If empty, re-run the installation in Step 1.1.

### 1.3 Fix PATH Variables
Ensure your shell session includes standard system directories.

```bash
export PATH=$PATH:/usr/bin:/usr/sbin:/sbin:/bin
```
> **Explanation:** Appends standard binary directories to the current session's PATH.
> *   **Why:** Minimal shells or restricted sudo sessions may exclude `/sbin` or `/usr/sbin`, causing `ceph-volume` to fail finding `cryptsetup` or `lvm`.

### 1.4 Resolve Keyring Permission Error
`ceph-volume` needs the `client.bootstrap-osd` key to authenticate with the cluster when creating new OSDs.

#### Step A: Create Directory
```bash
mkdir -p /var/lib/ceph/bootstrap-osd/
```
> **Explanation:** Ensures the standard directory for bootstrap keys exists.

#### Step B: Extract and Place Keyring
```bash
ceph auth get client.bootstrap-osd > /var/lib/ceph/bootstrap-osd/ceph.keyring
```
> **Explanation:** Retrieves the bootstrap key from the Monitor and saves it locally.
> *   `ceph auth get`: Fetches the authentication key from the cluster.
> *   `> .../ceph.keyring`: Saves it to the path where `ceph-volume` automatically looks for credentials.

---

## 2. Phase 2: Safe OSD Removal & Disk Cleanup

If you are replacing a disk or fixing a failed OSD, you must remove it cleanly from the cluster before wiping the physical drive.

### 2.1 Identify Target OSD and Disk
Get the OSD ID and the corresponding physical device path.

```bash
ceph osd tree
lsblk -o NAME,TYPE,FSTYPE,MOUNTPOINT,SIZE
```
> **Explanation:**
> *   `ceph osd tree`: Lists all OSDs, their IDs, status (up/down), and host mapping.
> *   `lsblk`: Shows physical disks. Look for the disk (e.g., `/dev/vdd`) associated with the OSD you want to remove.

### 2.2 Mark OSD as Out
Stop data writes to the OSD and trigger rebalancing.

```bash
ceph osd out <OSD_ID>
```
> **Example:** `ceph osd out 11`
> **Explanation:**
> *   `ceph osd out`: Marks the OSD as "out" of the CRUSH map.
> *   `<OSD_ID>`: Replace with the actual ID (e.g., `11`).
> *   **Note:** Wait for `ceph -s` to show `HEALTH_OK` (rebalancing complete) before proceeding, if possible.

### 2.3 Stop the OSD Daemon
Gracefully stop the service.

```bash
ceph orch daemon stop osd.<OSD_ID>
```
> **Example:** `ceph orch daemon stop osd.11`
> **Explanation:**
> *   `ceph orch daemon stop`: Stops the specific container/process managed by cephadm.
> *   `osd.<OSD_ID>`: Specifies the target daemon.

### 2.4 Remove OSD from Cluster
Remove the OSD from the orchestration and cluster map.

```bash
ceph orch osd rm <OSD_ID> --force
```
> **Example:** `ceph orch osd rm 11 --force`
> **Explanation:**
> *   `ceph orch osd rm`: Initiates removal via the orchestrator.
> *   `--force`: Forces removal even if the OSD is down or stuck.

### 2.5 Clean Up Authentication and CRUSH Map
If the orchestrator doesn't fully clean up, remove entries manually.

```bash
ceph osd crush remove osd.<OSD_ID>
ceph auth del osd.<OSD_ID>
ceph osd rm <OSD_ID>
```
> **Example:**
> ```bash
> ceph osd crush remove osd.11
> ceph auth del osd.11
> ceph osd rm 11
> ```
> **Explanation:**
> *   `crush remove`: Removes the OSD from the hierarchy.
> *   `auth del`: Deletes authentication credentials.
> *   `osd rm`: Permanently removes the ID from the cluster map.

### 2.6 Zap and Wipe the Disk
Clean the physical disk of all metadata.

```bash
ceph orch device zap <HOSTNAME> <DEVICE_PATH> --force
```
> **Example:** `ceph orch device zap ceph1 /dev/vdd --force`
> **Explanation:**
> *   `ceph orch device zap`: Wipes partition tables and signatures.
> *   `<HOSTNAME>`: The node where the disk resides.
> *   `<DEVICE_PATH>`: The disk to wipe (e.g., `/dev/vdd`).
> *   `--force`: Ignores safety checks.

---

## 3. Phase 3: Troubleshooting "Device Busy" Errors

If Step 2.6 fails with `Device or resource busy`, the kernel still holds a lock on the disk (common with encrypted LVM volumes). Follow these steps to force-unlock it.

### 3.1 Kill Holding Processes
Identify and kill any process accessing the disk.

```bash
fuser -km /dev/vdd
```
> **Explanation:**
> *   `fuser -km`: Kills (`-k`) all processes accessing the mount/device (`-m`).
> *   `/dev/vdd`: The target disk.

### 3.2 Remove Device-Mapper (Crypto/LVM) Links
Forcefully remove encrypted or LVM mappings.

```bash
dmsetup ls | grep ceph
dmsetup remove --force <MAPPER_NAME>
```
> **Example:**
> ```bash
> dmsetup ls | grep ceph
> dmsetup remove --force ceph--4e0b8bbd--ade6--43ca--b258--fabd3573be8d-osd--block--fc90d120
> ```
> **Explanation:**
> *   `dmsetup ls`: Lists active device-mapper entries.
> *   `dmsetup remove --force`: Forces removal of the mapping, releasing the kernel lock.

### 3.3 Force Wipe Signatures
Use `dd` to overwrite headers if `wipefs` fails.

```bash
dd if=/dev/zero of=/dev/vdd bs=1M count=100 status=progress
sync
partprobe /dev/vdd
```
> **Explanation:**
> *   `dd if=/dev/zero`: Writes zeros to the first 100MB of the disk.
> *   `bs=1M count=100`: Block size 1MB, 100 blocks. Destroys GPT, LVM, and LUKS headers.
> *   `partprobe`: Updates the kernel's partition table view.

### 3.4 Verify Clean State
Confirm the disk is ready for reuse.

```bash
lsblk -f /dev/vdd
ceph orch device ls --refresh
```
> **Explanation:**
> *   `lsblk -f`: Should show **no** FSTYPE or MOUNTPOINT.
> *   `ceph orch device ls --refresh`: Ceph should now list the disk as `Available`.

---

## 4. Phase 4: Manual Encrypted OSD Creation

This section details how to manually create an encrypted OSD using `ceph-volume`. This gives you full control over the process and is useful for debugging or specific deployment scenarios.

### 4.1 Prepare the Encrypted OSD
This command creates the LVM structure, sets up LUKS encryption, and prepares the BlueStore backend. It does **not** start the OSD yet.

```bash
ceph-volume lvm prepare --bluestore --dmcrypt --data /dev/vdd
```
> **Explanation:**
> *   `ceph-volume lvm prepare`: Prepares the device for use as an OSD.
> *   `--bluestore`: Uses the modern BlueStore storage engine.
> *   `--dmcrypt`: Enables Data-at-Rest Encryption using LUKS/dm-crypt.
> *   `--data /dev/vdd`: Specifies the raw disk to be encrypted and used.
>
> **⚠️ Important Output:**
> The command will output two critical values at the end:
> 1.  **OSD ID**: e.g., `12`
> 2.  **FSID (UUID)**: e.g., `e7d13869-0428-46f7-9235-87c9f4dd9726`
> **Save these values.** You need them for activation.

### 4.2 Verify Encryption Layer
Confirm that the disk is now encrypted and the LVM structure is in place.

```bash
lsblk -f /dev/vdd
dmsetup ls --target crypt
```
> **Explanation:**
> *   `lsblk -f`: Should show `crypto_LUKS` as the FSTYPE for `/dev/vdd`.
> *   `dmsetup ls --target crypt`: Lists active encrypted device-mapper targets. You should see an entry related to your OSD.

### 4.3 Inspect LVM Details (Optional but Recommended)
Verify the logical volumes created by `ceph-volume`.

```bash
ceph-volume lvm list
```
> **Explanation:**
> *   Lists all Ceph-managed LVM volumes.
> *   Check that your new OSD ID matches the device `/dev/vdd` and that `encrypted: true` is shown.

---

## 5. Phase 5: Activation, Validation & Maintenance

The OSD is prepared but not running. We must activate it, ensure it starts on boot, and verify cluster health.

### 5.1 Activate the OSD
Start the OSD service using the ID and FSID obtained in Step 4.1.

```bash
ceph-volume lvm activate <OSD_ID> <FSID>
```
> **Example:**
> ```bash
> ceph-volume lvm activate 12 e7d13869-0428-46f7-9235-87c9f4dd9726
> ```
> **Explanation:**
> *   `ceph-volume lvm activate`: Mounts the encrypted volume, generates the systemd unit file, and starts the OSD process.
> *   `<OSD_ID>`: The numeric ID (e.g., `12`).
> *   `<FSID>`: The UUID generated during preparation.

### 5.2 Verify Service Status
Check if the OSD daemon is running correctly.

```bash
systemctl status ceph-osd@<OSD_ID>
```
> **Example:** `systemctl status ceph-osd@12`
> **Explanation:**
> *   `systemctl status`: Checks the state of the systemd service.
> *   **Expected State:** `active (running)`.
> *   **Troubleshooting:** If failed, check logs with `journalctl -u ceph-osd@12 -f`.

### 5.3 Enable Auto-Start on Boot
Ensure the OSD and volume discovery services start after a reboot.

```bash
systemctl enable ceph-osd@<OSD_ID>
systemctl enable ceph-volume@lvm
```
> **Explanation:**
> *   `enable ceph-osd@...`: Ensures the specific OSD starts on boot.
> *   `enable ceph-volume@lvm`: Ensures the background scanner detects and activates LVM volumes on boot.

### 5.4 Final Cluster Health Check
Verify the OSD is integrated into the cluster.

```bash
ceph osd tree
ceph -s
ceph health detail
```
> **Explanation:**
> *   `ceph osd tree`: The new OSD should appear under the host with status `up`.
> *   `ceph -s`: Cluster health should eventually return to `HEALTH_OK`.
> *   `ceph health detail`: Check for any warnings about data rebalancing or slow requests.

### 5.5 Refresh Orchestrator View
Update the Ceph Dashboard/Orchestrator inventory.

```bash
ceph orch device ls --refresh
```
> **Explanation:**
> *   Forces Ceph to rescan the host.
> *   The disk `/dev/vdd` should now show as `In Use` by the new OSD.

---

## 💡 Practical Pro-Tip: The Reboot Test

**Scenario:** You have configured a new encrypted OSD. How do you know it will survive a power outage?

**Action:** Reboot the node once.

```bash
reboot
```

**Verification:** After the node comes back online, check the OSD status.

```bash
ceph osd tree
```

**Interpretation:**
*   **If OSD is `up`:** Your encryption key management (via `ceph-volume` and systemd) is correctly configured. The node successfully unlocked the LUKS container and started the OSD.
*   **If OSD is `down`:** The node failed to retrieve the dmcrypt key or the systemd service failed to start. Check logs using `journalctl -u ceph-osd@<ID> -f`.

---

**End of Guide.**
*This document serves as a blueprint for Linux Storage Engineers working with Ceph-LVM-Dmcrypt stacks.*
