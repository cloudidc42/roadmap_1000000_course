# Part 060: Infrastructure as Code ด้วย Terraform
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Advanced
> **Steps:** 591-600
> **เวลาโดยประมาณ:** 6 ชั่วโมง
> **Prerequisites:** Part 051-059 (AWS Services ทั้งหมด)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- Terraform concepts: providers, resources, state
- ติดตั้ง Terraform และ AWS Provider
- State management ด้วย S3 backend + DynamoDB locking
- Modules สำหรับ reusable infrastructure
- Complete Terraform สำหรับ chuaikan.com: VPC, EKS, RDS, ElastiCache, S3
- terraform.tfvars สำหรับ environments ต่างๆ
- terraform plan, apply, destroy
- Terragrunt สำหรับ DRY Terraform
- CI/CD กับ Terraform ด้วย GitHub Actions

---

## 📖 ทฤษฎีและแนวคิด

### Step 591: Terraform คืออะไร

```
Infrastructure as Code (IaC) คืออะไร:

แบบเก่า (ClickOps):
AWS Console → Manual Click → สร้าง VPC
                           → สร้าง EC2
                           → ตั้งค่า Security Group
                           → ... (100+ steps, ทำซ้ำ 3 environments)

แบบใหม่ (Terraform):
main.tf → terraform apply → สร้างทุกอย่างอัตโนมัติ
                          → Reproducible
                          → Version controlled ด้วย Git
                          → Review ได้ก่อน apply

Terraform State:
├── terraform.tfstate - เก็บ current state
├── .terraform/       - provider plugins
└── .terraform.lock.hcl - provider version lock
```

### Step 592: Terraform Workflow

```
1. Write: เขียน .tf files
   
2. Plan: ดู changes ก่อน apply
   terraform plan → shows what will be created/modified/destroyed
   
3. Apply: สร้าง infrastructure จริง
   terraform apply → execute the plan
   
4. Destroy: ลบ infrastructure ทั้งหมด
   terraform destroy → CAREFUL! ลบจริง
   
Resource Lifecycle:
create → read → update-in-place or destroy+create → destroy
```

### Step 593: Terraform File Structure สำหรับ chuaikan.com

```
terraform/
├── modules/                    # Reusable modules
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── eks/
│   ├── rds/
│   ├── elasticache/
│   ├── s3/
│   └── cloudfront/
├── environments/
│   ├── dev/
│   │   ├── main.tf             # สั่งใช้ modules
│   │   ├── terraform.tfvars    # values สำหรับ dev
│   │   └── backend.tf          # state backend config
│   ├── staging/
│   └── prod/
└── global/
    ├── iam.tf                  # Global IAM resources
    └── route53.tf              # DNS
```

---

## ⚙️ Environment Setup

### Step 594: ติดตั้งและตั้งค่า Terraform

```bash
# ติดตั้ง Terraform (Linux)
wget -O- https://apt.releases.hashicorp.com/gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# ตรวจสอบ
terraform version
# Terraform v1.7.0

# ติดตั้ง tfenv (version manager)
git clone https://github.com/tfutils/tfenv.git ~/.tfenv
echo 'export PATH="$HOME/.tfenv/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

tfenv install 1.7.0
tfenv use 1.7.0

# ติดตั้ง tflint (linter)
curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash

# ติดตั้ง terraform-docs
brew install terraform-docs  # macOS
```

### Step 595: ตั้งค่า Remote State Backend

```bash
# สร้าง S3 bucket สำหรับ Terraform state
aws s3api create-bucket \
  --bucket chuaikan-terraform-state \
  --region ap-southeast-1 \
  --create-bucket-configuration LocationConstraint=ap-southeast-1

# เปิด versioning
aws s3api put-bucket-versioning \
  --bucket chuaikan-terraform-state \
  --versioning-configuration Status=Enabled

# Block public access
aws s3api put-public-access-block \
  --bucket chuaikan-terraform-state \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# เปิด encryption
aws s3api put-bucket-encryption \
  --bucket chuaikan-terraform-state \
  --server-side-encryption-configuration '{
    "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}]
  }'

# สร้าง DynamoDB table สำหรับ state locking
aws dynamodb create-table \
  --table-name chuaikan-terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region ap-southeast-1
```

