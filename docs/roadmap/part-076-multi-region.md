# Part 076: Multi-region Deployment
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Professional
> **Steps:** 751–760
> **เวลาโดยประมาณ:** 8 ชั่วโมง
> **Prerequisites:** Part 031 (Kubernetes), Part 051 (AWS Overview), Part 071 (High Availability)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ความแตกต่างระหว่าง Active-Active และ Active-Passive multi-region
- การตั้งค่า chuaikan.com ให้มี primary region ที่ Singapore + DR ที่ Tokyo
- Route 53 latency-based routing พร้อม health checks
- Data replication strategy สำหรับ PostgreSQL, Redis, และ S3
- Terraform สำหรับสร้าง multi-region infrastructure

---

## 📖 ทฤษฎีและแนวคิด

### Active-Active vs Active-Passive

**Active-Active**: ทั้งสอง region รับ traffic พร้อมกัน
- ข้อดี: High availability, ลด latency สำหรับ user ใกล้ DR region
- ข้อเสีย: ซับซ้อนกว่า, ต้องจัดการ conflict ของข้อมูล, ค่าใช้จ่ายสูงกว่า

**Active-Passive**: มีแค่ primary region ที่รับ traffic, DR region standby อยู่
- ข้อดี: ง่ายกว่า, ข้อมูลสอดคล้องกัน
- ข้อเสีย: DR region ไม่ได้ใช้งาน, failover ใช้เวลานานกว่า

**chuaikan.com เลือก Active-Passive** เพราะ:
- User หลักอยู่ในไทย → Singapore ใกล้ที่สุด
- ลด complexity ของ data conflict
- ประหยัดค่าใช้จ่ายในระยะแรก

### RPO/RTO Targets
- **RPO (Recovery Point Objective)**: ข้อมูลสูญหายได้ไม่เกิน 5 นาที
- **RTO (Recovery Time Objective)**: ระบบต้องกลับมาทำงานภายใน 30 นาที

---

## ⚙️ Environment Setup

### Step 751: สร้าง AWS Accounts Structure

```bash
# สร้าง directory structure สำหรับ Terraform
mkdir -p infra/terraform/multi-region/{singapore,tokyo,global}

# ติดตั้ง Terraform
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform -y

terraform version
# Terraform v1.9.x
```

---

## 🛠️ Step-by-Step Implementation

### Step 752: Terraform Global Configuration

```hcl
# infra/terraform/multi-region/versions.tf
terraform {
  required_version = ">= 1.9.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  backend "s3" {
    bucket         = "chuaikan-terraform-state"
    key            = "multi-region/terraform.tfstate"
    region         = "ap-southeast-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}

# Provider สำหรับ Singapore (Primary)
provider "aws" {
  alias  = "singapore"
  region = "ap-southeast-1"
}

# Provider สำหรับ Tokyo (DR)
provider "aws" {
  alias  = "tokyo"
  region = "ap-northeast-1"
}

# Provider สำหรับ us-east-1 (Route 53, ACM global)
provider "aws" {
  alias  = "global"
  region = "us-east-1"
}
```

```hcl
# infra/terraform/multi-region/variables.tf
variable "project_name" {
  default = "chuaikan"
}

variable "primary_region" {
  default = "ap-southeast-1"
  description = "Singapore - Primary region"
}

variable "dr_region" {
  default = "ap-northeast-1"
  description = "Tokyo - DR region"
}

variable "domain_name" {
  default = "chuaikan.com"
}

variable "primary_vpc_cidr" {
  default = "10.0.0.0/16"
}

variable "dr_vpc_cidr" {
  default = "10.1.0.0/16"
}
```

### Step 753: VPC Setup สำหรับทั้งสอง Region

