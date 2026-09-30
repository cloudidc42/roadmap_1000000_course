# Part 075: Push Notifications at Scale (10M Devices)

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Advanced
> **Steps:** 741-750
> **เวลาโดยประมาณ:** 5 ชั่วโมง
> **Prerequisites:** Part 072 (Kafka), Part 073 (Feed Algorithm)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- Firebase Cloud Messaging (FCM) v1 API
- FCM token management
- Topic messaging vs individual messaging
- Notification batching
- BullMQ workers (100 workers สำหรับ 10M push)
- SOS critical alerts: APNs priority 10, FCM high priority
- Cost optimization

---

## 📖 ทฤษฎีและแนวคิด

### Push Notification Architecture

```
Trigger (SOS/Post/Follow)
         │
         ▼
   Kafka Topic: notifications
         │
         ▼
   BullMQ Queue (Redis)
   ┌─────────────────────────────────────┐
   │   Notification Queue                │
   │   [job1] [job2] [job3] ... [jobN]  │
   └─────────────────────────────────────┘
         │
         ▼ (100 concurrent workers)
   FCM Sender Workers
   ┌──────┬──────┬──────┬──────────────┐
   │ W1   │ W2   │ W3   │  ...  W100   │
   └──────┴──────┴──────┴──────────────┘
         │
         ▼ (batch 500 tokens/request)
   Google FCM API (Android/Web)
   Apple APNs (iOS)
         │
         ▼
   10,000,000 Devices

Target throughput: 10M notifications in ~5 minutes
Rate: ~33,000 notifications/second
```

### FCM Types

```
1. Individual (Token): ส่งไปยัง device เดียว
   → ใช้สำหรับ personal notifications (DM, follow, etc.)

2. Topic: ส่งไปยัง topic subscribers ทั้งหมด
   → ใช้สำหรับ area SOS alerts (เช่น topic "sos-bkk-001")
   → FCM จัดการ fan-out เอง (ไม่ต้องดู tokens)

3. Multicast: ส่งไปยัง tokens หลายอัน (max 500/request)
   → ใช้สำหรับ batch sends

สำหรับ chuaikan.com:
- SOS local area → Topic messaging ("sos-area-{areaCode}")
- New post notification → Individual tokens (ส่งให้ followers)
- System announcements → Topic messaging ("all-users")
```

---

## ⚙️ Environment Setup

### Step 741: ติดตั้ง Firebase Admin SDK

```bash
npm install firebase-admin bullmq ioredis prom-client
```

```javascript
// config/firebase.js
const admin = require('firebase-admin');

const serviceAccount = JSON.parse(process.env.FIREBASE_SERVICE_ACCOUNT_JSON);

admin.initializeApp({
  credential: admin.credential.cert(serviceAccount),
  projectId: process.env.FIREBASE_PROJECT_ID,
});

const messaging = admin.messaging();
module.exports = { messaging };
```

---

## 🛠️ Step-by-Step Implementation

### Step 742: FCM Token Management

```javascript
// models/device-token.js - FCM Token management

const db = require('../config/db');
const redis = require('../config/redis');

/**
 * Register/Update FCM token สำหรับ user
 */
async function registerToken(userId, token, platform) {
  // ตรวจสอบว่า token ถูกต้อง
  if (!token || token.length < 20) {
    throw new Error('Invalid FCM token');
  }
  
  // บันทึกใน database
  await db.query(`
    INSERT INTO device_tokens (user_id, token, platform, last_seen_at)
    VALUES ($1, $2, $3, NOW())
    ON CONFLICT (token) DO UPDATE
      SET user_id = $1,
          platform = $3,
          last_seen_at = NOW(),
          is_active = TRUE
  `, [userId, token, platform]);
  
  // Cache token สำหรับ quick lookup
  await redis.setex(`token:${token}`, 86400 * 30, userId.toString());
  
  // Subscribe user to their area topic (สำหรับ SOS alerts)
  const user = await getUserAreaCode(userId);
  if (user.areaCode) {
    await subscribeToTopic(token, `sos-area-${user.areaCode}`);
  }
  
  // Subscribe to all-users topic
  await subscribeToTopic(token, 'all-users');
}

/**
 * Subscribe token ไปยัง FCM topic
 */
async function subscribeToTopic(token, topic) {
  const { messaging } = require('../config/firebase');
  
  await messaging.subscribeToTopic([token], topic);
}

/**
 * ลบ token ที่ invalid ออก (cleanup)
 */
async function invalidateTokens(invalidTokens) {
  if (invalidTokens.length === 0) return;
  
  await db.query(`
    UPDATE device_tokens
    SET is_active = FALSE, invalidated_at = NOW()
    WHERE token = ANY($1)
  `, [invalidTokens]);
  
  // ลบออกจาก cache
  const pipeline = redis.pipeline();
  for (const token of invalidTokens) {
    pipeline.del(`token:${token}`);
  }
  await pipeline.exec();
  
  console.log(`Invalidated ${invalidTokens.length} stale tokens`);
}

/**
 * ดึง active tokens สำหรับ user
 */
async function getUserTokens(userId) {
  const { rows } = await db.query(`
    SELECT token, platform
    FROM device_tokens
    WHERE user_id = $1 AND is_active = TRUE
    ORDER BY last_seen_at DESC
    LIMIT 5  -- สูงสุด 5 devices ต่อ user
  `, [userId]);
  
  return rows;
}

module.exports = { registerToken, subscribeToTopic, invalidateTokens, getUserTokens };
```

