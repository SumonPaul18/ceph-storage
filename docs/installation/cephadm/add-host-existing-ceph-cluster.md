# Ceph Cluster Expansion & Production Hardening Guide
**Adding a 4th Node to a 3-Node Ceph Cluster (Cephadm/Orchestrator)**

## Table of Contents
1. [Introduction & Architecture Overview](#1-introduction--architecture-overview)
2. [Prerequisites & Environment Preparation](#2-prerequisites--environment-preparation)
3. [Step-by-Step Host Addition](#3-step-by-step-host-addition)
4. [Service Deployment (MON, MGR, OSD)](#4-service-deployment-mon-mgr-osd)
5. [Quorum Logic & High Availability Strategy](#5-quorum-logic--high-availability-strategy)
6. [Verification & Health Checks](#6-verification--health-checks)
7. [Production Maintenance & Best Practices](#7-production-maintenance--best-practices)
8. [Troubleshooting Common Issues](#8-troubleshooting-common-issues)

---

## 1. Introduction & Architecture Overview

### Context
This guide documents the practical process of expanding an existing **3-node Ceph Cluster** (running on Proxmox VE VMs) to a **4-node Cluster** using the modern `cephadm` orchestrator. The goal is to achieve a production-ready state with high availability (HA), proper quorum management, and distributed storage services.

### Why Expand?
*   **Capacity:** Adding more storage disks (OSDs).
*   **Performance:** Distributing I/O load across more nodes.
*   **Resilience:** Increasing fault tolerance (though MON quorum logic must be carefully managed).

### Key Components
*   **Admin Node (`ceph1`):** The node where `ceph.conf` and admin keys reside. All orchestration commands are run from here.
*   **New Node (`ceph5`):** The new host being added to the cluster.
*   **Ceph Orchestrator (`ceph orch`):** The tool that automates the deployment of daemons (MON, MGR, OSD) via containers (Docker/Podman).

---

## 2. Prerequisites & Environment Preparation

Before adding the new host, ensure both the Admin Node and the New Node meet the following requirements.

### 2.1 Network Requirements
*   **Same Subnet:** All nodes must be on the same Layer 2 network (e.g., `192.168.10.0/24`).
*   **Connectivity:** Full bidirectional SSH access between Admin Node and New Node.
*   **Firewall:** For RnD/Internal clusters, disable firewalls. For Production, open specific ports (6789 for MON, 3300/6800 for OSD/MGR).

### 2.2 Operating System Setup (On New Node `ceph5`)

#### Step 1: Set Hostname and IP
Ensure the hostname matches the intended name (`ceph5`) and is resolvable.

**On ceph5 (New Node)**
```bash
hostnamectl set-hostname ceph5
echo "192.168.10.252 ceph5" >> /etc/hosts
```

#### Step 2: Time Synchronization (Critical)
Ceph relies heavily on time synchronization for authentication and data consistency.

**Install Chrony**
```bash
apt update && apt install -y chrony
```
**Enable and Start**
```
systemctl enable --now chronyd
```
**Verify Sync**
```
chronyc sources
```
*Expected Output:* You should see your NTP servers marked with `*` or `+`.

#### Step 3: Install Container Engine (Docker)
`cephadm` requires a container engine. Docker is recommended for Ubuntu/Debian environments.

**Official Source Check:**
Always refer to [Docker Official Docs](https://docs.docker.com/engine/install/ubuntu/) for the latest installation steps. Below is the standard procedure for Ubuntu 22.04/24.04.

**1. Uninstall old versions**
```bash
apt remove docker docker-engine docker.io containerd runc
```
#### 2. Install prerequisites
```
apt install -y apt-transport-https ca-certificates curl gnupg lsb-release
```
#### 3. Add Docker’s official GPG key
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```
#### 4. Set up the stable repository
```
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
```
#### 5. Install Docker Engine
```
apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```
#### 6. Start Docker
```
systemctl start docker
systemctl enable docker
```
#### 7. Verify Installation
```
docker --version
```

#### Step 4: Disable Firewall (RnD Environment)
```bash
ufw disable
```

---

## 3. Step-by-Step Host Addition

This section covers the core task: adding `ceph5` to the cluster managed by `ceph1`.

### 3.1 Establish Passwordless SSH Access
The Admin Node (`ceph1`) must be able to SSH into `ceph5` as `root` without a password. This is how `cephadm` pushes configurations.

**On Admin Node (`ceph1`):**

#### Generate SSH Key if not exists
```bash
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519
```
#### Copy Key to New Node (ceph5)
```
ssh-copy-id root@192.168.10.252
```

**Verification:**
Test the connection. It should **not** ask for a password.

```bash
ssh root@192.168.10.252 "hostname"
```
*Expected Output:* `ceph5`

> ⚠️ **Common Mistake:** Do not try to SSH as a non-root user (e.g., `ceph5@ceph5`). Ceph Orchestrator operates strictly via `root` or a sudo-enabled user with specific privileges. Always use `root`.

### 3.2 Add the Host to Ceph Orchestrator

**On Admin Node (`ceph1`):**

#### Add the new host
```bash
ceph orch host add ceph5 192.168.10.252
```
#### List all hosts to confirm
```
ceph orch host ls
```

*Expected Output:*
```
HOST     ADDR             LABELS   STATUS
ceph1    192.168.10.248   _admin   
ceph2    192.168.10.249            
ceph3    192.168.10.250            
ceph5    192.168.10.252            
```

If you see `ceph5` in the list with no error status, the host is successfully registered.

---

## 4. Service Deployment (MON, MGR, OSD)

Adding the host is only step one. Now we must deploy Ceph daemons to utilize the new node.

### 4.1 Monitor (MON) Deployment Strategy

**Theory:** Monitors maintain the cluster map. They require a **Quorum** (majority) to function.
*   **Rule:** Always use an **odd number** of MONs (3, 5, 7...).
*   **Why?** If you have 4 MONs, you need 3 to form a quorum. If 2 go down, you lose quorum. With 3 MONs, you also need 2. So, 4 MONs offer **no additional fault tolerance** over 3 MONs but consume more resources.

**Recommendation for 4-Node Cluster:**
Keep **3 MONs** on `ceph1`, `ceph2`, and `ceph3`. Use `ceph5` for Storage (OSD) and Management (MGR).

**Action:**
Check current MON placement. If it automatically added a MON to `ceph5`, you may want to restrict it to 3 nodes for stability unless you plan to add a 5th node soon.

#### Check current MON deployment
```bash
ceph orch ls mon
```
#### If you want to explicitly pin MONs to ceph1, ceph2, ceph3:
```
ceph orch apply mon --placement="3 ceph1 ceph2 ceph3"
```

### 4.2 Manager (MGR) Deployment

Managers handle dashboard, metrics, and orchestration logic. At least 2 MGRs are recommended for HA (Active + Standby).

**Action:**
Deploy MGRs on `ceph1` (Admin) and `ceph5` (New Node) to distribute load.

```bash
ceph orch apply mgr --placement="2 ceph1 ceph5"
```
#### Verify
```
ceph orch ps | grep mgr
```
*Expected Output:* Two MGR instances, one `active`, one `standby`.

### 4.3 OSD (Storage) Deployment

OSDs are the actual storage daemons. They run on available raw disks.

**Step 1: Identify Available Devices**
Check which disks are detected and available on the new node.

```bash
ceph orch device ls ceph5
```
*Look for:* Disks marked as `Available`. Ensure these are **not** your OS disk (usually `/dev/sda` or `/dev/vda`). Look for secondary disks like `/dev/sdb`, `/dev/nvme0n1`, etc.

**Step 2: Deploy OSDs**

*Option A: Auto-deploy all available devices (Recommended for simplicity)*
```bash
ceph orch apply osd --all-available-devices
```

*Option B: Manual deployment (Specific disk)*
```bash
ceph orch daemon add osd ceph5:/dev/sdb
```

**Step 3: Verify OSD Status**
```bash
ceph osd tree
ceph osd stat
```
*Expected Output:* You should see new OSDs under `ceph5` with status `up` and `in`.

---

## 5. Quorum Logic & High Availability Strategy

This section explains the "Why" behind the configuration, crucial for production planning.

### The Quorum Math
| Total MONs | Quorum Required (Majority) | Max Nodes Can Fail | Risk Level |
| :---: | :---: | :---: | :---: |
| 1 | 1 | 0 | 🔴 Critical |
| 2 | 2 | 0 | 🔴 Critical |
| **3** | **2** | **1** |  **Standard Prod** |
| 4 | 3 | 1 |  Risky (No gain over 3) |
| **5** | **3** | **2** | 🟢 **High Avail** |

### Practical Implication for Your Cluster
*   **Current State:** 4 Nodes (`ceph1-3`, `ceph5`).
*   **Best Practice:** Run **3 MONs**.
*   **Scenario:** If `ceph1` fails, `ceph2` and `ceph3` maintain quorum (2/3). Cluster stays online.
*   **If you ran 4 MONs:** If `ceph1` and `ceph2` fail, you have 2 MONs left. Quorum requires 3. Cluster goes offline.
*   **Conclusion:** Adding a 4th MON does not improve resilience against single-node failures compared to 3 MONs, but it *does* increase complexity. Stick to 3 MONs until you have a 5th node.

---

## 6. Verification & Health Checks

After deployment, perform these checks to ensure the cluster is healthy.

### 6.1 Cluster Health
```bash
ceph health detail
```
*Goal:* `HEALTH_OK`. If `HEALTH_WARN`, read the details. Common warnings include "too few PGs" or "clock skew".

### 6.2 Service Status
#### Check all running daemons

```bash
ceph orch ps
```
#### Check host status
```
ceph orch host ls
```

### 6.3 Data Path Test
Create a test pool and write data to verify end-to-end functionality.

#### 1. Create a pool with 64 PGs

```bash
ceph osd pool create testpool 64 64
```
#### 2. Set replication size to 3 (default)
```
ceph osd pool set testpool size 3
```
#### 3. Write a test object
```
echo "Ceph Expansion Test Data" > testfile.txt
rados put test-object-1 testfile.txt --pool=testpool
```

#### 4. Read it back
```
rados get test-object-1 testfile-read.txt --pool=testpool
cat testfile-read.txt
```
#### 5. Map the object to see where it lives
```
ceph osd map testpool test-object-1
```
*Expected Output:* The map should show OSDs located on different hosts (e.g., one on `ceph1`, one on `ceph2`, one on `ceph5`), confirming data distribution.

---

## 7. Production Maintenance & Best Practices

### 7.1 Enable Ceph Dashboard
For visual monitoring, enable the built-in dashboard.

# Enable module

```bash
ceph mgr module enable dashboard
```
#### Generate self-signed cert (for internal/RnD)
```
ceph dashboard create-self-signed-cert
```
#### Create admin user
```
ceph dashboard ac-user-create admin <your-password> administrator
```
#### Get URL
```
ceph mgr services
```
Access via: `https://<ceph1-ip>:8443`

### 7.2 Scrubbing Optimization
Scrubbing checks data integrity but impacts performance. Schedule it during off-peak hours.

#### Set scrub window (e.g., 10 PM to 6 AM)

```bash
ceph config set osd osd_scrub_begin_hour 22
ceph config set osd osd_scrub_end_hour 6
```

### 7.3 Regular Updates
Keep Ceph packages updated on all nodes.

#### On all nodes (ceph1, ceph2, ceph3, ceph5)

```bash
apt update
apt upgrade ceph-common ceph-base ceph-mon ceph-mgr ceph-osd
systemctl restart ceph-target@* # Restart services if needed
```

### 7.4 Backup Configuration
Backup the `/etc/ceph` directory regularly.

```bash
tar -czf ceph-config-backup-$(date +%F).tar.gz /etc/ceph
```

---

## 8. Troubleshooting Common Issues

### Issue 1: `ssh-copy-id` still asks for password
*   **Cause:** Incorrect user or permissions.
*   **Fix:** Ensure you are copying to `root`. Check permissions on `ceph5`:
    ```bash
    chmod 700 /root/.ssh
    chmod 600 /root/.ssh/authorized_keys
    ```

### Issue 2: `check-host failed: No container engine binary found`
*   **Cause:** Docker/Podman not installed on new node.
*   **Fix:** Follow Section 2.3 to install Docker on `ceph5`.

### Issue 3: `HEALTH_WARN: too few PGs per OSD`
*   **Cause:** Default PG count is too low for the number of OSDs.
*   **Fix:** Increase PGs for affected pools.
    ```bash
    ceph osd pool set <pool-name> pg_num <new-pg-count>
    ceph osd pool set <pool-name> pgp_num <new-pg-count>
    ```
    *Note:* Use the [Ceph PG Calculator](https://ceph.io/pgcalc/) to determine the correct value.

### Issue 4: OSDs not appearing
*   **Cause:** Disks are partitioned or mounted.
*   **Fix:** Wipe the disk manually (CAUTION: Data Loss).
    ```bash
    # On ceph5
    sgdisk --zap-all /dev/sdb
    dd if=/dev/zero of=/dev/sdb bs=1M count=100
    partprobe /dev/sdb
    ```
    Then retry `ceph orch device ls ceph5`.

---

## Conclusion

You have successfully expanded your Ceph cluster from 3 to 4 nodes. By adhering to the **3-MON Quorum rule**, deploying **MGRs for HA**, and distributing **OSDs across all nodes**, your cluster is now more robust and scalable.

**Next Steps:**
1.  Integrate with OpenStack/RBD for VM storage.
2.  Set up RGW for Object Storage (S3-compatible).
3.  Configure external monitoring (Prometheus/Grafana) for long-term trends.

*Guide Version: 1.0 | Last Updated: April 30, 2026*
