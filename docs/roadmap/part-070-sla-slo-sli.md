# Part 070: SLA/SLO/SLI Definition

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Advanced
> **Steps:** 691-700
> **เวลาโดยประมาณ:** 4 ชั่วโมง
> **Prerequisites:** Part 061-069

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ความแตกต่างระหว่าง SLA, SLO, SLI และ Error Budget
- กำหนด SLA สำหรับ chuaikan.com (99.9% uptime)
- กำหนด SLO สำหรับ availability, latency, throughput, error rate
- Implement SLI measurement
- คำนวณ Error Budget และ Burn Rate
- สร้าง SLO Dashboard ใน Grafana
- ตั้งค่า Alert สำหรับ Error Budget burn rate
- กำหนด SLO review cadence

---

## 📖 ทฤษฎีและแนวคิด

### SLA vs SLO vs SLI

```
┌──────────────────────────────────────────────────────────────┐
│                    Reliability Hierarchy                      │
├──────────────┬───────────────────────────────────────────────┤
│ SLI          │ Service Level Indicator                        │
│              │ → การวัดค่าจริงๆ (metric)                      │
│              │ → "99.95% ของ requests ตอบภายใน 200ms"         │
├──────────────┼───────────────────────────────────────────────┤
│ SLO          │ Service Level Objective                        │
│              │ → เป้าหมายภายใน (ไม่ต้องบอก users)              │
│              │ → "เราจะ achieve 99.9% availability"           │
├──────────────┼───────────────────────────────────────────────┤
│ SLA          │ Service Level Agreement                        │
│              │ → สัญญากับ users/customers (เข้มกว่า SLO)       │
│              │ → "เรา guarantee 99.5% uptime, มิฉะนั้น refund" │
├──────────────┼───────────────────────────────────────────────┤
│ Error Budget │ = 100% - SLO target                           │
│              │ → "งบประมาณ" ที่เราใช้ได้เพื่อ innovation       │
│              │ → 99.9% SLO = 0.1% Error Budget = 8.7h/yr    │
└──────────────┴───────────────────────────────────────────────┘
```

### Error Budget Math

```
SLO: 99.9% availability per month

Error Budget = (1 - 0.999) × 30 days × 24 hours × 60 minutes
             = 0.001 × 43,200 minutes
             = 43.2 minutes downtime allowed per month

Burn Rate Examples:
- 1x burn rate: ใช้ budget เท่ากับ rate ปกติ → budget หมดใน 30 วัน
- 2x burn rate: ใช้ budget 2 เท่า → budget หมดใน 15 วัน  
- 10x burn rate: ใช้ budget 10 เท่า → ALERT! หมดใน 3 วัน
```

---

## ⚙️ Environment Setup

```bash
# ติดตั้ง Grafana (ถ้ายังไม่มี)
helm repo add grafana https://grafana.github.io/helm-charts
helm install grafana grafana/grafana \
  --namespace monitoring \
  --set adminPassword=admin \
  --set persistence.enabled=true

# ติดตั้ง Prometheus
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring

# ติดตั้ง packages
npm install prom-client express
```

---

## 🛠️ Step-by-Step Implementation

### Step 691: SLI Implementation