### Step 743: FCM Sender Service

```javascript
// services/fcm-sender.js - Core FCM sending

const { messaging } = require('../config/firebase');
const { invalidateTokens } = require('../models/device-token');
const client = require('prom-client');

// Metrics
const notificationsSent = new client.Counter({
  name: 'fcm_notifications_sent_total',
  help: 'Total FCM notifications sent',
  labelNames: ['platform', 'type', 'status'],
});

const sendDuration = new client.Histogram({
  name: 'fcm_send_duration_seconds',
  help: 'FCM API call duration',
  buckets: [0.1, 0.5, 1, 2, 5],
});

/**
 * ส่ง notification ไปยัง single device
 */
async function sendToDevice(token, notification) {
  const message = {
    token,
    notification: {
      title: notification.title,
      body: notification.body,
      imageUrl: notification.imageUrl,
    },
    data: sanitizeData(notification.data || {}),
    android: {
      priority: notification.priority === 'high' ? 'high' : 'normal',
      notification: {
        channelId: getAndroidChannel(notification.type),
        color: '#FF4444',
        clickAction: 'FLUTTER_NOTIFICATION_CLICK',
      },
    },
    apns: {
      headers: {
        'apns-priority': notification.priority === 'high' ? '10' : '5',
        'apns-expiration': notification.expiration || '0',
      },
      payload: {
        aps: {
          alert: {
            title: notification.title,
            body: notification.body,
          },
          sound: notification.priority === 'high' ? 'sos_alert.aiff' : 'default',
          badge: 1,
          'content-available': 1,
          'mutable-content': 1,
        },
      },
    },
  };
  
  const timer = sendDuration.startTimer();
  
  try {
    await messaging.send(message);
    notificationsSent.inc({ platform: 'fcm', type: notification.type, status: 'success' });
    timer();
    
  } catch (err) {
    timer();
    
    if (isTokenInvalid(err)) {
      await invalidateTokens([token]);
    }
    
    notificationsSent.inc({ platform: 'fcm', type: notification.type, status: 'error' });
    throw err;
  }
}

/**
 * ส่ง notification ไปยัง tokens หลายอัน (batch)
 * FCM Multicast: สูงสุด 500 tokens ต่อ request
 */
async function sendToDevices(tokens, notification) {
  if (tokens.length === 0) return { successCount: 0, failureCount: 0 };
  
  // Split ออกเป็น batches ของ 500
  const batchSize = 500;
  const batches = [];
  
  for (let i = 0; i < tokens.length; i += batchSize) {
    batches.push(tokens.slice(i, i + batchSize));
  }
  
  let totalSuccess = 0;
  let totalFailure = 0;
  const invalidTokenList = [];
  
  for (const batch of batches) {
    const message = {
      notification: {
        title: notification.title,
        body: notification.body,
      },
      data: sanitizeData(notification.data || {}),
      android: {
        priority: notification.priority === 'high' ? 'high' : 'normal',
        notification: {
          channelId: getAndroidChannel(notification.type),
        },
      },
      apns: {
        headers: {
          'apns-priority': notification.priority === 'high' ? '10' : '5',
        },
        payload: {
          aps: {
            alert: {
              title: notification.title,
              body: notification.body,
            },
            sound: notification.priority === 'high' ? 'sos_alert.aiff' : 'default',
          },
        },
      },
      tokens: batch,
    };
    
    const response = await messaging.sendEachForMulticast(message);
    
    totalSuccess += response.successCount;
    totalFailure += response.failureCount;
    
    // รวบรวม invalid tokens
    response.responses.forEach((resp, idx) => {
      if (!resp.success && isTokenInvalid(resp.error)) {
        invalidTokenList.push(batch[idx]);
      }
    });
  }
  
  // Cleanup invalid tokens (async)
  if (invalidTokenList.length > 0) {
    invalidateTokens(invalidTokenList).catch(console.error);
  }
  
  return { successCount: totalSuccess, failureCount: totalFailure };
}

/**
 * ส่งไปยัง FCM Topic (fan-out จัดการโดย FCM)
 */
async function sendToTopic(topic, notification) {
  const message = {
    topic,
    notification: {
      title: notification.title,
      body: notification.body,
    },
    data: sanitizeData(notification.data || {}),
    android: {
      priority: 'high',
    },
    apns: {
      headers: { 'apns-priority': '10' },
      payload: {
        aps: {
          sound: 'sos_alert.aiff',
          'content-available': 1,
        },
      },
    },
  };
  
  const response = await messaging.send(message);
  return response;
}

function isTokenInvalid(error) {
  if (!error?.errorInfo) return false;
  const invalidCodes = [
    'messaging/invalid-registration-token',
    'messaging/registration-token-not-registered',
    'messaging/invalid-recipient',
  ];
  return invalidCodes.includes(error.errorInfo.code);
}

function getAndroidChannel(type) {
  const channels = {
    SOS_ALERT: 'sos-channel',
    NEW_POST: 'post-channel',
    NEW_MESSAGE: 'message-channel',
    FOLLOW: 'social-channel',
    DEFAULT: 'default-channel',
  };
  return channels[type] || channels.DEFAULT;
}

function sanitizeData(data) {
  const sanitized = {};
  for (const [key, value] of Object.entries(data)) {
    sanitized[key] = String(value);  // FCM data values ต้องเป็น string เท่านั้น
  }
  return sanitized;
}

module.exports = { sendToDevice, sendToDevices, sendToTopic };
```

