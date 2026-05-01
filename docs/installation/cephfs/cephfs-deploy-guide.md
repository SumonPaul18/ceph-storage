# cephfs_prod_ops_guide.md

---

# 📘 Professional Practical Guide: CephFS Implementation, Integration, and Operations
**Target Audience:** DevOps Engineers, System Administrators, Cloud Infrastructure Architects  
**Context:** 5-Node Production Ceph Cluster (Cephadm managed) integrated with OpenStack  
**Version Focus:** Ceph Reef/Squid (Latest Stable), RHEL/Ubuntu, OpenStack 2024.1+

---

##  Table of Contents

1. [Executive Summary & Architecture Overview](#1-executive-summary--architecture-overview)
2. [Pre-requisites & Environment Validation](#2-pre-requisites--environment-validation)
3. [Phase 1: Core CephFS Deployment via Cephadm](#3-phase-1-core-cephfs-deployment-via-cephadm)
4. [Phase 2: Metadata Server (MDS) High Availability Configuration](#4-phase-2-metadata-server-mds-high-availability-configuration)
5. [Phase 3: Client-Side Connectivity & Mounting Strategies](#5-phase-3-client-side-connectivity--mounting-strategies)
6. [Phase 4: OpenStack Manila Integration (Shared File Systems)](#6-phase-4-openstack-manila-integration-shared-file-systems)
7. [Phase 5: Advanced Gateway Setup (NFS-Ganesha)](#7-phase-5-advanced-gateway-setup-nfs-ganesha)
8. [Phase 6: Operational Management, Monitoring & Maintenance](#8-phase-6-operational-management-monitoring--maintenance)
9. [Phase 7: Troubleshooting, Cleanup & Decommissioning](#9-phase-7-troubleshooting-cleanup--decommissioning)
10. [Best Practices & Security Hardening](#10-best-practices--security-hardening)

---

## 1. Executive Summary & Architecture Overview

### The Story of Data in Modern Infrastructure
In a modern cloud environment, data is not just stored; it is accessed concurrently by hundreds of virtual machines, containers, and legacy applications. While **RBD (RADOS Block Device)** serves as the high-performance backbone for Virtual Machine disks (Block Storage), and **RGW (RADOS Gateway)** handles unstructured object data via S3 APIs (Object Storage), there remains a critical gap: **POSIX-compliant shared file storage**.

This guide addresses that gap using **CephFS**. Imagine a scenario where your development team needs a shared home directory accessible from multiple Linux VMs simultaneously, or your Kubernetes cluster requires `ReadWriteMany` (RWX) persistent volumes. CephFS provides this capability natively on top of your existing Ceph cluster, leveraging the same reliability and scalability mechanisms.

### Key Architectural Components
*   **CephFS:** The POSIX file system interface.
*   **MDS (Metadata Server):** The brain of CephFS. It manages the file hierarchy (names, permissions, directory structures) in memory for speed. *Crucial Note: MDS does not store file content.*
*   **OSDs (Object Storage Daemons):** Store the actual file data and metadata objects.
*   **MONs (Monitors):** Maintain the cluster map and authentication keys.
*   **Clients:** Linux Kernel Driver (native, fast) or FUSE (user-space, flexible).

### Why This Guide?
Most documentation is fragmented. This guide consolidates the entire lifecycle—from verifying an empty cluster to integrating with OpenStack Manila—into a single, narrative-driven operational manual. It assumes you are using `cephadm` (the modern orchestrator) and focuses on **production-grade** practices, such as separating metadata pools onto SSDs and ensuring MDS high availability.

---

## 2. Pre-requisites & Environment Validation

Before deploying any new service, we must ensure the foundation is solid. In our 5-node cluster story, we start by validating health and identifying existing services.

### 2.1. Validate Cluster Health
Ensure your cluster is in a `HEALTH_OK` state. Deploying CephFS on an unhealthy cluster can lead to data corruption or deployment failures.

**Practical Command:**
```bash
# Check overall cluster status
ceph -s

# Expected Output:
# cluster:
#   id:     <cluster-id>
#   health: HEALTH_OK
# ...
```

### 2.2. Verify Existing CephFS Status
It is common to inherit clusters where services may be partially configured. We must verify if CephFS is already enabled to avoid conflicts.

**Practical Commands:**
```bash
# 1. Check for active filesystems
ceph fs ls
# If output is "No filesystems enabled", proceed to Phase 1.

# 2. Check MDS daemon status
ceph mds stat
# If output is empty or shows no active ranks, CephFS is not running.

# 3. Check Orchestrator Services
ceph orch ls | grep mds
# If no output, the MDS service is not deployed via cephadm.

# 4. Check for CephFS-specific pools
ceph osd pool ls detail | grep cephfs
```

### 2.3. Hardware & Network Requirements
*   **Nodes:** Minimum 3 nodes for production (we have 5).
*   **Storage:** 
    *   **Metadata Pool:** Must reside on **SSD/NVMe** for low latency. MDS performance is directly tied to disk IOPS.
    *   **Data Pool:** Can be HDD or SSD depending on throughput needs.
*   **Network:** 
    *   Public Network: For client access (MON/MDS/RGW).
    *   Cluster Network: For OSD replication (internal traffic).
*   **Firewall:** Ensure ports `6789` (MON), `6800-7300` (OSD/MDS), and `2049` (NFS, if used) are open between clients and cluster nodes.

---

## 3. Phase 1: Core CephFS Deployment via Cephadm

In this phase, we transition from a "Block/Object Only" cluster to a "File-Enabled" cluster. We will create dedicated pools and initialize the filesystem.

### 3.1. Concept: Why Dedicated Pools?
While CephFS *can* use default pools, best practice dictates creating separate pools for **Metadata** and **Data**. This allows us to apply different CRUSH rules (e.g., placing metadata on fast SSDs) and manage quotas independently.

### 3.2. Step-by-Step Deployment

#### Step 1: Create the Metadata Pool
The metadata pool stores inodes, directory entries, and file attributes. It requires high IOPS.

```bash
# Create metadata pool with 32 PGs (adjust based on cluster size, 32-64 is good for start)
ceph osd pool create cephfs_metadata 32 32

# Enable the 'cephfs' application type on the pool
ceph osd pool application enable cephfs_metadata cephfs
```

> **Pro Tip:** If you have a specific CRUSH rule for SSDs (e.g., `ssd-rule`), create the pool using that rule:
> `ceph osd pool create cephfs_metadata 32 32 ssd-rule`

#### Step 2: Create the Data Pool
The data pool stores the actual file contents.

```bash
# Create data pool with 64 PGs (higher than metadata due to larger volume)
ceph osd pool create cephfs_data 64 64

# Enable the 'cephfs' application type
ceph osd pool application enable cephfs_data cephfs
```

#### Step 3: Initialize the Filesystem
Now we bind these pools together into a logical filesystem named `mycephfs`.

```bash
# Syntax: ceph fs new <fs_name> <metadata_pool> <data_pool>
ceph fs new mycephfs cephfs_metadata cephfs_data
```

#### Step 4: Verification
Confirm the filesystem is recognized by the cluster.

```bash
ceph fs ls
# Output should show: name: mycephfs, metadata pool: cephfs_metadata, data pools: [cephfs_data]
```

---

## 4. Phase 2: Metadata Server (MDS) High Availability Configuration

A CephFS filesystem is useless without an MDS. The MDS is the stateful component that caches the file tree. In production, we never run a single MDS; we run at least two for High Availability (Active-Standby model).

### 4.1. Understanding MDS Roles
*   **Active MDS:** Handles client requests for a specific rank (subtree of the file system).
*   **Standby MDS:** Hot-standby ready to take over if the Active MDS fails.
*   **Rank:** A logical partition of the filesystem. By default, `max_mds` is 1, meaning only one Active MDS at a time.

### 4.2. Deploying MDS via Cephadm

We will deploy 2 MDS daemons. You can target specific hosts or use labels.

#### Option A: Target Specific Hosts (Recommended for Control)
Assume `ceph1` and `ceph2` are powerful nodes suitable for MDS.

```bash
# Deploy 2 MDS instances for 'mycephfs' on specific hosts
ceph orch apply mds mycephfs --placement="2 ceph1 ceph2"
```

#### Option B: Use Host Labels (Flexible)
If you prefer dynamic placement:

```bash
# Label nodes
ceph orch host label add ceph1 mds
ceph orch host label add ceph2 mds

# Deploy using label
ceph orch apply mds mycephfs --placement="2 label:mds"
```

### 4.3. Verification & Troubleshooting

After deployment, monitor the status. It may take 1-2 minutes for MDS daemons to start and elect an active leader.

```bash
# Watch the cluster status
watch ceph -s

# Check MDS specific status
ceph mds stat
```

**Expected Healthy Output:**
```text
mycephfs:1 {0=mycephfs.ceph1.abcde=up:active} 1 up:standby
```
*   `up:active`: One daemon is handling requests.
*   `up:standby`: The other is ready to failover.

**Common Issue: `MDS_ALL_DOWN`**
If you see `MDS_ALL_DOWN`, check:
1.  Are the containers running? `ceph orch ps | grep mds`
2.  Are there firewall issues blocking port `6800-7300`?
3.  Check logs: `ceph orch logs mds.mycephfs.<daemon-id>`

---

## 5. Phase 3: Client-Side Connectivity & Mounting Strategies

Now that the server side is ready, we connect clients. We focus on the **Linux Kernel Driver** as it offers the best performance and stability for Linux clients.

### 5.1. Prerequisites on Client Node
*   OS: Ubuntu 20.04+, RHEL 8+, CentOS Stream 8+.
*   Package: `ceph-common`.

### 5.2. Installation & Configuration

#### Step 1: Install Ceph Common Tools
```bash
# Ubuntu/Debian
sudo apt update && sudo apt install ceph-common -y

# RHEL/CentOS/Rocky
sudo dnf install ceph-common -y
```

#### Step 2: Transfer Configuration Files
From your Ceph Admin node (`ceph1`), copy the following to the client's `/etc/ceph/` directory:
1.  `ceph.conf`
2.  `ceph.client.admin.keyring` (Or a restricted user keyring, see Security Section).

```bash
# On Admin Node
scp /etc/ceph/ceph.conf user@client-ip:/etc/ceph/
scp /etc/ceph/ceph.client.admin.keyring user@client-ip:/etc/ceph/
```

#### Step 3: Create Mount Point
```bash
sudo mkdir /mnt/mycephfs
```

#### Step 4: Mount CephFS (Kernel Driver)
Use the `mount` command with the `ceph` type.

```bash
# Syntax: mount -t ceph <MON_IP>:<PORT>:/ <MountPoint> -o <Options>

sudo mount -t ceph 192.168.68.250:6789:/ /mnt/mycephfs \
     -o name=admin,secretfile=/etc/ceph/ceph.client.admin.keyring
```
*   Replace `192.168.68.250` with any of your MON node IPs.
*   `name=admin`: Specifies the Ceph user.
*   `secretfile`: Path to the keyring containing the secret key.

#### Step 5: Verification
```bash
# Check mounted filesystem
df -h /mnt/mycephfs

# Test Write Access
touch /mnt/mycephfs/test_file.txt
echo "Hello CephFS" > /mnt/mycephfs/test_file.txt
cat /mnt/mycephfs/test_file.txt
```

### 5.3. Persistent Mounting (/etc/fstab)
To ensure the mount survives reboots, add an entry to `/etc/fstab`.

**Edit `/etc/fstab`:**
```text
192.168.68.250:6789:/   /mnt/mycephfs   ceph   name=admin,secretfile=/etc/ceph/ceph.client.admin.keyring,noatime,_netdev   0   0
```
*   `_netdev`: Ensures the network is up before attempting to mount.
*   `noatime`: Improves performance by not updating access times on reads.

**Test Persistence:**
```bash
sudo mount -a
# If no error, the configuration is valid.
```

---

## 6. Phase 4: OpenStack Manila Integration (Shared File Systems)

For cloud environments, manual mounting is not scalable. OpenStack **Manila** provides "File Shares as a Service." Users can request a share via Horizon/API, and Manila provisions it on CephFS.

### 6.1. Architecture Overview
*   **Manila Share Manager:** Communicates with Ceph to create subvolumes.
*   **CephFS Native Driver:** Allows Manila to use CephFS directly without intermediate gateways (most efficient).
*   **Access Control:** Manila manages IP-based or User-based access rules.

### 6.2. Ceph Side Preparation

Create a dedicated user for Manila with restricted permissions. Do **not** use the admin key.

```bash
# On Ceph Admin Node
ceph auth get-or-create client.manila \
  mon 'allow r' \
  mds 'allow rw' \
  osd 'allow rw pool=cephfs_data, allow rw pool=cephfs_metadata' \
  -o /etc/ceph/ceph.client.manila.keyring
```
Copy `ceph.client.manila.keyring` to your **OpenStack Controller** node.

### 6.3. OpenStack Controller Configuration

Edit `/etc/manila/manila.conf` on the controller node.

**Key Settings:**
```ini
[DEFAULT]
enabled_share_backends = cephfsnative
enabled_share_protocols = CephFS

[cephfsnative]
share_backend_name = cephfsnative
share_driver = manila.share.drivers.cephfs.driver.CephFSDriver
driver_handles_share_servers = False
cephfs_auth_id = manila
cephfs_cluster_name = ceph
cephfs_enable_snapshots = True
# Optional: If using NFS-Ganesha for export
# cephfs_ganesha_server_ip = <IP_OF_NFS_NODE>
```

**Restart Services:**
```bash
systemctl restart openstack-manila-share
systemctl restart openstack-manila-scheduler
```

### 6.4. Creating Share Types & Using in Horizon

#### Step 1: Create Share Type (CLI)
```bash
manila type-create cephfs_type true
manila type-key cephfs_type set share_backend_name=cephfsnative
```

#### Step 2: Create Share via Horizon (GUI)
1.  Navigate to **Project > Shares > Shares**.
2.  Click **Create Share**.
3.  **Name:** `web-content-share`
4.  **Size:** `10 GB`
5.  **Share Protocol:** `CephFS`
6.  **Share Type:** `cephfs_type`
7.  Click **Create Share**.

#### Step 3: Grant Access
1.  Go to the **Share Details** page.
2.  Click **Manage Rules** or **Add Rule**.
3.  **Access Type:** `IP`
4.  **Access Level:** `RW`
5.  **Access To:** `<VM_IP_ADDRESS>` (The IP of the VM that needs access).

#### Step 4: Mount in VM
Inside the VM, install `ceph-common` and mount using the export path provided by Manila.

```bash
# Get Export Location from Horizon or CLI: manila share-export-location-list <SHARE_ID>

sudo mount -t ceph <MON_IP>:6789:/volumes/_nogroup/<SHARE_UUID> /mnt/manila_share \
     -o name=manila,secretfile=/path/to/manila.keyring
```

---

## 7. Phase 5: Advanced Gateway Setup (NFS-Ganesha)

Some clients (Windows, macOS, or legacy apps) cannot use the native Ceph kernel driver. **NFS-Ganesha** acts as a bridge, exporting CephFS over standard NFSv3/v4 protocols.

### 7.1. Deploy NFS-Ganesha Service

We will deploy a single NFS gateway instance. For HA, you can deploy multiple behind a VIP (Keepalived), but for this guide, we focus on basic setup.

```bash
# Deploy NFS service named 'mynfs' on node 'ceph3'
ceph orch apply nfs mynfs --placement="1 ceph3"
```

### 7.2. Create NFS Export for CephFS

We need to tell the NFS gateway which CephFS path to export. We use a YAML specification.

**Create `nfs-export.yaml`:**
```yaml
service_type: nfs
service_id: mynfs
placement:
  count: 1
spec:
  port: 2049
  user_id: admin
  backend:
    type: cephfs
    cephfs:
      name: mycephfs
      path: /
```

**Apply the Configuration:**
```bash
ceph orch apply -i nfs-export.yaml
```

### 7.3. Client Connectivity (NFS)

Now, any client with NFS support can mount the share.

**Linux/macOS Client:**
```bash
sudo mkdir /mnt/nfs_cephfs

# Mount using the IP of the node where NFS-Ganesha is running (ceph3)
sudo mount -t nfs 192.168.68.253:/mycephfs /mnt/nfs_cephfs

# Verify
df -h /mnt/nfs_cephfs
```

**Windows Client:**
1.  Open File Explorer.
2.  Right-click "This PC" -> "Map Network Drive".
3.  Folder: `\\192.168.68.253\mycephfs`
4.  Connect.

---

## 8. Phase 6: Operational Management, Monitoring & Maintenance

Running CephFS in production requires ongoing management. Here are the daily operational tasks.

### 8.1. Monitoring CephFS Health

#### Check MDS Performance
```bash
# View real-time MDS stats
ceph mds stat

# View detailed performance counters
ceph daemon mds.<daemon-name> perf dump
```
*Look for:* High latency in `mds_server.handle_client_request`.

#### Check Client Sessions
```bash
# List all connected clients
ceph fs session ls mycephfs

# View detailed session info
ceph fs session info mycephfs <session-id>
```
*Use Case:* Identify which client is holding locks or causing slowness.

### 8.2. Managing Quotas

You can limit the size or number of files in a specific directory.

```bash
# Set a 10GB quota on /projects/teamA
setfattr -n ceph.quota.max_bytes -v 10737418240 /mnt/mycephfs/projects/teamA

# Set a limit of 10,000 files
setfattr -n ceph.quota.max_files -v 10000 /mnt/mycephfs/projects/teamA

# Verify quota
getfattr -n ceph.quota.max_bytes /mnt/mycephfs/projects/teamA
```

### 8.3. Snapshots

CephFS supports snapshots at the directory level.

```bash
# Create a snapshot of /data
mkdir /mnt/mycephfs/data/.snap/snapshot_2026_05_02

# Restore a file from snapshot
cp /mnt/mycephfs/data/.snap/snapshot_2026_05_02/lost_file.txt /mnt/mycephfs/data/

# Remove snapshot
rmdir /mnt/mycephfs/data/.snap/snapshot_2026_05_02
```

### 8.4. Scaling MDS

If you experience high load, you can increase the number of active MDS ranks.

```bash
# Increase max_mds to 2 (requires at least 2 standby MDS daemons)
ceph fs set mycephfs max_mds 2

# Verify
ceph mds stat
# Should show 2 active ranks if load is distributed
```

---

## 9. Phase 7: Troubleshooting, Cleanup & Decommissioning

### 9.1. Common Issues & Solutions

| Issue | Symptom | Solution |
| :--- | :--- | :--- |
| **Mount Hangs** | `mount` command freezes. | Check Firewall (Ports 6789, 6800-7300). Check `dmesg` for network errors. Ensure MON is reachable. |
| **Permission Denied** | `touch` fails with EACCES. | Check Ceph Auth Caps. Ensure the keyring matches the user specified in mount options. |
| **Slow Performance** | High latency on file ops. | Check if Metadata Pool is on SSD. Check `ceph mds stat` for laggy MDS. Check network congestion. |
| **MDS Crash** | `ceph -s` shows MDS down. | Check logs: `ceph orch logs mds.<id>`. Often caused by OOM (Out of Memory). Increase RAM for MDS nodes. |

### 9.2. Removing CephFS (Decommissioning)

If you need to remove CephFS entirely, follow this strict order to prevent data loss.

**Warning:** This deletes all data in CephFS.

#### Step 1: Unmount All Clients
Ensure no clients are writing to the filesystem.
```bash
umount /mnt/mycephfs
```

#### Step 2: Remove MDS Service
```bash
ceph orch rm mds.mycephfs
```

#### Step 3: Remove Filesystem
```bash
ceph fs rm mycephfs --yes-i-really-mean-it
```

#### Step 4: Remove Pools
```bash
ceph osd pool rm cephfs_metadata cephfs_metadata --yes-i-really-really-mean-it
ceph osd pool rm cephfs_data cephfs_data --yes-i-really-really-mean-it
```

#### Step 5: Clean Up Keys (Optional)
```bash
ceph auth del client.manila
ceph auth del client.cephfs_client
```

### 9.3. Handling Stray Daemons

As seen in our initial troubleshooting, `cephadm` may report "Stray Daemons" if OSDs were manually added or left over from a previous setup.

**Resolution:**
1.  Identify stray OSDs: `ceph orch device ls --hostname <host>`
2.  If they are not managed by cephadm, remove them from the cluster map:
    ```bash
    ceph osd out <osd-id>
    ceph osd rm <osd-id>
    ```
3.  Re-add them via cephadm:
    ```bash
    ceph orch apply osd --all-available-devices
    ```

---

## 10. Best Practices & Security Hardening

### 10.1. Security Best Practices
1.  **Least Privilege:** Never use the `admin` keyring for clients. Create specific users with caps limited to their required pools and paths.
    ```bash
    ceph auth caps client.app_user mds 'allow rw path=/app_data' osd 'allow rw pool=cephfs_data'
    ```
2.  **Encryption:** Enable encryption at rest (BlueStore encryption) and in transit (msgr2 protocol).
    *   *In Transit:* Ensure `msgr2` is enabled in `ceph.conf` (`ms_bind_msgr2 = true`).
3.  **Network Isolation:** Place CephFS traffic on a separate VLAN from public internet traffic.

### 10.2. Performance Tuning
1.  **Metadata on SSD:** This is the single most important factor for CephFS performance.
2.  **Client Cache:** Tune client cache size in `/etc/ceph/ceph.conf`:
    ```ini
    [client]
    rsize = 1048576
    wsize = 1048576
    readahead_max_kb = 4096
    ```
3.  **MDS Cache Size:** If MDS is memory-bound, increase `mds_cache_memory_limit` in the MDS config.

### 10.3. Backup Strategy
1.  **Snapshots:** Use CephFS snapshots for quick recovery of accidental deletions.
2.  **Off-site Copy:** Use `rsync` or `robocopy` to mirror critical directories to an external RGW bucket or another storage system periodically.

---

## Appendix: Quick Reference Command Sheet

| Task | Command |
| :--- | :--- |
| **Check FS Status** | `ceph fs status` |
| **List Clients** | `ceph fs session ls <fs_name>` |
| **Create Subvolume** | `ceph fs subvolume create <fs_name> <subvol_name>` |
| **Mount Kernel** | `mount -t ceph <MON>:6789:/ /mnt -o name=<user>,secretfile=<key>` |
| **Mount NFS** | `mount -t nfs <NFS_IP>:/<fs_name> /mnt` |
| **Set Quota** | `setfattr -n ceph.quota.max_bytes -v <bytes> <dir>` |
| **Log Tail** | `ceph orch logs mds.<id>` |

---

*This guide represents a consolidated, professional approach to implementing CephFS in a production environment, derived from real-world troubleshooting and deployment scenarios. Always test in a non-production lab before applying changes to live infrastructure.*