# Complete Guide to Uninstalling and Removing Ceph
### A Step-by-Step Professional Cleanup Guide for Lab and Production Environments

---

## 📑 Table of Contents
1. [⚠️ Critical Warnings & Pre-requisites](#-critical-warnings--pre-requisites)
2. [Phase 1: Stop and Remove the Ceph Cluster](#phase-1-stop-and-remove-the-ceph-cluster)
3. [Phase 2: Clean Up Containers (Docker/Podman)](#phase-2-clean-up-containers-dockerpodman)
4. [Phase 3: Uninstall System Packages](#phase-3-uninstall-system-packages)
5. [Phase 4: Remove Configuration Files and Data](#phase-4-remove-configuration-files-and-data)
6. [Phase 5: Clean SSH Keys and Repository Settings](#phase-5-clean-ssh-keys-and-repository-settings)
7. [Phase 6: Final Verification and Reboot](#phase-6-final-verification-and-reboot)
8. [✅ Ready for Fresh Installation](#-ready-for-fresh-installation)

---

## ⚠️ Critical Warnings & Pre-requisites

> **🛑 DANGER: DATA LOSS WARNING**
> The steps below will **permanently delete** all Ceph data, including:
> - All Storage Pools and Images (RBD)
> - All Object Gateway buckets
> - All Configuration files (`ceph.conf`)
> - All OSD data on physical disks
>
> **Action Required:**
> 1. Ensure you have a valid backup of any critical data.
> 2. Verify you are running these commands on the correct nodes.
> 3. This guide assumes you have `root` or `sudo` privileges.

---

## Phase 1: Stop and Remove the Ceph Cluster

Before removing packages, we must gracefully stop the cluster services using `cephadm`.

### Step 1.1: Identify the Cluster FSID
Every Ceph cluster has a unique ID called FSID. We need this to remove the cluster.

```bash
sudo cephadm ls
```
*Look for the output similar to:*
```text
[{"fsid": "438ec756-12f7-11f1-89d6-bc241192dd00", "container_id": "...", ...}]
```
**Copy the FSID** (e.g., `438ec756-12f7-11f1-89d6-bc241192dd00`).

### Step 1.2: Remove the Cluster
Run the removal command with the `--force` flag to ensure all services are stopped and removed immediately.

```bash
# Replace <YOUR-FSID> with the ID you copied above
sudo cephadm rm-cluster --fsid <YOUR-FSID> --force
```
*Example:*
```bash
sudo cephadm rm-cluster --fsid 438ec756-12f7-11f1-89d6-bc241192dd00 --force
```

---

## Phase 2: Clean Up Containers (Docker/Podman)

`cephadm` runs Ceph daemons inside containers. Even after removing the cluster, some leftover containers or volumes might remain.

### Option A: If you use Docker
```bash
# Stop and remove all Ceph-related containers
sudo docker rm -f $(sudo docker ps -aq --filter name=ceph-) 2>/dev/null

# Remove unused volumes, networks, and images
sudo docker volume prune -f
sudo docker network prune -f
sudo docker image prune -a -f
sudo docker system prune -a -f --volumes
```

### Option B: If you use Podman (Default on some newer systems)
```bash
# Stop and remove all Ceph-related containers
sudo podman rm -f $(sudo podman ps -aq --filter name=ceph-) 2>/dev/null

# Remove unused volumes and networks
sudo podman volume prune -f
sudo podman network prune -f
```

---

## Phase 3: Uninstall System Packages

Now, remove the `cephadm` tool and any Ceph client libraries installed via APT.

```bash
# Remove cephadm and ceph-common packages completely
sudo apt remove --purge cephadm ceph-common ceph-mon ceph-mgr ceph-osd -y

# Remove any unused dependencies
sudo apt autoremove -y
```

---

## Phase 4: Remove Configuration Files and Data

This step deletes all remaining configuration files, logs, and data directories from the disk.

```bash
# Remove main configuration and data directories
sudo rm -rf /etc/ceph
sudo rm -rf /var/lib/ceph
sudo rm -rf /var/log/ceph
sudo rm -rf /var/cache/ceph

# Remove Rook directory if it exists (from previous Kubernetes setups)
sudo rm -rf /var/lib/rook

# Remove temporary bootstrap files if they exist in current directory
rm -f ceph.conf ceph.client.admin.keyring
```

---

## Phase 5: Clean SSH Keys and Repository Settings

`cephadm` uses SSH keys to communicate between nodes. These must be cleaned to avoid conflicts during the new installation.

### Step 5.1: Clean SSH Authorized Keys
Remove the specific Ceph public key from the root user's authorized keys.

```bash
# Backup authorized_keys first (Optional but safe)
cp /root/.ssh/authorized_keys /root/.ssh/authorized_keys.bak

# Remove lines containing 'ceph-public' or 'ceph-admin'
sudo sed -i '/ceph-public/d' /root/.ssh/authorized_keys
sudo sed -i '/ceph-admin/d' /root/.ssh/authorized_keys
```

### Step 5.2: Remove Ceph Binaries and Repositories
Ensure no old scripts or repository lists remain.

```bash
# Remove cephadm binary from common locations
sudo rm -f /usr/sbin/cephadm
sudo rm -f /usr/local/bin/cephadm
sudo rm -f ./cephadm

# Remove GPG Key and Apt Source List
sudo rm -f /usr/share/keyrings/ceph-archive-keyring.gpg
sudo rm -f /etc/apt/sources.list.d/ceph.list
```

---

## Phase 6: Final Verification and Reboot

A reboot ensures that any mounted Ceph filesystems are unmounted and kernel modules are cleared.

### Step 6.1: Reboot the Server
```bash
sudo reboot
```

### Step 6.2: Verify Cleanup (After Login)
Once the server is back online, log in and run these checks to confirm the system is clean.

```bash
# Check 1: Config directory should not exist
ls /etc/ceph
# Expected Output: ls: cannot access '/etc/ceph': No such file or directory

# Check 2: cephadm command should not be found
cephadm --version
# Expected Output: Command 'cephadm' not found...

# Check 3: No Ceph processes running
ps aux | grep ceph
# Expected Output: Only the 'grep' command itself should appear.

# Check 4: No Ceph mount points
mount | grep ceph
# Expected Output: (Empty)
```

---

## ✅ Ready for Fresh Installation

Your system is now completely clean. You can proceed with a fresh Ceph installation using your preferred method (e.g., `curl` script or official APT repository) without any conflicts from the previous setup.

**Next Steps:**
1. Update your package list: `sudo apt update`
2. Install `cephadm` again using the official guide.
3. Bootstrap your new cluster.
