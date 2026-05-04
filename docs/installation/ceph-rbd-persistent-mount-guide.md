# Ceph RBD: Persistent Mapping and Mounting Guide

## Table of Contents
1. [Introduction](#introduction)
2. [Scenario Analysis: New vs. Existing Images](#scenario-analysis-new-vs-existing-images)
3. [How to Identify Image Status](#how-to-identify-image-status)
4. [Mapping Multiple Images](#mapping-multiple-images)
5. [Production Setup: Persistent Mapping & Mounting](#production-setup-persistent-mapping--mounting)
6. [Verification and Testing](#verification-and-testing)

---

## Introduction

In a production cloud environment, managing storage efficiently is critical. When using Ceph RBD (RADOS Block Device), you often need to map block devices to client machines (like OpenStack compute nodes or bare-metal servers). A common confusion arises when moving an RBD image from one client to another: **Do we need to format it again?**

This guide provides a practical, step-by-step approach to handling RBD images, identifying their state, and ensuring they remain mounted even after server reboots. We will focus on real-world commands and avoid unnecessary theory.

---

## Scenario Analysis: New vs. Existing Images

### The Core Rule
*   **New Image:** If the RBD image is brand new (just created), it has no filesystem. You **must** format it (e.g., with `ext4` or `xfs`) before mounting.
*   **Existing Image:** If the image was previously used and formatted, it already contains a filesystem signature. **DO NOT FORMAT IT AGAIN.** Formatting will erase all existing data. You simply map and mount it.

### Real-World Scenario
Imagine you have an RBD image named `db-data` used by **Server A**. You unmount it from Server A and want to attach it to **Server B** for maintenance or migration.
*   **Question:** Do you run `mkfs` on Server B?
*   **Answer:** No. The filesystem structure exists inside the image. You just map and mount it.

---

## How to Identify Image Status

Before mounting, you must verify if the mapped device contains data or is empty. This prevents accidental data loss.

### Step 1: Map the RBD Image
First, map the image to the local block device.

```bash
# Map the RBD image to a local device (e.g., /dev/rbd0)
sudo rbd map <pool-name>/<image-name>
```

*Example:*
```bash
sudo rbd map mypool/db-data
```

### Step 2: Check Filesystem Signature
Use the `file` command to inspect the block device. This tells you if a filesystem exists.

```bash
# Check the type of data on the mapped device
sudo file -s /dev/rbd/<pool-name>/<image-name>
```

*Example:*
```bash
sudo file -s /dev/rbd/mypool/db-data
```

#### Interpreting the Output

**Case A: Empty/New Image**
If the output is:
```text
/dev/rbd/mypool/db-data: data
```
*   **Meaning:** The image is empty. No filesystem exists.
*   **Action:** You can safely run `mkfs` to create a filesystem.

**Case B: Existing Ext4 Filesystem**
If the output is:
```text
/dev/rbd/mypool/db-data: Linux rev 1.0 ext4 filesystem data...
```
*   **Meaning:** The image has an ext4 filesystem and likely contains data.
*   **Action:** **SKIP** `mkfs`. Proceed directly to mounting.

**Case C: Existing XFS Filesystem**
If the output is:
```text
/dev/rbd/mypool/db-data: SGI XFS filesystem data...
```
*   **Meaning:** The image has an XFS filesystem.
*   **Action:** **SKIP** `mkfs`. Proceed directly to mounting.

### Step 3: Optional Data Verification
If you want to see the files without committing to a permanent mount, use a temporary mount point.

```bash
# Create a temporary directory
sudo mkdir -p /mnt/temp-check

# Mount the device temporarily
sudo mount /dev/rbd/mypool/db-data /mnt/temp-check

# List files to verify data
ls -la /mnt/temp-check

# Unmount immediately after checking
sudo umount /mnt/temp-check
```

---

## Mapping Multiple Images

### Can I map multiple RBD images to one client?
**Yes.** In production, it is standard practice to map multiple RBD images to a single client. Each image acts as an independent virtual disk.

### Best Practices
1.  **Isolation:** Use separate images for different applications (e.g., `web-data`, `db-log`, `backup-store`).
2.  **Performance:** Ceph handles parallel I/O efficiently. Mapping multiple images does not degrade performance if your network bandwidth is sufficient.
3.  **Device Names:** Linux assigns device names like `/dev/rbd0`, `/dev/rbd1`, etc. However, rely on the symbolic links under `/dev/rbd/<pool>/<image>` for consistency.

### Example: Mapping Two Images
```bash
# Map first image
sudo rbd map mypool/web-data

# Map second image
sudo rbd map mypool/db-logs

# Verify both are mapped
lsblk | grep rbd
```

---

## Production Setup: Persistent Mapping & Mounting

In a production environment, you cannot manually map and mount devices after every reboot. You must automate this process using two system configurations:
1.  **`/etc/ceph/rbdmap`**: Handles automatic mapping of RBD images at boot.
2.  **`/etc/fstab`**: Handles automatic mounting of the filesystem at boot.

### Step 1: Configure Auto-Mapping (rbdmap)

The `ceph-common` package provides the `rbdmap` service.

#### Edit the rbdmap Configuration File
Open the configuration file:

```bash
sudo nano /etc/ceph/rbdmap
```

Add your RBD image entry at the end of the file. The format is:
`<pool>/<image> id=<user>,keyring=<path-to-keyring>`

*Example Entry:*
```text
mypool/db-data id=admin,keyring=/etc/ceph/ceph.client.admin.keyring
```

> **Note:** For better security, create a specific Ceph user with limited permissions instead of using `admin`.

#### Enable and Start the rbdmap Service
Ensure the service starts automatically on boot.

```bash
# Enable the service to start on boot
sudo systemctl enable rbdmap

# Start the service now
sudo systemctl start rbdmap
```

### Step 2: Configure Auto-Mounting (fstab)

Now that the device maps automatically, we need to mount it to a directory.

#### Create the Mount Point
Create the directory where you want to access the storage.

```bash
sudo mkdir -p /mnt/ceph-db-data
```

#### Edit the fstab File
Open the filesystem table:

```bash
sudo nano /etc/fstab
```

Add the following line. Replace `<pool>/<image>` and filesystem type (`ext4` or `xfs`) as appropriate.

**Important:** Use the `_netdev` option. This tells Linux to wait for the network to be ready before mounting, which is crucial for network storage like Ceph.

*For Ext4:*
```text
/dev/rbd/mypool/db-data  /mnt/ceph-db-data  ext4  _netdev,noatime  0  0
```

*For XFS:*
```text
/dev/rbd/mypool/db-data  /mnt/ceph-db-data  xfs  _netdev,noatime  0  0
```

> **Why use `/dev/rbd/pool/image`?**
> Using the full path `/dev/rbd/mypool/db-data` is safer than `/dev/rbd0`. The numeric ID (`rbd0`) can change after a reboot, but the symbolic link path remains constant.

---

## Verification and Testing

After configuring `rbdmap` and `fstab`, you must verify that everything works correctly without rebooting immediately.

### Step 1: Unmap and Unmount Current Session
Clean up any existing manual mounts to simulate a fresh start.

```bash
# Unmount the filesystem
sudo umount /mnt/ceph-db-data

# Unmap the RBD device
sudo rbd unmap /dev/rbd/mypool/db-data
```

### Step 2: Trigger Auto-Mapping
Restart the `rbdmap` service to simulate the boot process mapping.

```bash
sudo systemctl restart rbdmap
```

Verify the device is mapped:
```bash
ls -l /dev/rbd/mypool/db-data
```

### Step 3: Trigger Auto-Mounting
Use `mount -a` to test the `fstab` entries.

```bash
# Mount all filesystems defined in fstab
sudo mount -a
```

### Step 4: Final Verification
Check if the storage is available and writable.

```bash
# Check disk usage
df -h | grep ceph-db-data

# Test write permission
sudo touch /mnt/ceph-db-data/test-file
ls -la /mnt/ceph-db-data/test-file

# Clean up test file
sudo rm /mnt/ceph-db-data/test-file
```

### Step 5: Reboot Test (Recommended)
If the steps above succeed, perform a final test by rebooting the server.

```bash
sudo reboot
```

After the server comes back online, log in and check:
```bash
df -h | grep ceph-db-data
```
If the mount appears, your production setup is complete and persistent.

---

### Summary Checklist for Production

| Task | Command/File | Key Note |
| :--- | :--- | :--- |
| **Check Image State** | `file -s /dev/rbd/pool/img` | Never format if output shows `filesystem`. |
| **Auto-Map Config** | `/etc/ceph/rbdmap` | Add image entry with keyring path. |
| **Enable Mapping Service** | `systemctl enable rbdmap` | Ensures mapping on boot. |
| **Auto-Mount Config** | `/etc/fstab` | Use `_netdev` option. |
| **Device Path** | `/dev/rbd/pool/img` | Use symbolic link, not `/dev/rbd0`. |

This guide ensures your Ceph RBD storage is robust, data-safe, and automatically available across reboots in a professional infrastructure environment.
