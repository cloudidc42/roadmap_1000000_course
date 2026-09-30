# Part 092: Distributed Systems Fundamentals

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** World Class
> **Steps:** 911-920
> **เวลาโดยประมาณ:** 5 ชั่วโมง
> **Prerequisites:** Part 091 (100M Architecture), Part 041 (DB Sharding)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

Distributed Systems คือหัวใจของ Backend ที่ Scale ได้ ใน Part นี้เราจะเข้าใจ:

- CAP Theorem: ทำไมถึงเลือกได้แค่ 2 ใน 3
- BASE vs ACID: เมื่อไรต้องการ Consistency แบบไหน
- Eventual Consistency ทำงานยังไงกับ Feed ของ chuaikan.com
- Distributed Transactions ที่ปลอดภัยด้วย Saga Pattern
- Distributed Locking เพื่อป้องกัน Race Condition

---

## 📖 ทฤษฎีและแนวคิด

### 1. CAP Theorem

Eric Brewer เสนอ CAP Theorem ในปี 2000 ซึ่งบอกว่า:

> **Distributed System ไม่สามารถรับประกัน C, A, P ได้พร้อมกันทั้งหมด — ต้องเลือกแค่ 2**

```
         Consistency
              /\
             /  \
            /    \
           /      \
          /  CP    \  CA
         /----------\
        /     AP     \
       /______________\
  Availability    Partition
                 Tolerance
```

**C — Consistency:**
ทุก node เห็นข้อมูลเหมือนกัน ณ เวลาเดียวกัน
หลัง write สำเร็จ ทุก read จะได้ข้อมูลใหม่

**A — Availability:**
ทุก request ได้รับ response เสมอ (แม้อาจไม่ใช่ข้อมูลล่าสุด)
System ไม่ reject request

**P — Partition Tolerance:**
System ยังทำงานได้แม้ network ขาดระหว่าง nodes

**ความจริงคือ Partition Tolerance ไม่ใช่ option:**
ใน production network partition เกิดขึ้นเสมอ ดังนั้นเราต้องเลือกระหว่าง **CP** หรือ **AP**

### 2. CP vs AP ใน chuaikan.com

```
chuaikan.com Services:

┌─────────────────────────┬────────┬──────────────────────────┐
│ Service                 │ Choice │ เหตุผล                   │
├─────────────────────────┼────────┼──────────────────────────┤
│ SOS Alerts              │ CP     │ ต้องการ Consistency 100% │
│                         │        │ ถ้า SOS ไม่ปรากฎอาจตาย  │
├─────────────────────────┼────────┼──────────────────────────┤
│ Financial Transactions  │ CP     │ ต้องการ ACID guarantee   │
├─────────────────────────┼────────┼──────────────────────────┤
│ User Feed               │ AP     │ OK ถ้า post ปรากฎช้า 1s  │
├─────────────────────────┼────────┼──────────────────────────┤
│ Like Counts             │ AP     │ Approximate count OK     │
├─────────────────────────┼────────┼──────────────────────────┤
│ User Profile            │ AP     │ Stale profile 5s คนไม่รู้ │
├─────────────────────────┼────────┼──────────────────────────┤
│ Notifications           │ AP     │ Delayed OK               │
└─────────────────────────┴────────┴──────────────────────────┘
```

### 3. ACID vs BASE

**ACID (Traditional Databases):**
- **A**tomicity: transaction สำเร็จทั้งหมดหรือล้มเหลวทั้งหมด
- **C**onsistency: data ต้องอยู่ใน valid state เสมอ
- **I**solation: transactions ไม่กระทบกัน
- **D**urability: data ที่ commit แล้วจะไม่หาย

**BASE (NoSQL/Distributed Systems):**
- **B**asically **A**vailable: System พยายามตอบ requests เสมอ
- **S**oft state: state อาจเปลี่ยนเมื่อเวลาผ่านไปแม้ไม่มี input
- **E**ventually consistent: ในที่สุดข้อมูลจะ consistent

### 4. Eventual Consistency

```
คิดแบบนี้: Facebook Feed
เมื่อ User A โพสต์รูป
- User A เห็นรูปทันที
- Friends ใน same datacenter เห็นหลัง ~1 วินาที  
- Friends ใน different region เห็นหลัง ~3 วินาที
- ทุกคนจะเห็นรูปในที่สุด (eventually)

นี่คือ Eventual Consistency
```

---

## 🛠️ Step-by-Step Implementation

