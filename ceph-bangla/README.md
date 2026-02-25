# 📘 Ceph শেখার পূর্ণ রোডম্যাপ (Step-by-Step with Practical Examples)

## 🔹 পর্ব ১: Ceph পরিচিতি ও আর্কিটেকচার

### 🧠 **Ceph Storage কী?**

**Ceph** হল একটি **distributed storage system**, যা object, block এবং file storage একসাথে সাপোর্ট করে। এটা **high performance**, **fault-tolerant**, এবং **scalable**। Ceph মূলত cloud এবং enterprise-গ্রেড storage solution হিসেবে ব্যবহৃত হয়।

---

### 🏗️ **Ceph Storage-এর আর্কিটেকচার (Architecture)**

Ceph মূলত নিচের component-গুলো নিয়ে কাজ করে:

1. **Ceph Monitor (MON)**
   → Cluster-এর health, map, authentication ইত্যাদি ট্র্যাক করে।

2. **Ceph Manager (MGR)**
   → ক্লাস্টারের performance data, dashboard, এবং মেট্রিকস হ্যান্ডেল করে।

3. **Ceph OSD (Object Storage Daemon)**
   → মূল ডেটা রাখে এবং replication, recovery, backfilling করে।

4. **Ceph MDS (Metadata Server)**
   → File system (CephFS) এর metadata manage করে।

5. **RADOS (Reliable Autonomic Distributed Object Store)**
   → Ceph-এর মূল distributed object storage layer।

---

### 📦 **Ceph Storage টাইপস**

1. **Object Storage (RADOS Gateway - RGW)**
   → Amazon S3/Swift compatible API দিয়ে কাজ করে।

2. **Block Storage (RBD - RADOS Block Device)**
   → Virtual machines বা bare-metal server এ use করা হয় (e.g., OpenStack Cinder backend)।

3. **File System Storage (CephFS)**
   → POSIX-compliant distributed file system।

---

### 🧱 **Ceph Cluster তৈরি করার ধাপসমূহ (Step-by-Step)**

#### 🧰 ধাপ ১: সিস্টেম প্রস্তুতি

* Ubuntu/RHEL server প্রস্তুত করুন (minimum 3 nodes)
* `ntp` / `chronyd` sync করুন
* Hostname ও `/etc/hosts` সঠিকভাবে কনফিগার করুন
* ফায়ারওয়াল ও SELinux disable করুন

### 🧰 ধাপ ২: Ceph ইনস্টলেশন

#### 📦 Manual Installation:

```bash
# Ubuntu এর জন্য
sudo apt update
sudo apt install ceph-deploy ceph-common
```
#### 📦 Or use Cephadm (for production)
#### 📦 Or use Ceph-Ansible (for production)

### 🧰 ধাপ ৩: Cluster Bootstrapping (ceph-deploy দিয়ে)

```bash
ceph-deploy new mon1
ceph-deploy install mon1 osd1 osd2 mgr1
ceph-deploy mon create-initial
ceph-deploy admin mon1 osd1 osd2
```

### 🧰 ধাপ ৪: OSD যোগ করুন

```bash
ceph-deploy osd create --data /dev/sdb osd1
ceph-deploy osd create --data /dev/sdb osd2
```

### 🧰 ধাপ ৫: Cluster Status যাচাই

```bash
ceph -s
```

---

### 📊 **Ceph Monitoring & Management**

* **Ceph Dashboard**: Web UI
* **Prometheus + Grafana** integration
* `ceph status`, `ceph df`, `ceph osd tree` CLI commands

---

### ⚙️ **Ceph Storage Pools & CRUSH Map**

* **Storage Pool**: Data logical group
* **CRUSH Map**: Intelligent data placement algorithm যা redundancy ও performance নিশ্চিত করে

---

### 🔐 **Ceph এর Data Redundancy & Recovery**

* **Replication**: Same data multiple OSD-তে থাকে
* **Erasure Coding**: More efficient redundancy technique
* **Self-healing**: এক বা একাধিক ডেটা লস হলেও নিজে থেকেই restore করে

---

### 🎯 **Ceph এর ব্যবহার ক্ষেত্র**

| Use Case           | Explanation                |
| ------------------ | -------------------------- |
| OpenStack Backend  | Cinder, Glance, Nova       |
| Kubernetes         | Rook দিয়ে Ceph চালানো      |
| Enterprise Backup  | Large-scale object storage |
| Video Surveillance | Large file archival        |

---

---

