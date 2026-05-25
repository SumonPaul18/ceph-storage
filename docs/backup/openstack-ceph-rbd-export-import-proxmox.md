# OpenStack (Ceph) to Proxmox VE Migration: The Complete Practical Guide

## Table of Contents
1. [Pre-Migration Preparation: Enabling Password Authentication](#1-pre-migration-preparation-enabling-password-authentication)
2. [Environment Setup and Tool Verification](#2-environment-setup-and-tool-verification)
3. [Exporting OpenStack Instance from Ceph Cluster](#3-exporting-openstack-instance-from-ceph-cluster)
4. [Optimizing Disk Images for Migration](#4-optimizing-disk-images-for-migration)
5. [Importing and Configuring VM in Proxmox VE](#5-importing-and-configuring-vm-in-proxmox-ve)
6. [Post-Migration Network and Service Configuration](#6-post-migration-network-and-service-configuration)
7. [Verification, Maintenance, and Cleanup](#7-verification-maintenance-and-cleanup)

---

## 1. Pre-Migration Preparation: Enabling Password Authentication

**The Story:** Imagine you have successfully migrated your Virtual Machine (VM) from OpenStack to Proxmox VE. The VM boots up, the services are running, but when you try to log in via SSH or the Console, you realize you don't have the SSH key pair that was used in OpenStack, or the cloud-init configuration has stripped away password access. You are locked out.

In OpenStack, instances are typically launched using **SSH Key Pairs** and **Cloud-Init**. Cloud-Init often disables root password login for security (`PasswordAuthentication no`). When you migrate this disk to Proxmox, Cloud-Init might not run again (or might fail because the OpenStack metadata service is gone), leaving you with a VM that accepts no passwords.

**The Solution:** Before exporting the disk, you **MUST** enable password-based authentication and set a known root password inside the OpenStack instance. This ensures you can log in after migration.

### Step 1: Access the OpenStack Instance
Log in to your OpenStack instance using your existing SSH key or via the Horizon Dashboard Console.

```bash
# SSH into the instance from your local machine or jump host
ssh -i /path/to/your-key.pem ubuntu@<OPENSTACK_VM_IP>
```
*   **Command:** `ssh -i /path/to/your-key.pem ubuntu@<OPENSTACK_VM_IP>`
*   **Explanation:** Connects to the VM using the private key. Replace `ubuntu` with your specific user (e.g., `centos`, `root`).
*   **Flags/Args:**
    *   `-i`: Specifies the identity file (private key).
    *   `ubuntu@<OPENSTACK_VM_IP>`: The username and IP address of the source VM.

### Step 2: Set a Root Password
You need a strong password for the root user to ensure secure access after migration.

```bash
# Switch to root user
sudo -i

# Set a new password for root
passwd root
```
*   **Command:** `passwd root`
*   **Explanation:** Prompts you to enter and confirm a new password for the root account.
*   **Flags/Args:**
    *   `root`: The user account to modify.

### Step 3: Enable Password Authentication in SSHD
By default, many cloud images disable password login. We must enable it.

```bash
# Edit the SSH daemon configuration file
nano /etc/ssh/sshd_config
```
*   **Command:** `nano /etc/ssh/sshd_config`
*   **Explanation:** Opens the SSH configuration file in the Nano text editor.
*   **Flags/Args:**
    *   `/etc/ssh/sshd_config`: The path to the SSH daemon configuration file.

**Action inside the file:**
Find the line `#PasswordAuthentication yes` or `PasswordAuthentication no`.
Change it to:
```text
PasswordAuthentication yes
PermitRootLogin yes
```

```bash
# Save and exit Nano (Ctrl+O, Enter, Ctrl+X)

# Restart the SSH service to apply changes
systemctl restart sshd
```
*   **Command:** `systemctl restart sshd`
*   **Explanation:** Restarts the SSH daemon so the new configuration takes effect immediately.
*   **Flags/Args:**
    *   `restart`: Stops and starts the service.
    *   `sshd`: The name of the SSH service.

### Step 4: Disable Cloud-Init (Optional but Recommended)
To prevent Cloud-Init from resetting your network or SSH configs on the next boot in Proxmox, it is safer to disable it.

```bash
# Create a file to disable cloud-init
touch /etc/cloud/cloud-init.disabled
```
*   **Command:** `touch /etc/cloud/cloud-init.disabled`
*   **Explanation:** Creates an empty file that signals Cloud-Init to skip execution on boot.
*   **Flags/Args:**
    *   `/etc/cloud/cloud-init.disabled`: The standard flag file for disabling cloud-init.

### Step 5: Verify Connectivity
Before proceeding, test if you can log in with the password.

```bash
# Exit the current session
exit

# Try logging in with password (from a new terminal)
ssh root@<OPENSTACK_VM_IP>
```
*   **Command:** `ssh root@<OPENSTACK_VM_IP>`
*   **Explanation:** Tests if password authentication is working. You should be prompted for the password you set.

---

## 2. Environment Setup and Tool Verification

Now that the VM is prepared, we move to the **Ceph Cluster** node (or a client node with Ceph access) to perform the export.

### Requirements
*   Access to a node with `ceph-common` installed.
*   Network connectivity to the Ceph Public and Cluster networks.
*   Sufficient storage space in `/backup` or `/tmp` to hold the raw image.

### Step 1: Verify Ceph Cluster Health
Always check the cluster health before performing I/O intensive operations.

```bash
# Check Ceph cluster status
ceph -s
```
*   **Command:** `ceph -s`
*   **Explanation:** Displays the overall health of the Ceph cluster. Ensure it says `HEALTH_OK`.
*   **Flags/Args:**
    *   `-s`: Short status output.

### Step 2: Identify the Correct Pool and Image Name
OpenStack uses specific pools for different types of storage.
*   `vms`: Typically stores ephemeral disks (local to compute node).
*   `volumes`: Stores Cinder volumes (block storage).
*   `images`: Stores Glance images (templates).

Since you are migrating a running instance's disk, it is likely in the `vms` pool.

```bash
# List images in the 'vms' pool
rbd ls vms
```
*   **Command:** `rbd ls vms`
*   **Explanation:** Lists all RBD images in the `vms` pool. Look for the UUID of your instance.
*   **Flags/Args:**
    *   `vms`: The name of the Ceph pool.

```bash
# List images in the 'volumes' pool (if using Cinder boot)
rbd ls volumes
```
*   **Command:** `rbd ls volumes`
*   **Explanation:** Checks the `volumes` pool if your instance was booted from a Cinder volume.

```bash
# Get detailed info about the specific image
# Replace <UUID>_disk with your actual image name
rbd info vms/4d73b7d5-41e6-4475-931a-1d59d53b6a44_disk
```
*   **Command:** `rbd info vms/4d73b7d5-41e6-4475-931a-1d59d53b6a44_disk`
*   **Explanation:** Shows the size, format, and features of the RBD image. Verify the size matches your expectation.
*   **Flags/Args:**
    *   `vms/...`: The pool and image name.

### Step 3: Prepare Backup Directory
Create a dedicated directory for the backup to keep things organized.

```bash
# Create a backup directory
mkdir -p /backup
```
*   **Command:** `mkdir -p /backup`
*   **Explanation:** Creates the `/backup` directory. `-p` ensures no error if it already exists.
*   **Flags/Args:**
    *   `-p`: Parents; creates intermediate directories as needed.

---

## 3. Exporting OpenStack Instance from Ceph Cluster

We will export the RBD image to a local file. We have two options: export directly to the final destination or export to a temporary location first.

### Option A: Direct Export to Backup Directory (Recommended for Space Efficiency)

```bash
# Export the RBD image to a raw file in /backup
rbd export vms/4d73b7d5-41e6-4475-931a-1d59d53b6a44_disk /backup/instance-backup.raw
```
*   **Command:** `rbd export vms/... /backup/instance-backup.raw`
*   **Explanation:** Reads the data from the Ceph RBD image and writes it to a local raw file. This file will be the same size as the virtual disk (e.g., 20GB).
*   **Flags/Args:**
    *   `vms/...`: Source RBD image.
    *   `/backup/instance-backup.raw`: Destination file path.

### Option B: Export to Temp and Move (If /backup is on a slower disk)

```bash
# Export to /tmp (usually faster local SSD)
rbd export vms/4d73b7d5-41e6-4475-931a-1d59d53b6a44_disk /tmp/instance.raw
```
*   **Command:** `rbd export vms/... /tmp/instance.raw`
*   **Explanation:** Exports the image to `/tmp`. Useful if `/tmp` is on a high-speed NVMe drive.

```bash
# Verify the file exists and check size
ls -lh /tmp/instance.raw
```
*   **Command:** `ls -lh /tmp/instance.raw`
*   **Explanation:** Lists the file with human-readable size.
*   **Flags/Args:**
    *   `-l`: Long listing format.
    *   `-h`: Human-readable sizes (KB, MB, GB).

---

## 4. Optimizing Disk Images for Migration

Raw files are large and inefficient for transfer. We convert them to **QCOW2** (QEMU Copy On Write), which is the native format for KVM/Proxmox. QCOW2 supports compression, thin provisioning, and snapshots.

### Step 1: Convert Raw to QCOW2 with Compression

```bash
# Convert raw image to compressed QCOW2
qemu-img convert -f raw -O qcow2 -p -c /tmp/instance.raw /backup/instance.qcow2
```
*   **Command:** `qemu-img convert -f raw -O qcow2 -p -c /tmp/instance.raw /backup/instance.qcow2`
*   **Explanation:** Converts the raw disk image to QCOW2 format.
*   **Flags/Args:**
    *   `-f raw`: Specifies the input format is RAW.
    *   `-O qcow2`: Specifies the output format is QCOW2.
    *   `-p`: Shows progress bar during conversion.
    *   `-c`: Compresses the output file. This significantly reduces file size for transfer.
    *   `/tmp/instance.raw`: Input file.
    *   `/backup/instance.qcow2`: Output file.

### Step 2: Verify the Converted Image

```bash
# Check the new QCOW2 file details
qemu-img info /backup/instance.qcow2
```
*   **Command:** `qemu-img info /backup/instance.qcow2`
*   **Explanation:** Displays information about the QCOW2 file, including virtual size, disk size (actual usage), and cluster size.
*   **Flags/Args:**
    *   `/backup/instance.qcow2`: Path to the image file.

```bash
# Compare file sizes
ls -lh /backup/instance-backup.raw /backup/instance.qcow2
```
*   **Command:** `ls -lh /backup/instance-backup.raw /backup/instance.qcow2`
*   **Explanation:** Shows both files side-by-side. You will notice `instance.qcow2` is much smaller due to compression and thin provisioning.

---

## 5. Importing and Configuring VM in Proxmox VE

Now we move the `instance.qcow2` file to your Proxmox VE server. You can use `scp` or the Proxmox Web UI.

### Step 1: Upload Image to Proxmox VE

**Method A: Using SCP (Command Line)**
```bash
# Copy the QCOW2 file to the Proxmox host
scp /backup/instance.qcow2 root@<PROXMOX_IP>:/var/lib/vz/images/
```
*   **Command:** `scp /backup/instance.qcow2 root@<PROXMOX_IP>:/var/lib/vz/images/`
*   **Explanation:** Securely copies the file to the Proxmox local storage directory.
*   **Flags/Args:**
    *   `root@<PROXMOX_IP>`: Root user on the Proxmox host.
    *   `/var/lib/vz/images/`: Default directory for local storage images.

**Method B: Using Proxmox Web UI**
1.  Log in to Proxmox VE Web Interface.
2.  Select your Node (e.g., `pve`).
3.  Go to **Local (pve)** -> **Content**.
4.  Click **Upload**.
5.  Select **Content Type**: `Disk image`.
6.  Select the `instance.qcow2` file from your computer.
7.  Click **Upload**.

### Step 2: Create a New VM Shell

We create a VM container without a disk first.

1.  Click **Create VM** in the top right corner.
2.  **General Tab:**
    *   **VM ID:** Choose an unused ID (e.g., `101`).
    *   **Name:** `Migrated-OpenStack-VM`.
3.  **OS Tab:**
    *   **Guest OS:** Linux.
    *   **Version:** Select the appropriate version (e.g., `Ubuntu 20.04/22.04` or `Other Linux`).
4.  **System Tab:**
    *   **Graphic card:** Default (SPICE) or VirtIO-GPU.
    *   **Machine:** `q35` (Modern) or `i440fx` (Legacy). `q35` is recommended for newer OS.
    *   **SCSI Controller:** `VirtIO SCSI` (Best performance).
5.  **Disks Tab:**
    *   **Do NOT add a disk here.** We will import it later.
6.  **CPU/Memory/Network:**
    *   Configure CPU cores and RAM to match or exceed the original OpenStack instance.
    *   **Network:** Select `VirtIO` model for best performance. Bridge to `vmbr0` (or your main bridge).
7.  Click **Finish**.

### Step 3: Import the Disk to the VM

Now we attach the uploaded QCOW2 file to the VM.

**Using Web UI:**
1.  Select the new VM (e.g., `101`).
2.  Go to **Hardware** tab.
3.  Click **Add** -> **Hard Disk**.
4.  **Bus/Device:** `SCSI` (matches the controller we selected).
5.  **Storage:** `local-lvm` (or your desired storage).
6.  **Disk size:** It doesn't matter, we will overwrite it.
7.  Click **Add**.
8.  Now, go to **Shell** of the Proxmox Node (not the VM).
9.  Run the following command to import the disk:

```bash
# Import the disk image to the VM
# Replace 101 with your VM ID
qm importdisk 101 /var/lib/vz/images/instance.qcow2 local-lvm --format qcow2
```
*   **Command:** `qm importdisk 101 /var/lib/vz/images/instance.qcow2 local-lvm --format qcow2`
*   **Explanation:** Imports the QCOW2 file into the Proxmox storage and attaches it to VM 101 as an unused disk.
*   **Flags/Args:**
    *   `101`: The VM ID.
    *   `/var/lib/vz/images/instance.qcow2`: Path to the uploaded file.
    *   `local-lvm`: Target storage pool.
    *   `--format qcow2`: Keeps the format as QCOW2.

10. Go back to the VM's **Hardware** tab.
11. You will see an **Unused Disk 0**. Double-click it.
12. Click **Add**. This attaches the disk to the VM.
13. Ensure the **Boot Order** in **Options** tab includes this SCSI disk.

---

## 6. Post-Migration Network and Service Configuration

When you start the VM, it will likely fail to get an IP address because the network interface name or MAC address has changed. OpenStack uses `eth0` or `ens3`, while Proxmox might assign `ens18` or similar. Also, Cloud-Init is disabled, so static IP configuration is required.

### Step 1: Start the VM and Access Console

1.  Start the VM from the Proxmox Web UI.
2.  Click **Console** to open the VNC/SPICE console.
3.  Log in with `root` and the password you set in **Section 1**.

### Step 2: Identify the New Network Interface

```bash
# List all network interfaces
ip addr show
```
*   **Command:** `ip addr show`
*   **Explanation:** Displays all network interfaces and their IP addresses. Look for an interface that is `UP` but has no IP, or check `dmesg | grep eth` to see detected interfaces.
*   **Flags/Args:** None.

### Step 3: Configure Static IP (Ubuntu/Netplan Example)

Most modern Ubuntu versions use Netplan.

```bash
# List netplan configuration files
ls /etc/netplan/
```
*   **Command:** `ls /etc/netplan/`
*   **Explanation:** Shows the YAML configuration files for networking.

```bash
# Edit the netplan config (replace filename with yours)
nano /etc/netplan/00-installer-config.yaml
```
*   **Command:** `nano /etc/netplan/00-installer-config.yaml`
*   **Explanation:** Opens the network configuration file.

**Update the file content:**
```yaml
network:
  version: 2
  ethernets:
    ens18:  # Replace with your actual interface name from 'ip addr'
      dhcp4: no
      addresses:
        - 192.168.1.100/24  # Your desired static IP
      gateway4: 192.168.1.1  # Your gateway
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
```

```bash
# Apply the new network configuration
netplan apply
```
*   **Command:** `netplan apply`
*   **Explanation:** Applies the new network settings immediately.
*   **Flags/Args:** None.

### Step 4: Configure Static IP (CentOS/RHEL Example)

```bash
# Navigate to network scripts
cd /etc/sysconfig/network-scripts/

# Edit the interface config (replace ifcfg-ens18 with your interface)
nano ifcfg-ens18
```
*   **Command:** `nano ifcfg-ens18`
*   **Explanation:** Edits the interface configuration file.

**Update the file content:**
```text
DEVICE=ens18
BOOTPROTO=static
ONBOOT=yes
IPADDR=192.168.1.100
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
DNS1=8.8.8.8
```

```bash
# Restart network service
systemctl restart network
```
*   **Command:** `systemctl restart network`
*   **Explanation:** Restarts the network service to apply changes.

### Step 5: Verify Connectivity

```bash
# Ping the gateway
ping -c 4 192.168.1.1

# Ping an external site
ping -c 4 google.com
```
*   **Command:** `ping -c 4 google.com`
*   **Explanation:** Tests internet connectivity.
*   **Flags/Args:**
    *   `-c 4`: Sends 4 packets and stops.

---

## 7. Verification, Maintenance, and Cleanup

### Step 1: Verify Services

Check if your application services (Apache, Nginx, MySQL, etc.) are running.

```bash
# Check status of all services
systemctl list-units --type=service --state=running

# Check specific service (e.g., apache2)
systemctl status apache2
```
*   **Command:** `systemctl status apache2`
*   **Explanation:** Shows the status of the Apache web server.
*   **Flags/Args:**
    *   `apache2`: Name of the service.

### Step 2: Install QEMU Guest Agent in Proxmox

For better integration with Proxmox (shutdown, freeze, IP reporting), install the guest agent.

```bash
# Install qemu-guest-agent
apt update && apt install -y qemu-guest-agent

# Enable and start the service
systemctl enable --now qemu-guest-agent
```
*   **Command:** `systemctl enable --now qemu-guest-agent`
*   **Explanation:** Ensures the agent starts on boot and runs now.

**In Proxmox Web UI:**
1.  Go to VM **Options**.
2.  Set **QEMU Guest Agent** to **Enabled**.
3.  Reboot the VM.

### Step 3: Cleanup Old Files

Once the migration is verified and successful, clean up the temporary files on the Ceph cluster and Proxmox host.

```bash
# Remove raw backup file from Ceph node
rm /backup/instance-backup.raw

# Remove temporary raw file
rm /tmp/instance.raw

# Remove the uploaded QCOW2 from Proxmox if no longer needed as a template
rm /var/lib/vz/images/instance.qcow2
```
*   **Command:** `rm /backup/instance-backup.raw`
*   **Explanation:** Deletes the large raw backup file to free up space.
*   **Flags/Args:**
    *   `/backup/instance-backup.raw`: File to delete.

### Step 4: Final Verification in Proxmox

```bash
# Check VM status from Proxmox shell
qm status 101
```
*   **Command:** `qm status 101`
*   **Explanation:** Confirms the VM is running.
*   **Flags/Args:**
    *   `101`: VM ID.

---

## Summary of the Journey

1.  **Preparation:** We enabled password auth and disabled Cloud-Init in OpenStack to ensure access after migration.
2.  **Export:** We identified the RBD image in Ceph and exported it to a raw file.
3.  **Conversion:** We converted the raw file to compressed QCOW2 for efficiency.
4.  **Import:** We uploaded the QCOW2 to Proxmox and attached it to a new VM shell.
5.  **Configuration:** We fixed the network settings manually since Cloud-Init was disabled.
6.  **Verification:** We confirmed services were running and installed the QEMU Guest Agent for better management.

This guide provides a robust, manual method for migrating OpenStack instances to Proxmox VE. For large-scale migrations, consider automating these steps with tools like `virtnbdbackup` or custom Python scripts using the Proxmox API.