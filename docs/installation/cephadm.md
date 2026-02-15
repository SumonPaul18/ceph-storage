# 🚀 Ceph Installation using cephadm (Recommended)

This guide provides a **step-by-step walkthrough** to deploy a Ceph cluster using `cephadm`, the modern deployment tool introduced in Ceph Octopus and recommended in all later releases.

---

## 📌 Table of Contents

- [⚙️ Prerequisites](#️-prerequisites)
- [📡 Network & Hostname Setup](#-network--hostname-setup)
- [📥 Install cephadm](#-install-cephadm)
- [🚀 Bootstrap the Ceph Cluster](#-bootstrap-the-ceph-cluster)
- [🧩 Adding Hosts to the Cluster](#-adding-hosts-to-the-cluster)
- [🧱 Deploy OSDs](#-deploy-osds)
- [📊 Enable Dashboard](#-enable-dashboard)
- [🔐 Authentication](#-authentication)
- [📂 Directory Layout](#-directory-layout)
- [🧰 Troubleshooting](#-troubleshooting)
- [🔗 References](#-references)

---

## ⚙️ Prerequisites

| Requirement | Minimum |
|-------------|---------|
| OS          | Ubuntu 20.04 or 22.04 LTS |
| Memory      | 4 GB+ per node |
| CPU         | Dual-core |
| Network     | 1 Gbps or more |
| Access      | Passwordless SSH from bootstrap node to all others |
| Container   | Podman (preferred) or Docker |

---

## 📡 Network & Hostname Setup

Ensure all nodes have:

- Static IP addresses or reserved DHCP leases
- FQDN or resolvable hostnames across nodes

Edit `/etc/hosts`:

```bash
192.168.1.10 ceph-bootstrap
192.168.1.11 ceph-node1
192.168.1.12 ceph-node2
````

Enable passwordless SSH from bootstrap:

```bash
ssh-keygen
ssh-copy-id ceph-node1
ssh-copy-id ceph-node2
```

---

## 📥 Install cephadm

On the **bootstrap node**:

```bash
sudo apt update
sudo apt install -y curl lsb-release gnupg

curl -fsSL https://download.ceph.com/keys/release.asc | gpg --dearmor -o /usr/share/keyrings/ceph.gpg
echo deb [signed-by=/usr/share/keyrings/ceph.gpg] https://download.ceph.com/debian-reef $(lsb_release -sc) main \
  | sudo tee /etc/apt/sources.list.d/ceph.list

sudo apt update
sudo apt install -y cephadm
```

Check version:

```bash
cephadm --version
```

---

## 🚀 Bootstrap the Ceph Cluster

```bash
sudo cephadm bootstrap \
  --mon-ip 192.168.1.10 \
  --initial-dashboard-user admin \
  --initial-dashboard-password admin123
```

This command:

* Bootstraps a monitor and manager
* Creates a new cluster
* Deploys the dashboard
* Creates SSH keys in `/etc/ceph`

Verify cluster status:

```bash
ceph -s
```

---

## 🧩 Adding Hosts to the Cluster

### Step 1: Copy SSH key

```bash
ceph cephadm get-pub-key > ceph.pub
ssh-copy-id -f -i ceph.pub root@ceph-node1
ssh-copy-id -f -i ceph.pub root@ceph-node2
```

### Step 2: Add nodes

```bash
ceph orch host add ceph-node1
ceph orch host add ceph-node2
```

List hosts:

```bash
ceph orch host ls
```

---

## 🧱 Deploy OSDs

### Step 1: Identify available disks

```bash
ceph orch device ls
```

### Step 2: Deploy OSDs automatically

```bash
ceph orch apply osd --all-available-devices
```

Or for a specific host and disk:

```bash
ceph orch daemon add osd ceph-node1:/dev/sdb
```

---

## 📊 Enable Dashboard

If not already enabled:

```bash
ceph mgr module enable dashboard
ceph dashboard set-login-credentials admin admin123
ceph mgr services
```

Access dashboard:

```
https://<MON-IP>:8443
```

---

## 🔐 Authentication

List authentication keys:

```bash
ceph auth list
```

To create a user:

```bash
ceph auth get-or-create client.myuser mon 'allow r' osd 'allow rw pool=my-pool'
```

---

## 📂 Directory Layout

Ceph stores configuration and service info in:

```bash
/etc/ceph/
└── ceph.conf
└── ceph.client.admin.keyring

/var/lib/ceph/
└── mon/
└── mgr/
└── osd/
└── crash/
```

---

## 🧰 Troubleshooting

| Issue                 | Solution                                         |
| --------------------- | ------------------------------------------------ |
| Dashboard not working | Check `ceph mgr services` for URL                |
| Cannot add OSD        | Use `ceph orch device zap` to clean device       |
| Time drift            | Ensure NTP/Chrony is installed                   |
| SSH key not working   | Use `ceph cephadm get-pub-key` again and re-copy |

---

## 🔗 References

* [Official Cephadm Docs](https://docs.ceph.com/en/latest/cephadm/)
* [Ceph Dashboard](https://docs.ceph.com/en/latest/mgr/dashboard/)
* [Ceph Monitoring](https://docs.ceph.com/en/latest/monitoring/)

---

### ✅ Bonus Tips

- `cephadm` is fully container-based, so it's future-proof and great for production.
- You can create custom services like RGW, NFS, iSCSI using `ceph orch apply`.

---
