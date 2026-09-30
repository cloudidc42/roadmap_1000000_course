# Part 051: AWS/GCP Architecture Overview
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Advanced
> **Steps:** 501-510
> **เวลาโดยประมาณ:** 4 ชั่วโมง
> **Prerequisites:** Part 001-050 (Linux, Docker, Kubernetes, Database, Redis, CI/CD)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- เปรียบเทียบ AWS vs GCP สำหรับ chuaikan.com ในแง่ latency ไปยัง Thailand, pricing และ services
- เลือก Region ที่เหมาะสม: AWS ap-southeast-1 (Singapore)
- วางแผน Multi-region strategy: Primary Singapore + Backup Tokyo
- รู้จัก AWS services ที่จำเป็น: EC2/EKS, RDS, ElastiCache, S3, CloudFront, SQS, Lambda
- ประมาณค่าใช้จ่ายสำหรับ 100k users/day
- ตั้งค่า AWS Account, Organizations, IAM, Billing Alerts
- ติดตั้งและตั้งค่า AWS CLI

---

## 📖 ทฤษฎีและแนวคิด

### Step 501: ทำไม Cloud Computing ถึงสำคัญ

เมื่อ chuaikan.com เติบโตถึง 100,000 users/day การจัดการ infrastructure แบบ on-premise จะกลายเป็นปัญหาใหญ่:

- **Scaling ยาก**: ต้องซื้อ server ล่วงหน้า ใช้เวลาเป็นสัปดาห์
- **High Availability ต้องทำเอง**: ต้องมี datacenter backup
- **ค่าใช้จ่ายสูง**: ต้องจ่ายทั้ง hardware + network + electricity + staff

Cloud computing แก้ปัญหาเหล่านี้ด้วย:
- **Elasticity**: scale up/down ได้ภายใน minutes
- **Pay-as-you-go**: จ่ายตามที่ใช้จริง
- **Managed Services**: ไม่ต้องดูแล infrastructure เอง
- **Global Infrastructure**: deploy ใกล้ user ทั่วโลก

### Step 502: AWS vs GCP เปรียบเทียบสำหรับ chuaikan.com

#### Latency จาก Thailand

```
AWS ap-southeast-1 (Singapore):   ~10-20ms
AWS ap-southeast-2 (Sydney):      ~80-120ms  
AWS ap-northeast-1 (Tokyo):       ~50-70ms
GCP asia-southeast1 (Singapore):  ~12-22ms
GCP asia-southeast2 (Jakarta):    ~20-35ms
GCP asia-east1 (Taiwan):          ~40-60ms
```

ทั้ง AWS Singapore และ GCP Singapore ให้ latency ใกล้เคียงกัน แต่ AWS มีความได้เปรียบในแง่:

| Feature | AWS | GCP |
|---------|-----|-----|
| Market Share | ~33% (leader) | ~11% |
| Bangkok Developer Community | ใหญ่กว่า | เล็กกว่า |
| Thai Baht Billing | ไม่มี (USD) | ไม่มี (USD) |
| Free Tier | 12 เดือน | 12 เดือน |
| Spot/Preemptible Instances | Spot Instances | Preemptible (cheaper) |
| Kubernetes | EKS | GKE (better UX) |
| Database | RDS | Cloud SQL |
| Serverless | Lambda (mature) | Cloud Functions |
| CDN | CloudFront | Cloud CDN |
| Object Storage | S3 (standard) | GCS |
| Egress Cost | แพงกว่า | แพงกว่า (Cloudflare R2 แทนได้) |

**สรุป: เลือก AWS ap-southeast-1 (Singapore)** เพราะ:
1. Community ใหญ่กว่า หา help ได้ง่ายกว่า
2. Services ครบครันกว่า โดยเฉพาะ Lambda, SQS
3. EKS + RDS + ElastiCache integration ดีกว่า
4. Documentation ภาษาไทยมีมากกว่า

### Step 503: Multi-Region Architecture

```
Primary Region: ap-southeast-1 (Singapore)
├── AZ: ap-southeast-1a
├── AZ: ap-southeast-1b  
└── AZ: ap-southeast-1c

Backup Region: ap-northeast-1 (Tokyo)
├── S3 Cross-Region Replication
├── RDS Read Replica (standby)
└── Route 53 Failover Routing

CDN Edge: CloudFront (45+ Thailand PoPs)
```

### Step 504: AWS Services ที่จำเป็นสำหรับ chuaikan.com

```
Compute Layer:
├── EKS (Elastic Kubernetes Service) - container orchestration
├── EC2 Auto Scaling - worker nodes
└── Lambda - serverless functions

Storage Layer:
├── S3 - object storage (images, videos, backups)
├── EBS - block storage for EC2/EKS
└── EFS - shared filesystem

Database Layer:
├── RDS PostgreSQL - primary database
├── ElastiCache Redis - caching + sessions
└── DynamoDB - high-speed key-value (optional)

Network Layer:
├── VPC - isolated network
├── ALB - Application Load Balancer
├── CloudFront - CDN
└── Route 53 - DNS

Messaging Layer:
├── SQS - message queue
├── SNS - notification service
└── EventBridge - event bus

Security Layer:
├── IAM - identity and access management
├── WAF - web application firewall
├── Shield - DDoS protection
└── Secrets Manager - secrets storage

Monitoring Layer:
├── CloudWatch - metrics and logs
├── X-Ray - distributed tracing
└── CloudTrail - audit logs
```