```javascript
// sli-metrics.js - Implement SLI measurements
const client = require('prom-client');

// ===== SLI 1: Availability =====
// Definition: percentage ของ requests ที่ตอบ status code 2xx หรือ 3xx
// (ไม่รวม 4xx ที่เป็น client errors)

const httpRequestsTotal = new client.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'path', 'status_code', 'le'],
});

const httpRequestsGood = new client.Counter({
  name: 'http_requests_good_total', 
  help: 'Total successful HTTP requests (non-5xx)',
  labelNames: ['method', 'path'],
});

// ===== SLI 2: Latency =====
// Definition: percentage ของ requests ที่ response time < 500ms

const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'path', 'status_code'],
  buckets: [0.01, 0.025, 0.05, 0.1, 0.2, 0.5, 1, 2, 5],
});

// ===== SLI 3: Error Rate =====
const httpErrors = new client.Counter({
  name: 'http_errors_total',
  help: 'Total HTTP errors (5xx)',
  labelNames: ['method', 'path', 'error_type'],
});

// ===== SLI 4: Throughput =====
const requestsPerSecond = new client.Gauge({
  name: 'requests_per_second',
  help: 'Current requests per second',
});

// Middleware สำหรับ collect SLI metrics
function sliMiddleware(req, res, next) {
  const start = Date.now();
  const path = normalizePath(req.path);
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    const statusCode = res.statusCode.toString();
    const method = req.method;
    
    // Track total requests
    httpRequestsTotal.inc({ method, path, status_code: statusCode });
    
    // Track response duration
    httpRequestDuration.observe({ method, path, status_code: statusCode }, duration);
    
    // Track good requests (non-5xx)
    if (!statusCode.startsWith('5')) {
      httpRequestsGood.inc({ method, path });
    } else {
      httpErrors.inc({ 
        method, 
        path, 
        error_type: statusCode === '503' ? 'service_unavailable' : 'server_error'
      });
    }
  });
  
  next();
}

// Normalize paths เพื่อลด cardinality
function normalizePath(path) {
  return path
    .replace(/\/\d+/g, '/:id')           // /users/123 → /users/:id
    .replace(/\/[a-f0-9-]{36}/g, '/:uuid') // UUIDs
    .replace(/\?.*/, '');                  // remove query strings
}

module.exports = { sliMiddleware, httpRequestsTotal, httpRequestsGood, httpRequestDuration };
```

### Step 692: SLO Definitions สำหรับ chuaikan.com

```yaml
# slo-definitions.yaml - SLO document สำหรับ chuaikan.com
slos:
  # SLO 1: Availability
  - name: api_availability
    description: "API returns successful responses"
    sli:
      metric: |
        sum(rate(http_requests_good_total[5m])) /
        sum(rate(http_requests_total[5m]))
      threshold: 0.999  # 99.9%
    slo_target: 0.999  # 99.9%
    error_budget_window: "30d"
    alert_burn_rate_fast: 14.4   # 1 hour window
    alert_burn_rate_slow: 6      # 6 hour window
  
  # SLO 2: Latency
  - name: api_latency
    description: "95% of API requests complete within 500ms"
    sli:
      metric: |
        histogram_quantile(0.95, 
          sum(rate(http_request_duration_seconds_bucket[5m])) 
          by (le)
        )
      threshold: 0.5  # 500ms
    slo_target: 0.95  # 95% of requests must meet this
    error_budget_window: "30d"
  
  # SLO 3: SOS Critical Path
  - name: sos_availability
    description: "SOS alerts must always be processed"
    sli:
      metric: |
        sum(rate(sos_alerts_processed_total[5m])) /
        sum(rate(sos_alerts_created_total[5m]))
    slo_target: 0.9999  # 99.99% (more strict!)
    error_budget_window: "30d"
    alert_burn_rate_fast: 50    # Very sensitive!
```

### Step 693: SLA Definition สำหรับ chuaikan.com

```markdown
# Service Level Agreement - chuaikan.com

## Scope
บริการ API และ web application ของ chuaikan.com

## Commitments

### 1. Availability
- **Target**: 99.9% uptime per calendar month
- **Measurement**: HTTP success rate (non-5xx responses)
- **Excluded**: Scheduled maintenance windows (announced 48h in advance)
- **Penalty**: Service credits if availability < 99.5%

### 2. API Response Time  
- **Target**: P95 < 500ms for all API endpoints
- **Measurement**: Server-side timing (excludes network latency)
- **Exception**: /api/search and /api/analytics may take up to 1000ms

### 3. SOS Service
- **Target**: 99.99% availability (ไม่มี downtime tolerance)
- **Alert Processing**: SOS alerts must be dispatched within 60 seconds
- **Priority**: SOS takes precedence over all other traffic

## Scheduled Maintenance
- Maintenance window: Tue-Thu 02:00-04:00 AM ICT
- Max 4 hours per month
- Announced via status page and email 48h in advance

## Status Page
https://status.chuaikan.com
```

### Step 694: Error Budget Calculation

