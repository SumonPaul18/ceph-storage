# File Name: `ceph-production-hardening-guide.md`

---

# Ceph Production Hardening & Operations Guide
**A Practical Narrative for DevOps Engineers and System Administrators**

## Table of Contents
1. [Introduction & Scenario Overview](#1-introduction--scenario-overview)
2. [Phase 1: Host Preparation & Identity Management](#2-phase-1-host-preparation--identity-management)
3. [Phase 2: Secure Cluster Bootstrap with cephadm](#3-phase-2-secure-cluster-bootstrap-with-cephadm)
4. [Phase 3: Network Security – Enabling Data-in-Transit Encryption (msgr2)](#4-phase-3-network-security--enabling-data-in-transit-encryption-msgr2)
5. [Phase 4: Storage Security – Enabling Data-at-Rest Encryption (dmcrypt)](#5-phase-4-storage-security--enabling-data-at-rest-encryption-dmcrypt)
6. [Phase 5: OpenStack Integration – RBD Pools & Keyring Management](#6-phase-5-openstack-integration--rbd-pools--keyring-management)
7. [Phase 6: Operational Maintenance – OSD Removal & Disk Sanitization](#7-phase-6-operational-maintenance--osd-removal--disk-sanitization)
8. [Phase 7: Troubleshooting & Health Monitoring](#8-phase-7-troubleshooting--health-monitoring)

---

## 1. Introduction & Scenario Overview

### The Real-World Context
Imagine you are a DevOps Engineer at a Cloud Service Provider. You have just provisioned new bare-metal servers or virtual machines for a high-performance Ceph storage cluster. Your goal is not just to make it work, but to make it **secure**, **compliant**, and **production-ready**. 

In this guide, we walk through the journey of building a Ceph cluster from scratch using `cephadm`. We will cover:
*   Preparing the underlying Linux hosts (Ubuntu).
*   Bootstrapping the cluster with secure identities.
*   Encrypting data moving across the network (Data-in-Transit).
*   Encrypting data sitting on the disks (Data-at-Rest).
*   Integrating with OpenStack (Glance, Cinder, Nova).
*   Handling real-world failures, such as removing a failed disk that refuses to wipe due to encryption locks.

This guide assumes you are working with **Ceph Quincy/Reef** or later, using **cephadm** (the container-native orchestrator), and running on **Ubuntu 22.04/24.04**.

---

## 2. Phase 1: Host Preparation & Identity Management

Before installing Ceph, every node in your cluster must have a unique identity and consistent network configuration. Cloning VMs often results in duplicate `machine-id`s, which causes severe issues in distributed systems like Ceph and Kubernetes.

### Requirements
*   Root or sudo access on all nodes.
*   Static IP addresses configured via Netplan.
*   DNS resolution or `/etc/hosts` entries for all nodes.

### Step 1: Fixing Machine ID Conflicts
If you cloned a VM, the `/etc/machine-id` is likely identical across all nodes. This must be unique.

**Practical Guide:**
1.  Check the current ID:
    ```bash
    cat /etc/machine-id
    ```
2.  Remove the old ID and generate a new one:
    ```bash
    sudo rm -f /etc/machine-id
    sudo rm -f /var/lib/dbus/machine-id
    sudo systemd-machine-id-setup
    ```
3.  Verify the new ID is unique:
    ```bash
    cat /etc/machine-id
    ```
4.  **Reboot** the node to ensure all services pick up the new ID.
    ```bash
    sudo reboot
    ```

### Step 2: Network Configuration
Ensure your network interfaces are stable. Use Netplan for persistent configuration.

**Practical Guide:**
1.  Edit the Netplan config:
    ```bash
    sudo nano /etc/netplan/50-cloud-init.yaml
    ```
2.  Ensure static IPs are set for your cluster network (e.g., `192.168.68.0/24`).
3.  Apply changes:
    ```bash
    sudo netplan apply
    ```

### Step 3: SSH Key Distribution
`cephadm` relies heavily on SSH to manage nodes. You need passwordless SSH access from the admin node (e.g., `ceph1`) to all other nodes (`ceph2`, `ceph3`, `ceph4`).

**Practical Guide:**
1.  Generate an SSH key pair on the admin node:
    ```bash
    ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/ceph_id_rsa
    ```
2.  Copy the public key to all target nodes:
    ```bash
    ssh-copy-id -i ~/.ssh/ceph_id_rsa.pub root@ceph2
    ssh-copy-id -i ~/.ssh/ceph_id_rsa.pub root@ceph3
    ssh-copy-id -i ~/.ssh/ceph_id_rsa.pub root@ceph4
    ```
3.  Test connectivity:
    ```bash
    ssh -i ~/.ssh/ceph_id_rsa root@ceph2 "echo 'Connection Successful'"
    ```

### Verification
*   Ensure `ping ceph2` works from `ceph1`.
*   Ensure `ssh root@ceph2` works without a password prompt.

---

## 3. Phase 2: Secure Cluster Bootstrap with cephadm

We will use `cephadm` to bootstrap the first monitor and manager daemons. This initializes the cluster FSID (Filesystem ID) and creates the initial admin keyring.

### Requirements
*   Docker or Podman installed on the bootstrap node.
*   The Ceph container image available (pulled automatically by cephadm).

### Step 1: Install Prerequisites
Install Docker and basic tools on all nodes.

**Practical Guide:**
```bash
sudo apt update && sudo apt install -y curl wget gnupg2 lsb-release software-properties-common apt-transport-https

# Install Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```

### Step 2: Bootstrap the Cluster
Run the bootstrap command on the first node (`ceph1`).

**Practical Guide:**
```bash
sudo cephadm bootstrap --mon-ip 192.168.68.248
```
*   `--mon-ip`: The IP address of the first monitor.

**Output:**
This command will output:
1.  The path to the `ceph.conf` file.
2.  The path to the `ceph.client.admin.keyring`.
3.  The SSH public key for `cephadm`.
4.  The dashboard URL and credentials.

### Step 3: Add Hosts to the Cluster
Tell Ceph about the other nodes so it can deploy monitors and managers there.

**Practical Guide:**
1.  Copy the Ceph public SSH key to other nodes:
    ```bash
    ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph2
    ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph3
    ```
2.  Add the hosts:
    ```bash
    ceph orch host add ceph2 192.168.68.249
    ceph orch host add ceph3 192.168.68.250
    ```

### Verification
Check the cluster status and host list:
```bash
ceph -s
ceph orch host ls
```
You should see all hosts listed with status `online`.

---

## 4. Phase 3: Network Security – Enabling Data-in-Transit Encryption (msgr2)

By default, Ceph may use legacy networking protocols. Modern Ceph uses **msgr2** (Protocol v2), which supports encryption and authentication. We will enforce `secure` mode to ensure all traffic between daemons is encrypted.

### Theory: Why msgr2?
*   **msgr1 (v1):** Legacy, unencrypted, prone to man-in-the-middle attacks.
*   **msgr2 (v2):** Supports TLS encryption, message signing, and better performance.
*   **Modes:**
    *   `legacy`: Allows v1 only.
    *   `crc`: Uses checksums but no encryption.
    *   `secure`: Enforces encryption and authentication.

### Step 1: Check Current Network Mode
**Practical Guide:**
```bash
ceph config dump | grep ms_mode
ceph config get global ms_cluster_mode
ceph config get global ms_service_mode
ceph config get global ms_client_mode
```

### Step 2: Enable msgr2 Binding
Ensure monitors and OSDs are listening on port `3300` (v2) and optionally `6789` (v1) during transition.

**Practical Guide:**
```bash
# Enable v2 binding for Monitors and OSDs
ceph config set mon ms_bind_msgr2 true
ceph config set osd ms_bind_msgr2 true

# Disable v1 binding (Optional: Do this only after verifying v2 works)
# ceph config set mon ms_bind_msgr1 false
# ceph config set osd ms_bind_msgr1 false
```

### Step 3: Enforce Secure Mode
Set the communication mode to `secure` for cluster internal traffic, service traffic, and client traffic.

**Practical Guide:**
```bash
ceph config set global ms_cluster_mode secure
ceph config set global ms_service_mode secure
ceph config set global ms_client_mode secure
```

### Step 4: Restart Daemons
Apply the changes by restarting the daemons.

**Practical Guide:**
```bash
ceph orch restart mon
ceph orch restart osd
ceph orch restart mgr
```

### Verification
1.  Check listening ports. You should see port `3300` active.
    ```bash
    ss -tlnp | grep ceph
    ```
2.  Check the config dump again.
    ```bash
    ceph config dump | grep ms_.*_mode
    ```
    Output should show `secure` for all modes.
3.  Monitor cluster health.
    ```bash
    watch ceph -s
    ```
    Ensure the cluster returns to `HEALTH_OK`. If daemons crash, check logs:
    ```bash
    ceph crash ls-new
    journalctl -u ceph-mon@<id> -f
    ```

---

## 5. Phase 4: Storage Security – Enabling Data-at-Rest Encryption (dmcrypt)

To comply with data sovereignty and security policies, we must encrypt data on the physical disks. Ceph uses **LUKS** (Linux Unified Key Setup) via `dm-crypt` for this purpose.

### Theory: How Ceph Encryption Works
*   When `osd_encrypt` is enabled, `ceph-volume` creates a LUKS container on the raw disk.
*   The encryption key is stored securely in the Ceph Monitor's `config-key` store.
*   When the OSD starts, it retrieves the key, unlocks the LUKS container, and mounts the BlueStore filesystem inside it.

### Step 1: Enable Encryption Globally (For New OSDs)
**Practical Guide:**
```bash
ceph config set global osd_encrypt true
```

### Step 2: Prepare Disks for Encrypted OSDs
Assume we have a new disk `/dev/vdd` on `ceph1`. We will create an OSD service specification to deploy it with encryption.

**Practical Guide:**
1.  Create a service spec file (`osd-encrypted-spec.yaml`):
    ```yaml
    service_type: osd
    service_id: encrypted_osds_vdd
    placement:
      hosts:
        - ceph1
    data_devices:
      paths:
        - /dev/vdd
    encrypted: true
    ```
2.  Apply the spec:
    ```bash
    ceph orch apply -i osd-encrypted-spec.yaml
    ```

### Step 3: Verify Encryption
**Practical Guide:**
1.  Check if the OSD is created:
    ```bash
    ceph orch ps | grep osd
    ```
2.  Check LVM and LUKS status:
    ```bash
    lsblk -f /dev/vdd
    ```
    You should see a `crypto_LUKS` filesystem type on `/dev/vdd`.
3.  Check Ceph metadata:
    ```bash
    ceph osd metadata <osd_id> | grep dmcrypt
    ```
    Output should be `"dmcrypt": true`.

### Troubleshooting Encryption Issues
If an encrypted OSD fails to start, it’s often because the key couldn’t be retrieved or the LUKS container is busy.
*   Check logs: `journalctl -u ceph-osd@<id> -f`
*   Check keys: `ceph config-key ls | grep dm-crypt`

---

## 6. Phase 5: OpenStack Integration – RBD Pools & Keyring Management

Ceph is the backend storage for OpenStack. We need to create specific pools for Glance (Images), Cinder (Volumes), and Nova (VMs), and generate restricted keyrings for each service.

### Step 1: Create RBD Pools
**Practical Guide:**
```bash
# Create pools with appropriate PG counts (adjust based on cluster size)
ceph osd pool create images 32 32
ceph osd pool create volumes 32 32
ceph osd pool create backups 32 32
ceph osd pool create vms 32 32

# Initialize pools for RBD usage
rbd pool init images
rbd pool init volumes
rbd pool init backups
rbd pool init vms

# Enable application tag
ceph osd pool application enable images rbd
ceph osd pool application enable volumes rbd
ceph osd pool application enable backups rbd
ceph osd pool application enable vms rbd
```

### Step 2: Create Client Keys with Least Privilege
Never use the `admin` key for OpenStack services. Create specific users with minimal permissions.

**Practical Guide:**
1.  **Glance (Images):** Needs read/write to `images` pool.
    ```bash
    ceph auth get-or-create client.glance \
      mon 'profile rbd' \
      osd 'profile rbd pool=images' \
      mgr 'profile rbd pool=images' \
      -o /etc/ceph/ceph.client.glance.keyring
    ```

2.  **Cinder (Volumes):** Needs read/write to `volumes` and `backups`, and read-only to `images` (for cloning).
    ```bash
    ceph auth get-or-create client.cinder \
      mon 'profile rbd' \
      osd 'profile rbd pool=volumes, profile rbd pool=backups, profile rbd-read-only pool=images' \
      mgr 'profile rbd pool=volumes, profile rbd pool=backups' \
      -o /etc/ceph/ceph.client.cinder.keyring
    ```

3.  **Nova (VMs):** Needs read/write to `vms` and read-only to `images`.
    ```bash
    ceph auth get-or-create client.nova \
      mon 'profile rbd' \
      osd 'profile rbd pool=vms, profile rbd-read-only pool=images' \
      mgr 'profile rbd pool=vms' \
      -o /etc/ceph/ceph.client.nova.keyring
    ```

### Step 3: Distribute Keyrings to OpenStack Controllers
Copy the generated keyrings and `ceph.conf` to your OpenStack controller nodes.

**Practical Guide:**
```bash
scp /etc/ceph/ceph.conf controller@192.168.68.69:/etc/ceph/
scp /etc/ceph/ceph.client.glance.keyring controller@192.168.68.69:/etc/ceph/
scp /etc/ceph/ceph.client.cinder.keyring controller@192.168.68.69:/etc/ceph/
scp /etc/ceph/ceph.client.nova.keyring controller@192.168.68.69:/etc/ceph/
```

### Verification
On the OpenStack controller, test connectivity:
```bash
ceph -n client.glance -s
rbd -p images ls
```

---

## 7. Phase 6: Operational Maintenance – OSD Removal & Disk Sanitization

Disks fail. Or you might need to repurpose a disk. Removing an OSD from a Ceph cluster, especially an encrypted one, can be tricky due to device locks.

### The Challenge
When you try to zap (wipe) an encrypted OSD, you might encounter:
`Error EINVAL: Zap failed: ... Device or resource busy`
This happens because the LVM volume or LUKS mapper is still active in the kernel.

### Step 1: Mark OSD as Out
Stop data flow to the OSD.
**Practical Guide:**
```bash
ceph osd out <osd_id>
```
Wait for data rebalancing to complete (`ceph -s` shows `HEALTH_OK`).

### Step 2: Stop the OSD Daemon
**Practical Guide:**
```bash
ceph orch daemon stop osd.<osd_id>
```

### Step 3: Remove OSD from Cluster
**Practical Guide:**
```bash
ceph orch osd rm <osd_id>
```
If it hangs, force it:
```bash
ceph orch osd rm <osd_id> --force
```

### Step 4: Manual Cleanup (If Orchestrator Fails)
Sometimes the orchestrator leaves behind authentication keys or CRUSH entries.
**Practical Guide:**
```bash
ceph osd crush remove osd.<osd_id>
ceph auth del osd.<osd_id>
ceph osd rm <osd_id>
```

### Step 5: Force Wipe the Disk (The Hard Part)
If `ceph orch device zap` fails with "Device or resource busy", follow this aggressive cleanup procedure.

**Practical Guide:**
1.  **Kill Holding Processes:**
    ```bash
    fuser -km /dev/vdd
    ```
2.  **Remove Device-Mapper (LUKS/LVM) Links:**
    Identify the mapper name:
    ```bash
    dmsetup ls | grep ceph
    ```
    Remove it forcefully:
    ```bash
    dmsetup remove --force /dev/mapper/ceph-<uuid>-osd-block-<uuid>
    ```
3.  **Deactivate LVM:**
    ```bash
    lvchange -an /dev/ceph-<vg-name>/osd-block-<uuid>
    vgremove -f ceph-<vg-name>
    pvremove -ff /dev/vdd
    ```
4.  **Zero Out Headers:**
    Use `dd` to overwrite the partition table and LUKS headers.
    ```bash
    dd if=/dev/zero of=/dev/vdd bs=1M count=100 status=progress
    sync
    partprobe /dev/vdd
    ```
5.  **Verify Clean State:**
    ```bash
    wipefs /dev/vdd  # Should return empty
    lsblk -f /dev/vdd # Should show no FSTYPE
    ```

### Step 6: Refresh Ceph Inventory
**Practical Guide:**
```bash
ceph orch device ls --refresh
```
The disk should now appear as `Available` and ready for reuse.

---

## 8. Phase 7: Troubleshooting & Health Monitoring

A production cluster requires constant vigilance. Here are the essential commands for diagnosing issues.

### Common Issues & Solutions

#### 1. Slow OSDs
**Symptom:** `ceph health detail` shows `slow ops`.
**Diagnosis:**
```bash
ceph osd perf
ceph daemon osd.<id> perf dump | grep latency
journalctl -u ceph-osd@<id> -f | grep slow
```
**Action:** Check disk I/O wait (`iostat -x 1`), network latency, or CPU saturation.

#### 2. Cluster Health Warnings
**Symptom:** `HEALTH_WARN` or `HEALTH_ERR`.
**Diagnosis:**
```bash
ceph health detail
ceph crash ls-new
ceph crash info <crash-id>
```
**Action:** Read the crash log. Common causes include memory leaks, network partitions, or disk failures.

#### 3. Network Connectivity Issues
**Symptom:** Monitors can't talk to OSDs.
**Diagnosis:**
```bash
ss -tlnp | grep 3300  # Check msgr2 port
ss -tlnp | grep 6789  # Check msgr1 port
ping <other-node-ip>
```
**Action:** Ensure firewalls allow ports `3300` and `6789`. Verify `ms_mode` is consistent across all daemons.

#### 4. Authentication Failures
**Symptom:** OpenStack services can't connect to Ceph.
**Diagnosis:**
```bash
ceph auth get client.cinder
# Compare key with the one in /etc/ceph/ceph.client.cinder.keyring
```
**Action:** Regenerate the keyring and redistribute it if mismatched.

### Monitoring Stack Integration
For long-term visibility, integrate Ceph with Prometheus and Grafana.
1.  Enable the Prometheus module in Ceph Manager:
    ```bash
    ceph mgr module enable prometheus
    ```
2.  Configure Prometheus to scrape the Ceph manager endpoint (default port `9283`).
3.  Import the official Ceph Dashboard into Grafana.

---

## Conclusion

Building a secure, production-grade Ceph cluster involves more than just installation. It requires careful attention to:
1.  **Identity:** Unique machine IDs and proper SSH management.
2.  **Network Security:** Enforcing `msgr2` with `secure` mode for data-in-transit encryption.
3.  **Storage Security:** Using `dmcrypt` for data-at-rest encryption.
4.  **Integration:** Creating least-privilege keys for OpenStack services.
5.  **Maintenance:** Knowing how to aggressively clean up failed, encrypted disks when standard tools fail.

By following this guide, you ensure that your Ceph cluster is not only functional but also resilient and compliant with modern security standards.

---

### Appendix: Quick Reference Commands

| Task | Command |
| :--- | :--- |
| Check Cluster Status | `ceph -s` |
| List OSDs | `ceph osd tree` |
| Check Disk Usage | `ceph df` |
| List Pools | `ceph osd pool ls` |
| Check Network Mode | `ceph config dump \| grep ms_` |
| Check Encryption | `ceph osd metadata <id> \| grep dmcrypt` |
| Restart All OSDs | `ceph orch restart osd` |
| Zap a Disk | `ceph orch device zap <host> <dev> --force` |
| View Logs | `journalctl -u ceph-osd@<id> -f` |