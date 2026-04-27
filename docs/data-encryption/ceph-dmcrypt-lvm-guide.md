# 🛠️ Ceph OSD Engineering: LVM & Dmcrypt Practical Handbook
**Target Audience:** DevOps Engineers, System Administrators, Storage Engineers  
**Context:** Managing Ceph Storage Clusters on Ubuntu/RHEL with Encryption (Data at Rest) and LVM.

---

## 📋 Table of Contents

1. [Environment Readiness & Requirements](#1-environment-readiness--requirements)
2. [Troubleshooting Binary & Path Errors](#2-troubleshooting-binary--path-errors)
3. [Practical Guide: Creating Encrypted OSDs](#3-practical-guide-creating-encrypted-osds)
4. [Practical Guide: Creating Standard OSDs](#4-practical-guide-creating-standard-osds)
5. [OSD Activation & Service Management](#5-osd-activation--service-management)
6. [Managing Mixed-Mode OSDs (YAML Specs)](#6-managing-mixed-mode-osds-yaml-specs)
7. [Maintenance & Best Practices](#7-maintenance--best-practices)

---

## 1. Environment Readiness & Requirements

Before deploying storage, we must ensure the host operating system is ready. In a real-world scenario, skipping this step often leads to "command not found" errors later.

### Step 1.1: Verify Ceph Installation
We need to confirm that the Ceph binaries are installed and accessible.

```bash
ceph --version
```
> **Explanation:** Checks the installed version of Ceph. If this fails, Ceph is not installed or the path is broken.
> *   `ceph`: The main command-line interface for Ceph.
> *   `--version`: Flag to display version information.

### Step 1.2: Check Cluster Health
Ensure the cluster is currently healthy before adding new components.

```bash
ceph status
```
> **Explanation:** Displays the overall health status of the Ceph cluster (e.g., `HEALTH_OK`).
> *   `status`: Subcommand to show cluster state, monitor quorum, and OSD status.

### Step 1.3: Identify Available Disks
List all block devices to identify which disks are raw and available for OSD creation.

```bash
lsblk
```
> **Explanation:** Lists information about all available block devices.
> *   Look for disks (e.g., `/dev/vdd`, `/dev/sdb`) that have no partitions or mount points.
> *   **Warning:** Do not use disks that contain existing data or OS partitions.

---

## 2. Troubleshooting Binary & Path Errors

A common issue in manual deployments is when `ceph-volume` cannot find helper tools like `ceph-bluestore-tool`. This usually happens if the `ceph-osd` package is missing or the PATH variable is incorrect.

### Step 2.1: Install Missing Packages
If you see `FileNotFoundError: No such file or directory: 'ceph-bluestore-tool'`, install the OSD package.

```bash
apt-get update && apt-get install ceph-osd -y
```
> **Explanation:** Updates the package list and installs the Ceph OSD daemon package.
> *   `apt-get update`: Refreshes the local package index.
> *   `apt-get install`: Installs the specified package.
> *   `ceph-osd`: The package containing the OSD binary and related tools.
> *   `-y`: Automatically answers "yes" to prompts, enabling non-interactive installation.

### Step 2.2: Verify Binary Paths
Confirm that the system can locate the necessary executables.

```bash
which ceph-osd
which ceph-bluestore-tool
```
> **Explanation:** Shows the full path of the executable if found.
> *   `which`: Command to locate a binary in the user's PATH.
> *   **Expected Output:** `/usr/bin/ceph-osd` or similar. If empty, the binary is missing.

### Step 2.3: Fix PATH Variable (If Needed)
If binaries exist but are not found, update the session PATH.

```bash
export PATH=$PATH:/usr/bin:/usr/sbin:/sbin:/bin
```
> **Explanation:** Adds standard system directories to the current session's PATH.
> *   `export`: Sets an environment variable.
> *   `$PATH`: The current PATH variable.
> *   `/usr/bin:/usr/sbin...`: Standard directories where Linux stores executables.

### Step 2.4: Zap Disk Before Retry
If a previous OSD creation failed, the disk may have leftover metadata. Clean it before retrying.

```bash
ceph-volume lvm zap /dev/vdd --destroy
```
> **Explanation:** Removes all LVM and Ceph metadata from the specified disk.
> *   `ceph-volume lvm zap`: Command to clean a device.
> *   `/dev/vdd`: The target disk device.
> *   `--destroy`: Flag to completely wipe partition tables and signatures. **Use with caution.**

---

## 3. Practical Guide: Creating Encrypted OSDs

Encryption at rest is critical for compliance and security. Ceph uses `dm-crypt` (Linux Kernel Device Mapper) to encrypt data before writing it to the disk.

### Step 3.1: Create Encrypted OSD
This command prepares the disk, creates LVM volumes, encrypts them, and activates the OSD.

```bash
ceph-volume lvm create --bluestore --dmcrypt --data /dev/vdd
```
> **Explanation:** Creates a new OSD using BlueStore backend with encryption enabled.
> *   `ceph-volume lvm create`: Main command to create an OSD using LVM.
> *   `--bluestore`: Specifies the storage backend (modern default for Ceph).
> *   `--dmcrypt`: Enables encryption using LUKS/dm-crypt.
> *   `--data /dev/vdd`: Specifies the raw disk device to use for data storage.

### Step 3.2: Verify Encryption Layer
Check if the encrypted layer is active in the device mapper.

```bash
lsblk -f | grep crypto
```
> **Explanation:** Filters the block device list to show only encrypted filesystems.
> *   `lsblk -f`: Lists block devices with filesystem information.
> *   `| grep crypto`: Pipes output to search for the string "crypto".
> *   **Expected Output:** You should see `crypto_LUKS` under the FSTYPE column for your device.

---

## 4. Practical Guide: Creating Standard OSDs

For non-sensitive data or performance-heavy workloads where encryption overhead is undesirable, use standard (unencrypted) OSDs.

### Step 4.1: Create Standard OSD
Create an OSD without encryption.

```bash
ceph-volume lvm create --bluestore --data /dev/vdb
```
> **Explanation:** Creates a standard unencrypted OSD.
> *   `--bluestore`: Uses the BlueStore backend.
> *   `--data /dev/vdb`: Specifies the raw disk.
> *   **Note:** The `--dmcrypt` flag is omitted, so no encryption layer is created.

### Step 4.2: Verify OSD Status
Confirm the new OSD is part of the cluster and running.

```bash
ceph osd tree
```
> **Explanation:** Displays the hierarchical structure of OSDs in the cluster.
> *   `osd tree`: Shows OSD IDs, weights, status (up/down), and host mapping.
> *   **Check:** Ensure the new OSD ID (e.g., `osd.12`) is listed and status is `up`.

---

## 5. OSD Activation & Service Management

Sometimes, after a reboot or manual intervention, an OSD may remain "down." This is common with encrypted OSDs if the key isn't automatically unlocked.

### Step 5.1: List LVM Volumes
Get the UUID and ID of the prepared OSDs.

```bash
ceph-volume lvm list
```
> **Explanation:** Lists all Ceph LVM volumes managed by `ceph-volume`.
> *   **Output:** Shows OSD ID, UUID, device path, and encryption status.
> *   **Use:** Copy the `<OSD_ID>` and `<OSD_UUID>` for activation.

### Step 5.2: Manually Activate OSD
If an OSD is down, activate it manually using its ID and UUID.

```bash
ceph-volume lvm activate <OSD_ID> <OSD_UUID>
```
> **Explanation:** Starts the OSD service for a specific volume.
> *   `<OSD_ID>`: Replace with the numeric ID (e.g., `11`).
> *   `<OSD_UUID>`: Replace with the long UUID string from `ceph-volume lvm list`.
> *   **Note:** This command generates the systemd unit file and starts the service.

### Step 5.3: Enable Auto-Start on Boot
Ensure the OSD service starts automatically after a server reboot.

```bash
systemctl enable ceph-osd@11
```
> **Explanation:** Enables the systemd service for a specific OSD.
> *   `systemctl enable`: Configures the service to start at boot.
> *   `ceph-osd@11`: The service instance for OSD ID 11. Replace `11` with your actual OSD ID.

### Step 5.4: Enable Volume Discovery Service
Ensures new or existing volumes are detected on boot.

```bash
systemctl enable ceph-volume@lvm
```
> **Explanation:** Enables the background service that scans for Ceph LVM volumes.
> *   `ceph-volume@lvm`: The systemd template unit for LVM volume management.

---

## 6. Managing Mixed-Mode OSDs (YAML Specs)

In large clusters, running manual commands for each disk is inefficient. Use Declarative YAML specs with `ceph orch` (Orchestrator) to manage bulk deployments.

### Step 6.1: Create OSD Configuration YAML
Create a file named `osd-config.yaml` with the following structure.

```yaml
service_type: osd
service_id: default_drive_group
placement:
  hosts:
    - node1
    - node2
data_devices:
  all: true
encryption: true
```
> **Explanation:** Defines a drive group for automatic OSD deployment.
> *   `service_type: osd`: Specifies this is an OSD service definition.
> *   `placement.hosts`: Lists the hostnames where OSDs should be created.
> *   `data_devices.all: true`: Uses all available unused disks on the listed hosts.
> *   `encryption: true`: Enables dmcrypt for all OSDs in this group. Set to `false` for standard OSDs.

### Step 6.2: Apply the Configuration
Deploy the OSDs based on the YAML file.

```bash
ceph orch apply -i osd-config.yaml
```
> **Explanation:** Applies the declarative configuration to the cluster.
> *   `orch apply`: Orchestrator command to apply service specifications.
> *   `-i osd-config.yaml`: Input file containing the specification.

### Step 6.3: Rescan Host Inventory
If a disk is not showing as "Available," force a rescan.

```bash
ceph orch host rescan <HOSTNAME>
```
> **Explanation:** Forces the orchestrator to re-inventory the hardware on a specific host.
> *   `<HOSTNAME>`: Replace with the actual hostname of the node (e.g., `node1`).

---

## 7. Maintenance & Best Practices

Regular maintenance ensures longevity and security of the storage cluster.

### Step 7.1: Check Physical Disk Health
Monitor SMART data to predict disk failures.

```bash
smartctl -a /dev/vdd
```
> **Explanation:** Displays detailed SMART (Self-Monitoring, Analysis and Reporting Technology) information.
> *   `smartctl`: Utility for controlling and monitoring SMART disks.
> *   `-a`: Shows all SMART information.
> *   `/dev/vdd`: The target physical disk.
> *   **Look for:** `Reallocated_Sector_Ct` or `Current_Pending_Sector` values greater than 0.

### Step 7.2: Verify Encryption Keys
Ensure encryption keys are stored securely in the Ceph Monitor config-key store.

```bash
ceph config-key get dm-crypt/osd.<OSD_ID>/luks
```
> **Explanation:** Retrieves the LUKS header/key reference for a specific OSD.
> *   `config-key get`: Fetches a key from the monitor's key-value store.
> *   `dm-crypt/osd.<OSD_ID>/luks`: The key path for the OSD's encryption metadata.
> *   **Note:** This does not show the raw password but confirms the key metadata exists.

### Step 7.3: Safely Remove/Zap a Disk
Before removing or repurposing a disk, fully clean it.

```bash
ceph-volume lvm zap /dev/vdd --destroy
```
> **Explanation:** Wipes all Ceph and LVM metadata.
> *   **Critical:** Always run this before physically removing a drive or reusing it for a different purpose.

---

## 💡 Practical Pro-Tip: The Reboot Test

**Scenario:** You have configured a new encrypted OSD. How do you know it will survive a power outage?

**Action:** Reboot the node once.

```bash
reboot
```

**Verification:** After the node comes back online, check the OSD status.

```bash
ceph osd tree
```

**Interpretation:**
*   **If OSD is `up`:** Your encryption key management (via `ceph-volume` and systemd) is correctly configured. The node successfully unlocked the LUKS container and started the OSD.
*   **If OSD is `down`:** The node failed to retrieve the dmcrypt key or the systemd service failed to start. Check logs using `journalctl -u ceph-osd@<ID> -f`.

---

**End of Guide.**  
*This document serves as a blueprint for Linux Storage Engineers working with Ceph-LVM-Dmcrypt stacks.*
