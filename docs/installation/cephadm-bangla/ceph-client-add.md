# 🖥️ লিনাক্স ক্লায়েন্টে RBD ইমেজ ম্যাপিং - সম্পূর্ণ গাইড
## ধাপে ধাপে সেটআপ, অতিরিক্ত কনফিগারেশন ও ট্রাবলশুটিং

Ceph ক্লাস্টার সেটআপ করার পর ক্লায়েন্ট মেশিন থেকে RBD ইমেজ ব্যবহার করা মূল লক্ষ্য। নিচে **লিনাক্স ক্লায়েন্টে RBD ইমেজ ম্যাপিং**-এর সম্পূর্ণ গাইডলাইন দেওয়া হলো।

---

## 1️⃣ প্রি-রিকোয়ারমেন্টস ও ক্লায়েন্ট প্রিপারেশন

### ১.১ ক্লায়েন্ট মেশিন স্পেসিফিকেশন

| রিসোর্স | মিনিমাম | প্রডাকশন রিকমেন্ডেশন |
|--------|---------|---------------------|
| **OS** | Ubuntu 20.04+ | Ubuntu 24.04 LTS |
| **RAM** | 2 GB | 4+ GB |
| **Network** | 1 GbE | 10 GbE (প্রডাকশন) |
| **CPU** | 2 Cores | 4+ Cores |

### ১.২ প্রয়োজনীয় প্যাকেজ ইনস্টলেশন

#### ১. সিস্টেম আপডেট করুন
```
sudo apt update && sudo apt upgrade -y
```
#### ২. Ceph ক্লায়েন্ট টুল ইনস্টল করুন
```
sudo apt install -y ceph-common ceph-fuse rbd-nbd
```
#### ৩. অতিরিক্ত টুলস (অপশনাল কিন্তু রিকমেন্ডেড)
```
sudo apt install -y xfsprogs ext4-utils nfs-common
```
#### ৪. ইনস্টলেশন ভেরিফাই করুন
```
rbd --version
ceph --version
```

### ১.৩ নেটওয়ার্ক কানেক্টিভিটি চেক


#### Ceph ক্লাস্টার নোডগুলোর সাথে কানেক্টিভিটি চেক করুন
```
ping -c 3 ceph-node1
ping -c 3 ceph-node2
ping -c 3 ceph-node3
```
#### প্রয়োজনীয় পোর্ট খোলা আছে কিনা চেক করুন
```
nc -zv ceph-node1 6789
nc -zv ceph-node1 3300
nc -zv ceph-node1 6800
```
#### ✅ সফল হলে: "succeeded!" মেসেজ দেখাবে

---

## 2️⃣ Ceph কনফিগারেশন ফাইল সেটআপ

### ২.১ Ceph ডিরেক্টরি তৈরি করুন


#### Ceph কনফিগারেশন ডিরেক্টরি তৈরি করুন
```
sudo mkdir -p /etc/ceph
sudo chmod 755 /etc/ceph
```

### ২.২ Ceph কনফিগারেশন ফাইল কপি করুন

**অপশন A: SCP দিয়ে কপি করুন (রিকমেন্ডেড)**


#### Ceph ক্লাস্টার থেকে কনফিগারেশন ফাইল কপি করুন
```
sudo scp root@ceph-node1:/etc/ceph/ceph.conf /etc/ceph/
sudo scp root@ceph-node1:/etc/ceph/ceph.client.admin.keyring /etc/ceph/
```
#### পারমিশন ঠিক করুন
```
sudo chmod 644 /etc/ceph/ceph.conf
sudo chmod 600 /etc/ceph/ceph.client.admin.keyring
sudo chown root:root /etc/ceph/ceph.conf
sudo chown root:root /etc/ceph/ceph.client.admin.keyring
```

**অপশন B: ম্যানুয়ালি তৈরি করুন**


#### ceph.conf ফাইল ম্যানুয়ালি তৈরি করুন
```
sudo nano /etc/ceph/ceph.conf
```

**কনটেন্ট যুক্ত করুন:**
```ini
[global]
fsid = a1b2c3d4-e5f6-7890-abcd-ef1234567890
mon_initial_members = ceph-node1, ceph-node2, ceph-node3
mon_host = 192.168.10.11,192.168.10.12,192.168.10.13
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx

[client]
keyring = /etc/ceph/ceph.client.admin.keyring
```

