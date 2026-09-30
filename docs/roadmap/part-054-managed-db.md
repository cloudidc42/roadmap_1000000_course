# Part 054: RDS / Cloud SQL (Managed Database)
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Advanced
> **Steps:** 531-540
> **เวลาโดยประมาณ:** 5 ชั่วโมง
> **Prerequisites:** Part 052 (VPC Network), Part 053 (EKS)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ตั้งค่า RDS PostgreSQL 17 (db.r6g.xlarge, Multi-AZ)
- ปรับแต่ง Parameter Group สำหรับ performance
- ตั้งค่า RDS Proxy สำหรับ connection pooling
- ตั้งค่า Automated Backup และ PITR
- สร้าง Read Replicas สำหรับ scaling
- เขียน Terraform code สำหรับ RDS
- ตั้งค่า RDS Monitoring
- ประหยัดค่าใช้จ่ายด้วย Reserved Instances

---

## 📖 ทฤษฎีและแนวคิด

### Step 531: ทำไมต้องใช้ RDS

**RDS (Relational Database Service)** คือ Managed Database ที่ AWS ดูแลให้:

```
ถ้าใช้ PostgreSQL บน EC2 เอง:              ถ้าใช้ RDS:
├── ติดตั้ง PostgreSQL เอง                 ├── AWS ติดตั้งให้
├── ตั้งค่า replication เอง                ├── Multi-AZ built-in
├── backup เอง                             ├── Automated backup
├── upgrade version เอง                   ├── Minor version auto-update
├── monitoring เอง                         ├── CloudWatch + Performance Insights
└── failover เอง (downtime)                └── Automatic failover <60s
```

**Multi-AZ Architecture:**

```
Primary DB (AZ: 1a)     Standby DB (AZ: 1b)
10.0.20.x               10.0.21.x
┌──────────────┐         ┌──────────────┐
│  PostgreSQL  │ ──────► │  PostgreSQL  │
│  (Active)    │  sync   │  (Standby)   │
└──────────────┘         └──────────────┘
       │                        │
       └────────────────────────┘
              Auto Failover
              (DNS flips, <60s)
```

### Step 532: Instance Type Selection

```
db.t3.medium  (2 vCPU, 4GB)   - Dev/Test: $0.068/hr
db.r6g.large  (2 vCPU, 16GB)  - Small prod: $0.194/hr  
db.r6g.xlarge (4 vCPU, 32GB)  - Medium prod: $0.388/hr ← เลือกนี้
db.r6g.2xlarge(8 vCPU, 64GB)  - Large prod: $0.776/hr
db.r6g.4xlarge(16 vCPU, 128GB)- Enterprise: $1.552/hr
```

**สำหรับ chuaikan.com ที่ 100k users/day**: db.r6g.xlarge Multi-AZ

### Step 533: Read Replica Architecture

```
Write Traffic → Primary DB (Master)
                    │
                    ├── Sync replication → Standby (Multi-AZ)
                    │
                    ├── Async replication → Read Replica 1 (AZ: 1a)
                    └── Async replication → Read Replica 2 (AZ: 1b)

Application Code:
- Write queries → primary endpoint
- Read queries → reader endpoint (load balanced across replicas)
```

---

## ⚙️ Environment Setup

### ตั้งค่า Terraform Variables

```bash
# terraform.tfvars
cat > terraform/environments/prod/terraform.tfvars << 'EOF'
aws_region   = "ap-southeast-1"
environment  = "production"
project_name = "chuaikan"

# RDS Settings
db_instance_class    = "db.r6g.xlarge"
db_engine_version    = "17.2"
db_allocated_storage = 100
db_max_allocated_storage = 500
db_multi_az          = true
db_backup_retention  = 7
db_name              = "chuaikan"
db_username          = "chuaikan_admin"
EOF
```

---

## 🛠️ Step-by-Step Implementation

### Step 534: สร้าง DB Subnet Group

```bash
# สร้าง DB Subnet Group
aws rds create-db-subnet-group \
  --db-subnet-group-name chuaikan-db-subnet-group \
  --db-subnet-group-description "Subnet group for chuaikan RDS" \
  --subnet-ids subnet-xxxxxx subnet-yyyyyy \
  --tags Key=Name,Value=chuaikan-db-subnet-group
```

### Step 535: สร้าง RDS Parameter Group

