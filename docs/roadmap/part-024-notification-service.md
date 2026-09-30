# Part 024: Notification Service (Push/SMS/Email)
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 231-240
> **เวลาโดยประมาณ:** 8 ชั่วโมง
> **Prerequisites:** Part 021 (Microservices), Part 022 (API Gateway), Redis 7, BullMQ

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ออกแบบ Event-Driven Notification Service
- สร้าง Priority Queues ด้วย BullMQ
- Implement Email worker ด้วย Resend API พร้อม retry
- Implement SMS worker ด้วย Twilio สำหรับเบอร์ไทย
- Push Notification ด้วย FCM (Android/iOS/Web)
- Template engine ด้วย Handlebars (Thai + English)
- User preference management (opt-in/opt-out)
- Dead Letter Queue (DLQ) handling
- Rate limiting per user per channel
- Metrics: delivery rate, open rate

---

## 📖 ทฤษฎีและแนวคิด

### Step 231: Notification Service Design

**Event-Driven Architecture สำหรับ Notifications:**

```
Event Flow:
┌───────────────────────────────────────────────────────────────┐
│                   Events from Other Services                  │
│  post.created | sos.triggered | user.followed | comment.added │
└────────────────────────────┬──────────────────────────────────┘
                             │ (Message Queue / Redis Streams)
                             ▼
┌───────────────────────────────────────────────────────────────┐
│              Notification Orchestrator                        │
│  1. Receive event                                             │
│  2. Determine who to notify                                   │
│  3. Check user preferences (opt-in/out)                       │
│  4. Check rate limits                                         │
│  5. Route to correct channel queue                            │
└──────────────┬───────────────┬────────────────────────────────┘
               │               │               │
    ┌──────────▼──┐  ┌─────────▼──┐  ┌────────▼────────┐
    │ Email Queue │  │  SMS Queue │  │   Push Queue    │
    │  (BullMQ)   │  │  (BullMQ)  │  │   (BullMQ)      │
    └──────────┬──┘  └─────────┬──┘  └────────┬────────┘
               │               │               │
    ┌──────────▼──┐  ┌─────────▼──┐  ┌────────▼────────┐
    │Email Worker │  │ SMS Worker │  │  Push Worker    │
    │  (Resend)   │  │  (Twilio)  │  │   (FCM/APNs)    │
    └─────────────┘  └────────────┘  └─────────────────┘

Priority Levels:
CRITICAL (SOS): ส่งทุก channel ทันที, bypass rate limit
HIGH: ส่งภายใน 30 วินาที
MEDIUM: ส่งภายใน 5 นาที
LOW (digest): รวมส่งวันละครั้ง
```

### Step 232: BullMQ Queue Architecture

```
BullMQ Queue Structure ใน Redis:
┌─────────────────────────────────────────────────────────┐
│                     Redis 7                             │
│                                                         │
│  bull:notifications:email:wait      [pending jobs]      │
│  bull:notifications:email:active    [processing now]    │
│  bull:notifications:email:delayed   [scheduled future]  │
│  bull:notifications:email:completed [done jobs]         │
│  bull:notifications:email:failed    [failed jobs]       │
│  bull:notifications:email:dlq       [dead letter]       │
│                                                         │
│  bull:notifications:sms:*           [same structure]    │
│  bull:notifications:push:*          [same structure]    │
│  bull:notifications:digest:*        [same structure]    │
└─────────────────────────────────────────────────────────┘

Priority Numbers:
1 = CRITICAL (SOS alerts)
2 = HIGH (mentions, direct messages)
3 = MEDIUM (likes, comments, follows)
4 = LOW (digest, weekly summary)
```

---

## ⚙️ Environment Setup

### Step 233: Project Setup

```bash
# สร้าง notification service
mkdir -p ~/chuaikan-platform/apps/notification-service
cd ~/chuaikan-platform/apps/notification-service

# สร้าง package.json
cat > package.json << 'EOF'
{
  "name": "@chuaikan/notification-service",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "test": "vitest run",
    "test:coverage": "vitest run --coverage",
    "db:migrate": "prisma migrate dev",
    "db:generate": "prisma generate"
  },
  "dependencies": {
    "@chuaikan/shared-types": "workspace:*",
    "@chuaikan/utils": "workspace:*",
    "@prisma/client": "^5.17.0",
    "bullmq": "^5.12.0",
    "express": "^4.19.0",
    "firebase-admin": "^12.3.0",
    "handlebars": "^4.7.8",
    "ioredis": "^5.4.1",
    "resend": "^3.4.0",
    "twilio": "^5.2.0",
    "winston": "^3.14.0",
    "zod": "^3.23.0"
  },
  "devDependencies": {
    "@types/express": "^4.17.21",
    "@types/handlebars": "^4.1.0",
    "@types/node": "^22.0.0",
    "@vitest/coverage-v8": "^2.0.0",
    "prisma": "^5.17.0",
    "tsx": "^4.16.0",
    "typescript": "^5.5.0",
    "vitest": "^2.0.0"
  }
}
EOF

# สร้าง directory structure
mkdir -p src/{config,controllers,queues,workers,templates,services,middleware,__tests__}
mkdir -p src/templates/{email,sms}

# ติดตั้ง dependencies
pnpm install
```

---

## 🛠️ Step-by-Step Implementation

### Step 234: BullMQ Queue Setup

