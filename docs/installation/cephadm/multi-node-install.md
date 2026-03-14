# 🏗️ Complete Guide: Production Grade & Enterprise Level Ceph Cluster Setup
**Using cephadm on Ubuntu 24.04.4 | Simple English | Real-World Examples**

---

## 📋 Table of Contents
1. [Architecture & Prerequisites](#1-architecture--prerequisites)
2. [Core Components & Terminology](#2-core-components--terminology)
3. [VM/Hardware Specifications](#3-vmhardware-specifications)
4. [Step-by-Step Installation Guide](#4-step-by-step-installation-guide-a-to-z)
5. [Production Maintenance & Troubleshooting](#5-production-maintenance--troubleshooting)
6. [Best Practices Checklist](#6-best-practices-checklist)

---

## 1. Architecture & Prerequisites

### 1.1 High-Level Architecture (Logical View)

```
                    ┌─────────────────────────┐
                    │   Client Applications   │
                    │   (VMs, K8s, Databases) │
                    └────────┬────────────────┘
                             │
              ┌──────────────┴──────────────┐
              │   Public Network (Frontend) │
              │   Port: 6789, 3300, 6800+   │
              └──────────────┬──────────────┘
                             │
    ┌────────────────────────┼────────────────────────┐
    │                        │                        │
┌───▼────┐           ┌────▼─────┐           ┌────▼─────┐
│Node 1  │           │ Node 2   │           │ Node 3   │
│MON+MGR │◄─Cluster─►│ MON+OSD  │◄─Cluster─►│ MON+OSD  │
│+OSD    │  Network  │ +OSD     │  Network  │ +OSD     │
│        │(Backend)  │          │(Backend)  │          │
└────────┘           └──────────┘           └──────────┘
```

### 1.2 Prerequisites Checklist (With Real-World Examples)

| Requirement | Technical Detail | Real-World Example | Why It Matters |
|------------|-----------------|------------------|---------------|
| **Minimum Nodes** | 3 nodes for production | Like a 3-legged stool - if one leg breaks, it still stands. 2 nodes = risky (if 1 fails, system stops). | Ensures high availability. If 1 node fails, 2 nodes maintain Quorum and keep cluster running. |
| **Network Separation** | Public Network + Cluster Network | Like a hospital: Public network = patient entrance, Cluster network = staff-only corridor for critical supplies. | Replication traffic (OSD to OSD) won't slow down client requests. Prevents network congestion. |
| **Time Sync (NTP/Chrony)** | All nodes must have same time (within 100ms) | Like a team meeting: If everyone's watch shows different times, coordination fails. | Ceph uses timestamps for data ordering. Time difference >100ms can break Quorum and cause data inconsistency. |
| **DNS/Host Resolution** | Each node must resolve other node hostnames | Like a phone directory: You need names to match numbers to call colleagues. | cephadm uses hostnames to manage nodes. Wrong resolution = failed deployment. |
| **Disk Preparation** | OSD disks must be empty (no partition, no filesystem) | Like a new notebook: You can't write new notes if pages are already filled with old content. | cephadm will fail to create OSD on disks with existing data. Use `wipefs -a /dev/sdX` to clean. |
| **Container Runtime** | Docker CE v20.10+ or Podman v3.4+ | Like a shipping container port: You need standard containers (Docker) for cephadm to deploy services. | cephadm runs Ceph services inside containers. No runtime = no Ceph. |
| **Firewall Rules** | Open required ports: 6789, 3300, 6800-7300, 8443 | Like building security: Only allow entry through designated doors (ports), block the rest. | Blocked ports = nodes can't communicate = cluster fails to form or replicate data. |
| **Kernel Parameters** | `vm.swappiness=1`, `net.ipv4.ip_forward=1` | Like tuning a car engine: Default settings work, but optimized settings give better performance. | Prevents memory swapping (slows down Ceph) and enables proper network routing. |

### 1.3 Network Architecture (Production Recommended)

```
┌─────────────────────────────────────────┐
│           Physical Network Layout        │
├─────────────────────────────────────────┤
│                                          │
│  ┌─────────────┐     ┌─────────────┐    │
│  │ Public NIC  │     │Cluster NIC  │    │
│  │ 192.168.10.x│     │ 10.10.10.x  │    │
│  │ (Client)    │     │ (Replication)│   │
│  └──────┬──────┘     └──────┬──────┘    │
│         │                   │            │
│  ┌──────▼───────────────────▼──────┐    │
│  │        Ceph Node (Each)         │    │
│  │  • MON/MGR on Public Network    │    │
│  │  • OSD replication on Cluster   │    │
│  │    Network (fast, isolated)     │    │
│  └─────────────────────────────────┘    │
│                                          │
└─────────────────────────────────────────┘
```

**Configuration in `/etc/ceph/ceph.conf`:**
```ini
[global]
public_network = 192.168.10.0/24
cluster_network = 10.10.10.0/24
osd_pool_default_size = 3
osd_pool_default_min_size = 2
```

---

## 2. Core Components & Terminology (With Real-World Examples)

### 2.1 Essential Components Table

| Component | What It Does | Real-World Analogy | Production Impact |
|-----------|-------------|-------------------|------------------|
| **Monitor (MON)** | Maintains cluster state map, handles authentication | Like a library catalog system: It doesn't store books (data), but knows where every book is located. | Minimum 3 MONs for production. If MONs lose Quorum, cluster becomes read-only. |
| **Manager (MGR)** | Collects metrics, runs dashboard, handles plugins | Like a building manager: Monitors electricity, water, security cameras, and provides reports to owners. | Enables monitoring via Grafana/Prometheus. Dashboard for GUI management. |
| **OSD (Object Storage Daemon)** | Stores data on disk, handles replication, recovery | Like a warehouse worker: Receives items, stores them on shelves (disks), and helps find/repair lost items. | One OSD per disk. If disk fails, only that OSD goes down. Cluster auto-recovers from other copies. |
| **Metadata Server (MDS)** | Required only for CephFS (file storage), not for RBD/RGW | Like a file cabinet index: Helps find files by name, not by location. | Not needed for block/object storage. Only deploy if using CephFS. |

### 2.2 Critical Terminology (With Problems & Solutions)

#### 🔹 Quorum

**Definition:** The minimum number of Monitor nodes that must agree on cluster state for the cluster to function. Formula: `(N/2) + 1` where N = total MONs.

**Real-World Example:** 
> Imagine a 5-member board of directors voting on a decision. At least 3 members (50% + 1) must be present to make a valid decision. If only 2 show up, no official decision can be made.

**Common Problem:** 
> "Cluster stuck in `HEALTH_WARN: mon clock skew detected` or `mon map epoch mismatch`"

**Root Cause:** 
> - Time synchronization failure between nodes
> - Network partition isolating some MONs
> - MON process crashed on multiple nodes

**Solution:**
#### 1. Check time sync on all nodes
```bash
chronyc sources -v
```
#### 2. Check MON status
```
cephadm shell -- ceph mon dump
```
#### 3. If one MON is down, restart it
```
cephadm shell -- ceph orch restart mon.ceph-node2
```
#### 4. If multiple MONs down, manually restore quorum (advanced)
> Stop all MONs, start one with --force-quorum, then add others back

**Prevention:** 
- Always use odd number of MONs (3, 5, 7)
- Deploy MONs on separate physical hosts
- Ensure reliable NTP/Chrony on all nodes

---

#### 🔹 CRUSH Map

**Definition:** Algorithm that calculates where data should be stored, without a central lookup table. Clients compute location directly.

**Real-World Example:**
> Like a smart parking system: Instead of asking a guard "where should I park?", your car's GPS calculates the best spot based on rules: "Park in Zone A if electric, Zone B if SUV, avoid rows near exit during rush hour."

**Common Problem:**
> "Data unevenly distributed: Some OSDs 90% full, others 30% full"

**Root Cause:**
> - CRUSH rules not matching physical layout
> - Wrong `pg_num` calculation for pool size
> - New OSDs added but not rebalanced

**Solution:**

#### 1. Check data distribution
```
cephadm shell -- ceph osd df
```
#### 2. View current CRUSH map
```
cephadm shell -- ceph osd crush dump
```
#### 3. Rebalance cluster (automatic, but can trigger manually)
```
cephadm shell -- ceph osd reweight-by-utilization
```
#### 4. For new pools, calculate proper pg_num:
#### Formula: (OSD_count * 100) / replication_factor
#### Example: 6 OSDs, replication 3 → (6*100)/3 = 200 → use 256 (power of 2)
```
cephadm shell -- ceph osd pool create mypool 256 256
```

**Prevention:**
- Plan CRUSH hierarchy before deployment (host → rack → datacenter)
- Use `cephadm shell -- ceph osd pool create <name> <pg_num> <pgp_num>` with correct values
- Monitor utilization regularly

---

#### 🔹 Split Brain

**Definition:** Network failure causes cluster to split into two groups, each believing it is the "real" cluster. Both may accept writes, causing data conflict.

**Real-World Example:**
> Two bank branches lose connection. Both continue processing transactions. When connection restores, same account has two different balances. Which one is correct?

**Common Problem:**
> "Cluster shows `HEALTH_ERR: split brain detected` or data inconsistency after network recovery"

**Root Cause:**
> - Network partition between MON nodes
> - Insufficient Quorum (e.g., 2 MONs in 3-node cluster, both isolated)
> - Misconfigured firewall blocking MON communication

**Solution:**

#### 1. Identify which partition has majority
```
cephadm shell -- ceph mon dump
```
#### 2. On minority partition, STOP all Ceph services to prevent data corruption
```
systemctl stop ceph-mon@*.service
systemctl stop ceph-osd@*.service
```
#### 3. Restore network connectivity

#### 4. On majority partition, verify cluster health
```
cephadm shell -- ceph -s
```
#### 5. If data conflict occurred, manual recovery may be needed (contact Ceph support)


**Prevention:**
- Always maintain odd number of MONs
- Use redundant network paths (bonding, multiple switches)
- Configure proper firewall rules to allow MON communication
- Monitor network health with tools like `ping`, `mtr`, `iperf3`

---

#### 🔹 Placement Groups (PGs)

**Definition:** Logical groups of objects that OSDs manage together. Reduces metadata overhead.

**Real-World Example:**
> Like organizing books in a library: Instead of tracking 10,000 individual books, you track 100 "sections" (PGs). Each section contains ~100 books. Finding a book means finding its section first.

**Common Problem:**
> "PGs in `incomplete` or `down` state" or "Too few PGs per OSD" warning

**Root Cause:**
> - Incorrect `pg_num` during pool creation
> - OSD failure causing PGs to lose required copies
> - Cluster not fully initialized

**Solution:**

#### 1. Check PG status
```
cephadm shell -- ceph pg stat
cephadm shell -- ceph pg dump_stuck
```
#### 2. For stuck PGs, try recovery
```
cephadm shell -- ceph pg repair <pg_id>
```
#### 3. If pool has too few PGs, increase (careful: requires data migration)
```
cephadm shell -- ceph osd pool set mypool pg_num 512
cephadm shell -- ceph osd pool set mypool pgp_num 512
```
#### 4. Wait for rebalancing to complete
```
cephadm shell -- ceph -s  # Check until "active+clean"
```

**Prevention:**
- Calculate PGs before creating pool: [Ceph PG Calculator](https://www.ceph.com/pgcalc/)
- Start with `pg_num = 32` for small clusters, scale up as needed
- Monitor `ceph health detail` for PG warnings

---

#### 🔹 Journal/WAL (Write-Ahead Log)

**Definition:** Fast storage (SSD/NVMe) used to temporarily store writes before flushing to slower HDDs. Improves write performance.

**Real-World Example:**
> Like a restaurant order system: Waiters write orders on small notepads (WAL) instantly, then kitchen staff later transfers them to the main order book (HDD). Customers get fast confirmation.

**Common Problem:**
> "Slow write performance on HDD-based OSDs" or "WAL device full"

**Root Cause:**
> - Using HDDs without dedicated SSD for WAL
> - WAL device too small for write workload
> - WAL device failure

**Solution:**

#### 1. Check OSD performance
```
cephadm shell -- ceph osd perf
```
#### 2. If using HDDs, add dedicated SSD for WAL/DB during OSD creation:
```
cephadm shell -- ceph orch daemon add osd ceph-node1:/dev/sdb:/dev/nvme0n1p1
```
> Format: <data_disk>:<wal_db_device>

#### 3. For existing OSDs, migration is complex - consider replacing with new OSDs

#### 4. Monitor WAL usage
```
cephadm shell -- ceph daemon osd.<id> bluestore allocator stats block
```

**Prevention:**
- For HDD OSDs: Always pair with SSD for WAL/DB (ratio ~1:10, e.g., 100GB SSD for 1TB HDD)
- For NVMe OSDs: No separate WAL needed
- Monitor disk health with `smartctl`

---

## 3. VM/Hardware Specifications (Per Node)

| Resource | Minimum (Lab) | Production (Enterprise) | Notes |
|----------|--------------|------------------------|-------|
| **CPU** | 4 cores | 8+ cores (Xeon/EPYC) | More cores help with encryption, compression, and parallel OSD operations |
| **RAM** | 8 GB | 16+ GB (1GB per OSD + 4GB OS) | OSDs use RAM for caching. Insufficient RAM = slow performance |
| **OS Disk** | 50 GB SSD | 100 GB NVMe (RAID 1) | For Ubuntu + Docker + Ceph containers. RAID 1 for redundancy |
| **Data Disk (OSD)** | 1x 100 GB | 4x 4TB HDD (no RAID) + 1x 480GB SSD for WAL | Ceph handles redundancy. Don't use hardware RAID for OSDs |
| **Network** | 1x 1 GbE | 2x 10 GbE (separated) | Public: client traffic. Cluster: replication traffic. Jumbo frames (MTU 9000) recommended |
| **OS** | Ubuntu 24.04.4 LTS | Ubuntu 24.04.4 LTS (latest kernel) | Keep all nodes on identical OS version |
| **Swap** | Disabled | Disabled (`vm.swappiness=1`) | Swapping slows down Ceph. Use `sudo swapoff -a` and remove from `/etc/fstab` |

---

## 4. Step-by-Step Installation Guide (A-to-Z)

### Prerequisites Check (Run on ALL Nodes: Node1, Node2, Node3)


#### 1. Update system
```
sudo apt update && sudo apt upgrade -y
```
#### 2. Install basic tools
```
sudo apt install -y curl wget gnupg2 lsb-release software-properties-common apt-transport-https
```
#### 3. Set hostname (replace with actual node name)
```
sudo hostnamectl set-hostname ceph-node1
sudo hostnamectl set-hostname ceph-node2
sudo hostnamectl set-hostname ceph-node3
```
#### 4. Configure /etc/hosts (on ALL nodes)
```
sudo nano /etc/hosts
```
#### Add these lines:
```
192.168.10.11  ceph-node1
192.168.10.12  ceph-node2
192.168.10.13  ceph-node3
```


#### 5. Install and configure Chrony (time sync)
```
sudo apt install -y chrony
sudo systemctl enable --now chrony
chronyc sources -v  
```
> Verify: look for '*' next to server

#### 6. Configure firewall (production-safe)
```
sudo ufw allow 22/tcp
sudo ufw allow 6789/tcp
sudo ufw allow 3300/tcp
sudo ufw allow 6800:7300/tcp
sudo ufw allow 8443/tcp
sudo ufw enable
```
> sudo ufw allow 22/tcp       # SSH
> sudo ufw allow 6789/tcp     # Ceph MON
> sudo ufw allow 3300/tcp     # Ceph MGR
> sudo ufw allow 6800:7300/tcp # Ceph OSD
> sudo ufw allow 8443/tcp     # Dashboard
> sudo ufw enable

#### For lab testing only, you can disable:
```
sudo ufw disable
```

---

### Docker Installation (Run on ALL Nodes)


#### 1. Remove old/conflicting Docker packages
```
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do 
    sudo apt remove -y $pkg
done
```
#### 2. Add Docker's official GPG key
```
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```
#### 3. Add Docker repository
```
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
#### 4. Install Docker
```
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
#### 5. Start and enable Docker
```
sudo systemctl enable --now docker
```
#### 6. Add current user to docker group (avoid sudo for docker commands)
```
sudo usermod -aG docker $USER
newgrp docker
```
#### 7. Verify Docker
```
docker --version
```
#### Should show "Hello from Docker!"
```
docker run hello-world  
```

---

### SSH Configuration for cephadm (Run on Bootstrap Node: Node1)


#### 1. Generate SSH key (on Node1)
```
ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/ceph_id_rsa
```
#### 2. Copy public key to other nodes
```
ssh-copy-id -i ~/.ssh/ceph_id_rsa.pub root@ceph-node2
ssh-copy-id -i ~/.ssh/ceph_id_rsa.pub root@ceph-node3
```
> Enter 'yes' and root password when prompted

#### 3. Create SSH config file (on Node1) for easier access
```
nano ~/.ssh/config
```
> Add:
```
Host ceph-node1
    Hostname ceph-node1
    IdentityFile ~/.ssh/ceph_id_rsa
    User root

Host ceph-node2
    Hostname ceph-node2
    IdentityFile ~/.ssh/ceph_id_rsa
    User root

Host ceph-node3
    Hostname ceph-node3
    IdentityFile ~/.ssh/ceph_id_rsa
    User root
```

#### 4. Test passwordless SSH
```
ssh ceph-node2
ssh ceph-node3
```

---

### Cephadm Installation & Bootstrap (Run on Node1 ONLY)


#### 1. Download cephadm tool
```
sudo apt update
sudo apt install -y cephadm
```
#### 2. Verify installation path

```
which cephadm
```
> **Expected Output:** /usr/sbin/cephadm

#### 3. Verify installation
```
cephadm version
```
> **Note:** Sometimes this might show 'UNKNOWN' on some Ubuntu packages, but check with:

```
dpkg -l | grep cephadm
```
#### 3. Bootstrap the cluster (first MON + MGR)
**Initialize the first monitor and manager daemons**

```bash
sudo cephadm bootstrap --mon-ip 192.168.68.180 
```

**After successful bootstrap, save these outputs:**
- Dashboard URL: `https://192.168.10.11:8443`
- Username: `admin`
- Password: `[random password shown in output]`

#### 4. Install `ceph-common`
**Option A: Via cephadm (Recommended)**
```bash
sudo cephadm install ceph-common
```

**Option B: Via APT (Manual)**
```bash
sudo apt update
sudo apt install -y ceph-common
```
> Now you can run `ceph` commands directly without opening the interactive shell.

#### 5. Verify Installation
Check if the package is installed correctly.
```bash
dpkg -l | grep ceph-common
```

#### 6. Run Commands Directly on Host
Now you can run Ceph commands without the shell wrapper.
```bash
ceph -v
ceph -s
```
#### OR 
** Without Install ``ceph-common`` Check cluster status**
```
sudo cephadm shell -- ceph -s
```
> Initial state: HEALTH_WARN (no OSDs yet) is normal

---

### Add Remaining Nodes (Run on Node1)


#### 1. Copy cephadm public SSH key to other nodes
```
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-node2
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-node3
```
#### 2. Add hosts to cluster
```
sudo ceph orch host add ceph-node1
sudo ceph orch host add ceph-node2
```
#### 3. Verify hosts are added
```
sudo cephadm shell -- ceph orch host ls
```
> Should show 3 hosts with status "Ready"

---

### Prepare Data Disks (Run on ALL Nodes)


#### 1. Identify data disks (example: /dev/sdb)
```
lsblk
```
> Confirm disk has no partitions or mount points

#### 2. Clean disk if needed (WARNING: erases all data!)
```
sudo wipefs -a /dev/sdb
```
#### 3. Verify disk is clean
```
lsblk /dev/sdb
```
> Should show no partitions

---

### Deploy OSDs (Run on Node1)


#### Option A: Auto-deploy all available empty disks (simple)
```
sudo cephadm shell -- ceph orch apply osd --all-available-devices
```
#### Option B: Deploy specific disks (production recommended)
```
sudo cephadm shell -- ceph orch daemon add osd ceph-node1:/dev/sdb
sudo cephadm shell -- ceph orch daemon add osd ceph-node2:/dev/sdb
sudo cephadm shell -- ceph orch daemon add osd ceph-node3:/dev/sdb
```
#### For HDD + SSD WAL setup:
#### sudo cephadm shell -- ceph orch daemon add osd ceph-node1:/dev/sdb:/dev/nvme0n1p1

#### 1. Check OSD deployment status
```
sudo cephadm shell -- ceph orch ps --daemon_type osd
```
#### 2. Verify OSD tree
```
sudo cephadm shell -- ceph osd tree
```
> Should show OSDs in "up" and "in" state under each host


---

### Enable Dashboard & Monitoring (Run on Node1)


#### 1. Enable dashboard module
```
sudo cephadm shell -- ceph mgr module enable dashboard
```
#### 2. Create self-signed certificate
```
sudo cephadm shell -- ceph dashboard create-self-signed-cert
```
#### 3. Create admin user with custom password
```
sudo cephadm shell -- ceph dashboard ac-user-create admin YourSecurePassword123 --role administrator
```
#### 4. Get dashboard URL
```
sudo cephadm shell -- ceph mgr services
```
> Look for "dashboard" entry

> 5. Access in browser: https://192.168.10.11:8443
> Login with admin / YourSecurePassword123


---

### Optional: Deploy Prometheus + Grafana (Production Monitoring)

```bash
sudo cephadm shell -- ceph orch apply prometheus
sudo cephadm shell -- ceph orch apply grafana
sudo cephadm shell -- ceph orch apply node_exporter
sudo cephadm shell -- ceph orch apply alertmanager
```
#### Access Grafana: http://<node-ip>:3000 (default admin/admin)


---

## 5. Production Maintenance & Troubleshooting

### Daily Health Checks


#### Overall cluster health
```
sudo cephadm shell -- ceph -s
```
#### Detailed health issues
```
sudo cephadm shell -- ceph health detail
```
#### Storage usage
```
sudo cephadm shell -- ceph df
```
#### OSD status
```
sudo cephadm shell -- ceph osd tree
sudo cephadm shell -- ceph osd stat
```
#### PG status (should be "active+clean")
```
sudo cephadm shell -- ceph pg stat
```

### Common Issues & Solutions

| Issue | Symptoms | Solution |
|-------|----------|----------|
| **OSD down** | `ceph osd tree` shows OSD as "down" | 1. Check disk health: `smartctl -a /dev/sdb`<br>2. Restart OSD: `cephadm shell -- ceph orch restart osd.<id>`<br>3. If disk failed: replace hardware, then `ceph orch apply osd --all-available-devices` |
| **Slow writes** | High latency, low throughput | 1. Check network: `iperf3 -c <node>`<br>2. Verify WAL on SSD for HDD OSDs<br>3. Check for rebalancing: `ceph -s` (avoid heavy writes during rebalance) |
| **MON flapping** | MON repeatedly going up/down | 1. Check time sync: `chronyc sources`<br>2. Check disk space on MON node<br>3. Review logs: `cephadm shell -- ceph logs mon.ceph-node1` |
| **PG stuck** | `ceph pg dump_stuck` shows entries | 1. Try repair: `ceph pg repair <pg_id>`<br>2. If persistent, check OSD health<br>3. As last resort: `ceph pg force_create_pg <pg_id>` (use with caution) |
| **Dashboard not loading** | Browser shows connection error | 1. Check service: `cephadm shell -- ceph mgr services`<br>2. Verify firewall: `sudo ufw status`<br>3. Check certificate: `openssl s_client -connect <ip>:8443` |

### Backup & Recovery


#### Backup critical Ceph configuration
```
sudo tar -czf /backup/ceph-config-$(date +%F).tar.gz /etc/ceph
```
#### Backup monitor keyring (essential for cluster recovery)
```
sudo cp /etc/ceph/ceph.client.admin.keyring /backup/
```
#### Restore procedure (if all MONs lost - advanced):
- 1. Stop all Ceph services on all nodes
- 2. On one node, restore /etc/ceph from backup
- 3. Start MON with --force-quorum
- 4. Add other MONs back one by one
- (Contact Ceph support for complex recovery scenarios)


---

## 6. Best Practices Checklist

### ✅ Pre-Deployment
- [ ] All nodes have identical Ubuntu 24.04.4 installation
- [ ] Hostnames and `/etc/hosts` configured correctly
- [ ] Chrony/NTP synchronized on all nodes (verify with `chronyc sources`)
- [ ] SSH passwordless access from bootstrap node to all others
- [ ] Docker installed and working (`docker run hello-world`)
- [ ] Data disks cleaned with `wipefs -a` and verified empty
- [ ] Firewall rules allow required ports (or disabled for lab)

### ✅ Deployment
- [ ] Bootstrap with correct `--mon-ip` (public network IP)
- [ ] Save dashboard credentials securely
- [ ] Verify all hosts added: `ceph orch host ls`
- [ ] Deploy OSDs with appropriate WAL/DB strategy
- [ ] Wait for cluster to reach `HEALTH_OK` before adding workloads

### ✅ Post-Deployment
- [ ] Enable dashboard and set strong admin password
- [ ] Configure monitoring (Prometheus/Grafana) for alerts
- [ ] Document cluster configuration and recovery procedures
- [ ] Test failure scenarios: stop one OSD, verify auto-recovery
- [ ] Schedule regular backups of `/etc/ceph`

### ✅ Ongoing Maintenance
- [ ] Monitor `ceph -s` daily (or set up alerting)
- [ ] Check disk health monthly with `smartctl`
- [ ] Plan capacity: add OSDs before pools reach 80% full
- [ ] Test recovery procedures quarterly
- [ ] Keep Ubuntu and Ceph updated (use `cephadm upgrade`)

### 🚫 What to Avoid
- ❌ Don't use hardware RAID for OSD disks (Ceph handles redundancy)
- ❌ Don't run OSDs on partitions with other data
- ❌ Don't ignore `HEALTH_WARN` messages (they become `HEALTH_ERR`)
- ❌ Don't change network configuration after deployment without planning
- ❌ Don't use even number of MONs (risk of split brain)

---

### Final Verification

After completing all steps, run this final check:

```bash
sudo cephadm shell -- ceph -s
```

✅ **Expected Output for Healthy Cluster:**
```
  cluster:
    id:     xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum ceph-node1,ceph-node2,ceph-node3
    mgr: ceph-node1.active
    osd: 3 osds: 3 up, 3 in
 
  data:
    pools:   1 pools, 128 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     128 active+clean
```

🎉 **Congratulations!** You now have a Production Grade, Enterprise Level Ceph Cluster running on Ubuntu 24.04.4 using cephadm.

---

> **Need Help?**  
> - Official Docs: https://docs.ceph.com/en/latest/  
> - Community: https://ceph.io/community/  
> - For enterprise support: Contact Red Hat or SUSE (Ceph enterprise distributors)

This guide provides a solid foundation. As your cluster grows, explore advanced topics like erasure coding, multi-site replication, and integration with Kubernetes (Rook/Ceph). Always test changes in a lab environment before applying to production.

Happy clustering! 🐘✨
