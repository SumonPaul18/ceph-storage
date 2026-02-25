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

## 🎉 এখন একটি চলমান Ceph Cluster তৈরি হয়েছে!

---

# 🧪 **PART 1: Single Node Ceph Cluster Setup (Lab/Test Purpose Only)**

> ⚠️ এটি শুধুমাত্র Learning/Test এর জন্য। Production-এ একাধিক node ব্যবহার করা বাধ্যতামূলক।

---

### 🖥️ System Requirements:

* OS: Ubuntu 20.04+
* RAM: 4 GB+
* Disk: একাধিক ডিভাইস (একটি OS এর জন্য, অন্যটি OSD-এর জন্য, e.g. `/dev/sdb`)
* Hostname: `ceph-node`

---

### 🔧 Step-by-Step:

#### 🔹 Step 1: Hostname & `/etc/hosts` ঠিক করুন

```bash
sudo hostnamectl set-hostname ceph-node
sudo nano /etc/hosts
```

```txt
127.0.0.1 ceph-node
```

---

#### 🔹 Step 2: Ceph Tools ইনস্টল করুন

```bash
sudo apt update
sudo apt install -y ceph-deploy ceph-common
```

---

#### 🔹 Step 3: Ceph Cluster ফোল্ডার তৈরি

```bash
mkdir ceph-cluster && cd ceph-cluster
```

---

#### 🔹 Step 4: New Cluster তৈরি করুন

```bash
ceph-deploy new ceph-node
```

---

#### 🔹 Step 5: Ceph ইনস্টল করুন

```bash
ceph-deploy install ceph-node
```

---

#### 🔹 Step 6: Monitor এবং Manager তৈরি

```bash
ceph-deploy mon create-initial
ceph-deploy mgr create ceph-node
```

---

#### 🔹 Step 7: Admin Key কপি করুন

```bash
ceph-deploy admin ceph-node
```

---

#### 🔹 Step 8: OSD তৈরি (ধরি `/dev/sdb` আছে)

```bash
ceph-deploy osd create --data /dev/sdb ceph-node
```

---

#### 🔹 Step 9: ক্লাস্টার চেক করুন

```bash
ceph -s
```

---

✅ **Single Node Ceph Ready!**

---

# 🧪 **PART 2: Two Node Ceph Cluster Setup**

> 📌 Example:

* Node1: `mon1` (Monitor + MGR)
* Node2: `osd1` (OSD only)

---

### 🔹 Step 1: Hosts Setup

**On both:**

```bash
# Set hostname
hostnamectl set-hostname mon1      # on node1
hostnamectl set-hostname osd1      # on node2

# Update /etc/hosts on both
192.168.56.10 mon1
192.168.56.11 osd1
```

---

### 🔹 Step 2: Install Ceph-deploy on `mon1`

```bash
sudo apt install ceph-deploy -y
```

---

### 🔹 Step 3: SSH Key Setup from `mon1`

```bash
ssh-keygen
ssh-copy-id user@osd1
```

---

### 🔹 Step 4: Cluster তৈরি

```bash
mkdir ceph-cluster && cd ceph-cluster
ceph-deploy new mon1
```

---

### 🔹 Step 5: Install Ceph on all

```bash
ceph-deploy install mon1 osd1
```

---

### 🔹 Step 6: Create MON & MGR

```bash
ceph-deploy mon create-initial
ceph-deploy mgr create mon1
```

---

### 🔹 Step 7: Admin key push

```bash
ceph-deploy admin mon1 osd1
```

---

### 🔹 Step 8: Create OSD on osd1

```bash
ceph-deploy osd create --data /dev/sdb osd1
```

---

### 🔹 Step 9: Status Check

```bash
ceph -s
```

✅ Two node Ceph cluster তৈরি ✅

---

# 🧪 **PART 3: Three Node Production-Grade Ceph Cluster Setup**

| Hostname | Role      | IP            |
| -------- | --------- | ------------- |
| mon1     | MON + MGR | 192.168.56.10 |
| osd1     | OSD       | 192.168.56.11 |
| osd2     | OSD       | 192.168.56.12 |

