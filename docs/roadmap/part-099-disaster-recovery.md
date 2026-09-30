# Part 099: Disaster Recovery & Business Continuity

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** World Class
> **Steps:** 981-990
> **เวลาโดยประมาณ:** 5 ชั่วโมง
> **Prerequisites:** Part 098 (SRE), Part 093 (Consensus Algorithms)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

DR ไม่ใช่ "เผื่อไว้" แต่เป็นส่วนสำคัญของ reliability engineering สำหรับ chuaikan.com ที่รองรับ SOS alerts ใน Part นี้เราจะ:

- วาง Business Continuity Plan (BCP)
- กำหนด RTO/RPO ตาม service tier
- เขียน Disaster Recovery Runbook
- ทำ DR Testing (Tabletop + Live Drill)
- ตั้ง Automated Failover

---

## 📖 ทฤษฎีและแนวคิด

### 1. RTO vs RPO

```
RTO (Recovery Time Objective):
"นานแค่ไหนที่ยอมรับได้ให้ service down?"

RPO (Recovery Point Objective):
"data loss มากแค่ไหนที่ยอมรับได้?"

ยิ่งต้องการ RTO/RPO น้อย → ค่าใช้จ่ายสูงขึ้น

Timeline:
│←─────────── RPO ──────────→│←── RTO ──→│
│                             │           │
Disaster                    Backup      Full
happened                   restored   recovery
```

### 2. Service Tiers สำหรับ chuaikan.com

```
┌─────────────────────────────────────────────────────────────────┐
│  Tier  │  Service              │  RTO    │  RPO    │  Cost       │
├─────────────────────────────────────────────────────────────────┤
│   1    │  SOS Alerts           │  5 min  │  0 sec  │  High       │
│        │  Emergency Response   │         │ (0 loss)│             │
├─────────────────────────────────────────────────────────────────┤
│   2    │  User Feed            │  30 min │  5 min  │  Medium-High│
│        │  Authentication       │         │         │             │
│        │  API Gateway          │         │         │             │
├─────────────────────────────────────────────────────────────────┤
│   3    │  Search               │  2 hours│  1 hour │  Medium     │
│        │  Notifications        │         │         │             │
│        │  Analytics            │         │         │             │
├─────────────────────────────────────────────────────────────────┤
│   4    │  Admin Dashboard      │  24 hours│ 4 hours │  Low        │
│        │  Reporting            │         │         │             │
└─────────────────────────────────────────────────────────────────┘
```

### 3. Disaster Scenarios

```
Scenario 1: Single AZ Failure
  Probability: Medium (happens ~1x/year in AWS)
  Impact: 1 AZ ใน 3 หยุดทำงาน
  Recovery: Auto-failover ไป surviving AZs

Scenario 2: Region Failure
  Probability: Low (< 1x/5 years)
  Impact: ap-southeast-1 ทั้งหมดหยุด
  Recovery: Failover ไป ap-northeast-1 (Tokyo)

Scenario 3: Database Corruption
  Probability: Very Low
  Impact: data ผิดพลาดหรือสูญหาย
  Recovery: Restore จาก backup

Scenario 4: Ransomware/Cyber Attack
  Probability: Low-Medium
  Impact: Systems compromised
  Recovery: Isolate + restore from clean backup

Scenario 5: Key Person Unavailable (Bus Factor)
  Probability: Medium
  Impact: ไม่มีใครรู้จะ fix ปัญหา
  Recovery: Documentation + cross-training
```

---

## 🛠️ Step-by-Step Implementation

### Step 981: Database Continuous Backup

```bash
# PostgreSQL Continuous Archiving (WAL-E/WAL-G)
# ทุก WAL segment จะถูก upload ไป S3 ทันที

# ติดตั้ง WAL-G
cat /etc/postgresql/postgresql.conf | grep -i wal

# postgresql.conf
cat >> /etc/postgresql/postgresql.conf << 'EOF'
# WAL Archiving Configuration
wal_level = replica
archive_mode = on
archive_command = 'wal-g wal-push %p'
archive_timeout = 60  # สูงสุด 60 วินาทีระหว่าง WAL segments

# Recovery
restore_command = 'wal-g wal-fetch %f %p'
EOF
```

