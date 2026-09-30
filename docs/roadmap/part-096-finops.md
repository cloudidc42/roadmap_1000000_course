# Part 096: FinOps — Cloud Cost Management

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** World Class
> **Steps:** 951-960
> **เวลาโดยประมาณ:** 4 ชั่วโมง
> **Prerequisites:** Part 095 (Cost Optimization)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

FinOps ไม่ใช่แค่การประหยัดเงิน แต่คือการ **ใช้เงินอย่างมีประสิทธิภาพ** เพื่อสร้าง business value ใน Part นี้เราจะ:

- เข้าใจ FinOps Framework ทั้ง 3 Phase
- คำนวณ Unit Economics: cost per user, cost per DAU
- ตั้งระบบ Cost Allocation ให้แต่ละ team
- ตั้ง Anomaly Detection สำหรับ cost spikes
- วาง Monthly FinOps Review Process

---

## 📖 ทฤษฎีและแนวคิด

### 1. FinOps Framework

FinOps Foundation กำหนด 3 phases:

```
┌──────────────────────────────────────────────────────────┐
│                    FinOps Lifecycle                      │
│                                                          │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐           │
│   │          │   │          │   │          │           │
│   │  INFORM  │──►│ OPTIMIZE │──►│  OPERATE │           │
│   │          │   │          │   │          │           │
│   └──────────┘   └──────────┘   └──────────┘           │
│        │               │               │                │
│   Visibility      Take Action     Continuous            │
│   Allocation      Save Money      Improvement           │
│   Benchmarking    Reserve/Spot    Automation            │
│                                                         │
└──────────────────────────────────────────────────────────┘
```

**Phase 1 — Inform:**
- ใครใช้เงินเท่าไร?
- Cost per service/team
- Cost vs Budget comparison

**Phase 2 — Optimize:**
- Reserved Instances
- Spot Instances
- Right-sizing
- Auto-shutdown

**Phase 3 — Operate:**
- Automation ของ optimization
- FinOps culture
- Team accountability

### 2. Unit Economics

**Unit Economics คือการวัด cost ต่อหน่วย business:**

```
Business Unit    Formula                    Target
─────────────────────────────────────────────────────
Cost per DAU     Total Cost / DAU           < $0.001/day
Cost per Post    Total Cost / Posts/day     < $0.0001
Cost per SOS     Total Cost / SOS/day       ยอมรับได้สูงกว่า
Cost per API     Total Cost / API calls     < $0.000001
Revenue / Cost   Revenue / AWS Cost         > 5x (5:1 ratio)
```

---

## 🛠️ Step-by-Step Implementation

### Step 951: ตั้งค่า Cost Visibility

```python
# cost-dashboard.py
# ดึง cost data และสร้าง weekly report

import boto3
import json
from datetime import datetime, timedelta

ce = boto3.client('ce', region_name='us-east-1')

def get_cost_by_team(start_date, end_date):
    """ดู cost แยกตาม team tag"""
    response = ce.get_cost_and_usage(
        TimePeriod={'Start': start_date, 'End': end_date},
        Granularity='DAILY',
        Metrics=['UnblendedCost'],
        GroupBy=[
            {'Type': 'TAG', 'Key': 'Team'},
            {'Type': 'DIMENSION', 'Key': 'SERVICE'},
        ]
    )
    
    # จัดรูปแบบ output
    results = {}
    for day in response['ResultsByTime']:
        date = day['TimePeriod']['Start']
        for group in day['Groups']:
            team = group['Keys'][0].replace('Team$', '') or 'untagged'
            service = group['Keys'][1]
            cost = float(group['Metrics']['UnblendedCost']['Amount'])
            
            if team not in results:
                results[team] = {}
            if service not in results[team]:
                results[team][service] = 0
            results[team][service] += cost
    
    return results

def calculate_unit_economics(total_cost, dau, posts_per_day, sos_per_day):
    return {
        'cost_per_dau': total_cost / dau if dau > 0 else 0,
        'cost_per_post': total_cost / posts_per_day if posts_per_day > 0 else 0,
        'cost_per_sos': total_cost / sos_per_day if sos_per_day > 0 else 0,
        'cost_per_1k_users': (total_cost / dau * 1000) if dau > 0 else 0,
    }

# รัน monthly report
end_date = datetime.now().strftime('%Y-%m-%d')
start_date = (datetime.now() - timedelta(days=30)).strftime('%Y-%m-%d')

cost_by_team = get_cost_by_team(start_date, end_date)

print("=== Monthly Cost by Team ===")
for team, services in cost_by_team.items():
    team_total = sum(services.values())
    print(f"\n{team}: ${team_total:,.2f}")
    for service, cost in sorted(services.items(), key=lambda x: -x[1])[:5]:
        print(f"  {service}: ${cost:,.2f}")

# Unit Economics
unit_econ = calculate_unit_economics(
    total_cost=20000,  # $20K/month
    dau=1_000_000,
    posts_per_day=500_000,
    sos_per_day=1_000,
)

print("\n=== Unit Economics ===")
for metric, value in unit_econ.items():
    print(f"{metric}: ${value:.6f}")
```