```typescript
// apps/notification-service/src/queues/index.ts

import { Queue, QueueEvents, Worker } from 'bullmq';
import { Redis } from 'ioredis';

// Notification job types
export type NotificationJobType =
  | 'send_email'
  | 'send_sms'
  | 'send_push'
  | 'send_digest';

export interface BaseNotificationJob {
  type: NotificationJobType;
  userId: string;
  notificationId: string;
  priority: 1 | 2 | 3 | 4;  // 1=CRITICAL, 4=LOW
  attempts: number;
}

export interface EmailJob extends BaseNotificationJob {
  type: 'send_email';
  to: string;
  templateId: string;
  templateData: Record<string, unknown>;
  subject: string;
}

export interface SmsJob extends BaseNotificationJob {
  type: 'send_sms';
  to: string;  // Thai phone number: +66xxxxxxxxx
  message: string;
  isSOS?: boolean;
}

export interface PushJob extends BaseNotificationJob {
  type: 'send_push';
  deviceTokens: string[];
  title: string;
  body: string;
  data?: Record<string, string>;
  imageUrl?: string;
  badge?: number;
}

export interface DigestJob extends BaseNotificationJob {
  type: 'send_digest';
  digestType: 'daily' | 'weekly';
  notificationIds: string[];
}

export type NotificationJob = EmailJob | SmsJob | PushJob | DigestJob;

// Queue configuration
const redisConnection = new Redis(process.env.REDIS_URL!, {
  maxRetriesPerRequest: null,
  enableReadyCheck: false,
});

const defaultJobOptions = {
  removeOnComplete: { count: 1000, age: 24 * 3600 }, // เก็บ 1000 jobs หรือ 24h
  removeOnFail: { count: 5000, age: 7 * 24 * 3600 },  // เก็บ failures 7 วัน
};

// Create queues
export const emailQueue = new Queue<EmailJob>('notifications:email', {
  connection: redisConnection,
  defaultJobOptions: {
    ...defaultJobOptions,
    attempts: 3,
    backoff: { type: 'exponential', delay: 5000 },
  },
});

export const smsQueue = new Queue<SmsJob>('notifications:sms', {
  connection: redisConnection,
  defaultJobOptions: {
    ...defaultJobOptions,
    attempts: 5,  // SMS retry มากกว่า
    backoff: { type: 'exponential', delay: 10000 },
  },
});

export const pushQueue = new Queue<PushJob>('notifications:push', {
  connection: redisConnection,
  defaultJobOptions: {
    ...defaultJobOptions,
    attempts: 3,
    backoff: { type: 'fixed', delay: 5000 },
  },
});

export const digestQueue = new Queue<DigestJob>('notifications:digest', {
  connection: redisConnection,
  defaultJobOptions: {
    ...defaultJobOptions,
    attempts: 2,
    backoff: { type: 'fixed', delay: 60000 },
  },
});

// Dead Letter Queue
export const dlqQueue = new Queue('notifications:dlq', {
  connection: redisConnection,
  defaultJobOptions: {
    removeOnComplete: false,  // เก็บไว้ตลอด
    removeOnFail: false,
  },
});

// Queue orchestrator: รับ event และส่งไปยัง queues ที่เหมาะสม
export class NotificationOrchestrator {
  constructor(
    private readonly prisma: any,
    private readonly userPreferenceService: UserPreferenceService
  ) {}

  async handleEvent(event: {
    type: string;
    userId?: string;
    payload: Record<string, unknown>;
  }) {
    const handlers: Record<string, () => Promise<void>> = {
      'sos.triggered': () => this.handleSOSTrigger(event.payload),
      'post.created': () => this.handlePostCreated(event.payload),
      'user.followed': () => this.handleUserFollowed(event.payload),
      'comment.added': () => this.handleCommentAdded(event.payload),
      'post.liked': () => this.handlePostLiked(event.payload),
    };

    const handler = handlers[event.type];
    if (handler) {
      await handler();
    }
  }

  private async handleSOSTrigger(payload: any) {
    // SOS: ส่งทุก channel ทันที, CRITICAL priority
    const nearbyUsers = await this.getNearbyUsers(payload.location, 5); // 5km radius

    for (const userId of nearbyUsers) {
      const prefs = await this.userPreferenceService.getPreferences(userId);
      
      // SOS bypass ทุก preference (safety critical)
      const notificationId = await this.createNotificationRecord({
        userId,
        type: 'sos_alert',
        priority: 'critical',
        data: payload,
      });

      // Push (ทุกคน)
      if (prefs.pushDeviceTokens?.length) {
        await pushQueue.add('sos_push', {
          type: 'send_push',
          userId,
          notificationId,
          priority: 1,
          attempts: 0,
          deviceTokens: prefs.pushDeviceTokens,
          title: '🆘 SOS Alert ใกล้บ้านคุณ',
          body: `มีเหตุฉุกเฉินภายใน ${payload.distanceKm?.toFixed(1)} กม.`,
          data: { alertId: payload.alertId, type: 'sos' },
        }, { priority: 1 });  // BullMQ priority: 1 = highest
      }

      // SMS (ถ้ามีเบอร์)
      if (prefs.phoneNumber) {
        await smsQueue.add('sos_sms', {
          type: 'send_sms',
          userId,
          notificationId,
          priority: 1,
          attempts: 0,
          to: prefs.phoneNumber,
          message: `[chuaikan SOS] มีเหตุฉุกเฉินใกล้บ้านคุณ ${payload.distanceKm?.toFixed(1)} กม. ดูรายละเอียด: ${process.env.APP_URL}/sos/${payload.alertId}`,
          isSOS: true,
        }, { priority: 1 });
      }
    }
  }

  private async handlePostCreated(payload: any) {
    // Notify followers ที่ opt-in
    const followers = await this.getFollowersWithPreference(
      payload.authorId,
      'new_post'
    );

    for (const follower of followers) {
      const notificationId = await this.createNotificationRecord({
        userId: follower.userId,
        type: 'new_post',
        priority: 'medium',
        data: payload,
      });

      await pushQueue.add('new_post_push', {
        type: 'send_push',
        userId: follower.userId,
        notificationId,
        priority: 3,
        attempts: 0,
        deviceTokens: follower.deviceTokens,
        title: `${payload.authorName} โพสต์ใหม่`,
        body: payload.content.substring(0, 100),
        data: { postId: payload.postId, type: 'new_post' },
      }, { priority: 3 });
    }
  }

  private async createNotificationRecord(data: any): Promise<string> {
    const notification = await this.prisma.notification.create({
      data: {
        userId: data.userId,
        type: data.type,
        priority: data.priority,
        data: data.data,
        status: 'pending',
      },
    });
    return notification.id;
  }

  private async getNearbyUsers(location: any, radiusKm: number): Promise<string[]> {
    // ใช้ PostgreSQL PostGIS หรือ simple distance calculation
    const users = await this.prisma.$queryRaw`
      SELECT id FROM users 
      WHERE ST_DWithin(
        location::geography,
        ST_MakePoint(${location.lng}, ${location.lat})::geography,
        ${radiusKm * 1000}
      )
      AND id != ${location.userId}
      LIMIT 500
    `;
    return users.map((u: any) => u.id);
  }

  private async getFollowersWithPreference(authorId: string, prefType: string) {
    // Join follows table with notification_preferences
    return this.prisma.$queryRaw`
      SELECT f.follower_id as "userId", array_agg(d.token) as "deviceTokens"
      FROM follows f
      JOIN notification_preferences np ON np.user_id = f.follower_id 
        AND np.type = ${prefType} AND np.enabled = true
      LEFT JOIN device_tokens d ON d.user_id = f.follower_id AND d.is_active = true
      WHERE f.following_id = ${authorId}
      GROUP BY f.follower_id
      LIMIT 1000
    `;
  }
}
```

