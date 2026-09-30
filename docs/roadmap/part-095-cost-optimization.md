# Part 095: Cost Optimization at Scale

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** World Class
> **Steps:** 941-950
> **เวลาโดยประมาณ:** 5 ชั่วโมง
> **Prerequisites:** Part 091 (100M Architecture), Part 051 (AWS/GCP Overview)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

Cloud Cost เป็นหนึ่งในค่าใช้จ่ายหลักของ Tech Company เมื่อ scale ใหญ่ขึ้น ใน Part นี้เราจะ:

- วิเคราะห์ AWS Cost Breakdown ที่ 1M users/day
- ลดค่า EC2 ด้วย Reserved + Spot Instances (ประหยัดได้ 75%)
- ลดค่า RDS ด้วย Reserved Instances (ประหยัดได้ 40%)
- ใช้ S3 Intelligent-Tiering อย่างถูกต้อง
- ตั้งค่า Auto-shutdown สำหรับ Non-prod environments
- Target: ลด Cloud Cost 40% โดยไม่ลด Capacity

---

## 📖 ทฤษฎีและแนวคิด

### 1. AWS Cost Breakdown ที่ 1M Users/Day

```
ค่าใช้จ่าย AWS ต่อเดือน (ก่อน optimize):

┌──────────────────────────────────────────────────────┐
│  Service              Monthly Cost    % of Total     │
├──────────────────────────────────────────────────────┤
│  EC2 (EKS nodes)      $12,000         35%            │
│  RDS (PostgreSQL)     $8,000          23%            │
│  ElastiCache (Redis)  $3,000          9%             │
│  CloudFront           $4,000          12%            │
│  S3                   $2,500          7%             │
│  EBS (storage)        $1,500          4%             │
│  NAT Gateway          $1,500          4%             │
│  Data Transfer        $1,000          3%             │
│  CloudWatch/Other     $1,000          3%             │
├──────────────────────────────────────────────────────┤
│  TOTAL                $34,500         100%           │
└──────────────────────────────────────────────────────┘

Target หลัง optimize: $20,000/month (ลด 42%)
```

### 2. EC2 Pricing Models

```
┌──────────────────────────────────────────────────────────────┐
│  Pricing Model        Savings    Use Case                    │
├──────────────────────────────────────────────────────────────┤
│  On-Demand            0%         Testing, unpredictable load │
│  Reserved (1yr)       40%        Baseline capacity           │
│  Reserved (3yr)       60%        Core infrastructure         │
│  Savings Plans        up to 66%  Flexible Reserved           │
│  Spot Instances       up to 90%  Fault-tolerant workloads    │
└──────────────────────────────────────────────────────────────┘
```

**Optimal Mix Strategy:**
```
60% Reserved (baseline load)  → ประหยัด 40-60%
30% Spot (burstable workload) → ประหยัด 70-90%
10% On-Demand (emergency)     → ไม่มี discount
```

---

## 🛠️ Step-by-Step Implementation

### Step 941: วิเคราะห์ Current Cost ด้วย AWS Cost Explorer

```bash
# ดู cost breakdown ด้วย AWS CLI
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --filter '{
    "Tags": {
      "Key": "Project",
      "Values": ["chuaikan"]
    }
  }' \
  --query 'ResultsByTime[0].Groups[*].{Service: Keys[0], Cost: Metrics.BlendedCost.Amount}' \
  --output table
```

```python
# cost-analysis.py — วิเคราะห์ cost per user
import boto3
from datetime import datetime, timedelta

ce = boto3.client('ce', region_name='ap-southeast-1')

def get_monthly_cost(start_date, end_date, tag_key, tag_value):
    response = ce.get_cost_and_usage(
        TimePeriod={'Start': start_date, 'End': end_date},
        Granularity='MONTHLY',
        Metrics=['BlendedCost'],
        Filter={
            'Tags': {'Key': tag_key, 'Values': [tag_value]}
        }
    )
    return float(response['ResultsByTime'][0]['Total']['BlendedCost']['Amount'])

# คำนวณ cost per DAU
monthly_cost = get_monthly_cost('2024-01-01', '2024-02-01', 'Project', 'chuaikan')
avg_daily_users = 1_000_000
days_in_month = 31

cost_per_user_per_day = monthly_cost / (avg_daily_users * days_in_month)
print(f"Monthly cost: ${monthly_cost:,.2f}")
print(f"Cost per DAU per day: ${cost_per_user_per_day:.6f}")
print(f"Cost per 1000 DAU per day: ${cost_per_user_per_day * 1000:.4f}")
```

