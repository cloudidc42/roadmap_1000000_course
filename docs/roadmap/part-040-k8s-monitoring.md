# Part 040: K8s Monitoring ด้วย Prometheus Stack

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 391-400
> **เวลาโดยประมาณ:** 4 ชั่วโมง
> **Prerequisites:** Part 039 (Rolling Updates), Helm ติดตั้งแล้ว, cluster พร้อมใช้งาน

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ติดตั้ง kube-prometheus-stack ด้วย Helm
- ตั้งค่า Prometheus scraping K8s metrics
- สร้าง Grafana dashboards สำหรับ chuaikan.com
- ตั้งค่า Alert rules สำหรับ critical scenarios
- ใช้ PodMonitor และ ServiceMonitor CRDs

---

## 📖 ทฤษฎีและแนวคิด

### Step 391 — kube-prometheus-stack คืออะไร

**kube-prometheus-stack** รวม tools สำคัญไว้ใน chart เดียว:

```
kube-prometheus-stack ประกอบด้วย:
1. Prometheus          - time-series database + query engine
2. Grafana            - visualization + dashboards
3. Alertmanager       - routing alerts → Slack, PagerDuty, Email
4. Node Exporter      - system metrics (CPU, Memory, Disk)
5. kube-state-metrics - K8s object metrics (pod status, replica count)
6. Prometheus Operator - จัดการ Prometheus ด้วย K8s CRDs
```

### Step 392 — Metrics Flow

```
chuaikan.com Services
    │ expose /metrics (Prometheus format)
    ▼
ServiceMonitor / PodMonitor (CRD)
    │ tells Prometheus what to scrape
    ▼
Prometheus Server
    │ store metrics
    ▼
Grafana Dashboard ◄─────────── Query (PromQL)
    │
    └─ Alert Rules ──► Alertmanager ──► Slack / PagerDuty
```

---

## ⚙️ Environment Setup

```bash
# สร้าง namespace
kubectl create namespace monitoring

# เพิ่ม Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# ดู chart values
helm show values prometheus-community/kube-prometheus-stack > prometheus-default-values.yaml
```

---

## 🛠️ Step-by-Step Implementation

### Step 393 — ติดตั้ง kube-prometheus-stack

```yaml
# prometheus-stack-values.yaml
prometheus:
  prometheusSpec:
    replicas: 1
    retention: 15d                    # เก็บ metrics 15 วัน
    retentionSize: "50GB"
    resources:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        cpu: 2000m
        memory: 8Gi
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: local-path
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 100Gi
    # Scrape configs เพิ่มเติม
    additionalScrapeConfigs:
    - job_name: 'chuaikan-nextjs'
      static_configs:
      - targets: ['nextjs-service.chuaikan-production.svc.cluster.local:3000']
      metrics_path: /api/metrics

alertmanager:
  alertmanagerSpec:
    replicas: 1
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: local-path
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi
  config:
    global:
      slack_api_url: 'https://hooks.slack.com/services/xxx/yyy/zzz'
      resolve_timeout: 5m
    route:
      group_by: ['alertname', 'namespace']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: 'slack-critical'
      routes:
      - match:
          severity: critical
        receiver: 'slack-critical'
      - match:
          severity: warning
        receiver: 'slack-warning'
    receivers:
    - name: 'slack-critical'
      slack_configs:
      - channel: '#chuaikan-alerts-critical'
        title: '🚨 [{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Labels.alertname }}
          *Severity:* {{ .Labels.severity }}
          *Namespace:* {{ .Labels.namespace }}
          *Description:* {{ .Annotations.description }}
          *Runbook:* {{ .Annotations.runbook_url }}
          {{ end }}
    - name: 'slack-warning'
      slack_configs:
      - channel: '#chuaikan-alerts-warning'
        title: '⚠️ [{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}'

grafana:
  enabled: true
  adminPassword: "Gr@fan@Adm1n!"
  replicas: 1
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi
  persistence:
    enabled: true
    storageClassName: local-path
    size: 10Gi
  ingress:
    enabled: true
    ingressClassName: nginx
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
    hosts:
    - grafana.chuaikan.com
    tls:
    - secretName: grafana-tls
      hosts:
      - grafana.chuaikan.com

nodeExporter:
  enabled: true

kubeStateMetrics:
  enabled: true

prometheusOperator:
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
```

