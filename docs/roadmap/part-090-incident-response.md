# Part 090: Security Incident Response
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Professional
> **Steps:** 891–900
> **เวลาโดยประมาณ:** 8 ชั่วโมง
> **Prerequisites:** Part 088 (Pentest), Part 089 (PDPA)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- Incident classification: P1 (critical), P2 (major), P3 (minor)
- Incident response team roles
- Security Operations Center (SOC) basics
- SIEM ด้วย Grafana LGTM stack
- Playbook: data breach response
- Playbook: DDoS response
- Playbook: compromised credentials
- Post-incident review (blameless postmortem)
- Incident metrics: MTTD, MTTR

---

## 📖 ทฤษฎีและแนวคิด

### Incident Classification

| Priority | Definition | Response Time | Example |
|----------|-----------|---------------|---------|
| P1 Critical | กระทบ user ทั้งหมด หรือข้อมูลรั่วไหล | 15 นาที | Site down, Data breach |
| P2 Major | กระทบ user บางส่วน หรือ security issue | 1 ชั่วโมง | Payment failing, Auth bypass |
| P3 Minor | กระทบ feature เล็กน้อย | 4 ชั่วโมง | Notification delayed, UI bug |

### MTTD vs MTTR

- **MTTD** (Mean Time to Detect): เวลาเฉลี่ยตั้งแต่เกิดปัญหาจนพบ
- **MTTR** (Mean Time to Recover): เวลาเฉลี่ยตั้งแต่พบปัญหาจนแก้ไขเสร็จ

เป้าหมาย chuaikan.com:
- MTTD < 5 นาที (P1), < 30 นาที (P2)
- MTTR < 30 นาที (P1), < 4 ชั่วโมง (P2)

---

## ⚙️ Environment Setup

### Step 891: Grafana LGTM Stack สำหรับ SIEM

```bash
# LGTM = Loki (logs) + Grafana (viz) + Tempo (traces) + Mimir (metrics)

# ติดตั้งด้วย Helm
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Deploy Grafana LGTM stack
helm install grafana-lgtm grafana/lgtm-distributed \
  --namespace monitoring \
  --create-namespace \
  --values k8s/monitoring/lgtm-values.yaml
```

```yaml
# k8s/monitoring/lgtm-values.yaml
loki:
  enabled: true
  persistence:
    enabled: true
    size: 100Gi
  retention: 90d  # เก็บ logs 90 วัน

mimir:
  enabled: true
  persistence:
    enabled: true
    size: 200Gi

tempo:
  enabled: true
  persistence:
    enabled: true
    size: 50Gi

grafana:
  enabled: true
  adminPassword: "secure_grafana_password"
  persistence:
    enabled: true
  ingress:
    enabled: true
    hosts:
      - grafana.internal.chuaikan.com
```

---

## 🛠️ Step-by-Step Implementation

### Step 892: Security Alerts Rules

```yaml
# k8s/monitoring/security-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: security-alerts
  namespace: monitoring
spec:
  groups:
    - name: security.critical
      rules:
        # P1: Authentication failures spike
        - alert: AuthenticationSpike
          expr: |
            sum(rate(http_requests_total{path="/v1/auth/login",status="401"}[5m])) > 100
          for: 2m
          labels:
            severity: critical
            team: security
          annotations:
            summary: "Authentication failures spike detected"
            description: "{{ $value }} failed logins per second - possible brute force attack"
            runbook: "https://wiki.chuaikan.com/runbooks/auth-spike"

        # P1: Large data export
        - alert: LargeDataExport
          expr: |
            sum(rate(s3_request_total{bucket="chuaikan-data", operation="GetObject"}[5m])) > 1000
          for: 1m
          labels:
            severity: critical
            team: security
          annotations:
            summary: "Unusually large data export detected"

        # P1: Admin access from unknown IP
        - alert: AdminAccessUnknownIP
          expr: |
            sum(increase(admin_access_unknown_ip_total[5m])) > 0
          labels:
            severity: critical
            team: security

        # P2: Privilege escalation attempt
        - alert: PrivilegeEscalationAttempt
          expr: |
            sum(rate(http_requests_total{path=~"/v1/admin/.*",status="403"}[5m])) > 10
          for: 5m
          labels:
            severity: warning
            team: security

        # P2: SQL injection attempt
        - alert: SQLInjectionAttempt
          expr: |
            sum(rate(waf_blocked_requests_total{reason="sql_injection"}[5m])) > 5
          for: 1m
          labels:
            severity: warning
            team: security
```

