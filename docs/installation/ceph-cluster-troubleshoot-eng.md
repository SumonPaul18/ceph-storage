# Ceph Production Cluster Troubleshooting Guide
## Complete Step-by-Step Resolution Documentation

---

## 📋 Table of Contents

1. [Introduction & Overview](#introduction--overview)
2. [Initial Cluster Status Assessment](#initial-cluster-status-assessment)
3. [Identifying Health Warnings](#identifying-health-warnings)
4. [Resolving Zabbix Configuration Warning](#resolving-zabbix-configuration-warning)
5. [Fixing Ceph Daemon Crashes](#fixing-ceph-daemon-crashes)
6. [Resolving Cephadm Orchestration Issues](#resolving-cephadm-orchestration-issues)
7. [Manager Daemon Problems](#manager-daemon-problems)
8. [Production OSD Recovery - Flash1 Node](#production-osd-recovery---flash1-node)
9. [Understanding OSD Out vs Restart](#understanding-osd-out-vs-restart)
10. [Dashboard vs CLI Operations](#dashboard-vs-cli-operations)
11. [Complete Production-Safe Recovery Procedure](#complete-production-safe-recovery-procedure)
12. [Verification & Monitoring](#verification--monitoring)
13. [Preventive Measures & Best Practices](#preventive-measures--best-practices)
14. [Troubleshooting Common Issues](#troubleshooting-common-issues)
15. [Emergency Procedures](#emergency-procedures)

---

## Introduction & Overview

### Real-World Scenario

You are managing a **7-node production Ceph cluster** running version 18.2.0 (Reef) with the following configuration:

- **Total OSDs**: 146 (134 up, 12 down, 12 out)
- **Storage Capacity**: 1.2 PiB total, 79.5 TiB used (6.55%)
- **Monitors**: 3 nodes in quorum
- **Managers**: 2 nodes (1 active, 1 standby)
- **Orchestrator**: Cephadm
- **Problem Node**: flash1.meghnacloud.com (172.16.10.111)

### The Problem Story

One morning, you receive alerts that multiple OSDs on the flash1 node have stopped running. The cluster shows HEALTH_WARN status with various warnings including:
- No Zabbix server configured
- CephDaemonCrash alerts
- CephadmPaused and CephadmDaemonFailed alerts
- Multiple OSDs in stopped state

This is a **production environment** serving live customer data, so you need to:
1. Fix issues without data loss
2. Minimize service disruption
3. Follow safe, documented procedures
4. Understand what each command does

This guide documents the complete troubleshooting journey from initial diagnosis to final resolution.

---

## Initial Cluster Status Assessment

### Step 1.1: Connect to the Cluster

First, establish SSH connection to your Ceph admin node (usually the first monitor node):

```bash
ssh root@ceph1
```

**Why this matters**: You need administrative access to run Ceph commands. The ceph1 node typically has the cephadm tool and administrative credentials configured.

### Step 1.2: Check Overall Cluster Health

```bash
ceph -s
```

**What this shows**: 
- Cluster ID and health status
- Monitor quorum status
- Manager daemon status
- OSD count and state
- Pool and placement group (PG) status
- Storage capacity usage

**Expected output interpretation**:
- `HEALTH_OK` = Everything is fine
- `HEALTH_WARN` = Non-critical issues (investigate soon)
- `HEALTH_ERR` = Critical issues (act immediately)

### Step 1.3: Get Detailed Health Information

```bash
ceph health detail
```

**What this shows**: 
- Specific warnings or errors
- Which components are affected
- Timestamps of when issues started
- Recommended actions

**Real-world example from our case**:
```
HEALTH_WARN
- No Zabbix server configured
- 1 osds down
- 12 osds out
```

### Step 1.4: Check OSD Tree and Status

```bash
ceph osd tree
```

**What this shows**:
- Hierarchical view of OSDs by host
- Which OSDs are up/down, in/out
- Weight and utilization
- CRUSH hierarchy

**Why this is critical**: This helps you identify exactly which OSDs are problematic and on which hosts they reside.

### Step 1.5: Check Specific Host Status

```bash
ceph orch host ls
```

**What this shows**:
- All hosts managed by cephadm
- Host labels and status
- Last communication time
- Number of daemons per host

### Step 1.6: List All Running Daemons

```bash
ceph orch ps
```

**What this shows**:
- Every daemon in the cluster
- Daemon names and types
- Current status (running/stopped)
- Host where each daemon runs
- Version information
- Resource usage

**In our case**: This revealed that osd.108 through osd.117 on flash1 were in "stopped" state.

---

## Identifying Health Warnings

### Step 2.1: View Active Alerts

```bash
ceph alerts
```

**What this shows**:
- All active alert rules
- Which alerts are currently firing
- Alert severity (critical/warning)
- Duration of the alert
- Summary of the issue

### Step 2.2: Check Ceph Dashboard Alerts

Access the web interface:
```
https://<ceph-node>:8443/#/monitoring/alerts
```

**What you'll see**:
- Visual representation of alerts
- Ability to filter by severity
- Alert history and trends
- Direct links to affected components

### Step 2.3: Common Alert Types in Our Scenario

**CephHealthWarning**:
```
Summary: Ceph is in the WARNING state
Severity: warning
Description: The cluster state has been HEALTH_WARN for more than 15 minutes
```

**CephDaemonCrash**:
```
Summary: One or more Ceph daemons have crashed
Severity: critical
Action: Archive crash reports and investigate root cause
```

**CephadmPaused**:
```
Summary: Orchestration tasks via cephadm are PAUSED
Severity: warning
Impact: No automatic deployments or updates
```

**CephadmDaemonFailed**:
```
Summary: A ceph daemon managed by cephadm is down
Severity: critical
Action: Restart failed daemon or investigate
```

### Step 2.4: Check Cluster Logs

```bash
ceph log last 50
```

**What this shows**:
- Recent cluster events
- Error messages with timestamps
- Which components generated the logs
- Sequence of events leading to the issue

### Step 2.5: Check Specific Daemon Logs

```bash
ceph daemon mgr.ceph1 log latest
```

**What this shows**:
- Detailed logs from specific daemon
- Debug-level information
- Configuration issues
- Runtime errors

---

## Resolving Zabbix Configuration Warning

### Understanding the Problem

The cluster shows: `HEALTH_WARN No Zabbix server configured`

**What happened**: 
Someone previously configured Ceph Dashboard to send metrics to a Zabbix monitoring server, but either:
- The Zabbix server was removed
- The configuration was incomplete
- The Zabbix integration was abandoned

**Impact**: 
- No actual data loss risk
- Just a warning message
- Dashboard monitoring features partially disabled
- HEALTH_WARN status (prevents HEALTH_OK)

### Step 3.1: Verify Current Zabbix Configuration

```bash
ceph config dump | grep zabbix
```

**What this shows**:
- All configuration keys containing "zabbix"
- Current values (host, URL, credentials)
- Which manager holds the configuration

**Expected output**:
```
mgr mgr/dashboard/zabbix_host 192.168.x.x
mgr mgr/dashboard/zabbix_url http://192.168.x.x/zabbix
```

### Step 3.2: Remove Zabbix Configuration Keys

Remove each Zabbix-related configuration key:

```bash
ceph config rm mgr mgr/dashboard/zabbix_host
```
**What this does**: Removes the Zabbix server hostname/IP configuration

```bash
ceph config rm mgr mgr/dashboard/zabbix_url
```
**What this does**: Removes the Zabbix API URL configuration

```bash
ceph config rm mgr mgr/dashboard/zabbix_prefix
```
**What this does**: Removes any URL path prefix for Zabbix API

```bash
ceph config rm mgr mgr/dashboard/zabbix_api_path
```
**What this does**: Removes custom API path if configured

```bash
ceph config rm mgr mgr/dashboard/zabbix_username
```
**What this does**: Removes Zabbix authentication username

```bash
ceph config rm mgr mgr/dashboard/zabbix_password
```
**What this does**: Removes Zabbix authentication password

```bash
ceph config rm mgr mgr/dashboard/zabbix_verify_ssl
```
**What this does**: Removes SSL verification setting

### Step 3.3: Verify Configuration Removal

```bash
ceph config dump | grep -i zabbix
```

**Expected output**: Nothing (empty) - confirms all Zabbix config is removed

### Step 3.4: Restart Manager Daemon

The manager daemon needs to reload its configuration. You have two options:

**Option A - Restart specific manager**:

```bash
ceph orch restart mgr
```
**What this does**: Restarts all manager daemons gracefully using cephadm orchestrator

**Option B - Using systemctl** (if Option A fails):

```bash
systemctl restart ceph-mgr@ceph1
```
**What this does**: Directly restarts the ceph-mgr service on ceph1 host

**Why restart is needed**: The manager daemon caches configuration. Restarting forces it to reload and clear the Zabbix warning.

### Step 3.5: Alternative - Disable/Enable Dashboard Module

If restart doesn't clear the warning, reload the dashboard module:

```bash
ceph mgr module disable dashboard
```
**What this does**: Completely disables the dashboard module

```bash
sleep 3
```
**Why wait**: Allows the module to fully shut down

```bash
ceph mgr module enable dashboard
```
**What this does**: Re-enables the dashboard module with fresh configuration

### Step 3.6: Verify Resolution

```bash
ceph health
```

**Expected output**: `HEALTH_OK` (if no other issues exist)

```bash
ceph -s
```

**Expected output**: Health section should show `HEALTH_OK` without Zabbix warning

### Step 3.7: If Warning Persists - Check Manager Status

```bash
ceph mgr stat
```

**What to look for**:
- Active manager name
- Standby managers
- Whether managers are healthy

```bash
ceph mgr dump
```

**What this shows**:
- Detailed manager configuration
- Active and standby managers
- Enabled modules
- Service endpoints

---

## Fixing Ceph Daemon Crashes

### Understanding Daemon Crashes

When you see `CephDaemonCrash` alerts, it means:
- A Ceph daemon (mon, mgr, osd, etc.) crashed unexpectedly
- The crash was detected by the monitor
- A crash report was generated and stored
- The daemon may have automatically restarted

**Common causes**:
- Out of memory (OOM) killer
- Segmentation fault (bug)
- Disk I/O errors
- Network timeouts
- Configuration errors

### Step 4.1: List All Crash Reports

```bash
ceph crash ls
```

**What this shows**:
- List of all recorded crashes
- Crash IDs (unique identifiers)
- Timestamps of crashes
- Which daemon crashed

**Example output**:
```
2024-03-15T10:30:00.123456 mgr.ceph1
2024-03-15T09:15:00.654321 osd.45
```

### Step 4.2: Get Detailed Crash Information

```bash
ceph crash info <crash_id>
```

**Replace `<crash_id>`** with actual ID from the previous command

**What this shows**:
- Full crash report
- Stack trace (if available)
- Daemon metadata
- Host information
- Error messages
- Memory state at crash time

**Why this matters**: Helps identify root cause - was it a bug, resource exhaustion, or hardware issue?

### Step 4.3: Archive All Crash Reports

```bash
ceph crash archive-all
```

**What this does**:
- Moves all crash reports from "new" to "archived" state
- Clears the crash warnings from cluster health
- Keeps the data for future reference (doesn't delete)

**Why archive instead of delete**: 
- Archived crashes can be reviewed later
- Useful for trend analysis
- Helps identify recurring issues
- Maintains audit trail

### Step 4.4: Archive Specific Crash (Alternative)

If you want to archive crashes one by one:

```bash
ceph crash archive <crash_id>
```

**When to use this**: 
- When you want to review each crash individually
- When some crashes need immediate attention
- For selective cleanup

### Step 4.5: Configure Crash Pruning (Optional)

To automatically manage old crash reports:

```bash
ceph config set mgr mgr/crash/prune_interval 3600
```
**What this does**: Sets automatic cleanup to run every 3600 seconds (1 hour)

```bash
ceph config set mgr mgr/crash/warn_delay 86400
```
**What this does**: Only warn about crashes older than 86400 seconds (24 hours)

**Why configure this**: 
- Prevents accumulation of old crash data
- Reduces noise from transient issues
- Focuses on recent, relevant crashes

### Step 4.6: Verify Crash Resolution

```bash
ceph crash ls-new
```

**Expected output**: Should be empty (no new crashes)

```bash
ceph health detail
```

**Expected output**: No CephDaemonCrash warnings

---

## Resolving Cephadm Orchestration Issues

### Understanding Cephadm

**Cephadm** is Ceph's deployment and orchestration tool. It:
- Deploys Ceph daemons across hosts
- Manages daemon lifecycle (start/stop/restart)
- Handles upgrades automatically
- Ensures desired state matches actual state

When cephadm is paused or fails, no automatic deployments happen.

### Step 5.1: Check Cephadm Status

```bash
ceph orch status
```

**What this shows**:
- Whether orchestrator is active
- Available hosts
- Service status
- Last refresh time

### Step 5.2: Resume Paused Orchestration

If orchestration is paused:

```bash
ceph orch resume
```

**What this does**:
- Resumes all paused orchestration tasks
- Allows cephadm to continue managing daemons
- Re-enables automatic deployments

**Why it gets paused**:
- Manual intervention (admin paused it)
- Upgrade failure (auto-paused for safety)
- Error condition (paused to prevent damage)

### Step 5.3: Resume Specific Host (If Needed)

```bash
ceph orch host ls
```

First, identify which host might be problematic

```bash
ceph orch resume --host=flash1.meghnacloud.com
```

**What this does**: Resumes orchestration only on the specified host

**When to use**: 
- When only one host has issues
- When you want to isolate problems
- For gradual recovery

### Step 5.4: Check for Failed Daemons

```bash
ceph orch ps | grep -E "failed|error"
```

**What this shows**:
- Daemons in failed state
- Error messages
- Which hosts are affected

### Step 5.5: Restart Failed Daemons

For each failed daemon:

```bash
ceph orch restart <daemon_name>
```

**Example**:
```bash
ceph orch restart mgr.ceph1
ceph orch restart mon.ceph2
```

**What this does**:
- Stops the daemon gracefully
- Starts it again
- Cephadm ensures proper configuration
- Updates placement if needed

### Step 5.6: Check Upgrade Status

If you see `CephadmUpgradeFailed` alert:

```bash
ceph orch upgrade status
```

**What this shows**:
- Current upgrade state
- Target version
- Progress percentage
- Any errors encountered

### Step 5.7: Stop Failed Upgrade

If upgrade is stuck:

```bash
ceph orch upgrade stop
```

**What this does**:
- Halts the upgrade process
- Rolls back if in progress
- Returns to stable state

**Why stop an upgrade**:
- Compatibility issues detected
- Insufficient resources
- Configuration conflicts
- Daemon failures during upgrade

### Step 5.8: Check Cephadm Logs

```bash
journalctl -u cephadm -f --since "1 hour ago"
```

**What this shows**:
- Real-time cephadm logs
- Deployment actions
- Error messages
- Communication with hosts

**Key things to look for**:
- "Failed to" messages
- Timeout errors
- Authentication failures
- Network connectivity issues

### Step 5.9: Verify Host Connectivity

```bash
ceph orch host ls
```

Check the "Status" column - should show "available"

**If host shows "unavailable"**:
- SSH key authentication issue
- Network connectivity problem
- Host is down
- Cephadm agent not running

### Step 5.10: Re-add Problematic Host (Last Resort)

If a host is completely unavailable:

```bash
ceph orch host rm flash1.meghnacloud.com
```

**Warning**: Only do this if host is permanently lost

```bash
ceph orch host add flash1.meghnacloud.com <labels>
```

**What this does**:
- Removes old host configuration
- Re-adds with fresh configuration
- Re-establishes SSH connection

---

## Manager Daemon Problems

### Understanding Ceph Managers (MGR)

Ceph managers provide:
- Additional monitoring and metrics
- Dashboard web interface
- Orchestration capabilities
- External integration (Prometheus, Grafana, etc.)

**Typical setup**:
- 1 active manager
- 1+ standby managers
- Automatic failover if active fails

### Step 6.1: Check Manager Status

```bash
ceph mgr stat
```

**What this shows**:
- Active manager name
- Standby managers
- Leader election status
- Module status

**Example output**:
```
ceph1.gmpclf(active, since 25h), standbys: ceph2.btupjx
```

### Step 6.2: Get Detailed Manager Information

```bash
ceph mgr dump
```

**What this shows**:
- Active manager details
- Standby manager list
- Enabled modules
- Service endpoints
- Configuration

### Step 6.3: List Available Manager Modules

```bash
ceph mgr module ls
```

**What this shows**:
- All available modules
- Which are enabled
- Module descriptions

**Important modules**:
- `dashboard` - Web UI
- `prometheus` - Metrics export
- `cephadm` - Orchestration
- `crash` - Crash reporting

### Step 6.4: Check Manager Services

```bash
ceph mgr services
```

**What this shows**:
- Dashboard URL and status
- Prometheus endpoint
- Other module endpoints

**Example**:
```
{
    "dashboard": "https://ceph1:8443/",
    "prometheus": "http://ceph1:9283/"
}
```

### Step 6.5: Restart Problematic Manager

If a manager is unhealthy:

```bash
ceph orch ps | grep mgr
```

First identify the manager daemon name

```bash
ceph orch restart mgr.ceph1.gmpclf
```

**What this does**: Restarts the specific manager daemon

### Step 6.6: Force Manager Failover

If active manager is stuck:

```bash
ceph mgr fail ceph1.gmpclf
```

**What this does**:
- Forces the active manager to step down
- Triggers leader election
- Standby becomes active

**When to use**:
- Active manager unresponsive
- Dashboard not loading
- Module failures
- Memory leaks

### Step 6.7: Enable/Disable Manager Modules

If a module is causing issues:

```bash
ceph mgr module disable dashboard
```

**What this does**: Completely disables the dashboard module

```bash
sleep 3
```

**Wait for clean shutdown**

```bash
ceph mgr module enable dashboard
```

**What this does**: Re-enables dashboard with fresh state

### Step 6.8: Check Manager Resource Usage

```bash
ceph orch ps | grep mgr
```

Look at "Memory Usage" and "CPU Usage" columns

**Warning signs**:
- Memory > 2GB (possible leak)
- CPU constantly > 50% (high load)
- Frequent restarts (instability)

### Step 6.9: Manager Log Analysis

```bash
ceph daemon mgr.ceph1.gmpclf log latest
```

**What this shows**:
- Recent manager logs
- Error messages
- Module loading issues
- Configuration problems

### Step 6.10: Verify Manager Health

```bash
ceph -s | grep mgr
```

**Expected output**:
```
mgr: ceph1.gmpclf(active, since 25h), standbys: ceph2.btupjx
```

**Healthy indicators**:
- One active manager
- At least one standby
- "since" time is reasonable (not constantly restarting)

---

## Production OSD Recovery - Flash1 Node

### Understanding the Flash1 Node Problem

**Current situation**:
- Host: flash1.meghnacloud.com (172.16.10.111)
- Multiple OSDs stopped: osd.108, 109, 110, 111, 112, 113, 115, 116, 117
- Only osd.114 is running
- Production cluster serving live traffic
- 12 OSDs down/out affecting cluster health

**Why this is critical**:
- Production data at risk
- Reduced redundancy
- Performance degradation
- Potential data loss if more failures occur

### Step 7.1: SSH to Problem Node

```bash
ssh root@172.16.10.111
```

**Why**: Need direct access to diagnose and fix OSD issues

### Step 7.2: Check System Health

Before touching OSDs, verify the host is healthy:

```bash
dmesg | tail -100
```

**What to look for**:
- Disk I/O errors
- SCSI/SATA errors
- Out of memory (OOM) messages
- Filesystem errors
- Hardware failures

**Example concerning messages**:
```
sd 0:0:0:0: [sda] Sense Key : Medium Error
EXT4-fs error (device sda1): ext4_lookup
Out of memory: Kill process 1234 (ceph-osd)
```

### Step 7.3: Check Disk Status

```bash
lsblk
```

**What this shows**:
- All block devices
- Partition layout
- Mount points
- Device sizes

**Verify**:
- OSD devices are present
- No missing disks
- Correct device names

### Step 7.4: Check Disk Health with SMART

```bash
smartctl -a /dev/sdX
```

**Replace `/dev/sdX`** with actual OSD device

**What to check**:
- SMART overall health: "PASSED" or "FAILED"
- Reallocated sector count
- Current pending sector count
- Temperature
- Power-on hours

```bash
smartctl -H /dev/sdX
```

**Quick health check**: Should show "SMART Health Status: PASSED"

### Step 7.5: Check Filesystem Status

```bash
df -h
```

**What to verify**:
- `/var/lib/ceph` has free space
- No filesystems at 100%
- No read-only filesystems

### Step 7.6: Check Ceph OSD Directory

```bash
ls -la /var/lib/ceph/osd/
```

**What this shows**:
- All OSD directories
- Permissions
- Ownership (should be ceph:ceph)

### Step 7.7: Check Specific OSD Status

```bash
systemctl status ceph-osd@108
```

**What this shows**:
- Whether service is running
- Recent log entries
- Exit code if failed
- Memory/CPU usage

**Key indicators**:
- `Active: active (running)` = Good
- `Active: failed` = Problem
- `Active: activating` = Starting up

### Step 7.8: Check OSD Logs

```bash
tail -50 /var/log/ceph/ceph-osd@108.log
```

**What to look for**:
- Error messages
- Crash reports
- Disk I/O errors
- Configuration issues
- Network timeouts

**Common errors**:
```
failed to open device /dev/sdX: Device or resource busy
OSD::do_osd_ops: osd_op failed
filestore: error reading object
```

### Step 7.9: Check Journal Logs

```bash
journalctl -u ceph-osd@108 --since "1 hour ago" --no-pager
```

**What this shows**:
- Systemd service logs
- Startup sequence
- Failure reasons
- Dependency issues

### Step 7.10: Check Ceph Volume Configuration

```bash
ceph-volume lvm list
```

**What this shows**:
- LVM volumes for OSDs
- Device mappings
- Volume groups
- Logical volumes

**Verify**:
- OSD volumes exist
- Correct device paths
- No corrupted volumes

### Step 7.11: Check Network Connectivity

```bash
ping ceph1
ping ceph2
ping ceph3
```

**Why**: OSDs need network connectivity to monitors and other OSDs

```bash
ceph mon dump
```

**What this shows**:
- Monitor addresses
- Quorum status
- Network endpoints

### Step 7.12: Check System Resources

```bash
free -h
```

**What to verify**:
- Available memory
- Swap usage
- No OOM conditions

```bash
top -bn1 | head -20
```

**What to check**:
- CPU load
- Memory usage
- Running processes
- Ceph OSD processes

### Step 7.13: Check Ceph Configuration

```bash
ceph config dump
```

**What this shows**:
- All cluster configuration
- Custom settings
- Overrides

### Step 7.14: Verify OSD Permissions

```bash
ls -la /var/lib/ceph/osd/ceph-108/
```

**Expected ownership**: `ceph:ceph`

If wrong:
```bash
chown -R ceph:ceph /var/lib/ceph/osd/ceph-108/
chmod 755 /var/lib/ceph/osd/ceph-108/
```

### Step 7.15: Check for Running OSD Processes

```bash
ps aux | grep ceph-osd | grep 108
```

**What this shows**:
- Whether OSD process is running
- Process ID (PID)
- Resource usage
- Command line arguments

---

## Understanding OSD Out vs Restart

### Critical Concept: OSD Out vs Direct Restart

This is one of the **most important concepts** in production Ceph management.

### What Happens When You Restart Without "Out"

**Scenario**: You directly restart an OSD

```bash
systemctl restart ceph-osd@108
```

**Timeline of events**:

```
T+0s:  OSD.108 stops (restart command)
T+1s:  Monitor detects OSD down
T+2s:  PGs (Placement Groups) using OSD.108 become "degraded"
T+3s:  Client I/O to affected PGs redirected to other replicas
T+5s:  Clients may see "slow request" warnings
T+10s: OSD.108 starts back up
T+15s: OSD.108 rejoining cluster
T+20s: OSD.108 marked "up" but "behind" (needs sync)
T+25s: Recovery starts - OSD.108 catching up
T+30s: PGs become "active+clean" again
```

**Problems with this approach**:

1. **Sudden Degradation**: PGs instantly become degraded
2. **Client Impact**: I/O timeouts and retries
3. **Recovery Storm**: Sudden spike in network/disk I/O
4. **PG Flapping**: Rapid state changes stress monitors
5. **Cascading Risk**: If another OSD fails during this window, data loss possible

### What Happens When You Use "Out" First

**Scenario**: Graceful removal and restart

```bash
ceph osd out 108
sleep 120
systemctl restart ceph-osd@108
ceph osd in 108
```

**Timeline of events**:

```
T+0s:  "ceph osd out 108" command issued
T+1s:  Cluster marks OSD.108 as "out" (intentional)
T+2s:  Backfill starts - data copying to other OSDs
T+30s: Backfill in progress (controlled, throttled)
T+60s: Backfill continues (PGs still healthy)
T+90s: Backfill completes or mostly complete
T+120s: OSD.108 stopped (restart command)
T+121s: PGs already have data elsewhere (no degradation)
T+125s: Clients unaffected (data available from other OSDs)
T+150s: OSD.108 starts back up
T+180s: "ceph osd in 108" - OSD rejoins
T+181s: Rebalance starts (slow, controlled)
T+300s: OSD.108 fully integrated
```

**Benefits of this approach**:

1. **Controlled Backfill**: Data migration happens gradually
2. **No Degradation**: PGs stay healthy throughout
3. **Client Transparency**: No I/O impact
4. **Monitor Stability**: No PG flapping
5. **Safety Margin**: Time to abort if issues arise

### Visual Comparison

**Direct Restart (Risky)**:
```
[OSD UP] ──(sudden DOWN)──> [DEGRADED] ──(fast UP)──> [ACTIVE]
                ⚠️                        ⚠️
         Clients see errors        Rush to recover
         PG state unstable         Network spike
```

**Graceful Out/In (Safe)**:
```
[OSD UP] ──(graceful OUT)──> [BACKFILLING] ──(restart)──> [IN] ──> [REBALANCE]
                ✅                      ✅                    ✅
         Controlled migration    No client impact    Gradual integration
         PGs stay healthy        Safe operation      Stable cluster
```

### When Can You Skip "Out"?

**Safe to skip "out" when**:

1. **Test/Dev Environment**: No production data
2. **Single OSD Test**: Only 1 OSD, replication >= 3
3. **Maintenance Window**: No client I/O expected
4. **Low Cluster Load**: Plenty of spare capacity
5. **Quick Restart**: Expected downtime < 30 seconds

**Never skip "out" when**:

1. **Production Cluster**: Live customer data ✅ YOUR CASE
2. **Multiple OSDs**: Restarting several at once
3. **Replication = 2**: Low redundancy
4. **High Cluster Load**: Already stressed
5. **Existing Warnings**: Cluster already unhealthy

### The Golden Rule

**Production Environment = Always use "out" first**

The extra 2-3 minutes of waiting prevents:
- Client outages
- Data loss risk
- Emergency rollbacks
- Pager duty at 3 AM

---

## Dashboard vs CLI Operations

### Ceph Dashboard Overview

**What is Dashboard**:
- Web-based graphical interface
- Part of Ceph Manager module
- Provides visual cluster monitoring
- Allows some administrative actions

**Access**:
```
https://<ceph-node>:8443/
```

### Dashboard Capabilities

**What you CAN do in Dashboard**:

1. **View Cluster Status**:
   - Health indicators
   - Capacity graphs
   - Performance charts
   - Alert lists

2. **Browse Components**:
   - Hosts list
   - OSD status
   - Pool configuration
   - PG state

3. **Basic Operations**:
   - Restart individual daemons
   - Create pools
   - Manage users
   - View logs (limited)

4. **Monitoring**:
   - Real-time metrics
   - Historical graphs
   - Alert notifications
   - Performance counters

### Dashboard Limitations

**What you CANNOT do (or shouldn't)**:

1. **No Graceful OSD Out/In**:
   - Dashboard restart = direct restart
   - No automatic "ceph osd out"
   - No controlled backfill
   - ⚠️ Risky for production

2. **Limited Batch Operations**:
   - Can't easily restart multiple OSDs sequentially
   - No built-in waiting/delays
   - No automated verification

3. **No Advanced Troubleshooting**:
   - Can't run arbitrary ceph commands
   - Limited log access
   - No debug mode
   - Can't modify configuration deeply

4. **Delayed Updates**:
   - UI refreshes every 5-30 seconds
   - Not real-time
   - May show stale data

5. **No Scripting**:
   - Can't automate repetitive tasks
   - Manual clicking required
   - Error-prone for bulk operations

### CLI (Command Line Interface) Advantages

**Why CLI is better for production**:

1. **Full Control**:
   - Every ceph command available
   - Precise timing
   - Exact sequencing

2. **Automation**:
   - Can write scripts
   - Batch operations
   - Loops and conditions

3. **Immediate Feedback**:
   - Real-time output
   - Exit codes
   - Detailed errors

4. **Logging**:
   - Command history
   - Can redirect output
   - Audit trail

5. **Safety Features**:
   - Can verify each step
   - Can abort mid-process
   - Can rollback easily

### When to Use Dashboard

**Dashboard is good for**:

1. **Monitoring**:
   - Daily health checks
   - Capacity planning
   - Performance trends
   - Alert review

2. **Quick Lookups**:
   - "Which OSD is on which host?"
   - "What's the pool size?"
   - "How much capacity used?"

3. **Single Operations**:
   - Restart one daemon
   - Create one pool
   - Add one user

4. **Demonstrations**:
   - Show management cluster status
   - Training new admins
   - Executive dashboards

### When to Use CLI

**CLI is essential for**:

1. **Production Maintenance**:
   - OSD recovery ✅ YOUR CASE
   - Cluster upgrades
   - Configuration changes
   - Troubleshooting

2. **Bulk Operations**:
   - Restart 10+ OSDs
   - Migrate data
   - Rebalance cluster

3. **Emergency Response**:
   - Failure recovery
   - Data rescue
   - Performance crisis

4. **Automation**:
   - Scheduled maintenance
   - Health checks
   - Reporting

### Hybrid Approach (Best Practice)

**Use both together**:

```bash
# 1. Use Dashboard for initial assessment
# Open: https://ceph:8443/#/dashboard
# Check: Health, OSD status, Alerts

# 2. Use CLI for detailed diagnosis
ssh root@ceph1
ceph -s
ceph osd tree

# 3. Use CLI for safe operations
ceph osd out 108
sleep 120
systemctl restart ceph-osd@108

# 4. Use Dashboard for visual verification
# Refresh dashboard to see OSD status change
# Watch graphs for recovery progress

# 5. Use CLI for final verification
ceph -s
ceph osd stat
```

### Dashboard Restart Procedure (If You Must)

**If you choose to use Dashboard** (not recommended for production):

1. Navigate to Cluster → Services
2. Find the OSD (e.g., osd.108)
3. Click the Actions menu (⋮)
4. Select "Restart"
5. Confirm the action
6. Wait for status to change
7. Refresh page to see updates

**Problems with this**:
- No "out" step
- No controlled backfill
- No verification between steps
- Can't easily monitor progress

### CLI Restart Procedure (Recommended)

**Step-by-step CLI approach**:

```bash
# Step 1: Check current status
ceph osd tree | grep 108

# Step 2: Mark OSD out (graceful)
ceph osd out 108

# Step 3: Wait for backfill
sleep 120

# Step 4: Verify backfill started
ceph -s | grep backfill

# Step 5: Restart OSD
systemctl restart ceph-osd@108

# Step 6: Wait for OSD to start
sleep 30

# Step 7: Verify OSD is up
ceph osd tree | grep 108

# Step 8: Mark OSD in
ceph osd in 108

# Step 9: Monitor rebalance
ceph -w

# Step 10: Final verification
ceph -s
```

---

## Complete Production-Safe Recovery Procedure

### Pre-Recovery Checklist

Before starting any recovery work:

**1. Verify Backup Status**:
```bash
# Check if backups are recent
ls -lth /backup/ceph/ | head -5

# Verify backup integrity
rbd export pool/image /tmp/test-export
```

**2. Notify Stakeholders**:
- Send maintenance notification
- Inform about potential brief I/O latency
- Set expected duration (2-4 hours)
- Provide emergency contact

**3. Schedule Maintenance Window**:
- Choose low-traffic period (e.g., 2 AM - 6 AM)
- Ensure team availability
- Prepare rollback plan

**4. Document Current State**:
```bash
# Save cluster state
ceph -s > /tmp/pre-recovery-status.txt
ceph osd tree > /tmp/pre-recovery-osd-tree.txt
ceph pg dump > /tmp/pre-recovery-pg-dump.txt
ceph df > /tmp/pre-recovery-df.txt

# Timestamp
date >> /tmp/pre-recovery-status.txt
```

### Recovery Procedure for Flash1 Node

**Total estimated time**: 3-4 hours for all OSDs

#### Phase 1: Initial Assessment (15 minutes)

```bash
# Connect to cluster admin node
ssh root@ceph1

# Check overall cluster health
ceph -s
ceph health detail

# Identify all stopped OSDs on flash1
ceph orch ps | grep flash1 | grep stopped

# Count affected OSDs
ceph osd tree | grep flash1 | grep -c "down\|out"

# Check cluster capacity and redundancy
ceph df
ceph osd stat
```

**Decision point**: 
- If cluster HEALTH_ERR → Stop and investigate
- If < 50% OSDs available → Consider emergency mode
- If adequate redundancy → Proceed with caution

#### Phase 2: Test Recovery - First OSD (30 minutes)

Start with one OSD to validate the procedure:

```bash
# Choose first OSD (e.g., osd.108)
OSD_ID=108

# Verify current state
ceph osd tree | grep $OSD_ID

# Mark OSD out
ceph osd out $OSD_ID

# Wait for backfill to start
echo "Waiting 120 seconds for backfill..."
sleep 120

# Check backfill progress
ceph -s | grep -E "backfill|recovery"

# Verify PG state
ceph pg stat

# SSH to flash1 node
ssh root@172.16.10.111

# Check OSD service status
systemctl status ceph-osd@$OSD_ID

# Check OSD logs for errors
tail -50 /var/log/ceph/ceph-osd@$OSD_ID.log

# Restart OSD
systemctl restart ceph-osd@$OSD_ID

# Wait for OSD to start
echo "Waiting 30 seconds for OSD to start..."
sleep 30

# Verify OSD is running
systemctl status ceph-osd@$OSD_ID --no-pager

# Check if OSD is up in cluster
ceph osd dump | grep "osd.$OSD_ID " | grep "up"

# If OSD is up, mark it in
ceph osd in $OSD_ID

# Wait for rebalance
echo "Waiting 120 seconds for rebalance..."
sleep 120

# Verify recovery
ceph -s
ceph osd tree | grep $OSD_ID
```

**Verification checkpoint**:
- ✅ OSD is "up" and "in"
- ✅ Cluster HEALTH_OK or HEALTH_WARN (not ERR)
- ✅ PGs are "active+clean" or recovering
- ✅ No new errors in logs

**If successful**: Proceed with remaining OSDs  
**If failed**: Investigate and fix before continuing

#### Phase 3: Batch Recovery - Remaining OSDs (2-3 hours)

Process OSDs one at a time with adequate waiting:

```bash
# Define OSD list (adjust based on your situation)
OSD_LIST="109 110 111 112 113 115 116 117"

# Process each OSD
for OSD_ID in $OSD_LIST; do
    echo "========================================="
    echo "Starting OSD $OSD_ID recovery at $(date)"
    echo "========================================="
    
    # Step 1: Mark OSD out
    echo "[1/7] Marking OSD $OSD_ID as OUT..."
    ceph osd out $OSD_ID
    
    # Step 2: Wait for backfill
    echo "[2/7] Waiting 120 seconds for backfill..."
    sleep 120
    
    # Step 3: Check cluster health
    echo "[3/7] Checking cluster health..."
    ceph -s | grep -E "HEALTH|backfill|recovery"
    
    # Step 4: Verify PG state
    echo "[4/7] Verifying PG state..."
    ceph pg stat
    
    # Step 5: Restart OSD
    echo "[5/7] Restarting OSD $OSD_ID..."
    systemctl restart ceph-osd@$OSD_ID
    
    # Step 6: Wait for OSD startup
    echo "[6/7] Waiting 30 seconds for OSD startup..."
    sleep 30
    
    # Step 7: Verify OSD is running
    echo "[7/7] Verifying OSD status..."
    OSD_STATUS=$(ceph osd dump | grep "osd.$OSD_ID " | grep -c "up" || echo "0")
    
    if [ "$OSD_STATUS" -gt 0 ]; then
        echo "✓ OSD $OSD_ID is UP"
        ceph osd in $OSD_ID
        echo "✓ OSD $OSD_ID marked IN"
    else
        echo "✗ OSD $OSD_ID failed to start"
        echo "Investigating..."
        systemctl status ceph-osd@$OSD_ID
        tail -20 /var/log/ceph/ceph-osd@$OSD_ID.log
        echo "Skipping to next OSD"
        continue
    fi
    
    # Wait for rebalance before next OSD
    echo "Waiting 120 seconds for rebalance..."
    sleep 120
    
    # Final check for this OSD
    echo "Final status for OSD $OSD_ID:"
    ceph osd tree | grep $OSD_ID
    echo ""
    
    # Safety check - ensure cluster is healthy
    CLUSTER_HEALTH=$(ceph health | grep -c "HEALTH_OK\|HEALTH_WARN" || echo "0")
    if [ "$CLUSTER_HEALTH" -eq 0 ]; then
        echo "⚠️ WARNING: Cluster health degraded!"
        ceph -s
        echo "Pausing recovery - manual intervention required"
        break
    fi
done

echo "========================================="
echo "Batch recovery completed at $(date)"
echo "========================================="
```

#### Phase 4: Post-Recovery Verification (15 minutes)

After all OSDs processed:

```bash
# 1. Check overall cluster health
ceph -s

# 2. Verify all OSDs on flash1 are up
ceph osd tree | grep flash1

# 3. Check PG state
ceph pg stat

# 4. Verify no stuck PGs
ceph pg dump | grep -E "down\|incomplete\|stuck"

# 5. Check data distribution
ceph df

# 6. Verify OSD performance
ceph osd perf

# 7. Check for any remaining warnings
ceph health detail

# 8. Review recent logs
ceph log last 20

# 9. Test client I/O (if possible)
rbd create test-image --size 1G --pool rbd
rbd map test-image
mkfs.ext4 /dev/rbd0
mount /dev/rbd0 /mnt
dd if=/dev/zero of=/mnt/testfile bs=1M count=100
umount /mnt
rbd unmap /dev/rbd0
rbd rm test-image

# 10. Save final state
ceph -s > /tmp/post-recovery-status.txt
ceph osd tree > /tmp/post-recovery-osd-tree.txt
date >> /tmp/post-recovery-status.txt
```

#### Phase 5: Documentation and Handover

```bash
# Create recovery report
cat > /tmp/recovery-report.txt << 'EOF'
CEPH CLUSTER RECOVERY REPORT
============================
Date: $(date)
Node: flash1.meghnacloud.com
OSDs Recovered: 108, 109, 110, 111, 112, 113, 115, 116, 117

Pre-Recovery Status:
- Cluster Health: HEALTH_WARN
- OSDs Down: 11
- OSDs Out: 12

Recovery Actions:
- Method: Graceful out/in with backfill
- Duration: ~4 hours
- Issues Encountered: None/Specify

Post-Recovery Status:
$(ceph -s)

Recommendations:
- Monitor cluster for 24 hours
- Check for recurring issues
- Review hardware health
- Update runbooks

Engineer: [Your Name]
EOF

cat /tmp/recovery-report.txt
```

---

## Verification & Monitoring

### Immediate Verification Commands

Run these immediately after recovery:

```bash
# 1. Cluster health summary
ceph -s

# 2. Detailed health status
ceph health detail

# 3. OSD statistics
ceph osd stat

# 4. OSD tree view
ceph osd tree

# 5. Placement group status
ceph pg stat

# 6. Pool usage
ceph df

# 7. Monitor status
ceph mon stat

# 8. Manager status
ceph mgr stat
```

### Ongoing Monitoring (First 24 Hours)

**Set up continuous monitoring**:

```bash
# Watch cluster status in real-time
ceph -w

# Or watch specific metrics
watch -n 5 'ceph -s | grep -E "HEALTH|osds|pgs"'

# Monitor recovery progress
watch -n 10 'ceph pg stat'

# Check for slow requests
watch -n 30 'ceph health detail | grep -i slow'
```

### Performance Verification

```bash
# OSD performance
ceph osd perf

# PG distribution
ceph pg dump | grep -E "pgs_sum"

# I/O statistics
ceph osd io-profile ls

# Latency metrics
ceph osd histogram dump
```

### Log Monitoring

```bash
# Watch cluster logs
tail -f /var/log/ceph/ceph.log

# Watch specific OSD logs
tail -f /var/log/ceph/ceph-osd@108.log

# System logs
journalctl -f -u ceph-osd@108

# Filter for errors
tail -f /var/log/ceph/ceph.log | grep -i error
```

### Dashboard Monitoring

Access dashboard for visual monitoring:
```
https://172.16.10.111:8443/
```

**Check these sections**:
- **Dashboard**: Overall health and capacity
- **Cluster → OSDs**: Individual OSD status
- **Cluster → Hosts**: flash1 node health
- **Observability → Logs**: Recent events
- **Observability → Alerts**: Active alerts

### Alerting Setup

```bash
# Check active alerts
ceph alerts

# Verify alerting configuration
ceph config dump | grep alert

# Test alert manager
curl http://localhost:9093/api/v1/alerts
```

---

## Preventive Measures & Best Practices

### Regular Health Checks

**Daily checks**:
```bash
# Morning health check
ceph -s
ceph health detail

# Check for new crashes
ceph crash ls-new

# Verify OSD status
ceph osd stat
```

**Weekly checks**:
```bash
# Full cluster audit
ceph report

# Capacity planning
ceph df

# Performance baseline
ceph osd perf
```

### Automated Monitoring

**Set up cron job for health checks**:

```bash
# Edit crontab
crontab -e

# Add health check every 5 minutes
*/5 * * * * /usr/bin/ceph -s >> /var/log/ceph-health.log 2>&1

# Daily summary at 8 AM
0 8 * * * /usr/bin/ceph -s > /var/log/ceph-daily-$(date +\%Y\%m\%d).log
```

### Capacity Management

```bash
# Monitor capacity usage
ceph df

# Set capacity warnings
ceph config set mon mon_osd_full_ratio 0.95
ceph config set mon mon_osd_backfillfull_ratio 0.90
ceph config set mon mon_osd_nearfull_ratio 0.85
```

### Hardware Monitoring

```bash
# Check disk SMART status
smartctl -H /dev/sdX

# Monitor disk temperature
smartctl -A /dev/sdX | grep Temperature

# Check for disk errors
dmesg | grep -i "error\|fail" | tail -20
```

### Configuration Backup

```bash
# Backup cluster configuration
ceph config dump > /backup/ceph-config-$(date +%Y%m%d).txt

# Backup CRUSH map
ceph osd getcrushmap -o /backup/crushmap-$(date +%Y%m%d).bin
ceph osd crush dump > /backup/crushmap-$(date +%Y%m%d).json

# Backup monitor map
ceph mon dump > /backup/monmap-$(date +%Y%m%d).txt
```

### Documentation Updates

**Maintain runbooks for**:
- OSD failure recovery
- Monitor failure recovery
- Network partition handling
- Full cluster restore
- Upgrade procedures

### Team Training

**Ensure team knows**:
- How to access cluster
- Where logs are located
- Emergency contact procedures
- Escalation paths
- Backup/restore procedures

---

## Troubleshooting Common Issues

### Issue 1: OSD Won't Start

**Symptoms**:
```bash
systemctl status ceph-osd@108
# Shows: failed or activating (repeatedly)
```

**Diagnosis**:
```bash
# Check logs
journalctl -u ceph-osd@108 --since "1 hour ago"

# Check disk
lsblk | grep 108
smartctl -H /dev/sdX

# Check permissions
ls -la /var/lib/ceph/osd/ceph-108/
```

**Solutions**:
```bash
# Fix permissions
chown -R ceph:ceph /var/lib/ceph/osd/ceph-108/

# Activate OSD manually
ceph-volume activate --osd-id 108

# Restart service
systemctl daemon-reload
systemctl restart ceph-osd@108
```

### Issue 2: PGs Stuck in Degraded State

**Symptoms**:
```bash
ceph pg stat
# Shows: X pgs degraded
```

**Diagnosis**:
```bash
# Find stuck PGs
ceph pg dump | grep degraded

# Check which OSDs are missing
ceph pg map <pg_id>
```

**Solutions**:
```bash
# Force PG recovery
ceph pg repair <pg_id>

# If OSD is permanently lost
ceph osd down <osd_id>
ceph osd out <osd_id>
ceph osd purge <osd_id> --yes-i-really-mean-it
```

### Issue 3: Slow Recovery/Backfill

**Symptoms**:
```bash
ceph -s
# Shows: recovering for hours
```

**Diagnosis**:
```bash
# Check recovery rate
ceph -w | grep recovery

# Check network/disk I/O
iostat -x 5
iftop
```

**Solutions**:
```bash
# Increase recovery limits (temporary)
ceph config set global osd_recovery_max_active 10
ceph config set global osd_recovery_max_single_start 5

# Monitor impact
ceph -w

# Reset to defaults after recovery
ceph config rm global osd_recovery_max_active
ceph config rm global osd_recovery_max_single_start
```

### Issue 4: Cluster Full or Near Full

**Symptoms**:
```bash
ceph df
# Shows: >85% usage
```

**Solutions**:
```bash
# Identify large pools
ceph df detail

# Delete unnecessary data
rbd rm pool/image

# Add more OSDs
ceph orch apply osd --all-available-devices

# Or expand existing OSDs (if possible)
```

### Issue 5: Monitor Quorum Lost

**Symptoms**:
```bash
ceph mon stat
# Shows: no quorum
```

**Emergency Recovery**:
```bash
# On one monitor
systemctl stop ceph-mon@ceph1

# Reset monitor store (LAST RESORT)
rm -rf /var/lib/ceph/mon/ceph-ceph1/*
ceph-mon -i ceph1 --mkfs --monmap /tmp/monmap

# Restart
systemctl start ceph-mon@ceph1
```

---

## Emergency Procedures

### Emergency Contact List

**Maintain updated contacts**:
- Primary Ceph Admin: [Name] [Phone]
- Secondary Admin: [Name] [Phone]
- Infrastructure Lead: [Name] [Phone]
- Management: [Name] [Phone]
- Vendor Support: [Company] [Phone/Ticket]

### Emergency Decision Tree

**Scenario: Multiple OSD Failures**

```
1. Assess Impact
   ├─ ceph -s
   ├─ ceph osd stat
   └─ ceph health detail

2. Determine Severity
   ├─ HEALTH_OK/WARN → Normal procedure
   ├─ HEALTH_ERR → Emergency mode
   └─ Data inaccessible → Critical incident

3. Immediate Actions
   ├─ Notify stakeholders
   ├─ Stop writes if possible
   ├─ Prevent further failures
   └─ Document everything

4. Recovery Strategy
   ├─ If OSDs recoverable → Graceful restart
   ├─ If hardware failure → Replace hardware
   └─ If data loss → Restore from backup

5. Post-Incident
   ├─ Root cause analysis
   ├─ Update procedures
   ├─ Team debrief
   └─ Management report
```

### Data Loss Emergency

**If data is at risk**:

```bash
# 1. IMMEDIATELY stop all writes
# If possible, make pools read-only
ceph osd set norecover
ceph osd set nobackfill
ceph osd set noscrub

# 2. Assess damage
ceph -s
ceph pg dump

# 3. Preserve evidence
ceph report > /tmp/emergency-report.txt
cp -r /var/log/ceph /tmp/ceph-logs-emergency/

# 4. Contact backup team
# Prepare for restore
rbd export pool/image /backup/emergency-export

# 5. DO NOT proceed without expert consultation
# Call senior engineer or vendor support
```

### Rollback Procedure

**If recovery makes things worse**:

```bash
# 1. Stop current operation
# Press Ctrl+C if running script
# Or kill process

# 2. Assess new situation
ceph -s
ceph health detail

# 3. If OSD caused issue
ceph osd out <problematic_osd>
ceph osd down <problematic_osd>

# 4. Restore previous state
# Use saved configurations
ceph config set <key> <previous_value>

# 5. Document what went wrong
# For post-mortem analysis
```

### Communication Template

**Emergency notification email**:

```
Subject: [URGENT] Ceph Cluster Incident - flash1 Node

Team,

INCIDENT SUMMARY:
- Time: [Timestamp]
- Affected System: Ceph Cluster - flash1.meghnacloud.com
- Impact: [Data availability/Performance degradation]
- Severity: [Critical/High/Medium]

CURRENT STATUS:
- [Brief description of current state]
- [Number of OSDs affected]
- [Client impact]

ACTIONS TAKEN:
1. [Action 1]
2. [Action 2]
3. [Action 3]

NEXT STEPS:
1. [Planned action 1]
2. [Planned action 2]

ESTIMATED RESOLUTION: [Time estimate]

POINT OF CONTACT: [Name] [Phone]

Updates will be provided every [30 minutes/1 hour].

Regards,
[Your Name]
```

---

## Appendix A: Command Reference Quick Guide

### Essential Commands

```bash
# Cluster Status
ceph -s
ceph status
ceph health
ceph health detail

# OSD Commands
ceph osd stat
ceph osd tree
ceph osd dump
ceph osd out <id>
ceph osd in <id>
ceph osd down <id>
ceph osd up <id>

# PG Commands
ceph pg stat
ceph pg dump
ceph pg ls
ceph pg repair <pg_id>

# Pool Commands
ceph osd pool ls
ceph osd pool ls detail
ceph df
ceph df detail

# Monitor Commands
ceph mon dump
ceph mon stat
ceph quorum_status

# Manager Commands
ceph mgr stat
ceph mgr dump
ceph mgr module ls

# Orchestrator Commands
ceph orch status
ceph orch ls
ceph orch ps
ceph orch host ls

# Log Commands
ceph log last 50
ceph log get <log_channel>

# Configuration
ceph config dump
ceph config get <who> <key>
ceph config set <who> <key> <value>
ceph config rm <who> <key>
```

### Service Management

```bash
# Systemctl commands
systemctl status ceph-osd@<id>
systemctl start ceph-osd@<id>
systemctl stop ceph-osd@<id>
systemctl restart ceph-osd@<id>

systemctl status ceph-mon@<hostname>
systemctl status ceph-mgr@<hostname>

# Journal logs
journalctl -u ceph-osd@<id> -f
journalctl -u ceph-mon@<hostname> --since "1 hour ago"
```

### File Locations

```bash
# Log files
/var/log/ceph/ceph.log
/var/log/ceph/ceph-osd@<id>.log
/var/log/ceph/ceph-mon@<hostname>.log
/var/log/ceph/ceph-mgr@<hostname>.log

# Data directories
/var/lib/ceph/osd/ceph-<id>/
/var/lib/ceph/mon/ceph-<hostname>/
/var/lib/ceph/mgr/ceph-<hostname>/

# Configuration
/etc/ceph/ceph.conf
```

---

## Appendix B: Glossary

**OSD (Object Storage Daemon)**: Stores data, handles data replication, recovery, etc.

**MON (Monitor)**: Maintains cluster state, configuration, and authentication

**MGR (Manager)**: Provides additional monitoring, metrics, and external interfaces

**PG (Placement Group)**: Logical grouping of objects for placement and replication

**CRUSH**: Algorithm that determines data placement across OSDs

**Backfill**: Process of copying data to new OSDs or rebalancing

**Recovery**: Process of restoring redundancy after OSD failure

**Quorum**: Minimum number of monitors needed for cluster operation

**Cephadm**: Ceph's deployment and orchestration tool

**Replication Factor**: Number of copies of each object (typically 2 or 3)

---

## Appendix C: Useful Scripts (Reference Only)

**Note**: These are for reference. Always understand commands before running.

### Quick Health Check

```bash
#!/bin/bash
echo "=== CEPH CLUSTER HEALTH CHECK ==="
echo "Date: $(date)"
echo ""
echo "Cluster Status:"
ceph -s
echo ""
echo "OSD Status:"
ceph osd stat
echo ""
echo "PG Status:"
ceph pg stat
echo ""
echo "Capacity:"
ceph df
echo ""
echo "=== END CHECK ==="
```

### OSD Status Report

```bash
#!/bin/bash
echo "=== OSD STATUS REPORT ==="
ceph osd tree
echo ""
echo "Down OSDs:"
ceph osd dump | grep down
echo ""
echo "Out OSDs:"
ceph osd dump | grep out
```

---

## Document Information

**Document Version**: 1.0  
**Created**: Based on production incident - March 2026  
**Author**: Ceph Infrastructure Team  
**Last Updated**: Current session  
**Review Cycle**: Quarterly  
**Classification**: Internal Technical Documentation

**Related Documentation**:
- Ceph Official Documentation: https://docs.ceph.com/
- Cephadm Guide: https://docs.ceph.com/en/latest/cephadm/
- Operations Guide: https://docs.ceph.com/en/latest/rados/operations/
- Troubleshooting: https://docs.ceph.com/en/latest/rados/troubleshooting/

**Feedback**:  
If you find errors or have suggestions for improvement, please contact the infrastructure team.

---

## Final Notes

This guide was created from a real production incident involving:
- 7-node Ceph cluster
- 146 OSDs total
- 12 OSDs stopped on flash1 node
- Multiple health warnings (Zabbix, crashes, cephadm)
- Production environment with live customer data

**Key Lessons Learned**:

1. **Always use "ceph osd out" before restarting** - Prevents data loss and client impact

2. **One OSD at a time** - Don't rush; stability over speed

3. **Monitor continuously** - Watch cluster health throughout recovery

4. **Document everything** - Future you will thank present you

5. **Test procedures** - Practice in non-production first

6. **Have rollback plan** - Always know how to undo changes

7. **Communicate clearly** - Keep stakeholders informed

8. **Know when to stop** - If things get worse, pause and reassess

**Remember**: In production environments, cautious and methodical approaches always beat fast and risky ones. Your primary goal is data integrity and service availability, not speed of resolution.

---

**END OF DOCUMENT**