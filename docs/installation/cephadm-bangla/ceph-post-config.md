# **Ceph Cluster Setup & Management - বাস্তব কাজের সম্পূর্ণ গাইড**
## *একটি ৩-নোড ক্লাস্টারের গল্প: শুরু থেকে প্রোডাকশন পর্যন্ত*

---

## **📖 প্রস্তাবনা: আমাদের বর্তমান অবস্থা**

> *"আমি আমার ৩টি সার্ভারে Ceph Squid v19.2.3 ইনস্টল করে ক্লাস্টার রেডি করেছি। প্রতিটি নোডে ২টি করে ৩০০জিবি HDD এবং ১টি ২০০জিবি SSD আছে। মোট ৯টি OSD। এখন ক্লাস্টার রেডি, কিন্তু এরপর কী করবো? Pool কীভাবে তৈরি করবো? কোন সেটিংস ঠিক হবে? এই গাইডে আমি ধাপে ধাপে সব শেখাবো, ঠিক যেমন আমি নিজে করছি।"*

```
🏢 আমাদের ইনফ্রাস্ট্রাকচার:
┌─────────────────────────────────┐
│ 3 Nodes: ceph1, ceph2, ceph3    │
│                                 │
│ প্রতিটি নোডে:                  │
│ ├── 2× HDD 300GB (osd.0-5)     │
│ └── 1× SSD 200GB (osd.6-8)     │
│                                 │
│ মোট Storage: 2.4TB (Raw)       │
│ Replication: 3x                │
│ ব্যবহারযোগ্য: ~800GB           │
└─────────────────────────────────┘
```

---

# **🔧 ধাপ ১: ক্লাস্টার হেলথ চেক - প্রথম কাজ**

> *"ক্লাস্টার রেডি হওয়ার পর প্রথম কাজ হলো চেক করা সব ঠিক আছে কিনা।"*

## **১.১ CLI দিয়ে হেলথ চেক**

```bash
# ১. ক্লাস্টার স্ট্যাটাস (সবচেয়ে গুরুত্বপূর্ণ)
ceph -s

# আউটপুট উদাহরণ:
  cluster:
    id:     abc123...
    health: HEALTH_OK    ← ✅ সব ঠিক!
  
  services:
    mon: 3 daemons, quorum ceph1,ceph2,ceph3
    mgr: ceph1(active), standbys: ceph2, ceph3
    osd: 9 osds: 9 up, 9 in
    
  
    pools:   3 pools, 96 pgs
    objects: 250 objects, 10 GiB
    usage:   30 GiB used, 2.3 TiB / 2.4 TiB avail
    pg stats: 96 active+clean
```

**Health Status মানে**:
```
🟢 HEALTH_OK      = সব ঠিক, কোনো সমস্যা নেই
🟡 HEALTH_WARN    = সতর্কতা, কিছু ঠিক করা দরকার
🔴 HEALTH_ERR     = সমস্যা আছে, দ্রুত ঠিক করতে হবে
```

```bash
# ২. বিস্তারিত হেলথ চেক (যদি WARN/ERR থাকে)
ceph health detail

# ৩. OSD স্ট্যাটাস চেক
ceph osd stat
ceph osd tree

# আউটপুট:
ID  CLASS  WEIGHT   TYPE NAME     STATUS  REWEIGHT  PRI-AFF
-1         2.39900  root default                              
-3         0.79900      host ceph1                            
 0    hdd  0.29900          osd.0      up   1.00000  1.00000
 1    hdd  0.29900          osd.1      up   1.00000  1.00000
 6    ssd  0.20000          osd.6      up   1.00000  1.00000
...
```

## **১.২ Dashboard দিয়ে হেলথ চেক**

```
🌐 ব্রাউজারে যান: https://<ceph-node-ip>:8443
🔐 Login: admin / আপনার পাসওয়ার্ড

Dashboard → Cluster → Health
├── 🟢 HEALTH_OK দেখবেন
├── OSDs: 9/9 Up & In
├── MONs: 3/3 Quorum
└── PGs: সব active+clean
```

---

# **💾 ধাপ ২: OSD ম্যানেজমেন্ট - বাস্তব গাইড**

> *"OSD হলো Ceph এর হার্ড ডিস্ক। এগুলো ঠিকমতো ম্যানেজ না করলে পুরো ক্লাস্টার সমস্যায় পড়বে।"*

## **২.১ OSD চেক করার প্র্যাকটিক্যাল কমান্ড**

```bash
# ১. সব OSD এর লিস্ট ও স্ট্যাটাস
ceph osd tree

# ২. প্রতিটি OSD এর বিস্তারিত তথ্য
ceph osd df

# আউটপুট উদাহরণ:
ID  CLASS  WEIGHT   REWEIGHT  SIZE     RAW USE  DATA     OMAP  META     AVAIL    %USE  VAR   PGS  STATUS
 0   hdd   0.29900   1.00000  299 GiB  100 GiB  90 GiB   1 MiB  9 GiB   199 GiB  33.44  0.99   32  up, in
 6   ssd   0.20000   1.00000  200 GiB   50 GiB  45 GiB   512 KiB 5 GiB   150 GiB  25.00  0.74   32  up, in

# গুরুত্বপূর্ণ কলাম:
# - %USE: কতটা ব্যবহার হচ্ছে (80% এর উপরে গেলে সতর্ক!)
# - PGS: এই OSD তে কতটি PG আছে (সবগুলোতে প্রায় সমান হতে হবে)
# - VAR: variation, 1.00 মানে পারফেক্ট ব্যালেন্স
```

```bash
# ৩. নির্দিষ্ট OSD এর বিস্তারিত
ceph osd df osd.0

# ৪. OSD লগ চেক (সমস্যা হলে)
ceph orch logs osd.0

# ৫. OSD পারফরম্যান্স চেক
ceph osd perf

# আউটপুট:
osd   commit_latency(ms)   apply_latency(ms)
 0            2.1                  1.8      ← SSD হলে 1-5ms, HDD হলে 10-50ms
 6            0.8                  0.6      ← SSD ফাস্ট!
```

## **২.২ নতুন OSD অ্যাড করার প্র্যাকটিক্যাল গাইড**

> *"ধরুন, আমরা ceph1 নোডে আরেকটি 500GB HDD অ্যাড করতে চাই।"*

### **পদ্ধতি ১: cephadm দিয়ে (সুপারিশকৃত)**

```bash
# ১. নতুন ডিস্ক চেক করুন (সার্ভারে লগইন করে)
lsblk
# output: sdb  500G  ← এটি আমাদের নতুন ডিস্ক

# ২. cephadm দিয়ে OSD অ্যাড করুন
# Syntax: ceph orch daemon add osd <host>:<device>
ceph orch daemon add osd ceph1:/dev/sdb

# ৩. প্রোগ্রেস দেখুন
watch 'ceph -s'
# output: "osd.9 created" দেখবেন

# ৪. ভেরিফাই করুন
ceph osd tree
# নতুন osd.9 দেখতে পাবেন
```

### **পদ্ধতি ২: ম্যানুয়ালি (যদি cephadm না থাকে)**