```hcl
# infra/terraform/multi-region/singapore/vpc.tf
module "vpc_singapore" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  providers = {
    aws = aws.singapore
  }

  name = "${var.project_name}-primary"
  cidr = var.primary_vpc_cidr

  azs             = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  database_subnets = ["10.0.201.0/24", "10.0.202.0/24", "10.0.203.0/24"]

  enable_nat_gateway     = true
  single_nat_gateway     = false
  one_nat_gateway_per_az = true
  enable_vpn_gateway     = false
  enable_dns_hostnames   = true
  enable_dns_support     = true

  tags = {
    Project     = var.project_name
    Environment = "production"
    Region      = "primary"
  }
}

# infra/terraform/multi-region/tokyo/vpc.tf
module "vpc_tokyo" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  providers = {
    aws = aws.tokyo
  }

  name = "${var.project_name}-dr"
  cidr = var.dr_vpc_cidr

  azs             = ["ap-northeast-1a", "ap-northeast-1b", "ap-northeast-1c"]
  private_subnets = ["10.1.1.0/24", "10.1.2.0/24", "10.1.3.0/24"]
  public_subnets  = ["10.1.101.0/24", "10.1.102.0/24", "10.1.103.0/24"]
  database_subnets = ["10.1.201.0/24", "10.1.202.0/24", "10.1.203.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = true  # DR: ประหยัดค่าใช้จ่าย
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Project     = var.project_name
    Environment = "production"
    Region      = "dr"
  }
}
```

### Step 754: Route 53 Latency-based Routing

```hcl
# infra/terraform/multi-region/global/route53.tf
data "aws_route53_zone" "main" {
  name = "chuaikan.com"
}

# Health check สำหรับ Singapore
resource "aws_route53_health_check" "singapore" {
  provider          = aws.global
  fqdn              = "sg.chuaikan.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30

  tags = {
    Name = "singapore-health-check"
  }
}

# Health check สำหรับ Tokyo
resource "aws_route53_health_check" "tokyo" {
  provider          = aws.global
  fqdn              = "jp.chuaikan.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30

  tags = {
    Name = "tokyo-health-check"
  }
}

# Latency-based record สำหรับ Singapore
resource "aws_route53_record" "api_singapore" {
  provider       = aws.global
  zone_id        = data.aws_route53_zone.main.zone_id
  name           = "api.chuaikan.com"
  type           = "A"
  set_identifier = "singapore"

  latency_routing_policy {
    region = "ap-southeast-1"
  }

  health_check_id = aws_route53_health_check.singapore.id

  alias {
    name                   = aws_lb.singapore.dns_name
    zone_id                = aws_lb.singapore.zone_id
    evaluate_target_health = true
  }
}

# Latency-based record สำหรับ Tokyo (DR)
resource "aws_route53_record" "api_tokyo" {
  provider       = aws.global
  zone_id        = data.aws_route53_zone.main.zone_id
  name           = "api.chuaikan.com"
  type           = "A"
  set_identifier = "tokyo"

  latency_routing_policy {
    region = "ap-northeast-1"
  }

  health_check_id = aws_route53_health_check.tokyo.id

  alias {
    name                   = aws_lb.tokyo.dns_name
    zone_id                = aws_lb.tokyo.zone_id
    evaluate_target_health = true
  }
}
```

### Step 755: PostgreSQL Logical Replication

```sql
-- บน Primary (Singapore) - postgresql.conf
-- แก้ไขไฟล์ /etc/postgresql/16/main/postgresql.conf
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
```

```bash
# บน Primary: สร้าง replication user
psql -U postgres << 'EOF'
CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'secure_replication_password_here';
GRANT USAGE ON SCHEMA public TO replicator;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO replicator;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO replicator;
EOF

# สร้าง publication สำหรับ tables ที่ต้องการ replicate
psql -U postgres -d chuaikan << 'EOF'
CREATE PUBLICATION chuaikan_pub FOR TABLE
  users,
  posts,
  comments,
  likes,
  follows,
  notifications,
  sos_alerts;

-- ตรวจสอบ publication
SELECT * FROM pg_publication;
SELECT * FROM pg_publication_tables;
EOF
```