```javascript
// error-budget.js - Error budget tracker
const client = require('prom-client');

const SLO_CONFIGS = {
  api_availability: {
    target: 0.999,           // 99.9%
    window_days: 30,
  },
  sos_availability: {
    target: 0.9999,          // 99.99%
    window_days: 30,
  },
  api_latency: {
    target: 0.95,            // 95% requests < 500ms
    window_days: 30,
  },
};

// Error budget remaining (เป็น ratio 0-1)
const errorBudgetRemaining = new client.Gauge({
  name: 'slo_error_budget_remaining_ratio',
  help: 'Remaining error budget as ratio of total budget',
  labelNames: ['slo_name'],
});

// Error budget burn rate (ควรอยู่ที่ 1.0 เพื่อใช้หมดพอดีใน 30 วัน)
const errorBudgetBurnRate = new client.Gauge({
  name: 'slo_error_budget_burn_rate',
  help: 'Error budget burn rate (1.0 = normal consumption rate)',
  labelNames: ['slo_name', 'window'],
});

async function calculateErrorBudget(sloName) {
  const config = SLO_CONFIGS[sloName];
  if (!config) return null;
  
  const totalBudget = 1 - config.target;  // e.g., 0.001 for 99.9%
  const totalMinutes = config.window_days * 24 * 60;
  const budgetMinutes = totalBudget * totalMinutes;
  
  // Query Prometheus for actual error rate
  const errorRate = await queryPrometheusRange(
    sloName === 'api_availability' 
      ? '1 - (sum(rate(http_requests_good_total[30d])) / sum(rate(http_requests_total[30d])))'
      : '1 - (sum(rate(sos_alerts_processed_total[30d])) / sum(rate(sos_alerts_created_total[30d])))',
    config.window_days
  );
  
  const usedBudget = errorRate;
  const remainingBudget = Math.max(0, totalBudget - usedBudget);
  const remainingRatio = remainingBudget / totalBudget;
  
  // Burn rate = (current error rate) / (error budget rate per time)
  // Normal rate = totalBudget / window
  const burnRate1h = await queryPrometheusBurnRate(sloName, '1h');
  const burnRate6h = await queryPrometheusBurnRate(sloName, '6h');
  
  const result = {
    slo_name: sloName,
    target: config.target,
    total_budget_percent: (totalBudget * 100).toFixed(3) + '%',
    total_budget_minutes: budgetMinutes.toFixed(0),
    used_budget_percent: (usedBudget * 100).toFixed(3) + '%',
    remaining_budget_percent: (remainingBudget * 100).toFixed(3) + '%',
    remaining_budget_minutes: (remainingBudget * totalMinutes).toFixed(0),
    remaining_ratio: remainingRatio.toFixed(4),
    burn_rate_1h: burnRate1h.toFixed(2),
    burn_rate_6h: burnRate6h.toFixed(2),
    projected_exhaustion_hours: burnRate1h > 0 
      ? ((remainingBudget / (burnRate1h * totalBudget / (config.window_days * 24))) ).toFixed(0)
      : 'Never',
  };
  
  // Update Prometheus metrics
  errorBudgetRemaining.set({ slo_name: sloName }, remainingRatio);
  errorBudgetBurnRate.set({ slo_name: sloName, window: '1h' }, burnRate1h);
  errorBudgetBurnRate.set({ slo_name: sloName, window: '6h' }, burnRate6h);
  
  return result;
}

// Helper functions
async function queryPrometheusRange(query, days) {
  // Mock implementation - ใน production query Prometheus API
  return 0.0005; // 0.05% error rate
}

async function queryPrometheusBurnRate(sloName, window) {
  // Mock implementation
  return 1.2; // slightly above 1x normal rate
}

module.exports = { calculateErrorBudget };
```

### Step 695: SLO Alerting Rules

