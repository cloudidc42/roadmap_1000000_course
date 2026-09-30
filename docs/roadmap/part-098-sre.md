# Part 098: SRE (Site Reliability Engineering)

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** World Class
> **Steps:** 971-980
> **เวลาโดยประมาณ:** 5 ชั่วโมง
> **Prerequisites:** Part 097 (Platform Engineering), Part 094 (Distributed Tracing)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

SRE (Site Reliability Engineering) คือ discipline ที่ Google สร้างขึ้นเพื่อ balance ระหว่าง "reliability" และ "velocity" ใน Part นี้เราจะ:

- เข้าใจ SRE Principles และ Error Budget
- ลด Toil ด้วย Automation
- ตั้ง On-call Rotation ที่ sustainable
- สร้าง Production Readiness Review (PRR)
- ทำ Blameless Postmortem

---

## 📖 ทฤษฎีและแนวคิด

### 1. SRE Core Principles

**From Google SRE Book:**

> "Hope is not a strategy"

SRE มองว่า reliability เป็น **engineering problem** ไม่ใช่ operational problem

**5 Core Principles:**

1. **Embracing Risk:** 100% reliability ไม่ใช่ goal — cost too high, slows down development
2. **Service Level Objectives (SLO):** วัด reliability ด้วย numbers
3. **Eliminating Toil:** งาน manual ที่ซ้ำซากต้องถูก automate
4. **Monitoring Distributed Systems:** The 4 Golden Signals
5. **Release Engineering:** Safe and fast releases

### 2. SLO, SLI, SLA

```
SLI (Service Level Indicator) = วัดอะไร
SLO (Service Level Objective) = เป้าหมาย
SLA (Service Level Agreement) = สัญญากับลูกค้า (consequence ถ้าไม่ได้)

ตัวอย่าง chuaikan.com:

SLI: "Percentage of requests completed in < 200ms"
SLO: "99.9% of requests complete in < 200ms over 30 days"
SLA: "If SLO drops below 99%, users get credit"

Error Budget = 1 - SLO
Error Budget ของ 99.9% = 0.1% = 43.8 minutes/month downtime allowed
```

### 3. Error Budget Policy

```
Error Budget Remaining    Team Action
──────────────────────────────────────────────────────
> 50%                    Feature development proceeds normally
25-50%                   Review release process, increase testing
< 25%                    Freeze non-critical feature releases
< 0% (exceeded)          All hands on reliability, no features
```

### 4. The 4 Golden Signals

```
Latency:    "How long does it take?" 
Errors:     "How often does it fail?"
Traffic:    "How much demand is there?"
Saturation: "How full is the system?"
```

---

## 🛠️ Step-by-Step Implementation

### Step 971: กำหนด SLOs สำหรับ chuaikan.com

```yaml
# slo-definitions.yaml

slos:
  feed_availability:
    description: "Feed API must be available"
    sli: "sum(rate(http_requests_total{service='feed-service',code!~'5..'}[5m])) / sum(rate(http_requests_total{service='feed-service'}[5m]))"
    target: 0.999  # 99.9%
    window: 30d
    error_budget_minutes: 43.8  # per month
  
  feed_latency:
    description: "Feed API P95 latency must be < 500ms"
    sli: "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{service='feed-service'}[5m])) by (le))"
    target: 0.5  # 500ms
    objective: 0.95  # 95% of requests
    window: 30d
  
  sos_availability:
    description: "SOS API must be highly available (critical service)"
    sli: "sum(rate(http_requests_total{service='sos-service',code!~'5..'}[5m])) / sum(rate(http_requests_total{service='sos-service'}[5m]))"
    target: 0.9999  # 99.99% — SOS is life-critical
    window: 30d
    error_budget_minutes: 4.38  # only 4.38 min/month!
  
  sos_latency:
    description: "SOS creation must be fast (P99 < 1s)"
    sli: "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{service='sos-service',endpoint='/sos'}[5m])) by (le))"
    target: 1.0  # 1 second P99
    objective: 0.99
    window: 30d
```

