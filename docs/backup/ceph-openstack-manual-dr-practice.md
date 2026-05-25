# OpenStack-Ceph Disaster Recovery: Manual RBD Backup & Restore Guide

## 1. Introduction & Objective

### Overview
This guide provides a comprehensive, step-by-step practical methodology for performing **Disaster Recovery (DR)** on OpenStack instances backed by a Ceph cluster. Specifically, it focuses on the **Manual RBD (Raw Block Device) Export/Import** technique. While modern OpenStack environments utilize high-level APIs like `cinder-backup`, understanding the underlying storage mechanics via Ceph RBD is critical for deep troubleshooting, emergency recovery when control planes fail, and educational purposes in home labs or private clouds.

### Scenario
You are managing a Single-Node or Multi-Node OpenStack deployment using **Kolla-Ansible**, integrated with a **Ceph Cluster**.
*   **Current State:** An OpenStack Instance is running, and its root disk resides on a Ceph RBD image within the `volumes` pool.
*   **Goal:**
    1.  Create a consistent backup of the instance's data directly from the Ceph storage layer.
    2.  Simulate a disaster by deleting the underlying Ceph RBD image.
    3.  Verify that the OpenStack instance fails (becomes inactive/corrupted).
    4.  Restore the data from the backup to the Ceph cluster.
    5.  Recover the OpenStack instance to an `ACTIVE` state and verify data integrity.

### Why This Method?
*   **Deep Dive:** It exposes the direct relationship between OpenStack Cinder volumes and Ceph RBD images.
*   **Emergency Use:** Useful if the Cinder API is down but Ceph monitors are accessible.
*   **Verification:** Proves that data persistence relies entirely on the storage backend.

### Limitations
*   **Downtime:** This method requires manual intervention and potential downtime during restore.
*   **Complexity:** Higher risk of human error compared to automated Cinder backups.
*   **Not Recommended for Production Daily Use:** For production, always prefer `openstack volume backup` or Ceph RBD Mirroring. This guide is for **understanding, testing, and emergency recovery**.

---

## 2. Table of Contents

