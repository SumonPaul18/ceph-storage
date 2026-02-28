# 🐘 Ceph Single-Node Cluster Installation Guide using cephadm + Docker
> **Official Documentation Reference:** https://docs.ceph.com/en/latest/cephadm/install/

---

## 📋 My Environment Summary

| Item | Value |
|------|-------|
| **Operating System** | Ubuntu 24.04.3 LTS |
| **Hostname** | `ceph1` |
| **FQDN** | `ceph1.jmc.com` |
| **IP Address** | `192.168.68.180` |
| **Container Runtime** | Docker (as requested) |
| **OS Disk** | 100 GB (Only for OS) |
| **OSD Disk** | `/dev/sdb` (100 GB, Raw/Unpartitioned) |

---

## ⚠️ Pre-Installation Checklist

Before starting, ensure the following:

- ✅ Root or sudo access to the VM
- ✅ Internet connectivity for package downloads
- ✅ SSH server enabled (required for cephadm orchestration)
- ✅ Time synchronization enabled (chrony or ntp)
- ✅ No existing Ceph data on `/dev/sdb` (must be completely raw)

---

## 🔧 Step 1: System Preparation and Prerequisites

### Update system packages and install essential tools

#### Update package index and upgrade existing packages
```
sudo apt update && sudo apt upgrade -y
```
### Configure hostname and FQDN
#### Set the system hostname
```bash
sudo hostnamectl set-hostname ceph1
```
#### Add FQDN mapping to /etc/hosts for local resolution
```
echo "192.168.68.180 ceph1.paulco.xyz ceph1" | sudo tee -a /etc/hosts
```
#### Verify hostname resolution
```
hostnamectl status
ping -c 2 ceph1.paulco.xyz
```

#### Install required packages: SSH, LVM2, time sync, and utilities
```
sudo apt install -y ssh openssh-server lvm2 chrony curl jq apt-transport-https ca-certificates gnupg
```
### Configure time synchronization with chrony
#### Enable and start chrony service for accurate time sync
```bash
sudo systemctl enable --now chrony
```
#### Verify time sources are synchronized
```
chronyc sources -v
```

### Enable and start SSH service (mandatory for cephadm)

#### Enable SSH to start on boot and start it immediately
```
sudo systemctl enable --now ssh
```
#### Verify SSH service is active
```
sudo systemctl status ssh
```

### Install Docker Engine (Official Method for Ubuntu 24.04)

#### Add Docker's official GPG key
```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```
#### Add Docker repository to Apt sources
```
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
#### Update package index and install Docker Engine
```
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
#### Add current user to docker group (optional, for non-root docker usage)
```
sudo usermod -aG docker $USER
```
#### Enable and start Docker service
```
sudo systemctl enable --now docker
```
#### Verify Docker installation
```
docker --version
```

### Configure passwordless SSH access for root (required by cephadm)
#### Generate SSH key pair for root user (no passphrase)
```bash
sudo ssh-keygen -t rsa -b 4096 -N "" -f /root/.ssh/id_rsa
```
#### Copy public key to authorized_keys for local passwordless access
```
sudo cp /root/.ssh/id_rsa.pub /root/.ssh/authorized_keys
sudo chmod 600 /root/.ssh/authorized_keys
sudo chmod 700 /root/.ssh
```
#### Test passwordless SSH to localhost
```
sudo ssh ceph1@ceph1 "echo 'SSH connection successful'"
```

---

## 📦 Step 2: Install cephadm (Official Method)

You have two methods to install `cephadm`. Choose the one that fits your needs.

---

### **Method 1: Install via APT Repository (Ubuntu Default)**

**Best for:** Quick setup on Ubuntu where the default version is acceptable.

**⚠️ Limitation:** Installs the Ceph version bundled with your specific Ubuntu release (e.g., Ubuntu 24.04 installs Ceph Squid v19). You cannot easily choose an older or specific LTS version like Reef (v18).

#### 1. Install cephadm from Ubuntu repositories
```bash
sudo apt update
sudo apt install -y cephadm
```
#### 2. Verify installation path
```
which cephadm
```
> **Expected Output:** /usr/sbin/cephadm

