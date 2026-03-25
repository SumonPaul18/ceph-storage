# Ceph Storage Mastery: From Single Image to Production-Grade Cluster
### A Practical Guide for System Administrators and DevOps Engineers

**Version:** 1.0 (Updated for 2026 Infrastructure Standards)  
**Author:** Based on real-world troubleshooting and architecture discussions  
**Target Audience:** System Administrators, DevOps Engineers, Cloud Architects  

---

## 📑 Table of Contents

1. [Introduction: The Modern Storage Challenge](#1-introduction-the-modern-storage-challenge)
2. [Multi-Client Access Strategies: Beyond Simple Mounting](#2-multi-client-access-strategies-beyond-simple-mounting)
3. [OpenStack Integration: The Automated Symphony](#3-openstack-integration-the-automated-symphony)
4. [Deep Dive: Data Distribution & The CRUSH Algorithm](#4-deep-dive-data-distribution--the-crush-algorithm)
5. [Physical Reality: Inspecting OSDs and Raw Data](#5-physical-reality-inspecting-osds-and-raw-data)
6. [Failure Scenarios: Node Down, Recovery, and Rebalancing](#6-failure-scenarios-node-down-recovery-and-rebalancing)
7. [Cluster Evolution: Replacing Nodes and Scaling Out](#7-cluster-evolution-replacing-nodes-and-scaling-out)
8. [Mixed Media Architecture: HDD + SSD Best Practices](#8-mixed-media-architecture-hdd--ssd-best-practices)
9. [Operational Maintenance: Daily Checks and Health Monitoring](#9-operational-maintenance-daily-checks-and-health-monitoring)
10. [Conclusion: Building Resilient Storage Infrastructure](#10-conclusion-building-resilient-storage-infrastructure)

---

## 1. Introduction: The Modern Storage Challenge

In today's infrastructure landscape, storage is no longer just about "where to save files." It is about **availability**, **scalability**, and **intelligence**. Imagine you are running a critical application where downtime means financial loss. You have built a 3-node Ceph cluster with 9 physical disks (a mix of HDDs and SSDs). You have created a pool, an RBD image, and stored data. 

But the real questions begin now: 
*   *Where exactly did that data go?*
*   *What happens if a server room loses power and one node dies?*
*   *Can five different servers read that same file at the exact same time without corrupting it?*
*   *How does this magic integrate with OpenStack to spin up hundreds of VMs?*

This guide is not a textbook definition list. It is a **story of data journey**—from the moment you write a file to the moment a disk fails and heals itself. We will walk through practical scenarios, command-line verifications, and architectural decisions based on real-world production environments. We will avoid complex scripts and focus on **understanding the behavior** of your cluster so you can troubleshoot like an expert.

---

## 2. Multi-Client Access Strategies: Beyond Simple Mounting

### The Scenario
You have a single RBD image containing critical data. Your manager asks: *"Can we mount this on all five web servers simultaneously so they all see the same files?"*

### The Theory: The Danger of Naive Mounting
If you simply map an RBD image (`rbd map`) and format it with a standard filesystem like `ext4` or `xfs`, then try to mount it on two servers at once, **you will destroy your data**. Standard filesystems do not know other servers exist. They cache metadata locally. If Server A writes a file and Server B tries to read it without knowing the cache changed, the filesystem structure collapses [[1]].

### Practical Solution 1: Cluster-Aware Filesystems (GFS2/OCFS2)
To safely mount a block device (RBD) on multiple nodes with **Read/Write** access, you need a "Cluster-Aware" filesystem. These filesystems use a locking mechanism (like DLM - Distributed Lock Manager) to ensure only one server writes to a specific block at a time.

**Real-World Use Case:** High-Availability Database clusters or shared home directories for users.

**Practical Steps:**
1.  **Map the RBD on all nodes:**
    ```bash
    rbd map <pool-name>/<image-name>
    # Output: /dev/rbd0
    ```
2.  **Create GFS2 Filesystem (Run on ONE node only):**
    *Note: This requires `dlm` and `corosync` to be running.*
    ```bash
    mkfs.gfs2 -p lock_dlm -t my_cluster:my_share /dev/rbd0
    ```
3.  **Mount on All Nodes:**
    ```bash
    mount -t gfs2 /dev/rbd0 /mnt/shared_data
    ```
    *Verification:* Create a file on Node 1 (`touch /mnt/shared_data/test.txt`). Check Node 2 immediately. It should appear instantly [[3]].

### Practical Solution 2: Read-Only Mounting
If your clients only need to **read** data (e.g., serving static website content, distributing software packages), you can mount the same RBD image on multiple nodes using standard `ext4`/`xfs` in **Read-Only** mode.

**Command:**
```bash
mount -o ro /dev/rbd0 /mnt/read_only_data
```
*Safety:* Since no one is writing, there is no risk of corruption.

### Practical Solution 3: CephFS (The Native Way)
For most modern shared storage needs, **CephFS** is superior to RBD+GFS2. It is a POSIX-compliant filesystem built directly on top of Ceph's object store. It natively supports thousands of clients mounting the same directory with Read/Write access without needing external lock managers [[20]].

**Why choose CephFS?**
*   No need to manage GFS2/DLM complexity.
*   Better scalability for large numbers of clients.
*   Automatic handling of file locking.

**Implementation Check:**
Ensure you have Metadata Servers (MDS) running:
```bash
ceph orch apply mds <filesystem_name> --placement="3"
ceph fs new <fs_name> <metadata_pool> <data_pool>
```
Then mount on any client:
```bash
ceph-fuse -m <monitor_ip>:6789 /mnt/ceph_share
```

### Decision Matrix

| Requirement | Recommended Approach | Risk Level | Complexity |
| :--- | :--- | :--- | :--- |
| **Shared Read/Write (Few Nodes)** | RBD + GFS2/OCFS2 | Low (if configured correctly) | High |
| **Shared Read-Only (Many Nodes)** | RBD + ext4/xfs (RO flag) | None | Low |
| **Shared Read/Write (Many Nodes)** | **CephFS** (Native) | Low | Medium |
| **VM Disks (OpenStack/K8s)** | RBD (One per VM) | None | Low |
| **Object Access (Web/App)** | RGW (S3 API) | None | Low |

---

## 3. OpenStack Integration: The Automated Symphony

### The Story of Automation
You asked: *"If I need separate RBD images for every client, how does OpenStack handle this?"*

In a manual world, you would run `rbd create` hundreds of times. In the OpenStack world, **you do nothing**. OpenStack acts as a conductor, telling Ceph exactly what to do via its components: **Glance**, **Cinder**, and **Nova**.

### How It Works Behind the Scenes

#### 1. Glance (The Image Library)
When you upload an Ubuntu ISO to OpenStack (`glance image-create`), Glance doesn't save it as a `.iso` file on a hard drive. It converts it into a Ceph RBD object in the `images` pool.
*   **Benefit:** This image becomes a "Gold Master." It sits there safely in the cluster.

#### 2. Cinder (The Volume Manager)
When a user requests a 50GB volume (`cinder create 50`), Cinder talks to Ceph.
*   **Action:** Cinder executes a logical `rbd create` command in the `volumes` pool.
*   **Result:** A new, unique RBD image named `volume-<uuid>` is instantly created. You don't see this command; Cinder hides it.

#### 3. Nova (The Compute Engine)
When you boot a VM:
*   Nova asks Cinder for the volume.
*   Cinder tells Nova: "Here is the RBD image name: `volumes/volume-xyz`."
*   Nova (on the compute node) maps this RBD image internally using the kernel RBD driver or librbd.
*   **Copy-on-Write (CoW):** To save space and time, Nova often creates a thin-provisioned layer on top of the Glance image. The VM boots instantly. As the VM writes data, only the *changes* are written to the new RBD volume. The base image remains untouched [[14]].

### Practical Verification in OpenStack
You can verify this integration without logging into Ceph monitors directly.

**Check Volume Backend:**
```bash
openstack volume show <volume-id>
# Look for 'os-volume-host': it should show something like 'ceph@backend#pool'
```

**Check RBD Images from OpenStack Side:**
While you shouldn't manage them manually, you can inspect what OpenStack created:
```bash
rbd ls -l volumes
# You will see images like 'volume-8234-2342-...' corresponding to your OpenStack volumes.
```

### Why Separate Images Matter
In this integrated model, **every VM gets its own dedicated RBD image**.
*   VM-A uses `volume-uuid-A`.
*   VM-B uses `volume-uuid-B`.
*   **No Conflict:** Since they are separate block devices, VM-A and VM-B can run on the same physical host or different hosts without ever touching each other's data. This eliminates the "multi-mount" corruption risk entirely [[11]].

**Exception - Multi-Attach:**
If you specifically need two VMs to share the *same* volume (e.g., an active-passive cluster), OpenStack Cinder supports `--multiattach`. However, just like the manual RBD scenario, the guest OS inside the VMs must use a cluster-aware filesystem (GFS2) to prevent corruption [[4]].

---

## 4. Deep Dive: Data Distribution & The CRUSH Algorithm

### The Mystery of "Where is my file?"
You saved `important_data.zip` (100MB) to your RBD image. You have 9 disks across 3 nodes. Where is it?
**Answer:** It is nowhere and everywhere.

### The CRUSH Algorithm Explained
Ceph does not use a central lookup table (which would be a bottleneck). Instead, it uses **CRUSH** (Controlled Replication Under Scalable Hashing). Think of it as a deterministic mathematical formula [[29]].

**The Process:**
1.  **Object Division:** Your 100MB file is sliced into small **Objects** (default 4MB each). Let's say you now have 25 objects.
2.  **Hashing:** Ceph takes the name of `Object_1` and runs it through a hash function.
3.  **Placement:** The hash result points to a **Placement Group (PG)**.
4.  **Replication:** The CRUSH rule says "Replicate 3 times." The algorithm calculates exactly which 3 OSDs (Out of your 9) should hold these copies based on the cluster map.
    *   It ensures copies are on **different nodes** (so if one node dies, you don't lose all copies).
    *   It ensures copies are on **different physical disks**.

### Mixed Pool Dynamics (HDD + SSD)
Since you have a **Mixed Pool**, the CRUSH algorithm treats all 9 OSDs as part of the same bucket unless you configured specific **Device Classes**.
*   **Scenario A (Default):** Data is distributed randomly across all 9 disks. Some objects of your file might be on fast SSDs, others on slow HDDs. The overall speed will be limited by the slowest disk involved in the read operation [[42]].
*   **Scenario B (Optimized):** You define CRUSH rules to prefer SSDs for primary copies and HDDs for secondary copies, or separate pools entirely.

### Practical Guide: Tracing Your Data
Want to see exactly where your data lives?

**Step 1: Get the PG ID**
Find out which Placement Group your RBD image belongs to.
```bash
rbd info <pool>/<image-name>
# Look for "block_name_prefix". Then map this to a PG.
# Alternatively, check pool stats:
ceph pg dump | grep <pool-id>
```

**Step 2: Map PG to OSDs**
Once you have a PG ID (e.g., `1.2a`):
```bash
ceph pg map 1.2a
```
**Output Interpretation:**
`up [osd.2, osd.5, osd.8] acting [osd.2, osd.5, osd.8]`
*   This means your data has 3 copies.
*   Copy 1 is on **Node 1, Disk 3** (osd.2).
*   Copy 2 is on **Node 2, Disk 2** (osd.5).
*   Copy 3 is on **Node 3, Disk 3** (osd.8).
*   *Notice:* They are spread across different nodes! This is fault tolerance in action [[30]].

---

## 5. Physical Reality: Inspecting OSDs and Raw Data

### Can I see my files on the disk?
You SSH into Node 1, go to `/var/lib/ceph/osd/ceph-2/`, and look around. Do you see `important_data.zip`?
**No.** And if you do, it won't look like a zip file.

### What You Will See
1.  **Filesystem:** The disk is formatted (usually XFS).
2.  **Directory Structure:** Inside `/var/lib/ceph/osd/ceph-ID/current/`, you will see folders named after PGs (e.g., `1.2a_head/`).
3.  **Objects:** Inside those folders, you see files with names like `rb.0.1234.5678...`. These are the raw **Objects**.
4.  **Content:** If you try to `cat rb.0.1234...`, you will see garbage binary data. Why? Because it's just a 4MB slice of your file, possibly compressed or encoded, without the context of the other slices [[6]].

### How to Properly Inspect Data
Never inspect raw OSD disks for application data. Use the Ceph tools:

**Option A: Mount the RBD (Recommended)**
```bash
sudo rbd map <pool>/<image-name>
sudo mount /dev/rbd0 /mnt/recovery
ls -lh /mnt/recovery
# Now you see your actual files.
```

**Option B: Export the Image**
If you can't mount it, export the whole image to a single file:
```bash
rbd export <pool>/<image-name> /tmp/full_backup.img
# Now you can mount full_backup.img or scan it.
```

**Option C: Rados Tool (Advanced)**
Get an object directly (rarely needed):
```bash
rados -p <pool> get <object-name> /tmp/object_dump
```

### Verification Checklist
*   [ ] **Disk Usage:** Use `ceph osd df tree` to see if data is balanced across all 9 disks. If one disk is 90% full and others are 10%, your CRUSH weights might be wrong.
*   [ ] **File Integrity:** Mount the RBD and run `md5sum` on your files to ensure they match the source.

---

## 6. Failure Scenarios: Node Down, Recovery, and Rebalancing

This is the moment of truth. Production environments *will* fail. How does your 3-node cluster react?

### Scenario 1: Temporary Node Failure (Power Loss/Crash)
**Event:** Node 2 suddenly goes offline.
**Immediate Impact:**
*   3 OSDs (out of 9) disappear from the cluster map.
*   Ceph Monitor detects the failure within seconds.
*   **Status:** `ceph -s` shows `HEALTH_WARN`. It will say "fewer OSDs up than expected" or "degraded PGs".
*   **Data Access:** **Zero Downtime.** Your client machine keeps reading/writing. Why? Because for every lost object on Node 2, there are 2 other healthy copies on Node 1 and Node 3. Ceph automatically redirects traffic to the surviving copies [[31]].

**Recovery (Node Comes Back):**
*   You fix the power, Node 2 boots up.
*   Ceph OSD services start automatically (via systemd).
*   **Backfilling:** Ceph realizes Node 2 is back. It checks: "Did any data change while you were gone?"
    *   If yes, it copies only the changed objects from Node 1/3 back to Node 2.
    *   If no, it just marks the OSD as `up` and `in`.
*   **Status:** Returns to `HEALTH_OK`.

### Scenario 2: Permanent Node Failure (Hardware Death)
**Event:** Node 2's motherboard is fried. It will never return.
**Action Required:**
1.  **Wait:** Give it a few minutes (default `mon_osd_down_out_interval` is usually 300s) to see if it comes back.
2.  **Mark Out:** If it's dead, tell Ceph to stop waiting and start healing.
    ```bash
    ceph osd out osd.3 osd.4 osd.5
    ```
3.  **Rebalancing Begins:** Ceph now knows those 3 OSDs are gone forever. It starts copying data from the remaining survivors to the free space on Node 1 and Node 3 to restore the "3 replicas" rule.
    *   *Note:* Your cluster is now running with reduced redundancy (some objects might only have 2 copies temporarily) until rebalancing finishes. Performance might dip due to heavy network traffic [[39]].

### Practical Monitoring During Failure
Keep this command running in a terminal during a test:
```bash
watch ceph -w
```
*   Watch for `pg recovering`, `backfilling`, or `degraded` messages.
*   Observe the `recovery_ops` counter increasing.

---

## 7. Cluster Evolution: Replacing Nodes and Scaling Out

### Scenario: Replacing a Dead Node with New Hardware
You have Node 1, Node 2 (Dead), Node 3. You buy a new server (Node 4).

**Step-by-Step Replacement:**
1.  **Destroy Old OSD Records:**
    Once data rebalancing is complete and the old node is confirmed dead:
    ```bash
    ceph osd purge <old_osd_id> --yes-i-really-mean-it
    ```
2.  **Prepare New Node:**
    Install Ceph, add keys, and join the cluster.
3.  **Add New OSDs:**
    Plug in the 3 new disks (2 HDD, 1 SSD) to Node 4.
    ```bash
    ceph orch daemon add osd <node4_hostname>:/dev/sdb
    ceph orch daemon add osd <node4_hostname>:/dev/sdc
    ceph orch daemon add osd <node4_hostname>:/dev/nvme0n1
    ```
4.  **Automatic Rebalancing:**
    As soon as the new OSDs are `up` and `in`, CRUSH recalculates. It sees new capacity. It will start moving data from Node 1 and Node 3 to Node 4 to balance the cluster evenly again.
    *   *Result:* You are back to a balanced, fully redundant 3-node cluster (now Node 1, 3, 4) [[32]].

### Key Concept: "Rebalancing" vs. "Recovery"
*   **Recovery:** Restoring lost replicas because a disk failed. (Urgent)
*   **Rebalancing:** Moving data to new disks to ensure even usage. (Background task)
Both happen automatically, but you can tune their speed so they don't choke your network during work hours.

---

## 8. Mixed Media Architecture: HDD + SSD Best Practices

You have a **Mixed Pool** (HDD + SSD). While functional, this setup has pitfalls in production.

### The Problem with Mixed Pools
If you put HDDs and SSDs in the same pool without rules, Ceph might place a "hot" database object on a slow HDD, or a "cold" backup object on a fast SSD. Worse, if a PG spans both, the performance is limited by the HDD [[42]].

### Recommended Strategy: Device Classes & Tiers
Instead of one big mixed pool, modern best practices suggest separating them logically or physically.

**Option A: Separate Pools (Simplest & Most Effective)**
*   Create `fast-pool` (SSD only): For VM boot disks, Databases, Logs.
*   Create `bulk-pool` (HDD only): For Backups, Archives, Cold Storage.
*   **How:** When creating the pool, specify the CRUSH rule that only includes SSD OSDs or HDD OSDs.
    ```bash
    # Example logic (via ceph orch or crushtool)
    ceph osd pool create fast-pool 32 32
    ceph osd pool set fast-pool crush_rule ssd_only_rule
    ```

**Option B: Cache Tiering (Advanced/Legacy)**
Use SSDs as a cache front-end for HDDs. Hot data automatically moves to SSD; cold data sinks to HDD. *Note: This feature is complex and less favored in newer Ceph versions compared to simple separate pools [[45]].*

**Option C: DB/WAL Separation (Bluestore)**
If you haven't already, ensure your **WAL (Write Ahead Log)** and **DB (Metadata)** for the HDD OSDs are stored on the SSDs.
*   In Bluestore (default), you can dedicate a partition of the SSD to act as the WAL/DB for the HDD OSDs. This drastically speeds up write operations on HDDs [[41]].
*   *Check:* `ceph osd metadata <osd_id>` to see if `bluefs_db_dev` points to your SSD.

### Action Plan for Your Cluster
1.  **Audit:** Run `ceph osd tree` and verify device classes (`ceph osd crush rule dump`).
2.  **Segregate:** If possible, create two pools. Move your RBD images for critical VMs to the SSD pool. Move backups to the HDD pool.
3.  **Optimize:** If you must keep one mixed pool, ensure CRUSH rules distribute replicas across different device types for safety, but be aware of performance variance.

---

## 9. Operational Maintenance: Daily Checks and Health Monitoring

Running a Ceph cluster requires a routine. Here is your **Daily/Weekly Checklist**.

### Daily Health Check
Run this every morning:
```bash
ceph -s
```
*   **Look for:** `HEALTH_OK`.
*   **Warning Signs:** `HEALTH_WARN` (degraded PGs, slow requests), `HEALTH_ERR` (data inconsistency).

### Capacity Planning
Check disk usage trends:
```bash
ceph osd df tree
```
*   **Goal:** All OSDs should be within 5-10% of each other in usage.
*   **Alert:** If any OSD is >85% full, add more disks immediately. Ceph performance drops sharply near capacity limits.

### Scrubbing (Data Integrity)
Ceph automatically "scrubs" data to check for bit rot. Verify it's happening:
```bash
ceph pg dump_stuck
# Should return nothing. If not, some PGs are stuck and need attention.
```

### Log Monitoring
Check for hardware errors (critical for HDDs):
```bash
journalctl -u ceph-osd -f
# Look for "I/O error", "sector read error", or "slow ops".
```

### Maintenance Window Tasks
*   **Firmware Updates:** Update disk firmware one OSD at a time.
    1.  `ceph osd out osd.X`
    2.  Wait for rebalance to finish.
    3.  Stop service, update firmware, restart.
    4.  `ceph osd in osd.X`
*   **Kernel Updates:** Reboot nodes one by one. Ceph handles the transient failure gracefully.

---

## 10. Conclusion: Building Resilient Storage Infrastructure

You started with a simple question: *"Where is my data?"* and *"What if a node dies?"*
Now you understand that:
1.  **Data is Everywhere:** It's sliced, diced, and scattered across 9 disks by the CRUSH algorithm for maximum safety.
2.  **Failure is Expected:** Ceph doesn't fear failure; it expects it. A single node death is just a minor blip, handled automatically with zero downtime for your users.
3.  **Integration is Key:** With OpenStack, this complexity is hidden, allowing you to scale from 1 VM to 1000 VMs seamlessly, with each getting its own secure RBD volume.
4.  **Maintenance is Proactive:** By monitoring `ceph -s` and understanding the difference between HDD and SSD roles, you keep the cluster healthy.

**Final Advice for Your Journey:**
*   **Test Failure:** Don't wait for a real crash. Schedule a maintenance window, shut down one node, and watch your applications keep running. This builds confidence.
*   **Document Your Topology:** Keep a diagram of which physical disk corresponds to which OSD ID. When an alert says `osd.5` is failing, you want to know exactly which screwdriver to grab.
*   **Stay Updated:** Ceph evolves fast. Keep your cluster version aligned with stable releases (like Reef or Squid) to benefit from performance improvements and security patches.

Your 3-node, 9-disk cluster is a robust foundation. Treat it with respect, monitor it diligently, and it will serve your infrastructure reliably for years.

---
*End of Guide*
