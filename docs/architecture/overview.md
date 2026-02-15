# 🧠 Ceph Architecture Overview

Ceph is a **distributed storage system** designed for excellent performance, reliability, and scalability. It provides **object, block, and file storage** in a unified system that is self-healing, self-managing, and highly available.

---

## 📌 Table of Contents

- [🎯 Ceph Architecture Goals](#-ceph-architecture-goals)
- [📐 High-Level Architecture Diagram](#-high-level-architecture-diagram)
- [🔧 Core Components](#-core-components)
  - [1. MON (Monitor)](#1-mon-monitor)
  - [2. MGR (Manager)](#2-mgr-manager)
  - [3. OSD (Object Storage Daemon)](#3-osd-object-storage-daemon)
  - [4. MDS (Metadata Server)](#4-mds-metadata-server)
  - [5. RGW (RADOS Gateway)](#5-rgw-rados-gateway)
- [💾 Storage Interfaces](#-storage-interfaces)
- [⚙️ CRUSH Algorithm](#️-crush-algorithm)
- [📦 Data Flow Summary](#-data-flow-summary)
- [📊 Ceph Cluster Example](#-ceph-cluster-example)
- [📚 References](#-references)

---

## 🎯 Ceph Architecture Goals

- **Scalability**: From a few nodes to thousands
- **Fault Tolerance**: Self-healing with replication/erasure coding
- **Unified Storage**: Object, Block, and File
- **Automation**: Self-managing and self-balancing
- **Open Source**: 100% community-driven

---

## 📐 High-Level Architecture Diagram

![Ceph Architecture](https://raw.githubusercontent.com/ceph/ceph/master/doc/images/ceph-architecture.png)

> _Diagram Source: Official Ceph Documentation_

---

## 🔧 Core Components

### 1. MON (Monitor)

- Maintains cluster **map & state**
- Requires **quorum** to function
- Typically deployed as an **odd number (3 or 5)** for HA

```bash
ceph mon stat
````

---

### 2. MGR (Manager)

* Works with MON to manage and monitor cluster
* Provides:

  * **Dashboard**
  * **Orchestrator integration**
  * **Metrics for Prometheus**
* Usually 2 MGRs (1 active, 1 standby)

```bash
ceph mgr stat
```

---

### 3. OSD (Object Storage Daemon)

* Stores actual data on disk
* Handles replication, recovery, and rebalancing
* Minimum **3 OSDs required**

```bash
ceph osd tree
```

---

### 4. MDS (Metadata Server)

* Needed only for **CephFS**
* Handles filesystem metadata (not user data)
* At least **2 MDS** for HA in CephFS

```bash
ceph mds stat
```

---

### 5. RGW (RADOS Gateway)

* Provides **S3/Swift-compatible API**
* Enables object storage for applications
* RESTful interface over HTTP/S

```bash
radosgw-admin user list
```

---

## 💾 Storage Interfaces

| Interface | Description                    | Use Case            |
| --------- | ------------------------------ | ------------------- |
| RADOS     | Core object storage engine     | Internal Ceph use   |
| RBD       | Block device over RADOS        | VMs, Databases      |
| CephFS    | POSIX-compliant file system    | Shared file systems |
| RGW       | REST API (S3/Swift compatible) | Cloud-native apps   |

---

## ⚙️ CRUSH Algorithm

**CRUSH** (Controlled Replication Under Scalable Hashing) is a **placement algorithm** used by Ceph to determine:

* **Where to store data**
* **How to distribute replicas**
* **How to rebalance on failure**

Advantages:

* No central lookup table
* Fully decentralized
* Fast and scalable

![CRUSH Map](https://docs.ceph.com/en/latest/_images/crush-map.png)

---

## 📦 Data Flow Summary

1. **Client** sends read/write request.
2. Uses **CRUSH Map** to identify target OSD.
3. Request routed directly to OSD – no bottlenecks.
4. Data is replicated or EC-encoded and stored.

---

## 📊 Ceph Cluster Example

Example of a small production-grade Ceph cluster:

| Node       | MON | MGR | OSD | MDS | RGW |
| ---------- | --- | --- | --- | --- | --- |
| ceph-node1 | ✅   | ✅   | ✅   | ✅   | ✅   |
| ceph-node2 | ✅   | 🔄  | ✅   | 🔄  | ✅   |
| ceph-node3 | ✅   | ❌   | ✅   | ❌   | ❌   |

---

## 📚 References

* [Ceph Official Docs](https://docs.ceph.com/en/latest/)
* [Ceph CRUSH Map](https://docs.ceph.com/en/latest/rados/operations/crush-map/)
* [Ceph YouTube Channel](https://www.youtube.com/c/CephProject)
* [Ceph GitHub Repo](https://github.com/ceph/ceph)

---



