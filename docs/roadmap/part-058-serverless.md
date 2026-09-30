# Part 058: Lambda / Cloud Functions (Serverless)
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Advanced
> **Steps:** 571-580
> **เวลาโดยประมาณ:** 5 ชั่วโมง
> **Prerequisites:** Part 056 (S3), Part 057 (CloudFront), Part 051 (AWS Account)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- Lambda use cases สำหรับ chuaikan.com
- Lambda function สำหรับ image thumbnail generation (Node.js 22, Sharp)
- Lambda@Edge สำหรับ auth check
- API Gateway integration
- Cold start optimization ด้วย Provisioned Concurrency
- Lambda Layers สำหรับ shared dependencies
- Environment variables และ Secrets
- Lambda Monitoring (CloudWatch, X-Ray)
- ประมาณค่าใช้จ่าย 1M invocations/month

---

## 📖 ทฤษฎีและแนวคิด

### Step 571: Lambda Use Cases สำหรับ chuaikan.com

```
Serverless Architecture สำหรับ chuaikan.com:

1. Image Processing (S3 Trigger):
   S3 Upload → Lambda (resize) → S3 (thumbnails)
   
2. Email Notifications (SQS Trigger):
   SQS Queue → Lambda (send email via SES)
   
3. Scheduled Tasks (EventBridge):
   Cron → Lambda (daily digest, cleanup)
   
4. Webhook Processing (API Gateway):
   Webhook → API GW → Lambda → Database
   
5. Auth Check (Lambda@Edge):
   CloudFront → Lambda@Edge → Validate JWT
   
6. Image Optimization (CloudFront):
   CloudFront → S3 Object Lambda → Return optimized image
```

### Step 572: Lambda Execution Model

```
Cold Start (แรกสุด หรือ หลัง idle):
┌─────────────────────────────────────────────┐
│  Init Phase (~500ms - 3s):                   │
│  ├── Download code package                   │
│  ├── Start runtime (Node.js)                 │
│  ├── Execute module-level code               │
│  └── Initialize SDK clients                  │
│                                              │
│  Invoke Phase:                               │
│  └── Execute handler function                │
└─────────────────────────────────────────────┘

Warm Start (container ยังอยู่):
┌─────────────────────────────────────────────┐
│  Invoke Phase only (~1-50ms):                │
│  └── Execute handler function                │
└─────────────────────────────────────────────┘

ลด Cold Start:
├── Provisioned Concurrency (ค่าใช้จ่ายเพิ่ม)
├── เพิ่ม memory (CPU ก็เพิ่มตาม)
├── ใช้ Lambda Snap Start (Java)
└── Init SDK clients นอก handler function
```

### Step 573: Lambda Pricing

```
Requests: $0.20 per million requests

Duration:
├── First 6B GB-seconds/month: $0.0000166667/GB-sec
└── Next 15B GB-seconds/month: $0.0000133334/GB-sec

ตัวอย่าง 1M invocations × 3s × 512MB:
├── Requests: $0.20
├── Duration: 1,000,000 × 3 × 0.5GB × $0.0000166667 = $25.00
└── Total: ~$25.20/month

Free Tier (ทุกเดือน):
├── 1M requests
└── 400,000 GB-seconds
```

---

## ⚙️ Environment Setup

### ติดตั้ง AWS SAM CLI

```bash
# Linux
pip install aws-sam-cli

# macOS
brew install aws/tap/aws-sam-cli

# ตรวจสอบ
sam --version
# SAM CLI, version 1.100.0
```

---

## 🛠️ Step-by-Step Implementation

### Step 574: Image Thumbnail Lambda

```bash
# สร้าง project structure
mkdir -p lambda/image-processor
cd lambda/image-processor

# package.json
cat > package.json << 'EOF'
{
  "name": "image-processor",
  "version": "1.0.0",
  "description": "Image thumbnail generator for chuaikan.com",
  "main": "index.mjs",
  "type": "module",
  "dependencies": {
    "sharp": "^0.33.2",
    "@aws-sdk/client-s3": "^3.500.0"
  }
}
EOF

npm install
```

