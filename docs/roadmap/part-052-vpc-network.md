# Part 052: VPC & Network Design
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Advanced
> **Steps:** 511-520
> **เวลาโดยประมาณ:** 5 ชั่วโมง
> **Prerequisites:** Part 051 (AWS Account, CLI)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ออกแบบ VPC สำหรับ chuaikan.com ด้วย CIDR 10.0.0.0/16
- แบ่ง Subnets: public, private, database
- ตั้งค่า Multi-AZ layout (ap-southeast-1a, 1b, 1c)
- ตั้งค่า Internet Gateway, NAT Gateway
- ตั้งค่า Route Tables
- ตั้งค่า Security Groups สำหรับแต่ละ tier
- เปิด VPC Flow Logs สำหรับ security monitoring
- เขียน Terraform code สำหรับ VPC

---

## 📖 ทฤษฎีและแนวคิด

### Step 511: VPC คืออะไร

**VPC (Virtual Private Cloud)** คือ network ส่วนตัวที่แยกออกมาบน AWS ทำหน้าที่เหมือนมี datacenter เป็นของตัวเอง

```
Internet
    │
    ▼
Internet Gateway
    │
    ▼
┌─────────────────────────────────────────┐
│  VPC: 10.0.0.0/16                       │
│                                          │
│  ┌──────────────┐  ┌──────────────┐     │
│  │ Public Subnet│  │ Public Subnet│     │
│  │ 10.0.1.0/24  │  │ 10.0.2.0/24  │     │
│  │ (AZ: 1a)    │  │ (AZ: 1b)    │     │
│  │ ALB, NAT GW  │  │ ALB, NAT GW  │     │
│  └──────────────┘  └──────────────┘     │
│                                          │
│  ┌──────────────┐  ┌──────────────┐     │
│  │Private Subnet│  │Private Subnet│     │
│  │ 10.0.10.0/24 │  │ 10.0.11.0/24 │     │
│  │ (AZ: 1a)    │  │ (AZ: 1b)    │     │
│  │ EKS Nodes   │  │ EKS Nodes   │     │
│  └──────────────┘  └──────────────┘     │
│                                          │
│  ┌──────────────┐  ┌──────────────┐     │
│  │  DB Subnet   │  │  DB Subnet   │     │
│  │ 10.0.20.0/24 │  │ 10.0.21.0/24 │     │
│  │ (AZ: 1a)    │  │ (AZ: 1b)    │     │
│  │ RDS, Redis   │  │ RDS, Redis   │     │
│  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────┘
```

### Step 512: CIDR Planning

```
VPC CIDR: 10.0.0.0/16 (65,536 addresses)

Public Subnets (สำหรับ resources ที่ต้องเชื่อม internet):
├── 10.0.1.0/24 (256 addresses) - AZ: ap-southeast-1a
├── 10.0.2.0/24 (256 addresses) - AZ: ap-southeast-1b
└── 10.0.3.0/24 (256 addresses) - AZ: ap-southeast-1c

Private Subnets (สำหรับ application servers):
├── 10.0.10.0/24 (256 addresses) - AZ: ap-southeast-1a
├── 10.0.11.0/24 (256 addresses) - AZ: ap-southeast-1b
└── 10.0.12.0/24 (256 addresses) - AZ: ap-southeast-1c

Database Subnets (สำหรับ RDS, Redis):
├── 10.0.20.0/24 (256 addresses) - AZ: ap-southeast-1a
├── 10.0.21.0/24 (256 addresses) - AZ: ap-southeast-1b
└── 10.0.22.0/24 (256 addresses) - AZ: ap-southeast-1c

Reserved for future use:
└── 10.0.100.0/24 ... 10.0.255.0/24
```

### Step 513: Security Groups Design

```
web-sg (ALB):
├── Inbound: 80 (HTTP) from 0.0.0.0/0
├── Inbound: 443 (HTTPS) from 0.0.0.0/0
└── Outbound: All to app-sg

app-sg (EKS Nodes):
├── Inbound: 8080 from web-sg
├── Inbound: NodePort 30000-32767 from web-sg
└── Outbound: All to db-sg, internet (via NAT)

db-sg (RDS, Redis):
├── Inbound: 5432 (PostgreSQL) from app-sg
├── Inbound: 6379 (Redis) from app-sg
└── Outbound: None (database ไม่ควร initiate connections)
```