1.  [Prerequisites & Environment Setup](#3-prerequisites--environment-setup)
2.  [Step 1: Identification of Instance and Ceph Resources](#4-step-1-identification-of-instance-and-ceph-resources)
3.  [Step 2: Creating a Consistent Backup (Snapshot & Export)](#5-step-2-creating-a-consistent-backup-snapshot--export)
4.  [Step 3: Simulating Disaster (Data Deletion)](#6-step-3-simulating-disaster-data-deletion)
5.  [Step 4: Verifying Instance Failure](#7-step-4-verifying-instance-failure)
6.  [Step 5: Restoring Data to Ceph (Import)](#8-step-5-restoring-data-to-ceph-import)
7.  [Step 6: OpenStack Instance Recovery & Re-attachment](#9-step-6-openstack-instance-recovery--re-attachment)
8.  [Step 7: Final Verification & Data Integrity Check](#10-step-7-final-verification--data-integrity-check)
9.  [Production Best Practices & Alternative DR Strategies](#11-production-best-practices--alternative-dr-strategies)
10. [Troubleshooting & Common Errors](#12-troubleshooting--common-errors)

---

## 3. Prerequisites & Environment Setup

Before executing any commands, ensure your environment meets the following requirements. This guide assumes you are operating from the **Deploy Node** or a **Controller Node** where Kolla-Ansible tools and Ceph clients are installed.

### Requirements
1.  **Access:** Root or sudo access to the node running Kolla containers.
2.  **Tools Installed:**
    *   `openstack` CLI client.
    *   `ceph` and `rbd` CLI clients.
    *   `qemu-img` (for verifying disk formats).
3.  **Ceph Configuration:** Access to `ceph.conf` and `ceph.client.admin.keyring`. In Kolla deployments, these are typically inside containers or mapped to `/etc/kolla/config/ceph/`.
4.  **Test Instance:** A running OpenStack instance with a known Volume ID (preferably a Boot-from-Volume instance for easier testing).

### Checking Tool Versions
Ensure your tools are up-to-date and compatible with your OpenStack/Ceph version.

**Check OpenStack Client Version**
Ensures compatibility with your OpenStack Cloud
```bash
openstack --version
```

**Check Ceph Version**
Verifies the Ceph cluster version you are interacting with
```bash
ceph --version
```

**Check RBD Tool Version**
Ensures rbd command supports required flags like --progress
```bash
rbd --version
```

### Sourcing OpenStack Credentials
You must source the `admin-openrc.sh` or your user-specific openrc file to interact with the OpenStack API.

**Source the admin credentials**
Replace with the actual path to your openrc file
```bash
source /etc/kolla/admin-openrc.sh
```

---

## 4. Step 1: Identification of Instance and Ceph Resources

The first critical step is mapping the OpenStack logical resources (Instance/Volume) to the physical Ceph storage objects (Pool/Image).

### Concept
OpenStack Cinder creates a volume, which translates to an RBD image in a specific Ceph pool (usually named `volumes` or `cinder-volumes`). We need to find this exact image name to back it up.

### Practical Guide

#### 1. Identify the Target Instance
Get the details of the instance you want to back up.

**List all servers to find your target instance name or ID**
Look for the 'Name' and 'ID' columns
```bash
openstack server list
```

#### 2. Get Attached Volume ID
Instances may have multiple volumes. We need the ID of the root volume or the specific data volume.

**Show detailed info of the instance, specifically attached volumes**
Replace `<INSTANCE_NAME>` with your actual instance name
```bash
openstack server show <INSTANCE_NAME> -c volumes_attached
```

#### 3. Verify Volume Details in Cinder
Confirm the volume status and size before proceeding.

**Show volume details using the Volume ID obtained above**
Replace `<VOLUME_ID>` with the actual UUID
```bash
openstack volume show <VOLUME_ID> -c size -c status -c name
```

#### 4. Map to Ceph RBD Image
In Kolla-Ansible, the RBD image name usually matches the Cinder Volume ID with a prefix `volume-`. The pool name is typically `volumes`.

**Define variables for easier scripting (Optional but recommended)**
Set environment variables for Pool, Image, and Volume ID
```bash
export VOLUME_ID="<VOLUME_ID>"
export CEPH_POOL="volumes"
export RBD_IMAGE="volume-${VOLUME_ID}"
```

**Check if the RBD image exists in the Ceph pool**
This command lists all images in the pool and filters for your volume
```bash
rbd ls ${CEPH_POOL} | grep ${VOLUME_ID}
```

#### 5. Inspect RBD Image Metadata
Verify the image format and size at the Ceph level.

**Get detailed info about the RBD image**
Displays metadata like size, format (v1/v2), and features
```bash
rbd info ${CEPH_POOL}/${RBD_IMAGE}
```

---

## 5. Step 2: Creating a Consistent Backup (Snapshot & Export)

Directly exporting a live RBD image can lead to data corruption if the VM is writing data during the export. The best practice at the block level is to take a **Snapshot** first, then export the snapshot.

### Concept
*   **Snapshot:** A point-in-time read-only copy of the RBD image. It is instant and consistent.
*   **Export:** Copying the snapshot data to a local file system (e.g., `/root/backup.img`).

### Practical Guide

#### 1. Create a Backup Directory
Organize your backups to avoid clutter.

**Create a directory for storing backups**
Creates the folder and navigates into it
```bash
mkdir -p /root/ceph-dr-backups
cd /root/ceph-dr-backups
```

#### 2. Create an RBD Snapshot
Take a snapshot of the live volume.

**Define a unique snapshot name using timestamp**
Generates a dynamic name based on current date/time
```bash
export SNAP_NAME="dr-snap-$(date +%Y%m%d-%H%M%S)"
```

**Create the snapshot**
Syntax: `rbd snap create <pool>/<image>@<snap-name>`
```bash
rbd snap create ${CEPH_POOL}/${RBD_IMAGE}@${SNAP_NAME}
```

#### 3. Verify Snapshot Creation
Ensure the snapshot exists before exporting.

**List all snapshots for the specific RBD image**
Shows all snapshots associated with the image
```bash
rbd snap list ${CEPH_POOL}/${RBD_IMAGE}
```

#### 4. Export Snapshot to Local File
Copy the snapshot data to a local `.img` file. This file is your portable backup.

**Define the backup file path**
Sets the destination path for the exported image
```bash
export BACKUP_FILE="/root/ceph-dr-backups/${RBD_IMAGE}.img"
```

**Export the snapshot to the local file**
Using `--progress` flag to see transfer status (if supported by your Ceph version)
```bash
rbd export ${CEPH_POOL}/${RBD_IMAGE}@${SNAP_NAME} ${BACKUP_FILE} --progress
```

#### 5. Verify Backup File Integrity
Check if the file was created correctly and inspect its format.

**Check file size and existence**
Human-readable file size. Ensure it matches the expected volume size
```bash
ls -lh ${BACKUP_FILE}
```

**Inspect the disk image format and virtual size**
Shows the virtual size, disk size, and format (usually `raw`)
```bash
qemu-img info ${BACKUP_FILE}
```

---

## 6. Step 3: Simulating Disaster (Data Deletion)

Now we simulate a catastrophic failure where the underlying storage data is lost.

### Concept
Deleting the RBD image from Ceph removes the actual data blocks. OpenStack Nova/Cinder will still think the volume exists in its database, but any I/O operation will fail.

### Practical Guide

#### 1. Delete the RBD Image
Remove the active image from the Ceph pool.

**WARNING: This deletes the live data. Ensure backup is complete.**
Delete the RBD image permanently
```bash
rbd rm ${CEPH_POOL}/${RBD_IMAGE}
```

#### 2. Confirm Deletion
Verify that the image no longer exists in Ceph.

**Try to list the image again**
Should return empty output
```bash
rbd ls ${CEPH_POOL} | grep ${VOLUME_ID}
```

**Or try to get info (should fail)**
Should return "No such file or directory" error
```bash
rbd info ${CEPH_POOL}/${RBD_IMAGE}
```

---

## 7. Step 4: Verifying Instance Failure

After deleting the backend storage, the OpenStack instance should fail.

### Concept
Nova computes rely on Libvirt/QEMU to access the RBD device. If the device disappears, the VM crashes or hangs.

### Practical Guide

#### 1. Check Instance Status
Observe how OpenStack reports the instance state.

**Check the status of the instance**
Likely shows `ERROR`, `SHUTOFF`, or remains `ACTIVE` temporarily until next heartbeat
```bash
openstack server show <INSTANCE_NAME> -c status -c fault
```

#### 2. Check Console Logs
Look for I/O errors in the guest OS logs.

**Retrieve the last 20 lines of the console log**
Look for kernel panic or I/O errors
```bash
openstack console log show <INSTANCE_NAME> | tail -n 20
```

#### 3. Attempt SSH (Should Fail)
Try to connect to the instance to confirm unreachability.

**Replace with your actual Floating IP or Username**
Connection should time out or be refused
```bash
ssh cirros@<FLOATING_IP>
```

---

## 8. Step 5: Restoring Data to Ceph (Import)

To recover, we must recreate the RBD image and import the data from our backup file.

### Concept
1.  Create a new, empty RBD image of the same size.
2.  Import the backup `.img` file into this new image.

### Practical Guide

#### 1. Determine Disk Size
We need the exact size to create the new image. Check the backup file info again.

**Get the virtual size from the backup file**
Note the size in MB or GB (e.g., 20 GiB)
```bash
qemu-img info ${BACKUP_FILE} | grep "virtual size"
```

#### 2. Create New Empty RBD Image
Create a placeholder image in the Ceph pool.

**Define size in MB (e.g., 20 GB = 20480 MB)**
Adjust this value based on the output from the previous step
```bash
export DISK_SIZE_MB=20480
```

**Create the new RBD image**
`--image-format 2` is standard for modern Ceph/OpenStack
```bash
rbd create ${CEPH_POOL}/${RBD_IMAGE} --size ${DISK_SIZE_MB} --image-format 2
```

#### 3. Import Backup Data
Write the backup file content into the new RBD image.

**Import the local backup file into the new RBD image**
Reads the local file and writes it block-by-block to Ceph
```bash
rbd import ${BACKUP_FILE} ${CEPH_POOL}/${RBD_IMAGE} --progress
```

#### 4. Verify Restored Image
Confirm the image is back and has the correct size.

**Check image info**
Validation: The size and format should match the original
```bash
rbd info ${CEPH_POOL}/${RBD_IMAGE}
```

**Check if it appears in the pool list**
Confirms the image is visible in the pool
```bash
rbd ls ${CEPH_POOL} | grep ${VOLUME_ID}
```

---

## 9. Step 6: OpenStack Instance Recovery & Re-attachment

Even though the data is back in Ceph, OpenStack Nova might still hold a stale state or the Libvirt domain might need a refresh.

### Concept
We need to force OpenStack to re-evaluate the volume attachment. The safest way is to detach and re-attach the volume, then reboot the instance.

### Practical Guide

#### 1. Detach Volume from Instance
Forcefully remove the volume record from the instance.

**Detach the volume**
Replace `<INSTANCE_NAME>` and `<VOLUME_ID>` with actual values
```bash
openstack server remove volume <INSTANCE_NAME> <VOLUME_ID>
```

#### 2. Attach Volume Back to Instance
Re-attach the now-restored volume.

**Attach the volume back**
Maps the Cinder volume back to the Nova instance
```bash
openstack server add volume <INSTANCE_NAME> <VOLUME_ID>
```

#### 3. Hard Reboot the Instance
Restart the VM to force the Guest OS to re-detect the disk and boot.

**Perform a hard reboot (power cycle)**
Simulates pulling the power plug and turning it back on
```bash
openstack server reboot <INSTANCE_NAME> --hard
```

#### 4. Monitor Boot Status
Watch the instance state transition.

**Watch the status change**
Expected Flow: `REBOOT` -> `BUILD` -> `ACTIVE`
```bash
watch -n 5 openstack server show <INSTANCE_NAME> -c status
```

---

## 10. Step 7: Final Verification & Data Integrity Check

The ultimate test: Is the data intact?

### Practical Guide

#### 1. Check Console Log for Successful Boot
Ensure the OS booted without kernel panics.

**Check the end of the console log for login prompt**
Look for systemd startup messages completing
```bash
openstack console log show <INSTANCE_NAME> | tail -n 10
```

#### 2. SSH into the Instance
Connect to the VM.

**SSH into the instance**
Use the floating IP assigned to the instance
```bash
ssh cirros@<FLOATING_IP>
```

#### 3. Verify Data Integrity
Check if the files you created before the disaster are present.

**Inside the VM: List home directory**
Verify user files exist
```bash
ls -l /home/cirros/
```

**Inside the VM: Check disk usage**
Ensure filesystem is mounted and readable
```bash
df -h /
```

**Inside the VM: Check specific test file**
If you created a test file earlier, cat it to verify content
```bash
cat /home/cirros/test-data.txt
```

**Exit VM**
Return to the host terminal
```bash
exit
```

---

## 11. Production Best Practices & Alternative DR Strategies

While the manual RBD method is educational, production environments require robust, automated, and consistent strategies.

### Strategy 1: Cinder Backup (Recommended)
Use the native OpenStack backup service. It integrates with Ceph Object Gateway (RGW) or other backends.

**Why?**
*   Application-consistent (can quiesce filesystem).
*   Incremental backups supported.
*   Managed via API.

**Implementation Steps:**
1.  Enable `cinder_backup` in Kolla-Ansible `globals.yml`.
2.  Set backend to `ceph`.
3.  Use `openstack volume backup create`.

**Production Backup Command**
Creates a backup via Cinder API
```bash
openstack volume backup create --name prod-backup-01 --force <VOLUME_ID>
```

**Production Restore Command**
Restores backup to a new or existing volume
```bash
openstack volume backup restore <BACKUP_ID> <NEW_VOLUME_ID>
```

### Strategy 2: Ceph RBD Mirroring (Site-to-Site DR)
For high availability across two data centers.

**Why?**
*   Real-time replication.
*   Near-zero RPO (Recovery Point Objective).
*   Automatic failover capability.

**Implementation Overview:**
1.  Deploy two Ceph clusters (Site A and Site B).
2.  Enable `rbd-mirror` daemon on both.
3.  Bootstrap peer trust.
4.  Enable journaling on RBD images.

### Strategy 3: Snapshot-Based Cloning
For quick testing or non-critical DR.

**Create Snapshot**
Takes a Cinder-level snapshot
```bash
openstack volume snapshot create --volume <VOLUME_ID> snap-01
```

**Create Volume from Snapshot**
Creates a new volume from the snapshot
```bash
openstack volume create --snapshot snap-01 restored-vol-01
```

---

## 12. Troubleshooting & Common Errors

### Error 1: `rbd: permission denied`
**Cause:** Missing Ceph keyring or wrong permissions.
**Solution:**
Ensure you are using the admin keyring. In Kolla, run commands inside the `ceph_mon` or `kolla_toolbox` container.

**Run inside kolla_toolbox**
Executes shell inside the toolbox container and sets Ceph args
```bash
docker exec -it kolla_toolbox bash
export CEPH_ARGS="--conf /etc/ceph/ceph.conf --keyring /etc/ceph/ceph.client.admin.keyring"
rbd ls volumes
```

### Error 2: `Image is locked` or `Exclusive lock conflict`
**Cause:** The VM is running and holding an exclusive lock on the RBD image.
**Solution:**
You cannot delete or overwrite a locked image easily.
1.  Stop the instance: `openstack server stop <INSTANCE_NAME>`
2.  Break the lock (Advanced):

**List locks on the image**
Identifies the locker ID and address
```bash
rbd lock list ${CEPH_POOL}/${RBD_IMAGE}
```

**Remove the lock manually**
Replace `<locker-id>` and `<locker-address>` with values from the list command
```bash
rbd lock remove ${CEPH_POOL}/${RBD_IMAGE} <locker-id> <locker-address>
```

### Error 3: `No such file or directory` during Export
**Cause:** Typo in Pool name or Image name.
**Solution:**
Double-check names with `rbd ls <pool>`. Remember that Cinder Volume IDs are UUIDs, and RBD images are usually `volume-<UUID>`.

### Error 4: Instance stays in `ERROR` after Restore
**Cause:** Nova database state mismatch or Libvirt configuration stale.
**Solution:**
1.  Detach and Re-attach the volume (as shown in Step 6).
2.  If that fails, rebuild the instance using the restored volume as the root disk (requires creating a new server with `--volume` flag).

### Error 5: Slow Export/Import
**Cause:** Network bottleneck or slow OSDs.
**Solution:**
*   Run the backup job from a node close to the Ceph Public Network.
*   Increase `rbd_concurrent_management_ops` in Ceph config (advanced).
*   Use `--parallel` flag if available in newer RBD versions.

---

## Conclusion

This guide has walked you through the manual process of backing up and restoring an OpenStack instance using Ceph RBD commands. You have learned how to:
1.  Map OpenStack volumes to Ceph images.
2.  Create consistent snapshots and export them.
3.  Simulate storage failure.
4.  Restore data and recover the OpenStack instance.

While this method is powerful for understanding the infrastructure, always prioritize **Cinder Backup** or **RBD Mirroring** for production workloads to ensure data consistency, automation, and minimal downtime.