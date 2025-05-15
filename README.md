# 🚀 Dive into Ceph: Your Comprehensive Learning Hub! 🚀

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![GitHub Contributors](https://img.shields.io/github/contributors/SumonPaul18/ceph-storage)](https://github.com/SumonPaul18/ceph-storage/graphs/contributors)
[![GitHub Issues](https://img.shields.io/github/issues/SumonPaul18/ceph-storage)](https://github.com/SumonPaul18/ceph-storage/issues)
[![GitHub Pull Requests](https://img.shields.io/github/pulls/SumonPaul18/ceph-storage)](https://github.com/SumonPaul18/ceph-storage/pulls)

---

## 📚 Table of Contents

- [📖 What is Ceph?](#-what-is-ceph)
- [✨ Features](#-features)
- [🧱 Ceph Architecture](#-ceph-architecture)
- [⚙️ Installation](#-installation)
- [🛠️ Configuration](#-configuration)
- [💻 Common Ceph Commands](#-common-ceph-commands)
- [📊 Monitoring](#-monitoring)
- [📦 Use Cases](#-use-cases)
- [📘 Tutorials](#-tutorials)
- [📌 FAQs](#-faqs)
- [🤝 Contributing](#-contributing)
- [📄 License](#-license)

---

## 📖 What is Ceph?

**Ceph** is a free and open-source software storage platform that implements object storage on a single distributed computer cluster, and provides interfaces for object-, block- and file-level storage.

---

## ✨ Features

- 🧠 Self-healing & Self-managing
- 🚀 High Scalability (from GBs to PBs)
- 💡 No single point of failure
- 🔒 Strong consistency and reliability
- ⚙️ Unified storage: block, object, and file
- 🌐 Open source and vendor-neutral

---

## 🧱 Ceph Architecture

Ceph has several core components:
- **Monitors (MON):** Maintain cluster maps
- **Managers (MGR):** Monitoring & interfaces
- **Object Storage Daemons (OSD):** Store actual data
- **Metadata Servers (MDS):** File system metadata (only for CephFS)
- **Clients:** Access Ceph through RBD, RGW, or CephFS

→ [Read more](./architecture/overview.md)

---

## ⚙️ Installation

Choose your preferred method:

1. 🐳 [Docker Compose Setup](./installation/docker-compose.md)
2. 💻 [Bare Metal Installation](./installation/bare-metal.md)
3. 🔧 [Cephadm Deployment](./installation/cephadm.md)

> Make sure your system meets the prerequisites like kernel version, disk size, memory, and network setup.

---

## 🛠️ Configuration

- [ceph.conf Explained](./configuration/ceph.conf.md)
- [CRUSH Map & Topology](./configuration/crush-map.md)
- [Creating & Managing Pools](./configuration/pools.md)

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

