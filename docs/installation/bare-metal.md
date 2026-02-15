# 🖥️ Ceph Installation on Bare-Metal (Ubuntu)

This guide will walk you through the step-by-step process of installing and configuring a Ceph cluster on **bare-metal Ubuntu servers**. This setup is suitable for learning, lab environments, and small-scale production.

---

## 📌 Table of Contents

- [⚙️ Requirements](#️-requirements)
- [📡 Network Setup](#-network-setup)
- [🔧 Prepare All Nodes](#-prepare-all-nodes)
- [🛠️ Install Ceph Packages](#️-install-ceph-packages)
- [🚀 Create the Cluster](#-create-the-cluster)
- [📥 Add OSDs (Storage Devices)](#-add-osds-storage-devices)
- [🧠 Monitor & Dashboard](#-monitor--dashboard)
- [🔐 Authentication](#-authentication)
- [📂 Directory Structure](#-directory-structure)
- [🛑 Stopping & Restarting](#-stopping--restarting)
- [🧰 Troubleshooting](#-troubleshooting)
- [🔗 References](#-references)

---

## ⚙️ Requirements

| Component | Minimum |
|----------|---------|
| OS       | Ubuntu 20.04 or 22.04 LTS |
| RAM      | 4 GB per node (8 GB+ recommended) |
| CPU      | Dual-core minimum |
| Disk     | 1 SSD or HDD for Ceph OSD |
| Network  | At least 1 Gbps Ethernet |
| Access   | Passwordless SSH between all nodes |

You'll need **at least 3 nodes**:
- `ceph-mon` (Monitor + Manager)
- `ceph-osd1` (OSD)
- `ceph-osd2` (OSD)

---

## 📡 Network Setup

Ensure all nodes can ping each other by hostname.

**Edit `/etc/hosts` on all nodes:**

```

192.168.1.10 ceph-mon
192.168.1.11 ceph-osd1
192.168.1.12 ceph-osd2

````

**Enable passwordless SSH from mon node:**

```bash
ssh-keygen
ssh-copy-id ceph-osd1
ssh-copy-id ceph-osd2
````

---

## 🔧 Prepare All Nodes

Update packages and install dependencies:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ntp curl lvm2 chrony
sudo systemctl enable --now chrony
```

Ensure time is synchronized across nodes.

---

## 🛠️ Install Ceph Packages

### Step 1: Add Ceph Repo

```bash
sudo apt install -y software-properties-common
sudo add-apt-repository ppa:ceph/octopus  # or pacific/quincy/recent
sudo apt update
```

### Step 2: Install Ceph

```bash
sudo apt install -y ceph-deploy ceph-common
```

---

## 🚀 Create the Cluster

Only on the **Monitor node** (`ceph-mon`):

### Step 1: Create a directory for cluster setup

```bash
mkdir ~/ceph-cluster && cd ~/ceph-cluster
```

### Step 2: Create a new cluster

```bash
ceph-deploy new ceph-mon
```

### Step 3: Install Ceph on all nodes

```bash
ceph-deploy install ceph-mon ceph-osd1 ceph-osd2
```

### Step 4: Deploy MON and MGR daemons

```bash
ceph-deploy mon create-initial
```

---

## 📥 Add OSDs (Storage Devices)

### Step 1: Prepare a clean disk on each OSD node

Example (on `ceph-osd1` and `ceph-osd2`):

```bash
lsblk     # Find your unused disk, e.g., /dev/sdb
```

Make sure it's not mounted and has no partitions.

### Step 2: Create OSDs

```bash
ceph-deploy osd create --data /dev/sdb ceph-osd1
ceph-deploy osd create --data /dev/sdb ceph-osd2
```

---

## 🧠 Monitor & Dashboard

Enable Ceph Dashboard:

```bash
ceph mgr module enable dashboard
ceph dashboard create-self-signed-cert
ceph dashboard set-login-credentials admin admin123
```

Access the dashboard via:
➡️ `https://<mon-ip>:8443`

---

## 🔐 Authentication

Enable and check Ceph authentication (enabled by default):

```bash
ceph auth list
```

You’ll see keys for MON, MGR, OSD, etc.

---

## 📂 Directory Structure

```bash
~/ceph-cluster/
├── ceph.conf         # Main config
├── ceph.client.admin.keyring
├── mon.keyring
└── log/              # Logs
```

---

## 🛑 Stopping & Restarting

To restart Ceph services:

```bash
sudo systemctl restart ceph.target
```

To stop:

```bash
sudo systemctl stop ceph.target
```

---

## 🧰 Troubleshooting

| Issue                      | Solution                        |
| -------------------------- | ------------------------------- |
| Monitor not forming quorum | Check hostname/IP mismatch      |
| OSD not up                 | Check disk permissions & format |
| Time drift                 | Use `chronyd` or `ntpd`         |
| Network unreachable        | Check `/etc/hosts` and firewall |

---

## 🔗 References

* [Ceph Docs](https://docs.ceph.com/en/latest/)
* [Ceph Deploy Guide](https://docs.ceph.com/en/latest/install/)
* [Ceph Dashboard](https://docs.ceph.com/en/latest/mgr/dashboard/)

---