```bash
# สร้าง Parameter Group
aws rds create-db-parameter-group \
  --db-parameter-group-name chuaikan-pg17-params \
  --db-parameter-group-family postgres17 \
  --description "PostgreSQL 17 params for chuaikan"

# ปรับแต่ง Parameters สำหรับ performance
aws rds modify-db-parameter-group \
  --db-parameter-group-name chuaikan-pg17-params \
  --parameters \
    ParameterName=shared_buffers,ParameterValue="{DBInstanceClassMemory/4}",ApplyMethod=pending-reboot \
    ParameterName=effective_cache_size,ParameterValue="{DBInstanceClassMemory*3/4}",ApplyMethod=pending-reboot \
    ParameterName=max_connections,ParameterValue=200,ApplyMethod=pending-reboot \
    ParameterName=wal_buffers,ParameterValue=16384,ApplyMethod=pending-reboot \
    ParameterName=checkpoint_completion_target,ParameterValue=0.9,ApplyMethod=immediate \
    ParameterName=random_page_cost,ParameterValue=1.1,ApplyMethod=immediate \
    ParameterName=effective_io_concurrency,ParameterValue=200,ApplyMethod=immediate \
    ParameterName=work_mem,ParameterValue=4096,ApplyMethod=immediate \
    ParameterName=maintenance_work_mem,ParameterValue=65536,ApplyMethod=immediate \
    ParameterName=log_min_duration_statement,ParameterValue=1000,ApplyMethod=immediate \
    ParameterName=log_connections,ParameterValue=1,ApplyMethod=immediate \
    ParameterName=log_disconnections,ParameterValue=1,ApplyMethod=immediate \
    ParameterName=pg_stat_statements.track,ParameterValue=all,ApplyMethod=immediate
```

### Step 536: สร้าง RDS Instance

```bash
# สร้าง Master Password ใน Secrets Manager
aws secretsmanager create-secret \
  --name chuaikan/rds/master-password \
  --description "RDS master password for chuaikan" \
  --secret-string '{"username":"chuaikan_admin","password":"SuperSecure@Password123!"}'

# สร้าง RDS Instance
aws rds create-db-instance \
  --db-instance-identifier chuaikan-prod-db \
  --db-instance-class db.r6g.xlarge \
  --engine postgres \
  --engine-version "17.2" \
  --master-username chuaikan_admin \
  --master-user-password "SuperSecure@Password123!" \
  --db-name chuaikan \
  --db-subnet-group-name chuaikan-db-subnet-group \
  --vpc-security-group-ids sg-xxxxxxxxx \
  --db-parameter-group-name chuaikan-pg17-params \
  --allocated-storage 100 \
  --max-allocated-storage 500 \
  --storage-type gp3 \
  --iops 3000 \
  --storage-throughput 125 \
  --storage-encrypted \
  --kms-key-id alias/aws/rds \
  --multi-az \
  --backup-retention-period 7 \
  --preferred-backup-window "17:00-18:00" \
  --preferred-maintenance-window "sun:18:00-sun:19:00" \
  --deletion-protection \
  --enable-performance-insights \
  --performance-insights-retention-period 7 \
  --monitoring-interval 60 \
  --monitoring-role-arn arn:aws:iam::123456789012:role/rds-monitoring-role \
  --copy-tags-to-snapshot \
  --tags Key=Environment,Value=production Key=Project,Value=chuaikan

# รอ RDS พร้อม (ใช้เวลาประมาณ 10-15 นาที)
aws rds wait db-instance-available --db-instance-identifier chuaikan-prod-db
echo "RDS is ready!"
```

### Step 537: สร้าง Read Replicas

```bash
# สร้าง Read Replica 1 (AZ: 1a)
aws rds create-db-instance-read-replica \
  --db-instance-identifier chuaikan-prod-db-replica-1 \
  --source-db-instance-identifier chuaikan-prod-db \
  --db-instance-class db.r6g.large \
  --availability-zone ap-southeast-1a \
  --db-parameter-group-name chuaikan-pg17-params \
  --tags Key=Environment,Value=production Key=Role,Value=replica

# สร้าง Read Replica 2 (AZ: 1b)
aws rds create-db-instance-read-replica \
  --db-instance-identifier chuaikan-prod-db-replica-2 \
  --source-db-instance-identifier chuaikan-prod-db \
  --db-instance-class db.r6g.large \
  --availability-zone ap-southeast-1b \
  --db-parameter-group-name chuaikan-pg17-params \
  --tags Key=Environment,Value=production Key=Role,Value=replica
```