```javascript
// slo-calculator.js
// คำนวณ Error Budget consumption

function calculateErrorBudget(sloTarget, window = 30) {
  const daysInSeconds = window * 24 * 60 * 60;
  const allowedErrors = (1 - sloTarget) * daysInSeconds;
  const allowedMinutes = allowedErrors / 60;
  
  return {
    sloTarget,
    windowDays: window,
    allowedDowntimeSeconds: allowedErrors,
    allowedDowntimeMinutes: allowedMinutes.toFixed(1),
    allowedDowntimeHours: (allowedMinutes / 60).toFixed(2),
  };
}

// ตัวอย่าง
console.log('Feed (99.9%):', calculateErrorBudget(0.999));
// { allowedDowntimeMinutes: '43.8', allowedDowntimeHours: '0.73' }

console.log('SOS (99.99%):', calculateErrorBudget(0.9999));
// { allowedDowntimeMinutes: '4.4', allowedDowntimeHours: '0.07' }
```

### Step 972: Prometheus SLO Alerts

```yaml
# slo-alerts.yaml

groups:
- name: slo_alerts
  rules:
  
  # Alert เมื่อ Error Budget เหลือ < 25%
  - alert: FeedSLOErrorBudgetLow
    expr: |
      (
        1 - sum(rate(http_requests_total{service="feed-service",code!~"5.."}[30d]))
          / sum(rate(http_requests_total{service="feed-service"}[30d]))
      ) > (1 - 0.999) * 0.75
    for: 5m
    labels:
      severity: warning
      team: feed-team
    annotations:
      summary: "Feed SLO Error Budget is below 25%"
      description: "Current error rate is {{ $value | humanizePercentage }}, consuming more than 75% of monthly error budget"
      runbook: "https://runbook.chuaikan.com/feed-slo-budget"
  
  # Fast burn rate (spending budget 10x faster than expected)
  - alert: FeedSLOHighErrorBudgetBurnRate
    expr: |
      (
        sum(rate(http_requests_total{service="feed-service",code=~"5.."}[1h]))
        / sum(rate(http_requests_total{service="feed-service"}[1h]))
      ) > (1 - 0.999) * 10
    for: 5m
    labels:
      severity: critical
      team: feed-team
    annotations:
      summary: "Feed SLO burning error budget 10x too fast!"
      description: "If sustained, will exhaust monthly error budget in {{ $value }}h"
  
  # SOS critical alert — ไม่มี grace period
  - alert: SOSServiceDown
    expr: |
      sum(rate(http_requests_total{service="sos-service",code!~"5.."}[5m]))
      / sum(rate(http_requests_total{service="sos-service"}[5m])) < 0.99
    for: 1m  # Alert ทันทีใน 1 นาที (SOS = critical)
    labels:
      severity: critical
      team: sos-team
      pagerduty: "true"
    annotations:
      summary: "SOS Service availability dropped below 99%!"
      description: "This is a CRITICAL life-safety service. Current availability: {{ $value | humanizePercentage }}"
```

### Step 973: Toil Reduction

**Toil คือ:**
- งานที่ทำ manually, repetitive
- ไม่ก่อให้เกิด permanent value
- Scale linearly กับ traffic

```
ตัวอย่าง Toil ใน chuaikan.com:
- Manual restart ของ services ที่ crash
- Manual cleanup ของ old logs/files
- Manual certificate renewal
- Manual DB vacuum
- Manual scaling ก่อน events
```

```bash
# ก่อน: Manual cert renewal (toil)
# ทุก 3 เดือน engineer ต้องทำ manual
certbot renew
kubectl create secret tls chuaikan-tls --cert=cert.pem --key=key.pem

# หลัง: Automated ด้วย cert-manager (0 toil)
```

```yaml
# cert-manager-certificate.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: chuaikan-tls
  namespace: production
spec:
  secretName: chuaikan-tls-secret
  duration: 2160h   # 90 days
  renewBefore: 360h # Renew 15 days before expiry (auto!)
  subject:
    organizations:
      - chuaikan
  commonName: chuaikan.com
  dnsNames:
    - chuaikan.com
    - "*.chuaikan.com"
    - api.chuaikan.com
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

```python
# toil-tracker.py
# วัด toil ในแต่ละ week

import pandas as pd
from datetime import datetime

