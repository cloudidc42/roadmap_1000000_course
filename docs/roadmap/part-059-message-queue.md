# Part 059: SQS / Pub/Sub (Message Queue)
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Advanced
> **Steps:** 581-590
> **เวลาโดยประมาณ:** 5 ชั่วโมง
> **Prerequisites:** Part 058 (Lambda), Part 053 (EKS)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- เปรียบเทียบ SQS vs SNS vs EventBridge
- SQS Standard Queue vs FIFO Queue
- ใช้ SQS FIFO สำหรับ SOS alert processing (ordered, exactly-once)
- ตั้งค่า Dead Letter Queue (DLQ)
- Lambda trigger จาก SQS
- Message Visibility Timeout
- SNS สำหรับ fan-out (SOS alert → SMS + Email + Push พร้อมกัน)
- EventBridge สำหรับ event-driven architecture

---

## 📖 ทฤษฎีและแนวคิด

### Step 581: SQS vs SNS vs EventBridge

```
┌─────────────────────────────────────────────────────────────┐
│  SQS (Simple Queue Service) - Pull-based                    │
│  ├── Consumer ดึง messages เอง                              │
│  ├── Message ถูก process ครั้งเดียว (visibility timeout)    │
│  ├── Fan-out: ไม่รองรับ (ใช้ SNS → SQS แทน)                │
│  └── Use case: Task queue, work distribution                │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  SNS (Simple Notification Service) - Push-based             │
│  ├── Publisher push ไปยัง Subscribers ทันที                 │
│  ├── Fan-out: ส่ง 1 message ไปหลาย subscribers             │
│  ├── Protocols: Email, SMS, HTTP, Lambda, SQS               │
│  └── Use case: Notifications, fan-out pattern               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  EventBridge - Event Bus                                    │
│  ├── Rule-based routing (filter events)                    │
│  ├── Schema registry                                        │
│  ├── Integration กับ 200+ AWS services                      │
│  └── Use case: Event-driven microservices, integrations     │
└─────────────────────────────────────────────────────────────┘
```

### Step 582: SQS Standard vs FIFO

| Feature | Standard Queue | FIFO Queue |
|---------|---------------|------------|
| Throughput | Unlimited | 300 msg/sec (3000 batched) |
| Ordering | Best-effort | Strict (FIFO) |
| Delivery | At-least-once | Exactly-once |
| Deduplication | ไม่มี | ได้ (5 minutes window) |
| Price | $0.40/million | $0.50/million |
| Use case | High throughput | Order-sensitive tasks |

### Step 583: SOS Alert Architecture ใน chuaikan.com

```
สถานการณ์: เจ้าของสัตว์กด SOS Alert ขอความช่วยเหลือ

Event Flow:
User Press SOS → API Server
                    │
                    ▼
              SNS Topic: sos-alerts
                    │
         ┌──────────┼──────────┐
         │          │          │
         ▼          ▼          ▼
    SQS FIFO    Lambda      SQS Standard
    (Database   (Send SMS   (Push
    Update)     via Twilio) Notification)

กฎ Ordering ใน SOS:
1. สร้าง SOS record ใน database (ต้องก่อน)
2. แจ้ง volunteers ใกล้เคียง
3. ส่ง confirmation ให้เจ้าของ

FIFO ป้องกัน:
- สร้าง SOS ซ้ำ 2 ครั้ง (deduplication)
- Process SOS ผิดลำดับ
```

---

## ⚙️ Environment Setup

```bash
export AWS_REGION="ap-southeast-1"
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

---

## 🛠️ Step-by-Step Implementation

### Step 584: สร้าง SQS Queues

```bash
# SQS FIFO Queue สำหรับ SOS alerts
SOS_QUEUE_URL=$(aws sqs create-queue \
  --queue-name chuaikan-sos-alerts.fifo \
  --attributes '{
    "FifoQueue": "true",
    "ContentBasedDeduplication": "true",
    "MessageRetentionPeriod": "86400",
    "VisibilityTimeout": "60",
    "RedrivePolicy": "{\"deadLetterTargetArn\":\"arn:aws:sqs:ap-southeast-1:123456789012:chuaikan-sos-alerts-dlq.fifo\",\"maxReceiveCount\":\"3\"}"
  }' \
  --query 'QueueUrl' --output text)

