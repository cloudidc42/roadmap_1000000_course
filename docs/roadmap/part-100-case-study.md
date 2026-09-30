# Part 100: Case Study — chuaikan.com จาก 0 ถึง 1M Users/Day

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** World Class
> **Steps:** 991-1000
> **เวลาโดยประมาณ:** 4 ชั่วโมง
> **Prerequisites:** Part 091-099 (All World Class parts)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

Part นี้เป็นการรวมทุกอย่างที่เรียนรู้มาในรูปแบบ Case Study ของ chuaikan.com ที่เดินทางจาก 0 ไปถึง 1 ล้าน users/day เราจะเห็น:

- การตัดสินใจ Technical ที่แต่ละ Milestone
- สิ่งที่ Work และไม่ Work จริงๆ
- Lessons Learned ที่นำไปใช้ได้
- Final Architecture ที่รองรับ 1M users/day

---

## 📖 chuaikan.com Growth Story

chuaikan.com (ชวยกัน = "Help Each Other") เป็น Community Platform สำหรับ Disaster Response ในประเทศไทย เริ่มต้นจาก Hackathon Project ใน 2022 และเติบโตขึ้นจนกลายเป็น Critical Infrastructure ที่คนไทยใช้เมื่อเกิดน้ำท่วม ไฟป่า และภัยพิบัติต่างๆ

---

## Month 1: MVP Launch (100 Users/Day)

### สถานการณ์

ทีม 2 คน (Full-stack Developer 1 คน, Designer 1 คน) เปิดตัวหลัง Hackathon ชนะ โดยมีงบ $500/เดือนจาก Prize money

### Architecture

```
MVP Architecture (Month 1):

Internet → Single VPS (DigitalOcean, $40/month)
              ├── Nginx
              ├── Node.js (Express)
              ├── PostgreSQL (same server)
              └── Redis (same server)

Static Files → Cloudflare Free (CDN)
Images → S3 ($10/month)
```

### Technical Decisions Made

```javascript
// ตัดสินใจ: Monolith ก่อน เพราะทีมเล็ก
// เหตุผล: Microservices ตอนนี้ = premature optimization

// database schema แบบง่าย
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  username VARCHAR(50) UNIQUE,
  email VARCHAR(255) UNIQUE,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE sos_alerts (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id),
  latitude DECIMAL(9,6),
  longitude DECIMAL(9,6),
  severity VARCHAR(20),
  description TEXT,
  status VARCHAR(20) DEFAULT 'active',
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_sos_location ON sos_alerts USING GIST (
  ST_MakePoint(longitude, latitude)
);
```

### Metrics at Month 1

```
Users/day: 100 (peak: 500 during flood event)
Posts/day: 300
SOS alerts/day: 20
Uptime: 94% (crashed once during high traffic)
Monthly cost: $60
Cost per user: $0.02/day
```

### What Worked

- Simple architecture = fast to build, easy to debug
- Cloudflare free CDN = fast enough for Thailand
- PostgreSQL PostGIS = location queries ทำได้ง่ายมาก

### What Didn't Work

- Server crashed ตอน flood season เพราะ single server
- ไม่มี monitoring — รู้ว่า down หลังจาก user แจ้ง
- ไม่มี backup — โชคดีที่ไม่มีข้อมูลสูญหาย

---

## Month 3: 1,000 Users/Day

### สถานการณ์

หลัง Flood event ใหญ่ใน Chiang Mai news cover → users เพิ่ม 10x ใน 1 สัปดาห์ ทีมขยายเป็น 5 คน ได้ Angel Investment

### Architecture Changes

```
Month 3 Architecture:

Internet → Cloudflare (Pro, $20/month)
              ↓
           AWS ALB ($20/month)
              ↓
        ┌─────┴─────┐
   App Server 1   App Server 2   (t3.medium × 2, $70/month each)
        └─────┬─────┘
              ↓
     PostgreSQL RDS (db.t3.medium, $80/month)
     + Read Replica ($40/month)
              ↓
     Redis ElastiCache (cache.t2.micro, $25/month)
              ↓
     S3 + CloudFront ($50/month)

Monthly: ~$400 → $500/month with monitoring
```