### Step 235: Email Worker

```typescript
// apps/notification-service/src/workers/email.worker.ts

import { Worker, Job } from 'bullmq';
import { Resend } from 'resend';
import { Redis } from 'ioredis';
import Handlebars from 'handlebars';
import fs from 'fs';
import path from 'path';
import { EmailJob } from '../queues';
import { dlqQueue } from '../queues';

const resend = new Resend(process.env.RESEND_API_KEY);
const redis = new Redis(process.env.REDIS_URL!);

// Load และ compile templates
const templateCache = new Map<string, HandlebarsTemplateDelegate>();

function getTemplate(templateId: string, lang: 'th' | 'en' = 'th'): HandlebarsTemplateDelegate {
  const cacheKey = `${templateId}-${lang}`;
  
  if (!templateCache.has(cacheKey)) {
    const templatePath = path.join(
      __dirname,
      '../templates/email',
      `${templateId}.${lang}.hbs`
    );
    
    if (!fs.existsSync(templatePath)) {
      throw new Error(`Email template not found: ${templatePath}`);
    }
    
    const source = fs.readFileSync(templatePath, 'utf8');
    templateCache.set(cacheKey, Handlebars.compile(source));
  }
  
  return templateCache.get(cacheKey)!;
}

async function checkDailyEmailLimit(userId: string): Promise<boolean> {
  const key = `email:daily:${userId}:${new Date().toISOString().split('T')[0]}`;
  const count = await redis.incr(key);
  
  if (count === 1) {
    await redis.expire(key, 86400); // expire ใน 24h
  }
  
  return count <= 50; // Max 50 emails/day per user
}

export function createEmailWorker(): Worker {
  const worker = new Worker<EmailJob>(
    'notifications:email',
    async (job: Job<EmailJob>) => {
      const { to, templateId, templateData, subject, userId, notificationId } = job.data;

      // Check daily limit (ยกเว้น SOS/critical)
      if (job.data.priority > 1) {
        const withinLimit = await checkDailyEmailLimit(userId);
        if (!withinLimit) {
          console.log(`Email daily limit exceeded for user ${userId}`);
          return { skipped: true, reason: 'daily_limit_exceeded' };
        }
      }

      // Compile template
      const lang = templateData.lang as 'th' | 'en' || 'th';
      const template = getTemplate(templateId, lang);
      const htmlContent = template(templateData);

      // Track delivery start
      await updateNotificationStatus(notificationId, 'sending');

      // Send via Resend
      const result = await resend.emails.send({
        from: 'chuaikan.com <noreply@chuaikan.com>',
        to: [to],
        subject,
        html: htmlContent,
        // Bounce/complaint webhooks ตั้งค่าใน Resend dashboard
        headers: {
          'X-Notification-ID': notificationId,
          'X-User-ID': userId,
        },
      });

      if (result.error) {
        throw new Error(`Resend error: ${result.error.message}`);
      }

      await updateNotificationStatus(notificationId, 'sent', {
        externalId: result.data?.id,
        sentAt: new Date(),
      });

      return { success: true, messageId: result.data?.id };
    },
    {
      connection: new Redis(process.env.REDIS_URL!, { maxRetriesPerRequest: null }),
      concurrency: 10,  // Process 10 emails concurrently
      limiter: {
        max: 100,
        duration: 1000,  // 100 emails/second (Resend limit)
      },
    }
  );

  // Event handlers
  worker.on('completed', (job) => {
    console.log(`Email sent: job=${job.id}, to=${job.data.to}`);
  });

  worker.on('failed', async (job, error) => {
    console.error(`Email failed: job=${job?.id}, error=${error.message}`);
    
    if (job && job.attemptsMade >= (job.opts.attempts || 3)) {
      // Move to DLQ after max retries
      await dlqQueue.add('failed_email', {
        originalJob: job.data,
        error: error.message,
        failedAt: new Date(),
        attempts: job.attemptsMade,
      });
      
      await updateNotificationStatus(job.data.notificationId, 'failed', {
        error: error.message,
      });
    }
  });

  return worker;
}

async function updateNotificationStatus(
  notificationId: string,
  status: string,
  extra?: Record<string, unknown>
): Promise<void> {
  // Update ใน PostgreSQL ผ่าน service
  await fetch(`http://localhost:${process.env.PORT}/internal/notifications/${notificationId}/status`, {
    method: 'PATCH',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ status, ...extra }),
  });
}
```

### Step 236: SMS Worker

```typescript
// apps/notification-service/src/workers/sms.worker.ts