### Step 744: BullMQ Workers (100 workers)

```javascript
// workers/notification-worker.js - 100 concurrent workers

const { Worker, Queue, QueueScheduler } = require('bullmq');
const { sendToDevices, sendToTopic } = require('../services/fcm-sender');
const { getUserTokens } = require('../models/device-token');
const db = require('../config/db');

const connection = {
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT || '6379'),
  maxRetriesPerRequest: 3,
};

// สร้าง notification queue
const notificationQueue = new Queue('notifications', { connection });

// Worker factory
function createNotificationWorker(workerId) {
  return new Worker('notifications', async (job) => {
    const { type, data } = job.data;
    
    switch (type) {
      case 'INDIVIDUAL':
        await processIndividualNotification(data);
        break;
      
      case 'BATCH_TOKENS':
        await processBatchTokenNotification(data);
        break;
      
      case 'TOPIC':
        await processTopicNotification(data);
        break;
      
      case 'SOS_ALERT':
        await processSOSNotification(data);
        break;
    }
    
  }, {
    connection,
    concurrency: 10,        // แต่ละ worker handle 10 jobs พร้อมกัน
    limiter: {
      max: 100,             // 100 jobs/second per worker
      duration: 1000,
    },
    
    // Retry settings
    defaultJobOptions: {
      attempts: 3,
      backoff: {
        type: 'exponential',
        delay: 1000,
      },
    },
  });
}

// สร้าง 100 workers
const WORKER_COUNT = 100;
const workers = [];

for (let i = 0; i < WORKER_COUNT; i++) {
  workers.push(createNotificationWorker(i));
}

console.log(`Started ${WORKER_COUNT} notification workers`);

/**
 * Process individual notification (ส่งไปยัง user คนเดียว)
 */
async function processIndividualNotification({ userId, notification }) {
  const tokens = await getUserTokens(userId);
  
  if (tokens.length === 0) return;
  
  const tokenList = tokens.map(t => t.token);
  await sendToDevices(tokenList, notification);
}

/**
 * Process batch of tokens
 * ใช้สำหรับ follower notifications
 */
async function processBatchTokenNotification({ tokens, notification }) {
  await sendToDevices(tokens, notification);
}

/**
 * Process SOS notification
 * ใช้ Topic messaging สำหรับ speed
 */
async function processSOSNotification({ areaCode, alert, streamUrl }) {
  const topic = `sos-area-${areaCode}`;
  
  // ส่งผ่าน Topic (fast, FCM handles fan-out)
  await sendToTopic(topic, {
    title: '🚨 SOS Alert ใกล้คุณ',
    body: `${alert.emergencyType}: ${alert.description?.slice(0, 80) || 'คลิกเพื่อดูรายละเอียด'}`,
    data: {
      type: 'SOS_ALERT',
      alertId: alert.id.toString(),
      streamUrl: streamUrl || '',
      latitude: alert.latitude.toString(),
      longitude: alert.longitude.toString(),
    },
    priority: 'high',
  });
}

module.exports = { notificationQueue, workers };
```