---

## ⚙️ Environment Setup

### ติดตั้ง Terraform

```bash
# Linux/WSL
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# macOS
brew install terraform

# ตรวจสอบ version
terraform version
# Terraform v1.7.0
```

---

## 🛠️ Step-by-Step Implementation

### Step 514: สร้าง VPC ด้วย AWS CLI

```bash
# สร้าง VPC
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=chuaikan-vpc},{Key=Environment,Value=production}]' \
  --query 'Vpc.VpcId' \
  --output text)

echo "VPC ID: $VPC_ID"

# เปิด DNS hostnames
aws ec2 modify-vpc-attribute \
  --vpc-id $VPC_ID \
  --enable-dns-hostnames

# เปิด DNS support
aws ec2 modify-vpc-attribute \
  --vpc-id $VPC_ID \
  --enable-dns-support
```

### Step 515: สร้าง Subnets

```bash
# Public Subnets
PUBLIC_SUBNET_1A=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.1.0/24 \
  --availability-zone ap-southeast-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=chuaikan-public-1a},{Key=Type,Value=public}]' \
  --query 'Subnet.SubnetId' --output text)

PUBLIC_SUBNET_1B=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.2.0/24 \
  --availability-zone ap-southeast-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=chuaikan-public-1b},{Key=Type,Value=public}]' \
  --query 'Subnet.SubnetId' --output text)

# Private Subnets
PRIVATE_SUBNET_1A=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.10.0/24 \
  --availability-zone ap-southeast-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=chuaikan-private-1a},{Key=Type,Value=private},{Key=kubernetes.io/role/internal-elb,Value=1}]' \
  --query 'Subnet.SubnetId' --output text)

PRIVATE_SUBNET_1B=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.11.0/24 \
  --availability-zone ap-southeast-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=chuaikan-private-1b},{Key=Type,Value=private},{Key=kubernetes.io/role/internal-elb,Value=1}]' \
  --query 'Subnet.SubnetId' --output text)

# Database Subnets
DB_SUBNET_1A=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.20.0/24 \
  --availability-zone ap-southeast-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=chuaikan-db-1a},{Key=Type,Value=database}]' \
  --query 'Subnet.SubnetId' --output text)

DB_SUBNET_1B=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.21.0/24 \
  --availability-zone ap-southeast-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=chuaikan-db-1b},{Key=Type,Value=database}]' \
  --query 'Subnet.SubnetId' --output text)
```

### Step 516: สร้าง Internet Gateway และ NAT Gateway

```bash
# สร้าง Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=chuaikan-igw}]' \
  --query 'InternetGateway.InternetGatewayId' --output text)

# Attach IGW ไปยัง VPC
aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $VPC_ID

# สร้าง Elastic IPs สำหรับ NAT Gateways
EIP_1A=$(aws ec2 allocate-address \
  --domain vpc \
  --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=chuaikan-nat-eip-1a}]' \
  --query 'AllocationId' --output text)

EIP_1B=$(aws ec2 allocate-address \
  --domain vpc \
  --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=chuaikan-nat-eip-1b}]' \
  --query 'AllocationId' --output text)

# สร้าง NAT Gateways (ใน Public Subnet)
NAT_1A=$(aws ec2 create-nat-gateway \
  --subnet-id $PUBLIC_SUBNET_1A \
  --allocation-id $EIP_1A \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=chuaikan-nat-1a}]' \
  --query 'NatGateway.NatGatewayId' --output text)

NAT_1B=$(aws ec2 create-nat-gateway \
  --subnet-id $PUBLIC_SUBNET_1B \
  --allocation-id $EIP_1B \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=chuaikan-nat-1b}]' \
  --query 'NatGateway.NatGatewayId' --output text)

# รอ NAT Gateways พร้อม
echo "Waiting for NAT Gateways..."
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_1A $NAT_1B
echo "NAT Gateways are ready!"
```