```bash
# ১. ডিস্ক প্রিপেয়ার করুন (সতর্কতা: সব ডেটা মুছে যাবে!)
ceph-volume lvm create --data /dev/sdb

# ২. Ceph অটোমেটিক নতুন OSD ডিটেক্ট করবে
# ৩. ভেরিফাই করুন
ceph osd tree
```

### **OSD অ্যাড করার পর কী হয়?**

```
🔄 অটোমেটিক প্রক্রিয়া:
1. নতুন OSD "up" এবং "in" স্টেটাস পায়
2. PG Autoscale (যদি on থাকে) নতুন PG calculate করে
3. Ceph ব্যাকগ্রাউন্ডে PG গুলো rebalance করে
4. 30-60 মিনিট সময় লাগতে পারে
5. শেষে সব OSD তে সমান ডেটা ডিস্ট্রিবিউশন হয়

📊 মনিটরিং:
watch 'ceph -s'
# "rebalancing" → "peering" → "active+clean" দেখবেন
```

## **২.৩ OSD রিলেটেড গুরুত্বপূর্ণ টপিকস**

### **টপিক ১: OSD Out/In করা (মেইনটেন্যান্সের জন্য)**

```bash
# Scenario: osd.3 রিপ্লেস করতে চান

# ১. OSD কে "out" করুন (ডেটা অন্য জায়গায় কপি হবে)
ceph osd out osd.3

# ২. প্রোগ্রেস দেখুন
watch 'ceph -s'
# "degraded" → "recovering" → "active+clean"

# ৩. OSD স্টপ করুন
ceph orch stop osd.3

# ৪. ডিস্ক রিপ্লেস করুন (ফিজিক্যালি)

# ৫. নতুন OSD অ্যাড করুন (উপরের মতো)

# ৬. নতুন OSD অটোমেটিক "in" হবে
# Ceph আবার ডেটা রিব্যালেন্স করবে
```

### **টপিক ২: OSD রিওয়েট (Weight ম্যানেজমেন্ট)**

```bash
# Scenario: osd.6 (SSD 200GB) এর weight কম, তাই কম PG পায়

# ১. বর্তমান weight চেক করুন
ceph osd df | grep osd.6
# output: weight = 0.15 (কম)

# ২. সঠিক weight ক্যালকুলেট করুন
# Formula: capacity_GB / 1000
# SSD 200GB = 200/1000 = 0.20

# ৩. weight আপডেট করুন
ceph osd crush reweight osd.6 0.20

# ৪. প্রভাব দেখুন
watch 'ceph -s'
# osd.6 এ এখন বেশি PG আসবে
```

### **টপিক ৩: OSD ফেইল হলে কী করবেন?**

```bash
# Scenario: osd.2 ডাউন দেখাচ্ছে

# ১. কনফার্ম করুন
ceph osd tree | grep osd.2
# STATUS: down

# ২. কারণ খুঁজুন
ceph health detail | grep osd.2
ceph orch logs osd.2

# ৩. সম্ভাব্য সমাধান:
# ক) নেটওয়ার্ক সমস্যা: সার্ভার রিস্টার্ট
# খ) ডিস্ক ফেইল: রিপ্লেস করুন
# গ) সফটওয়্যার সমস্যা: OSD রিস্টার্ট

# ৪. রিস্টার্ট ট্রাই করুন
ceph orch restart osd.2

# ৫. যদি না আসে, "out" করে রিপ্লেস প্ল্যান করুন
ceph osd out osd.2
```

---

# **🗄️ ধাপ ৩: Pool Creation - সম্পূর্ণ প্র্যাকটিক্যাল গাইড**

> *"Pool হলো Ceph এর লজিকাল স্টোরেজ কন্টেইনার। সঠিক Pool কনফিগারেশন মানে ভালো পারফরম্যান্স এবং রিলায়েবিলিটি।"*

## **৩.১ Pool তৈরির আগে: আমাদের রিকোয়ারমেন্ট**

```
🎯 আমাদের ব্যবহারের ক্ষেত্র:
1. Proxmox VM ডিস্ক (High Performance) → SSD Pool
2. Backup Storage (Cost Effective) → HDD Pool  
3. General Purpose (Mixed) → Mixed Pool

📋 আমাদের Pool প্ল্যান:
┌────────────────────────────────────┐
│ Pool 1: ssd-pool                   │
│ ├── Use: Production VMs, Database │
│ ├── Type: Replicated              │
│ ├── Size: 3 replicas              │
│ ├── PG Autoscale: on              │
│ └── Compression: none             │
│                                    │
│ Pool 2: hdd-pool                   │
│ ├── Use: Backups, Archives        │
│ ├── Type: Replicated              │
│ ├── Size: 3 replicas              │
│ ├── PG Autoscale: on              │
│ └── Compression: passive          │
└────────────────────────────────────┘
```

---

## **৩.২ Pool Type - বাস্তব উদাহরণ**

### **থিওরি: Pool Type কী?**

```
Ceph এ ২ ধরণের Pool:

1️⃣ Replicated Pool (প্রতিলিপি)
   ├── কি করে: প্রতিটি ডেটা একাধিক OSD তে কপি করে
   ├── উদাহরণ: Size=3 মানে ডেটা ৩টি ভিন্ন OSD তে সেভ হবে
   ├── সুবিধা: ফাস্ট রিকভারি, সহজ কনফিগারেশন
   ├── অসুবিধা: বেশি স্টোরেজ লাগে (3x)
   └── ব্যবহার: VM ডিস্ক, Database, High Performance

2️⃣ Erasure Coded Pool (ইরেজার কোড)
   ├── কি করে: ডেটাকে ভেঙে parity তৈরি করে (RAID 5/6 এর মতো)
   ├── উদাহরণ: k=4, m=2 মানে 4 chunk ডেটা + 2 chunk parity
   ├── সুবিধা: 50% কম স্টোরেজ লাগে
   ├── অসুবিধা: স্লো রাইট, কমপ্লেক্স রিকভারি
   └── ব্যবহার: Backup, Archive, বড় ক্লাস্টার (10+ নোড)
```

### **প্র্যাকটিক্যাল: আমাদের ক্লাস্টারে কোনটি?**

```bash
# আমাদের ৩-নোড, ৯-OSD ক্লাস্টারের জন্য:
# ✅ Replicated Pool ব্যবহার করবো

# কারণ:
# - ছোট ক্লাস্টার (৩ নোড)
# - Erasure Code এর জন্য মিনিমাম ৫-৬ নোড দরকার
# - আমাদের পারফরম্যান্স দরকার
# - ম্যানেজমেন্ট সহজ রাখতে চাই
```

**Dashboard থেকে তৈরি**:
```
Dashboard → Ceph Cluster → Pools → Create Pool

┌─────────────────────────────┐
│ Pool Name: ssd-pool        │
│ Pool Type: ● Replicated    │ ← সিলেক্ট করুন
│ Erasure Code Profile: -    │ ← ডিজেবল থাকবে
└─────────────────────────────┘
```

**CLI দিয়ে তৈরি**:
```bash
# Replicated pool তৈরি
ceph osd pool create ssd-pool 32 32 replicated

# Pool এ application enable করুন
ceph osd pool application enable ssd-pool rbd

# ভেরিফাই করুন
ceph osd pool ls detail | grep ssd-pool
```

---

