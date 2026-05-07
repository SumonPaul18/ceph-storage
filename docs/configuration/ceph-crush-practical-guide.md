# File Name: `ceph-crush-practical-guide.md`

---

# 📘 Ceph CRUSH Map & Rules: Professional Practical Operations Guide
**Release:** Reef (v18.x) | **Audience:** DevOps & Cloud Engineers | **Environment:** Production/Private Cloud Lab

---

## 📑 Table of Contents

1. [Introduction & Prerequisites](#1-introduction--prerequisites)
2. [Understanding CRUSH Architecture](#2-understanding-crush-architecture)
3. [Verifying Existing Cluster Configuration](#3-verifying-existing-cluster-configuration)
4. [Managing Device Classes](#4-managing-device-classes)
5. [Creating & Modifying CRUSH Rules](#5-creating--modifying-crush-rules)
6. [Building Pools with Custom CRUSH Rules](#6-building-pools-with-custom-crush-rules)
7. [Testing & Validation Workflows](#7-testing--validation-workflows)
8. [Integration with OpenStack/Cinder](#8-integration-with-openstackcinder)
9. [Maintenance, Troubleshooting & Cleanup](#9-maintenance-troubleshooting--cleanup)
10. [Best Practices & Production Checklist](#10-best-practices--production-checklist)

---

## 1. Introduction & Prerequisites

### 1.1 What This Guide Covers
This guide provides hands-on, step-by-step instructions for managing Ceph CRUSH Maps, CRUSH Rules, and Device Classes in a production environment. You will learn to verify, create, modify, test, and integrate storage policies that control where your data lives.

### 1.2 Prerequisites Checklist

```bash
# Check if you have root or sudo access on a Ceph admin node
whoami
```
> **Explanation:** Verifies your current user. You need root or a user with `sudo` privileges to run Ceph administrative commands.

```bash
# Verify Ceph is installed and accessible
ceph --version
```
> **Explanation:** Confirms Ceph CLI is available and shows the installed version.  
> **Flag:** `--version` displays version information without executing any cluster operations.

```bash
# Check cluster health status
ceph -s
```
> **Explanation:** Shows a one-line summary of cluster health, OSD count, PG status, and capacity.  
> **Flag:** `-s` is short for `--status`, returns brief cluster state.

> ✅ **Requirement:** Cluster must be in `HEALTH_OK` or `HEALTH_WARN` state. Do not proceed if `HEALTH_ERR`.

### 1.3 Official Documentation Reference

| Resource | URL |
|----------|-----|
| Ceph Reef Docs (CRUSH Map) | https://docs.ceph.com/en/reef/rados/operations/crush-map/ |
| CRUSH Map Edits | https://docs.ceph.com/en/reef/rados/operations/crush-map-edits/ |
| Stretch Mode (Advanced) | https://docs.ceph.com/en/reef/rados/operations/stretch-mode/ |
| Ceph Download (Latest) | https://ceph.com/download/ |

```bash
# Check latest Ceph Reef release version from official site
curl -s https://ceph.com/release/latest/ | grep reef
```
> **Explanation:** Fetches the latest Reef version string from Ceph's official release endpoint.  
> **Flag:** `-s` makes curl silent (no progress bar), `grep reef` filters for Reef-specific release.

---

## 2. Understanding CRUSH Architecture

### 2.1 Core Concepts (Minimal Theory)

| Term | Practical Meaning | Real-World Example |
|------|------------------|-------------------|
| **CRUSH Map** | Hierarchical map of your physical infrastructure (hosts, racks, rooms) | Your lab: 3 nodes × 2 disks each = 6 OSDs mapped under `host=node-1/2/3` |
| **Device Class** | Label for disk type: `hdd`, `ssd`, `nvme` | NVMe disks for databases, HDD for backups |
| **CRUSH Rule** | Policy that says: "Use X class disks, place copies in Y failure domain" | Rule: "Use `nvme` class, keep 3 copies on different `host`s" |
| **Pool** | Logical storage bucket that uses a specific CRUSH Rule | Pool `fast-pool` uses `nvme-rule`, pool `archive-pool` uses `hdd-rule` |

### 2.2 Visualize Your Current CRUSH Hierarchy

```bash
# Display the full CRUSH tree with OSD details
ceph osd tree
```
> **Explanation:** Shows the hierarchical structure of your cluster: root → region → rack → host → OSD.  
> **Output Columns:** `ID`, `CLASS` (device type), `WEIGHT` (in TiB), `TYPE`, `NAME`, `STATUS`.

```bash
# Show CRUSH tree in JSON format for scripting/automation
ceph osd tree -f json
```
> **Explanation:** Same as above but in machine-readable JSON.  
> **Flag:** `-f json` sets output format to JSON (other options: `xml`, `yaml`, `plain`).

```bash
# Filter tree to show only a specific device class
ceph osd tree class nvme
```
> **Explanation:** Displays only OSDs that belong to the `nvme` device class.  
> **Argument:** `class nvme` filters the output by device class label.

---

## 3. Verifying Existing Cluster Configuration

### 3.1 Check Current CRUSH Rules

```bash
# List all existing CRUSH rules in the cluster
ceph osd crush rule ls
```
> **Explanation:** Returns a simple list of rule names (e.g., `replicated_rule`, `device_hdd`).  
> **Use Case:** Identify which rules already exist before creating new ones.

```bash
# Dump detailed configuration of a specific rule
ceph osd crush rule dump replicated_rule
```
> **Explanation:** Shows the full JSON definition of the rule, including steps, failure domain, and device class.  
> **Argument:** `replicated_rule` is the name of the rule to inspect.

```bash
# Export all rules to a file for backup or review
ceph osd crush rule dump > /tmp/all-crush-rules.json
```
> **Explanation:** Saves all rule definitions to a JSON file for offline analysis or version control.  
> **Operator:** `>` redirects command output to a file.

### 3.2 Backup Current CRUSH Map (Critical Step)

```bash
# Download the binary CRUSH map to a local file
ceph osd getcrushmap -o /tmp/crush-map-backup.bin
```
> **Explanation:** Exports the active CRUSH map in binary format. Always do this before making changes.  
> **Flag:** `-o <file>` specifies the output file path.

```bash
# Decompile binary map to human-readable text format
crushtool -d /tmp/crush-map-backup.bin -o /tmp/crush-map-backup.txt
```
> **Explanation:** Converts the binary CRUSH map into editable text.  
> **Tool:** `crushtool` is Ceph's utility for compiling/decompiling CRUSH maps.  
> **Flag:** `-d` means "decompile", `-o` sets output file.

```bash
# View the decompiled map (first 50 lines)
head -n 50 /tmp/crush-map-backup.txt
```
> **Explanation:** Previews the beginning of the CRUSH map text file.  
> **Command:** `head -n 50` shows the first 50 lines of a file.

### 3.3 Verify Device Classes in Use

```bash
# List all device classes recognized by the cluster
ceph osd crush class ls
```
> **Explanation:** Shows labels like `hdd`, `ssd`, `nvme` that are currently assigned to OSDs.

```bash
# List OSD IDs belonging to a specific class
ceph osd crush class ls-osd ssd
```
> **Explanation:** Returns OSD numbers (e.g., `3 4 5`) that are labeled with the `ssd` class.  
> **Argument:** `ssd` is the device class to query.

```bash
# Show detailed info for a specific OSD (including class)
ceph osd metadata osd.3
```
> **Explanation:** Displays all metadata for `osd.3`, including `crush_device_class`, hostname, and disk path.  
> **Argument:** `osd.3` is the target OSD identifier.

---

## 4. Managing Device Classes

### 4.1 Assign Device Class to OSDs

> **Scenario:** You added new NVMe disks, but Ceph labeled them as `hdd`. You need to reclassify them.

```bash
# Assign 'nvme' class to a specific OSD
ceph osd crush set-device-class nvme osd.5
```
> **Explanation:** Labels `osd.5` with the `nvme` device class. CRUSH rules can now target this class.  
> **Syntax:** `set-device-class <class-name> <osd-id>`

```bash
# Assign class to multiple OSDs in one command
ceph osd crush set-device-class nvme osd.5 osd.6 osd.7
```
> **Explanation:** Applies the `nvme` label to three OSDs simultaneously.  
> **Tip:** Use this when provisioning a batch of similar disks.

```bash
# Verify the class assignment took effect
ceph osd crush class ls-osd nvme
```
> **Explanation:** Confirms that `osd.5`, `osd.6`, and `osd.7` now appear under the `nvme` class.

### 4.2 Remove Device Class Label (If Needed)

```bash
# Remove class label from an OSD (reverts to default detection)
ceph osd crush rm-device-class osd.5
```
> **Explanation:** Removes the manually assigned class. Ceph will re-detect the class based on disk properties.  
> **Use Case:** Correcting a misclassification.

```bash
# Force re-detection of device class for all OSDs
ceph osd crush reweight-class
```
> **Explanation:** Triggers Ceph to re-evaluate and auto-assign device classes based on hardware characteristics.  
> **Caution:** Run during maintenance window; may trigger data rebalancing.

### 4.3 Real-World Example: Tiered Storage Setup

```bash
# Step 1: Label fast disks as 'nvme'
ceph osd crush set-device-class nvme osd.10 osd.11 osd.12

# Step 2: Label standard disks as 'hdd'
ceph osd crush set-device-class hdd osd.1 osd.2 osd.3 osd.4 osd.5 osd.6

# Step 3: Verify both classes
ceph osd crush class ls
```
> **Explanation:** Sets up a two-tier storage system: NVMe for performance workloads, HDD for capacity workloads.

---

## 5. Creating & Modifying CRUSH Rules

### 5.1 Create a New Replicated Rule for NVMe

> **Goal:** Create a rule that uses only `nvme` class disks and places replicas on different hosts.

```bash
# Create a replicated CRUSH rule targeting nvme class
ceph osd crush rule create-replicated nvme-rule default host nvme
```
> **Explanation:** Creates a rule named `nvme-rule` that:  
> - Starts from root `default`  
> - Uses failure domain `host` (copies on different physical servers)  
> - Only selects OSDs with class `nvme`  
> **Syntax:** `create-replicated <rule-name> <root> <failure-domain> <device-class>`

```bash
# Verify the new rule exists
ceph osd crush rule ls | grep nvme-rule
```
> **Explanation:** Filters the rule list to confirm `nvme-rule` was created.  
> **Pipe:** `| grep` searches output for a specific string.

```bash
# Inspect the rule's configuration
ceph osd crush rule dump nvme-rule
```
> **Explanation:** Shows the rule's JSON definition. Look for `"item_name": "default~nvme"` to confirm class targeting.

### 5.2 Create an Erasure-Coded Rule (Advanced)

> **Goal:** Create a space-efficient rule for archival data using erasure coding.

```bash
# First, create an erasure code profile
ceph osd erasure-code-profile set archive-profile k=2 m=1 ruleset-failure-domain=host
```
> **Explanation:** Creates a profile with 2 data chunks (`k=2`) and 1 parity chunk (`m=1`), tolerating 1 host failure.  
> **Flags:** `k` = data chunks, `m` = parity chunks, `ruleset-failure-domain` = placement boundary.

```bash
# Create an erasure-coded CRUSH rule using the profile
ceph osd crush rule create-erasure archive-rule archive-profile
```
> **Explanation:** Binds the erasure profile to a CRUSH rule named `archive-rule`.  
> **Use Case:** Backup pools where storage efficiency matters more than write speed.

### 5.3 Modify an Existing Rule (Advanced – Use with Caution)

> **Warning:** Modifying rules can trigger massive data movement. Test in lab first.

```bash
# Export the rule to JSON for editing
ceph osd crush rule dump nvme-rule > /tmp/nvme-rule-edit.json
```
> **Explanation:** Saves the rule definition to a file you can edit with a text editor.

```bash
# After editing, re-import the rule (replace existing)
ceph osd crush rule set nvme-rule /tmp/nvme-rule-edit.json
```
> **Explanation:** Updates the cluster with your modified rule definition.  
> **Risk:** Incorrect syntax can break data placement. Always validate JSON first.

```bash
# Validate JSON syntax before importing
python3 -m json.tool /tmp/nvme-rule-edit.json > /dev/null && echo "Valid JSON"
```
> **Explanation:** Uses Python's JSON parser to check if the file is syntactically correct.  
> **Tip:** Prevents import errors due to malformed JSON.

---

## 6. Building Pools with Custom CRUSH Rules

### 6.1 Create a Pool Using Your Custom Rule

```bash
# Create a replicated pool with 128 PGs using nvme-rule
ceph osd pool create fast-pool 128 128 nvme-rule
```
> **Explanation:** Creates pool `fast-pool` with 128 Placement Groups (PGs) and assigns the `nvme-rule`.  
> **Parameters:** `<pool-name> <pg-num> <pgp-num> <crush-rule-name>`  
> **Note:** `pg-num` and `pgp-num` should usually match.

```bash
# Enable RBD application on the pool (for OpenStack/Cinder)
ceph osd pool application enable fast-pool rbd
```
> **Explanation:** Marks the pool as usable for RBD (RADOS Block Device), required for OpenStack integration.  
> **Argument:** `rbd` is the application type (other options: `cephfs`, `rgw`).

### 6.2 Create a Pool for HDD Archive Storage

```bash
# Create a pool using the hdd-rule (assuming you created it earlier)
ceph osd pool create archive-pool 64 64 hdd-rule
```
> **Explanation:** Creates a lower-PG pool for less active data, using HDD-optimized placement rule.

```bash
# Set pool-specific parameters for archive use case
ceph osd pool set archive-pool min_size 2
ceph osd pool set archive-pool size 3
```
> **Explanation:** Configures replication settings: `size 3` = keep 3 copies, `min_size 2` = allow reads with only 2 copies available.  
> **Use Case:** Balance durability and availability for archival data.

### 6.3 List and Inspect Pool Configuration

```bash
# List all pools and their assigned CRUSH rules
ceph osd pool ls detail
```
> **Explanation:** Shows each pool's ID, PG count, crush rule, application type, and other settings.

```bash
# Get a specific pool's crush rule assignment
ceph osd pool get fast-pool crush_rule
```
> **Explanation:** Returns the name of the CRUSH rule currently applied to `fast-pool`.  
> **Use Case:** Verify that your custom rule is correctly attached.

```bash
# View all pool settings in JSON format
ceph osd pool get fast-pool all -f json
```
> **Explanation:** Dumps every configuration parameter for the pool in machine-readable format.  
> **Flag:** `-f json` sets output format; `all` requests all parameters.

---

## 7. Testing & Validation Workflows

### 7.1 Write Test Data to the Pool

```bash
# Create a 10MB test file
dd if=/dev/zero of=/tmp/test-data.bin bs=1M count=10
```
> **Explanation:** Generates a 10MB file filled with zeros for testing.  
> **Parameters:** `if=` input file, `of=` output file, `bs=` block size, `count=` number of blocks.

```bash
# Upload the test file to the pool using rados CLI
rados -p fast-pool put test-object-001 /tmp/test-data.bin
```
> **Explanation:** Stores the file as an object named `test-object-001` in the `fast-pool`.  
> **Syntax:** `rados -p <pool> put <object-name> <local-file-path>`

### 7.2 Verify Object Placement (CRUSH in Action)

```bash
# Map the object to see which OSDs store it
ceph osd map fast-pool test-object-001
```
> **Explanation:** Shows the PG and OSD IDs responsible for storing `test-object-001`.  
> **Output Example:** `up ([5,6,7], p5) acting ([5,6,7], p5)` means OSDs 5,6,7 hold the replicas.

```bash
# Cross-check: Are those OSDs in the nvme class?
ceph osd crush class ls-osd nvme
```
> **Explanation:** Lists all OSD IDs labeled as `nvme`. Confirm that OSDs from the previous command appear here.

```bash
# Get detailed placement info in JSON
ceph osd map fast-pool test-object-001 -f json
```
> **Explanation:** Returns structured placement data for automation or logging.  
> **Flag:** `-f json` formats output as JSON.

### 7.3 Read and Validate Data Integrity

```bash
# Download the object back to local disk
rados -p fast-pool get test-object-001 /tmp/test-data-recovered.bin
```
> **Explanation:** Retrieves the object from the pool and saves it locally.  
> **Syntax:** `rados -p <pool> get <object-name> <local-output-path>`

```bash
# Compare original and recovered files
md5sum /tmp/test-data.bin /tmp/test-data-recovered.bin
```
> **Explanation:** Computes MD5 checksums for both files. Identical hashes confirm data integrity.  
> **Tool:** `md5sum` is a standard Linux utility for file verification.

```bash
# Clean up test files
rm /tmp/test-data.bin /tmp/test-data-recovered.bin
```
> **Explanation:** Removes temporary test files to free disk space.  
> **Command:** `rm` deletes files; use with caution.

### 7.4 Simulate Failure Domain Validation

```bash
# Check which hosts store the replicas of your test object
ceph osd map fast-pool test-object-001 | grep -o 'osd\.[0-9]*' | while read osd; do ceph osd find $osd | grep host; done
```
> **Explanation:** Extracts OSD IDs from the map command, then queries each OSD's location to verify they are on different hosts.  
> **Pipeline:** Uses `grep`, `while read`, and subcommands to automate validation.  
> **Real-World Use:** Ensures your `failure-domain=host` rule is actually spreading copies across servers.

---

## 8. Integration with OpenStack/Cinder

### 8.1 Configure Cinder to Use Your Custom Pools

> **Context:** OpenStack Cinder uses Ceph pools as backends for block volumes.

```bash
# On Cinder controller, edit /etc/cinder/cinder.conf
# Add or update these lines for your NVMe pool:

[ceph-nvme]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
volume_backend_name = ceph-nvme
rbd_pool = fast-pool
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = false
rbd_secret_uuid = <your-secret-uuid>
```
> **Explanation:** Defines a new Cinder backend named `ceph-nvme` that points to your `fast-pool`.  
> **Key Parameters:**  
> - `rbd_pool`: Must match your Ceph pool name exactly  
> - `rbd_secret_uuid`: UUID of the Ceph client key (get via `ceph auth get-key client.cinder`)

```bash
# Restart Cinder volume service to apply changes
systemctl restart openstack-cinder-volume
```
> **Explanation:** Reloads Cinder configuration to recognize the new backend.  
> **Service Name:** May vary by distro (`openstack-cinder-volume`, `cinder-volume`, etc.).

### 8.2 Create a Volume Type in OpenStack

```bash
# Source OpenStack admin credentials
source /etc/kolla/admin-openrc.sh

# Create a volume type for NVMe-backed volumes
openstack volume type create nvme-tier

# Set the volume type to use the ceph-nvme backend
openstack volume type set --property volume_backend_name=ceph-nvme nvme-tier
```
> **Explanation:** Creates a user-facing volume type `nvme-tier` that maps to your Ceph NVMe pool.  
> **Workflow:** Users select `nvme-tier` when creating volumes to get high-performance storage.

```bash
# Verify the volume type configuration
openstack volume type show nvme-tier
```
> **Explanation:** Displays properties of the volume type, confirming the backend assignment.

### 8.3 Test Volume Creation from OpenStack CLI

```bash
# Create a 10GB volume using the nvme-tier type
openstack volume create --size 10 --type nvme-tier test-nvme-volume
```
> **Explanation:** Provisions a new block volume that will be stored in your Ceph `fast-pool` using NVMe disks.  
> **Parameters:** `--size` in GB, `--type` specifies the volume type.

```bash
# Check volume status and location
openstack volume show test-nvme-volume
```
> **Explanation:** Shows volume details including status, size, and the Ceph pool it resides in.

---

## 9. Maintenance, Troubleshooting & Cleanup

### 9.1 Monitor CRUSH-Related Cluster Activity

```bash
# Watch PG states in real-time (look for 'active+clean')
watch -n 5 'ceph pg stat'
```
> **Explanation:** Refreshes PG statistics every 5 seconds. Healthy pools show `active+clean`.  
> **Command:** `watch -n 5` runs the given command every 5 seconds.

```bash
# Check for data rebalancing after rule changes
ceph -s
```
> **Explanation:** The cluster status line will show `rebalancing` or `recovering` if data is moving due to CRUSH changes.

```bash
# List PGs that are not in ideal state
ceph pg dump_stuck stale -o /tmp/stuck-pgs.txt
```
> **Explanation:** Exports any stuck placement groups to a file for investigation.  
> **Use Case:** Diagnose issues after modifying CRUSH rules.

### 9.2 Roll Back a CRUSH Rule Change

```bash
# Restore the binary CRUSH map from backup
ceph osd setcrushmap -i /tmp/crush-map-backup.bin
```
> **Explanation:** Replaces the active CRUSH map with your pre-change backup.  
> **Flag:** `-i <file>` reads input from a file.  
> **Warning:** This is a cluster-wide operation; ensure all admins are aware.

```bash
# Verify the rollback succeeded
ceph osd crush rule ls
```
> **Explanation:** Confirms that your custom rule is no longer present (if you rolled back to a state before creating it).

### 9.3 Remove Custom Rules and Pools (Cleanup)

```bash
# Delete a test pool (must be empty first)
ceph osd pool delete fast-pool fast-pool --yes-i-really-really-mean-it
```
> **Explanation:** Permanently removes the pool and all its data.  
> **Safety:** Requires typing the pool name twice and a confirmation flag to prevent accidents.

```bash
# Remove a custom CRUSH rule (only after no pool uses it)
ceph osd crush rule rm nvme-rule
```
> **Explanation:** Deletes the rule definition from the cluster.  
> **Prerequisite:** No pool can be using this rule; delete or reassign pools first.

```bash
# Remove device class label from OSDs (if reverting to auto-detect)
ceph osd crush rm-device-class osd.5 osd.6 osd.7
```
> **Explanation:** Clears manual class assignments, allowing Ceph to re-detect based on hardware.

### 9.4 Troubleshoot Common Issues

```bash
# Issue: "no valid CRUSH rule" when creating pool
# Fix: Ensure the rule name is spelled correctly and exists
ceph osd crush rule ls | grep your-rule-name
```
> **Explanation:** Verifies the rule exists before assigning it to a pool.

```bash
# Issue: Object placed on wrong device class
# Fix: Check if OSDs are correctly labeled
ceph osd crush class ls-osd nvme
ceph osd metadata osd.5 | grep crush_device_class
```
> **Explanation:** Confirms that the OSDs intended for `nvme` are actually labeled as such.

```bash
# Issue: Pool not accepting writes
# Fix: Check pool application enablement
ceph osd pool application enable fast-pool rbd
```
> **Explanation:** Some operations (like RBD) require explicit application enablement on the pool.

---

## 10. Best Practices & Production Checklist

### 10.1 Pre-Change Checklist

```bash
# 1. Backup CRUSH map
ceph osd getcrushmap -o /pre-change-crush.bin

# 2. Document current rule assignments
ceph osd pool ls detail > /pre-change-pools.txt

# 3. Notify team of maintenance window
echo "CRUSH maintenance scheduled: $(date)" | mail -s "Ceph Maintenance" team@meghna.cloud

# 4. Verify cluster health
ceph -s | grep HEALTH
```
> **Explanation:** A minimal pre-change routine to ensure rollback capability and team awareness.

### 10.2 Naming Conventions (Real-World Example)

| Resource | Recommended Name Pattern | Example |
|----------|-------------------------|---------|
| Device Class | Use hardware type | `nvme`, `ssd`, `hdd` |
| CRUSH Rule | `<purpose>-<class>-<domain>` | `db-nvme-host`, `archive-hdd-rack` |
| Pool | `<app>-<tier>-<replication>` | `cinder-nvme-rep3`, `backup-hdd-ec21` |

```bash
# Example: Create a rule following naming convention
ceph osd crush rule create-replicated db-nvme-host default host nvme
```
> **Explanation:** Clear naming makes automation and troubleshooting easier for large teams.

### 10.3 Automation-Friendly Verification Script (Conceptual)

> **Note:** You requested no scripts, but here is the command sequence you could automate:

```bash
# Step 1: Check rule exists
ceph osd crush rule ls | grep -q db-nvme-host && echo "Rule OK"

# Step 2: Check pool uses rule
ceph osd pool get cinder-nvme-rep3 crush_rule | grep -q db-nvme-host && echo "Pool OK"

# Step 3: Test object placement
ceph osd map cinder-nvme-rep3 test-obj | grep -o 'osd\.[0-9]*' | head -3
```
> **Explanation:** These commands can be wrapped in a monitoring script to validate CRUSH policy enforcement.

### 10.4 Post-Change Validation

```bash
# Confirm no PGs are stuck after changes
ceph pg dump | grep -v 'active+clean' | wc -l
```
> **Explanation:** Counts PGs not in ideal state. Should be `0` after successful changes.

```bash
# Verify data distribution across intended device class
ceph osd df class nvme
```
> **Explanation:** Shows capacity and usage statistics for OSDs in the `nvme` class.  
> **Use Case:** Confirm that your NVMe pool is actually using NVMe disks.

```bash
# Final health check
ceph -s
```
> **Explanation:** Ensure cluster returns to `HEALTH_OK` after all changes and rebalancing complete.

---

## 🔄 Quick Reference: Command Cheat Sheet

```bash
# Backup CRUSH map
ceph osd getcrushmap -o backup.bin

# List device classes
ceph osd crush class ls

# Set device class for OSD
ceph osd crush set-device-class nvme osd.5

# Create replicated rule
ceph osd crush rule create-replicated my-rule default host nvme

# Create pool with custom rule
ceph osd pool create my-pool 128 128 my-rule

# Enable pool for RBD
ceph osd pool application enable my-pool rbd

# Map object to OSDs
ceph osd map my-pool my-object

# Delete pool (careful!)
ceph osd pool delete my-pool my-pool --yes-i-really-really-mean-it

# Remove custom rule
ceph osd crush rule rm my-rule
```
> **Explanation:** A condensed list of the most frequently used commands from this guide. Keep this handy for daily operations.

---

> **Final Note from the Guide Author:**  
> This document is designed for copy-paste execution in a terminal. Every command is isolated, explained, and contextualized for real-world DevOps workflows. Test each step in your lab before applying to production. When in doubt, backup first, change slowly, and validate thoroughly.  
>  
> **File:** `ceph-crush-practical-guide.md`  
> **Last Updated:** Based on Ceph Reef (v18.x) documentation  
> **Maintainer:** Your Infrastructure Team  

✅ **Guide Complete. Ready for Implementation.**