### Step 745: Notification Queue Publisher

```javascript
// services/notification-publisher.js - เพิ่ม jobs ลงใน queue

const { notificationQueue } = require('../workers/notification-worker');

/**
 * ส่ง notification ไปยัง user คนเดียว
 */
async function notifyUser(userId, notification) {
  await notificationQueue.add('individual', {
    type: 'INDIVIDUAL',
    data: { userId, notification },
  }, {
    removeOnComplete: 100,
    removeOnFail: 50,
  });
}

/**
 * ส่ง SOS alert notification ทันที (priority queue)
 */
async function notifySOSAlert(alert, streamUrl) {
  await notificationQueue.add('sos-alert', {
    type: 'SOS_ALERT',
    data: {
      areaCode: alert.areaCode,
      alert,
      streamUrl,
    },
  }, {
    priority: 1,          // Highest priority
    removeOnComplete: true,
    attempts: 5,          // Retry 5 ครั้ง (critical!)
  });
}

/**
 * Fan-out notification ไปยัง followers ทั้งหมด
 * ใช้สำหรับ new post notifications
 */
async function notifyFollowers(authorId, notification) {
  const db = require('../config/db');
  
  // ดึง followers ทั้งหมด
  const { rows: followers } = await db.query(`
    SELECT f.follower_id, dt.token, dt.platform
    FROM follows f
    JOIN device_tokens dt ON dt.user_id = f.follower_id
    WHERE f.following_id = $1
      AND f.is_active = TRUE
      AND dt.is_active = TRUE
    ORDER BY f.follower_id
  `, [authorId]);
  
  if (followers.length === 0) return;
  
  // Split ออกเป็น batches ของ 500 tokens
  const tokens = followers.map(f => f.token);
  const batchSize = 500;
  const jobs = [];
  
  for (let i = 0; i < tokens.length; i += batchSize) {
    jobs.push({
      name: 'batch-tokens',
      data: {
        type: 'BATCH_TOKENS',
        data: {
          tokens: tokens.slice(i, i + batchSize),
          notification,
        },
      },
      opts: {
        removeOnComplete: 50,
        attempts: 3,
      },
    });
  }
  
  // เพิ่ม jobs ทั้งหมดใน one call
  await notificationQueue.addBulk(jobs);
  
  console.log(`Queued ${jobs.length} batch jobs for ${followers.length} followers`);
}

module.exports = { notifyUser, notifySOSAlert, notifyFollowers };
```

### Step 746: Notification Rate Limiting