### Step 517: ตั้งค่า Route Tables

```bash
# Public Route Table
PUBLIC_RT=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=chuaikan-public-rt}]' \
  --query 'RouteTable.RouteTableId' --output text)

# เพิ่ม route ไปยัง Internet
aws ec2 create-route \
  --route-table-id $PUBLIC_RT \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID

# Associate public subnets
aws ec2 associate-route-table --route-table-id $PUBLIC_RT --subnet-id $PUBLIC_SUBNET_1A
aws ec2 associate-route-table --route-table-id $PUBLIC_RT --subnet-id $PUBLIC_SUBNET_1B

# Private Route Tables (แยกต่อ AZ เพื่อ HA)
PRIVATE_RT_1A=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=chuaikan-private-rt-1a}]' \
  --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-route \
  --route-table-id $PRIVATE_RT_1A \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NAT_1A

aws ec2 associate-route-table --route-table-id $PRIVATE_RT_1A --subnet-id $PRIVATE_SUBNET_1A
aws ec2 associate-route-table --route-table-id $PRIVATE_RT_1A --subnet-id $DB_SUBNET_1A

PRIVATE_RT_1B=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=chuaikan-private-rt-1b}]' \
  --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-route \
  --route-table-id $PRIVATE_RT_1B \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NAT_1B

aws ec2 associate-route-table --route-table-id $PRIVATE_RT_1B --subnet-id $PRIVATE_SUBNET_1B
aws ec2 associate-route-table --route-table-id $PRIVATE_RT_1B --subnet-id $DB_SUBNET_1B
```

### Step 518: สร้าง Security Groups

```bash
# Web Tier Security Group (ALB)
WEB_SG=$(aws ec2 create-security-group \
  --group-name chuaikan-web-sg \
  --description "Security group for ALB" \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=chuaikan-web-sg}]' \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $WEB_SG --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $WEB_SG --protocol tcp --port 443 --cidr 0.0.0.0/0

# App Tier Security Group (EKS Nodes)
APP_SG=$(aws ec2 create-security-group \
  --group-name chuaikan-app-sg \
  --description "Security group for EKS nodes" \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=chuaikan-app-sg}]' \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $APP_SG --protocol tcp --port 8080 --source-group $WEB_SG
aws ec2 authorize-security-group-ingress --group-id $APP_SG --protocol tcp --port 30000-32767 --source-group $WEB_SG
# EKS nodes communicate with each other
aws ec2 authorize-security-group-ingress --group-id $APP_SG --protocol -1 --source-group $APP_SG

# Database Tier Security Group
DB_SG=$(aws ec2 create-security-group \
  --group-name chuaikan-db-sg \
  --description "Security group for RDS and Redis" \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=chuaikan-db-sg}]' \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $DB_SG --protocol tcp --port 5432 --source-group $APP_SG
aws ec2 authorize-security-group-ingress --group-id $DB_SG --protocol tcp --port 6379 --source-group $APP_SG
```

### Step 519: ตั้งค่า VPC Flow Logs

```bash
# สร้าง CloudWatch Log Group
aws logs create-log-group \
  --log-group-name /aws/vpc/flowlogs/chuaikan

# สร้าง IAM Role สำหรับ Flow Logs
cat > vpc-flow-logs-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "vpc-flow-logs.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

FLOW_LOGS_ROLE=$(aws iam create-role \
  --role-name VPCFlowLogsRole \
  --assume-role-policy-document file://vpc-flow-logs-policy.json \
  --query 'Role.Arn' --output text)

# Attach CloudWatch Logs policy
aws iam attach-role-policy \
  --role-name VPCFlowLogsRole \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess

# เปิด VPC Flow Logs
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids $VPC_ID \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/flowlogs/chuaikan \
  --deliver-logs-permission-arn $FLOW_LOGS_ROLE \
  --log-format '${version} ${account-id} ${interface-id} ${srcaddr} ${dstaddr} ${srcport} ${dstport} ${protocol} ${packets} ${bytes} ${windowstart} ${windowend} ${action} ${flowlogstatus}'
```