```javascript
// lambda/image-processor/index.mjs
import { S3Client, GetObjectCommand, PutObjectCommand } from '@aws-sdk/client-s3';
import sharp from 'sharp';
import { Readable } from 'stream';

const s3Client = new S3Client({ region: process.env.AWS_REGION || 'ap-southeast-1' });

const THUMBNAIL_CONFIGS = [
  { suffix: 'thumbnail', width: 150, height: 150, fit: 'cover' },
  { suffix: 'medium', width: 600, height: null, fit: 'inside' },
  { suffix: 'large', width: 1200, height: null, fit: 'inside' }
];

async function streamToBuffer(stream) {
  const chunks = [];
  for await (const chunk of stream) {
    chunks.push(chunk);
  }
  return Buffer.concat(chunks);
}

export const handler = async (event) => {
  const results = [];

  for (const record of event.Records) {
    const bucket = record.s3.bucket.name;
    const key = decodeURIComponent(record.s3.object.key.replace(/\+/g, ' '));
    
    // Skip if not in uploads folder
    if (!key.startsWith('uploads/')) {
      console.log(`Skipping non-upload file: ${key}`);
      continue;
    }
    
    console.log(`Processing: ${bucket}/${key}`);
    
    try {
      // Download original image
      const getCommand = new GetObjectCommand({ Bucket: bucket, Key: key });
      const response = await s3Client.send(getCommand);
      const originalBuffer = await streamToBuffer(response.Body);
      
      // Extract path info
      // uploads/{userId}/{timestamp}-{randomId}.{ext}
      const pathParts = key.split('/');
      const userId = pathParts[1];
      const filename = pathParts[2];
      const ext = filename.split('.').pop();
      const baseKey = filename.replace(`.${ext}`, '');
      
      // Generate unique dog ID from timestamp
      const dogId = baseKey.split('-')[0];
      
      // Process each thumbnail size
      for (const config of THUMBNAIL_CONFIGS) {
        const sharpInstance = sharp(originalBuffer)
          .resize({
            width: config.width,
            height: config.height,
            fit: config.fit,
            withoutEnlargement: true
          })
          .webp({ quality: 85 }); // Convert to WebP
        
        const processedBuffer = await sharpInstance.toBuffer();
        
        // Save to images folder
        const destKey = `images/${dogId}/${config.suffix}.webp`;
        
        await s3Client.send(new PutObjectCommand({
          Bucket: bucket,
          Key: destKey,
          Body: processedBuffer,
          ContentType: 'image/webp',
          CacheControl: 'max-age=604800', // 1 week
          Metadata: {
            'original-key': key,
            'user-id': userId,
            'processed-at': new Date().toISOString()
          }
        }));
        
        console.log(`Created: ${destKey}`);
        results.push({ key: destKey, size: processedBuffer.length });
      }
      
      // Also save optimized original
      const originalWebP = await sharp(originalBuffer)
        .webp({ quality: 90 })
        .toBuffer();
      
      await s3Client.send(new PutObjectCommand({
        Bucket: bucket,
        Key: `images/${dogId}/original.webp`,
        Body: originalWebP,
        ContentType: 'image/webp'
      }));
      
    } catch (error) {
      console.error(`Error processing ${key}:`, error);
      throw error; // Re-throw to trigger retry
    }
  }
  
  return { processed: results.length, files: results };
};
```

### Step 575: สร้าง Lambda Deployment Package

