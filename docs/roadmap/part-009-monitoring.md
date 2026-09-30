# Part 009: Monitoring พื้นฐาน
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Beginner
> **Steps:** 81-90
> **เวลาโดยประมาณ:** 6 ชั่วโมง
> **Prerequisites:** Part 001-008 (Linux, Git, Node.js, PostgreSQL, Redis, Docker, Cloudflare, CI/CD)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

1. เข้าใจ Monitoring stack: Prometheus + Grafana + Alertmanager
2. ติดตั้ง Prometheus บน Ubuntu 24.04 (binary + systemd)
3. ตั้งค่า prometheus.yml พร้อม scrape configs
4. ติดตั้ง Node Exporter สำหรับ system metrics
5. ติดตั้ง Grafana และตั้งค่า datasource
6. Import dashboards ที่มีอยู่แล้ว (Node Exporter Full, PostgreSQL)
7. สร้าง custom dashboard สำหรับ chuaikan.com
8. ตั้งค่า Alert Rules สำหรับ critical issues
9. ตั้งค่า Alertmanager ส่ง notifications ผ่าน Discord
10. ติดตั้ง exporters สำหรับ PostgreSQL, Redis, Node.js

---

## 📖 ทฤษฎีและแนวคิด

### Monitoring Stack Overview

```
┌─────────────────────────────────────────────────────────┐
│                   Monitoring Stack                       │
│                                                         │
│   ┌──────────┐  scrape   ┌────────────┐                │
│   │ Your App │ ────────→ │ Prometheus │                │
│   │ (metrics)│           │ (TSDB)     │                │
│   └──────────┘           └─────┬──────┘                │
│                                │                        │
│   ┌──────────┐                 │ query                 │
│   │   Node   │ ────────→       ↓                       │
│   │ Exporter │           ┌────────────┐                │
│   └──────────┘           │  Grafana   │ ← User Views   │
│                           │(Dashboard) │                │
│   ┌──────────┐           └─────┬──────┘                │
│   │PostgreSQL│ ────────→       │                        │
│   │ Exporter │           ┌─────↓──────┐                │
│   └──────────┘           │ Alert-     │                │
│                           │ manager   │ → Discord       │
│   ┌──────────┐           └────────────┘                │
│   │  Redis   │                                         │
│   │ Exporter │                                         │
│   └──────────┘                                         │
└─────────────────────────────────────────────────────────┘
```

### Metrics Types ใน Prometheus

```
Counter: ตัวเลขที่เพิ่มขึ้นเรื่อยๆ (ไม่ลดลง)
  ตัวอย่าง: http_requests_total, errors_total

Gauge: ตัวเลขที่เพิ่มหรือลดได้
  ตัวอย่าง: memory_usage, active_users, temperature

Histogram: กระจาย requests ตาม buckets
  ตัวอย่าง: request_duration_seconds (0.1s, 0.5s, 1s, 5s)

Summary: คล้าย Histogram แต่คำนวณ quantiles ที่ client
  ตัวอย่าง: request_duration_quantile{quantile="0.95"}
```

---

## ⚙️ Environment Setup

### Step 81: ติดตั้ง Prometheus บน Ubuntu 24.04

```bash
# ดาวน์โหลด Prometheus (ตรวจสอบ version ล่าสุดที่ github.com/prometheus/prometheus)
PROM_VERSION="2.49.1"
wget -q https://github.com/prometheus/prometheus/releases/download/v${PROM_VERSION}/prometheus-${PROM_VERSION}.linux-amd64.tar.gz

# แตกไฟล์
tar xzf prometheus-${PROM_VERSION}.linux-amd64.tar.gz
cd prometheus-${PROM_VERSION}.linux-amd64

# สร้าง user สำหรับ prometheus (ไม่มี home, ไม่มี shell)
sudo useradd --no-create-home --shell /bin/false prometheus

# สร้าง directories
sudo mkdir -p /etc/prometheus /var/lib/prometheus

# Copy binaries
sudo cp prometheus promtool /usr/local/bin/
sudo cp -r consoles/ console_libraries/ /etc/prometheus/

# ตั้ง permissions
sudo chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus

# ตรวจสอบ version
prometheus --version
# prometheus, version 2.49.1 (branch: HEAD, ...)
```

สร้าง systemd service สำหรับ Prometheus:

```bash
sudo tee /etc/systemd/system/prometheus.service <<'EOF'
[Unit]
Description=Prometheus Monitoring System
Documentation=https://prometheus.io/docs/introduction/overview/
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecReload=/bin/kill -HUP $MAINPID
ExecStart=/usr/local/bin/prometheus \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/var/lib/prometheus/data \
    --storage.tsdb.retention.time=15d \
    --storage.tsdb.retention.size=5GB \
    --web.enable-lifecycle \
    --web.enable-admin-api \
    --web.listen-address=127.0.0.1:9090 \
    --log.level=info

SyslogIdentifier=prometheus
Restart=always
RestartSec=5s

# Security hardening
NoNewPrivileges=yes
ProtectHome=yes
ProtectSystem=strict
ReadWritePaths=/var/lib/prometheus

[Install]
WantedBy=multi-user.target
EOF

# เปิดใช้งาน
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

### Step 82: ติดตั้ง Node Exporter

```bash
# ดาวน์โหลด Node Exporter
NODE_EXP_VERSION="1.7.0"
wget -q https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXP_VERSION}/node_exporter-${NODE_EXP_VERSION}.linux-amd64.tar.gz

tar xzf node_exporter-${NODE_EXP_VERSION}.linux-amd64.tar.gz
sudo cp node_exporter-${NODE_EXP_VERSION}.linux-amd64/node_exporter /usr/local/bin/

sudo useradd --no-create-home --shell /bin/false node_exporter

sudo tee /etc/systemd/system/node_exporter.service <<'EOF'
[Unit]
Description=Prometheus Node Exporter
Documentation=https://github.com/prometheus/node_exporter
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=node_exporter
Group=node_exporter
ExecStart=/usr/local/bin/node_exporter \
    --collector.systemd \
    --collector.processes \
    --collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/) \
    --web.listen-address=127.0.0.1:9100
Restart=always
RestartSec=5s
NoNewPrivileges=yes

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter

# ทดสอบ
curl -s http://localhost:9100/metrics | grep node_cpu_seconds_total | head -5
```

---

## 🛠️ Step-by-Step Implementation

### Step 83: prometheus.yml Configuration

```bash
sudo tee /etc/prometheus/prometheus.yml <<'EOF'
# prometheus.yml — chuaikan.com Monitoring Configuration
global:
  scrape_interval: 15s          # scrape ทุก 15 วินาที
  evaluation_interval: 15s      # evaluate rules ทุก 15 วินาที
  scrape_timeout: 10s
  
  # Labels ที่จะแนบกับ metrics ทั้งหมด
  external_labels:
    environment: 'production'
    project: 'chuaikan'
    region: 'asia-southeast1'

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - 'localhost:9093'
      timeout: 10s

# Load alert rules
rule_files:
  - '/etc/prometheus/rules/*.yml'

# ============================
# Scrape Configurations
# ============================
scrape_configs:

  # Prometheus self-monitoring
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
        labels:
          instance: 'prometheus-server'

  # Node Exporter (system metrics)
  - job_name: 'node'
    static_configs:
      - targets:
          - 'localhost:9100'
        labels:
          instance: 'chuaikan-prod-01'
          server_type: 'web'
    # Relabel config
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: '([^:]+)(:[0-9]+)?'
        replacement: '${1}'

  # PostgreSQL Exporter
  - job_name: 'postgres'
    static_configs:
      - targets: ['localhost:9187']
        labels:
          instance: 'postgres-prod'
          db_name: 'chuaikan'

  # Redis Exporter
  - job_name: 'redis'
    static_configs:
      - targets: ['localhost:9121']
        labels:
          instance: 'redis-prod'

  # Next.js Application
  - job_name: 'nextjs'
    metrics_path: '/api/metrics'
    static_configs:
      - targets: ['localhost:3000']
        labels:
          app: 'chuaikan-web'
          version: '1.0.0'
    # Auth header ถ้า metrics endpoint ต้องการ
    # authorization:
    #   credentials: 'your-metrics-secret'

  # Node.js API
  - job_name: 'api'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['localhost:4000']
        labels:
          app: 'chuaikan-api'

  # Blackbox Exporter (uptime monitoring)
  - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://chuaikan.com
          - https://chuaikan.com/api/health
          - https://api.chuaikan.com/health
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9115

EOF

# Validate config
promtool check config /etc/prometheus/prometheus.yml