### Step 538: ตั้งค่า RDS Proxy

```bash
# สร้าง RDS Proxy (ช่วย connection pooling)
aws rds create-db-proxy \
  --db-proxy-name chuaikan-rds-proxy \
  --engine-family POSTGRESQL \
  --auth '[{
    "AuthScheme": "SECRETS",
    "SecretArn": "arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:chuaikan/rds/master-password",
    "IAMAuth": "DISABLED"
  }]' \
  --role-arn arn:aws:iam::123456789012:role/rds-proxy-role \
  --vpc-subnet-ids subnet-xxxxxx subnet-yyyyyy \
  --vpc-security-group-ids sg-xxxxxxxxx \
  --tags Key=Environment,Value=production

# Associate RDS Proxy กับ RDS Instance
aws rds register-db-proxy-targets \
  --db-proxy-name chuaikan-rds-proxy \
  --db-instance-identifiers chuaikan-prod-db \
  --target-group-name default
```

---

## 🔧 Configuration Files

### Terraform RDS Module (terraform/modules/rds/main.tf)

```hcl
# DB Subnet Group
resource "aws_db_subnet_group" "main" {
  name        = "${var.project}-${var.environment}-db-subnet-group"
  description = "DB subnet group for ${var.project}"
  subnet_ids  = var.database_subnet_ids

  tags = merge(var.tags, {
    Name = "${var.project}-${var.environment}-db-subnet-group"
  })
}

# DB Parameter Group
resource "aws_db_parameter_group" "main" {
  name        = "${var.project}-${var.environment}-pg17"
  family      = "postgres17"
  description = "PostgreSQL 17 parameters for ${var.project}"

  parameter {
    name         = "shared_buffers"
    value        = "{DBInstanceClassMemory/4}"
    apply_method = "pending-reboot"
  }

  parameter {
    name         = "effective_cache_size"
    value        = "{DBInstanceClassMemory*3/4}"
    apply_method = "pending-reboot"
  }

  parameter {
    name         = "max_connections"
    value        = "200"
    apply_method = "pending-reboot"
  }

  parameter {
    name         = "log_min_duration_statement"
    value        = "1000"
    apply_method = "immediate"
  }

  parameter {
    name         = "work_mem"
    value        = "4096"
    apply_method = "immediate"
  }

  parameter {
    name         = "random_page_cost"
    value        = "1.1"
    apply_method = "immediate"
  }

  tags = var.tags
}

# IAM Role for Enhanced Monitoring
resource "aws_iam_role" "rds_monitoring" {
  name = "${var.project}-${var.environment}-rds-monitoring"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "monitoring.rds.amazonaws.com"
      }
    }]
  })

  managed_policy_arns = [
    "arn:aws:iam::aws:policy/service-role/AmazonRDSEnhancedMonitoringRole"
  ]

  tags = var.tags
}

# RDS Instance
resource "aws_db_instance" "main" {
  identifier = "${var.project}-${var.environment}-db"

  engine         = "postgres"
  engine_version = var.db_engine_version
  instance_class = var.db_instance_class

  db_name  = var.db_name
  username = var.db_username
  password = random_password.db_password.result

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [var.database_security_group_id]
  parameter_group_name   = aws_db_parameter_group.main.name

  allocated_storage     = var.db_allocated_storage
  max_allocated_storage = var.db_max_allocated_storage
  storage_type          = "gp3"
  storage_encrypted     = true

  multi_az               = var.db_multi_az
  availability_zone      = var.db_multi_az ? null : "ap-southeast-1a"
  publicly_accessible    = false

  backup_retention_period   = var.db_backup_retention
  backup_window             = "17:00-18:00"
  maintenance_window        = "sun:18:00-sun:19:00"
  delete_automated_backups  = false

  deletion_protection = var.environment == "production"
  skip_final_snapshot = var.environment != "production"
  final_snapshot_identifier = var.environment == "production" ? "${var.project}-${var.environment}-final-snapshot" : null

  performance_insights_enabled          = true
  performance_insights_retention_period = 7

  monitoring_interval = 60
  monitoring_role_arn = aws_iam_role.rds_monitoring.arn

  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]

  copy_tags_to_snapshot = true

  tags = merge(var.tags, {
    Name = "${var.project}-${var.environment}-db"
  })
}

# Read Replicas
resource "aws_db_instance" "replica" {
  count = var.read_replica_count

  identifier = "${var.project}-${var.environment}-db-replica-${count.index + 1}"

  replicate_source_db = aws_db_instance.main.identifier

  instance_class = var.replica_instance_class
  
  availability_zone = var.replica_azs[count.index]
  
  parameter_group_name = aws_db_parameter_group.main.name

  performance_insights_enabled = true
  monitoring_interval          = 60
  monitoring_role_arn          = aws_iam_role.rds_monitoring.arn

  auto_minor_version_upgrade = true

  tags = merge(var.tags, {
    Name = "${var.project}-${var.environment}-db-replica-${count.index + 1}"
    Role = "replica"
  })
}

# Store password in Secrets Manager
resource "random_password" "db_password" {
  length           = 32
  special          = true
  override_special = "!#$%^&*()-_=+[]{}<>:?"
}

resource "aws_secretsmanager_secret" "db_password" {
  name                    = "${var.project}/${var.environment}/rds/master-password"
  description             = "RDS master password for ${var.project}"
  recovery_window_in_days = 7

  tags = var.tags
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id = aws_secretsmanager_secret.db_password.id
  secret_string = jsonencode({
    username = var.db_username
    password = random_password.db_password.result
    host     = aws_db_instance.main.address
    port     = aws_db_instance.main.port
    dbname   = var.db_name
  })
}

# RDS Proxy (สำหรับ connection pooling)
resource "aws_db_proxy" "main" {
  name                   = "${var.project}-${var.environment}-rds-proxy"
  debug_logging          = false
  engine_family          = "POSTGRESQL"
  idle_client_timeout    = 1800
  require_tls            = true
  role_arn               = aws_iam_role.rds_proxy.arn
  vpc_security_group_ids = [var.database_security_group_id]
  vpc_subnet_ids         = var.database_subnet_ids

  auth {
    auth_scheme = "SECRETS"
    description = "RDS master credentials"
    iam_auth    = "DISABLED"
    secret_arn  = aws_secretsmanager_secret.db_password.arn
  }

  tags = var.tags
}

resource "aws_db_proxy_default_target_group" "main" {
  db_proxy_name = aws_db_proxy.main.name

  connection_pool_config {
    connection_borrow_timeout    = 120
    max_connections_percent      = 100
    max_idle_connections_percent = 50
  }
}

resource "aws_db_proxy_target" "main" {
  db_instance_identifier = aws_db_instance.main.identifier
  db_proxy_name          = aws_db_proxy.main.name
  target_group_name      = aws_db_proxy_default_target_group.main.name
}

resource "aws_iam_role" "rds_proxy" {
  name = "${var.project}-${var.environment}-rds-proxy-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "rds.amazonaws.com"
      }
    }]
  })

  tags = var.tags
}

resource "aws_iam_role_policy" "rds_proxy" {
  name = "${var.project}-${var.environment}-rds-proxy-policy"
  role = aws_iam_role.rds_proxy.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ]
      Resource = [aws_secretsmanager_secret.db_password.arn]
    }]
  })
}
```