### Critical Technical Decisions

```javascript
// 1. Stateless sessions — ย้าย session ไป Redis
// เพราะต้อง run 2 app servers

// ก่อน: เก็บ session ใน memory
const sessions = new Map(); // พังเมื่อมี load balancer!

// หลัง: เก็บ session ใน Redis
const session = require('express-session');
const RedisStore = require('connect-redis')(session);

app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: { secure: true, maxAge: 24 * 60 * 60 * 1000 }
}));
```

```javascript
// 2. Read/Write Splitting
// เพราะ 90% ของ traffic คือ read

const readPool = new Pool({ 
  host: process.env.DB_REPLICA_HOST,
  database: 'chuaikan' 
});

const writePool = new Pool({ 
  host: process.env.DB_PRIMARY_HOST,
  database: 'chuaikan' 
});

// Middleware สำหรับ route DB queries
async function withDB(req, res, next) {
  req.db = {
    read: readPool,
    write: writePool,
    // Convenience: query ไป read replica ถ้าเป็น GET
    auto: req.method === 'GET' ? readPool : writePool,
  };
  next();
}
```

### Metrics at Month 3

```
Users/day: 1,000 (peak: 5,000)
Posts/day: 3,000
SOS alerts/day: 100
Uptime: 99.2%
Monthly cost: $500
Cost per user: $0.017/day
Response time P95: 180ms
```

### Key Lesson

**"ย้ายไปใช้ Managed Services เร็วกว่าที่คิด"**

ก่อนหน้า manage database เองบน EC2 — waste เวลา 20% ไปกับ backup, updates, monitoring database

หลังจากย้ายไป RDS — team focus กับ product 100%

---

## Month 6: 10,000 Users/Day

### สถานการณ์

ได้รับ Series A Funding ทีมขยายเป็น 15 คน มีคู่แข่งรายแรกเข้าตลาด ต้องการ feature velocity สูง

### Architecture Changes

```
Month 6 Architecture:

Internet → Cloudflare (WAF + CDN)
              ↓
        AWS Route53 (latency-based)
              ↓
           AWS ALB
              ↓
    Auto Scaling Group (3-10 nodes)
    ECS Fargate (ย้ายจาก EC2)
              ↓
    PostgreSQL RDS (db.r6g.large)
    + 2 Read Replicas
              ↓
    Redis ElastiCache Cluster (3 nodes)
    + Elasticsearch (Search)
              ↓
    S3 + CloudFront (Multi-region)
    
Monitoring:
- Datadog (APM + Logs + Infrastructure)
- PagerDuty (On-call alerts)

Monthly: ~$3,000
```

### Critical Technical Decisions

```javascript
// 1. Cache Strategy
// ปัญหา: 80% ของ DB reads คือ feed — ช้า

// Cache-aside pattern
async function getUserFeed(userId) {
  const cacheKey = `feed:v2:${userId}`;
  
  // ลองอ่าน cache ก่อน
  const cached = await redis.get(cacheKey);
  if (cached) {
    metrics.increment('cache.feed.hit');
    return JSON.parse(cached);
  }
  
  metrics.increment('cache.feed.miss');
  
  // Query DB
  const feed = await db.read.query(`
    SELECT p.*, u.username, u.avatar_url,
           COUNT(l.id) as likes_count,
           COUNT(c.id) as comments_count
    FROM posts p
    JOIN users u ON p.user_id = u.id
    JOIN follows f ON p.user_id = f.following_id
    LEFT JOIN likes l ON p.id = l.post_id
    LEFT JOIN comments c ON p.id = c.post_id
    WHERE f.follower_id = $1
    GROUP BY p.id, u.id
    ORDER BY p.created_at DESC
    LIMIT 20
  `, [userId]);
  
  // Cache 60 วินาที
  await redis.setex(cacheKey, 60, JSON.stringify(feed.rows));
  
  return feed.rows;
}
```

