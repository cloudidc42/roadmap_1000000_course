# Part 057: CloudFront / Cloud CDN
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Advanced
> **Steps:** 561-570
> **เวลาโดยประมาณ:** 5 ชั่วโมง
> **Prerequisites:** Part 056 (S3), Part 053 (EKS), Part 052 (VPC)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ตั้งค่า CloudFront Distribution สำหรับ chuaikan.com
- ตั้งค่า Origins: S3 (media), ALB (web app), API Gateway
- ตั้งค่า Cache Behaviors สำหรับ path patterns ต่างๆ
- ใช้ Lambda@Edge สำหรับ URL rewriting และ auth
- เปรียบเทียบ CloudFront Functions vs Lambda@Edge
- ตั้งค่า Geographic Restrictions
- Integrate กับ WAF
- ตั้งค่า Real-time Logs ไปยัง Kinesis
- Cache Invalidation API

---

## 📖 ทฤษฎีและแนวคิด

### Step 561: CloudFront Architecture สำหรับ chuaikan.com

```
User Request Flow:

1. DNS (Route 53) → CloudFront Edge Location
2. CloudFront ตรวจสอบ Cache:
   - HIT: ส่ง response จาก cache (latency <5ms)
   - MISS: ดึงจาก Origin

CloudFront Origins:
┌─────────────────────────────────────────────────────┐
│  CloudFront Distribution                            │
│  Domain: d1234567890.cloudfront.net               │
│  Custom Domain: chuaikan.com                      │
│                                                      │
│  Cache Behaviors (Priority Order):                  │
│  ┌─────────────────────────────────────────────┐  │
│  │ /api/*  → ALB Origin (No Cache)            │  │
│  │ /static/* → S3 Origin (Cache 1 year)       │  │
│  │ /images/* → S3 Origin (Cache 1 week)       │  │
│  │ /* (default) → ALB Origin (Cache 5 min)    │  │
│  └─────────────────────────────────────────────┘  │
│                                                      │
│  Origins:                                            │
│  ├── S3 Origin (media, static files)               │
│  ├── ALB Origin (web app, API)                     │
│  └── API GW Origin (Lambda functions)              │
└─────────────────────────────────────────────────────┘
```

### Step 562: Cache Behavior Design

```
Path Pattern    Origin    TTL         Cache Policy
/api/*          ALB       0 (No cache)  None
/auth/*         ALB       0 (No cache)  None
/static/*       S3        31536000 (1yr) Immutable  
/images/*       S3        604800 (1wk)  CachingOptimized
/thumbnails/*   S3        604800 (1wk)  CachingOptimized
/avatars/*      S3        86400 (1day)  CachingOptimized
/* (default)    ALB       300 (5min)    Standard
```

### Step 563: CloudFront Functions vs Lambda@Edge

| Feature | CloudFront Functions | Lambda@Edge |
|---------|---------------------|-------------|
| Execution | CloudFront edge (~45 PoPs) | Lambda regional edge |
| Runtime | JavaScript (V8) | Node.js 18, Python 3.11 |
| Timeout | 1ms | 5s (viewer), 30s (origin) |
| Memory | 2MB code | 128MB - 10GB |
| Cost | $0.10/million | $0.60/million + $0.00005001/GB-sec |
| Use case | Simple rewrites, auth | Complex logic |
| Network access | ไม่มี | ได้ |

**สรุป:**
- CloudFront Functions: URL rewriting, simple header manipulation
- Lambda@Edge: JWT validation, A/B testing, personalization

---

## ⚙️ Environment Setup

```bash
# ตรวจสอบ AWS Certificate Manager (ACM) - ต้องอยู่ใน us-east-1
aws acm list-certificates --region us-east-1

# ถ้ายังไม่มี certificate
aws acm request-certificate \
  --domain-name chuaikan.com \
  --subject-alternative-names "*.chuaikan.com" \
  --validation-method DNS \
  --region us-east-1  # CloudFront ต้องการ cert ที่ us-east-1
```

---

