# File Name: `Ceph_RGW_Proxmox_Deployment_Guide.md`

---

#  Comprehensive Practical Guide: Deploying & Managing Ceph RGW Object Storage on Proxmox VE

## 📑 Table of Contents

1.  **Introduction & Architecture Overview**
2.  **Prerequisites & Environment Setup**
3.  **Deploying RADOS Gateway (RGW) Service**
4.  **User Management & Access Key Generation**
5.  **Client-Side Connectivity: Troubleshooting & Configuration**
6.  **Operational Guide: Using MinIO Client (`mc`) for Daily Tasks**
7.  **Advanced Operations: Versioning, Lifecycle, and Policies**
8.  **Application Integration & Real-World Use Cases**
9.  **Monitoring, Maintenance, and Cleanup**

---

## 1. Introduction & Architecture Overview

### What is Ceph RGW?
In a modern Cloud Infrastructure, specifically within a Proxmox VE environment running a Ceph Cluster, **RADOS Gateway (RGW)** serves as the interface that transforms your block-based Ceph storage into an **Object Storage** system. It provides compatibility with **Amazon S3** and **OpenStack Swift** APIs.

### Why Use RGW?
*   **Unstructured Data:** Ideal for storing images, videos, logs, backups, and static web assets.
*   **Scalability:** Unlike traditional file systems, object storage scales horizontally without performance degradation.
*   **Cost Efficiency:** Eliminates the need for expensive public cloud storage (like AWS S3) by hosting data locally in your data center (e.g., Paulco Cloud R&D Lab).
*   **Data Sovereignty:** Keeps sensitive data within local borders, complying with regional data protection laws.

### When to Use RGW vs. Other Ceph Interfaces?
| Feature | Ceph RGW (Object) | CephFS (File) | RBD (Block) |
| :--- | :--- | :--- | :--- |
| **Protocol** | S3 / Swift API | POSIX (NFS/CIFS) | iSCSI / Librbd |
| **Structure** | Flat (Bucket/Object) | Hierarchical (Dir/File) | Raw Block Device |
| **Use Case** | Web Assets, Backups, Logs | Home Directories, Shared Docs | VM Disks, Databases |
| **Complexity** | Medium | High (Requires MDS) | Low |

> **Note:** RGW is **disabled by default** in Ceph clusters managed by `cephadm`. It must be explicitly deployed.

---

## 2. Prerequisites & Environment Setup

Before deploying RGW, ensure your infrastructure meets the following requirements.

### Infrastructure Requirements
*   **Proxmox VE Cluster:** At least 3-5 nodes for high availability.
*   **Ceph Cluster:** A healthy Ceph cluster managed by `cephadm` (Quincy or Reef version recommended).
*   **Network:**
    *   All nodes must be on the same network segment or have proper routing.
    *   Firewalls must allow traffic on the RGW port (Default: `7480` or Custom: `8080`).
*   **Client Machine:** A Linux machine (Ubuntu/Debian/CentOS) or Windows/Mac with terminal access to test connectivity.

### Verification Steps on Admin Node
Log in to your Ceph Admin node (usually the first node where you bootstrapped the cluster) and verify the cluster health.

```bash
# Check Ceph Health
ceph -s

# Verify existing services
ceph orch ls

# Check if RGW is already running
ceph orch ps | grep rgw
```

If `ceph orch ps | grep rgw` returns nothing, RGW is not yet deployed. Proceed to the next step.

---

## 3. Deploying RADOS Gateway (RGW) Service

We will use `cephadm` to deploy the RGW service. This method ensures that the service is containerized, monitored, and automatically restarted if it fails.

### Step 3.1: Choose Deployment Strategy
You can deploy RGW on specific nodes or all nodes. For a 5-node cluster, deploying on 3 nodes is recommended for High Availability (HA).

### Step 3.2: Deploy RGW via Command Line
Run the following command on the Ceph Admin Node.

