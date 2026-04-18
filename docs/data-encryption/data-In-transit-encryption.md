# 🛡️ Mastering Ceph In-Transit Encryption: The MSGR2 Handbook

In a modern data center, data moving over the wire is vulnerable. Imagine your 3-node Ceph cluster as a digital bank. Without encryption, your data travels in glass trucks; anyone on the network can see the contents. By enabling Messenger v2 (msgr2) encryption, we turn those glass trucks into invisible, armored tanks using AES-GCM.

This guide provides a practical, step-by-step roadmap for enabling, verifying, managing, and disabling in-transit encryption in a `cephadm` environment.

## 📑 Table of Contents

1. [System Requirements & Pre-Flight Checks](#1-system-requirements--pre-flight-checks)
2. [Demonstrating the Need for MSGR2](#2-demonstrating-the-need-for-msgr2)
3. [Enabling In-Transit Encryption (Step-by-Step)](#3-enabling-in-transit-encryption-step-by-step)
4. [Verification & Post-Implementation Operations](#4-verification--post-implementation-operations)
5. [Managing & Maintaining Encryption States](#5-managing--maintaining-encryption-states)
6. [Reverting Changes: Disabling Encryption Safely](#6-reverting-changes-disabling-encryption-safely)

---

## 1. System Requirements & Pre-Flight Checks

Before making architectural changes, you must ensure your "infrastructure foundation" supports modern encryption.

### Technical Prerequisites
*   **Cluster Engine:** Ceph version Quincy (17.2.x), Reef (18.2.x), or Squid (19.2.x).
*   **Deployment:** `cephadm` based 3-node setup.
*   **Kernel Support:** Client nodes (for RBD/CephFS) must run Linux Kernel 5.11+.
*   **Network:** Port `3300` must be open across all nodes.

### Action: Version Verification

Check your current Ceph version to ensure compatibility with msgr2 secure modes.

```bash
ceph version
```

Check the Linux Kernel version on all client nodes to ensure they support encrypted mounts.

```bash
uname -r
```

> **Note:** If your kernel is older than 5.11, you may face mounting issues with encrypted RBD images. Refer to [Ceph Release Notes](https://docs.ceph.com/en/latest/releases/) for specific breaking changes.

---

## 2. Demonstrating the Need for MSGR2

Ceph historically used `msgr1` (Port 6789). It was built for speed, not security. `msgr2` (Port 3300) is the encrypted successor.

*   **msgr1 (v1):** Plain text or CRC checksums. Vulnerable to packet sniffing.
*   **msgr2 (v2):** Supports `secure` mode using AES-GCM encryption.

### Practical Check: Is v2 already there?

By default, `cephadm` enables v2, but it usually runs in "CRC" mode (checksum only, no encryption). Verify if daemons are listening on the secure port.

```bash
ss -tlnp | grep 3300
```

> **Expected Output:** You should see `ceph-mon` or `ceph-osd` listening on port `3300`. If this port is closed, msgr2 is not active.

---

## 3. Enabling In-Transit Encryption (Step-by-Step)

Turning on encryption is like enforcing a "Secret Handshake" for all daemons. If they don't speak encrypted, they can't join the conversation.

### The "Secure" Workflow

Run these commands on your bootstrap/main admin node. This enforces encryption for all traffic types.

**Step 1:** Enforce encryption for internal OSD-to-OSD replication traffic.

```bash
ceph config set global ms_cluster_mode secure
```

**Step 2:** Enforce encryption for Public/Service traffic (Monitor to Clients).

```bash
ceph config set global ms_service_mode secure
```

**Step 3:** Enforce encryption for all incoming client requests (RBD/CephFS clients).

```bash
ceph config set global ms_client_mode secure
```

### Optional: Hardening the Cluster

If you want to completely block the unencrypted legacy port (6789) to force strict security:

```bash
ceph config set global ms_bind_msgr1 false
```

> **Warning:** Ensure all your clients and internal services are successfully connecting via port 3300 before disabling port 6789, otherwise, you may lose connectivity.

---

## 4. Verification & Post-Implementation Operations

Configuration is only half the battle. You must verify that the daemons have actually switched to encrypted tunnels.

### The Orchestration Restart

`cephadm` managed daemons need a restart to pick up the new `secure` flags from the configuration database.

Restart Monitor daemons to apply the secure handshake settings.

```bash
ceph orch restart mon
```

Restart all OSD daemons across all hosts to enforce encrypted replication.

```bash
ceph orch restart osd.all-hosts
```

### Verification Commands

Verify that the configuration has been applied globally.

```bash
ceph config dump | grep ms_
```

Check the Monitor Map to ensure `v2` addresses are prioritized and active.

```bash
ceph mon dump
```

Check a specific daemon (e.g., `osd.0`) to confirm its local runtime mode is set to secure.

```bash
ceph config show osd.0 | grep ms_mode
```

> **Success Indicator:** The output should reflect `secure` for `ms_cluster_mode`, `ms_service_mode`, and `ms_client_mode`.

---

## 5. Managing & Maintaining Encryption States

Ongoing maintenance ensures that new OSDs or Nodes added to the cluster follow the security policy automatically.

### Operations Guide

*   **Adding New Nodes:** When adding a 4th node, `cephadm` will automatically apply the `global` secure settings to the new daemons. No extra steps are needed.
*   **Audit Logs:** Monitor `ceph -w` for "connection refused" errors. This often indicates a legacy client trying to connect without encryption support.

### Troubleshooting: The Mon Map Cleanup

If `ceph mon dump` still shows `v1:6789` and you want to force a clean state using only `v2`, update the monitor addresses manually.

Update Monitor 1 address to use only v2 (replace IP with your actual Mon IP).

```bash
ceph mon set-addrs ceph1 [v2:192.168.68.248:3300/0]
```

Update Monitor 2 address to use only v2.

```bash
ceph mon set-addrs ceph2 [v2:192.168.68.249:3300/0]
```

Update Monitor 3 address to use only v2.

```bash
ceph mon set-addrs ceph3 [v2:192.168.68.250:3300/0]
```

> **Note:** Replace `ceph1`, `ceph2`, etc., with your actual monitor hostnames or IDs, and update the IP addresses to match your environment.

---

## 6. Reverting Changes: Disabling Encryption Safely

If you encounter performance degradation or need to support legacy hardware that does not support msgr2, you can revert to CRC mode (Checksum only, no encryption).

### Practical Downgrade Workflow

**Step 1:** Revert all communication modes to `crc` (non-encrypted checksum).

```bash
ceph config set global ms_cluster_mode crc
ceph config set global ms_service_mode crc
ceph config set global ms_client_mode crc
```

**Step 2:** Re-enable the legacy port (if it was previously disabled).

```bash
ceph config set global ms_bind_msgr1 true
```

**Step 3:** Apply changes by restarting the services via the Orchestrator.

```bash
ceph orch restart mon
ceph orch restart osd.all-hosts
```

### Operational Warning

Disabling encryption in a production environment exposes data to packet sniffing. This should only be done in air-gapped networks or for extreme performance troubleshooting where latency is critical.

---

## 📝 Final Checklist for Administrators

*   [ ] **Kernel:** Are all clients on Linux Kernel 5.11+?
*   [ ] **Config:** Is `ms_cluster_mode` set to `secure`?
*   [ ] **Ports:** Is port `3300` open on the firewall and listening?
*   [ ] **Restart:** Have all `mon` and `osd` services been restarted via `ceph orch`?

**Guide Status:** ✅ Complete  
**Environment:** 3-Node Cephadm  
**Security Level:** AES-GCM Encrypted (msgr2)