---

## ⚙️ Environment Setup

### Step 505: AWS Account Setup

#### 1. สร้าง AWS Account หลัก (Root Account)

```bash
# ไปที่ https://aws.amazon.com/
# คลิก "Create an AWS Account"
# ใส่ email, password, account name: "chuaikan-root"
# ใส่ credit card (จะไม่ถูกชาร์จถ้าอยู่ใน free tier)
```

#### 2. ตั้งค่า Root Account Security

```bash
# Enable MFA for root account
# ไปที่ IAM > Security credentials > Assign MFA device
# เลือก Virtual MFA (Google Authenticator)

# อย่าใช้ Root Account ในการทำงานประจำวัน
# Root Account ใช้แค่:
# - ตั้งค่า billing
# - ปิด account
# - กู้คืน IAM admin
```

#### 3. สร้าง IAM Admin User

```bash
# ไปที่ IAM > Users > Create user
# Username: admin
# Access type: AWS Management Console access
# Attach policy: AdministratorAccess

# สร้าง Access Key สำหรับ CLI
# IAM > Users > admin > Security credentials > Create access key
```

### Step 506: ติดตั้ง AWS CLI

```bash
# Linux/WSL
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# macOS
brew install awscli

# Windows
# ดาวน์โหลด https://awscli.amazonaws.com/AWSCLIV2.msi

# ตรวจสอบ version
aws --version
# aws-cli/2.15.0 Python/3.11.6 Linux/6.1.0 exe/x86_64.ubuntu.22
```

---

## 🛠️ Step-by-Step Implementation

### Step 507: ตั้งค่า AWS CLI

```bash
# ตั้งค่า default profile
aws configure
# AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name [None]: ap-southeast-1
# Default output format [None]: json

# ตั้งค่า named profile สำหรับแต่ละ environment
aws configure --profile chuaikan-dev
aws configure --profile chuaikan-prod

# ตรวจสอบ identity
aws sts get-caller-identity
# {
#   "UserId": "AIDAIOSFODNN7EXAMPLE",
#   "Account": "123456789012",
#   "Arn": "arn:aws:iam::123456789012:user/admin"
# }

# ใช้ profile ที่ต้องการ
export AWS_PROFILE=chuaikan-prod
aws s3 ls
```

### Step 508: ตั้งค่า AWS Organizations

```bash
# สร้าง Organization (ให้ใช้หลาย accounts แยก billing)
aws organizations create-organization --feature-set ALL

# สร้าง Organizational Units (OUs)
aws organizations create-organizational-unit \
  --parent-id r-xxxx \
  --name "Production"

aws organizations create-organizational-unit \
  --parent-id r-xxxx \
  --name "Development"

# สร้าง child accounts
aws organizations create-account \
  --email "chuaikan-prod@example.com" \
  --account-name "chuaikan-production"

aws organizations create-account \
  --email "chuaikan-dev@example.com" \
  --account-name "chuaikan-development"
```

### Step 509: ตั้งค่า Billing Alerts

```bash
# เปิด Billing Alerts ใน Billing preferences
aws ce put-anomaly-monitor \
  --anomaly-monitor '{
    "MonitorName": "chuaikan-cost-monitor",
    "MonitorType": "DIMENSIONAL",
    "MonitorDimension": "SERVICE"
  }'

# สร้าง Budget Alert เมื่อค่าใช้จ่ายเกิน $500/month
aws budgets create-budget \
  --account-id 123456789012 \
  --budget '{
    "BudgetName": "chuaikan-monthly-budget",
    "BudgetLimit": {
      "Amount": "500",
      "Unit": "USD"
    },
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST"
  }' \
  --notifications-with-subscribers '[
    {
      "Notification": {
        "NotificationType": "ACTUAL",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 80,
        "ThresholdType": "PERCENTAGE",
        "NotificationState": "ALARM"
      },
      "Subscribers": [
        {
          "SubscriptionType": "EMAIL",
          "Address": "admin@chuaikan.com"
        }
      ]
    }
  ]'
```

### Step 510: ตั้งค่า IAM Policies และ Roles

```bash
# สร้าง IAM Policy สำหรับ EKS node
cat > eks-node-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:GetRepositoryPolicy",
        "ecr:DescribeRepositories",
        "ecr:ListImages",
        "ecr:BatchGetImage",
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name ChuaikanEKSNodePolicy \
  --policy-document file://eks-node-policy.json

# สร้าง IAM Role สำหรับ developers
cat > dev-role-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name ChuaikanDeveloper \
  --assume-role-policy-document file://dev-role-policy.json
```

