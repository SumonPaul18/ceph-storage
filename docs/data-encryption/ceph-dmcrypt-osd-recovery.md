YES.

Here is the professional, practical guide based on your command history. It follows a narrative of troubleshooting, cleaning up, and manually re-adding an encrypted OSD in a Ceph cluster managed by `cephadm`.

***

# File Name: `ceph-dmcrypt-osd-recovery.md`

# Table of Contents
1. [Phase 1: Initial Diagnosis & Verification](#phase-1-initial-diagnosis--verification)
2. [Phase 2: Safe Removal of Existing OSD](#phase-2-safe-removal-of-existing-osd)
3. [Phase 3: Disk Cleaning & Preparation](#phase-3-disk-cleaning--preparation)
4. [Phase 4: Manual Encrypted OSD Creation](#phase-4-manual-encrypted-osd-creation)
5. [Phase 5: Activation & Final Validation](#phase-5-activation--final-validation)

---

# Phase 1: Initial Diagnosis & Verification

Before making changes, we verify the current state of the devices, encryption status, and cluster health.

### 1.1 Check Available Devices
Identify all storage devices recognized by the Ceph orchestrator.

```bash
ceph orch device ls
```
*   **Command:** `ceph orch device ls`
*   **Explanation:** Lists all physical disks detected by `cephadm`. Look for the target disk (e.g., `/dev/vdd`) and its status (Available/In Use).

### 1.2 Inspect Device Mapper Tree
Check if any logical volumes or encryption layers are currently active on the system.

```bash
dmsetup ls --tree
```
*   **Command:** `dmsetup ls --tree`
*   **Explanation:** Displays the device-mapper tree. Useful to see if LVM or dm-crypt (LUKS) maps are already established for the target disk.

### 1.3 Detailed Block Device Listing
View filesystem types, mount points, and sizes for all block devices.

```bash
lsblk -o NAME,TYPE,FSTYPE,MOUNTPOINT,SIZE
```
*   **Command:** `lsblk -o NAME,TYPE,FSTYPE,MOUNTPOINT,SIZE`
*   **Parameters:**
    *   `-o`: Specifies output columns.
    *   `FSTYPE`: Shows if the partition has a filesystem (e.g., xfs, bluestore).
    *   `MOUNTPOINT`: Shows where it is mounted.

### 1.4 Verify LVM Encryption Status
Check if existing Logical Volumes are configured with encryption.

```bash
ceph-volume lvm list --format json | python3 -m json.tool | grep -A2 "encryption"
```
*   **Command:** `ceph-volume lvm list ... | grep -A2 "encryption"`
*   **Explanation:** Lists LVM volumes in JSON format and filters for the "encryption" key. Confirms if `dmcrypt` is enabled on existing OSDs.

### 1.5 Identify OSD Devices
Map OSD IDs to their underlying physical devices.

```bash
ceph-volume lvm list | grep "devices"
```
*   **Command:** `ceph-volume lvm list | grep "devices"`
*   **Explanation:** Quick text-based check to see which physical path (e.g., `/dev/vdd`) belongs to which OSD ID.

### 1.6 Inspect LUKS Header
Directly check if the physical disk has a LUKS encryption header.

```bash
cryptsetup luksDump /dev/vdd
```
*   **Command:** `cryptsetup luksDump /dev/vdd`
*   **Explanation:** Dumps the LUKS header information. If it returns an error like "not a valid LUKS device," the disk is not encrypted. If it shows version/keyslots, it is encrypted.

### 1.7 Check Active Crypt Targets
List only device-mapper targets related to encryption.

```bash
dmsetup ls --target crypt
```
*   **Command:** `dmsetup ls --target crypt`
*   **Explanation:** Filters the device mapper list to show only active cryptographic mappings.

### 1.8 View Crypt Table Details
Show the underlying parameters of active crypt mappings.

```bash
dmsetup table --target crypt
```
*   **Command:** `dmsetup table --target crypt`
*   **Explanation:** Displays the cipher, key size, and sector offset for active encrypted devices.

### 1.9 Search Ceph Config for Keys
Check if Ceph stores encryption keys internally.

```bash
ceph config-key ls | grep dm-crypt
ceph config-key ls | grep luks
```
*   **Command:** `ceph config-key ls | grep ...`
*   **Explanation:** `ceph config-key` stores internal metadata. Searching for `dm-crypt` or `luks` helps verify if Ceph is managing the encryption keys for specific OSDs.

### 1.10 Cluster Health & Topology
Verify the overall cluster status and OSD tree structure.

```bash
ceph osd tree
ceph -s
ceph health detail
```
*   **Commands:**
    *   `ceph osd tree`: Shows the hierarchy of hosts and OSDs.
    *   `ceph -s`: Brief cluster status (HEALTH_OK/WARN/ERR).
    *   `ceph health detail`: Detailed explanation of any health warnings.

---

# Phase 2: Safe Removal of Existing OSD

If the disk was previously part of the cluster or is in a bad state, we must remove it cleanly before reusing it.

### 2.1 Remove OSD Daemon
Forcefully remove the OSD daemon from the cluster orchestration.

```bash
ceph orch daemon rm osd.11 --force
```
*   **Command:** `ceph orch daemon rm osd.11 --force`
*   **Parameters:**
    *   `osd.11`: The specific OSD ID to remove.
    *   `--force`: Bypasses safety checks (use with caution).
*   **Explanation:** Tells `cephadm` to stop and remove the container/process for OSD 11.

### 2.2 Verify Removal Status
Confirm the OSD is gone from the tree and cluster status updates.

```bash
ceph -s
ceph osd tree
```
*   **Explanation:** Ensures the cluster acknowledges the removal. The OSD should no longer appear in the tree or should be marked as `down/out`.

---

# Phase 3: Disk Cleaning & Preparation

Before creating a new encrypted OSD, the disk must be completely wiped of old partitions, LVM signatures, and encryption headers.

### 3.1 Zap the Device
Use Ceph's built-in tool to wipe the disk completely.

```bash
ceph orch device zap ceph1 /dev/vdd --force
```
*   **Command:** `ceph orch device zap ceph1 /dev/vdd --force`
*   **Parameters:**
    *   `ceph1`: The hostname where the disk resides.
    *   `/dev/vdd`: The target disk.
    *   `--force`: Confirms destructive action.
*   **Explanation:** Removes partition tables, LVM metadata, and superblocks. This is the preferred method in `cephadm` environments.

### 3.2 Verify Clean State
Check if the disk is now raw and unmounted.

```bash
lsblk
```
*   **Explanation:** `/dev/vdd` should show no children (partitions) and no mount points.

### 3.3 Confirm Encryption Removal
Ensure LUKS headers are gone.

```bash
ceph config-key ls | grep luks
ceph config-key ls | grep dm-crypt
dmsetup table --target crypt
dmsetup ls --target crypt
cryptsetup luksDump /dev/vdd
lsblk -f | grep crypto
```
*   **Explanation:** These commands should return **empty** results or errors indicating no encryption exists. This confirms the disk is clean.

### 3.4 Alternative Wipe (If Orchestrator Zap Fails)
If `ceph orch device zap` fails, use `ceph-volume` directly.

```bash
ceph-volume lvm zap /dev/vdd --destroy
```
*   **Command:** `ceph-volume lvm zap /dev/vdd --destroy`
*   **Parameters:**
    *   `--destroy`: Destroys any existing LVM volume group or logical volume associated with the disk.
*   **Explanation:** A lower-level wipe command that ensures LVM structures are removed.

### 3.5 Final Pre-Check
Verify the disk is completely raw.

```bash
lsblk
lsblk -f | grep crypto
```
*   **Explanation:** No `crypto_LUKS` filesystem type should be visible.

---

# Phase 4: Manual Encrypted OSD Creation

Since automatic deployment might not be configured for encryption, we manually prepare the OSD with `dmcrypt` enabled.

### 4.1 Ensure Ceph Packages are Installed
Verify necessary tools are available on the host.

```bash
apt-get update
apt-get install ceph-osd
```
*   **Explanation:** Ensures `ceph-volume` and `ceph-osd` binaries are present and up-to-date on the host node (`ceph1`).

### 4.2 Prepare the Encrypted OSD
Create the LVM volumes and LUKS encryption layer.

```bash
ceph-volume lvm prepare --bluestore --dmcrypt --data /dev/vdd
```
*   **Command:** `ceph-volume lvm prepare --bluestore --dmcrypt --data /dev/vdd`
*   **Parameters:**
    *   `prepare`: Creates the necessary LVM volumes and formats them but does not start the OSD.
    *   `--bluestore`: Uses the BlueStore storage backend (standard for modern Ceph).
    *   `--dmcrypt`: Enables data-at-rest encryption using LUKS.
    *   `--data /dev/vdd`: Specifies the raw disk to use.
*   **Explanation:** This command creates a LUKS container on `/dev/vdd`, creates LVM PV/VG/LV inside it, and writes the Ceph configuration. It outputs an **OSD ID** and **FSID** (UUID) which are needed for activation.

### 4.3 Verify Preparation
Check if the encryption layer is created.

```bash
lsblk
lsblk -f | grep crypto
cryptsetup luksDump /dev/vdd
```
*   **Explanation:** You should now see a `crypto_LUKS` filesystem type on `/dev/vdd` and child devices mapped via device-mapper.

### 4.4 Check Ceph Key Storage
Verify Ceph stored the encryption key.

```bash
ceph config-key ls | grep dm-crypt
```
*   **Explanation:** Should show a new key entry corresponding to the new OSD's FSID.

---

# Phase 5: Activation & Final Validation

The OSD is prepared but not yet running. We must activate it and verify its integration into the cluster.

### 5.1 Activate the OSD
Start the OSD process using the ID and FSID obtained from the `prepare` step.

```bash
ceph-volume lvm activate 11 e7d13869-0428-46f7-9235-87c9f4dd9726
```
*   **Command:** `ceph-volume lvm activate <OSD_ID> <FSID>`
*   **Parameters:**
    *   `11`: The OSD ID generated during preparation.
    *   `e7d13869...`: The FSID (UUID) generated during preparation.
*   **Explanation:** Mounts the encrypted volume, generates the systemd service file, and starts the OSD daemon.

### 5.2 Verify Service Status
Check if the OSD process is running correctly.

```bash
systemctl status ceph-osd@11
```
*   **Command:** `systemctl status ceph-osd@11`
*   **Explanation:** Should show `active (running)`. If it fails, check logs with `journalctl -u ceph-osd@11`.

### 5.3 Cluster Integration Check
Verify the OSD is up and in the cluster tree.

```bash
ceph orch device ls
ceph osd tree
ceph -s
```
*   **Explanation:**
    *   `ceph orch device ls`: The disk `/dev/vdd` should now show as "Used" or associated with an OSD.
    *   `ceph osd tree`: OSD 11 should appear under `ceph1` with status `up`.
    *   `ceph -s`: Cluster health should return to `HEALTH_OK` once data rebalancing completes.

### 5.4 Refresh Orchestrator Inventory
Force `cephadm` to refresh its view of the devices.

```bash
ceph orch device ls --refresh
```
*   **Explanation:** Ensures the dashboard and orchestrator have the latest state of the hardware.

### 5.5 Final Validation
Confirm encryption is active and data path is secure.

```bash
lsblk -f | grep crypto
ceph-volume lvm list
```
*   **Explanation:**
    *   `lsblk`: Confirms the active LUKS mapping.
    *   `ceph-volume lvm list`: Shows the OSD details, confirming `encrypted: true` (or similar flag depending on version output).