### Step 911: Implement Eventual Consistency บน Feed

```javascript
// feed-publisher.js
// เมื่อ user โพสต์ — fan-out to followers

const { Kafka } = require('kafkajs');
const redis = require('ioredis');

const kafka = new Kafka({ brokers: ['kafka:9092'] });
const producer = kafka.producer();
const redisClient = new redis(process.env.REDIS_URL);

async function publishPost(post) {
  // 1. Save post to database (strong consistency)
  await db.query(
    'INSERT INTO posts (id, user_id, content, created_at) VALUES ($1, $2, $3, $4)',
    [post.id, post.userId, post.content, new Date()]
  );
  
  // 2. Publish event to Kafka (eventual consistency starts here)
  await producer.send({
    topic: 'post-published',
    messages: [{
      key: post.userId.toString(),
      value: JSON.stringify({
        postId: post.id,
        userId: post.userId,
        content: post.content,
        timestamp: Date.now(),
      }),
    }],
  });
  
  // 3. Immediately update author's own feed (sync)
  await updateFeedForUser(post.userId, post);
  
  // 4. Followers will get updated eventually (async via Kafka)
  return post;
}
```

```javascript
// feed-consumer.js
// Consumer ที่รับ events และ fan-out ไปหา followers

const consumer = kafka.consumer({ groupId: 'feed-fanout' });

async function startFeedFanout() {
  await consumer.connect();
  await consumer.subscribe({ topic: 'post-published' });
  
  await consumer.run({
    // เพิ่ม concurrency เพื่อ process เร็วขึ้น
    partitionsConsumedConcurrently: 10,
    
    eachMessage: async ({ message }) => {
      const post = JSON.parse(message.value.toString());
      
      // หา followers ของ user นี้
      const followers = await getFollowers(post.userId);
      
      // Fan-out: อัพเดท feed ของทุก follower
      // ใช้ batch pipeline เพื่อความเร็ว
      const pipeline = redisClient.pipeline();
      
      for (const followerId of followers) {
        const feedKey = `feed:${followerId}`;
        pipeline.lpush(feedKey, JSON.stringify({
          postId: post.postId,
          score: Date.now(),
        }));
        // เก็บแค่ 1000 items ล่าสุดใน feed
        pipeline.ltrim(feedKey, 0, 999);
        pipeline.expire(feedKey, 86400); // 24 hours
      }
      
      await pipeline.exec();
    },
  });
}
```

### Step 912: Strong Consistency สำหรับ SOS Alerts

```javascript
// sos-service.js
// SOS ต้องการ Strong Consistency — ใช้ ACID transaction

const { Pool } = require('pg');
const db = new Pool({ connectionString: process.env.DB_URL });

async function createSOSAlert(sosData) {
  const client = await db.connect();
  
  try {
    // เริ่ม transaction
    await client.query('BEGIN');
    
    // 1. สร้าง SOS record
    const result = await client.query(
      `INSERT INTO sos_alerts 
       (id, user_id, location, severity, description, created_at)
       VALUES ($1, $2, ST_GeomFromText($3, 4326), $4, $5, NOW())
       RETURNING *`,
      [sosData.id, sosData.userId, 
       `POINT(${sosData.lng} ${sosData.lat})`,
       sosData.severity, sosData.description]
    );
    
    // 2. Notify nearby responders (ต้องอยู่ใน same transaction)
    const nearbyResponders = await client.query(
      `SELECT user_id FROM sos_responders 
       WHERE ST_DWithin(
         location::geography,
         ST_GeomFromText($1, 4326)::geography,
         10000  -- 10km
       )`,
      [`POINT(${sosData.lng} ${sosData.lat})`]
    );
    
    // 3. บันทึก notification log
    for (const responder of nearbyResponders.rows) {
      await client.query(
        `INSERT INTO sos_notifications (sos_id, user_id, notified_at)
         VALUES ($1, $2, NOW())`,
        [sosData.id, responder.user_id]
      );
    }
    
    // Commit — ถ้า commit ไม่ได้ทุกอย่าง rollback หมด
    await client.query('COMMIT');
    
    // หลัง commit: publish event สำหรับ push notifications
    // (นอก transaction — eventual consistency OK สำหรับ push)
    await publishSOSCreatedEvent(result.rows[0]);
    
    return result.rows[0];
    
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

### Step 913: Saga Pattern สำหรับ Distributed Transactions

เมื่อต้องทำ transaction ข้าม services หลายตัว 2PC ไม่เหมาะ ใช้ Saga แทน

```javascript
// donation-saga.js
// Scenario: User บริจาค → ตัด balance → สร้าง donation record → แจ้งเตือน