# Dead Letter Queue สำหรับ SOS
aws sqs create-queue \
  --queue-name chuaikan-sos-alerts-dlq.fifo \
  --attributes '{
    "FifoQueue": "true",
    "MessageRetentionPeriod": "1209600"
  }'

# Standard Queue สำหรับ Email notifications
EMAIL_QUEUE_URL=$(aws sqs create-queue \
  --queue-name chuaikan-email-notifications \
  --attributes '{
    "MessageRetentionPeriod": "86400",
    "VisibilityTimeout": "30",
    "RedrivePolicy": "{\"deadLetterTargetArn\":\"arn:aws:sqs:ap-southeast-1:123456789012:chuaikan-email-dlq\",\"maxReceiveCount\":\"3\"}"
  }' \
  --query 'QueueUrl' --output text)

echo "SOS Queue URL: $SOS_QUEUE_URL"
echo "Email Queue URL: $EMAIL_QUEUE_URL"
```

### Step 585: สร้าง SNS Topic สำหรับ Fan-out

```bash
# สร้าง SNS Topic สำหรับ SOS alerts
SOS_TOPIC_ARN=$(aws sns create-topic \
  --name chuaikan-sos-alerts \
  --attributes '{
    "DisplayName": "chuaikan SOS Alerts"
  }' \
  --query 'TopicArn' --output text)

# Subscribe SQS Queues ไปยัง SNS Topic
# 1. Database update queue
aws sns subscribe \
  --topic-arn $SOS_TOPIC_ARN \
  --protocol sqs \
  --notification-endpoint arn:aws:sqs:ap-southeast-1:123456789012:chuaikan-sos-alerts.fifo \
  --attributes '{
    "FilterPolicy": "{\"eventType\": [\"sos_created\", \"sos_updated\"]}",
    "RawMessageDelivery": "true"
  }'

# 2. Push notification queue
aws sns subscribe \
  --topic-arn $SOS_TOPIC_ARN \
  --protocol sqs \
  --notification-endpoint arn:aws:sqs:ap-southeast-1:123456789012:chuaikan-push-notifications \
  --attributes '{
    "FilterPolicy": "{\"eventType\": [\"sos_created\"]}",
    "RawMessageDelivery": "true"
  }'

# 3. SMS (ส่งไปยัง Lambda ที่จัดการ SMS)
aws sns subscribe \
  --topic-arn $SOS_TOPIC_ARN \
  --protocol lambda \
  --notification-endpoint arn:aws:lambda:ap-southeast-1:123456789012:function:chuaikan-sms-sender

echo "SNS Topic ARN: $SOS_TOPIC_ARN"
```

### Step 586: SQS Message Processing ด้วย Node.js

```typescript
// src/queues/sos-processor.ts
import { 
  SQSClient, 
  ReceiveMessageCommand,
  DeleteMessageCommand,
  ChangeMessageVisibilityCommand,
  Message
} from '@aws-sdk/client-sqs';

const sqsClient = new SQSClient({ region: 'ap-southeast-1' });
const SOS_QUEUE_URL = process.env.SOS_QUEUE_URL!;

interface SOSAlert {
  alertId: string;
  userId: string;
  dogId?: string;
  location: { lat: number; lng: number };
  emergencyType: 'lost' | 'injured' | 'stolen';
  description: string;
  timestamp: string;
}

async function processSOSAlert(alert: SOSAlert): Promise<void> {
  console.log(`Processing SOS alert: ${alert.alertId}`);
  
  // 1. อัพเดท database
  // await db.sosAlerts.create({ data: alert });
  
  // 2. หา volunteers ใกล้เคียง (radius 5km)
  // const volunteers = await findNearbyVolunteers(alert.location, 5);
  
  // 3. ส่ง notification ไปยัง volunteers
  // await notifyVolunteers(volunteers, alert);
  
  // 4. ส่ง confirmation ให้เจ้าของ
  // await confirmToOwner(alert.userId);
  
  console.log(`SOS alert ${alert.alertId} processed successfully`);
}

