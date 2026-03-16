# Ceph ক্লাস্টার ট্রাবলশুটিং এবং প্রোডাকশন মেইনটেন্যান্স গাইড

## 📋 সূচিপত্র

1. [ভূমিকা এবং প্রেক্ষাপট](#ভূমিকা-এবং-প্রেক্ষাপট)
2. [প্রাথমিক সমস্যা চিহ্নিতকরণ](#প্রাথমিক-সমস্যা-চিহ্নিতকরণ)
3. [Ceph ক্লাস্টার স্বাস্থ্য সমস্যা সমাধান](#ceph-ক্লাস্টার-স্বাস্থ্য-সমস্যা-সমাধান)
4. [Zabbix কনফিগারেশন ওয়ার্নিং রিমুভ](#zabbix-কনফিগারেশন-ওয়ার্নিং-রিমুভ)
5. [প্রোডাকশন OSD মেইনটেন্যান্স](#প্রোডাকশন-osd-মেইনটেন্যান্স)
6. [Dashboard বনাম CLI পদ্ধতি](#dashboard-বনাম-cli-পদ্ধতি)
7. [নিরাপদ OSD রিস্টার্ট প্রক্রিয়া](#নিরাপদ-osd-রিস্টার্ট-প্রক্রিয়া)
8. [ট্রাবলশুটিং এবং ডায়াগনোসিস](#ট্রাবলশুটিং-এবং-ডায়াগনোসিস)
9. [প্রতিরোধমূলক ব্যবস্থা](#প্রতিরোধমূলক-ব্যবস্থা)
10. [জরুরি সহায়তা এবং রেফারেন্স](#জরুরি-সহায়তা-এবং-রেফারেন্স)

---

## ভূমিকা এবং প্রেক্ষাপট

### এই গাইড কেন তৈরি করা হলো

এই ডকুমেন্টটি একটি বাস্তব প্রোডাকশন Ceph ক্লাস্টারের সমস্যা সমাধানের অভিজ্ঞতা থেকে তৈরি করা হয়েছে। আমরা একটি ৭-নোডের Ceph ক্লাস্টার পরিচালনা করছি যা প্রোডাকশন এনভায়রনমেন্টে চলমান। ক্লাস্টারটিতে একাধিক সমস্যা দেখা দেয় যার মধ্যে রয়েছে:

- **HEALTH_WARN** স্ট্যাটাস (Zabbix সার্ভার কনফিগারেশন সমস্যা)
- **CephDaemonCrash** এরর
- **CephadmPaused** এবং **CephadmDaemonFailed** অ্যালার্ট
- **Manager ডিমন** সমস্যা
- **flash1.meghnacloud.com** নোডে ১১+ OSD ডিমন বন্ধ হয়ে যাওয়া

এই গাইডটি ধাপে ধাপে দেখাবে কীভাবে এই সমস্যাগুলো চিহ্নিত করা হলো, কীভাবে ডায়াগনোসিস করা হলো, এবং সবচেয়ে গুরুত্বপূর্ণভাবে - **প্রোডাকশন এনভায়রনমেন্টে নিরাপদে** কীভাবে সমাধান করা হলো।

### ক্লাস্টার আর্কিটেকচার

```
ক্লাস্টার তথ্য:
- মোট নোড: ৭টি
- Ceph ভার্সন: 18.2.0 reef (stable) / 19.2.3 squid (stable)
- অর্কেস্ট্রেটর: cephadm
- মোনিটর: ৩টি (ceph1, ceph2, ceph3)
- ম্যানেজার: ২টি (active + standby)
- OSD: ১৪৬টি (১৩৪টি active, ১২টি down, ১২টি out)
- স্টোরেজ ক্যাপাসিটি: ১.৮ TiB - ১.২ PiB
```

### টার্গেট পাঠক

- System Administrators
- DevOps Engineers
- Storage Engineers
- IT Infrastructure Team
- যারা Ceph ক্লাস্টার ম্যানেজ করেন

---

## প্রাথমিক সমস্যা চিহ্নিতকরণ

### প্রথম ধাপ: ক্লাস্টার স্বাস্থ্য চেক

যখনই Ceph ক্লাস্টারে কোনো সমস্যা দেখা দেয়, প্রথম কাজ হলো ক্লাস্টারের বর্তমান অবস্থা জানা। নিচের কমান্ডগুলো দিয়ে শুরু করুন:

```bash
ceph health
```

**কমান্ড ব্যাখ্যা:** এই কমান্ডটি ক্লাস্টারের সামগ্রিক স্বাস্থ্য দেখায়। তিনটি স্ট্যাটাস হতে পারে:
- `HEALTH_OK` - সবকিছু ঠিক আছে
- `HEALTH_WARN` - কিছু সমস্যা আছে কিন্তু ক্লাস্টার কাজ করছে
- `HEALTH_ERR` - গুরুতর সমস্যা, তাৎক্ষণিক মনোযোগ প্রয়োজন

আমাদের ক্ষেত্রে আউটপুট ছিল:
```
HEALTH_WARN
No Zabbix server configured
```

```bash
ceph -s
```

**কমান্ড ব্যাখ্যা:** এটি ক্লাস্টারের বিস্তারিত স্ট্যাটাস দেখায়। এখানে আপনি পাবেন:
- ক্লাস্টার ID
- হেলথ স্ট্যাটাস
- সার্ভিসেস (mon, mgr, osd) স্ট্যাটাস
- ডাটা পুল, অবজেক্ট, ক্যাপাসিটি তথ্য
- PG (Placement Group) স্ট্যাটাস

```bash
ceph health detail
```

**কমান্ড ব্যাখ্যা:** এটি HEALTH_WARN বা HEALTH_ERR এর বিস্তারিত কারণ দেখায়। প্রতিটি ওয়ার্নিং বা এরর এর জন্য নির্দিষ্ট তথ্য দেয়।

### দ্বিতীয় ধাপ: অ্যালার্ট এবং লগ চেক

Ceph Dashboard এ গিয়ে Observability সেকশনে অ্যালার্ট এবং লগ চেক করা জরুরি:

**Dashboard নেভিগেশন:**
```
Dashboard → Observability → Alerts → Active Alerts
Dashboard → Observability → Logs
```

**লগ চেক করার কমান্ড:**

```bash
# রিয়েল-টাইম লগ মনিটরিং
tail -f /var/log/ceph/ceph.log

# শেষ ১০০ লাইন লগ দেখা
tail -100 /var/log/ceph/ceph.log

# নির্দিষ্ট সময়ের লগ
journalctl -u ceph-mon --since "1 hour ago"
journalctl -u ceph-mgr --since "30 minutes ago"
journalctl -u ceph-osd@108 --since "2 hours ago"
```

### তৃতীয় ধাপ: ডিমন স্ট্যাটাস চেক

```bash
# সব ডিমনের স্ট্যাটাস
ceph orch ps

# নির্দিষ্ট হোস্টের ডিমন
ceph orch ps --host=flash1

# নির্দিষ্ট টাইপের ডিমন
ceph orch ps --daemon_type=osd
ceph orch ps --daemon_type=mgr
ceph orch ps --daemon_type=mon

# JSON ফরম্যাটে (স্ক্রিপ্টিং এর জন্য)
ceph orch ps --format json-pretty
```

**আমাদের চিহ্নিত সমস্যা:**
```
flash1.meghnacloud.com নোডে:
- osd.108: stopped
- osd.109: stopped
- osd.110: stopped
- osd.111: stopped
- osd.112: stopped
- osd.113: stopped
- osd.115: stopped
- osd.116: stopped
- osd.117: stopped
(মোট ১১+ OSD stopped)

শুধুমাত্র osd.114 running
```

### চতুর্থ ধাপ: হোস্ট এবং সার্ভিস ইনভেন্টরি

```bash
# সব হোস্টের লিস্ট
ceph orch host ls

# নির্দিষ্ট হোস্টের ডিটেইলস
ceph orch host ls flash1

# সব সার্ভিসের লিস্ট
ceph orch ls

# ডিভাইস ইনভেন্টরি
ceph orch device ls
ceph orch device ls flash1
```

---

## Ceph ক্লাস্টার স্বাস্থ্য সমস্যা সমাধান

### সমস্যা ১: CephDaemonCrash

**লক্ষণ:**
```
Active Alerts এ দেখা যাচ্ছে:
CephDaemonCrash - critical severity
"One or more Ceph daemons have crashed, and are pending acknowledgement"
```

**কারণ:**
- কোনো ডিমন ক্র্যাশ করেছে (mgr, mon, osd, mds ইত্যাদি)
- ক্র্যাশ রিপোর্ট স্বীকার (acknowledge) করা হয়নি
- মেমোরি আউট-অফ-মেমোরি (OOM) সমস্যা
- কনফিগারেশন এরর

**সমাধান ধাপ:**

**ধাপ ১: ক্র্যাশ লিস্ট চেক করুন**

```bash
ceph crash ls
```

এই কমান্ডটি সব ক্র্যাশ রিপোর্টের লিস্ট দেখাবে। প্রতিটি ক্র্যাশ এর একটি unique ID থাকে।

**ধাপ ২: নির্দিষ্ট ক্র্যাশ ডিটেইলস দেখুন**

```bash
ceph crash info <crash_id>
```

এখানে `<crash_id>` এর জায়গায় উপরের লিস্ট থেকে ID বসাতে হবে। এটি ক্র্যাশ এর বিস্তারিত তথ্য দেখাবে:
- কোন ডিমন ক্র্যাশ করেছে
- কখন ক্র্যাশ হয়েছে
- ক্র্যাশ এর কারণ (যদি পাওয়া যায়)

**ধাপ ৩: সব ক্র্যাশ আর্কাইভ করুন**

```bash
ceph crash archive-all
```

**কমান্ড ব্যাখ্যা:** এই কমান্ডটি সব ক্র্যাশ রিপোর্ট আর্কাইভে সরিয়ে দেয়। এটি ওয়ার্নিং রিমুভ করে কিন্তু ক্র্যাশ ডাটা মুছে ফেলে না - শুধু active alerts থেকে সরিয়ে দেয়।

**ধাপ ৪: নির্দিষ্ট ক্র্যাশ আর্কাইভ**

```bash
ceph crash archive <crash_id>
```

যদি সব ক্র্যাশ আর্কাইভ না করে নির্দিষ্ট কিছু আর্কাইভ করতে চান।

**ধাপ ৫: ক্র্যাশ প্রুনিং কনফিগার (ঐচ্ছিক)**

```bash
# ক্র্যাশ রিপোর্ট কতদিন রাখা হবে (ডিফল্ট: ৭ দিন)
ceph config set mgr mgr/crash/warn_delay 86400

# অটো-প্রুনিং ইন্টারভ্যাল (ডিফল্ট: ১ ঘণ্টা)
ceph config set mgr mgr/crash/prune_interval 3600
```

### সমস্যা ২: CephadmPaused এবং CephadmDaemonFailed

**লক্ষণ:**
```
Active Alerts:
- CephadmPaused (warning): "Orchestration tasks via cephadm are PAUSED"
- CephadmDaemonFailed (critical): "A ceph daemon managed by cephadm is down"
- CephadmUpgradeFailed (critical): "Ceph version upgrade has failed"
```

**কারণ:**
- Cephadm অর্কেস্ট্রেশন ম্যানুয়ালি বা অটোমেটিক পজ হয়েছে
- কোনো ডিমন স্টার্ট করতে ব্যর্থ হয়েছে
- আপগ্রেড প্রসেস ফেইল হয়েছে
- হোস্ট বা নেটওয়ার্ক সমস্যা

**সমাধান ধাপ:**

**ধাপ ১: অর্কেস্ট্রেশন রিজিউম করুন**

```bash
ceph orch resume
```

**কমান্ড ব্যাখ্যা:** এটি cephadm অর্কেস্ট্রেশন আবার চালু করে। যদি ম্যানুয়ালি পজ করা হয়ে থাকে বা কোনো এররের কারণে পজ হয়ে থাকে, এটি রিজিউম করে।

**ধাপ ২: অর্কেস্ট্রেশন স্ট্যাটাস চেক করুন**

```bash
ceph orch status
```

এটি দেখাবে অর্কেস্ট্রেশন active নাকি paused আছে।

**ধাপ ৩: নির্দিষ্ট হোস্টে রিজিউম**

```bash
ceph orch resume --host=<hostname>
```

যদি নির্দিষ্ট কোনো হোস্টে অর্কেস্ট্রেশন পজড থাকে।

**ধাপ ৪: ফেইল করা ডিমন চিহ্নিত করুন**

```bash
ceph orch ps | grep -E "failed|error"
```

এটি সব failed বা error স্ট্যাটাসের ডিমন দেখাবে।

**ধাপ ৫: ডিমন রিস্টার্ট করুন**

```bash
# নির্দিষ্ট ডিমন রিস্টার্ট
ceph orch restart <daemon_name>

# উদাহরণ:
ceph orch restart mgr.ceph1
ceph orch restart mon.ceph2
ceph orch restart osd.108
```

**ধাপ ৬: আপগ্রেড স্ট্যাটাস চেক**

```bash
ceph orch upgrade status
```

যদি আপগ্রেড ফেইল দেখায়, এটি বিস্তারিত তথ্য দেবে।

**ধাপ ৭: আপগ্রেড স্টপ (যদি হ্যাং হয়ে থাকে)**

```bash
ceph orch upgrade stop
```

**ধাপ ৮: আপগ্রেড লগ চেক**

```bash
journalctl -u cephadm -f --since "1 hour ago"
```

**ধাপ ৯: ম্যানুয়ালি আপগ্রেড (প্রয়োজন হলে)**

```bash
# নির্দিষ্ট ইমেজ দিয়ে আপগ্রেড
cephadm upgrade start --image quay.io/ceph/ceph:v19.2.3

# অথবা বর্তমান ভার্সনে পিন করুন
cephadm upgrade start --image quay.io/ceph/ceph:18.2.0
```

### সমস্যা ৩: Manager ডিমন সমস্যা

**লক্ষণ:**
```
Dashboard এ Inventory সেকশনে:
"2 Managers - 1 ✓ 1 !"

অর্থাৎ ২টি mgr এর মধ্যে ১টি সমস্যায়
```

**সমাধান ধাপ:**

**ধাপ ১: MGR স্ট্যাটাস চেক**

```bash
ceph mgr stat
```

এটি দেখাবে কোন mgr active এবং কোনটি standby।

**ধাপ ২: MGR ডাম্প (বিস্তারিত)**

```bash
ceph mgr dump
```

এটি MGR এর বিস্তারিত কনফিগারেশন এবং স্ট্যাটাস দেখায়।

**ধাপ ৩: MGR সার্ভিসেস**

```bash
ceph mgr services
```

এটি দেখাবে কোন MGR মডিউলগুলো active আছে (dashboard, prometheus, ইত্যাদি)।

**ধাপ ৪: স্ট্যান্ডবাই MGR রিস্টার্ট**

```bash
# cephadm দিয়ে
ceph orch restart mgr.<standby_hostname>

# systemctl দিয়ে
systemctl restart ceph-mgr@<hostname>
```

**ধাপ ৫: MGR মডিউল চেক**

```bash
# এনাবলড মডিউল লিস্ট
ceph mgr module ls

# নির্দিষ্ট মডিউল এনাবল
ceph mgr module enable <module_name>

# ড্যাশবোর্ড মডিউল
ceph mgr module enable dashboard
```

**ধাপ ৬: MGR ফেইলওভার টেস্ট**

```bash
# বর্তমান active mgr ফেইল করুন (টেস্টিং এর জন্য)
ceph mgr fail <active_mgr_name>

# standby mgr automatic active হয়ে যাবে
ceph mgr stat
```

---

## Zabbix কনফিগারেশন ওয়ার্নিং রিমুভ

### সমস্যার বিবরণ

**লক্ষণ:**
```bash
ceph health
# আউটপুট: HEALTH_WARN
#        No Zabbix server configured
```

**কারণ:**
Ceph Dashboard এ Zabbix মনিটরিং কনফিগার করা ছিল, কিন্তু Zabbix সার্ভার কনফিগার করা হয়নি বা রিমুভ করা হয়েছে। Dashboard নিয়মিত Zabbix এ ডাটা পাঠানোর চেষ্টা করে কিন্তু সার্ভার না থাকায় ওয়ার্নিং দেখায়।

### সমাধান পদ্ধতি ১: কনফিগারেশন রিমুভ (প্রস্তাবিত)

আপনি যদি Zabbix ব্যবহার না করেন, তাহলে কনফিগারেশন সম্পূর্ণ রিমুভ করে দিন।

**ধাপ ১: Zabbix কনফিগারেশন রিমুভ করুন**

```bash
ceph config rm mgr mgr/dashboard/zabbix_host
ceph config rm mgr mgr/dashboard/zabbix_url
ceph config rm mgr mgr/dashboard/zabbix_prefix
ceph config rm mgr mgr/dashboard/zabbix_api_path
ceph config rm mgr mgr/dashboard/zabbix_password
ceph config rm mgr mgr/dashboard/zabbix_username
ceph config rm mgr mgr/dashboard/zabbix_verify_ssl
```

**কমান্ড ব্যাখ্যা:** 
প্রতিটি কমান্ড একটি করে Zabbix রিলেটেড কনফিগারেশন রিমুভ করে। যদি কোনো কনফিগ না থাকে, তাহলে এরর দেখাবে না - চুপচাপ পরের কমান্ডে চলে যাবে।

**ধাপ ২: কনফিগারেশন ভেরিফাই করুন**

```bash
ceph config dump | grep -i zabbix
```

**কমান্ড ব্যাখ্যা:** এটি সব কনফিগারেশন লিস্ট করে এবং grep দিয়ে zabbix রিলেটেড কোনো কনফিগ আছে কিনা চেক করে। যদি কোনো আউটপুট না আসে, তার মানে সব কনফিগ রিমুভ হয়েছে।

**ধাপ ৩: MGR সার্ভিস রিস্টার্ট করুন**

```bash
# সঠিক MGR নাম খুঁজে বের করুন
ceph orch ls

# MGR সার্ভিস রিস্টার্ট
ceph orch restart mgr

# অথবা systemctl দিয়ে
systemctl restart ceph-mgr@ceph1
systemctl restart ceph-mgr@ceph2
```

**কমান্ড ব্যাখ্যা:** 
MGR সার্ভিস রিস্টার্ট করলে নতুন কনফিগারেশন লোড হয়। `ceph orch ls` কমান্ডটি দেখাবে MGR সার্ভিসের সঠিক নাম কী (সাধারণত শুধু `mgr`)।

**ধাপ ৪: MGR মডিউল রিলোড (বিকল্প)**

```bash
# ড্যাশবোর্ড মডিউল ডিসএবল
ceph mgr module disable dashboard

# ৩ সেকেন্ড অপেক্ষা
sleep 3

# আবার এনাবল
ceph mgr module enable dashboard
```

**কমান্ড ব্যাখ্যা:** 
ড্যাশবোর্ড মডিউল পুরোপুরি রিলোড হয়। এটি কনফিগারেশন ক্লিয়ার করতে সাহায্য করে।

**ধাপ ৫: হেলথ চেক**

```bash
# ৫-১০ সেকেন্ড অপেক্ষা করুন
sleep 10

# হেলথ চেক করুন
ceph health
ceph -s
```

**আশা করা হচ্ছে:**
```
HEALTH_OK
```

### সমাধান পদ্ধতি ২: Dashboard UI থেকে

যদি আপনি কমান্ড লাইন ব্যবহার না করে গ্রাফিক্যালি করতে চান:

**ধাপ ১: Dashboard এ লগইন করুন**

```
https://<mgr-ip>:8443
```

**ধাপ ২: Settings এ যান**

```
Dashboard → Settings → Monitoring
```

**ধাপ ৩: Zabbix Configuration খুঁজে বের করুন**

স্ক্রল করে "Zabbix Configuration" বা "Monitoring" সেকশন খুঁজে বের করুন।

**ধাপ ৪: কনফিগারেশন ক্লিয়ার করুন**

- সব ফিল্ড খালি করুন
- অথবা "Disable" চেকবক্স টিক দিন
- "Save" বাটনে ক্লিক করুন

**ধাপ ৫: পেজ রিফ্রেশ করুন**

কিছুক্ষণ পর পেজ রিফ্রেশ করুন এবং ওয়ার্নিং চলে গেছে কিনা চেক করুন।

### সমাধান পদ্ধতি ৩: Zabbix ব্যবহার করতে চাইলে

যদি আপনি ভবিষ্যতে Zabbix ব্যবহার করতে চান, তাহলে সঠিকভাবে কনফিগার করুন:

```bash
# Zabbix হোস্ট আইপি
ceph config set mgr mgr/dashboard/zabbix_host "192.168.x.x"

# Zabbix URL
ceph config set mgr mgr/dashboard/zabbix_url "http://192.168.x.x/zabbix"

# API পাথ (ঐচ্ছিক)
ceph config set mgr mgr/dashboard/zabbix_api_path "/api_jsonrpc.php"

# ইউজারনেম এবং পাসওয়ার্ড
ceph config set mgr mgr/dashboard/zabbix_username "admin"
ceph config set mgr mgr/dashboard/zabbix_password "your_password"

# SSL ভেরিফিকেশন (Zabbix HTTPS হলে)
ceph config set mgr mgr/dashboard/zabbix_verify_ssl "true"
```

### ট্রাবলশুটিং: যদি ওয়ার্নিং না যায়

যদি উপরের সব করার পরেও `HEALTH_WARN` থাকে:

**ধাপ ১: MGR ডিমন ফোর্স রিস্টার্ট**

```bash
# MGR fail করুন
ceph mgr fail ceph1.gmpclf
ceph mgr fail ceph2.btupjx

# ৫ সেকেন্ড অপেক্ষা
sleep 5

# MGR আবার active হবে অটোমেটিক
ceph mgr stat
```

**ধাপ ২: সব mgr কনফিগ রিসেট**

```bash
# সতর্কতা: এটি সব mgr কনফিগ রিসেট করবে
ceph config reset mgr

# অথবা শুধু dashboard রিলেটেড
ceph config dump mgr | grep dashboard
```

**ধাপ ৩: লগ চেক করুন**

```bash
# ড্যাশবোর্ড লগ
tail -f /var/log/ceph/ceph-mgr.ceph1.log | grep -i zabbix

# অথবা
journalctl -u ceph-mgr@ceph1 -f | grep -i zabbix
```

**ধাপ ৪: MGR অর্কেস্ট্রেশন রি-অ্যাপ্লাই**

```bash
# MGR প্লেসমেন্ট রি-অ্যাপ্লাই
ceph orch apply mgr --placement="2 ceph1 ceph2"

# ১০ সেকেন্ড অপেক্ষা
sleep 10

# চেক করুন
ceph orch ps | grep mgr
ceph health
```

---

## প্রোডাকশন OSD মেইনটেন্যান্স

### সমস্যার বিবরণ

**flash1.meghnacloud.com (172.16.10.111) নোডে:**

```
stopped OSD গুলো:
- osd.108: stopped
- osd.109: stopped
- osd.110: stopped
- osd.111: stopped
- osd.112: stopped
- osd.113: stopped
- osd.114: running ✓
- osd.115: stopped
- osd.116: stopped
- osd.117: stopped
(এবং আরও)

মোট: ১১+ OSD stopped, শুধু ১টি running
```

### কেন এতগুলো OSD একসাথে বন্ধ?

**সম্ভাব্য কারণ:**

১. **ডিস্ক/হার্ডওয়্যার সমস্যা**
   - কন্ট্রোলার ফেইলিউর
   - পাওয়ার সমস্যা
   - কেবল/কানেকশন সমস্যা

২. **সিস্টেম সমস্যা**
   - কার্নেল প্যানিক
   - মেমোরি সমস্যা
   - ফাইলসিস্টেম এরর

৩. **নেটওয়ার্ক সমস্যা**
   - হোস্ট আনরিচেবল
   - নেটওয়ার্ক পার্টিশন

৪. **Ceph কনফিগারেশন**
   - অটোমেটিক স্টপ
   - হার্টবিট ফেইলিউর

### প্রাথমিক ডায়াগনোসিস

**ধাপ ১: হোস্টে কানেক্ট করুন**

```bash
ssh root@172.16.10.111
# অথবা
ssh root@flash1.meghnacloud.com
```

**ধাপ ২: সিস্টেম লগ চেক করুন**

```bash
# শেষ ১০০ লাইন সিস্টেম লগ
dmesg | tail -100

# ডিস্ক/স্টোরেজ রিলেটেড এরর
dmesg | grep -i "error\|fail\|sd[a-z]"

# নির্দিষ্ট সময়ের লগ
journalctl --since "2 hours ago" | grep -i "error\|fail"
```

**ধাপ ৩: ডিস্ক স্ট্যাটাস চেক**

```bash
# সব ডিস্কের লিস্ট
lsblk

# ডিস্ক ইউজেজ
df -h

# ডিস্ক হেলথ (SMART)
smartctl -H /dev/sda
smartctl -H /dev/sdb
# প্রতিটি ডিস্কের জন্য
```

**ধাপ ৪: OSD সার্ভিস স্ট্যাটাস**

```bash
# নির্দিষ্ট OSD সার্ভিস স্ট্যাটাস
systemctl status ceph-osd@108

# সব OSD সার্ভিস
systemctl status ceph-osd@*

# failed সার্ভিস
systemctl --failed | grep osd
```

**ধাপ ৫: Ceph ক্লাস্টার থেকে চেক**

```bash
# ceph1 বা অন্য নোড থেকে
ceph osd tree | grep flash1

# OSD স্ট্যাটাস
ceph osd stat

# কোন OSD down/out
ceph osd dump | grep -E "down\|out" | grep flash1
```

### OSD রিস্টার্ট: Out না করে বনাম Out করে

এটি একটি **অত্যন্ত গুরুত্বপূর্ণ** বিষয় যা প্রোডাকশন এনভায়রনমেন্টে বুঝতে হবে।

#### **পদ্ধতি ১: Out না করে সরাসরি রিস্টার্ট**

```bash
# সরাসরি রিস্টার্ট
systemctl restart ceph-osd@108
# অথবা
ceph orch restart osd.108
```

**কি ঘটবে:**

```
সময় লাইন:
t=0s:  OSD.108 running → হঠাৎ stopped
t=1s:  Monitors ডিটেক্ট করে "OSD 108 down"
t=5s:  PG গুলো degraded স্টেটে চলে যায়
t=10s: ক্লাইন্ট রিকোয়েস্ট অন্য Replica এ যায়
t=15s: OSD.108 আবার up
t=20s: PG গুলো আবার active+clean
```

**সমস্যা:**

❌ **PG Flapping:** PG গুলো বারবার degraded → active → degraded হয়
❌ **ক্লাইন্ট ইমপ্যাক্ট:** ৫-৩০ সেকেন্ডের জন্য slow request, টাইমআউট
❌ **রিকভারি স্টর্ম:** হঠাৎ নেটওয়ার্ক/ডিস্ক ইউজেশন স্পাইক
❌ **চেইন রিঅ্যাকশন:** একাধিক OSD হলে বড় সমস্যা

**কখন করতে পারেন:**

✅ টেস্ট/ডেভ এনভায়রনমেন্ট
✅ শুধু ১টি OSD, Replication >= 3
✅ লোড কম, মেইনটেন্যান্স উইন্ডো
✅ ক্লাইন্ট আইও নেই

#### **পদ্ধতি ২: Out করে রিস্টার্ট (নিরাপদ)**

```bash
# ১. OSD out করুন
ceph osd out 108

# ২. ৬০-১২০ সেকেন্ড অপেক্ষা করুন
sleep 90

# ৩. রিস্টার্ট করুন
systemctl restart ceph-osd@108

# ৪. OSD up হওয়া পর্যন্ত অপেক্ষা
sleep 30

# ৫. OSD in করুন
ceph osd in 108
```

**কি ঘটবে:**

```
সময় লাইন:
t=0s:   ceph osd out 108
t=5s:   ক্লাস্টার জানে "OSD 108 intentionally removed"
t=10s:  PG গুলো অন্য OSD থেকে ব্যাকফিল শুরু করে
t=60s:  ডাটা অন্যত্র কপি হয়েছে (আংশিক/সম্পূর্ণ)
t=90s:  systemctl restart ceph-osd@108
t=120s: OSD.108 up
t=125s: ceph osd in 108
t=130s: ধীরে ধীরে ডাটা ফিরে আসে (rebalance)
```

**সুবিধা:**

✅ **গ্র্যাচুয়াল ট্রানজিশন:** ক্লাইন্টরা মসৃণভাবে অন্য OSD এ শিফট হয়
✅ **কন্ট্রোলড রিকভারি:** ব্যাকফিল/রিকভারি নিয়ন্ত্রণে থাকে
✅ **কম ইমপ্যাক্ট:** ক্লাইন্ট আইওতে নগণ্য প্রভাব
✅ **নিরাপদ:** প্রোডাকশনের জন্য উপযুক্ত

**কখন করবেন:**

✅ প্রোডাকশন ক্লাস্টার
✅ একাধিক OSD
✅ Replication = 2
✅ অ্যাক্টিভ ক্লাইন্ট আইও

### নিরাপদ OSD রিস্টার্ট প্রক্রিয়া (Step-by-Step)

#### **প্রস্তুতিমূলক ধাপ**

**ধাপ ১: ব্যাকআপ নিন**

```bash
# ক্লাস্টার ম্যাপ ব্যাকআপ
ceph mon dump > /tmp/mon_dump_$(date +%Y%m%d_%H%M%S).txt
ceph osd tree > /tmp/osd_tree_$(date +%Y%m%d_%H%M%S).txt
ceph pg dump > /tmp/pg_dump_$(date +%Y%m%d_%H%M%S).txt

# কনফিগারেশন ব্যাকআপ
ceph config dump > /tmp/ceph_config_$(date +%Y%m%d_%H%M%S).txt
```

**ধাপ ২: বর্তমান স্ট্যাটাস ডকুমেন্ট করুন**

```bash
# ক্লাস্টার স্ট্যাটাস
ceph -s > /tmp/ceph_status_before.txt

# OSD ট্রি
ceph osd tree > /tmp/osd_tree_before.txt

# PG স্ট্যাটাস
ceph pg stat > /tmp/pg_stat_before.txt

# হেলথ ডিটেইলস
ceph health detail > /tmp/health_detail_before.txt
```

**ধাপ ৩: মেইনটেন্যান্স উইন্ডো নিশ্চিত করুন**

- ট্রাফিক কম থাকে এমন সময় নির্বাচন করুন
- ক্লাইন্টদের আগে থেকে জানান
- টিম মেম্বারদের অবগত করুন
- ব্যাকআপ ভেরিফাই করুন

#### **একটি OSD রিস্টার্ট (উদাহরণ: osd.108)**

**ধাপ ১: বর্তমান স্ট্যাটাস চেক**

```bash
# OSD ট্রি তে দেখুন
ceph osd tree | grep 108

# OSD স্ট্যাটাস
ceph osd dump | grep "osd.108 "

# PG যা এই OSD তে আছে
ceph pg ls-by-osd 108
```

**ধাপ ২: OSD out করুন**

```bash
ceph osd out 108
```

**আউটপুট:**
```
osd.108 marked out
```

**ধাপ ৩: ব্যাকফিল শুরু হওয়ার জন্য অপেক্ষা**

```bash
# ৯-১২০ সেকেন্ড অপেক্ষা করুন
sleep 90

# অথবা মনিটর করুন
ceph -w
```

**কি দেখবেন:**
```
pgmap v1234: 801 pgs: 792 active+clean, 9 backfilling
             recovering 123/456 kB (10 obj/s, 50 KiB/s)
```

**ধাপ ৪: ক্লাস্টার হেলথ চেক**

```bash
ceph -s
ceph health detail
```

**লক্ষ্য করুন:**
- HEALTH_OK বা HEALTH_WARN (HEALTH_ERR না হলে ভালো)
- backfilling/recovery চলছে কিনা
- degraded PG আছে কিনা

**ধাপ ৫: OSD রিস্টার্ট**

```bash
# systemctl দিয়ে (নির্ভরযোগ্য)
systemctl restart ceph-osd@108

# অথবা cephadm দিয়ে
ceph orch restart osd.108
```

**ধাপ ৬: OSD শুরু হওয়া পর্যন্ত অপেক্ষা**

```bash
# ৩০ সেকেন্ড অপেক্ষা
sleep 30

# OSD স্ট্যাটাস চেক
ceph osd dump | grep "osd.108 "

# অথবা
systemctl status ceph-osd@108 --no-pager
```

**আশা করা আউটপুট:**
```
osd.108 up (since ...)
```

**ধাপ ৭: OSD in করুন**

```bash
ceph osd in 108
```

**আউটপুট:**
```
osd.108 marked in
```

**ধাপ ৮: রিব্যালেন্সের জন্য অপেক্ষা**

```bash
# ২-৩ মিনিট অপেক্ষা
sleep 120

# অথবা মনিটর করুন
ceph -w
```

**ধাপ ৯: ফাইনাল ভেরিফিকেশন**

```bash
# OSD ট্রি
ceph osd tree | grep 108

# ক্লাস্টার স্ট্যাটাস
ceph -s

# PG স্ট্যাটাস
ceph pg stat
```

#### **একাধিক OSD রিস্টার্ট (একটির পর একটি)**

**সতর্কতা:** একসাথে একাধিক OSD রিস্টার্ট করবেন না!

```bash
# ❌ ভুল পদ্ধতি
systemctl restart ceph-osd@{108..117}

# ✅ সঠিক পদ্ধতি
# একটি করে, প্রতিটির মধ্যে ৩-৫ মিনিট বিরতি
```

**প্রক্রিয়া:**

```bash
# OSD 108
ceph osd out 108
sleep 90
systemctl restart ceph-osd@108
sleep 30
ceph osd in 108
sleep 120

# OSD 109
ceph osd out 109
sleep 90
systemctl restart ceph-osd@109
sleep 30
ceph osd in 109
sleep 120

# OSD 110
ceph osd out 110
sleep 90
systemctl restart ceph-osd@110
sleep 30
ceph osd in 110
sleep 120

# ... এভাবে চলবে
```

**প্রতিটি ধাপের পর চেক করুন:**

```bash
ceph -s
ceph health detail
```

---

## Dashboard বনাম CLI পদ্ধতি

### Ceph Dashboard থেকে OSD রিস্টার্ট

#### **পদ্ধতি**

**ধাপ ১: Dashboard এ লগইন**

```
https://172.16.10.111:8443
```

**ধাপ ২: Services এ যান**

```
Dashboard → Cluster → Services
```

**ধাপ ৩: OSD ট্যাব সিলেক্ট**

OSD সার্ভিসের লিস্ট দেখাবে।

**ধাপ ৪: OSD সিলেক্ট করুন**

যে OSD রিস্টার্ট করতে চান, সেটির পাশে Actions (⋮) বাটনে ক্লিক করুন।

**ধাপ ৫: Restart সিলেক্ট**

"Restart" অপশন সিলেক্ট করে Confirm করুন।

#### **Dashboard পদ্ধতির সুবিধা**

✅ **ভিজ্যুয়াল:** গ্রাফিক্যাল ইন্টারফেস, সহজ বোধগম্য
✅ **দ্রুত:** কয়েক ক্লিকে করা যায়
✅ **স্ট্যাটাস:** রিয়েল-টাইম স্ট্যাটাস দেখা যায়
✅ **নতুনদের জন্য:** সহজ

#### **Dashboard পদ্ধতির অসুবিধা**

❌ **গ্রানুলার কন্ট্রোল নেই:**
   - OSD out/in করার অপশন নেই
   - ব্যাকফিল মনিটর করার সুযোগ নেই
   - একাধিক OSD একসাথে রিস্টার্ট হয়ে যেতে পারে

❌ **ভিজ্যুয়াল ফিডব্যাক সীমিত:**
   - রিয়েল-টাইম লগ দেখা যায় না
   - রিস্টার্ট স্ট্যাটাস ডিলেতে আপডেট হয়
   - এরর মেসেজ বিস্তারিত দেখা যায় না

❌ **অটোমেটেড আউট/ইন নেই:**
   - সরাসরি systemctl restart হয়
   - ceph osd out হয় না
   - ব্যাকফিল শুরু হয় না
   - হঠাৎ degraded হয়

### CLI (কমান্ড লাইন) পদ্ধতি

#### **পদ্ধতি**

উপরে বিস্তারিত আলোচনা করা হয়েছে।

#### **CLI পদ্ধতির সুবিধা**

✅ **পূর্ণ নিয়ন্ত্রণ:**
   - প্রতিটি ধাপ ম্যানুয়ালি কন্ট্রোল করা যায়
   - OSD out/in করা যায়
   - ব্যাকফিল মনিটর করা যায়

✅ **রিয়েল-টাইম লগ:**
   - journalctl দিয়ে লগ দেখা যায়
   - ceph -w দিয়ে রিয়েল-টাইম স্ট্যাটাস
   - বিস্তারিত এরর মেসেজ

✅ **অটোমেশন:**
   - স্ক্রিপ্ট লেখা যায়
   - একাধিক OSD অটোমেটেড করা যায়
   - লুপে করা যায়

✅ **নিরাপদ:**
   - প্রতিটি ধাপ ভেরিফাই করা যায়
   - সমস্যা হলে থামা যায়
   - রোলব্যাক করা যায়

#### **CLI পদ্ধতির অসুবিধা**

❌ **টেক্সট-বেসড:** গ্রাফিক্যাল নয়
❌ **শেখার সময়:** নতুনদের জন্য একটু কঠিন
❌ **টাইপিং এরর:** ভুল কমান্ড টাইপ হতে পারে

### তুলনামূলক বিশ্লেষণ

| বৈশিষ্ট্য | Dashboard | CLI |
|-----------|-----------|-----|
| **OSD Out/In** | ❌ ম্যানুয়াল করতে হয় | ✅ স্ক্রিপ্টে অটোমেটেড |
| **গ্রানুলার কন্ট্রোল** | ❌ সীমিত | ✅ পূর্ণ নিয়ন্ত্রণ |
| **রিয়েল-টাইম লগ** | ❌ নেই | ✅ journalctl দেখা যায় |
| **ব্যাচ অপারেশন** | ⚠️ সম্ভব কিন্তু ঝুঁকিপূর্ণ | ✅ নিরাপদে লুপে করা যায় |
| **ভিজ্যুয়াল ফিডব্যাক** | ✅ গ্রাফিক্যাল | ❌ টেক্সট-বেসড |
| **ভুল করার সম্ভাবনা** | 🔶 মাঝারি | 🟢 কম (স্ক্রিপ্টে) |
| **প্রোডাকশন সেফটি** | 🔴 কম |  উচ্চ |
| **শেখার বক্ররেখা** | ✅ সহজ | ⚠️ মাঝারি |
| **ট্রাবলশুটিং** | ❌ সীমিত | ✅ বিস্তারিত |

### কখন কোন পদ্ধতি ব্যবহার করবেন?

#### **Dashboard ব্যবহার করুন যদি:**

✅ শুধুমাত্র ১টি OSD রিস্টার্ট
✅ টেস্ট/ডেভ এনভায়রনমেন্ট
✅ ক্লাস্টার লোড কম
✅ মেইনটেন্যান্স উইন্ডোতে
✅ দ্রুত একটি OSD ফিক্স
✅ নতুন টিম মেম্বার

#### **CLI ব্যবহার করুন যদি:**

✅ প্রোডাকশন ক্লাস্টার ⭐
✅ একাধিক OSD রিস্টার্ট
✅ ক্লাস্টারে আগে থেকেই ওয়ার্নিং
✅ অ্যাক্টিভ ক্লাইন্ট আইও
✅ Replication Factor = 2
✅ অভিজ্ঞ অ্যাডমিন

### হাইব্রিড পদ্ধতি (Best of Both)

Dashboard এবং CLI একসাথে ব্যবহার করা যেতে পারে:

```bash
# ১. Dashboard থেকে OSD সিলেক্ট করুন
# ২. CLI তে গিয়ে OSD out করুন
ceph osd out 108

# ৩. ২ মিনিট অপেক্ষা করুন
sleep 120

# ৪. Dashboard থেকে Restart বাটনে ক্লিক করুন
# ৫. CLI তে চেক করুন
ceph osd tree | grep 108

# ৬. OSD up হলে, CLI থেকে in করুন
ceph osd in 108
```

---

## নিরাপদ OSD রিস্টার্ট প্রক্রিয়া

### প্রোডাকশন চেকলিস্ট

OSD রিস্টার্ট করার আগে এই চেকলিস্ট ফলো করুন:

#### **রিস্টার্টের আগে**

```bash
# □ ১. ক্লাস্টার হেলথ চেক
ceph -s
ceph health detail
# HEALTH_OK বা HEALTH_WARN হতে হবে (HEALTH_ERR না)

# □ ২. OSD স্ট্যাটাস
ceph osd stat
ceph osd tree

# □ ৩. PG স্ট্যাটাস
ceph pg stat
# degraded PG কম হতে হবে

# □ ৪. ক্লাইন্ট আইও চেক
ceph -w
# খুব বেশি read/write না থাকলে ভালো

# □ ৫. ব্যাকআপ ভেরিফাই
ls -lh /tmp/ceph_*_*.txt
# ব্যাকআপ ফাইল আছে কিনা

# □ ৬. মেইনটেন্যান্স উইন্ডো
# ট্রাফিক কম, ক্লাইন্টদের জানানো হয়েছে

# □ ৭. টিম নোটিফিকেশন
# অন্যান্য টিম মেম্বারদের জানানো হয়েছে

# □ ৮. রোলব্যাক প্ল্যান
# সমস্যা হলে কী করবেন, প্ল্যান রেডি
```

#### **রিস্টার্টের সময়**

```bash
# □ ৯. একবারে একটি OSD
# একাধিক OSD একসাথে নয়

# □ ১০. প্রতিটি OSD এর মধ্যে ৩-৫ মিনিট বিরতি
# ব্যাকফিল/রিকভারি শেষ হওয়ার জন্য

# □ ১১. ceph -w দিয়ে মনিটর
# রিয়েল-টাইম স্ট্যাটাস দেখুন

# □ ১২. PG স্ট্যাটাস চেক
ceph pg stat
# degraded PG বাড়ছে কিনা

# □ ১৩. ক্লাস্টার হেলথ
ceph health
# HEALTH_ERR এ যাচ্ছে কিনা

# □ ১৪. ক্লাইন্ট আইও মনিটর
# ক্লাইন্টরা সমস্যা পাচ্ছে কিনা
```

#### **রিস্টার্টের পর**

```bash
# □ ১৫. ক্লাস্টার হেলথ
ceph -s
ceph health
# HEALTH_OK হতে হবে

# □ ১৬. OSD স্ট্যাটাস
ceph osd tree
# সব OSD up/in আছে কিনা

# □ ১৭. PG স্ট্যাটাস
ceph pg stat
# সব PG active+clean

# □ ১৮. ক্লাইন্ট আইও টেস্ট
# ক্লাইন্টরা нормально কাজ করছে কিনা

# □ ১৯. লগ চেক
tail -50 /var/log/ceph/ceph.log
# কোনো এরর নেই

# □ ২০. ডকুমেন্টেশন
# কী করা হয়েছে, লগ রাখুন
```

### ট্রাবলশুটিং: সমস্যা হলে কী করবেন

#### **সমস্যা ১: OSD শুরু হচ্ছে না**

```bash
# সার্ভিস স্ট্যাটাস চেক
systemctl status ceph-osd@108

# লগ চেক
journalctl -u ceph-osd@108 --since "10 minutes ago"

# ডিস্ক চেক
lsblk
ceph-volume lvm list

# পারমিশন চেক
ls -la /var/lib/ceph/osd/ceph-108/

# পারমিশন ফিক্স
chown -R ceph:ceph /var/lib/ceph/osd/ceph-108/
```

#### **সমস্যা ২: PG degraded রয়ে গেছে**

```bash
# কোন PG degraded
ceph pg dump | grep degraded

# কোন OSD তে সমস্যা
ceph pg ls-by-osd 108

# OSD out করে দিন
ceph osd out 108

# ব্যাকফিল শেষ হওয়ার জন্য অপেক্ষা
ceph -w

# প্রয়োজনে OSD purge
ceph osd purge 108 --yes-i-really-mean-it
```

#### **সমস্যা ৩: ক্লাস্টার HEALTH_ERR**

```bash
# বিস্তারিত দেখুন
ceph health detail

# সমস্যার কারণ খুঁজে বের করুন
ceph -s

# জরুরি: OSD out করুন
ceph osd out 108

# ক্লাস্টার স্থিতিশীল হওয়া পর্যন্ত অপেক্ষা
ceph -w

# প্রয়োজনে স্পেশালিস্টের সাহায্য নিন
```

#### **সমস্যা ৪: ক্লাইন্ট আইও ফেইল**

```bash
# ক্লাইন্ট সাইড থেকে চেক
rbd map <image>
mount /dev/rbd0 /mnt

# এরর দেখলে
dmesg | tail

# Ceph সাইড থেকে
ceph osd stat
ceph mon stat

# প্রয়োজনে ক্লাস্টার রিস্টার্ট
# (শুধুমাত্র জরুরি অবস্থায়)
```

### রোলব্যাক প্ল্যান

যদি রিস্টার্টের পর সমস্যা হয়:

```bash
# ১. OSD out করুন
ceph osd out 108

# ২. OSD down করুন
ceph osd down 108

# ৩. ক্লাস্টার স্থিতিশীল হওয়া পর্যন্ত অপেক্ষা
ceph -w

# ৪. লগ সংগ্রহ করুন
journalctl -u ceph-osd@108 > /tmp/osd108_error.log
ceph report > /tmp/ceph_report.log

# ৫. আগের স্ট্যাটাস রিস্টোর (প্রয়োজন হলে)
# ব্যাকআপ থেকে কনফিগারেশন রিস্টোর

# ৬. সাহায্য নিন
# Ceph কমিউনিটি বা এক্সপার্টের সাহায্য
```

---

## ট্রাবলশুটিং এবং ডায়াগনোসিস

### ডিস্ক/হার্ডওয়্যার সমস্যা

#### **লক্ষণ:**

- OSD বারবার ক্র্যাশ করে
- I/O এরর
- স্লো পারফরম্যান্স
- SMART এরর

#### **ডায়াগনোসিস:**

```bash
# SMART হেলথ চেক
smartctl -H /dev/sda
smartctl -H /dev/sdb

# বিস্তারিত SMART ইনফো
smartctl -a /dev/sda

# ডিস্ক এরর
dmesg | grep -i "sd[a-z].*error"
dmesg | grep -i "I/O error"

# ব্যাড সেক্টর
badblocks -v /dev/sda

# ডিস্ক পারফরম্যান্স
hdparm -tT /dev/sda
```

#### **সমাধান:**

```bash
# ১. OSD out করুন
ceph osd out 108

# ২. OSD remove করুন
ceph osd purge 108 --yes-i-really-mean-it

# ৩. cephadm থেকে remove
ceph orch osd rm 108

# ৪. নতুন ডিস্ক যোগ করুন
ceph orch daemon add osd flash1:/dev/new_disk

# অথবা replace
ceph orch osd replace 108 /dev/new_disk
```

### নেটওয়ার্ক সমস্যা

#### **লক্ষণ:**

- হোস্ট আনরিচেবল
- OSD হার্টবিট ফেইলিউর
- ক্লাস্টার পার্টিশন

#### **ডায়াগনোসিস:**

```bash
# নেটওয়ার্ক কানেক্টিভিটি
ping 172.16.10.111

# পোর্ট চেক
nc -zv 172.16.10.111 6800
nc -zv 172.16.10.111 6801

# ফায়ারওয়াল
systemctl status firewalld
iptables -L -n | grep ceph

# নেটওয়ার্ক ইন্টারফেস
ip addr show
ip link show

# রাউটিং
ip route show
```

#### **সমাধান:**

```bash
# ফায়ারওয়াল রুলস
firewall-cmd --permanent --add-port=6800-7300/tcp
firewall-cmd --reload

# নেটওয়ার্ক রিস্টার্ট
systemctl restart NetworkManager

# হোস্ট ফাইল চেক
cat /etc/hosts
# সব নোডের এন্ট্রি আছে কিনা
```

### মেমোরি/রিসোর্স সমস্যা

#### **লক্ষণ:**

- OOM Killer
- OSD ক্র্যাশ
- স্লো পারফরম্যান্স

#### **ডায়াগনোসিস:**

```bash
# মেমোরি ইউজেজ
free -h

# OOM Killer লগ
dmesg | grep -i "out of memory"
dmesg | grep -i "killed process"

# প্রসেস লিস্ট
top -bn1 | head -20
ps aux | grep ceph

# ডিস্ক স্পেস
df -h /var/lib/ceph
```

#### **সমাধান:**

```bash
# মেমোরি লিমিট বাড়ান
# /etc/ceph/ceph.conf তে
[osd]
osd_memory_target = 4294967296  # 4GB

# সার্ভিস রিস্টার্ট
systemctl restart ceph-osd@108

# সোয়াপ যোগ করুন (যদি না থাকে)
swapon -s
```

### কনফিগারেশন সমস্যা

#### **লক্ষণ:**

- OSD শুরু হয় না
- কনফিগ এরর
- ভ্যালিডেশন ফেইল

#### **ডায়াগনোসিস:**

```bash
# কনফিগারেশন চেক
ceph-conf --show-config

# নির্দিষ্ট OSD কনফিগ
ceph daemon osd.108 config show

# লগ চেক
tail -50 /var/log/ceph/ceph-osd@108.log
```

#### **সমাধান:**

```bash
# কনফিগারেশন রিসেট
ceph config rm osd.108 <key>

# অথবা
ceph config reset osd.108

# কনফিগারেশন ভেরিফাই
ceph daemon osd.108 config diff
```

---

## প্রতিরোধমূলক ব্যবস্থা

### নিয়মিত মনিটরিং

#### **অটোমেটেড হেলথ চেক**

```bash
# Cron job সেটআপ
crontab -e

# প্রতি ৫ মিনিটে হেলথ চেক
*/5 * * * * /usr/bin/ceph -s >> /var/log/ceph-health.log 2>&1

# প্রতি ঘণ্টায় ডিটেইলস রিপোর্ট
0 * * * * /usr/bin/ceph health detail >> /var/log/ceph-detail.log 2>&1

# প্রতিদিন OSD স্ট্যাটাস
0 0 * * * /usr/bin/ceph osd tree >> /var/log/ceph-osd-tree.log 2>&1
```

#### **অ্যালার্ট কনফিগারেশন**

```bash
# Prometheus এনাবল
ceph mgr module enable prometheus

# Alertmanager কনফিগার
ceph orch apply alertmanager

# Grafana ড্যাশবোর্ড
ceph orch apply grafana
```

### অটো-হিলিং কনফিগারেশন

```bash
# অটো-প্রুনিং
ceph config set mgr mgr/crash/prune_interval 3600
ceph config set mgr mgr/crash/warn_delay 86400

# অটো-রিকভারি
ceph osd set norecover false
ceph osd set nobackfill false
ceph osd set norebalance false

# স্লো রিকোয়েস্ট থ্রেশহোল্ড
ceph config set mon mon_osd_slow_op_warning_threshold 30
```

### ব্যাকআপ স্ট্র্যাটেজি

#### **ক্লাস্টার ম্যাপ ব্যাকআপ**

```bash
#!/bin/bash
# /usr/local/bin/ceph-backup.sh

BACKUP_DIR="/backup/ceph"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

ceph mon dump > $BACKUP_DIR/mon_dump_$DATE.txt
ceph osd tree > $BACKUP_DIR/osd_tree_$DATE.txt
ceph pg dump > $BACKUP_DIR/pg_dump_$DATE.txt
ceph config dump > $BACKUP_DIR/config_$DATE.txt
ceph -s > $BACKUP_DIR/status_$DATE.txt

# ৩ দিনের পুরনো ব্যাকআপ ডিলিট
find $BACKUP_DIR -mtime +30 -delete
```

#### **কনফিগারেশন ব্যাকআপ**

```bash
# Ceph কনফিগারেশন ফাইল
cp /etc/ceph/ceph.conf /backup/ceph/ceph.conf.$(date +%Y%m%d)

# কী রিং ব্যাকআপ
cp /etc/ceph/*.keyring /backup/ceph/
```

### পারফরম্যান্স টিউনিং

```bash
# OSD পারফরম্যান্স
ceph config set osd osd_memory_target 4294967296
ceph config set osd osd_op_threads 8
ceph config set osd osd_disk_threads 4

# নেটওয়ার্ক
ceph config set global ms_type = async+posix

# মনিটর
ceph config set mon mon_compact_on_start true
```

### ডকুমেন্টেশন এবং রানবুক

#### **মেইনটেন্যান্স রানবুক তৈরি করুন**

```
1. প্রতি সপ্তাহে:
   - ক্লাস্টার হেলথ রিভিউ
   - ক্যাপাসিটি প্ল্যানিং
   - লগ রিভিউ

2. প্রতি মাসে:
   - ব্যাকআপ ভেরিফিকেশন
   - ড্রিল টেস্ট (OSD রিস্টার্ট)
   - পারফরম্যান্স বেসলাইন

3. প্রতি কোয়ার্টারে:
   - ডিজাস্টার রিকভারি টেস্ট
   - কনফিগারেশন অডিট
   - সিকিউরিটি রিভিউ
```

### ক্যাপাসিটি প্ল্যানিং

```bash
# ক্যাপাসিটি ইউজেজ
ceph df

# ভবিষ্যৎ প্রজেকশন
# বর্তমান গ্রোথ রেট * দিন

# থ্রেশহোল্ড সেট করুন
# Warning: 70%
# Critical: 85%
# Emergency: 95%

# অটোমেটেড অ্যালার্ট
ceph config set mon mon_osd_full_ratio 0.95
ceph config set mon mon_osd_nearfull_ratio 0.85
```

---

## জরুরি সহায়তা এবং রেফারেন্স

### জরুরি কমান্ড

#### **ক্লাস্টার স্টপ (শুধুমাত্র জরুরি)**

```bash
# সব OSD স্টপ
ceph osd set noout
ceph osd set norecover
ceph osd set nobackfill
ceph osd set norebalance

# এক একটি করে OSD স্টপ
for osd in $(ceph osd ls); do
    ceph osd out $osd
done

# সার্ভিস স্টপ
systemctl stop ceph-osd@*
systemctl stop ceph-mon@*
systemctl stop ceph-mgr@*
```

#### **ক্লাস্টার স্টার্ট**

```bash
# সার্ভিস স্টার্ট
systemctl start ceph-mon@*
systemctl start ceph-mgr@*
systemctl start ceph-osd@*

# নরমাল মোডে ফিরে আসুন
ceph osd unset noout
ceph osd unset norecover
ceph osd unset nobackfill
ceph osd unset norebalance
```

### লগ সংগ্রহ (সাপোর্টের জন্য)

```bash
# লগ ডিরেক্টরি
mkdir -p /tmp/ceph-logs-$(date +%Y%m%d)

# সব লগ কপি
cp /var/log/ceph/*.log /tmp/ceph-logs-$(date +%Y%m%d)/

# journalctl লগ
journalctl -u ceph-mon > /tmp/ceph-logs-$(date +%Y%m%d)/mon.log
journalctl -u ceph-mgr > /tmp/ceph-logs-$(date +%Y%m%d)/mgr.log
journalctl -u ceph-osd > /tmp/ceph-logs-$(date +%Y%m%d)/osd.log

# ক্লাস্টার রিপোর্ট
ceph report > /tmp/ceph-logs-$(date +%Y%m%d)/report.txt
ceph -s > /tmp/ceph-logs-$(date +%Y%m%d)/status.txt

# কম্প্রেস
tar -czf /tmp/ceph-logs-$(date +%Y%m%d).tar.gz /tmp/ceph-logs-$(date +%Y%m%d)/
```

### রেফারেন্স লিংক

#### **অফিসিয়াল ডকুমেন্টেশন**

- Ceph ডকুমেন্টেশন: https://docs.ceph.com/
- Ceph কমিউনিটি: https://ceph.io/community/
- Ceph GitHub: https://github.com/ceph/ceph

#### **ট্রাবলশুটিং গাইড**

- OSD ট্রাবলশুটিং: https://docs.ceph.com/en/latest/osd/troubleshooting/
- মনিটর ট্রাবলশুটিং: https://docs.ceph.com/en/latest/mon/troubleshooting/
- নেটওয়ার্ক ট্রাবলশুটিং: https://docs.ceph.com/en/latest/rados/troubleshooting/troubleshooting-network/

#### **কমিউনিটি সাপোর্ট**

- Ceph ডিসকোর্স: https://discuss.ceph.com/
- Ceph IRC: #ceph on Libera.Chat
- Stack Overflow: https://stackoverflow.com/questions/tagged/ceph

### যোগাযোগ

#### **ইন্টারনাল টিম**

```
Storage Team Lead: [নাম] - [ইমেইল] - [ফোন]
DevOps Team: [ইমেইল] - [স্ল্যাক চ্যানেল]
Infrastructure: [ইমেইল] - [টিকিট সিস্টেম]
```

#### **এস্কেলেশন ম্যাট্রিক্স**

```
Level 1: Storage Admin (0-30 মিনিট)
Level 2: Senior DevOps (30-60 মিনিট)
Level 3: Infrastructure Lead (1-2 ঘণ্টা)
Level 4: External Consultant/Support (2+ ঘণ্টা)
```

---

## উপসংহার

এই গাইডটি একটি বাস্তব প্রোডাকশন Ceph ক্লাস্টারের সমস্যা সমাধানের সম্পূর্ণ প্রক্রিয়া ডকুমেন্ট করেছে। মূল শিক্ষাগুলো হলো:

### **মূল শিক্ষা**

১. **সর্বদা প্রস্তুত থাকুন:**
   - ব্যাকআপ নিন
   - ডকুমেন্টেশন রাখুন
   - রানবুক তৈরি করুন

২. **নিরাপদে কাজ করুন:**
   - প্রোডাকশনে CLI পদ্ধতি ব্যবহার করুন
   - OSD out করে তারপর রিস্টার্ট করুন
   - একাধিক OSD একসাথে নয়

৩. **মনিটর করুন:**
   - রিয়েল-টাইম মনিটরিং
   - অটোমেটেড অ্যালার্ট
   - নিয়মিত হেলথ চেক

৪. **ডকুমেন্ট করুন:**
   - প্রতিটি পরিবর্তন লগ রাখুন
   - সমস্যা এবং সমাধান ডকুমেন্ট করুন
   - টিমের সাথে শেয়ার করুন

### **ভবিষ্যৎ সুপারিশ**

- অটোমেটেড ব্যাকআপ সিস্টেম ইমপ্লিমেন্ট করুন
- মনিটরিং এবং অ্যালার্টিং শক্তিশালী করুন
- নিয়মিত ড্রিল টেস্ট করুন
- টিম ট্রেনিং দিন

---

**ডকুমেন্ট ভার্সন:** 1.0  
**শেষ আপডেট:** 2026  
**প্রস্তুতকারক:** DevOps/Storage Team  
**পর্যালোচনা:** প্রতি ৬ মাসে

---