```bash
# Syntax: ceph orch apply rgw <service_name> --placement="<count> <host1> <host2> ..." --port=<port>

# Example: Deploy 'my-rgw' on 3 nodes (ceph1, ceph2, ceph3) using port 7480 (Default)
ceph orch apply rgw my-rgw --placement="3 ceph1 ceph2 ceph3" --port=7480
```

> **Why Port 7480?** This is the standard default port for Ceph RGW. If you prefer `8080`, you can change it, but ensure your firewall allows it. In our troubleshooting journey, we found that `7480` was the active port.

### Step 3.3: Deploy RGW via YAML (Recommended for Production)
For better reproducibility and version control, create a YAML specification file.

1.  Create a file named `rgw-spec.yaml`:

```yaml
service_type: rgw
service_id: my-rgw
placement:
  count: 3
  hosts:
    - ceph1
    - ceph2
    - ceph3
spec:
  rgw_frontend_port: 7480
  rgw_realm: default
  rgw_zone: default
```

2.  Apply the configuration:

```bash
ceph orch apply -i rgw-spec.yaml
```

### Step 3.4: Verification
Wait for 1-2 minutes for the containers to spin up.

```bash
# Check if RGW processes are running
ceph orch ps | grep rgw

# Expected Output:
# rgw.my-rgw.ceph1...  ceph1  *:7480  running ...
# rgw.my-rgw.ceph2...  ceph2  *:7480  running ...
# rgw.my-rgw.ceph3...  ceph3  *:7480  running ...

# Check Cluster Status
ceph -s
```

Ensure the state is `running` and the cluster health is `HEALTH_OK`.

---

## 4. User Management & Access Key Generation

To interact with Object Storage, you need credentials similar to AWS IAM Users: an **Access Key** and a **Secret Key**.

### Step 4.1: Create an RGW User
On the Ceph Admin Node, run:

```bash
# Create a user with UID 'sumon-user'
radosgw-admin user create --uid="sumon-user" --display-name="Sumon Admin"
```

### Step 4.2: Extract Credentials
The output will contain a `keys` section. Look for:
*   `access_key`: e.g., `R0IALSX6V6ZU3AHP2CJM`
*   `secret_key`: e.g., `UuoVCABbyITOmsxwR3UfdEtvsIQAagyyVjcCJo7A`

> **️ Security Warning:** Save these keys securely. The Secret Key is shown only once during creation. If lost, you must generate a new key pair.

### Step 4.3: Verify User Info
```bash
radosgw-admin user info --uid=sumon-user
```

---

## 5. Client-Side Connectivity: Troubleshooting & Configuration

This section details how to connect from a local client (e.g., Ubuntu Desktop) to the Ceph RGW. We encountered several common issues during setup, which are addressed below.

### Step 5.1: Install Client Tools
We recommend **MinIO Client (`mc`)** over `s3cmd` due to its modern architecture, better SigV4 support, and ease of use. However, `s3cmd` is also covered for legacy compatibility.

#### Option A: Install MinIO Client (`mc`) - **Recommended**
```bash
# Download the latest binary
wget https://dl.min.io/client/mc/release/linux-amd64/mc

# Make it executable
chmod +x mc

# Move to system path
sudo mv mc /usr/local/bin/

# Verify installation
mc --version
```

#### Option B: Install s3cmd (Legacy)
```bash
sudo apt install s3cmd -y
```

### Step 5.2: Configure Connectivity

#### Using MinIO Client (`mc`)
This is the most straightforward method.

```bash
# Syntax: mc alias set <ALIAS> <ENDPOINT> <ACCESS_KEY> <SECRET_KEY>

mc alias set myceph http://192.168.68.249:7480 R0IALSX6V6ZU3AHP2CJM UuoVCABbyITOmsxwR3UfdEtvsIQAagyyVjcCJo7A
```