#### 3. Check version
```
cephadm version
```
> **Note:** Sometimes this might show 'UNKNOWN' on some Ubuntu packages, but check with:
```
dpkg -l | grep cephadm
```

---

### **Method 2: Install Specific Version using `curl` (Recommended for Production)**
**Best for:** Production environments where you need a specific version (e.g., Reef v18 LTS) regardless of your OS version.
**✅ Advantage:** Full control over the Ceph release.

**🔍 How to find the correct download link:**

1. Go to **[https://download.ceph.com/](https://download.ceph.com/)**.

2. Look for your selected  `ceph version` (e.g., `rpm-reef`, `rpm-squid`, `rpm-quincy`).
   * `reef` = Ceph v18 (LTS)
   * `squid` = Ceph v19 (Latest)
   * `quincy` = Ceph v17 (Older LTS)

3. Navigate to the: `selected version` -> Click on `el9` After then -> Click on `noarch` and lookout `cephadm`.

4. Copy the link address (Right-click > Copy Link Address). It will look like:
- `https://download.ceph.com/rpm-squid/el9/noarch/cephadm`.

5. Replace the URL in the command below with your copied link.

#### 1. Define your desired release (e.g., reef, squid, quincy)

```bash
CEPH_RELEASE=squid
```
#### 2. Download the specific cephadm binary
#### Replace the URL below if you found a different link from the website
```
curl --silent --remote-name --location https://download.ceph.com/rpm-${CEPH_RELEASE}/el9/noarch/cephadm
```
#### 3. Make it executable
```
chmod +x cephadm
```
#### 4. Move to system path
```
sudo mv cephadm /usr/sbin/cephadm
```
#### 5. Verify the specific version
```
cephadm version
```
> **Expected Output:** cephadm version 19.2.x ... squid (stable)

---

## 🚀 Step 3: Bootstrap the New Ceph Cluster

### Initialize the first monitor and manager daemons

#### Bootstrap a new Ceph cluster with Docker engine and single-node optimizations

```bash
sudo cephadm bootstrap \
  --mon-ip 192.168.68.180 \
  --single-host-defaults \
  --container-engine docker  \
  --initial-dashboard-user admin \
  --initial-dashboard-password admin123
```

#### 🔍 Explanation of Bootstrap Flags:
| Flag | Purpose |
|------|---------|
| `--mon-ip 192.168.68.180` | Specifies the IP address for the initial Monitor daemon |
| `--single-host-defaults` | Optimizes CRUSH rules for single-node deployment (reduces replica requirements) |
| `--container-engine docker` | Explicitly uses Docker instead of default Podman |
| `--initial-dashboard-user admin` | Setup Your User Name Replace `admin` |
| `--initial-dashboard-password admin123` | Setup the User Password replace `admin123` |


#### ✅ What this command does automatically:

1. Pulls required Ceph container images (`ceph/ceph:v19` or latest stable)
2. Deploys the first Monitor (`mon.ceph1`) and Manager (`mgr.ceph1`) daemons
3. Generates `/etc/ceph/ceph.conf` and `/etc/ceph/ceph.client.admin.keyring`
4. Configures SSH key management for future node orchestration
5. Enables basic services: Dashboard, Prometheus, Grafana, Node-exporter

> ⏱️ **Note:** First execution may take 2-5 minutes to pull container images.

---

## 🔧 Step 4: Managing Ceph Cluster via Command Line 

 **Best Practice Note:** By default, `cephadm` does not install the `ceph` binary directly on the host to avoid dependency conflicts. The recommended approach for production environments is to use `cephadm shell`, which runs commands inside a container with the exact matching version of Ceph tools.

---

### 🔹 Method 1: Using `cephadm shell` (Recommended for Production)
*This method ensures you are always using the correct version of the Ceph CLI that matches your cluster, regardless of what is installed on the host OS.*

#### 1. Check Cluster Status
```bash
sudo cephadm shell -- ceph -s
```

#### 2. Verify Ceph Version
Ensure the running version matches your target release (e.g., Reef v18).
```bash
sudo cephadm shell -- ceph -v
```
> **Expected Output:** `ceph version 18.2.x (...) reef (stable)`

#### 3. Check Dashboard Access URL
Retrieve the HTTPS URL and login credentials for the Ceph Dashboard.
```bash
sudo cephadm shell -- ceph mgr services
```
> **Output Example:** `{"dashboard": "https://ceph-node-01:8443/"}`

#### 4. Detailed Cluster Health
Check for any warnings or errors in the cluster health status.
```bash
sudo cephadm shell -- ceph health detail
```

#### 5. List Orchestrator Services
View all running Ceph daemons (MON, MGR, OSD, RGW, etc.) managed by the orchestrator.
```bash
sudo cephadm shell -- ceph orch ls
```

#### 6. List Cluster Hosts
Verify which nodes are part of the Ceph cluster.
```bash
sudo cephadm shell -- ceph orch host ls
```

#### 7. Check Component Versions
See the version of each individual Ceph component running across the cluster.
```bash
sudo cephadm shell -- ceph versions
```

#### 8. Orchestrator Status
Check if the orchestrator module is active and functioning.
```bash
sudo cephadm shell -- ceph orch status
```

#### 9. Verify Network Configuration
Inspect network settings (Public/Cluster networks) applied to the daemons.
```bash
sudo cephadm shell -- ceph config dump | grep network
```

---

### 🔹 Method 2: Interactive Shell Mode
*Useful for running multiple commands sequentially without typing `sudo cephadm shell --` every time.*

#### 1. Enter the Interactive Shell
```bash
sudo cephadm shell
```
*(You will see the prompt change, indicating you are now inside the Ceph container context)*

#### 2. Run Commands Directly
```bash
ceph status
ceph osd tree
ceph df
```

#### 3. Exit the Shell
Return to the host system shell.
```bash
exit
```

---

### 🔹 Method 3: Installing `ceph-common` on Host (Alternative)
*If you prefer running `ceph` commands directly from the host shell (e.g., for scripting or automation tools like Ansible), you can install the `ceph-common` package.*

> ⚠️ **Production Warning:** Ensure the version of `ceph-common` installed on the host matches the cluster version to avoid protocol mismatches. Using `cephadm shell` is generally safer for manual administration.

#### 1. Install `ceph-common`
**Option A: Via cephadm (Recommended)**
```bash
sudo cephadm install ceph-common
```

**Option B: Via APT (Manual)**
```bash
sudo apt update
sudo apt install -y ceph-common
```
> Now you can run `ceph` commands directly without opening the interactive shell.
#### 2. Verify Installation
Check if the package is installed correctly.
```bash
dpkg -l | grep ceph-common
```

#### 3. Run Commands Directly on Host
Now you can run Ceph commands without the shell wrapper.
```bash
ceph -v
ceph -s
```

---

## 💾 Step 5: Add OSD Using /dev/sdb

### Verify that /dev/sdb is detected as available for OSD use
#### List all storage devices and their availability for Ceph OSD
```bash
sudo cephadm shell -- ceph orch device ls
```

#### 📋 Expected Output Example:
```
Hostname  Path      Type  Size   Available  Reject reasons
ceph1     /dev/sdb  hdd   100G   Yes        -
```

> ✅ If `Available` column shows `Yes`, the disk is ready for OSD deployment.

---

### ⚠️ Important: About /dev/sdb Disk Preparation

> **Question:** Do I need to partition or format `/dev/sdb` before using it as a Ceph OSD?  
> **Answer:** **No. Do not partition or format the disk.**

Cephadm uses `ceph-volume` which automatically:
- ✅ Detects raw/unpartitioned disks
- ✅ Initializes the disk with BlueStore format (Ceph's default storage backend)
- ✅ Creates required LVM structures internally
- ✅ Adds the disk as an OSD to the cluster

> **✅ Keep `/dev/sdb` completely raw.** If the disk has existing partitions, LVM metadata, or a filesystem, Ceph will reject it for OSD use.

---

### Deploy OSD daemon on /dev/sdb
#### Add /dev/sdb as a new OSD daemon to the cluster
```bash
sudo cephadm shell -- ceph orch daemon add osd ceph1:/dev/sdb
```

### Verify OSD deployment status
#### Check OSD tree to confirm new OSD is registered
```bash
sudo cephadm shell -- ceph osd tree
```
#### List running OSD daemon processes
```
sudo cephadm shell -- ceph orch ps --daemon-type osd
```
#### Check detailed OSD status and utilization
```
sudo cephadm shell -- ceph osd df
```

#### 📋 Expected Successful Output:
```
ID  CLASS  WEIGHT   TYPE NAME      STATUS  REWEIGHT  PRI-AFF
 0    hdd  0.09770      osd.0        up   1.00000  1.00000
```

---

## 🧪 Step 6: Verify Cluster Health and Functionality

### Check overall cluster health status
#### Display real-time cluster health and status summary
```bash
sudo cephadm shell -- ceph -s
```

#### ✅ Healthy Cluster Output Should Show:
```
  cluster:
    id:     <unique-cluster-id>
    health: HEALTH_OK  # or HEALTH_WARN for single-node (expected)
  services:
    mon: 1 daemons, quorum ceph1
    mgr: ceph1.active
    osd: 1 osds: 1 up, 1 in
  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 100 GiB / 100 GiB avail
    pgs:     1 active+clean
```

### Check storage pool usage and capacity
#### Display detailed storage usage across all pools
```bash
sudo cephadm shell -- ceph df
```
#### Check raw capacity and utilization per OSD
```
sudo cephadm shell -- ceph osd df tree
```

### (Optional) Create a test RBD pool and image for validation
#### Create a new RBD pool named 'test_pool'
```bash
sudo cephadm shell -- ceph osd pool create test_pool
```
#### Initialize the pool for RBD usage
```
sudo cephadm shell -- rbd pool init test_pool
```
#### Create a 1GB test image in the pool
```
sudo cephadm shell -- rbd create test_pool/my_test_image --size 1G
```
#### List images to confirm creation
```
sudo cephadm shell -- rbd ls test_pool
```

---

## 🌐 Step 7: Access Ceph Dashboard (Optional but Recommended)

### Retrieve Dashboard admin credentials
#### Get the auto-generated admin username and password for Dashboard
```bash
sudo cephadm shell -- ceph dashboard get-login-credentials
```

#### 📋 Sample Output:
```
Dashboard username: admin
Dashboard password: <auto-generated-password>
```

### Access the Dashboard via Web Browser

```
URL: https://192.168.68.180:8443
Username: admin
Password: <retrieved-from-previous-command>
```

> ⚠️ **Browser Warning:** You will see a "Self-signed certificate" warning. This is normal for initial setup. Click "Advanced" → "Proceed to site" to continue.

### (Optional) Reset Dashboard password if needed
#### Set a new custom password for the admin user
```bash
sudo cephadm shell -- ceph dashboard ac-user-set-password admin
```
> You will be prompted to enter the new password interactively
---

## 🛠️ Troubleshooting and Common Issues

### Issue: /dev/sdb not showing as "Available" in device list
#### Check if disk has existing partitions or filesystems
```bash
lsblk /dev/sdb
sudo blkid /dev/sdb
```
#### If disk was previously used, wipe existing metadata (CAUTION: destroys all data)
```
sudo cephadm shell -- ceph orch device zap ceph1 /dev/sdb --destroy
```
#### Re-scan devices after zap operation
```
sudo cephadm shell -- ceph orch device ls
```

### Issue: Cluster shows HEALTH_WARN after bootstrap
#### Check detailed health warnings
```bash
sudo cephadm shell -- ceph health detail
```
#### Common single-node warning: insufficient OSDs for replica=2
#### This is expected with only 1 OSD. Add more OSDs or adjust pool size:
```
sudo cephadm shell -- ceph osd pool set test_pool size 1 --yes-i-really-mean-it
```

### Issue: Docker permission denied errors
#### Ensure current user is in docker group and re-login
```bash
sudo usermod -aG docker $USER
```
#### Log out and log back in, or run:
```
newgrp docker
```
#### Verify docker access without sudo
```
docker ps
```

### Issue: Firewall blocking Ceph ports
#### Allow essential Ceph ports through UFW firewall
```bash
sudo ufw allow 6789/tcp    # Monitor port
sudo ufw allow 3300/tcp    # Manager port
sudo ufw allow 6800:7300/tcp  # OSD, MDS, RGW ports
sudo ufw allow 6800:7300/udp  # Bluestore messaging
sudo ufw allow 8443/tcp    # Dashboard HTTPS
```
#### Enable firewall if not already active
```
sudo ufw enable
```

### Issue: Time synchronization warnings
#### Check chrony synchronization status
```bash
chronyc tracking
```
#### Force immediate time sync if needed
```
sudo chronyc -a makestep
```

---

## 📋 Quick Reference Command Sheet

### Cluster Management
#### View cluster status summary
```bash
sudo cephadm shell -- ceph -s
```
#### View detailed health information
```
sudo cephadm shell -- ceph health detail
```
#### Restart a specific daemon (e.g., osd.0)
```
sudo cephadm shell -- ceph orch restart osd.0
```
#### View logs for a specific daemon
```
sudo cephadm shell -- ceph orch logs osd.0
```

### OSD Management
#### List all storage devices and availability
```bash
sudo cephadm shell -- ceph orch device ls
```
#### Add a new OSD from available disk
```
sudo cephadm shell -- ceph orch daemon add osd ceph1:/dev/sdc
```
#### Remove an OSD (replace X with OSD ID)
```
sudo cephadm shell -- ceph orch daemon rm osd.X
```
#### Wipe a disk for reuse (DESTROYS DATA)
```
sudo cephadm shell -- ceph orch device zap ceph1 /dev/sdc --destroy
```

### Pool and RBD Management

#### List all storage pools
```bash
sudo cephadm shell -- ceph osd pool ls
```
#### Create a new RBD pool
```
sudo cephadm shell -- ceph osd pool create my_pool
```
#### Initialize pool for RBD
```
sudo cephadm shell -- rbd pool init my_pool
```
#### Create an RBD image
```
sudo cephadm shell -- rbd create my_pool/my_image --size 10G
```
#### Map RBD image to local block device (requires rbd-nbd package)
```
sudo cephadm shell -- rbd map my_pool/my_image
```

### Dashboard Management

#### Get dashboard login credentials
```bash
sudo cephadm shell -- ceph dashboard get-login-credentials
```
#### Enable or disable dashboard module
```
sudo cephadm shell -- ceph mgr module enable dashboard
sudo cephadm shell -- ceph mgr module disable dashboard
```
#### Change dashboard port (default: 8443)
```
sudo cephadm shell -- ceph config set mgr mgr/dashboard/server_port 9443
```

---

## 🎯 Final Verification Checklist

- [ ] `ceph -s` shows `HEALTH_OK` or expected `HEALTH_WARN` for single-node
- [ ] At least 1 OSD is `up` and `in` in `ceph osd tree`
- [ ] Dashboard is accessible at `https://192.168.68.180:8443`
- [ ] `/dev/sdb` is fully utilized as OSD with no manual partitioning
- [ ] All Ceph commands execute successfully via `cephadm shell`

---

> 🚀 **Congratulations!** Your single-node Ceph cluster is now operational using cephadm + Docker.  
> You can now proceed to:  
> • Create RBD images for VM storage  
> • Configure CephFS for shared filesystem access  
> • Integrate with OpenStack, Kubernetes, or Proxmox  
> • Scale horizontally by adding more nodes using `ceph orch host add`

For any issues, check logs with `cephadm shell -- ceph orch logs <daemon>` or consult the official docs: https://docs.ceph.com/en/latest/

Happy Ceph-ing! 🐘✨