### Step 942: Reserved Instances สำหรับ Baseline

```bash
# ดู recommendation จาก AWS
aws ec2 describe-reserved-instances-offerings \
  --instance-type m6i.xlarge \
  --product-description "Linux/UNIX" \
  --offering-type "All Upfront" \
  --duration 31536000 \
  --output table

# หรือใช้ Compute Optimizer recommendations
aws compute-optimizer get-ec2-instance-recommendations \
  --filter name=FindingReasonCodes,values=CPUUnderprovisioned,MemoryUnderprovisioned \
  --output json | jq '.instanceRecommendations[] | {
    instance: .instanceArn,
    finding: .finding,
    recommendedType: .recommendationOptions[0].instanceType,
    savingsPercent: .recommendationOptions[0].estimatedMonthlySavings.value
  }'
```

```bash
# ซื้อ Reserved Instances
aws ec2 purchase-reserved-instances-offering \
  --reserved-instances-offering-id "offering-id-here" \
  --instance-count 10 \
  --limit-price Amount=500.00,CurrencyCode=USD
```

### Step 943: Spot Instances บน Kubernetes (Karpenter)

Karpenter คือ Kubernetes Node Provisioner ที่ฉลาดกว่า Cluster Autoscaler

```bash
# ติดตั้ง Karpenter
helm repo add karpenter https://charts.karpenter.sh/
helm install karpenter karpenter/karpenter \
  --namespace karpenter \
  --create-namespace \
  --set settings.aws.clusterName=chuaikan-prod \
  --set settings.aws.defaultInstanceProfile=KarpenterNodeInstanceProfile
```

```yaml
# karpenter-nodepool.yaml — Mix ของ Spot และ On-Demand
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: general-purpose
spec:
  template:
    spec:
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1beta1
        kind: EC2NodeClass
        name: default
      
      requirements:
        # ใช้ Spot ก่อน, fall back ไป On-Demand
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
        
        # ยอมรับ instance families หลายแบบ (ป้องกัน Spot interruption)
        - key: node.kubernetes.io/instance-type
          operator: In
          values: 
            - m6i.xlarge
            - m6a.xlarge
            - m5.xlarge
            - m5a.xlarge
            - m4.xlarge
            - c6i.2xlarge
            - c5.2xlarge
        
        - key: topology.kubernetes.io/zone
          operator: In
          values: ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]
  
  # Scale down nodes ที่ idle นานเกิน 30 minutes
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30m
  
  limits:
    cpu: 1000
    memory: 4000Gi
```

```yaml
# กำหนดให้ Stateless workloads ใช้ Spot
apiVersion: apps/v1
kind: Deployment
metadata:
  name: feed-worker
spec:
  template:
    metadata:
      labels:
        app: feed-worker
    spec:
      # Tolerate Spot interruptions
      tolerations:
      - key: karpenter.sh/capacity-type
        operator: Equal
        value: spot
        effect: NoSchedule
      
      # Prefer Spot nodes
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: karpenter.sh/capacity-type
                operator: In
                values: ["spot"]
      
      # Graceful shutdown เมื่อ Spot ถูก interrupt (2 นาที notice)
      terminationGracePeriodSeconds: 120
      
      containers:
      - name: feed-worker
        # Handle SIGTERM gracefully
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10"]
```

### Step 944: RDS Cost Optimization

```bash
# 1. Reserved Instances สำหรับ Production DB
aws rds purchase-reserved-db-instances-offering \
  --reserved-db-instances-offering-id "offering-id" \
  --db-instance-count 1

# 2. ดู underutilized RDS ด้วย Performance Insights
aws pi get-resource-metrics \
  --service-type RDS \
  --identifier db:chuaikan-prod \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-07T00:00:00Z \
  --metric-queries '[
    {
      "Metric": "db.load.avg",
      "GroupBy": {"Group": "db.wait_event", "Limit": 5}
    }
  ]'
```