```bash
# 2. ย้าย CI/CD ไปใช้ GitHub Actions

# .github/workflows/deploy.yml
name: Deploy to Production
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Tests
        run: npm test
      
      - name: Build Docker Image
        run: |
          docker build -t chuaikan/api:$GITHUB_SHA .
          docker push chuaikan/api:$GITHUB_SHA
      
      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster production \
            --service api \
            --force-new-deployment
```

### Metrics at Month 6

```
Users/day: 10,000 (peak: 50,000 during major flood)
Posts/day: 30,000
SOS alerts/day: 500
Uptime: 99.8%
Monthly cost: $3,000
Cost per user: $0.01/day
Response time P95: 120ms
Deploy frequency: 5-10 times/day
```

---

## Month 12: 100,000 Users/Day

### สถานการณ์

ได้ Partnership กับ กทม. และ กรมป้องกันและบรรเทาสาธารณภัย ทีมขยายเป็น 40 คน เริ่มเห็น Database Bottleneck ชัดเจน

### Architecture Changes

```
Month 12 Architecture — Kubernetes Era:

                 Cloudflare
                      ↓
                 AWS ALB
                      ↓
              ┌───────┴───────┐
         API Gateway      WebSocket Gateway
              ↓                  ↓
    ┌─────────┼─────────┐        │
Feed Service  SOS Service  User Service
    │              │            │
    └──────────────┼────────────┘
                   ↓
     PostgreSQL (Sharded × 4)
     + Elasticsearch
     + Redis Cluster (6 nodes)
     + Kafka (Event Streaming)
                   ↓
           S3 + CloudFront

Kubernetes:
- EKS Cluster (20 nodes)
- Horizontal Pod Autoscaling
- ArgoCD for GitOps deploys

Monthly: ~$15,000
```

### Critical Technical Decisions

```javascript
// 1. Database Sharding — ตัดสินใจยากที่สุดในประวัติ chuaikan.com

// เหตุผล: PostgreSQL primary ถึง 70% CPU utilization ตลอดเวลา
// Write throughput: 8,000 writes/sec (limit ประมาณ 10,000)

// Shard key เลือก user_id % 4
// เพราะ: ใช้ user_id มากที่สุดในทุก query

function getShardIndex(userId) {
  return userId % 4;
}

const shards = [
  new Pool({ host: 'postgres-shard-0.internal' }),
  new Pool({ host: 'postgres-shard-1.internal' }),
  new Pool({ host: 'postgres-shard-2.internal' }),
  new Pool({ host: 'postgres-shard-3.internal' }),
];

function getShardPool(userId) {
  return shards[getShardIndex(userId)];
}

// Migration ใช้เวลา 3 สัปดาห์
// ความยากที่สุด: ทำ zero-downtime migration
// วิธี: dual-write period → verify → switch reads → remove old
```

```javascript
// 2. Event-Driven Architecture ด้วย Kafka
// เพราะ: SOS alert ต้องแจ้งหลาย services พร้อมกัน

// ก่อน: Synchronous calls (ช้า, brittle)
async function createSOS(data) {
  await db.save(data);
  await notificationService.sendPushNotifications(data); // อาจล้มเหลว
  await emailService.sendAlerts(data);                   // อาจล้มเหลว  
  await analyticsService.trackEvent(data);               // ช้า
}

// หลัง: Kafka Event Streaming
async function createSOS(data) {
  await db.save(data); // Strong consistency สำหรับ SOS
  
  // Publish event — services อื่นรับ async
  await kafka.publish('sos.created', {
    sosId: data.id,
    location: { lat: data.lat, lng: data.lng },
    severity: data.severity,
    timestamp: Date.now(),
  });
  
  return data; // ตอบ user ทันที
}

// ในแต่ละ service: consume events independently
kafka.consume('sos.created', async (event) => {
  await sendPushNotifications(event);
});
```

