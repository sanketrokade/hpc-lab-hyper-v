# Phase 10: Monitoring (Prometheus + Grafana)

## Overview
Deploy Prometheus for metrics collection, Node Exporter on all nodes
for hardware metrics, and Grafana for visualization. Provides real-time
and historical visibility into cluster health, resource utilization,
and node status.

## Why Monitoring Matters in HPC

Monitoring answers three critical operational questions:

1. **Is the cluster healthy right now?**
   Are all nodes up? Are daemons running? Is NFS responding?

2. **Who is using what?**
   Which user is consuming all memory? Which job has been running for 3 days?

3. **What happened?**
   Node crashed at 3am — why? Historical data shows CPU spiked, memory
   hit 100%, then node went down. Without monitoring you're debugging blind.

## Stack Architecture

| Component     | Purpose                                    | Port |
|---------------|--------------------------------------------|------|
| Prometheus    | Metrics collection and time-series storage | 9090 |
| Node Exporter | Hardware metrics exporter (all nodes)      | 9100 |
| Grafana       | Visualization and dashboards               | 3000 |

**Data flow:**
```
headnode:9100  ──┐
compute-1:9100 ──┤──► Prometheus:9090 ──► Grafana:3000 ──► Browser
compute-2:9100 ──┘
```

Prometheus scrapes Node Exporter on all nodes every 15 seconds.
Grafana queries Prometheus and renders dashboards.
Browser accesses Grafana via headnode's LabSwitch1 IP (192.168.30.2).

## Design Decisions

- **Prometheus on headnode** — central collection point, sufficient for 3-node lab
- **Node Exporter on all nodes** — hardware metrics from every node
- **Grafana on headnode** — accessible from Windows host via LabSwitch1
- **Grafana on public zone firewall** — needs to be reachable from host machine
- **Prometheus on trusted zone only** — internal cluster use, not exposed externally
- **30 day retention** — sufficient for lab, production typically 90-365 days
- **Prometheus LTS 3.5.2** — stability over latest features
- **Pre-built dashboard (ID 1860)** — Node Exporter Full, 20M+ downloads

**Single point of failure note:** In production, Prometheus and Grafana run
on dedicated monitoring nodes, sometimes with HA. For a 3-node lab,
headnode overhead is acceptable.

## Prerequisites
- All nodes networked and reachable by hostname
- Firewall configured (Phase 4 complete)
- Internet access on headnode for downloads

## 10.1 Install Prometheus

### Create System User and Directories

```bash
useradd -r -s /sbin/nologin -d /var/lib/prometheus -c "Prometheus" prometheus
mkdir -p /etc/prometheus
mkdir -p /var/lib/prometheus
mkdir -p /var/log/prometheus
```

### Download and Install

```bash
cd /tmp
curl -LO https://github.com/prometheus/prometheus/releases/download/v3.5.2/prometheus-3.5.2.linux-amd64.tar.gz
tar xzf prometheus-3.5.2.linux-amd64.tar.gz

cp /tmp/prometheus-3.5.2.linux-amd64/prometheus /usr/local/bin/
cp /tmp/prometheus-3.5.2.linux-amd64/promtool /usr/local/bin/
cp /tmp/prometheus-3.5.2.linux-amd64/prometheus.yml /etc/prometheus/

chown -R prometheus:prometheus /etc/prometheus
chown -R prometheus:prometheus /var/lib/prometheus
chown -R prometheus:prometheus /var/log/prometheus
chown prometheus:prometheus /usr/local/bin/prometheus
chown prometheus:prometheus /usr/local/bin/promtool
```

### Configure Prometheus