```bash
# ติดตั้ง kube-prometheus-stack
helm install prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values prometheus-stack-values.yaml \
  --version 57.2.0

# รอ pods พร้อม
kubectl rollout status deployment/prometheus-stack-grafana -n monitoring
kubectl rollout status statefulset/prometheus-prometheus-stack-kube-prom-prometheus -n monitoring

# ตรวจสอบ
kubectl get pods -n monitoring
kubectl get services -n monitoring
```

### Step 394 — ServiceMonitor และ PodMonitor

```yaml
# servicemonitor-nextjs.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nextjs-monitor
  namespace: chuaikan-production
  labels:
    app: nextjs
    release: prometheus-stack    # ต้องตรงกับ Helm release name
spec:
  selector:
    matchLabels:
      app: nextjs
  endpoints:
  - port: http
    path: /api/metrics
    interval: 30s              # scrape ทุก 30 วิ
    scrapeTimeout: 10s
  namespaceSelector:
    matchNames:
    - chuaikan-production
```

```yaml
# servicemonitor-api.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nodejs-api-monitor
  namespace: chuaikan-production
  labels:
    release: prometheus-stack
spec:
  selector:
    matchLabels:
      app: nodejs-api
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
  namespaceSelector:
    matchNames:
    - chuaikan-production
```

```yaml
# podmonitor-workers.yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: workers-monitor
  namespace: chuaikan-production
  labels:
    release: prometheus-stack
spec:
  selector:
    matchLabels:
      app: email-processor
  podMetricsEndpoints:
  - port: metrics
    path: /metrics
    interval: 30s
  namespaceSelector:
    matchNames:
    - chuaikan-production
```

```bash
kubectl apply -f servicemonitor-nextjs.yaml
kubectl apply -f servicemonitor-api.yaml
kubectl apply -f podmonitor-workers.yaml

# ตรวจสอบใน Prometheus
kubectl port-forward svc/prometheus-stack-kube-prom-prometheus \
  9090:9090 \
  -n monitoring

# เปิด http://localhost:9090
# ไปที่ Status > Targets เพื่อดู ServiceMonitors
```

### Step 395 — Prometheus Metrics ใน Next.js App

```javascript
// app/api/metrics/route.ts (Next.js App Router)
import { NextResponse } from 'next/server';
import { Registry, collectDefaultMetrics, Counter, Histogram, Gauge } from 'prom-client';

const register = new Registry();
collectDefaultMetrics({ register });

// Custom metrics
const httpRequestCounter = new Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'status', 'path'],
  registers: [register],
});

const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'path'],
  buckets: [0.1, 0.3, 0.5, 0.7, 1, 3, 5, 7, 10],
  registers: [register],
});

const activeUsers = new Gauge({
  name: 'active_users',
  help: 'Number of active users',
  registers: [register],
});

export async function GET() {
  const metrics = await register.metrics();
  return new NextResponse(metrics, {
    headers: {
      'Content-Type': register.contentType,
    },
  });
}
```

```bash
# package.json dependencies
# "prom-client": "^15.1.0"

# ตรวจสอบ metrics endpoint
curl http://localhost:3000/api/metrics
```

### Step 396 — Alert Rules สำหรับ chuaikan.com