class DonationSaga {
  constructor({ paymentService, donationService, notificationService }) {
    this.paymentService = paymentService;
    this.donationService = donationService;
    this.notificationService = notificationService;
  }
  
  async execute(donationRequest) {
    const steps = [
      {
        execute: () => this.paymentService.debitUser(
          donationRequest.userId,
          donationRequest.amount
        ),
        compensate: (result) => this.paymentService.refundUser(
          donationRequest.userId,
          donationRequest.amount,
          result.transactionId
        ),
      },
      {
        execute: () => this.donationService.createDonation({
          userId: donationRequest.userId,
          sosId: donationRequest.sosId,
          amount: donationRequest.amount,
        }),
        compensate: (result) => this.donationService.cancelDonation(
          result.donationId
        ),
      },
      {
        execute: () => this.notificationService.notifyDonation(
          donationRequest.sosOwnerId,
          donationRequest.amount
        ),
        // Notification failure ไม่ต้อง compensate — not critical
        compensate: null,
      },
    ];
    
    const completedSteps = [];
    
    try {
      for (const step of steps) {
        const result = await step.execute();
        completedSteps.push({ step, result });
      }
      
      return { success: true };
      
    } catch (error) {
      // Compensate ย้อนกลับทุก step ที่ทำไปแล้ว
      console.error('Saga failed, compensating...', error);
      
      for (const { step, result } of completedSteps.reverse()) {
        if (step.compensate) {
          try {
            await step.compensate(result);
          } catch (compensateError) {
            // Log compensation failure — ต้องมี manual review
            console.error('Compensation failed!', compensateError);
            await this.flagForManualReview(donationRequest, compensateError);
          }
        }
      }
      
      throw error;
    }
  }
}
```

### Step 914: Idempotency ใน Distributed Systems

```javascript
// idempotent-payment.js
// ป้องกัน double payment เมื่อ network timeout แล้ว retry

const crypto = require('crypto');

async function processPayment(paymentRequest) {
  // idempotency key = hash ของ request parameters
  const idempotencyKey = `payment:${paymentRequest.userId}:${paymentRequest.amount}:${paymentRequest.requestId}`;
  
  // Check ว่าเคย process แล้วหรือยัง
  const existing = await redis.get(idempotencyKey);
  if (existing) {
    console.log('Duplicate request, returning cached result');
    return JSON.parse(existing);
  }
  
  // ได้ distributed lock เพื่อป้องกัน race condition
  const lock = await redlock.acquire([`lock:${idempotencyKey}`], 10000);
  
  try {
    // Double-check หลัง lock
    const existingAfterLock = await redis.get(idempotencyKey);
    if (existingAfterLock) {
      return JSON.parse(existingAfterLock);
    }
    
    // Process payment
    const result = await actuallyProcessPayment(paymentRequest);
    
    // Cache result สำหรับ idempotency (24 hours)
    await redis.setex(idempotencyKey, 86400, JSON.stringify(result));
    
    return result;
    
  } finally {
    await lock.release();
  }
}
```

### Step 915: Distributed Locking ด้วย Redlock

Redlock คือ algorithm สำหรับ distributed lock ที่ใช้ Redis หลายตัว

```javascript
// redlock-setup.js
const Redlock = require('redlock');
const Redis = require('ioredis');

// ต้องใช้ Redis หลายตัว (ควรเป็น odd number)
// เพื่อป้องกัน single point of failure
const redisInstances = [
  new Redis({ host: 'redis-1.internal', port: 6379 }),
  new Redis({ host: 'redis-2.internal', port: 6379 }),
  new Redis({ host: 'redis-3.internal', port: 6379 }),
];

const redlock = new Redlock(redisInstances, {
  // ยอมรับ drift ได้ไม่เกิน 10ms
  driftFactor: 0.01,
  
  // retry 3 ครั้งก่อน throw error
  retryCount: 3,
  retryDelay: 200,  // 200ms between retries
  retryJitter: 100, // ±100ms jitter เพื่อหลีก thundering herd
});