```sql
-- ตรวจ Unused Indexes (cost ใน storage + write overhead)
SELECT 
  schemaname,
  tablename,
  indexname,
  idx_scan,
  pg_size_pretty(pg_relation_size(indexrelid)) as index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0  -- ไม่เคยถูกใช้!
  AND indisunique = false
ORDER BY pg_relation_size(indexrelid) DESC;

-- Drop unused indexes หลัง review
-- DROP INDEX CONCURRENTLY idx_posts_old_column;
```

```bash
# 3. Aurora Serverless v2 สำหรับ Non-Prod
# ปกติ RDS จ่ายตลอดเวลา แต่ Aurora Serverless scale to zero

aws rds create-db-cluster \
  --db-cluster-identifier chuaikan-staging \
  --engine aurora-postgresql \
  --engine-version 15.3 \
  --serverless-v2-scaling-configuration MinCapacity=0.5,MaxCapacity=4 \
  --master-username admin \
  --manage-master-user-password
```

### Step 945: S3 Intelligent-Tiering

```bash
# ตั้งค่า Intelligent-Tiering สำหรับ bucket ที่มี objects หลายอายุ
aws s3api put-bucket-intelligent-tiering-configuration \
  --bucket chuaikan-media \
  --id EntireBucket \
  --intelligent-tiering-configuration '{
    "Id": "EntireBucket",
    "Status": "Enabled",
    "Tierings": [
      {
        "Days": 90,
        "AccessTier": "ARCHIVE_ACCESS"
      },
      {
        "Days": 180,
        "AccessTier": "DEEP_ARCHIVE_ACCESS"
      }
    ]
  }'
```

```python
# s3-lifecycle-analysis.py
# วิเคราะห์ว่าควรใช้ storage tier อะไร

import boto3

s3 = boto3.client('s3')

# ดู storage class distribution
response = s3.list_objects_v2(Bucket='chuaikan-media', MaxKeys=1000)

storage_classes = {}
for obj in response.get('Contents', []):
    sc = obj.get('StorageClass', 'STANDARD')
    storage_classes[sc] = storage_classes.get(sc, 0) + obj['Size']

total = sum(storage_classes.values())
for sc, size in sorted(storage_classes.items(), key=lambda x: -x[1]):
    print(f"{sc}: {size/1e9:.1f}GB ({size/total*100:.1f}%)")
```

### Step 946: CloudFront Cost Reduction

```bash
# Origin Shield — ลด requests ถึง origin S3
# ประหยัดได้มากเมื่อมีหลาย CloudFront edges

aws cloudfront create-distribution \
  --distribution-config '{
    "Origins": {
      "Items": [{
        "Id": "chuaikan-media",
        "DomainName": "chuaikan-media.s3.amazonaws.com",
        "OriginShield": {
          "Enabled": true,
          "OriginShieldRegion": "ap-southeast-1"
        },
        "S3OriginConfig": {
          "OriginAccessIdentity": ""
        }
      }]
    },
    "DefaultCacheBehavior": {
      "Compress": true,
      "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6"
    }
  }'
```

### Step 947: Auto-Shutdown Non-Prod Environments

```python
# lambda-auto-shutdown.py
# Lambda function ที่รัน schedule (evening/weekend)

import boto3
import os
from datetime import datetime

ec2 = boto3.client('ec2')
rds = boto3.client('rds')

def handler(event, context):
    action = event.get('action', 'stop')
    env = event.get('environment', 'staging')
    
    # ดู instances ที่ tag ด้วย environment=staging
    instances = ec2.describe_instances(
        Filters=[
            {'Name': 'tag:Environment', 'Values': [env]},
            {'Name': 'tag:AutoShutdown', 'Values': ['true']},
            {'Name': 'instance-state-name', 'Values': ['running' if action == 'stop' else 'stopped']},
        ]
    )
    
    instance_ids = [
        i['InstanceId']
        for r in instances['Reservations']
        for i in r['Instances']
    ]
    
    if not instance_ids:
        return {'message': f'No {env} instances to {action}'}
    
    if action == 'stop':
        ec2.stop_instances(InstanceIds=instance_ids)
        print(f"Stopped {len(instance_ids)} {env} instances: {instance_ids}")
    else:
        ec2.start_instances(InstanceIds=instance_ids)
        print(f"Started {len(instance_ids)} {env} instances: {instance_ids}")
    
    # ทำเหมือนกันสำหรับ RDS
    clusters = rds.describe_db_clusters(
        Filters=[{'Name': 'tag:Environment', 'Values': [env]}]
    )
    
    for cluster in clusters['DBClusters']:
        if cluster.get('Status') in (['available'] if action == 'stop' else ['stopped']):
            if action == 'stop':
                rds.stop_db_cluster(DBClusterIdentifier=cluster['DBClusterIdentifier'])
            else:
                rds.start_db_cluster(DBClusterIdentifier=cluster['DBClusterIdentifier'])
    
    return {'message': f'Successfully {action}ped {env} environment'}
```