### Step 893: Incident Response Automation

```python
# scripts/incident_responder.py
"""
Automated Incident Response สำหรับ common scenarios
"""

import boto3
import requests
import logging
from typing import Dict, Any
from datetime import datetime
import json

logger = logging.getLogger(__name__)

PAGERDUTY_KEY = "your_pagerduty_integration_key"
SLACK_WEBHOOK = "https://hooks.slack.com/services/xxx/yyy/zzz"


def trigger_pagerduty_alert(severity: str, summary: str, details: Dict):
    """ส่ง PagerDuty alert"""
    payload = {
        "routing_key": PAGERDUTY_KEY,
        "event_action": "trigger",
        "payload": {
            "summary": summary,
            "severity": severity,  # critical/error/warning/info
            "source": "chuaikan-soc",
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "custom_details": details
        }
    }

    response = requests.post(
        "https://events.pagerduty.com/v2/enqueue",
        json=payload,
        headers={"Content-Type": "application/json"}
    )

    if response.status_code == 202:
        logger.info(f"PagerDuty alert triggered: {summary}")
    else:
        logger.error(f"PagerDuty error: {response.text}")

    return response.json().get('dedup_key')


def send_slack_alert(channel: str, severity: str, title: str, details: str):
    """ส่ง Slack notification"""
    color_map = {
        'critical': '#FF0000',
        'warning': '#FFA500',
        'info': '#0000FF'
    }

    payload = {
        "channel": channel,
        "attachments": [{
            "color": color_map.get(severity, '#808080'),
            "title": f"[{severity.upper()}] {title}",
            "text": details,
            "footer": "chuaikan SOC",
            "ts": datetime.utcnow().timestamp()
        }]
    }

    requests.post(SLACK_WEBHOOK, json=payload)


class IncidentResponder:
    """Automated incident response actions"""

    def handle_ddos_detection(self, source_ips: list, rps: float):
        """จัดการ DDoS attack"""
        logger.warning(f"DDoS detected: {rps:.0f} RPS from {len(source_ips)} IPs")

        # 1. Block IPs ใน Cloudflare
        for ip in source_ips[:100]:  # Block top 100 IPs
            self._cloudflare_block_ip(ip)

        # 2. Enable rate limiting ที่เข้มงวดขึ้น
        self._update_cloudflare_rate_limit(requests_per_minute=30)

        # 3. Scale up ถ้าจำเป็น
        if rps > 50000:
            self._scale_up_ingress()

        # 4. Alert
        trigger_pagerduty_alert(
            severity="critical",
            summary=f"DDoS attack detected: {rps:.0f} RPS",
            details={"source_ips": source_ips[:10], "rps": rps}
        )

        send_slack_alert(
            channel="#incidents",
            severity="critical",
            title="DDoS Attack Detected",
            details=f"RPS: {rps:.0f}\nTop IPs: {', '.join(source_ips[:5])}\nActions: IPs blocked, rate limits tightened"
        )

    def handle_credential_breach(self, userId: str, anomaly_type: str):
        """จัดการ compromised credentials"""
        logger.critical(f"Credential anomaly for user {userId}: {anomaly_type}")

        # 1. Lock user account ทันที
        self._lock_user_account(userId)

        # 2. Revoke all sessions
        self._revoke_all_sessions(userId)

        # 3. Force password reset
        self._send_password_reset(userId)

        # 4. ตรวจสอบ audit log
        self._flag_for_security_review(userId, anomaly_type)

        trigger_pagerduty_alert(
            severity="critical",
            summary=f"Credential anomaly detected for user {userId}",
            details={"userId": userId, "anomalyType": anomaly_type}
        )

    def handle_data_access_anomaly(self, userId: str, records_accessed: int):
        """จัดการ unusual data access"""
        if records_accessed > 1000:
            logger.critical(f"User {userId} accessed {records_accessed} records - possible data exfiltration")

            # Throttle user
            self._apply_user_throttle(userId, requests_per_minute=10)

            # Alert SOC
            trigger_pagerduty_alert(
                severity="critical",
                summary=f"Possible data exfiltration: {records_accessed} records",
                details={"userId": userId, "recordsAccessed": records_accessed}
            )

    def _cloudflare_block_ip(self, ip: str):
        """Block IP ใน Cloudflare"""
        cf_token = "cloudflare_api_token"
        zone_id = "cloudflare_zone_id"

        requests.post(
            f"https://api.cloudflare.com/client/v4/zones/{zone_id}/firewall/access_rules/rules",
            headers={"Authorization": f"Bearer {cf_token}"},
            json={
                "mode": "block",
                "configuration": {"target": "ip", "value": ip},
                "notes": f"Auto-blocked by SOC at {datetime.utcnow().isoformat()}"
            }
        )

    def _update_cloudflare_rate_limit(self, requests_per_minute: int):
        """อัพเดท rate limit ใน Cloudflare"""
        logger.info(f"Updating rate limit to {requests_per_minute} req/min")

    def _scale_up_ingress(self):
        """Scale up ingress pods"""
        import subprocess
        subprocess.run([
            "kubectl", "scale", "deployment/nginx-ingress-controller",
            "--replicas=10", "-n", "ingress-nginx"
        ])

    def _lock_user_account(self, userId: str):
        """Lock user account"""
        logger.info(f"Locking account for user {userId}")

    def _revoke_all_sessions(self, userId: str):
        """Revoke all sessions"""
        logger.info(f"Revoking all sessions for user {userId}")

    def _send_password_reset(self, userId: str):
        """Send password reset email"""
        logger.info(f"Sending password reset to user {userId}")

    def _flag_for_security_review(self, userId: str, reason: str):
        """Flag for security review"""
        logger.info(f"Flagging user {userId} for security review: {reason}")

    def _apply_user_throttle(self, userId: str, requests_per_minute: int):
        """Apply throttle to specific user"""
        logger.info(f"Throttling user {userId} to {requests_per_minute} req/min")
```

