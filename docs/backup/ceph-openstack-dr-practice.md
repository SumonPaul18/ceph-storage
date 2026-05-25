# OpenStack & Ceph Disaster Recovery: Manual RBD Backup/Restore & Production Strategies

## 1. Introduction & Scenario Overview

### 1.1 Context
This guide is designed for DevOps Engineers and System Administrators managing a **Kolla-Ansible deployed OpenStack** infrastructure integrated with a **Ceph Storage Cluster**. The primary focus is on **Disaster Recovery (DR)** strategies for OpenStack Instances whose data resides on Ceph RBD (RADOS Block Device).

### 1.2 Objective
The user (Sumon) requested a practical, step-by-step solution to:
1.  Manually backup a specific OpenStack Instance's data directly from the Ceph Cluster.
2.  Simulate a disaster by deleting the data from Ceph.
3.  Verify the instance failure in OpenStack.
4.  Restore the data from the manual backup.
5.  Verify the instance recovery.
6.  Understand production-grade alternatives for robust DR planning.

### 1.3 Scope
-   **Manual Method:** Direct interaction with Ceph RBD commands (`rbd export/import`) for deep understanding and emergency recovery when OpenStack services are compromised.
-   **Production Method:** Using OpenStack Cinder Backup services and Ceph RBD Mirroring for automated, consistent, and low-downtime recovery.
-   **Environment:** Single-node or Multi-node Kolla-Ansible OpenStack with Ceph backend.

---

## 2. Table of Contents

1.  Prerequisites & Environment Setup
2.  Identification of Instance & Ceph Resources
3.  Manual Disaster Recovery: Step-by-Step Execution
4.  Production-Grade Disaster Recovery Strategies
5.  Automated Backup Scripting & Maintenance
6.  Troubleshooting & Verification
7.  Cleanup & Uninstall Procedures

---

## 3. Prerequisites & Environment Setup

Before executing any DR operations, ensure your environment is ready. This section covers the necessary tools and access rights.

### 3.1 Required Tools
You need access to the **Deploy Node** or **Controller Node** where OpenStack CLI and Ceph tools are available.

*   **OpenStack Client:** To manage instances and volumes.
*   **Ceph Common Tools:** `ceph`, `rbd` commands to interact with the storage cluster.
*   **QEMU Utils:** `qemu-img` to verify disk images.

### 3.2 Access Rights
*   Root or Sudo access on the host machine.
*   Access to Ceph Admin Keyring (usually located in `/etc/kolla/config/ceph/` or inside the `kolla_toolbox` container).

### 3.3 Verifying Tool Availability

Check if the necessary commands are installed and accessible.

```bash
# Check OpenStack Client version
# Ensures the openstack CLI is installed and reachable
openstack --version
```

```bash
# Check Ceph RBD tool version
# Ensures the rbd command is available for block device operations
rbd --version
```

```bash
# Check QEMU Image tool version
# Ensures qemu-img is available for disk format verification
qemu-img --version
```

### 3.4 Accessing Ceph Tools in Kolla-Ansible
In a Kolla-Ansible deployment, Ceph tools are often containerized. You can either install them on the host or use the `kolla_toolbox` container.

**Option A: Using Host Installed Tools (If available)**
Ensure `/etc/ceph/ceph.conf` and `ceph.client.admin.keyring` are present.

**Option B: Using Kolla Toolbox Container (Recommended)**
This ensures you use the exact versions compatible with your deployment.

```bash
# Enter the kolla_toolbox container
# This provides a shell with all OpenStack and Ceph clients pre-configured
docker exec -it kolla_toolbox bash
```

```bash
# Export Ceph arguments inside the container
# Sets the config and keyring path for subsequent ceph/rbd commands
export CEPH_ARGS="--conf /etc/ceph/ceph.conf --keyring /etc/ceph/ceph.client.admin.keyring"
```

---

## 4. Identification of Instance & Ceph Resources

To backup or restore, you must map the OpenStack Instance to its underlying Ceph RBD Image.

### 4.1 Identify OpenStack Instance and Volume

First, find the Instance ID and the attached Volume ID.

```bash
# List all servers to find the target instance
# Replace 'test-vm-01' with your actual instance name
openstack server list --name test-vm-01
```

```bash
# Get detailed information about the instance
# Extracts the volume ID attached to the instance
INSTANCE_NAME="test-vm-01"
VOLUME_ID=$(openstack server show $INSTANCE_NAME -f value -c volumes_attached | awk '{print $2}' | tr -d "'")
echo "Target Volume ID: $VOLUME_ID"
```