// Poll and process messages
async function pollMessages(): Promise<void> {
  while (true) {
    try {
      const response = await sqsClient.send(new ReceiveMessageCommand({
        QueueUrl: SOS_QUEUE_URL,
        MaxNumberOfMessages: 10,
        WaitTimeSeconds: 20,  // Long polling (ลด empty receives)
        MessageAttributeNames: ['All'],
        AttributeNames: ['MessageGroupId', 'MessageDeduplicationId']
      }));
      
      if (!response.Messages?.length) continue;
      
      const processPromises = response.Messages.map(async (message: Message) => {
        const alert: SOSAlert = JSON.parse(message.Body!);
        
        try {
          // Extend visibility timeout ถ้า processing นาน
          await sqsClient.send(new ChangeMessageVisibilityCommand({
            QueueUrl: SOS_QUEUE_URL,
            ReceiptHandle: message.ReceiptHandle!,
            VisibilityTimeout: 120
          }));
          
          await processSOSAlert(alert);
          
          // Delete message after successful processing
          await sqsClient.send(new DeleteMessageCommand({
            QueueUrl: SOS_QUEUE_URL,
            ReceiptHandle: message.ReceiptHandle!
          }));
          
        } catch (error) {
          console.error(`Failed to process message ${message.MessageId}:`, error);
          // ไม่ delete → message จะ return ไปยัง queue หลัง visibility timeout
          // หลัง maxReceiveCount ถึง limit → ไป DLQ
        }
      });
      
      await Promise.all(processPromises);
      
    } catch (error) {
      console.error('Error polling messages:', error);
      await new Promise(resolve => setTimeout(resolve, 5000));
    }
  }
}
```

### Step 587: ส่ง SOS Alert ผ่าน SNS

```typescript
// src/services/sos-service.ts
import { SNSClient, PublishCommand } from '@aws-sdk/client-sns';
import { randomUUID } from 'crypto';

const snsClient = new SNSClient({ region: 'ap-southeast-1' });
const SOS_TOPIC_ARN = process.env.SOS_SNS_TOPIC_ARN!;

interface CreateSOSAlertInput {
  userId: string;
  dogId?: string;
  location: { lat: number; lng: number };
  emergencyType: 'lost' | 'injured' | 'stolen';
  description: string;
}

export async function createSOSAlert(input: CreateSOSAlertInput): Promise<string> {
  const alertId = randomUUID();
  const timestamp = new Date().toISOString();
  
  const alert = {
    alertId,
    eventType: 'sos_created',
    ...input,
    timestamp,
    status: 'active'
  };
  
  // Publish to SNS (fan-out to all subscribers)
  const result = await snsClient.send(new PublishCommand({
    TopicArn: SOS_TOPIC_ARN,
    Message: JSON.stringify(alert),
    Subject: `SOS Alert: ${input.emergencyType}`,
    MessageAttributes: {
      eventType: {
        DataType: 'String',
        StringValue: 'sos_created'
      },
      emergencyType: {
        DataType: 'String',
        StringValue: input.emergencyType
      },
      region: {
        DataType: 'String',
        StringValue: 'thailand'
      }
    }
  }));
  
  console.log(`SOS alert published: ${alertId}, SNS Message ID: ${result.MessageId}`);
  return alertId;
}
```

---

## 🔧 Configuration Files

### Step 588: EventBridge Rules

```bash
# สร้าง Custom Event Bus
aws events create-event-bus \
  --name chuaikan-events

# สร้าง Rule สำหรับ Dog Profile View (analytics)
aws events put-rule \
  --name chuaikan-dog-view-rule \
  --event-bus-name chuaikan-events \
  --event-pattern '{
    "source": ["chuaikan.app"],
    "detail-type": ["Dog Profile Viewed"]
  }' \
  --state ENABLED