```javascript
// middleware/notification-rate-limit.js

const redis = require('../config/redis');

const RATE_LIMITS = {
  SOS_ALERT:  { max: 20, window: 3600 },   // SOS: 20/hour (ไม่ limit มาก)
  NEW_POST:   { max: 5, window: 3600 },    // Posts: 5/hour per user
  FOLLOW:     { max: 10, window: 86400 },  // Follow: 10/day
  DM:         { max: 50, window: 3600 },   // DM: 50/hour
  SYSTEM:     { max: 3, window: 86400 },   // System: 3/day
};

/**
 * ตรวจสอบว่า user ยัง receive notifications ได้ไหม
 */
async function checkNotificationRateLimit(userId, notificationType) {
  const limit = RATE_LIMITS[notificationType] || RATE_LIMITS.SYSTEM;
  const key = `notif-rate:${userId}:${notificationType}`;
  
  const count = await redis.incr(key);
  
  if (count === 1) {
    await redis.expire(key, limit.window);
  }
  
  if (count > limit.max) {
    console.log(`Rate limited notification ${notificationType} for user ${userId}: ${count}/${limit.max}`);
    return false;
  }
  
  return true;
}

/**
 * Batch ตรวจสอบ rate limits สำหรับหลาย users
 */
async function filterRateLimitedUsers(userIds, notificationType) {
  const limit = RATE_LIMITS[notificationType] || RATE_LIMITS.SYSTEM;
  
  const pipeline = redis.pipeline();
  for (const userId of userIds) {
    pipeline.get(`notif-rate:${userId}:${notificationType}`);
  }
  
  const results = await pipeline.exec();
  
  return userIds.filter((userId, idx) => {
    const count = parseInt(results[idx][1] || '0');
    return count < limit.max;
  });
}

module.exports = { checkNotificationRateLimit, filterRateLimitedUsers };
```

### Step 747: APNs Direct (iOS High Priority)

```javascript
// services/apns.js - Apple Push Notification Service
// ใช้สำหรับ SOS alerts บน iOS ที่ต้องการ critical alerts

const http2 = require('http2');
const { readFileSync } = require('fs');
const jwt = require('jsonwebtoken');

const APNS_KEY_ID = process.env.APNS_KEY_ID;
const APNS_TEAM_ID = process.env.APNS_TEAM_ID;
const APNS_BUNDLE_ID = process.env.APNS_BUNDLE_ID;
const APNS_KEY = readFileSync(process.env.APNS_KEY_PATH || '/etc/certs/apns.p8');

let apnsToken = null;
let tokenGeneratedAt = 0;

/**
 * Generate JWT สำหรับ APNs authentication
 */
function getAPNsToken() {
  const now = Math.floor(Date.now() / 1000);
  
  // Regenerate ทุก 50 นาที (token หมดอายุใน 60 นาที)
  if (!apnsToken || now - tokenGeneratedAt > 3000) {
    apnsToken = jwt.sign(
      { iss: APNS_TEAM_ID, iat: now },
      APNS_KEY,
      { algorithm: 'ES256', keyid: APNS_KEY_ID }
    );
    tokenGeneratedAt = now;
  }
  
  return apnsToken;
}

/**
 * ส่ง Critical Alert ไปยัง iOS device
 * Critical Alerts ต้องการ APNs entitlement พิเศษจาก Apple
 */
async function sendCriticalAlert(deviceToken, notification) {
  return new Promise((resolve, reject) => {
    const client = http2.connect('https://api.push.apple.com:443', {
      key: APNS_KEY,
      cert: null,  // Token-based auth
    });
    
    const headers = {
      ':method': 'POST',
      ':path': `/3/device/${deviceToken}`,
      ':scheme': 'https',
      ':authority': 'api.push.apple.com',
      'authorization': `bearer ${getAPNsToken()}`,
      'apns-topic': APNS_BUNDLE_ID,
      'apns-priority': '10',        // Critical = 10, Normal = 5
      'apns-push-type': 'alert',
      'content-type': 'application/json',
    };
    
    const payload = JSON.stringify({
      aps: {
        alert: {
          title: notification.title,
          body: notification.body,
        },
        sound: {
          critical: 1,              // Critical alert (plays even in Do Not Disturb!)
          name: 'sos_alert.aiff',
          volume: 1.0,
        },
        badge: 1,
        'content-available': 1,
        'mutable-content': 1,
      },
      data: notification.data || {},
    });
    
    const req = client.request(headers);
    
    req.on('response', (headers) => {
      const status = headers[':status'];
      if (status === 200) {
        resolve({ success: true });
      } else {
        reject(new Error(`APNs error: ${status}`));
      }
    });
    
    req.write(payload);
    req.end();
    
    client.on('error', reject);
  });
}

module.exports = { sendCriticalAlert };
```