## ✅ পর্ব ২: Ceph Cluster তৈরি করা (Installation + Configuration + Initial Setup)

এখানে আমরা দেখবো:

* ৩টি VM/Server দিয়ে Cluster বানানো
* Ceph-deploy দিয়ে Ceph ইনস্টল করা
* OSD, Monitor, MGR কনফিগার করা

---

### 🖥️ **Step 1: সার্ভার প্রস্তুতি (3 Node Lab Setup)**

#### 🖥️ আমরা ধরছি আমাদের কাছে ৩টি Ubuntu Server আছে:

| Node Name | IP Address    | Role             |
| --------- | ------------- | ---------------- |
| mon1      | 192.168.56.10 | Monitor, Manager |
| osd1      | 192.168.56.11 | OSD Node         |
| osd2      | 192.168.56.12 | OSD Node         |

---

### 🛠️ **Step 2: প্রতিটি সার্ভারে এই Common Setup করুন**

```bash
# Hostname সেট করুন (প্রতিটি নোডে ভিন্নভাবে)
sudo hostnamectl set-hostname mon1   # অথবা osd1, osd2

# hosts ফাইল আপডেট করুন
sudo nano /etc/hosts
```

```bash
192.168.56.10 mon1
192.168.56.11 osd1
192.168.56.12 osd2
```

```bash
# Update & Install basic tools
sudo apt update && sudo apt install -y ntp sshpass python3-pip
```

---

### 📥 **Step 3: Ceph-deploy ইনস্টল করুন (Only on mon1)**

```bash
sudo apt install ceph-deploy -y
```

---

### 🔐 **Step 4: SSH Key তৈরি এবং password-less login enable করুন**

```bash
ssh-keygen
ssh-copy-id user@osd1
ssh-copy-id user@osd2
```

---

### 📂 **Step 5: ক্লাস্টার ইনিশিয়ালাইজ করুন (mon1-এ)**

```bash
mkdir ceph-cluster
cd ceph-cluster

# নতুন cluster তৈরি করুন
ceph-deploy new mon1
```

---

### 📦 **Step 6: সমস্ত Node-এ Ceph ইনস্টল করুন**

```bash
ceph-deploy install mon1 osd1 osd2
```

---

### 🧠 **Step 7: Monitor এবং Manager তৈরি করুন**

```bash
ceph-deploy mon create-initial
ceph-deploy mgr create mon1
```

---

### 🛡️ **Step 8: Admin Key বিতরণ করুন**

```bash
ceph-deploy admin mon1 osd1 osd2
chmod +r /etc/ceph/ceph.client.admin.keyring
```

---

### 💾 **Step 9: OSD তৈরি করুন (ধরি `/dev/sdb` device আছে)**

```bash
ceph-deploy osd create --data /dev/sdb osd1
ceph-deploy osd create --data /dev/sdb osd2
```

---

### 🔍 **Step 10: ক্লাস্টার স্টেটাস চেক করুন**

```bash
ceph -s
```

উদাহরণ Output:

```bash
cluster:
  id:     3a1b-4e32-91a1-1e2443e9c9d6
  health: HEALTH_OK

services:
  mon: 1 daemons, quorum mon1
  mgr: mon1(active)
  osd: 2 osds: 2 up, 2 in

data:
  pools:   1 pools, 100 pgs
  objects: 0 objects, 0B
```

---

#### 🎉 এখন একটি চলমান Ceph Cluster তৈরি হয়েছে!

---

# ✅ পর্ব ৩: Ceph Block Storage (RBD) – Complete Practical Guide

---

# 📌 3.1 Ceph RBD কী?

**RBD (RADOS Block Device)** হলো Ceph-এর Block Storage সিস্টেম।

এটি ব্যবহার করা হয়:

- KVM / QEMU Virtual Machine disk হিসেবে
    
- OpenStack Cinder backend হিসেবে
    
- Kubernetes Persistent Volume হিসেবে
    
- Database storage backend হিসেবে
    

---

# 📌 3.2 RBD Architecture বুঝে নেই

Flow:

Application → RBD → RADOS → OSD → Disk

Ceph Block Storage কাজ করে:

- Pool এর ভিতরে image তৈরি করে
    
- Image = virtual disk
    
- VM সেই image কে disk হিসেবে ব্যবহার করে
    

---

# 📌 3.3 Step 1: RBD Pool তৈরি করা

Ceph default pool ব্যবহার না করে আমরা আলাদা block pool তৈরি করবো।

```bash
ceph osd pool create rbd_pool 128
```

