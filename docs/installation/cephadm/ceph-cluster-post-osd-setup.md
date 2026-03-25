# 🚀 Ceph Cluster Post-OSD Setup: Complete Guide (Simple English)
**After OSD Deployment | Production-Ready Configuration | Real-World Examples**

---

## 📋 Table of Contents
1. [Verify OSD Deployment](#1-verify-osd-deployment)
2. [Client Machine Preparation](#2-client-machine-preparation)
3. [Storage Pools: What, Why & How](#3-storage-pools-what-why--how)
4. [RGW / Object Storage Setup](#4-rgw--object-storage-setup)
5. [CephFS / File Storage Setup](#5-cephfs--file-storage-setup)
6. [Client Access Configuration](#6-client-access-configuration)
7. [Cluster Testing & Production Readiness](#7-cluster-testing--production-readiness)

---

## 1. Verify OSD Deployment

Before moving forward, confirm your OSDs are healthy.

#### Check cluster health
``` 
sudo cephadm shell -- ceph -s
```
#### Expected output should show:
```
#### - health: HEALTH_OK (or HEALTH_WARN if still balancing)
#### - osd: X osds: X up, X in
```
#### Check OSD tree
```
sudo cephadm shell -- ceph osd tree
```
#### Example output:
```
# ID  CLASS  WEIGHT   TYPE NAME       STATUS  REWEIGHT  PRI-AFF
# -1         0.58800  root default
# -3         0.19600      host ceph-node1
#  0    hdd  0.09800          osd.0      up   1.00000  1.00000
#  1    hdd  0.09800          osd.1      up   1.00000  1.00000
```
#### Check OSD status details
```
sudo cephadm shell -- ceph osd stat
sudo cephadm shell -- ceph osd df
```
#### Wait until all PGs show "active+clean"
```
sudo cephadm shell -- ceph pg stat
```

✅ **Success Criteria:**
- All OSDs show `up` and `in`
- No OSDs in `down` or `out` state
- PGs (Placement Groups) show `active+clean`
- Cluster health is `HEALTH_OK`

---

## 2. Client Machine Preparation

### 2.1 What is a Ceph Client?

> A **Ceph Client** is any machine that needs to read/write data to your Ceph cluster. This could be:
> - A virtual machine using Ceph RBD disks
> - An application using CephFS for file storage
> - A backup server using S3 API for object storage

### 2.2 Client Machine Requirements

| Requirement | Details | Why It Matters |
|------------|---------|---------------|
| **OS** | Ubuntu 24.04.4 LTS (recommended) | Same OS as cluster = fewer compatibility issues |
| **Network** | Can reach Ceph public network (192.168.10.x) | Client must talk to MONs and OSDs |
| **Ceph Tools** | `ceph-common` package | Provides `rbd`, `cephfs`, and config tools |
| **Time Sync** | Chrony/NTP configured | Time mismatch can cause auth failures |
| **Storage** | Depends on use case (RBD/CephFS/S3) | Plan disk space based on your needs |

### 2.3 Step-by-Step Client Setup (Ubuntu Client Example)


#### On your CLIENT machine (not Ceph cluster nodes)

#### 1. Update system
```
sudo apt update && sudo apt upgrade -y
```
#### 2. Install Ceph client tools
```
sudo apt install -y ceph-common
```
#### 3. Copy ceph.conf & keyring from Ceph Bootstrap node
#### Run From Client:
```
scp root@ceph-node1:/etc/ceph/ceph.conf /etc/ceph/
scp root@ceph-node1:/etc/ceph/ceph.client.admin.keyring /etc/ceph/
```
#### Given right permission
```
sudo chmod 644 /etc/ceph/ceph.conf
sudo chmod 600 /etc/ceph/ceph.client.admin.keyring
```
#### Then on client:
```
sudo mv /tmp/ceph.conf /etc/ceph/ceph.conf
sudo chmod 644 /etc/ceph/ceph.conf
```

---

## 3. Storage Pools: What, Why & How

### 3.1 What is a Storage Pool?

> A **Pool** is a logical partition of your Ceph cluster where you store data. Think of it like a "folder" or "volume" in traditional storage, but with Ceph's distributed features.

**Real-World Analogy:**
> Imagine a large warehouse (your Ceph cluster). You divide it into sections:
> - Section A: For fragile items (high replication)
> - Section B: For bulk items (erasure coding for space saving)
> - Section C: For temporary items (lower replication)
> 
> Each section is a **Pool** with its own rules.

### 3.2 Why Do We Need Pools?

| Reason | Explanation | Example |
|--------|-------------|---------|
| **Isolation** | Different apps/data types don't interfere | Database pool vs Backup pool |
| **Performance Tuning** | Each pool can have different PG numbers, replication settings | High-IOPS pool for databases, high-capacity pool for archives |
| **Security** | Different access permissions per pool | Dev team can't access production pool |
| **Data Protection** | Choose replication or erasure coding per pool | Critical data: 3x replication, Logs: erasure coding |

### 3.3 Key Pool Concepts You Must Know

| Concept | What It Is | Recommended Value |
|---------|-----------|------------------|
| **Replication Size** | How many copies of each object | `size=3` for production (tolerates 2 failures) |
| **Min Size** | Minimum copies needed for write to succeed | `min_size=2` (allows writes if 1 OSD down) |
| **PG Number** | Placement Groups count for data distribution | Start with 32-128, scale based on OSD count |
| **PGP Number** | PG placement number (usually same as PG) | Same as `pg_num` |
| **Crush Rule** | Where data is placed physically | Default rule works for most cases |

### 3.4 How to Calculate PG Number

**Formula:**
```
Total PGs = (OSD_Count × 100) / Replication_Size
```

**Example Calculation:**
```
- You have: 3 nodes × 2 OSDs each = 6 OSDs
- Replication: 3 copies
- Calculation: (6 × 100) / 3 = 200
- Round to nearest power of 2: 256

✅ Use pg_num = 256, pgp_num = 256
```

**Quick Reference Table:**

| OSD Count | Replication | Suggested pg_num |
|-----------|------------|-----------------|
| 1-10 | 3 | 32-128 |
| 10-50 | 3 | 256-512 |
| 50-100 | 3 | 1024-2048 |
| Any | 2 | Double the above |

### 3.5 Create a Storage Pool (Step-by-Step)


#### On Ceph Admin Node (Node1)

#### 1. Create a replicated pool for RBD (block storage)
```
sudo cephadm shell -- ceph osd pool create rbd-pool 128 128 replicated
```
#### Format: pool_name pg_num pgp_num [replicated|erasure]

#### 2. Set replication size (default is 3, but be explicit)
```
sudo cephadm shell -- ceph osd pool set rbd-pool size 3
sudo cephadm shell -- ceph osd pool set rbd-pool min_size 2
```
#### 3. Initialize the pool for RBD usage
```
sudo cephadm shell -- rbd pool init rbd-pool
```
#### 4. Verify pool creation
```
sudo cephadm shell -- ceph osd pool ls detail
sudo cephadm shell -- ceph osd pool stats rbd-pool
```
#### 5. Create a client user with access to this pool only
```
sudo cephadm shell -- ceph auth get-or-create client.vm-user mon 'allow r' osd 'allow rwx pool=rbd-pool' -o /etc/ceph/ceph.client.vm-user.keyring
```

### 3.6 Real-World Pool Examples

#### Example 1: Database Pool (High Performance)

#### Create pool with more PGs for better distribution
```
sudo cephadm shell -- ceph osd pool create db-pool 256 256 replicated
sudo cephadm shell -- ceph osd pool set db-pool size 3
sudo cephadm shell -- ceph osd pool set db-pool min_size 2
sudo cephadm shell -- rbd pool init db-pool
```
#### Optional: Enable compression for space saving
```
sudo cephadm shell -- ceph osd pool set db-pool compression_algorithm zstd
sudo cephadm shell -- ceph osd pool set db-pool compression_mode passive
```

#### Example 2: Backup Pool (Space Efficient)

#### Use erasure coding for 50% space savings (more complex, slower writes)
```
sudo cephadm shell -- ceph osd erasure-code-profile set backup-profile k=2 m=1
sudo cephadm shell -- ceph osd pool create backup-pool 128 128 erasure backup-profile
sudo cephadm shell -- ceph osd pool set backup-pool min_size 2
```

#### Example 3: Temporary/Cache Pool

#### Lower replication for non-critical data
```
sudo cephadm shell -- ceph osd pool create cache-pool 64 64 replicated
sudo cephadm shell -- ceph osd pool set cache-pool size 2
sudo cephadm shell -- ceph osd pool set cache-pool min_size 1
sudo cephadm shell -- rbd pool init cache-pool
```

---

## 4. RGW / Object Storage Setup

### 4.1 What is Object Storage?

> **Object Storage** stores data as "objects" (files + metadata + unique ID) instead of blocks or files in folders. It's accessed via HTTP/S API (S3-compatible).

**Real-World Analogy:**
> Think of Amazon S3 or Google Cloud Storage. You upload a file, get a unique URL, and anyone with permission can download it via web. No mounting, no drive letters.

### 4.2 Why Use RGW (Ceph Object Gateway)?

| Use Case | Why RGW? | Example |
|----------|----------|---------|
| **Web Applications** | S3 API is standard for cloud apps | Upload user photos to your app |
| **Backup & Archive** | Cheap, scalable, HTTP access | Store database backups with lifecycle policies |
| **Static Content** | Direct HTTP delivery | Serve website images, videos, downloads |
| **Multi-Tenancy** | Isolated buckets per customer | SaaS platform with separate storage per client |

### 4.3 RGW Architecture Overview

```
┌─────────────────┐
│   Client App    │
│  (S3 API Call)  │
└────────┬────────┘
         │ HTTPS
┌────────▼────────┐
│   RGW Daemon    │
│  (Runs in Docker│
│   via cephadm)  │
└────────┬────────┘
         │
┌────────▼────────┐
│  Ceph Pool      │
│  (.rgw.buckets) │
│  (Stores objects│
│   as RADOS objs)│
└─────────────────┘
```

### 4.4 Step-by-Step RGW Deployment


#### On Ceph Admin Node (Node1)

#### 1. Create a dedicated pool for RGW (optional but recommended)
```
sudo cephadm shell -- ceph osd pool create .rgw.buckets 128 128 replicated
sudo cephadm shell -- ceph osd pool set .rgw.buckets size 3
sudo cephadm shell -- ceph osd pool set .rgw.buckets min_size 2
```
#### 2. Deploy RGW daemon via cephadm orchestrator
```
sudo cephadm shell -- ceph orch apply rgw my-realm my-zone --port=8080 --placement="3 ceph-node1 ceph-node2 ceph-node3"
```
#### 3. Wait for RGW to start (check status)
```
sudo cephadm shell -- ceph orch ps --daemon_type rgw
```
#### Should show 3 rgw daemons (one per node) running

#### 4. Create an RGW admin user (for managing buckets/users)
```
sudo cephadm shell -- radosgw-admin user create --uid="admin" --display-name="RGW Admin" --access-key=RGWADMINKEY123 --secret=RGWADMINSECRET456 --system
```

#### 5. Create a regular user for applications
```
sudo cephadm shell -- radosgw-admin user create --uid="app-user" --display-name="Application User" --access-key=APPACCESSKEY789 --secret=APPSECRET012
```
#### 6. Get RGW endpoint URL
```
sudo cephadm shell -- ceph orch ls --service_name=rgw.my-realm.my-zone
```
> Note the endpoint, usually: http://<node-ip>:8080


### 4.5 Real-World RGW Example: Photo Upload Service

**Scenario:** You're building a web app where users upload profile pictures.


#### 1. Create a bucket for user photos
#### Using AWS CLI (install with: sudo apt install awscli)
```
aws --endpoint-url=http://192.168.10.11:8080 \
    s3api create-bucket \
    --bucket user-photos \
    --access-key APPACCESSKEY789 \
    --secret-key APPSECRET012
```
#### 2. Upload a file
```
aws --endpoint-url=http://192.168.10.11:8080 \
    s3 cp profile.jpg \
    s3://user-photos/users/user123/profile.jpg \
    --access-key APPACCESSKEY789 \
    --secret-key APPSECRET012
```
#### 3. Make file publicly readable (optional)
```
aws --endpoint-url=http://192.168.10.11:8080 \
    s3api put-object-acl \
    --bucket user-photos \
    --key users/user123/profile.jpg \
    --acl public-read \
    --access-key APPACCESSKEY789 \
    --secret-key APPSECRET012
```
#### 4. Access via URL:
#### http://192.168.10.11:8080/user-photos/users/user123/profile.jpg


### 4.6 RGW Best Practices


#### Enable SSL/TLS for production (generate cert or use Let's Encrypt)
```
sudo cephadm shell -- ceph orch apply rgw my-realm my-zone --port=443 --ssl-cert=/path/to/cert.pem --ssl-key=/path/to/key.pem
```
#### Configure bucket lifecycle (auto-delete old backups)
```
cat > lifecycle.json << 'EOF'
{
  "Rules": [{
    "ID": "Delete old backups",
    "Status": "Enabled",
    "Prefix": "backups/",
    "Expiration": {"Days": 30}
  }]
}
EOF
```
```
aws` --endpoint-url=http://192.168.10.11:8080 \
    s3api put-bucket-lifecycle-configuration \
    --bucket backups \
    --lifecycle-configuration file://lifecycle.json \
    --access-key APPACCESSKEY789 \
    --secret-key APPSECRET012
```

---

## 5. CephFS / File Storage Setup

### 5.1 What is CephFS?

> **CephFS** is a POSIX-compliant file system that runs on top of Ceph. It lets you mount Ceph storage like a normal folder (`/mnt/ceph`) on Linux.

**Real-World Analogy:**
> Like NFS or SMB shares, but distributed and highly available. Multiple servers can read/write the same files simultaneously.

### 5.2 Why Use CephFS?

| Use Case | Why CephFS? | Example |
|----------|-------------|---------|
| **Shared Home Directories** | Users access same files from any server | University lab: students log into any PC, see their files |
| **Application Data** | Apps need file-based access (not block) | WordPress media library, Docker volumes |
| **HPC/Scientific Computing** | Parallel read/write from many nodes | Research cluster processing large datasets |
| **Legacy App Migration** | Apps that only understand file paths | Old ERP system moving to cloud |

### 5.3 CephFS Architecture

```
┌─────────────────┐
│   Client Linux  │
│   mount -t ceph │
└────────┬────────┘
         │
┌────────▼────────┐
│  MDS Daemons    │
│  (Metadata      │
│   Servers)      │
│  Handle file    │
│  names, perms   │
└────────┬────────┘
         │
┌────────▼────────┐
│  OSDs           │
│  Store actual   │
│  file data      │
└─────────────────┘
```

> ⚠️ **Important:** CephFS requires **MDS (Metadata Server)** daemons. These are separate from MON/MGR/OSD.

### 5.4 Step-by-Step CephFS Deployment

```bash
# On Ceph Admin Node (Node1)

# 1. Create pools for CephFS (metadata + data)
sudo cephadm shell -- ceph osd pool create cephfs_metadata 64 64 replicated
sudo cephadm shell -- ceph osd pool set cephfs_metadata size 3
sudo cephadm shell -- ceph osd pool create cephfs_data 128 128 replicated
sudo cephadm shell -- ceph osd pool set cephfs_data size 3

# 2. Create the CephFS file system
sudo cephadm shell -- ceph fs new myfs cephfs_metadata cephfs_data

# 3. Deploy MDS daemons (at least 1, recommend 2 for HA)
sudo cephadm shell -- ceph orch apply mds myfs --placement="2 ceph-node1 ceph-node2"

# 4. Wait for MDS to start
sudo cephadm shell -- ceph fs status
# Should show: myfs-0 active, myfs-1 standby

# 5. Create a client user for CephFS access
sudo cephadm shell -- ceph auth get-or-create client.cephfs-user mon 'allow r' mds 'allow r, allow rw path=/' osd 'allow rw tag cephfs metadata=cephfs_metadata, allow rw tag cephfs data=cephfs_data' -o /etc/ceph/ceph.client.cephfs-user.keyring
```

### 5.5 Mount CephFS on Client (Step-by-Step)

```bash
# On your CLIENT machine

# 1. Install Ceph FS tools (if not already installed)
sudo apt install -y ceph-fuse

# 2. Create mount point
sudo mkdir -p /mnt/cephfs

# 3. Mount using ceph-fuse (user-space, easier)
sudo ceph-fuse -m 192.168.10.11:6789 /mnt/cephfs \
  --name client.cephfs-user \
  --keyring /etc/ceph/ceph.client.cephfs-user.keyring

# 4. Verify mount
df -h /mnt/cephfs
mount | grep ceph

# 5. Test read/write
echo "Hello CephFS" | sudo tee /mnt/cephfs/test.txt
cat /mnt/cephfs/test.txt

# 6. Make mount persistent (add to /etc/fstab)
echo "ceph-fuse#/mnt/cephfs fuse _netdev,name=client.cephfs-user,secretfile=/etc/ceph/ceph.client.cephfs-user.keyring 0 0" | sudo tee -a /etc/fstab
```

### 5.6 Real-World CephFS Example: Shared Web Server

**Scenario:** 3 web servers need to serve the same WordPress files.

```bash
# On each web server (web1, web2, web3):

# 1. Mount CephFS to WordPress directory
sudo mkdir -p /var/www/html
sudo ceph-fuse -m 192.168.10.11:6789 /var/www/html \
  --name client.webserver \
  --keyring /etc/ceph/ceph.client.webserver.keyring

# 2. Set proper permissions
sudo chown -R www-data:www-data /var/www/html

# 3. Install WordPress (only on one server, others see same files)
# On web1 only:
sudo -u www-data wp core download --path=/var/www/html
sudo -u www-data wp config create --dbname=wordpress --dbuser=wpuser --dbpass=secret --path=/var/www/html

# 4. All 3 web servers now serve identical WordPress files
# Changes on web1 instantly visible on web2 and web3
```

### 5.7 CephFS Best Practices

```bash
# Enable quotas to prevent one user filling all storage
sudo cephadm shell -- ceph fs quota set myfs /users --max-files 10000 --max-bytes 10737418240  # 10GB

# Monitor MDS performance
sudo cephadm shell -- ceph fs dump
sudo cephadm shell -- ceph daemon mds.ceph-node1 status

# Backup metadata regularly (critical for recovery)
sudo cephadm shell -- ceph fs snapshot metadata myfs backup-$(date +%F)
```

---

## 6. Client Access Configuration

### 6.1 RBD Client Access (Block Storage)

#### What is RBD?
> **RBD (RADOS Block Device)** provides block-level storage that you can attach to VMs or containers like a virtual hard disk.

#### Step-by-Step: Create & Attach RBD Image

```bash
# On Ceph Admin Node (Node1)

# 1. Create an RBD image (virtual disk)
sudo cephadm shell -- rbd create vm-disk-1 --size 50G --pool rbd-pool --image-format 2

# 2. Enable features for better performance
sudo cephadm shell -- rbd feature enable vm-disk-1 exclusive-lock object-map fast-diff

# 3. Map the RBD image to a client (using kernel module)
# On CLIENT machine:
sudo rbd map vm-disk-1 --pool rbd-pool --name client.vm-user
# Output: /dev/rbd0

# 4. Format and mount the disk
sudo mkfs.ext4 /dev/rbd0
sudo mkdir -p /mnt/rbd-storage
sudo mount /dev/rbd0 /mnt/rbd-storage

# 5. Make it persistent (optional)
# Add to /etc/fstab:
echo "/dev/rbd0 /mnt/rbd-storage ext4 _netdev 0 0" | sudo tee -a /etc/fstab

# 6. Unmap when done
sudo umount /mnt/rbd-storage
sudo rbd unmap /dev/rbd0
```

#### Real-World Example: KVM Virtual Machine with RBD

```bash
# 1. Create RBD image for VM
sudo cephadm shell -- rbd create vm-prod-db --size 100G --pool db-pool

# 2. On KVM host, define VM with RBD disk
cat > /tmp/vm-prod-db.xml << 'EOF'
<domain type='kvm'>
  <name>vm-prod-db</name>
  <memory unit='GiB'>8</memory>
  <devices>
    <disk type='network' device='disk'>
      <driver name='qemu' type='raw'/>
      <source protocol='rbd' name='db-pool/vm-prod-db'>
        <host name='192.168.10.11' port='6789'/>
        <host name='192.168.10.12' port='6789'/>
        <host name='192.168.10.13' port='6789'/>
        <auth username='vm-user' key='BASE64-KEY-HERE'/>
      </source>
      <target dev='vda' bus='virtio'/>
    </disk>
  </devices>
</domain>
EOF

# 3. Get client key in base64
cephadm shell -- ceph auth get-key client.vm-user | base64

# 4. Start VM
virsh define /tmp/vm-prod-db.xml
virsh start vm-prod-db
```

---

### 6.2 S3 Client Access (RGW / Object Storage)

#### Step-by-Step: Access RGW from Application

```bash
# Using Python boto3 library (most common for S3)

# 1. Install boto3 on client
pip install boto3

# 2. Python script to upload/download
cat > s3-example.py << 'EOF'
import boto3
from botocore.client import Config

# Configure connection to Ceph RGW
s3 = boto3.client(
    's3',
    endpoint_url='http://192.168.10.11:8080',  # Your RGW endpoint
    aws_access_key_id='APPACCESSKEY789',
    aws_secret_access_key='APPSECRET012',
    config=Config(signature_version='s3v4'),
    region_name='default'
)

# Upload a file
s3.upload_file('local-file.txt', 'user-photos', 'uploads/file.txt')
print("Upload complete!")

# List objects in bucket
response = s3.list_objects_v2(Bucket='user-photos')
for obj in response.get('Contents', []):
    print(f"  - {obj['Key']} ({obj['Size']} bytes)")

# Download a file
s3.download_file('user-photos', 'uploads/file.txt', 'downloaded-file.txt')
print("Download complete!")

# Generate pre-signed URL for temporary access
url = s3.generate_presigned_url(
    'get_object',
    Params={'Bucket': 'user-photos', 'Key': 'uploads/file.txt'},
    ExpiresIn=3600  # URL valid for 1 hour
)
print(f"Share this URL: {url}")
EOF

# 3. Run the script
python3 s3-example.py
```

#### Real-World Example: Backup Script Using S3

```bash
#!/bin/bash
# backup-to-ceph.sh - Daily backup to Ceph RGW

# Configuration
ENDPOINT="http://192.168.10.11:8080"
ACCESS_KEY="APPACCESSKEY789"
SECRET_KEY="APPSECRET012"
BUCKET="server-backups"
SOURCE_DIR="/etc /var/www /home"
DATE=$(date +%F)

# Install AWS CLI if needed
# sudo apt install -y awscli

# Create backup archive
tar -czf /tmp/backup-${DATE}.tar.gz ${SOURCE_DIR}

# Upload to Ceph RGW
aws --endpoint-url=${ENDPOINT} \
    s3 cp /tmp/backup-${DATE}.tar.gz \
    s3://${BUCKET}/daily/backup-${DATE}.tar.gz \
    --access-key ${ACCESS_KEY} \
    --secret-key ${SECRET_KEY}

# Verify upload
aws --endpoint-url=${ENDPOINT} \
    s3 ls s3://${BUCKET}/daily/ \
    --access-key ${ACCESS_KEY} \
    --secret-key ${SECRET_KEY}

# Clean up local archive
rm /tmp/backup-${DATE}.tar.gz

echo "Backup completed: ${DATE}"
```

---

## 7. Cluster Testing & Production Readiness

### 7.1 Pre-Production Testing Checklist

```bash
# Run these tests BEFORE going to production

# ✅ Test 1: Basic Cluster Health
sudo cephadm shell -- ceph -s
# Must show: HEALTH_OK

# ✅ Test 2: Pool Operations
sudo cephadm shell -- rbd create test-image --size 1G --pool rbd-pool
sudo cephadm shell -- rbd ls rbd-pool
sudo cephadm shell -- rbd rm test-image --pool rbd-pool

# ✅ Test 3: Data Write/Read Test
# Create a test file via RBD
sudo cephadm shell -- rbd create io-test --size 10G --pool rbd-pool
sudo rbd map io-test --pool rbd-pool
sudo mkfs.ext4 /dev/rbd/io-test
sudo mount /dev/rbd/io-test /mnt
sudo dd if=/dev/urandom of=/mnt/testfile bs=1M count=1000
sudo umount /mnt
sudo rbd unmap /dev/rbd/io-test
# Verify data integrity by remounting and checking file

# ✅ Test 4: OSD Failure Simulation
# Find an OSD ID
sudo cephadm shell -- ceph osd tree
# Stop one OSD daemon (replace X with actual OSD number)
sudo cephadm shell -- ceph orch daemon stop osd.X
# Check cluster reacts correctly
sudo cephadm shell -- ceph -s
# Should show: HEALTH_WARN, osd.X down, data rebalancing
# Restart OSD
sudo cephadm shell -- ceph orch daemon start osd.X
# Verify recovery
sudo cephadm shell -- ceph -s  # Should return to HEALTH_OK

# ✅ Test 5: Node Failure Simulation
# Stop all Ceph services on one node (Node2)
ssh root@ceph-node2 "systemctl stop ceph-mon@*.service ceph-mgr@*.service ceph-osd@*.service"
# Check cluster status from Node1
sudo cephadm shell -- ceph -s
# Should show: Node2 down, but cluster still HEALTH_OK (if quorum maintained)
# Restart services
ssh root@ceph-node2 "systemctl start ceph-mon@*.service ceph-mgr@*.service ceph-osd@*.service"

# ✅ Test 6: Network Partition Test (Advanced)
# Block cluster network traffic temporarily (use with caution!)
ssh root@ceph-node2 "iptables -A OUTPUT -d 192.168.10.0/24 -j DROP"
# Monitor cluster behavior
sudo cephadm shell -- ceph -s
# Restore network
ssh root@ceph-node2 "iptables -D OUTPUT -d 192.168.10.0/24 -j DROP"
```

### 7.2 Performance Benchmarking

```bash
# Install benchmarking tools
sudo apt install -y fio sysstat

# Test RBD performance
sudo cephadm shell -- rbd create perf-test --size 10G --pool rbd-pool
sudo rbd map perf-test --pool rbd-pool

# Run sequential write test
sudo fio --name=seq-write --ioengine=libaio --iodepth=32 \
  --rw=write --bs=1M --direct=1 --size=5G \
  --filename=/dev/rbd/perf-test --group_reporting

# Run random read test
sudo fio --name=rand-read --ioengine=libaio --iodepth=32 \
  --rw=randread --bs=4k --direct=1 --size=5G \
  --filename=/dev/rbd/perf-test --group_reporting

# Clean up
sudo rbd unmap /dev/rbd/perf-test
sudo cephadm shell -- rbd rm perf-test --pool rbd-pool

# Record results for production baseline
```

### 7.3 Backup Configuration (Critical for Production)

#### Option 1: Ceph Native Snapshots + Remote Copy

```bash
# 1. Create snapshot of RBD image
sudo cephadm shell -- rbd snap create rbd-pool/vm-disk-1@daily-backup

# 2. Protect snapshot (required for cloning)
sudo cephadm shell -- rbd snap protect rbd-pool/vm-disk-1@daily-backup

# 3. Export snapshot to remote location
sudo rbd export rbd-pool/vm-disk-1@daily-backup - | \
  ssh backup-server "cat > /backup/vm-disk-1-$(date +%F).img"

# 4. Schedule with cron (daily at 2 AM)
echo "0 2 * * * root /usr/local/bin/ceph-backup.sh" | sudo tee -a /etc/cron.d/ceph-backup
```

#### Option 2: RBD Mirroring (Active-Active Replication)

```bash
# For disaster recovery to a second Ceph cluster

# On PRIMARY cluster:
sudo cephadm shell -- ceph fs mirror peer bootstrap create myfs --site-name primary

# On SECONDARY cluster:
# Get token from primary, then:
sudo cephadm shell -- ceph fs mirror peer bootstrap import myfs --site-name secondary <token>

# Enable mirroring for specific directories
sudo cephadm shell -- ceph fs mirror enable myfs /important-data
```

#### Option 3: RGW Bucket Replication

```bash
# Enable cross-region replication for S3 buckets

# On RGW admin:
radosgw-admin bucket policy set --bucket=user-photos --policy=- << 'EOF'
{
  "ReplicationConfiguration": {
    "Role": "",
    "Rules": [{
      "ID": "replicate-to-dr",
      "Status": "Enabled",
      "Prefix": "",
      "Destination": {
        "Bucket": "arn:aws:s3:::user-photos-dr",
        "StorageClass": "STANDARD"
      }
    }]
  }
}
EOF
```

### 7.4 Monitoring & Alerting Setup

```bash
# 1. Enable Prometheus metrics (if not already done)
sudo cephadm shell -- ceph orch apply prometheus
sudo cephadm shell -- ceph orch apply grafana

# 2. Create alerting rules for critical issues
cat > ceph-alerts.yml << 'EOF'
groups:
- name: ceph-alerts
  rules:
  - alert: CephHealthError
    expr: ceph_health_status == 2
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Ceph cluster in ERROR state"
      
  - alert: OSDDown
    expr: ceph_osd_up == 0
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "OSD {{ $labels.ceph_daemon }} is down"
      
  - alert: PoolNearFull
    expr: ceph_pool_percent_used > 85
    for: 15m
    labels:
      severity: warning
    annotations:
      summary: "Pool {{ $labels.pool }} is {{ $value }}% full"
EOF

# 3. Configure Alertmanager to send notifications
# (Email, Slack, PagerDuty, etc.)
```

### 7.5 Production Go-Live Checklist

```bash
# ✅ Final Verification Before Production

# 1. Cluster Health
[ ] ceph -s shows HEALTH_OK
[ ] All OSDs up and in
[ ] All MONs in quorum
[ ] No stuck PGs

# 2. Security
[ ] Firewall rules configured (only required ports open)
[ ] Client keys have minimal permissions (principle of least privilege)
[ ] RGW using HTTPS in production
[ ] SSH access restricted to admin IPs

# 3. Performance Baseline
[ ] Documented read/write IOPS and latency
[ ] Network throughput tested
[ ] PG distribution balanced across OSDs

# 4. Backup & Recovery
[ ] Backup procedure documented and tested
[ ] Recovery procedure tested (restore from backup)
[ ] RBD snapshots working
[ ] RGW bucket versioning enabled (if needed)

# 5. Monitoring
[ ] Grafana dashboards configured
[ ] Alerting rules active
[ ] Log aggregation set up (optional: ELK stack)

# 6. Documentation
[ ] Network diagram updated
[ ] IP addresses and hostnames documented
[ ] Recovery runbook created
[ ] Contact list for on-call support

# 7. Change Management
[ ] Maintenance window scheduled for go-live
[ ] Rollback plan documented
[ ] Stakeholders notified
```

---

## 🎯 Quick Reference Commands Cheat Sheet

```bash
# ===== CLUSTER MANAGEMENT =====
ceph -s                          # Cluster status
ceph health detail              # Detailed health issues
ceph osd tree                   # OSD hierarchy
ceph osd df                     # OSD disk usage
ceph pg stat                    # Placement group status

# ===== POOL MANAGEMENT =====
ceph osd pool ls detail         # List all pools
ceph osd pool create <name> <pg> <pgp>  # Create pool
ceph osd pool delete <name> <name> --yes-i-really-really-mean-it  # Delete pool

# ===== RBD MANAGEMENT =====
rbd ls <pool>                   # List RBD images
rbd create <image> --size <GB> --pool <pool>  # Create image
rbd map <image> --pool <pool>   # Map to client
rbd unmap /dev/rbd/<image>      # Unmap from client
rbd snap create <pool>/<image>@<snap>  # Create snapshot

# ===== RGW MANAGEMENT =====
radosgw-admin user create --uid=<id> --display-name="<name>"  # Create user
radosgw-admin bucket list       # List all buckets
radosgw-admin bucket stats --bucket=<name>  # Bucket statistics

# ===== CEPHFS MANAGEMENT =====
ceph fs ls                      # List file systems
ceph fs status <fsname>         # File system status
ceph fs dump                    # Detailed FS info

# ===== CLIENT ACCESS =====
ceph auth get-or-create client.<name> mon 'allow r' osd 'allow rwx pool=<pool>'  # Create user
ceph auth list                  # List all auth keys
ceph auth del client.<name>     # Delete user

# ===== TROUBLESHOOTING =====
ceph daemon osd.<id> dump_mempools  # OSD memory debug
ceph log recent               # Recent cluster logs
ceph tell mon.* injectargs '--debug_mon 20'  # Enable debug logging
```

---

## 🚨 Common Issues & Solutions

| Issue | Symptom | Solution |
|-------|---------|----------|
| **Client can't connect** | `ceph -s` fails with "connection refused" | 1. Check firewall allows port 6789<br>2. Verify mon_host in ceph.conf<br>3. Test ping/telnet to MON IPs |
| **RBD map fails** | `rbd: sysfs write failed` | 1. Load kernel module: `modprobe rbd`<br>2. Check client kernel supports RBD<br>3. Use `rbd-nbd` as alternative |
| **RGW returns 403** | S3 operations denied | 1. Verify access key/secret<br>2. Check bucket policy<br>3. Ensure user has correct permissions |
| **CephFS mount hangs** | `mount` command never completes | 1. Check MDS daemons running: `ceph orch ps --daemon_type mds`<br>2. Verify network to MDS<br>3. Try `ceph-fuse` instead of kernel mount |
| **Slow performance** | High latency, low throughput | 1. Check network utilization: `iftop`, `nethogs`<br>2. Verify OSDs not overloaded: `ceph osd df`<br>3. Check for rebalancing: `ceph -s` |
| **OSD won't start** | `ceph orch ps` shows OSD failed | 1. Check disk health: `smartctl -a /dev/sdX`<br>2. Review logs: `ceph logs osd.<id>`<br>3. Verify disk not mounted elsewhere |

---

## 📚 Additional Resources

- **Official Ceph Docs**: https://docs.ceph.com/en/latest/
- **Ceph GitHub**: https://github.com/ceph/ceph
- **Community Forum**: https://discuss.ceph.com/
- **PG Calculator**: https://www.ceph.com/pgcalc/
- **Ceph Best Practices**: https://docs.ceph.com/en/latest/rados/configuration/bluestore-config-ref/

---

> 💡 **Pro Tip**: Always test changes in a lab environment before applying to production. Ceph is powerful but complex—small misconfigurations can have big impacts.

> 🔐 **Security Reminder**: Never use the `admin` keyring on client machines. Always create dedicated users with minimal required permissions.

> 📈 **Monitoring is Key**: Set up alerts BEFORE issues happen. A 5-minute warning about a failing OSD can prevent hours of downtime.

---

🎉 **Congratulations!** You now have a complete, production-ready guide for:
- ✅ Preparing client machines
- ✅ Creating and managing storage pools
- ✅ Deploying RGW for object storage
- ✅ Setting up CephFS for file storage
- ✅ Configuring client access (RBD + S3)
- ✅ Testing and validating your cluster

Your Ceph cluster is ready for real-world workloads! 🐘✨

*Need help with a specific step? Ask away!*