# Reload config
sudo systemctl reload prometheus
```

### Step 84: ติดตั้ง Grafana

```bash
# ติดตั้ง Grafana จาก official repository
sudo apt-get install -y apt-transport-https software-properties-common wget

sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | \
    gpg --dearmor | \
    sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | \
    sudo tee -a /etc/apt/sources.list.d/grafana.list

sudo apt-get update
sudo apt-get install -y grafana

# เปิดใช้งาน
sudo systemctl daemon-reload
sudo systemctl enable grafana-server
sudo systemctl start grafana-server

# ตรวจสอบ status
sudo systemctl status grafana-server

echo "Grafana available at: http://localhost:3001"
echo "Default credentials: admin/admin"
```

ตั้งค่า Grafana:

```bash
# แก้ไข /etc/grafana/grafana.ini
sudo tee -a /etc/grafana/grafana.ini <<'EOF'

[server]
http_port = 3001
domain = monitoring.chuaikan.com
root_url = https://monitoring.chuaikan.com

[security]
admin_user = admin
# admin_password = ตั้งรหัสผ่านใหม่ใน UI

[users]
allow_sign_up = false

[auth.anonymous]
enabled = false

[alerting]
enabled = true

[unified_alerting]
enabled = true

[smtp]
enabled = false

EOF

sudo systemctl restart grafana-server
```

ตั้งค่า Prometheus Datasource ผ่าน API:

```bash
# รอ Grafana เริ่มทำงาน
sleep 10

# ตั้งค่า Prometheus datasource
curl -s -X POST \
    http://admin:admin@localhost:3001/api/datasources \
    -H "Content-Type: application/json" \
    --data '{
        "name": "Prometheus",
        "type": "prometheus",
        "url": "http://localhost:9090",
        "access": "proxy",
        "isDefault": true,
        "jsonData": {
            "timeInterval": "15s",
            "httpMethod": "POST"
        }
    }' | jq '.message'
# Expected: "Datasource added"
```

### Step 85: Import Grafana Dashboards

```bash
# Import Node Exporter Full Dashboard (ID: 1860)
curl -s -X POST \
    http://admin:admin@localhost:3001/api/dashboards/import \
    -H "Content-Type: application/json" \
    --data '{
        "dashboard": null,
        "inputs": [{"name": "DS_PROMETHEUS", "pluginId": "prometheus", "type": "datasource", "value": "Prometheus"}],
        "folderId": 0,
        "overwrite": false,
        "path": "1860"
    }' | jq '.status'

# Download dashboard จาก Grafana.com แล้ว import
DASHBOARD_ID=1860
curl -s "https://grafana.com/api/dashboards/${DASHBOARD_ID}/revisions/latest/download" \
    -o /tmp/dashboard-${DASHBOARD_ID}.json

# Import dashboard
curl -s -X POST \
    http://admin:admin@localhost:3001/api/dashboards/db \
    -H "Content-Type: application/json" \
    --data "{
        \"dashboard\": $(cat /tmp/dashboard-${DASHBOARD_ID}.json),
        \"overwrite\": true,
        \"inputs\": [{
            \"name\": \"DS_PROMETHEUS\",
            \"type\": \"datasource\",
            \"pluginId\": \"prometheus\",
            \"value\": \"Prometheus\"
        }]
    }" | jq '.status'