```bash
# บน Replica (Tokyo): สร้าง subscription
psql -U postgres -d chuaikan << 'EOF'
-- ต้องมี schema เหมือน primary ก่อน (ใช้ pg_dump --schema-only)
CREATE SUBSCRIPTION chuaikan_sub
  CONNECTION 'host=sg.db.chuaikan.com port=5432 dbname=chuaikan user=replicator password=secure_replication_password_here sslmode=require'
  PUBLICATION chuaikan_pub
  WITH (copy_data = true, create_slot = true);

-- ตรวจสอบ subscription status
SELECT * FROM pg_stat_subscription;
EOF
```

```python
# scripts/monitor_replication.py
# ตรวจสอบ replication lag

import psycopg2
import boto3
import json
from datetime import datetime

def check_replication_lag():
    """ตรวจสอบ lag ของ logical replication"""
    
    conn_primary = psycopg2.connect(
        host="sg.db.chuaikan.com",
        database="chuaikan",
        user="monitor",
        password="monitor_password"
    )
    
    cur = conn_primary.cursor()
    
    # ตรวจ replication slot lag
    cur.execute("""
        SELECT
            slot_name,
            active,
            pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS lag_size,
            EXTRACT(EPOCH FROM (now() - pg_catalog.pg_last_xact_replay_timestamp())) AS lag_seconds
        FROM pg_replication_slots
        WHERE slot_name LIKE '%chuaikan%';
    """)
    
    results = cur.fetchall()
    
    cloudwatch = boto3.client('cloudwatch', region_name='ap-southeast-1')
    
    for row in results:
        slot_name, active, lag_size, lag_seconds = row
        
        print(f"Slot: {slot_name}, Active: {active}, Lag: {lag_size}, Lag seconds: {lag_seconds}")
        
        # ส่ง metric ไป CloudWatch
        cloudwatch.put_metric_data(
            Namespace='chuaikan/Replication',
            MetricData=[
                {
                    'MetricName': 'ReplicationLagSeconds',
                    'Dimensions': [{'Name': 'SlotName', 'Value': slot_name}],
                    'Value': float(lag_seconds or 0),
                    'Unit': 'Seconds',
                    'Timestamp': datetime.utcnow()
                }
            ]
        )
        
        # Alert ถ้า lag > 5 นาที
        if lag_seconds and lag_seconds > 300:
            print(f"ALERT: Replication lag {lag_seconds}s exceeds 5 minutes!")
    
    cur.close()
    conn_primary.close()

if __name__ == "__main__":
    check_replication_lag()
```

### Step 756: Redis Global Datastore (ElastiCache)

```hcl
# infra/terraform/multi-region/redis.tf

# Primary Redis Cluster (Singapore)
resource "aws_elasticache_replication_group" "primary" {
  provider                    = aws.singapore
  replication_group_id        = "chuaikan-redis-primary"
  description                 = "chuaikan primary Redis cluster"
  node_type                   = "cache.r7g.large"
  num_cache_clusters          = 3
  automatic_failover_enabled  = true
  multi_az_enabled            = true
  engine_version              = "7.1"
  port                        = 6379
  subnet_group_name           = aws_elasticache_subnet_group.singapore.name
  security_group_ids          = [aws_security_group.redis_sg.id]
  at_rest_encryption_enabled  = true
  transit_encryption_enabled  = true
  auth_token                  = var.redis_auth_token

  tags = {
    Name    = "chuaikan-redis-primary"
    Region  = "primary"
  }
}

# Global Datastore
resource "aws_elasticache_global_replication_group" "chuaikan" {
  global_replication_group_id_suffix = "chuaikan-global"
  primary_replication_group_id       = aws_elasticache_replication_group.primary.id
}

# Secondary Redis Cluster (Tokyo) - ดึง data จาก Global Datastore
resource "aws_elasticache_replication_group" "secondary" {
  provider                    = aws.tokyo
  replication_group_id        = "chuaikan-redis-dr"
  description                 = "chuaikan DR Redis cluster"
  global_replication_group_id = aws_elasticache_global_replication_group.chuaikan.id
  node_type                   = "cache.r7g.large"
  num_cache_clusters          = 2
  automatic_failover_enabled  = true
  subnet_group_name           = aws_elasticache_subnet_group.tokyo.name
  security_group_ids          = [aws_security_group.redis_sg_tokyo.id]

  tags = {
    Name    = "chuaikan-redis-dr"
    Region  = "dr"
  }
}
```