```bash
cat > /etc/prometheus/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
        labels:
          instance: "headnode"

  - job_name: "node"
    static_configs:
      - targets:
          - "headnode:9100"
          - "compute-1:9100"
          - "compute-2:9100"
        labels:
          cluster: "hpc-lab"
EOF

chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

**Why `cluster: "hpc-lab"` label?**
Every metric from every node gets tagged with the cluster name. In a
multi-cluster environment you can filter by cluster instantly. Good habit
even in a single-cluster lab.

### Create Systemd Service

```bash
cat > /etc/systemd/system/prometheus.service << 'EOF'
[Unit]
Description=Prometheus Monitoring
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --storage.tsdb.retention.time=30d \
  --web.listen-address=0.0.0.0:9090
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now prometheus
```

### Configure Firewall

```bash
# Trusted zone — cluster internal access
firewall-cmd --zone=trusted --add-port=9090/tcp --permanent
# Public zone — host browser access
firewall-cmd --zone=public --add-port=9090/tcp --permanent
firewall-cmd --reload
```

### Verify

```bash
systemctl status prometheus --no-pager
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool | grep health
```

## 10.2 Install Node Exporter

Node Exporter must run on **all three nodes** to expose hardware metrics.

### Download on Headnode

```bash
cd /tmp
curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.11.1/node_exporter-1.11.1.linux-amd64.tar.gz
tar xzf node_exporter-1.11.1.linux-amd64.tar.gz
```

### Install on Headnode

```bash
useradd -r -s /sbin/nologin -c "Node Exporter" node_exporter
cp /tmp/node_exporter-1.11.1.linux-amd64/node_exporter /usr/local/bin/
chown node_exporter:node_exporter /usr/local/bin/node_exporter

cat > /etc/systemd/system/node_exporter.service << 'EOF'
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now node_exporter
firewall-cmd --zone=trusted --add-port=9100/tcp --permanent
firewall-cmd --reload
```

### Distribute to Compute Nodes

```bash
scp /usr/local/bin/node_exporter root@compute-1:/usr/local/bin/
scp /usr/local/bin/node_exporter root@compute-2:/usr/local/bin/
scp /etc/systemd/system/node_exporter.service root@compute-1:/etc/systemd/system/
scp /etc/systemd/system/node_exporter.service root@compute-2:/etc/systemd/system/

# Setup on compute-1
ssh compute-1 "useradd -r -s /sbin/nologin -c 'Node Exporter' node_exporter && \
  chown node_exporter:node_exporter /usr/local/bin/node_exporter && \
  systemctl daemon-reload && \
  systemctl enable --now node_exporter && \
  firewall-cmd --add-port=9100/tcp --permanent && \
  firewall-cmd --reload"

# Setup on compute-2
ssh compute-2 "useradd -r -s /sbin/nologin -c 'Node Exporter' node_exporter && \
  chown node_exporter:node_exporter /usr/local/bin/node_exporter && \
  systemctl daemon-reload && \
  systemctl enable --now node_exporter && \
  firewall-cmd --add-port=9100/tcp --permanent && \
  firewall-cmd --reload"
```

### Verify All Nodes

```bash
systemctl status node_exporter --no-pager | grep Active
ssh compute-1 "systemctl status node_exporter --no-pager | grep Active"
ssh compute-2 "systemctl status node_exporter --no-pager | grep Active"
```

All three must show `active (running)`.

### Verify Prometheus Scraping

```bash
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool | \
  grep -E "health|instance" | head -20
```

All three node targets must show `"health": "up"`.

Test with a PromQL query:
```bash
curl -s 'http://localhost:9090/api/v1/query?query=node_memory_MemTotal_bytes' | \
  python3 -m json.tool | grep -E "instance|value"
```

Expected — three results with correct memory values:
```
headnode:9100   → ~8GB
compute-1:9100  → ~3.8GB
compute-2:9100  → ~3.8GB
```

## 10.3 Install Grafana

### Add Grafana Repository

```bash
cat > /etc/yum.repos.d/grafana.repo << 'EOF'
[grafana]
name=grafana
baseurl=https://rpm.grafana.com
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://rpm.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
EOF

