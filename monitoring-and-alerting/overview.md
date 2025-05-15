# 📊 Monitoring & Alerts for Ceph Cluster

Monitoring is critical for maintaining a healthy and performant Ceph cluster. Ceph offers native monitoring tools and also integrates well with popular monitoring stacks like **Prometheus**, **Grafana**, and **Alertmanager**.

---

## 📋 Table of Contents

- [Why Monitoring Matters](#why-monitoring-matters)
- [Tools & Components](#tools--components)
- [Setup: Prometheus + Grafana](#setup-prometheus--grafana)
- [Ceph Metrics Exporter](#ceph-metrics-exporter)
- [Grafana Dashboards](#grafana-dashboards)
- [Alerts with Alertmanager](#alerts-with-alertmanager)
- [Best Practices](#best-practices)
- [Troubleshooting Monitoring](#troubleshooting-monitoring)

---

## ❓ Why Monitoring Matters

Monitoring helps you:

- Detect OSD/Mon/MDS failures in real-time.
- Understand storage performance trends.
- Visualize pool usage and object distribution.
- Track latency, throughput, and recovery times.
- Receive alerts before things break!

---

## 🛠 Tools & Components

| Tool         | Purpose                              |
|--------------|--------------------------------------|
| Ceph Manager | Exposes performance metrics          |
| Prometheus   | Metrics collection & storage         |
| Grafana      | Metrics visualization                |
| Alertmanager | Alert routing & notifications        |
| Node Exporter| Exports host-level metrics           |

---

## 🚀 Setup: Prometheus + Grafana

### Step 1: Enable the Ceph Prometheus Module

```bash
ceph mgr module enable prometheus
````

> 🔍 This exposes metrics at: `http://<mgr-host>:9283/metrics`

### Step 2: Deploy Prometheus

```bash
docker run -d \
  -p 9090:9090 \
  -v /etc/prometheus:/etc/prometheus \
  prom/prometheus
```

Sample `prometheus.yml` config:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'ceph'
    static_configs:
      - targets: ['<mgr-ip>:9283']
```

### Step 3: Deploy Grafana

```bash
docker run -d -p 3000:3000 grafana/grafana
```

Login: `admin / admin`
Import dashboards using JSON files or Grafana.com IDs.

---

## 📦 Ceph Metrics Exporter

The built-in `prometheus` manager module in Ceph exposes over **700+ metrics**, including:

* OSD performance (latency, throughput)
* Monitor health
* Recovery/backfill status
* Pool usage
* Cluster IOPS and usage heatmaps

Access it at:

```
http://<ceph-mgr>:9283/metrics
```

---

## 📈 Grafana Dashboards

You can download official and community dashboards:

* Grafana.com → Ceph Dashboard ID: `2842`, `5336`, `12280`
* Or import from JSON.

Example Panel Ideas:

* OSD Latency
* Pool Usage Breakdown
* Recovery Status Timeline
* Monitor Quorum Status
* Host Resource Graphs

---

## 🚨 Alerts with Alertmanager

### Step 1: Install Alertmanager

```bash
docker run -d \
  -p 9093:9093 \
  -v /etc/alertmanager:/etc/alertmanager \
  prom/alertmanager
```

Sample `alertmanager.yml`:

```yaml
route:
  receiver: 'email'

receivers:
- name: 'email'
  email_configs:
  - to: 'admin@example.com'
    from: 'ceph-monitor@example.com'
    smarthost: 'smtp.example.com:587'
    auth_username: 'ceph-monitor@example.com'
    auth_password: 'yourpassword'
```

### Step 2: Create Alert Rules in Prometheus

```yaml
groups:
- name: ceph-alerts
  rules:
  - alert: OSDDown
    expr: ceph_osd_up == 0
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "OSD Down"
      description: "OSD is down for more than 2 minutes"
```

---

## 🧠 Best Practices

* Use labels in alerting rules to define severity.
* Add Slack, PagerDuty, or Telegram integrations to Alertmanager.
* Group alerts per service to reduce noise.
* Always test alerts using:

```bash
amtool alert add test-alert
```

* Monitor the `ceph_health_status` metric regularly:

```bash
ceph_health_status{severity="HEALTH_WARN"}
```

---

## 🔧 Troubleshooting Monitoring

| Issue                           | Fix                                         |
| ------------------------------- | ------------------------------------------- |
| Prometheus not scraping metrics | Check if port `9283` is accessible          |
| Grafana dashboard empty         | Verify Prometheus data source is configured |
| Missing Ceph metrics            | Ensure Prometheus target points to MGR node |
| Alerts not triggering           | Test with synthetic rules                   |

---

## 🖼 Monitoring Architecture (Diagram)

```
+------------------+       +-------------------+        +--------------------+
|   Ceph Manager   | <---> |    Prometheus     | <--->  |     Grafana        |
|  (metrics:9283)  |       |  (metrics store)  |        |  (dashboards)      |
+------------------+       +-------------------+        +--------------------+
                                 |
                                 v
                        +--------------------+
                        |   Alertmanager     |
                        | (email/slack/etc.) |
                        +--------------------+
```

---

## 📚 References

* [Ceph Monitoring Doc](https://docs.ceph.com/en/latest/mgr/prometheus/)
* [Prometheus](https://prometheus.io/)
* [Grafana](https://grafana.com/)
* [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/)

---

> 📝 Maintained by: [Suman Pal](https://github.com/SumonPaul18)

---