---

## 🔧 Configuration Files

### Terraform VPC Module (terraform/modules/vpc/main.tf)

```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

locals {
  name = "chuaikan-${var.environment}"
  azs  = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]
  
  public_subnets   = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  private_subnets  = ["10.0.10.0/24", "10.0.11.0/24", "10.0.12.0/24"]
  database_subnets = ["10.0.20.0/24", "10.0.21.0/24", "10.0.22.0/24"]
  
  tags = {
    Environment = var.environment
    Project     = "chuaikan"
    ManagedBy   = "terraform"
  }
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(local.tags, {
    Name = "${local.name}-vpc"
  })
}

# Public Subnets
resource "aws_subnet" "public" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.public_subnets[count.index]
  availability_zone = local.azs[count.index]
  
  map_public_ip_on_launch = true

  tags = merge(local.tags, {
    Name = "${local.name}-public-${local.azs[count.index]}"
    Type = "public"
    "kubernetes.io/role/elb" = "1"
  })
}

# Private Subnets
resource "aws_subnet" "private" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.private_subnets[count.index]
  availability_zone = local.azs[count.index]

  tags = merge(local.tags, {
    Name = "${local.name}-private-${local.azs[count.index]}"
    Type = "private"
    "kubernetes.io/role/internal-elb" = "1"
  })
}

# Database Subnets
resource "aws_subnet" "database" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.database_subnets[count.index]
  availability_zone = local.azs[count.index]

  tags = merge(local.tags, {
    Name = "${local.name}-db-${local.azs[count.index]}"
    Type = "database"
  })
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = merge(local.tags, {
    Name = "${local.name}-igw"
  })
}

# Elastic IPs for NAT Gateways
resource "aws_eip" "nat" {
  count  = 2
  domain = "vpc"

  tags = merge(local.tags, {
    Name = "${local.name}-nat-eip-${count.index + 1}"
  })

  depends_on = [aws_internet_gateway.main]
}

# NAT Gateways
resource "aws_nat_gateway" "main" {
  count         = 2
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = merge(local.tags, {
    Name = "${local.name}-nat-${count.index + 1}"
  })

  depends_on = [aws_internet_gateway.main]
}

# Public Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = merge(local.tags, {
    Name = "${local.name}-public-rt"
  })
}

resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Private Route Tables
resource "aws_route_table" "private" {
  count  = 2
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = merge(local.tags, {
    Name = "${local.name}-private-rt-${count.index + 1}"
  })
}

resource "aws_route_table_association" "private" {
  count          = length(aws_subnet.private)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index % 2].id
}

resource "aws_route_table_association" "database" {
  count          = length(aws_subnet.database)
  subnet_id      = aws_subnet.database[count.index].id
  route_table_id = aws_route_table.private[count.index % 2].id
}

# Security Groups
resource "aws_security_group" "web" {
  name        = "${local.name}-web-sg"
  description = "Security group for ALB"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(local.tags, { Name = "${local.name}-web-sg" })
}

resource "aws_security_group" "app" {
  name        = "${local.name}-app-sg"
  description = "Security group for EKS nodes"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.web.id]
  }

  ingress {
    from_port       = 30000
    to_port         = 32767
    protocol        = "tcp"
    security_groups = [aws_security_group.web.id]
  }

  ingress {
    from_port = 0
    to_port   = 0
    protocol  = "-1"
    self      = true
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(local.tags, { Name = "${local.name}-app-sg" })
}

resource "aws_security_group" "database" {
  name        = "${local.name}-db-sg"
  description = "Security group for RDS and Redis"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }

  ingress {
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }

  tags = merge(local.tags, { Name = "${local.name}-db-sg" })
}

# VPC Flow Logs
resource "aws_cloudwatch_log_group" "flow_logs" {
  name              = "/aws/vpc/flowlogs/${local.name}"
  retention_in_days = 30

  tags = local.tags
}

resource "aws_flow_log" "main" {
  vpc_id          = aws_vpc.main.id
  traffic_type    = "ALL"
  iam_role_arn    = aws_iam_role.flow_logs.arn
  log_destination = aws_cloudwatch_log_group.flow_logs.arn

  tags = merge(local.tags, { Name = "${local.name}-flow-logs" })
}

resource "aws_iam_role" "flow_logs" {
  name = "${local.name}-flow-logs-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "vpc-flow-logs.amazonaws.com"
      }
    }]
  })

  tags = local.tags
}

resource "aws_iam_role_policy" "flow_logs" {
  name = "${local.name}-flow-logs-policy"
  role = aws_iam_role.flow_logs.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams"
      ]
      Resource = "*"
    }]
  })
}
```