*   **Endpoint:** `http://<RGW_Node_IP>:<Port>` (Use HTTP if SSL is not configured).
*   **Verification:**
    ```bash
    mc ls myceph
    ```
    If this lists buckets (or shows empty without error), connectivity is successful.

#### Using s3cmd (Troubleshooting Guide)
If you must use `s3cmd`, follow these critical steps to avoid `Connection Refused` and `SignatureDoesNotMatch` errors.

1.  **Initial Configuration:**
    ```bash
    s3cmd --configure
    ```
    *   **Access Key:** Enter your Access Key.
    *   **Secret Key:** Enter your Secret Key.
    *   **S3 Endpoint:** `192.168.68.249:7480` (Note: Use the correct port discovered via `ceph orch ps`).
    *   **Use HTTPS protocol?** **No** (Unless you have valid SSL certs).
    *   **Test Access:** Say `Y`. If it fails with `Connection Refused`, check Firewall and Port.

2.  **Fixing `Name or Service Not Known` (DNS Issue):**
    By default, `s3cmd` uses Virtual Hosted Style (`bucket.host.com`). Local labs lack DNS for this. You must switch to **Path Style**.

    Edit `/root/.s3cfg`:
    ```ini
    host_base = 192.168.68.249:7480
    host_bucket = 192.168.68.249:7480/%(bucket)s
    use_path_style = True
    use_https = False
    ```

3.  **Fixing `SignatureDoesNotMatch`:**
    *   **Check Time Sync:** Ensure both Client and Server times are synchronized using NTP. A difference of >5 minutes causes signature failures.
        ```bash
        sudo ntpdate pool.ntp.org
        ```
    *   **Verify Keys:** Re-copy keys from `radosgw-admin user info` ensuring no extra spaces.

---

## 6. Operational Guide: Using MinIO Client (`mc`) for Daily Tasks

Once connected, here are the practical commands for daily operations.

### Step 6.1: Bucket Management

```bash
# Create a new bucket
mc mb myceph/my-app-bucket

# List all buckets
mc ls myceph

# Delete an empty bucket
mc rb myceph/old-bucket

# Force delete a bucket with content
mc rb --force myceph/old-bucket
```

### Step 6.2: File Upload & Download

```bash
# Create a test file
echo "Hello Ceph RGW" > hello.txt

# Upload file to bucket
mc cp hello.txt myceph/my-app-bucket/

# List files in bucket
mc ls myceph/my-app-bucket/

# Download file
mc cp myceph/my-app-bucket/hello.txt ./downloaded_hello.txt

# Upload entire directory recursively
mc cp --recursive local-folder/ myceph/my-app-bucket/folder-backup/
```

### Step 6.3: Public Access (Static Website Hosting)
To make a bucket publicly readable (e.g., for website images):

```bash
# Set anonymous read-only access
mc anonymous set download myceph/my-app-bucket

# Access URL: http://192.168.68.249:7480/my-app-bucket/hello.txt
```

> **Security Note:** Only use this for non-sensitive static assets. Never expose private data.

---

## 7. Advanced Operations: Versioning, Lifecycle, and Policies

### Step 7.1: Enable Versioning
Protect against accidental deletion or overwriting by enabling versioning.

```bash
# Enable versioning on a bucket
mc version enable myceph/my-app-bucket

# List versions of a file
mc version list myceph/my-app-bucket/hello.txt
```

### Step 7.2: Lifecycle Policies (Auto-Delete)
Automatically delete old log files to save space.

1.  Create a JSON policy file (`lifecycle.json`):
    ```json
    {
      "Rules": [
        {
          "ID": "DeleteOldLogs",
          "Status": "Enabled",
          "Filter": { "Prefix": "" },
          "Expiration": { "Days": 7 }
        }
      ]
    }
    ```

2.  Import the policy:
    ```bash
    mc ilm import myceph/my-app-bucket < lifecycle.json
    ```