### ২.৩ ক্লায়েন্ট কী রিং ভেরিফাই করুন


#### কী রিং ফাইল চেক করুন
```
sudo cat /etc/ceph/ceph.client.admin.keyring
```

#### আউটপুট এমন হওয়া উচিত:
```
# [client.admin]
#     key = AQBxxxxxxxxxxxxxxxxxxxxxxxxxxxxx==
#     caps mds = "allow *"
#     caps mon = "allow *"
#     caps osd = "allow *"
#     caps mgr = "allow *"
```

### ২.৪ ক্লাস্টার কানেক্টিভিটি টেস্ট করুন


#### Ceph ক্লাস্টার স্ট্যাটাস চেক করুন
```
ceph -s
```
#### ✅ সফল হলে ক্লাস্টার হেলথ দেখাবে
#### ❌ ফেল হলে: "Error EACCES: authentication error" বা কানেকশন এরর


---

## 3️⃣ RBD ইমেজ ম্যাপিং (ধাপে ধাপে)

### ৩.১ বিদ্যমান RBD ইমেজ লিস্ট করুন


#### সব পুলের তালিকা দেখুন
```
rbd pool ls
```
#### নির্দিষ্ট পুলের ইমেজ লিস্ট দেখুন
```
rbd ls rbd-pool
```
#### ইমেজের বিস্তারিত তথ্য দেখুন
```
rbd info rbd-pool/vm-disk-01
```

**স্যাম্পল আউটপুট:**
```
rbd image 'vm-disk-01':
        size 100 GiB in 25600 objects
        order 22 (4 MiB objects)
        snapshot_count: 0
        id: 1a2b3c4d5e6f
        block_name_prefix: rbd_data.1a2b3c4d5e6f
        format: 2
        features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
        op_features:
        flags:
        create_timestamp: Mon Jan 15 10:30:00 2024
        access_timestamp: Mon Jan 15 10:30:00 2024
        modify_timestamp: Mon Jan 15 10:30:00 2024
```

### ৩.২ RBD ইমেজ ম্যাপ করুন (Kernel Module)


#### ১. RBD কার্নেল মডিউল লোড করুন
```
sudo modprobe rbd
```
#### ২. ইমেজ ম্যাপ করুন
```
sudo rbd map rbd-pool/vm-disk-01 --id admin
```
#### অথবা পূর্ণ সিনট্যাক্স:
```
sudo rbd map rbd-pool/vm-disk-01 \
  --id admin \
  --keyring /etc/ceph/ceph.client.admin.keyring \
  --conf /etc/ceph/ceph.conf
```
#### ৩. ম্যাপ করা ডিভাইস চেক করুন
```
rbd showmapped
```
#### অথবা
```
lsblk | grep rbd
```

**স্যাম্পল আউটপুট:**
```
id  pool       image          snap device
0   rbd-pool   vm-disk-01     -    /dev/rbd0
```

### ৩.৩ RBD ইমেজ ম্যাপ করুন (NBD - Network Block Device)


#### NBD মডিউল লোড করুন
```
sudo modprobe nbd max_part=16
```
#### NBD দিয়ে ম্যাপ করুন (অল্টারনেটিভ পদ্ধতি)
```
sudo rbd-nbd map rbd-pool/vm-disk-01 --id admin
```
# ডিভাইস চেক করুন
lsblk | grep nbd
# আউটপুট: /dev/nbd0
```

### ৩.৪ ফাইলসিস্টেম তৈরি ও মাউন্ট করুন

```bash
# ১. ম্যাপ করা ডিভাইস চেক করুন
ls -la /dev/rbd0

# ২. ফাইলসিস্টেম তৈরি করুন (প্রথমবারের জন্য)
# ext4 ফাইলসিস্টেম (সাধারণ ব্যবহারের জন্য)
sudo mkfs.ext4 /dev/rbd0

# অথবা XFS ফাইলসিস্টেম (হাই পারফরমেন্সের জন্য)
sudo mkfs.xfs /dev/rbd0

# ৩. মাউন্ট পয়েন্ট তৈরি করুন
sudo mkdir -p /mnt/rbd-storage

# ৪. ডিভাইস মাউন্ট করুন
sudo mount /dev/rbd0 /mnt/rbd-storage