## 🛠️ Step-by-Step Implementation

### Step 564: สร้าง Origin Access Control สำหรับ S3

```bash
# สร้าง OAC (แทน OAI ที่เก่าแล้ว)
OAC_ID=$(aws cloudfront create-origin-access-control \
  --origin-access-control-config '{
    "Name": "chuaikan-s3-oac",
    "Description": "OAC for chuaikan S3 bucket",
    "SigningProtocol": "sigv4",
    "SigningBehavior": "always",
    "OriginAccessControlOriginType": "s3"
  }' \
  --query 'OriginAccessControl.Id' --output text)

echo "OAC ID: $OAC_ID"
```

### Step 565: สร้าง CloudFront Distribution

```bash
# สร้าง CloudFront Distribution
CERT_ARN=$(aws acm list-certificates --region us-east-1 \
  --query 'CertificateSummaryList[?DomainName==`chuaikan.com`].CertificateArn' \
  --output text)

aws cloudfront create-distribution \
  --distribution-config file://cloudfront-config.json
```

### Step 566: Lambda@Edge สำหรับ Auth Check

```javascript
// lambda-edge/auth-check/index.mjs
// Deploy ไปยัง us-east-1 (Lambda@Edge requirement)

import jwt from 'jsonwebtoken';

const JWT_SECRET = process.env.JWT_SECRET || 'your-jwt-secret';
const PUBLIC_PATHS = ['/api/v1/auth', '/api/v1/dogs', '/api/v1/health'];

export const handler = async (event) => {
  const request = event.Records[0].cf.request;
  const headers = request.headers;
  const uri = request.uri;
  
  // Skip auth for public paths
  if (PUBLIC_PATHS.some(path => uri.startsWith(path))) {
    return request;
  }
  
  // Skip auth for static assets
  if (uri.match(/\.(js|css|png|jpg|gif|ico|woff2?)$/)) {
    return request;
  }
  
  // Check Authorization header
  const authHeader = headers.authorization?.[0]?.value;
  
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return {
      status: '401',
      statusDescription: 'Unauthorized',
      headers: {
        'content-type': [{ key: 'Content-Type', value: 'application/json' }],
        'www-authenticate': [{ key: 'WWW-Authenticate', value: 'Bearer' }]
      },
      body: JSON.stringify({ error: 'Unauthorized', message: 'Missing or invalid token' })
    };
  }
  
  const token = authHeader.substring(7);
  
  try {
    const decoded = jwt.verify(token, JWT_SECRET);
    
    // Pass user info to origin via custom header
    request.headers['x-user-id'] = [{ key: 'X-User-Id', value: decoded.userId }];
    request.headers['x-user-role'] = [{ key: 'X-User-Role', value: decoded.role }];
    
    return request;
  } catch (err) {
    return {
      status: '401',
      statusDescription: 'Unauthorized',
      headers: {
        'content-type': [{ key: 'Content-Type', value: 'application/json' }]
      },
      body: JSON.stringify({ error: 'Unauthorized', message: 'Invalid or expired token' })
    };
  }
};
```

### Step 567: CloudFront Function สำหรับ URL Rewriting

```javascript
// cloudfront-functions/url-rewrite.js
// ทำงานที่ Viewer Request stage

function handler(event) {
  var request = event.request;
  var uri = request.uri;
  
  // เพิ่ม /index.html สำหรับ SPA routing
  // ถ้า URI ไม่มี extension และไม่ใช่ API
  if (!uri.includes('/api/') && 
      !uri.match(/\.[a-zA-Z0-9]+$/) && 
      uri !== '/') {
    // ตรวจสอบว่าไม่ใช่ directory path
    if (!uri.endsWith('/')) {
      uri = uri + '/';
    }
    request.uri = uri + 'index.html';
  }
  
  // Redirect www ไป non-www
  var host = request.headers.host?.value || '';
  if (host.startsWith('www.')) {
    return {
      statusCode: 301,
      statusDescription: 'Moved Permanently',
      headers: {
        location: { value: 'https://' + host.replace('www.', '') + uri }
      }
    };
  }
  
  return request;
}
```