```bash
# wal-g environment variables
cat > /etc/wal-g.env << 'EOF'
WALG_S3_PREFIX=s3://chuaikan-db-backup/postgres
AWS_REGION=ap-southeast-1
WALG_COMPRESSION_METHOD=lz4
WALG_DELTA_MAX_STEPS=6
EOF

# ทำ full backup ทุกวัน
cat > /etc/cron.d/walg-backup << 'EOF'
0 2 * * * postgres wal-g backup-push /var/lib/postgresql/data >> /var/log/walg-backup.log 2>&1
EOF

# ลบ backup เก่ากว่า 30 วัน
0 3 * * * postgres wal-g delete --confirm retain FULL 30 >> /var/log/walg-cleanup.log 2>&1
```

### Step 982: Multi-Region Setup สำหรับ Tier 1 Services

```hcl
# multi-region-rds.tf
# Aurora Global Database สำหรับ SOS (RPO = 0)

resource "aws_rds_global_cluster" "sos" {
  global_cluster_identifier = "chuaikan-sos-global"
  engine                    = "aurora-postgresql"
  engine_version            = "15.3"
  database_name             = "sos"
  deletion_protection       = true
}

# Primary cluster (ap-southeast-1 — Bangkok)
resource "aws_rds_cluster" "sos_primary" {
  provider                    = aws.primary
  cluster_identifier          = "chuaikan-sos-primary"
  engine                      = "aurora-postgresql"
  engine_version              = "15.3"
  global_cluster_identifier   = aws_rds_global_cluster.sos.id
  master_username             = "sosadmin"
  manage_master_user_password = true
  
  # Aurora Global replication: ~1 second cross-region lag
  # For RPO=0, we use synchronous writes (slower but safer)
  
  tags = {
    Tier = "1"
    RPO  = "0"
    RTO  = "5min"
  }
}

# Secondary cluster (ap-northeast-1 — Tokyo)
resource "aws_rds_cluster" "sos_secondary" {
  provider                  = aws.secondary
  cluster_identifier        = "chuaikan-sos-secondary"
  engine                    = "aurora-postgresql"
  engine_version            = "15.3"
  global_cluster_identifier = aws_rds_global_cluster.sos.id
  
  # Secondary cluster สำหรับ read-only ใน normal ops
  # ใช้เป็น primary เมื่อ failover
}
```

### Step 983: Automated Failover Logic

```python
# failover-controller.py
# ตรวจ health และทำ automatic failover

import boto3
import time
import logging
from enum import Enum

class ServiceStatus(Enum):
    HEALTHY = "healthy"
    DEGRADED = "degraded"
    FAILED = "failed"

class FailoverController:
    def __init__(self):
        self.route53 = boto3.client('route53', region_name='us-east-1')
        self.rds = boto3.client('rds', region_name='ap-southeast-1')
        self.cloudwatch = boto3.client('cloudwatch', region_name='ap-southeast-1')
        
        self.primary_region = 'ap-southeast-1'
        self.secondary_region = 'ap-northeast-1'
        
        self.health_check_id_primary = 'abc123'
        self.health_check_id_secondary = 'def456'
    
    def check_region_health(self, region):
        """ตรวจสอบ health ของ region"""
        try:
            # ตรวจ SOS service health
            import urllib.request
            url = f"https://sos-{region}.internal.chuaikan.com/health"
            
            req = urllib.request.Request(url, headers={'User-Agent': 'failover-controller'})
            with urllib.request.urlopen(req, timeout=5) as response:
                if response.status == 200:
                    return ServiceStatus.HEALTHY
                else:
                    return ServiceStatus.DEGRADED
        except Exception as e:
            logging.error(f"Health check failed for {region}: {e}")
            return ServiceStatus.FAILED
    
    def initiate_failover(self, from_region, to_region):
        """ทำ failover — เรียกเฉพาะหลัง human confirmation (ไม่ fully auto!)"""
        logging.warning(f"Initiating failover: {from_region} → {to_region}")
        
        # 1. Promote RDS secondary ให้เป็น primary
        logging.info("Promoting RDS secondary to primary...")
        self.rds.failover_global_cluster(
            GlobalClusterIdentifier='chuaikan-sos-global',
            TargetDbClusterIdentifier=f'arn:aws:rds:{to_region}:123456789:cluster:chuaikan-sos-secondary',
        )
        
        # รอ RDS failover เสร็จ
        logging.info("Waiting for RDS promotion...")
        time.sleep(60)
        
        # 2. อัพเดท Route53 ให้ชี้ไป region ใหม่
        logging.info("Updating DNS...")
        self._update_dns_failover(to_region)
        
        # 3. Scale up capacity ใน secondary region
        logging.info("Scaling up capacity in secondary region...")
        self._scale_up_region(to_region)
        
        # 4. Send notification
        logging.critical(f"FAILOVER COMPLETE: Traffic now routing to {to_region}")
        self._send_incident_notification(from_region, to_region)
    
    def _update_dns_failover(self, active_region):
        """อัพเดท Route53 Failover records"""
        hosted_zone_id = 'Z1234567890'
        
        self.route53.change_resource_record_sets(
            HostedZoneId=hosted_zone_id,
            ChangeBatch={
                'Changes': [{
                    'Action': 'UPSERT',
                    'ResourceRecordSet': {
                        'Name': 'api.chuaikan.com',
                        'Type': 'A',
                        'AliasTarget': {
                            'DNSName': f'alb.{active_region}.amazonaws.com',
                            'HostedZoneId': 'Z1H1FL5HABSF5',
                            'EvaluateTargetHealth': True,
                        },
                    }
                }]
            }
        )
    
    def _send_incident_notification(self, from_region, to_region):
        """แจ้งทีม ของ failover"""
        import requests
        
        message = {
            'text': f':rotating_light: *DISASTER FAILOVER COMPLETE*\n'
                   f'Primary: {from_region} (FAILED)\n'
                   f'Active: {to_region} (PROMOTED)\n'
                   f'Time: {time.strftime("%Y-%m-%d %H:%M:%S UTC")}\n'
                   f'Status: All Tier 1 services now serving from {to_region}\n'
                   f'Action needed: Monitor {to_region} capacity'
        }
        
        requests.post(os.environ['SLACK_WEBHOOK_URL'], json=message)
```