```yaml
# prometheus/slo-alerts.yaml
groups:
- name: slo_alerts
  rules:
  
  # ===== Availability SLO Alerts =====
  
  # Fast burn alert: 2% budget in 1 hour = 14.4x burn rate
  # หมายถึง budget จะหมดภายใน ~2 วัน
  - alert: ErrorBudgetBurnRateFast
    expr: |
      (
        1 - (
          sum(rate(http_requests_good_total[1h])) /
          sum(rate(http_requests_total[1h]))
        )
      ) > (14.4 * 0.001)
    for: 2m
    labels:
      severity: critical
      slo: api_availability
    annotations:
      summary: "High error budget burn rate (fast)"
      description: |
        Error budget burning at {{ $value | humanizePercentage }}/hour
        Current burn rate: {{ printf "%.1f" (div $value 0.001) }}x
        At this rate, monthly budget exhausted in:
        {{ printf "%.0f" (div (mul 0.001 720) $value) }} hours
  
  # Slow burn alert: 5% budget in 6 hours = 6x burn rate  
  - alert: ErrorBudgetBurnRateSlow
    expr: |
      (
        1 - (
          sum(rate(http_requests_good_total[6h])) /
          sum(rate(http_requests_total[6h]))
        )
      ) > (6 * 0.001)
    for: 15m
    labels:
      severity: warning
      slo: api_availability
    annotations:
      summary: "Elevated error budget burn rate (slow)"
      description: |
        Error budget burning at {{ $value | humanizePercentage }}/hour over 6h window.
        Investigate gradual degradation.
  
  # Budget exhaustion warning
  - alert: ErrorBudgetNearExhaustion
    expr: |
      slo_error_budget_remaining_ratio{slo_name="api_availability"} < 0.10
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Error budget nearly exhausted!"
      description: |
        Only {{ $value | humanizePercentage }} of error budget remaining for month.
        Consider feature freeze and focus on reliability.
  
  # ===== Latency SLO Alerts =====
  - alert: LatencySLOViolation
    expr: |
      histogram_quantile(0.95,
        sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
      ) > 0.5
    for: 10m
    labels:
      severity: warning
      slo: api_latency
    annotations:
      summary: "Latency SLO violation"
      description: "P95 latency {{ $value | humanizeDuration }} exceeds 500ms SLO"
  
  # ===== SOS Service SLO Alerts (Critical!) =====
  - alert: SOSServiceDegraded
    expr: |
      (
        sum(rate(sos_alerts_processed_total[5m])) /
        sum(rate(sos_alerts_created_total[5m]))
      ) < 0.999
    for: 1m
    labels:
      severity: critical
      slo: sos_availability
      team: emergency
    annotations:
      summary: "SOS service availability below SLO!"
      description: |
        SOS processing rate: {{ $value | humanizePercentage }}
        SLO target: 99.99%
        This is a CRITICAL service - escalate immediately!
```

### Step 696: SLO Dashboard ใน Grafana

```json
// grafana/slo-dashboard.json
{
  "title": "SLO Dashboard - chuaikan.com",
  "uid": "slo-chuaikan",
  "tags": ["slo", "reliability"],
  "panels": [
    {
      "title": "API Availability SLO",
      "type": "stat",
      "gridPos": { "h": 4, "w": 6, "x": 0, "y": 0 },
      "targets": [{
        "expr": "sum(rate(http_requests_good_total[30d])) / sum(rate(http_requests_total[30d])) * 100",
        "legendFormat": "Current Availability %"
      }],
      "options": {
        "colorMode": "background",
        "thresholds": {
          "steps": [
            { "color": "red", "value": null },
            { "color": "yellow", "value": 99.5 },
            { "color": "green", "value": 99.9 }
          ]
        }
      },
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "decimals": 3,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "red", "value": null },
              { "color": "yellow", "value": 99.5 },
              { "color": "green", "value": 99.9 }
            ]
          }
        }
      }
    },
    {
      "title": "Error Budget Remaining (30 days)",
      "type": "gauge",
      "gridPos": { "h": 6, "w": 6, "x": 6, "y": 0 },
      "targets": [{
        "expr": "slo_error_budget_remaining_ratio{slo_name='api_availability'} * 100"
      }],
      "options": {
        "reduceOptions": {
          "calcs": ["lastNotNull"]
        },
        "orientation": "auto",
        "thresholds": {
          "mode": "percentage",
          "steps": [
            { "color": "red", "value": 0 },
            { "color": "yellow", "value": 25 },
            { "color": "green", "value": 50 }
          ]
        },
        "minValue": 0,
        "maxValue": 100
      }
    },
    {
      "title": "Error Budget Burn Rate",
      "type": "timeseries",
      "gridPos": { "h": 8, "w": 12, "x": 0, "y": 4 },
      "targets": [
        {
          "expr": "slo_error_budget_burn_rate{window='1h'}",
          "legendFormat": "1h burn rate"
        },
        {
          "expr": "slo_error_budget_burn_rate{window='6h'}",
          "legendFormat": "6h burn rate"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "custom": {
            "lineWidth": 2
          },
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "yellow", "value": 2 },
              { "color": "red", "value": 6 }
            ]
          }
        }
      }
    }
  ]
}
```