### Step 7.3: User Quotas
Limit storage usage per user on the Ceph Admin Node.

```bash
# Set a 10GB quota for user 'sumon-user'
radosgw-admin quota set --uid=sumon-user --quota-scope=user --max-size=10G
```

---

## 8. Application Integration & Real-World Use Cases

### Scenario: Video Streaming Application
You are building a video platform where users upload videos. Instead of storing them on the web server's disk, you store them in Ceph RGW.

#### Python (Boto3) Integration Example
Install the library:
```bash
pip install boto3
```

Python Script (`upload_video.py`):
```python
import boto3
from botocore.client import Config

# Configure S3 Client for Ceph RGW
s3_client = boto3.client(
    's3',
    endpoint_url='http://192.168.68.249:7480',
    aws_access_key_id='R0IALSX6V6ZU3AHP2CJM',
    aws_secret_access_key='UuoVCABbyITOmsxwR3UfdEtvsIQAagyyVjcCJo7A',
    config=Config(signature_version='s3v4'),
    region_name='us-east-1'
)

# Upload Video
with open("video.mp4", "rb") as data:
    s3_client.upload_fileobj(data, 'user-videos', 'video_001.mp4')

print("Video uploaded successfully to local Ceph Storage!")
```

### Scenario: Automated Backup with `mc mirror`
Sync data from your local Ceph to another location (e.g., another Ceph cluster or AWS S3) for disaster recovery.

```bash
# Mirror local bucket to remote backup bucket
mc mirror myceph/production-data aws-s3/backup-data
```

---

## 9. Monitoring, Maintenance, and Cleanup

### Step 9.1: Monitoring with Prometheus & Grafana
Ceph exposes metrics that can be scraped by Prometheus.

1.  **Enable Ceph Dashboard Module:**
    ```bash
    ceph mgr module enable dashboard
    ceph dashboard create-self-signed-cert
    ```
2.  **Access Dashboard:** Navigate to `https://<Mgr_Node_IP>:8443` to view RGW request rates, latency, and errors.
3.  **Grafana:** Import the "Ceph Cluster" dashboard (ID: 2842) to visualize RGW metrics alongside other cluster stats.

### Step 9.2: Routine Maintenance
*   **Check RGW Logs:**
    ```bash
    # View logs for a specific RGW daemon
    ceph orch ps --daemon-type rgw
    # Or check container logs directly on the node
    docker logs <container_id>
    ```
*   **Restart RGW Service:**
    ```bash
    ceph orch restart rgw.my-rgw
    ```

### Step 9.3: Uninstalling & Cleanup
If you need to remove RGW from the cluster:

1.  **Remove the Service:**
    ```bash
    ceph orch rm rgw.my-rgw
    ```
2.  **Delete Users & Buckets (Optional):**
    *   Delete buckets using `mc rb --force myceph/<bucket>`.
    *   Remove users on Admin Node:
        ```bash
        radosgw-admin user rm --uid=sumon-user
        ```
3.  **Clean Up Client Configs:**
    *   Remove `~/.mc/config.json` or `~/.s3cfg` from client machines.

---

## 💡 Final Best Practices Summary

1.  **Use `mc` over `s3cmd`:** It is more robust, handles SigV4 better, and requires less configuration.
2.  **Time Sync is Critical:** Always ensure NTP is running on all Ceph nodes and clients to prevent `SignatureDoesNotMatch` errors.
3.  **Path Style for Local Labs:** If using `s3cmd`, always configure `use_path_style = True` and set `host_bucket` to `IP:PORT/%(bucket)s` to avoid DNS resolution issues.
4.  **Secure Keys:** Treat Access/Secret Keys like passwords. Rotate them periodically.
5.  **Monitor Metrics:** Keep an eye on RGW latency and error rates in Grafana to detect issues early.

This guide provides a complete, end-to-end workflow for deploying and managing Ceph RGW in a Proxmox environment, tailored for practical, real-world application.