import { Worker, Job } from 'bullmq';
import twilio from 'twilio';
import { Redis } from 'ioredis';
import { SmsJob } from '../queues';

const twilioClient = twilio(
  process.env.TWILIO_ACCOUNT_SID,
  process.env.TWILIO_AUTH_TOKEN
);

const redis = new Redis(process.env.REDIS_URL!);

function formatThaiNumber(phoneNumber: string): string {
  // แปลง 0812345678 → +66812345678
  const cleaned = phoneNumber.replace(/\D/g, '');
  if (cleaned.startsWith('0') && cleaned.length === 10) {
    return `+66${cleaned.substring(1)}`;
  }
  if (cleaned.startsWith('66') && cleaned.length === 11) {
    return `+${cleaned}`;
  }
  if (cleaned.startsWith('+66')) {
    return cleaned;
  }
  throw new Error(`Invalid Thai phone number: ${phoneNumber}`);
}

async function checkSmsDailyLimit(userId: string, isSOS: boolean): Promise<boolean> {
  if (isSOS) return true; // SOS ไม่มี limit
  
  const key = `sms:daily:${userId}:${new Date().toISOString().split('T')[0]}`;
  const count = await redis.incr(key);
  if (count === 1) await redis.expire(key, 86400);
  
  return count <= 10; // Max 10 SMS/day per user
}

export function createSmsWorker(): Worker {
  const worker = new Worker<SmsJob>(
    'notifications:sms',
    async (job: Job<SmsJob>) => {
      const { to, message, userId, isSOS, notificationId } = job.data;

      // Check limit
      const withinLimit = await checkSmsDailyLimit(userId, isSOS || false);
      if (!withinLimit) {
        return { skipped: true, reason: 'daily_sms_limit' };
      }

      // Format Thai number
      const formattedNumber = formatThaiNumber(to);

      // Send via Twilio
      const twilioMessage = await twilioClient.messages.create({
        body: message,
        from: process.env.TWILIO_PHONE_NUMBER,
        to: formattedNumber,
        // สำหรับ SOS ใช้ statusCallback เพื่อ track delivery
        statusCallback: isSOS
          ? `${process.env.APP_URL}/webhooks/twilio/status`
          : undefined,
      });

      if (twilioMessage.errorCode) {
        throw new Error(`Twilio error ${twilioMessage.errorCode}: ${twilioMessage.errorMessage}`);
      }

      return {
        success: true,
        sid: twilioMessage.sid,
        status: twilioMessage.status,
      };
    },
    {
      connection: new Redis(process.env.REDIS_URL!, { maxRetriesPerRequest: null }),
      concurrency: 5,  // Twilio rate limit ต่ำกว่า
      limiter: {
        max: 30,
        duration: 1000,  // 30 SMS/second
      },
    }
  );

  worker.on('failed', async (job, error) => {
    console.error(`SMS failed: ${error.message}`, { jobId: job?.id, to: job?.data.to });
  });

  return worker;
}
```

### Step 237: Push Notification Worker (FCM)

```typescript
// apps/notification-service/src/workers/push.worker.ts

import { Worker, Job } from 'bullmq';
import admin, { messaging } from 'firebase-admin';
import { Redis } from 'ioredis';
import { PushJob } from '../queues';

// Initialize Firebase Admin
if (!admin.apps.length) {
  admin.initializeApp({
    credential: admin.credential.cert({
      projectId: process.env.FCM_PROJECT_ID,
      clientEmail: process.env.FCM_CLIENT_EMAIL,
      privateKey: process.env.FCM_PRIVATE_KEY?.replace(/\\n/g, '\n'),
    }),
  });
}

const fcm = admin.messaging();
const redis = new Redis(process.env.REDIS_URL!);