Application enable করুন:

```bash
ceph osd pool application enable rbd_pool rbd
```

Pool verify করুন:

```bash
ceph osd pool ls
```

---

# 📌 3.4 Step 2: RBD Image তৈরি করা

ধরি আমরা 10GB virtual disk তৈরি করবো:

```bash
rbd create myblock1 --size 10240 --pool rbd_pool
```

List দেখুন:

```bash
rbd ls rbd_pool
```

Details দেখুন:

```bash
rbd info rbd_pool/myblock1
```

---

# 📌 3.5 Step 3: RBD Image Map করা (Linux Server এ)

Client machine এ ceph-common install থাকতে হবে।

```bash
sudo apt install ceph-common
```

Image map করুন:

```bash
sudo rbd map rbd_pool/myblock1
```

Output:

```
/dev/rbd0
```

এখন এটি একটি real block device হিসেবে কাজ করছে।

---

# 📌 3.6 Step 4: Filesystem তৈরি ও Mount করা

```bash
sudo mkfs.ext4 /dev/rbd0
```

Mount করুন:

```bash
sudo mkdir /mnt/rbd1
sudo mount /dev/rbd0 /mnt/rbd1
```

Test করুন:

```bash
sudo touch /mnt/rbd1/testfile
```

---

# 📌 3.7 Step 5: Unmap করা

```bash
sudo umount /mnt/rbd1
sudo rbd unmap /dev/rbd0
```

---

# 📌 3.8 RBD Resize করা

ধরি 10GB → 20GB করতে চাই

```bash
rbd resize rbd_pool/myblock1 --size 20480
```

Filesystem resize:

```bash
resize2fs /dev/rbd0
```

---

# 📌 3.9 Snapshot তৈরি করা

Snapshot তৈরি:

```bash
rbd snap create rbd_pool/myblock1@snap1
```

List:

```bash
rbd snap ls rbd_pool/myblock1
```

Rollback:

```bash
rbd snap rollback rbd_pool/myblock1@snap1
```

Delete snapshot:

```bash
rbd snap rm rbd_pool/myblock1@snap1
```

---

# 📌 3.10 Clone তৈরি করা

Clone করার আগে snapshot protect করতে হবে:

```bash
rbd snap protect rbd_pool/myblock1@snap1
```

Clone:

```bash
rbd clone rbd_pool/myblock1@snap1 rbd_pool/myclone1
```

---

# 📌 3.11 KVM / QEMU তে RBD ব্যবহার

Install:

```bash
sudo apt install qemu-kvm libvirt-daemon-system
```

VM create করার সময় disk source হিসেবে:

```bash
--disk path=rbd:rbd_pool/myblock1
```

Or XML config এ:

```xml
<disk type='network' device='disk'>
  <driver name='qemu' type='raw'/>
  <source protocol='rbd' name='rbd_pool/myblock1'>
  </source>
</disk>
```

এভাবে VM disk হিসেবে Ceph RBD ব্যবহার করবে।

---

# 📌 3.12 OpenStack এ Ceph RBD Backend

OpenStack Cinder config:

File: `/etc/cinder/cinder.conf`

```ini
[ceph]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
rbd_pool = rbd_pool
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = false
```

Restart services:

```bash
systemctl restart cinder-volume
```

এখন OpenStack Volume তৈরি করলে তা Ceph এ যাবে।

---

# 📌 3.13 Kubernetes এ Ceph RBD ব্যবহার (CSI)

Modern Kubernetes এ CSI driver ব্যবহার হয়।

Deploy:

- ceph-csi
    
- secret
    
- storageclass
    

StorageClass example:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd
provisioner: rbd.csi.ceph.com
parameters:
  pool: rbd_pool
