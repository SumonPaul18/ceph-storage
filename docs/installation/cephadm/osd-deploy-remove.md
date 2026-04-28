# Ceph OSD Management: A Practical Guide for Production Environments

**Version:** 1.0  
**Target Release:** Ceph Reef (v18.x)  
**Author:** DevOps & Cloud Engineering Team  
**Context:** Meghna Cloud R&D Lab – Infrastructure Operations  

---

## Table of Contents

1. [Introduction and Environment Preparation](#1-introduction-and-environment-preparation)
2. [Deploying Ceph OSDs: Strategies and Execution](#2-deploying-ceph-osds-strategies-and-execution)
3. [Advanced OSD Deployment with YAML Specifications](#3-advanced-osd-deployment-with-yaml-specifications)
4. [Removing, Erasing, and Cleaning Up OSDs](#4-removing-erasing-and-cleaning-up-osds)
5. [Troubleshooting Common OSD Errors](#5-troubleshooting-common-osd-errors)

---

## 1. Introduction and Environment Preparation

### Overview
In a modern Cloud Service Provider environment like Meghna Cloud, storage is the backbone of our Infrastructure as a Service (IaaS). Ceph provides scalable, software-defined storage. The **OSD (Object Storage Daemon)** is the core component that stores data on physical disks. 

This guide focuses on the practical, hands-on management of OSDs using `cephadm`, the recommended deployment tool for Ceph Reef. We will cover how to safely add storage capacity and how to securely remove and wipe disks for reuse or decommissioning.

### Prerequisites
Before executing any commands, ensure your environment meets these criteria:
*   **Access:** Root or sudo access to the Ceph admin node and target storage nodes.
*   **Cluster Health:** The Ceph cluster must be in `HEALTH_OK` state before major changes.
*   **Network:** All nodes must have connectivity to the Ceph public and cluster networks.
*   **Disks:** Target disks must be physically attached and visible to the Linux OS (`lsblk`).

### Step 1: Verify Cluster Health
Always start by checking the current state of the cluster.

```bash
ceph -s
```
> **Explanation:** This command displays the overall health status of the Ceph cluster.
>
> **Key Output to Check:**
> *   `health`: Should be `HEALTH_OK`. If it shows `HEALTH_WARN` or `HEALTH_ERR`, investigate existing issues before adding/removing OSDs.
> *   `osd`: Shows the number of OSDs currently up and in.

### Step 2: Identify Available Disks
Ceph needs to know which disks are empty and ready to be used.

```bash
ceph orch device ls
```
> **Explanation:** Lists all storage devices detected by `cephadm` across all hosts.
>
> **Key Columns:**
> *   `HOST`: The node where the disk is located.
> *   `PATH`: The device path (e.g., `/dev/sdb`).
> *   `AVAILABLE`: `True` means the disk is empty and ready. `False` means it is in use or has existing data/partitions.
> *   `REJECT REASONS`: Explains why a disk is not available (e.g., "has filesystem", "mounted").

### Step 3: Refresh Device Inventory (If Needed)
If you just added a new physical disk and it doesn't appear, force a refresh.

```bash
ceph orch device ls --refresh
```
> **Explanation:** Forces `cephadm` to rescan all hosts for new or changed storage devices.
>
> **When to Use:** Immediately after physically attaching new disks or replacing failed ones.

---

## 2. Deploying Ceph OSDs: Strategies and Execution

There are three primary ways to deploy OSDs in Ceph Reef:
1.  **Automatic Deployment:** Uses all available empty disks on all hosts.
2.  **Specific Device Deployment:** Targets specific disks on specific hosts.
3.  **YAML Specification:** Defines complex rules for disk selection (e.g., separate SSDs for DB/WAL).

### Strategy A: Automatic Deployment (All Available Devices)

This is the fastest way to expand storage if you want to use **all** empty disks on **all** nodes.

#### Dry Run (Simulation)
Before making changes, simulate the deployment to see what *would* happen.

```bash
ceph orch apply osd --all-available-devices --dry-run
```
> **Explanation:** Simulates the OSD creation process without actually creating any OSDs.
>
> **Why Use It:**
> *   To verify which disks Ceph will select.
> *   To ensure no unexpected disks are wiped.
> *   **Safety First:** Always run this before the actual command.
>
> **Output:** A list of hosts and devices that *would* be converted to OSDs.

#### Execute Automatic Deployment
If the dry run looks correct, execute the deployment.

```bash
ceph orch apply osd --all-available-devices
```
> **Explanation:** Creates a service specification that tells `cephadm` to automatically deploy OSDs on all available devices on all hosts.
>
> **Behavior:**
> *   Ceph will continuously monitor for new empty disks. If you add a new disk later, it will automatically be converted to an OSD.
> *   This is a **managed** service. Ceph controls the lifecycle.

#### Understanding `--unmanaged` Flags
Sometimes, you want to define the specification but prevent Ceph from automatically acting on it immediately, or you want to stop automatic management.

```bash
ceph orch apply osd --all-available-devices --unmanaged
```
> **Explanation:** Applies the specification but sets the service to "unmanaged" mode.
>
> **Why Use It:**
> *   **Freezing State:** You want to keep the current configuration but prevent Ceph from automatically adding *new* disks that might appear later.
> *   **Manual Control:** You plan to manage OSD additions manually via specific commands or YAML files, but want to keep the "all-available" logic as a baseline template.
>
> **Note:** In unmanaged mode, `cephadm` will not automatically create OSDs on new disks. You must manually trigger actions or update the spec.

```bash
ceph orch apply osd --all-available-devices --unmanaged=true
```
> **Explanation:** Explicitly sets the unmanaged flag to true. Functionally identical to the previous command.
>
> **When to Use:** When scripting or automating configurations where explicit boolean values are preferred for clarity.

#### Verification of Automatic Deployment
Check if the OSDs are running.

```bash
ceph orch ps | grep osd
```
> **Explanation:** Lists all running OSD daemons managed by the orchestrator.
>
> **Expected Output:** New OSD entries should appear with status `running`.

```bash
ceph osd tree
```
> **Explanation:** Shows the hierarchical structure of OSDs. Ensure new OSDs are listed under their respective hosts and are marked `up` and `in`.

---

### Strategy B: Specific Device Deployment

Use this when you want to add OSDs on **specific disks** on **specific hosts**. This is common when you have mixed disk types or want to control placement precisely.


#### Wipe the disk through the orchestrator:
You need to "zap" (wipe) the disk to remove all signatures before Ceph will accept it.
```
ceph orch device zap ceph4 /dev/sdb --force
```
> Replace `hostname` and `/dev/sdX` with your actual values.

#### Add OSD on a Specific Device
Replace `hostname` and `/dev/sdX` with your actual values.

```bash
ceph orch daemon add osd ceph-node-01:/dev/sdc
```
> **Explanation:** Instructs `cephadm` to create a single OSD using `/dev/sdc` on `ceph-node-01`.
>
> **Syntax Breakdown:**
> *   `orch daemon add osd`: Subcommand to add a specific OSD daemon.
> *   `ceph-node-01`: The hostname of the target machine.
> *   `:/dev/sdc`: The device path. The colon separates the host from the device.

#### Command 2: Add OSD with Encryption (DM-Crypt)
If you need data-at-rest encryption, specify the encrypted flag.

```bash
ceph orch daemon add osd ceph-node-01:/dev/sdc --encrypted
```
> **Explanation:** Creates an OSD with LUKS encryption enabled on the specified device.
>
> **Why Use It:**
> *   **Compliance:** Required for sensitive data (e.g., financial, healthcare) to meet regulatory standards.
> *   **Security:** Protects data if the physical disk is stolen or improperly disposed of.
>
> **Note:** Encryption adds CPU overhead for encryption/decryption operations. Ensure your nodes have sufficient CPU resources.

#### Verification of Specific Deployment
Check the status of the newly added OSD.

```bash
ceph orch ps --daemon_type osd --host ceph-node-01
```
> **Explanation:** Lists OSD daemons specifically on `ceph-node-01`.

```bash
ceph osd metadata <osd-id>
```
> **Explanation:** Displays detailed metadata for a specific OSD. Replace `<osd-id>` with the actual ID (e.g., `0`).
>
> **Key Check:** Look for `osd_encryption` field to confirm if encryption is enabled.

---

## 3. Advanced OSD Deployment with YAML Specifications

For complex environments (e.g., separating HDDs for data and SSDs for WAL/DB), use YAML specification files. This approach is declarative and version-controllable.

### Scenario 1: Basic YAML for All Available Devices
Create a file named `osd-all.yaml`.

```yaml
service_type: osd
service_id: default_drive_group
placement:
  hosts:
    - ceph-node-01
    - ceph-node-02
data_devices:
  all: true
```

#### Apply the YAML Specification
```bash
ceph orch apply -i osd-all.yaml
```
> **Explanation:** Applies the OSD configuration defined in `osd-all.yaml`.
>
> **Breakdown:**
> *   `service_type: osd`: Defines the service type.
> *   `placement`: Specifies which hosts are included.
> *   `data_devices: all: true`: Tells Ceph to use all available devices on the specified hosts.

### Scenario 2: YAML with Encryption
Create a file named `osd-encrypted.yaml`.

```yaml
service_type: osd
service_id: encrypted_drive_group
placement:
  hosts:
    - ceph-node-01
data_devices:
  all: true
encrypted: true
```

#### Apply the Encrypted YAML Specification
```bash
ceph orch apply -i osd-encrypted.yaml
```
> **Explanation:** Deploys OSDs with encryption enabled on all available devices on `ceph-node-01`.
>
> **Key Field:**
> *   `encrypted: true`: Enables LUKS encryption for all OSDs created by this specification.

### Scenario 3: Mixed Media (HDD Data + SSD WAL/DB)
This is a common production setup for performance optimization. Create a file named `osd-mixed.yaml`.

```yaml
service_type: osd
service_id: mixed_media_group
placement:
  hosts:
    - ceph-node-01
data_devices:
  rotational: true
db_devices:
  rotational: false
```

#### Apply the Mixed Media YAML Specification
```bash
ceph orch apply -i osd-mixed.yaml
```
> **Explanation:** Configures Ceph to use rotational disks (HDDs) for data storage and non-rotational disks (SSDs/NVMe) for Write-Ahead Logs (WAL) and Database (DB) partitions.
>
> **Why Use It:**
> *   **Performance:** SSDs handle small, random I/O operations (WAL/DB) much faster than HDDs.
> *   **Cost Efficiency:** Uses cheaper HDDs for bulk storage while leveraging SSD speed for metadata and journaling.
>
> **Key Fields:**
> *   `data_devices: rotational: true`: Selects HDDs for data.
> *   `db_devices: rotational: false`: Selects SSDs/NVMe for DB/WAL.

### Verification of YAML Deployment
Check if the service specification was applied correctly.

```bash
ceph orch ls | grep osd
```
> **Explanation:** Lists all orchestrated services. You should see your custom service IDs (e.g., `mixed_media_group`).

```bash
ceph orch ps | grep osd
```
> **Explanation:** Verify that OSDs are running according to the new specification.

---

## 4. Removing, Erasing, and Cleaning Up OSDs

Removing an OSD is a critical operation. It involves stopping data flow, removing the daemon, and optionally wiping the disk for reuse.

### Step 1: Mark the OSD as Out
This stops new data from being written to the OSD and triggers data rebalancing.

```bash
ceph osd out <osd-id>
```
> **Explanation:** Marks the OSD as `out` of the cluster.
>
> **Example:** `ceph osd out 5`
>
> **What Happens:**
> *   Ceph starts replicating data from this OSD to other OSDs.
> *   The OSD remains `up` (running) but is `out` (not accepting new data).
>
> **Wait Time:** Depending on the amount of data, this can take hours or days. Monitor progress with `ceph -w`.

### Step 2: Wait for Rebalancing to Complete
Monitor the cluster until it returns to `HEALTH_OK`.

```bash
ceph -w
```
> **Explanation:** Watches cluster events in real-time.
>
> **What to Look For:**
> *   `recovery` messages decreasing.
> *   Final message: `cluster is now healthy`.

### Step 3: Stop and Remove the OSD Daemon
Once rebalancing is complete, remove the daemon from the host.

```bash
ceph orch daemon rm osd.<osd-id> --force
```
> **Explanation:** Removes the OSD daemon from the host.
>
> **Example:** `ceph orch daemon rm osd.5 --force`
>
> **Flags:**
> *   `--force`: Forces removal even if the OSD is not perfectly clean. Use with caution.
>
> **Note:** This stops the OSD process and removes it from `cephadm` management. The disk still contains data.

### Step 4: Wipe the Disk (Erasing/Cleanup)
If you plan to reuse the disk or return it to storage, you must wipe it.

#### Option A: Wipe Using Ceph Orchestrator (Recommended)
```bash
ceph orch device zap <hostname> <device-path> --destroy
```
> **Explanation:** Completely wipes the partition table and data from the disk.
>
> **Example:** `ceph orch device zap ceph-node-01 /dev/sdc --destroy`
>
> **Why Use It:**
> *   **Safety:** Ensures no residual Ceph metadata remains.
> *   **Reuse:** Prepares the disk for new OSD deployment or other uses.
>
> **Warning:** This action is **irreversible**. All data on the disk will be lost.

#### Option B: Manual Wipe (If Ceph Orchestrator Fails)
If `ceph orch device zap` fails, you can manually wipe the disk from the host.

```bash
ssh ceph-node-01
sudo wipefs -a /dev/sdc
sudo dd if=/dev/zero of=/dev/sdc bs=1M count=100
```
> **Explanation:**
> 1.  `wipefs -a`: Removes all filesystem signatures.
> 2.  `dd if=/dev/zero...`: Overwrites the first 100MB with zeros to clear partition tables and boot sectors.
>
> **When to Use:** Only if the orchestrator cannot access the disk or fails to zap it.

### Step 5: Verify Removal
Confirm the OSD is gone from the cluster.

```bash
ceph osd tree
```
> **Explanation:** The removed OSD should no longer appear in the tree.

```bash
ceph orch device ls
```
> **Explanation:** The disk should now show as `AVAILABLE: True` (if wiped correctly) or be absent if physically removed.

---

## 5. Troubleshooting Common OSD Errors

### Issue 1: Disk Not Showing as "Available"
**Symptom:** `ceph orch device ls` shows `AVAILABLE: False` for a new disk.

**Diagnosis:**
```bash
ceph orch device ls | grep <hostname>
```
Check the `REJECT REASONS` column. Common reasons:
*   `has filesystem`: The disk has an existing filesystem (e.g., ext4, xfs).
*   `mounted`: The disk is mounted to a directory.
*   `has partitions`: The disk has existing partitions.

**Solution:**
1.  Unmount the disk if mounted:
    ```bash
    umount /dev/sdc
    ```
2.  Wipe the disk signatures:
    ```bash
    wipefs -a /dev/sdc
    ```
3.  Refresh Ceph inventory:
    ```bash
    ceph orch device ls --refresh
    ```

### Issue 2: OSD Fails to Start After Deployment
**Symptom:** `ceph orch ps` shows OSD status as `failed` or `stopped`.

**Diagnosis:**
Check the logs for the specific OSD.
```bash
ceph logs osd.<osd-id>
```
> **Explanation:** Displays the recent logs for the specified OSD daemon.
>
> **Common Errors:**
> *   `Permission denied`: Check SELinux or AppArmor settings.
> *   `Device busy`: Another process is using the disk.
> *   `Encryption error`: LUKS key issue or missing kernel modules.

**Solution:**
*   If `Device busy`, identify the process:
    ```bash
    lsof /dev/sdc
    ```
    Kill the process or unmount the filesystem.
*   If `Encryption error`, ensure `cryptsetup` is installed and the kernel supports LUKS.

### Issue 3: Rebalancing Takes Too Long
**Symptom:** After marking an OSD `out`, the cluster remains in `HEALTH_WARN` for a long time.

**Diagnosis:**
Check the recovery status.
```bash
ceph -s
```
Look for `recovery` or `backfill` progress.

**Solution:**
*   **Be Patient:** Rebalancing large amounts of data takes time.
*   **Increase Recovery Threads (Temporary):**
    ```bash
    ceph config set osd osd_max_backfills 5
    ceph config set osd osd_recovery_max_active 5
    ```
    > **Warning:** Increasing these values increases network and disk I/O load. Revert to defaults after rebalancing completes.
    ```bash
    ceph config rm osd osd_max_backfills
    ceph config rm osd osd_recovery_max_active
    ```

### Issue 4: Cannot Zap Disk (Device Busy)
**Symptom:** `ceph orch device zap` fails with "Device or resource busy".

**Diagnosis:**
The disk might still be mounted or used by a lingering process.

**Solution:**
1.  Check for mounts:
    ```bash
    mount | grep /dev/sdc
    ```
2.  Unmount if necessary:
    ```bash
    umount /dev/sdc
    ```
3.  Check for LVM volumes:
    ```bash
    lvdisplay | grep /dev/sdc
    ```
4.  Remove LVM volumes if present:
    ```bash
    lvremove /dev/vg_name/lv_name
    vgremove vg_name
    pvremove /dev/sdc
    ```
5.  Retry the zap command:
    ```bash
    ceph orch device zap ceph-node-01 /dev/sdc --destroy
    ```

---

## Conclusion

Managing Ceph OSDs is a critical task for maintaining a healthy and scalable storage infrastructure. By following this practical guide, you can:
*   Safely add storage capacity using automatic or specific deployment methods.
*   Implement encryption for data security compliance.
*   Use YAML specifications for complex, reproducible configurations.
*   Securely remove and wipe disks for reuse or decommissioning.
*   Troubleshoot common issues effectively.

Always remember to:
1.  **Verify** cluster health before and after changes.
2.  **Backup** critical data before performing destructive operations like wiping.
3.  **Monitor** rebalancing progress to ensure cluster stability.

For further details, refer to the official Ceph documentation:
*   [Cephadm OSD Services](https://docs.ceph.com/en/reef/cephadm/services/osd/)
*   [Add or Remove OSDs](https://docs.ceph.com/en/reef/rados/operations/add-or-rm-osds/)

---