```

### Step 86: Custom Dashboard สำหรับ chuaikan.com

สร้างไฟล์ `/etc/grafana/provisioning/dashboards/chuaikan.json`:

```json
{
  "annotations": {
    "list": []
  },
  "description": "chuaikan.com — Application Dashboard",
  "panels": [
    {
      "datasource": "Prometheus",
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "thresholds" },
          "thresholds": {
            "steps": [
              { "color": "green", "value": null },
              { "color": "yellow", "value": 500 },
              { "color": "red", "value": 1000 }
            ]
          },
          "unit": "reqps"
        }
      },
      "gridPos": { "h": 4, "w": 6, "x": 0, "y": 0 },
      "id": 1,
      "options": { "reduceOptions": { "calcs": ["lastNotNull"] } },
      "title": "Requests per Second",
      "type": "stat",
      "targets": [{
        "expr": "sum(rate(http_requests_total{app='chuaikan-api'}[5m]))",
        "legendFormat": "RPS"
      }]
    },
    {
      "datasource": "Prometheus",
      "fieldConfig": {
        "defaults": {
          "unit": "ms",
          "thresholds": {
            "steps": [
              { "color": "green", "value": null },
              { "color": "yellow", "value": 500 },
              { "color": "red", "value": 1000 }
            ]
          }
        }
      },
      "gridPos": { "h": 4, "w": 6, "x": 6, "y": 0 },
      "id": 2,
      "title": "P95 Latency",
      "type": "stat",
      "targets": [{
        "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{app='chuaikan-api'}[5m])) by (le)) * 1000",
        "legendFormat": "P95"
      }]
    },
    {
      "datasource": "Prometheus",
      "fieldConfig": {
        "defaults": {
          "unit": "percentunit",
          "thresholds": {
            "steps": [
              { "color": "green", "value": null },
              { "color": "yellow", "value": 0.01 },
              { "color": "red", "value": 0.05 }
            ]
          }
        }
      },
      "gridPos": { "h": 4, "w": 6, "x": 12, "y": 0 },
      "id": 3,
      "title": "Error Rate",
      "type": "stat",
      "targets": [{
        "expr": "sum(rate(http_requests_total{app='chuaikan-api',status=~'5..'}[5m])) / sum(rate(http_requests_total{app='chuaikan-api'}[5m]))",
        "legendFormat": "Error Rate"
      }]
    },
    {
      "datasource": "Prometheus",
      "fieldConfig": {
        "defaults": { "unit": "short" }
      },
      "gridPos": { "h": 4, "w": 6, "x": 18, "y": 0 },
      "id": 4,
      "title": "Active Users",
      "type": "stat",
      "targets": [{
        "expr": "chuaikan_active_users_total",
        "legendFormat": "Active Users"
      }]
    },
    {
      "datasource": "Prometheus",
      "gridPos": { "h": 8, "w": 12, "x": 0, "y": 4 },
      "id": 5,
      "title": "HTTP Requests Rate",
      "type": "timeseries",
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{app='chuaikan-api',status=~'2..'}[5m])) by (status)",
          "legendFormat": "2xx"
        },
        {
          "expr": "sum(rate(http_requests_total{app='chuaikan-api',status=~'4..'}[5m])) by (status)",
          "legendFormat": "4xx"
        },
        {
          "expr": "sum(rate(http_requests_total{app='chuaikan-api',status=~'5..'}[5m])) by (status)",
          "legendFormat": "5xx"
        }
      ]
    }
  ],
  "refresh": "30s",
  "schemaVersion": 38,
  "tags": ["chuaikan", "production"],
  "time": { "from": "now-3h", "to": "now" },
  "title": "chuaikan.com — Overview",
  "uid": "chuaikan-overview-v1",
  "version": 1
}
```

### Step 87: Alert Rules

```bash
# สร้าง directory สำหรับ rules
sudo mkdir -p /etc/prometheus/rules