class ToilTracker:
    def __init__(self):
        self.entries = []
    
    def log_toil(self, description, minutes, category, automated=False):
        self.entries.append({
            'date': datetime.now().isoformat(),
            'description': description,
            'minutes': minutes,
            'category': category,  # manual-restart, cleanup, cert-renewal, etc.
            'automated': automated,
        })
    
    def weekly_report(self):
        df = pd.DataFrame(self.entries)
        
        print("=== Weekly Toil Report ===")
        print(f"Total toil: {df['minutes'].sum()} minutes")
        print(f"Automated: {df[df['automated']]['minutes'].sum()} minutes")
        print(f"Manual remaining: {df[~df['automated']]['minutes'].sum()} minutes")
        
        # SRE target: toil < 50% of eng time
        total_eng_time = 40 * 60  # 40 hours/week
        toil_percent = df['minutes'].sum() / total_eng_time * 100
        
        if toil_percent > 50:
            print(f"WARNING: Toil is {toil_percent:.1f}% of eng time (target < 50%)")
        
        # Top toil by category
        print("\nToil by category:")
        print(df.groupby('category')['minutes'].sum().sort_values(ascending=False))
```

### Step 974: On-Call Rotation

```yaml
# PagerDuty schedule (conceptual)
# ตั้งค่าผ่าน PagerDuty Web UI หรือ Terraform

on_call_schedule:
  name: "chuaikan-production-oncall"
  timezone: "Asia/Bangkok"
  
  layers:
    - name: "Primary On-Call"
      rotation_type: "weekly"
      handoff_time: "Monday 09:00"
      users:
        - engineer-1
        - engineer-2
        - engineer-3
        - engineer-4
      restrictions:
        - type: weekday_restriction
          start_day_of_week: 1   # Monday
          start_time_of_day: "09:00:00"
          end_day_of_week: 5     # Friday
          end_time_of_day: "18:00:00"
    
    - name: "Secondary On-Call (After Hours)"
      rotation_type: "weekly"
      handoff_time: "Monday 09:00"
      users:
        - senior-engineer-1
        - senior-engineer-2
      restrictions:
        # After hours: 18:00-09:00 + weekends
        - type: weekday_restriction
          start_day_of_week: 1
          start_time_of_day: "18:00:00"
          end_day_of_week: 2
          end_time_of_day: "09:00:00"

escalation_policy:
  - step: 1
    timeout: 5   # minutes
    targets: ["Primary On-Call"]
  - step: 2
    timeout: 15  # minutes  
    targets: ["Secondary On-Call"]
  - step: 3
    timeout: 30  # minutes
    targets: ["Engineering Manager"]
  - step: 4
    timeout: 45  # minutes
    targets: ["CTO"]
```

```python
# on-call-health.py
# ตรวจสอบ on-call health metrics

def calculate_oncall_load(oncall_data):
    """
    Healthy on-call metrics:
    - < 5 pages/shift ที่ต้อง action
    - < 2 overnight pages per week
    - > 50% pages can wait until morning (non-urgent)
    """
    
    pages = oncall_data['pages']
    overnight_pages = [p for p in pages if is_overnight(p['time'])]
    urgent_pages = [p for p in pages if p['severity'] == 'critical']
    
    metrics = {
        'total_pages': len(pages),
        'overnight_pages': len(overnight_pages),
        'pages_per_shift': len(pages) / oncall_data['shifts'],
        'urgent_rate': len(urgent_pages) / len(pages) if pages else 0,
    }
    
    health_issues = []
    
    if metrics['pages_per_shift'] > 5:
        health_issues.append(f"Too many pages per shift: {metrics['pages_per_shift']:.1f}")
    
    if metrics['overnight_pages'] > 2:
        health_issues.append(f"Too many overnight pages: {metrics['overnight_pages']}")
    
    if health_issues:
        print("On-call health issues:")
        for issue in health_issues:
            print(f"  - {issue}")
        print("Action: Review alert thresholds, increase automation")
    else:
        print("On-call load is healthy")
    
    return metrics
```

### Step 975: Production Readiness Review (PRR)

```markdown
# Production Readiness Review Checklist
# chuaikan.com — Required before any service goes to production

## Service: _______________
## Date: _______________
## Reviewed by: _______________

## 1. Architecture (15 points)
- [ ] Architecture Decision Record (ADR) เขียนแล้ว
- [ ] Single points of failure ระบุแล้ว
- [ ] Graceful degradation ออกแบบแล้ว (ถ้า service นี้ล้ม service อื่น impact อะไร?)
- [ ] Capacity estimate: max load ที่ service รองรับได้?

