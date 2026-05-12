# ceph-crush-maps-ops-guide.md

---

# Ceph Storage Practical Guide: CRUSH Maps, Rules, Pools, and Crash Management

## Table of Contents
1. [Introduction](#1-introduction)
2. [Understanding CRUSH Maps](#2-understanding-crush-maps)
3. [Managing CRUSH Rules](#3-managing-crush-rules)
4. [Working with Pools](#4-working-with-pools)
5. [Handling Daemon Crashes](#5-handling-daemon-crashes)
6. [Troubleshooting & Verification](#6-troubleshooting--verification)

---

## 1. Introduction

In modern cloud infrastructure, especially when building a private cloud like Paulco Cloud’s R&D lab, storage reliability is key. Ceph uses a system called **CRUSH** to decide where data lives. Unlike traditional storage that uses a lookup table, Ceph calculates the location. This guide walks you through managing these maps, creating rules for specific hardware (like SSDs vs HDDs), setting up pools, and handling system crashes professionally.

We will focus on **hands-on commands** you can copy and paste directly into your terminal.

---

## 2. Understanding CRUSH Maps

The CRUSH map is the blueprint of your storage cluster. It tells Ceph which hard drives (OSDs) belong to which server (Host), which rack they are in, and how much data each drive should hold.

### Step 1: View the Current Cluster Hierarchy

Before changing anything, you must see the current layout. This command shows you the tree structure of your storage.

```bash
ceph osd tree
```
*This displays a simple tree showing hosts, racks, and OSDs with their status (up/down) and weight.*

For a more detailed view including device classes (ssd/hdd), use this format:

```bash
ceph osd tree --format json-pretty
```
*This outputs the hierarchy in JSON format, which is useful for scripts or detailed inspection.*

### Step 2: Export the CRUSH Map for Editing

If you need to make complex changes, you often download the map, edit it locally, and upload it back.

```bash
ceph osd getcrushmap -o compiled-crushmap.bin
```
*This saves the binary version of the map to a file named `compiled-crushmap.bin`.*

To read or edit it, you must decompile it into text format:

```bash
crushtool -d compiled-crushmap.bin -o decompiled-crushmap.txt
```
*Now `decompiled-crushmap.txt` is a human-readable text file you can edit with `vim` or `nano`.*

### Step 3: Inject a Modified CRUSH Map

After editing the text file, you must compile it back to binary and upload it.

First, compile the text file back to binary:

```bash
crushtool -c decompiled-crushmap.txt -o new-crushmap.bin
```

Then, test the new map safely before applying it:

```bash
ceph osd test-crushmap -i new-crushmap.bin
```
*This simulates data placement using the new map without actually changing the cluster. Check the output for errors.*

If the test passes, apply the new map:

```bash
ceph osd setcrushmap -i new-crushmap.bin
```
*⚠️ Warning: This triggers data rebalancing. Ensure your cluster is healthy before running this.*

---

## 3. Managing CRUSH Rules

Rules determine how many copies of data are kept and where they are placed. For example, you might want one rule for "High Performance SSDs" and another for "Standard HDDs."

### Step 1: List Existing Rules

See what rules are currently available in your cluster.

```bash
ceph osd crush rule ls
```

To see the details of a specific rule (e.g., how it places replicas):

```bash
ceph osd crush rule dump replicated_rule
```
*Replace `replicated_rule` with the name of the rule you want to inspect.*

### Step 2: Create a Rule for Specific Hardware (Device Classes)

In modern Ceph, we use **Device Classes** to separate SSDs from HDDs automatically.

First, ensure your OSDs have the correct class assigned. Ceph usually detects this automatically, but you can verify:

```bash
ceph osd crush show-device-class
```

If you need to manually set an OSD as an SSD:

```bash
ceph osd crush set-device-class ssd osd.1
```
*Replace `osd.1` with the actual OSD ID.*

Now, create a rule that **only** uses SSDs:

```bash
ceph osd crush rule create-replicated ssd-fast-rule default host ssd
```
*Breakdown:*
*   `ssd-fast-rule`: Name of your new rule.
*   `default`: The root bucket.
*   `host`: Failure domain (copies will be on different hosts).
*   `ssd`: Device class (only picks OSDs marked as ssd).

### Step 3: Create a Rule for Rack-Level Redundancy

If you have multiple racks and want to ensure data survives a whole rack failure:

```bash
ceph osd crush rule create-replicated rack-safe-rule default rack hdd
```
*This ensures replicas are placed in different racks, using only HDD devices.*

---

## 4. Working with Pools

Pools are the logical buckets where your data (VM disks, Object Store files) actually lives. Each pool uses one CRUSH Rule.

### Step 1: List All Pools

```bash
ceph osd pool ls
```

For detailed information about size, replication, and rules:

```bash
ceph osd pool ls detail
```

### Step 2: Create a New Pool with a Custom Rule

Let's create a high-performance pool for VM boot disks using the `ssd-fast-rule` we created earlier.

```bash
ceph osd pool create vm-boot-pool 64 64 replicated ssd-fast-rule
```
*Breakdown:*
*   `vm-boot-pool`: Name of the pool.
*   `64 64`: Initial PG count and PGP count. (Start small, increase later if needed).
*   `replicated`: Type of pool.
*   `ssd-fast-rule`: The CRUSH rule this pool will use.

### Step 3: Assign an Application Type

Modern Ceph requires you to tag pools so it knows what kind of data is inside (RBD, RGW, CephFS).

```bash
ceph osd pool application enable vm-boot-pool rbd
```
*Use `rgw` for object storage or `cephfs` for file systems.*

### Step 4: Adjust Replication Size

By default, Ceph might set 3 replicas. For a smaller lab or less critical data, you might want 2.

```bash
ceph osd pool set vm-boot-pool size 2
```

Set the minimum number of replicas required for I/O to continue (usually size - 1):

```bash
ceph osd pool set vm-boot-pool min_size 1
```
*⚠️ Warning: Setting `min_size` too low risks data loss if multiple OSDs fail simultaneously.*

### Step 5: Verify Pool Status

Check if the pool is active and clean:

```bash
rados df
```

Or check specific pool statistics:

```bash
ceph osd pool stats vm-boot-pool
```

---

## 5. Handling Daemon Crashes

Sometimes, an OSD or Monitor process crashes. Ceph has a built-in **Crash Module** to capture these events for debugging.

### Step 1: Enable the Crash Module

Ensure the manager module is enabled:

```bash
ceph mgr module enable crash
```

### Step 2: Configure Authentication for Crash Reporting

Each node needs permission to send crash logs to the manager.

Generate the keyring on the admin node:

```bash
ceph auth get-or-create client.crash mon 'profile crash' mgr 'profile crash' -o /etc/ceph/ceph.client.crash.keyring
```

Copy this keyring to all other nodes in the cluster at `/etc/ceph/ceph.client.crash.keyring`.

### Step 3: Start the Crash Service

On every node (OSD/MON/MGR servers), enable the service that watches for crashes:

```bash
systemctl enable --now ceph-crash.service
```

Verify it is running:

```bash
systemctl status ceph-crash.service
```

### Step 4: View Recent Crashes

If your cluster health shows `HEALTH_WARN` due to recent crashes, list them:

```bash
ceph crash ls-new
```

Get detailed information about a specific crash ID:

```bash
ceph crash info <crash-id>
```
*Replace `<crash-id>` with the ID from the previous list. This shows the stack trace and logs.*

### Step 5: Archive and Clear Warnings

Once you have analyzed the crash and fixed the issue, you must "archive" it to clear the health warning.

Archive a single crash:

```bash
ceph crash archive <crash-id>
```

Archive all new crashes at once:

```bash
ceph crash archive-all
```

### Step 6: Automatic Cleanup

Configure how long Ceph keeps crash logs. By default, it keeps them for a year. You can reduce this to 30 days:

```bash
ceph config set mgr mgr/crash/retain_interval 30d
```

Prune old crashes manually if needed:

```bash
ceph crash prune 30
```
*This deletes crash reports older than 30 days.*

---

## 6. Troubleshooting & Verification

### Scenario 1: Data is not balancing across SSDs and HDDs

**Check:** Are the device classes correctly assigned?

```bash
ceph osd crush show-device-class
```

**Fix:** If an SSD is listed as `hdd`, reclassify it:

```bash
ceph osd crush set-device-class ssd osd.X
```
*Then, ensure your pool is using the correct rule:*

```bash
ceph osd pool set my-pool crush_rule ssd-fast-rule
```

### Scenario 2: Pool is stuck in "creating" state

**Check:** Do you have enough OSDs to satisfy the rule?

```bash
ceph osd tree
```

**Fix:** If your rule requires 3 different hosts but you only have 2 hosts up, the pool cannot form. Bring up more OSDs or lower the replica size temporarily:

```bash
ceph osd pool set my-pool size 2
```

### Scenario 3: Health Warning "RECENT_CRASH" persists

**Check:** Did you archive the crashes?

```bash
ceph crash ls-new
```

**Fix:** If the list is empty but warning persists, restart the manager module:

```bash
ceph mgr module disable crash
ceph mgr module enable crash
```

---

## Summary Checklist for Production

1.  **Backup CRUSH Map**: Always run `ceph osd getcrushmap` before changes.
2.  **Test Changes**: Use `ceph osd test-crushmap` before injecting.
3.  **Label Devices**: Ensure SSDs/NVMe are tagged with `ceph osd crush set-device-class`.
4.  **Assign Rules**: Create specific rules for different performance tiers.
5.  **Tag Pools**: Always run `ceph osd pool application enable`.
6.  **Monitor Crashes**: Keep `ceph-crash.service` running on all nodes for stability analysis.

This guide provides the essential workflow for managing Ceph storage infrastructure in a professional DevOps environment.