```

PVC তৈরি করুন:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  storageClassName: ceph-rbd
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

Pod attach করলে dynamic RBD create হবে।

---

# 📌 3.14 RBD Performance Tuning

Important Parameters:

```bash
ceph config set osd osd_op_threads 8
ceph config set osd osd_max_backfills 4
```

RBD Cache enable:

```bash
rbd feature enable rbd_pool/myblock1 exclusive-lock
```

---

# 📌 3.15 Real World Use Case Example

### Scenario:

একটি Production Database Server

- MySQL VM running on KVM
    
- Disk backed by Ceph RBD
    
- Snapshot taken every 6 hour
    
- Clone used for testing environment
    
- If VM crash → move VM to another host → attach same RBD → start
    

Result:

- No data loss
    
- High availability
    
- Centralized storage
    

---

# 📌 3.16 RBD Important Commands Summary

|কাজ|কমান্ড|
|---|---|
|Pool create|ceph osd pool create|
|Image create|rbd create|
|Map|rbd map|
|Resize|rbd resize|
|Snapshot|rbd snap create|
|Clone|rbd clone|
|Delete|rbd rm|

---

# ✅ পর্ব ৪: CephFS – File System তৈরি ও ব্যবহার (Complete Practical Guide)

---

# 📌 4.1 CephFS কী?

**CephFS** হলো Ceph-এর Distributed POSIX-compliant File System।

এটি ব্যবহার করা হয়:

- Shared storage (NFS alternative)
    
- Kubernetes shared volume
    
- Web server shared data
    
- Application cluster storage
    
- Big data analytics
    

---

# 📌 4.2 CephFS Architecture বুঝে নেই

CephFS কাজ করে:

Client → MDS → RADOS → OSD

এখানে:

- **MDS (Metadata Server)** → file metadata handle করে
    
- **OSD** → actual data store করে
    

Block storage থেকে পার্থক্য:

|RBD|CephFS|
|---|---|
|Block level|File level|
|VM disk|Shared file storage|
|Single attach|Multiple client mount|

---

# 📌 4.3 Step 1: MDS (Metadata Server) তৈরি করা

আপনার cluster এ যদি MDS না থাকে:

```bash
ceph-deploy mds create mon1
```

Status দেখুন:

```bash
ceph -s
```

Output এ দেখতে পাবেন:

```
mds: 1/1 daemons up
```

---

# 📌 4.4 Step 2: CephFS এর জন্য Pool তৈরি করা

CephFS এর জন্য ২টি pool লাগে:

- Metadata Pool
    
- Data Pool
    

```bash
ceph osd pool create cephfs_data 128
ceph osd pool create cephfs_metadata 64
```

Application enable করুন:

```bash
ceph osd pool application enable cephfs_data cephfs
ceph osd pool application enable cephfs_metadata cephfs
```

---

# 📌 4.5 Step 3: CephFS তৈরি করা

```bash
ceph fs new mycephfs cephfs_metadata cephfs_data
```

Verify করুন:

```bash
ceph fs ls
```

---

# 📌 4.6 Step 4: CephFS Mount করা (Kernel Client Method)

Client machine এ:

```bash
sudo apt install ceph-common
```

Mount করুন:

```bash
sudo mkdir /mnt/cephfs

sudo mount -t ceph mon1:6789:/ /mnt/cephfs \
-o name=admin,secretfile=/etc/ceph/ceph.client.admin.keyring
```

Test করুন:

```bash
sudo touch /mnt/cephfs/testfile1
sudo mkdir /mnt/cephfs/project-data
```

---

# 📌 4.7 Persistent Mount (fstab)

```bash
sudo nano /etc/fstab
```

Add:

```
mon1:6789:/ /mnt/cephfs ceph name=admin,secretfile=/etc/ceph/ceph.client.admin.keyring,_netdev 0 0
```

---

# 📌 4.8 FUSE দিয়ে Mount করা (Alternative Method)

```bash
sudo apt install ceph-fuse
```

```bash
sudo ceph-fuse -m mon1:6789 /mnt/cephfs
```

Kernel mount বেশি performance দেয়।

---

# 📌 4.9 Subvolume তৈরি করা (Production Use)

Enterprise environment এ পুরো root share না দিয়ে subvolume দেয়া হয়।

```bash
ceph fs subvolume create mycephfs user1vol
```

List করুন:

```bash
ceph fs subvolume ls mycephfs
```

Mount path জানুন:

```bash
ceph fs subvolume getpath mycephfs user1vol
```

---

# 📌 4.10 CephFS User তৈরি (Restricted Access)

New user create:

```bash
ceph auth get-or-create client.user1 \
mon 'allow r' \
mds 'allow rw path=/volumes/_nogroup/user1vol' \
osd 'allow rw pool=cephfs_data'
```

Key save করুন:

```bash
ceph auth get-key client.user1 > user1.key
```

Mount using user1:

```bash
sudo mount -t ceph mon1:6789:/volumes/_nogroup/user1vol /mnt/user1 \
-o name=user1,secretfile=user1.key
```

---

# 📌 4.11 Kubernetes এ CephFS ব্যবহার (CSI Driver)

CephFS CSI ব্যবহার করলে multiple pod shared storage পায়।

StorageClass example:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cephfs-sc
provisioner: cephfs.csi.ceph.com
parameters:
  fsName: mycephfs
  pool: cephfs_data
```

PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: cephfs-sc
  resources:
    requests:
      storage: 5Gi
```

Pod এ mount করলে shared storage হবে।

---

# 📌 4.12 Real World Use Case Example

### Scenario 1: Web Server Cluster

- 3টি Nginx server
    
- Shared upload folder
    
- CephFS mounted on `/var/www/html/uploads`
    

Result:

- যে কোন server এ upload করলে সব server এ visible
    
- No NFS single point of failure
    
- High scalability
    

---

### Scenario 2: AI / Big Data

- Multiple processing node
    
- Shared dataset directory
    
- Parallel processing possible
    

---

# 📌 4.13 Performance Tuning

MDS scaling:

```bash
ceph fs set mycephfs max_mds 2
```

Cache size tune:

```bash
ceph config set mds mds_cache_memory_limit 4G
```

---

# 📌 4.14 CephFS Snapshot

Directory snapshot:

```bash
mkdir /mnt/cephfs/mydir/.snap
mkdir /mnt/cephfs/mydir/.snap/snap1
```

List snapshot:

```bash
ls /mnt/cephfs/mydir/.snap
```

---

# 📌 4.15 CephFS Delete করা

```bash
ceph fs rm mycephfs --yes-i-really-mean-it
```

---

# 📌 4.16 CephFS vs RBD Comparison

|Feature|RBD|CephFS|
|---|---|---|
|Access|Single client|Multiple client|
|Type|Block|File|
|Best for|VM Disk|Shared App Storage|
|Kubernetes|RWO|RWX|

---

# 🎯 পর্ব ৪ শেষে আপনি জানলেন:

✔ CephFS Architecture  
✔ Pool creation  
✔ Filesystem create  
✔ Mount (Kernel & Fuse)  
✔ Subvolume  
✔ Restricted user  
✔ Kubernetes integration  
✔ Snapshot  
✔ Performance tuning  
✔ Real production use

---

# ✅ পর্ব ৫: Object Storage (RGW) দিয়ে S3 Bucket তৈরি (Complete Practical Guide)

---

# 📌 5.1 Ceph Object Storage কী?

Ceph Object Storage কাজ করে **RADOS Gateway (RGW)** এর মাধ্যমে।

এটি Compatible:

- Amazon S3 API
    
- OpenStack Swift API
    

ব্যবহার করা হয়:

- Backup storage
    
- Image storage
    
- Log storage
    
- Application object storage
    
- Kubernetes backup (Velero)
    
- Media storage
    

---

# 📌 5.2 Ceph RGW Architecture

Client → S3 API → RGW → RADOS → OSD

এখানে:

- RGW = S3-compatible gateway
    
- Data = object হিসেবে store হয়
    
- Bucket = logical container
    

---

# 📌 5.3 Step 1: RGW Install করা

mon1 node এ:

```bash
ceph-deploy rgw create mon1
```

Service check করুন:

```bash
systemctl status ceph-radosgw@rgw.mon1
```

Port check করুন (default 7480):

```bash
ss -tulnp | grep rados
```

Browser এ test করুন:

```
http://mon1:7480
```

---

# 📌 5.4 Step 2: Object Gateway User তৈরি করা

Admin user তৈরি করুন:

```bash
radosgw-admin user create \
--uid="adminuser" \
--display-name="Admin User"
```

Output এ পাবেন:

- access_key
    
- secret_key
    

এই key সংরক্ষণ করুন।

---

# 📌 5.5 Step 3: S3 Client Install করা (s3cmd)

Client machine এ:

```bash
sudo apt install s3cmd
```

Configure করুন:

```bash
s3cmd --configure
```

Input দিন:

```
Access Key: <your_access_key>
Secret Key: <your_secret_key>
Default Region: us-east-1
S3 Endpoint: mon1:7480
DNS-style bucket: No
```

---

# 📌 5.6 Step 4: Bucket তৈরি করা

```bash
s3cmd mb s3://mybucket1
```

List bucket:

```bash
s3cmd ls
```

---

# 📌 5.7 Step 5: File Upload / Download

File upload:

```bash
s3cmd put testfile.txt s3://mybucket1
```

List objects:

```bash
s3cmd ls s3://mybucket1
```

Download:

```bash
s3cmd get s3://mybucket1/testfile.txt
```

Delete:

```bash
s3cmd del s3://mybucket1/testfile.txt
```

---

# 📌 5.8 Step 6: Bucket Policy & Public Access

Public read permission:

```bash
s3cmd setacl s3://mybucket1 --acl-public
```

Now file access:

```
http://mon1:7480/mybucket1/testfile.txt
```

---

# 📌 5.9 Multiple User তৈরি করা

```bash
radosgw-admin user create \
--uid="appuser1" \
--display-name="Application User"
```

Quota সেট করুন:

```bash
radosgw-admin quota set \
--uid=appuser1 \
--quota-scope=user \
--max-size=10G \
--enabled=true
```

---

# 📌 5.10 Real World Use Case Example

---

## Scenario 1: Application Image Storage

ধরি একটি Web Application:

- User image upload করে
    
- Application server image store করে Ceph bucket এ
    
- Backend শুধুমাত্র URL সংরক্ষণ করে
    

Benefits:

- Application server disk ব্যবহার হয় না
    
- Scalable storage
    
- Multiple app server share same object storage
    

---

## Scenario 2: Backup Storage

- Daily database dump
    
- Cron job দিয়ে upload to S3
    
- Cheap & scalable storage
    
- Lifecycle policy দিয়ে old backup delete
    

---

## Scenario 3: Kubernetes Velero Backup

Velero config:

```yaml
apiVersion: velero.io/v1
kind: BackupStorageLocation
spec:
  provider: aws
  objectStorage:
    bucket: mybucket1