Same steps as above — শুধু `ceph-deploy install mon1 osd1 osd2` এবং `osd create` তিনটিতে করবেন।

---

# 🐳 **PART 4: Docker দিয়ে Ceph Cluster চালানো (Lab/Test Purpose Only)**

> ⚠️ Docker-এ Ceph production এ use করা হয় না। এটি শুধুমাত্র educational/testing এর জন্য।

---

### 🧱 Docker Compose দিয়ে Ceph চালাতে চাইলে: (via [Ceph-Docker](https://github.com/ceph/ceph-docker))

#### 🔹 Step 1: Clone the repo

```bash
git clone https://github.com/ceph/ceph-docker.git
cd ceph-docker
```

#### 🔹 Step 2: Prepare your config

Use sample `docker-compose.yml`:

```yaml
version: '3'
services:
  ceph-mon:
    image: ceph/daemon
    environment:
      - MON_IP=192.168.56.10
      - CEPH_PUBLIC_NETWORK=192.168.56.0/24
      - MON_NAME=mon
    volumes:
      - /etc/ceph:/etc/ceph
      - /var/lib/ceph/:/var/lib/ceph/
    network_mode: host
```

#### 🔹 Step 3: Run Docker Compose

```bash
docker-compose up -d
```

#### 🔹 Step 4: Exec inside container

```bash
docker exec -it ceph-mon bash
ceph -s
```

---

এখানে **Docker Compose দিয়ে Ceph ইনস্টলেশন ও কনফিগারেশন** এর পূর্ণ গাইড ধাপে ধাপে — প্রাথমিক থেকে অ্যাডভান্সড পর্যন্ত।

---

# 🚀 PART 1: Docker Compose দিয়ে Ceph Storage Setup (Step-by-Step)

> ✅ আপনি সহজেই Block Storage (RBD), Object Storage (RGW), এবং File System (CephFS) চালাতে পারবেন।

---

## 🧱 Step 1: System Requirements

* OS: Ubuntu 20.04 / 22.04 (or any Linux distro with Docker)
* RAM: 4–8 GB+ (more for full services)
* Docker: version 20.x+
* Docker Compose: version 1.25+

---

## ⚙️ Step 2: Docker & Docker Compose ইনস্টল করুন

```bash
sudo apt update
sudo apt install -y docker.io docker-compose
sudo systemctl enable --now docker
```

> 🔑 Optional: Add your user to Docker group:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

---

## 📦 Step 3: Create Project Folder

```bash
mkdir ceph-docker && cd ceph-docker
```

---

## 📝 Step 4: Docker Compose ফাইল তৈরি করুন

```bash
nano docker-compose.yml
```

এখানে একটি Minimal Ceph Cluster Compose config:

```yaml
version: '3'

services:
  ceph-mon:
    image: ceph/daemon
    network_mode: "host"
    environment:
      - MON_IP=127.0.0.1
      - CEPH_PUBLIC_NETWORK=127.0.0.1/24
      - MON_NAME=mon
      - CLUSTER=ceph
      - DEBUG=verbose
      - RGW_CIVETWEB_PORT=8080
    volumes:
      - /etc/ceph:/etc/ceph
      - /var/lib/ceph:/var/lib/ceph
    restart: always
```

> ℹ️ এই Compose ফাইলটি Ceph Monitor (MON), Manager (MGR), এবং Object Gateway (RGW) সহ একটি Single Node cluster তৈরি করবে।

---

## ▶️ Step 5: Run the Ceph Container

```bash
docker-compose up -d
```

Check logs:

```bash
docker logs -f ceph-docker_ceph-mon_1
```

---

## 🔍 Step 6: Container-এর মধ্যে গিয়ে Ceph Status চেক করুন

```bash
docker exec -it ceph-docker_ceph-mon_1 bash
ceph -s
```

> ✅ যদি সব ঠিক থাকে, আপনি দেখবেন: `HEALTH_OK`, MON running, OSD না থাকলেও শুরু হয়েছে।