### Step 757: S3 Cross-Region Replication

```hcl
# infra/terraform/multi-region/s3.tf

# Primary S3 bucket (Singapore)
resource "aws_s3_bucket" "primary" {
  provider = aws.singapore
  bucket   = "chuaikan-assets-primary"
}

resource "aws_s3_bucket_versioning" "primary" {
  provider = aws.singapore
  bucket   = aws_s3_bucket.primary.id
  versioning_configuration {
    status = "Enabled"
  }
}

# DR S3 bucket (Tokyo)
resource "aws_s3_bucket" "dr" {
  provider = aws.tokyo
  bucket   = "chuaikan-assets-dr"
}

resource "aws_s3_bucket_versioning" "dr" {
  provider = aws.tokyo
  bucket   = aws_s3_bucket.dr.id
  versioning_configuration {
    status = "Enabled"
  }
}

# IAM Role สำหรับ S3 replication
resource "aws_iam_role" "s3_replication" {
  name = "s3-replication-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "s3.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy" "s3_replication" {
  name = "s3-replication-policy"
  role = aws_iam_role.s3_replication.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action   = ["s3:GetReplicationConfiguration", "s3:ListBucket"]
        Effect   = "Allow"
        Resource = aws_s3_bucket.primary.arn
      },
      {
        Action   = ["s3:GetObjectVersionForReplication", "s3:GetObjectVersionAcl", "s3:GetObjectVersionTagging"]
        Effect   = "Allow"
        Resource = "${aws_s3_bucket.primary.arn}/*"
      },
      {
        Action   = ["s3:ReplicateObject", "s3:ReplicateDelete", "s3:ReplicateTags"]
        Effect   = "Allow"
        Resource = "${aws_s3_bucket.dr.arn}/*"
      }
    ]
  })
}

# Replication configuration
resource "aws_s3_bucket_replication_configuration" "primary_to_dr" {
  provider = aws.singapore
  role     = aws_iam_role.s3_replication.arn
  bucket   = aws_s3_bucket.primary.id

  rule {
    id     = "replicate-all"
    status = "Enabled"

    filter {}

    destination {
      bucket        = aws_s3_bucket.dr.arn
      storage_class = "STANDARD_IA"  # ประหยัดค่าใช้จ่ายสำหรับ DR

      encryption_configuration {
        replica_kms_key_id = aws_kms_key.dr.arn
      }
    }

    delete_marker_replication {
      status = "Enabled"
    }
  }
}
```

### Step 758: Terraform Apply และ Verify

```bash
# Initialize Terraform
cd infra/terraform/multi-region
terraform init

# Plan
terraform plan -out=multi-region.tfplan

# Apply
terraform apply multi-region.tfplan

# ตรวจสอบ Route 53 records
aws route53 list-resource-record-sets \
  --hosted-zone-id $(aws route53 list-hosted-zones --query 'HostedZones[?Name==`chuaikan.com.`].Id' --output text | cut -d/ -f3) \
  --query 'ResourceRecordSets[?Name==`api.chuaikan.com.`]'

# ทดสอบ latency routing
dig api.chuaikan.com
# ควรได้ IP ของ Singapore ALB

# ทดสอบจาก Tokyo (simulate)
dig @8.8.8.8 api.chuaikan.com
```

---

## 📋 Runbook: Failover to DR Region

### เมื่อ Primary (Singapore) ล้มเหลว