### Step 984: DR Runbook

```markdown
# Disaster Recovery Runbook
# SOS Service — Region Failover

## STEP 1: CONFIRM DISASTER (5 minutes)

Verify these are ALL true before proceeding:
1. [ ] AWS Status Page shows ap-southeast-1 incident
   URL: https://health.aws.amazon.com/
2. [ ] SOS health check failing for > 5 minutes
   CMD: curl -f https://api.chuaikan.com/sos/health
3. [ ] Multiple engineers confirm in #sos-alerts Slack channel
4. [ ] On-call lead gives GO decision

If NOT all true → do NOT failover (could make things worse)

## STEP 2: NOTIFY STAKEHOLDERS (2 minutes)

Send to #incidents Slack:
```
@channel DISASTER DECLARED — ap-southeast-1 failure
Beginning failover to ap-northeast-1
ETA: 20-30 minutes until full restoration
SOS service will be degraded during failover
```

## STEP 3: PROMOTE SECONDARY DATABASE (10 minutes)

# Check secondary cluster status
aws rds describe-db-clusters \
  --region ap-northeast-1 \
  --db-cluster-identifier chuaikan-sos-secondary

# Confirm lag < 5 seconds before promoting
aws rds describe-global-clusters \
  --global-cluster-identifier chuaikan-sos-global | \
  jq '.GlobalClusters[0].GlobalClusterMembers[] | select(.IsWriter == false) | .GlobalWriteForwardingStatus'

# PROMOTE (this takes 5-10 minutes)
aws rds failover-global-cluster \
  --region us-east-1 \
  --global-cluster-identifier chuaikan-sos-global \
  --target-db-cluster-identifier arn:aws:rds:ap-northeast-1:ACCOUNT:cluster:chuaikan-sos-secondary

# Wait for promotion
watch -n 5 "aws rds describe-db-clusters \
  --region ap-northeast-1 \
  --db-cluster-identifier chuaikan-sos-secondary | \
  jq '.DBClusters[0].Status'"

## STEP 4: UPDATE DNS (2 minutes)

# Update Route53
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch file://failover-dns.json

# Verify DNS propagation
dig +short api.chuaikan.com  # ต้องเป็น Tokyo IP

## STEP 5: VERIFY SERVICES (5 minutes)

curl -f https://api.chuaikan.com/health
curl -f https://api.chuaikan.com/sos/health

# Test SOS creation end-to-end
curl -X POST https://api.chuaikan.com/v1/sos \
  -H "Authorization: Bearer $TEST_TOKEN" \
  -d '{"lat": 13.7, "lng": 100.5, "severity": "low", "test": true}'

## STEP 6: SCALE UP SECONDARY (5 minutes)

# เพิ่ม capacity ใน Tokyo
kubectl --context tokyo scale deployment sos-service --replicas=10
kubectl --context tokyo scale deployment api-gateway --replicas=5

## ESTIMATED TOTAL TIME: 22-30 minutes
```