---

## 🛠️ Step-by-Step Implementation

### Step 596: Main Terraform Structure

```hcl
# terraform/environments/prod/backend.tf
terraform {
  backend "s3" {
    bucket         = "chuaikan-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "ap-southeast-1"
    encrypt        = true
    dynamodb_table = "chuaikan-terraform-locks"
  }
}
```

```hcl
# terraform/environments/prod/main.tf
terraform {
  required_version = ">= 1.5"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Environment = var.environment
      Project     = "chuaikan"
      ManagedBy   = "terraform"
      Repository  = "github.com/chuaikan/infrastructure"
    }
  }
}

# Module: VPC
module "vpc" {
  source = "../../modules/vpc"

  environment = var.environment
  project     = var.project_name
}

# Module: EKS
module "eks" {
  source = "../../modules/eks"

  cluster_name    = "${var.project_name}-${var.environment}"
  cluster_version = "1.29"
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnet_ids

  node_groups = {
    general = {
      instance_types = ["m6g.large"]
      min_size       = 2
      max_size       = 20
      desired_size   = 3
    }
    memory = {
      instance_types = ["r6g.xlarge"]
      min_size       = 1
      max_size       = 5
      desired_size   = 2
    }
  }

  environment = var.environment
  tags        = local.common_tags
}

# Module: RDS
module "rds" {
  source = "../../modules/rds"

  project                    = var.project_name
  environment                = var.environment
  vpc_id                     = module.vpc.vpc_id
  database_subnet_ids        = module.vpc.database_subnet_ids
  database_security_group_id = module.vpc.database_security_group_id

  db_instance_class        = var.db_instance_class
  db_engine_version        = var.db_engine_version
  db_allocated_storage     = var.db_allocated_storage
  db_max_allocated_storage = var.db_max_allocated_storage
  db_multi_az              = var.db_multi_az
  db_backup_retention      = var.db_backup_retention
  db_name                  = var.db_name
  db_username              = var.db_username
  read_replica_count       = var.read_replica_count
  replica_instance_class   = var.replica_instance_class
  replica_azs              = ["ap-southeast-1a", "ap-southeast-1b"]

  tags = local.common_tags
}

# Module: ElastiCache
module "elasticache" {
  source = "../../modules/elasticache"

  project                    = var.project_name
  environment                = var.environment
  database_subnet_ids        = module.vpc.database_subnet_ids
  database_security_group_id = module.vpc.database_security_group_id

  cache_node_type    = var.cache_node_type
  num_shards         = var.num_shards
  replicas_per_shard = var.replicas_per_shard

  tags = local.common_tags
}

# Module: S3
module "s3" {
  source = "../../modules/s3"

  project     = var.project_name
  environment = var.environment
  domain      = var.domain

  tags = local.common_tags
}

# Module: SQS
module "sqs" {
  source = "../../modules/sqs"

  project     = var.project_name
  environment = var.environment

  tags = local.common_tags
}

locals {
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
  }
}
```

### Terraform Variables

```hcl
# terraform/environments/prod/variables.tf
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "ap-southeast-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "project_name" {
  description = "Project name"
  type        = string
  default     = "chuaikan"
}

variable "domain" {
  description = "Domain name"
  type        = string
  default     = "chuaikan.com"
}

# RDS
variable "db_instance_class" {
  type    = string
  default = "db.r6g.xlarge"
}

variable "db_engine_version" {
  type    = string
  default = "17.2"
}

variable "db_allocated_storage" {
  type    = number
  default = 100
}

variable "db_max_allocated_storage" {
  type    = number
  default = 500
}

variable "db_multi_az" {
  type    = bool
  default = true
}

variable "db_backup_retention" {
  type    = number
  default = 7
}

variable "db_name" {
  type    = string
  default = "chuaikan"
}

variable "db_username" {
  type    = string
  default = "chuaikan_admin"
}

variable "read_replica_count" {
  type    = number
  default = 2
}

variable "replica_instance_class" {
  type    = string
  default = "db.r6g.large"
}

# ElastiCache
variable "cache_node_type" {
  type    = string
  default = "cache.r6g.large"
}

variable "num_shards" {
  type    = number
  default = 3
}

variable "replicas_per_shard" {
  type    = number
  default = 1
}
```