```bash
# Import dashboard to Grafana
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $GRAFANA_API_KEY" \
  -d @grafana/slo-dashboard.json \
  http://grafana.monitoring.svc.cluster.local:3000/api/dashboards/import
```

### Step 697: SLI/SLO API Endpoint

```javascript
// routes/slo.js - API สำหรับ SLO status
const express = require('express');
const router = express.Router();
const { calculateErrorBudget } = require('../sli-metrics');

// GET /api/slo/status - Current SLO status
router.get('/status', async (req, res) => {
  try {
    const slos = ['api_availability', 'sos_availability', 'api_latency'];
    
    const statuses = await Promise.all(
      slos.map(name => calculateErrorBudget(name))
    );
    
    res.json({
      timestamp: new Date().toISOString(),
      slos: statuses,
      overall_status: statuses.every(s => parseFloat(s.remaining_ratio) > 0.1)
        ? 'healthy'
        : statuses.some(s => parseFloat(s.remaining_ratio) <= 0)
          ? 'slo_violated'
          : 'degraded',
    });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// GET /api/slo/:name/history - Historical SLO data
router.get('/:name/history', async (req, res) => {
  // Return 30-day history of SLO compliance
  const mockHistory = Array.from({ length: 30 }, (_, i) => ({
    date: new Date(Date.now() - (29 - i) * 86400000).toISOString().split('T')[0],
    availability: 99.9 + Math.random() * 0.099,
    error_budget_remaining: 100 - (i * 100 / 30) * Math.random(),
  }));
  
  res.json({ slo: req.params.name, history: mockHistory });
});

module.exports = router;
```

### Step 698: Error Budget Policy

```markdown
# Error Budget Policy - chuaikan.com

## เมื่อไรที่เราใช้ Error Budget?

Error Budget คือ "สิทธิ์" ที่ทีมได้รับเพื่อ:
1. **Deploy** features ใหม่ (มีความเสี่ยงต่อ reliability)
2. **Experiment** กับ new technologies
3. **Conduct** planned maintenance

## Error Budget States

### 🟢 Healthy (> 50% remaining)
- Deploy freely
- รัน experiments
- Focus on feature development
- Chaos engineering tests allowed

### 🟡 Caution (10-50% remaining)  
- ลด deployment frequency
- Review all changes สำหรับ reliability impact
- Increase testing coverage
- Hold non-critical experiments

### 🔴 Critical (< 10% remaining)
- **Feature Freeze**: หยุด deploy features ใหม่
- Focus on reliability improvements
- On-call rotation เข้มข้นขึ้น
- Post-mortem สำหรับ recent incidents

### ⚫ Exhausted (0% remaining)
- Emergency: แจ้ง management
- Only critical bug fixes deployed
- All engineering focused on reliability
- Consider rolling back recent changes
```

### Step 699: SLO Review Process