```bash
#!/bin/bash
# scripts/failover-to-dr.sh
set -euo pipefail

PRIMARY_REGION="ap-southeast-1"
DR_REGION="ap-northeast-1"
DOMAIN="chuaikan.com"
HOSTED_ZONE_ID="Z0123456789EXAMPLE"

echo "=== Starting Failover to DR Region (Tokyo) ==="
echo "Time: $(date -u)"

# Step 1: ตรวจสอบว่า Primary จริงๆ ล้มเหลว
echo "Step 1: Verifying Primary Region failure..."
if curl -sf --max-time 10 "https://sg.${DOMAIN}/health" > /dev/null 2>&1; then
    echo "ERROR: Primary region is still responding. Aborting failover."
    exit 1
fi
echo "Confirmed: Primary region is down."

# Step 2: หยุด replication (ป้องกันข้อมูลถูก overwrite)
echo "Step 2: Stopping replication..."
psql "host=jp.db.${DOMAIN} user=postgres" << 'EOF'
-- บน DR: Drop subscription เพื่อ promote เป็น standalone
ALTER SUBSCRIPTION chuaikan_sub DISABLE;
DROP SUBSCRIPTION chuaikan_sub;
EOF

# Step 3: Route traffic ไป DR
echo "Step 3: Routing traffic to DR region..."
aws route53 change-resource-record-sets \
  --hosted-zone-id ${HOSTED_ZONE_ID} \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.chuaikan.com",
        "Type": "A",
        "SetIdentifier": "singapore",
        "LatencyRoutingPolicy": {"Region": "ap-southeast-1"},
        "HealthCheckId": "DISABLED",
        "TTL": 60,
        "ResourceRecords": [{"Value": "203.0.113.0"}]
      }
    }]
  }'

# Step 4: Scale up DR region
echo "Step 4: Scaling up DR region..."
aws eks update-nodegroup-config \
  --cluster-name chuaikan-dr \
  --nodegroup-name main \
  --scaling-config minSize=3,maxSize=20,desiredSize=10 \
  --region ${DR_REGION}

# Step 5: ตรวจสอบ DR health
echo "Step 5: Verifying DR region health..."
sleep 60
if curl -sf "https://jp.${DOMAIN}/health" > /dev/null 2>&1; then
    echo "SUCCESS: DR region is serving traffic."
else
    echo "WARNING: DR region health check failed. Check manually."
fi

echo "=== Failover Complete ==="
echo "Remember to: 1) Notify team, 2) Fix primary, 3) Failback when ready"
```

---

## ✅ Checklist

- [ ] Terraform code สำหรับ multi-region infrastructure เขียนเสร็จ
- [ ] VPC สร้างแล้วทั้ง Singapore และ Tokyo
- [ ] Route 53 latency-based routing ทำงานถูกต้อง
- [ ] Health checks ตรวจสอบ `/health` endpoint ได้
- [ ] PostgreSQL logical replication ทำงาน (lag < 30 วินาที)
- [ ] Redis Global Datastore sync ระหว่าง regions
- [ ] S3 cross-region replication เปิดใช้งาน
- [ ] Failover runbook ทดสอบแล้ว (quarterly drill)
- [ ] Monitoring dashboard แสดง replication lag
- [ ] CloudWatch alarms ตั้งไว้สำหรับ replication lag > 5 นาที
- [ ] RPO ≤ 5 นาที (ทดสอบแล้ว)
- [ ] RTO ≤ 30 นาที (ทดสอบแล้ว)

---

## 🔗 References

- [AWS Multi-Region Architecture](https://aws.amazon.com/architecture/multi-region/)
- [PostgreSQL Logical Replication](https://www.postgresql.org/docs/current/logical-replication.html)
- [ElastiCache Global Datastore](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Redis-Global-Datastore.html)
- [Route 53 Routing Policies](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html)

---
*Part 076 | Road to 1,000,000 Users/Day | chuaikan.com*