// ตัวอย่าง: Lock สำหรับ daily SOS digest generation
async function generateDailySosDigest(date) {
  const lockKey = `digest:daily:${date}`;
  const lockTTL = 5 * 60 * 1000; // 5 minutes
  
  let lock;
  try {
    lock = await redlock.acquire([lockKey], lockTTL);
    
    // ได้ lock แล้ว — generate digest
    const digest = await buildSosDigest(date);
    await saveSosDigest(date, digest);
    await sendDigestEmails(digest);
    
  } catch (error) {
    if (error instanceof Redlock.ResourceLockedError) {
      console.log('Another instance is generating digest, skipping');
      return;
    }
    throw error;
  } finally {
    if (lock) {
      await lock.release();
    }
  }
}
```

### Step 916: Vector Clocks และ Conflict Resolution

```javascript
// vector-clock.js
// ใช้สำหรับ resolve conflicts ใน multi-region setup

class VectorClock {
  constructor(nodeId, initialClock = {}) {
    this.nodeId = nodeId;
    this.clock = { ...initialClock };
  }
  
  // Increment clock สำหรับ event ที่เกิดขึ้นที่ node นี้
  tick() {
    this.clock[this.nodeId] = (this.clock[this.nodeId] || 0) + 1;
    return { ...this.clock };
  }
  
  // Update clock เมื่อรับ message จาก node อื่น
  update(otherClock) {
    for (const [nodeId, time] of Object.entries(otherClock)) {
      this.clock[nodeId] = Math.max(
        this.clock[nodeId] || 0,
        time
      );
    }
    this.clock[this.nodeId] = (this.clock[this.nodeId] || 0) + 1;
  }
  
  // Compare: ว่า event A เกิดก่อน B, หลัง B, หรือ concurrent
  compare(otherClock) {
    let aBefore = false;
    let bBefore = false;
    
    const allNodes = new Set([
      ...Object.keys(this.clock),
      ...Object.keys(otherClock),
    ]);
    
    for (const nodeId of allNodes) {
      const aTime = this.clock[nodeId] || 0;
      const bTime = otherClock[nodeId] || 0;
      
      if (aTime < bTime) aBefore = true;
      if (aTime > bTime) bBefore = true;
    }
    
    if (aBefore && !bBefore) return 'before';
    if (bBefore && !aBefore) return 'after';
    if (!aBefore && !bBefore) return 'equal';
    return 'concurrent'; // conflict!
  }
}

// ตัวอย่าง: Resolve conflict เมื่อ user edit profile พร้อมกัน 2 regions
async function resolveProfileConflict(version1, version2) {
  const vc1 = new VectorClock('node1', version1.vectorClock);
  const comparison = vc1.compare(version2.vectorClock);
  
  switch (comparison) {
    case 'before':
      return version2; // version2 ใหม่กว่า
    case 'after':
      return version1; // version1 ใหม่กว่า
    case 'concurrent':
      // Conflict! ต้องใช้ merge strategy
      return mergeProfiles(version1, version2);
  }
}

function mergeProfiles(v1, v2) {
  // Last-write-wins สำหรับ most fields
  // ใช้ timestamp เป็น tiebreaker
  return v1.updatedAt > v2.updatedAt ? v1 : v2;
}
```

---

## 🔧 Configuration Files

### Kafka Configuration สำหรับ Exactly-Once Semantics

```yaml
# kafka-values.yaml (Helm chart values)
kafka:
  config:
    # ป้องกัน duplicate messages
    enable.idempotence: "true"
    
    # Exactly-once transactions
    transactional.id: "chuaikan-feed-producer"
    
    # Replication สำหรับ durability
    default.replication.factor: "3"
    min.insync.replicas: "2"
    
    # acks=all คือ wait สำหรับ all replicas
    acks: "all"
    
    # Retry settings
    retries: "2147483647"
    max.in.flight.requests.per.connection: "5"
```

### PostgreSQL Synchronous Replication

```sql
-- postgresql.conf — Primary server
-- Synchronous replication สำหรับ SOS data (strong consistency)
synchronous_standby_names = 'FIRST 1 (replica1, replica2)'
synchronous_commit = on

-- สำหรับ non-critical data (feed, likes):
-- synchronous_commit = off  (เร็วกว่า แต่อาจ lose recent data)
```

---

## 🧪 Testing

### Test Eventual Consistency

```javascript
// eventual-consistency.test.js
const assert = require('assert');

