# 🐘 Ceph CRUSH Map: Complete Practical Guide
## Real-World Implementation, Operations & Management for Production Environments

> **Version:** 1.0 | **Last Updated:** March 2026 | **Target Audience:** DevOps Engineers, System Administrators, Infrastructure Teams

---

## 📋 Table of Contents

1. [Getting Started: CRUSH Map Fundamentals](#1-getting-started-crush-map-fundamentals)
2. [CRUSH Map Components: Hands-On Exploration](#2-crush-map-components-hands-on-exploration)
3. [Bucket Types & Hierarchy: Real-World Implementation](#3-bucket-types--hierarchy-real-world-implementation)
4. [CRUSH Rules: Creation, Management & Operations](#4-crush-rules-creation-management--operations)
5. [CRUSH Location: Configuration & Validation](#5-crush-location-configuration--validation)
6. [Lab Environment: 3-Node Practical Exercises](#6-lab-environment-3-node-practical-exercises)
7. [Failure Testing & Fault Tolerance Validation](#7-failure-testing--fault-tolerance-validation)
8. [Troubleshooting & Production Best Practices](#8-troubleshooting--production-best-practices)
9. [Quick Reference: Essential Commands](#9-quick-reference-essential-commands)

---

## 1. Getting Started: CRUSH Map Fundamentals

### 🎯 What is CRUSH Map? (The Practical Story)

Imagine you are managing a warehouse with 1000 shelves. When a new box arrives, you need to decide: Which shelf? Which row? Which section? If you do this manually, it takes time and errors happen.

**CRUSH Map is Ceph's intelligent warehouse manager.** It automatically decides where to store each piece of data based on rules you define — no central lookup table, no bottlenecks, just smart distribution.

```
Real-World Analogy:
📦 Your Data = Packages in a warehouse
🗺️ CRUSH Map = Smart routing system
🤖 Algorithm = Automated forklift that knows exactly where to place each package
✅ Result: Fast storage, automatic balancing, zero manual intervention
```

### 🔍 Why CRUSH Map Matters in Production

| Scenario | Without CRUSH Map | With CRUSH Map |
|----------|----------------|----------------|
| Adding new storage | Manual data migration required | Automatic rebalancing |
| Disk failure | Data loss risk, manual recovery | Automatic replication to healthy disks |
| Scaling out | Downtime for redistribution | Zero-downtime expansion |
| Performance tuning | Complex manual configuration | Rule-based optimization |

### ✅ Prerequisites Checklist

Before you start working with CRUSH Map, ensure your environment meets these requirements:

```bash
# 1. Verify Ceph cluster is healthy
ceph -s
# Expected: HEALTH_OK

# 2. Check Ceph version (must be Luminous 12.2.0+ for device classes)
ceph version
# Expected: ceph version 18.x.x (Reef) or later

# 3. Verify OSDs are up and running
ceph osd tree
# Expected: All OSDs showing 'up,in' status

# 4. Confirm network connectivity between nodes
ping ceph-node-2
ping ceph-node-3

# 5. Ensure you have admin credentials
ceph auth list | grep admin
# Expected: client.admin with appropriate capabilities
```

### 📚 Official Documentation Sources

Always refer to official sources for the latest information:

```
🔗 Ceph CRUSH Map Documentation:
   https://docs.ceph.com/en/latest/rados/operations/crush-map/

🔗 CRUSH Algorithm Technical Paper:
   https://ceph.com/assets/pdfs/weil-crush-sc06.pdf

🔗 Ceph Release Notes (for version-specific changes):
   https://docs.ceph.com/en/latest/releases/

🔗 Proxmox VE + Ceph Integration Guide:
   https://pve.proxmox.com/wiki/Ceph
```

### 🔧 Initial Setup Verification

```bash
# Step 1: Check cluster health status
ceph health detail

# Step 2: Verify monitor quorum
ceph mon dump

# Step 3: List all pools and their CRUSH rules
ceph osd pool ls detail

# Step 4: Check current CRUSH map version
ceph osd getcrushmap -o /tmp/current-crush.bin
crushtool -i /tmp/current-crush.bin --show-crush-map

# Step 5: Document current state (for rollback reference)
ceph osd tree --format json > /tmp/crush-baseline-$(date +%F).json
```

> 💡 **Pro Tip:** Always create a baseline backup before making any CRUSH Map changes. This is your safety net for production environments.

---

## 2. CRUSH Map Components: Hands-On Exploration

### 🎯 Understanding the 5 Core Components

CRUSH Map consists of five interconnected components. Think of them as building blocks that work together to determine data placement.

```
Component Hierarchy:
📦 Devices (OSDs) → The actual storage disks
🪣 Types → Categories for organizing devices (host, rack, datacenter)
🗂️ Buckets → Instances of types forming the hierarchy tree
⚙️ Rules → Policies that control how data is placed
🏷️ Device Classes → Labels for grouping similar devices (hdd, ssd, nvme)
```

### 🔍 Component 1: Devices (OSD List) - Practical Exploration

**What it is:** A list of all storage devices (OSDs) in your cluster with their unique identifiers.

```bash
# View all devices in CRUSH map
ceph osd crush dump | jq '.devices[]'

# Alternative: Clean table view
ceph osd crush show-root-class

# Check device classes assignment
ceph osd crush class ls
ceph osd crush class ls-osd hdd
ceph osd crush class ls-osd ssd
```

**Real-World Example:** Your 3-node lab setup

```
Device List Output:
[
  {"id": 0, "name": "osd.0", "class": "hdd"},
  {"id": 1, "name": "osd.1", "class": "hdd"},
  {"id": 2, "name": "osd.2", "class": "hdd"},
  {"id": 3, "name": "osd.3", "class": "hdd"},
  {"id": 4, "name": "osd.4", "class": "hdd"},
  {"id": 5, "name": "osd.5", "class": "hdd"}
]

Interpretation:
✅ 6 OSDs total, all classified as 'hdd'
✅ IDs 0-5 correspond to your 6x 300GB disks
✅ Each OSD is ready for data placement
```

**Management Operations:**

```bash
# Assign a device class manually (if auto-detection fails)
ceph osd crush set-device-class ssd osd.0 osd.1

# Remove a device class assignment
ceph osd crush rm-device-class osd.0

# Verify the change
ceph osd find osd.0 | jq '.crush_location'
```

### 🔍 Component 2: Types (Bucket Categories) - Practical Exploration

**What it is:** Predefined categories that organize your physical infrastructure into logical levels.

```bash
# View all available bucket types
ceph osd crush dump | jq '.types[]'

# Common types in order (bottom to top):
# osd → host → chassis → rack → row → pdu → pod → room → datacenter → zone → region → root
```

**Real-World Design Decision:**

```
Small Lab (Your Setup):
root → host → osd
✅ Simple, sufficient for 3 VMs

Medium Datacenter:
root → rack → host → osd
✅ Protects against rack-level failures

Large Enterprise:
root → datacenter → rack → host → osd
✅ Protects against site-level disasters

Rule of Thumb: Choose the failure domain that matches your risk tolerance.
```

**Practical Type Management:**

```bash
# Types are predefined and rarely modified
# But you can reference them when creating rules:

# Create a rule using 'rack' as failure domain
ceph osd crush rule create-replicated rack-rule default rack

# Create a rule using 'host' as failure domain (your current setup)
ceph osd crush rule create-replicated host-rule default host
```

### 🔍 Component 3: Buckets (Hierarchy Instances) - Practical Exploration

**What it is:** Actual instances of types that form your cluster's physical topology tree.

```bash
# View the complete bucket hierarchy
ceph osd tree

# Detailed view with weights and classes
ceph osd crush tree --show-shadow --format json-pretty

# Focus on a specific branch
ceph osd crush tree ceph-node-1
```

**Your Lab's Bucket Structure (Expected Output):**

```
TYPE NAME           WEIGHT  CLASS
root default        1.75798
├─ host ceph-node-1  0.58599
│  ├─ osd.0        0.29299  hdd
│  └─ osd.1        0.29299  hdd
├─ host ceph-node-2  0.58599
│  ├─ osd.2        0.29299  hdd
│  └─ osd.3        0.29299  hdd
└─ host ceph-node-3  0.58599
   ├─ osd.4        0.29299  hdd
   └─ osd.5        0.29299  hdd
```

**Understanding Weight Values:**

```
Weight Calculation:
• 0.29299 ≈ 300GB (each OSD)
• 0.58599 ≈ 600GB (each host with 2 OSDs)
• 1.75798 ≈ 1.8TB (total cluster capacity)

Why Weight Matters:
✅ CRUSH uses weight to distribute data proportionally
✅ Heavier OSDs receive more data
✅ Adjust weight to balance heterogeneous hardware
```

**Bucket Management Operations:**

```bash
# Add a new bucket (e.g., for future expansion)
ceph osd crush add-bucket rack-01 rack

# Move a host into a rack bucket
ceph osd crush move ceph-node-1 rack=rack-01

# Rename a bucket (use with caution)
ceph osd crush rename-bucket old-name new-name

# Remove an empty bucket (must be empty first)
ceph osd crush remove rack-01

# Verify hierarchy changes
ceph osd tree
```

### 🔍 Component 4: Rules (Placement Policies) - Practical Exploration

**What it is:** The decision-making logic that tells CRUSH how to select OSDs for data placement.

```bash
# List all available rules
ceph osd crush rule ls

# View detailed rule configuration
ceph osd crush rule dump replicated_rule

# View rule in human-readable format
ceph osd crush rule dump replicated_rule | jq '.steps[]'
```

**Decoding a Rule Step-by-Step:**

```json
{
  "rule_name": "replicated_rule",
  "type": "replicated",
  "min_size": 1,
  "max_size": 10,
  "steps": [
    {"op": "take", "item_name": "default"},
    {"op": "chooseleaf", "num": 0, "type": "host"},
    {"op": "emit"}
  ]
}
```

```
Step-by-Step Interpretation:

1️⃣ "take default"
   → Start from the 'default' root bucket
   → This is your entry point into the hierarchy

2️⃣ "chooseleaf firstn 0 type host"
   → firstn 0 = select N items where N = pool's replication size (usually 3)
   → type host = ensure each selection is from a different host
   → chooseleaf = pick the leaf node (OSD) within each selected host

3️⃣ "emit"
   → Output the selected OSDs as the final placement decision

✅ Result: 3 copies of data on 3 different hosts = fault tolerance
```

**Rule Creation Workflow:**

```bash
# Method 1: Quick creation via CLI (recommended for most cases)
ceph osd crush rule create-replicated <rule-name> <root> <failure-domain> [device-class]

# Example: Create SSD-optimized rule
ceph osd crush rule create-replicated fast-ssd default host ssd

# Method 2: Manual editing for complex scenarios
# Step 1: Backup current map
ceph osd getcrushmap -o /tmp/crush-backup.bin

# Step 2: Decompile to text
crushtool -d /tmp/crush-backup.bin -o /tmp/crush.txt

# Step 3: Edit /tmp/crush.txt with your custom rule
# Step 4: Recompile and apply
crushtool -c /tmp/crush.txt -o /tmp/crush-new.bin
ceph osd setcrushmap -i /tmp/crush-new.bin
```

### 🔍 Component 5: Device Classes - Practical Exploration

**What it is:** Labels that group OSDs by performance characteristics, enabling tiered storage policies.

```bash
# List all device classes in use
ceph osd crush class ls

# Show which OSDs belong to a specific class
ceph osd crush class ls-osd hdd
ceph osd crush class ls-osd ssd

# Check class distribution across the cluster
ceph osd df class
```

**Real-World Use Cases:**

```
Scenario 1: Mixed HDD/SSD Environment
• Class 'hdd' OSDs: High capacity, lower cost, for archival data
• Class 'ssd' OSDs: High performance, for hot data and databases
• Rule: Direct fast pools to SSD class, cold pools to HDD class

Scenario 2: NVMe Tier for Ultra-Fast Workloads
• Class 'nvme' OSDs: Sub-millisecond latency, for real-time analytics
• Rule: Create a dedicated rule that only selects nvme-class OSDs

Scenario 3: Future-Proofing Your Lab
• Even if all current OSDs are 'hdd', define classes now
• When you add SSDs later, rules are ready to use them
```

**Device Class Management:**

```bash
# Create a new device class (if needed)
ceph osd crush class create nvme-premium

# Assign OSDs to a class
ceph osd crush set-device-class nvme osd.0 osd.1

# Verify assignment
ceph osd find osd.0 | jq '.class'

# Remove class assignment (reverts to auto-detected)
ceph osd crush rm-device-class osd.0

# Create a rule that filters by class
ceph osd crush rule create-replicated nvme-rule default host nvme
```

---

## 3. Bucket Types & Hierarchy: Real-World Implementation

### 🎯 Designing Your Hierarchy: A Practical Approach

Your hierarchy design directly impacts fault tolerance, performance, and manageability. Let's build it step by step.

### 🔍 Step 1: Map Your Physical Infrastructure

```bash
# Document your current setup
cat > /tmp/infrastructure-map.txt << 'EOF'
Physical Layout:
- Proxmox VE Host: 1 physical server
- Ceph VMs: 3 virtual machines (ceph-node-1, ceph-node-2, ceph-node-3)
- OSDs per VM: 2 x 300GB disks each
- Network: Single subnet, 1Gbps

Failure Domains to Consider:
- Disk failure: Individual OSD goes down
- Host failure: Entire VM crashes or reboots
- Hypervisor failure: Proxmox host goes down (affects all 3 VMs)
- Network failure: Switch or cable issue

Recommended Hierarchy for This Setup:
root=default → host=<vm-name> → osd=<osd-id>
EOF

cat /tmp/infrastructure-map.txt
```

### 🔍 Step 2: Verify Current Hierarchy

```bash
# Check how Ceph currently sees your topology
ceph osd tree

# Get detailed JSON for programmatic analysis
ceph osd tree --format json | jq '.nodes[] | select(.type=="host") | {name, weight, children}'

# Verify OSD locations match expectations
for osd in $(seq 0 5); do
    echo "OSD $osd location:"
    ceph osd find $osd | jq '.crush_location'
done
```

### 🔍 Step 3: Adjust Hierarchy if Needed

```bash
# Scenario: OSDs were auto-placed incorrectly during initial setup
# Fix: Manually set correct locations

# Example: Move osd.2 to correct host
ceph osd crush set osd.2 0.29299 root=default host=ceph-node-2

# Verify the change
ceph osd find osd.2 | jq '.crush_location'

# Repeat for other OSDs if needed
# Note: Weight value 0.29299 ≈ 300GB in TiB
```

### 🔍 Step 4: Test Hierarchy with Data Placement

```bash
# Create a test pool using default rule
ceph osd pool create hierarchy-test 128 128

# Write test objects
for i in {1..10}; do
    rados -p hierarchy-test put "obj-$i" /etc/hostname
done

# Check where objects are placed
for i in {1..3}; do
    echo "Object obj-$i placement:"
    ceph osd map hierarchy-test "obj-$i"
done
```

**Expected Output Interpretation:**

```
Example: ceph osd map hierarchy-test obj-1
→ up ([2,0,4], p2) acting ([2,0,4], p2)

Breakdown:
• up ([2,0,4]): Data currently on osd.2, osd.0, osd.4
• p2: osd.2 is the primary (handles writes first)
• All three OSDs are on different hosts ✓
• This confirms host-level failure domain is working ✓
```

### 🔍 Step 5: Advanced Hierarchy for Future Scaling

```bash
# Prepare for rack-level organization (even if you have one rack now)
ceph osd crush add-bucket rack-alpha rack

# Move hosts into the rack
ceph osd crush move ceph-node-1 rack=rack-alpha
ceph osd crush move ceph-node-2 rack=rack-alpha
ceph osd crush move ceph-node-3 rack=rack-alpha

# Move rack under root
ceph osd crush move rack-alpha root=default

# Create a rack-aware rule
ceph osd crush rule create-replicated rack-aware-rule default rack

# Test with new pool
ceph osd pool create rack-test 128 128 replicated rack-aware-rule
rados -p rack-test put demo /etc/hostname
ceph osd map rack-test demo
```

> 💡 **Design Principle:** Start simple (host-level), then add complexity (rack, datacenter) as your infrastructure grows. Over-engineering early creates unnecessary management overhead.

---

## 4. CRUSH Rules: Creation, Management & Operations

### 🎯 Rule Creation Workflow: From Concept to Production

### 🔍 Step 1: Define Your Requirements

Before creating a rule, answer these questions:

```
✅ What type of data will this pool store?
   - Hot data (frequent access) → SSD/NVMe rule
   - Cold data (archival) → HDD rule
   - Mixed → Hybrid rule

✅ What is your fault tolerance requirement?
   - Survive 1 disk failure → 3-way replication, host-level domain
   - Survive 1 rack failure → 3-way replication, rack-level domain
   - Survive site failure → Multi-datacenter setup

✅ What are your performance expectations?
   - Low latency → Primary on SSD, replicas on HDD
   - High throughput → Striping across many OSDs
   - Cost-efficient → Erasure coding instead of replication
```

### 🔍 Step 2: Create Rules Using CLI (Recommended Method)

```bash
# Pattern: ceph osd crush rule create-replicated <name> <root> <failure-domain> [class]

# Example 1: Standard replicated rule (host-level)
ceph osd crush rule create-replicated standard-host-rule default host

# Example 2: Rack-aware rule for larger deployments
ceph osd crush rule create-replicated standard-rack-rule default rack

# Example 3: SSD-optimized rule
ceph osd crush rule create-replicated fast-ssd-rule default host ssd

# Example 4: NVMe ultra-fast rule
ceph osd crush rule create-replicated nvme-rule default host nvme

# Example 5: Hybrid rule (requires manual editing - see advanced section)
```

### 🔍 Step 3: Verify Rule Configuration

```bash
# List all rules
ceph osd crush rule ls

# Inspect a specific rule in detail
ceph osd crush rule dump standard-host-rule

# Check rule steps in readable format
ceph osd crush rule dump standard-host-rule | jq '.steps[] | {op, type, num, item_name}'
```

**Expected Output for standard-host-rule:**

```json
{
  "op": "take",
  "item_name": "default"
}
{
  "op": "chooseleaf",
  "num": 0,
  "type": "host"
}
{
  "op": "emit"
}
```

### 🔍 Step 4: Apply Rule to a Pool

```bash
# Create a new pool with your rule
ceph osd pool create my-app-pool 128 128 replicated standard-host-rule

# Or change rule on existing pool
ceph osd pool set existing-pool crush_rule standard-host-rule

# Verify the assignment
ceph osd pool get my-app-pool crush_rule
# Expected output: crush_rule: standard-host-rule
```

### 🔍 Step 5: Test Data Placement with New Rule

```bash
# Write test data to the pool
rados -p my-app-pool put test-object-1 /etc/hostname

# Check placement decision
ceph osd map my-app-pool test-object-1

# Verify placement matches rule expectations:
# - For host-level rule: 3 OSDs on 3 different hosts
# - For SSD rule: All OSDs have class=ssd
# - For rack rule: 3 OSDs on 3 different racks
```

### 🔍 Advanced: Manual Rule Editing for Complex Scenarios

```bash
# Use case: Hybrid rule with SSD primary + HDD replicas

# Step 1: Backup current CRUSH map
ceph osd getcrushmap -o /tmp/crush-backup-$(date +%F).bin

# Step 2: Decompile to editable text format
crushtool -d /tmp/crush-backup-$(date +%F).bin -o /tmp/crush-edit.txt

# Step 3: Add custom rule to the file (append to end)
cat >> /tmp/crush-edit.txt << 'RULE'

rule hybrid-ssd-primary {
    id 10
    type replicated
    min_size 1
    max_size 10
    # Step 1: Select 1 OSD from SSD class for primary
    step take default class ssd
    step chooseleaf firstn 1 type host
    step emit
    # Step 2: Select remaining replicas from HDD class
    step take default class hdd
    step chooseleaf firstn -1 type host
    step emit
}
RULE

# Step 4: Recompile and validate syntax
crushtool -c /tmp/crush-edit.txt -o /tmp/crush-new.bin
crushtool -i /tmp/crush-new.bin --show-crush-map | grep -A 15 "hybrid-ssd-primary"

# Step 5: Apply to cluster (during maintenance window)
ceph osd setcrushmap -i /tmp/crush-new.bin

# Step 6: Verify application
ceph osd crush rule ls | grep hybrid
ceph osd crush rule dump hybrid-ssd-primary
```

### 🔍 Rule Management Operations

```bash
# List all rules with brief info
ceph osd crush rule ls

# Get detailed JSON for automation
ceph osd crush rule dump standard-host-rule --format json

# Delete a rule (only if no pool uses it)
ceph osd crush rule rm unused-rule-name

# Test a rule without applying to cluster
crushtool -i /tmp/crush.bin --test \
    --rule 0 \
    --num-rep 3 \
    --min-x 0 --max-x 100 \
    --show-mappings | head -20
```

### 🔍 Rule Performance Optimization

```bash
# Check rule efficiency statistics
crushtool -i /tmp/crush.bin --test \
    --rule 0 \
    --num-rep 3 \
    --min-x 0 --max-x 1000 \
    --show-statistics

# Key metrics to watch:
# - "average mappings per placement": Should be close to replication size
# - "std dev of mappings": Lower is better (more predictable placement)
# - "collisions": Should be minimal

# Adjust tunables if needed (advanced)
ceph osd crush tunables
# Available profiles: legacy, argonaut, bobtail, firefly, hammer, jewel, optimal

# Apply optimal settings (test in lab first)
ceph osd crush tunables optimal
```

---

## 5. CRUSH Location: Configuration & Validation

### 🎯 Understanding CRUSH Location in Practice

CRUSH Location defines where each OSD sits in your hierarchy. Correct location assignment is critical for fault tolerance.

### 🔍 Method 1: Configuration via ceph.conf (Recommended)

```bash
# Edit Ceph configuration file on each OSD node
# File: /etc/ceph/ceph.conf

# Add under [osd] section:
[osd]
# For ceph-node-1 VM:
crush_location = root=default host=ceph-node-1

# For ceph-node-2 VM:
# crush_location = root=default host=ceph-node-2

# For ceph-node-3 VM:
# crush_location = root=default host=ceph-node-3

# Save and restart OSD daemons
systemctl restart ceph-osd.target
```

**Verification:**

```bash
# Check if location was applied correctly
ceph osd find osd.0 | jq '.crush_location'
# Expected: {"root":"default","host":"ceph-node-1"}

# View all OSD locations in tree format
ceph osd tree
```

### 🔍 Method 2: Command-Line Assignment

```bash
# Assign location during OSD creation or reconfiguration
# Syntax: ceph osd crush set <osd> <weight> <location-key>=<value> ...

# Example for osd.0 on ceph-node-1
ceph osd crush set osd.0 0.29299 root=default host=ceph-node-1

# Example for osd.2 on ceph-node-2
ceph osd crush set osd.2 0.29299 root=default host=ceph-node-2

# Verify assignments
ceph osd tree | grep -E "ceph-node-1|osd.[01]"
ceph osd tree | grep -E "ceph-node-2|osd.[23]"
```

### 🔍 Method 3: Dynamic Location Hook (Advanced)

```bash
# Use case: Automatically set location based on VM metadata or cloud tags

# Step 1: Create location script
cat > /usr/local/bin/ceph-crush-location-hook << 'EOF'
#!/bin/bash
# Reads VM metadata to determine CRUSH location

# Example: Read rack info from Proxmox tags or cloud-init
RACK=$(grep -oP 'rack:\K\w+' /etc/proxmox-vm-info 2>/dev/null || echo "default")
HOST=$(hostname -s)

# Output in CRUSH location format
echo "root=default rack=$RACK host=$HOST"
EOF

chmod +x /usr/local/bin/ceph-crush-location-hook

# Step 2: Configure Ceph to use the hook
# In /etc/ceph/ceph.conf under [osd]:
[osd]
crush_location_hook = /usr/local/bin/ceph-crush-location-hook

# Step 3: Restart OSD to apply
systemctl restart ceph-osd@0
```

### 🔍 Location Validation Workflow

```bash
# Step 1: Check individual OSD location
ceph osd find osd.0

# Step 2: Verify hierarchy integrity
ceph osd tree --format json | jq '
  .nodes[] | 
  select(.type=="host") | 
  {host: .name, osd_count: (.children | length), weight: .weight}
'

# Step 3: Validate failure domain compliance
# For host-level rule: No two replicas of same PG on same host
ceph pg dump pgs_brief | head -5 | while read pg; do
    echo "PG $pg placement:"
    ceph pg map $pg 2>/dev/null | grep -oP 'up \(\[\K[0-9,]+' | tr ',' '\n' | while read osd; do
        ceph osd find $osd 2>/dev/null | jq -r '.crush_location.host'
    done | sort -u | wc -l
done
```

### 🔍 Location Troubleshooting

```bash
# Issue: OSD shows wrong host after VM migration
# Solution: Re-assign location

ceph osd crush set osd.3 0.29299 root=default host=ceph-node-3
ceph osd perf

# Issue: New OSD not appearing in expected bucket
# Solution: Check auto-placement settings

grep crush /etc/ceph/ceph.conf
# Ensure crush_location or crush_location_hook is configured

# Issue: Weight mismatch causing uneven distribution
# Solution: Recalculate and update weights

# Get actual OSD size in TiB
lsblk -b /dev/sdb | awk 'NR==2 {printf "%.4f\n", $4/1024^4}'

# Update CRUSH weight
ceph osd crush reweight osd.0 0.29299
```

### 🔍 Location Best Practices

```
✅ Always include root=default in location strings
✅ Use consistent naming: host=ceph-node-1 (not ceph_node_1 or cephnode1)
✅ Document location assignments in your infrastructure inventory
✅ Test location changes in lab before production
✅ Monitor rebalancing after location changes: ceph -w
```

---

## 6. Lab Environment: 3-Node Practical Exercises

### 🎯 Your Lab Setup Recap

```
Infrastructure:
• Hypervisor: Proxmox VE (1 physical host)
• Ceph Nodes: 3 VMs (ceph-node-1, ceph-node-2, ceph-node-3)
• OSDs: 2 x 300GB per VM = 6 OSDs total
• Network: Single subnet, 1Gbps
• Ceph Version: 18.x (Reef) or later

Current CRUSH Configuration:
• Hierarchy: root=default → host=<vm> → osd=<id>
• Default Rule: replicated_rule (host-level failure domain)
• Device Class: All OSDs classified as 'hdd'
```

### 🔍 Exercise 1: Baseline Health Check

```bash
# Connect to any Ceph node
ssh root@ceph-node-1

# Check overall cluster health
ceph -s
# Expected: HEALTH_OK, 6/6 OSDs up, 96/96 PGs active+clean

# Verify OSD status
ceph osd tree
# Expected: 3 hosts, each with 2 OSDs, all 'up,in'

# Check pool configuration
ceph osd pool ls detail
# Note the crush_rule assigned to each pool

# Document baseline for comparison
ceph osd tree --format json > /tmp/lab-baseline.json
```

### 🔍 Exercise 2: Create and Test a New Pool

```bash
# Create a pool with explicit CRUSH rule
ceph osd pool create lab-exercise-pool 128 128 replicated replicated_rule

# Initialize the pool for RBD (if using block storage)
rbd pool init lab-exercise-pool

# Create a test RBD image
rbd create lab-exercise-pool/test-image --size 1024

# Map and format the image (on a client node)
rbd map lab-exercise-pool/test-image
mkfs.ext4 /dev/rbd/lab-exercise-pool/test-image

# Mount and write test data
mkdir /mnt/test-storage
mount /dev/rbd/lab-exercise-pool/test-image /mnt/test-storage
echo "CRUSH Map Lab Test - $(date)" > /mnt/test-storage/test-file.txt

# Verify data persistence
cat /mnt/test-storage/test-file.txt
```

### 🔍 Exercise 3: Trace Data Placement

```bash
# Find which OSDs store your test data
# First, get the object name for your file
rados -p lab-exercise-pool ls | grep test

# Map object to OSDs
ceph osd map lab-exercise-pool test-file.txt

# Interpret the output:
# Example: up ([2,0,4], p2)
# → Primary: osd.2 (ceph-node-2)
# → Secondary: osd.0 (ceph-node-1)
# → Tertiary: osd.4 (ceph-node-3)
# → All on different hosts ✓

# Verify physical distribution
for osd_id in 2 0 4; do
    echo "OSD $osd_id location:"
    ceph osd find $osd_id | jq '.crush_location.host'
done
```

### 🔍 Exercise 4: Simulate OSD Failure

```bash
# ⚠️ Lab-only exercise - do not run in production without planning

# Step 1: Mark osd.0 as 'out' (triggers data rebalancing)
ceph osd out osd.0

# Step 2: Monitor rebalancing progress
watch -n 2 'ceph -s | grep -E "health|recovery|osd.0"'

# Expected progression:
# HEALTH_WARN → recovery starts → HEALTH_OK when complete

# Step 3: Verify data remains accessible during recovery
# From a client node, try to read your test file
cat /mnt/test-storage/test-file.txt
# Should succeed even during rebalancing ✓

# Step 4: Bring osd.0 back online
ceph osd in osd.0

# Step 5: Monitor rebalancing back to original state
watch -n 2 'ceph -s'
```

### 🔍 Exercise 5: Simulate Host Failure

```bash
# ⚠️ Requires Proxmox VE host access

# Step 1: From Proxmox host, stop ceph-node-2 VM
# (Replace <VM-ID> with actual ID from 'qm list')
qm stop <VM-ID-of-ceph-node-2>

# Step 2: From remaining Ceph node, check cluster status
ceph -s
# Expected: HEALTH_WARN, 2 OSDs down, some PGs degraded

# Step 3: Verify data accessibility
# From client node, attempt to read test file
cat /mnt/test-storage/test-file.txt
# Should succeed - replicas on other hosts provide redundancy ✓

# Step 4: Check which OSDs are serving the data now
ceph osd map lab-exercise-pool test-file.txt
# Should show only osd.0 and osd.4 (the surviving replicas)

# Step 5: Restart the VM
qm start <VM-ID-of-ceph-node-2>

# Step 6: Monitor automatic recovery
ceph -w
# Wait for HEALTH_OK and all PGs active+clean
```

### 🔍 Exercise 6: Create and Apply a Custom Rule

```bash
# Scenario: Create a rule that prefers SSDs (for future hardware upgrades)

# Step 1: Create the rule (even though all current OSDs are HDD)
ceph osd crush rule create-replicated future-ssd-rule default host ssd

# Step 2: Create a pool using this rule
ceph osd pool create ssd-ready-pool 128 128 replicated future-ssd-rule

# Step 3: Attempt to write data (will fail initially - no SSD OSDs)
rados -p ssd-ready-pool put test /etc/hostname 2>&1
# Expected error: "no suitable OSDs found" - this is expected

# Step 4: Document the rule for future use
ceph osd crush rule dump future-ssd-rule > /tmp/ssd-rule-config.json

# Step 5: When you add SSD OSDs later:
# a) Assign them to 'ssd' class:
#    ceph osd crush set-device-class ssd osd.6 osd.7
# b) The rule will automatically start working
# c) Existing pools can be migrated:
#    ceph osd pool set existing-pool crush_rule future-ssd-rule
```

### 🔍 Exercise 7: Cleanup and Reset

```bash
# Remove test pools
ceph osd pool delete lab-exercise-pool lab-exercise-pool --yes-i-really-really-mean-it
ceph osd pool delete ssd-ready-pool ssd-ready-pool --yes-i-really-really-mean-it

# Remove custom rules (if no longer needed)
ceph osd crush rule rm future-ssd-rule

# Verify clean state
ceph osd pool ls
ceph osd crush rule ls

# Final health check
ceph -s
# Expected: HEALTH_OK, all systems nominal
```

---

## 7. Failure Testing & Fault Tolerance Validation

### 🎯 Why Test Failure Scenarios?

Testing failure scenarios in your lab prepares you for real-world incidents. The goal is not to break things, but to verify that your CRUSH configuration provides the fault tolerance you expect.

### 🔍 Test Framework: The 3-Layer Approach

```
Layer 1: Component Failure
• Single OSD failure
• Single disk I/O error
• OSD daemon crash

Layer 2: Host Failure
• VM crash or reboot
• Hypervisor resource exhaustion
• Network interface failure on a node

Layer 3: Infrastructure Failure
• Network partition between nodes
• Storage backend failure (Proxmox storage)
• Power loss simulation (via VM pause)
```

### 🔍 Test 1: Single OSD Failure

```bash
# Preparation
ceph -s  # Confirm HEALTH_OK baseline
rados -p test-pool put failure-test-1 /etc/hostname

# Induce failure: Mark OSD as down
ceph osd fail osd.1

# Immediate verification
ceph -s
# Expected: HEALTH_WARN, 1 osd down, some PGs degraded

# Data accessibility test
rados -p test-pool get failure-test-1 /tmp/recovered-1
diff /etc/hostname /tmp/recovered-1  # Should show no differences

# Recovery monitoring
ceph -w  # Watch until HEALTH_OK returns

# Post-recovery validation
ceph osd ok osd.1  # Clear the failure flag
ceph osd tree | grep osd.1  # Should show 'up,in'
```

### 🔍 Test 2: Host-Level Failure

```bash
# Preparation: Document current PG distribution
ceph pg dump pgs_brief > /tmp/pg-baseline.txt

# Induce failure: Stop the VM (from Proxmox host)
qm stop <VM-ID-of-ceph-node-3>

# Verification from surviving nodes
ceph -s
# Expected: HEALTH_WARN, 2 osds down, multiple PGs degraded

# Critical test: Can clients still access data?
# From a separate client machine:
rbd map test-pool/test-image
mount /dev/rbd/test-pool/test-image /mnt/test
cat /mnt/test/critical-data.txt  # Should succeed

# Check PG recovery status
ceph pg stat
# Note: recovery_progress should show active rebuilding

# Restore the host
qm start <VM-ID-of-ceph-node-3>

# Monitor full recovery
watch 'ceph -s | grep -E "health|recovery|pgmap"'
# Wait for: HEALTH_OK, 0 recovering PGs

# Validate data integrity post-recovery
rbd diff test-pool/test-image | wc -l  # Should match pre-failure count
```

### 🔍 Test 3: Network Partition Simulation

```bash
# ⚠️ Advanced test - use with caution

# Preparation: Enable detailed logging
ceph tell osd.* injectargs '--debug_osd=20'

# Induce partition: Block traffic to ceph-node-2
# From Proxmox host or another node:
iptables -A INPUT -s <ceph-node-2-IP> -j DROP
iptables -A OUTPUT -d <ceph-node-2-IP> -j DROP

# Verification
ceph -s
# Expected: HEALTH_WARN, osds on ceph-node-2 marked 'down'

# Monitor monitor quorum (should remain stable)
ceph mon dump

# Test client operations
# Writes should still succeed (to surviving OSDs)
# Reads may be slower (fetching from secondary replicas)

# Restore connectivity
iptables -D INPUT -s <ceph-node-2-IP> -j DROP
iptables -D OUTPUT -d <ceph-node-2-IP> -j DROP

# Monitor reintegration
ceph -w
# Watch for: osds on ceph-node-2 returning to 'up,in'
# Wait for: HEALTH_OK, all PGs active+clean

# Cleanup logging
ceph tell osd.* injectargs '--debug_osd=0'
```

### 🔍 Validation Checklist: Fault Tolerance Metrics

```bash
# Create a validation script for regular testing
cat > /usr/local/bin/crush-fault-validation.sh << 'EOF'
#!/bin/bash
echo "=== CRUSH Fault Tolerance Validation ==="
echo "Timestamp: $(date)"
echo ""

# 1. Cluster Health
echo "1. Cluster Health Status:"
ceph health detail | head -5
echo ""

# 2. OSD Availability
echo "2. OSD Availability:"
ceph osd tree | grep -E "up.*in" | wc -l
echo "OSDs up/in out of $(ceph osd tree | grep osd. | wc -l) total"
echo ""

# 3. PG State Distribution
echo "3. Placement Group States:"
ceph pg stat | grep -oP '[0-9]+ [a-z+]+' | sort | uniq -c
echo ""

# 4. Failure Domain Compliance
echo "4. Failure Domain Check (host-level):"
TEST_POOL=$(ceph osd pool ls | head -1)
TEST_OBJ="validation-$(date +%s)"
rados -p $TEST_POOL put $TEST_OBJ /etc/hostname 2>/dev/null
PLACEMENT=$(ceph osd map $TEST_POOL $TEST_OBJ 2>/dev/null | grep -oP 'up \(\[\K[0-9,]+')
UNIQUE_HOSTS=$(echo $PLACEMENT | tr ',' '\n' | xargs -I{} ceph osd find {} 2>/dev/null | jq -r '.crush_location.host' | sort -u | wc -l)
echo "Test object placed on $UNIQUE_HOSTS unique hosts (expected: 3)"
rados -p $TEST_POOL rm $TEST_OBJ 2>/dev/null
echo ""

# 5. Recovery Capability
echo "5. Recovery Readiness:"
ceph osd dump | grep -E "full|nearfull" || echo "No capacity warnings"
echo ""

echo "=== Validation Complete ==="
EOF

chmod +x /usr/local/bin/crush-fault-validation.sh
```

### 🔍 Interpreting Test Results

```
✅ PASS Indicators:
• HEALTH_OK returns within expected time after failure
• Client I/O continues during recovery (may be slower)
• No data loss or corruption detected post-recovery
• PGs return to 'active+clean' state
• Recovery traffic doesn't saturate network

⚠️ WARNING Indicators:
• Recovery takes longer than expected (check OSD performance)
• Temporary I/O latency spikes during rebalancing
• Some PGs remain 'degraded' after OSD returns

❌ FAIL Indicators:
• Data corruption or loss detected
• PGs stuck in 'incomplete' or 'down' state
• Cluster fails to return to HEALTH_OK
• Client I/O completely blocked during failure
```

### 🔍 Post-Test Documentation

```bash
# After each test, document findings
cat > /var/log/crush-test-report-$(date +%F).md << EOF
# CRUSH Fault Tolerance Test Report
Date: $(date)
Test Type: [OSD Failure | Host Failure | Network Partition]

## Pre-Test State
$(ceph -s | head -10)

## Test Actions
[Document exact commands and timing]

## Observations During Test
[Note client behavior, recovery progress, any anomalies]

## Post-Test State
$(ceph -s | head -10)

## Validation Results
- Data integrity: [PASS/FAIL]
- Client availability: [PASS/FAIL]
- Recovery time: [X minutes]
- Issues encountered: [None / List]

## Recommendations
[Action items for production hardening]
EOF
```

---

## 8. Troubleshooting & Production Best Practices

### 🎯 Common Issues and Practical Solutions

### 🔍 Issue: "HEALTH_WARN: crush map has legacy tunables"

```bash
# Diagnosis
ceph osd crush tunables
# Output shows: profile: legacy

# Impact: Suboptimal data distribution, potential rebalancing on upgrade

# Solution (test in lab first):
ceph osd crush tunables optimal

# Verification
ceph health detail | grep crush  # Warning should disappear
ceph -w  # Monitor for rebalancing activity

# Rollback if issues occur:
ceph osd crush tunables legacy
```

### 🔍 Issue: "PGs stuck degraded after OSD recovery"

```bash
# Diagnosis
ceph pg dump_stuck stale | head -10
ceph pg map <pg-id>

# Common causes:
# 1. OSD not fully synced after returning online
# 2. Network partition preventing peer communication
# 3. Insufficient recovery resources

# Resolution steps:
# Step 1: Verify OSD status
ceph osd tree | grep -E "osd.X|ceph-node-Y"

# Step 2: Check OSD logs for errors
journalctl -u ceph-osd@X -n 50

# Step 3: Manually trigger recovery (if needed)
ceph pg repair <pg-id>

# Step 4: Adjust recovery parameters temporarily
ceph tell osd.* injectargs '--osd-recovery-max-active 2'
ceph tell osd.* injectargs '--osd-max-backfills 1'

# Step 5: Monitor progress
ceph -w | grep -E "recover|backfill"
```

### 🔍 Issue: "Uneven data distribution across OSDs"

```bash
# Diagnosis
ceph osd df tree
# Look for OSDs with significantly higher/lower %USE

# Common causes:
# 1. Incorrect weight values in CRUSH map
# 2. Recent OSD additions not fully rebalanced
# 3. Heterogeneous hardware without proper weighting

# Resolution:
# Step 1: Calculate correct weight (1.0 ≈ 1 TiB)
lsblk -b /dev/sdX | awk 'NR==2 {printf "%.4f\n", $4/1024^4}'

# Step 2: Update OSD weight
ceph osd crush reweight osd.0 0.29299

# Step 3: Enable automatic balancing (Luminous+)
ceph mgr module enable balancer
ceph balancer mode upmap
ceph balancer on

# Step 4: Monitor rebalancing
ceph balancer status
ceph -w | grep -E "balance|reweight"
```

### 🔍 Issue: "Rule cannot find enough OSDs for placement"

```bash
# Diagnosis
ceph osd crush rule dump <rule-name>
ceph osd crush class ls-osd <class-if-specified>

# Common causes:
# 1. Device class filter excludes available OSDs
# 2. Failure domain too restrictive for cluster size
# 3. OSDs marked 'out' or 'down'

# Resolution:
# Step 1: Verify OSD availability
ceph osd tree | grep -E "up.*in"

# Step 2: Check device class assignments
ceph osd crush class ls
ceph osd crush class ls-osd hdd  # or ssd/nvme

# Step 3: Adjust rule if needed
# Option A: Remove class filter
ceph osd crush rule create-replicated new-rule default host

# Option B: Assign correct class to OSDs
ceph osd crush set-device-class hdd osd.0 osd.1

# Step 4: Test rule placement
crushtool -i /tmp/crush.bin --test --rule <rule-id> --num-rep 3 --show-mappings
```

### 🎯 Production Best Practices Checklist

```yaml
🔐 Change Management:
  • Always backup CRUSH map before modifications:
      ceph osd getcrushmap -o /backup/crush-$(date +%F).bin
  • Test changes in lab environment first
  • Document every change: what, why, when, rollback steps
  • Use maintenance windows for major modifications:
      ceph osd set noout
      ceph osd set norebalance
      # ... make changes ...
      ceph osd unset noout
      ceph osd unset norebalance

📊 Monitoring & Alerting:
  • Essential metrics to track:
      - ceph_health_status (0=OK, 1=WARN, 2=ERR)
      - ceph_osd_up (count of up OSDs)
      - ceph_pg_active (count of active PGs)
      - ceph_recovery_ops (recovery throughput)
  • Set up alerts for:
      - HEALTH_WARN/ERR lasting > 5 minutes
      - OSD down for > 10 minutes
      - PGs stuck degraded for > 30 minutes
  • Use Prometheus + Grafana for visualization:
      ceph mgr module enable prometheus

🔄 Performance Optimization:
  • Tune recovery parameters for your network:
      ceph config set osd osd_recovery_max_active 3
      ceph config set osd osd_max_backfills 2
  • Adjust CRUSH tunables for cluster size:
      ceph osd crush tunables optimal  # For clusters > 10 OSDs
  • Use device classes for tiered performance:
      • SSD/NVMe for hot data, metadata, journals
      • HDD for cold data, backups, archives
  • Enable balancer module for automatic optimization:
      ceph balancer mode upmap
      ceph balancer on

🛡️ Security & Access Control:
  • Limit CRUSH map modification to admin users:
      ceph auth caps client.admin mon 'allow *' osd 'allow *'
  • Audit CRUSH changes via Ceph logs:
      grep "crush" /var/log/ceph/ceph.log
  • Encrypt CRUSH map backups if storing offsite
  • Regularly rotate admin credentials

🧪 Testing Strategy:
  • Monthly: Run automated fault validation script
  • Quarterly: Simulate single host failure in lab
  • Bi-annually: Test disaster recovery procedures
  • Before upgrades: Validate CRUSH compatibility with new version
```

### 🔍 Maintenance Workflow: Monthly CRUSH Health Check

```bash
#!/bin/bash
# /usr/local/bin/crush-monthly-check.sh

echo "=== Monthly CRUSH Map Health Check ==="
echo "Date: $(date)"
echo ""

# 1. Backup current state
ceph osd getcrushmap -o /backup/crush-monthly-$(date +%F).bin
echo "✓ Backup created"

# 2. Health overview
echo -e "\n--- Cluster Health ---"
ceph -s | grep -E "health:|osd:|pgmap:"

# 3. CRUSH map integrity
echo -e "\n--- CRUSH Map Summary ---"
echo "Rules: $(ceph osd crush rule ls | wc -l)"
echo "Buckets: $(ceph osd tree | grep -v "TYPE\|root" | wc -l)"
echo "OSDs: $(ceph osd tree | grep osd. | wc -l)"

# 4. Weight distribution check
echo -e "\n--- Weight Distribution ---"
ceph osd df tree | grep -E "osd\." | awk '{print $1, $3, $6}' | column -t

# 5. Rule validation
echo -e "\n--- Rule Validation ---"
for rule in $(ceph osd crush rule ls); do
    echo "Rule: $rule"
    ceph osd crush rule dump $rule | jq -r '.steps[] | select(.op=="chooseleaf") | "  Failure domain: \(.type)"'
done

# 6. Tunables check
echo -e "\n--- Tunables ---"
ceph osd crush tunables

# 7. Recommendations
echo -e "\n--- Recommendations ---"
if ceph health detail | grep -q "legacy.*tunables"; then
    echo "⚠ Consider updating CRUSH tunables to 'optimal'"
fi
if ceph osd df | awk 'NR>1 {if($6>90) print}' | grep -q .; then
    echo "⚠ Some OSDs >90% full - consider rebalancing"
fi

echo -e "\n=== Check Complete ==="
```

---

## 9. Quick Reference: Essential Commands

### 🔍 CRUSH Map Inspection

```bash
# View hierarchy tree
ceph osd tree
ceph osd tree --format json-pretty

# Detailed CRUSH map dump
ceph osd crush dump
ceph osd crush dump --format json | jq '.buckets[] | select(.type_name=="host")'

# List and inspect rules
ceph osd crush rule ls
ceph osd crush rule dump <rule-name>

# Check device classes
ceph osd crush class ls
ceph osd crush class ls-osd hdd

# Find OSD location
ceph osd find osd.0
ceph osd find osd.0 | jq '.crush_location'
```

### 🔍 CRUSH Map Modification

```bash
# Backup and restore
ceph osd getcrushmap -o crush-backup.bin
ceph osd setcrushmap -i crush-backup.bin

# Decompile/edit/recompile workflow
crushtool -d crush.bin -o crush.txt        # Edit crush.txt
crushtool -c crush.txt -o crush-new.bin
ceph osd setcrushmap -i crush-new.bin

# Add/move/remove buckets
ceph osd crush add-bucket <name> <type>
ceph osd crush move <name> <parent>=<value>
ceph osd crush remove <name>               # Must be empty first

# Manage OSD placement
ceph osd crush set <osd> <weight> <location>
ceph osd crush reweight <osd> <new-weight>
ceph osd crush remove <osd>                # Use with caution

# Device class management
ceph osd crush set-device-class <class> <osd>...
ceph osd crush rm-device-class <osd>...
```

### 🔍 Rule Management

```bash
# Create rules (CLI method)
ceph osd crush rule create-replicated <name> <root> <domain> [class]
ceph osd crush rule create-erasure <name> <profile>

# Apply rules to pools
ceph osd pool create <pool> <pg_num> <pgp_num> replicated <rule>
ceph osd pool set <pool> crush_rule <rule>

# Test rules without applying
crushtool -i crush.bin --test \
    --rule <id> \
    --num-rep <N> \
    --min-x 0 --max-x 100 \
    --show-mappings

# Delete unused rules
ceph osd crush rule rm <rule-name>         # Only if no pool uses it
```

### 🔍 Location Configuration

```bash
# Via ceph.conf (persistent)
[osd]
crush_location = root=default rack=rack-01 host=node-01

# Via command line (immediate)
ceph osd crush set osd.0 0.29299 root=default host=node-01

# Via hook script (dynamic)
[osd]
crush_location_hook = /usr/local/bin/crush-location-hook

# Verify assignments
ceph osd tree
ceph osd find osd.0 | jq '.crush_location'
```

### 🔍 Testing & Validation

```bash
# Test data placement
ceph osd map <pool> <object>
rados -p <pool> put test-obj /etc/hostname
ceph osd map <pool> test-obj

# Simulate failures
ceph osd out osd.0           # Mark OSD out (triggers rebalance)
ceph osd in osd.0            # Mark OSD in (triggers rebalance back)
ceph osd fail osd.0          # Mark OSD failed (for testing)
ceph osd ok osd.0            # Clear failure flag

# Monitor recovery
ceph -w                      # Real-time cluster events
ceph pg dump pgs_brief      # PG state summary
ceph osd perf               # OSD performance metrics
```

### 🔍 Maintenance & Optimization

```bash
# Enable automatic balancing
ceph mgr module enable balancer
ceph balancer mode upmap
ceph balancer on
ceph balancer status

# Tune recovery parameters
ceph config set osd osd_recovery_max_active 3
ceph config set osd osd_max_backfills 2
ceph config set osd osd_recovery_op_priority 3

# Update CRUSH tunables (test first)
ceph osd crush tunables
ceph osd crush tunables optimal

# Maintenance mode
ceph osd set noout           # Prevent OSDs from being marked out
ceph osd set norebalance     # Pause automatic rebalancing
# ... perform maintenance ...
ceph osd unset noout
ceph osd unset norebalance
```

---

## 🎯 Conclusion & Next Steps

### What You've Accomplished

✅ **Understood CRUSH Map fundamentals** - Not just theory, but how it actually decides where your data lives

✅ **Mastered the 5 core components** - Devices, Types, Buckets, Rules, Classes - and how they interact

✅ **Designed practical hierarchies** - From simple 3-node labs to enterprise multi-datacenter deployments

✅ **Created and managed CRUSH rules** - Using both CLI shortcuts and manual editing for complex scenarios

✅ **Configured and validated locations** - Ensuring your physical topology matches your fault tolerance goals

✅ **Tested failure scenarios** - Building confidence that your cluster survives real-world incidents

✅ **Established operational practices** - Monitoring, maintenance, and troubleshooting workflows

### Recommended Next Steps

```
🚀 Short-term (This Week):
• Run the monthly health check script on your lab
• Document your current CRUSH configuration in a wiki
• Practice one failure scenario test (OSD out/in)

🚀 Medium-term (This Month):
• Add a second device class (even if just labeling existing OSDs)
• Create a custom rule for a specific workload
• Set up Prometheus monitoring for CRUSH metrics

🚀 Long-term (This Quarter):
• Design a rack-aware hierarchy for future expansion
• Implement automated CRUSH validation in CI/CD pipeline
• Contribute improvements back to Ceph community docs
```

### Final Pro Tips

```
💡 Tip 1: Start simple, scale complexity
   Begin with host-level failure domain. Add rack/datacenter levels only when your infrastructure justifies it.

💡 Tip 2: Test before you deploy
   Every CRUSH change should be validated in a lab environment first. Use crushtool for syntax checking.

💡 Tip 3: Document everything
   Your future self (and your team) will thank you for clear documentation of CRUSH decisions and changes.

💡 Tip 4: Monitor rebalancing
   CRUSH changes trigger data movement. Watch ceph -w and adjust recovery parameters to avoid performance impact.

💡 Tip 5: Embrace automation
   Use the balancer module, automated validation scripts, and infrastructure-as-code for CRUSH management.
```

---

> 📚 **Official Resources**
> - Ceph CRUSH Documentation: https://docs.ceph.com/en/latest/rados/operations/crush-map/
> - CRUSH Algorithm Paper: https://ceph.com/assets/pdfs/weil-crush-sc06.pdf
> - Ceph Community Forum: https://discuss.ceph.com/
> - Proxmox VE + Ceph Guide: https://pve.proxmox.com/wiki/Ceph

> 🤝 **Need Help?**
> This guide covers practical implementation based on real-world scenarios. For environment-specific questions, consult your Ceph community or professional support channel.

---

*Document Version: 1.0 | Last Updated: March 2026 | Author: Ceph Practical Guide Series*

*This document is intended for educational and operational reference. Always test changes in a non-production environment before applying to production systems.*