## **৩.৩ PG (Placement Groups) - বাস্তব কাজের গাইড**

### **থিওরি: PG আসলে কী?**

```
🧠 সহজ উদাহরণ:
মনে করুন আপনার একটি গুদাম আছে:
- 📦 পণ্য = আপনার ডেটা (Objects)
- 🗄️ র্যাক = আপনার OSDs (Hard Disks)  
- 📋 ইনভেন্টরি লিস্ট = Placement Groups (PGs)

সমস্যা: ১০,০০০ পণ্য সরাসরি ৯টি র্যাকে ম্যাপ করা কঠিন!
সমাধান: পণ্যগুলোকে ১২৮টি গ্রুপে (PG) ভাগ করুন।
        প্রতিটি গ্রুপের লিস্ট রাখুন।
        এখন খুঁজে পাওয়া সহজ!

🔄 Ceph এর কাজ:
হাজার হাজার Objects → কয়েকশ PG → কয়েকটি OSD তে ম্যাপ

✅ কেন দরকার:
- Ceph হাজার হাজার Object ট্র্যাক করে না
- Ceph কয়েকশ PG ট্র্যাক করে (যেগুলোর মধ্যে Objects গ্রুপ করা)
- এটি পারফরম্যান্স এবং স্কেলেবিলিটি বাড়ায়
```

### **প্র্যাকটিক্যাল: আমাদের ক্লাস্টারে কতটি PG?**

```bash
# সূত্র:
Total PGs = (OSD সংখ্যা × 100) / Replication Size

# আমাদের ক্ষেত্রে:
OSD = 9
Replication = 3

Total PGs = (9 × 100) / 3 = 300 PGs

# প্রতি pool এ ভাগ করলে:
# - ssd-pool (3 OSD): (3×100)/3 = 100 → 128 PGs (2 এর পাওয়ার)
# - hdd-pool (6 OSD): (6×100)/3 = 200 → 256 PGs (2 এর পাওয়ার)

# গুরুত্বপূর্ণ: PG count সবসময় 2 এর পাওয়ার হতে হবে!
# ✅ 8, 16, 32, 64, 128, 256, 512, 1024...
# ❌ 100, 150, 200 - এগুলো হবে না
```

### **Dashboard থেকে PG সেটিংস**:

```
Pool Creation Form → PG Settings:

┌─────────────────────────────────┐
│ PG Autoscale: ● on             │ ← সুপারিশকৃত!
│                                  │
│ ⚪ off  = ম্যানুয়ালি সেট করতে হবে │
│ ⚪ warn  = warning দেবে, change করবে না │
│ ● on   = Ceph নিজেই manage করবে │
└─────────────────────────────────┘
```

### **প্র্যাকটিক্যাল: PG Autoscale এর ৩টি মোড**

#### **মোড ১: `on` (সুপারিশকৃত) ✅**

```bash
# কি করে:
# - Ceph নিজে অটোমেটিক সঠিক PG সংখ্যা calculate করে
# - ডেটা বাড়লে/কমলে PG সংখ্যা অটোমেটিক adjust করে
# - আপনাকে কিছু করতে হয় না

# কখন ব্যবহার করবেন:
# ✅ নতুন ক্লাস্টার (আপনার মতো)
# ✅ ছোট/মাঝারি ক্লাস্টার (৩-১০ নোড)
# ✅ যখন আপনি PG calculation নিয়ে চিন্তা করতে চান না

# আমাদের জন্য:
ceph osd pool set ssd-pool pg_autoscale_mode on
ceph osd pool set hdd-pool pg_autoscale_mode on

# চেক করতে:
ceph osd pool autoscale-status
# Output:
# POOL        TARGET  MINE  SUGGESTED
# ssd-pool    128     ok    128    ← সব ঠিক!
# hdd-pool    256     ok    256
```

#### **মোড ২: `off` (বন্ধ) ❌**

```bash
# কি করে:
# - Ceph কোনো অটোমেটিক PG adjustment করে না
# - আপনাকে ম্যানুয়ালি PG সংখ্যা সেট করতে হয়

# কখন ব্যবহার করবেন:
# ⚠️ খুব নির্দিষ্ট পারফরম্যান্স প্রয়োজন
# ⚠️ আপনি PG calculation সম্পর্কে এক্সপার্ট

# উদাহরণ:
ceph osd pool create mypool 128  # ম্যানুয়ালি 128 PG
ceph osd pool set mypool pg_autoscale_mode off

# ঝুঁকি:
# ❌ ভুল PG count = খারাপ পারফরম্যান্স
# ❌ ডেটা বাড়লেও PG বাড়বে না
```

#### **মোড ৩: `warn` (সতর্কতা) ⚠️**

```bash
# কি করে:
# - Ceph অটোমেটিক PG adjust করে না
# - কিন্তু PG count ভুল হলে সতর্কতা (warning) দেয়

# কখন ব্যবহার করবেন:
# ⚠️ আপনি নিজে PG manage করতে চান
# ⚠️ কিন্তু ভুল হলে warning পেতে চান

# উদাহরণ:
ceph osd pool set mypool pg_autoscale_mode warn

# যদি ভুল PG দেন:
# Warning: "pool 'mypool' has too few PGs (16), recommended: 128"
# আপনি warning দেখে ম্যানুয়ালি ঠিক করবেন
```

### **আপনার সমস্যার সমাধান: কেন আপনার PG ৩২?**

```bash
# আপনার বর্তমান অবস্থা:
# - পুরনো pool: 32 PG
# - নতুন OSD অ্যাড করেছেন ২ বার
# - কিন্তু PG বাড়েনি

# সম্ভাব্য কারণ:
# ১. PG Autoscale "warn" মোডে আছে, "on" না
# ২. Pool এ ডেটা কম, তাই Ceph মনে করে "এখনও বেশি PG দরকার নেই"
# ৩. PG Autoscale ধীরে কাজ করে (30 মিনিট - 2 ঘণ্টা)

# সমাধান:
# ১. PG Autoscale mode চেক করুন
ceph osd pool get <pool-name> pg_autoscale_mode

# ২. যদি "warn" বা "off" থাকে, "on" করুন
ceph osd pool set <pool-name> pg_autoscale_mode on

# ৩. Target size ratio সেট করুন (ঐচ্ছিক)
# এটি Ceph কে বলে যে pool কত বড় হবে
ceph osd pool set <pool-name> target_size_ratio 0.1

# ৪. অথবা ম্যানুয়ালি PG বাড়ান (দ্রুত সমাধান)
ceph osd pool set <pool-name> pg_num 128
ceph osd pool set <pool-name> pgp_num 128

# ৫. মনিটর করুন
watch 'ceph -s'
# 10-20 মিনিট পর নতুন PG active হবে
```

---

## **৩.৪ Replicated Size - বাস্তব উদাহরণ**

### **থিওরি: Replicated Size কী?**