# ৫. মাউন্ট ভেরিফাই করুন
df -h | grep rbd
mount | grep rbd
```

### ৩.৫ ডেটা রাইট/রিড টেস্ট করুন

```bash
# ১. টেস্ট ফাইল তৈরি করুন
echo "RBD Storage Test - $(date)" | sudo tee /mnt/rbd-storage/test-file.txt

# ২. বড় ফাইল তৈরি করুন (পারফরমেন্স টেস্ট)
sudo dd if=/dev/urandom of=/mnt/rbd-storage/random-data.bin bs=1M count=500

# ৩. ফাইল ভেরিফাই করুন
cat /mnt/rbd-storage/test-file.txt
ls -lh /mnt/rbd-storage/
md5sum /mnt/rbd-storage/random-data.bin

# ৪. ডিস্ক ইউসেজ চেক করুন
df -h /mnt/rbd-storage
```

---

## 4️⃣ অটো-মাউন্ট কনফিগারেশন (fstab)

### ৪.১ fstab এন্ট্রি যুক্ত করুন

```bash
# ১. বর্তমান fstab ব্যাকআপ নিন
sudo cp /etc/fstab /etc/fstab.backup-$(date +%F)

# ২. RBD ডিভাইসের UUID বের করুন
sudo blkid /dev/rbd0
```

**স্যাম্পল আউটপুট:**
```
/dev/rbd0: UUID="a1b2c3d4-e5f6-7890-abcd-ef1234567890" TYPE="ext4"
```

```bash
# ৩. fstab এ এন্ট্রি অ্যাড করুন
sudo nano /etc/fstab
```

**নিচের লাইন যুক্ত করুন:**

```bash
# অপশন A: ডিভাইস পাথ দিয়ে (সহজ)
/dev/rbd0  /mnt/rbd-storage  ext4  _netdev,noatime,nodiratime  0  2

# অপশন B: RBD নাম দিয়ে (রিকমেন্ডেড)
rbd-pool/vm-disk-01  /mnt/rbd-storage  ext4  _netdev,noatime,nodiratime,id=admin  0  2

# অপশন C: UUID দিয়ে (সবচেয়ে নিরাপদ)
UUID=a1b2c3d4-e5f6-7890-abcd-ef1234567890  /mnt/rbd-storage  ext4  _netdev,noatime,nodiratime  0  2
```

### ৪.২ RBD অটো-ম্যাপ সার্ভিস তৈরি করুন

```bash
# ১. সিস্টেমড সার্ভিস ফাইল তৈরি করুন
sudo nano /etc/systemd/system/rbd-map.service
```

**কনটেন্ট যুক্ত করুন:**
```ini
[Unit]
Description=Map RBD Image at Boot
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/bin/rbd map rbd-pool/vm-disk-01 --id admin --conf /etc/ceph/ceph.conf
ExecStop=/usr/bin/rbd unmap /dev/rbd0
RemainAfterExit=yes
TimeoutStartSec=60

[Install]
WantedBy=multi-user.target
```

```bash
# ২. সার্ভিস এনেবল ও স্টার্ট করুন
sudo systemctl daemon-reload
sudo systemctl enable rbd-map.service
sudo systemctl start rbd-map.service

# ৩. সার্ভিস স্ট্যাটাস চেক করুন
sudo systemctl status rbd-map.service
```

### ৪.৩ fstab ভেরিফিকেশন ও টেস্ট

```bash
# ১. বর্তমান মাউন্ট আনমাউন্ট করুন
sudo umount /mnt/rbd-storage

# ২. fstab টেস্ট করুন (actual mount ছাড়া)
sudo mount -a --dry-run

# ৩. যদি dry-run সফল হয়, actual mount করুন
sudo mount -a

# ৪. মাউন্ট ভেরিফাই করুন
df -h | grep rbd-storage
mount | grep rbd-storage
```

---

## 5️⃣ অ্যাডভান্সড কনফিগারেশন

### ৫.১ পারফরমেন্স টিউনিং (ceph.conf)

```bash
# ক্লায়েন্ট ceph.conf এ পারফরমেন্স অপ্টিমাইজেশন অ্যাড করুন
sudo nano /etc/ceph/ceph.conf
```

**[client] সেকশনে যুক্ত করুন:**
```ini
[client]
keyring = /etc/ceph/ceph.client.admin.keyring

# রিড/রাইট বাফার সাইজ
rbd_cache = true
rbd_cache_size = 268435456
rbd_cache_max_dirty = 134217728
rbd_cache_target_dirty = 67108864
rbd_cache_max_dirty_age = 1.0