# สร้าง alert rules
sudo tee /etc/prometheus/rules/chuaikan-alerts.yml <<'EOF'
groups:
  - name: system.alerts
    interval: 30s
    rules:
    
      # CPU Usage > 80%
      - alert: HighCPUUsage
        expr: >
          100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
          team: infrastructure
        annotations:
          summary: "High CPU Usage on {{ $labels.instance }}"
          description: "CPU usage is {{ printf \"%.1f\" $value }}% on {{ $labels.instance }} for more than 5 minutes."
          runbook: "https://wiki.chuaikan.com/runbooks/high-cpu"

      # CPU Usage > 95% (Critical)
      - alert: CriticalCPUUsage
        expr: >
          100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 95
        for: 2m
        labels:
          severity: critical
          team: infrastructure
        annotations:
          summary: "CRITICAL: CPU Usage Extremely High on {{ $labels.instance }}"
          description: "CPU usage is {{ printf \"%.1f\" $value }}% — immediate action required!"

      # Memory Usage > 85%
      - alert: HighMemoryUsage
        expr: >
          (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High Memory Usage on {{ $labels.instance }}"
          description: "Memory usage is {{ printf \"%.1f\" $value }}%."

      # Disk Usage > 90%
      - alert: DiskSpaceRunningOut
        expr: >
          (1 - (node_filesystem_avail_bytes{fstype!~"tmpfs|fuse.lxcfs"} / 
                node_filesystem_size_bytes{fstype!~"tmpfs|fuse.lxcfs"})) * 100 > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Disk Space Critical on {{ $labels.instance }}"
          description: "Disk {{ $labels.mountpoint }} is {{ printf \"%.1f\" $value }}% full."

      # Service Down
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service Down: {{ $labels.job }} on {{ $labels.instance }}"
          description: "{{ $labels.job }} has been down for more than 1 minute."

  - name: application.alerts
    rules:
    
      # High Error Rate > 5%
      - alert: HighErrorRate
        expr: >
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (app) /
          sum(rate(http_requests_total[5m])) by (app) > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High Error Rate for {{ $labels.app }}"
          description: "Error rate is {{ printf \"%.1f\" (mul $value 100) }}%."

      # High P95 Latency > 1 second
      - alert: HighLatency
        expr: >
          histogram_quantile(0.95, 
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, app)
          ) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High P95 Latency for {{ $labels.app }}"
          description: "P95 latency is {{ printf \"%.2f\" $value }}s."

      # Website Down (Blackbox)
      - alert: WebsiteDown
        expr: probe_success{job="blackbox-http"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Website Down: {{ $labels.instance }}"
          description: "{{ $labels.instance }} is not responding."

  - name: database.alerts
    rules:
    
      # Too Many Connections
      - alert: PostgreSQLTooManyConnections
        expr: >
          pg_stat_activity_count > pg_settings_max_connections * 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL Too Many Connections"
          description: "{{ $value }} connections out of max."

      # Replication Lag (ถ้ามี replica)
      - alert: PostgreSQLReplicationLag
        expr: pg_replication_lag > 30
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL Replication Lag High"
          description: "Replication lag is {{ $value }} seconds."

      # Redis Memory > 80%
      - alert: RedisMemoryHigh
        expr: >
          redis_memory_used_bytes / redis_memory_max_bytes > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis Memory Usage High"
          description: "Redis is using {{ printf \"%.1f\" (mul $value 100) }}% of max memory."
EOF

# Validate rules
promtool check rules /etc/prometheus/rules/chuaikan-alerts.yml

# Reload Prometheus
sudo systemctl reload prometheus
```

### Step 88: Alertmanager Setup

```bash
# ดาวน์โหลด Alertmanager
AM_VERSION="0.27.0"
wget -q https://github.com/prometheus/alertmanager/releases/download/v${AM_VERSION}/alertmanager-${AM_VERSION}.linux-amd64.tar.gz

tar xzf alertmanager-${AM_VERSION}.linux-amd64.tar.gz
sudo cp alertmanager-${AM_VERSION}.linux-amd64/alertmanager \
        alertmanager-${AM_VERSION}.linux-amd64/amtool \
        /usr/local/bin/

sudo useradd --no-create-home --shell /bin/false alertmanager
sudo mkdir -p /etc/alertmanager /var/lib/alertmanager
sudo chown alertmanager:alertmanager /var/lib/alertmanager

# สร้าง config
sudo tee /etc/alertmanager/alertmanager.yml <<'AMEOF'
global:
  resolve_timeout: 5m

# Templates
templates:
  - '/etc/alertmanager/templates/*.tmpl'

# Routing
route:
  receiver: 'discord-default'
  group_by: ['alertname', 'severity']
  group_wait: 10s
  group_interval: 5m
  repeat_interval: 12h
  
  routes:
    # Critical alerts: immediate notification
    - receiver: 'discord-critical'
      match:
        severity: critical
      group_wait: 0s
      repeat_interval: 1h
      
    # Warning alerts: standard timing
    - receiver: 'discord-warning'
      match:
        severity: warning
      group_wait: 1m
      repeat_interval: 4h

# Receivers
receivers:
  - name: 'discord-default'
    webhook_configs:
      - url: 'https://discord.com/api/webhooks/YOUR_WEBHOOK_ID/YOUR_WEBHOOK_TOKEN'
        send_resolved: true
        http_config:
          follow_redirects: true
        title: '{{ .GroupLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          **{{ .Annotations.summary }}**
          {{ .Annotations.description }}
          Status: {{ .Status }}
          {{ end }}

  - name: 'discord-critical'
    webhook_configs:
      - url: 'https://discord.com/api/webhooks/YOUR_WEBHOOK_ID/YOUR_WEBHOOK_TOKEN'
        send_resolved: true
        title: '🚨 CRITICAL ALERT'

  - name: 'discord-warning'
    webhook_configs:
      - url: 'https://discord.com/api/webhooks/YOUR_WEBHOOK_ID/YOUR_WEBHOOK_TOKEN'
        send_resolved: true

# Inhibit rules (ไม่ส่ง warning ถ้ามี critical อยู่แล้ว)
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
AMEOF

# Validate config
amtool check-config /etc/alertmanager/alertmanager.yml

# สร้าง systemd service
sudo tee /etc/systemd/system/alertmanager.service <<'EOF'
[Unit]
Description=Prometheus Alertmanager
After=network-online.target

[Service]
Type=simple
User=alertmanager
Group=alertmanager
ExecStart=/usr/local/bin/alertmanager \
    --config.file=/etc/alertmanager/alertmanager.yml \
    --storage.path=/var/lib/alertmanager \
    --web.listen-address=127.0.0.1:9093
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable alertmanager
sudo systemctl start alertmanager
```

### Step 89: PostgreSQL และ Redis Exporters

```bash
# ติดตั้ง postgres_exporter
PG_EXP_VERSION="0.15.0"
wget -q https://github.com/prometheus-community/postgres_exporter/releases/download/v${PG_EXP_VERSION}/postgres_exporter-${PG_EXP_VERSION}.linux-amd64.tar.gz

tar xzf postgres_exporter-${PG_EXP_VERSION}.linux-amd64.tar.gz
sudo cp postgres_exporter-${PG_EXP_VERSION}.linux-amd64/postgres_exporter /usr/local/bin/

# สร้าง monitoring user ใน PostgreSQL
sudo -u postgres psql <<'SQL'
CREATE USER prometheus WITH PASSWORD 'prometheus_monitoring_pass';
GRANT pg_monitor TO prometheus;
SQL

# ตั้งค่า connection string
sudo tee /etc/default/postgres_exporter <<'EOF'
DATA_SOURCE_NAME="postgresql://prometheus:prometheus_monitoring_pass@localhost:5432/postgres?sslmode=disable"
EOF

# สร้าง systemd service
sudo tee /etc/systemd/system/postgres_exporter.service <<'EOF'
[Unit]
Description=PostgreSQL Prometheus Exporter
After=network-online.target postgresql.service

[Service]
Type=simple
User=postgres_exporter
Group=postgres_exporter
EnvironmentFile=/etc/default/postgres_exporter
ExecStart=/usr/local/bin/postgres_exporter \
    --web.listen-address=127.0.0.1:9187 \
    --log.level=info
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
EOF

sudo useradd --no-create-home --shell /bin/false postgres_exporter
sudo systemctl daemon-reload
sudo systemctl enable postgres_exporter
sudo systemctl start postgres_exporter

# ตรวจสอบ
curl -s http://localhost:9187/metrics | grep pg_up
# Expected: pg_up 1

# ติดตั้ง redis_exporter
REDIS_EXP_VERSION="1.58.0"
wget -q https://github.com/oliver006/redis_exporter/releases/download/v${REDIS_EXP_VERSION}/redis_exporter-v${REDIS_EXP_VERSION}.linux-amd64.tar.gz

tar xzf redis_exporter-v${REDIS_EXP_VERSION}.linux-amd64.tar.gz
sudo cp redis_exporter-v${REDIS_EXP_VERSION}.linux-amd64/redis_exporter /usr/local/bin/

sudo useradd --no-create-home --shell /bin/false redis_exporter

sudo tee /etc/systemd/system/redis_exporter.service <<'EOF'
[Unit]
Description=Redis Prometheus Exporter
After=network-online.target redis-server.service

[Service]
Type=simple
User=redis_exporter
Group=redis_exporter
Environment=REDIS_ADDR=redis://localhost:6379
Environment=REDIS_PASSWORD=your_redis_password
ExecStart=/usr/local/bin/redis_exporter \
    --web.listen-address=127.0.0.1:9121
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable redis_exporter
sudo systemctl start redis_exporter
```

### Step 90: Node.js Metrics ด้วย prom-client

```bash
npm install prom-client
```

```typescript
// apps/api/src/metrics.ts
import { collectDefaultMetrics, Registry, Counter, Histogram, Gauge } from 'prom-client';

// สร้าง Registry
export const registry = new Registry();

// Collect default Node.js metrics (heap, GC, event loop, etc.)
collectDefaultMetrics({
  register: registry,
  prefix: 'chuaikan_',
});

// HTTP Requests Counter
export const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status', 'app'],
  registers: [registry],
});

