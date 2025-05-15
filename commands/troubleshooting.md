# 🛠️ Troubleshooting Ceph

Running a Ceph cluster in production can occasionally lead to issues like OSD down, degraded PGs, or MON quorum loss. Here's how to identify and resolve them.

---

## 🧭 Table of Contents

- [🚨 Common Issues](#-common-issues)
- [📋 Logs & Health](#-logs--health)
- [🧪 OSD Recovery](#-osd-recovery)
- [🔁 PG Stuck / Degraded](#-pg-stuck--degraded)
- [💡 MON Quorum Issues](#-mon-quorum-issues)
- [🛡 Slow Ops & Performance](#-slow-ops--performance)
- [📚 Resources](#-resources)

---

## 🚨 Common Issues

| Problem                        | Symptom                           | Solution                                      |
|-------------------------------|------------------------------------|-----------------------------------------------|
| OSD Down                      | `osd.X down`                      | Restart OSD, check logs                       |
| MON Quorum Lost              | `no quorum` warning                | Ensure 3 MONs, check network/firewall         |
| PGs Stuck or Degraded        | `pgs degraded, stuck`             | Check `ceph pg dump`, ensure all OSDs are in  |
| Slow Ops                     | `slow ops` warning                | Review client ops, check OSD/mgr stats        |

---

## 📋 Logs & Health

```bash
ceph status
ceph health detail
ceph log last 50
journalctl -u ceph-osd@0
journalctl -u ceph-mon@ceph-mon1
````

### 🔍 Check specific daemon logs:

```bash
sudo journalctl -u ceph-<daemon>@<id> -f
```

---

## 🧪 OSD Recovery

```bash
ceph osd tree
ceph osd out 3
systemctl restart ceph-osd@3
ceph osd in 3
```

### 🧠 If an OSD is consistently failing:

* Check disk health: `smartctl -a /dev/sdX`
* Replace disk, re-add OSD

---

## 🔁 PG Stuck / Degraded

```bash
ceph pg dump | grep "stuck"
ceph pg repair <pg_id>
```

### 🔁 To force repair:

```bash
ceph pg repair 1.23
```

### ✅ Ensure:

* All OSDs are `up` and `in`
* Network connectivity is fine
* Time is synced (use `chrony` or `ntpd`)

---

## 💡 MON Quorum Issues

### Symptoms:

```bash
ceph status
# shows "no quorum"
```

### Solutions:

```bash
# Check MON map
ceph mon dump

# Restart MON
systemctl restart ceph-mon@ceph-mon1

# Check quorum status
ceph quorum_status --format json-pretty
```

> 🧠 Ensure all MONs have correct entries in `/etc/hosts` and ceph.conf

---

## 🛡 Slow Ops & Performance

### Investigate:

```bash
ceph health detail
ceph osd perf
ceph tell osd.* bench
```

### Actions:

* Identify overloaded OSDs
* Use `iostat`, `iotop`, or `dstat` for system analysis
* Check for slow disks

---

## 📚 Resources

* [Official Ceph Troubleshooting Guide](https://docs.ceph.com/en/latest/rados/troubleshooting/)
* [cephadm logs](https://docs.ceph.com/en/latest/cephadm/services/)

---