dnf install -y grafana
```

### Start Grafana

```bash
systemctl enable --now grafana-server
```

### Configure Firewall

Grafana needs to be accessible from the Windows host via LabSwitch1.
Open on public zone — not just trusted:

```bash
firewall-cmd --zone=public --add-port=3000/tcp --permanent
firewall-cmd --reload
```

### Access Grafana

From Windows host browser: `http://192.168.30.2:3000`

Default credentials:
- Username: `admin`
- Password: `admin`

Change password on first login.

## 10.4 Connect Prometheus to Grafana

1. Click **Connections** → **Data sources**
2. Click **Add data source**
3. Select **Prometheus**
4. Set URL: `http://localhost:9090`
5. Click **Save & test**

Expected: `Successfully queried the Prometheus API.`

## 10.5 Import Node Exporter Dashboard

1. Click **Dashboards** → **New** → **Import**
2. Enter dashboard ID: `1860`
3. Click **Load**
4. Select Prometheus data source
5. Click **Import**

Dashboard ID 1860 is "Node Exporter Full" — most widely used Linux/HPC
monitoring dashboard with 20M+ downloads.

### Dashboard Features

| Panel              | What it shows                          |
|--------------------|----------------------------------------|
| CPU Busy           | Real-time CPU utilization %            |
| RAM Used           | Memory usage % and totals              |
| Root FS Used       | Disk usage %                           |
| CPU Cores          | Total CPU count                        |
| Uptime             | Node uptime                            |
| Network Traffic    | Rx/Tx per interface                    |
| Disk Space Used    | Per-mount utilization including NFS    |

Switch between nodes using the **Nodename** dropdown at the top.

### Verified Metrics Per Node

| Node      | RAM Total | CPU Cores | Interfaces      |
|-----------|-----------|-----------|-----------------|
| headnode  | 7 GiB     | 4         | eth0, eth1, lo  |
| compute-1 | 4 GiB     | 4         | eth0, lo        |
| compute-2 | 4 GiB     | 4         | eth0, lo        |

headnode shows two interfaces (LabSwitch1 + ClusterSwitch).
Compute nodes show one interface (ClusterSwitch only).
NFS mounts visible in disk panels on all nodes.

## Key Files Reference

| File                                    | Purpose                    |
|-----------------------------------------|----------------------------|
| /etc/prometheus/prometheus.yml          | Prometheus configuration   |
| /etc/systemd/system/prometheus.service  | Prometheus systemd unit    |
| /etc/systemd/system/node_exporter.service | Node Exporter unit       |
| /var/lib/prometheus/                    | Prometheus TSDB storage    |
| /etc/yum.repos.d/grafana.repo           | Grafana repository         |

## Useful PromQL Queries

```promql
# CPU usage per node
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage per node
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100

# Disk usage per mount
(node_filesystem_size_bytes - node_filesystem_free_bytes) / node_filesystem_size_bytes * 100

# Network receive rate
rate(node_network_receive_bytes_total[5m])

# Total memory per node
node_memory_MemTotal_bytes
```

## Troubleshooting

### Prometheus targets show "down"
Check Node Exporter is running on the target node:
```bash
ssh compute-1 "systemctl status node_exporter"
```
Check port is open:
```bash
bash -c "echo > /dev/tcp/compute-1/9100" && echo "open" || echo "closed"
```

### Grafana not accessible from browser
Check firewall — must be public zone, not just trusted:
```bash
firewall-cmd --zone=public --list-ports | grep 3000
```

### No data in Grafana dashboard
Check Prometheus data source connection:
- Connections → Data sources → prometheus → Save & test
Check time range — ensure dashboard time range covers when data was collected.

### Node Exporter shows wrong metrics
Node Exporter reads directly from kernel — metrics are always accurate.
If values seem wrong, verify against system tools:
```bash
free -h        # compare with RAM metrics
df -h          # compare with disk metrics
top            # compare with CPU metrics
```