// HTTP Request Duration Histogram
export const httpRequestDurationSeconds = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status', 'app'],
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
  registers: [registry],
});

// Active users gauge
export const activeUsersTotal = new Gauge({
  name: 'chuaikan_active_users_total',
  help: 'Number of currently active users',
  registers: [registry],
});

// Database query duration
export const dbQueryDurationSeconds = new Histogram({
  name: 'chuaikan_db_query_duration_seconds',
  help: 'Duration of database queries in seconds',
  labelNames: ['operation', 'table'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1],
  registers: [registry],
});

// Middleware สำหรับ Express/Fastify
export function metricsMiddleware() {
  return async (req: any, res: any, next: any) => {
    const start = Date.now();
    const route = req.route?.path || req.path || 'unknown';
    
    res.on('finish', () => {
      const duration = (Date.now() - start) / 1000;
      const labels = {
        method: req.method,
        route: route,
        status: res.statusCode.toString(),
        app: 'chuaikan-api',
      };
      
      httpRequestsTotal.inc(labels);
      httpRequestDurationSeconds.observe(labels, duration);
    });
    
    next();
  };
}
```

```typescript
// apps/api/src/routes/metrics.ts
import { Router } from 'express';
import { registry } from '../metrics';

const router = Router();

// /metrics endpoint สำหรับ Prometheus scrape
router.get('/metrics', async (req, res) => {
  // ป้องกัน unauthorized access
  const authHeader = req.headers.authorization;
  const metricsSecret = process.env.METRICS_SECRET;
  
  if (metricsSecret && authHeader !== `Bearer ${metricsSecret}`) {
    return res.status(403).json({ error: 'Forbidden' });
  }
  
  try {
    res.set('Content-Type', registry.contentType);
    res.end(await registry.metrics());
  } catch (err) {
    res.status(500).end(err);
  }
});