### Step 894: Playbook: Data Breach

```bash
#!/bin/bash
# scripts/playbook-data-breach.sh
# Runbook: Data Breach Response

echo "=== DATA BREACH RESPONSE PLAYBOOK ==="
echo "Time: $(date -u)"
echo "Incident Commander: $(whoami)"

# Phase 1: DETECTION & CONTAINMENT (0-15 นาที)
echo ""
echo "=== PHASE 1: CONTAINMENT ==="

# 1. ระบุ scope
echo "1. Identifying breach scope..."
read -p "Affected data categories (e.g., 'emails, passwords'): " DATA_SCOPE
read -p "Estimated affected users: " USER_COUNT
read -p "Breach vector (e.g., 'SQL injection', 'credential stuffing'): " BREACH_VECTOR

# 2. Contain breach
echo "2. Containing breach..."
case "${BREACH_VECTOR}" in
  "sql injection")
    echo "  - Taking affected endpoint offline"
    kubectl scale deployment chuaikan-api --replicas=0 -n chuaikan
    ;;
  "credential stuffing")
    echo "  - Enabling strict rate limiting"
    ;;
  "unauthorized access")
    echo "  - Revoking compromised credentials"
    ;;
esac

# Phase 2: ASSESSMENT (15-60 นาที)
echo ""
echo "=== PHASE 2: ASSESSMENT ==="
echo "3. Checking audit logs..."
kubectl exec -n monitoring deploy/loki -- \
  logcli query '{app="chuaikan-api"} |= "error" | json' \
  --from="-2h" --to="now" --limit=100

# Phase 3: NOTIFICATION (ภายใน 72 ชั่วโมง)
echo ""
echo "=== PHASE 3: NOTIFICATION ==="
echo "PDPA requires notification to PDPC within 72 hours"
echo "Deadline: $(date -u -d '+72 hours')"

echo "4. Notifying affected users..."
read -p "Send notification emails? (yes/no): " SEND_NOTIF

if [ "$SEND_NOTIF" = "yes" ]; then
    echo "  - Queuing breach notification emails..."
    # python3 scripts/send_breach_notification.py --user-count="${USER_COUNT}"
fi

# Phase 4: RECOVERY
echo ""
echo "=== PHASE 4: RECOVERY ==="
echo "5. Applying patches..."
echo "6. Restoring services..."
echo "7. Monitoring for re-occurrence..."

echo ""
echo "=== BREACH RESPONSE COMPLETE ==="
echo "Remember: Submit PDPC notification before: $(date -u -d '+72 hours')"
echo "Schedule post-incident review within 5 business days."
```