```
🔄 Replicated Size = প্রতিটি ডেটা কতবার কপি হবে

উদাহরণ:
- Size = 1: ডেটা ১বার সেভ (কোনো রেপ্লিকা নেই) ❌
- Size = 2: ডেটা ২টি ভিন্ন OSD তে কপি ✅
- Size = 3: ডেটা ৩টি ভিন্ন OSD তে কপি ✅✅ (সুপারিশকৃত)

📊 Storage Efficiency:
- Size=2: 50% efficiency (1TB ডেটার জন্য 2TB জায়গা)
- Size=3: 33% efficiency (1TB ডেটার জন্য 3TB জায়গা)

🛡️ Fault Tolerance:
- Size=2: 1টি OSD ফেল করলেও ডেটা নিরাপদ
- Size=3: 2টি OSD ফেল করলেও ডেটা নিরাপদ
```

### **প্র্যাকটিক্যাল: আমাদের ক্লাস্টারে কী সেট করবো?**

```bash
# আমাদের ৩-নোড ক্লাস্টারের জন্য:

# ssd-pool (Production VMs):
# Size = 3 (সুপারিশকৃত)
# কারণ: ডেটা নিরাপত্তা সবচেয়ে গুরুত্বপূর্ণ

# hdd-pool (Backups):
# Size = 2 (ঐচ্ছিক)
# কারণ: বেশি স্টোরেজ সাশ্রয়, ব্যাকআপ তো আছেই

# Dashboard থেকে:
┌─────────────────────────────┐
│ Replicated Size: [3]       │
│ Min Size: [2]              │ ← Size=3 হলে Min=2 রাখুন
└─────────────────────────────┘

# CLI দিয়ে:
ceph osd pool set ssd-pool size 3
ceph osd pool set ssd-pool min_size 2

# ভেরিফাই করুন:
ceph osd pool get ssd-pool size
ceph osd pool get ssd-pool min_size
```

**Min Size এর গুরুত্ব**:
```
Scenario: নেটওয়ার্ক সমস্যায় ১টি নোড আনরিচেবল

- Size=3, Min Size=2 হলে:
  ✅ কমপক্ষে ২টি নোডে লেখা নিশ্চিত হলেই write সাকসেস
  ✅ নেটওয়ার্ক ঠিক হলে ৩য় রেপ্লিকা অটোমেটিক আপডেট হবে

- Size=3, Min Size=3 হলে:
  ❌ সব ৩টি নোডে লিখতে না পারলে write ফেইল
  ❌ নেটওয়ার্ক সমস্যায় সার্ভিস ডাউন

✅ সুপারিশ: Min Size = Size - 1
```

---

## **৩.৫ Applications - বাস্তব উদাহরণ**

### **থিওরি: Application কী?**

```
🏷️ Application = এই pool কোন সফটওয়্যার/সার্ভিস ব্যবহার করবে

Ceph কে জানানো জরুরি যাতে:
- সঠিক ডিফল্ট সেটিংস অ্যাপ্লাই করে
- Dashboard এ সঠিক ফিচার দেখায়
- Monitoring ও আলার্ট সঠিকভাবে কাজ করে

📦 Available Applications:
1️⃣ rbd (RADOS Block Device)
   ├── ব্যবহার: Virtual Machines, Container storage, Database volumes
   ├── উদাহরণ: Proxmox VM ডিস্ক, Kubernetes PVC, MySQL data
   └── আমাদের জন্য: ✅ ssd-pool ও hdd-pool উভয়তেই

2️⃣ cephfs (Ceph File System)
   ├── ব্যবহার: Shared file storage, Home directories, NFS replacement
   ├── উদাহরণ: /home শেয়ার, প্রজেক্ট ফাইল শেয়ার
   └── আমাদের জন্য: ❌ এখন দরকার নেই

3️⃣ rgw (RADOS Gateway)
   ├── ব্যবহার: S3 compatible object storage, Backup storage
   ├── উদাহরণ: Amazon S3 এর মতো API, Media files storage
   └── আমাদের জন্য: ❌ এখন দরকার নেই
```

### **প্র্যাকটিক্যাল: Application Enable করা**

```bash
# Pool তৈরির পর application enable করতে হবে:

# ssd-pool এর জন্য (VM ডিস্ক)
ceph osd pool application enable ssd-pool rbd

# hdd-pool এর জন্য (Backup)
ceph osd pool application enable hdd-pool rbd

# ভেরিফাই করুন:
ceph osd pool application get ssd-pool
# Output: rbd

# Dashboard থেকে:
Pool Creation Form → Applications:
┌─────────────────────────────┐
│ ☑ rbd                      │ ← চেক করুন
│ ☐ cephfs                   │
│ ☐ rgw                      │
└─────────────────────────────┘
```

**গুরুত্বপূর্ণ নোট**:
```
⚠️ একই pool এ একাধিক application enable করবেন না!
❌ ceph osd pool application enable mypool rbd
❌ ceph osd pool application enable mypool rgw  ← করবেন না!

✅ আলাদা pool ব্যবহার করুন:
- rbd-pool: VM ডিস্কের জন্য
- rgw-pool: Object storage এর জন্য
```

---

## **৩.৬ CRUSH Map & Ruleset - বাস্তব গাইড**

### **থিওরি: CRUSH Map কী?**

```
🗺️ CRUSH = Controlled Replication Under Scalable Hashing

সহজ কথা: CRUSH Map হলো Ceph এর "ম্যাপ" যা বলে:
"কোন ডেটা কোন OSD তে যাবে?"

🎯 CRUSH এর কাজ:
1️⃣ ডেটা ডিস্ট্রিবিউশন: বিভিন্ন OSD তে সমানভাবে ডেটা ছড়িয়ে দেয়
2️⃣ Failure Domain: একই নোড/র্যাকের ডিস্কে একই ডেটার রেপ্লিকা না রাখা
3️⃣ Device Class: SSD ও HDD আলাদা করে হ্যান্ডেল করা

🏗️ CRUSH Hierarchy (আমাদের ক্লাস্টার):
root default
├── host ceph1
│   ├── osd.0 (hdd-300GB)
│   ├── osd.1 (hdd-300GB)
│   └── osd.6 (ssd-200GB)
├── host ceph2
│   ├── osd.2 (hdd-300GB)
│   ├── osd.3 (hdd-300GB)
│   └── osd.7 (ssd-200GB)
└── host ceph3
    ├── osd.4 (hdd-300GB)
    ├── osd.5 (hdd-300GB)
    └── osd.8 (ssd-200GB)
```

### **প্র্যাকটিক্যাল: CRUSH Ruleset তৈরি ও ব্যবহার**

#### **সিনারিও ১: SSD ও HDD আলাদা Pool এ রাখা (সুপারিশকৃত)**

```bash
# ১. বর্তমান CRUSH rules দেখুন
ceph osd crush rule ls
# Output: replicated_rule

# ২. SSD এর জন্য আলাদা rule তৈরি
ceph osd crush rule create-replicated ssd-rule host ssd

# ৩. HDD এর জন্য আলাদা rule তৈরি
ceph osd crush rule create-replicated hdd-rule host hdd

# ৪. Device class চেক করুন (SSD/HDD আলাদা করা আছে কিনা)
ceph osd crush class ls
# Output: hdd, ssd

# ৫. যদি class না থাকে, ম্যানুয়ালি সেট করুন
ceph osd crush classify osd.6 ssd
ceph osd crush classify osd.7 ssd
ceph osd crush classify osd.8 ssd
# (osd.0-5 অটোমেটিক hdd থাকবে)
```

#### **সিনারিও ২: Pool এ Ruleset Apply করা**