```javascript
// slo-review.js - Monthly SLO review report generator
async function generateMonthlyReview(month, year) {
  // Gather data from Prometheus/database
  const data = await gatherSLOData(month, year);
  
  const report = {
    period: `${year}-${String(month).padStart(2, '0')}`,
    generated_at: new Date().toISOString(),
    
    summary: {
      slos_met: data.filter(s => s.compliance >= s.target).length,
      slos_missed: data.filter(s => s.compliance < s.target).length,
      worst_slo: data.sort((a, b) => a.compliance - b.compliance)[0],
    },
    
    slo_results: data.map(slo => ({
      name: slo.name,
      target: slo.target,
      actual: slo.compliance,
      met: slo.compliance >= slo.target,
      error_budget_consumed: ((1 - slo.compliance) / (1 - slo.target) * 100).toFixed(1) + '%',
      incidents: slo.incidents,
    })),
    
    action_items: generateActionItems(data),
    
    next_month_targets: {
      // ปรับ targets ตามผล actual
      api_availability: Math.max(0.999, data.find(s => s.name === 'api_availability')?.compliance - 0.001),
    },
  };
  
  return report;
}

function generateActionItems(data) {
  const items = [];
  
  for (const slo of data) {
    if (slo.compliance < slo.target) {
      items.push({
        type: 'SLO_MISSED',
        priority: 'HIGH',
        description: `${slo.name} missed target: ${slo.compliance} < ${slo.target}`,
        action: `Investigate ${slo.name} root cause and improve reliability`,
        owner: 'Platform Team',
        due: 'Next Sprint',
      });
    }
    
    if (slo.incidents > 0) {
      items.push({
        type: 'INCIDENT_REVIEW',
        priority: 'MEDIUM',
        description: `${slo.incidents} incidents in ${slo.name}`,
        action: 'Complete post-mortems and implement preventive measures',
        owner: 'On-Call Team',
        due: '2 Weeks',
      });
    }
  }
  
  return items;
}

async function gatherSLOData(month, year) {
  // Mock data - ใน production query จาก Prometheus + incident database
  return [
    {
      name: 'api_availability',
      target: 0.999,
      compliance: 0.9993,
      incidents: 1,
    },
    {
      name: 'sos_availability',
      target: 0.9999,
      compliance: 0.99995,
      incidents: 0,
    },
    {
      name: 'api_latency',
      target: 0.95,
      compliance: 0.962,
      incidents: 0,
    },
  ];
}

module.exports = { generateMonthlyReview };
```

### Step 700: Integrate SLO into Incident Response

```javascript
// incident-manager.js - SLO-aware incident management
class IncidentManager {
  constructor(prometheusClient, slackClient) {
    this.prometheus = prometheusClient;
    this.slack = slackClient;
    this.activeIncidents = new Map();
  }
  
  async checkSLOBreach() {
    const sloConfigs = [
      {
        name: 'api_availability',
        query: 'sum(rate(http_requests_good_total[5m])) / sum(rate(http_requests_total[5m]))',
        threshold: 0.999,
        severity: 'p2',
      },
      {
        name: 'sos_availability',
        query: 'sum(rate(sos_processed_total[5m])) / sum(rate(sos_created_total[5m]))',
        threshold: 0.9999,
        severity: 'p1',  // Critical!
      },
    ];
    
    for (const slo of sloConfigs) {
      const value = await this.prometheus.query(slo.query);
      
      if (value < slo.threshold) {
        await this.triggerIncident({
          type: 'SLO_BREACH',
          slo_name: slo.name,
          current_value: value,
          target: slo.threshold,
          severity: slo.severity,
          deficit: slo.threshold - value,
        });
      }
    }
  }
  
  async triggerIncident(incident) {
    const key = `${incident.type}:${incident.slo_name}`;
    
    // Deduplicate incidents
    if (this.activeIncidents.has(key)) {
      return;
    }
    
    this.activeIncidents.set(key, {
      ...incident,
      started_at: new Date(),
    });
    
    // Calculate error budget impact
    const burnRate = incident.deficit / (1 - incident.target);
    const budgetHoursRemaining = (
      (1 - burnRate) * 30 * 24  // rough calculation
    ).toFixed(1);
    
    // Notify
    await this.slack.sendAlert({
      channel: incident.severity === 'p1' ? '#incidents-p1' : '#incidents',
      text: `🚨 SLO Breach: ${incident.slo_name}`,
      attachments: [{
        color: 'danger',
        fields: [
          { title: 'SLO', value: incident.slo_name, short: true },
          { title: 'Severity', value: incident.severity.toUpperCase(), short: true },
          { title: 'Current Value', value: (incident.current_value * 100).toFixed(3) + '%', short: true },
          { title: 'SLO Target', value: (incident.target * 100).toFixed(3) + '%', short: true },
          { title: 'Error Budget Impact', value: `${(burnRate * 100).toFixed(1)}% of budget/hour`, short: true },
        ],
      }],
    });
    
    console.error(`INCIDENT: SLO Breach - ${incident.slo_name}`);
  }
  
  resolveIncident(sloName) {
    const key = `SLO_BREACH:${sloName}`;
    const incident = this.activeIncidents.get(key);
    
    if (incident) {
      const duration = (Date.now() - incident.started_at) / 1000 / 60;
      this.activeIncidents.delete(key);
      
      this.slack.sendMessage({
        channel: '#incidents',
        text: `✅ Resolved: ${sloName} SLO restored after ${duration.toFixed(0)} minutes`,
      });
    }
  }
}

// Run check every minute
const im = new IncidentManager(prometheusClient, slackClient);
setInterval(() => im.checkSLOBreach(), 60000);
```