### Step 895: Playbook: DDoS Response

```bash
#!/bin/bash
# scripts/playbook-ddos.sh

CURRENT_RPS=$(kubectl exec -n monitoring deploy/prometheus -- \
  promtool query instant 'sum(rate(nginx_ingress_controller_requests[1m]))' \
  | grep -o '[0-9]*\.' | head -1)

echo "=== DDoS RESPONSE PLAYBOOK ==="
echo "Current RPS: ${CURRENT_RPS}"

# Step 1: ยืนยันว่าเป็น DDoS ไม่ใช่ traffic spike จริง
echo "1. Verifying DDoS vs legitimate traffic spike..."
kubectl exec -n monitoring deploy/prometheus -- \
  promtool query instant \
  'topk(10, sum by (remote_addr) (rate(nginx_access_log_requests[5m])))' \
  | head -20

# Step 2: Enable Cloudflare Under Attack Mode
echo "2. Enabling Cloudflare Under Attack Mode..."
curl -X PATCH "https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/settings/security_level" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"value": "under_attack"}'

echo "Under Attack Mode: ENABLED"

# Step 3: Scale up infrastructure
echo "3. Scaling up infrastructure..."
kubectl scale deployment chuaikan-api --replicas=50 -n chuaikan
kubectl scale deployment nginx-ingress-controller --replicas=10 -n ingress-nginx

# Step 4: Block top attacker IPs
echo "4. Blocking top attacker IPs..."
# Script to block top 100 IPs

# Step 5: Monitor
echo "5. Monitoring..."
while true; do
    RPS=$(kubectl exec -n monitoring deploy/prometheus -- \
      promtool query instant 'sum(rate(nginx_ingress_controller_requests[1m]))' \
      | grep -o '[0-9.]*' | head -1)
    echo "$(date): ${RPS} RPS"
    
    if (( $(echo "${RPS} < 1000" | bc -l 2>/dev/null || echo 0) )); then
        echo "Attack subsiding. Current RPS: ${RPS}"
        break
    fi
    
    sleep 60
done

echo "=== DDoS CONTAINED ==="
echo "Disable Under Attack Mode when normal: bash scripts/disable-under-attack-mode.sh"
```

### Step 896: Blameless Postmortem Template

```markdown
# Post-Incident Review
## Incident: [INC-YYYY-XXX] — [Title]

**Date**: [วันที่]
**Duration**: [เวลาเริ่ม] - [เวลาสิ้นสุด] ([X ชั่วโมง X นาที])
**Severity**: P1 / P2 / P3
**Incident Commander**: [ชื่อ]
**Author**: [ชื่อ]

---

## 🎯 Impact Summary

- **ผู้ใช้ที่ได้รับผลกระทบ**: X,XXX users
- **Revenue impact**: ฿XX,XXX (ถ้าคำนวณได้)
- **MTTD**: X นาที
- **MTTR**: X นาที

---

## 📋 Timeline

| เวลา (UTC) | เหตุการณ์ |
|-----------|---------|
| 14:23 | Alert firing: Error rate > 5% |
| 14:25 | On-call เริ่มสืบสวน |
| 14:31 | Root cause ระบุได้: Database connection exhausted |
| 14:45 | Fix deployed |
| 14:52 | Error rate กลับปกติ |

---

## 🔍 Root Cause Analysis

**5 Whys:**
1. ทำไม user ไม่สามารถ login ได้?
   → API ส่ง 500 errors
2. ทำไม API ส่ง 500 errors?
   → Database connection pool exhausted
3. ทำไม connection pool exhausted?
   → Slow query ทำให้ connections ค้าง
4. ทำไม query ช้า?
   → New index ไม่ถูก apply ก่อน deploy
5. ทำไม index ไม่ถูก apply?
   → Migration ไม่มีใน deployment pipeline

**Root Cause**: Database migration script ไม่ถูก run ก่อน deploy new version

---

## ✅ Action Items

| Action | Owner | Due Date | Status |
|--------|-------|----------|--------|
| เพิ่ม migration check ใน deployment pipeline | DevOps | 2024-02-01 | Open |
| เพิ่ม database connection pool metrics alert | Ops | 2024-02-03 | Open |
| Review slow query threshold | Backend | 2024-02-07 | Open |

---

## 📚 Lessons Learned

1. **What went well**: Alert firing ภายใน 2 นาที, rollback เสร็จภายใน 10 นาที
2. **What could be improved**: Database migration ต้องเป็น step แรกของ deployment
3. **Lucky breaks**: ปัญหาเกิดช่วง low traffic (14:23 น.)

---

*Note: This is a blameless postmortem. The focus is on process improvement, not individual fault.*
```