# Target: Kinesis Firehose สำหรับ analytics
aws events put-targets \
  --rule chuaikan-dog-view-rule \
  --event-bus-name chuaikan-events \
  --targets '[{
    "Id": "analytics-firehose",
    "Arn": "arn:aws:firehose:ap-southeast-1:123456789012:deliverystream/chuaikan-analytics"
  }]'
```

```typescript
// src/events/publisher.ts
import { EventBridgeClient, PutEventsCommand } from '@aws-sdk/client-eventbridge';

const eventBridgeClient = new EventBridgeClient({ region: 'ap-southeast-1' });
const EVENT_BUS_NAME = 'chuaikan-events';

export async function publishEvent(
  source: string,
  detailType: string,
  detail: object
): Promise<void> {
  await eventBridgeClient.send(new PutEventsCommand({
    Entries: [{
      EventBusName: EVENT_BUS_NAME,
      Source: source,
      DetailType: detailType,
      Detail: JSON.stringify(detail),
      Time: new Date()
    }]
  }));
}

// ตัวอย่างการใช้งาน
export async function trackDogView(dogId: string, userId: string) {
  await publishEvent('chuaikan.app', 'Dog Profile Viewed', {
    dogId,
    userId,
    timestamp: new Date().toISOString(),
    platform: 'web'
  });
}

export async function trackAdoption(dogId: string, userId: string) {
  await publishEvent('chuaikan.app', 'Dog Adopted', {
    dogId,
    adopterId: userId,
    timestamp: new Date().toISOString()
  });
}
```

### Terraform SQS Module

```hcl
# terraform/modules/sqs/main.tf

# DLQ ต้องสร้างก่อน
resource "aws_sqs_queue" "sos_dlq" {
  name                      = "${var.project}-${var.environment}-sos-alerts-dlq.fifo"
  fifo_queue                = true
  message_retention_seconds = 1209600  # 14 days

  tags = var.tags
}

resource "aws_sqs_queue" "sos_alerts" {
  name                       = "${var.project}-${var.environment}-sos-alerts.fifo"
  fifo_queue                 = true
  content_based_deduplication = true
  message_retention_seconds  = 86400
  visibility_timeout_seconds = 60

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.sos_dlq.arn
    maxReceiveCount     = 3
  })

  tags = var.tags
}

resource "aws_sqs_queue" "email_dlq" {
  name                      = "${var.project}-${var.environment}-email-dlq"
  message_retention_seconds = 1209600

  tags = var.tags
}

resource "aws_sqs_queue" "email_notifications" {
  name                      = "${var.project}-${var.environment}-email-notifications"
  message_retention_seconds = 86400
  visibility_timeout_seconds = 30

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.email_dlq.arn
    maxReceiveCount     = 3
  })

  tags = var.tags
}

# SNS Topic
resource "aws_sns_topic" "sos_alerts" {
  name = "${var.project}-${var.environment}-sos-alerts"

  tags = var.tags
}

# Subscribe SQS to SNS
resource "aws_sns_topic_subscription" "sos_to_sqs" {
  topic_arn            = aws_sns_topic.sos_alerts.arn
  protocol             = "sqs"
  endpoint             = aws_sqs_queue.sos_alerts.arn
  raw_message_delivery = true

  filter_policy = jsonencode({
    eventType = ["sos_created", "sos_updated"]
  })
}

# Allow SNS to send to SQS
resource "aws_sqs_queue_policy" "sos_queue_policy" {
  queue_url = aws_sqs_queue.sos_alerts.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Service = "sns.amazonaws.com"
      }
      Action   = "sqs:SendMessage"
      Resource = aws_sqs_queue.sos_alerts.arn
      Condition = {
        ArnEquals = {
          "aws:SourceArn" = aws_sns_topic.sos_alerts.arn
        }
      }
    }]
  })
}

output "sos_queue_url" {
  value = aws_sqs_queue.sos_alerts.url
}