---

# 🧠 PART 2: Add Advanced Components

---

## 🧱 Add OSD (Object Storage Device)

* Ceph OSD চালাতে হলে আপনার একটি আলাদা volume বা block device দরকার।

### 🔧 Example OSD container:

```yaml
  ceph-osd:
    image: ceph/daemon
    network_mode: "host"
    environment:
      - OSD_TYPE=directory
      - OSD_DIRECTORY=/var/lib/ceph/osd/osd1
      - CLUSTER=ceph
    volumes:
      - /etc/ceph:/etc/ceph
      - /var/lib/ceph:/var/lib/ceph
    restart: always
```

> 📁 আপনি চাইলে `/data/osd1` নামে একটি ডিরেক্টরি ব্যবহার করতে পারেন।

```bash
mkdir -p /var/lib/ceph/osd/osd1
```

---

## ☁️ Add RGW (Object Storage Gateway like S3)

RGW চালাতে হলে `RGW_CIVETWEB_PORT` সহ Config দিন:

```yaml
  ceph-rgw:
    image: ceph/daemon
    network_mode: "host"
    environment:
      - RGW_NAME=rgw1
      - RGW_CIVETWEB_PORT=8080
      - CLUSTER=ceph
    volumes:
      - /etc/ceph:/etc/ceph
      - /var/lib/ceph:/var/lib/ceph
    restart: always
```

📦 Access RGW via:

```
http://localhost:8080/
```

---

## 📁 Add CephFS (File System Support)

CephFS চালাতে `ceph-mds` container চালাতে হবে:

```yaml
  ceph-mds:
    image: ceph/daemon
    network_mode: "host"
    environment:
      - CLUSTER=ceph
      - MDS_NAME=mds1
    volumes:
      - /etc/ceph:/etc/ceph
      - /var/lib/ceph:/var/lib/ceph
    restart: always
```

---

## 📊 Add Ceph Dashboard

```yaml
  ceph-mgr:
    image: ceph/daemon
    network_mode: "host"
    environment:
      - CLUSTER=ceph
      - MGR_NAME=mgr
      - ENABLE_CEPH_DASHBOARD=true
    volumes:
      - /etc/ceph:/etc/ceph
      - /var/lib/ceph:/var/lib/ceph
    restart: always
```

Dashboard Access:

```
https://localhost:8443/
```

Login Setup:

```bash
ceph dashboard set-login-credentials admin yourpassword
```

---

# 📘 PART 3: Ceph ব্যবহার – RBD, CephFS, RGW

## 📦 RBD (Block Device):

```bash
ceph osd pool create rbd 128
rbd create mydisk --size 10240
rbd map mydisk
mkfs.ext4 /dev/rbd0
mount /dev/rbd0 /mnt
```

---

## ☁️ RGW (S3 Compatible)

### Create S3 User:

```bash
radosgw-admin user create --uid="testuser" --display-name="Test User"
```

### Get S3 credentials:

```bash
radosgw-admin user info --uid="testuser"
```

---

## 📁 CephFS:

```bash
ceph fs volume create myfs
mount -t ceph 127.0.0.1:6789:/ /mnt -o name=admin,secretfile=/etc/ceph/ceph.client.admin.keyring
```

---

# 🛡️ Security and Tips

| Practice                | Description                  |
| ----------------------- | ---------------------------- |
| Use separate volumes    | Avoid data loss              |
| Enable SSL for RGW      | Use HTTPS                    |
| Monitor with Prometheus | Metrics scraping             |
| Backup config files     | `/etc/ceph`, `/var/lib/ceph` |

---

# ✅ Summary

| Step | What You Did                    |
| ---- | ------------------------------- |
| 1    | Docker & Compose Setup          |
| 2    | Minimal Ceph Cluster তৈরি       |
| 3    | Ceph MON/MGR/RGW চালু           |
| 4    | Advanced Components Add         |
| 5    | RBD, RGW, CephFS ব্যবহার শিখলেন |

---