### terraform.tfvars สำหรับ Environments

```hcl
# terraform/environments/prod/terraform.tfvars
environment  = "production"
aws_region   = "ap-southeast-1"
project_name = "chuaikan"
domain       = "chuaikan.com"

# RDS - Production (larger)
db_instance_class        = "db.r6g.xlarge"
db_engine_version        = "17.2"
db_allocated_storage     = 100
db_max_allocated_storage = 500
db_multi_az              = true
db_backup_retention      = 7
db_name                  = "chuaikan"
db_username              = "chuaikan_admin"
read_replica_count       = 2
replica_instance_class   = "db.r6g.large"

# ElastiCache - Production
cache_node_type    = "cache.r6g.large"
num_shards         = 3
replicas_per_shard = 1
```

```hcl
# terraform/environments/dev/terraform.tfvars
environment  = "development"
aws_region   = "ap-southeast-1"
project_name = "chuaikan"
domain       = "dev.chuaikan.com"

# RDS - Development (smaller to save cost)
db_instance_class        = "db.t3.medium"
db_engine_version        = "17.2"
db_allocated_storage     = 20
db_max_allocated_storage = 100
db_multi_az              = false
db_backup_retention      = 1
db_name                  = "chuaikan_dev"
db_username              = "chuaikan_dev"
read_replica_count       = 0
replica_instance_class   = "db.t3.small"

# ElastiCache - Development
cache_node_type    = "cache.t3.micro"
num_shards         = 1
replicas_per_shard = 0
```

---

## 🔧 Configuration Files

### Step 597: Terragrunt Configuration

```hcl
# terragrunt.hcl (root)
locals {
  common_vars = read_terragrunt_config(find_in_parent_folders("common.hcl"))
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  region_vars = read_terragrunt_config(find_in_parent_folders("region.hcl"))
  env_vars = read_terragrunt_config(find_in_parent_folders("env.hcl"))

  aws_region  = local.region_vars.locals.aws_region
  environment = local.env_vars.locals.environment
  project     = local.common_vars.locals.project
}

# Remote state config
remote_state {
  backend = "s3"
  config = {
    bucket         = "chuaikan-terraform-state"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = local.aws_region
    encrypt        = true
    dynamodb_table = "chuaikan-terraform-locks"
  }

  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
}

# Generate provider
generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
provider "aws" {
  region = "${local.aws_region}"
  
  default_tags {
    tags = {
      Environment = "${local.environment}"
      Project     = "${local.project}"
      ManagedBy   = "terragrunt"
    }
  }
}
EOF
}
```

```hcl
# terragrunt/live/prod/ap-southeast-1/vpc/terragrunt.hcl
include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "../../../../modules//vpc"
}

inputs = {
  environment = "production"
  project     = "chuaikan"
}
```

### Step 598: GitHub Actions สำหรับ Terraform CI/CD