# রিড-আহেড সেটিংস
rbd_default_features = 63
rbd_default_stripe_count = 1
rbd_default_stripe_unit = 4194304

# নেটওয়ার্ক অপ্টিমাইজেশন
ms_mode = secure
rbd_concurrent_management_ops = 10
rbd_skip_partial_discard = false
```

### ৫.২ সিকিউরিটি কনফিগারেশন

#### A. রিস্ট্রিক্টেড ক্লায়েন্ট কী তৈরি করুন

```bash
# Ceph ক্লাস্টারে (Node1 থেকে) নতুন ক্লায়েন্ট কী তৈরি করুন
sudo cephadm shell -- ceph auth get-or-create client.app-server \
  mon 'profile rbd' \
  osd 'profile rbd pool=rbd-pool' \
  mgr 'profile rbd'
```

**আউটপুট থেকে কী কপি করুন:**
```
[client.app-server]
    key = AQCxxxxxxxxxxxxxxxxxxxxxxxxxxxxx==
```

```bash
# ক্লায়েন্টে নতুন কী রিং ফাইল তৈরি করুন
sudo nano /etc/ceph/ceph.client.app-server.keyring
```

**কনটেন্ট:**
```ini
[client.app-server]
    key = AQCxxxxxxxxxxxxxxxxxxxxxxxxxxxxx==
```

```bash
# পারমিশন ঠিক করুন
sudo chmod 600 /etc/ceph/ceph.client.app-server.keyring
sudo chown root:root /etc/ceph/ceph.client.app-server.keyring
```

```bash
# নতুন কী দিয়ে ম্যাপ করুন
sudo rbd map rbd-pool/vm-disk-01 --id app-server \
  --keyring /etc/ceph/ceph.client.app-server.keyring
```

#### B. এনক্রিপশন এনেবল করুন (LUKS)

```bash
# ১. cryptsetup ইনস্টল করুন
sudo apt install -y cryptsetup

# ২. RBD ইমেজ এনক্রিপ্ট করুন
sudo cryptsetup luksFormat /dev/rbd0

# ৩. এনক্রিপ্টেড ডিভাইস ওপেন করুন
sudo cryptsetup open /dev/rbd0 rbd-encrypted

# ৪. এনক্রিপ্টেড ভলিউমে ফাইলসিস্টেম তৈরি করুন
sudo mkfs.ext4 /dev/mapper/rbd-encrypted

# ৫. মাউন্ট করুন
sudo mkdir -p /mnt/rbd-encrypted
sudo mount /dev/mapper/rbd-encrypted /mnt/rbd-encrypted
```

### ৫.৩ কোটা ও লিমিট সেটআপ

```bash
# ১. RBD ইমেজ সাইজ রিসাইজ করুন
sudo rbd resize rbd-pool/vm-disk-01 --size 150G --allow-shrink

# ২. বর্তমান সাইজ চেক করুন
rbd info rbd-pool/vm-disk-01 | grep size

# ৩. পুল কোটা সেট করুন (Ceph ক্লাস্টারে)
sudo cephadm shell -- ceph osd pool set-quota rbd-pool max_objects 100000
sudo cephadm shell -- ceph osd pool set-quota rbd-pool max_bytes 1099511627776
```

### ৫.৪ QoS (Quality of Service) কনফিগারেশন

```bash
# RBD ইমেজে IOPS লিমিট সেট করুন
sudo rbd config set rbd-pool/vm-disk-01 qos_read_iops_limit 1000
sudo rbd config set rbd-pool/vm-disk-01 qos_write_iops_limit 500
sudo rbd config set rbd-pool/vm-disk-01 qos_read_bps_limit 104857600
sudo rbd config set rbd-pool/vm-disk-01 qos_write_bps_limit 52428800

# QoS সেটিংস ভেরিফাই করুন
sudo rbd config get rbd-pool/vm-disk-01
```

---

## 6️⃣ মাল্টিপথ ও হাই-আভেইলেবিলিটি সেটআপ

### ৬.১ মাল্টিপথ কনফিগারেশন

```bash
# ১. multipath-tools ইনস্টল করুন
sudo apt install -y multipath-tools

# ২. multipath কনফিগারেশন এডিট করুন
sudo nano /etc/multipath.conf
```

**কনটেন্ট যুক্ত করুন:**
```ini
defaults {
    user_friendly_names yes
    find_multipaths yes
}