```bash
# สร้าง deployment package
cd lambda/image-processor

# Build สำหรับ Lambda Linux environment
npm ci --platform=linux --arch=x64 --libc=glibc

# Create zip
zip -r9 ../image-processor.zip . -x "*.test.js" -x "*.md"

# Deploy Lambda
aws lambda create-function \
  --function-name chuaikan-image-processor \
  --runtime nodejs22.x \
  --role arn:aws:iam::123456789012:role/lambda-image-processor-role \
  --handler index.handler \
  --zip-file fileb://../image-processor.zip \
  --timeout 60 \
  --memory-size 1024 \
  --environment Variables='{
    "NODE_ENV": "production",
    "AWS_REGION": "ap-southeast-1"
  }' \
  --layers "arn:aws:lambda:ap-southeast-1:123456789012:layer:sharp-layer:1" \
  --tracing-config Mode=Active \
  --tags Environment=production,Project=chuaikan

# ตั้งค่า S3 trigger
aws lambda add-permission \
  --function-name chuaikan-image-processor \
  --principal s3.amazonaws.com \
  --statement-id s3-trigger \
  --action lambda:InvokeFunction \
  --source-arn arn:aws:s3:::chuaikan-media-production \
  --source-account 123456789012

# เพิ่ม S3 notification
aws s3api put-bucket-notification-configuration \
  --bucket chuaikan-media-production \
  --notification-configuration '{
    "LambdaFunctionConfigurations": [{
      "LambdaFunctionArn": "arn:aws:lambda:ap-southeast-1:123456789012:function:chuaikan-image-processor",
      "Events": ["s3:ObjectCreated:*"],
      "Filter": {
        "Key": {
          "FilterRules": [{
            "Name": "prefix",
            "Value": "uploads/"
          }]
        }
      }
    }]
  }'
```

### Step 576: Lambda Layer สำหรับ Sharp

```bash
# สร้าง Sharp Layer (เพราะ Sharp ต้องการ native binaries)
mkdir -p sharp-layer/nodejs
cd sharp-layer

# ติดตั้ง sharp สำหรับ Lambda environment
npm install --prefix nodejs sharp --platform=linux --arch=x64 --libc=glibc

# Zip layer
zip -r9 sharp-layer.zip nodejs/

# Create Lambda Layer
LAYER_ARN=$(aws lambda publish-layer-version \
  --layer-name sharp-layer \
  --description "Sharp image processing library for Lambda" \
  --zip-file fileb://sharp-layer.zip \
  --compatible-runtimes nodejs20.x nodejs22.x \
  --compatible-architectures x86_64 \
  --query 'LayerVersionArn' --output text)

echo "Layer ARN: $LAYER_ARN"
```

### Step 577: Email Notification Lambda

```javascript
// lambda/email-sender/index.mjs
import { SESClient, SendEmailCommand } from '@aws-sdk/client-ses';
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';

const sesClient = new SESClient({ region: 'ap-southeast-1' });

// Cache secrets (warm start reuse)
let emailConfig = null;

async function getEmailConfig() {
  if (emailConfig) return emailConfig;
  
  const secretsClient = new SecretsManagerClient({ region: 'ap-southeast-1' });
  const command = new GetSecretValueCommand({
    SecretId: process.env.EMAIL_CONFIG_SECRET_ARN
  });
  const response = await secretsClient.send(command);
  emailConfig = JSON.parse(response.SecretString);
  return emailConfig;
}

export const handler = async (event) => {
  const config = await getEmailConfig();
  const results = [];
  
  for (const record of event.Records) {
    const message = JSON.parse(record.body);
    
    try {
      const emailCommand = new SendEmailCommand({
        Source: `chuaikan.com <noreply@chuaikan.com>`,
        Destination: {
          ToAddresses: [message.to]
        },
        Message: {
          Subject: { Data: message.subject },
          Body: {
            Html: { Data: message.htmlBody },
            Text: { Data: message.textBody }
          }
        },
        Tags: [
          { Name: 'EmailType', Value: message.type || 'notification' }
        ]
      });
      
      const response = await sesClient.send(emailCommand);
      results.push({ success: true, messageId: response.MessageId, to: message.to });
      
    } catch (error) {
      console.error('Failed to send email:', error);
      results.push({ success: false, error: error.message, to: message.to });
      // Re-throw to send back to DLQ
      throw error;
    }
  }
  
  return { results };
};
```

### Step 578: Provisioned Concurrency

```bash
# Publish Lambda version
VERSION=$(aws lambda publish-version \
  --function-name chuaikan-image-processor \
  --description "v1.0.0 - Initial deployment" \
  --query 'Version' --output text)

# สร้าง alias
aws lambda create-alias \
  --function-name chuaikan-image-processor \
  --name production \
  --function-version $VERSION \
  --description "Production alias"

# ตั้งค่า Provisioned Concurrency (5 instances พร้อมเสมอ)
aws lambda put-provisioned-concurrency-config \
  --function-name chuaikan-image-processor \
  --qualifier production \
  --provisioned-concurrent-executions 5

# ตรวจสอบ status
aws lambda get-provisioned-concurrency-config \
  --function-name chuaikan-image-processor \
  --qualifier production
```