export default router;
```

---

## 🔧 Configuration Files

### Blackbox Exporter Config

```bash
# ดาวน์โหลด Blackbox Exporter
BB_VERSION="0.24.0"
wget -q https://github.com/prometheus/blackbox_exporter/releases/download/v${BB_VERSION}/blackbox_exporter-${BB_VERSION}.linux-amd64.tar.gz

tar xzf blackbox_exporter-${BB_VERSION}.linux-amd64.tar.gz
sudo cp blackbox_exporter-${BB_VERSION}.linux-amd64/blackbox_exporter /usr/local/bin/

sudo mkdir /etc/blackbox_exporter

sudo tee /etc/blackbox_exporter/config.yml <<'EOF'
modules:
  http_2xx:
    prober: http
    timeout: 10s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: [200, 201]
      method: GET
      follow_redirects: true
      fail_if_ssl: false
      fail_if_not_ssl: true
      tls_config:
        insecure_skip_verify: false

  http_post_2xx:
    prober: http
    http:
      method: POST
      headers:
        Content-Type: application/json
EOF
```

---

## 🧪 Testing

```bash
# Test 1: ตรวจสอบ services ทั้งหมด
for service in prometheus node_exporter grafana-server alertmanager postgres_exporter redis_exporter; do
    STATUS=$(systemctl is-active $service 2>/dev/null || echo "not-found")
    echo "$service: $STATUS"
done

# Test 2: ตรวจสอบ Prometheus targets
curl -s http://localhost:9090/api/v1/targets | \
    jq '.data.activeTargets[] | {job: .labels.job, health: .health, instance: .labels.instance}'

# Test 3: query metrics
curl -s 'http://localhost:9090/api/v1/query?query=up' | \
    jq '.data.result[] | {job: .metric.job, value: .value[1]}'

# Test 4: ตรวจสอบ alert rules
curl -s http://localhost:9090/api/v1/rules | \
    jq '.data.groups[].rules[] | {name: .name, state: .state}'

# Test 5: ทดสอบ prom-client metrics
curl http://localhost:4000/metrics | grep chuaikan_

# Test 6: ตรวจสอบ Grafana datasource
curl -s http://admin:admin@localhost:3001/api/datasources | jq '.[].name'