### Node.js Database Connection (src/db/connection.ts)

```typescript
import { Pool, PoolConfig } from 'pg';
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';

interface DBSecret {
  username: string;
  password: string;
  host: string;
  port: number;
  dbname: string;
}

async function getDBSecret(): Promise<DBSecret> {
  const client = new SecretsManagerClient({ region: 'ap-southeast-1' });
  const command = new GetSecretValueCommand({
    SecretId: process.env.DB_SECRET_ARN || 'chuaikan/production/rds/master-password'
  });
  
  const response = await client.send(command);
  return JSON.parse(response.SecretString!);
}

async function createPool(): Promise<Pool> {
  const secret = await getDBSecret();
  
  const config: PoolConfig = {
    host: process.env.DB_PROXY_HOST || secret.host,  // ใช้ RDS Proxy host
    port: secret.port,
    database: secret.dbname,
    user: secret.username,
    password: secret.password,
    max: 20,                    // max connections
    min: 5,                     // min connections
    idleTimeoutMillis: 30000,
    connectionTimeoutMillis: 5000,
    ssl: { rejectUnauthorized: false }
  };

  return new Pool(config);
}

// Write pool (Master)
export const writePool = createPool();

// Read pool (Read Replica)
export const readPool = (() => {
  const readConfig: PoolConfig = {
    host: process.env.DB_READER_HOST,  // RDS Reader endpoint
    // ... same as write pool
  };
  return new Pool(readConfig);
})();

// Helper functions
export async function query(text: string, params?: any[]) {
  const pool = await writePool;
  const start = Date.now();
  const result = await pool.query(text, params);
  const duration = Date.now() - start;
  
  if (duration > 1000) {
    console.warn('Slow query', { text, duration });
  }
  
  return result;
}

export async function readQuery(text: string, params?: any[]) {
  const pool = readPool;
  return pool.query(text, params);
}
```

