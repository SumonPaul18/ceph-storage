# 🚀 Dive into Ceph: Your Comprehensive Learning Hub!

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![GitHub Contributors](https://img.shields.io/github/contributors/SumonPaul18/ceph-storage)](https://github.com/SumonPaul18/ceph-storage/graphs/contributors)
[![GitHub Issues](https://img.shields.io/github/issues/SumonPaul18/ceph-storage)](https://github.com/SumonPaul18/ceph-storage/issues)
[![GitHub Pull Requests](https://img.shields.io/badge/pulls-0-blue.svg)](https://github.com/SumonPaul18/ceph-storage/pulls)

---

## 📚 Table of Contents

- [📖 What is Ceph?](#-what-is-ceph)
- [✨ Features](#-features)
- [🧱 Ceph Architecture](#-ceph-architecture)
- [⚙️ Installation](#-installation-methods)
- [🛠️ Configuration](#%EF%B8%8F-configuration--management)
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

---

## 📊 Monitoring

Monitoring is essential for understanding the performance and health of your Ceph cluster. Ceph provides several built-in and third-party tools for visualization and alerting.

### 🔭 Tools & Integrations

| Tool         | Description                                  |
|--------------|----------------------------------------------|
| Ceph Dashboard | Built-in web-based UI for monitoring cluster |
| Prometheus    | Metrics collection and alerting              |
| Grafana       | Visualize Ceph metrics from Prometheus       |
| Alertmanager  | Define and route alerts                      |

### 🛠 How to Set Up

```bash
# Enable Prometheus module
ceph mgr module enable prometheus

# Export metrics endpoint (default: 9283)
curl http://<ceph-mgr>:9283/metrics
````

📘 For detailed setup: [`/monitoring-and-alerting/overview.md`](./monitoring-and-alerting/overview.md)

---

## 📦 Use Cases

Ceph is highly flexible and supports multiple real-world use cases across various industries:

### 🖼 Object Storage (Ceph RGW)

* S3 & Swift-compatible API
* Ideal for backups, logs, media assets

### 🧠 Block Storage (Ceph RBD)

* Persistent volumes for VMs & containers
* Integration with OpenStack & Kubernetes

### 📁 File Storage (CephFS)

* POSIX-compliant distributed file system
* Suitable for HPC, AI/ML workloads

### 🏢 Enterprise Scenarios

* Hybrid-cloud deployments
* Scalable archival & video surveillance storage

> 💡 **Did you know?** Leading platforms like OpenStack, Kubernetes, and SUSE use Ceph as their backend storage solution.

---

## 📘 Tutorials

Want hands-on experience? Check out these guides:

* 📦 [Deploy Ceph with Docker Compose](./installation/docker-compose.md)
* 🖥️ [Install on Bare Metal](./installation/bare-metal.md)
* 🤖 [Bootstrap with cephadm](./installation/cephadm.md)
* ⚙️ [Tune ceph.conf & CRUSH Map](./configuration-management/ceph.conf.md)
* 📡 [Setup Monitoring & Alerts](./monitoring-and-alerting/overview.md)

🎓 More coming soon: CephFS snapshots, Multi-site RGW, Ceph on Kubernetes, etc.

---

## 📌 FAQs

### ❓ What is the minimum number of nodes for a production Ceph cluster?

At least **3 MONs** and **3 OSDs** are recommended for quorum and redundancy.

---

### ❓ Is Ceph suitable for small-scale or home lab setups?

✅ Yes. With tools like cephadm and containers, even a single-node deployment is possible.

---

### ❓ How does Ceph handle node failure?

Ceph uses data replication (or erasure coding) and automatic rebalancing to maintain availability even after hardware failure.

---

### ❓ Can I use Ceph with Kubernetes?

Absolutely. Ceph integrates well via **Rook**, **CSI drivers**, and **RBD/NFS** backends for dynamic provisioning.

---

## 🤝 Contributing

We welcome contributions from the community! 🚀

### 🛠 How to Contribute

1. Fork this repository
2. Create a new branch: `git checkout -b feature/your-feature`
3. Make your changes
4. Submit a Pull Request

### 📃 Guidelines

* Follow markdown consistency and formatting
* Add relevant diagrams if helpful
* Keep explanations simple and clear
* Reference official documentation when possible

### 🙌 Contributors

Thanks to all the contributors who make this project better every day. 💙

---

## 📄 License

This project is licensed under the **MIT License**.

```
MIT License

Copyright (c) 2025 Suman Pal

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction...
```

📃 [View Full License](./LICENSE)

---

> Made with ❤️ by [Suman Pal](https://github.com/sumanpalx) | Ceph ❤️ Open Source

---