---

## 🔧 Configuration Files

### CloudFront Distribution Config

```json
{
  "CallerReference": "chuaikan-dist-2024",
  "Comment": "chuaikan.com CloudFront Distribution",
  "DefaultCacheBehavior": {
    "TargetOriginId": "alb-origin",
    "ViewerProtocolPolicy": "redirect-to-https",
    "AllowedMethods": ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"],
    "CachedMethods": ["GET", "HEAD"],
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "OriginRequestPolicyId": "216adef6-5c7f-47e4-b989-5492eafa07d3",
    "Compress": true
  },
  "CacheBehaviors": {
    "Quantity": 4,
    "Items": [
      {
        "PathPattern": "/api/*",
        "TargetOriginId": "alb-origin",
        "ViewerProtocolPolicy": "https-only",
        "AllowedMethods": ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"],
        "CachePolicyId": "4135ea2d-6df8-44a3-9df3-4b5a84be39ad",
        "Compress": true,
        "FunctionAssociations": {
          "Quantity": 1,
          "Items": [
            {
              "FunctionARN": "arn:aws:cloudfront::123456789012:function/url-rewrite",
              "EventType": "viewer-request"
            }
          ]
        }
      },
      {
        "PathPattern": "/static/*",
        "TargetOriginId": "s3-origin",
        "ViewerProtocolPolicy": "https-only",
        "AllowedMethods": ["GET", "HEAD"],
        "CachePolicyId": "b2884449-e4de-46a7-ac36-70bc7f1ddd6d",
        "Compress": true
      },
      {
        "PathPattern": "/images/*",
        "TargetOriginId": "s3-origin",
        "ViewerProtocolPolicy": "https-only",
        "AllowedMethods": ["GET", "HEAD"],
        "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
        "Compress": true
      }
    ]
  },
  "Origins": {
    "Quantity": 2,
    "Items": [
      {
        "Id": "s3-origin",
        "DomainName": "chuaikan-media-production.s3.ap-southeast-1.amazonaws.com",
        "S3OriginConfig": {
          "OriginAccessIdentity": ""
        },
        "OriginAccessControlId": "OACI_ID_HERE"
      },
      {
        "Id": "alb-origin",
        "DomainName": "alb-xxxxx.ap-southeast-1.elb.amazonaws.com",
        "CustomOriginConfig": {
          "HTTPSPort": 443,
          "OriginProtocolPolicy": "https-only",
          "OriginSSLProtocols": {
            "Quantity": 1,
            "Items": ["TLSv1.2"]
          }
        }
      }
    ]
  },
  "Aliases": {
    "Quantity": 2,
    "Items": ["chuaikan.com", "www.chuaikan.com"]
  },
  "ViewerCertificate": {
    "ACMCertificateArn": "arn:aws:acm:us-east-1:123456789012:certificate/xxxxx",
    "SSLSupportMethod": "sni-only",
    "MinimumProtocolVersion": "TLSv1.2_2021"
  },
  "HttpVersion": "http2and3",
  "IsIPV6Enabled": true,
  "PriceClass": "PriceClass_200",
  "Enabled": true,
  "Logging": {
    "Enabled": true,
    "Bucket": "chuaikan-cloudfront-logs.s3.amazonaws.com",
    "Prefix": "cloudfront/"
  }
}
```

### Cache Invalidation Script

```bash
#!/bin/bash
# scripts/invalidate-cache.sh

DISTRIBUTION_ID="E1234567890ABC"
PATHS=${1:-"/*"}

echo "Creating CloudFront invalidation for: $PATHS"

INVALIDATION_ID=$(aws cloudfront create-invalidation \
  --distribution-id $DISTRIBUTION_ID \
  --paths $PATHS \
  --query 'Invalidation.Id' --output text)

echo "Invalidation ID: $INVALIDATION_ID"

# รอการ invalidation เสร็จ
aws cloudfront wait invalidation-completed \
  --distribution-id $DISTRIBUTION_ID \
  --id $INVALIDATION_ID

echo "Cache invalidation completed!"
```