describe('Feed Eventual Consistency', () => {
  it('should eventually show post to followers', async () => {
    // 1. User A สร้าง post
    const post = await createPost(userA, 'SOS: น้ำท่วม!');
    
    // 2. ตรวจทันทีหลัง post — follower อาจยังไม่เห็น
    const feedImmediate = await getFeed(followerB);
    // ไม่ assert ว่าต้องเห็นทันที
    
    // 3. รอ eventual consistency
    await waitForCondition(
      async () => {
        const feed = await getFeed(followerB);
        return feed.some(p => p.id === post.id);
      },
      { timeout: 5000, interval: 200 }
    );
    
    // 4. ตอนนี้ต้องเห็นแล้ว
    const feedFinal = await getFeed(followerB);
    assert(feedFinal.some(p => p.id === post.id), 'Post should appear eventually');
  });
});

async function waitForCondition(condition, options) {
  const { timeout, interval } = options;
  const deadline = Date.now() + timeout;
  
  while (Date.now() < deadline) {
    if (await condition()) return;
    await new Promise(r => setTimeout(r, interval));
  }
  
  throw new Error(`Condition not met within ${timeout}ms`);
}
```

### Test Idempotency

```javascript
// idempotency.test.js
describe('Payment Idempotency', () => {
  it('should not double-charge on retry', async () => {
    const requestId = 'unique-request-123';
    
    // ส่ง request แรก
    const result1 = await processPayment({
      userId: 'user-1',
      amount: 100,
      requestId,
    });
    
    // Simulate network timeout แล้ว retry
    const result2 = await processPayment({
      userId: 'user-1',
      amount: 100,
      requestId, // same requestId!
    });
    
    // ต้องได้ผลลัพธ์เดิม ไม่ charge 2 ครั้ง
    assert.strictEqual(result1.transactionId, result2.transactionId);
    
    // ตรวจ balance ว่าถูกตัดแค่ครั้งเดียว
    const balance = await getBalance('user-1');
    assert.strictEqual(balance.deducted, 100);
  });
});
```

---

## ❌ Common Errors & Solutions

### Error 1: Split Brain บน Database Cluster

```
Scenario: Network partition → 2 nodes ต่างคิดว่าตัวเองเป็น Primary
Result: ทั้ง 2 nodes รับ writes → ข้อมูล diverge
```

```bash
# ป้องกัน Split Brain ด้วย STONITH (Shoot The Other Node In The Head)
# หรือใช้ Patroni (PostgreSQL HA)

# patroni.yml
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576  # 1MB
  
  # ถ้า primary ไม่ตอบ TTL > 30s → elect new primary
  # primary เก่าจะ fence ตัวเองออก (STONITH)
```

### Error 2: Kafka Consumer Lag สะสม

```bash
# ตรวจ consumer lag
kafka-consumer-groups.sh \
  --bootstrap-server kafka:9092 \
  --describe \
  --group feed-fanout

# OUTPUT ที่มีปัญหา:
# TOPIC          PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# post-published 0          1000            10000           9000  ← lag สูง!

# วิธีแก้: เพิ่ม consumer instances
kubectl scale deployment feed-fanout --replicas=10
```

---

## ✅ Checklist

- [ ] เข้าใจ CAP Theorem และเลือก CP/AP ให้แต่ละ service แล้ว
- [ ] Feed service ใช้ Eventual Consistency
- [ ] SOS service ใช้ Strong Consistency (ACID transactions)
- [ ] Saga Pattern implement แล้วสำหรับ multi-service transactions
- [ ] Idempotency key ใส่ทุก payment/critical API endpoint
- [ ] Distributed locking ด้วย Redlock ใช้งานได้
- [ ] Kafka consumer lag monitoring ตั้งค่าแล้ว
- [ ] Test Eventual Consistency ด้วย automated tests
- [ ] Split Brain scenario ป้องกันด้วย Patroni หรือ similar tool
- [ ] Documentation: ระบุ consistency model ของแต่ละ service

---

## 🔗 References

- [CAP Theorem — Gilbert and Lynch 2002](https://dl.acm.org/doi/10.1145/564585.564601)
- [Designing Data-Intensive Applications — Martin Kleppmann](https://dataintensive.net/)
- [Saga Pattern — Chris Richardson](https://microservices.io/patterns/data/saga.html)
- [Redlock Algorithm](https://redis.io/docs/manual/patterns/distributed-locks/)
- [Vector Clocks — Leslie Lamport](https://lamport.azurewebsites.net/pubs/time-clocks.pdf)
- [BASE: An ACID Alternative — Dan Pritchett](https://queue.acm.org/detail.cfm?id=1394128)

---

*Part 092 | Road to 1,000,000 Users/Day | chuaikan.com*
