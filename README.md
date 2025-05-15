# 🚀 Dive into Ceph: Your Comprehensive Learning Hub!

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![GitHub Contributors](https://img.shields.io/github/contributors/SumonPaul18/ceph-storage)](https://github.com/SumonPaul18/ceph-storage/graphs/contributors)
[![GitHub Issues](https://img.shields.io/github/issues/SumonPaul18/ceph-storage)](https://github.com/SumonPaul18/ceph-storage/issues)
[![GitHub Pull Requests](https://img.shields.io/badge/pulls-0-.svg)](https://github.com/SumonPaul18/ceph-storage/pulls)

---

## 📚 Table of Contents

- [📖 What is Ceph?](#-what-is-ceph)
- [✨ Features](#-features)
- [🧱 Ceph Architecture](#-ceph-architecture)
- [⚙️ Installation](#-installation-methods)
- [🛠️ Configuration](#--configuration--management)
- [💻 Common Ceph Commands](#-common-ceph-commands)
- [📊 Monitoring](#-monitoring)
- [📦 Use Cases](#-use-cases)
- [📘 Tutorials](#-tutorials)
- [📌 FAQs](#-faqs)
- [🤝 Contributing](#-contributing)
- [📄 License](#-license)

---

## 📖 What is Ceph?

**Ceph** is a powerful, scalable, and reliable distributed storage system designed to provide:
- **Block storage** (RBD)
- **Object storage** (RGW/S3 compatible)
- **File system** (CephFS)

It is fault-tolerant, self-healing, self-managing, and ideal for modern cloud-native infrastructure.

---

## ✨ Features

| Feature                  | Description                                                                 |
|--------------------------|-----------------------------------------------------------------------------|
| 🔄 Scalability            | Grows from GBs to multi-PB seamlessly                                      |
| 🔧 Unified Storage        | Block, Object, and File storage in one platform                            |
| 🛠 Self-healing            | Automatically recovers from hardware or disk failures                      |
| 🧠 Intelligent Clustering | Uses CRUSH algorithm for data placement without bottlenecks                |
| 🔐 High Availability      | No single point of failure, supports replication and erasure coding        |
| 💻 Open Source            | Backed by a large community and enterprise-ready (Red Hat Ceph, etc.)      |

---

## 🧱 Ceph Architecture

Ceph’s architecture consists of the following components:

| Component | Description |
|----------|-------------|
| **MON (Monitor)** | Maintains cluster map and health |
| **MGR (Manager)** | Provides dashboard and monitoring |
| **OSD (Object Storage Daemon)** | Stores actual data on disk |
| **MDS (Metadata Server)** | Manages metadata for CephFS |
| **Client Interfaces** | RBD, RGW (S3), CephFS for applications |

📖 [Detailed Architecture Guide →](./architecture/overview.md)

---

## 🚀 Installation Methods

You can deploy Ceph in multiple ways. We provide detailed, step-by-step guides for each:

1. 🐳 [Install Ceph using Docker Compose](./installation/docker-compose.md) – Ideal for testing
2. 🧰 [Install on Bare Metal (Manual Setup)](./installation/bare-metal.md) – Full control
3. 🔧 [Deploy using Cephadm (Recommended)](./installation/cephadm.md) – Production-ready and simple

📌 Minimum Requirements:
- Linux OS (Ubuntu/CentOS)
- 2+ CPUs, 4GB+ RAM per node
- SSD/HDD for OSDs
- Proper network configuration

---

## ⚙️ Configuration & Management

Ceph is highly configurable. This section covers essential configurations:

- [📄 Understanding `ceph.conf`](./configuration/ceph.conf.md)
- [🏗 Pools & Replication](./configuration/pools.md)
- [🧠 CRUSH Map and Data Placement](./configuration/crush-map.md)

> Each topic includes examples, diagrams, and best practices.

---

## 💻 Common Ceph Commands

- [Ceph CLI Basics](./commands/ceph-cli.md)
- [Troubleshooting Tips](./commands/troubleshooting.md)

Examples:
```bash
ceph status
ceph osd tree
ceph df
ceph health detail