---

## 🧪 Testing

```bash
# ทดสอบ RDS connection
aws rds describe-db-instances \
  --db-instance-identifier chuaikan-prod-db \
  --query 'DBInstances[0].{Status:DBInstanceStatus,Endpoint:Endpoint.Address,AZ:AvailabilityZone,MultiAZ:MultiAZ}'

# Test connectivity จาก EKS pod
kubectl run db-test --image=postgres:17 -n production --rm -it -- \
  psql -h chuaikan-prod-db.xxxxxxxx.ap-southeast-1.rds.amazonaws.com \
       -U chuaikan_admin -d chuaikan -c "SELECT version();"

# ดู Performance Insights
aws pi get-resource-metrics \
  --service-type RDS \
  --identifier db:chuaikan-prod-db \
  --metric-queries '[{"Metric": "db.load.avg"}]' \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period-in-seconds 60

# Test PITR (Point-in-Time Recovery)
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier chuaikan-prod-db \
  --target-db-instance-identifier chuaikan-prod-db-restored \
  --restore-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --db-instance-class db.t3.medium  # ใช้ instance เล็กกว่าสำหรับทดสอบ

# ดู Slow Query Logs
aws logs filter-log-events \
  --log-group-name /aws/rds/instance/chuaikan-prod-db/postgresql \
  --filter-pattern "duration" \
  --start-time $(date -u -d '1 hour ago' +%s000)
```

---

## ❌ Common Errors & Solutions

### Error 1: "too many connections"

```
FATAL: remaining connection slots are reserved for non-replication superuser connections
```

**แก้ไข:**
```bash
# ตรวจสอบ connection count
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=chuaikan-prod-db \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 --statistics Average

# ใช้ RDS Proxy ลด connections
# หรือเพิ่ม max_connections parameter
```

### Error 2: Read Replica lag สูง

```
ReplicaLag: 30 seconds
```

**แก้ไข:**
```bash
# Monitor replica lag
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name ReplicaLag \
  --dimensions Name=DBInstanceIdentifier,Value=chuaikan-prod-db-replica-1 \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 --statistics Average

# อาจต้อง upgrade instance class ของ replica
```

---

## 📊 Cost Estimation

```
RDS Multi-AZ (db.r6g.xlarge):
├── On-demand: $0.776/hr × 730h = $567/month
├── 1-year Reserved: $0.407/hr × 730h = $297/month (ประหยัด 48%)
└── 3-year Reserved: $0.254/hr × 730h = $185/month (ประหยัด 67%)

Storage (100GB gp3): $0.138/GB × 100 = $14/month

Read Replicas (2 × db.r6g.large):
├── On-demand: $0.388/hr × 2 × 730h = $567/month
└── 1-year Reserved: $0.204/hr × 2 × 730h = $298/month

RDS Proxy: $0.015/vCPU/hr × 4 × 730h = $43.8/month

TOTAL (on-demand): ~$1,192/month
TOTAL (1-yr reserved): ~$653/month (ประหยัด 45%)
```

---

## ✅ Checklist

- [ ] Step 531: เข้าใจ RDS Multi-AZ Architecture
- [ ] Step 532: เลือก Instance Type ที่เหมาะสม
- [ ] Step 533: วางแผน Read Replica Architecture
- [ ] Step 534: สร้าง DB Subnet Group
- [ ] Step 535: สร้างและปรับแต่ง Parameter Group
- [ ] Step 536: สร้าง RDS Instance (Multi-AZ)
- [ ] Step 537: สร้าง Read Replicas (2 replicas)
- [ ] Step 538: ตั้งค่า RDS Proxy
- [ ] Step 539: ตั้งค่า Performance Insights และ Monitoring
- [ ] Step 540: ทดสอบ failover และ PITR

---

## 🔗 References

- [RDS PostgreSQL User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_PostgreSQL.html)
- [RDS Proxy](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.html)
- [Performance Insights](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PerfInsights.html)
- [RDS Reserved Instances](https://aws.amazon.com/rds/reserved-instances/)

---
*Part 054 | Road to 1,000,000 Users/Day | chuaikan.com*