## 2. Reliability (20 points)
- [ ] Health check endpoint (/health) ใช้งานได้
- [ ] Liveness + Readiness probes ตั้งค่าแล้ว
- [ ] Graceful shutdown (handle SIGTERM, drain connections)
- [ ] Retry logic สำหรับ external dependencies
- [ ] Circuit breaker สำหรับ dependencies
- [ ] Timeout ทุก external call มีกำหนด

## 3. Monitoring (20 points)
- [ ] Prometheus metrics expose (/metrics)
- [ ] Grafana dashboard พร้อม
- [ ] Error rate alert ตั้งค่าแล้ว
- [ ] Latency alert ตั้งค่าแล้ว
- [ ] SLO กำหนดแล้วและ alert ทำงาน
- [ ] Logs structured (JSON) และ index ใน Loki

## 4. Security (15 points)
- [ ] Authentication/Authorization ครบ
- [ ] Input validation ทุก endpoint
- [ ] Secrets ใน AWS Secrets Manager (ไม่ใช่ env vars)
- [ ] HTTPS only (ไม่มี HTTP fallback)
- [ ] Rate limiting เปิดใช้

## 5. Operations (15 points)
- [ ] Runbook เขียนแล้ว (วิธี restart, rollback, debug)
- [ ] On-call escalation path ชัดเจน
- [ ] Rollback plan พร้อม (< 5 นาที)
- [ ] Deployment ผ่าน CI/CD (ไม่ใช่ manual)

## 6. Testing (15 points)
- [ ] Unit tests > 70% coverage
- [ ] Integration tests ผ่าน
- [ ] Load test ผ่านที่ 2x expected traffic
- [ ] Chaos test: ทดสอบเมื่อ dependency ล้ม

## Scoring
- 90-100: Ready for production
- 70-89:  Ready with documented risks
- < 70:   Not ready, address gaps first
```

### Step 976: Blameless Postmortem

```markdown
# Postmortem Template — chuaikan.com

## Incident Title
SOS Service Unavailable for 15 minutes — 2024-01-15 14:30-14:45 UTC+7

## Summary
ระหว่าง 14:30-14:45 ผู้ใช้ไม่สามารถส่ง SOS alerts ได้ ส่งผลกระทบ 3 SOS requests ที่ล่าช้า

## Impact
- Duration: 15 minutes
- Users affected: ~5,000 active users
- SOS requests delayed: 3 (no life-safety impact confirmed)
- Revenue impact: $0 (free service)
- SLO impact: Used 15 min of 4.38 min/month SOS error budget (exceeded!)

## Timeline
| Time (UTC+7) | Event |
|---|---|
| 14:28 | Deploy sos-service v2.3.1 completed |
| 14:30 | Error rate spike ตรวจพบใน Grafana |
| 14:31 | PagerDuty alert sent to on-call |
| 14:35 | On-call engineer ตอบ alert |
| 14:38 | Root cause identified: memory leak ใน new code |
| 14:42 | Rollback sos-service → v2.3.0 initiated |
| 14:45 | Service restored to normal |
| 14:50 | Incident marked resolved |

## Root Cause
sos-service v2.3.1 มี memory leak ใน image processing feature ใหม่
เมื่อ memory ถึง 90%: OOMKilled → pod restart → brief unavailability

## Contributing Factors
1. Load test ไม่ได้ include large images
2. Memory limit ตั้งต่ำเกิน (256Mi แทน 512Mi)
3. Staging environment มี memory น้อยกว่า production

## Lessons Learned
1. ต้องเพิ่ม memory-intensive tests ใน CI
2. Production readiness checklist ต้อง include memory profiling
3. Canary deployment ควรใช้สำหรับ memory-sensitive features

## Action Items
| Action | Owner | Due Date |
|---|---|---|
| เพิ่ม memory profiling ใน CI pipeline | Backend Team | 2024-01-22 |
| อัพเดท PRR checklist เพิ่ม memory test | Platform Team | 2024-01-20 |
| ตั้ง memory leak detector (clinic.js) ใน staging | SRE Team | 2024-01-25 |
| Review memory limits ทุก services | Platform Team | 2024-01-31 |

## What Went Well
- Alert ทำงานได้เร็ว (2 นาทีหลัง deploy)
- Rollback สำเร็จใน 3 นาที (target < 5 min)
- Communication ชัดเจนและทันเวลา