```bash
# ssd-pool এ ssd-rule apply করুন
ceph osd pool set ssd-pool crush_rule ssd-rule

# hdd-pool এ hdd-rule apply করুন
ceph osd pool set hdd-pool crush_rule hdd-rule

# ভেরিফাই করুন:
ceph osd pool get ssd-pool crush_rule
# Output: ssd-rule
```

#### **সিনারিও ৩: Failure Domain কনফিগার করা**

```bash
# যদি চান ডেটা ভিন্ন নোডে যাবে (হোস্ট লেভেল)
ceph osd crush rule create-replicated host-rule host host

# যদি চান ডেটা ভিন্ন OSD তে যাবে (OSD লেভেল - ডিফল্ট)
ceph osd crush rule create-replicated osd-rule host osd

# আমাদের জন্য:
# ✅ host-level rule ভালো (একই নোডের ২টি ডিস্ক ফেল করলেও ডেটা নিরাপদ)
ceph osd crush rule create-replicated safe-rule host host
ceph osd pool set ssd-pool crush_rule safe-rule
```

### **Dashboard থেকে CRUSH Ruleset**:

```
Pool Creation Form → CRUSH Ruleset:

┌─────────────────────────────────┐
│ CRUSH Ruleset: [▼ replicated_rule] │
│                                  │
│ Options:                         │
│ • replicated_rule  (ডিফল্ট)      │
│ • ssd-rule         (আমাদের তৈরি) │
│ • hdd-rule         (আমাদের তৈরি) │
│ • safe-rule        (host-level) │
└─────────────────────────────────┘

✅ সুপারিশ: ssd-pool এর জন্য ssd-rule
           hdd-pool এর জন্য hdd-rule
```

---

## **৩.৭ Compression Mode - বাস্তব উদাহরণ**

### **থিওরি: Compression কী এবং কেন দরকার?**

```
🗜️ Compression = ডেটা ছোট করে সেভ করা

✅ সুবিধা:
- কম ডিস্ক স্পেস ব্যবহার
- কম নেটওয়ার্ক ব্যান্ডউইথ
- বিশেষ করে HDD তে ভালো কাজ করে

❌ অসুবিধা:
- CPU ব্যবহার বাড়ে
- SSD তে পারফরম্যান্স লস হতে পারে

🎯 Compression Algorithm:
1️⃣ snappy: ফাস্ট, কম কমপ্রেশন (ডিফল্ট)
2️⃣ zlib: মাঝারি স্পিড, ভালো কমপ্রেশন
3️⃣ zstd: ভালো ব্যালেন্স (সুপারিশকৃত)
4️⃣ lz4: খুব ফাস্ট, কম কমপ্রেশন
```

### **প্র্যাকটিক্যাল: Compression Mode সেটিংস**

#### **Compression Mode এর ৪টি অপশন**:

```bash
# ১. none (ডিফল্ট - সুপারিশকৃত SSD এর জন্য)
#    - কোনো কমপ্রেশন হবে না
#    - সর্বোচ্চ পারফরম্যান্স
#    - বেশি জায়গা লাগবে

ceph osd pool set ssd-pool compression_mode none

# ২. passive (সুপারিশকৃত HDD এর জন্য)
#    - ক্লাইন্ট যদি compressible hint দেয়, তাহলেই compress করবে
#    - Database, VM ডিস্কের জন্য ভালো
#    - Moderate CPU ব্যবহার

ceph osd pool set hdd-pool compression_mode passive
ceph osd pool set hdd-pool compression_algorithm zstd

# ৩. aggressive
#    - ক্লাইন্ট যদি non-compressible hint না দেয়, তাহলে compress করবে
#    - Text files, logs এর জন্য ভালো
#    - বেশি CPU ব্যবহার

ceph osd pool set archive-pool compression_mode aggressive

# ৪. force
#    - সব ডেটা compress করবে (যে কোনো মূল্যে)
#    - Archive, backup এর জন্য
#    - সর্বোচ্চ CPU ব্যবহার, সর্বোচ্চ স্টোরেজ সাশ্রয়

ceph osd pool set backup-pool compression_mode force
ceph osd pool set backup-pool compression_algorithm zstd
```

### **আমাদের ক্লাস্টারের জন্য সুপারিশ**:

```bash
# ssd-pool (Production VMs):
ceph osd pool set ssd-pool compression_mode none
# কারণ: SSD ফাস্ট, কমপ্রেশন পারফরম্যান্স কমাবে

# hdd-pool (Backups):
ceph osd pool set hdd-pool compression_mode passive
ceph osd pool set hdd-pool compression_algorithm zstd
ceph osd pool set hdd-pool compression_required_ratio 0.85
# কারণ: HDD তে স্পেস সাশ্রয় গুরুত্বপূর্ণ, zstd ভালো ব্যালেন্স দেয়

# ভেরিফাই করুন:
ceph osd pool get hdd-pool compression_mode
ceph osd pool get hdd-pool compression_algorithm
```

### **Compression Stats চেক করা**:

```bash
# Pool এর কমপ্রেশন স্ট্যাটাস
ceph osd pool stats hdd-pool | grep compress

# Output উদাহরণ:
# compress_bytes: 50 GiB
# compress_original_bytes: 80 GiB  
# compress_efficiency: 37.5%  ← 37.5% স্পেস সেভ হয়েছে!

# OSD লেভেলে কমপ্রেশন
ceph osd df | grep compress
```

---

## **৩.৮ Quotas (কোটা) - বাস্তব উদাহরণ**

### **থিওরি: Quota কেন দরকার?**

```
📏 Quota = Pool এর সর্বোচ্ক ব্যবহার লিমিট

✅ কেন দরকার:
1️⃣ এক pool অন্য pool এর স্পেস না খায়
2️⃣ Unexpected growth থেকে বাঁচায়
3️⃣ Multi-tenant environment এ ফেয়ার ইউজেজ নিশ্চিত করে
4️⃣ Budget planning এ সাহায্য করে

📊 Quota Types:
1️⃣ Max Bytes: সর্বোচ্চ কত জিবি/টিবি ব্যবহার করবে
2️⃣ Max Objects: সর্বোচ্চ কতটি ফাইল/অবজেক্ট থাকবে
```

### **প্র্যাকটিক্যাল: Quota সেট করা**

#### **সিনারিও ১: VM Pool এর জন্য Quota**

```bash
# ssd-pool: ম্যাক্সিমাম 200GB ব্যবহার করবে
# (আমাদের মোট SSD space 600GB, replication 3x = 200GB usable)

ceph osd pool set-quota ssd-pool max_bytes 214748364800
# 214748364800 bytes = 200 GiB

# ভেরিফাই করুন:
ceph osd pool get-quota ssd-pool
# Output:
# max_bytes: 214748364800
# max_objects: unset
```

#### **সিনারিও ২: Backup Pool এর জন্য Quota**

```bash
# hdd-pool: ম্যাক্সিমাম 600GB ব্যবহার করবে
# (আমাদের মোট HDD space 1.8TB, replication 3x = 600GB usable)

ceph osd pool set-quota hdd-pool max_bytes 644245094400
# 644245094400 bytes = 600 GiB

# Object limit (ঐচ্ছিক):
# যদি ছোট ফাইল বেশি হয়, object count লিমিট করতে পারেন
ceph osd pool set-quota hdd-pool max_objects 1000000
# ১০ লক্ষ অবজেক্ট
```

