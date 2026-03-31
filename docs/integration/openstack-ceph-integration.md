# Practical Guide: Integrating Ceph Storage with OpenStack via Kolla-Ansible
### Real-World Implementation for Single-Node Kolla-Ansible + 3-Node Ceph Cluster
#### Reference:
- [github.com/filip-lebiecki/ceph](https://github.com/filip-lebiecki/ceph)
-[github.com/hojat-gazestani/openstack/](https://github.com/hojat-gazestani/openstack/tree/main/Ceph/first)

> **Infrastructure Context**
> - **OpenStack Node (All-in-One Kolla-Ansible)**: `192.168.68.69`
> - **Ceph Cluster (3 Nodes)**: Primary/MON access via `192.168.68.248`
> - **Goal**: Connect OpenStack services (Glance, Cinder, Nova) to external Ceph RBD storage using official Kolla-Ansible and Ceph documentation workflows.

---

## Table of Contents
1. [Pre-Integration Checklist & Requirements](#1-pre-integration-checklist--requirements)
2. [Ceph Cluster Side Preparation](#2-ceph-cluster-side-preparation)
3. [OpenStack Kolla-Ansible Side Configuration](#3-openstack-kolla-ansible-side-configuration)
4. [Service-by-Service Integration: Glance, Cinder, Nova](#4-service-by-service-integration-glance-cinder-nova)
5. [Verification & Testing Workflow](#5-verification--testing-workflow)
6. [Operations & Day-2 Management](#6-operations--day-2-management)
7. [Troubleshooting & Maintenance Practices](#7-troubleshooting--maintenance-practices)

---

## 1. Pre-Integration Checklist & Requirements

### 1.1 What You Need Before Starting
Think of this like preparing a kitchen before cooking a complex meal. You don't want to realize you're missing salt when the dish is halfway done.

| Requirement | Why It Matters | How to Verify |
|-------------|---------------|---------------|
| **Ceph Cluster Healthy** | OpenStack will fail to connect if Ceph is degraded or unreachable | `ceph -s` on any Ceph node → should show `HEALTH_OK` |
| **Network Connectivity** | OpenStack node must reach Ceph MONs on port 6789 and OSDs on 6800-7300 | `telnet 192.168.68.248 6789` from OpenStack node |
| **Time Synchronization** | Cephx authentication fails if clocks drift > 300 seconds | `chronyc tracking` or `ntpstat` on all nodes |
| **Root/Sudo Access** | Required for config file deployment and service restarts | `sudo -v` on both OpenStack and Ceph nodes |
| **Kolla-Ansible Installed** | You're using Kolla to deploy OpenStack, so it must be ready | `kolla-ansible --version` on `192.168.68.69` |
| **Ceph Client Packages** | OpenStack services need `ceph-common` and `python3-rbd` to talk to Ceph | Check availability via `apt search ceph-common` or `yum list ceph-common` |

### 1.2 Version Compatibility Check (Official Sources)
Always verify versions against official compatibility matrices. Here's how to do it practically:


#### On OpenStack node (192.168.68.69)
#### Check Kolla-Ansible version
```
kolla-ansible --version
```
#### Check OpenStack release (if already deployed)

```
openstack --version
```

#### On Ceph node (192.168.68.248)
#### Check Ceph version
```
ceph -v
```
#### Cross-reference with official docs:
#### - Kolla-Ansible 2025.2 supports Ceph Pacific, Quincy, Reef, Squid
#### - Ceph docs: https://docs.ceph.com/en/latest/rbd/rbd-openstack/


> 💡 **Pro Tip**: If your Ceph version is newer than what Kolla-Ansible explicitly lists, test in a lab first. Most newer Ceph versions work with older OpenStack, but the reverse isn't always true.

---

## 2. Ceph Cluster Side Preparation

This is where you prepare the "storage backend" that OpenStack will consume. Think of Ceph as a warehouse, and OpenStack as the delivery truck that needs proper access credentials and designated storage zones.

### 2.1 Create Dedicated Pools for OpenStack Services
Each OpenStack service gets its own pool for isolation and performance tuning.


#### SSH to any Ceph monitor node (e.g., 192.168.68.248)
```
ssh admin@192.168.68.248
```
#### Create pools (adjust PG count based on your OSD count - use ceph pg calc tool)
#### For a small 3-node cluster with ~10 OSDs total, 32 PGs per pool is safe
```
ceph osd pool create images 32 32
ceph osd pool create volumes 32 32
ceph osd pool create backups 32 32
ceph osd pool create vms 32 32
```
#### Enable the 'rbd' (RADOS Block Device) application on each pool.
```
ceph osd pool application enable volumes rbd
ceph osd pool application enable images rbd
ceph osd pool application enable vms rbd
ceph osd pool application enable backups rbd
```
#### Initialize pools for RBD usage (mandatory step for newer Ceph versions)
```
rbd pool init images
rbd pool init volumes
rbd pool init backups
rbd pool init vms
```
#### Verify pools exist
```
ceph osd pool ls detail
rbd pool ls
```

### 2.2 Create Ceph Users with Minimal Permissions (Security Best Practice)
Never use `client.admin` for OpenStack. Create service-specific users with least-privilege access.


#### Create user for Glance (images pool - read/write)
```
ceph auth get-or-create client.glance \
  mon 'profile rbd' \
  osd 'profile rbd pool=images' \
  mgr 'profile rbd pool=images'
```
#### Create user for Cinder (volumes, vms pools - read/write; images pool - read-only for cloning)
```
ceph auth get-or-create client.cinder \
  mon 'profile rbd' \
  osd 'profile rbd pool=volumes, profile rbd pool=vms, profile rbd-read-only pool=images' \
  mgr 'profile rbd pool=volumes, profile rbd pool=vms'
```
#### Create user for Cinder Backup (backups pool only)
```
ceph auth get-or-create client.cinder-backup \
  mon 'profile rbd' \
  osd 'profile rbd pool=backups' \
  mgr 'profile rbd pool=backups'
```
#### Optional: Create user for Nova ephemeral disks (if using nova_backend_ceph)
```
ceph auth get-or-create client.nova \
  mon 'profile rbd' \
  osd 'profile rbd pool=vms' \
  mgr 'profile rbd pool=vms'
```
#### Verify users created
```
ceph auth ls
```

### 2.3 Export Keyrings and Config for OpenStack Node
You'll copy these files to your Kolla-Ansible configuration directory.


#### On Ceph node, export ceph.conf (clean version - no tabs!)
```
ceph config generate-minimal-conf > /tmp/ceph.conf
```
#### Remove any leading tabs that break Kolla's ini parser (critical!)
```
sed -i 's/^\t/    /g' /tmp/ceph.conf
```
Ensure configuration files use spaces instead of tabs to prevent parsing errors.

```bash
sed -i 's/\t/  /g' /etc/kolla/config/glance/ceph.conf
```
*Replaces tab characters with two spaces in the Glance `ceph.conf` file.*

```bash
cat -A /etc/kolla/config/glance/ceph.conf
```
*Displays the file content with special characters visible to verify formatting.*


#### Export keyrings
```
ceph auth get client.cinder -o /tmp/ceph.client.cinder.keyring
ceph auth get client.glance -o /tmp/ceph.client.glance.keyring
ceph auth get client.nova -o /tmp/ceph.client.nova.keyring
ceph auth get client.cinder-backup -o /tmp/ceph.client.cinder-backup.keyring

```
#### Verifying keyring files
```
ls /tmp/
cat /tmp/ceph.client.cinder.keyring
```
#### Export keyrings
```
ceph auth get-key client.glance > /tmp/ceph.client.glance.keyring
ceph auth get-key client.cinder > /tmp/ceph.client.cinder.keyring
ceph auth get-key client.cinder-backup > /tmp/ceph.client.cinder-backup.keyring
ceph auth get-key client.nova > /tmp/ceph.client.nova.keyring
```
#### Copy files to OpenStack node (192.168.68.69)
```
scp /tmp/ceph.conf admin@192.168.68.69:/tmp/
scp /tmp/ceph.client.*.keyring admin@192.168.68.69:/tmp/
```

> ⚠️ **Critical Note**: The `ceph.conf` file must NOT have leading tabs. Kolla-Ansible uses an INI parser that breaks on tabs. Always sanitize with `sed` as shown above.

---

## 3. OpenStack Kolla-Ansible Side Configuration

Now you configure the "client side" - your Kolla-Ansible deployment to consume the Ceph storage you just prepared.

### 3.1 Prepare Kolla Configuration Directory Structure
Kolla-Ansible uses `/etc/kolla/` for global configs and `/etc/kolla/config/<service>/` for service-specific overrides.


#### On OpenStack node (192.168.68.69)
```
ssh admin@192.168.68.69
```
#### Create required directories for Ceph config injection
```
sudo mkdir -p /etc/kolla/config/glance
sudo mkdir -p /etc/kolla/config/cinder/cinder-volume
sudo mkdir -p /etc/kolla/config/cinder/cinder-backup
sudo mkdir -p /etc/kolla/config/nova
```
```
####  Check folder structure
tree /etc/kolla/config/
```
#### Copy the sanitized ceph.conf and keyrings from /tmp/ to appropriate locations
```
sudo cp /tmp/ceph.conf /etc/kolla/config/glance/ceph.conf
sudo cp /tmp/ceph.conf /etc/kolla/config/cinder/ceph.conf
sudo cp /tmp/ceph.conf /etc/kolla/config/nova/ceph.conf
```
#### Copy keyrings with correct naming and ownership expectations
```
sudo cp /tmp/ceph.client.glance.keyring /etc/kolla/config/glance/ceph.client.glance.keyring
sudo cp /tmp/ceph.client.cinder.keyring /etc/kolla/config/cinder/cinder-volume/ceph.client.cinder.keyring
sudo cp /tmp/ceph.client.cinder.keyring /etc/kolla/config/cinder/cinder-backup/ceph.client.cinder.keyring
sudo cp /tmp/ceph.client.cinder-backup.keyring /etc/kolla/config/cinder/cinder-backup/ceph.client.cinder-backup.keyring
sudo cp /tmp/ceph.client.cinder.keyring /etc/kolla/config/nova/ceph.client.cinder.keyring
```
#### If using nova backend:
```
sudo cp /tmp/ceph.client.nova.keyring /etc/kolla/config/nova/ceph.client.nova.keyring
```

### 3.2 Update Kolla globals.yml for Ceph Integration
Edit `/etc/kolla/globals.yml` to enable Ceph backends and define authentication parameters.

```
nano /etc/kolla/globals.yml
```

```yaml
# === Glance Ceph Backend ===
glance_backend_ceph: "yes"
ceph_glance_user: "glance"
ceph_glance_pool_name: "images"

# === Cinder Ceph Backend ===
# Must be need to disable `enable_cinder_backend_lvm`
#enable_cinder_backend_lvm: "no" 
cinder_backend_ceph: "yes"
ceph_cinder_user: "cinder"
ceph_cinder_pool_name: "volumes"
#ceph_cinder_backup_user: "cinder" no sure
ceph_cinder_backup_user: "cinder-backup"
ceph_cinder_backup_pool_name: "backups"
enable_cinder_backup: "yes"
cinder_backup_driver: "ceph"


# === Nova Ceph Backend (for ephemeral disks) ===
nova_backend_ceph: "yes"
ceph_nova_user: "nova"  # or "cinder" if reusing Cinder key
ceph_nova_pool_name: "vms"

# === Optional: Enable copy-on-write cloning in Glance ===
# Note: Only enable if your Glance API endpoint is not publicly exposed
#glance_api_extra_config: |
#  [DEFAULT]
#  show_image_direct_url = True
```

> 🔐 **Security Note**: `show_image_direct_url = True` exposes backend URLs via Glance API. Only enable this if your Glance endpoint is behind authentication/firewall.

### 3.3 Handle the "Storage Group" Requirement (Kolla-Specific Quirk)
Kolla-Ansible expects nodes in the `[storage]` group for Cinder services. If you're running all-in-one, add your node:

```ini
# Edit your Kolla inventory file (e.g., /etc/kolla/inventory)
[storage]
192.168.68.69

# Ensure this node is also in [control] and [compute] for all-in-one
[control]
192.168.68.69

[compute]
192.168.68.69
```

### 3.4 Deploy Configuration with Kolla-Ansible
Now apply the configuration. Kolla will inject the Ceph configs into containers and restart services.


#### Validate configuration first (always do this!)
```
kolla-ansible prechecks -i all-in-one
```
#### Bootstrap servers if not done already
```
kolla-ansible bootstrap-servers -i all-in-one
```
#### If only updating config (not full redeploy), use:
```
kolla-ansible reconfigure -i all-in-one 
```
#### Deploy/reconfigure OpenStack with new Ceph settings
```
kolla-ansible deploy -i all-in-one 
```


> ⏱️ **Expect Downtime**: Services like `cinder-volume` and `glance-api` will restart. Plan for a brief maintenance window.

### 3.5 Verify Services
Check if services are up and storage works.

```bash
openstack volume service list
```
> **Action:** Check Cinder service status (Should be `UP`).
---

## 4. Service-by-Service Integration: Glance, Cinder, Nova

### 4.1 Glance: Store Images as RBD Objects
After deployment, verify Glance can write to Ceph.


#### On OpenStack node, source admin credentials
```
source /etc/kolla/admin-openrc.sh
```
#### Create a test image in RAW format (QCOW2 not recommended for Ceph)
```
wget https://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img
openstack image create "cirros-ceph-test" \
  --file cirros-0.6.2-x86_64-disk.img \
  --disk-format raw \
  --container-format bare \
  --public
```
#### Verify image list
```bash
openstack image list
```
> **Action:** Confirm image exists.

#### Verify image is stored in Ceph
#### First, get the image ID
```
openstack image show cirros-ceph-test -c id -f value
```
#### Then check Ceph RBD list (on Ceph node)
```
ssh admin@192.168.68.248
rbd -p images ls
```
#### You should see an RBD image matching the Glance image ID


### 4.2 Cinder: Create Volumes from Ceph Pools
Test volume creation and attachment.


#### Create a Cinder volume type (optional but recommended)
```
openstack volume type create ceph-rbd
openstack volume type set --property volume_backend_name=ceph ceph-rbd
```
#### Create a 1GB volume from the 'volumes' pool
```
openstack volume create --size 1 --type ceph-rbd test-ceph-volume
```
#### Check volume status
```
openstack volume show test-ceph-volume
```
#### Verify in Ceph (on Ceph node)
```
rbd -p volumes ls
```
#### Should see an RBD image named like volume-<uuid>


### 4.3 Nova: Boot Instances with Ceph-Backed Disks
Test both ephemeral disk and boot-from-volume scenarios.


#### Scenario 1: Boot with ephemeral disk on Ceph (if nova_backend_ceph enabled)
```
openstack server create --flavor m1.tiny --image cirros-ceph-test \
  --key-name mykey test-ephemeral-ceph
```
#### Scenario 2: Boot from Cinder volume (copy-on-write clone from Glance image)
```
openstack volume create --size 2 --image cirros-ceph-test ceph-boot-volume
openstack server create --flavor m1.tiny --volume ceph-boot-volume \
  --key-name mykey test-boot-from-volume
```
#### Verify instance is running
```
openstack server list
```
#### Check Ceph for active RBD objects (on Ceph node)
#### for ephemeral disks
```
rbd -p vms ls    
```
#### for boot-from-volume
```
rbd -p volumes ls    
```

> 🚀 **Performance Note**: Boot-from-volume with Ceph enables copy-on-write cloning. New volumes are created instantly as metadata pointers, not full data copies.

---

## 5. Verification & Testing Workflow

### 5.1 End-to-End Functional Test
Create a realistic workload to validate the integration.


#### 1. Upload a RAW image to Glance (Ceph backend)

```
openstack image create "ubuntu-24.04" \
  --file ubuntu-24.04-server-cloudimg-amd64.img \
  --disk-format raw \
  --container-format bare \
  --public
```
#### 2. Create a Cinder volume from that image
```
openstack volume create --size 10 --image ubuntu-24.04 prod-volume
```
#### 3. Boot an instance from the volume
```
openstack server create --flavor m1.small --volume prod-volume \
  --network private --key-name production-key prod-vm
```
#### 4. Attach an additional Cinder volume to the running instance
```
openstack volume create --size 5 --type ceph-rbd data-volume
openstack server add volume prod-vm data-volume
```
#### 5. Verify all components in Ceph
```
ssh admin@192.168.68.248
```
#### Should still show HEALTH_OK
```
ceph -s  
```
```
rbd -p images ls
rbd -p volumes ls
rbd -p vms ls
```

### 5.2 Monitoring & Metrics Validation
Ensure you can observe Ceph usage from OpenStack perspective.


#### Check Cinder backend capabilities
```
openstack volume service list
openstack volume backend pool list  # Should show 'ceph@ceph#volumes'
```
#### Check Glance backend
```
openstack image show <image-id> -c stores -f value  # Should include 'rbd'
```
#### Monitor Ceph performance during OpenStack operations
#### On Ceph node, watch real-time I/O
```
ceph -w
```
#### Or use ceph dashboard if enabled


---

## 6. Operations & Day-2 Management

### 6.1 Adding a New Ceph Pool for OpenStack
Need a new pool for a new project or isolation requirement?


#### On Ceph node
```
ceph osd pool create newproject 32 32
rbd pool init newproject
```
#### Create dedicated user
```
ceph auth get-or-create client.newproject \
  mon 'profile rbd' \
  osd 'profile rbd pool=newproject' \
  mgr 'profile rbd pool=newproject'
```
#### Export and copy keyring/config to OpenStack node (as in Section 2.3)

#### Update Kolla globals.yml for the service using this pool
#### Example for a new Cinder backend:
```
cat >> /etc/kolla/globals.yml << 'EOF'
cinder_ceph_backends:
  - name: "default-ceph"
    cluster: "ceph"
    user: "cinder"
    pool: "volumes"
    enabled: true
  - name: "newproject-ceph"
    cluster: "ceph"
    user: "newproject"
    pool: "newproject"
    enabled: true
EOF
```
#### Reconfigure Cinder
```
kolla-ansible -i /etc/kolla/inventory reconfigure --limit cinder
```

### 6.2 Rotating Ceph Keys (Security Maintenance)
Periodically rotate service keys without downtime.


#### On Ceph node, generate new key for client.cinder
```
ceph auth get client.cinder > /tmp/cinder-old-key
ceph auth del client.cinder
ceph auth get-or-create client.cinder \
  mon 'profile rbd' \
  osd 'profile rbd pool=volumes, profile rbd pool=vms, profile rbd-read-only pool=images' \
  mgr 'profile rbd pool=volumes, profile rbd pool=vms'
```
#### Export new key
```
ceph auth get-key client.cinder > /tmp/ceph.client.cinder.keyring.new
```
#### Copy to OpenStack node and update Kolla config
```
scp /tmp/ceph.client.cinder.keyring.new admin@192.168.68.69:/tmp/
ssh admin@192.168.68.69
sudo cp /tmp/ceph.client.cinder.keyring.new /etc/kolla/config/cinder/cinder-volume/ceph.client.cinder.keyring
sudo cp /tmp/ceph.client.cinder.keyring.new /etc/kolla/config/cinder/cinder-backup/ceph.client.cinder.keyring
sudo cp /tmp/ceph.client.cinder.keyring.new /etc/kolla/config/nova/ceph.client.cinder.keyring
```
#### Reconfigure affected services
```
kolla-ansible -i /etc/kolla/inventory reconfigure --limit "cinder:nova"
```

### 6.3 Scaling Ceph Capacity
When your Ceph cluster grows, OpenStack automatically benefits - no reconfiguration needed.


#### Add new OSDs to Ceph cluster (using cephadm or ceph-ansible)
#### Example with cephadm:
```
ceph orch daemon add osd node4:/dev/sdb
```
#### Verify cluster rebalancing
```
ceph -w
ceph osd df
```
# No OpenStack-side changes required!
# New capacity is immediately available for volume/image creation


---

## 7. Troubleshooting & Maintenance Practices

### 7.1 Common Issues & Quick Fixes

| Symptom | Likely Cause | Diagnostic Command | Fix |
|---------|-------------|-------------------|-----|
| `cinder create` hangs | Ceph MON unreachable from OpenStack node | `telnet 192.168.68.248 6789` from OpenStack node | Fix firewall rules; ensure `mon_host` in ceph.conf is reachable |
| Glance upload fails with "permission denied" | Keyring ownership or cephx caps wrong | `ceph auth get client.glance` | Recreate user with correct pool caps; ensure keyring file owned by `glance` user inside container |
| Nova instance fails to boot from volume | Libvirt secret UUID mismatch | `virsh secret-list` on compute node | Re-define libvirt secret with correct UUID; ensure `rbd_secret_uuid` matches in nova.conf |
| `kolla-ansible deploy` fails on cinder-volume | Missing `[storage]` group in inventory | Check `/etc/kolla/inventory` | Add OpenStack node to `[storage]` group |
| RBD images not appearing in expected pool | Wrong pool name in service config | `grep rbd_store_pool /etc/kolla/config/glance/ceph.conf` | Correct pool name in globals.yml and redeploy |

### 7.2 Log Locations for Debugging
When things go wrong, check these logs first:


#### On OpenStack node (192.168.68.69)
#### Glance logs (Ceph backend errors)
```
docker logs glance_api  # or check /var/log/kolla/glance/
```
#### Cinder volume logs
```
docker logs cinder_volume
```
#### Nova compute logs (for libvirt/RBD attach issues)
```
docker logs nova_compute
```
#### Ceph client logs inside containers
```
docker exec -it cinder_volume ceph --id cinder -s
```
#### On Ceph node (192.168.68.248)
#### Cluster health and recent events
```
ceph -s
ceph health detail
ceph log last 100
```

### 7.3 Maintenance Window Checklist
Before performing maintenance on either side:

```bash
# Pre-Maintenance: Ceph Side
ceph osd set noout          # Prevent OSDs from marking out during restarts
ceph osd set nobackfill     # Pause data rebalancing
ceph osd set norebalance

# Pre-Maintenance: OpenStack Side
# Drain compute node if doing host maintenance
openstack server migrate --live <instance-id> <target-host>

# During Maintenance:
# - Perform Ceph upgrades, hardware changes, etc.
# - Monitor with: ceph -w

# Post-Maintenance:
ceph osd unset noout
ceph osd unset nobackfill
ceph osd unset norebalance
ceph health detail  # Confirm HEALTH_OK

# Verify OpenStack services still functional
openstack volume service list
openstack image list
```

### 7.4 Backup Strategy for Integrated Environment
Protect both Ceph data and OpenStack metadata:

```bash
# Ceph Side: Pool snapshots for critical data
rbd snap create volumes/volume-<uuid>@pre-upgrade
rbd snap protect volumes/volume-<uuid>@pre-upgrade

# OpenStack Side: Backup service databases
# Example for Cinder DB backup
docker exec -it mariadb_backend mysqldump -u root -p$DB_PASS cinder > /backup/cinder-$(date +%F).sql

# Document the integration config
# Save a copy of your working globals.yml and Ceph configs:
scp /etc/kolla/globals.yml backup-server:/kolla-backups/
scp /etc/kolla/config/*/ceph*.conf backup-server:/kolla-backups/ceph-configs/
```

---

## Final Thoughts: Making It Production-Ready

You've now connected OpenStack to Ceph in a way that scales, performs, and maintains security. But real-world success isn't just about the initial setup—it's about the daily rhythm of operations.

**Think of this integration like a well-tuned engine**:  
- The Ceph cluster is the fuel system (storage, redundancy, performance)  
- Kolla-Ansible is the transmission (orchestration, configuration management)  
- OpenStack services are the wheels (user-facing functionality)  

When all parts are aligned, you get smooth acceleration: fast volume provisioning, instant image cloning, seamless live migration. But if one part is misconfigured, the whole system sputters.

**Your next steps for production hardening**:
1. **Enable Ceph Dashboard** for visual monitoring: `ceph dashboard create-self-signed-cert`
2. **Set up Prometheus/Grafana** for OpenStack + Ceph metrics correlation
3. **Automate key rotation** with Ansible playbooks (not covered here, but essential for compliance)
4. **Document your pool naming convention** (e.g., `project-env-purpose`) to avoid confusion at 3 AM
5. **Test failure scenarios**: What happens if a Ceph MON goes down? If an OSD fails? Practice recovery.

> 🌟 **Remember**: The most reliable systems aren't built in a day. They're iterated, monitored, and refined. Your integration is now a living system—tend to it, learn from it, and let it empower your infrastructure.

---

*This guide follows official documentation from:*  
- [Kolla-Ansible External Ceph Guide (2025.2)](https://docs.openstack.org/kolla-ansible/2025.2/reference/storage/external-ceph-guide.html)  
- [Ceph RBD with OpenStack](https://docs.ceph.com/en/latest/rbd/rbd-openstack/)  

*Last verified against: Ceph Squid (19.2.x), OpenStack 2025.2 (Caracal), Kolla-Ansible 2025.2*  
*Infrastructure tested: Single-node Kolla-Ansible + 3-node Ceph cluster (as per user specs)*  

> 📝 **Note**: Always test configuration changes in a staging environment before applying to production. Infrastructure is a story—write yours carefully.