```bash
# Verify Volume Status and Size
# Ensures the volume is 'in-use' or 'available' and notes the size for restoration
openstack volume show $VOLUME_ID -c status -c size -c name
```

### 4.2 Map Volume to Ceph RBD Image

OpenStack Cinder typically names Ceph RBD images as `volume-<UUID>`. The default pool is usually `volumes` or `cinder-volumes`.

```bash
# List Ceph Pools
# Identifies the correct pool where Cinder stores volumes
ceph osd pool ls
```

```bash
# Define Variables for Ceph Operations
# Set the pool name (commonly 'volumes') and the image name based on Volume ID
CEPH_POOL="volumes"
RBD_IMAGE="volume-$VOLUME_ID"
```

```bash
# Verify RBD Image Existence
# Confirms that the Ceph image exists and displays its metadata
rbd info $CEPH_POOL/$RBD_IMAGE
```

*   **`rbd info`**: Displays details like size, format, and features of the RBD image.
*   **`$CEPH_POOL`**: The Ceph pool name (e.g., `volumes`).
*   **`$RBD_IMAGE`**: The RBD image name (e.g., `volume-1234-5678`).

---

## 5. Manual Disaster Recovery: Step-by-Step Execution

This section details the manual process of exporting, deleting, and importing RBD images. This is a **high-risk** operation intended for learning or emergency scenarios where OpenStack services are down.

### 5.1 Phase 1: Backup (Export)

We will create a consistent snapshot and export it to a local file.

#### 5.1.1 Create a Backup Directory

```bash
# Create a directory for storing backups
# Organizes backup files in a specific location
mkdir -p /root/ceph-dr-backup
cd /root/ceph-dr-backup
```

#### 5.1.2 Create a Ceph Snapshot

Taking a snapshot ensures data consistency during the export process, even if the VM is running.

```bash
# Generate a unique snapshot name
# Uses timestamp to ensure uniqueness
SNAP_NAME="dr-snap-$(date +%s)"
```

```bash
# Create a read-only snapshot of the RBD image
# Locks the state of the image at this point in time
rbd snap create $CEPH_POOL/$RBD_IMAGE@$SNAP_NAME
```

```bash
# List snapshots to verify creation
# Confirms the snapshot exists and is protected
rbd snap list $CEPH_POOL/$RBD_IMAGE
```

*   **`rbd snap create`**: Creates a point-in-time copy of the image.
*   **`@$SNAP_NAME`**: Appends the snapshot name to the image identifier.

#### 5.1.3 Export Snapshot to Local File

Exporting converts the RBD image into a standard raw disk image (`.img`).

```bash
# Define the backup file path
# Stores the exported image in the backup directory
BACKUP_FILE="/root/ceph-dr-backup/${RBD_IMAGE}.img"
```

```bash
# Export the snapshot to a local file
# Copies data from Ceph cluster to local filesystem
rbd export $CEPH_POOL/$RBD_IMAGE@$SNAP_NAME $BACKUP_FILE
```

*   **`rbd export`**: Reads data from the Ceph cluster and writes it to a local file.
*   **`$BACKUP_FILE`**: The destination path for the exported image.

#### 5.1.4 Verify Backup Integrity

```bash
# Check file size and existence
# Ensures the file was created and has non-zero size
ls -lh $BACKUP_FILE
```

```bash
# Inspect the disk image format
# Verifies the virtual size and format (raw) of the exported image
qemu-img info $BACKUP_FILE
```

### 5.2 Phase 2: Disaster Simulation (Delete)

We will now delete the RBD image from Ceph to simulate data loss.

#### 5.2.1 Delete the RBD Image

```bash
# Remove the RBD image from the Ceph pool
# WARNING: This action is irreversible without backup
rbd rm $CEPH_POOL/$RBD_IMAGE
```

*   **`rbd rm`**: Permanently deletes the specified RBD image from the pool.

#### 5.2.2 Verify Deletion

```bash
# Confirm the image no longer exists
# Should return empty or error if image is gone
rbd ls $CEPH_POOL | grep $VOLUME_ID
```

#### 5.2.3 Observe OpenStack Instance Failure

With the backend storage gone, the Instance will fail.