#### **সিনারিও ৩: Quota Update/Remove**

```bash
# Quota আপডেট করতে (বড় করতে)
ceph osd pool set-quota ssd-pool max_bytes 429496729600
# এখন 400 GiB

# Quota remove করতে
ceph osd pool rm-quota ssd-pool max_bytes
ceph osd pool rm-quota ssd-pool max_objects

# সব quota একসাথে দেখুন
ceph osd dump | grep quota
```

### **Dashboard থেকে Quota সেট করা**:

```
Pool Edit Form → Quotas:

┌─────────────────────────────────┐
│ ☑ Enable Quotas                │
│                                  │
│ Max Bytes: [200] [▼ GiB]      │ ← ড্রপডাউন: KiB, MiB, GiB, TiB
│ Max Objects: [0]               │ ← 0 = unlimited
└─────────────────────────────────┘

✅ টিপস:
- সর্বদা কিছু বাফার রাখুন (১০-২০%)
- উদাহরণ: usable space 200GB হলে quota দিন 180GB
```

### **Quota Alert ও মনিটরিং**:

```bash
# Quota接近 হলে Ceph warning দেয়
ceph health detail | grep quota

# Pool usage চেক করুন নিয়মিত
ceph osd pool stats ssd-pool

# Output উদাহরণ:
# bytes_used: 150 GiB / 200 GiB (75%)
# ⚠️ 80% এর উপরে গেলে অ্যালার্ট আসবে

# অটোমেটেড মনিটরিং (ঐচ্ছিক):
# Prometheus + Grafana দিয়ে dashboard বানাতে পারেন
```

---

## **৩.৯ RBD Configuration - Quality of Service (QoS)**

### **থিওরি: RBD QoS কী এবং কেন দরকার?**

```
⚡ QoS = Quality of Service = পারফরম্যান্স গ্যারান্টি

🎯 কেন দরকার:
1️⃣ Multi-tenant environment: এক VM অন্যটার পারফরম্যান্স না খায়
2️⃣ Noisy neighbor সমস্যা সমাধান: একটি VM বেশি লোড দিলে অন্যটা ধীর না হয়
3️⃣ Critical application: Database, Production VM এর জন্য guaranteed performance

📊 QoS Parameters:
1️⃣ BPS Limit (Bytes Per Second): মোট throughput লিমিট
2️⃣ IOPS Limit: প্রতি সেকেন্ডে কতটি read/write অপারেশন
3️⃣ Read/Write আলাদা লিমিট: read-heavy বা write-heavy workload এর জন্য
```

### **প্র্যাকটিক্যাল: RBD Image এ QoS সেট করা**

> ⚠️ **গুরুত্বপূর্ণ**: QoS Pool level এ নয়, **Image level** এ সেট করতে হয়!

#### **সিনারিও ১: Production Database VM**

```bash
# ১. RBD image তৈরি করুন
rbd create ssd-pool/prod-db-1 --size 100G

# ২. High Performance QoS সেট করুন
# IOPS Limit: প্রতি সেকেন্ডে 5000 অপারেশন
rbd image-meta set ssd-pool/prod-db-1 rbd_qos_iops_limit 5000

# Read/Write আলাদা লিমিট
rbd image-meta set ssd-pool/prod-db-1 rbd_qos_read_iops_limit 3000
rbd image-meta set ssd-pool/prod-db-1 rbd_qos_write_iops_limit 2000

# BPS Limit: মোট 100MB/s throughput
rbd image-meta set ssd-pool/prod-db-1 rbd_qos_bps_limit 104857600
# 104857600 bytes = 100 MB

# Read/Write আলাদা BPS
rbd image-meta set ssd-pool/prod-db-1 rbd_qos_read_bps_limit 73400320   # 70 MB/s
rbd image-meta set ssd-pool/prod-db-1 rbd_qos_write_bps_limit 31457280  # 30 MB/s
```

#### **সিনারিও ২: Development/Test VM**

```bash
# ১. RBD image তৈরি
rbd create hdd-pool/dev-vm-1 --size 50G

# ২. Low Priority QoS (অন্য VM কে disturb না করে)
rbd image-meta set hdd-pool/dev-vm-1 rbd_qos_iops_limit 1000
rbd image-meta set hdd-pool/dev-vm-1 rbd_qos_bps_limit 52428800  # 50 MB/s
```

#### **সিনারিও ৩: Backup/Archive VM**

```bash
# ১. RBD image তৈরি
rbd create hdd-pool/backup-vm-1 --size 200G

# ২. Write-focused QoS (backup শুধু write করে)
rbd image-meta set hdd-pool/backup-vm-1 rbd_qos_write_iops_limit 500
rbd image-meta set hdd-pool/backup-vm-1 rbd_qos_write_bps_limit 26214400  # 25 MB/s
# Read limit না দিলে unlimited থাকবে (restore এর সময় দরকার)
```

### **QoS Settings চেক ও ম্যানেজ করা**:

```bash
# ১. Image এর সব QoS settings দেখুন
rbd image-meta list ssd-pool/prod-db-1 | grep qos

# Output:
# rbd_qos_iops_limit = 5000
# rbd_qos_read_iops_limit = 3000
# rbd_qos_write_iops_limit = 2000
# rbd_qos_bps_limit = 104857600
# ...

# ২. নির্দিষ্ট setting চেক করুন
rbd image-meta get ssd-pool/prod-db-1 rbd_qos_iops_limit
# Output: 5000

# ৩. QoS update করুন
rbd image-meta set ssd-pool/prod-db-1 rbd_qos_iops_limit 8000

# ৪. QoS remove করুন
rbd image-meta remove ssd-pool/prod-db-1 rbd_qos_iops_limit
```

### **Dashboard থেকে QoS সেট করা**:

```
Dashboard → RBD → Images → Create Image → Advanced Options → QoS:

┌─────────────────────────────────┐
│ ☑ Enable QoS Limits            │
│                                  │
│ IOPS Limit:      [5000]        │
│ Read IOPS Limit: [3000]        │
│ Write IOPS Limit:[2000]        │
│                                  │
│ BPS Limit:       [100] [MB/s]  │
│ Read BPS Limit:  [70]  [MB/s]  │
│ Write BPS Limit: [30]  [MB/s]  │
└─────────────────────────────────┘
```

### **QoS মনিটরিং ও ট্রাবলশুটিং**:

```bash
# ১. Image পারফরম্যান্স মনিটর করুন
rbd top ssd-pool/prod-db-1

# Output উদাহরণ:
# WATCHERS  IOPS   BPS        LATENCY
# 1         4500   95 MB/s    2.1 ms  ← QoS limit এর কাছাকাছি!

# ২. যদি performance কম মনে হয়:
# ক) QoS limit বেশি কিনা চেক করুন
# খ) OSD পারফরম্যান্স চেক করুন
ceph osd perf

# গ) নেটওয়ার্ক লেটেন্সি চেক করুন
ping -c 10 ceph2

# ৩. QoS এর কারণে throttling হচ্ছে কিনা চেক করুন
ceph daemon osd.0 perf dump | grep qos
```