### Terraform Variables (terraform/modules/vpc/variables.tf)

```hcl
variable "environment" {
  description = "Environment name (dev/staging/prod)"
  type        = string
}
```

### Terraform Outputs (terraform/modules/vpc/outputs.tf)

```hcl
output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}

output "database_subnet_ids" {
  value = aws_subnet.database[*].id
}

output "web_security_group_id" {
  value = aws_security_group.web.id
}

output "app_security_group_id" {
  value = aws_security_group.app.id
}

output "database_security_group_id" {
  value = aws_security_group.database.id
}
```

---

## 🧪 Testing

```bash
# ตรวจสอบ VPC
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=chuaikan-vpc" --query 'Vpcs[0].{ID:VpcId,CIDR:CidrBlock,DNS:EnableDnsHostnames}'

# ตรวจสอบ Subnets
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[*].{Name:Tags[?Key==`Name`]|[0].Value,CIDR:CidrBlock,AZ:AvailabilityZone}' \
  --output table

# ตรวจสอบ NAT Gateways
aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=$VPC_ID" \
  --query 'NatGateways[*].{ID:NatGatewayId,State:State,Subnet:SubnetId}'

# ทดสอบ connectivity จาก EC2 ใน private subnet
# Launch test EC2 ใน private subnet
TEST_INSTANCE=$(aws ec2 run-instances \
  --image-id ami-0df7a207adb9748c7 \
  --instance-type t3.micro \
  --subnet-id $PRIVATE_SUBNET_1A \
  --security-group-ids $APP_SG \
  --query 'Instances[0].InstanceId' --output text)

# SSM Connect (ไม่ต้องใช้ SSH key)
aws ssm start-session --target $TEST_INSTANCE

# ทดสอบ internet access ผ่าน NAT
curl -s https://checkip.amazonaws.com
# ควรแสดง IP ของ NAT Gateway
```

---

## ❌ Common Errors & Solutions

### Error 1: NAT Gateway ยังไม่พร้อม

```
Error: creating route: InvalidNatGatewayID.NotFound
```

**แก้ไข:**
```bash
# รอ NAT Gateway สร้างเสร็จก่อน
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_1A
```

### Error 2: Subnet CIDR Overlap

```
Error: InvalidSubnet.Conflict: The CIDR '10.0.1.0/24' conflicts with another subnet
```

**แก้ไข:**
```bash
# ตรวจสอบ subnets ที่มีอยู่
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[*].CidrBlock'
```

---

## ✅ Checklist

- [ ] Step 511: เข้าใจ VPC architecture
- [ ] Step 512: วางแผน CIDR และ subnet design
- [ ] Step 513: วางแผน Security Groups
- [ ] Step 514: สร้าง VPC ด้วย AWS CLI
- [ ] Step 515: สร้าง Public, Private, Database Subnets
- [ ] Step 516: สร้าง Internet Gateway และ NAT Gateways
- [ ] Step 517: ตั้งค่า Route Tables
- [ ] Step 518: สร้าง Security Groups (web, app, database)
- [ ] Step 519: เปิด VPC Flow Logs
- [ ] Step 520: เขียน Terraform module สำหรับ VPC

---

## 🔗 References

- [AWS VPC User Guide](https://docs.aws.amazon.com/vpc/latest/userguide/)
- [VPC Subnet Design Best Practices](https://aws.amazon.com/answers/networking/aws-single-vpc-design/)
- [Terraform AWS VPC Module](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/)

---
*Part 052 | Road to 1,000,000 Users/Day | chuaikan.com*