---

## 🔧 Configuration Files

```yaml
# prometheus/recording-rules.yaml - Pre-compute SLI metrics
groups:
- name: sli_recording_rules
  interval: 60s
  rules:
  # 5-minute availability rate
  - record: sli:availability:5m
    expr: |
      sum(rate(http_requests_good_total[5m])) /
      sum(rate(http_requests_total[5m]))
  
  # 30-day error budget
  - record: sli:error_budget_consumed:30d
    expr: |
      1 - (
        sum(rate(http_requests_good_total[30d])) /
        sum(rate(http_requests_total[30d]))
      ) / 0.001  # divide by error budget (1 - 0.999)
```

---

## 🧪 Testing

```bash
# Test SLI metrics collection
curl https://api.chuaikan.com/metrics | grep http_requests

# Test SLO status API
curl https://api.chuaikan.com/api/slo/status | jq .

# Test error budget calculation
node -e "
  const { calculateErrorBudget } = require('./error-budget');
  calculateErrorBudget('api_availability').then(console.log);
"

# Simulate SLO breach (test alerting)
# Generate 5xx errors
for i in $(seq 1 100); do
  curl -s -o /dev/null https://api.chuaikan.com/api/force-error
done

# Check if alert fires
kubectl get prometheusrule -n monitoring
```

---

## ❌ Common Errors & Solutions

### Error 1: Burn rate alert fires too frequently

```yaml
# ปัญหา: Alert noise จาก short spikes
# แก้ไข: เพิ่ม for: duration และใช้ longer window
- alert: ErrorBudgetBurnRateFast
  expr: |
    rate(http_errors_total[1h]) / rate(http_requests_total[1h]) > 0.0144
  for: 5m  # ต้องเกิน 5 นาที ถึง alert
```

### Error 2: SLO calculation ไม่รวม maintenance window

```yaml
# แก้ไข: ใช้ absent() หรือ label exclusion
- record: sli:availability_excluding_maintenance
  expr: |
    sli:availability:5m unless on() absent(maintenance_mode{active="true"})
```

---

## ✅ Checklist

- [ ] กำหนด SLI metrics สำหรับ availability, latency, error rate
- [ ] กำหนด SLO targets (99.9% availability, P95 < 500ms)
- [ ] สร้าง SLA document สำหรับ users
- [ ] Implement Prometheus recording rules สำหรับ SLI calculation
- [ ] สร้าง error budget tracking
- [ ] ตั้งค่า fast burn + slow burn alerts ใน Prometheus
- [ ] สร้าง SLO dashboard ใน Grafana
- [ ] กำหนด Error Budget Policy
- [ ] ตั้งค่า monthly SLO review process
- [ ] Integrate SLO breach detection กับ incident management

---

## 🔗 References

- [Google SRE Book: SLOs](https://sre.google/sre-book/service-level-objectives/)
- [The Site Reliability Workbook](https://sre.google/workbook/implementing-slos/)
- [SLO Alerting (Apdex)](https://engineering.bitnami.com/articles/implementing-slos-using-prometheus.html)
- [Prometheus SLO Recording Rules](https://github.com/slok/sloth)
- [Sloth: SLO/SLI generator](https://sloth.dev/)

---

*Part 070 | Road to 1,000,000 Users/Day | chuaikan.com*