---

# **🎯 ধাপ ৪: সম্পূর্ণ Pool Creation Workflow - এক নজরে**

## **৪.১ Dashboard দিয়ে Pool তৈরি - সম্পূর্ণ ফ্লো**

```
১. Login to Dashboard: https://<node-ip>:8443

২. Navigate: Ceph Cluster → Pools → Create Pool

৩. Basic Settings:
   ┌─────────────────────────┐
   │ Pool Name: ssd-pool    │
   │ Pool Type: ● Replicated│
   └─────────────────────────┘

৪. PG Settings:
   ┌─────────────────────────┐
   │ PG Autoscale: ● on    │
   │ (Advanced: pg_num: 128)│
   └─────────────────────────┘

৫. Replication:
   ┌─────────────────────────┐
   │ Replicated Size: [3]  │
   │ Min Size: [2]         │
   └─────────────────────────┘

৬. Application:
   ┌─────────────────────────┐
   │ ☑ rbd                  │
   └─────────────────────────┘

৭. CRUSH Ruleset:
   ┌─────────────────────────┐
   │ [▼ ssd-rule]          │
   └─────────────────────────┘

৮. Compression:
   ┌─────────────────────────┐
   │ Mode: [▼ none]        │
   └─────────────────────────┘

৯. Quotas:
   ┌─────────────────────────┐
   │ ☑ Enable Quotas       │
   │ Max Bytes: [200] GiB  │
   └─────────────────────────┘

১০. Create → Confirm → Done! ✅
```

## **৪.২ CLI দিয়ে একই Pool তৈরি - সম্পূর্ণ কমান্ড**

```bash
#!/bin/bash
# create-ssd-pool.sh

POOL_NAME="ssd-pool"
PG_NUM=128
RULE_NAME="ssd-rule"

# ১. CRUSH rule তৈরি (যদি না থাকে)
ceph osd crush rule ls | grep -q $RULE_NAME || \
  ceph osd crush rule create-replicated $RULE_NAME host ssd

# ২. Pool তৈরি
ceph osd pool create $POOL_NAME $PG_NUM $PG_NUM $RULE_NAME

# ৩. Application enable
ceph osd pool application enable $POOL_NAME rbd

# ৪. PG Autoscale on
ceph osd pool set $POOL_NAME pg_autoscale_mode on

# ৫. Replication settings
ceph osd pool set $POOL_NAME size 3
ceph osd pool set $POOL_NAME min_size 2

# ৬. Compression off (SSD এর জন্য)
ceph osd pool set $POOL_NAME compression_mode none

# ৭. Quota set
ceph osd pool set-quota $POOL_NAME max_bytes 214748364800  # 200 GiB

# ৮. ভেরিফিকেশন
echo "=== Pool Created Successfully ==="
ceph osd pool ls detail | grep $POOL_NAME
ceph osd pool autoscale-status | grep $POOL_NAME
```

---

# **🔍 ধাপ ৫: ভেরিফিকেশন ও মনিটরিং**

## **৫.১ Pool তৈরির পর চেকলিস্ট**

```bash
# ১. Pool লিস্টে আছে কিনা
ceph osd pool ls detail | grep ssd-pool

# ২. PG Autoscale status
ceph osd pool autoscale-status
# Output: ssd-pool  128  ok  128  ← ✅

# ৩. PG distribution balanced কিনা
ceph pg stat-by-osd | head -15

# ৪. Cluster health
ceph -s
# HEALTH_OK দেখতে হবে

# ৫. OSD usage balanced কিনা
ceph osd df tree
# সব OSD তে প্রায় সমান %USE থাকতে হবে

# ৬. Quota working কিনা
ceph osd pool stats ssd-pool | grep bytes_used
```

## **৫.২ রিয়েল-ওয়ার্ল্ড টেস্ট: RBD Image তৈরি ও ব্যবহার**

```bash
# ১. নতুন RBD image তৈরি (VM ডিস্ক)
rbd create ssd-pool/prod-vm-1 --size 50G --image-feature layering,fast-diff

# ২. Image info চেক করুন
rbd info ssd-pool/prod-vm-1

# Output:
# rbd image 'prod-vm-1':
#     size 50 GiB in 1280 objects
#     order 22 (4 MiB objects)
#     block_name_prefix: ssd-pool.12345
#     format: 2
#     features: layering, fast-diff
#     flags: 

# ৩. Image map করুন (Proxmox/KVM এর জন্য)
rbd map ssd-pool/prod-vm-1
# Output: /dev/rbd0

# ৪. ফাইলসিস্টেম তৈরি করুন
mkfs.ext4 /dev/rbd0

# ৫. মাউন্ট করুন
mkdir /mnt/vm-disk
mount /dev/rbd0 /mnt/vm-disk

# ৬. টেস্ট ডেটা লিখুন
dd if=/dev/zero of=/mnt/vm-disk/testfile bs=1M count=1000
# 1GB ডেটা লিখলাম

# ৭. পারফরম্যান্স চেক করুন
rbd bench --io-type write ssd-pool/prod-vm-1 --io-size 4M

# Output:
# bench write io_size: 4194304 io_total: 107374182400 bps: 245.6MiB/s iops: 61.4 avg_lat: 16.3ms

# ৮. Unmap করুন
umount /mnt/vm-disk
rbd unmap /dev/rbd0
```

---

# **🚨 ধাপ ৬: কমন সমস্যা ও সমাধান**

## **সমস্যা ১: Pool create করার পর "HEALTH_WARN" আসছে**

```bash
# সম্ভাব্য কারণ ও সমাধান:

# কারণ ১: PG Autoscale এখনও calculate করছে
ceph osd pool autoscale-status
# যদি "suggesting" দেখায়, 30 মিনিট অপেক্ষা করুন

# কারণ ২: PG imbalance
ceph health detail | grep PG
# সমাধান: 
ceph osd pool set ssd-pool pg_num 128
ceph osd pool set ssd-pool pgp_num 128

# কারণ ৩: OSD weight imbalance
ceph osd df tree
# সমাধান:
ceph osd crush reweight osd.6 0.20  # SSD এর জন্য
```

## **সমস্যা ২: RBD image map করতে পারছি না**

```bash
# সম্ভাব্য কারণ:

# কারণ ১: kernel module লোড নেই
lsmod | grep rbd
# যদি না থাকে:
modprobe rbd

# কারণ ২: pool এ application enable করা নেই
ceph osd pool application get ssd-pool
# যদি empty থাকে:
ceph osd pool application enable ssd-pool rbd

# কারণ ৩: image feature mismatch
rbd info ssd-pool/prod-vm-1 | grep features
# পুরনো kernel এ fast-diff সাপোর্ট নাও করতে পারে
# সমাধান: image-feature ছাড়া তৈরি করুন
rbd create ssd-pool/compat-vm --size 50G
```

## **সমস্যা ৩: Pool quota পেরিয়ে গেছে, write fail হচ্ছে**

```bash
# চেক করুন:
ceph osd pool stats ssd-pool | grep bytes_used

# সমাধান ১: Quota বাড়ান
ceph osd pool set-quota ssd-pool max_bytes 429496729600  # 400 GiB

# সমাধান ২: পুরনো ডেটা ডিলিট করুন
rbd rm ssd-pool/old-vm-1

# সমাধান ৩: নতুন pool তৈরি করুন
ceph osd pool create ssd-pool-2 128 128 ssd-rule
ceph osd pool application enable ssd-pool-2 rbd
```