blacklist {
    devnode "^sd[a-b]$"
}

devices {
    device {
        vendor "Ceph"
        product "RBD"
        path_checker directio
        path_selector "round-robin 0"
        hardware_handler "1 alua"
        failback immediate
        no_path_retry 5
    }
}
```

```bash
# ৩. multipath সার্ভিস রিস্টার্ট করুন
sudo systemctl restart multipathd
sudo systemctl enable multipathd

# ৪. multipath স্ট্যাটাস চেক করুন
sudo multipath -ll
```

### ৬.২ মাল্টিপল MON কনফিগারেশন

```bash
# ceph.conf এ একাধিক MON হোস্ট অ্যাড করুন
sudo nano /etc/ceph/ceph.conf
```

**[global] সেকশনে:**
```ini
[global]
mon_initial_members = ceph-node1, ceph-node2, ceph-node3
mon_host = 192.168.10.11,192.168.10.12,192.168.10.13
```

### ৬.৩ ওয়াচডগ টাইমার কনফিগারেশন

```bash
# সিস্টেমড ওয়াচডগ এনেবল করুন
sudo systemctl enable systemd-watchdog
sudo systemctl start systemd-watchdog

# RBD ওয়াচডগ টাইমআউট সেট করুন
sudo rbd config set rbd-pool/vm-disk-01 rbd_blacklist_on_break_lock true
sudo rbd config set rbd-pool/vm-disk-01 rbd_watch_timeout_seconds 30
```

---

## 7️⃣ মনিটরিং ও ট্রাবলশুটিং

### ৭.১ RBD স্ট্যাটাস মনিটরিং

```bash
# ১. ম্যাপ করা ইমেজ লিস্ট
rbd showmapped

# ২. ডিটেইলড ইনফরমেশন
rbd showmapped --format json

# ৩. পারফরমেন্স স্ট্যাটাস
sudo rbd status rbd-pool/vm-disk-01

# ৪. লক স্ট্যাটাস চেক করুন
sudo rbd lock list rbd-pool/vm-disk-01
```

### ৭.২ সাধারণ সমস্যা ও সমাধান

| সমস্যা | এরর মেসেজ | সমাধান |
|-------|----------|--------|
| **Authentication Failed** | `Error EACCES: authentication error` | কী রিং ফাইল চেক করুন, পারমিশন ঠিক করুন |
| **Connection Timeout** | `Error ENOENT: pool does not exist` | পুল নাম চেক করুন, নেটওয়ার্ক কানেক্টিভিটি ভেরিফাই করুন |
| **Device Busy** | `rbd: sysfs write failed` | ডিভাইস আনমাউন্ট করুন, লক রিলিজ করুন |
| **Map Failed** | `RBD image feature set mismatch` | ইমেজ ফিচার আপডেট করুন অথবা ক্লায়েন্ট আপগ্রেড করুন |
| **Mount Failed** | `wrong fs type` | ফাইলসিস্টেম টাইপ চেক করুন, fsck রান করুন |

### ৭.৩ ট্রাবলশুটিং কমান্ড

```bash
# ১. কানেক্টিভিটি ডিবাগ
ceph -s --conf /etc/ceph/ceph.conf

# ২. লগ চেক করুন
sudo journalctl -u rbd-map.service -f
sudo dmesg | grep rbd

# ৩. ইমেজ আনম্যাপ করুন (ফোর্স)
sudo rbd unmap /dev/rbd0 -o force

# ৪. লক রিলিজ করুন
sudo rbd lock release rbd-pool/vm-disk-01 <lock-id> --id admin

# ৫. ফাইলসিস্টেম চেক করুন
sudo fsck.ext4 -f /dev/rbd0

# ৬. পারফরমেন্স ডিবাগ
sudo iostat -x 1
sudo iotop -o
```

### ৭.৪ অটোমেটেড হেলথ চেক স্ক্রিপ্ট

```bash
#!/bin/bash
# /usr/local/bin/rbd-health-check.sh

POOL="rbd-pool"
IMAGE="vm-disk-01"
MOUNT_POINT="/mnt/rbd-storage"
LOG_FILE="/var/log/rbd-health.log"

# ১. RBD ম্যাপ চেক করুন
if ! rbd showmapped | grep -q "$IMAGE"; then
    echo "$(date): RBD image not mapped" >> $LOG_FILE
    rbd map $POOL/$IMAGE --id admin