```yaml
# alertrules-chuaikan.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: chuaikan-alerts
  namespace: monitoring
  labels:
    release: prometheus-stack
spec:
  groups:
  - name: pod.rules
    interval: 30s
    rules:
    # Pod Crash Loop
    - alert: PodCrashLooping
      expr: |
        rate(kube_pod_container_status_restarts_total{
          namespace=~"chuaikan-.*"
        }[15m]) * 60 * 15 > 5
      for: 5m
      labels:
        severity: critical
        namespace: "{{ $labels.namespace }}"
      annotations:
        summary: "Pod {{ $labels.pod }} is crash looping"
        description: |
          Pod {{ $labels.pod }} in namespace {{ $labels.namespace }}
          has restarted {{ $value }} times in the last 15 minutes.
        runbook_url: https://docs.chuaikan.com/runbooks/pod-crash-loop

    # Pod Not Ready
    - alert: PodNotReady
      expr: |
        kube_pod_status_ready{
          condition="true",
          namespace=~"chuaikan-.*"
        } == 0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} not ready"
        description: "Pod {{ $labels.pod }} has been not ready for 5 minutes"

  - name: resource.rules
    rules:
    # High CPU Usage
    - alert: HighCPUUsage
      expr: |
        (
          sum(rate(container_cpu_usage_seconds_total{
            namespace=~"chuaikan-.*",
            container!=""
          }[5m])) by (pod, namespace)
          /
          sum(kube_pod_container_resource_limits{
            resource="cpu",
            namespace=~"chuaikan-.*"
          }) by (pod, namespace)
        ) > 0.85
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "High CPU usage in pod {{ $labels.pod }}"
        description: |
          Pod {{ $labels.pod }} is using {{ $value | humanizePercentage }} of its CPU limit.

    # Memory Pressure
    - alert: HighMemoryUsage
      expr: |
        (
          container_memory_working_set_bytes{
            namespace=~"chuaikan-.*",
            container!=""
          }
          /
          kube_pod_container_resource_limits{
            resource="memory",
            namespace=~"chuaikan-.*"
          }
        ) > 0.90
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High memory usage in pod {{ $labels.pod }}"
        description: |
          Pod {{ $labels.pod }} is using {{ $value | humanizePercentage }} of its memory limit.
          Risk of OOMKill!

    # OOMKill
    - alert: OOMKilled
      expr: |
        kube_pod_container_status_last_terminated_reason{
          reason="OOMKilled",
          namespace=~"chuaikan-.*"
        } == 1
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }} was OOMKilled"
        description: |
          Container {{ $labels.container }} in pod {{ $labels.pod }}
          was killed due to out-of-memory condition.

  - name: availability.rules
    rules:
    # Deployment Replicas
    - alert: DeploymentReplicasMismatch
      expr: |
        kube_deployment_spec_replicas{namespace=~"chuaikan-.*"}
        !=
        kube_deployment_status_available_replicas{namespace=~"chuaikan-.*"}
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Deployment {{ $labels.deployment }} replicas mismatch"
        description: |
          Deployment {{ $labels.deployment }} in {{ $labels.namespace }}
          has {{ $value }} available replicas but desires more.

    # Node Disk Pressure
    - alert: NodeDiskPressure
      expr: |
        kube_node_status_condition{
          condition="DiskPressure",
          status="true"
        } == 1
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "Node {{ $labels.node }} has disk pressure"
        description: "Node {{ $labels.node }} is running low on disk space."

    # High Error Rate
    - alert: HighErrorRate
      expr: |
        rate(http_requests_total{
          status=~"5..",
          namespace=~"chuaikan-.*"
        }[5m])
        /
        rate(http_requests_total{
          namespace=~"chuaikan-.*"
        }[5m]) > 0.05
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "High error rate in {{ $labels.namespace }}"
        description: |
          Error rate is {{ $value | humanizePercentage }} (threshold: 5%)

    # High Response Time
    - alert: HighResponseTime
      expr: |
        histogram_quantile(0.95,
          rate(http_request_duration_seconds_bucket{
            namespace=~"chuaikan-.*"
          }[5m])
        ) > 2
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High 95th percentile response time"
        description: |
          95th percentile response time is {{ $value }}s (threshold: 2s)
```

```bash
kubectl apply -f alertrules-chuaikan.yaml

# ตรวจสอบ alert rules ใน Prometheus
# เปิด http://localhost:9090 → Alerts
```

### Step 397 — Grafana Dashboard Setup

```bash
# Login Grafana ผ่าน port-forward
kubectl port-forward svc/prometheus-stack-grafana \
  3000:80 \
  -n monitoring

# เปิด http://localhost:3000
# username: admin
# password: (จาก prometheus-stack-values.yaml)

# Import dashboards ที่เป็นที่นิยม
# Dashboard IDs สำหรับ import:
# 15760 - Kubernetes Cluster Overview
# 12206 - K8s Cluster Summary
# 13770 - 1 Node Exporter for Prometheus Dashboard
# 7249  - Kubernetes Pods (Namespace)
# 14517 - Node Exporter Full
```

