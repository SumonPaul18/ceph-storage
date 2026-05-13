# 🐙 Cephadm দিয়ে Single Node Ceph Cluster ইন্সটলেশন গাইড (Ubuntu 24.04 + Docker)

![Cephadm](https://github.com/SumonPaul18/ceph-storage/blob/2f644cda4bdd027ca282d3d3bb182ae7cb6953f7/src/images/singel-host-ceph-setup.png)

---

> **অফিশিয়াল ডকুমেন্টেশন Link:** https://docs.ceph.com/en/latest/cephadm/install/

**আমার এনভায়রনমেন্ট:**
| আইটেম | ভ্যালু |
|--------|--------|
| OS | Ubuntu 24.04.3 LTS |
| Hostname | `ceph1` |
| FQDN | `ceph1.paulco.com` |
| IP Address | `192.168.68.180` |
| Container Runtime | **Docker** |
| OS Disk | `/dev/sda` (200GB, Only OS) |
| OSD Disk | `/dev/sdb` (200GB, Unpartitioned) |

---

## 📋 ধাপ ১: প্রি-রিকোয়ারমেন্টস সেটআপ

প্রথমে SSH, Docker, LVM2 এবং টাইম সিঙ্ক নিশ্চিত করুন।

#### ১. সিস্টেম আপডেট ও প্রয়োজনীয় প্যাকেজ ইন্সটল
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ssh openssh-server lvm2 chrony curl jq
```
#### ২. SSH সার্ভিস চালু ও এনাবল করুন (cephadm এর জন্য mandatory)
```
sudo systemctl enable --now ssh
```
#### ৩. Docker ইন্সটল করুন
> Ubuntu 24.04 এর জন্য অফিশিয়াল Docker repo থেকে ইন্সটল করা নিরাপদ
```
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
sudo systemctl enable --now docker
```
#### ৪. Time Synchronization চালু করুন
```
sudo systemctl enable --now chrony
chronyc sources
```
> চেক করুন সময় সিঙ্ক হচ্ছে কিনা

#### ৫. Hostname ও FQDN সেট করুন
```
sudo hostnamectl set-hostname ceph1
echo "192.168.68.180 ceph1.paulco.com ceph1" | sudo tee -a /etc/hosts
```
#### ৬. Root SSH Key জেনারেট করুন (passwordless SSH এর জন্য)
```
sudo ssh-keygen -t rsa -N "" -f /root/.ssh/id_rsa
sudo cp /root/.ssh/id_rsa.pub /root/.ssh/authorized_keys
sudo chmod 600 /root/.ssh/authorized_keys
```

---

## 📦 ধাপ ২: `cephadm` ইন্সটলেশন (Official Method)

`cephadm` ইন্সটল করার জন্য আপনার কাছে দুটি পদ্ধতি আছে। আপনার প্রয়োজন অনুযায়ী যেকোনো একটি বেছে নিন।

---

### **পদ্ধতি ১: APT রিপোজিটরি ব্যবহার করে (Ubuntu ডিফল্ট)**

**উদ্দেশ্য:** দ্রুত সেটআপের জন্য যখন Ubuntu এর ডিফল্ট ভার্সন গ্রহণযোগ্য।

**⚠️ সীমাবদ্ধতা:** এটি আপনার Ubuntu ভার্সনের সাথে বাণ্ডিল করা Ceph ভার্সনটি ইন্সটল করবে (যেমন: Ubuntu 24.04 তে Ceph Squid v19 আসে)। আপনি সহজে নির্দিষ্ট কোনো পুরনো বা LTS ভার্সন (যেমন Reef v18) বেছে নিতে পারবেন না।

#### ১. Ubuntu রিপোজিটরি থেকে cephadm ইন্সটল করুন
```bash
sudo apt update
sudo apt install -y cephadm
```
#### ২. ইন্সটলেশন পাথ ভেরিফাই করুন
```
which cephadm
```
> *আউটপুট আসবে:* /usr/sbin/cephadm

#### ৩. ভার্সন চেক করুন

```
cephadm version
```
> নোট: কিছু Ubuntu প্যাকেজে এটি 'UNKNOWN' দেখাতে পারে, সঠিক ভার্সন দেখতে:

```
dpkg -l | grep cephadm
```

---

### **পদ্ধতি ২: `curl` ব্যবহার করে নির্দিষ্ট ভার্সন ইন্সটল (প্রডাকশনের জন্য রিকমেন্ডেড)**

**উদ্দেশ্য:** প্রডাকশন এনভায়রনমেন্টের জন্য যেখানে OS ভার্সন নির্বিশেষে আপনাকে একটি নির্দিষ্ট Ceph ভার্সন (যেমন Reef v18 LTS) প্রয়োজন।

**✅ সুবিধা:** Ceph রিলিজের উপর পূর্ণ নিয়ন্ত্রণ থাকে।

**🔍 সঠিক ডাউনলোড লিংক খুঁজে বের করুন:**

#### ১. **[https://download.ceph.com/](https://download.ceph.com/)** ওয়েবসাইটে যান।

#### ২. `আপনার প্রয়োজন অনুযায়ী ceph version` খুঁজুন (যেমন: `rpm-reef`, `rpm-squid`, `rpm-quincy`)।
   * `reef` = Ceph v18 (LTS - প্রডাকশনের জন্য সেরা)
   * `squid` = Ceph v19 (লেটেস্ট)
   * `quincy` = Ceph v17 (পুরনো LTS)

#### ৩. এরপর ক্রমান্বয়ে `নির্দিষ্ট version` ফোল্ডারে যান এবং `el9` ফাইলটিতে ক্লিক করুন তারপরে `noarch` ক্লিক করুন এবং `cephadm` ফাইলটির লিংক কপি করুন।

#### ৪. লিংকটি কপি করতে (`cephadm` ফাইলটির উপরে > Right-click > Copy Link Address)। লিংকটি এমন দেখাবে:
- `https://download.ceph.com/rpm-squid/el9/noarch/cephadm`

#### ৫. নিচের কমান্ডের URL টি আপনার কপি করা URL দিয়ে পরিবর্তন করুন অথবা `CEPH_RELEASE` ভেরিয়েবল ঠিক রেখে কমান্ডটি রান করুন।

> example: curl --silent --remote-name --location `https://download.ceph.com/rpm-squid/el9/noarch/cephadm`

```
curl --silent --remote-name --location [paste here your url]
```

#### ৬. 

#### ১. আপনার কাঙ্ক্ষিত রিলিজ নাম লিখুন (যেমন: reef, squid, quincy)

```bash
CEPH_RELEASE=squid
```
#### ২. নির্দিষ্ট cephadm বাইনারিটি ডাউনলোড করুন
#### আপনি চাইলে ওয়েবসাইট থেকে পাওয়া লিংকটি নিচে পেস্ট করেও ব্যবহার করতে পারেন
```
curl --silent --remote-name --location https://download.ceph.com/rpm-${CEPH_RELEASE}/el9/noarch/cephadm
```

#### ৩. ফাইলটি এক্সিকিউটেবল বা রানযোগ্য করুন
```
chmod +x cephadm
```
#### ৪. সিস্টেম পাথে মুভ করুন
```
sudo mv cephadm /usr/sbin/cephadm
```
#### ৫. নির্দিষ্ট ভার্সনটি ভেরিফাই করুন
```
cephadm version
```
> আশা করা হচ্ছে আউটপুট আসবে: cephadm version 19.2.x ... squid (stable)

---

## 🚀 ধাপ ৩: নতুন Ceph Cluster Bootstrap করা

এখন ক্লাস্টার বুটস্ট্রাপ করবো। যেহেতু আমি **Docker** ব্যবহার করতে চান এবং এটি **Single Node**, তাই নিচের ফ্ল্যাগগুলো অত্যন্ত গুরুত্বপূর্ণ:

* `--mon-ip 192.168.10.180`: মনিটর সার্ভিসের IP
* `--single-host-defaults`: সিঙ্গেল নোডের জন্য CRUSH রুলস অটো-অপ্টিমাইজ করে (replica=2, failure-domain=host)
* `--container-engine docker`: পডম্যানের বদলে ডকার ব্যবহার করার জন্য
* `--initial-dashboard-user`: `admin` আপনার ইউজার নেম সেট করুন 
* `--initial-dashboard-password`: `admin123` ইউজারের পাসওয়ার্ড সেট করুন   

#### ক্লাস্টার বুটস্ট্রাপ কমান্ড
```bash
sudo cephadm bootstrap \
  --mon-ip 192.168.10.180 \
  --single-host-defaults \
  --container-engine docker  \
  --initial-dashboard-user admin \
  --initial-dashboard-password admin123
```

✅ **এই কমান্ডটি যা করবে:**
1. প্রথম Monitor ও Manager ডিমন কন্টেইনার চালু করবে
2. `/etc/ceph/ceph.conf` এবং `ceph.client.admin.keyring` ফাইল তৈরি করবে
3. SSH কী ম্যানেজমেন্ট সেটআপ করবে
4. Dashboard, Prometheus, Grafana সহ বেসিক সার্ভিস ডিপ্লয় করবে

🕐 **সময়:** প্রথমবার রান করলে ইমেজ পুল করতে ২-৫ মিনিট সময় নিতে পারে।

---

## 🔧 ধাপ ৪: কমান্ড লাইন থেকে Ceph Cluster ম্যানেজ করা

Cephadm ডিফল্টভাবে হোস্টে `ceph` কমান্ড ইন্সটল করে না। কমান্ড রান করার জন্য `cephadm shell` ব্যবহার করাই বেস্ট প্র্যাকটিস।

#### ✅  ১ ক্লাস্টার স্ট্যাটাস চেক করুন
```
sudo cephadm shell -- ceph -s
```
#### ✅  ২ সঠিক ভার্সন কনফার্ম করুন
```
sudo cephadm shell -- ceph -v
```
> Expected: ceph version 18.2.7 (...) reef (stable)

#### ✅  ৩ ড্যাশবোর্ড এক্সেস চেক
```
sudo cephadm shell -- ceph mgr services
```
> Output: {"dashboard": "https://ceph2:8443/"}

#### ✅ ৪. ক্লাস্টার হেলথ
```
sudo cephadm shell -- ceph health detail
```
#### ✅ ৫. সার্ভিস লিস্ট
```
sudo cephadm shell -- ceph orch ls
```
#### ✅ ৬. হোস্ট লিস্ট
```
sudo cephadm shell -- ceph orch host ls
```
#### ✅ ৭. ভার্সন ম্যাট্রিক্স
```
sudo cephadm shell -- ceph versions
```
#### ✅ ৮. অর্কেস্ট্রেটর স্ট্যাটাস
```
sudo cephadm shell -- ceph orch status
```
#### ✅ ৯. নেটওয়ার্ক কনফিগ
```
sudo cephadm shell -- ceph config dump | grep network
```
---

### অথবা - ইন্টারেক্টিভ শেলে ঢুকুন
```
sudo cephadm shell
```
#### এখন ভেতরে সরাসরি ceph কমান্ড চালাতে পারবেন
```
ceph status
```
#### এবং ইন্টারেক্টিভ cephadm shell থেকে বাহির হওয়ার জন্যঃ 
```
exit
```
---

### ceph-common প্যাকেজ ইনস্টল করলে হোস্ট থেকে সরাসরি ceph কমান্ড চালাতে পারবেন
```
sudo cephadm install ceph-common
```
#### অথবা
```
sudo apt install -y ceph-common
```
> এখন আর ইন্টারেক্টিভ শেলে Open না করেই সরাসরি ceph কমান্ড চালাতে পারবো
#### ceph-common প্যাকেজ চেক করা
```
dpkg -l | grep ceph-common 
```
#### # Ceph ভার্সন চেক করা
```
ceph -v
```

---

## 💾 ধাপ ৫: `/dev/sdb` ডিস্ক দিয়ে OSD অ্যাড করা


#### ⚠️ গুরুত্বপূর্ণ নোট: `/dev/sdb` ডিস্ক প্রস্তুতি সম্পর্কে

> **প্রশ্ন:** *"এই ডিস্কটি আমি কোনো পার্টশন করি নাই, এটা কি OSD হিসাবে অ্যাড করতে হলে ডিস্কটিতে আগে কোনো কাজ করতে হবে?"*

**উত্তর: ❌ না, কোনো পার্টিশন বা ফরম্যাট করার প্রয়োজন নেই।**

Cephadm এবং `ceph-volume` সরাসরি **Raw Disk** (`/dev/sdb`) ব্যবহার করতে পারে। বরং, ডিস্কটি যদি আগে থেকে পার্টিশন করা, LVM কনফিগার করা, বা ফাইলসিস্টেম (ext4/xfs) দিয়ে ফরম্যাট করা থাকে, তাহলে Ceph সেটি OSD হিসেবে ব্যবহার করতে **রাজি হবে না**।

✅ **কিছুই করতে হবে না:**  ডিস্কটি যেমন আছে (খালি/raw), তেমনই রাখুন। Cephadm অটোমেটিক্যালি:
1. ডিস্কটি ডিটেক্ট করবে
2. প্রয়োজনীয় LVM ভলিউম তৈরি করবে
3. BlueStore ফরম্যাটে ইনিশিয়ালাইজ করবে
4. OSD হিসেবে ক্লাস্টারে অ্যাড করবে

---

এখন আপনার 100GB ডিস্কটি ক্লাস্টারে অ্যাড করবো।

### ৫.১: ডিস্ক ভেরিফিকেশন
প্রথমে চেক করুন Ceph ডিস্কটি "Available" হিসেবে দেখতে পাচ্ছে কিনা:

```bash
sudo cephadm shell -- ceph orch device ls
```

**আউটপুটে যা দেখবেন:**
```
Hostname  Path      Type  Size   Available  Reject reasons
ceph1     /dev/sdb  hdd   100G   Yes        -
```
> ✅ যদি `Available` কলামে `Yes` থাকে, তাহলে ডিস্কটি OSD এর জন্য রেডি।

### ৫.২: OSD ডিপ্লয় করা

```bash
# নির্দিষ্ট ডিস্ক দিয়ে OSD তৈরি
sudo cephadm shell -- ceph orch daemon add osd ceph1:/dev/sdb
```

### ৫.৩: OSD স্ট্যাটাস চেক করা

```bash
# OSD লিস্ট চেক করুন
sudo cephadm shell -- ceph osd tree

# অথবা
sudo cephadm shell -- ceph orch ps --daemon-type osd
```

✅ সফল হলে আউটপুটে `ceph1` হোস্টে `osd.0` (বা অন্য ID) রানিং অবস্থায় দেখাবে।

---

## 🧪 ধাপ ৬: ক্লাস্টার ভেরিফিকেশন ও টেস্ট

```bash
# ১. পুরো ক্লাস্টার হেলথ চেক
sudo cephadm shell -- ceph -s

# ২. ডিস্ক স্পেস চেক
sudo cephadm shell -- ceph df

# ৩. একটি টেস্ট RBD ইমেজ তৈরি ও মাউন্ট টেস্ট (ঐচ্ছিক)
sudo cephadm shell -- rbd pool create test_pool
sudo cephadm shell -- rbd create test_pool/my_image --size 1G
sudo cephadm shell -- rbd ls test_pool
```

✅ যদি `HEALTH_OK` দেখায়, তাহলে আপনার সিঙ্গেল-নোড Ceph ক্লাস্টার রেডি!

---

## 🌐 ধাপ ৭: Ceph Dashboard এক্সেস (ঐচ্ছিক)

Cephadm অটোমেটিক্যালি Dashboard ইন্সটল করে। এক্সেস করতে:

```bash
# Dashboard-এর admin পাসওয়ার্ড দেখুন
sudo cephadm shell -- ceph dashboard get-login-credentials
```

ব্রাউজারে যান: `https://192.168.68.180:8443`
> ⚠️ সিকিউরিটি ওয়ার্নিং আসলে "Proceed/Advanced" এ ক্লিক করুন (Self-signed certificate)।

---

## 🛠️ ট্রাবলশুটিং ও গুরুত্বপূর্ণ টিপস

### ❓ Docker vs Podman
Cephadm ডিফল্টভাবে Podman ব্যবহার করে। আপনি যদি ভুল করে `--container-engine docker` ফ্ল্যাগ না দেন, তাহলে এটি Podman ইন্সটল করার চেষ্টা করবে। সবসময় বুটস্ট্রাপের সময় এক্সপ্লিসিটলি Docker উল্লেখ করুন।

### ❓ `/dev/sdb` Available দেখাচ্ছে না?
যদি `ceph orch device ls` এ ডিস্কটি `No` দেখায় বা `Reject reasons` থাকে:
```bash
# ডিস্কটি আগে ব্যবহার করা হয়েছিল কিনা চেক করুন
lsblk /dev/sdb

# যদি কোনো পুরনো LVM/Partition থাকে, তাহলে জ্যাপ (zap) করে ক্লিন করুন
sudo cephadm shell -- ceph orch device zap ceph1 /dev/sdb

# এরপর আবার চেক করুন
sudo cephadm shell -- ceph orch device ls
```

### ❓ Firewall Issues
Ubuntu-তে UFW বা firewall চালু থাকলে পোর্ট ওপেন করুন:
```bash
sudo ufw allow 6789,3300,6800:7300,8443/tcp
sudo ufw allow 6800:7300/udp
```

### ❓ সিঙ্গেল নোডে ডেটা সেফটি
`--single-host-defaults` ফ্ল্যাগ `osd_pool_default_size = 2` সেট করে। অর্থাৎ, ডেটা রিপ্লিকা তৈরি করার জন্য কমপক্ষে ২টি OSD প্রয়োজন। যেহেতু আপনার এখন ১টি OSD (`/dev/sdb`), তাই ক্লাস্টার `HEALTH_WARN` দেখাতে পারে (insufficient number of OSDs)। এটি নরমাল। প্রোডাকশনে কমপক্ষে ৩টি নোড/OSD রিকমেন্ডেড।

---

## 📝 কুইক রেফারেন্স শিট

```bash
# ক্লাস্টার স্ট্যাটাস
cephadm shell -- ceph -s

# নতুন OSD অ্যাড (আরেকটি ডিস্ক /dev/sdc থাকলে)
cephadm shell -- ceph orch daemon add osd ceph1:/dev/sdc

# সকল Available ডিস্ক অটো-অ্যাড করুন (সতর্কতা: সব খালি ডিস্ক ব্যবহার করবে!)
cephadm shell -- ceph orch apply osd --all-available-devices

# ডিমন রিস্টার্ট
cephadm shell -- ceph orch restart osd

# লগ চেক
cephadm shell -- ceph orch logs osd.0
```

---

## 🧹 Ceph ক্লিনআপ (Uninstall & Remove)

> ⚠️ **সতর্কতা:** এই ধাপগুলো আপনার সব Ceph ডেটা, কনফিগারেশন, পুল, OSD এবং ক্লাস্টার মুছে ফেলবে। প্রডাকশনে রান করার আগে নিশ্চিত হোন ব্যাকআপ নেওয়া আছে।

### ধাপ ১: ক্লাস্টার FSID নোট করা ও রিমুভ

প্রথমে রানিং ক্লাস্টারের আইডি (FSID) বের করে সেটি রিমুভ করতে হবে।

#### ১. রানিং ক্লাস্টারের তালিকা ও FSID চেক করুন
```
sudo cephadm ls
```
#### আউটপুট থেকে আপনার FSID টি কপি করুন (উদাহরণ: 438ec756-12f7-11f1-89d6-bc241192dd00)

#### ২. ক্লাস্টারটি জোরপূর্বক রিমুভ করুন (--force ফ্ল্যাগ ব্যবহার করা হচ্ছে)
#### নোট: <YOUR-FSID> এর জায়গায় আপনার আসল FSID টি বসান
```
sudo cephadm rm-cluster --fsid <YOUR-FSID> --force
```

### ধাপ ২: কন্টেইনার ও ভলিউম ক্লিনআপ (Docker/Podman)

#### Ceph কন্টেইনারাইজড হওয়ায় ডকার বা পডম্যানের অবশিষ্ট ফাইল মুছতে হবে। আপনি কোন রানটাইম ব্যবহার করছেন তা চেক করে সংশ্লিষ্ট কমান্ড দিন।

#### যদি Docker ব্যবহার করেন:

# সব Ceph কন্টেইনার বন্ধ করে রিমুভ
```
sudo docker rm -f $(sudo docker ps -aq --filter name=ceph-) 2>/dev/null
```
#### অব্যবহৃত ভলিউম, নেটওয়ার্ক এবং ইমেজ ক্লিনআপ
```
sudo docker volume prune -f
sudo docker network prune -f
sudo docker image prune -a -f
sudo docker system prune -a -f --volumes
```
### যদি Podman ব্যবহার করেন (Cephadm ডিফল্ট):

#### সব Ceph কন্টেইনার বন্ধ করে রিমুভ
```
sudo podman rm -f $(sudo podman ps -aq --filter name=ceph-) 2>/dev/null
sudo podman volume prune -f
sudo podman network prune -f
sudo podman image prune -a -f
```

### ধাপ ৩: APT প্যাকেজ আনইনস্টল

#### cephadm ও ceph-common রিমুভ
সিস্টেম থেকে `cephadm` এবং সম্পর্কিত টুলস সরিয়ে ফেলুন।
```bash
sudo apt remove --purge cephadm ceph-common -y
sudo apt autoremove -y
```

#### ধাপ ৪: ফাইল ও ডিরেক্টরি ক্লিনআপ

# প্রধান Ceph ডিরেক্টরিগুলো রিমুভ
```
sudo rm -rf /etc/ceph
sudo rm -rf /var/lib/ceph
sudo rm -rf /var/log/ceph
sudo rm -rf /var/cache/ceph
```
### যদি Rook ব্যবহার করে থাকেন তবে সেই ডিরেক্টরিও রিমুভ
```
sudo rm -rf /var/lib/rook
```
#### cephadm বাইনারি এবং স্ক্রিপ্ট ফাইল মুছে ফেলা (বিভিন্ন লোকেশন চেক করে)
```
sudo rm -f /usr/sbin/cephadm
sudo rm -f /usr/local/bin/cephadm
sudo rm -f ./cephadm  # বর্তমান ডিরেক্টরিতে থাকলে
```
#### APT রিপোজিটরি এবং GPG কি রিমুভ
```
sudo rm -f /etc/apt/sources.list.d/ceph.list
sudo rm -f /usr/share/keyrings/ceph-archive-keyring.gpg
```
#### রুট ইউজারের SSH authorized_keys থেকে Ceph কি এন্ট্রি মুছে ফেলা (যদি থাকে)

```
sudo sed -i '/ceph-public/d' /root/.ssh/authorized_keys 2>/dev/null
sudo sed -i '/ceph-admin/d' /root/.ssh/authorized_keys 2>/dev/null
```

#### ধাপ ৫: সিস্টেম রিস্টার্ট (Optional but Recommended)
```bash
sudo reboot
```
#### রিস্টার্টের পর যাচাইকরণ (Verification): 
> লগইন করার পর নিচের কমান্ডগুলো দিয়ে নিশ্চিত হোন সব পরিষ্কার হয়েছে:

#### ১. কনফিগ ডিরেক্টরি চেক (No such file or directory আসা উচিত)
```
ls /etc/ceph
```
#### ২. cephadm কমান্ড চেক (Command not found আসা উচিত)
```
cephadm --version
```
#### ৩. রানিং কন্টেইনার চেক (কোনো ceph কন্টেইনার থাকার কথা নয়)
```
sudo docker ps -a | grep ceph
```
এখন সিস্টেম থেকে Ceph (Uninstall & Remove) হয়েছে।


---

## 🛠️ অতিরিক্ত ট্রাবলশুটিং ও গুরুত্বপূর্ণ টিপস

```
Ceph Cluster Bootstrap করার সময়ঃ
১।  "--single-host-defaults" ফ্ল্যাগ কেনো দিতে হয়? না দিলে কি হবে? আর দিলে কি হবে? যখন মাল্টিনোডে Ceph Cluster Bootstrap করবো তখনও কি "--single-host-defaults" ফ্ল্যাগ দিতে হবে? 
২। Ceph Cluster Bootstrap করার আগেই তো নোডে Docker install করেছি, তাহলে Ceph Cluster Bootstrap করার সময় কেনো আবার  "--container-engine docker" এই ফ্ল্যাগ ব্যবহার করতে হবে ? আমাকে বুঝিয়ে বলুন।

```

# 🎯 Cephadm Bootstrap Flags - বিস্তারিত ব্যাখ্যা

আপনার দুটি প্রশ্নই খুবই গুরুত্বপূর্ণ এবং কনসেপচুয়াল। চলুন প্রতিটি বিষয় বিস্তারিত ও প্র্যাকটিক্যাল উদাহরণ সহ বুঝে নেই।

---

## ❓ প্রশ্ন ১: `--single-host-defaults` ফ্ল্যাগ কেনো দিতে হয়?

### 🔹 এই ফ্ল্যাগটি কী করে?

এই ফ্ল্যাগটি Ceph-কে বলে: *"ভাই, আমি শুধু একটি নোডে ক্লাস্টার চালাচ্ছি, তাই প্লিজ তোমার ডিফল্ট সেটিংসগুলো আমার সিঙ্গেল-নোড এনভায়রনমেন্ট অনুযায়ী অ্যাডজাস্ট করে দাও।"*

### 🔹 টেকনিক্যালি এটি কী পরিবর্তন করে?

| কনফিগারেশন | ডিফল্ট (Multi-Node) | `--single-host-defaults` সহ |
|------------|-------------------|---------------------------|
| `osd_pool_default_size` | 3 (৩টি রেপ্লিকা) | 2 (২টি রেপ্লিকা - মিনিমাম) |
| `osd_pool_default_min_size` | 2 | 1 |
| `mon_allow_pool_size_one` | false | true |
| CRUSH Failure Domain | `host` | `host` (কিন্তু single-host aware) |
| PG Autoscaling | Standard | Conservative for single node |

### 🔹 না দিলে কী হবে? (Without Flag)

```bash
# উদাহরণ: ফ্ল্যাগ ছাড়া বুটস্ট্রাপ
sudo cephadm bootstrap --mon-ip 192.168.68.180
```

**ফলাফল:**
```
✅ ক্লাস্টার বুটস্ট্রাপ সফল হবে
⚠️ কিন্তু HEALTH_WARN দেখাবে:
   "pool 'rbd' has 1 pg(s) not deploying: not enough OSDs"
   "too few OSDs to create pool replicas"
```

**কেনো?**
- Ceph ডিফল্টভাবে ৩টি রেপ্লিকা চায় (`size=3`)
- কিন্তু আপনার কাছে মাত্র ১টি OSD (`/dev/sdb`)
- তাই ডেটা রাইট করতে পারবে না → `HEALTH_WARN`

### 🔹 দিলে কী হবে? (With Flag)

```bash
# সিঙ্গেল-নোড ফ্ল্যাগ সহ
sudo cephadm bootstrap --mon-ip 192.168.68.180 --single-host-defaults
```

**ফলাফল:**
```
✅ ক্লাস্টার বুটস্ট্রাপ সফল
✅ HEALTH_OK (অথবা সাময়িক HEALTH_WARN যেটা পরে ঠিক হয়ে যাবে)
✅ ২টি রেপ্লিকা সেট হবে, যা ১টি নোডেও কাজ করবে
✅ CRUSH ম্যাপ সিঙ্গেল-হোস্টের জন্য অপ্টিমাইজড হবে
```

### 🔹 মাল্টি-নোডে কি এই ফ্ল্যাগ দেবো?

> ❌ **না, মাল্টি-নোডে এই ফ্ল্যাগ দেবেন না!**

**কেনো?**

| সিনারিও | রিকমেন্ডেশন | কারণ |
|---------|------------|-------|
| **Single Node (Lab/Dev)** | ✅ `--single-host-defaults` দিন | রেপ্লিকা সেটিংস রিল্যাক্স করে, ১টি নোডেও ক্লাস্টার চালু রাখে |
| **Multi-Node (Production)** | ❌ ফ্ল্যাগ দেবেন না | Ceph অটোমেটিক্যালি ডেটা ডিস্ট্রিবিউট করবে বিভিন্ন হোস্টে, ৩টি রেপ্লিকা সেফটি দেবে |
| **Multi-Node কিন্তু কম OSD** | ⚠️ কাস্টম পুল সেটিংস ব্যবহার করুন | `ceph osd pool set <pool> size 2` ম্যানুয়ালি সেট করুন |

### 🔹 প্র্যাকটিক্যাল ডেমো: ফ্ল্যাগের প্রভাব চেক করা

```bash
# বুটস্ট্রাপের পর এই কমান্ডগুলো রান করে দেখুন পার্থক্য:

# ১. পুলের রেপ্লিকা সেটিংস চেক
sudo cephadm shell -- ceph osd pool get rbd size

# Without flag: size: 3
# With --single-host-defaults: size: 2

# ২. CRUSH রুল চেক
sudo cephadm shell -- ceph osd crush rule dump replicated_rule

# লক্ষ্য করুন: failure_domain = host, কিন্তু single-host মোডে 
# একই হোস্টে একাধিক OSD থাকলেও রেপ্লিকা রাখতে দেবে
```

---

## ❓ প্রশ্ন ২: Docker আগেই ইন্সটল, তবু `--container-engine docker` কেনো?

### 🔹 সহজ উত্তর:
> **Docker ইন্সটল থাকলেই Cephadm অটোমেটিক্যালি Docker ব্যবহার করবে না।** 
> Cephadm-কে এক্সপ্লিসিটলি বলে দিতে হয় কোন রানটাইম ব্যবহার করতে হবে।

### 🔹 বিস্তারিত কারণ:

#### ১. Cephadm-এর ডিফল্ট বিহেভিয়ার
Cephadm বুটস্ট্রাপের সময় **অটো-ডিটেকশন** করে:

```
Cephadm Bootstrap Flow:
├─ Podman installed? → YES → Use Podman (DEFAULT)
├─ Podman not found? → Check Docker → Use Docker
├─ Neither found? → Install Podman automatically
```

> 📌 **Ubuntu 24.04-তেও** যদি আপনি ম্যানুয়ালি Podman ইন্সটল করে থাকেন (অথবা কোনো ডিপেন্ডেন্সি ইন্সটল করে থাকে), তাহলে Cephadm Podman-ই ব্যবহার করবে!

#### ২. ফ্ল্যাগ না দিলে কী সমস্যা হতে পারে?

```bash
# Scenario: আপনার সিস্টেমে Docker + Podman দুটোই ইন্সটল
sudo cephadm bootstrap --mon-ip 192.168.68.180
# (ফ্ল্যাগ ছাড়া)

# সম্ভাব্য ফলাফল:
⚠️ Cephadm Podman ডিফল্ট হিসেবে নেবে
⚠️ সব Ceph কন্টেইনার Podman দিয়ে রান করবে
⚠️ কিন্তু আপনি Docker-এর কমান্ড (`docker ps`, `docker logs`) দিয়ে মনিটর করতে যাবেন → কাজ করবে না!
⚠️ ভবিষ্যতে `ceph orch` কমান্ডগুলোও Podman ব্যবহার করবে → কনফিউশন
```

#### ৩. ফ্ল্যাগ দিলে কী লাভ?

```bash
sudo cephadm bootstrap \
  --mon-ip 192.168.68.180 \
  --container-engine docker  # ← এক্সপ্লিসিট ডিক্লারেশন
```

**লাভসমূহ:**
| বিষয় | বিবরণ |
|-------|--------|
| ✅ **Consistency** | সব Ceph ডিমন, সব `ceph orch` কমান্ড Docker-ই ব্যবহার করবে |
| ✅ **Predictability** | আপনি জানেন কোথায় কন্টেইনার রান করছে: `docker ps` দিয়ে চেক করতে পারবেন |
| ✅ **Debugging** | Docker-এর লগ, নেটওয়ার্ক, ভলিউম ম্যানেজমেন্ট টুলস ব্যবহার করতে পারবেন |
| ✅ **Future Commands** | `cephadm adopt`, `cephadm upgrade` - সব কমান্ড Docker-ই ব্যবহার করবে |

### 🔹 প্র্যাকটিক্যাল ভেরিফিকেশন

```bash
# বুটস্ট্রাপের পর চেক করুন কোন ইঞ্জিন ব্যবহার হচ্ছে:

# ১. Ceph কনফিগ চেক
sudo cephadm shell -- ceph config get mgr mgr/cephadm/container_engine

# Output: "docker" অথবা "podman"

# ২. কন্টেইনার রানটাইম চেক
sudo docker ps | grep ceph    # Docker ব্যবহার হলে আউটপুট আসবে
sudo podman ps | grep ceph    # Podman ব্যবহার হলে আউটপুট আসবে

# ৩. Cephadm কনফিগ ফাইল চেক
cat /etc/ceph/ceph.conf | grep container_engine
```

### 🔹 মাল্টি-নোডেও কি এই ফ্ল্যাগ দিতে হবে?

> ✅ **হ্যাঁ, যদি আপনি পুরো ক্লাস্টারে Docker ব্যবহার করতে চান।**

```bash
# Multi-Node Bootstrap (First Node)
sudo cephadm bootstrap \
  --mon-ip 192.168.68.180 \
  --container-engine docker

# পরবর্তী নোডগুলো অ্যাড করার সময়ও Docker ব্যবহার নিশ্চিত করতে:
sudo cephadm add-host --host ceph2.jmc.com --container-engine docker
```

> 💡 **টিপস:** একবার বুটস্ট্রাপের সময় `--container-engine docker` দিলে, Ceph এই সেটিংস ক্লাস্টার-ওয়াইড কনফিগারেশনে সেভ করে রাখে। পরবর্তীতে নোড অ্যাড করার সময় আবার উল্লেখ না করলেও এটি Docker-ই ব্যবহার করবে। তবে এক্সপ্লিসিটলি দেওয়া ভালো প্র্যাকটিস।

---

## 🎯 কুইক ডিসিশন গাইড

### `--single-host-defaults` কখন দেবেন?

```
✅ দিন যখন:
   - ল্যাব/টেস্টিং এনভায়রনমেন্ট
   - মাত্র ১টি ফিজিক্যাল/VM নোড
   - OSD সংখ্যা ১-২টি

❌ দেবেন না যখন:
   - প্রোডাকশন মাল্টি-নোড ক্লাস্টার
   - ৩+ নোড এবং ৩+ OSD
   - হাই-এভেইলেবিলিটি রিকোয়ারমেন্ট
```

### `--container-engine docker` কখন দেবেন?

```
✅ দিন যখন:
   - আপনি Docker-এর সাথে কমফোর্টেবল
   - Docker-এর টুলিং (`docker logs`, `docker exec`) ব্যবহার করতে চান
   - আপনার অর্গানাইজেশনে Docker স্ট্যান্ডার্ড

✅ Podman ডিফল্ট রাখতে চাইলে:
   - ফ্ল্যাগ দেবেন না (RHEL/CentOS-এ এটাই ডিফল্ট)
   - অথবা এক্সপ্লিসিটলি --container-engine podman দিন

⚠️ কখনোই একই ক্লাস্টারে মিক্স করবেন না!
```

---

## 🧪 বোনাস: আপনার এনভায়রনমেন্টের জন্য ফাইনাল কমান্ড

```bash
# আপনার সিঙ্গেল-নোড + Docker + Ubuntu সেটআপের জন্য অপ্টিমাইজড:
sudo cephadm bootstrap \
  --mon-ip 192.168.68.180 \
  --single-host-defaults \
  --container-engine docker \
  --dashboard-password-noprompt \
  --allow-overwrite
```

> 📝 `--allow-overwrite`: যদি ভুল করে আগে বুটস্ট্রাপ করে থাকেন এবং রি-ইন্সটল করতে চান।

---

> 🎯 **সারসংক্ষেপ:**
> 1. `--single-host-defaults` → সিঙ্গেল নোডে রেপ্লিকা সেটিংস রিল্যাক্স করে, মাল্টি-নোডে দেবেন না
> 2. `--container-engine docker` → Cephadm-কে ফোর্স করে Docker ব্যবহার করতে, না দিলে Podman ডিফল্ট হতে পারে
> 3. দুটো ফ্ল্যাগই **বুটস্ট্রাপের সময়** একবারই দিতে হয়, পরে ক্লাস্টার-ওয়াইড সেভ হয়ে যায়

---