fi

# ২. মাউন্ট চেক করুন
if ! mountpoint -q $MOUNT_POINT; then
    echo "$(date): Mount point not available" >> $LOG_FILE
    mount /dev/rbd0 $MOUNT_POINT
fi

# ৩. ডিস্ক স্পেস চেক করুন
USAGE=$(df $MOUNT_POINT | tail -1 | awk '{print $5}' | sed 's/%//')
if [ $USAGE -gt 85 ]; then
    echo "$(date): Disk usage warning: ${USAGE}%" >> $LOG_FILE
fi

# ৪. ক্লাস্টার হেলথ চেক করুন
if ! ceph -s | grep -q "HEALTH_OK"; then
    echo "$(date): Ceph cluster health warning" >> $LOG_FILE
fi
```

**Cron জব অ্যাড করুন:**
```bash
# প্রতি ৫ মিনিটে হেলথ চেক
echo "*/5 * * * * root /usr/local/bin/rbd-health-check.sh" | sudo tee -a /etc/cron.d/rbd-health
```

---

## 8️⃣ বেস্ট প্র্যাকটিস চেকলিস্ট

### ✅ ডিপ্লয়মেন্ট চেকলিস্ট

```
□ Ceph ক্লায়েন্ট প্যাকেজ ইনস্টল করা হয়েছে
□ /etc/ceph/ceph.conf কনফিগার করা হয়েছে
□ কী রিং ফাইল পারমিশন সঠিক (600)
□ নেটওয়ার্ক কানেক্টিভিটি ভেরিফাইড
□ RBD ইমেজ সফলভাবে ম্যাপ করা হয়েছে
□ ফাইলসিস্টেম তৈরি ও মাউন্ট করা হয়েছে
□ fstab এন্ট্রি অ্যাড ও টেস্ট করা হয়েছে
□ অটো-ম্যাপ সার্ভিস কনফিগার করা হয়েছে
```

### ✅ সিকিউরিটি চেকলিস্ট

```
□ রিস্ট্রিক্টেড ক্লায়েন্ট কী ব্যবহার করা হয়েছে
□ কী রিং ফাইল পারমিশন সুরক্ষিত
□ ফায়ারওয়াল রুলস কনফিগার করা হয়েছে
□ এনক্রিপশন এনেবল করা হয়েছে (যদি প্রয়োজন)
□ অপ্রয়োজনীয় ক্লায়েন্ট অ্যাক্সেস রিমুভ করা হয়েছে
```

### ✅ পারফরমেন্স চেকলিস্ট

```
□ RBD ক্যাশিং এনেবল করা হয়েছে
□ নেটওয়ার্ক MTU অপ্টিমাইজড (জাম্বো ফ্রেম)
□ I/O শিডিউলার সেট করা হয়েছে (deadline/noop)
□ মাল্টিপথ কনফিগার করা হয়েছে
□ QoS লিমিট সেট করা হয়েছে
```

### ✅ মনিটরিং চেকলিস্ট

```
□ অটোমেটেড হেলথ চেক স্ক্রিপ্ট রানিং
□ লগ মনিটরিং কনফিগার করা হয়েছে
□ অ্যালার্ট থ্রেশহোল্ড সেট করা হয়েছে
□ ব্যাকআপ স্ট্র্যাটেজি ইমপ্লিমেন্ট করা হয়েছে
□ ডিজাস্টার রিকভারি প্ল্যান ডকুমেন্টেড
```

---

## 🎯 ফাইনাল ভেরিফিকেশন কমান্ড

```bash
# সবকিছু ঠিক আছে কিনা চেক করুন

# ১. RBD ম্যাপ স্ট্যাটাস
rbd showmapped

# ২. মাউন্ট স্ট্যাটাস
df -h | grep rbd-storage

# ৩. ক্লাস্টার কানেক্টিভিটি
ceph -s

# ৪. ডেটা রাইট/রিড টেস্ট
echo "Final Test - $(date)" | sudo tee /mnt/rbd-storage/final-test.txt
cat /mnt/rbd-storage/final-test.txt

# ৫. সার্ভিস স্ট্যাটাস
sudo systemctl status rbd-map.service

# ✅ সবকিছু ঠিক থাকলে:
echo "🎉 RBD Client Setup Complete!"
```

---