```yaml
# grafana-dashboard-configmap.yaml
# สร้าง dashboard ด้วย ConfigMap (GitOps friendly)
apiVersion: v1
kind: ConfigMap
metadata:
  name: chuaikan-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"       # Grafana sidecar จะ import อัตโนมัติ
data:
  chuaikan-overview.json: |
    {
      "title": "Chuaikan.com Overview",
      "uid": "chuaikan-overview",
      "panels": [
        {
          "title": "HTTP Request Rate",
          "type": "stat",
          "gridPos": {"x": 0, "y": 0, "w": 4, "h": 3},
          "targets": [{
            "expr": "sum(rate(http_requests_total{namespace=\"chuaikan-production\"}[5m]))",
            "legendFormat": "RPS"
          }]
        },
        {
          "title": "Error Rate",
          "type": "stat",
          "gridPos": {"x": 4, "y": 0, "w": 4, "h": 3},
          "targets": [{
            "expr": "sum(rate(http_requests_total{namespace=\"chuaikan-production\",status=~\"5..\"}[5m])) / sum(rate(http_requests_total{namespace=\"chuaikan-production\"}[5m])) * 100",
            "legendFormat": "Error %"
          }]
        },
        {
          "title": "Active Pods",
          "type": "stat",
          "gridPos": {"x": 8, "y": 0, "w": 4, "h": 3},
          "targets": [{
            "expr": "count(kube_pod_status_ready{namespace=\"chuaikan-production\",condition=\"true\"} == 1)",
            "legendFormat": "Pods"
          }]
        }
      ]
    }
```

```bash
kubectl apply -f grafana-dashboard-configmap.yaml
# Grafana จะ auto-import dashboard

# Port-forward Grafana
kubectl port-forward svc/prometheus-stack-grafana 3000:80 -n monitoring
```

### Step 398 — PromQL Queries ที่ใช้บ่อย

```bash
# เปิด Prometheus UI
kubectl port-forward svc/prometheus-stack-kube-prom-prometheus \
  9090:9090 \
  -n monitoring

# --- Performance Queries ---

# HTTP RPS (requests per second) ทั้ง cluster
sum(rate(http_requests_total{namespace="chuaikan-production"}[5m]))

# 95th percentile response time
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket{namespace="chuaikan-production"}[5m]))
  by (le)
)

# Error rate percentage
sum(rate(http_requests_total{namespace="chuaikan-production",status=~"5.."}[5m]))
/ sum(rate(http_requests_total{namespace="chuaikan-production"}[5m])) * 100

# --- Resource Queries ---

# CPU usage per pod
sum(rate(container_cpu_usage_seconds_total{namespace="chuaikan-production",container!=""}[5m]))
by (pod)

# Memory usage per pod
container_memory_working_set_bytes{namespace="chuaikan-production",container!=""}

# CPU limit utilization %
sum(rate(container_cpu_usage_seconds_total{namespace="chuaikan-production"}[5m]))
by (pod)
/ sum(kube_pod_container_resource_limits{resource="cpu",namespace="chuaikan-production"})
by (pod) * 100

# --- K8s Queries ---

# Pod restart count (last 1 hour)
increase(kube_pod_container_status_restarts_total{namespace="chuaikan-production"}[1h])

# Number of running pods per deployment
kube_deployment_status_available_replicas{namespace="chuaikan-production"}

# Node disk usage %
100 - (node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes{fstype!="tmpfs"} * 100)
```

### Step 399 — Grafana Dashboards ที่สำคัญ

```bash
# Dashboard 1: K8s Cluster Overview
# Import ID: 15760

# Dashboard 2: Namespace Resource Usage
# สร้าง panels:
# - CPU request/limit/usage per pod
# - Memory request/limit/usage per pod
# - Pod count by status
# - HPA replicas (current vs desired)

# Dashboard 3: chuaikan.com Application
# สร้าง panels:
# - HTTP RPS
# - P50, P95, P99 response times
# - Error rate %
# - Active users (gauge)
# - Top 10 slowest endpoints

# Dashboard 4: Alerts Overview
# - Active alerts list
# - Alert history timeline
# - Alert count by severity
```

### Step 400 — Production Monitoring Checklist