### Step 952: Cost Allocation Tags และ Chargeback

```bash
# activate cost allocation tags
aws ce update-cost-allocation-tags-status \
  --cost-allocation-tags-status \
    TagKey=Team,Status=Active \
    TagKey=Environment,Status=Active \
    TagKey=Project,Status=Active \
    TagKey=CostCenter,Status=Active

# สร้าง Cost Category สำหรับ teams
aws ce create-cost-category-definition \
  --name "TeamAllocation" \
  --rules-version "CostCategoryExpression.v1" \
  --rules '[
    {
      "Value": "Feed Team",
      "Rule": {
        "Tags": {
          "Key": "Team",
          "Values": ["feed", "feed-team"]
        }
      }
    },
    {
      "Value": "SOS Team",
      "Rule": {
        "Tags": {
          "Key": "Team",
          "Values": ["sos", "sos-team"]
        }
      }
    },
    {
      "Value": "Platform Team",
      "Rule": {
        "Tags": {
          "Key": "Team",
          "Values": ["platform", "infra"]
        }
      }
    }
  ]' \
  --default-value "Shared/Unallocated"
```

```python
# chargeback-report.py
# สร้าง monthly chargeback report สำหรับแต่ละ team

def generate_chargeback_report(month):
    cost_by_team = get_cost_by_team_for_month(month)
    shared_costs = get_shared_infrastructure_costs(month)  # k8s, networking, monitoring
    
    # แบ่ง shared costs ตาม team size (จำนวน engineers)
    team_sizes = {
        'Feed Team': 8,
        'SOS Team': 6,
        'Platform Team': 5,
        'Other Teams': 11,
    }
    total_engineers = sum(team_sizes.values())
    
    report = {}
    for team, direct_cost in cost_by_team.items():
        # Direct cost ของ team เอง
        # + สัดส่วน shared infrastructure
        team_engineers = team_sizes.get(team, 3)
        shared_allocation = shared_costs * (team_engineers / total_engineers)
        
        report[team] = {
            'direct_cost': direct_cost,
            'shared_allocation': shared_allocation,
            'total_cost': direct_cost + shared_allocation,
            'cost_per_engineer': (direct_cost + shared_allocation) / team_engineers,
        }
    
    return report

report = generate_chargeback_report('2024-01')
print("\n=== Monthly Chargeback Report ===")
for team, data in report.items():
    print(f"\n{team}:")
    print(f"  Direct:     ${data['direct_cost']:>10,.2f}")
    print(f"  Shared:     ${data['shared_allocation']:>10,.2f}")
    print(f"  Total:      ${data['total_cost']:>10,.2f}")
    print(f"  Per Eng:    ${data['cost_per_engineer']:>10,.2f}")
```

### Step 953: Anomaly Detection

