# Ceph OpenStack to Proxmox VE Migration & Disaster Recovery Guide

## Table of Contents

1.  [Prerequisites & Environment Setup](#1-prerequisites--environment-setup)
2.  [Phase 1: Pre-Migration Preparation](#2-phase-1-pre-migration-preparation-openStack-side)
3.  **Phase 2: Data Export from Ceph Cluster**
4.  **Phase 3: Image Transfer & Optimization**
5.  **Phase 4: VM Creation & Disk Import in Proxmox VE**
6.  **Phase 5: Post-Migration Configuration & Network Setup**
7.  **Phase 6: Disaster Recovery Testing (Backup/Delete/Restore)**
8.  **Maintenance, Cleanup & Best Practices**

---

## 1. Prerequisites & Environment Setup

Before starting the migration, ensure you have the necessary access and tools installed on both the Source (OpenStack/Ceph) and Destination (Proxmox VE) environments. This guide assumes you are migrating a Linux-based instance.

### Requirements
*   **Source Side (OpenStack/Ceph):**
    *   Root or Sudo access to a machine with Ceph CLI tools (`ceph`, `rbd`).
    *   Access to OpenStack CLI (`openstack`) for metadata verification.
    *   Network connectivity to the Ceph Monitor nodes.
*   **Destination Side (Proxmox VE):**
    *   Proxmox VE 8.x installed.
    *   Root access to Proxmox Host via SSH or Web UI.
    *   Sufficient storage space in `local` (Directory) or `local-lvm` to hold the imported disk.
*   **Workstation:**
    *   A temporary workspace with enough disk space to hold the raw/qcow2 image during conversion.

### Tool Verification

Verify that the necessary CLI tools are installed and working.

#### Check Ceph RBD Tools Version
**Purpose:** Ensure Ceph block device tools are available for exporting images.
```bash
rbd --version
```
*   `--version`: Displays the version of the librbd tool. Ensure it matches your Ceph cluster version for compatibility.

#### Check QEMU Image Tools Version
**Purpose:** Verify tools are available for format conversion (Raw to QCOW2).
```bash
qemu-img --version
```
*   `--version`: Displays the version of qemu-img utility. Required for compression and format conversion.

#### Check Disk Space Availability
**Purpose:** Ensure you have enough free space for the export and conversion process.
```bash
df -h /tmp
df -h /backup
```
*   `-h`: Human-readable output (GB/MB).
*   `/tmp` or `/backup`: The directory where you plan to store the temporary files.

---

## 2. Phase 1: Pre-Migration Preparation (OpenStack Side)

**Critical Step:** Cloud images in OpenStack often disable password authentication by default, relying solely on SSH keys. If you do not have the original private key, you **must** enable password authentication inside the instance before exporting. Otherwise, you will be locked out after migration.

### Step 2.1: Access the Instance

Log in to the OpenStack instance using your existing SSH key or Console access.

#### SSH into Instance
**Purpose:** Gain access to the OS to modify SSH configuration.
```bash
ssh -i <private-key.pem> <username>@<instance-floating-ip>
```
*   `-i <private-key.pem>`: Path to your SSH private key.
*   `<username>`: Default user (e.g., `ubuntu`, `centos`, `cloud-user`).
*   `<instance-floating-ip>`: The public IP of the instance.

### Step 2.2: Enable Password Authentication

Modify the SSH daemon configuration to allow password logins.

#### Edit SSH Configuration File
**Purpose:** Open the `sshd_config` file for editing.
```bash
sudo nano /etc/ssh/sshd_config
```
*   `sudo`: Execute with root privileges.
*   `nano`: Text editor. You can also use `vi` or `vim`.

#### Modify PasswordAuthentication Directive
**Purpose:** Change the setting to allow passwords.
*   Find the line: `PasswordAuthentication no`
*   Change it to: `PasswordAuthentication yes`
*   Remove the `#` if the line is commented out.

#### Modify PermitRootLogin (Optional but Recommended for Migration)
**Purpose:** Allow root login for easier troubleshooting post-migration.
*   Find the line: `PermitRootLogin prohibit-password`
*   Change it to: `PermitRootLogin yes`

#### Save and Exit
*   In Nano: Press `Ctrl + O`, `Enter` to save, then `Ctrl + X` to exit.

### Step 2.3: Set/Reset Root Password

Ensure you know the root password or set a new one.

#### Set Root Password
**Purpose:** Define a strong password for the root user.
```bash
sudo passwd root
```
*   `passwd root`: Command to change the root password.
*   Follow the prompts to enter and confirm the new password.

### Step 2.4: Restart SSH Service

Apply the configuration changes.

#### Restart SSH Daemon
**Purpose:** Reload the SSH service to apply the new settings.
```bash
sudo systemctl restart sshd
```
*   `systemctl restart sshd`: Restarts the SSH service. On some older systems, use `service ssh restart`.

### Step 2.5: Test Password Login

Before proceeding, verify that password login works.

#### Test SSH Login with Password
**Purpose:** Confirm you can log in using the password.
```bash
ssh root@<instance-floating-ip>
```
*   Enter the password you set in Step 2.3.
*   If successful, you are ready to export. If failed, re-check `sshd_config` and firewall rules.

---

## 3. Phase 2: Data Export from Ceph Cluster

This phase involves identifying the correct volume associated with the OpenStack instance and exporting it from the Ceph cluster. We will export as `raw` first for speed, then convert to `qcow2` for storage efficiency.

### Step 3.1: Identify Volume Details

You need to map the OpenStack Instance to the underlying Ceph RBD image.

#### List Ceph Pools
**Purpose:** Identify the pool where the VM disks are stored (usually `vms`, `volumes`, or `images`).
```bash
rbd ls vms
rbd ls volumes
rbd ls images
```
*   `ls`: Lists all RBD images in the specified pool.
*   Look for a UUID-like name that matches your OpenStack Volume ID.

#### Get Volume Info
**Purpose:** Verify the size and details of the specific disk.
```bash
rbd info vms/<volume-uuid>_disk
```
*   `info`: Displays detailed information about the RBD image.
*   `vms/<volume-uuid>_disk`: Replace with the actual pool and image name found in the previous step.
*   **Note:** Note the `size` field to ensure you have enough export space.

### Step 3.2: Prepare Export Directory

Create a dedicated directory for backups to keep things organized.

#### Create Backup Directory
**Purpose:** Create a folder to store exported images.
```bash
mkdir /backup
```
*   `mkdir`: Make directory command.
*   `/backup`: The path for the backup directory. Ensure this partition has enough free space.

### Step 3.3: Export RBD Image to Raw Format

Exporting directly to `qcow2` from Ceph can be slow. Exporting to `raw` is faster, and we will convert it later.

#### Export RBD to Raw File
**Purpose:** Copy the Ceph RBD image to a local raw file.
```bash
rbd export vms/<volume-uuid>_disk /backup/instance-backup.raw
```
*   `export`: Command to copy data from Ceph to a local file.
*   `vms/<volume-uuid>_disk`: Source RBD image.
*   `/backup/instance-backup.raw`: Destination local file path.
*   **Note:** This operation may take time depending on the disk size and network speed.

#### Verify Exported Raw File
**Purpose:** Check if the file was created and its size.
```bash
ls -lh /backup/instance-backup.raw
```
*   `-lh`: Long list format with human-readable sizes.
*   Compare the size with the `rbd info` output to ensure completeness.

---

## 4. Phase 3: Image Transfer & Optimization

Now that we have the raw image, we will convert it to `qcow2` with compression to save space and make transfer to Proxmox faster.

### Step 4.1: Move to Temporary Workspace (Optional)

If `/backup` is on a slow network mount, move the file to a local fast disk like `/tmp` for conversion.

#### Copy File to /tmp
**Purpose:** Move the raw file to a local high-speed directory for processing.
```bash
cp /backup/instance-backup.raw /tmp/instance.raw
```
*   `cp`: Copy command.
*   `/tmp/instance.raw`: Destination in temporary storage.

#### Verify Copy
**Purpose:** Ensure the file exists in `/tmp`.
```bash
ls -lh /tmp/instance.raw
```

### Step 4.2: Convert Raw to QCOW2 with Compression

Convert the raw image to qcow2 format. QCOW2 supports compression and sparse allocation, making it smaller and easier to upload.

#### Convert and Compress Image
**Purpose:** Convert raw format to qcow2 with compression enabled.
```bash
qemu-img convert -f raw -O qcow2 -p -c /tmp/instance.raw /backup/instance.qcow2
```
*   `convert`: Command to change disk format.
*   `-f raw`: Specifies the input format is RAW.
*   `-O qcow2`: Specifies the output format is QCOW2.
*   `-p`: Shows progress bar during conversion.
*   `-c`: Enables compression (reduces file size significantly for empty spaces).
*   `/tmp/instance.raw`: Input file.
*   `/backup/instance.qcow2`: Output file.

#### Verify QCOW2 File
**Purpose:** Check the final compressed file size.
```bash
ls -lh /backup/instance.qcow2
```
*   Compare this size with the `.raw` file. It should be significantly smaller if the disk had empty space.

#### Clean Up Temporary Files
**Purpose:** Remove the large raw file to free up space.
```bash
rm /tmp/instance.raw
ls /tmp/
```
*   `rm`: Remove file command.
*   `ls /tmp/`: Verify the raw file is deleted.

---

## 5. Phase 4: VM Creation & Disk Import in Proxmox VE

Proxmox VE allows importing disks via the Web UI or CLI. Here we describe the process using the Web UI "Import Disk" feature or manual upload followed by attachment.

### Step 5.1: Upload Image to Proxmox

Transfer the `instance.qcow2` file to the Proxmox host. You can use SCP or the Proxmox Web UI "Upload" button if available for ISOs (but for disks, SCP is better).

#### Transfer File via SCP
**Purpose:** Securely copy the qcow2 file to the Proxmox host.
```bash
scp /backup/instance.qcow2 root@<proxmox-ip>:/var/lib/vz/template/iso/
```
*   `root@<proxmox-ip>`: Proxmox host IP.
*   `/var/lib/vz/template/iso/`: A common storage location accessible by Proxmox. You can also use `/root/`.

### Step 5.2: Create New VM in Proxmox

Create a shell VM to attach the disk to.

#### Create VM via Web UI
1.  Log in to Proxmox Web UI.
2.  Click **Create VM**.
3.  **General:** Set VM ID (e.g., `100`) and Name.
4.  **OS:** Select "Do not use any media" (since we are attaching an existing disk).
5.  **System:** Keep defaults (Machine: i440fx or q35, BIOS: SeaBIOS/OVMF).
6.  **Disks:** Do not add a disk here.
7.  **CPU/Memory:** Match the resources from the OpenStack instance.
8.  **Network:** Add a VirtIO bridge (vmbr0).
9.  Finish creation.

### Step 5.3: Import/Attach the Disk

Use the CLI to import the uploaded qcow2 file into the VM's storage.

#### Import Disk via CLI
**Purpose:** Convert and import the qcow2 file into the Proxmox storage pool attached to the VM.
```bash
qm importdisk 100 /var/lib/vz/template/iso/instance.qcow2 local-lvm --format raw
```
*   `100`: The VM ID created in Step 5.2.
*   `/var/lib/vz/template/iso/instance.qcow2`: Path to the uploaded file.
*   `local-lvm`: The target storage pool (change to `ceph-pool` if using Ceph backend).
*   `--format raw`: Converts to raw during import for better performance on LVM/Ceph.

#### Attach the Unused Disk
**Purpose:** The import creates an "Unused Disk". You must attach it.
1.  Go to Proxmox Web UI -> VM `100` -> **Hardware**.
2.  Double-click **Unused Disk 0**.
3.  Click **Add**. It will become `scsi0` or `virtio0`.
4.  **Important:** Set the **Bus/Device** to `SCSI` or `VirtIO Block` depending on what the guest OS supports. Most cloud images support `VirtIO`.

### Step 5.4: Configure Boot Order

Ensure the VM boots from the imported disk.

#### Set Boot Order via CLI
**Purpose:** Define the boot priority.
```bash
qm set 100 --boot order=scsi0
```
*   `--boot order=scsi0`: Sets the SCSI disk as the first boot device. Adjust to `virtio0` if you used VirtIO.

---

## 6. Phase 5: Post-Migration Configuration & Network Setup

The VM is now running, but the network configuration from OpenStack (DHCP/Metadata) will not work in Proxmox. You must configure static IP or DHCP manually inside the VM.

### Step 6.1: Access VM Console

Log in to the VM using the Proxmox Console (NoVNC/SPICE) since the network is not yet configured.

#### Open Console
1.  Go to Proxmox Web UI -> VM `100` -> **Console**.
2.  Log in with `root` and the password you set in Phase 1.

### Step 6.2: Identify Network Interface

Find the name of the network interface.

#### Check Network Interfaces
**Purpose:** List all network devices.
```bash
ip link show
```
*   Look for interfaces like `eth0`, `ens18`, or `enp1s0`. In Proxmox with VirtIO, it is often `ens18` or `eth0`.

### Step 6.3: Configure Static IP (Example for Ubuntu/Netplan)

Most modern Linux distros use Netplan or NetworkManager.

#### Edit Netplan Configuration (Ubuntu 18.04+)
**Purpose:** Set a static IP address.
```bash
nano /etc/netplan/00-installer-config.yaml
```
*   Replace the content with:
    ```yaml
    network:
      version: 2
      ethernets:
        ens18:  # Replace with your interface name
          dhcp4: no
          addresses:
            - 192.168.1.100/24  # Your desired IP
          gateway4: 192.168.1.1
          nameservers:
              addresses: [8.8.8.8, 8.8.4.4]
    ```
*   Apply changes:
    ```bash
    netplan apply
    ```

### Step 6.4: Configure Static IP (Example for CentOS/RHEL)

#### Edit Interface Config
**Purpose:** Set static IP for RHEL-based systems.
```bash
nano /etc/sysconfig/network-scripts/ifcfg-ens18
```
*   Update the file:
    ```text
    TYPE=Ethernet
    BOOTPROTO=static
    NAME=ens18
    DEVICE=ens18
    ONBOOT=yes
    IPADDR=192.168.1.100
    NETMASK=255.255.255.0
    GATEWAY=192.168.1.1
    DNS1=8.8.8.8
    ```
*   Restart network:
    ```bash
    systemctl restart NetworkManager
    ```

### Step 6.5: Verify Network Connectivity

#### Ping Gateway and External IP
**Purpose:** Confirm internet and LAN access.
```bash
ping -c 4 192.168.1.1
ping -c 4 google.com
```
*   If ping works, your network is configured correctly.

---

## 7. Phase 6: Disaster Recovery Testing (Backup/Delete/Restore)

To ensure your backup is valid, perform a full restore test.

### Step 7.1: Verify Active Instance

Ensure the migrated VM is running and data is intact.

#### Check VM Status
**Purpose:** Confirm VM is running.
```bash
qm status 100
```

#### Verify Data Integrity
**Purpose:** SSH in and check critical files.
```bash
ssh root@192.168.1.100 "ls -l /home && df -h"
```

### Step 7.2: Simulate Failure (Delete Instance)

**Warning:** This deletes the VM. Ensure your `instance.qcow2` backup is safe.

#### Stop and Destroy VM
**Purpose:** Completely remove the VM.
```bash
qm stop 100
qm destroy 100 --purge
```
*   `--purge`: Removes all disks and config.

#### Verify Deletion
**Purpose:** Confirm VM is gone.
```bash
qm list | grep 100
```

### Step 7.3: Restore from Backup

#### Create New VM Shell
**Purpose:** Create a new VM container for restoration.
```bash
qm create 101 --name restored-vm --ostype l26 --memory 4096 --net0 virtio,bridge=vmbr0
```

#### Import Backup Disk
**Purpose:** Import the original qcow2 backup.
```bash
qm importdisk 101 /var/lib/vz/template/iso/instance.qcow2 local-lvm --format raw
```

#### Attach Disk and Set Boot
**Purpose:** Configure the restored disk.
```bash
qm set 101 --scsi0 local-lvm:vm-101-disk-0
qm set 101 --boot order=scsi0
```

### Step 7.4: Verify Restoration

#### Start Restored VM
**Purpose:** Power on the restored instance.
```bash
qm start 101
```

#### Validate Data
**Purpose:** SSH in and confirm data matches the pre-deletion state.
```bash
ssh root@<new-restored-ip> "cat /etc/hostname"
```

---

## 8. Maintenance, Cleanup & Best Practices

### Cleanup Temporary Files

Remove temporary files from both Source and Destination to save space.

#### Clean Up Ceph/Source
**Purpose:** Remove local export files if no longer needed.
```bash
rm /backup/instance-backup.raw
rm /backup/instance.qcow2
```

#### Clean Up Proxmox
**Purpose:** Remove the uploaded qcow2 file from `/var/lib/vz/template/iso/`.
```bash
rm /var/lib/vz/template/iso/instance.qcow2
```

### Best Practices

1.  **Always Enable Password Auth Before Export:** As demonstrated in Phase 1, this is the most common reason for lockouts.
2.  **Use QCOW2 for Transfer:** It is smaller and faster to upload than RAW.
3.  **Match Hardware Types:** If the OpenStack instance used VirtIO, use VirtIO in Proxmox. If it used IDE/SATA, match that to avoid boot failures.
4.  **Network Reconfiguration:** Never expect the old IP to work. Always plan for a new IP assignment and update DNS records accordingly.
5.  **Regular DR Drills:** Perform the Delete/Restore test quarterly to ensure backup integrity.

### Uninstalling/Removing Components

If you need to completely remove a failed migration attempt:

#### Remove VM and Disks
```bash
qm stop <vm-id>
qm destroy <vm-id> --purge
```

#### Remove Orphaned Disks (Manual Cleanup)
If `qm destroy` fails to remove a disk, find it manually:
```bash
# For LVM
lvremove /dev/pve/data/vm-<vm-id>-disk-0

# For Ceph
rbd rm -p <pool> vm-<vm-id>-disk-0
```

---

**End of Guide**