```yaml
# .github/workflows/terraform.yml
name: Terraform CI/CD

on:
  pull_request:
    branches: [main]
    paths:
      - 'terraform/**'
  push:
    branches: [main]
    paths:
      - 'terraform/**'

env:
  TF_VERSION: "1.7.0"
  AWS_REGION: "ap-southeast-1"

permissions:
  contents: read
  pull-requests: write
  id-token: write  # For OIDC

jobs:
  terraform-plan:
    name: Terraform Plan
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: terraform/environments/prod

    steps:
    - uses: actions/checkout@v4

    - name: Configure AWS Credentials (OIDC)
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789012:role/github-actions-terraform-role
        aws-region: ${{ env.AWS_REGION }}

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: ${{ env.TF_VERSION }}

    - name: Terraform Format Check
      run: terraform fmt -check -recursive

    - name: Terraform Init
      run: terraform init

    - name: Terraform Validate
      run: terraform validate

    - name: Run tflint
      uses: terraform-linters/setup-tflint@v4
      with:
        tflint_version: latest

    - name: tflint
      run: |
        tflint --init
        tflint

    - name: Terraform Plan
      id: plan
      run: terraform plan -no-color -out=tfplan 2>&1 | tee plan.txt
      continue-on-error: true

    - name: Comment PR with Plan
      uses: actions/github-script@v7
      if: github.event_name == 'pull_request'
      env:
        PLAN: ${{ steps.plan.outputs.stdout }}
      with:
        github-token: ${{ secrets.GITHUB_TOKEN }}
        script: |
          const fs = require('fs');
          const plan = fs.readFileSync('terraform/environments/prod/plan.txt', 'utf8');
          const maxLength = 65000;
          const truncatedPlan = plan.length > maxLength 
            ? plan.substring(0, maxLength) + '\n... (truncated)'
            : plan;
          
          const output = `## Terraform Plan Output
          
          <details>
          <summary>Click to expand plan</summary>
          
          \`\`\`terraform
          ${truncatedPlan}
          \`\`\`
          </details>
          
          *Pushed by @${{ github.actor }}, Action: \`${{ github.event_name }}\`*`;
          
          github.rest.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.repo,
            body: output
          });

    - name: Terraform Plan Status
      if: steps.plan.outcome == 'failure'
      run: exit 1

    - name: Upload Plan
      uses: actions/upload-artifact@v4
      with:
        name: tfplan
        path: terraform/environments/prod/tfplan

  terraform-apply:
    name: Terraform Apply
    runs-on: ubuntu-latest
    needs: terraform-plan
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production  # Requires manual approval in GitHub
    defaults:
      run:
        working-directory: terraform/environments/prod

    steps:
    - uses: actions/checkout@v4

    - name: Configure AWS Credentials (OIDC)
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789012:role/github-actions-terraform-role
        aws-region: ${{ env.AWS_REGION }}

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: ${{ env.TF_VERSION }}

    - name: Terraform Init
      run: terraform init

    - name: Download Plan
      uses: actions/download-artifact@v4
      with:
        name: tfplan
        path: terraform/environments/prod/

    - name: Terraform Apply
      run: terraform apply -auto-approve tfplan

    - name: Notify Slack
      if: always()
      uses: slackapi/slack-github-action@v1
      with:
        payload: |
          {
            "text": "Terraform Apply ${{ job.status }}: ${{ github.repository }}"
          }
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## 🧪 Testing

```bash
# Step 599: Terraform Plan และ Apply

# Initialize
cd terraform/environments/prod
terraform init

# Validate syntax
terraform validate
# Success! The configuration is valid.

# Check formatting
terraform fmt -check
# (ถ้ามี formatting issues จะแสดง file names)

terraform fmt -recursive  # Auto-fix formatting

# Plan (dry run)
terraform plan
# Terraform will perform the following actions:
#   # module.vpc.aws_vpc.main will be created
#   + resource "aws_vpc" "main" {
#       + cidr_block = "10.0.0.0/16"
#       ...
#   }
# Plan: 45 to add, 0 to change, 0 to destroy.

# Apply (สร้าง infrastructure จริง)
terraform apply
# Enter "yes" to confirm

# หรือ auto-approve (ใช้ใน CI/CD)
terraform apply -auto-approve

# ดู current state
terraform show

# ดู outputs
terraform output
# vpc_id = "vpc-xxxxxxxxxxxxxxxxx"
# rds_endpoint = "chuaikan-prod-db.xxxxxxxxx.rds.amazonaws.com"

# ดู specific resource
terraform state show module.rds.aws_db_instance.main

# Import existing resource (ถ้า resource ถูกสร้างด้วย Console)
terraform import module.s3.aws_s3_bucket.media chuaikan-media-production

# Destroy (ระวัง! ลบทุกอย่าง)
# terraform destroy  # อย่าทำใน production!

# Destroy specific resource เท่านั้น
terraform destroy -target=module.eks
```

### Step 600: Best Practices และ Checklist Final

```bash
# ตั้งค่า .gitignore สำหรับ Terraform
cat > .gitignore << 'EOF'
# Terraform state files
*.tfstate
*.tfstate.backup
*.tfstate.lock.info

# Terraform plans
*.tfplan
*.plan

# Terraform directories
.terraform/
.terraform.lock.hcl

# Environment variables
*.env
.env.*

# Sensitive files
terraform.tfvars.json

# macOS
.DS_Store
EOF

# ตั้งค่า pre-commit hooks
cat > .pre-commit-config.yaml << 'EOF'
repos:
- repo: https://github.com/antonbabenko/pre-commit-terraform
  rev: v1.88.4
  hooks:
    - id: terraform_fmt
    - id: terraform_validate
    - id: terraform_tflint
    - id: terraform_docs
      args:
        - '--output-file=README.md'
        - '--output-mode=inject'
EOF

pre-commit install

# ทดสอบทุก modules
for module in terraform/modules/*/; do
  echo "Testing $module..."
  cd $module
  terraform init -backend=false
  terraform validate
  cd -
done
```

