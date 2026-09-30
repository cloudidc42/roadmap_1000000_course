# Part 056: S3 / GCS (Object Storage)
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Advanced
> **Steps:** 551-560
> **เวลาโดยประมาณ:** 4 ชั่วโมง
> **Prerequisites:** Part 051 (AWS Account), Part 052 (VPC)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ตั้งค่า S3 Bucket สำหรับ chuaikan.com media
- ตั้งค่า Bucket Policies และ IAM
- ตั้งค่า S3 Lifecycle Rules (move to Glacier หลัง 90 วัน)
- ใช้ S3 Object Lambda สำหรับ on-the-fly image processing
- สร้าง Presigned URLs สำหรับ direct upload
- ตั้งค่า CORS
- ตั้งค่า S3 Cross-Region Replication (DR)
- ประมาณค่าใช้จ่าย 1TB media storage
- ใช้ Cloudflare R2 แทน S3 ประหยัดค่า egress

---

## 📖 ทฤษฎีและแนวคิด

### Step 551: S3 Architecture สำหรับ chuaikan.com

```
chuaikan.com Media Flow:

1. Upload (ผู้ใช้อัพโหลดรูปสัตว์):
   User → Presigned URL → S3 (direct upload)
   
2. Processing (resize ให้หลายขนาด):
   S3 Event → Lambda → Generate thumbnails → S3

3. Serving (แสดงรูปให้ผู้ใช้):
   User → CloudFront (CDN) → S3 Origin
   
S3 Bucket Structure:
chuaikan-media-production/
├── uploads/           # Original files (temporary)
│   └── {user_id}/{timestamp}/{filename}
├── images/            # Processed images
│   ├── {dog_id}/
│   │   ├── original.jpg
│   │   ├── large.jpg (1200px)
│   │   ├── medium.jpg (600px)
│   │   └── thumbnail.jpg (150px)
│   └── ...
├── videos/
│   └── {dog_id}/video.mp4
└── avatars/
    └── {user_id}/avatar.jpg
```

### Step 552: S3 Storage Classes

```
Standard          → Hot data, frequently accessed    $0.023/GB
Standard-IA       → Infrequent access, >30 days     $0.0125/GB
One Zone-IA       → Infrequent, single AZ            $0.01/GB
Glacier Instant   → Archive, millisecond retrieval   $0.004/GB
Glacier Flexible  → Archive, 1-12h retrieval          $0.0036/GB
Glacier Deep      → Long-term archive, 12-48h         $0.00099/GB

Lifecycle Strategy สำหรับ chuaikan.com:
├── 0-30 days:    Standard (images ใหม่)
├── 30-90 days:   Standard-IA (images เก่า)
├── 90-365 days:  Glacier Instant (archive)
└── 365+ days:    Glacier Deep Archive (long-term)
```

### Step 553: Cloudflare R2 vs S3

| Feature | AWS S3 | Cloudflare R2 |
|---------|--------|---------------|
| Storage | $0.023/GB | $0.015/GB |
| Egress | $0.085/GB | **FREE** |
| Class A ops | $0.005/1K | $4.50/million |
| Class B ops | $0.0004/1K | $0.36/million |
| S3 Compatible | - | ใช่ (ใช้ AWS SDK ได้) |
| CDN | ต้องใช้ CloudFront | Built-in Cloudflare CDN |

**ถ้า chuaikan.com serve 5TB/month**: R2 ประหยัด ~$425/month เพราะ egress ฟรี

---

## ⚙️ Environment Setup

```bash
# ติดตั้ง AWS CLI (ถ้ายังไม่ติดตั้ง)
aws --version

# ตั้งค่า environment variables
export AWS_REGION="ap-southeast-1"
export BUCKET_NAME="chuaikan-media-production"
export BACKUP_BUCKET="chuaikan-media-backup-tokyo"
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

---

## 🛠️ Step-by-Step Implementation

### Step 554: สร้าง S3 Buckets

```bash
# สร้าง Main Bucket
aws s3api create-bucket \
  --bucket chuaikan-media-production \
  --region ap-southeast-1 \
  --create-bucket-configuration LocationConstraint=ap-southeast-1

# Block public access (เพราะ serve ผ่าน CloudFront)
aws s3api put-public-access-block \
  --bucket chuaikan-media-production \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# เปิด Versioning
aws s3api put-bucket-versioning \
  --bucket chuaikan-media-production \
  --versioning-configuration Status=Enabled

