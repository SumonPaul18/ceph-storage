# Ceph Cluster Security: A Practical Guide to Data at Rest & In-Transit Encryption

**Author:** Sumon (Senior DevOps & Infrastructure Engineer)  
**Date:** April 14, 2026  
**Ceph Version:** Squid (19.2.3) / Reef (18.x)  
**Environment:** 3-Node Cluster (cephadm managed)

---

## Table of Contents

1. [Introduction & Security Architecture](#1-introduction--security-architecture)
2. [Prerequisites & Environment Setup](#2-prerequisites--environment-setup)
3. [Part I: Data in Transit Encryption (msgr2/TLS)](#part-i-data-in-transit-encryption-msgr2tls)
4. [Part II: Data at Rest Encryption (LUKS/dm-crypt)](#part-ii-data-at-rest-encryption-luksdm-crypt)
5. [Verification & Proof of Concept](#5-verification--proof-of-concept)
6. [Operations, Management & Maintenance](#6-operations-management--maintenance)
7. [Troubleshooting & Rollback Procedures](#7-troubleshooting--rollback-procedures)

---

## 1. Introduction & Security Architecture

In modern IT infrastructure, securing a Ceph cluster is not just about access control; it is about protecting data throughout its lifecycle. This guide focuses on two critical pillars of Ceph security:

### 1.1 Data in Transit Encryption
When data moves between clients, monitors (MONs), managers (MGRs), and Object Storage Daemons (OSDs), it travels over the network. Without encryption, this data is vulnerable to packet sniffing and man-in-the-middle attacks.
*   **Protocol:** Ceph uses `msgr2` (Message Protocol version 2).
*   **Mechanism:** TLS 1.2/1.3 for encryption and authentication.
*   **Port:** Default port `3300` for secure communication.

### 1.2 Data at Rest Encryption
When data is stored on physical disks (HDD/SSD/NVMe), it must be encrypted to protect against physical theft or unauthorized disk removal.
*   **Mechanism:** Linux Kernel `dm-crypt` with `LUKS` (Linux Unified Key Setup).
*   **Key Management:** Ceph automatically manages encryption keys within the Monitor database (MON DB).
*   **Impact:** Minimal performance overhead if CPU supports AES-NI instructions.

> **Real-World Context:** Imagine your Ceph cluster stores patient records for a hospital. If a hacker intercepts network traffic, they could steal data (In-Transit risk). If a technician steals a hard drive from a decommissioned server, they could read the data directly (At-Rest risk). This guide eliminates both risks.

---

## 2. Prerequisites & Environment Setup

Before implementing encryption, ensure your environment meets the following criteria.

### 2.1 Hardware Requirements
*   **CPU:** Intel or AMD processors with **AES-NI** support.
    *   *Check:* `grep aes /proc/cpuinfo`
    *   *Why:* Hardware acceleration reduces the performance penalty of encryption by ~90%.
*   **Memory:** Sufficient RAM for OSDs (minimum 4GB per OSD recommended).
*   **Storage:** Raw block devices (e.g., `/dev/sdb`, `/dev/vdb`) for OSDs. No existing partitions or LVM volumes.

### 2.2 Software Requirements
*   **OS:** Ubuntu 22.04/24.04 LTS, RHEL 8/9, or CentOS Stream.
*   **Ceph Version:** Squid (19.x) or Reef (18.x). Older versions (Octopus/Nautilus) have limited `msgr2` support.
    *   *Check:* `ceph --version`
*   **Deployment Tool:** `cephadm` (recommended for modern clusters).

### 2.3 Network Configuration
*   **Public Network:** Client-facing network (e.g., `192.168.68.0/24`).
*   **Cluster Network:** Internal replication network (optional but recommended for isolation).
*   **Firewall:** Ensure ports `3300` (msgr2), `6789` (mon legacy), and `6800-7300` (osd legacy) are open during migration. After migration, only `3300` is strictly required for secure comms.

### 2.4 Backup Strategy
*   **MON Database:** Encrypting OSDs relies on keys stored in the MON DB. **Backup your MON DB** before starting.
    ```bash
    ceph mon dump > /tmp/monmap_backup
    # For cephadm:
    cephadm shell -- ceph config-key get mon/keyring > /tmp/mon_keyring_backup
    ```

---

## Part I: Data in Transit Encryption (msgr2/TLS)

This section guides you through enabling TLS encryption for all network communications in your Ceph cluster.

### Phase 1: Assessment (Current State)

Before making changes, verify the current state.

**Step 1: Check Current Protocol Bindings**
```bash
ceph config dump | grep ms_bind
```
*Expected Output (Unencrypted):*
```text
mon advanced ms_bind_msgr1 true
mon advanced ms_bind_msgr2 false
osd advanced ms_bind_msgr1 true
osd advanced ms_bind_msgr2 false
```

**Step 2: Check Active Ports**
```bash
ss -tlnp | grep ceph-osd
```
*Expected Output:* Ports in the range `6800-68xx`. This indicates legacy `msgr1` usage.

**Step 3: Check Global Security Modes**
```bash
ceph config dump | grep ms_.*_mode
```
*Expected Output:* `crc` or `legacy`. This means no TLS enforcement.

### Phase 2: Implementation (Enabling Encryption)

We will enable `msgr2` globally and enforce secure modes.

**Step 1: Enable msgr2 Binding**
Run these commands on any node with Ceph CLI access (usually the first monitor node).

```bash
# Enable msgr2 for Monitors and OSDs
ceph config set mon ms_bind_msgr2 true
ceph config set osd ms_bind_msgr2 true

# Disable legacy msgr1 (Optional but Recommended for Security)
# Note: Do this only after verifying msgr2 works. For now, keep both enabled during transition.
ceph config set mon ms_bind_msgr1 false
ceph config set osd ms_bind_msgr1 false
```

**Step 2: Enforce Secure Communication Modes**
This ensures that even if a client tries to connect insecurely, the cluster rejects it.

```bash
# Set global security modes to 'secure' (TLS)
ceph config set global ms_client_mode secure
ceph config set global ms_cluster_mode secure
ceph config set global ms_service_mode secure
```

**Step 3: Update Monitor Addresses (Critical for cephadm)**
In `cephadm` deployments, monitors need to explicitly advertise their v2 (msgr2) addresses.

```bash
# Get FSID
FSID=$(ceph fsid)

# Update Mon addresses for each node
# Replace IPs with your actual Monitor IPs
ceph mon set-addrs mon.ceph1 [v2:192.168.68.248:3300/0]
ceph mon set-addrs mon.ceph2 [v2:192.168.68.249:3300/0]
ceph mon set-addrs mon.ceph3 [v2:192.168.68.250:3300/0]
```

**Step 4: Rolling Restart of Services**
Changes require service restarts. Use a rolling restart to maintain availability.

*Option A: Using Ceph Orchestrator (Recommended)*
```bash
ceph orch restart mon
ceph orch restart mgr
ceph orch restart osd
```

*Option B: Manual Systemctl Restart (If Orchestrator fails)*
On **each node** (ceph1, ceph2, ceph3):
```bash
# Restart Monitors
systemctl restart ceph-${FSID}@mon.*.service

# Restart Managers
systemctl restart ceph-${FSID}@mgr.*.service

# Restart OSDs (One by one or all)
for svc in $(systemctl list-units --type=service --state=running | grep 'ceph.*osd\.' | awk '{print $1}'); do
    systemctl restart $svc
done
```

> **Note:** Wait for the cluster to return to `HEALTH_OK` between restarting nodes if doing a manual rolling restart.

### Phase 3: Verification (Proof of Encryption)

**Step 1: Verify Port Changes**
```bash
ss -tlnp | grep ceph-mon
ss -tlnp | grep ceph-osd
```
*Success Criteria:*
*   Monitors should listen on **Port 3300**.
*   OSDs may still show `68xx` ports due to backward compatibility listeners, but the primary communication will be via msgr2.
*   Check `ceph mon dump`: You should see `v2:<IP>:3300/0` entries.

**Step 2: Verify Configuration**
```bash
ceph config dump | grep ms_
```
*Success Criteria:*
*   `ms_bind_msgr1`: `false`
*   `ms_bind_msgr2`: `true`
*   `ms_client_mode`: `secure`
*   `ms_cluster_mode`: `secure`

**Step 3: Network Packet Analysis (The Ultimate Proof)**
Use `tcpdump` to confirm traffic is encrypted.

1.  Start capture on a Monitor node:
    ```bash
    sudo tcpdump -i any port 3300 -w encrypted_traffic.pcap
    ```
2.  Generate traffic from a client:
    ```bash
    rados bench -p <pool_name> 10 write --no-cleanup
    ```
3.  Stop tcpdump (`Ctrl+C`) and analyze in Wireshark.
    *   *Result:* You will see **TLSv1.2/1.3** packets. The payload will be unreadable garbage, proving encryption.

---

## Part II: Data at Rest Encryption (LUKS/dm-crypt)

This section guides you through encrypting existing OSDs. **Warning:** This process involves destroying and recreating OSDs. Data safety depends on proper replication.

### Phase 1: Pre-Requisites & Safety Checks

**Step 1: Ensure Cluster Health**
```bash
ceph -s
```
*   Cluster must be `HEALTH_OK`.
*   Replication size (`size`) should be at least 2 (preferably 3).
*   Minimum size (`min_size`) should be 1 or 2.

**Step 2: Identify OSD Devices**
```bash
ceph-volume lvm list
lsblk -f
```
Note down which device (e.g., `/dev/vdb`) corresponds to which OSD ID (e.g., `osd.0`).

### Phase 2: Migration Strategy (One OSD at a Time)

We will migrate OSDs one by one to avoid data loss.

#### Step 1: Out the OSD
Remove the OSD from the CRUSH map so data migrates to other OSDs.

```bash
ceph osd out osd.0
```

#### Step 2: Wait for Rebalancing
Monitor the cluster until data migration is complete.

```bash
watch ceph -s
```
*   Wait until `pgs` show `active+clean`.
*   Ensure no `degraded` or `undersized` PGs remain for this OSD.

#### Step 3: Stop and Destroy the OSD
Once data is safe on other nodes, destroy the local OSD instance.

```bash
# Stop the OSD service
systemctl stop ceph-${FSID}@osd.0.service

# Zap the device (WARNING: This deletes all data on the device)
ceph-volume lvm zap --destroy /dev/vdb
```
*Replace `/dev/vdb` with the actual device path for `osd.0`.*

#### Step 4: Create Encrypted OSD
Recreate the OSD with the `--dmcrypt` flag.

```bash
ceph-volume lvm create --data /dev/vdb --dmcrypt
```
*   Ceph will automatically:
    1.  Create a LUKS container on `/dev/vdb`.
    2.  Generate an encryption key.
    3.  Store the key in the Monitor DB.
    4.  Format the inner volume with BlueStore/XFS.
    5.  Start the OSD.

#### Step 5: Verify Encryption
```bash
# Check OSD Metadata
ceph osd metadata 0 | grep dmcrypt
# Output should be: "dmcrypt": "true"

# Check Block Device
lsblk -f /dev/vdb
# Output should show:
# vdb       crypto_LUKS ...
# └─vdb_crypt LVM2_member ...
```

#### Step 6: Repeat for All OSDs
Repeat Steps 1-5 for `osd.1`, `osd.2`, ..., `osd.N`.

> **Performance Tip:** If you have many OSDs, you can parallelize this process across different nodes, but never migrate multiple OSDs on the same node simultaneously unless you have sufficient bandwidth and CPU.

### Phase 3: Verification (Proof of At-Rest Encryption)

**Step 1: Strings Test (The Practical Proof)**
Try to read raw data from the disk.

1.  Write a test file to a RBD image or CephFS mount.
    ```bash
    echo "SECRET_DATA_123" > /mnt/ceph/test.txt
    sync
    ```
2.  On the OSD node, scan the raw device:
    ```bash
    sudo strings /dev/vdb | grep "SECRET_DATA_123"
    ```
3.  *Result:* **No output.** If encryption is working, the raw disk contains only encrypted binary data, and `strings` will find nothing readable.

**Step 2: LUKS Header Check**
```bash
sudo cryptsetup luksDump /dev/vdb
```
*   *Result:* Should display LUKS header information, confirming the device is a LUKS container.

---

## 5. Verification & Proof of Concept Summary

| Feature | Verification Method | Expected Result (Encrypted) |
| :--- | :--- | :--- |
| **In-Transit** | `ss -tlnp \| grep ceph` | Monitors on Port **3300**. |
| **In-Transit** | `ceph config dump \| grep ms_` | `ms_bind_msgr1: false`, `ms_mode: secure`. |
| **In-Transit** | Wireshark/Tcpdump | **TLSv1.2/1.3** packets, unreadable payload. |
| **At-Rest** | `ceph osd metadata <id>` | `"dmcrypt": "true"`. |
| **At-Rest** | `lsblk -f` | FSTYPE shows **crypto_LUKS**. |
| **At-Rest** | `strings /dev/sdX` | **No readable text** found on raw disk. |

---

## 6. Operations, Management & Maintenance

### 6.1 Key Management
*   **Automatic:** Ceph manages LUKS keys automatically. They are stored in the MON DB.
*   **Backup:** Regularly backup your MON DB. If you lose the MON DB, you lose the keys, and the data becomes unrecoverable.
    ```bash
    # Example backup command for cephadm
    cephadm shell -- ceph config-key get mon/keyring > /backup/mon_keyring_$(date +%F).key
    ```

### 6.2 Performance Monitoring
Encryption adds CPU overhead. Monitor CPU usage, specifically `cryptd` or `kworker` processes.

```bash
top -p $(pgrep -d ',' ceph-osd)
```
*   If CPU usage spikes significantly, check if AES-NI is enabled in BIOS.
*   Use `ceph perf dump` to check OSD latency.

### 6.3 Adding New OSDs
When adding new disks to an encrypted cluster, always use the `--dmcrypt` flag.

```bash
ceph-volume lvm create --data /dev/new_disk --dmcrypt
```

### 6.4 Client Configuration
Ensure clients are using a Ceph version that supports `msgr2`.
*   **Linux Kernel:** Kernel 5.11+ has better msgr2 support for RBD.
*   **Ceph.conf:** Add `ms_secure_mode = true` to client configs if forced.

---

## 7. Troubleshooting & Rollback Procedures

### 7.1 Common Issues

**Issue 1: OSDs Not Switching to Port 3300**
*   *Cause:* Configuration not loaded or legacy mode stuck.
*   *Fix:*
    ```bash
    ceph tell osd.* injectargs '--ms-bind-msgr2=true --ms-bind-msgr1=false'
    ceph tell mon.* injectargs '--ms-bind-msgr2=true --ms-bind-msgr1=false'
    systemctl restart ceph-osd.target
    ```

**Issue 2: Cluster Stuck in HEALTH_WARN after OSD Migration**
*   *Cause:* Slow rebalancing or insufficient space.
*   *Fix:* Check `ceph health detail`. Increase `osd_recovery_max_active` if needed, or wait.

**Issue 3: Client Connection Errors**
*   *Cause:* Old client library not supporting msgr2.
*   *Fix:* Upgrade Ceph client packages (`ceph-common`, `librbd1`).

### 7.2 Rollback: Disabling Encryption

If you need to revert to non-encrypted mode (e.g., for performance testing):

**Step 1: Re-enable Legacy Protocol**
```bash
ceph config set mon ms_bind_msgr1 true
ceph config set osd ms_bind_msgr1 true
ceph config set global ms_client_mode crc
ceph config set global ms_cluster_mode crc
ceph config set global ms_service_mode crc
```

**Step 2: Restart Services**
```bash
ceph orch restart mon
ceph orch restart osd
```

**Step 3: Verify**
Check `ss -tlnp` for ports `6789` and `68xx`. Traffic will now be unencrypted (CRC integrity only).

> **Security Warning:** Rolling back disables TLS encryption. Do not do this in production environments handling sensitive data.

---

## Conclusion

By following this guide, you have successfully secured your Ceph cluster against both network interception and physical disk theft.
1.  **Data in Transit** is now protected by TLS 1.2/1.3 via `msgr2`.
2.  **Data at Rest** is now protected by LUKS/dm-crypt.

Regularly monitor your cluster's health and performance, and keep your MON backups safe. Security is not a one-time task but a continuous process of maintenance and vigilance.

---
*End of Document*