output "sos_topic_arn" {
  value = aws_sns_topic.sos_alerts.arn
}

output "email_queue_url" {
  value = aws_sqs_queue.email_notifications.url
}
```

---

## 🧪 Testing

```bash
# ส่ง test message ไปยัง SOS queue
aws sqs send-message \
  --queue-url $SOS_QUEUE_URL \
  --message-body '{
    "alertId": "test-001",
    "eventType": "sos_created",
    "userId": "user-123",
    "location": {"lat": 13.7563, "lng": 100.5018},
    "emergencyType": "lost",
    "description": "Lost golden retriever near Chatuchak",
    "timestamp": "2024-01-15T10:00:00Z"
  }' \
  --message-group-id "sos-alerts" \
  --message-deduplication-id "test-001"

# ดูจำนวน messages ใน queue
aws sqs get-queue-attributes \
  --queue-url $SOS_QUEUE_URL \
  --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible

# Receive message (ทดสอบ)
aws sqs receive-message \
  --queue-url $SOS_QUEUE_URL \
  --max-number-of-messages 1 \
  --wait-time-seconds 5

# ดู DLQ
aws sqs get-queue-attributes \
  --queue-url $SOS_DLQ_URL \
  --attribute-names ApproximateNumberOfMessages

# ทดสอบ SNS fan-out
aws sns publish \
  --topic-arn $SOS_TOPIC_ARN \
  --message '{"alertId":"test-002","eventType":"sos_created"}' \
  --message-attributes '{"eventType":{"DataType":"String","StringValue":"sos_created"}}'

# ดู CloudWatch metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/SQS \
  --metric-name NumberOfMessagesReceived \
  --dimensions Name=QueueName,Value=chuaikan-production-sos-alerts.fifo \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Sum
```

---

## ❌ Common Errors & Solutions

### Error 1: InvalidParameterValue (FIFO deduplication)

```
Error: InvalidParameterValue: The queue should either have ContentBasedDeduplication enabled or MessageDeduplicationId provided explicitly
```

**แก้ไข:**
```typescript
// ต้องส่ง MessageDeduplicationId เมื่อ ContentBasedDeduplication=false
await sqsClient.send(new SendMessageCommand({
  QueueUrl: SOS_QUEUE_URL,
  MessageBody: JSON.stringify(alert),
  MessageGroupId: 'sos-group',
  MessageDeduplicationId: alert.alertId  // ต้อง unique ใน 5 minutes
}));
```

### Error 2: Message visible ก่อน processing เสร็จ

```
Message appears in queue again while still processing
```

**แก้ไข:**
```typescript
// เพิ่ม Visibility Timeout ระหว่าง processing
await sqsClient.send(new ChangeMessageVisibilityCommand({
  QueueUrl: SOS_QUEUE_URL,
  ReceiptHandle: message.ReceiptHandle!,
  VisibilityTimeout: 300  // 5 minutes
}));
```

---

## ✅ Checklist

- [ ] Step 581: เปรียบเทียบ SQS vs SNS vs EventBridge
- [ ] Step 582: เข้าใจ SQS Standard vs FIFO
- [ ] Step 583: ออกแบบ SOS Alert Architecture
- [ ] Step 584: สร้าง SQS Queues (FIFO + DLQ)
- [ ] Step 585: สร้าง SNS Topic และ subscriptions
- [ ] Step 586: เขียน SQS consumer ด้วย Node.js
- [ ] Step 587: เขียน SNS publisher สำหรับ SOS alerts
- [ ] Step 588: ตั้งค่า EventBridge rules
- [ ] Step 589: ทดสอบ fan-out pattern
- [ ] Step 590: ตั้งค่า DLQ monitoring

---

## 🔗 References

- [Amazon SQS Developer Guide](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/)
- [Amazon SNS Documentation](https://docs.aws.amazon.com/sns/latest/dg/)
- [Amazon EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/)
- [SQS FIFO Best Practices](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues-exactly-once-processing.html)

---
*Part 059 | Road to 1,000,000 Users/Day | chuaikan.com*