```

Velero Ceph S3 ব্যবহার করে backup store করবে।

---

# 📌 5.11 RGW Multi-site (Advanced Concept)

Production environment এ:

- Multiple region
    
- Bucket replication
    
- Disaster recovery
    

Command example:

```bash
radosgw-admin realm create --rgw-realm=realm1 --default
```

---

# 📌 5.12 Performance Tuning

RGW thread increase:

```bash
ceph config set client.rgw rgw_thread_pool_size 512
```

Bucket index sharding:

```bash
radosgw-admin bucket reshard --bucket=mybucket1 --num-shards=16
```

---

# 📌 5.13 Object Versioning Enable করা

```bash
s3cmd setversioning s3://mybucket1 enable
```

---

# 📌 5.14 RGW Log & Troubleshooting

Log location:

```
/var/log/ceph/ceph-client.rgw.mon1.log
```

Health check:

```bash
ceph -s
```

---

# 📌 5.15 Ceph Object vs CephFS vs RBD Comparison

|Feature|RBD|CephFS|Object|
|---|---|---|---|
|Type|Block|File|Object|
|API|Block device|POSIX|S3|
|Use|VM Disk|Shared App|Backup, Media|
|Access|Single|Multi|HTTP|

---

# ✅ পর্ব ৬: Monitoring & Web Dashboard চালু করা (Complete Practical Guide)

---

# 📌 6.1 কেন Ceph Monitoring গুরুত্বপূর্ণ?

Production environment এ Monitoring অত্যন্ত গুরুত্বপূর্ণ কারণ:

- Disk failure detect করা
    
- OSD down হলে alert পাওয়া
    
- Capacity planning করা
    
- Performance bottleneck ধরতে পারা
    
- Cluster health realtime দেখা
    

---

# 📌 6.2 Ceph Monitoring Architecture

Monitoring Layer:

- Ceph CLI
    
- Ceph Dashboard (Web UI)
    
- Prometheus
    
- Grafana
    
- Alertmanager
    

Flow:

Ceph Cluster → Metrics → Prometheus → Grafana → Alert

---

# 📌 6.3 Step 1: Ceph Manager Module Enable করা

Ceph Dashboard MGR module এর মাধ্যমে কাজ করে।

Check করুন:

```bash
ceph mgr module ls
```

Enable করুন:

```bash
ceph mgr module enable dashboard
```

---

# 📌 6.4 Step 2: SSL Certificate তৈরি করা

```bash
ceph dashboard create-self-signed-cert
```

---

# 📌 6.5 Step 3: Admin User তৈরি করা

```bash
ceph dashboard set-login-credentials admin admin123
```

---

# 📌 6.6 Step 4: Dashboard Port Check

Default port: 8443

Check করুন:

```bash
ceph mgr services
```

Browser এ যান:

```
https://mon1:8443
```

Login:

- Username: admin
    
- Password: admin123
    

---

# 📌 6.7 Dashboard এ কী কী দেখতে পাবেন?

Dashboard Overview এ:

- Cluster health
    
- OSD status
    
- MON status
    
- MDS status
    
- Pool usage
    
- Object count
    
- Throughput graph
    
- IOPS graph
    

---

# 📌 6.8 Real World Monitoring Scenario

### Scenario 1: OSD Disk Failure

যদি একটি disk fail করে:

```bash
ceph osd out osd.1
```

Dashboard এ দেখবেন:

- HEALTH_WARN
    
- PG degraded
    
- Recovery progress bar
    

আপনি visually recovery progress দেখতে পারবেন।

---

### Scenario 2: Capacity Planning

Dashboard → Pools → Usage

ধরি:

- rbd_pool 75% full
    

আপনি আগেই নতুন OSD যোগ করতে পারবেন।

---

# 📌 6.9 Prometheus Enable করা

```bash
ceph mgr module enable prometheus
```

Check endpoint:

```
http://mon1:9283/metrics
```

---

# 📌 6.10 Grafana Enable করা

```bash
ceph mgr module enable grafana
```

Install Grafana (if not installed):

```bash
sudo apt install grafana
```

Start service:

```bash
systemctl start grafana-server
```

Access:

```
http://mon1:3000
```

Default login:

- admin
    
- admin
    

---

# 📌 6.11 Ceph Built-in Grafana Dashboard

Ceph automatically import dashboard templates।

Metrics পাবেন:

- OSD latency
    
- Read/Write IOPS
    
- Throughput
    
- PG state
    
- CPU usage
    
- Disk utilization
    

---

# 📌 6.12 Alertmanager Setup

Enable:

```bash
ceph mgr module enable alertmanager
```

Alert example:

- OSD down
    
- Disk full
    
- MON quorum lost
    
- Slow requests
    

Email alert configure করতে হলে:

```bash
ceph dashboard set-alertmanager-api-host http://localhost:9093
```

---

# 📌 6.13 Important CLI Monitoring Commands

Cluster health:

```bash
ceph -s
```

Detailed health:

```bash
ceph health detail
```

OSD tree:

```bash
ceph osd tree
```

Pool usage:

```bash
ceph df
```

IO performance:

```bash
ceph osd perf
```

PG status:

```bash
ceph pg stat
```

---

# 📌 6.14 Log Monitoring

Log location:

```
/var/log/ceph/
```

Live log:

```bash
tail -f /var/log/ceph/ceph.log
```

---

# 📌 6.15 Real Production Monitoring Example

## Scenario: Production Kubernetes Cluster

- RBD storage for database
    
- CephFS for shared storage
    
- RGW for backup
    

Monitoring setup:

- Dashboard for quick view
    
- Grafana for deep performance metrics
    
- Alertmanager for email alert
    
- Weekly capacity review
    

Result:

- Proactive issue detection
    
- Zero surprise outage
    
- Scalable planning
    

---

# 📌 6.16 Performance Analysis Example

High latency detect করলে:

```bash
ceph osd perf
```

High commit latency মানে:

- Disk slow
    
- Network issue
    
- CPU bottleneck
    

Dashboard graph দিয়ে root cause identify করা যায়।

---

# 📌 6.17 Security Best Practice

Dashboard password change:

```bash
ceph dashboard ac-user-set-password admin
```

HTTPS mandatory রাখুন।

Firewall open করুন শুধুমাত্র trusted IP এর জন্য।

---

# ✅ পর্ব ৭: Disaster Recovery, Performance Tuning & Pool Management (Complete Practical Guide)

এটি সবচেয়ে গুরুত্বপূর্ণ অধ্যায়।  
Production environment এ Ceph চালাতে হলে এই অংশ ভালোভাবে বুঝতেই হবে।

---

# 📌 7.1 Disaster Recovery (DR) Overview

Ceph naturally fault-tolerant কারণ:

- Data replication
    
- Self-healing
    
- No single point of failure
    

কিন্তু Production DR মানে:

- Node failure recovery
    
- OSD failure recovery
    
- Pool recovery
    
- Cluster backup
    
- Multi-site replication
    

---

# 📌 7.2 OSD Failure Recovery (Real Scenario)

### Scenario:

একটি disk হঠাৎ নষ্ট হয়ে গেছে।

Check করুন:

```bash
ceph -s
```

Output:

```
HEALTH_WARN
1 osd down
```

Identify করুন:

```bash
ceph osd tree
```

---

## Step 1: OSD mark out করুন

```bash
ceph osd out osd.1
```

Cluster rebalancing শুরু হবে।

---

## Step 2: Disk replace করুন

New disk detect হলে:

```bash
ceph-volume lvm create --data /dev/sdb
```

---

## Step 3: Verify recovery

```bash
ceph -s
```

HEALTH_OK না আসা পর্যন্ত অপেক্ষা করুন।

---

# 📌 7.3 Node Failure Recovery

### Scenario:

পুরো একটি OSD node down।

Cluster automatically:

- Replication maintain করবে
    
- Data অন্য OSD তে redistribute করবে
    

Node repair হলে:

```bash
systemctl start ceph-osd.target
```

---

# 📌 7.4 Pool Replication Management

Replication check করুন:

```bash
ceph osd pool get rbd_pool size
```

Replication change করুন:

```bash
ceph osd pool set rbd_pool size 3
```

Meaning:

- size=3 → 3 copies থাকবে
    

Minimum replica:

```bash
ceph osd pool set rbd_pool min_size 2
```

---

# 📌 7.5 Erasure Coding (Storage Efficiency)

Replication expensive।

Erasure coding efficient।

Create EC profile:

```bash
ceph osd erasure-code-profile set ecprofile k=2 m=1
```

Create EC pool:

```bash
ceph osd pool create ecpool 128 128 erasure ecprofile
```

Use case:

- Backup storage
    
- Archive storage
    

---

# 📌 7.6 CRUSH Map (Data Placement Control)

CRUSH algorithm determine করে:

- কোন OSD তে data যাবে
    
- Rack awareness
    
- Failure domain
    

Check rule:

```bash
ceph osd crush rule ls
```

Custom rule create (rack based):

```bash
ceph osd crush rule create-replicated rack_rule default rack
```

Apply to pool:

```bash
ceph osd pool set rbd_pool crush_rule rack_rule
```

Production datacenter এ অত্যন্ত গুরুত্বপূর্ণ।

---

# 📌 7.7 Performance Tuning (Advanced)

---

## 1️⃣ OSD Thread Tune

```bash
ceph config set osd osd_op_threads 8
```

---

## 2️⃣ Backfill Limit

```bash
ceph config set osd osd_max_backfills 4
```

---

## 3️⃣ Recovery Speed Tune

```bash
ceph config set osd osd_recovery_max_active 5
```

---

## 4️⃣ BlueStore Optimization

```bash
ceph config set osd bluestore_cache_size 4G
```

---

# 📌 7.8 Network Optimization

Production recommendation:

- 10Gbps network
    
- Separate public & cluster network
    

ceph.conf example:

```
public network = 192.168.1.0/24
cluster network = 10.0.0.0/24
```

---

# 📌 7.9 Backup Strategy (Critical Part)

Ceph backup options:

1. RBD snapshot export
    
2. RGW multi-site replication
    
3. CephFS snapshot
    
4. External backup system
    

---

## RBD Export Example

```bash
rbd export rbd_pool/myblock1 backup.img
```

Restore:

```bash
rbd import backup.img rbd_pool/restoredblock
```

---

# 📌 7.10 Cluster Backup (Config Backup)

Backup these files:

```
/etc/ceph/
/var/lib/ceph/
```

---

# 📌 7.11 Pool Management

List pools:

```bash
ceph osd pool ls
```

Delete pool:

```bash
ceph osd pool delete testpool testpool --yes-i-really-really-mean-it
```

Rename pool:

```bash
ceph osd pool rename oldname newname
```

---

# 📌 7.12 PG (Placement Group) Optimization

Check PG:

```bash
ceph osd pool get rbd_pool pg_num
```

Increase PG:

```bash
ceph osd pool set rbd_pool pg_num 256
```

Rule:

PG ≈ (OSD * 100) / replica size

---

# 📌 7.13 Real Production Disaster Scenario

### Scenario:

Datacenter A completely down।

Solution:

- Multi-site RGW replication
    
- Secondary Ceph cluster
    
- Backup import
    
- DNS failover
    

Result:

- Minimal downtime
    
- Business continuity
    

---

# 📌 7.14 Cluster Upgrade Strategy

Upgrade safely:

```bash
ceph orch upgrade start --ceph-version <version>
```

Upgrade order:

1. MON
    
2. MGR
    
3. OSD
    
4. MDS
    
5. RGW
    

Never upgrade all at once।

---

# 📌 7.15 High Availability Best Practice Summary

✔ Minimum 3 MON  
✔ Odd number MON  
✔ Multiple OSD per node  
✔ Separate network  
✔ SSD for DB/WAL  
✔ Monitoring + Alert  
✔ Regular snapshot  
✔ Periodic health audit

---