## คำถาม (ไม่ใช่ blame)
- ทำไม load test ไม่ cover กรณีนี้?
- Process อะไรที่ป้องกัน regression แบบนี้ได้?
```

---

## 🔧 Configuration Files

### SRE Metrics Dashboard (Grafana)

```json
{
  "title": "SRE Metrics — chuaikan.com",
  "panels": [
    {
      "title": "Error Budget Remaining (Feed)",
      "type": "gauge",
      "targets": [{
        "expr": "1 - (sum(rate(http_requests_total{service='feed-service',code=~'5..'}[30d])) / sum(rate(http_requests_total{service='feed-service'}[30d]))) / (1 - 0.999)",
        "legendFormat": "Feed Error Budget"
      }],
      "thresholds": [
        { "value": 0, "color": "red" },
        { "value": 0.25, "color": "yellow" },
        { "value": 0.5, "color": "green" }
      ]
    },
    {
      "title": "MTTR (Mean Time to Recover)",
      "type": "stat",
      "description": "Average time from alert to resolution"
    },
    {
      "title": "Change Failure Rate",
      "type": "timeseries",
      "description": "% of deploys that caused incident"
    }
  ]
}
```

---

## 🧪 Testing

### Chaos Engineering

```bash
# chaos-test.sh — ทดสอบ reliability ด้วย chaos experiments

# 1. Kill random pod — ตรวจว่า service ยัง healthy
kubectl exec -n production \
  $(kubectl get pods -n production -l app=feed-service -o name | shuf | head -1) \
  -- kill -9 1

# รอ 30 วินาที แล้วตรวจ
sleep 30
curl -f https://api.chuaikan.com/health || echo "FAILED!"

# 2. Network partition — ตรวจ circuit breaker
# ใช้ Chaos Mesh หรือ Litmus Chaos
kubectl apply -f - << 'EOF'
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: feed-to-db-partition
spec:
  action: partition
  mode: one
  selector:
    namespaces: [production]
    labelSelectors:
      app: feed-service
  direction: to
  target:
    selector:
      namespaces: [production]
      labelSelectors:
        app: postgresql
  duration: 60s
EOF
```

---

## ❌ Common Errors & Solutions

### Error 1: SLO Alert เยอะเกินไป (Alert Fatigue)

```
ปัญหา: ได้รับ alert ทุก 5 นาที → ignore alerts → miss real incident
```

```yaml
# แก้: ใช้ multi-window multi-burn-rate alerting
# Alert เฉพาะเมื่อ burn rate สูงผิดปกติ ไม่ใช่แค่ error มีเพิ่มขึ้น

- alert: HighErrorBudgetBurnRate
  expr: |
    (
      sum(rate(http_errors_total[1h])) / sum(rate(http_requests_total[1h]))
      > 14.4 * (1 - 0.999)  # 14.4 = 1% burn rate/hour = budget gone in 3 days
    )
    AND
    (
      sum(rate(http_errors_total[5m])) / sum(rate(http_requests_total[5m]))
      > 14.4 * (1 - 0.999)
    )
  # เฉพาะเมื่อ BOTH 1h และ 5m windows ผิดปกติ = ลด false positives
```

---

## ✅ Checklist

- [ ] SLO กำหนดสำหรับทุก critical service (SOS 99.99%, Feed 99.9%)
- [ ] Error Budget tracking ใน Grafana
- [ ] Error Budget Policy เป็น written policy
- [ ] SLO alerts (burn rate based, ไม่ใช่ threshold based)
- [ ] Toil measurement ใน weekly team meetings
- [ ] On-call rotation fair (ไม่มีคนเดียวรับผิดชอบทั้งหมด)
- [ ] On-call compensation ชัดเจน
- [ ] PRR checklist ทุก service ต้องผ่านก่อน production
- [ ] Postmortem ทุก incident P1/P2 ภายใน 48 ชั่วโมง
- [ ] Postmortem action items ติดตาม completion rate

---

## 🔗 References

- [Google SRE Book (Free Online)](https://sre.google/sre-book/table-of-contents/)
- [Google SRE Workbook](https://sre.google/workbook/table-of-contents/)
- [SLO Adoption Guide](https://sre.google/workbook/implementing-slos/)
- [DORA State of DevOps Report](https://dora.dev/)
- [Blameless Postmortem Guide](https://www.atlassian.com/incident-management/postmortem/blameless)

---

*Part 098 | Road to 1,000,000 Users/Day | chuaikan.com*
