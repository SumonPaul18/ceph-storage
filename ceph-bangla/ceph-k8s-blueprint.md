# ✅ Kubernetes + Ceph Full Production Blueprint

(Enterprise Design + Real Use Case + Step-by-Step)

---

# 📌 1️⃣ Production Goal

আমাদের লক্ষ্য:

- Kubernetes cluster (HA control plane)
    
- Ceph cluster (RBD + CephFS + RGW)
    
- Dynamic provisioning
    
- High availability
    
- Monitoring + Backup
    
- Zero single point of failure
    

---

# 📌 2️⃣ High-Level Architecture Diagram

```
                ┌──────────────────────────────┐
                │         End Users            │
                └──────────────┬───────────────┘
                               │
                        Load Balancer
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
  ┌────────────┐        ┌────────────┐        ┌────────────┐
  │  Master1   │        │  Master2   │        │  Master3   │
  │ (Control)  │        │ (Control)  │        │ (Control)  │
  └────────────┘        └────────────┘        └────────────┘

        ┌──────────────────────────────────────────────┐
        │              Worker Nodes                   │
        │  App Pods + CSI Driver + RBD/CephFS Mount  │
        └──────────────────────────────────────────────┘

                ┌────────────────────────────────┐
                │          Ceph Cluster          │
                │ MON + MGR + OSD + MDS + RGW   │
                └────────────────────────────────┘
```

---

# 📌 3️⃣ Production Environment Example

## 🏢 Scenario: SaaS Company

Workload:

- 50 microservices
    
- PostgreSQL database
    
- File upload service
    
- Daily backup
    
- Log archival
    

Storage mapping:

|Service|Ceph Type|
|---|---|
|PostgreSQL|RBD|
|Shared media|CephFS|
|Backup|RGW|

---

# 📌 4️⃣ Step 1: Ceph Cluster Ready থাকা

Prerequisite:

- 3 MON
    
- 3+ OSD node
    
- RBD pool created
    
- CephFS created
    
- RGW running
    

(আমরা আগের অধ্যায়ে এগুলো করেছি)

---

# 📌 5️⃣ Step 2: Ceph CSI Driver Deploy করা

Kubernetes এখন CSI driver ব্যবহার করে।

Deploy Ceph CSI:

```bash
kubectl create namespace ceph-csi
```

Secret তৈরি করুন:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret
  namespace: ceph-csi
stringData:
  userID: kubernetes
  userKey: <ceph-key>
```

Apply:

```bash
kubectl apply -f secret.yaml
```

---

# 📌 6️⃣ Step 3: RBD StorageClass তৈরি করা

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd
provisioner: rbd.csi.ceph.com
parameters:
  clusterID: <cluster-id>
  pool: rbd_pool
  imageFeatures: layering
reclaimPolicy: Delete
allowVolumeExpansion: true
```

Apply:

```bash
kubectl apply -f rbd-sc.yaml
```

---

# 📌 7️⃣ Step 4: RBD PVC Example (Database)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  storageClassName: ceph-rbd
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

Apply:

```bash
kubectl apply -f pvc.yaml
```

Pod এ attach করলে dynamic RBD image তৈরি হবে।

---

# 📌 8️⃣ Real Scenario: PostgreSQL Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  template:
    spec:
      containers:
      - name: postgres
        image: postgres:15
        volumeMounts:
        - mountPath: "/var/lib/postgresql/data"
          name: db-storage
      volumes:
      - name: db-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
```

Result:

- PostgreSQL data Ceph RBD তে থাকবে
    
- Pod restart হলেও data safe
    

---

# 📌 9️⃣ CephFS StorageClass (Shared Storage)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cephfs-sc
provisioner: cephfs.csi.ceph.com
parameters:
  fsName: mycephfs
  pool: cephfs_data
```

PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-media
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: cephfs-sc
  resources:
    requests:
      storage: 50Gi
```

---

# 📌 🔟 Real Scenario: Web App with Shared Upload

- 3 replica web pod
    
- Same upload directory
    

All pods mount CephFS PVC।

Result:

- Any pod upload → visible to all
    
- No NFS bottleneck
    
- Highly scalable
    

---

# 📌 1️⃣1️⃣ Backup Using RGW + Velero

Velero config:

```yaml
spec:
  provider: aws
  objectStorage:
    bucket: k8s-backup
```

Ceph RGW acts as S3 backend।

Result:

- Namespace backup
    
- PVC snapshot backup
    
- Disaster recovery ready
    

---

# 📌 1️⃣2️⃣ Scaling Example

More traffic → scale app:

```bash
kubectl scale deployment web --replicas=10
```

Ceph automatically:

- Handle more IOPS
    
- Distribute load across OSD
    

No storage downtime।

---

# 📌 1️⃣3️⃣ High Availability Model

Production Setup:

- 3 Master node
    
- 5+ Worker node
    
- 3 MON
    
- 6+ OSD
    
- Separate network
    

Failure Example:

Worker node crash → Pod reschedule → PVC reattach → No data loss

---

# 📌 1️⃣4️⃣ Performance Tuning

For Database workload:

- Use SSD-backed pool
    
- Enable RBD exclusive-lock
    
- Use separate pool for DB
    

For File workload:

- Increase MDS count
    

---

# 📌 1️⃣5️⃣ Monitoring Integration

Monitor:

- Ceph Dashboard
    
- Prometheus
    
- Kubernetes metrics
    
- Alertmanager
    

Alert example:

- PVC nearly full
    
- OSD high latency
    
- Node disk pressure
    

---

# 📌 1️⃣6️⃣ Security Best Practice

- Separate Ceph user for Kubernetes
    
- Limited pool permission
    
- Network isolation
    
- TLS enabled CSI
    

---

# 📌 1️⃣7️⃣ Production Capacity Example

Suppose:

- 6 OSD
    
- 10TB each
    
- Replication 3
    

Raw: 60TB  
Usable: 20TB

Plan 70% threshold max।

---

# 📌 1️⃣8️⃣ Full Data Flow Summary

```
User → LoadBalancer → Pod
Pod → PVC → CSI
CSI → Ceph (RBD/CephFS)
Ceph → OSD → Disk
```

---

# 🎯 Final Production Blueprint Summary

✔ HA Kubernetes  
✔ Dynamic Storage Provisioning  
✔ RBD for Database  
✔ CephFS for Shared Storage  
✔ RGW for Backup  
✔ Monitoring + Alert  
✔ Scaling without downtime  
✔ DR Ready

---