### Step 748: Cost Optimization

```javascript
// services/notification-optimizer.js - Cost optimization

/**
 * Deduplicate notifications ก่อนส่ง
 * ป้องกัน user รับ SOS alert ซ้ำจาก multiple topics
 */
async function deduplicateNotification(userId, notificationId) {
  const key = `notif-dedup:${userId}:${notificationId}`;
  
  // NX: Only set if not exists
  const set = await redis.set(key, '1', 'EX', 3600, 'NX');
  
  return set === 'OK';  // true = not duplicate, false = already sent
}

/**
 * Smart notification batching
 * รวม notifications หลาย types เข้าด้วยกัน (digest)
 */
async function createNotificationDigest(userId, notifications) {
  if (notifications.length === 1) {
    return notifications[0];
  }
  
  // สร้าง digest notification
  const types = [...new Set(notifications.map(n => n.type))];
  
  if (types.length === 1 && types[0] === 'NEW_POST') {
    const count = notifications.length;
    const authors = [...new Set(notifications.map(n => n.data.authorName))];
    
    return {
      title: `${count} โพสต์ใหม่สำหรับคุณ`,
      body: authors.length === 1
        ? `${authors[0]} โพสต์ ${count} ครั้ง`
        : `${authors[0]} และอีก ${authors.length - 1} คน`,
      data: {
        type: 'POST_DIGEST',
        count: count.toString(),
      },
    };
  }
  
  return {
    title: `${notifications.length} การแจ้งเตือนใหม่`,
    body: 'แตะเพื่อดูทั้งหมด',
    data: { type: 'DIGEST' },
  };
}

/**
 * ตรวจสอบ Do Not Disturb hours
 */
async function shouldSendNotification(userId, notificationType) {
  // SOS alerts ส่งเสมอ (ไม่ว่าจะ DND)
  if (notificationType === 'SOS_ALERT') return true;
  
  const prefs = await getUserNotificationPrefs(userId);
  
  if (!prefs.dndEnabled) return true;
  
  const now = new Date();
  const hour = now.getHours();
  const dndStart = prefs.dndStart || 22;  // 10 PM
  const dndEnd = prefs.dndEnd || 7;       // 7 AM
  
  const isDND = dndStart > dndEnd
    ? hour >= dndStart || hour < dndEnd  // crosses midnight
    : hour >= dndStart && hour < dndEnd;
  
  return !isDND;
}

module.exports = { deduplicateNotification, createNotificationDigest, shouldSendNotification };
```

### Step 749: Notification Analytics

```javascript
// kafka/consumers/notification-analytics.js
// Track notification delivery and open rates

const { Kafka } = require('kafkajs');

const kafka = new Kafka({
  clientId: 'notification-analytics',
  brokers: (process.env.KAFKA_BROKERS || 'localhost:9092').split(','),
});

const consumer = kafka.consumer({ groupId: 'notification-analytics' });

async function startNotificationAnalytics() {
  await consumer.connect();
  await consumer.subscribe({ topics: ['notification-events'] });
  
  await consumer.run({
    eachMessage: async ({ message }) => {
      const event = JSON.parse(message.value.toString());
      
      switch (event.type) {
        case 'SENT':
          await trackNotificationSent(event);
          break;
        
        case 'DELIVERED':
          await trackNotificationDelivered(event);
          break;
        
        case 'OPENED':
          await trackNotificationOpened(event);
          break;
        
        case 'FAILED':
          await trackNotificationFailed(event);
          break;
      }
    },
  });
}

async function trackNotificationOpened(event) {
  await db.query(`
    UPDATE notification_logs
    SET opened_at = NOW(), is_opened = TRUE
    WHERE notification_id = $1 AND user_id = $2
  `, [event.notificationId, event.userId]);
  
  // อัพเดท open rate metrics
  await redis.incr(`notif-opens:${event.notificationType}:${getDayKey()}`);
}

function getDayKey() {
  return new Date().toISOString().slice(0, 10);
}
```

### Step 750: Notification Dashboard