```python
# anomaly-detection.py
# ตั้ง AWS Cost Anomaly Detection

import boto3

ce = boto3.client('ce', region_name='us-east-1')

# สร้าง Anomaly Monitor
monitor = ce.create_anomaly_monitor(
    AnomalyMonitor={
        'MonitorName': 'chuaikan-cost-monitor',
        'MonitorType': 'DIMENSIONAL',
        'MonitorDimension': 'SERVICE',  # monitor แยก service
    }
)

# สร้าง Subscription (alert เมื่อ anomaly พบ)
subscription = ce.create_anomaly_subscription(
    AnomalySubscription={
        'MonitorArnList': [monitor['MonitorArn']],
        'Subscribers': [
            {
                'Address': 'engineering@chuaikan.com',
                'Type': 'EMAIL',
            },
            {
                'Address': 'arn:aws:sns:ap-southeast-1:123456789:cost-alerts',
                'Type': 'SNS',
            }
        ],
        'Threshold': 100,  # Alert ถ้า anomaly > $100
        'ThresholdExpression': {
            'Dimensions': {
                'Key': 'ANOMALY_TOTAL_IMPACT_PERCENTAGE',
                'Values': ['10'],  # หรือ > 10% จาก baseline
                'MatchOptions': ['GREATER_THAN_OR_EQUAL']
            }
        },
        'SubscriptionName': 'chuaikan-cost-alerts',
        'Frequency': 'DAILY',
    }
)

print(f"Monitor ARN: {monitor['MonitorArn']}")
print(f"Subscription ARN: {subscription['SubscriptionArn']}")
```

```python
# custom-anomaly-detection.py
# Custom anomaly detection ด้วย statistical analysis

import numpy as np
from scipy import stats
import boto3
from datetime import datetime, timedelta

def detect_cost_anomaly(service_name, window_days=30, z_threshold=2.5):
    """
    ตรวจ cost anomaly โดยใช้ Z-score
    Z > 2.5 = likely anomaly
    """
    ce = boto3.client('ce', region_name='us-east-1')
    
    end = datetime.now()
    start = end - timedelta(days=window_days)
    
    # ดึง daily cost ย้อนหลัง 30 วัน
    response = ce.get_cost_and_usage(
        TimePeriod={
            'Start': start.strftime('%Y-%m-%d'),
            'End': end.strftime('%Y-%m-%d')
        },
        Granularity='DAILY',
        Metrics=['UnblendedCost'],
        Filter={
            'Dimensions': {
                'Key': 'SERVICE',
                'Values': [service_name]
            }
        }
    )
    
    costs = [
        float(day['Total']['UnblendedCost']['Amount'])
        for day in response['ResultsByTime']
    ]
    
    if len(costs) < 7:
        return None
    
    mean = np.mean(costs[:-1])  # average ของ historical data
    std = np.std(costs[:-1])
    today_cost = costs[-1]
    
    z_score = (today_cost - mean) / std if std > 0 else 0
    
    return {
        'service': service_name,
        'today_cost': today_cost,
        'mean_cost': mean,
        'std_cost': std,
        'z_score': z_score,
        'is_anomaly': abs(z_score) > z_threshold,
        'anomaly_direction': 'spike' if z_score > 0 else 'drop',
    }

# ตรวจทุก major service
services = ['Amazon EC2', 'Amazon RDS', 'Amazon ElastiCache', 'Amazon CloudFront']
for service in services:
    result = detect_cost_anomaly(service)
    if result and result['is_anomaly']:
        print(f"ANOMALY DETECTED: {result['service']}")
        print(f"  Today: ${result['today_cost']:.2f}")
        print(f"  Expected: ${result['mean_cost']:.2f} ± ${result['std_cost']:.2f}")
        print(f"  Z-score: {result['z_score']:.2f}")
```

### Step 954: Cost Forecasting