# Test 7: ส่ง test alert
curl -s -X POST \
    http://localhost:9093/api/v1/alerts \
    -H "Content-Type: application/json" \
    --data '[{
        "labels": {"alertname": "TestAlert", "severity": "warning"},
        "annotations": {"summary": "Test alert from chuaikan.com"},
        "startsAt": "2024-12-01T00:00:00Z",
        "endsAt": "2024-12-01T01:00:00Z"
    }]'
```

---

## ❌ Common Errors & Solutions

### Error 1: Prometheus Target Down

```
Error: Get "http://localhost:9100/metrics": connection refused
```

**Solution:**
```bash
# ตรวจสอบว่า exporter รันอยู่
sudo systemctl status node_exporter

# ตรวจสอบ port
sudo ss -tlnp | grep 9100

# ตรวจสอบ firewall
sudo ufw status | grep 9100
```

### Error 2: Grafana Can't Connect to Prometheus

```
Error: Get http://localhost:9090/api/v1/query: dial tcp: connection refused
```

**Solution:**
```bash
# ตรวจสอบ prometheus ทำงานอยู่
sudo systemctl status prometheus
curl http://localhost:9090/-/healthy

# ตรวจสอบ Grafana datasource URL
# ถ้า Grafana อยู่ใน Docker แต่ Prometheus อยู่ใน host
# ใช้ host.docker.internal แทน localhost
```

### Error 3: Alertmanager ไม่ส่ง Discord

```
# Alerts ขึ้นใน Prometheus แต่ไม่มี Discord notification
```

**Solution:**
```bash
# ดู Alertmanager logs
sudo journalctl -u alertmanager -n 50

# ทดสอบ webhook URL
curl -X POST YOUR_DISCORD_WEBHOOK_URL \
    -H "Content-Type: application/json" \
    --data '{"content": "Test from Alertmanager"}'

# ตรวจสอบ Alertmanager status
curl http://localhost:9093/api/v1/status

# ดู alerts ที่กำลัง fire
curl http://localhost:9093/api/v1/alerts
```

### Error 4: High Cardinality Metrics (Prometheus OOM)

```
# Prometheus ใช้ memory มาก
```

**Solution:**
```typescript
// ❌ อย่าใช้ user ID หรือ dynamic values เป็น labels
httpRequestsTotal.inc({ user_id: userId });  // cardinality สูงมาก!

// ✅ ใช้แค่ static values
httpRequestsTotal.inc({ method: 'GET', route: '/api/users', status: '200' });
```

---

## ✅ Checklist

### Installation
- [ ] Prometheus ติดตั้งสำเร็จและ running
- [ ] Node Exporter ติดตั้งและ running
- [ ] Grafana ติดตั้งและ accessible ที่ port 3001
- [ ] Alertmanager ติดตั้งและ running
- [ ] postgres_exporter ติดตั้งและ running
- [ ] redis_exporter ติดตั้งและ running
- [ ] blackbox_exporter ติดตั้งและ running

### Configuration
- [ ] prometheus.yml ตั้งค่า scrape configs ครบ
- [ ] Alert rules ตั้งค่าแล้ว (CPU, memory, disk, service down)
- [ ] Alertmanager ตั้งค่า Discord webhook แล้ว

### Grafana
- [ ] Prometheus datasource ตั้งค่าแล้ว
- [ ] Node Exporter Full dashboard (1860) import แล้ว
- [ ] PostgreSQL dashboard import แล้ว
- [ ] Custom chuaikan.com dashboard สร้างแล้ว
- [ ] Admin password เปลี่ยนแล้ว

### Application Metrics
- [ ] prom-client ติดตั้งใน Node.js API
- [ ] /metrics endpoint ทำงาน
- [ ] HTTP requests metrics ถูก collect
- [ ] Latency histogram ถูก collect

### Testing
- [ ] ทุก Prometheus targets สถานะ "up"
- [ ] Test alert ส่ง Discord notification สำเร็จ
- [ ] Dashboard แสดง metrics จริงจาก application

---

## 🔗 References

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/grafana/latest/)
- [Node Exporter](https://github.com/prometheus/node_exporter)
- [postgres_exporter](https://github.com/prometheus-community/postgres_exporter)
- [redis_exporter](https://github.com/oliver006/redis_exporter)
- [prom-client for Node.js](https://github.com/siimon/prom-client)
- [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/)
- [Grafana Dashboard Library](https://grafana.com/grafana/dashboards/)

---
*Part 009 | Road to 1,000,000 Users/Day | chuaikan.com*