### Step 897: Incident Metrics Dashboard

```yaml
# k8s/grafana/incident-metrics.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: incident-metrics-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "true"
data:
  incident-metrics.json: |
    {
      "title": "Incident Metrics - chuaikan.com",
      "panels": [
        {
          "title": "MTTD (Mean Time to Detect) - 30 days",
          "type": "stat",
          "targets": [{
            "expr": "avg(incident_mttd_minutes)",
            "legendFormat": "MTTD"
          }],
          "fieldConfig": {
            "defaults": {
              "unit": "m",
              "thresholds": {
                "steps": [
                  {"color": "green", "value": null},
                  {"color": "yellow", "value": 10},
                  {"color": "red", "value": 30}
                ]
              }
            }
          }
        },
        {
          "title": "MTTR (Mean Time to Recover) - 30 days",
          "type": "stat",
          "targets": [{
            "expr": "avg(incident_mttr_minutes)",
            "legendFormat": "MTTR"
          }],
          "fieldConfig": {
            "defaults": {
              "unit": "m",
              "thresholds": {
                "steps": [
                  {"color": "green", "value": null},
                  {"color": "yellow", "value": 60},
                  {"color": "red", "value": 240}
                ]
              }
            }
          }
        },
        {
          "title": "Incidents by Severity (30 days)",
          "type": "barchart",
          "targets": [
            {
              "expr": "sum(incidents_total{severity='p1'})",
              "legendFormat": "P1 Critical"
            },
            {
              "expr": "sum(incidents_total{severity='p2'})",
              "legendFormat": "P2 Major"
            },
            {
              "expr": "sum(incidents_total{severity='p3'})",
              "legendFormat": "P3 Minor"
            }
          ]
        }
      ]
    }
```

---

## 🧪 Testing

```bash
# ทดสอบ incident response ด้วย chaos engineering (staging only)
# ติดตั้ง Chaos Monkey for Kubernetes
kubectl apply -f https://raw.githubusercontent.com/linki/chaoskube/master/rbac.yaml

# ทดสอบ alert pipeline
curl -X POST "http://prometheus.monitoring.svc:9090/api/v1/alerts" \
  -H "Content-Type: application/json" \
  -d '[{
    "labels": {
      "alertname": "TestAlert",
      "severity": "critical",
      "team": "security"
    },
    "annotations": {
      "summary": "Test alert - please ignore"
    }
  }]'

# ตรวจสอบว่า PagerDuty รับ alert
# ตรวจสอบว่า Slack รับ notification
```

---

## ✅ Checklist

- [ ] Incident classification (P1/P2/P3) ชัดเจนและทีมเข้าใจ
- [ ] On-call rotation ตั้งไว้ใน PagerDuty
- [ ] Grafana LGTM stack ทำงานและ logs ingested
- [ ] Security alerts rules เปิดใช้งาน (brute force, privilege escalation)
- [ ] Data breach playbook ทดสอบแล้ว
- [ ] DDoS playbook ทดสอบแล้ว
- [ ] Compromised credentials playbook ทดสอบแล้ว
- [ ] Blameless postmortem template ทีมใช้งาน
- [ ] MTTD < 5 นาที สำหรับ P1
- [ ] MTTR < 30 นาที สำหรับ P1
- [ ] Incident metrics dashboard พร้อม
- [ ] Quarterly incident response drill

---

## 🔗 References

- [Google SRE Book - Incident Management](https://sre.google/sre-book/managing-incidents/)
- [PagerDuty Incident Response](https://response.pagerduty.com/)
- [Grafana LGTM Stack](https://grafana.com/go/webinar/getting-started-with-the-grafana-lgtm-stack/)

---
*Part 090 | Road to 1,000,000 Users/Day | chuaikan.com*