---

## 🔧 Configuration Files

### AWS CLI Config (~/.aws/config)

```ini
[default]
region = ap-southeast-1
output = json

[profile chuaikan-dev]
region = ap-southeast-1
output = json
role_arn = arn:aws:iam::111111111111:role/ChuaikanDeveloper
source_profile = default

[profile chuaikan-prod]
region = ap-southeast-1
output = json
role_arn = arn:aws:iam::222222222222:role/ChuaikanAdmin
source_profile = default
mfa_serial = arn:aws:iam::123456789012:mfa/admin
```

### AWS Credentials (~/.aws/credentials)

```ini
[default]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

---

## 📊 Cost Estimation

### 100,000 Users/Day บน AWS ap-southeast-1

```
EKS:
├── EKS Control Plane:        $0.10/hour × 730h = $73/month
├── EC2 t3.large (4 nodes):   $0.0928/h × 4 × 730h = $271/month
└── Total EKS:                ~$344/month

RDS PostgreSQL:
├── db.r6g.large Multi-AZ:    $0.285/h × 730h = $208/month  
├── Storage 100GB gp3:        $0.138/GB × 100 = $14/month
└── Total RDS:                ~$222/month

ElastiCache Redis:
├── cache.r6g.large (2 nodes):$0.166/h × 2 × 730h = $242/month
└── Total Redis:              ~$242/month

S3 + CloudFront:
├── S3 Storage 1TB:           $0.023/GB × 1000 = $23/month
├── CloudFront Data 5TB:      $0.085/GB × 5000 = $425/month
└── Total CDN:                ~$448/month

ALB:
├── Load Balancer:            $0.008/h × 730h = $5.84/month
├── LCU:                      ~$10-20/month
└── Total ALB:                ~$26/month

NAT Gateway:
├── Per NAT:                  $0.045/h × 2 × 730h = $65.7/month
└── Total NAT:                ~$66/month

SQS + Lambda:
├── SQS 10M messages:         $0.40/million × 10 = $4/month
├── Lambda 5M invocations:    ~$1/month
└── Total Serverless:         ~$5/month

TOTAL ESTIMATE: ~$1,353/month
สำหรับ 100,000 users/day = $0.000452 per user per day
```

---

## 🧪 Testing

```bash
# ทดสอบ AWS CLI connection
aws sts get-caller-identity

# ทดสอบ list services ที่ใช้ได้
aws ec2 describe-regions --output table

# ทดสอบ billing
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity MONTHLY \
  --metrics BlendedCost

# ทดสอบ IAM permissions
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/admin \
  --action-names "s3:ListBucket" \
  --resource-arns "arn:aws:s3:::chuaikan-media"
```

---

## ❌ Common Errors & Solutions

### Error 1: "Unable to locate credentials"

```
An error occurred (AuthFailure) when calling the DescribeInstances operation:
Unable to locate credentials
```

**แก้ไข:**
```bash
aws configure
# หรือ
export AWS_ACCESS_KEY_ID=your_key
export AWS_SECRET_ACCESS_KEY=your_secret
export AWS_DEFAULT_REGION=ap-southeast-1
```

### Error 2: "Access Denied"

```
An error occurred (AccessDenied) when calling the CreateBucket operation
```

**แก้ไข:**
```bash
# ตรวจสอบ permissions
aws iam get-user
aws iam list-attached-user-policies --user-name admin

# ตรวจสอบ ว่าใช้ profile ถูกต้อง
aws configure list
echo $AWS_PROFILE
```

### Error 3: "Region not enabled"

```
OptInRequired: You are not subscribed to this service
```

**แก้ไข:**
```bash
# บาง regions ต้อง opt-in ก่อน
# ไปที่ AWS Console > Account > Regions
# Enable region ที่ต้องการ
```

---

## ✅ Checklist

- [ ] Step 501: เข้าใจ Cloud Computing concepts
- [ ] Step 502: เปรียบเทียบ AWS vs GCP และเลือก AWS
- [ ] Step 503: วางแผน Multi-region strategy
- [ ] Step 504: รู้จัก AWS services ที่ต้องใช้
- [ ] Step 505: สร้าง AWS Root Account และ enable MFA
- [ ] Step 506: สร้าง IAM Admin User และ Access Key
- [ ] Step 507: ติดตั้งและ configure AWS CLI
- [ ] Step 508: ตั้งค่า AWS Organizations
- [ ] Step 509: ตั้งค่า Billing Alerts ($500/month)
- [ ] Step 510: สร้าง IAM Policies และ Roles

---

## 🔗 References

- [AWS Global Infrastructure](https://aws.amazon.com/about-aws/global-infrastructure/)
- [AWS Pricing Calculator](https://calculator.aws/pricing/2/home)
- [AWS Free Tier](https://aws.amazon.com/free/)
- [IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [AWS CLI Configuration](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)

---
*Part 051 | Road to 1,000,000 Users/Day | chuaikan.com*
