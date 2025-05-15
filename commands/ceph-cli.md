# 🧰 Ceph CLI Basics

The Ceph CLI (`ceph`) is your primary interface for managing and monitoring a Ceph cluster. It allows you to interact with MONs, OSDs, MGRs, pools, and more.

---

## 📑 Table of Contents

- [🔍 Cluster Status](#-cluster-status)
- [📦 OSD Operations](#-osd-operations)
- [🧠 MON & MGR](#-mon--mgr)
- [🏗 Pool Management](#-pool-management)
- [🔒 User & Auth](#-user--auth)
- [📁 Configuration](#-configuration)
- [📚 Miscellaneous](#-miscellaneous)

---

## 🔍 Cluster Status

```bash
ceph status                  # Overall health status
ceph df                     # Cluster usage stats
ceph osd df                 # OSD usage and distribution
ceph health detail          # Health warnings in detail
ceph -s                     # Short summary of the cluster
ceph versions               # Shows daemon versions
````

---

## 📦 OSD Operations

```bash
ceph osd tree               # Visual hierarchy of OSDs
ceph osd dump               # OSD map dump
ceph osd down               # List of down OSDs
ceph osd in|out <id>        # Mark OSD in/out of cluster
ceph osd crush reweight     # Manually reweight an OSD

# Example:
ceph osd out 3
ceph osd in 3
```

---

## 🧠 MON & MGR

```bash
ceph mon stat               # MON status
ceph mon metadata           # MON details
ceph mgr stat               # MGR status
ceph mgr module ls          # List enabled/disabled modules
ceph mgr module enable prometheus
```

---

## 🏗 Pool Management

```bash
ceph osd pool ls            # List all pools
ceph osd pool create <name> <pg_num>
ceph osd pool delete <name> <name> --yes-i-really-really-mean-it
ceph osd pool set <pool> size <num>           # Set replication size
ceph osd pool application enable <pool> rbd   # Enable RBD or other services
```

---

## 🔒 User & Auth

```bash
ceph auth list                        # List all keys
ceph auth get <client>               # Get key for specific client
ceph auth get-or-create client.admin mon 'allow *' osd 'allow *' mgr 'allow *'
ceph auth del <client>               # Delete client
```

---

## 📁 Configuration

```bash
ceph config dump                     # Show all configs
ceph config get <daemon> <key>
ceph config set <daemon> <key> <val>
ceph config generate-minimal-conf   # Generate minimal ceph.conf
```

---

## 📚 Miscellaneous

```bash
ceph fs status                      # Check CephFS status
ceph osd perf                       # Show OSD performance stats
ceph quorum_status --format json   # JSON MON quorum details
```

> 🔔 You can also use `ceph --help` for a full command list.

---

📘 **Tip**: Always execute commands as a privileged user or use `sudo` if needed.


---
