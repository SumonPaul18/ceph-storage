# ceph-crush-rule-professional-guide.md

# Professional Practical Guide: Ceph CRUSH Rule Management
### Optimizing Storage Performance with SSD, NVMe, and Multi-Tier Architectures

---

## Table of Contents

1. [Introduction & Modern Storage Context](#1-introduction--modern-storage-context)
2. [Pre-Requisites & Environment Validation](#2-pre-requisites--environment-validation)
3. [Understanding Device Classes & Hardware Discovery](#3-understanding-device-classes--hardware-discovery)
4. [Step-by-Step: Creating Custom CRUSH Rules](#4-step-by-step-creating-custom-crush-rules)
5. [Pool Creation & Rule Assignment](#5-pool-creation--rule-assignment)
6. [Operational Verification & Data Placement Testing](#6-operational-verification--data-placement-testing)
7. [Advanced Scenarios: Hybrid & Erasure Coding](#7-advanced-scenarios-hybrid--erasure-coding)
8. [Maintenance, Monitoring & Troubleshooting](#8-maintenance-monitoring--troubleshooting)

---

## 1. Introduction & Modern Storage Context

In modern DevOps and Cloud Infrastructure, storage is no longer just about capacity; it is about **performance tiers**. Ten years ago, we treated all disks the same. Today, a production environment might have ultra-fast NVMe drives for databases, standard SSDs for Virtual Machines (VMs), and high-capacity HDDs for backups and logs.

Ceph handles this through **CRUSH Rules** (Controlled Replication Under Scalable Hashing). Think of a CRUSH Rule as a "traffic law" for your data. It tells Ceph exactly where to place data copies based on the hardware type and failure domains.

**Why do we need custom rules?**
If you mix SSDs and HDDs in one pool without rules, Ceph might accidentally put your high-performance database on a slow hard drive, or worse, try to replicate data across drives that don't exist in that class. By creating specific rules like `ssd-replicated` or `hdd-archive`, you guarantee that critical applications get the speed they need, while bulk storage remains cost-effective.

This guide walks you through the professional workflow of identifying your hardware, creating these rules via the Dashboard and CLI, and managing them in a production-grade cluster.

---

## 2. Pre-Requisites & Environment Validation

Before modifying the storage map, we must ensure the foundation is solid. In a real-world scenario, blindly creating rules on an unstable cluster can lead to data imbalance or performance degradation.

### 2.1 Verify Cluster Health
Always start by ensuring the cluster is healthy (`HEALTH_OK`). If the cluster is recovering from a failure, wait until it stabilizes.

```bash
ceph status
```
> **Explanation**: This command provides a high-level summary of the cluster state, including health status, monitor quorum, and OSD availability.
> *   **Output Focus**: Look for `health HEALTH_OK`. If it says `HEALTH_WARN` or `HEALTH_ERR`, investigate alerts before proceeding.

```bash
ceph -s
```
> **Explanation**: A shorthand version of `ceph status`. It gives a quick snapshot of the cluster's heartbeat.
> *   **Usage**: Use this frequently during operations to ensure the cluster remains stable.

### 2.2 Check Ceph Version
Features related to device classes and CRUSH maps evolve. Ensure you are running a supported version (preferably Quincy, Reef, or Squid for modern features).

```bash
ceph --version
```
> **Explanation**: Displays the installed Ceph version number.
> *   **Why**: Newer versions handle device classes automatically; older versions might require manual tagging.

```bash
cat /etc/os-release
```
> **Explanation**: Shows the underlying operating system details (e.g., Ubuntu 22.04, Rocky Linux 9).
> *   **Context**: Ensures compatibility between the OS kernel and Ceph version.

### 2.3 Access Requirements
Ensure you have administrative access. In enterprise environments, you typically log in as `root` or use `sudo`.

```bash
whoami
```
> **Explanation**: Displays the current logged-in username.
> *   **Requirement**: Should be `root` or a user with `sudo` privileges.

```bash
sudo ceph auth list
```
> **Explanation**: Lists all authentication keys and their capabilities.
> *   **Check**: Ensure your user has `mon` (monitor) and `osd` (object storage daemon) write permissions.

---

## 3. Understanding Device Classes & Hardware Discovery

The most critical step in creating a rule is knowing what hardware you actually have. Ceph needs to know which OSDs (Object Storage Daemons) are SSDs, which are HDDs, and which are NVMe.

### 3.1 List All Devices and Classes
We need to see how Ceph currently classifies your drives.

```bash
ceph osd device ls
```
> **Explanation**: Lists every OSD ID, its associated device path, host name, and the assigned device class.
> *   **Key Columns**: `OSD`, `DEVICE_CLASS` (e.g., hdd, ssd, nvme), `HOST`.
> *   **Goal**: Verify that your fast drives are labeled `ssd` or `nvme` and slow drives are `hdd`.

```bash
ceph osd crush class ls
```
> **Explanation**: Lists the unique device classes currently recognized by the CRUSH map.
> *   **Expected Output**: `hdd`, `ssd`, `nvme`.
> *   **Action**: If your SSDs show up as `hdd` here, they are misclassified, and we must fix this before making rules.

### 3.2 Inspect Physical Disk Properties (Linux Level)
Sometimes Ceph auto-detection fails. We verify using Linux tools to confirm rotational status (HDDs rotate, SSDs/NVMe do not).

```bash
lsblk -d -o NAME,ROTA,TYPE,SIZE
```
> **Explanation**: Lists block devices showing if they are rotational (`ROTA=1` means HDD, `ROTA=0` means SSD/NVMe).
> *   **ROTA 1**: Mechanical Hard Drive.
> *   **ROTA 0**: Solid State Drive or NVMe.

```bash
sudo hdparm -I /dev/sdX | grep "Nominal"
```
> **Explanation**: Checks detailed drive info (replace `sdX` with actual device like `sda`).
> *   **Usage**: Helps identify rotation speed (e.g., 7200rpm) to confirm it's an HDD.

### 3.3 Correcting Device Classes (If Needed)
If Ceph mistakenly labeled an SSD as an HDD, we must correct it manually so our new rule works correctly.

```bash
ceph osd crush device class set osd.0 ssd
```
> **Explanation**: Forces OSD ID `0` to be classified as `ssd`.
> *   **Argument `osd.0`**: The specific OSD identifier.
> *   **Argument `ssd`**: The target class name.
> *   **Impact**: Immediate update to the CRUSH map; data may rebalance.

```bash
ceph osd crush device class set osd.1 hdd
```
> **Explanation**: Forces OSD ID `1` to be classified as `hdd`.
> *   **Scenario**: Useful when mixing drives in a lab or legacy migration.

---

## 4. Step-by-Step: Creating Custom CRUSH Rules

Now that hardware is identified, we create the rules. You can do this via the Dashboard (GUI) or CLI. In production, CLI is preferred for automation and auditing, but the Dashboard is excellent for visualization.

### 4.1 Strategy: Naming Conventions
Professional naming prevents confusion later.
*   **Bad Name**: `rule1`, `ssd`
*   **Good Name**: `ssd-replicated-host`, `hdd-ec-rack`, `nvme-fast-cache`

### 4.2 Creating a Replicated Rule for SSDs (CLI Method)
This creates a rule that stores 3 copies of data on different hosts, specifically targeting SSDs.

```bash
ceph osd crush rule create-replicated ssd-replicated-host default host ssd
```
> **Explanation**: Creates a new replicated CRUSH rule.
> *   **`ssd-replicated-host`**: The unique name of the rule.
> *   **`default`**: The root of the CRUSH tree (usually 'default' unless you have multi-site setups).
> *   **`host`**: The failure domain. Data replicas will be placed on different physical servers.
> *   **`ssd`**: The device class filter. Only OSDs tagged as 'ssd' will be used.

### 4.3 Creating a Rule via Dashboard (GUI Workflow)
Since you provided a screenshot of the Dashboard, here is the professional workflow for that interface:

1.  Navigate to **Cluster > Pools**.
2.  Click the **CRUSH** tab or look for **Create Crush Rule**.
3.  **Name**: Enter `ssd-replicated-host`.
    *   *Tip*: Be descriptive.
4.  **Root**: Select `default`.
    *   *Tip*: Keep this unless you configured custom roots for disaster recovery.
5.  **Failure Domain Type**: Select `host`.
    *   *Critical*: Ensure you have at least 3 hosts. If you only have 1 server with multiple disks, select `osd` instead, or the rule will fail to place data.
6.  **Device Class**: Select `ssd`.
    *   *Verification*: Ensure this matches the output from `ceph osd device ls`.
7.  Click **Create Crush Rule**.

### 4.4 Verifying the New Rule
After creation, always verify the rule exists and inspect its logic.

```bash
ceph osd crush rule ls
```
> **Explanation**: Lists all available CRUSH rules in the cluster.
> *   **Check**: Confirm `ssd-replicated-host` appears in the list.

```bash
ceph osd crush rule dump ssd-replicated-host
```
> **Explanation**: Dumps the JSON configuration of the specific rule.
> *   **Analysis**: Check `min_size`, `max_size`, and `device_class` fields to ensure they match your intent.

```bash
ceph osd crush rule status ssd-replicated-host
```
> **Explanation**: Shows real-time statistics for the rule, such as how many pools use it and data distribution.
> *   **Usage**: Useful for monitoring rule health over time.

---

## 5. Pool Creation & Rule Assignment

A CRUSH rule is useless until it is attached to a **Pool**. Pools are the logical buckets where users store data (RBD images, CephFS, Objects).

### 5.1 Creating a Pool with the New Rule
We create a pool specifically for high-performance VMs using our new SSD rule.

```bash
ceph osd pool create vm-storage-ssd 128 128 replicated ssd-replicated-host
```
> **Explanation**: Creates a new pool named `vm-storage-ssd`.
> *   **`128`**: Number of Placement Groups (PGs) for initial creation.
> *   **`128`**: Number of PGs per pool (legacy argument, often same as first).
> *   **`replicated`**: The type of pool (copies data).
> *   **`ssd-replicated-host`**: The CRUSH rule we just created.

### 5.2 Setting Application Tag
Modern Ceph requires you to declare what application will use the pool (e.g., RBD, CephFS, RGW). This prevents accidental misuse.

```bash
ceph osd pool application enable vm-storage-ssd rbd
```
> **Explanation**: Tags the pool `vm-storage-ssd` for use by the `rbd` (RADOS Block Device) application.
> *   **Options**: `rbd`, `cephfs`, `rgw`, `nvmeof`.
> *   **Importance**: Without this, mounting the pool as a block device may be blocked by safety checks.

### 5.3 Adjusting Pool Size (Replication Factor)
By default, Ceph replicates data 3 times (`size=3`). For a 3-node cluster, this is perfect. For smaller labs or higher durability, adjust it.

```bash
ceph osd pool set vm-storage-ssd size 3
```
> **Explanation**: Sets the number of data copies to 3.
> *   **Logic**: Total copies = 1 original + 2 replicas.
> *   **Constraint**: Must be less than or equal to the number of available hosts in the failure domain.

```bash
ceph osd pool set vm-storage-ssd min_size 2
```
> **Explanation**: Sets the minimum number of copies required to allow I/O.
> *   **Scenario**: If one node fails, you have 2 copies left. I/O continues. If 2 nodes fail, you have 1 copy; I/O stops to prevent split-brain.

### 5.4 Assigning Rule to Existing Pool
If you already have a pool and want to migrate it to the new SSD rule (data will rebalance automatically).

```bash
ceph osd pool set existing-pool-name crush_rule ssd-replicated-host
```
> **Explanation**: Changes the CRUSH rule of an existing pool.
> *   **Effect**: Ceph immediately starts moving data from HDDs (or old locations) to the new SSD targets defined in the rule.
> *   **Warning**: This causes heavy network/disk IO during migration.

---

## 6. Operational Verification & Data Placement Testing

Theory is good; proof is better. We must verify that data is actually landing on the SSDs and distributed across hosts as expected.

### 6.1 Simulating Data Placement
Before writing real data, ask Ceph where it *would* put an object.

```bash
ceph osd map vm-storage-ssd test-object-name
```
> **Explanation**: Simulates placing an object named `test-object-name` in the pool `vm-storage-ssd`.
> *   **Output**: Shows the specific OSD IDs (e.g., `osd.1, osd.4, osd.7`).
> *   **Validation**: Cross-reference these OSD IDs with `ceph osd device ls` to confirm they are indeed SSDs.

### 6.2 Checking Data Distribution
Ensure data is balanced evenly across the SSDs.

```bash
ceph pg dump | grep vm-storage-ssd
```
> **Explanation**: Dumps placement group stats filtered by the pool name.
> *   **Focus**: Look for `active+clean` status. Any `degraded` or `undersized` indicates issues.

```bash
ceph osd df
```
> **Explanation**: Shows disk usage per OSD.
> *   **Analysis**: Check the `AVAIL` and `USE` columns. Ensure SSD OSDs are being utilized and HDD OSDs remain empty (if the rule is working correctly).

### 6.3 Real-World Write Test (RBD)
Create a small image and write data to it to trigger actual movement.

```bash
rbd create --size 1024 --pool vm-storage-ssd test-image-01
```
> **Explanation**: Creates a 1GB block device image in the SSD pool.
> *   **Flag `--size 1024`**: Size in MB.
> *   **Flag `--pool`**: Target pool name.

```bash
rbd bench --io-type write --size 1024 vm-storage-ssd/test-image-01
```
> **Explanation**: Runs a write benchmark on the image.
> *   **Purpose**: Forces data onto the disks and tests IOPS performance.
> *   **Observation**: Watch `ceph -w` to see the write activity and PG states.

---

## 7. Advanced Scenarios: Hybrid & Erasure Coding

In enterprise environments, we often mix replication (for speed) and erasure coding (for capacity).

### 7.1 Scenario: High-Capacity Archive (Erasure Coding)
For backup data, we don't need 3 full copies. We can use Erasure Coding (EC) to save space (e.g., 4 data chunks + 2 parity chunks = 1.5x overhead vs 3x).

```bash
ceph osd erasure-code-profile set my-ec-profile k=4 m=2
```
> **Explanation**: Creates an EC profile named `my-ec-profile`.
> *   **`k=4`**: Number of data chunks.
> *   **`m=2`**: Number of parity chunks.
> *   **Result**: Can lose any 2 OSDs without data loss, but uses less space than replication.

```bash
ceph osd crush rule create-erased hdd-archive-rule default host my-ec-profile hdd
```
> **Explanation**: Creates a CRUSH rule specifically for Erasure Coded data on HDDs.
> *   **`create-erased`**: Command specifically for EC rules.
> *   **`hdd`**: Targets only HDD device class.

### 7.2 Scenario: Multi-Tier Pool (Hot/Cold Data)
You can create two pools: one `ssd-replicated` for active VMs and one `hdd-ec` for snapshots/backups.

```bash
ceph osd pool create backup-storage 64 64 erasure hdd-archive-rule
```
> **Explanation**: Creates a pool using the Erasure Coded rule for efficient backup storage.
> *   **Use Case**: Store daily snapshots of your VMs here to save costs.

### 7.3 Handling Small Clusters (Failure Domain: OSD)
If you only have 1 or 2 physical servers but many disks, `host` failure domain will fail. You must drop down to `osd` level.

```bash
ceph osd crush rule create-replicated ssd-lab-rule default osd ssd
```
> **Explanation**: Creates a rule where replicas are placed on different *disks* within the same server.
> *   **Risk**: If the server power supply fails, all data is lost.
> *   **Benefit**: Allows testing replication logic in a single-node lab.

---

## 8. Maintenance, Monitoring & Troubleshooting

Storage infrastructure requires ongoing care. Rules may need adjustment as hardware ages or expands.

### 8.1 Monitoring Rule Usage
Regularly check which pools are using which rules to avoid configuration drift.

```bash
ceph osd pool ls detail
```
> **Explanation**: Lists all pools with detailed configuration, including the `crush_rule` assigned to each.
> *   **Audit**: Verify that `vm-storage-ssd` is still using `ssd-replicated-host`.

### 8.2 Rebalancing and Backfilling
When you add new SSDs or change a rule, Ceph moves data. Monitor this process to prevent network saturation.

```bash
ceph -w
```
> **Explanation**: Watches the cluster log in real-time.
> *   **Look for**: `recovery`, `backfill`, `peering`.
> *   **Action**: If recovery is too slow, you can limit bandwidth (see below).

```bash
ceph tell osd.* injectargs --osd-max-backfills 1
```
> **Explanation**: Temporarily limits the number of simultaneous backfill operations per OSD to 1.
> *   **Use Case**: During business hours to prevent storage slowness affecting VMs.
> *   **Reset**: Set back to default (usually 10) after maintenance.

### 8.3 Troubleshooting: "No OSDs Available"
If you create a rule but the pool stays stuck in `creating` or `active+remapped`, you might not have enough devices.

```bash
ceph osd crush rule test ssd-replicated-host replicated rbd
```
> **Explanation**: Tests if the rule can successfully map objects given the current cluster topology.
> *   **Error Message**: If it says "unable to select osds", it means you don't have enough SSDs or Hosts to satisfy the `size=3` and `host` failure domain requirements.
> *   **Fix**: Add more SSDs or change failure domain to `osd`.

### 8.4 Removing a Rule
Never delete a rule that is currently in use by a pool. First, move the pool to a different rule.

```bash
ceph osd pool set old-pool crush_rule replicated_rule
```
> **Explanation**: Moves `old-pool` off the custom rule to the default `replicated_rule`.
> *   **Prerequisite**: Wait for data migration to finish (`ceph -w`).

```bash
ceph osd crush rule rm ssd-replicated-host
```
> **Explanation**: Deletes the custom CRUSH rule.
> *   **Safety**: Ceph will refuse to delete the rule if any pool is still actively using it.

### 8.5 Exporting and Backup CRUSH Map
Always backup your CRUSH map before making major changes. It is the "brain" of your storage layout.

```bash
ceph osd crush export -f /tmp/crush-map-backup.json
```
> **Explanation**: Exports the current CRUSH map to a JSON file.
> *   **Best Practice**: Save this file with a date stamp (e.g., `crush-map-2026-05-07.json`) before any maintenance window.

```bash
ceph osd crush import -i /tmp/crush-map-backup.json
```
> **Explanation**: Restores a CRUSH map from a file.
> *   **Disaster Recovery**: Use this if a manual edit corrupts the data placement logic.

---

### Final Thoughts for the Administrator

Managing CRUSH rules is about balancing **performance**, **durability**, and **cost**.
*   Use **SSD Rules** for anything requiring low latency (Databases, Kubernetes PVCs).
*   Use **HDD Rules** for bulk storage (Logs, Backups, Media).
*   Always validate with `ceph osd map` before trusting the setup with production data.

Your private cloud lab with public IPs is the perfect sandbox to test these scenarios. Try breaking things (remove an OSD, simulate a host failure) and watch how your custom rules react. This hands-on experimentation is the fastest way to master Ceph architecture.