```python
# cost-forecast.py
# ทำนาย cost เดือนหน้าด้วย trend analysis

import boto3
from datetime import datetime, timedelta

def get_cost_forecast(days_ahead=30):
    ce = boto3.client('ce', region_name='us-east-1')
    
    today = datetime.now()
    end_date = (today + timedelta(days=days_ahead)).strftime('%Y-%m-%d')
    
    response = ce.get_cost_forecast(
        TimePeriod={
            'Start': today.strftime('%Y-%m-%d'),
            'End': end_date,
        },
        Metric='UNBLENDED_COST',
        Granularity='MONTHLY',
        PredictionIntervalLevel=95,  # 95% confidence interval
    )
    
    forecast = response['Total']
    mean = float(forecast['Amount'])
    lower = float(response['ForecastResultsByTime'][0]['PredictionIntervalLowerBound'])
    upper = float(response['ForecastResultsByTime'][0]['PredictionIntervalUpperBound'])
    
    print(f"Cost Forecast (next {days_ahead} days):")
    print(f"  Expected:  ${mean:,.2f}")
    print(f"  95% Range: ${lower:,.2f} - ${upper:,.2f}")
    
    # Alert ถ้า forecast เกิน budget
    monthly_budget = 25000
    if mean > monthly_budget:
        print(f"\n  ⚠️ Forecast (${mean:,.2f}) exceeds budget (${monthly_budget:,.2f})!")
        print(f"  Action needed: Reduce costs by ${mean - monthly_budget:,.2f}")

get_cost_forecast()
```

### Step 955: Monthly FinOps Review Process

```markdown
# Monthly FinOps Review Template

## Meeting Details
- Date: [First Wednesday of each month]
- Duration: 1 hour
- Attendees: CTO, VP Engineering, Team Leads, FinOps Lead

## Agenda

### 1. Last Month Summary (15 min)
- Total spend vs budget
- Cost breakdown by team
- Top 5 cost drivers
- Unit economics trends (cost per DAU)

### 2. Anomalies & Incidents (10 min)
- Any unexpected cost spikes?
- Root cause and prevention

### 3. Optimization Wins (10 min)
- What savings did we achieve last month?
- Which team contributed most?

### 4. Next Month Plan (15 min)
- Optimization opportunities identified
- Reserved Instance purchases planned
- Non-prod environment changes

### 5. Action Items (10 min)
- Owner, Due Date, Expected Savings
```

```python
# monthly-finops-report.py
# Auto-generate monthly FinOps report

def generate_monthly_report(month):
    data = {
        'month': month,
        'total_cost': get_total_cost(month),
        'budget': 25000,
        'cost_by_team': get_cost_by_team(month),
        'unit_economics': calculate_unit_economics(month),
        'top_services': get_top_services(month, limit=10),
        'month_over_month_change': get_mom_change(month),
        'reserved_coverage': get_reserved_coverage(month),
        'spot_utilization': get_spot_utilization(month),
        'savings_vs_ondemand': calculate_savings(month),
    }
    
    # Generate HTML report
    report_html = render_report_template(data)
    
    # Send via email
    send_report_email(
        to=['cto@chuaikan.com', 'engineering-leads@chuaikan.com'],
        subject=f"FinOps Report — {month}",
        html=report_html,
    )
    
    return data
```

---

## 🔧 Configuration Files

### Terraform สำหรับ Cost Management Infrastructure

```hcl
# finops-infrastructure.tf

# SNS Topic สำหรับ cost alerts
resource "aws_sns_topic" "cost_alerts" {
  name = "chuaikan-cost-alerts"
  
  tags = {
    Team    = "platform"
    Project = "chuaikan"
  }
}

resource "aws_sns_topic_subscription" "cost_alerts_email" {
  topic_arn = aws_sns_topic.cost_alerts.arn
  protocol  = "email"
  endpoint  = "engineering@chuaikan.com"
}

# Budget สำหรับทุก environment
resource "aws_budgets_budget" "monthly_production" {
  name         = "chuaikan-production-monthly"
  budget_type  = "COST"
  limit_amount = "25000"
  limit_unit   = "USD"
  time_unit    = "MONTHLY"
  
  cost_filter {
    name   = "TagKeyValue"
    values = ["user:Environment$production"]
  }
  
  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = ["engineering@chuaikan.com"]
    subscriber_sns_topic_arns  = [aws_sns_topic.cost_alerts.arn]
  }
  
  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type             = "PERCENTAGE"
    notification_type          = "FORECASTED"
    subscriber_email_addresses = ["cto@chuaikan.com"]
  }
}

resource "aws_budgets_budget" "monthly_staging" {
  name         = "chuaikan-staging-monthly"
  budget_type  = "COST"
  limit_amount = "2000"
  limit_unit   = "USD"
  time_unit    = "MONTHLY"
  
  cost_filter {
    name   = "TagKeyValue"
    values = ["user:Environment$staging"]
  }
  
  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 90
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = ["platform-team@chuaikan.com"]
  }
}
```

