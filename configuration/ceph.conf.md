# ⚙️ ceph.conf Explained

The `ceph.conf` file is the primary configuration file for Ceph. It defines how the Ceph daemons behave, how nodes communicate, and how storage policies are applied.

---

## 📁 File Location

Depending on your installation method, you will typically find the `ceph.conf` file here:

- **Default location**: `/etc/ceph/ceph.conf`
- **Containerized setups (e.g. cephadm)**: `/var/lib/ceph/<cluster-fsid>/mon.<hostname>/config`

### 🔍 View the file

```bash
cat /etc/ceph/ceph.conf
````

### 📝 Edit the file

```bash
sudo nano /etc/ceph/ceph.conf
```

> ⚠️ Always back up the file before making changes!

---

## 🧱 ceph.conf Structure

The `ceph.conf` file is INI-style and structured with section headers.

### 🔹 Global Section

Global options for all daemons.

```ini
[global]
fsid = 12345678-1234-1234-1234-123456789abc
mon_initial_members = mon1,mon2,mon3
mon_host = 192.168.1.1,192.168.1.2,192.168.1.3
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
```

### 🔹 MON Section (Monitor)

Monitor-specific settings.

```ini
[mon.mon1]
host = mon1
mon_addr = 192.168.1.1
```

### 🔹 OSD Section

OSD-specific parameters.

```ini
[osd]
osd_journal_size = 1024
osd_mkfs_type = xfs
osd_mkfs_options_xfs = -f
```

### 🔹 MDS Section (for CephFS)

```ini
[mds.mds1]
host = mds1
mds_data = /var/lib/ceph/mds/mds1
```

---

## 💡 Best Practices

* Use **ceph-authtool** and **ceph config** CLI when possible instead of editing manually.
* Avoid putting secrets (keys, passwords) in `ceph.conf`.
* Use consistent indentation and lowercase section headers.
* For dynamic configuration changes, prefer the CLI:

```bash
ceph config set osd osd_scrub_sleep 0.1
```

* Validate configuration changes using:

```bash
ceph config dump
```

---

## 🧪 Sample ceph.conf File (Minimal)

```ini
[global]
fsid = 1111aaaa-2222-bbbb-3333-cccc4444dddd
mon_initial_members = mon1
mon_host = 192.168.0.10
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx

[mon.mon1]
host = mon1
mon_addr = 192.168.0.10

[osd]
osd_journal_size = 1024
osd_mkfs_type = xfs
```

---

## 🧠 Configuration Diagram (Text-based)

```
[global]
 ├── Applies to all nodes
 ├── MON addresses
 ├── Authentication settings

[mon.<name>]
 ├── Individual monitor node settings

[osd]
 ├── Settings for all OSDs

[mds.<name>]
 ├── CephFS metadata server settings
```

---

## 🧰 Troubleshooting Tips

| Issue                     | Fix                                                   |
| ------------------------- | ----------------------------------------------------- |
| Daemons not starting      | Check `fsid`, `mon_host`, and IPs                     |
| OSD crash                 | Verify `osd_mkfs_type` and device mount               |
| Configuration not applied | Run `ceph config dump` or restart daemon              |
| Duplicate MONs            | Ensure `mon_initial_members` and hostnames are unique |

---

## 🔄 Reload or Restart after Change

After modifying `ceph.conf`, restart affected services:

```bash
sudo systemctl restart ceph-mon@<hostname>
sudo systemctl restart ceph-osd@<id>
```

Or if you're using `cephadm`:

```bash
ceph orch restart mon
ceph orch restart osd
```

---

## 📚 References

* [Ceph Official Configuration Guide](https://docs.ceph.com/en/latest/rados/configuration/ceph-conf/)
* [CLI Config Tool](https://docs.ceph.com/en/latest/rados/operations/configuration/)

---

> 📝 Maintained by: [Suman Pal](https://github.com/SumonPaul18)

---