---

## 🧪 Testing

```bash
# ทดสอบ distribution
DIST_ID="E1234567890ABC"

# ดู distribution status
aws cloudfront get-distribution --id $DIST_ID \
  --query 'Distribution.{Status:Status,Domain:DomainName}'

# ทดสอบ cache headers
curl -I https://chuaikan.com/images/dogs/123/thumbnail.jpg | grep -i cache

# ทดสอบ latency จาก Thailand (ควรได้จาก Singapore PoP)
curl -w "\nTotal time: %{time_total}s\nConnect time: %{time_connect}s\n" \
  -o /dev/null -s https://chuaikan.com/images/dogs/123/thumbnail.jpg

# ดู CloudFront metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/CloudFront \
  --metric-name CacheHitRate \
  --dimensions Name=DistributionId,Value=$DIST_ID \
              Name=Region,Value=Global \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Average

# Invalidate specific path
aws cloudfront create-invalidation \
  --distribution-id $DIST_ID \
  --paths "/images/dogs/123/*"
```

---

## ❌ Common Errors & Solutions

### Error 1: 403 Access Denied จาก S3 Origin

```
<Code>AccessDenied</Code>
<Message>Access Denied</Message>
```

**แก้ไข:**
```bash
# ตรวจสอบ OAC assignment
aws cloudfront get-distribution --id $DIST_ID \
  --query 'Distribution.DistributionConfig.Origins.Items[*].{Id:Id,OAC:OriginAccessControlId}'

# Update S3 bucket policy ให้ CloudFront เข้าถึงได้
aws s3api put-bucket-policy \
  --bucket chuaikan-media-production \
  --policy file://bucket-policy.json
```

### Error 2: HTTPS Certificate Error

```
SSL_ERROR_RX_RECORD_TOO_LONG
```

**แก้ไข:**
```bash
# ตรวจสอบ ALB listener ใช้ HTTPS
aws elbv2 describe-listeners \
  --load-balancer-arn arn:aws:elasticloadbalancing:...

# CloudFront origin ต้องใช้ HTTPS เท่านั้น
# ตรวจสอบ Origin Protocol Policy
```

---

## 📊 Cost Estimation

```
CloudFront (5TB/month egress):
├── HTTP Requests 100M: $0.01/10K = $100/month
├── HTTPS Requests 100M: $0.012/10K = $120/month
├── Data Transfer 5TB to Asia: $0.085/GB × 5000 = $425/month
└── Lambda@Edge 50M invocations: $0.60/M × 50 = $30/month

TOTAL CloudFront: ~$675/month

vs Cloudflare Pro ($20/month ไม่จำกัด bandwidth):
ประหยัด: ~$655/month!
```

---

## ✅ Checklist

- [ ] Step 561: เข้าใจ CloudFront Architecture
- [ ] Step 562: วางแผน Cache Behaviors
- [ ] Step 563: เปรียบเทียบ CloudFront Functions vs Lambda@Edge
- [ ] Step 564: สร้าง Origin Access Control สำหรับ S3
- [ ] Step 565: สร้าง CloudFront Distribution
- [ ] Step 566: Deploy Lambda@Edge สำหรับ JWT auth
- [ ] Step 567: สร้าง CloudFront Function สำหรับ URL rewriting
- [ ] Step 568: ตั้งค่า WAF integration
- [ ] Step 569: ตั้งค่า Real-time logs
- [ ] Step 570: ทดสอบ cache hit rate และ latency

---

## 🔗 References

- [CloudFront Developer Guide](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/)
- [Lambda@Edge Documentation](https://docs.aws.amazon.com/lambda/latest/dg/lambda-edge.html)
- [CloudFront Functions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-functions.html)
- [CloudFront Cache Policies](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-managed-cache-policies.html)

---
*Part 057 | Road to 1,000,000 Users/Day | chuaikan.com*