```yaml
# EventBridge schedule สำหรับ auto-shutdown
AWSTemplateFormatVersion: '2010-09-09'
Resources:
  StopStagingRule:
    Type: AWS::Events::Rule
    Properties:
      Name: stop-staging-evening
      Description: Stop staging environment at 8PM weekdays
      # วันจันทร์-ศุกร์ 20:00 Bangkok time = 13:00 UTC
      ScheduleExpression: "cron(0 13 ? * MON-FRI *)"
      State: ENABLED
      Targets:
        - Arn: !GetAtt AutoShutdownFunction.Arn
          Id: StopStaging
          Input: '{"action": "stop", "environment": "staging"}'
  
  StartStagingRule:
    Type: AWS::Events::Rule
    Properties:
      Name: start-staging-morning
      Description: Start staging environment at 8AM weekdays
      ScheduleExpression: "cron(0 1 ? * MON-FRI *)"  # 08:00 Bangkok = 01:00 UTC
      State: ENABLED
      Targets:
        - Arn: !GetAtt AutoShutdownFunction.Arn
          Id: StartStaging
          Input: '{"action": "start", "environment": "staging"}'
```

### Step 948: Cost Allocation Tags

```bash
# ตั้งค่า Cost Allocation Tags
aws ce create-cost-category-definition \
  --name "Team" \
  --rules-version "CostCategoryExpression.v1" \
  --rules '[
    {
      "Value": "feed-team",
      "Rule": {
        "Tags": {
          "Key": "Team",
          "Values": ["feed"],
          "MatchOptions": ["EQUALS"]
        }
      }
    },
    {
      "Value": "sos-team",
      "Rule": {
        "Tags": {
          "Key": "Team",
          "Values": ["sos"]
        }
      }
    }
  ]'

# Tag resources ทั้งหมดด้วย team label
aws ec2 create-tags \
  --resources i-1234567890abcdef0 \
  --tags \
    Key=Team,Value=feed-team \
    Key=Environment,Value=production \
    Key=Project,Value=chuaikan \
    Key=CostCenter,Value=engineering
```

---

## 🔧 Configuration Files

### AWS Budgets และ Alerts

```python
# setup-budgets.py
import boto3

budgets = boto3.client('budgets', region_name='us-east-1')  # budgets is global

def create_budget():
    budgets.create_budget(
        AccountId='123456789012',
        Budget={
            'BudgetName': 'chuaikan-monthly-budget',
            'BudgetLimit': {
                'Amount': '25000',  # $25,000/month limit
                'Unit': 'USD'
            },
            'TimeUnit': 'MONTHLY',
            'BudgetType': 'COST',
            'CostFilters': {
                'TagKeyValue': ['user:Project$chuaikan']
            }
        },
        NotificationsWithSubscribers=[
            {
                'Notification': {
                    'NotificationType': 'ACTUAL',
                    'ComparisonOperator': 'GREATER_THAN',
                    'Threshold': 80,  # แจ้งเตือนที่ 80%
                    'ThresholdType': 'PERCENTAGE',
                },
                'Subscribers': [{
                    'SubscriptionType': 'EMAIL',
                    'Address': 'engineering@chuaikan.com'
                }]
            },
            {
                'Notification': {
                    'NotificationType': 'FORECASTED',
                    'ComparisonOperator': 'GREATER_THAN',
                    'Threshold': 110,  # แจ้งถ้า forecast เกิน 110%
                    'ThresholdType': 'PERCENTAGE',
                },
                'Subscribers': [{
                    'SubscriptionType': 'EMAIL',
                    'Address': 'cto@chuaikan.com'
                }]
            }
        ]
    )

create_budget()
```

---

## 🧪 Testing

### Cost Optimization Validation