---

## 🧪 Testing

### FinOps Dashboard Validation

```python
# finops-dashboard-test.py
import unittest

class TestFinOpsDashboard(unittest.TestCase):
    def test_unit_economics_calculation(self):
        """ตรวจว่า unit economics คำนวณถูก"""
        result = calculate_unit_economics(
            total_cost=20000,
            dau=1_000_000,
            posts_per_day=500_000,
            sos_per_day=1_000,
        )
        
        # Cost per DAU ต้องน้อยกว่า $0.001/day
        self.assertLess(result['cost_per_dau'], 0.001)
        
        # Cost per post ต้องน้อยกว่า $0.0001
        self.assertLess(result['cost_per_post'], 0.0001)
    
    def test_anomaly_detection_sensitivity(self):
        """ตรวจว่า anomaly detection ไม่ false positive เยอะเกินไป"""
        historical_costs = [100, 102, 98, 101, 99, 100, 103] * 4  # 28 days
        
        # Normal variation ไม่ควร trigger anomaly
        result = detect_anomaly(historical_costs + [105])  # slightly above average
        self.assertFalse(result['is_anomaly'])
        
        # 3x spike ควร trigger anomaly
        result = detect_anomaly(historical_costs + [300])
        self.assertTrue(result['is_anomaly'])

if __name__ == '__main__':
    unittest.main()
```

---

## ❌ Common Errors & Solutions

### Error 1: Cost ไม่ถูก allocate ครบ (Untagged Resources)

```bash
# ดู untagged resources
aws resourcegroupstaggingapi get-resources \
  --tag-filters 'Key=Team' \
  --include-compliance-details \
  --query 'ResourceTagMappingList[?!contains(Tags[*].Key, `Team`)]' \
  --output table

# หรือใช้ AWS Config Rule
aws config put-config-rule \
  --config-rule '{
    "ConfigRuleName": "required-tags",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "REQUIRED_TAGS"
    },
    "InputParameters": "{\"tag1Key\":\"Team\",\"tag2Key\":\"Environment\",\"tag3Key\":\"Project\"}"
  }'
```

---

## ✅ Checklist

- [ ] Cost Allocation Tags ทำงาน (Team, Environment, Project)
- [ ] Monthly budget alerts ตั้งค่าครบทุก environment
- [ ] Cost Anomaly Detection เปิดใช้งาน
- [ ] Unit Economics tracking: cost per DAU < $0.001
- [ ] Chargeback model: แต่ละ team รู้ cost ของตัวเอง
- [ ] Monthly FinOps Review meeting จัดแล้ว (recurring calendar)
- [ ] Cost forecast แม่นยำ ±10%
- [ ] Reserved Instance coverage > 60% ของ production
- [ ] ทุก resource มี Team tag (0% untagged)
- [ ] FinOps champion ใน each team

---

## 🔗 References

- [FinOps Foundation](https://www.finops.org/)
- [AWS Cost Management Tools](https://aws.amazon.com/aws-cost-management/)
- [AWS Anomaly Detection](https://docs.aws.amazon.com/cost-management/latest/userguide/getting-started-ad.html)
- [Unit Economics for SaaS](https://a16z.com/16-metrics/)
- [FinOps Certified Practitioner](https://www.finops.org/certification/certified-finops-practitioner/)

---

*Part 096 | Road to 1,000,000 Users/Day | chuaikan.com*