```bash
# Check Instance Status
# Expected status: ERROR or SHUTOFF
openstack server show $INSTANCE_NAME -c status -c fault
```

```bash
# Check Console Logs for I/O Errors
# Displays recent log entries showing disk access failures
openstack console log show $INSTANCE_NAME | tail -n 20
```

### 5.3 Phase 3: Restoration (Import)

We will recreate the RBD image and import the backup data.

#### 5.3.1 Recreate the RBD Image

You must create an empty image of the same size before importing.

```bash
# Get the original size from the backup file
# Extracts the virtual size in bytes or GB
ORIGINAL_SIZE_GB=$(qemu-img info $BACKUP_FILE | grep "virtual size" | awk '{print $4}' | tr -d '(')
echo "Original Size: $ORIGINAL_SIZE_GB"
```

```bash
# Convert size to MB for rbd create (if necessary)
# Example: 20G = 20480 MB
DISK_SIZE_MB=20480 
```

```bash
# Create a new empty RBD image
# Allocates space in the Ceph pool for the restored data
rbd create $CEPH_POOL/$RBD_IMAGE --size $DISK_SIZE_MB --image-format 2
```

*   **`--size`**: Specifies the size of the new image (in MB).
*   **`--image-format 2`**: Uses the latest RBD format with better features.

#### 5.3.2 Import Backup Data

```bash
# Import the local backup file into the new RBD image
# Writes data from the local file back into the Ceph cluster
rbd import $BACKUP_FILE $CEPH_POOL/$RBD_IMAGE
```

*   **`rbd import`**: Reads a local file and writes it into an RBD image.

#### 5.3.3 Verify Restoration

```bash
# Check RBD Image Info
# Confirms the image exists and matches the expected size
rbd info $CEPH_POOL/$RBD_IMAGE
```

### 5.4 Phase 4: OpenStack Recovery

OpenStack needs to re-establish the connection to the restored volume.

#### 5.4.1 Detach and Re-attach Volume

Forcing a detach/attach cycle refreshes the Nova-Cinder connection.

```bash
# Detach the volume from the instance
# Removes the volume mapping from the instance record
openstack server remove volume $INSTANCE_NAME $VOLUME_ID
```

```bash
# Attach the volume back to the instance
# Re-maps the volume to the instance
openstack server add volume $INSTANCE_NAME $VOLUME_ID
```

#### 5.4.2 Hard Reboot Instance

A hard reboot forces the hypervisor to re-read the disk configuration.

```bash
# Perform a hard reboot
# Powers off and on the instance, forcing OS to detect disk
openstack server reboot $INSTANCE_NAME --hard
```

#### 5.4.3 Verify Instance Activity

```bash
# Wait for the instance to become active
# Pauses execution for 60 seconds to allow boot process
sleep 60
```

```bash
# Check Final Status
# Expected status: ACTIVE
openstack server show $INSTANCE_NAME -c status
```

```bash
# Check Console Log for Boot Success
# Looks for login prompts or successful boot messages
openstack console log show $INSTANCE_NAME | grep "login"
```

---

## 6. Production-Grade Disaster Recovery Strategies

Manual RBD export/import is not suitable for production due to downtime and complexity. Use these methods instead.

### 6.1 Strategy 1: Cinder Backup to Object Store (Recommended)

This method uses OpenStack's native backup service to store volume backups in Ceph Object Gateway (RGW) or S3.

#### 6.1.1 Configuration in Kolla-Ansible

Enable Cinder Backup in `globals.yml`.

```yaml
# Enable Cinder Backup Service
enable_cinder_backup: "yes"

# Set Backend to Ceph
cinder_backup_driver: "ceph"

# Define Backup Pool
cinder_backup_ceph_pool: "backups"
```

#### 6.1.2 Deploy Changes

```bash
# Run Kolla-Ansible deploy for Cinder Backup
# Applies the configuration changes to the cluster
kolla-ansible deploy -t cinder-backup
```

#### 6.1.3 Taking a Backup

```bash
# Create a backup of the volume
# --force allows backup of attached volumes
openstack volume backup create --name daily-backup-vol1 --force <volume-id>
```

*   **`--force`**: Overrides the safety check for attached volumes.
*   **`--name`**: Assigns a human-readable name to the backup.

#### 6.1.4 Restoring from Backup

```bash
# Create a new volume from the backup
# Restores data to a new volume ID
openstack volume create --backup <backup-id> --size <size-in-gb> restored-vol1
```

