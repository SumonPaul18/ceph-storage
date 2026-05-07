# ceph-crush-crash-practical-guide.md

# Ceph CRUSH Map, Crash Module & Pool Management: Professional Practical Guide

> **Purpose:** A hands-on, step-by-step practical guide for DevOps engineers managing Ceph clusters with focus on Crash analysis, CRUSH Map operations, Rule creation, and Pool management in production environments.

> **Target Audience:** System Administrators, DevOps Engineers, Cloud Infrastructure Teams

> **Environment:** Ceph Pacific/Quincy/Reef, cephadm or traditional deployment, Linux (RHEL/Ubuntu/CentOS)

> **Language:** Simple English with practical command examples

---

## Table of Contents

1. [Introduction & Prerequisites](#1-introduction--prerequisites)
2. [Ceph Crash Module: Detection, Analysis & Management](#2-ceph-crash-module-detection-analysis--management)
3. [CRUSH Map: Architecture, Viewing & Editing](#3-crush-map-architecture-viewing--editing)
4. [Device Classes: SSD/HDD/NVME Separation](#4-device-classes-ssdhddnvme-separation)
5. [Creating & Managing CRUSH Rules](#5-creating--managing-crush-rules)
6. [Pool Creation: Replicated & Erasure Coded](#6-pool-creation-replicated--erasure-coded)
7. [Advanced CRUSH Map Operations](#7-advanced-crush-map-operations)
8. [Verification, Testing & Troubleshooting](#8-verification-testing--troubleshooting)
9. [Maintenance, Cleanup & Best Practices](#9-maintenance-cleanup--best-practices)

---

## 1. Introduction & Prerequisites

### 1.1 Requirements Checklist

Before starting any Ceph operations, ensure your environment meets these requirements:

- Ceph cluster is healthy: `ceph health` returns `HEALTH_OK` or acceptable `HEALTH_WARN`
- Administrative access: `ceph` command available with admin keyring
- Network connectivity between all monitor and OSD nodes
- Sufficient disk space for operations and logs
- Backup of critical configuration before major changes

### 1.2 Official Resources & Version Check

```bash
# Check your current Ceph version
ceph --version
```
> This command displays your installed Ceph version. Note this version to match with official documentation.

```bash
# Check Ceph dashboard version if using web UI
ceph mgr services
```
> Shows active manager services including dashboard URL and version information.

```bash
# Verify latest stable Ceph version from official source
curl -s https://download.ceph.com/releases/ | grep -oP 'ceph-\K[0-9.]+(?=\.tar)' | sort -V | tail -1
```
> Fetches the latest stable Ceph release version from official download server for upgrade planning.

**Official Documentation Links:**
- Main Docs: https://docs.ceph.com/en/latest/
- Crash Module: https://docs.ceph.com/en/latest/mgr/crash/
- CRUSH Map: https://docs.ceph.com/en/latest/rados/operations/crush-map/
- Pool Operations: https://docs.ceph.com/en/latest/rados/operations/pools/

### 1.3 Pre-Operation Verification

```bash
# Verify cluster health status
ceph health detail
```
> Displays detailed cluster health. Ensure no critical errors before proceeding with configuration changes.

```bash
# List all OSDs and their status
ceph osd tree
```
> Shows hierarchical view of OSDs under hosts. Verify all expected OSDs are 'up' and 'in'.

```bash
# Check manager modules status
ceph mgr module ls
```
> Lists all available and enabled manager modules. Confirm 'crash' module is enabled.

```bash
# Verify admin keyring permissions
ceph auth get client.admin
```
> Displays admin client capabilities. Ensure 'mon * allow *' and 'osd * allow *' permissions exist.

---

## 2. Ceph Crash Module: Detection, Analysis & Management

### 2.1 Enable and Configure Crash Module

```bash
# Enable the crash manager module
ceph mgr module enable crash
```
> Activates the crash reporting module in Ceph manager. Required for collecting daemon crash dumps.

```bash
# Verify crash module is enabled
ceph mgr module status crash
```
> Confirms the crash module is active and running. Should show 'enabled' status.

```bash
# Set crash retention period (example: 30 days)
ceph config set mgr mgr/crash/retain_interval 30d
```
> Configures automatic cleanup of crash reports older than specified duration. Prevents disk space exhaustion.

```bash
# View current crash module configuration
ceph config dump | grep crash
```
> Displays all crash-related configuration parameters and their current values.

### 2.2 List and Inspect Crash Reports

```bash
# List all new (unarchived) crash reports
ceph crash ls-new
```
> Shows crash reports that have not been archived yet. Marked with asterisk (*) in output.

```bash
# List all crash reports including archived
ceph crash ls
```
> Displays complete history of crash reports. Includes both new and previously archived entries.

```bash
# Get detailed information for specific crash ID
ceph crash info 2024-05-01_10:00:00.123456Z_osd.12
```
> Replaces `2024-05-01_10:00:00.123456Z_osd.12` with actual crash ID. Returns JSON with backtrace, assert message, and environment details.

```bash
# Export crash details to JSON file for analysis
ceph crash info <CRASH_ID> -f json > /tmp/crash_analysis.json
```
> Replaces `<CRASH_ID>` with actual crash identifier. Saves detailed crash data to file for offline analysis or sharing with support.

### 2.3 Analyze Crash Data Practically

```bash
# Install jq for JSON parsing (if not present)
sudo apt update && sudo apt install jq -y
```
> For Ubuntu/Debian systems. Installs jq tool for structured JSON data extraction.

```bash
# Install jq for JSON parsing (RHEL/CentOS alternative)
sudo yum install epel-release -y && sudo yum install jq -y
```
> For RHEL/CentOS systems. Enables EPEL repository then installs jq utility.

```bash
# Extract assert message from crash JSON
cat /tmp/crash_analysis.json | jq '.assert_message'
```
> Displays the assertion condition that triggered the crash. Critical for root cause identification.

```bash
# Extract backtrace from crash JSON
cat /tmp/crash_analysis.json | jq '.backtrace'
```
> Shows code execution stack at crash time. Essential for developers to debug the issue.

```bash
# Extract Ceph version from crash metadata
cat /tmp/crash_analysis.json | jq '.ceph_version'
```
> Identifies which Ceph version experienced the crash. Helps determine if issue is version-specific.

```bash
# Extract operating system details from crash
cat /tmp/crash_analysis.json | jq '.os_version'
```
> Shows OS version and kernel where crash occurred. Useful for identifying platform-specific issues.

### 2.4 Archive and Manage Crash Reports

```bash
# Archive a single crash report
ceph crash archive <CRASH_ID>
```
> Replaces `<CRASH_ID>` with actual identifier. Moves crash from 'new' to 'archived' state, removing health warning.

```bash
# Archive all new crash reports at once
ceph crash archive-all
```
> Archives all unarchived crashes in single operation. Use after reviewing all new crash reports.

```bash
# List archived crashes only
ceph crash ls | grep -v "NEW"
```
> Filters output to show only previously archived crash reports. Helps review historical issues.

```bash
# Remove a specific crash report permanently
ceph crash rm <CRASH_ID>
```
> Deletes crash report from storage completely. Use cautiously after analysis and archival.

```bash
# Prune crashes older than specified days
ceph crash prune 30
```
> Automatically removes archived crash reports older than 30 days. Helps manage storage consumption.

```bash
# Disable crash warning in health status (not recommended for production)
ceph config set mgr mgr/crash/warn_on_unarchived_crashes false
```
> Suppresses HEALTH_WARN for unarchived crashes. Use only for testing; not recommended for production monitoring.

### 2.5 Manual Crash Log Inspection

```bash
# Navigate to crash spool directory
ls -ltr /var/lib/ceph/crash/
```
> Lists crash report directories sorted by time. Shows location of raw crash data on each node.

```bash
# View crash metadata file
cat /var/lib/ceph/crash/<CRASH_DIR>/meta
```
> Replaces `<CRASH_DIR>` with actual crash directory name. Displays structured metadata about the crash event.

```bash
# View crash log content
cat /var/lib/ceph/crash/<CRASH_DIR>/log
```
> Shows raw log output from crashed daemon. Contains stderr output and additional context.

```bash
# Check system logs for related errors
journalctl -u ceph-osd@<OSD_ID> -n 100 --no-pager
```
> Replaces `<OSD_ID>` with actual OSD number. Shows last 100 log entries for specific OSD service.

---

## 3. CRUSH Map: Architecture, Viewing & Editing

### 3.1 Understand CRUSH Map Structure

```bash
# View OSD tree hierarchy
ceph osd tree
```
> Displays hierarchical structure: root -> datacenter -> rack -> host -> OSD. Critical for understanding data placement.

```bash
# View OSD tree with additional statistics
ceph osd tree --format json-pretty
```
> Shows same hierarchy in formatted JSON with capacity, usage, and utilization percentages.

```bash
# Export binary CRUSH map file
ceph osd getcrushmap -o /tmp/crush.map.bin
```
> Saves current CRUSH map to binary file. Required before any manual editing operations.

```bash
# Decompile binary map to readable text
crushtool -d /tmp/crush.map.bin -o /tmp/crush.txt
```
> Converts binary CRUSH map to human-readable text format. Enables manual editing with text editor.

### 3.2 Analyze CRUSH Map Content

```bash
# View decomplied CRUSH map content
head -100 /tmp/crush.txt
```
> Displays first 100 lines of text CRUSH map. Shows tunables, types, and bucket definitions.

```bash
# Search for specific host in CRUSH map
grep -A 10 "host ceph1" /tmp/crush.txt
```
> Shows host definition and its child OSDs. Replace `ceph1` with actual hostname from your environment.

```bash
# Count total OSDs in CRUSH map
grep -c "item osd" /tmp/crush.txt
```
> Returns total number of OSD entries in CRUSH map. Useful for validation after edits.

```bash
# Identify device classes in CRUSH map
grep "class" /tmp/crush.txt | sort | uniq
```
> Lists all device class references (ssd, hdd, nvme). Helps plan class-based rules.

### 3.3 Edit and Recompile CRUSH Map

```bash
# Edit CRUSH map with preferred editor
nano /tmp/crush.txt
```
> Opens text CRUSH map for editing. Modify weights, add buckets, or adjust rules as needed.

```bash
# Alternative: Use vim for editing
vim /tmp/crush.txt
```
> Vim editor alternative for experienced users. Same editing capabilities as nano.

```bash
# Recompile edited text map to binary
crushtool -c /tmp/crush.txt -o /tmp/new_crush.map.bin
```
> Converts edited text file back to binary format. Validates syntax before application.

```bash
# Test new CRUSH map without applying
ceph osd setcrushmap -i /tmp/new_crush.map.bin --dry-run
```
> Validates new map against cluster state without making changes. Safe way to test modifications.

```bash
# Apply new CRUSH map to cluster
ceph osd setcrushmap -i /tmp/new_crush.map.bin
```
> Replaces active CRUSH map with new version. Triggers data rebalancing if placement changes.

```bash
# Monitor rebalancing progress after map change
ceph -w
```
> Opens real-time cluster monitor. Shows PG migration and rebalancing progress after CRUSH changes.

### 3.4 CRUSH Map Rollback Procedure

```bash
# Keep backup of original map before changes
cp /tmp/crush.map.bin /tmp/crush.map.backup.bin
```
> Creates backup copy before modifications. Essential for quick rollback if issues occur.

```bash
# Rollback to previous CRUSH map if needed
ceph osd setcrushmap -i /tmp/crush.map.backup.bin
```
> Restores previous working CRUSH map. Use if new map causes unexpected behavior or performance issues.

```bash
# Verify rollback success
ceph osd tree
```
> Confirms cluster hierarchy matches expected state after rollback operation.

---

## 4. Device Classes: SSD/HDD/NVME Separation

### 4.1 Identify and Label Device Classes

```bash
# View OSDs with device class information
ceph osd df tree
```
> Displays OSD capacity, usage, and device class column. Identifies which OSDs are ssd, hdd, or nvme.

```bash
# Check automatic device class detection
ceph osd crush get-device-class osd.0
```
> Replaces `osd.0` with actual OSD ID. Returns detected device class for specific OSD.

```bash
# Manually set device class for OSD
ceph osd crush set-device-class ssd osd.0
```
> Assigns 'ssd' class to specified OSD. Use when automatic detection fails or for custom classification.

```bash
# Remove device class assignment
ceph osd crush rm-device-class osd.0
```
> Clears device class label from OSD. Allows reassignment or reversion to default classification.

```bash
# List all OSDs by device class
ceph osd crush class ls
```
> Shows available device classes and associated OSD counts. Useful for planning class-based rules.

```bash
# View OSDs belonging to specific class
ceph osd crush class ls-osd ssd
```
> Lists all OSD IDs classified as 'ssd'. Replace 'ssd' with 'hdd' or 'nvme' as needed.

### 4.2 Create Class-Specific CRUSH Rules

```bash
# Create replicated rule for SSD devices only
ceph osd crush rule create-replicated fast-ssd-rule default host ssd
```
> Creates rule named 'fast-ssd-rule' using 'host' failure domain and 'ssd' device class. Ensures data stays on SSDs.

```bash
# Create replicated rule for HDD devices only
ceph osd crush rule create-replicated capacity-hdd-rule default host hdd
```
> Creates rule for high-capacity, lower-cost storage using HDD-class devices only.

```bash
# Create rule for NVMe devices with OSD-level failure domain
ceph osd crush rule create-replicated ultra-fast-nvme default osd nvme
```
> Uses 'osd' failure domain for maximum performance (less redundancy). Use only for non-critical high-speed workloads.

```bash
# Verify newly created rule
ceph osd crush rule dump fast-ssd-rule
```
> Displays rule configuration in JSON format. Confirm 'type' is 'host' and 'class' is 'ssd' as intended.

```bash
# List all available CRUSH rules
ceph osd crush rule ls
```
> Shows all defined rules including default and custom. Helps avoid naming conflicts.

### 4.3 Apply Device Class Rules to Pools

```bash
# Create new pool with SSD rule
ceph osd pool create ssd-pool 64 64 replicated
```
> Creates replicated pool with 64 PGs. Initial creation uses default rule; rule assignment follows.

```bash
# Assign custom CRUSH rule to pool
ceph osd pool set ssd-pool crush_rule fast-ssd-rule
```
> Links pool to SSD-specific rule. Future data placements will follow SSD-only placement logic.

```bash
# Enable pool for RBD application
ceph osd pool application enable ssd-pool rbd
```
> Marks pool for RBD (block storage) use. Required to avoid warnings when creating RBD images.

```bash
# Verify pool configuration
ceph osd pool get ssd-pool crush_rule
```
> Confirms pool is using intended CRUSH rule. Should return 'fast-ssd-rule' if assignment succeeded.

```bash
# Create RBD image on SSD pool
rbd create ssd-image --size 10G --pool ssd-pool
```
> Creates 10GB RBD image in SSD pool. Data will be placed only on OSDs with 'ssd' device class.

---

## 5. Creating & Managing CRUSH Rules

### 5.1 Rule Types and Use Cases

```bash
# View default replicated rule configuration
ceph osd crush rule dump replicated_rule
```
> Shows default rule parameters. Reference for creating custom rules with similar structure.

```bash
# Create custom replicated rule with rack failure domain
ceph osd crush rule create-replicated rack-aware-rule default rack
```
> Creates rule ensuring replicas are placed in different racks. Requires rack-level bucket hierarchy in CRUSH map.

```bash
# Create rule with custom step count
ceph osd crush rule create-replicated custom-steps default host '' 5
```
> Creates rule with 5 placement steps instead of default 3. Advanced use case for complex topologies.

```bash
# Create erasure-coded rule for EC pool
ceph osd crush rule create-erasure ec-default default host
```
> Creates rule compatible with erasure-coded pools. Must match erasure code profile requirements.

### 5.2 Rule Validation and Testing

```bash
# Test rule placement for specific pool
ceph osd crush rule test fast-ssd-rule ssd-pool 100
```
> Simulates placement of 100 objects using rule. Shows distribution across OSDs without writing data.

```bash
# View rule statistics after testing
ceph osd crush rule status fast-ssd-rule
```
> Displays rule usage statistics including associated pools and placement success rate.

```bash
# Check rule compatibility with pool type
ceph osd pool get ssd-pool type
```
> Returns 'replicated' or 'erasure'. Ensure rule type matches pool type to avoid placement errors.

### 5.3 Modify and Remove CRUSH Rules

```bash
# Remove unused custom rule
ceph osd crush rule rm unused-rule-name
```
> Deletes rule that is not associated with any pool. Cannot remove rules actively used by pools.

```bash
# Check which pools use a specific rule
ceph osd pool ls detail | grep crush_rule
```
> Lists all pools with their assigned CRUSH rules. Helps identify rule dependencies before removal.

```bash
# Update pool to use different rule
ceph osd pool set existing-pool crush_rule new-rule-name
```
> Changes rule for existing pool. Triggers data migration to new placement locations.

```bash
# Monitor data migration after rule change
ceph pg dump_stuck stale
```
> Shows PGs that are stuck during migration. Helps identify issues after rule reassignment.

---

## 6. Pool Creation: Replicated & Erasure Coded

### 6.1 Replicated Pool Operations

```bash
# Create basic replicated pool
ceph osd pool create basic-pool 32 32 replicated
```
> Creates pool with 32 PGs (initial and pg_num). Uses default replicated rule with size 3.

```bash
# Set pool replication size
ceph osd pool set basic-pool size 3
```
> Configures number of data copies. Value 3 means 3 copies stored across different failure domains.

```bash
# Set minimum replicas for write operations
ceph osd pool set basic-pool min_size 2
```
> Allows writes with minimum 2 replicas available. Prevents write blocking during minor failures.

```bash
# Enable PG autoscale for pool
ceph osd pool set basic-pool pg_autoscale_mode on
```
> Enables automatic PG count adjustment. Recommended for most pools to optimize performance.

```bash
# Set application type for pool
ceph osd pool application enable basic-pool rbd
```
> Marks pool for RBD block storage. Use 'cephfs' for file system or 'rgw' for object storage.

### 6.2 Erasure Coded Pool Operations

```bash
# Create erasure code profile with k=4, m=2
ceph osd erasure-code-profile set ec-4-2 k=4 m=2
```
> Defines EC profile: 4 data chunks + 2 parity chunks. Requires minimum 6 OSDs per stripe.

```bash
# Create EC profile with plugin specification
ceph osd erasure-code-profile set ec-reedsol k=8 m=3 plugin=jerasure technique=reed_sol_van
```
> Advanced EC profile using Reed-Solomon encoding. Better fault tolerance with higher CPU overhead.

```bash
# Create erasure-coded pool with profile
ceph osd pool create ec-pool 64 64 erasure ec-4-2
```
> Creates EC pool using previously defined profile. Cannot use replicated rules with EC pools.

```bash
# Set application type for EC pool
ceph osd pool application enable ec-pool rgw
```
> EC pools work best with RGW object storage due to sequential write patterns.

```bash
# Verify EC pool configuration
ceph osd pool get ec-pool all
```
> Displays all pool parameters including erasure code profile and crush rule assignment.

### 6.3 Pool Management Operations

```bash
# List all pools with details
ceph osd pool ls detail
```
> Shows comprehensive pool information: PG count, size, min_size, crush rule, and application type.

```bash
# Get specific pool parameter
ceph osd pool get ssd-pool size
```
> Returns replication size for specified pool. Replace 'size' with any valid pool parameter.

```bash
# Increase pool PG count (requires careful planning)
ceph osd pool set basic-pool pg_num 64
```
> Increases placement groups for better distribution. Should be power of 2 and planned for capacity growth.

```bash
# Delete pool (use with extreme caution)
ceph osd pool delete basic-pool basic-pool --yes-i-really-really-mean-it
```
> Permanently removes pool and all data. Requires confirmation string to prevent accidental deletion.

```bash
# Rename pool
ceph osd pool rename old-pool-name new-pool-name
```
> Changes pool identifier. Updates all references but does not move data physically.

---

## 7. Advanced CRUSH Map Operations

### 7.1 Custom Bucket Creation

```bash
# Create custom rack bucket in CRUSH map
ceph osd crush add-bucket rack-01 rack
```
> Adds new rack-level bucket. Use for organizing hosts by physical location.

```bash
# Move host into rack bucket
ceph osd crush move ceph1 rack=rack-01
```
> Reassigns host 'ceph1' under 'rack-01' bucket. Updates hierarchy for rack-aware placement.

```bash
# Create datacenter bucket for multi-site
ceph osd crush add-bucket dc-east datacenter
```
> Adds datacenter-level bucket for geographic distribution planning.

```bash
# Move rack into datacenter bucket
ceph osd crush move rack-01 datacenter=dc-east
```
> Organizes rack under datacenter for multi-site CRUSH rules.

### 7.2 Weight Adjustment and Rebalancing

```bash
# Adjust OSD weight for capacity differences
ceph osd crush reweight osd.0 1.5
```
> Sets OSD weight to 1.5TB equivalent. Higher weight attracts more PGs during placement.

```bash
# Rebalance cluster after weight changes
ceph osd reweight-by-utilization 120
```
> Automatically adjusts OSD weights to balance utilization within 20% threshold.

```bash
# Monitor rebalancing progress
ceph pg dump | grep -E 'pg_id|up|acting'
```
> Shows PG placement status. 'up' and 'acting' sets should converge after rebalancing.

```bash
# Pause rebalancing during maintenance
ceph osd set norebalance
```
> Temporarily stops automatic PG migration. Use during planned maintenance windows.

```bash
# Resume rebalancing after maintenance
ceph osd unset norebalance
```
> Re-enables automatic rebalancing. Cluster will resume PG migrations to optimal placement.

### 7.3 Failure Domain Testing

```bash
# Simulate host failure for testing
ceph osd out osd.0 osd.1 osd.2
```
> Marks multiple OSDs as 'out' simultaneously. Tests data availability during host failure.

```bash
# Check cluster health during simulation
ceph health detail
```
> Verifies cluster maintains acceptable health with simulated failures. Should show degraded but functional state.

```bash
# Verify data accessibility during failure
rbd info --pool ssd-pool ssd-image
```
> Confirms RBD image remains accessible during OSD failures. Validates replication effectiveness.

```bash
# Restore OSDs to normal state
ceph osd in osd.0 osd.1 osd.2
```
> Returns OSDs to active service. Triggers data rebalancing to restore optimal placement.

---

## 8. Verification, Testing & Troubleshooting

### 8.1 Placement Verification

```bash
# Map object to OSDs using CRUSH
ceph osd map ssd-pool test-object
```
> Shows which OSDs would store 'test-object' in 'ssd-pool'. Validates rule placement logic.

```bash
# Check PG placement for specific PG
ceph pg map 1.0
```
> Displays up and acting OSDs for PG 1.0. Verifies actual data placement matches CRUSH calculation.

```bash
# Query placement for multiple PGs
ceph pg dump_pools_json | jq '.pool_stats[] | select(.pool_name=="ssd-pool")'
```
> Shows PG distribution statistics for specific pool. Helps identify uneven placement.

### 8.2 Performance Testing

```bash
# Write benchmark on pool
rados bench -p ssd-pool 30 write --no-cleanup
```
> Runs 30-second write test on pool. Measures throughput and latency without deleting test objects.

```bash
# Read benchmark after write test
rados bench -p ssd-pool 30 seq
```
> Runs sequential read test. Compares read performance against write metrics.

```bash
# Cleanup benchmark objects
rados -p ssd-pool cleanup
```
> Removes objects created by rados bench. Frees space after performance testing.

```bash
# Compare performance across pools
rados bench -p hdd-pool 30 write --no-cleanup && rados bench -p ssd-pool 30 write --no-cleanup
```
> Runs identical tests on different pools. Enables direct performance comparison between storage tiers.

### 8.3 Troubleshooting Common Issues

```bash
# Check for stuck PGs
ceph pg dump_stuck stale --format json-pretty
```
> Lists placement groups stuck in stale state. Indicates placement or communication issues.

```bash
# Diagnose OSD connectivity
ceph osd perf
```
> Shows OSD performance metrics including commit and apply latency. Identifies slow OSDs.

```bash
# Verify CRUSH rule calculation
ceph osd crush rule test fast-ssd-rule test-pool 10
```
> Tests rule placement for 10 objects. Helps debug rule configuration issues.

```bash
# Check for OSD weight imbalances
ceph osd df
```
> Displays OSD utilization percentages. Large variations may indicate weight or rule issues.

```bash
# Review recent cluster events
ceph health detail
```
> Shows detailed health warnings and errors. First step in diagnosing cluster issues.

```bash
# Check manager logs for crash module issues
ceph daemon mgr.<hostname> log dump | grep crash
```
> Replaces `<hostname>` with actual mgr node. Reviews crash module internal logs for errors.

---

## 9. Maintenance, Cleanup & Best Practices

### 9.1 Routine Maintenance Tasks

```bash
# Weekly health check script foundation
ceph health detail && ceph osd tree && ceph pg stat
```
> Combines key health commands for regular monitoring. Use in cron jobs or monitoring systems.

```bash
# Monthly crash report cleanup
ceph crash prune 30 && ceph crash archive-all
```
> Archives new crashes and removes reports older than 30 days. Maintains manageable crash storage.

```bash
# Quarterly CRUSH map review
ceph osd getcrushmap -o /tmp/crush_review.bin && crushtool -d /tmp/crush_review.bin -o /tmp/crush_review.txt
```
> Exports and decomplies CRUSH map for periodic architecture review. Ensures map matches physical infrastructure.

### 9.2 Cleanup Operations

```bash
# Remove unused pools safely
ceph osd pool delete test-pool test-pool --yes-i-really-really-mean-it
```
> Deletes pool after confirming no longer needed. Always verify pool contents before deletion.

```bash
# Clean orphaned crash reports
find /var/lib/ceph/crash -type d -mtime +90 -exec rm -rf {} \;
```
> Removes crash directories older than 90 days from filesystem. Use after verifying ceph crash prune completed.

```bash
# Remove deprecated CRUSH rules
ceph osd crush rule rm old-rule-name
```
> Deletes rules no longer associated with any pool. Keeps CRUSH configuration clean and maintainable.

### 9.3 Best Practices Checklist

```bash
# Verify backup before major changes
ceph -s && ceph osd tree > /tmp/pre_change_tree.txt
```
> Captures cluster state before modifications. Enables quick comparison if issues arise post-change.

```bash
# Test changes in staging first
# Always validate CRUSH edits and pool configurations in non-production environment
```
> Critical practice: Never apply untested CRUSH changes directly to production clusters.

```bash
# Document all custom rules and configurations
echo "Custom SSD rule created on $(date) for high-performance workloads" >> /root/ceph-changes.log
```
> Maintains change log for audit and troubleshooting. Include rule names, purposes, and creation dates.

```bash
# Monitor disk space for crash reports
df -h /var/lib/ceph/crash
```
> Regularly checks crash spool directory usage. Prevents disk exhaustion from unmanaged crash accumulation.

```bash
# Automate health alerts for crash events
# Configure monitoring system to alert on 'ceph crash ls-new' returning non-empty results
```
> Proactive monitoring: Alert team when new crashes occur for rapid response and analysis.

### 9.4 Upgrade and Version Management

```bash
# Check compatible Ceph versions for your deployment
curl -s https://docs.ceph.com/en/latest/releases/ | grep -A5 "stable"
```
> Reviews official release notes for upgrade paths and compatibility information.

```bash
# Verify cluster readiness for upgrade
ceph versions
```
> Shows Ceph daemon versions across cluster. Ensures all nodes are on compatible versions before upgrade.

```bash
# Backup configuration before upgrade
ceph-conf --dump > /backup/ceph-conf-$(date +%F).cfg
```
> Exports complete configuration for recovery if upgrade encounters issues.

---

## Appendix: Quick Reference Commands

### Crash Module Shortcuts
```bash
# One-liner: Archive all and prune old
ceph crash archive-all && ceph crash prune 30
```
> Efficient maintenance command combining archival and cleanup operations.

```bash
# One-liner: Check for new crashes
ceph crash ls-new | grep -q . && echo "ALERT: New crashes detected" || echo "OK: No new crashes"
```
> Simple check for automation scripts or monitoring integration.

### CRUSH Map Shortcuts
```bash
# One-liner: Export, edit, recompile workflow
ceph osd getcrushmap -o /tmp/cm.bin && crushtool -d /tmp/cm.bin -o /tmp/cm.txt
```
> Starts the CRUSH edit workflow with single command sequence.

```bash
# One-liner: Verify rule assignment
ceph osd pool get $(ceph osd pool ls | head -1) crush_rule
```
> Quickly checks CRUSH rule for first pool in list. Replace with specific pool name as needed.

### Pool Management Shortcuts
```bash
# One-liner: Create RBD-ready pool with SSD rule
ceph osd pool create new-pool 32 32 replicated && ceph osd pool set new-pool crush_rule fast-ssd-rule && ceph osd pool application enable new-pool rbd
```
> Complete pool setup command for SSD-backed RBD storage.

```bash
# One-liner: Pool utilization summary
ceph osd pool stats $(ceph osd pool ls | tr '\n' ' ') | jq '.[] | {pool: .pool_name, objects: .stats.sum.num_objects, bytes: .stats.sum.num_bytes}'
```
> Shows object count and size for all pools in JSON format. Requires jq for parsing.

---

> **Final Note:** This guide focuses on practical, hands-on operations. Always test changes in non-production environments first. Maintain backups of CRUSH maps and configurations before modifications. Monitor cluster health continuously during and after operations. For production deployments, integrate these commands with your existing monitoring and automation frameworks.

> **Documentation Updates:** Ceph evolves rapidly. Regularly check https://docs.ceph.com for latest command syntax and best practices. Version-specific features may affect command availability and behavior.

> **Community Support:** For issues beyond this guide, consult Ceph community resources: GitHub issues, IRC channels (#ceph on Libera.Chat), and the ceph-users mailing list.