async function checkPushDailyLimit(userId: string, priority: number): Promise<boolean> {
  if (priority === 1) return true; // CRITICAL bypass limit
  
  const key = `push:daily:${userId}:${new Date().toISOString().split('T')[0]}`;
  const count = await redis.incr(key);
  if (count === 1) await redis.expire(key, 86400);
  
  const limits = { 2: 50, 3: 30, 4: 10 }; // limits by priority
  return count <= (limits[priority as 2 | 3 | 4] || 10);
}

export function createPushWorker(): Worker {
  const worker = new Worker<PushJob>(
    'notifications:push',
    async (job: Job<PushJob>) => {
      const { deviceTokens, title, body, data, imageUrl, badge, userId, priority, notificationId } = job.data;

      if (!deviceTokens || deviceTokens.length === 0) {
        return { skipped: true, reason: 'no_device_tokens' };
      }

      // Check daily limit
      const withinLimit = await checkPushDailyLimit(userId, priority);
      if (!withinLimit) {
        return { skipped: true, reason: 'daily_push_limit' };
      }

      // Prepare FCM message
      const message: messaging.MulticastMessage = {
        notification: {
          title,
          body,
          imageUrl,
        },
        data: {
          ...data,
          notificationId,
          clickAction: 'FLUTTER_NOTIFICATION_CLICK',
        },
        android: {
          priority: priority === 1 ? 'high' : 'normal',
          notification: {
            channelId: priority === 1 ? 'sos_alerts' : 'default',
            sound: priority === 1 ? 'sos_alarm' : 'default',
            color: priority === 1 ? '#FF0000' : '#6366f1',
            tag: notificationId,
            clickAction: 'FLUTTER_NOTIFICATION_CLICK',
          },
        },
        apns: {
          payload: {
            aps: {
              alert: { title, body },
              sound: priority === 1 ? 'sos_alarm.aiff' : 'default',
              badge,
              contentAvailable: 1,
            },
          },
          headers: {
            'apns-priority': priority === 1 ? '10' : '5',
            'apns-push-type': 'alert',
          },
        },
        webpush: {
          notification: {
            title,
            body,
            icon: '/icons/icon-192x192.png',
            badge: '/icons/badge-72x72.png',
            image: imageUrl,
            requireInteraction: priority === 1,
          },
          fcmOptions: {
            link: `${process.env.APP_URL}/notifications/${notificationId}`,
          },
        },
        tokens: deviceTokens.slice(0, 500),  // FCM max 500 tokens per request
      };

      const response = await fcm.sendEachForMulticast(message);

      // Handle invalid tokens
      const invalidTokens: string[] = [];
      response.responses.forEach((r, idx) => {
        if (!r.success && r.error) {
          const errorCode = (r.error as any).code;
          if (
            errorCode === 'messaging/invalid-registration-token' ||
            errorCode === 'messaging/registration-token-not-registered'
          ) {
            invalidTokens.push(deviceTokens[idx]);
          }
        }
      });

      // Cleanup invalid tokens ใน background
      if (invalidTokens.length > 0) {
        await cleanupInvalidTokens(userId, invalidTokens);
      }

      return {
        success: true,
        successCount: response.successCount,
        failureCount: response.failureCount,
        invalidTokensRemoved: invalidTokens.length,
      };
    },
    {
      connection: new Redis(process.env.REDIS_URL!, { maxRetriesPerRequest: null }),
      concurrency: 20,
      limiter: {
        max: 500,
        duration: 1000,  // FCM allows high throughput
      },
    }
  );

  return worker;
}