```bash
# ตรวจสอบ monitoring stack ทั้งหมด
echo "=== Checking Monitoring Stack ==="

# Prometheus health
kubectl exec -n monitoring \
  statefulset/prometheus-prometheus-stack-kube-prom-prometheus -- \
  wget -qO- http://localhost:9090/-/healthy
echo "Prometheus: OK"

# Grafana health
kubectl exec -n monitoring \
  deployment/prometheus-stack-grafana -- \
  wget -qO- http://localhost:3000/api/health | python3 -m json.tool
echo "Grafana: OK"

# Alertmanager health
kubectl exec -n monitoring \
  statefulset/alertmanager-prometheus-stack-kube-prom-alertmanager -- \
  wget -qO- http://localhost:9093/-/healthy
echo "Alertmanager: OK"

# ตรวจสอบ targets ทั้งหมด healthy
kubectl exec -n monitoring \
  statefulset/prometheus-prometheus-stack-kube-prom-prometheus -- \
  wget -qO- http://localhost:9090/api/v1/targets | \
  python3 -c "
import json, sys
data = json.load(sys.stdin)
active = data['data']['activeTargets']
unhealthy = [t for t in active if t['health'] != 'up']
print(f'Total targets: {len(active)}, Unhealthy: {len(unhealthy)}')
for t in unhealthy:
    print(f'  UNHEALTHY: {t[\"labels\"][\"job\"]}')
"

# ส่ง test alert
kubectl exec -n monitoring \
  statefulset/alertmanager-prometheus-stack-kube-prom-alertmanager -- \
  wget -qO- --post-data='[{"labels":{"alertname":"TestAlert","severity":"warning","namespace":"chuaikan-production"},"annotations":{"summary":"Test alert from monitoring setup"}}]' \
  http://localhost:9093/api/v1/alerts

# ตรวจสอบ Slack ว่าได้รับ alert
echo "Check Slack channel #chuaikan-alerts-warning for TestAlert"
```

---

## 🔧 Configuration Files

### Prometheus Scrape Config สำหรับ Nginx Ingress

```yaml
# ServiceMonitor สำหรับ Nginx Ingress metrics
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nginx-ingress-monitor
  namespace: monitoring
  labels:
    release: prometheus-stack
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
  namespaceSelector:
    matchNames:
    - ingress-nginx
```

---

## ❌ Common Errors & Solutions

### Error: `Prometheus targets showing DOWN`
```bash
# ตรวจสอบ ServiceMonitor ถูก pick up
kubectl get servicemonitor -n chuaikan-production
kubectl describe servicemonitor nextjs-monitor -n chuaikan-production

# ตรวจสอบ label ตรงกับ Prometheus selector
kubectl get prometheus -n monitoring -o yaml | grep serviceMonitorSelector -A 5
```

### Error: `Grafana dashboards ว่างเปล่า`
```bash
# ตรวจสอบ data source
# Grafana → Configuration → Data Sources → Prometheus
# URL ควรเป็น: http://prometheus-stack-kube-prom-prometheus.monitoring.svc.cluster.local:9090

# ตรวจสอบ permissions
kubectl get clusterrolebinding | grep prometheus
```

### Error: `AlertManager ไม่ส่ง Slack`
```bash
# ตรวจสอบ config
kubectl get secret alertmanager-prometheus-stack-kube-prom-alertmanager \
  -n monitoring -o jsonpath='{.data.alertmanager\.yaml}' | base64 -d

# ตรวจสอบ logs
kubectl logs -n monitoring \
  -l app.kubernetes.io/name=alertmanager \
  --tail=50
```

---

## ✅ Checklist

- [ ] **Step 391**: เข้าใจ kube-prometheus-stack components และ metrics flow
- [ ] **Step 392**: อธิบาย Prometheus metrics flow ได้: service → ServiceMonitor → Prometheus → Grafana → Alert
- [ ] **Step 393**: ติดตั้ง kube-prometheus-stack สำเร็จ พร้อม persistent storage
- [ ] **Step 394**: สร้าง ServiceMonitor สำหรับ Next.js และ API สำเร็จ Prometheus เห็น targets
- [ ] **Step 395**: เพิ่ม /api/metrics endpoint ใน Next.js app ด้วย prom-client
- [ ] **Step 396**: สร้าง PrometheusRule: pod crash loop, high CPU, OOMKill, high error rate
- [ ] **Step 397**: Import Grafana dashboards และสร้าง chuaikan.com overview dashboard
- [ ] **Step 398**: เขียน PromQL queries สำหรับ RPS, response time, error rate, resource usage
- [ ] **Step 399**: ตั้งค่า dashboards: K8s cluster, namespace, application, alerts
- [ ] **Step 400**: ทดสอบ monitoring stack ทั้งหมด ส่ง test alert ไป Slack สำเร็จ

---

## 🔗 References

- [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Prometheus Operator](https://prometheus-operator.dev/)
- [PromQL Tutorial](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [Grafana Dashboards](https://grafana.com/grafana/dashboards/)
- [Alertmanager Configuration](https://prometheus.io/docs/alerting/latest/configuration/)

---

*Part 040 | Road to 1,000,000 Users/Day | chuaikan.com*