### Step 985: DR Testing Program

```bash
# tabletop-exercise.sh — จำลอง DR scenario ใน meeting room

cat << 'EOF'
=== Tabletop Exercise Script ===

Facilitator: "เวลา 14:30 น. AWS CloudWatch แจ้ง ap-southeast-1 unavailable"

Questions for team:
1. ใครเป็นคนแรกที่รับ alert?
2. ขั้นตอนแรกที่ทำคืออะไร?
3. ใครมีอำนาจในการ declare disaster?
4. Communication plan เป็นยังไง?
5. ถ้า primary on-call ไม่ตอบ ขั้นตอนต่อไปคืออะไร?

Inject complications:
- "Slack ใน ap-southeast-1 ก็ล้มด้วย"
- "CTO อยู่ต่างประเทศ ติดต่อไม่ได้"
- "Secondary region ยัง warm ไม่เต็ม 100%"
EOF
```

```bash
# live-failover-drill.sh — DR drill จริงบน production

echo "=== DR Failover Drill ==="
echo "Date: $(date)"
echo "Scenario: ap-southeast-1 region failure"

# 1. Record current state
echo "Current state:"
CURRENT_REGION=$(dig +short api.chuaikan.com | head -1)
echo "Current DNS: $CURRENT_REGION"

CURRENT_DB=$(aws rds describe-global-clusters \
  --global-cluster-identifier chuaikan-sos-global | \
  jq -r '.GlobalClusters[0].GlobalClusterMembers[] | select(.IsWriter == true) | .DBClusterArn')
echo "Current Primary DB: $CURRENT_DB"

# 2. Simulate failure
echo ""
echo "Simulating primary region failure..."
aws route53 update-health-check \
  --health-check-id $PRIMARY_HEALTH_CHECK_ID \
  --disabled  # Disable primary health check = simulate failure

# 3. Measure time to failover
START_TIME=$(date +%s)

# รอ DNS failover (Route53 detects unhealthy endpoint)
while true; do
  NEW_IP=$(dig +short api.chuaikan.com | head -1)
  if [ "$NEW_IP" != "$CURRENT_REGION" ]; then
    END_TIME=$(date +%s)
    FAILOVER_TIME=$((END_TIME - START_TIME))
    echo "DNS Failover completed in ${FAILOVER_TIME}s"
    break
  fi
  sleep 5
done

# 4. Verify services
echo "Verifying services..."
curl -f https://api.chuaikan.com/health && echo "✅ Health check passed"

# 5. Restore primary
echo ""
echo "Restoring primary region..."
aws route53 update-health-check \
  --health-check-id $PRIMARY_HEALTH_CHECK_ID \
  --no-disabled

# 6. Report
echo ""
echo "=== DR Drill Results ==="
echo "Failover time: ${FAILOVER_TIME}s"
echo "Target RTO: 300s (5 min)"
if [ $FAILOVER_TIME -lt 300 ]; then
  echo "✅ RTO target MET"
else
  echo "❌ RTO target MISSED — need to investigate"
fi
```

---

## 🔧 Configuration Files

### Route53 Failover Configuration

```hcl
# route53-failover.tf

# Health check สำหรับ primary region
resource "aws_route53_health_check" "primary" {
  fqdn              = "api-sg.chuaikan.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3  # 3 consecutive failures
  request_interval  = 10 # every 10 seconds
  
  tags = { Name = "primary-region-health" }
}

# Primary record (ap-southeast-1)
resource "aws_route53_record" "primary" {
  zone_id        = var.hosted_zone_id
  name           = "api.chuaikan.com"
  type           = "A"
  set_identifier = "primary"
  
  failover_routing_policy {
    type = "PRIMARY"
  }
  
  health_check_id = aws_route53_health_check.primary.id
  
  alias {
    name                   = aws_lb.primary.dns_name
    zone_id                = aws_lb.primary.zone_id
    evaluate_target_health = true
  }
}

# Secondary record (ap-northeast-1) — activate เมื่อ primary unhealthy
resource "aws_route53_record" "secondary" {
  zone_id        = var.hosted_zone_id
  name           = "api.chuaikan.com"
  type           = "A"
  set_identifier = "secondary"
  
  failover_routing_policy {
    type = "SECONDARY"
  }
  
  alias {
    name                   = aws_lb.secondary.dns_name
    zone_id                = aws_lb.secondary.zone_id
    evaluate_target_health = true
  }
}
```