async function cleanupInvalidTokens(userId: string, tokens: string[]): Promise<void> {
  // Mark tokens as inactive ใน database
  await fetch(`http://localhost:${process.env.PORT}/internal/device-tokens/cleanup`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ userId, tokens }),
  });
}
```

### Step 238: Handlebars Email Templates

```bash
# สร้าง templates
mkdir -p apps/notification-service/src/templates/email
```

```handlebars
{{!-- apps/notification-service/src/templates/email/welcome.th.hbs --}}
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>ยินดีต้อนรับสู่ chuaikan.com</title>
  <style>
    body { font-family: 'Sarabun', Arial, sans-serif; background: #f4f4f4; margin: 0; }
    .container { max-width: 600px; margin: 0 auto; background: white; }
    .header { background: #6366f1; color: white; padding: 30px; text-align: center; }
    .content { padding: 30px; }
    .button { 
      display: inline-block; padding: 12px 30px; 
      background: #6366f1; color: white; 
      text-decoration: none; border-radius: 6px;
    }
    .footer { background: #f4f4f4; padding: 20px; text-align: center; font-size: 12px; }
  </style>
</head>
<body>
  <div class="container">
    <div class="header">
      <h1>ยินดีต้อนรับสู่ chuaikan.com</h1>
    </div>
    <div class="content">
      <h2>สวัสดี {{displayName}},</h2>
      <p>ขอบคุณที่สมัครสมาชิกกับ chuaikan.com 
      กรุณายืนยันอีเมลของคุณเพื่อเริ่มใช้งาน</p>
      
      <p style="text-align: center; margin: 30px 0;">
        <a href="{{verifyUrl}}" class="button">ยืนยันอีเมล</a>
      </p>
      
      <p style="color: #666; font-size: 14px;">
        ลิงก์นี้จะหมดอายุใน 24 ชั่วโมง<br>
        หากคุณไม่ได้สมัครสมาชิก กรุณาเพิกเฉยต่ออีเมลนี้
      </p>
    </div>
    <div class="footer">
      <p>© {{year}} chuaikan.com | 
        <a href="{{unsubscribeUrl}}">ยกเลิกการรับอีเมล</a>
      </p>
    </div>
  </div>
</body>
</html>
```

```handlebars
{{!-- apps/notification-service/src/templates/email/sos-alert.th.hbs --}}
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <style>
    body { font-family: 'Sarabun', Arial, sans-serif; background: #fff5f5; margin: 0; }
    .container { max-width: 600px; margin: 0 auto; background: white; border-top: 6px solid #ef4444; }
    .header { background: #ef4444; color: white; padding: 20px 30px; }
    .alert-badge { 
      display: inline-block; background: #fff; color: #ef4444; 
      padding: 4px 12px; border-radius: 20px; font-weight: bold; font-size: 12px;
    }
    .content { padding: 30px; }
    .location-box {
      background: #fff5f5; border: 1px solid #fecaca;
      border-radius: 8px; padding: 16px; margin: 20px 0;
    }
    .button { 
      display: inline-block; padding: 12px 30px;
      background: #ef4444; color: white; 
      text-decoration: none; border-radius: 6px; font-weight: bold;
    }
  </style>
</head>
<body>
  <div class="container">
    <div class="header">
      <span class="alert-badge">🆘 SOS ALERT</span>
      <h2 style="margin: 10px 0 0">มีเหตุฉุกเฉินใกล้คุณ</h2>
    </div>
    <div class="content">
      <p>เรียน <strong>{{recipientName}}</strong>,</p>
      <p>มีผู้ใช้งาน chuaikan.com ส่งสัญญาณขอความช่วยเหลือ 
      ในรัศมี <strong>{{distanceKm}} กม.</strong> จากตำแหน่งของคุณ</p>
      
      <div class="location-box">
        <strong>ข้อมูลเหตุฉุกเฉิน:</strong><br>
        ระดับ: {{alertLevelText}}<br>
        ระยะทาง: {{distanceKm}} กม.<br>
        เวลา: {{alertTime}}<br>
        {{#if description}}คำอธิบาย: {{description}}{{/if}}
      </div>
      
      <p style="text-align: center; margin: 24px 0;">
        <a href="{{alertUrl}}" class="button">ดูรายละเอียด SOS</a>
      </p>
      
      <p style="color: #666; font-size: 13px;">
        ถ้าคุณอยู่ในพื้นที่และสามารถช่วยได้ กรุณาตอบรับในแอป<br>
        <strong>อย่าเสี่ยงชีวิตตัวเองถ้าสถานการณ์อันตราย</strong>
      </p>
    </div>
  </div>
</body>
</html>
```

### Step 239: User Preference Service

```typescript
// apps/notification-service/src/services/user-preference.service.ts

import { PrismaClient } from '@prisma/client';
import { Redis } from 'ioredis';

export type NotificationType =
  | 'sos_alert'
  | 'new_post'
  | 'new_follower'
  | 'post_like'
  | 'post_comment'
  | 'mention'
  | 'digest_daily'
  | 'digest_weekly';

export type NotificationChannel = 'push' | 'sms' | 'email';

export interface UserNotificationPreferences {
  userId: string;
  phoneNumber?: string;
  email: string;
  pushDeviceTokens: string[];
  preferences: Record<NotificationType, {
    push: boolean;
    sms: boolean;
    email: boolean;
  }>;
}

export class UserPreferenceService {
  private readonly CACHE_TTL = 300; // 5 minutes

  constructor(
    private readonly prisma: PrismaClient,
    private readonly redis: Redis
  ) {}

  async getPreferences(userId: string): Promise<UserNotificationPreferences> {
    // Check cache
    const cached = await this.redis.get(`prefs:${userId}`);
    if (cached) {
      return JSON.parse(cached);
    }

    // Fetch from DB
    const [user, preferences, deviceTokens] = await Promise.all([
      this.prisma.user.findUnique({
        where: { id: userId },
        select: { email: true, phoneNumber: true },
      }),
      this.prisma.notificationPreference.findMany({
        where: { userId },
      }),
      this.prisma.deviceToken.findMany({
        where: { userId, isActive: true },
        select: { token: true },
      }),
    ]);

    if (!user) throw new Error('User not found');

    // Default preferences (opt-in by default except digest)
    const defaultPrefs: UserNotificationPreferences['preferences'] = {
      sos_alert: { push: true, sms: true, email: true },
      new_post: { push: true, sms: false, email: false },
      new_follower: { push: true, sms: false, email: false },
      post_like: { push: true, sms: false, email: false },
      post_comment: { push: true, sms: false, email: false },
      mention: { push: true, sms: false, email: true },
      digest_daily: { push: false, sms: false, email: true },
      digest_weekly: { push: false, sms: false, email: true },
    };

    // Override with user settings
    for (const pref of preferences) {
      if (defaultPrefs[pref.type as NotificationType]) {
        defaultPrefs[pref.type as NotificationType][pref.channel as NotificationChannel] =
          pref.enabled;
      }
    }

    const result: UserNotificationPreferences = {
      userId,
      email: user.email,
      phoneNumber: user.phoneNumber || undefined,
      pushDeviceTokens: deviceTokens.map((d) => d.token),
      preferences: defaultPrefs,
    };

    // Cache for 5 minutes
    await this.redis.setex(`prefs:${userId}`, this.CACHE_TTL, JSON.stringify(result));

    return result;
  }

  async updatePreference(
    userId: string,
    type: NotificationType,
    channel: NotificationChannel,
    enabled: boolean
  ): Promise<void> {
    await this.prisma.notificationPreference.upsert({
      where: {
        userId_type_channel: { userId, type, channel },
      },
      create: { userId, type, channel, enabled },
      update: { enabled },
    });

    // Invalidate cache
    await this.redis.del(`prefs:${userId}`);
  }

  async registerDeviceToken(
    userId: string,
    token: string,
    platform: 'ios' | 'android' | 'web'
  ): Promise<void> {
    await this.prisma.deviceToken.upsert({
      where: { token },
      create: { userId, token, platform, isActive: true },
      update: { userId, isActive: true, updatedAt: new Date() },
    });

    // Invalidate cache
    await this.redis.del(`prefs:${userId}`);
  }
}
```

### Step 240: Notification Metrics

```typescript
// apps/notification-service/src/services/metrics.service.ts

import { Counter, Histogram, Gauge, Registry } from 'prom-client';

const register = new Registry();

export const notificationsSent = new Counter({
  name: 'notifications_sent_total',
  help: 'Total notifications sent by channel and type',
  labelNames: ['channel', 'type', 'status'],
  registers: [register],
});

export const notificationDeliveryTime = new Histogram({
  name: 'notification_delivery_duration_seconds',
  help: 'Time to deliver notification',
  labelNames: ['channel'],
  buckets: [0.1, 0.5, 1, 5, 10, 30, 60],
  registers: [register],
});

export const queueDepth = new Gauge({
  name: 'notification_queue_depth',
  help: 'Current depth of notification queues',
  labelNames: ['queue'],
  registers: [register],
});

export const deliveryRate = new Gauge({
  name: 'notification_delivery_rate',
  help: 'Delivery success rate (last 5 min)',
  labelNames: ['channel'],
  registers: [register],
});

// Queue metrics collector
export async function collectQueueMetrics(queues: any[]) {
  for (const queue of queues) {
    const counts = await queue.getJobCounts();
    queueDepth.labels(queue.name).set(counts.waiting + counts.active);
  }
}

export { register };
```

---

## 🔧 Configuration Files

### Prisma Schema สำหรับ Notification Service

```prisma
// apps/notification-service/prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Notification {
  id        String   @id @default(cuid())
  userId    String
  type      String
  priority  String   @default("medium")
  channel   String?
  title     String?
  body      String?
  data      Json?
  status    String   @default("pending")  // pending|sending|sent|delivered|read|failed|skipped
  
  externalId  String?   // Resend ID, Twilio SID, FCM message ID
  errorMessage String?
  
  sentAt      DateTime?
  deliveredAt DateTime?
  readAt      DateTime?
  
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  @@index([userId, createdAt])
  @@index([status])
  @@index([type])
}

model NotificationPreference {
  id      String  @id @default(cuid())
  userId  String
  type    String   // NotificationType
  channel String   // 'push' | 'sms' | 'email'
  enabled Boolean  @default(true)
  
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  @@unique([userId, type, channel], name: "userId_type_channel")
  @@index([userId])
}

model DeviceToken {
  id        String  @id @default(cuid())
  userId    String
  token     String  @unique
  platform  String  // 'ios' | 'android' | 'web'
  isActive  Boolean @default(true)
  
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  @@index([userId])
}
```

### Dead Letter Queue Handler

```typescript
// apps/notification-service/src/workers/dlq.worker.ts

import { Worker, Job } from 'bullmq';
import { Redis } from 'ioredis';

interface DLQJob {
  originalJob: any;
  error: string;
  failedAt: Date;
  attempts: number;
}

export function createDLQWorker(): Worker {
  const worker = new Worker<DLQJob>(
    'notifications:dlq',
    async (job: Job<DLQJob>) => {
      const { originalJob, error, failedAt, attempts } = job.data;

      // Log ไปยัง monitoring system
      console.error('DLQ: Notification permanently failed', {
        type: originalJob.type,
        userId: originalJob.userId,
        notificationId: originalJob.notificationId,
        error,
        attempts,
        failedAt,
      });

      // Alert engineering team (ถ้าเป็น critical notification)
      if (originalJob.priority === 1) {
        await alertEngineeringTeam({
          message: `CRITICAL: SOS notification failed after ${attempts} attempts`,
          details: { originalJob, error },
        });
      }

      // ไม่ retry DLQ jobs - เก็บไว้ให้ engineer review
      return { logged: true };
    },
    {
      connection: new Redis(process.env.REDIS_URL!, { maxRetriesPerRequest: null }),
      concurrency: 1,
    }
  );

  return worker;
}

async function alertEngineeringTeam(data: any): Promise<void> {
  // ส่ง Slack webhook หรือ PagerDuty alert
  if (process.env.SLACK_WEBHOOK_URL) {
    await fetch(process.env.SLACK_WEBHOOK_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        text: `🚨 ${data.message}`,
        attachments: [{
          color: 'danger',
          text: JSON.stringify(data.details, null, 2),
        }],
      }),
    });
  }
}
```

---

## 🧪 Testing

```typescript
// apps/notification-service/src/__tests__/orchestrator.test.ts

import { describe, it, expect, vi, beforeEach } from 'vitest';
import { NotificationOrchestrator } from '../queues';

const mockEmailQueue = { add: vi.fn().mockResolvedValue({ id: 'job-1' }) };
const mockSmsQueue = { add: vi.fn().mockResolvedValue({ id: 'job-2' }) };
const mockPushQueue = { add: vi.fn().mockResolvedValue({ id: 'job-3' }) };

vi.mock('../queues', () => ({
  emailQueue: mockEmailQueue,
  smsQueue: mockSmsQueue,
  pushQueue: mockPushQueue,
}));

describe('NotificationOrchestrator', () => {
  it('should send SOS to all channels with critical priority', async () => {
    const orchestrator = new NotificationOrchestrator(
      {} as any,
      {
        getPreferences: vi.fn().mockResolvedValue({
          email: 'user@test.com',
          phoneNumber: '+66812345678',
          pushDeviceTokens: ['token-abc'],
          preferences: { sos_alert: { push: true, sms: true, email: true } },
        }),
      } as any
    );

    await orchestrator.handleEvent({
      type: 'sos.triggered',
      payload: {
        alertId: 'alert-123',
        location: { lat: 13.7, lng: 100.5 },
        distanceKm: 2.5,
      },
    });

    // SOS ควร queue ทุก channel ด้วย priority 1
    expect(mockPushQueue.add).toHaveBeenCalledWith(
      expect.any(String),
      expect.objectContaining({ priority: 1 }),
      expect.objectContaining({ priority: 1 })
    );
  });
});
```

---

## ❌ Common Errors & Solutions

### Error 1: BullMQ "MaxListenersExceeded"

```bash
# สาเหตุ: สร้าง Redis connection มากเกินไป
# แก้ไข: reuse connection

const sharedRedis = new Redis(process.env.REDIS_URL!, {
  maxRetriesPerRequest: null,
  enableReadyCheck: false,
});

// ใช้ sharedRedis สำหรับทุก queue
const emailQueue = new Queue('notifications:email', {
  connection: sharedRedis,
});
```

### Error 2: FCM "auth/invalid-credential"

```bash
# ตรวจสอบ service account key
# ใน Firebase Console: Project Settings → Service Accounts → Generate new private key

# ตรวจสอบ environment variables
echo $FCM_PROJECT_ID
echo $FCM_CLIENT_EMAIL
# FCM_PRIVATE_KEY ต้องมี \n เป็น actual newlines

# แก้ไขใน code:
privateKey: process.env.FCM_PRIVATE_KEY?.replace(/\\n/g, '\n')
```

### Error 3: Twilio SMS ส่งไม่ได้เบอร์ไทย

```bash
# ตรวจสอบ format
# ถูก: +66812345678
# ผิด: 0812345678, 66812345678

# Twilio ต้องการ E.164 format
# ทดสอบ:
curl -X POST "https://api.twilio.com/2010-04-01/Accounts/$TWILIO_ACCOUNT_SID/Messages.json" \
  -u "$TWILIO_ACCOUNT_SID:$TWILIO_AUTH_TOKEN" \
  --data-urlencode "To=+66812345678" \
  --data-urlencode "From=$TWILIO_PHONE_NUMBER" \
  --data-urlencode "Body=Test message"
```

### Error 4: Email template ไม่ render Thai fonts

```bash
# แก้ไข: ใช้ Google Fonts ใน email template
# เพิ่มใน <head>:
<link href="https://fonts.googleapis.com/css2?family=Sarabun:wght@400;700&display=swap" rel="stylesheet">

# สำรอง: ใช้ system fonts
font-family: 'Sarabun', 'TH Sarabun New', Arial, sans-serif;
```

---

## ✅ Checklist

### Step 231-232: Design
- [ ] เข้าใจ event-driven notification flow
- [ ] วาง queue priority levels
- [ ] กำหนด rate limits ต่อ user ต่อ channel

### Step 233-234: Queue Setup
- [ ] BullMQ ติดตั้งและ queues สร้างสำเร็จ
- [ ] NotificationOrchestrator handle events ได้
- [ ] SOS events ส่งด้วย priority 1

### Step 235-236: Workers
- [ ] Email worker ส่ง email ผ่าน Resend ได้
- [ ] SMS worker format เบอร์ไทยถูกต้อง
- [ ] Push worker ส่ง FCM multicast ได้

### Step 237-238: Templates
- [ ] Handlebars templates render ถูกต้อง
- [ ] Thai language support ทำงาน
- [ ] SOS email template มี styling ที่ชัดเจน

### Step 239: User Preferences
- [ ] getPreferences ดึงข้อมูลพร้อม cache
- [ ] opt-in/out ต่อ type และ channel ทำงาน
- [ ] Device token registration ทำงาน

### Step 240: Monitoring & DLQ
- [ ] Prometheus metrics endpoint ทำงาน
- [ ] DLQ worker log failures
- [ ] Critical failures แจ้ง engineering team
- [ ] Queue depth metrics แสดงใน Grafana

---

## 🔗 References

- [BullMQ Documentation](https://docs.bullmq.io/)
- [Resend API](https://resend.com/docs)
- [Firebase Cloud Messaging](https://firebase.google.com/docs/cloud-messaging)
- [Twilio SMS Thailand](https://www.twilio.com/en-us/help/sms/thai-sms)
- [Handlebars Templates](https://handlebarsjs.com/)
- [Web Push Notifications](https://web.dev/push-notifications-overview/)

---
*Part 024 | Road to 1,000,000 Users/Day | chuaikan.com*
