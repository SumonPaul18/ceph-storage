# 🐳 Ceph Installation using Docker Compose

This guide helps you set up a lightweight Ceph cluster using Docker Compose. It's ideal for learning, development, and proof-of-concept (PoC) environments. Not recommended for production.

---

## 📌 Table of Contents

- [⚙️ Prerequisites](#️-prerequisites)
- [📁 Directory Structure](#-directory-structure)
- [🧾 Configuration Files](#-configuration-files)
- [📦 Docker Compose File](#-docker-compose-file)
- [🚀 Running the Cluster](#-running-the-cluster)
- [🔍 Verifying the Setup](#-verifying-the-setup)
- [📂 Ceph Data Volumes](#-ceph-data-volumes)
- [🛑 Stopping & Cleaning Up](#-stopping--cleaning-up)
- [🧠 Notes & Troubleshooting](#-notes--troubleshooting)
- [🔗 References](#-references)

---

## ⚙️ Prerequisites

Before you begin, make sure the following are installed:

- Docker
- Docker Compose v2+
- 8 GB+ RAM and 3 vCPUs (recommended)
- Internet access for pulling Ceph images

Install Docker Compose (if not already):

```bash
sudo apt install docker-compose

---

## 📁 Directory Structure

```bash
ceph-docker/
├── docker-compose.yml
├── ceph.conf
├── bootstrap.sh
├── data/
│   ├── mon/
│   ├── osd/
│   └── mgr/
```

---

## 🧾 Configuration Files

### ceph.conf

This is a basic Ceph configuration file for MON/OSD nodes:

```ini
[global]
fsid = e15bb3da-d2f0-4f2b-93d2-3aa4bfc3b2c1
mon_initial_members = mon
mon_host = 192.168.1.10
public network = 192.168.1.0/24
cluster network = 192.168.1.0/24
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
osd pool default size = 1
```

Replace `192.168.1.10` and network range with your actual host IP and subnet.

---

## 📦 Docker Compose File

### docker-compose.yml

```yaml
version: '3'

services:
  mon:
    image: ceph/ceph:v17
    container_name: ceph-mon
    environment:
      - MON_IP=192.168.1.10
      - CEPH_PUBLIC_NETWORK=192.168.1.0/24
    volumes:
      - ./ceph.conf:/etc/ceph/ceph.conf
      - ./data/mon:/var/lib/ceph/mon
    network_mode: "host"
    privileged: true
    command: mon

  mgr:
    image: ceph/ceph:v17
    container_name: ceph-mgr
    volumes:
      - ./ceph.conf:/etc/ceph/ceph.conf
      - ./data/mgr:/var/lib/ceph/mgr
    network_mode: "host"
    privileged: true
    depends_on:
      - mon
    command: mgr

  osd:
    image: ceph/ceph:v17
    container_name: ceph-osd
    volumes:
      - ./ceph.conf:/etc/ceph/ceph.conf
      - ./data/osd:/var/lib/ceph/osd
    network_mode: "host"
    privileged: true
    depends_on:
      - mon
    command: osd
```

---

## 🚀 Running the Cluster

1. Clone this repo:

```bash
git clone https://github.com/<your-username>/ceph-docker.git
cd ceph-docker
```

2. Launch the cluster:

```bash
docker compose up -d
```

---

## 🔍 Verifying the Setup

After all containers are up, run:

```bash
docker exec -it ceph-mon ceph -s
```

Expected output:

```
cluster:
  id: e15bb3da-d2f0-4f2b-93d2-3aa4bfc3b2c1
  health: HEALTH_OK
services:
  mon: 1 daemons, quorum mon (age 2m)
  mgr: active (age 1m)
  osd: 1 osds: 1 up (since 1m), 1 in (since 1m)
data:
  pools:   1 pools, 64 pgs
  objects: 0 objects, 0 B
```

---

## 📂 Ceph Data Volumes

To persist Ceph data, you can mount local disks or folders:

```yaml
volumes:
  - ./data/osd:/var/lib/ceph/osd
  - ./data/mon:/var/lib/ceph/mon
  - ./data/mgr:/var/lib/ceph/mgr
```

For better performance, use separate block devices or bind mounts.

---

## 🛑 Stopping & Cleaning Up

To stop the cluster:

```bash
docker compose down
```

To remove all data:

```bash
sudo rm -rf ./data/*
```

---

## 🧠 Notes & Troubleshooting

* If containers fail to start, ensure your host network is correct and ports aren't in use.
* Use `docker logs <container>` to debug.
* Ceph requires time sync, ensure `chronyd` or `ntpd` is active.
* Avoid `network_mode: host` if you're running multiple clusters or need isolation.

---

## 🔗 References

* [Official Ceph Docker Repo](https://github.com/ceph/ceph)
* [Ceph Documentation](https://docs.ceph.com/en/latest/)
* [Docker Hub - ceph/ceph](https://hub.docker.com/r/ceph/ceph)

```

---


