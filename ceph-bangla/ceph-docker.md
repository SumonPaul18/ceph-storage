# 🐳 **Docker দিয়ে Ceph Cluster চালানো (Lab/Test Purpose Only)**

> ⚠️ Docker-এ Ceph production এ use করা হয় না। এটি শুধুমাত্র educational/testing এর জন্য।

### 🚀Docker Compose দিয়ে Ceph Storage Setup (Step-by-Step)

#### 🧱 Docker Compose দিয়ে Ceph চালাতে চাইলে: (via [Ceph-Docker](https://github.com/ceph/ceph-docker))

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

এখানে **Docker Compose দিয়ে Ceph ইনস্টলেশন ও কনফিগারেশন** এর পূর্ণ গাইড ধাপে ধাপে — প্রাথমিক থেকে অ্যাডভান্সড পর্যন্ত।

---