### Metrics at Month 12

```
Users/day: 100,000 (peak: 500,000)
Posts/day: 300,000
SOS alerts/day: 2,000
Uptime: 99.95%
Monthly cost: $15,000
Cost per user: $0.0005/day
Response time P95: 80ms
Deploy frequency: 20+ times/day
Engineering team: 40 people
```

---

## Month 18: 500,000 Users/Day

### Architecture Changes

```
Multi-Region Active-Active:

           Users (Thailand)
                 ↓
        Cloudflare Anycast
                 ↓
    ┌────────────┴────────────┐
    │                        │
ap-southeast-1            ap-northeast-1
(Singapore — Primary)    (Tokyo — Secondary)
    │                        │
    └───────── Kafka ─────────┘
              (Global Replication)
```

---

## Month 24: 1,000,000 Users/Day — TARGET ACHIEVED!

### Final Architecture

```
                 ┌─────────────────────────────┐
                 │     Global Users             │
                 └─────────────┬───────────────┘
                               ↓
                    Cloudflare Global Network
                    (WAF + DDoS + CDN)
                               ↓
                    Route53 (Latency-based)
                               ↓
              ┌────────────────┴───────────────┐
              │                                │
     ap-southeast-1 (Primary)        ap-northeast-1 (DR)
              │                                │
     ┌────────┴────────┐             ┌────────┴────────┐
  API Gateway       API Gateway   API GW            API GW
     │                                              │
     ├── Feed Service (20 pods)
     ├── SOS Service (10 pods, HIGH PRIORITY)
     ├── User/Auth Service (15 pods)
     ├── Notification Service (10 pods)
     ├── Search Service (8 pods, Elasticsearch)
     └── Media Service (10 pods)
              │
     ┌────────┴──────────────────────────┐
     │          Data Layer               │
     │                                   │
     │  PostgreSQL Sharded × 8           │
     │  + Read Replicas × 3 each         │
     │  Aurora Global (SOS only)         │
     │                                   │
     │  Redis Cluster (12 nodes)         │
     │  Elasticsearch (9 nodes)          │
     │  ClickHouse (Analytics)           │
     │  Kafka (Event Streaming)          │
     └───────────────────────────────────┘
              │
     S3 + CloudFront (Multi-region)
```

### Metrics at Month 24 (Goal Achieved!)

```
Users/day: 1,000,000 ✅
Posts/day: 2,000,000
SOS alerts/day: 5,000
Uptime: 99.99% (SOS), 99.95% (App)
Monthly cost: $20,000
Cost per user: $0.00067/day
Response time P95: 60ms (Feed), 40ms (SOS)
Deploy frequency: 50+ times/day
Engineering team: 80 people in 8 teams
```

---

## Lessons Learned

### สิ่งที่ Work

1. **Start Simple, Scale When Needed:** Monolith → Microservices เมื่อ pain จริงๆ
2. **Managed Services > Self-Managed:** RDS, ElastiCache, EKS ประหยัดเวลาได้มาก
3. **Data-Driven Decisions:** Load test ก่อน scale ทุกครั้ง
4. **Stateless First:** Session ใน Redis ตั้งแต่ month 3 ทำให้ scale ง่ายมาก
5. **Cache Everything Possible:** Cache hit rate > 90% ลด cost ได้มาก

### สิ่งที่ไม่ Work

1. **Premature Optimization:** เคย implement Kafka ตั้งแต่ month 2 (ไม่จำเป็น, เสียเวลา)
2. **Over-Engineering:** Microservices สำหรับ 1,000 users ทำให้ debug ยาก
3. **Manual Everything:** ขาด monitoring ช่วงแรก ทำให้รู้ว่า down หลังจาก user complaint

