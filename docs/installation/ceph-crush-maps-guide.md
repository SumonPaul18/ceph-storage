# ceph-crush-hierarchy-guide.md

---

# Professional Practical Guide: Ceph CRUSH Map Hierarchy, Rules, and Device Class Management

**Version:** 1.0  
**Target Ceph Version:** Reef (v18.x) and later  
**Context:** Managing Hybrid Storage Clusters (SSD/HDD/NVMe) with Logical Grouping (Bare-Metal vs. VM) using CRUSH Maps and Rules.

## Table of Contents

1. [Introduction and Core Concepts](#1-introduction-and-core-concepts)
2. [Prerequisites and Environment Setup](#2-prerequisites-and-environment-setup)
3. [Inspecting and Backing Up CRUSH Map](#3-inspecting-and-backing-up-crush-map)
4. [Managing Device Classes](#4-managing-device-classes)
5. [Modifying CRUSH Hierarchy (Buckets & Hosts)](#5-modifying-crush-hierarchy-buckets--hosts)
6. [Creating and Managing Custom CRUSH Rules](#6-creating-and-managing-custom-crush-rules)
7. [Pool Creation and Rule Assignment](#7-pool-creation-and-rule-assignment)
8. [Verification and Testing Data Placement](#8-verification-and-testing-data-placement)
9. [Maintenance: Adding New Hosts and OSDs](#9-maintenance-adding-new-hosts-and-osds)
10. [Cleanup: Removing Down OSDs and Unused Resources](#10-cleanup-removing-down-osds-and-unused-resources)
11. [Troubleshooting Common Issues](#11-troubleshooting-common-issues)

---

## 1. Introduction and Core Concepts

This guide provides a step-by-step practical approach to managing the **CRUSH (Controlled Replication Under Scalable Hashing)** map in a Ceph cluster. The primary goal is to organize storage devices into logical categories such as **SSD**, **HDD**, **NVMe**, and logical environments like **Bare-Metal** and **Virtual Machines (VM)**.

### Key Concepts
*   **CRUSH Map:** The algorithmic map that determines where data is placed. It consists of a hierarchy of buckets.
*   **Buckets:** Containers in the hierarchy (e.g., `root`, `rack`, `host`). Every level except OSD is a bucket.
*   **Device Classes:** Labels automatically or manually assigned to OSDs (e.g., `ssd`, `hdd`, `nvme`) to filter devices without changing the physical hierarchy.
*   **CRUSH Rules:** Instructions that tell Ceph which devices to use for a specific pool based on device class and failure domain (e.g., "Use only SSDs in the Bare-Metal rack").

### Why Use This Approach?
In modern hybrid clusters, you often have mixed media (fast SSDs for performance, large HDDs for capacity) and mixed environments (physical bare-metal servers and virtualized nodes). Using **Device Classes** combined with **Custom CRUSH Rules** allows you to keep the hierarchy simple while enforcing strict data placement policies.

---

## 2. Prerequisites and Environment Setup

Before modifying the CRUSH map, ensure you have the necessary access and tools.

### Requirements
*   A running Ceph cluster (Reef v18+ recommended).
*   Root or sudo access to a Ceph admin node (typically a Monitor or Manager node).
*   `ceph` command-line tool installed.
*   Basic understanding of your hardware layout (which hosts are bare-metal, which are VMs, and what disk types they have).

### Verification of Ceph Installation
Ensure the Ceph CLI is available and connected to the cluster.

#### Check Ceph Version
**Description:** Verify the installed Ceph version to ensure compatibility with the commands used in this guide.
```bash
ceph --version
```
*   **Explanation:** Displays the Ceph daemon version. Ensure it is v18 (Reef) or later for best support of device classes and advanced CRUSH features.

#### Check Cluster Health
**Description:** Ensure the cluster is in a healthy state before making structural changes.
```bash
ceph health detail
```
*   **Explanation:** Shows any current warnings or errors. Ideally, the status should be `HEALTH_OK`. If there are `HEALTH_WARN` issues (like down OSDs), note them for cleanup later.

---

## 3. Inspecting and Backing Up CRUSH Map

Always inspect and backup the current CRUSH map before making modifications. This allows you to revert changes if something goes wrong.

### Step 3.1: View Current CRUSH Hierarchy
**Description:** Display the current tree structure of hosts, racks, and OSDs.
```bash
ceph osd tree
```
*   **Explanation:** Lists all buckets (roots, racks, hosts) and OSDs with their status (`up`/`down`), weight, and device class (`ssd`/`hdd`).
*   **Key Columns:**
    *   `ID`: Negative IDs are buckets, positive IDs are OSDs.
    *   `CLASS`: The device class (ssd, hdd, nvme).
    *   `TYPE`: The bucket type (root, rack, host, osd).
    *   `STATUS`: Whether the OSD is up or down.

### Step 3.2: Backup the CRUSH Map
**Description:** Export the current CRUSH map to a binary file for safekeeping.
```bash
ceph osd getcrushmap -o /tmp/crush-map-backup.bin
```
*   **Explanation:** Saves the compiled CRUSH map to `/tmp/crush-map-backup.bin`.
*   **Arguments:**
    *   `-o <file>`: Output file path.

### Step 3.3: List Existing CRUSH Rules
**Description:** View all currently defined CRUSH rules.
```bash
ceph osd crush rule ls
```
*   **Explanation:** Lists rule names (e.g., `replicated_rule`, `ssd`). These rules determine how data is distributed for pools.

### Step 3.4: Dump Specific Rule Details
**Description:** Inspect the definition of a specific rule to understand its logic.
```bash
ceph osd crush rule dump <rule_name>
```
*   **Example:**
```bash
ceph osd crush rule dump replicated_rule
```
*   **Explanation:** Shows the steps (take, chooseleaf, emit) and parameters (min_size, max_size, device class) of the rule.
*   **Key Fields:**
    *   `step take`: The starting bucket (e.g., `default`).
    *   `step chooseleaf`: How devices are selected (e.g., `type host`, `class ssd`).

---

## 4. Managing Device Classes

Device classes allow you to tag OSDs as `ssd`, `hdd`, or `nvme`. Ceph often auto-detects these, but manual verification and correction may be needed.

### Step 4.1: List Device Classes
**Description:** View all available device classes and which OSDs belong to them.
```bash
ceph osd crush class ls
```
*   **Explanation:** Lists classes like `hdd`, `ssd`, `nvme`.

### Step 4.2: List OSDs by Class
**Description:** See which OSDs are assigned to a specific class.
```bash
ceph osd crush class ls-osd <class_name>
```
*   **Example:**
```bash
ceph osd crush class ls-osd ssd
```
*   **Explanation:** Lists all OSD IDs tagged as `ssd`. Use this to verify if your SSDs are correctly identified.

### Step 4.3: Set or Change Device Class
**Description:** Manually assign a device class to an OSD if auto-detection failed or was incorrect.
```bash
ceph osd crush set-device-class <class_name> <osd_id_1> [<osd_id_2> ...]
```
*   **Example:**
```bash
ceph osd crush set-device-class nvme osd.10 osd.11
```
*   **Explanation:** Tags `osd.10` and `osd.11` as `nvme`.
*   **Arguments:**
    *   `<class_name>`: The class to assign (e.g., `ssd`, `hdd`, `nvme`).
    *   `<osd_id>`: One or more OSD IDs.

### Step 4.4: Remove Device Class
**Description:** Remove a class assignment from an OSD (resets to default detection).
```bash
ceph osd crush rm-device-class <osd_id_1> [<osd_id_2> ...]
```
*   **Example:**
```bash
ceph osd crush rm-device-class osd.10
```
*   **Explanation:** Removes the manual class tag from `osd.10`. Ceph will re-evaluate its class based on hardware properties.

---

## 5. Modifying CRUSH Hierarchy (Buckets & Hosts)

This section covers creating logical groups (Racks/Buckets) for **Bare-Metal** and **VM** environments and moving hosts into them.

### Step 5.1: Create Custom Buckets (Racks)
**Description:** Create new top-level buckets to represent logical environments. We use the `rack` type for simplicity, but they represent environments.

#### Create Bare-Metal Bucket
```bash
ceph osd crush add-bucket bare-metal rack
```
*   **Explanation:** Creates a bucket named `bare-metal` of type `rack`.

#### Create VM Bucket
```bash
ceph osd crush add-bucket vm rack
```
*   **Explanation:** Creates a bucket named `vm` of type `rack`.

### Step 5.2: Move Buckets under Root
**Description:** Attach the new buckets to the main `default` root.

#### Move Bare-Metal to Root
```bash
ceph osd crush move bare-metal root=default
```
*   **Explanation:** Places the `bare-metal` bucket directly under the `default` root.

#### Move VM to Root
```bash
ceph osd crush move vm root=default
```
*   **Explanation:** Places the `vm` bucket directly under the `default` root.

### Step 5.3: Move Hosts to Appropriate Buckets
**Description:** Move existing hosts into their respective logical environment buckets.

#### Move Bare-Metal Hosts
**Description:** Move physical servers (e.g., `ceph4`) to the `bare-metal` bucket.
```bash
ceph osd crush move ceph4 rack=bare-metal
```
*   **Explanation:** Moves host `ceph4` into the `bare-metal` rack.
*   **Note:** Replace `ceph4` with your actual bare-metal hostnames.

#### Move VM Hosts
**Description:** Move virtualized nodes (e.g., `ceph1`, `ceph2`, `ceph3`) to the `vm` bucket.
```bash
ceph osd crush move ceph1 rack=vm
ceph osd crush move ceph2 rack=vm
ceph osd crush move ceph3 rack=vm
```
*   **Explanation:** Moves hosts `ceph1`, `ceph2`, and `ceph3` into the `vm` rack.

### Step 5.4: Verify Hierarchy Changes
**Description:** Confirm that hosts are now under the correct logical buckets.
```bash
ceph osd tree
```
*   **Expected Output Structure:**
    ```text
    root default
       ├── rack bare-metal
       │   └── host ceph4
       │       ├── osd.13 (hdd)
       │       └── osd.14 (hdd)
       └── rack vm
           ├── host ceph1
           │   ├── osd.11 (hdd)
           │   └── osd.12 (ssd)
           ├── host ceph2
           │   ├── osd.1 (hdd)
           │   └── osd.7 (ssd)
           └── host ceph3
               ├── osd.0 (hdd)
               └── osd.6 (ssd)
    ```

---

## 6. Creating and Managing Custom CRUSH Rules

Now that the hierarchy is organized, we create rules that target specific combinations of **Environment (Rack)** and **Device Class**.

### Step 6.1: Create Rule for Bare-Metal HDD
**Description:** Create a rule that selects only `hdd` devices from the `bare-metal` rack.
```bash
ceph osd crush rule create-replicated rule-bm-hdd bare-metal host hdd
```
*   **Explanation:**
    *   `rule-bm-hdd`: Name of the new rule.
    *   `bare-metal`: The root bucket to start from (takes only this rack).
    *   `host`: Failure domain (ensures replicas are on different hosts).
    *   `hdd`: Device class filter (selects only HDDs).

### Step 6.2: Create Rule for VM SSD
**Description:** Create a rule that selects only `ssd` devices from the `vm` rack.
```bash
ceph osd crush rule create-replicated rule-vm-ssd vm host ssd
```
*   **Explanation:**
    *   `rule-vm-ssd`: Name of the new rule.
    *   `vm`: The root bucket to start from.
    *   `host`: Failure domain.
    *   `ssd`: Device class filter.

### Step 6.3: Create Rule for VM HDD
**Description:** Create a rule that selects only `hdd` devices from the `vm` rack.
```bash
ceph osd crush rule create-replicated rule-vm-hdd vm host hdd
```
*   **Explanation:** Similar to above, but filters for `hdd` in the `vm` rack.

### Step 6.4: Create Rule for Bare-Metal SSD (Optional/Future)
**Description:** If you add SSDs to bare-metal hosts later, use this rule. Currently, it may fail if no SSDs exist in that rack.
```bash
ceph osd crush rule create-replicated rule-bm-ssd bare-metal host ssd
```
*   **Note:** Only use this if you actually have SSDs in the `bare-metal` rack. Otherwise, data placement will fail (empty OSD list).

### Step 6.5: List and Verify New Rules
**Description:** Confirm the new rules are created.
```bash
ceph osd crush rule ls
```
*   **Expected Output:** Should include `rule-bm-hdd`, `rule-vm-ssd`, `rule-vm-hdd`, etc.

### Step 6.6: Dump Rule for Verification
**Description:** Inspect the generated rule to ensure it targets the correct rack and class.
```bash
ceph osd crush rule dump rule-vm-ssd
```
*   **Expected Output:** Look for `step take vm` and `class ssd` in the steps.

---

## 7. Pool Creation and Rule Assignment

Create pools that utilize the custom CRUSH rules to enforce data placement.

### Step 7.1: Enable Pool Deletion (Temporary)
**Description:** Allow pool deletion for cleanup purposes. Disable this after use for safety.
```bash
ceph config set mon mon_allow_pool_delete true
```
*   **Explanation:** Sets the monitor configuration to allow pool destruction.

### Step 7.2: Create Pool for VM SSD Performance
**Description:** Create a high-performance pool using VM SSDs.
```bash
ceph osd pool create vm-ssd-pool 32 32 replicated rule-vm-ssd
```
*   **Explanation:**
    *   `vm-ssd-pool`: Name of the pool.
    *   `32 32`: PG_num and PGP_num (adjust based on cluster size; 32 is small for testing, use calculator for production).
    *   `replicated`: Pool type.
    *   `rule-vm-ssd`: The custom CRUSH rule to use.

### Step 7.3: Set Pool Size (Replicas)
**Description:** Define the number of data copies.
```bash
ceph osd pool set vm-ssd-pool size 3
ceph osd pool set vm-ssd-pool min_size 2
```
*   **Explanation:**
    *   `size 3`: Keeps 3 copies of each object.
    *   `min_size 2`: Requires at least 2 copies to accept writes (safety margin).

### Step 7.4: Create Pool for Bare-Metal Capacity (HDD)
**Description:** Create a high-capacity pool using Bare-Metal HDDs.
```bash
ceph osd pool create bm-hdd-pool 64 64 replicated rule-bm-hdd
ceph osd pool set bm-hdd-pool size 3
ceph osd pool set bm-hdd-pool min_size 2
```
*   **Explanation:** Uses `rule-bm-hdd` to place data only on HDDs in the bare-metal rack.

### Step 7.5: Assign Application Type
**Description:** Tag the pool with its intended use (e.g., RBD, CephFS, RGW).
```bash
ceph osd pool application enable vm-ssd-pool rbd
ceph osd pool application enable bm-hdd-pool rbd
```
*   **Explanation:** Enables RBD (Block Device) features for the pools. Other options: `cephfs`, `rgw`.

### Step 7.6: Disable Pool Deletion (Safety)
**Description:** Re-disable pool deletion to prevent accidental data loss.
```bash
ceph config set mon mon_allow_pool_delete false
```

---

## 8. Verification and Testing Data Placement

Verify that the rules are working correctly by checking where data would be placed.

### Step 8.1: Test Data Placement for VM SSD Pool
**Description:** Simulate placing an object in the `vm-ssd-pool` to see which OSDs are selected.
```bash
ceph osd map vm-ssd-pool test-object-1
```
*   **Expected Output:**
    ```text
    osdmap e... pool 'vm-ssd-pool' ... -> up ([6, 7, 12], p6) acting ([6, 7, 12], p6)
    ```
*   **Verification:**
    1.  Note the OSD IDs (e.g., 6, 7, 12).
    2.  Check if they are SSDs:
        ```bash
        ceph osd crush class ls-osd ssd
        ```
    3.  Check if they are in the `vm` rack:
        ```bash
        ceph osd tree | grep -A 5 "rack vm"
        ```
    *   **Success Criteria:** All listed OSDs must be `ssd` class and under `rack vm`.

### Step 8.2: Test Data Placement for Bare-Metal HDD Pool
**Description:** Simulate placing an object in the `bm-hdd-pool`.
```bash
ceph osd map bm-hdd-pool test-object-1
```
*   **Expected Output:**
    ```text
    osdmap e... pool 'bm-hdd-pool' ... -> up ([13, 14, ...], p13) acting ([13, 14, ...], p13)
    ```
*   **Verification:**
    1.  Note the OSD IDs (e.g., 13, 14).
    2.  Check if they are HDDs:
        ```bash
        ceph osd crush class ls-osd hdd
        ```
    3.  Check if they are in the `bare-metal` rack:
        ```bash
        ceph osd tree | grep -A 5 "rack bare-metal"
        ```
    *   **Success Criteria:** All listed OSDs must be `hdd` class and under `rack bare-metal`.

### Step 8.3: Troubleshooting Empty OSD Lists
**Issue:** If `ceph osd map` returns `up ([], p-1)`, it means no OSDs match the rule criteria.
**Cause:** Usually, the specified rack does not contain devices of the specified class (e.g., trying to use `rule-bm-ssd` when `bare-metal` rack has only HDDs).
**Solution:**
1.  Verify device classes in the rack:
    ```bash
    ceph osd tree | grep -A 10 "bare-metal"
    ```
2.  If no SSDs exist in `bare-metal`, do not use `rule-bm-ssd`. Use `rule-bm-hdd` instead.
3.  Add SSDs to the bare-metal hosts or adjust the rule to target a different rack.

---

## 9. Maintenance: Adding New Hosts and OSDs

When expanding the cluster, new hosts and OSDs must be integrated into the custom hierarchy.

### Step 9.1: Add New OSDs (Automatic)
**Description:** When you provision a new OSD using `ceph-volume`, it is automatically added to the CRUSH map under its hostname.
```bash
# Example on new host ceph5
ceph-volume lvm create --data /dev/sdb
```
*   **Note:** By default, the new host `ceph5` will be placed directly under `root default`, **not** in your custom `bare-metal` or `vm` racks.

### Step 9.2: Move New Host to Correct Rack
**Description:** Manually move the new host to the appropriate logical environment.

#### If ceph5 is Bare-Metal with HDD:
```bash
ceph osd crush move ceph5 rack=bare-metal
```

#### If ceph5 is VM with SSD:
```bash
ceph osd crush move ceph5 rack=vm
```

### Step 9.3: Verify Device Class of New OSDs
**Description:** Ensure the new OSDs are correctly classified.
```bash
ceph osd tree | grep ceph5
```
*   **Action:** If the class is wrong (e.g., an SSD labeled as HDD), correct it:
    ```bash
    ceph osd crush set-device-class ssd osd.<new_id>
    ```

### Step 9.4: Trigger Rebalancing (Automatic)
**Description:** Ceph automatically starts rebalancing data to include the new OSDs based on the CRUSH rules. Monitor progress:
```bash
ceph -w
```
*   **Note:** Rebalancing can impact performance. Schedule during low-usage periods if possible.

---

## 10. Cleanup: Removing Down OSDs and Unused Resources

Remove failed OSDs and unused test resources to keep the cluster clean and healthy.

### Step 10.1: Identify Down OSDs
**Description:** List all OSDs and identify those marked `down`.
```bash
ceph osd tree | grep down
```
*   **Example Output:**
    ```text
    2    hdd   0.29300              osd.2      down         0  1.00000
    4    hdd   0.29300              osd.4      down         0  1.00000
    8    ssd   0.19530              osd.8      down         0  1.00000
    ```

### Step 10.2: Remove Down OSDs from Cluster
**Description:** Safely remove failed OSDs from the CRUSH map and authentication database.

#### Mark OSDs Out
```bash
ceph osd out osd.2 osd.4 osd.8
```
*   **Explanation:** Tells Ceph to stop using these OSDs for new data. Wait for data to migrate off them (monitor with `ceph -w`).

#### Remove from CRUSH Map
```bash
ceph osd crush remove osd.2
ceph osd crush remove osd.4
ceph osd crush remove osd.8
```
*   **Explanation:** Removes the OSD entries from the hierarchy.

#### Remove Authentication Keys
```bash
ceph auth del osd.2
ceph auth del osd.4
ceph auth del osd.8
```
*   **Explanation:** Removes the OSD's ability to authenticate with the cluster.

#### Remove OSD IDs
```bash
ceph osd rm osd.2
ceph osd rm osd.4
ceph osd rm osd.8
```
*   **Explanation:** Completely removes the OSD records from the cluster map.

### Step 10.3: Delete Unused Test Pools
**Description:** Remove temporary pools created for testing.

#### Enable Pool Deletion
```bash
ceph config set mon mon_allow_pool_delete true
```

#### Delete Pool
```bash
ceph osd pool delete <pool_name> <pool_name> --yes-i-really-really-mean-it
```
*   **Example:**
    ```bash
    ceph osd pool delete test-vm-ssd-pool test-vm-ssd-pool --yes-i-really-really-mean-it
    ```

#### Disable Pool Deletion
```bash
ceph config set mon mon_allow_pool_delete false
```

### Step 10.4: Remove Unused CRUSH Rules
**Description:** Remove rules that are no longer associated with any pool.

#### Check Rule Usage
```bash
ceph osd dump | grep <rule_name>
```
*   **Explanation:** If no pool lists the rule, it is safe to remove.

#### Remove Rule
```bash
ceph osd crush rule rm <rule_name>
```
*   **Example:**
    ```bash
    ceph osd crush rule rm rule-bm-ssd
    ```
    *(Only if no pool uses it and/or no SSDs exist in bare-metal rack)*

---

## 11. Troubleshooting Common Issues

### Issue 1: `not enough osds` Error when Creating Pool
**Symptom:** `Error ENOSPC: not enough osds` or `pgs are stuck inactive`.
**Cause:** The CRUSH rule cannot find enough devices to satisfy the replica count. For example, trying to create a pool with `size 3` using `rule-bm-hdd` when the `bare-metal` rack has only 2 OSDs.
**Solution:**
1.  Reduce pool size:
    ```bash
    ceph osd pool set <pool_name> size 2
    ```
2.  Add more OSDs to the targeted rack/class.
3.  Change the failure domain in the rule (e.g., from `host` to `osd`) – *Not recommended for production*.

### Issue 2: Data Not Balancing to New OSDs
**Symptom:** New OSDs remain empty or receive very little data.
**Cause:**
1.  The new OSDs are not in the correct rack/class targeted by the pool's rule.
2.  The new OSDs have a weight of 0.
**Solution:**
1.  Verify placement:
    ```bash
    ceph osd tree | grep <new_host>
    ```
2.  Ensure the host is moved to the correct rack (Step 9.2).
3.  Check weight:
    ```bash
    ceph osd df
    ```
    If weight is 0, reweight it:
    ```bash
    ceph osd crush reweight osd.<id> <auto-calculated-weight>
    ```

### Issue 3: `mon_allow_pool_delete` Error
**Symptom:** `Error EPERM: pool deletion is disabled`.
**Cause:** Safety feature preventing accidental pool deletion.
**Solution:**
1.  Enable deletion:
    ```bash
    ceph config set mon mon_allow_pool_delete true
    ```
2.  Delete the pool.
3.  Disable deletion:
    ```bash
    ceph config set mon mon_allow_pool_delete false
    ```

### Issue 4: Incorrect Device Class Detection
**Symptom:** An SSD is listed as `hdd` in `ceph osd tree`.
**Cause:** Auto-detection failed or was overridden.
**Solution:**
1.  Manually set the class:
    ```bash
    ceph osd crush set-device-class ssd osd.<id>
    ```
2.  Verify:
    ```bash
    ceph osd tree | grep osd.<id>
    ```

---

## Conclusion

This guide has provided a comprehensive, practical workflow for managing Ceph CRUSH maps with logical grouping. By separating **Bare-Metal** and **VM** environments into distinct racks and using **Device Classes** (`ssd`, `hdd`) within CRUSH rules, you achieve:

1.  **Performance Isolation:** Fast SSDs are used only for performance-critical pools.
2.  **Capacity Optimization:** Large HDDs are used for bulk storage.
3.  **Logical Organization:** The hierarchy reflects your physical/virtual infrastructure.
4.  **Scalability:** New hosts can be easily added to the correct logical group.

Always verify changes with `ceph osd tree` and `ceph osd map` before deploying to production workloads. Regularly clean up down OSDs and unused rules to maintain cluster health.