---

## 🔧 Configuration Files

### SAM Template

```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Runtime: nodejs22.x
    Timeout: 30
    MemorySize: 512
    Environment:
      Variables:
        NODE_ENV: !Ref Environment
        AWS_REGION: !Ref AWS::Region
    Tracing: Active
    Layers:
      - !Ref SharedLayer

Parameters:
  Environment:
    Type: String
    AllowedValues: [development, staging, production]
    Default: production

Resources:
  # Shared Lambda Layer
  SharedLayer:
    Type: AWS::Serverless::LayerVersion
    Properties:
      LayerName: !Sub chuaikan-shared-${Environment}
      ContentUri: layers/shared/
      CompatibleRuntimes:
        - nodejs22.x
    Metadata:
      BuildMethod: nodejs22.x

  # Image Processor
  ImageProcessor:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: !Sub chuaikan-image-processor-${Environment}
      CodeUri: lambda/image-processor/
      Handler: index.handler
      MemorySize: 1024
      Timeout: 60
      Layers:
        - !Ref SharpLayer
      Policies:
        - S3CrudPolicy:
            BucketName: !Sub chuaikan-media-${Environment}
      Events:
        S3Upload:
          Type: S3
          Properties:
            Bucket: !Sub chuaikan-media-${Environment}
            Events: s3:ObjectCreated:*
            Filter:
              S3Key:
                Rules:
                  - Name: prefix
                    Value: uploads/

  # Email Sender
  EmailSender:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: !Sub chuaikan-email-sender-${Environment}
      CodeUri: lambda/email-sender/
      Handler: index.handler
      Policies:
        - SESCrudPolicy:
            IdentityName: chuaikan.com
        - SQSPollerPolicy:
            QueueName: !GetAtt EmailQueue.QueueName
      Events:
        SQSEvent:
          Type: SQS
          Properties:
            Queue: !GetAtt EmailQueue.Arn
            BatchSize: 10
            FunctionResponseTypes:
              - ReportBatchItemFailures

  # SQS Queue for emails
  EmailQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub chuaikan-emails-${Environment}.fifo
      FifoQueue: true
      ContentBasedDeduplication: true
      MessageRetentionPeriod: 86400
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt EmailDLQ.Arn
        maxReceiveCount: 3

  EmailDLQ:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub chuaikan-emails-dlq-${Environment}.fifo
      FifoQueue: true

  # Scheduled cleanup
  CleanupFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: !Sub chuaikan-cleanup-${Environment}
      CodeUri: lambda/cleanup/
      Handler: index.handler
      Policies:
        - S3CrudPolicy:
            BucketName: !Sub chuaikan-media-${Environment}
      Events:
        ScheduledCleanup:
          Type: ScheduleV2
          Properties:
            ScheduleExpression: cron(0 0 * * ? *)  # Daily at midnight UTC
            Description: Clean up temporary uploads

Outputs:
  ImageProcessorArn:
    Value: !GetAtt ImageProcessor.Arn
  EmailSenderArn:
    Value: !GetAtt EmailSender.Arn
  EmailQueueUrl:
    Value: !Ref EmailQueue
```

---

## 🧪 Testing

