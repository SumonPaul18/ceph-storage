# ⚙️ Configuration & Management in Ceph

Ceph is designed to be highly configurable and adaptable to various workloads. This section covers the **essential configurations** required for managing and tuning a Ceph cluster effectively.

---

## 📑 Table of Contents

- [📄 Understanding ceph.conf](#-understanding-cephconf)
- [🏗 Pools & Replication](#-pools--replication)
- [🧠 CRUSH Map & Data Placement](#-crush-map--data-placement)
- [📚 References](#-references)

---

## 📄 Understanding `ceph.conf`

`ceph.conf` is the **central configuration file** for a Ceph cluster. It defines global and per-daemon settings.

### 📁 Typical Location:

```bash
/etc/ceph/ceph.conf
````

### 🧩 Basic Structure:

```ini
[global]
fsid = dbe3a72d-87f6-11ec-a4e5-525400f68554
mon_initial_members = ceph-mon1, ceph-mon2
mon_host = 192.168.0.11,192.168.0.12
public_network = 192.168.0.0/24
cluster_network = 10.0.0.0/24
osd_pool_default_size = 3
osd_pool_default_min_size = 2
osd_crush_chooseleaf_type = 1
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx

[mon.ceph-mon1]
host = ceph-mon1
mon_addr = 192.168.0.11

[osd.0]
host = ceph-osd1
```

### ✅ Best Practices

* Keep at least **3 MONs** for quorum
* Separate **public** and **cluster** networks for performance
* Enable **cephx** for authentication
* Always back up `ceph.conf` before changes

---

## 🏗 Pools & Replication

Ceph stores data in **storage pools**. Each pool can be configured for:

* **Replication** or **Erasure Coding**
* Number of **placement groups (PGs)**
* Application-specific optimizations

### 📦 Create a Pool

```bash
# Replicated Pool
ceph osd pool create mypool 128 128

# Erasure Coded Pool
ceph osd erasure-code-profile set ec-profile \
  k=2 m=1 plugin=jerasure technique=reed_sol_van

ceph osd pool create ec-pool 128 128 erasure ec-profile
```

### 🔁 Set Replication Size

```bash
ceph osd pool set mypool size 3
```

### 📊 PG Count Guidelines

| OSD Count | PGs per Pool |
| --------- | ------------ |
| 3–5       | 128–256      |
| 5–10      | 512          |
| 10+       | 1024+        |

> ℹ️ Formula: `Total PGs ≈ (OSDs × 100) / Pool Replication Size`

### ✅ Best Practices

* Keep replication size ≥ 3 in production
* Use Erasure Coding for archive workloads
* Adjust PG count during low IO periods
* Don’t mix EC and replicated pools without understanding the trade-offs

---

## 🧠 CRUSH Map & Data Placement

The **CRUSH (Controlled Replication Under Scalable Hashing)** algorithm decides how and where data is stored in the cluster — **without a centralized lookup table**.

### 🗺 Components of CRUSH Map

* **Buckets**: logical grouping like host, rack, datacenter
* **Rules**: define how replicas are placed
* **Weights**: control data distribution
* **Types**: like `host`, `rack`, `root`

---

### 🧬 CRUSH Map Hierarchy Example

```
root default
 └─ rack rack1
     ├─ host ceph-node1
     │   └─ osd.0
     ├─ host ceph-node2
     │   └─ osd.1
     └─ host ceph-node3
         └─ osd.2
```

### 🖼 CRUSH Map Diagram

![CRUSH Map Diagram](https://docs.ceph.com/en/latest/_images/crush-map.png)

---

### 🛠 View and Edit CRUSH Map

```bash
# View CRUSH Map
ceph osd crush tree

# Decompile
ceph osd getcrushmap -o crush.map
crushtool -d crush.map -o crush.txt

# Edit crush.txt, then recompile
crushtool -c crush.txt -o newcrush.map
ceph osd setcrushmap -i newcrush.map
```

---

### 🧠 Placement Rules Example

```bash
ceph osd crush rule create-simple rule_replicated default host firstn
ceph osd pool set mypool crush_rule rule_replicated
```

---

### ✅ Best Practices

* Define **failure domains** (host > rack > datacenter)
* Use **host-level replication** to avoid data loss from single host failure
* Review `ceph osd df tree` regularly for balance
* Use CRUSH rules for performance-optimized placement

---

## 📚 References

* [CRUSH Explained](https://docs.ceph.com/en/latest/rados/operations/crush-map/)
* [ceph.conf Guide](https://docs.ceph.com/en/latest/rados/configuration/ceph-conf/)
* [Ceph Pools](https://docs.ceph.com/en/latest/rados/operations/pools/)
* [Red Hat Storage Guide](https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/)

````

---