```javascript
// routes/admin/notifications.js - Admin dashboard

router.get('/stats', async (req, res) => {
  const today = new Date().toISOString().slice(0, 10);
  
  const [sentToday, openRate, failureRate, queueDepth] = await Promise.all([
    redis.get(`notif-sent:${today}`),
    calculateOpenRate(today),
    calculateFailureRate(today),
    notificationQueue.getWaitingCount(),
  ]);
  
  res.json({
    today: {
      sent: parseInt(sentToday || '0'),
      openRate: openRate.toFixed(2),
      failureRate: failureRate.toFixed(2),
    },
    queue: {
      depth: queueDepth,
      workers: WORKER_COUNT,
      processingRate: await getProcessingRate(),
    },
    estimated_time_to_clear: queueDepth > 0
      ? `${Math.ceil(queueDepth / (WORKER_COUNT * 10))} seconds`
      : '0 seconds',
  });
});

async function calculateOpenRate(date) {
  const sent = parseInt(await redis.get(`notif-sent:${date}`) || '1');
  const opened = parseInt(await redis.get(`notif-opens:${date}`) || '0');
  return (opened / sent) * 100;
}
```

---

## 🔧 Configuration Files

```javascript
// config/notification.js
module.exports = {
  fcm: {
    projectId: process.env.FIREBASE_PROJECT_ID,
    maxTokensPerRequest: 500,
  },
  
  workers: {
    count: 100,
    concurrencyPerWorker: 10,
    rateLimit: { max: 100, duration: 1000 },
  },
  
  rateLimits: {
    SOS_ALERT:  { max: 20, window: 3600 },
    NEW_POST:   { max: 5, window: 3600 },
    SYSTEM:     { max: 3, window: 86400 },
  },
  
  dedup: {
    ttl: 3600,
  },
};
```

---

## 🧪 Testing

```bash
# Test FCM token registration
curl -X POST https://api.chuaikan.com/api/devices/register \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"token":"test-fcm-token","platform":"android"}'

# Monitor queue
npx bullmq-monitor

# Check queue stats
node -e "
  const { notificationQueue } = require('./workers/notification-worker');
  notificationQueue.getJobCounts().then(console.log);
"
```

---

## ❌ Common Errors & Solutions

### Error 1: FCM Quota Exceeded

```javascript
// ปัญหา: FCM 429 Too Many Requests
// แก้ไข: Implement exponential backoff

const worker = new Worker('notifications', processor, {
  connection,
  limiter: {
    max: 500,        // ลดลงจาก 1000
    duration: 1000,
  },
  defaultJobOptions: {
    attempts: 5,
    backoff: {
      type: 'exponential',
      delay: 2000,   // Start at 2s, then 4s, 8s, 16s, 32s
    },
  },
});
```

### Error 2: Stale tokens accumulate

```bash
# ปัญหา: Database เต็มไปด้วย stale tokens
# แก้ไข: Cleanup job ทุกวัน

-- Run nightly
DELETE FROM device_tokens
WHERE is_active = FALSE
  AND invalidated_at < NOW() - INTERVAL '30 days';

-- หรือ archive แทน delete
UPDATE device_tokens
SET archived = TRUE
WHERE invalidated_at < NOW() - INTERVAL '7 days';
```

---

## ✅ Checklist

- [ ] ตั้งค่า Firebase Admin SDK
- [ ] Implement FCM token management (register, update, invalidate)
- [ ] สร้าง 100 BullMQ workers สำหรับ processing
- [ ] FCM multicast batch sending (500 tokens/request)
- [ ] Topic messaging สำหรับ SOS area alerts
- [ ] APNs Critical Alerts สำหรับ iOS SOS
- [ ] Rate limiting ป้องกัน spam notifications
- [ ] Notification deduplication
- [ ] DND (Do Not Disturb) hours support
- [ ] Analytics: track delivery, open rates
- [ ] Monitor queue depth ด้วย Prometheus

---

## 🔗 References

- [FCM v1 API](https://firebase.google.com/docs/cloud-messaging/http-server-ref)
- [BullMQ Documentation](https://docs.bullmq.io/)
- [APNs Documentation](https://developer.apple.com/documentation/usernotifications)
- [FCM Topic Messaging](https://firebase.google.com/docs/cloud-messaging/js/topic-messaging)
- [APNs Critical Alerts](https://developer.apple.com/documentation/usernotificationsui/customizing-the-appearance-of-notifications)

---

*Part 075 | Road to 1,000,000 Users/Day | chuaikan.com*