```bash
# Attach the restored volume to the instance
openstack server add volume <instance-id> restored-vol1
```

### 6.2 Strategy 2: Ceph RBD Mirroring (Site-to-Site DR)

For high availability across two physical sites, use Ceph RBD Mirroring.

#### 6.2.1 Enable RBD Mirror Daemon

```bash
# Enable RBD Mirror module on Manager
ceph mgr module enable rbd-mirror
```

#### 6.2.2 Bootstrap Peer Relationship

```bash
# Create bootstrap token on Primary Site
rbd mirror pool peer bootstrap create --site-name site-a --direction rx-tx > site-a-token
```

```bash
# Import token on Secondary Site
rbd mirror pool peer bootstrap import --site-name site-b --direction rx-tx site-a-token
```

#### 6.2.3 Enable Mirroring on Images

```bash
# Enable journaling mirroring for specific images
rbd mirror image enable volumes/<image-name> journal
```

---

## 7. Automated Backup Scripting & Maintenance

Automate the manual process using Bash scripts and Cron jobs.

### 7.1 Backup Script Structure

Create a script `/opt/scripts/openstack-dr-backup.sh`.

```bash
#!/bin/bash
# Define Retention Policy
RETENTION_DAYS=7

# Get Volume ID
VOLUME_ID=$(openstack volume list -f value -c ID -c Name | grep "important-vol" | awk '{print $1}')

# Create Backup
openstack volume backup create --name "dr-backup-$(date +%F)" --force $VOLUME_ID

# Cleanup Old Backups
openstack volume backup list -f value -c ID -c Created_At | while read line; do
    # Logic to delete backups older than RETENTION_DAYS
    echo "Cleanup logic here"
done
```

### 7.2 Schedule with Cron

```bash
# Edit Crontab
crontab -e
```

```cron
# Run backup every day at 2 AM
0 2 * * * /bin/bash /opt/scripts/openstack-dr-backup.sh >> /var/log/dr-backup.log 2>&1
```

---

## 8. Troubleshooting & Verification

Common issues and their solutions during DR operations.

### 8.1 Permission Denied

**Issue:** `rbd` commands fail with permission errors.
**Solution:** Ensure you are using the correct keyring.

```bash
# Specify keyring explicitly
rbd --keyring /etc/ceph/ceph.client.admin.keyring ls volumes
```

### 8.2 Image Not Found

**Issue:** `rbd info` returns "No such file or directory".
**Solution:** Verify the Pool Name and Image Name.

```bash
# List all images in the pool to find the correct name
rbd ls volumes
```

### 8.3 Instance Stuck in ERROR State

**Issue:** Instance does not recover after restore.
**Solution:** Force delete and rebuild.

```bash
# Reset State
openstack server set --state active $INSTANCE_NAME
```

```bash
# Rebuild Instance with existing volume
openstack server rebuild --volume $VOLUME_ID $INSTANCE_NAME
```

### 8.4 Data Consistency Check

**Issue:** Files are corrupted after restore.
**Solution:** Always use Snapshots before Export.

```bash
# Verify Filesystem
ssh user@<instance-ip> "fsck /dev/vdb"
```

---

## 9. Cleanup & Uninstall Procedures

How to remove backups and temporary files.

### 9.1 Remove Manual Backups

```bash
# Delete local backup files
rm -rf /root/ceph-dr-backup/*.img
```

### 9.2 Remove Ceph Snapshots

```bash
# List snapshots
rbd snap list $CEPH_POOL/$RBD_IMAGE
```

```bash
# Remove specific snapshot
rbd snap rm $CEPH_POOL/$RBD_IMAGE@dr-snap-123456
```

### 9.3 Remove Cinder Backups

```bash
# List backups
openstack volume backup list
```

```bash
# Delete specific backup
openstack volume backup delete <backup-id>
```

### 9.4 Disable Cinder Backup (If needed)

```yaml
# In globals.yml
enable_cinder_backup: "no"
```

```bash
# Redeploy
kolla-ansible deploy -t cinder-backup
```

---

## 10. Conclusion

This guide provided a comprehensive walkthrough of manual and production-grade Disaster Recovery for OpenStack instances on Ceph. While manual RBD export/import offers deep insight into the storage layer, **Cinder Backup** and **RBD Mirroring** are the recommended standards for production environments due to their automation, consistency, and minimal downtime capabilities. Regular DR drills using these methods ensure infrastructure resilience.