---

## 🧪 Testing

### Recovery Validation Tests

```bash
#!/bin/bash
# recovery-validation.sh
# รันหลังจาก DR recovery เพื่อ validate ทุกอย่าง

set -e

echo "=== Post-Disaster Recovery Validation ==="

# 1. API health
echo "Testing API endpoints..."
curl -f https://api.chuaikan.com/health
echo "✅ API Gateway healthy"

# 2. SOS creation
echo "Testing SOS creation (end-to-end)..."
SOS_ID=$(curl -s -X POST https://api.chuaikan.com/v1/sos \
  -H "Authorization: Bearer $TEST_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"lat": 13.7, "lng": 100.5, "severity": "low", "test": true, "message": "DR test - please ignore"}' | \
  jq -r '.id')

[ -n "$SOS_ID" ] && echo "✅ SOS creation works (ID: $SOS_ID)" || (echo "❌ SOS creation failed"; exit 1)

# 3. Feed loading
echo "Testing feed..."
FEED=$(curl -s https://api.chuaikan.com/v1/feed \
  -H "Authorization: Bearer $TEST_TOKEN")
echo $FEED | jq '.posts | length' | grep -q "^[0-9]" && echo "✅ Feed works" || echo "❌ Feed failed"

# 4. Database data integrity
echo "Testing data integrity..."
# ตรวจว่า count ของ records สมเหตุสมผล
RECORD_COUNT=$(psql $DATABASE_URL -t -c "SELECT COUNT(*) FROM sos_alerts WHERE created_at > NOW() - INTERVAL '24 hours'")
echo "SOS records in last 24h: $RECORD_COUNT"

# 5. Cache connectivity
echo "Testing Redis..."
redis-cli -u $REDIS_URL ping | grep -q PONG && echo "✅ Redis connected" || echo "❌ Redis failed"

echo ""
echo "=== Validation Complete ==="
echo "All critical services validated"
```

---

## ❌ Common Errors & Solutions

### Error 1: Database Promotion ช้าเกินไป

```bash
# เร่ง Aurora Global failover
# ปกติ 5-10 นาที แต่ถ้าช้ากว่านั้น:

# ตรวจ lag ก่อน failover
aws rds describe-global-clusters \
  --global-cluster-identifier chuaikan-sos-global | \
  jq '.GlobalClusters[0].GlobalClusterMembers[] | select(.IsWriter == false)'

# ถ้า GlobalWriteForwardingStatus != "enabled" → ต้อง enable ก่อน
# ถ้า lag สูง → รอให้ sync ก่อน failover
```

### Error 2: DNS TTL สูงเกินไป

```bash
# ตรวจ DNS TTL ปัจจุบัน
dig +ttl api.chuaikan.com

# ถ้า TTL = 300 (5 นาที) → clients cache นาน
# ก่อน DR drill/maintenance → ลด TTL เป็น 60 วินาทีล่วงหน้า 1 ชั่วโมง
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.chuaikan.com",
        "Type": "CNAME",
        "TTL": 60,
        ...
      }
    }]
  }'
```

---

## ✅ Checklist

- [ ] RTO/RPO กำหนดสำหรับทุก service tier
- [ ] DR Runbook เขียนและ team ทุกคน review แล้ว
- [ ] Automated DNS failover ด้วย Route53 health checks
- [ ] Aurora Global Database สำหรับ Tier 1 (SOS)
- [ ] WAL-G continuous backup สำหรับ PostgreSQL
- [ ] Cross-region backup ทุกวัน (S3 replication)
- [ ] DR drill ทำสำเร็จ (ปีละ 2 ครั้ง)
- [ ] Tabletop exercise ทำแล้ว (ทีมทุกคนรู้ role ตัวเอง)
- [ ] Recovery Validation Tests ผ่าน
- [ ] DR communication plan ชัดเจน (ใครแจ้งใคร?)
- [ ] Backup restore ทดสอบแล้ว (ปีละ 4 ครั้ง)

---

## 🔗 References

- [AWS Disaster Recovery](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-workloads-on-aws.html)
- [Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html)
- [Route53 Failover](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-failover.html)
- [WAL-G Documentation](https://github.com/wal-g/wal-g)
- [Google SRE — Data Integrity](https://sre.google/sre-book/data-integrity/)

---

*Part 099 | Road to 1,000,000 Users/Day | chuaikan.com*