---

# **📋 ধাপ ৭: ডকুমেন্টেশন ও ব্যাকআপ**

## **৭.১ কনফিগারেশন ব্যাকআপ**

```bash
# ১. Pool configuration backup
ceph osd dump > /backup/ceph-osd-dump-$(date +%F).txt
ceph osd pool ls detail > /backup/ceph-pools-$(date +%F).txt

# ২. CRUSH map backup
ceph osd crush dump > /backup/ceph-crush-$(date +%F).json

# ৩. RBD images backup list
rbd ls -l ssd-pool > /backup/rbd-ssd-pool-$(date +%F).txt
rbd ls -l hdd-pool > /backup/rbd-hdd-pool-$(date +%F).txt

# ৪. Automate with cron (প্রতিদিন রাত ২টায়)
# /etc/cron.d/ceph-backup
0 2 * * * root /usr/local/bin/ceph-config-backup.sh
```

## **৭.২ Disaster Recovery Plan**

```bash
# Scenario: পুরো ক্লাস্টার রিকভার করতে হবে

# ১. Monitors restore (সবচেয়ে গুরুত্বপূর্ণ)
ceph-mon -i ceph1 --mkfs --monmap /backup/monmap
ceph-mon -i ceph1 --inject-monmap /backup/monmap

# ২. OSDs restore
# প্রতিটি OSD এর জন্য:
ceph-volume lvm activate <osd-id> <osd-fsid>

# ৩. Pool metadata restore (যদি দরকার হয়)
# (সাধারণত দরকার হয় না, Ceph অটোমেটিক রিকভার করে)

# ৪. RBD images restore
# যদি snapshot backup থাকে:
rbd import /backup/vm-disk-1.img ssd-pool/vm-disk-1-restored
```

---

# **🎓 বোনাস: প্রোডাকশন টিপস**

## **টিপ ১: নিয়মিত মনিটরিং স্ক্রিপ্ট**

```bash
#!/bin/bash
# ceph-health-check.sh

echo "=== Ceph Health Check - $(date) ==="

# 1. Overall health
HEALTH=$(ceph -s | grep health | awk '{print $2}')
if [ "$HEALTH" != "HEALTH_OK" ]; then
    echo "⚠️  ALERT: $HEALTH"
    ceph health detail
fi

# 2. OSD status
DOWN_OSDS=$(ceph osd tree | grep down | wc -l)
if [ $DOWN_OSDS -gt 0 ]; then
    echo "⚠️  ALERT: $DOWN_OSDS OSDs are down!"
fi

# 3. Pool usage
echo -e "\n📊 Pool Usage:"
ceph osd pool stats | grep -E "ssd-pool|hdd-pool" | grep bytes_used

# 4. PG status
PG_NOT_CLEAN=$(ceph pg stat | grep -o '[0-9]* pgs not clean' | grep -o '[0-9]*')
if [ -n "$PG_NOT_CLEAN" ] && [ $PG_NOT_CLEAN -gt 0 ]; then
    echo "⚠️  ALERT: $PG_NOT_CLEAN PGs not clean"
fi

# 5. Quota warning
echo -e "\n⚠️  Quota Warnings:"
ceph osd pool stats | grep -B1 "bytes_used.*[89][0-9]%"
```

## **টিপ ২: Performance Tuning Quick Reference**

```bash
# SSD Pool এর জন্য অপ্টিমাইজেশন:
ceph config set osd osd_op_threads 8
ceph config set osd osd_disk_threads 4
ceph config set osd osd_client_message_size_cap 512M

# HDD Pool এর জন্য অপ্টিমাইজেশন:
ceph config set osd osd_op_threads 4
ceph config set osd osd_disk_threads 2
ceph config set osd osd_max_write_size 128

# Global settings (সতর্কতার সাথে):
ceph config set global osd_pool_default_pg_autoscale_mode on
ceph config set global mon_osd_full_ratio 0.95
ceph config set global mon_osd_backfillfull_ratio 0.90
ceph config set global mon_osd_nearfull_ratio 0.85
```

## **টিপ ৩: Dashboard Customization**

```
Dashboard → Manager Modules → Dashboard → Settings:

✅ Enable:
- Pool management
- RBD management
- OSD management
- Performance counters

✅ Customize:
- Refresh interval: 30 seconds
- Default pool view: ssd-pool
- Alert threshold: 80% for quota

✅ Create Custom Dashboard:
Dashboard → Dashboards → Create
- Add widgets: Pool Usage, OSD Status, PG Health
- Set auto-refresh: 1 minute
- Share with team: Read-only access
```

---

# **🏁 সমাপ্তি: আমাদের ক্লাস্টার এখন প্রোডাকশন রেডি!**

```
✅ যা আমরা করেছি:
1️⃣ ক্লাস্টার হেলথ চেক করেছি
2️⃣ OSD গুলো ম্যানেজ ও মনিটর করতে শিখেছি
3️⃣ দুটি আলাদা Pool তৈরি করেছি (SSD + HDD)
4️⃣ PG Autoscale সঠিকভাবে কনফিগার করেছি
5️⃣ CRUSH ruleset দিয়ে ডেটা ডিস্ট্রিবিউশন অপ্টিমাইজ করেছি
6️⃣ Compression, Quota, QoS সেটিংস অ্যাপ্লাই করেছি
7️⃣ রিয়েল-ওয়ার্ল্ড টেস্ট করে পারফরম্যান্স ভেরিফাই করেছি
8️⃣ মনিটরিং ও ব্যাকআপ স্ক্রিপ্ট সেটআপ করেছি

🎯 পরবর্তী ধাপ:
- Proxmox বা Kubernetes এ Ceph RBD integrate করুন
- Backup strategy implement করুন (RBD snapshot + export)
- Monitoring stack (Prometheus + Grafana) সেটআপ করুন
- Regular maintenance schedule তৈরি করুন

🔧 কমান্ড শিট (দ্রুত রেফারেন্স):
# Health check
ceph -s

# Pool list
ceph osd pool ls detail

# Create RBD image
rbd create <pool>/<image> --size <GB>G

# Map image
rbd map <pool>/<image>

# Monitor performance
rbd top <pool>/<image>

# Emergency: OSD out
ceph osd out osd.X

📞 Support:
- Official Docs: https://docs.ceph.com/en/squid/
- Community: https://discuss.ceph.com/
- Logs: ceph orch logs <service>
```

> *"এই গাইডটি অনুসরণ করে আপনি আপনার ৩-নোড Ceph ক্লাস্টারকে একটি প্রোডাকশন-গ্রেড স্টোরেজ সলিউশনে রূপান্তর করতে পারবেন। মনে রাখবেন, Ceph একটি পাওয়ারফুল কিন্তু কমপ্লেক্স সিস্টেম - ধৈর্য ধরে টেস্ট করুন, ডকুমেন্ট করুন, এবং নিয়মিত ব্যাকআপ নিন।"*

**শুভকামনা আপনার Ceph জার্নির জন্য!** 🚀🐧