### Top 5 Technical Decisions ที่ Impact มากที่สุด

```
1. PostgreSQL + PostGIS (Month 1)
   → Location queries ง่ายมาก ประหยัดเวลา development เป็นเดือน

2. Stateless Sessions ใน Redis (Month 3)
   → ทำให้ horizontal scaling ง่ายตลอด journey

3. Read Replicas (Month 3)
   → ลด DB load 70% ด้วย effort น้อยมาก

4. Kafka Event Streaming (Month 12)
   → decoupled services, ทำให้ feature velocity เพิ่มขึ้น 2x

5. Multi-Region Active-Active (Month 18)
   → ผ่าน AWS Region outage ครั้งใหญ่โดยไม่มี downtime
```

---

## 🔧 Final Architecture เป็น Code

```yaml
# final-k8s-architecture.yaml (simplified)

# Feed Service — High availability
apiVersion: apps/v1
kind: Deployment
metadata:
  name: feed-service
  namespace: production
  annotations:
    deployment.kubernetes.io/revision: "47"
spec:
  replicas: 20
  strategy:
    rollingUpdate:
      maxUnavailable: 2
      maxSurge: 4
  selector:
    matchLabels:
      app: feed-service
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: feed-service
            topologyKey: topology.kubernetes.io/zone
      containers:
      - name: feed-service
        image: chuaikan/feed-service:v4.2.1
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "2000m"
            memory: "2Gi"
        env:
        - name: DB_READ_URL
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: read-url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: redis-credentials
              key: url
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/deep
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
---
# SOS Service — Ultra high availability
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sos-service
  namespace: production
spec:
  replicas: 10
  # SOS ใช้ PriorityClass สูงสุด — ไม่ถูก evict
  template:
    spec:
      priorityClassName: system-cluster-critical
      containers:
      - name: sos-service
        image: chuaikan/sos-service:v3.8.0
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "4000m"  # burst ได้เยอะกว่า
            memory: "4Gi"
```

---

## What's Next: Road to 10M Users/Day

```
เป้าหมาย 2026: 10,000,000 users/day

สิ่งที่ต้องทำ:
1. Global Expansion: Southeast Asia (Vietnam, Indonesia)
2. Edge Computing: Cloudflare Workers สำหรับ API edge caching
3. ML-Powered Feed: Personalized ranking ด้วย Deep Learning
4. Computer Vision: Auto-detect flood severity จาก photos
5. Infrastructure: 3 Active Regions (Singapore, Tokyo, Jakarta)
6. Cost: ลงมาถึง $0.0005/user/day (ลด 25%)

ทีม: 150-200 engineers
Monthly Cost: ~$80,000-100,000
```

---

## ✅ Checklist — Journey to 1M

- [ ] Month 1: Monolith launch, basic monitoring
- [ ] Month 3: Stateless sessions, Read replicas, Load balancer
- [ ] Month 6: Auto scaling, Cache strategy, CI/CD automated
- [ ] Month 12: Kubernetes, Microservices, DB Sharding, Kafka
- [ ] Month 18: Multi-region, SLOs, SRE team
- [ ] Month 24: 1M users/day achieved!
- [ ] Document all Architecture Decision Records
- [ ] Postmortem for every major incident
- [ ] Cost per user ≤ $0.001/day
- [ ] Team: DORA Elite metrics

---

## 🔗 References

- [How Netflix Scales Its API](https://netflixtechblog.com/)
- [Airbnb Engineering Blog](https://medium.com/airbnb-engineering)
- [Twitter Engineering Blog](https://blog.twitter.com/engineering)
- [Uber Engineering Blog](https://www.uber.com/blog/engineering/)
- [Designing Data-Intensive Applications — Martin Kleppmann](https://dataintensive.net/)

---

*Part 100 | Road to 1,000,000 Users/Day | chuaikan.com*
