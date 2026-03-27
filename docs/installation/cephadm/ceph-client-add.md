# 🔗 Complete Guide: Connect Linux Client to Ceph Cluster
**Step-by-Step | Real-World Examples | Production-Ready**

---

## 📋 Table of Contents
1. [Part 1: Connect Linux Client to Ceph Cluster](#part-1-connect-linux-client-to-ceph-cluster)
2. [Part 2: RBD Image Mapping & Configuration](#part-2-rbd-image-mapping--configuration)
3. [Part 3: Connect RGW & CephFS Services](#part-3-connect-rgw--cephfs-services)

---

# PART 1: Connect Linux Client to Ceph Cluster

## 🎯 What We Will Do
> We will take a fresh Linux machine and connect it to your existing Ceph cluster so it can read/write data.

---

## Step 1: Prepare Your Linux Client Machine

### 1.1 Client Machine Requirements

| Item | Requirement | Why |
|------|-------------|-----|
| **OS** | Ubuntu 24.04 LTS (recommended) | Same as cluster = fewer issues |
| **Network** | Can ping Ceph MON IPs (192.168.10.11, .12, .13) | Client must talk to cluster |
| **RAM** | Minimum 2 GB | For Ceph client tools |
| **Disk** | 20 GB free (for OS + temp data) | For mounting Ceph storage |
| **User** | Root or sudo access | To install packages and configure |

### 1.2 Update Client Machine


#### Log into your CLIENT machine (not Ceph cluster nodes)
#### This is a NEW machine that will access Ceph storage

#### Update package list
```
sudo apt update
```
#### Upgrade existing packages
```
sudo apt upgrade -y
```
#### Install basic tools
```
sudo apt install -y curl wget net-tools dnsutils
```

### 1.3 Configure Network


#### Check network connectivity
```
ip addr show
```
#### Test ping to Ceph cluster nodes
```
ping -c 3 192.168.10.11
ping -c 3 192.168.10.12
ping -c 3 192.168.10.13
```
#### All should respond. If not, check network/firewall


### 1.4 Configure Time Synchronization (Important!)


#### Install chrony for time sync
```
sudo apt install -y chrony
```
#### Enable and start the service
```
sudo systemctl enable --now chrony
```
#### Verify time is synced
```
chronyc sources -v
```
#### Check current time
```
date
```
#### Time must match Ceph cluster nodes (within 100ms)


> ⚠️ **Warning:** If client time differs from cluster time by more than 100ms, authentication will fail!

---

## Step 2: Install Ceph Client Tools

### 2.1 Install Ceph Common Package


#### Install Ceph client tools on CLIENT machine
```
sudo apt install -y ceph-common
```
#### Verify installation
```
ceph --version
```
#### Expected output: ceph version 18.x.x or 19.x.x


### 2.2 Create Ceph Configuration Directory


#### Create directory for Ceph config files
```
sudo mkdir -p /etc/ceph
```
#### Set proper permissions
```
sudo chmod 755 /etc/ceph
```

---

## Step 3: Copy Cluster Configuration to Client

### 3.1 Get ceph.conf from Ceph Admin Node

**On your CEPH ADMIN NODE (Node1), run:**


#### View the ceph.conf file
```
cat /etc/ceph/ceph.conf
```
#### Copy it to your client machine
#### Replace 'client-ip' with your actual client machine IP
```
scp /etc/ceph/ceph.conf user@client-ip:/tmp/ceph.conf
```

### 3.2 Move ceph.conf to Correct Location on Client

**On your CLIENT machine, run:**


#### Move the config file to correct location
```
sudo mv /tmp/ceph.conf /etc/ceph/ceph.conf
```
#### Set proper permissions
```
sudo chmod 644 /etc/ceph/ceph.conf
```
#### Verify the file
```
cat /etc/ceph/ceph.conf
```

### 3.3 Verify ceph.conf Content

Your `/etc/ceph/ceph.conf` should look like this:

```ini
[global]
fsid = a1b2c3d4-e5f6-7890-abcd-ef1234567890
mon_host = 192.168.10.11,192.168.10.12,192.168.10.13
public_network = 192.168.10.0/24

# Optional: Client performance settings
rbd_cache = true
rbd_cache_writethrough_until_flush = true
```

> ✅ **Check:** Make sure `mon_host` has all 3 MON IP addresses separated by commas.

---

## Step 4: Create Client User Authentication

### 4.1 Why Create a Dedicated Client User?

| Approach | Security | Recommendation |
|----------|----------|----------------|
| Use admin keyring | ❌ Low (full cluster access) | Never use in production |
| Create dedicated user | ✅ High (limited access) | Always use this |

> 🔐 **Best Practice:** Create separate users for different purposes (RBD user, CephFS user, RGW user)

### 4.2 Create RBD Client User (On Ceph Admin Node)

**On your CEPH ADMIN NODE (Node1), run:**


#### Create a user specifically for RBD block storage
#### This user can ONLY access the rbd-pool, nothing else
```
sudo cephadm shell -- ceph auth get-or-create client.rbd-user \
  mon 'allow r' \
  osd 'allow rwx pool=rbd-pool' \
  -o /etc/ceph/ceph.client.rbd-user.keyring
```
#### Verify the user was created
```
ceph auth list
```
#### You should see 'client.rbd-user' in the list


### 4.3 Create CephFS Client User (On Ceph Admin Node)


#### Create a user specifically for CephFS file storage
```
sudo cephadm shell -- ceph auth get-or-create client.cephfs-user \
  mon 'allow r' \
  mds 'allow r, allow rw path=/' \
  osd 'allow rw tag cephfs metadata=cephfs_metadata, allow rw tag cephfs data=cephfs_data' \
  -o /etc/ceph/ceph.client.cephfs-user.keyring
```

### 4.4 Copy Keyring Files to Client Machine

**On your CEPH ADMIN NODE (Node1), run:**


#### Copy RBD user keyring to client
```
scp /etc/ceph/ceph.client.rbd-user.keyring user@client-ip:/tmp/ceph.client.rbd-user.keyring
```
#### Copy CephFS user keyring to client
```
scp /etc/ceph/ceph.client.cephfs-user.keyring user@client-ip:/tmp/ceph.client.cephfs-user.keyring
```
#### Also copy admin keyring for testing (optional, remove after testing)
```
scp /etc/ceph/ceph.client.admin.keyring user@client-ip:/tmp/ceph.client.admin.keyring
```

**On your CLIENT machine, run:**


#### Move keyrings to correct location
```
sudo mv /tmp/ceph.client.rbd-user.keyring /etc/ceph/ceph.client.rbd-user.keyring
sudo mv /tmp/ceph.client.cephfs-user.keyring /etc/ceph/ceph.client.cephfs-user.keyring
sudo mv /tmp/ceph.client.admin.keyring /etc/ceph/ceph.client.admin.keyring
```
#### Set secure permissions (VERY IMPORTANT!)
```
sudo chmod 600 /etc/ceph/ceph.client.rbd-user.keyring
sudo chmod 600 /etc/ceph/ceph.client.cephfs-user.keyring
sudo chmod 600 /etc/ceph/ceph.client.admin.keyring
```
#### Set ownership to root
```
sudo chown root:root /etc/ceph/ceph.client.*.keyring
```
#### Verify permissions
```
ls -la /etc/ceph/
```

> ⚠️ **Security Warning:** Keyring files must have 600 permissions. If permissions are too open, Ceph will refuse to use them!

---

## Step 5: Test Client Connection to Cluster

### 5.1 Test Connection with RBD User


#### Test cluster status using the rbd-user credentials
```
ceph --name client.rbd-user --keyring /etc/ceph/ceph.client.rbd-user.keyring -s
```

**Expected Output:**
```
  cluster:
    id:     a1b2c3d4-e5f6-7890-abcd-ef1234567890
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum ceph-node1,ceph-node2,ceph-node3
    mgr: ceph-node1.active
    osd: 6 osds: 6 up, 6 in
```

### 5.2 Test Connection & Verify

#### Check Ceph Cluster Health Status
```bash
ceph -s
```
#### Test with admin credentials (should show more details)
```
ceph --name client.admin --keyring /etc/ceph/ceph.client.admin.keyring -s
```
#### Discovery – Listing Pools and Images

Now that you are connected, let's see what RBD images you have created.

#### List All Pools
First, verify you can see the pools.
```bash
ceph osd pool ls
```

#### List RBD Images in a Specific Pool
Replace `<pool-name>` with your actual pool name (e.g., `rbd`, `data-pool`, `mixed-pool`).

```bash
rbd ls -p <pool-name>
```
*Example:*
```bash
rbd ls -p rbd
```
*Output:*
```text
my-first-image
web-server-disk
database-vol
```

#### Get Detailed Info About an Image
Before mounting, check the size and features of the image.
```bash
rbd info -p <pool-name> <image-name>
```
*Example:*
```bash
rbd info -p rbd my-first-image
```
*Look for:* `size`, `format`, and `features`. Ensure the size matches your expectation.

---



### 5.3 Troubleshooting Connection Issues

| Problem | Command to Diagnose | Solution |
|---------|--------------------|----------|
| Connection refused | `telnet 192.168.10.11 6789` | Open firewall port 6789 |
| Authentication failed | `ceph -s --log-to-stderr` | Check keyring permissions (must be 600) |
| Wrong fsid | Compare fsid in ceph.conf | Copy exact ceph.conf from cluster |
| Time mismatch | `chronyc sources -v` | Sync time with chrony |
| Can't resolve hostnames | `ping ceph-node1` | Add entries to /etc/hosts |

**Add hosts to /etc/hosts if DNS not available:**


#### Edit hosts file
```
sudo nano /etc/hosts
```
#### Add these lines
```
192.168.10.11  ceph-node1
192.168.10.12  ceph-node2
192.168.10.13  ceph-node3
```

---


# PART 2: RBD Image Mapping & Configuration

## 🎯 What We Will Do
> Create RBD images (virtual disks) on the Ceph cluster and map them to the client machine like local hard drives.

---

## Step 1: Create RBD Image on Ceph Cluster

#### 1.1 What is an RBD Image?

> An **RBD Image** is a virtual block device stored in your Ceph cluster. You can think of it as a "virtual hard disk" that lives on the cluster instead of inside your server.

**Real-World Analogy:**
> Like a USB drive, but instead of plugging it into your computer, it lives on the network. You can plug it into any computer that has permission.

<details>
<summary> If not Create RBD Image </summary>

#### 1.2 Create RBD Image (On Ceph Admin Node OR Client)

```bash
# You can run this from client machine if you have admin credentials
# Or run from Ceph admin node

# Create a 50 GB RBD image in rbd-pool
ceph --name client.admin --keyring /etc/ceph/ceph.client.admin.keyring \
  rbd create vm-disk-1 --size 50G --pool rbd-pool --image-format 2

# Verify image was created
ceph --name client.admin --keyring /etc/ceph/ceph.client.admin.keyring \
  rbd ls rbd-pool

# Expected output: vm-disk-1
```

#### RBD Image Options Explained

| Option | Value | Why |
|--------|-------|-----|
| `--size` | 50G | Size of virtual disk |
| `--pool` | rbd-pool | Which storage pool to use |
| `--image-format` | 2 | Use format 2 (supports features like snapshots) |

#### Enable Advanced Features (Optional but Recommended)

```bash
# Enable features for better performance and functionality
ceph --name client.admin --keyring /etc/ceph/ceph.client.admin.keyring \
  rbd feature enable vm-disk-1 exclusive-lock object-map fast-diff deep-flatten

# Verify features are enabled
ceph --name client.admin --keyring /etc/ceph/ceph.client.admin.keyring \
  rbd info vm-disk-1 --pool rbd-pool
```

**Features Explained:**

| Feature | What It Does | Benefit |
|---------|-------------|---------|
| `exclusive-lock` | Only one client can write at a time | Prevents data corruption |
| `object-map` | Tracks which objects contain data | Faster operations |
| `fast-diff` | Quickly calculates changed blocks | Faster snapshots |
| `deep-flatten` | Allows snapshot deletion without copying data | Saves space |

</details>


### 1.3 Mapping the RBD Image to the Client

"Mapping" makes the remote Ceph RBD image appear as a local block device (like `/dev/rbd0`) on your client Linux machine.

#### Check if RBD module is loaded
```
lsmod | grep rbd
```
#### If nothing shows, load the module
```
sudo modprobe rbd
```
#### Make it load automatically on boot
```
echo "rbd" | sudo tee /etc/modules-load.d/rbd.conf
```

#### Map the Image
Run the map command.
```bash
sudo rbd map -p <pool-name> <image-name>
```
*Example:*
```bash
sudo rbd map -p rbd my-first-image
```

*Successful Output:*
```text
/dev/rbd0
```
*Note:* If you map another image, it will become `/dev/rbd1`, and so on.

### Verify the Mapping
Check if the device is recognized by the kernel.
```bash
lsblk | grep rbd
```
*Output:*
```text
rbd0     252:0    0   10G  0 disk
```
You can also see active maps using:
```bash
rbd showmapped
```
*Output Table:*
| pool | image | snap | device |
| :--- | :--- | :--- | :--- |
| rbd | my-first-image | - | /dev/rbd0 |

> **Important:** If the image is **new** and never formatted, `/dev/rbd0` exists but contains no filesystem yet. Proceed to Phase 5. If the image **already has data** and a filesystem, skip to Step 6.2 (Mounting).

---

### 2.2 Understanding the Device


#### Check the device details
```
ls -la /dev/rbd0
```
#### Check device information
```
sudo fdisk -l /dev/rbd0
```
#### You should see something like:
#### Disk /dev/rbd0: 50 GiB, 53687091200 bytes


> ✅ **Success:** Your Ceph storage is now available as `/dev/rbd0` on your client!

---

## Step 3: Format and Mount the RBD Device

### 3.1 Create Filesystem on RBD Device

### Scenario A: New Image (Formatting Required)
If this is a brand new RBD image, you must format it with a filesystem (ext4 or xfs) before mounting.

**Warning:** This erases all data on the device. Only do this on new images.

#### Format with ext4 (Recommended for simplicity)
```bash
sudo mkfs.ext4 /dev/rbd0
```

#### Format with xfs (Recommended for large files/performance)
```bash
sudo mkfs.xfs /dev/rbd0
```

### Scenario B: Existing Image (Skip Formatting)
If the image was previously used and already has a filesystem, **DO NOT RUN MKFS**. Jump directly to mounting.

#### Create a Mount Point
Create a directory where you want to access the storage.
```bash
sudo mkdir -p /mnt/ceph-data
```
#### Set proper permissions
```
sudo chmod 755 /mnt/ceph-data
```

#### Mount the Device
Mount the mapped device to the directory.
```bash
sudo mount /dev/rbd0 /mnt/ceph-data
```

#### Verify Mount
Check if it is mounted successfully.
```bash
df -h | grep rbd
```
*Output:*
```text
/dev/rbd0      10G  100M  9.4G   1% /mnt/ceph-data
```

### 3.4 Test Read/Write Operations


#### Create a test file
```
echo "Hello Ceph RBD Storage!" | sudo tee /mnt/ceph-storage/test.txt
```
#### Read the file back
```
cat /mnt/ceph-storage/test.txt
```
#### Create larger test file (100 MB)
```
sudo dd if=/dev/urandom of=/mnt/ceph-storage/test-100mb.bin bs=1M count=100
```
#### Verify file was created
```
ls -lh /mnt/ceph-storage/
```
#### Check disk usage
```
df -h /mnt/ceph-storage
```

---

## Step 4: Make RBD Mount Persistent (Auto-Mount on Boot)

### 4.1 Create Systemd Service for RBD Mapping

> RBD mappings don't survive reboot by default. We need to create a service to remap on boot.

### Create Mount Entry in fstab


#### Backup fstab first
```
sudo cp /etc/fstab /etc/fstab.backup
```
Open the file system table configuration.
```bash
sudo nano /etc/fstab
```

Add the following line at the end of the file. Replace placeholders with your actual values.

**Syntax:**
```text
# <Device Spec>                                  <Mount Point>   <Type>  <Options>                                                       <Dump> <Pass>
```

**Entry for Ceph RBD:**
```text
/dev/rbd/<pool-name>/<image-name>   /mnt/ceph-data    ext4    defaults,_netdev,noatime,nofail,x-systemd.after=network-online.target   0 2
```

**Real Example:**
If your pool is `rbd` and image is `web-disk`:
```text
/dev/rbd/rbd/web-disk   /mnt/ceph-data    ext4    defaults,_netdev,noatime,nofail,x-systemd.after=network-online.target   0 2
```

#### Explanation of Options (Crucial for Stability):
*   `defaults`: Standard read/write options.
*   `_netdev`: **Most Important.** Tells Linux this is a network device. It waits for the network to be up before trying to mount. Without this, the server might hang on boot.
*   `noatime`: Improves performance by not updating "access time" metadata on every file read. Highly recommended for Ceph.
*   `nofail`: If the Ceph cluster is down during boot, the system will still boot up (instead of dropping to emergency mode). The mount will just fail silently until the cluster is back.
*   `x-systemd.after=network-online.target`: Ensures the mount happens only after the network is fully online, not just "starting".


#### Verify the entry
```
tail -3 /etc/fstab
```
---

### Advanced Permanent Mounting (Systemd Service Method)

For complex environments where `fstab` might be too rigid or if you need to run specific scripts before/after mounting, a **Systemd Service** is the modern, professional approach.

#### Create a Service File
Create a new service unit file.
```bash
sudo nano /etc/systemd/system/ceph-rbd-mount.service
```

#### Define the Service Logic
Paste the following content. Adjust `<pool>`, `<image>`, and `<mount-point>`.

```ini
[Unit]
Description=Mount Ceph RBD Image
After=network-online.target ceph-mon.service
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/sbin/modprobe rbd
ExecStart=/usr/bin/rbd map -p <pool-name> <image-name>
ExecStart=/bin/mount -o noatime /dev/rbd/<pool-name>/<image-name> /mnt/ceph-data
ExecStop=/bin/umount /mnt/ceph-data
ExecStop=/usr/bin/rbd unmap /dev/rbd/<pool-name>/<image-name>

[Install]
WantedBy=multi-user.target
```

**Example Configuration:**
```ini
[Unit]
Description=Mount Ceph RBD Web Disk
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/sbin/modprobe rbd
ExecStart=/usr/bin/rbd map -p rbd web-disk
ExecStart=/bin/mount -o noatime /dev/rbd/rbd/web-disk /mnt/ceph-data
ExecStop=/bin/umount /mnt/ceph-data
ExecStop=/usr/bin/rbd unmap /dev/rbd/rbd/web-disk

[Install]
WantedBy=multi-user.target
```

#### Enable and Start the Service

#### Reload systemd to recognize the new file
```
sudo systemctl daemon-reload
```
#### Enable to start on boot
```
sudo systemctl enable ceph-rbd-mount.service
```
#### Start immediately
```
sudo systemctl start ceph-rbd-mount.service
```
#### Check status
```
sudo systemctl status ceph-rbd-mount.service
```

**Why use this over fstab?**
*   Better logging via `journalctl -u ceph-rbd-mount.service`.
*   You can add `ExecStartPre` checks (e.g., ping the monitor IP before mapping).
*   Cleaner separation of concerns.

---

#### Test the service (unmap first, then start service)
```
sudo umount /mnt/ceph-storage
sudo rbd unmap /dev/rbd0
sudo systemctl start rbd-map-vm-disk-1.service
sudo mount /dev/rbd0 /mnt/ceph-storage
```
#### Verify everything works
```
df -h /mnt/ceph-storage
ls /mnt/ceph-storage/
```

---

## 5. Unmounting & Cleanup

When you are done or need to perform maintenance, follow the correct order to prevent data corruption.

####  Unmount the Filesystem
Always unmount first.
```bash
sudo umount /mnt/ceph-data
```
*Check:* `df -h` should no longer show `/dev/rbd0`.

#### Unmap the RBD Device
Disconnect the block device from the client.
```bash
sudo rbd unmap /dev/rbd0
```
*Alternatively, if you don't know the device name:*
```bash
sudo rbd unmap -p <pool-name> <image-name>
```

#### Verify Unmap
```bash
rbd showmapped
```
*Output:* Should be empty or not list the specific image anymore.

---

## 6. Troubleshooting Common Issues

#### Issue 1: `rbd: map failed: (2) No such file or directory`
*   **Cause:** The `rbd` kernel module is not loaded.
*   **Fix:**
    ```bash
    sudo modprobe rbd
    ```
    Try mapping again. To make it permanent, add `rbd` to `/etc/modules-load.d/ceph.conf`.

#### Issue 2: `Permission denied` or `authentication error`
*   **Cause:** Incorrect keyring permissions or wrong key content.
*   **Fix:**
    1.  Check permissions: `ls -l /etc/ceph/ceph.client.admin.keyring` (Must be `600`).
    2.  Ensure the keyring file has no extra spaces or missing brackets.
    3.  Test with verbose output: `rbd -l debug map ...`

#### Issue 3: `Device /dev/rbd0 is busy` during unmap
*   **Cause:** You forgot to unmount the filesystem, or a process is using a file inside the mount point.
*   **Fix:**
    1.  Find the process: `lsof +f -- /mnt/ceph-data`
    2.  Kill the process or close the terminal session.
    3.  Unmount: `sudo umount /mnt/ceph-data`
    4.  Then unmap.

#### Issue 4: Slow Performance
*   **Cause:** Network bottleneck or mixing HDD/SSD in the same PG without proper rules.
*   **Fix:** Check network speed (`ethtool`) and ensure your client has 1Gbps or 10Gbps connectivity to the cluster. Verify `ceph -w` for slow request warnings.

---

### Summary Checklist for Daily Operations

| Task | Command |
| :--- | :--- |
| **Check Connection** | `ceph -s` |
| **List Images** | `rbd ls -p <pool>` |
| **Map Image** | `sudo rbd map -p <pool> <image>` |
| **Show Mapped** | `rbd showmapped` |
| **Format (New)** | `sudo mkfs.ext4 /dev/rbdX` |
| **Mount** | `sudo mount /dev/rbdX /mnt/point` |
| **Unmount** | `sudo umount /mnt/point` |
| **Unmap** | `sudo rbd unmap /dev/rbdX` |

You now have a fully functional Ceph client capable of mounting and managing RBD images from your 3-node cluster. Treat `/dev/rbdX` like any other physical hard drive, but remember it is backed by the resilience of your distributed cluster.

---

## 7. RBD Snapshot and Clone (Backup & Testing)

### 7.1 Create a Snapshot

```bash
# Create a snapshot of the current state
ceph --name client.rbd-user --keyring /etc/ceph/ceph.client.rbd-user.keyring \
  rbd snap create rbd-pool/vm-disk-1@snapshot-1

# List snapshots
ceph --name client.rbd-user --keyring /etc/ceph/ceph.client.rbd-user.keyring \
  rbd snap ls rbd-pool/vm-disk-1

# Expected output:
# SNAPID  NAME         SIZE
# 4       snapshot-1   50 GiB
```

### 7.2 Protect Snapshot (Required for Cloning)

```bash
# Protect the snapshot
ceph --name client.rbd-user --keyring /etc/ceph/ceph.client.rbd-user.keyring \
  rbd snap protect rbd-pool/vm-disk-1@snapshot-1

# Verify protection
ceph --name client.rbd-user --keyring /etc/ceph/ceph.client.rbd-user.keyring \
  rbd snap ls rbd-pool/vm-disk-1
# Will show 'p' flag for protected
```

### 7.3 Create a Clone from Snapshot

```bash
# Create a clone (new independent image from snapshot)
ceph --name client.rbd-user --keyring /etc/ceph/ceph.client.rbd-user.keyring \
  rbd clone rbd-pool/vm-disk-1@snapshot-1 rbd-pool/vm-disk-1-clone

# Verify clone was created
ceph --name client.rbd-user --keyring /etc/ceph/ceph.client.rbd-user.keyring \
  rbd ls rbd-pool

# Expected: vm-disk-1, vm-disk-1-clone
```

### 7.4 Rollback to Snapshot (If Needed)

```bash
# If something goes wrong, rollback to snapshot
ceph --name client.rbd-user --keyring /etc/ceph/ceph.client.rbd-user.keyring \
  rbd snap rollback rbd-pool/vm-disk-1@snapshot-1

# This restores the image to the state when snapshot was taken
```

---

## 8. RBD Performance Testing

### 8.1 Install FIO Benchmark Tool

```bash
# Install fio for performance testing
sudo apt install -y fio
```

### 8.2 Run Sequential Write Test

```bash
# Test sequential write performance
sudo fio --name=seq-write \
  --ioengine=libaio \
  --iodepth=32 \
  --rw=write \
  --bs=1M \
  --direct=1 \
  --size=5G \
  --numjobs=4 \
  --filename=/mnt/ceph-storage/fio-test \
  --group_reporting \
  --runtime=60 \
  --time_based
```

**Look for these results:**
- `write: IOPS=XXXX` (Input/Output Operations Per Second)
- `write: BW=XXXX MiB/s` (Bandwidth)

### 8.3 Run Random Read Test

```bash
# Test random read performance (important for databases)
sudo fio --name=rand-read \
  --ioengine=libaio \
  --iodepth=32 \
  --rw=randread \
  --bs=4k \
  --direct=1 \
  --size=5G \
  --numjobs=4 \
  --filename=/mnt/ceph-storage/fio-test \
  --group_reporting \
  --runtime=60 \
  --time_based
```

### 8.4 Record Baseline Performance

```bash
# Save results for future comparison
sudo fio --name=baseline-test \
  --ioengine=libaio \
  --iodepth=32 \
  --rw=randrw \
  --bs=4k \
  --direct=1 \
  --size=5G \
  --numjobs=4 \
  --filename=/mnt/ceph-storage/fio-test \
  --group_reporting \
  --runtime=60 \
  --time_based \
  --output=/root/rbd-performance-baseline.txt
```

---

## 9. Unmap RBD Image (When Done)

```bash
# Unmount first
sudo umount /mnt/ceph-storage

# Unmap the RBD device
sudo rbd unmap /dev/rbd0

# Verify unmapped
rbd showmapped

# Should show no mappings
```

---

# PART 3: Connect RGW & CephFS Services

## 🎯 What We Will Do
> Connect the same Linux client to Ceph Object Storage (RGW/S3) and Ceph File System (CephFS).

---

## SECTION A: RGW / S3 Object Storage Access

### Step 1: What is RGW?

> **RGW (RADOS Gateway)** provides S3-compatible object storage API. You access it via HTTP/HTTPS, not by mounting like a disk.

**Real-World Use Cases:**
- Store user uploaded files (photos, documents)
- Backup archives with lifecycle policies
- Static website hosting
- Application data that needs HTTP access

### Step 2: Get RGW Endpoint Information

**On Ceph Admin Node, run:**

```bash
# Check RGW service status
cephadm shell -- ceph orch ps --daemon_type rgw

# Get RGW endpoint
cephadm shell -- ceph orch ls --service_name=rgw

# Note the endpoint URL, typically: http://192.168.10.11:8080
```

### Step 3: Create RGW User (On Ceph Admin Node)

```bash
# Create an S3 user for object storage access
cephadm shell -- radosgw-admin user create \
  --uid="client-app" \
  --display-name="Client Application User" \
  --access-key=MYACCESSKEY123 \
  --secret=MYSECRETKEY456789

# Save the access-key and secret-key securely!
```

### Step 4: Install AWS CLI on Client (For S3 Access)

```bash
# On your CLIENT machine

# Install AWS CLI
sudo apt install -y awscli

# Verify installation
aws --version
```

### Step 5: Configure AWS CLI for Ceph RGW

```bash
# Create AWS CLI configuration directory
mkdir -p ~/.aws

# Create config file
nano ~/.aws/config
```

**Add this content:**

```ini
[default]
region = default
output = json
```

```bash
# Create credentials file
nano ~/.aws/credentials
```

**Add this content:**

```ini
[default]
aws_access_key_id = MYACCESSKEY123
aws_secret_access_key = MYSECRETKEY456789
```

```bash
# Set secure permissions
chmod 600 ~/.aws/credentials
```

### Step 6: Test RGW Connection

```bash
# Create a bucket
aws --endpoint-url=http://192.168.10.11:8080 \
  s3api create-bucket \
  --bucket my-first-bucket

# List buckets
aws --endpoint-url=http://192.168.10.11:8080 \
  s3api list-buckets

# Expected output should show 'my-first-bucket'
```

### Step 7: Upload and Download Files

```bash
# Create a test file
echo "Hello RGW Object Storage!" > test-file.txt

# Upload to bucket
aws --endpoint-url=http://192.168.10.11:8080 \
  s3 cp test-file.txt \
  s3://my-first-bucket/test-file.txt

# List objects in bucket
aws --endpoint-url=http://192.168.10.11:8080 \
  s3 ls s3://my-first-bucket/

# Download the file
aws --endpoint-url=http://192.168.10.11:8080 \
  s3 cp s3://my-first-bucket/test-file.txt \
  downloaded-file.txt

# Verify content
cat downloaded-file.txt
```

### Step 8: Create Python Script for S3 Access

```bash
# Install boto3 (Python S3 library)
pip3 install boto3

# Create Python script
nano /root/s3-example.py
```

**Add this content:**

```python
#!/usr/bin/env python3
import boto3
from botocore.client import Config

# Configure connection to Ceph RGW
s3 = boto3.client(
    's3',
    endpoint_url='http://192.168.10.11:8080',
    aws_access_key_id='MYACCESSKEY123',
    aws_secret_access_key='MYSECRETKEY456789',
    config=Config(signature_version='s3v4'),
    region_name='default'
)

# Upload a file
print("Uploading file...")
s3.upload_file('test-file.txt', 'my-first-bucket', 'uploads/test.txt')
print("Upload complete!")

# List objects
print("\nObjects in bucket:")
response = s3.list_objects_v2(Bucket='my-first-bucket')
for obj in response.get('Contents', []):
    print(f"  - {obj['Key']} ({obj['Size']} bytes)")

# Generate pre-signed URL (temporary access link)
url = s3.generate_presigned_url(
    'get_object',
    Params={'Bucket': 'my-first-bucket', 'Key': 'uploads/test.txt'},
    ExpiresIn=3600
)
print(f"\nShareable URL (valid 1 hour): {url}")
```

```bash
# Run the script
python3 /root/s3-example.py
```

---

## SECTION B: CephFS File Storage Access

### Step 1: What is CephFS?

> **CephFS** is a POSIX-compliant file system that runs on top of Ceph. You mount it like NFS, but it's distributed and highly available.

**Real-World Use Cases:**
- Shared home directories for users
- Application data that needs file paths (not block)
- Multiple servers accessing same files simultaneously
- Legacy applications that don't support object storage

### Step 2: Verify CephFS is Deployed (On Ceph Admin Node)

```bash
# Check if CephFS exists
cephadm shell -- ceph fs ls

# If no filesystem exists, create one:
cephadm shell -- ceph osd pool create cephfs_metadata 64 64
cephadm shell -- ceph osd pool create cephfs_data 128 128
cephadm shell -- ceph fs new myfs cephfs_metadata cephfs_data

# Deploy MDS daemons
cephadm shell -- ceph orch apply mds myfs --placement="2 ceph-node1 ceph-node2"

# Check status
cephadm shell -- ceph fs status
```

### Step 3: Install CephFS Client Tools (On Client Machine)

```bash
# Install Ceph FUSE client
sudo apt install -y ceph-fuse

# Verify installation
ceph-fuse --version
```

### Step 4: Create CephFS Mount Point

```bash
# Create directory for mounting
sudo mkdir -p /mnt/cephfs

# Set permissions
sudo chmod 755 /mnt/cephfs
```

### Step 5: Mount CephFS Using FUSE

```bash
# Mount CephFS (user-space mount, easier than kernel)
sudo ceph-fuse -m 192.168.10.11:6789 /mnt/cephfs \
  --name client.cephfs-user \
  --keyring /etc/ceph/ceph.client.cephfs-user.keyring

# Verify mount
df -h /mnt/cephfs

# Expected output:
# Filesystem      Size  Used Avail Use% Mounted on
# ceph-fuse       100G   10G   90G  10% /mnt/cephfs
```

### Step 6: Test CephFS Read/Write

```bash
# Create test files
echo "Hello CephFS!" | sudo tee /mnt/cephfs/test-cephfs.txt

# Create directory structure
sudo mkdir -p /mnt/cephfs/projects/webapp
sudo mkdir -p /mnt/cephfs/projects/database

# Create multiple files
for i in {1..10}; do
  echo "File $i content" | sudo tee /mnt/cephfs/projects/webapp/file-$i.txt
done

# List files
ls -la /mnt/cephfs/projects/webapp/

# Check disk usage
du -sh /mnt/cephfs/*
```

### Step 7: Make CephFS Mount Persistent

```bash
# Backup fstab
sudo cp /etc/fstab /etc/fstab.backup

# Add CephFS mount entry
echo "ceph-fuse#/mnt/cephfs fuse _netdev,name=client.cephfs-user,secretfile=/etc/ceph/ceph.client.cephfs-user.keyring 0 0" | sudo tee -a /etc/fstab

# Verify entry
tail -3 /etc/fstab

# Test persistent mount
sudo umount /mnt/cephfs
sudo mount /mnt/cephfs
df -h /mnt/cephfs
```

### Step 8: CephFS Quotas (Limit Storage per Directory)

```bash
# Set quota on a directory (10 GB max)
ceph --name client.admin --keyring /etc/ceph/ceph.client.admin.keyring \
  fs quota set myfs /projects --max-bytes 10737418240

# Set file count quota (max 1000 files)
ceph --name client.admin --keyring /etc/ceph/ceph.client.admin.keyring \
  fs quota set myfs /projects --max-files 1000

# View quotas
ceph --name client.admin --keyring /etc/ceph/ceph.client.admin.keyring \
  fs quota ls myfs
```

---

## SECTION C: Verify All Services Connected

### Complete Connection Test Script

```bash
#!/bin/bash
# save as /root/test-ceph-connection.sh

echo "=========================================="
echo "Ceph Client Connection Test"
echo "=========================================="

echo -e "\n[1] Testing Cluster Connection..."
ceph --name client.rbd-user --keyring /etc/ceph/ceph.client.rbd-user.keyring -s | head -10

echo -e "\n[2] Testing RBD Storage..."
rbd showmapped
if [ -d /mnt/ceph-storage ]; then
  echo "RBD mount point exists: /mnt/ceph-storage"
  ls /mnt/ceph-storage/ | head -5
fi

echo -e "\n[3] Testing RGW/S3 Storage..."
aws --endpoint-url=http://192.168.10.11:8080 s3api list-buckets 2>/dev/null | grep -o '"Name": "[^"]*"' | head -5

echo -e "\n[4] Testing CephFS Storage..."
if [ -d /mnt/cephfs ]; then
  echo "CephFS mount point exists: /mnt/cephfs"
  df -h /mnt/cephfs
  ls /mnt/cephfs/ | head -5
fi

echo -e "\n=========================================="
echo "Test Complete!"
echo "=========================================="
```

```bash
# Make executable and run
chmod +x /root/test-ceph-connection.sh
/root/test-ceph-connection.sh
```

---

## ✅ Part 3 Completion Checklist

```bash
# Run these to verify all services:

# □ RGW/S3 accessible
aws --endpoint-url=http://192.168.10.11:8080 s3api list-buckets

# □ CephFS mounted
df -h /mnt/cephfs

# □ CephFS read/write works
ls /mnt/cephfs/

# □ RBD still accessible
df -h /mnt/ceph-storage

# □ All keyrings secure
ls -la /etc/ceph/ceph.client.*.keyring

# □ All mounts persistent
cat /etc/fstab | grep -E "(rbd|cephfs)"
```

**If all checks pass, Part 3 is complete!** 🎉

---

# 🎯 Final Summary: Your Complete Ceph Client Setup

## What You Have Now

| Service | Mount Point / Access | Use Case |
|---------|---------------------|----------|
| **RBD** | `/dev/rbd0` → `/mnt/ceph-storage` | VM disks, databases, block storage |
| **RGW/S3** | `http://192.168.10.11:8080` | Object storage, backups, web files |
| **CephFS** | `/mnt/cephfs` | Shared files, home directories, apps |

## Quick Command Reference

```bash
# ===== RBD Commands =====
rbd create <image> --size <GB> --pool <pool>     # Create image
rbd map <image> --pool <pool>                    # Map to client
rbd unmap /dev/rbd0                              # Unmap
rbd snap create <pool>/<image>@<snap>           # Create snapshot

# ===== RGW/S3 Commands =====
aws --endpoint-url=<url> s3api create-bucket --bucket <name>  # Create bucket
aws --endpoint-url=<url> s3 cp <file> s3://<bucket>/          # Upload file
aws --endpoint-url=<url> s3 ls s3://<bucket>/                 # List files

# ===== CephFS Commands =====
ceph-fuse -m <mon-ip>:6789 <mount-point>         # Mount CephFS
ceph fs quota set <fs> <path> --max-bytes <size> # Set quota
```

---

## 🔐 Security Best Practices

```bash
# 1. Never use admin keyring on production clients
# 2. Set keyring permissions to 600
chmod 600 /etc/ceph/ceph.client.*.keyring

# 3. Use HTTPS for RGW in production
# 4. Restrict firewall access to client IPs only
# 5. Rotate access keys periodically
```

---

## 📊 Monitoring Your Client

```bash
# Monitor RBD performance
iostat -x 5

# Monitor CephFS activity
watch -n 5 'df -h /mnt/cephfs'

# Monitor network to cluster
iftop -P -n -f "host 192.168.10.11"

# Check Ceph cluster health from client
watch -n 10 'ceph --name client.rbd-user --keyring /etc/ceph/ceph.client.rbd-user.keyring -s'
```

---

> 🎉 **Congratulations!** You now have a fully functional Ceph client machine connected to all major Ceph storage services (RBD, RGW, CephFS) with production-ready configuration!

> 💡 **Next Steps:** Consider setting up automated monitoring, backup schedules, and alerting for your client machines.

*Need help with any specific step? Feel free to ask!* 🐘✨