```bash
# ทดสอบ Lambda ด้วย test event
aws lambda invoke \
  --function-name chuaikan-image-processor \
  --payload file://test-event.json \
  --log-type Tail \
  --query 'LogResult' --output text | base64 --decode \
  /dev/null

# ดู CloudWatch Logs
aws logs get-log-events \
  --log-group-name /aws/lambda/chuaikan-image-processor \
  --log-stream-name $(aws logs describe-log-streams \
    --log-group-name /aws/lambda/chuaikan-image-processor \
    --order-by LastEventTime \
    --descending --limit 1 \
    --query 'logStreams[0].logStreamName' --output text)

# ดู X-Ray traces
aws xray get-trace-summaries \
  --start-time $(date -u -d '1 hour ago' +%s) \
  --end-time $(date -u +%s) \
  --filter-expression "service(\"chuaikan-image-processor\")"

# Load test Lambda
for i in {1..100}; do
  aws lambda invoke \
    --function-name chuaikan-image-processor \
    --payload file://test-event.json \
    /tmp/response_$i.json &
done
wait

# ดู Cold Start metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name InitDuration \
  --dimensions Name=FunctionName,Value=chuaikan-image-processor \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 --statistics Average,Maximum
```

---

## ❌ Common Errors & Solutions

### Error 1: Task timed out

```
Task timed out after 30.00 seconds
```

**แก้ไข:**
```bash
# เพิ่ม timeout
aws lambda update-function-configuration \
  --function-name chuaikan-image-processor \
  --timeout 60

# หรือ optimize code
# - ใช้ streaming แทน buffer ถ้าไฟล์ใหญ่
# - เพิ่ม memory (CPU เพิ่มด้วย)
```

### Error 2: ENOMEM / Out of Memory

```
Runtime exited with error: signal: killed
```

**แก้ไข:**
```bash
# เพิ่ม memory allocation
aws lambda update-function-configuration \
  --function-name chuaikan-image-processor \
  --memory-size 2048

# ตรวจสอบว่า Sharp ไม่ cache image ทั้งหมดใน memory
```

### Error 3: Sharp binary ไม่ compatible

```
Error: /lib/x86_64-linux-gnu/libm.so.6: version 'GLIBC_2.29' not found
```

**แก้ไข:**
```bash
# Build sharp สำหรับ Lambda environment โดยเฉพาะ
npm install --platform=linux --arch=x64 --libc=glibc sharp

# หรือใช้ Docker เพื่อ build
docker run --rm \
  -v $(pwd):/var/task \
  public.ecr.aws/lambda/nodejs:22 \
  npm install
```

---

## 📊 Cost Estimation

```
Image Processor Lambda (50,000 images/day):
├── Invocations: 50,000/day × 30 days = 1.5M/month
├── Duration: 1.5M × 5s × 1GB = 7.5M GB-seconds
├── Cost: $0.20 + $0.0000166667 × 7,500,000 = $125.20/month
│
Email Sender Lambda (10,000 emails/day):
├── Invocations: 300K/month
├── Duration: 300K × 2s × 0.5GB = 300K GB-seconds
├── Cost: $0.06 + $5.00 = $5.06/month
│
Provisioned Concurrency (5 instances):
├── $0.0000097222/GB-second × 0.5GB × 5 × 2,678,400s = $65/month
│
TOTAL Lambda: ~$195/month
```

---

## ✅ Checklist

- [ ] Step 571: วางแผน Lambda use cases สำหรับ chuaikan.com
- [ ] Step 572: เข้าใจ Cold Start และวิธีลด latency
- [ ] Step 573: เข้าใจ Lambda Pricing
- [ ] Step 574: เขียน Image Processor Lambda (Node.js + Sharp)
- [ ] Step 575: Deploy Lambda พร้อม S3 trigger
- [ ] Step 576: สร้าง Lambda Layer สำหรับ Sharp
- [ ] Step 577: เขียน Email Sender Lambda (SQS trigger)
- [ ] Step 578: ตั้งค่า Provisioned Concurrency
- [ ] Step 579: Deploy ด้วย SAM
- [ ] Step 580: ตั้งค่า X-Ray tracing และ monitoring

---

## 🔗 References

- [Lambda Developer Guide](https://docs.aws.amazon.com/lambda/latest/dg/)
- [AWS SAM Documentation](https://docs.aws.amazon.com/serverless-application-model/)
- [Sharp Image Processing](https://sharp.pixelplumbing.com/)
- [Lambda Cold Start Analysis](https://aws.amazon.com/blogs/compute/operating-lambda-performance-optimization-part-1/)

---
*Part 058 | Road to 1,000,000 Users/Day | chuaikan.com*