---

## ❌ Common Errors & Solutions

### Error 1: State Lock

```
Error: Error acquiring the state lock
Error message: ConditionalCheckFailedException: The conditional request failed
```

**แก้ไข:**
```bash
# ตรวจสอบ lock
aws dynamodb scan --table-name chuaikan-terraform-locks

# Force unlock (ทำเมื่อแน่ใจว่าไม่มีคนกำลัง apply)
terraform force-unlock LOCK_ID
```

### Error 2: Resource already exists

```
Error: creating S3 Bucket (chuaikan-media-production): 
BucketAlreadyExists: The requested bucket name is not available
```

**แก้ไข:**
```bash
# Import existing resource เข้า state
terraform import module.s3.aws_s3_bucket.media chuaikan-media-production
```

### Error 3: Circular dependency

```
Error: Cycle: module.eks.aws_eks_cluster.main, module.vpc.aws_subnet.private
```

**แก้ไข:**
```hcl
# แก้โดยใช้ depends_on ให้ถูกต้อง
resource "aws_eks_cluster" "main" {
  depends_on = [module.vpc]
  # ...
}
```

---

## 📊 Cost Summary - chuaikan.com Full Stack

```
Monthly Cost Breakdown (Production, ap-southeast-1):

Compute (EKS):
├── EKS Control Plane: $73
├── EC2 Nodes (5× m6g.large): $282
└── Subtotal: $355

Database (RDS):
├── RDS Multi-AZ (db.r6g.xlarge): $567
├── Read Replicas (2× db.r6g.large): $567
├── RDS Proxy: $44
└── Subtotal: $1,178

Cache (ElastiCache):
├── Redis Cluster (6× cache.r6g.large): $727
└── Subtotal: $727

Storage & CDN:
├── S3 (1TB): $23
├── CloudFront (5TB): $425
└── Subtotal: $448

Serverless:
├── Lambda: $195
├── SQS/SNS: $5
└── Subtotal: $200

Network:
├── NAT Gateway (2×): $66
├── ALB: $26
└── Subtotal: $92

TOTAL ON-DEMAND: ~$3,000/month

With Reserved Instances (1 year):
├── RDS Reserved: ประหยัด $270/month
├── ElastiCache Reserved: ประหยัด $298/month
└── TOTAL RESERVED: ~$2,432/month (ประหยัด 19%)

At 100,000 users/day = 3,000,000 users/month:
Cost per user: $2,432 / 3,000,000 = $0.00081 per user
= ประมาณ 0.03 บาทต่อผู้ใช้ต่อเดือน
```

---

## ✅ Checklist

- [ ] Step 591: เข้าใจ Terraform concepts และ workflow
- [ ] Step 592: เข้าใจ terraform plan/apply/destroy
- [ ] Step 593: วางแผน Terraform file structure
- [ ] Step 594: ติดตั้ง Terraform, tflint, terraform-docs
- [ ] Step 595: ตั้งค่า S3 backend + DynamoDB locking
- [ ] Step 596: เขียน main.tf ที่ใช้ modules ทั้งหมด
- [ ] Step 597: ตั้งค่า Terragrunt สำหรับ DRY code
- [ ] Step 598: ตั้งค่า GitHub Actions CI/CD
- [ ] Step 599: ทดสอบ terraform plan, apply
- [ ] Step 600: ตั้งค่า pre-commit hooks และ best practices

---

## 🔗 References

- [Terraform Documentation](https://developer.hashicorp.com/terraform/docs)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Terragrunt Documentation](https://terragrunt.gruntwork.io/docs/)
- [Terraform Best Practices](https://www.terraform-best-practices.com/)
- [GitHub Actions OIDC AWS](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)

---
*Part 060 | Road to 1,000,000 Users/Day | chuaikan.com*
