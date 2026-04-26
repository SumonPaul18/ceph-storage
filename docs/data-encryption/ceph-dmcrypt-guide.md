# ceph-osd-encryption-guide.md

---

# Ceph OSD Data-at-Rest Encryption: A Practical Operational Guide

## Table of Contents

1. [Introduction & Scenario Overview](#1-introduction--scenario-overview)
2. [Prerequisites & Environment Validation](#2-prerequisites--environment-validation)
3. [Enabling Cluster-Wide Encryption Policies](#3-enabling-cluster-wide-encryption-policies)
4. [Operational Workflow: Removing & Replacing OSDs](#4-operational-workflow-removing--replacing-osds)
5. [Disk Sanitization & Zapping Procedures](#5-disk-sanitization--zapping-procedures)
6. [Deploying Encrypted OSDs](#6-deploying-encrypted-osds)
7. [Verification & Troubleshooting](#7-verification--troubleshooting)
8. [Maintenance & Best Practices](#8-maintenance--best-practices)

---

## 1. Introduction & Scenario Overview

### The Real-World Context
In modern cloud infrastructure, particularly for Cloud Service Providers (CSPs) like Meghna Cloud, **Data Sovereignty** and **Security Compliance** are non-negotiable. One of the critical requirements is **Data-at-Rest Encryption**. This ensures that even if a physical hard drive is stolen from a data center rack, the data remains unreadable without the encryption keys.

This guide documents a real-life operational scenario where a DevOps Engineer needs to:
1.  Enable encryption for all new OSDs (Object Storage Daemons) in a Ceph cluster managed by `cephadm`.
2.  Safely remove an existing unencrypted OSD (`osd.11`) from a specific disk (`/dev/vdd`).
3.  Securely wipe (zap) the disk to remove old partition tables and LVM metadata.
4.  Deploy a new, fully encrypted OSD on the same disk.
5.  Verify that the encryption layer (LUKS/dm-crypt) is active and functioning.

### Why This Matters
*   **Compliance:** Meets international standards (GDPR, ISO 27001) for data protection.
*   **Security:** Protects against physical theft of storage media.
*   **Automation:** Uses `cephadm` and YAML specs for reproducible, infrastructure-as-code (IaC) deployments.

> **Note:** This guide focuses on **practical execution**. Theory is kept to a minimum. All commands are tested in a production-like environment using Ceph Quincy/Reef with `cephadm`.

---

## 2. Prerequisites & Environment Validation

Before making any changes, you must validate the current state of your cluster. Never assume the configuration; always verify.

### 2.1 Check Current Encryption Status

First, determine if encryption is already enabled globally or on specific OSDs.

**Command:**
```bash
ceph config get global osd_encrypt
```
**Explanation:**
*   `ceph config get`: Retrieves runtime configuration.
*   `global`: Scope of the setting.
*   `osd_encrypt`: The key that controls whether new OSDs are encrypted.
*   **Expected Output:** `false` (if not yet enabled) or `true`.

**Command:**
```bash
grep -i "osd_encrypt" /etc/ceph/ceph.conf
```
**Explanation:**
*   Checks the static configuration file. If `cephadm` manages your cluster, this file might be minimal, as most configs are stored in the MON database. However, it’s good practice to check for legacy manual overrides.

### 2.2 Inspect Existing OSDs for Encryption

You need to know which OSDs are currently encrypted and which are not.

**Command:**
```bash
ceph osd dump | grep -i encrypt
```
**Explanation:**
*   `ceph osd dump`: Dumps the full OSD map.
*   `grep -i encrypt`: Filters for lines containing "encrypt" (case-insensitive).
*   **Interpretation:** Look for flags like `dmcrypt` or `encrypted`. If empty, no OSDs are encrypted.

**Command:**
```bash
ceph-volume lvm list
```
**Explanation:**
*   Lists all Logical Volumes managed by Ceph.
*   **Key Detail:** Look for the `"encrypted": true` field in the JSON output for each OSD. This is the most reliable way to check per-OSD encryption status.

### 2.3 Verify Disk Layout & Device Mapper

Encryption in Ceph uses Linux’s `dm-crypt` and `LUKS` (Linux Unified Key Setup). You must ensure the kernel modules are loaded and the device mapper is functional.

**Command:**
```bash
lsblk -f | grep crypto_LUKS
```
**Explanation:**
*   `lsblk -f`: Lists block devices with filesystem types.
*   `grep crypto_LUKS`: Filters for LUKS-encrypted partitions.
*   **Expected Output:** If you have existing encrypted OSDs, you will see lines with `crypto_LUKS`. If empty, no disks are currently encrypted at the block level.

**Command:**
```bash
dmsetup ls --tree
```
**Explanation:**
*   Shows the Device Mapper tree.
*   **Why it matters:** Encrypted OSDs appear as nested devices (e.g., `ceph-osd-block-xxx` inside `luks-xxx`). This helps troubleshoot if an encrypted device fails to mount.

### 2.4 Check Available Devices for Orchestrator

Since we are using `cephadm`, the orchestrator must recognize the disks.

**Command:**
```bash
ceph orch device ls --refresh
```
**Explanation:**
*   `orch device ls`: Lists devices known to the orchestrator.
*   `--refresh`: Forces a rescan of all hosts. **Crucial** before adding/removing disks to ensure the latest state.
*   **Look for:** Devices marked as `Available`. If a disk is marked `Rejected` or `In Use`, you cannot deploy a new OSD on it until it is freed.

---

## 3. Enabling Cluster-Wide Encryption Policies

To ensure all **new** OSDs are encrypted automatically, you must set the global configuration flags.

### 3.1 Enable OSD Encryption

**Command:**
```bash
ceph config set global osd_encrypt true
```
**Explanation:**
*   `config set`: Updates the configuration database.
*   `global`: Applies to all daemons.
*   `osd_encrypt true`: Tells Ceph to encrypt data at rest for new OSDs.

### 3.2 Enable DM-Crypt Backend

**Command:**
```bash
ceph config set global osd_dm_crypt true
```
**Explanation:**
*   `osd_dm_crypt true`: Explicitly enables the `dm-crypt` backend. While `osd_encrypt` might default to this, setting it explicitly ensures clarity and compatibility with older versions.

### 3.3 Verify Configuration Changes

**Command:**
```bash
ceph config get global osd_encrypt
ceph config get global osd_dm_crypt
```
**Explanation:**
*   Confirm both return `true`. If they do, any new OSD created via `ceph orch` or `ceph-volume` will be encrypted.

---

## 4. Operational Workflow: Removing & Replacing OSDs

In this scenario, we are replacing `osd.11` on `/dev/vdd`. This is a delicate operation. We must ensure data is rebalanced before removing the OSD to prevent data loss.

### 4.1 Step 1: Mark OSD as "Out"

**Command:**
```bash
ceph osd out osd.11
```
**Explanation:**
*   `osd out`: Tells the CRUSH algorithm to stop writing new data to this OSD and start migrating existing data to other OSDs.
*   **Impact:** The cluster enters a `degraded` state temporarily. Performance may dip slightly during rebalancing.

### 4.2 Step 2: Monitor Rebalancing

**Command:**
```bash
watch ceph -s
```
**Explanation:**
*   `watch`: Refreshes the output every 2 seconds.
*   `ceph -s`: Shows cluster health.
*   **Wait For:** The `pgmap` section to show `active+clean` for all Placement Groups (PGs). Do **not** proceed until rebalancing is complete. If you remove the OSD too early, you risk data loss if replication factor is not met.

### 4.3 Step 3: Stop the OSD Daemon

Once data is safe, stop the running service.

**Command:**
```bash
ceph orch daemon stop osd.11
```
**Explanation:**
*   `orch daemon stop`: Gracefully stops the container/process via the orchestrator.
*   **Alternative:** If the orchestrator is unresponsive, use:
    ```bash
    podman stop ceph-osd.11
    ```
    Or directly via systemd (if not containerized):
    ```bash
    systemctl stop ceph-<FSID>@osd.11.service
    ```

### 4.4 Step 4: Remove OSD from Cluster Metadata

Now, remove the OSD’s identity from the cluster.

**Command:**
```bash
ceph osd crush remove osd.11
```
**Explanation:**
*   Removes the OSD from the CRUSH map. It will no longer be considered for data placement.

**Command:**
```bash
ceph auth del osd.11
```
**Explanation:**
*   Deletes the authentication key for `osd.11`. This is a security best practice to revoke access.

**Command:**
```bash
ceph osd rm osd.11
```
**Explanation:**
*   Permanently removes the OSD ID from the cluster map.
*   **Note:** After this, `osd.11` no longer exists in the cluster’s logic.

---

## 5. Disk Sanitization & Zapping Procedures

The physical disk `/dev/vdd` still contains old partition tables, LVM metadata, and potentially old encryption headers. These must be wiped clean before reuse.

### 5.1 Identify Associated Devices

**Command:**
```bash
lsblk | grep -B 5 osd.11
```
**Explanation:**
*   Helps identify which physical partitions belonged to `osd.11`.
*   **Goal:** Confirm that `/dev/vdd` is the correct disk to wipe.

### 5.2 Unmount & Deactivate Logical Volumes

If the OSD was using LVM, you must deactivate the volumes first.

**Command:**
```bash
lvchange -an /dev/ceph-<VG-ID>/osd-block-<OSD-ID>
```
**Explanation:**
*   `lvchange -an`: Deactivates the Logical Volume (Active = No).
*   Replace `<VG-ID>` and `<OSD-ID>` with actual values from `ceph-volume lvm list`.

### 5.3 Close LUKS Containers

If the disk was previously encrypted, close the mapper.

**Command:**
```bash
cryptsetup luksClose /dev/mapper/ceph-<UUID>-osd-block-<UUID>
```
**Explanation:**
*   `luksClose`: Detaches the decrypted mapping from the kernel.
*   **Error Handling:** If it says "device is busy," use `fuser -v /dev/mapper/...` to find holding processes and kill them.

### 5.4 Zap the Disk

Ceph provides a powerful tool to wipe disks completely.

**Command:**
```bash
ceph-volume lvm zap /dev/vdd --destroy
```
**Explanation:**
*   `lvm zap`: Destroys LVM structures, partition tables, and filesystem signatures.
*   `--destroy`: Confirms you want to erase data.

**Force Zap (If Standard Zap Fails):**
Sometimes, residual locks or kernel references prevent zapping.

**Command:**
```bash
ceph-volume lvm zap /dev/vdd --destroy --yes-i-really-really-mean-it
```
**Explanation:**
*   `--yes-i-really-really-mean-it`: Bypasses all safety checks. Use with extreme caution. This will irreversibly destroy all data on `/dev/vdd`.

### 5.5 Orchestrator-Level Zap

Since we are using `cephadm`, we should also tell the orchestrator to forget the device’s previous state.

**Command:**
```bash
ceph orch device zap <hostname> /dev/vdd --force
```
**Example:**
```bash
ceph orch device zap ceph1 /dev/vdd --force
```
**Explanation:**
*   `orch device zap`: Instructs the orchestrator to wipe the device on the remote host.
*   `<hostname>`: The name of the host where the disk resides (e.g., `ceph1`).
*   `--force`: Overrides rejection reasons (e.g., if the disk was marked "non-empty").

### 5.6 Final Cleanups (Manual Wipe)

To be absolutely sure, run standard Linux wiping tools.

**Command:**
```bash
wipefs -a /dev/vdd
```
**Explanation:**
*   `wipefs -a`: Erases all filesystem magic strings (signatures) from the disk.

**Command:**
```bash
pvremove -ff /dev/vdd
```
**Explanation:**
*   `pvremove -ff`: Forcefully removes LVM Physical Volume labels.

**Command:**
```bash
dd if=/dev/zero of=/dev/vdd bs=1M count=100
```
**Explanation:**
*   Overwrites the first 100MB with zeros. This destroys partition tables and boot sectors.
*   **Note:** Not strictly necessary if `zap` worked, but adds an extra layer of certainty.

---

## 6. Deploying Encrypted OSDs

Now that the disk is clean and the cluster is configured for encryption, we can deploy the new OSD.

### 6.1 Method A: Automatic Deployment (Recommended)

If you have `osd_encrypt` set to `true` and the device is available, `cephadm` can auto-deploy.

**Command:**
```bash
ceph orch apply osd --all-available-devices
```
**Explanation:**
*   `orch apply osd`: Deploys OSDs.
*   `--all-available-devices`: Uses all disks marked as `Available` by the orchestrator.
*   **Result:** Since `osd_encrypt` is `true`, the new OSD on `/dev/vdd` will be automatically encrypted with LUKS.

### 6.2 Method B: Manual Deployment with ceph-volume

For more control, or if orchestrator auto-deployment is disabled, use `ceph-volume`.

**Command:**
```bash
ceph-volume lvm create --data /dev/vdd --dmcrypt
```
**Explanation:**
*   `lvm create`: Creates a new OSD using LVM.
*   `--data /dev/vdd`: Specifies the raw disk.
*   `--dmcrypt`: **Critical Flag.** Enables encryption for this specific OSD.
*   **Process:**
    1.  Creates a LUKS header on `/dev/vdd`.
    2.  Opens the LUKS container.
    3.  Creates LVM PV/VG/LV inside the encrypted container.
    4.  Formats with XFS.
    5.  Starts the OSD daemon.

### 6.3 Method C: Using a Service Specification (YAML)

For structured, repeatable deployments (Infrastructure as Code).

**Create File:** `osd-encrypted-spec.yaml`

```yaml
service_type: osd
service_id: encrypted_osds_vdd
placement:
  hosts:
    - ceph1
spec:
  data_devices:
    paths:
      - /dev/vdd
  encrypted: true
```

**Apply Spec:**
```bash
ceph orch apply -i osd-encrypted-spec.yaml
```
**Explanation:**
*   Defines a specific service for `/dev/vdd` on `ceph1`.
*   `encrypted: true`: Ensures encryption is enforced for this spec.

---

## 7. Verification & Troubleshooting

After deployment, you must verify that encryption is active and the OSD is healthy.

### 7.1 Verify Block Device Encryption

**Command:**
```bash
lsblk -f /dev/vdd
```
**Expected Output:**
```text
NAME            FSTYPE      FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
vdd             crypto_LUKS 2           1234-5678-...                                
└─luks-1234...  LVM2_member LVM2  001   abcd-efgh-...                                
  ├─ceph-...    xfs               5678-9012-...                          /var/lib/ceph/osd/ceph-12
```
**Explanation:**
*   The top-level type must be `crypto_LUKS`.
*   The underlying filesystem (XFS) should be mounted inside the LUKS container.

### 7.2 Verify LUKS Header

**Command:**
```bash
sudo cryptsetup isLuks /dev/vdd && echo "LUKS Encrypted" || echo "NOT Encrypted"
```
**Expected Output:**
```text
LUKS Encrypted
```

**Command:**
```bash
sudo cryptsetup luksDump /dev/vdd
```
**Explanation:**
*   Displays LUKS version, cipher (e.g., `aes-xts-plain64`), and key slots.
*   **Security Note:** Ensure LUKS Version 2 is used if possible (more secure).

### 7.3 Verify Ceph OSD Metadata

**Command:**
```bash
ceph osd metadata <new_osd_id> | grep dmcrypt
```
**Expected Output:**
```json
"dmcrypt": "true",
```
**Explanation:**
*   Confirms that Ceph internally recognizes this OSD as encrypted.

### 7.4 Verify Device Mapper Tree

**Command:**
```bash
dmsetup ls --tree
```
**Expected Output:**
```text
ceph-<UUID>-osd-block-<UUID> (253:5)
 └─luks-<UUID> (253:4)
    └─vdd (252:16)
```
**Explanation:**
*   Shows the stacking: `OSD Block` -> `LUKS Container` -> `Physical Disk`.
*   If this tree is broken, the OSD will fail to start.

### 7.5 Check Cluster Health

**Command:**
```bash
ceph -s
```
**Expected Output:**
*   `health: HEALTH_OK`
*   `osd: XX osds: XX up, XX in`
*   Ensure the new OSD is `up` and `in`.

**Command:**
```bash
ceph osd tree
```
**Explanation:**
*   Verify the new OSD appears in the correct host bucket.

---

## 8. Maintenance & Best Practices

### 8.1 Key Management

Ceph stores LUKS keys in the Ceph Monitor’s `config-key` store or an external Vault.

**Check Key Storage:**
```bash
ceph config-key ls | grep dm-crypt
```
**Explanation:**
*   Lists all stored encryption keys.
*   **Best Practice:** Regularly backup these keys. If you lose the Ceph Monitors and the keys, you **cannot** recover the data on encrypted OSDs.

**Export Keys (Backup):**
```bash
ceph config-key get dm-crypt/osd/<id>/luks > /backup/osd_<id>_key.luks
```
**Warning:** Store these backups in a secure, offline location.

### 8.2 Performance Considerations

*   **CPU Overhead:** Encryption/Decryption uses CPU cycles. Ensure your nodes have AES-NI support (most modern Intel/AMD CPUs do).
*   **Throughput:** Expect a slight decrease in I/O throughput (typically 5-10%) due to encryption overhead.
*   **Monitoring:** Use Prometheus/Grafana to monitor CPU usage and OSD latency after enabling encryption.

### 8.3 Troubleshooting Common Issues

#### Issue 1: OSD Fails to Start After Reboot
**Symptom:** `ceph osd tree` shows OSD as `down`.
**Cause:** The LUKS container failed to open.
**Fix:**
1.  Check logs: `journalctl -u ceph-osd@<id>.service -f`
2.  Check Device Mapper: `dmsetup ls`
3.  Manually open LUKS (if key is available):
    ```bash
    cryptsetup luksOpen /dev/vdd my-test-open
    ```
4.  Restart OSD: `systemctl restart ceph-osd@<id>.service`

#### Issue 2: Disk Not Showing as "Available"
**Symptom:** `ceph orch device ls` shows `/dev/vdd` as `Rejected`.
**Cause:** Residual metadata.
**Fix:**
1.  Run `ceph orch device zap <host> /dev/vdd --force`.
2.  Run `wipefs -a /dev/vdd`.
3.  Wait 30 seconds and refresh: `ceph orch device ls --refresh`.

#### Issue 3: "Device Busy" During Zap
**Symptom:** `ceph-volume lvm zap` fails with "device is busy".
**Cause:** A process (like `podman` or `systemd`) is holding the device.
**Fix:**
1.  Find the process: `fuser -v /dev/vdd`
2.  Kill the process: `kill -9 <PID>`
3.  Retry zap.

### 8.4 Security Best Practices

1.  **Disable Root Login:** Ensure SSH root login is disabled on all Ceph nodes. Use sudo users.
2.  **Network Encryption:** Enable `ms_bind_secure` and configure TLS for cluster traffic (Data-in-Transit) to complement Data-at-Rest encryption.
3.  **Regular Audits:** Periodically run `ceph osd dump | grep dmcrypt` to ensure no new unencrypted OSDs are accidentally added.
4.  **Physical Security:** Encryption protects against disk theft, but physical access to running nodes can still allow data extraction. Secure your data center racks.

---

## Appendix: Quick Reference Command Sheet

| Task | Command |
|------|---------|
| **Check Encryption Status** | `ceph config get global osd_encrypt` |
| **Enable Encryption** | `ceph config set global osd_encrypt true` |
| **List Encrypted OSDs** | `ceph-volume lvm list \| grep encrypted` |
| **Remove OSD** | `ceph osd out <id> && ceph osd rm <id>` |
| **Zap Disk** | `ceph-volume lvm zap /dev/sdX --destroy` |
| **Create Encrypted OSD** | `ceph-volume lvm create --data /dev/sdX --dmcrypt` |
| **Verify LUKS** | `cryptsetup isLuks /dev/sdX` |
| **Check Device Mapper** | `dmsetup ls --tree` |

---

## Conclusion

This guide has walked you through the end-to-end process of enabling and managing Data-at-Rest Encryption in a Ceph cluster. By following these steps, you ensure that your cloud infrastructure meets high security standards while maintaining operational efficiency. Always test these procedures in a non-production lab environment before applying them to live production clusters.

**Author:** Sumon  
**Role:** DevOps & Cloud Engineer  
**Context:** Meghna Cloud R&D Lab  
**Date:** April 27, 2026