# เปิด Server-Side Encryption
aws s3api put-bucket-encryption \
  --bucket chuaikan-media-production \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms"
      },
      "BucketKeyEnabled": true
    }]
  }'

# สร้าง Backup Bucket ใน Tokyo สำหรับ DR
aws s3api create-bucket \
  --bucket chuaikan-media-backup-tokyo \
  --region ap-northeast-1 \
  --create-bucket-configuration LocationConstraint=ap-northeast-1
```

### Step 555: ตั้งค่า Bucket Policy

```bash
cat > bucket-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::chuaikan-media-production/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::${AWS_ACCOUNT_ID}:distribution/CLOUDFRONT_DIST_ID"
        }
      }
    },
    {
      "Sid": "AllowEKSNodeUpload",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::${AWS_ACCOUNT_ID}:role/chuaikan-production-app-role"
      },
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::chuaikan-media-production/*"
    },
    {
      "Sid": "AllowListBucket",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::${AWS_ACCOUNT_ID}:role/chuaikan-production-app-role"
      },
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::chuaikan-media-production"
    }
  ]
}
EOF

aws s3api put-bucket-policy \
  --bucket chuaikan-media-production \
  --policy file://bucket-policy.json
```

### Step 556: ตั้งค่า Lifecycle Rules

```bash
cat > lifecycle-rules.json << 'EOF'
{
  "Rules": [
    {
      "ID": "move-old-images-to-ia",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "images/"
      },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER_IR"
        },
        {
          "Days": 365,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ]
    },
    {
      "ID": "delete-temp-uploads",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "uploads/"
      },
      "Expiration": {
        "Days": 1
      }
    },
    {
      "ID": "delete-old-versions",
      "Status": "Enabled",
      "Filter": {},
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 30
      }
    },
    {
      "ID": "abort-incomplete-multipart",
      "Status": "Enabled",
      "Filter": {},
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      }
    }
  ]
}
EOF

aws s3api put-bucket-lifecycle-configuration \
  --bucket chuaikan-media-production \
  --lifecycle-configuration file://lifecycle-rules.json
```

### Step 557: ตั้งค่า CORS

```bash
cat > cors-config.json << 'EOF'
{
  "CORSRules": [
    {
      "AllowedOrigins": [
        "https://chuaikan.com",
        "https://www.chuaikan.com",
        "https://app.chuaikan.com"
      ],
      "AllowedMethods": ["GET", "PUT", "POST"],
      "AllowedHeaders": [
        "Content-Type",
        "Content-MD5",
        "Authorization",
        "x-amz-date",
        "x-amz-content-sha256",
        "x-amz-security-token"
      ],
      "ExposeHeaders": ["ETag"],
      "MaxAgeSeconds": 3600
    }
  ]
}
EOF

aws s3api put-bucket-cors \
  --bucket chuaikan-media-production \
  --cors-configuration file://cors-config.json
```

### Step 558: Presigned URLs สำหรับ Direct Upload

```typescript
// src/storage/presigned-upload.ts
import { 
  S3Client, 
  PutObjectCommand,
  GetObjectCommand 
} from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import crypto from 'crypto';

const s3Client = new S3Client({ region: 'ap-southeast-1' });
const BUCKET_NAME = process.env.S3_BUCKET_NAME || 'chuaikan-media-production';

interface PresignedUploadOptions {
  userId: string;
  contentType: string;
  fileExtension: string;
  maxSizeMB?: number;
}

interface PresignedUploadResult {
  uploadUrl: string;
  fileKey: string;
  expiresAt: Date;
}

// สร้าง Presigned URL สำหรับอัพโหลดตรงจาก browser
export async function createPresignedUpload(
  options: PresignedUploadOptions
): Promise<PresignedUploadResult> {
  const { userId, contentType, fileExtension, maxSizeMB = 10 } = options;
  
  // Validate content type
  const allowedTypes = ['image/jpeg', 'image/png', 'image/webp', 'image/gif'];
  if (!allowedTypes.includes(contentType)) {
    throw new Error(`Content type ${contentType} not allowed`);
  }
  
  // สร้าง unique file key
  const timestamp = Date.now();
  const randomId = crypto.randomBytes(8).toString('hex');
  const fileKey = `uploads/${userId}/${timestamp}-${randomId}.${fileExtension}`;
  
  const command = new PutObjectCommand({
    Bucket: BUCKET_NAME,
    Key: fileKey,
    ContentType: contentType,
    Metadata: {
      'uploaded-by': userId,
      'upload-timestamp': timestamp.toString()
    }
  });
  
  // URL expires in 5 minutes
  const uploadUrl = await getSignedUrl(s3Client, command, { expiresIn: 300 });
  
  return {
    uploadUrl,
    fileKey,
    expiresAt: new Date(Date.now() + 300 * 1000)
  };
}

// สร้าง Presigned URL สำหรับ download (สำหรับ private files)
export async function createPresignedDownload(
  fileKey: string,
  expiresInSeconds = 3600
): Promise<string> {
  const command = new GetObjectCommand({
    Bucket: BUCKET_NAME,
    Key: fileKey
  });
  
  return getSignedUrl(s3Client, command, { expiresIn: expiresInSeconds });
}

// API endpoint
export async function handleUploadRequest(req: any, res: any) {
  const { contentType, fileExtension } = req.body;
  const userId = req.user.id;
  
  try {
    const result = await createPresignedUpload({
      userId,
      contentType,
      fileExtension
    });
    
    res.json({
      uploadUrl: result.uploadUrl,
      fileKey: result.fileKey,
      expiresAt: result.expiresAt.toISOString()
    });
  } catch (error: any) {
    res.status(400).json({ error: error.message });
  }
}
```

### Step 559: S3 Cross-Region Replication

```bash
# สร้าง IAM Role สำหรับ Replication
cat > replication-role-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

REPLICATION_ROLE=$(aws iam create-role \
  --role-name S3ReplicationRole \
  --assume-role-policy-document file://replication-role-policy.json \
  --query 'Role.Arn' --output text)

cat > replication-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetReplicationConfiguration",
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::chuaikan-media-production"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObjectVersionForReplication",
        "s3:GetObjectVersionAcl",
        "s3:GetObjectVersionTagging"
      ],
      "Resource": "arn:aws:s3:::chuaikan-media-production/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ReplicateObject",
        "s3:ReplicateDelete",
        "s3:ReplicateTags"
      ],
      "Resource": "arn:aws:s3:::chuaikan-media-backup-tokyo/*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name S3ReplicationRole \
  --policy-name S3ReplicationPolicy \
  --policy-document file://replication-policy.json

# ตั้งค่า Replication
cat > replication-config.json << EOF
{
  "Role": "${REPLICATION_ROLE}",
  "Rules": [
    {
      "ID": "replicate-images",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "images/"
      },
      "Destination": {
        "Bucket": "arn:aws:s3:::chuaikan-media-backup-tokyo",
        "StorageClass": "STANDARD_IA"
      },
      "DeleteMarkerReplication": {
        "Status": "Enabled"
      }
    }
  ]
}
EOF

aws s3api put-bucket-replication \
  --bucket chuaikan-media-production \
  --replication-configuration file://replication-config.json
```

---

## 🔧 Configuration Files

### Step 560: Terraform S3 Module

```hcl
# terraform/modules/s3/main.tf

resource "aws_s3_bucket" "media" {
  bucket = "${var.project}-media-${var.environment}"

  tags = merge(var.tags, {
    Name = "${var.project}-media-${var.environment}"
  })

  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_versioning" "media" {
  bucket = aws_s3_bucket.media.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "media" {
  bucket = aws_s3_bucket.media.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "media" {
  bucket = aws_s3_bucket.media.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_lifecycle_configuration" "media" {
  bucket = aws_s3_bucket.media.id

  rule {
    id     = "move-to-ia"
    status = "Enabled"

    filter { prefix = "images/" }

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER_IR"
    }

    transition {
      days          = 365
      storage_class = "DEEP_ARCHIVE"
    }
  }

  rule {
    id     = "delete-temp-uploads"
    status = "Enabled"

    filter { prefix = "uploads/" }

    expiration { days = 1 }
  }

  rule {
    id     = "cleanup-old-versions"
    status = "Enabled"

    filter {}

    noncurrent_version_expiration {
      noncurrent_days = 30
    }

    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }
}

resource "aws_s3_bucket_cors_configuration" "media" {
  bucket = aws_s3_bucket.media.id

  cors_rule {
    allowed_headers = ["Content-Type", "Content-MD5", "Authorization", "x-amz-*"]
    allowed_methods = ["GET", "PUT", "POST"]
    allowed_origins = [
      "https://${var.domain}",
      "https://www.${var.domain}"
    ]
    expose_headers  = ["ETag"]
    max_age_seconds = 3600
  }
}

output "bucket_name" {
  value = aws_s3_bucket.media.bucket
}

output "bucket_arn" {
  value = aws_s3_bucket.media.arn
}

output "bucket_domain_name" {
  value = aws_s3_bucket.media.bucket_regional_domain_name
}
```

---

## 🧪 Testing

```bash
# ทดสอบอัพโหลด
aws s3 cp test-image.jpg s3://chuaikan-media-production/images/test/test-image.jpg

# ทดสอบดาวน์โหลด
aws s3 cp s3://chuaikan-media-production/images/test/test-image.jpg /tmp/

# ทดสอบ Presigned URL
aws s3 presign s3://chuaikan-media-production/images/test/test-image.jpg \
  --expires-in 300

# ตรวจสอบ lifecycle rules
aws s3api get-bucket-lifecycle-configuration \
  --bucket chuaikan-media-production

# ตรวจสอบ replication status
aws s3api get-bucket-replication \
  --bucket chuaikan-media-production

# ดู storage usage
aws s3api list-objects-v2 \
  --bucket chuaikan-media-production \
  --query 'sum(Contents[].Size)' \
  --output text | awk '{printf "%.2f GB\n", $1/1024/1024/1024}'
```

---

## ❌ Common Errors & Solutions

### Error 1: Access Denied เมื่อ CloudFront ดึง S3

```
AccessDenied: Access Denied
```

**แก้ไข:**
```bash
# ตรวจสอบ Bucket Policy
aws s3api get-bucket-policy --bucket chuaikan-media-production

# ตรวจสอบ CloudFront Origin Access Control (OAC) ID ถูกต้อง
# และ Bucket Policy อนุญาต CloudFront service principal
```

### Error 2: CORS error เมื่อ upload จาก browser

```
Access to fetch at 'https://s3.amazonaws.com/...' from origin 'https://chuaikan.com' 
has been blocked by CORS policy
```

**แก้ไข:**
```bash
# ตรวจสอบ CORS configuration
aws s3api get-bucket-cors --bucket chuaikan-media-production

# ตรวจสอบว่า domain ตรงกับ AllowedOrigins
```

---

## 📊 Cost Estimation

```
1TB Storage:
├── S3 Standard (Hot, 0-30 days, ~200GB): $4.60/month
├── S3 Standard-IA (30-90 days, ~300GB): $3.75/month
├── S3 Glacier IR (90-365 days, ~400GB): $1.60/month
└── S3 Deep Archive (>365 days, ~100GB): $0.10/month

Data Transfer (Egress):
├── Via CloudFront: $0.085/GB × 5,000GB = $425/month
├── Via Cloudflare R2: FREE
└── ประหยัดด้วย R2: ~$425/month

Operations:
├── PUT requests 100K: $0.50/month
└── GET requests 10M: $4.00/month

TOTAL (S3 + CloudFront): ~$440/month
TOTAL (R2 + Cloudflare): ~$20/month (ประหยัดมาก!)
```

---

## ✅ Checklist

- [ ] Step 551: เข้าใจ S3 Architecture สำหรับ media storage
- [ ] Step 552: วางแผน Storage Classes และ Lifecycle
- [ ] Step 553: เปรียบเทียบ S3 vs Cloudflare R2
- [ ] Step 554: สร้าง S3 Buckets พร้อม security settings
- [ ] Step 555: ตั้งค่า Bucket Policy
- [ ] Step 556: ตั้งค่า Lifecycle Rules
- [ ] Step 557: ตั้งค่า CORS Configuration
- [ ] Step 558: สร้าง Presigned URL สำหรับ direct upload
- [ ] Step 559: ตั้งค่า Cross-Region Replication
- [ ] Step 560: เขียน Terraform Module สำหรับ S3

---

## 🔗 References

- [S3 User Guide](https://docs.aws.amazon.com/AmazonS3/latest/userguide/)
- [S3 Presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [Cloudflare R2](https://developers.cloudflare.com/r2/)
- [S3 Lifecycle Configuration](https://docs.aws.amazon.com/AmazonS3/latest/userguide/lifecycle-configuration-examples.html)

---
*Part 056 | Road to 1,000,000 Users/Day | chuaikan.com*