```bash
#!/bin/bash
# validate-cost-savings.sh

echo "=== Cost Optimization Validation ==="

# 1. Spot Instance utilization
echo "Spot Instance Utilization:"
aws ec2 describe-spot-instance-requests \
  --filters "Name=state,Values=active" \
  --query 'SpotInstanceRequests[*].{Instance: InstanceId, Type: LaunchSpecification.InstanceType}' \
  --output table

# 2. Reserved Instance coverage
echo "Reserved Instance Coverage:"
aws ce get-reservation-coverage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity MONTHLY \
  --query 'Total.CoverageHours.CoverageHoursPercentage'

# Target: > 60% coverage

# 3. S3 Intelligent-Tiering savings
echo "S3 Intelligent-Tiering Distribution:"
aws s3api list-objects-v2 \
  --bucket chuaikan-media \
  --query 'Contents[*].StorageClass' \
  --output text | sort | uniq -c | sort -rn
```

---

## ❌ Common Errors & Solutions

### Error 1: Spot Instance ถูก Interrupt บ่อย

```bash
# ตรวจ Spot interruption frequency
aws ec2 describe-spot-price-history \
  --instance-types m6i.xlarge m5.xlarge \
  --product-descriptions "Linux/UNIX" \
  --start-time 2024-01-01 \
  --query 'SpotPriceHistory[*].{Type: InstanceType, Price: SpotPrice, Time: Timestamp}' \
  --output table

# เลือก instance types ที่มี interruption rate ต่ำ
# ดู: https://ec2.shop/spot-advisor
```

```yaml
# เพิ่ม instance diversity เพื่อลด interruption
requirements:
  - key: node.kubernetes.io/instance-type
    operator: In
    values:
      - m6i.xlarge   # เพิ่ม options มากขึ้น
      - m6a.xlarge   # ทำให้ Karpenter มีทางเลือก
      - m5.xlarge    # เมื่อ AZ หนึ่ง spot ราคาสูง
      - m5a.xlarge
      - c6i.2xlarge  # ใช้ c6i ถ้า m ไม่ว่าง
      - c5.2xlarge
      - r6i.large    # memory-optimized option
```

### Error 2: Non-Prod ลืม Shutdown

```bash
# เพิ่ม guard: ถ้า environment ไม่มี tag AutoShutdown=true → alert
aws config put-config-rule \
  --config-rule '{
    "ConfigRuleName": "ec2-must-have-autoshutdown-tag",
    "Scope": {
      "ComplianceResourceTypes": ["AWS::EC2::Instance"]
    },
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "REQUIRED_TAGS"
    },
    "InputParameters": "{\"tag1Key\":\"AutoShutdown\"}"
  }'
```

---

## ✅ Checklist

- [ ] AWS Cost Explorer configured with project tags
- [ ] Monthly budget alert ตั้งค่าแล้ว ($25K threshold)
- [ ] Reserved Instances สำหรับ production baseline (≥60% coverage)
- [ ] Spot Instances สำหรับ worker/batch workloads
- [ ] Karpenter ติดตั้งแล้วและ scale down idle nodes
- [ ] RDS Reserved Instances สำหรับ production
- [ ] Aurora Serverless สำหรับ staging/dev
- [ ] S3 Intelligent-Tiering เปิดสำหรับ media bucket
- [ ] CloudFront Origin Shield เปิดแล้ว
- [ ] Auto-shutdown staging: หยุด 20:00, เริ่ม 08:00 weekdays
- [ ] Cost allocation tags ใส่ทุก resource แล้ว
- [ ] Cost per DAU tracking ใน monthly report
- [ ] Unused EC2, RDS, EBS review ทุกเดือน

---

## 🔗 References

- [AWS Cost Optimization Best Practices](https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/welcome.html)
- [Karpenter Documentation](https://karpenter.sh/docs/)
- [AWS Compute Optimizer](https://docs.aws.amazon.com/compute-optimizer/latest/ug/what-is.html)
- [EC2 Spot Instance Advisor](https://aws.amazon.com/ec2/spot/instance-advisor/)
- [S3 Intelligent-Tiering](https://docs.aws.amazon.com/AmazonS3/latest/userguide/intelligent-tiering.html)

---

*Part 095 | Road to 1,000,000 Users/Day | chuaikan.com*
