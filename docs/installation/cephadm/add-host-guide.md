Filename: `ceph-scaleout-handbook.md`

# Ceph Cluster Scale-Out & Production Hardening Handbook
**A Practical Guide to Expanding a 3-Node Ceph Cluster to 4+ Nodes using Cephadm**

## Table of Contents
1. [Introduction & Architectural Context](#1-introduction--architectural-context)
2. [Pre-Requisites & Environment Preparation](#2-pre-requisites--environment-preparation)
3. [SSH Key Management & Security Strategy](#3-ssh-key-management--security-strategy)
4. [Step-by-Step Host Expansion](#4-step-by-step-host-expansion)
5. [Service Orchestration (MON, MGR, OSD)](#5-service-orchestration-mon-mgr-osd)
6. [Quorum Logic & High Availability Design](#6-quorum-logic--high-availability-design)
7. [Verification, Operations & Testing](#7-verification-operations--testing)
8. [Production Maintenance & Lifecycle Management](#8-production-maintenance--lifecycle-management)
9. [Troubleshooting & Real-World Scenarios](#9-troubleshooting--real-world-scenarios)

---

## 1. Introduction & Architectural Context

### The Scenario
You are managing an R&D Cloud Infrastructure at **Paulco Cloud**, running a **Ceph Storage Cluster** on top of **Proxmox VE** virtual machines. Your initial setup consists of **3 Nodes** (`ceph1`, `ceph2`, `ceph3`) managed via `cephadm` (the modern Ceph Orchestrator).

### The Goal
Expand the cluster by adding a **4th Node** (`ceph5` with IP `192.168.68.252`) to increase storage capacity and compute resources for distributed services. The guide focuses on **production-grade practices**, ensuring high availability (HA), security, and ease of maintenance.

### Why Cephadm?
Modern Ceph uses `cephadm` to deploy daemons inside containers (Docker/Podman). This eliminates manual package installation errors and simplifies scaling. You do **not** manually install `ceph-mon` or `ceph-osd` packages on new nodes; `cephadm` handles this automatically via SSH.

---

## 2. Pre-Requisites & Environment Preparation

Before touching the Ceph commands, the new host must be prepared to join the cluster. This is the most critical step for stability.

### 2.1 Network Requirements
*   **Connectivity:** All nodes must be on the same Layer 2 network subnet (e.g., `192.168.68.0/24`).
*   **Latency:** Low latency (<1ms) between nodes is ideal for OSD replication.
*   **Firewall:** For RnD, disable firewalls. For Production, ensure these ports are open:
    *   `6789`: Ceph Monitors (TCP)
    *   `3300`: Ceph v2 Protocol (MSGR2) (TCP)
    *   `6800-7300`: OSDs and MGRs (TCP)

### 2.2 Operating System Setup (On New Node `ceph5`)

#### Step 1: Hostname & Resolution
The hostname must be unique and resolvable.

```bash
# On ceph5
hostnamectl set-hostname ceph5
echo "192.168.68.252 ceph5" >> /etc/hosts
# Ensure Admin Node IP is also in hosts if DNS is not used
echo "192.168.68.248 ceph1" >> /etc/hosts
```

#### Step 2: Time Synchronization (NTP)
Ceph is extremely sensitive to time drift. Clock skew > 0.05 seconds can cause authentication failures.

**Official Source:** [Chrony Documentation](https://chrony-project.org/)

```bash
# Install Chrony
apt update && apt install -y chrony

# Enable Service
systemctl enable --now chronyd

# Verify Sync Status
chronyc sources
```
*Expected Output:* Look for lines starting with `^*` or `^+`, indicating synchronization with upstream NTP servers.

#### Step 3: Container Engine Installation (Docker)
`cephadm` requires a container runtime. Docker is the standard choice for Ubuntu/Debian environments.

**Official Source:** [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)

**Practical Installation Guide:**

1.  **Uninstall Old Versions:**
    ```bash
    apt remove docker docker-engine docker.io containerd runc
    ```

2.  **Install Prerequisites:**
    ```bash
    apt install -y apt-transport-https ca-certificates curl gnupg lsb-release
    ```

3.  **Add Docker’s Official GPG Key:**
    ```bash
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
    ```

4.  **Set Up Stable Repository:**
    ```bash
    echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
    ```

5.  **Install Docker Engine:**
    ```bash
    apt update
    apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
    ```

6.  **Start & Enable Docker:**
    ```bash
    systemctl start docker
    systemctl enable docker
    ```

7.  **Verify Installation:**
    ```bash
    docker --version
    # Expected: Docker version 24.x.x or higher
    ```

#### Step 4: Disable Firewall (RnD Only)
```bash
ufw disable
```
*Note: In production, configure `ufw` or `iptables` to allow specific Ceph ports instead of disabling it.*

---

## 3. SSH Key Management & Security Strategy

This section addresses the common confusion regarding SSH keys in automated clusters.

### 3.1 The Core Concept
*   **Admin Node (`ceph1`)** acts as the Control Plane.
*   It uses **one primary SSH Key Pair** to communicate with all other nodes.
*   **Do NOT generate a new key for every new node.** Reusing the same key simplifies management and is standard practice for infrastructure-as-code tools.

### 3.2 Identifying the Active Key
To verify which key your Admin Node is using:

```bash
# On ceph1 (Admin Node)
ls -l /root/.ssh/id_*
```
*Typical Output:*
```
-rw------- root root ... id_ed25519      (Private Key)
-rw-r--r-- root root ... id_ed25519.pub  (Public Key)
```
If you see `id_ed25519`, this is likely the key being used. If you only see `id_rsa`, then RSA is being used. Ed25519 is preferred for performance and security.

### 3.3 Distributing the Key (Passwordless Access)
You must copy the **Public Key** from `ceph1` to the `authorized_keys` file of `ceph5`.

**Action on Admin Node (`ceph1`):**

```bash
# Copy the public key to the new node
ssh-copy-id root@192.168.68.252
```

**Verification:**
Test the connection. It should **NOT** ask for a password.

```bash
ssh root@192.168.68.252 "hostname"
```
*Expected Output:* `ceph5`

> ⚠️ **Critical Note:** Always use the `root` user for Ceph Orchestrator operations. Using non-root users requires complex sudo configurations that are prone to errors. Stick to `root` for simplicity in RnD/Internal clusters.

### 3.4 Verifying Key Fingerprint (Security Audit)
To ensure no Man-in-the-Middle attack occurred and the correct key is installed:

1.  **Get Fingerprint on Admin Node:**
    ```bash
    ssh-keygen -lf /root/.ssh/id_ed25519.pub
    # Output: 256 SHA256:AbCdEf... root@ceph1 (ED25519)
    ```

2.  **Get Fingerprint on New Node:**
    ```bash
    # On ceph5
    ssh-keygen -lf /root/.ssh/authorized_keys
    # Output should match the SHA256 hash above exactly.
    ```

---

## 4. Step-by-Step Host Expansion

Now that the OS is ready and SSH is configured, we integrate the node into the Ceph Cluster.

### 4.1 Add the Host to Orchestrator
**Action on Admin Node (`ceph1`):**

```bash
# Add the new host
ceph orch host add ceph5 192.168.68.252

# List all hosts to confirm
ceph orch host ls
```

**Expected Output:**
```
HOST     ADDR             LABELS   STATUS
ceph1    192.168.68.248   _admin   
ceph2    192.168.68.249            
ceph3    192.168.68.250            
ceph5    192.168.68.252            
```
*Status:* If `STATUS` is empty or shows no error, the host is successfully added.

### 4.2 What Happens Behind the Scenes?
When you run `ceph orch host add`:
1.  Ceph connects via SSH to `ceph5`.
2.  It checks for Docker/Podman.
3.  It pulls the necessary Ceph container images (if not cached).
4.  It prepares the node to receive daemon deployments (MON, MGR, OSD).
5.  **No manual config file copying is required.** `cephadm` injects configs into containers dynamically.

---

## 5. Service Orchestration (MON, MGR, OSD)

Adding the host is useless without deploying services on it.

### 5.1 Monitor (MON) Deployment Strategy

**Theory:** Monitors maintain the cluster map. They require a **Quorum** (majority) to operate.
*   **Rule:** Always use an **odd number** of MONs (3, 5, 7...).
*   **Why?** 
    *   3 MONs need 2 to form quorum. Can tolerate 1 failure.
    *   4 MONs need 3 to form quorum. Can tolerate 1 failure.
    *   **Result:** 4 MONs provide **no additional fault tolerance** over 3 MONs but consume more resources. Therefore, avoid 4 MONs.

**Recommendation for 4-Node Cluster:**
Keep **3 MONs** on `ceph1`, `ceph2`, `ceph3`. Do **not** deploy a MON on `ceph5` unless you plan to add a 5th node soon.

**Action:**
```bash
# Check current MON placement
ceph orch ls mon

# If a MON was auto-deployed on ceph5, restrict it to 3 nodes:
ceph orch apply mon --placement="3 ceph1 ceph2 ceph3"
```

### 5.2 Manager (MGR) Deployment

Managers handle the Dashboard, metrics, and orchestration logic. HA requires at least 2 MGRs (Active + Standby).

**Action:**
Deploy MGRs on `ceph1` (Admin) and `ceph5` (New Node) to balance load.

```bash
ceph orch apply mgr --placement="2 ceph1 ceph5"

# Verify
ceph orch ps | grep mgr
```
*Expected Output:* Two MGR instances. One `active`, one `standby`.

### 5.3 OSD (Storage) Deployment

OSDs are the storage daemons. They utilize raw disks.

**Step 1: Identify Available Devices**
Check which disks are detected on the new node.

```bash
ceph orch device ls ceph5
```
*Look for:* Disks marked as `Available`. 
*Warning:* Ensure these are **not** your OS disk (usually `/dev/sda` or `/dev/vda`). Look for secondary disks like `/dev/sdb`, `/dev/nvme0n1`.

**Step 2: Deploy OSDs**

*Option A: Auto-deploy all available devices (Recommended)*
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
*Expected Output:* New OSDs under `ceph5` with status `up` and `in`.

---

## 6. Quorum Logic & High Availability Design

Understanding Quorum is vital for Production stability.

### The Quorum Math Table

| Total MONs | Quorum Required (Majority) | Max Nodes Can Fail | Risk Level | Recommendation |
| :---: | :---: | :---: | :---: | :---: |
| 1 | 1 | 0 | 🔴 Critical | Never use in Prod |
| 2 | 2 | 0 |  Critical | No HA |
| **3** | **2** | **1** |  Standard | **Min for Prod** |
| 4 | 3 | 1 |  Risky | Avoid (No gain over 3) |
| **5** | **3** | **2** | 🟢 High Avail | Best for Large Clusters |

### Practical Implication for Your Cluster
*   **Current State:** 4 Nodes (`ceph1-3`, `ceph5`).
*   **Best Practice:** Run **3 MONs**.
*   **Scenario:** If `ceph1` fails, `ceph2` and `ceph3` maintain quorum (2/3). Cluster stays online.
*   **If you ran 4 MONs:** If `ceph1` and `ceph2` fail, you have 2 MONs left. Quorum requires 3. Cluster goes offline.
*   **Conclusion:** Stick to 3 MONs until you have a 5th node. Use `ceph5` for Storage (OSD) and Management (MGR).

---

## 7. Verification, Operations & Testing

After deployment, perform these checks to ensure the cluster is healthy and functional.

### 7.1 Cluster Health Check
```bash
ceph health detail
```
*Goal:* `HEALTH_OK`. 
*Common Warning:* `HEALTH_WARN: too few PGs per OSD`. This is normal initially. It resolves as data is written or PGs are adjusted.

### 7.2 Service Status Check
```bash
# Check all running daemons
ceph orch ps

# Check host status
ceph orch host ls
```

### 7.3 Data Path Test (End-to-End)
Create a test pool and write data to verify distribution.

```bash
# 1. Create a pool with 64 PGs
ceph osd pool create testpool 64 64

# 2. Set replication size to 3 (default)
ceph osd pool set testpool size 3

# 3. Write a test object
echo "Ceph Expansion Test Data - $(date)" > testfile.txt
rados put test-object-1 testfile.txt --pool=testpool

# 4. Read it back
rados get test-object-1 testfile-read.txt --pool=testpool
cat testfile-read.txt

# 5. Map the object to see where it lives
ceph osd map testpool test-object-1
```
*Expected Output:* The map should show OSDs located on **different hosts** (e.g., one on `ceph1`, one on `ceph2`, one on `ceph5`). This confirms data distribution and CRUSH rules are working.

### 7.4 Enable Ceph Dashboard (Optional but Recommended)
For visual monitoring:

```bash
ceph mgr module enable dashboard
ceph dashboard create-self-signed-cert
ceph dashboard ac-user-create admin <your-password> administrator
ceph mgr services
```
Access via: `https://<ceph1-ip>:8443`

---

## 8. Production Maintenance & Lifecycle Management

### 8.1 Regular Updates
Keep Ceph packages updated on all nodes.

```bash
# On ALL nodes (ceph1, ceph2, ceph3, ceph5)
apt update
apt upgrade ceph-common ceph-base ceph-mon ceph-mgr ceph-osd
# Restart services if prompted or manually:
systemctl restart ceph-target@*
```

### 8.2 Scrubbing Optimization
Scrubbing checks data integrity but impacts I/O performance. Schedule it during off-peak hours.

```bash
# Set scrub window (e.g., 10 PM to 6 AM)
ceph config set osd osd_scrub_begin_hour 22
ceph config set osd osd_scrub_end_hour 6
```

### 8.3 Backup Configuration
Backup the `/etc/ceph` directory regularly.

```bash
tar -czf ceph-config-backup-$(date +%F).tar.gz /etc/ceph
scp ceph-config-backup-*.tar.gz backup-server:/path/to/backups/
```

### 8.4 Removing a Host (Cleanup)
If you need to remove `ceph5` later:

1.  **Drain OSDs:**
    ```bash
    ceph osd out <osd-id-on-ceph5>
    # Wait for data to rebalance
    ceph -w
    ```
2.  **Remove OSD Daemons:**
    ```bash
    ceph orch osd rm <osd-id>
    ```
3.  **Remove Host:**
    ```bash
    ceph orch host rm ceph5
    ```
4.  **Clean Up SSH Keys:**
    Remove `ceph1`'s public key from `ceph5`'s `/root/.ssh/authorized_keys` if decommissioning permanently.

---

## 9. Troubleshooting & Real-World Scenarios

### Scenario 1: `check-host failed: No container engine binary found`
*   **Cause:** Docker/Podman not installed on new node.
*   **Fix:** Follow Section 2.3 to install Docker on `ceph5`.

### Scenario 2: `Permission Denied` during SSH
*   **Cause:** Incorrect user or permissions.
*   **Fix:** 
    1. Ensure you are using `root`.
    2. Check permissions on `ceph5`:
       ```bash
       chmod 700 /root/.ssh
       chmod 600 /root/.ssh/authorized_keys
       ```
    3. Re-run `ssh-copy-id`.

### Scenario 3: `HEALTH_WARN: clock skew detected`
*   **Cause:** Time desynchronization.
*   **Fix:** 
    1. Check `chronyc sources` on all nodes.
    2. Ensure all nodes point to the same NTP server.
    3. Restart chrony: `systemctl restart chronyd`.

### Scenario 4: Disk Not Showing in `ceph orch device ls`
*   **Cause:** Disk is partitioned, mounted, or has existing filesystem signatures.
*   **Fix:** 
    1. Unmount: `umount /dev/sdb1`
    2. Zap disk: `ceph-volume lvm zap /dev/sdb --destroy`
    3. Refresh: `ceph orch device ls --refresh`

### Scenario 5: Slow Performance after Expansion
*   **Cause:** Data rebalancing (Backfilling) is consuming bandwidth.
*   **Fix:** 
    1. Check backfill status: `ceph -w`
    2. Limit backfill speed if necessary:
       ```bash
       ceph config set osd osd_max_backfills 1
       ceph config set osd osd_recovery_max_active 3
       ```
    3. Restore defaults after rebalancing completes.

---

## Conclusion

You have successfully expanded your Ceph cluster from 3 to 4 nodes. By adhering to the **3-MON Quorum rule**, deploying **MGRs for HA**, and distributing **OSDs across all nodes**, your cluster is now more robust and scalable.

**Key Takeaways:**
1.  **SSH Key:** Use one central key on Admin Node for all hosts.
2.  **Quorum:** Keep MON count odd (3 or 5).
3.  **Automation:** Let `cephadm` handle config files; do not copy them manually.
4.  **Verification:** Always test data path with `rados` put/get.

**Next Steps for Paulco Cloud RnD:**
1.  Integrate with OpenStack Cinder/Ceph driver.
2.  Set up RGW (Rados Gateway) for S3-compatible object storage.
3.  Implement Prometheus/Grafana monitoring stack using Docker Compose on Admin Node.

*Guide Version: 2.0 | Last Updated: May 02, 2026*