# Part 091: Architecture ที่รองรับ 100M Users

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** World Class
> **Steps:** 901-910
> **เวลาโดยประมาณ:** 6 ชั่วโมง
> **Prerequisites:** Part 081-090 (Kubernetes Advanced), Part 041 (DB Sharding)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

ใน Part นี้เราจะเรียนรู้ว่า Platform ขนาดใหญ่ระดับโลกอย่าง Twitter, Facebook สร้าง Architecture อย่างไร และนำมาประยุกต์ใช้กับ chuaikan.com ตั้งแต่ 0 จนถึง 100M users โดยจะเข้าใจว่า:

- ทำไม Architecture ที่ดีสำหรับ 100 users ถึงพังที่ 100,000 users
- แต่ละ Scale มีปัญหาอะไรบ้างและแก้ยังไง
- Database, Cache, CDN scaling ในแต่ละ Phase
- Engineering Team ต้องโตยังไงพร้อมกับ System

---

## 📖 ทฤษฎีและแนวคิด

### 1. Twitter/Facebook Scale Architecture

Twitter ณ วันนี้รองรับ **500M tweets/day** และ Facebook รองรับ **3B+ users** พวกเขาไม่ได้เริ่มต้นด้วย Architecture ที่ซับซ้อนแบบนี้เลย Twitter เริ่มจาก **Monolith Ruby on Rails** และ Facebook เริ่มจาก **PHP ธรรมดา**

**สิ่งที่ทั้งสอง Platform เรียนรู้ร่วมกัน:**

1. **Horizontal Scaling** เป็น Default ไม่ใช่ Vertical
2. **Data locality** สำคัญมาก — อ่านจาก Cache ก่อนเสมอ
3. **Async ทุกอย่างที่ทำได้** — อย่า Block user รอ background job
4. **Graceful Degradation** — เมื่อ system บางส่วนล้ม service อื่นยังทำงานได้
5. **Observability first** — ต้องรู้ว่าเกิดอะไรขึ้นตลอดเวลา

### 2. Growth Phases ของ chuaikan.com

```
Phase 1: 0 → 10,000 users/day
Phase 2: 10,000 → 100,000 users/day  
Phase 3: 100,000 → 1,000,000 users/day
Phase 4: 1,000,000 → 10,000,000 users/day
```

แต่ละ Phase มีความท้าทายที่แตกต่างกันอย่างสิ้นเชิง

---

## 🏗️ Architecture แต่ละ Phase

### Phase 1: 0 → 10,000 Users/Day

**ปัญหาหลัก:** ยังไม่รู้ว่า Product จะได้รับความนิยมหรือเปล่า ต้องการ **Speed of Development** มากกว่า Scale

```
┌─────────────────────────────────────────────────────┐
│                    Phase 1 Architecture             │
│                                                     │
│  Users → Nginx → Node.js App → PostgreSQL           │
│                      ↓                              │
│                   Redis Cache                       │
│                      ↓                              │
│                   S3 / Cloudflare                   │
└─────────────────────────────────────────────────────┘
```

**Stack:**
- 1x VPS (4 CPU, 8GB RAM) — ประมาณ $40/เดือน
- Monolith Node.js application
- PostgreSQL บน server เดียวกัน
- Redis เดียว
- Cloudflare CDN (free tier)

**ข้อดี:** Deploy ง่าย Debug ง่าย Cost ต่ำ
**ข้อเสีย:** Single Point of Failure ทุกอย่าง

**Cost estimate:** ~$50/เดือน

### Phase 2: 10,000 → 100,000 Users/Day

**ปัญหาหลัก:** Database เริ่ม Bottleneck, Application เริ่มใช้ CPU เยอะ

```
┌──────────────────────────────────────────────────────────┐
│                   Phase 2 Architecture                   │
│                                                          │
│  Users → Cloudflare → Load Balancer (AWS ALB)           │
│                            ↓                            │
│              ┌─────────────┴─────────────┐              │
│          App Server 1               App Server 2         │
│              └─────────────┬─────────────┘              │
│                            ↓                            │
│                  PostgreSQL Primary                      │
│                       ↓   ↓                             │
│              Read Replica 1  Read Replica 2              │
│                            ↓                            │
│                       Redis Cluster                      │
│                            ↓                            │
│                     S3 + CloudFront                      │
└──────────────────────────────────────────────────────────┘
```

**การเปลี่ยนแปลงสำคัญ:**
- เพิ่ม Load Balancer (AWS ALB)
- Horizontal scale App servers เป็น 2-4 nodes
- PostgreSQL Read Replicas (อ่านจาก replica ทั้งหมด)
- Redis สำหรับ Session store และ Cache
- CDN สำหรับ static assets และ images

**Cost estimate:** ~$500-1,000/เดือน

### Phase 3: 100,000 → 1,000,000 Users/Day

**ปัญหาหลัก:** Monolith เริ่มหนัก deploy ช้า, Database Write เริ่มเป็น bottleneck

```
┌───────────────────────────────────────────────────────────────────┐
│                      Phase 3 Architecture                         │
│                                                                   │
│  Users → Cloudflare (WAF + CDN) → AWS ALB                        │
│                                      ↓                           │
│           ┌────────────────┬─────────┴────────┬────────────────┐ │
│        API Gateway      Feed Service       SOS Service         │ │
│           │                │                    │              │ │
│           └────────────────┴────────────────────┘              │ │
│                            ↓                                    │ │
│              PostgreSQL (Sharded by user_id)                   │ │
│              ┌──────────┬──────────┬──────────┐                │ │
│           Shard 0    Shard 1    Shard 2    Shard 3              │ │
│              └──────────┴──────────┴──────────┘                │ │
│                            ↓                                    │ │
│                    Redis Cluster (6 nodes)                      │ │
│                            ↓                                    │ │
│              S3 + CloudFront (Multi-region)                     │ │
└───────────────────────────────────────────────────────────────────┘
```

**การเปลี่ยนแปลงสำคัญ:**
- เริ่ม Microservices (Feed, SOS, User, Notification)
- Database Sharding
- Kubernetes บน EKS
- Kafka สำหรับ Event streaming
- Elasticsearch สำหรับ Search

**Cost estimate:** ~$5,000-15,000/เดือน

### Phase 4: 1,000,000 → 10,000,000 Users/Day

**ปัญหาหลัก:** Multi-region จำเป็น, Data consistency ซับซ้อนขึ้น

```
┌──────────────────────────────────────────────────────────────────┐
│                     Phase 4 Architecture                         │
│               (Multi-Region Active-Active)                       │
│                                                                  │
│  ┌─────────────────┐          ┌─────────────────┐               │
│  │   Asia Pacific  │          │   Asia Pacific  │               │
│  │   (ap-se-1)     │◄────────►│   (ap-ne-1)     │               │
│  │   Primary       │          │   Secondary     │               │
│  └─────────────────┘          └─────────────────┘               │
│           │                            │                         │
│           └─────────────┬──────────────┘                        │
│                         │                                        │
│              Global Route 53 (Latency-based)                    │
│                         │                                        │
│            Cloudflare Global Network (ANYCAST)                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## ⚙️ Database Scaling Journey

### Single Database → Read Replicas

```sql
-- Phase 1: ทุก query ไป Master
SELECT * FROM posts WHERE user_id = 123;

-- Phase 2: Read queries ไป Replica
-- ใน code ต้องระบุ connection pool
const readPool = new Pool({ host: 'replica.db.internal' });
const writePool = new Pool({ host: 'primary.db.internal' });

async function getPost(id) {
  // READ ไป replica
  return readPool.query('SELECT * FROM posts WHERE id = $1', [id]);
}

async function createPost(data) {
  // WRITE ไป primary
  return writePool.query('INSERT INTO posts ...', [data]);
}
```

### Read Replicas → Sharding

```javascript
// Sharding logic — ใช้ user_id เป็น shard key
function getShardId(userId) {
  return userId % 4; // 4 shards
}

function getDbConnection(userId) {
  const shardId = getShardId(userId);
  return dbConnections[shardId];
}

async function getUserPosts(userId) {
  const db = getDbConnection(userId);
  return db.query('SELECT * FROM posts WHERE user_id = $1', [userId]);
}
```

**ข้อระวัง Cross-Shard Queries:**

```sql
-- นี่คือ NIGHTMARE ใน sharded database
-- Query ข้าม shard = ต้อง query ทุก shard แล้ว merge ผล
SELECT p.*, u.username 
FROM posts p 
JOIN users u ON p.user_id = u.id
WHERE p.created_at > NOW() - INTERVAL '1 hour'
ORDER BY p.likes_count DESC;

-- วิธีแก้: Denormalization — เก็บ username ใน posts table
-- เพื่อหลีกเลี่ยง cross-shard join
```

---

## ⚙️ Cache Scaling Journey

### Single Redis → Redis Cluster

```yaml
# Phase 1: Single Redis
# docker-compose.yml
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

# Phase 2-3: Redis Cluster บน Kubernetes
# redis-cluster.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
spec:
  serviceName: redis-cluster
  replicas: 6  # 3 masters + 3 replicas
  selector:
    matchLabels:
      app: redis-cluster
  template:
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        command: ["redis-server"]
        args:
          - "--cluster-enabled yes"
          - "--cluster-config-file /data/nodes.conf"
          - "--cluster-node-timeout 5000"
          - "--appendonly yes"
        ports:
        - containerPort: 6379
        volumeMounts:
        - name: redis-data
          mountPath: /data
```

```bash
# สร้าง Redis Cluster
kubectl exec -it redis-cluster-0 -- redis-cli \
  --cluster create \
  $(kubectl get pods -l app=redis-cluster -o jsonpath='{range .items[*]}{.status.podIP}:6379 {end}') \
  --cluster-replicas 1 \
  --cluster-yes
```

### Redis Cluster → Redis + CDN Cache

ที่ scale นี้ edge caching ใน CDN กลายเป็นสิ่งสำคัญ:

```nginx
# nginx.conf — Cache บน Edge
location /api/feed {
  # Cache response 30 วินาทีบน CDN
  proxy_cache_valid 200 30s;
  proxy_cache_key "$scheme$request_method$host$request_uri$http_authorization";
  
  add_header X-Cache-Status $upstream_cache_status;
  proxy_pass http://app_servers;
}

location /api/sos/active {
  # SOS ไม่ cache — ต้อง real-time เสมอ
  proxy_cache off;
  proxy_pass http://sos_service;
}
```

---

## 🏢 Engineering Team Scaling

### Conway's Law

> "Organizations which design systems are constrained to produce designs which are copies of the communication structures of those organizations."

ความหมายคือ: **Architecture ของ System สะท้อน Structure ของ Team**

```
Phase 1: 1-5 คน (Full-stack everything)
├── ทุกคนรู้ทุกอย่าง
└── 1 codebase, 1 team

Phase 2: 5-15 คน (Frontend/Backend split)
├── Frontend Team
├── Backend Team  
└── DevOps/Infra (1-2 คน)

Phase 3: 15-50 คน (Feature teams)
├── Feed Team (Frontend + Backend + Data)
├── SOS Team (Frontend + Backend + ML)
├── User/Auth Team
├── Platform/Infra Team
└── Data/Analytics Team

Phase 4: 50-200 คน (Platform + Product teams)
├── Platform Engineering
│   ├── Infrastructure Team
│   ├── Developer Experience Team
│   └── Security Team
├── Product Teams (6-8 teams)
│   ├── Feed & Discovery
│   ├── SOS & Emergency Response
│   ├── Community & Social
│   ├── Maps & Location
│   └── Notifications
└── Enablement Teams
    ├── Data & ML Platform
    └── QA Platform
```

---

## 💰 Cost at Each Scale

| Phase | Users/Day | Monthly Cost | Cost/User/Month |
|-------|-----------|--------------|-----------------|
| 1 | 0-10K | $50-200 | $0.02-0.04 |
| 2 | 10K-100K | $500-2K | $0.005-0.02 |
| 3 | 100K-1M | $5K-20K | $0.005-0.02 |
| 4 | 1M-10M | $30K-100K | $0.003-0.01 |

**Cost Breakdown ที่ 1M Users/Day:**

```
Compute (EKS/EC2):     ~$8,000/month   (40%)
Database (RDS):        ~$4,000/month   (20%)
CDN (CloudFront):      ~$3,000/month   (15%)
Storage (S3):          ~$2,000/month   (10%)
Cache (ElastiCache):   ~$1,500/month   (7.5%)
Monitoring/Logging:    ~$1,000/month   (5%)
Networking/Other:      ~$500/month     (2.5%)
─────────────────────────────────────
Total:                 ~$20,000/month
```

---

## 🛠️ Step-by-Step Implementation

### Step 901: สร้าง Architecture Decision Record (ADR)

ADR คือ Document ที่บันทึกว่าทำไมถึงตัดสินใจเลือก Architecture นี้

```bash
mkdir -p /home/user/roadmap_1000000_course/docs/adr
cat > docs/adr/001-database-sharding.md << 'EOF'
# ADR-001: Database Sharding Strategy

## Status: Accepted

## Context
ณ 100,000 users/day PostgreSQL primary server ถึง limit
Write throughput: 5,000 writes/sec (above safe limit of 3,000/sec)
Read throughput: 50,000 reads/sec (handled by replicas)

## Decision
ใช้ Horizontal Sharding โดย shard key = user_id % 8

## Consequences
+ Scale write throughput เป็น 8x
+ แต่ละ shard เล็กลง = query เร็วขึ้น
- Cross-shard queries ต้องทำ application-level join
- Database migrations ซับซ้อนขึ้น
EOF
```

### Step 902: Load Testing ก่อน Scaling Decision

```bash
# ติดตั้ง k6
brew install k6  # macOS
# หรือ
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
  --keyserver hkp://keyserver.ubuntu.com:80 \
  --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] \
  https://dl.k6.io/deb stable main" | \
  sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update && sudo apt-get install k6
```

```javascript
// load-test-feed.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },   // ramp up
    { duration: '5m', target: 1000 },  // stay at 1000 users
    { duration: '2m', target: 5000 },  // ramp to 5000
    { duration: '5m', target: 5000 },  // stay at peak
    { duration: '2m', target: 0 },     // ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% ต้องเร็วกว่า 500ms
    http_req_failed: ['rate<0.01'],    // Error rate < 1%
  },
};

export default function () {
  const res = http.get('https://api.chuaikan.com/v1/feed', {
    headers: { Authorization: `Bearer ${__ENV.TEST_TOKEN}` },
  });
  
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time OK': (r) => r.timings.duration < 500,
  });
  
  sleep(1);
}
```

```bash
k6 run load-test-feed.js
```

### Step 903: Stateless Architecture

```javascript
// ❌ WRONG: Stateful — เก็บ session ใน memory
const sessions = new Map(); // พอ restart app ข้อมูลหาย!

app.post('/login', (req, res) => {
  const token = generateToken();
  sessions.set(token, { userId: user.id });
  res.json({ token });
});

// ✅ CORRECT: Stateless — เก็บ session ใน Redis
const redis = require('ioredis');
const client = new redis(process.env.REDIS_URL);

app.post('/login', async (req, res) => {
  const token = generateToken();
  await client.setex(
    `session:${token}`,
    86400, // 24 hours TTL
    JSON.stringify({ userId: user.id })
  );
  res.json({ token });
});

// ตอนนี้ app สามารถ run หลาย instances ได้
// เพราะ state อยู่ที่ Redis ไม่ใช่ memory
```

### Step 904: Health Check Endpoints

```javascript
// health.js — สำหรับ Load Balancer health check
const express = require('express');
const router = express.Router();

// Shallow health check — ตรวจแค่ว่า process ยังทำงานอยู่
router.get('/health', (req, res) => {
  res.status(200).json({ status: 'ok', timestamp: new Date().toISOString() });
});

// Deep health check — ตรวจ dependencies ทั้งหมด
router.get('/health/deep', async (req, res) => {
  const checks = await Promise.allSettled([
    checkDatabase(),
    checkRedis(),
    checkExternalAPIs(),
  ]);
  
  const results = {
    database: checks[0].status === 'fulfilled' ? 'ok' : 'error',
    redis: checks[1].status === 'fulfilled' ? 'ok' : 'error',
    external: checks[2].status === 'fulfilled' ? 'ok' : 'error',
  };
  
  const allOk = Object.values(results).every(v => v === 'ok');
  
  res.status(allOk ? 200 : 503).json({
    status: allOk ? 'healthy' : 'degraded',
    checks: results,
    timestamp: new Date().toISOString(),
  });
});

module.exports = router;
```

```yaml
# kubernetes deployment with health checks
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
spec:
  template:
    spec:
      containers:
      - name: api
        image: chuaikan/api:latest
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health/deep
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
```

---

## 🔧 Configuration Files

### Terraform สำหรับ Multi-Region Setup

```hcl
# main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Primary Region: ap-southeast-1 (Singapore)
provider "aws" {
  alias  = "primary"
  region = "ap-southeast-1"
}

# Secondary Region: ap-northeast-1 (Tokyo)
provider "aws" {
  alias     = "secondary"
  region    = "ap-northeast-1"
}

# VPC Primary Region
module "vpc_primary" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
  
  providers = { aws = aws.primary }
  
  name = "chuaikan-primary"
  cidr = "10.0.0.0/16"
  
  azs             = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway = true
  enable_vpn_gateway = false
  
  tags = {
    Environment = "production"
    Team        = "platform"
    Project     = "chuaikan"
  }
}
```

---

## 🧪 Testing

### Architecture Fitness Test

```bash
#!/bin/bash
# architecture-fitness-test.sh
# ทดสอบว่า Architecture ตรงกับที่ออกแบบไว้

echo "=== Architecture Fitness Tests ==="

# Test 1: App server ไม่มี state ใน local memory
echo "Testing stateless application..."
# restart one pod และตรวจว่า sessions ยังทำงาน
kubectl rollout restart deployment/api-service
sleep 30
curl -H "Authorization: Bearer $SESSION_TOKEN" \
  https://api.chuaikan.com/v1/me | jq '.id'

# Test 2: Database read queries ไป replica ไม่ใช่ primary
echo "Testing read replica routing..."
REPLICA_QUERIES=$(kubectl exec -it postgres-replica-0 -- \
  psql -c "SELECT count(*) FROM pg_stat_activity WHERE state='active'" -t)
echo "Active queries on replica: $REPLICA_QUERIES"

# Test 3: Cache hit rate > 80%
echo "Testing cache hit rate..."
CACHE_HITS=$(redis-cli INFO stats | grep keyspace_hits | cut -d: -f2)
CACHE_MISSES=$(redis-cli INFO stats | grep keyspace_misses | cut -d: -f2)
HIT_RATE=$(echo "scale=2; $CACHE_HITS / ($CACHE_HITS + $CACHE_MISSES) * 100" | bc)
echo "Cache hit rate: $HIT_RATE%"

if (( $(echo "$HIT_RATE < 80" | bc -l) )); then
  echo "❌ Cache hit rate too low: $HIT_RATE%"
  exit 1
else
  echo "✅ Cache hit rate OK: $HIT_RATE%"
fi
```

---

## ❌ Common Errors & Solutions

### Error 1: "Too many connections" บน PostgreSQL

```
Error: remaining connection slots are reserved for non-replication superuser connections
```

**สาเหตุ:** App servers เปิด connection pool ใหญ่เกินไป

```javascript
// ❌ ปัญหา: ทุก app instance เปิด pool ใหญ่
const pool = new Pool({
  max: 100,  // 10 app instances × 100 = 1000 connections!
});

// ✅ แก้: ลด pool size + ใช้ PgBouncer
const pool = new Pool({
  max: 10,   // 10 app instances × 10 = 100 connections
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});
```

```bash
# ติดตั้ง PgBouncer (Connection Pooler)
helm install pgbouncer helm/pgbouncer \
  --set config.databases.chuaikan.host=postgres-primary \
  --set config.databases.chuaikan.pool_size=50 \
  --set config.pgbouncer.pool_mode=transaction
```

### Error 2: Cache Stampede

**สถานการณ์:** Cache หมดอายุพร้อมกัน → requests ทั้งหมดยิง DB พร้อมกัน

```javascript
// ❌ ปัญหา: ถ้า cache miss ทุก request ไป DB พร้อมกัน
async function getFeed(userId) {
  const cached = await redis.get(`feed:${userId}`);
  if (cached) return JSON.parse(cached);
  
  // ถ้า cache miss: 1000 requests อาจ query DB พร้อมกัน!
  const feed = await db.query('SELECT ...');
  await redis.setex(`feed:${userId}`, 60, JSON.stringify(feed));
  return feed;
}

// ✅ แก้: ใช้ mutex lock (Redlock)
const Redlock = require('redlock');
const redlock = new Redlock([redis]);

async function getFeed(userId) {
  const cached = await redis.get(`feed:${userId}`);
  if (cached) return JSON.parse(cached);
  
  // ได้ lock แค่ 1 request เท่านั้น
  const lock = await redlock.acquire([`lock:feed:${userId}`], 5000);
  
  try {
    // Double-check หลัง lock (อีก request อาจ set cache ไปแล้ว)
    const cachedAfterLock = await redis.get(`feed:${userId}`);
    if (cachedAfterLock) return JSON.parse(cachedAfterLock);
    
    const feed = await db.query('SELECT ...');
    await redis.setex(`feed:${userId}`, 60, JSON.stringify(feed));
    return feed;
  } finally {
    await lock.release();
  }
}
```

---

## ✅ Checklist

### Phase 1 → Phase 2 Readiness

- [ ] App server เป็น Stateless แล้ว (session ใน Redis)
- [ ] Health check endpoints (/health, /health/deep) พร้อมแล้ว
- [ ] PostgreSQL Read Replica ตั้งค่าแล้ว
- [ ] Read/Write separation ใน application code แล้ว
- [ ] Redis cache ทำงานอยู่ (cache hit rate > 70%)
- [ ] Load Balancer (ALB) ตั้งค่าแล้ว
- [ ] Load test ผ่านที่ 2x current traffic

### Phase 2 → Phase 3 Readiness

- [ ] Database sharding plan พร้อม (shard key เลือกแล้ว)
- [ ] Kubernetes cluster ตั้งค่าแล้ว
- [ ] Services แรกที่จะ extract ออกจาก monolith ตัดสินใจแล้ว
- [ ] Kafka cluster สำหรับ event streaming ตั้งค่าแล้ว
- [ ] Monitoring/Alerting ครอบคลุมทุก service แล้ว
- [ ] Team structure ปรับแล้วให้ตรงกับ Architecture

### Phase 3 → Phase 4 Readiness

- [ ] Multi-region architecture ออกแบบแล้ว
- [ ] Data replication strategy ระหว่าง regions ชัดเจนแล้ว
- [ ] Disaster recovery plan พร้อมและ test แล้ว
- [ ] Cost optimization review ทำแล้ว (ต้อง < $0.01/user/month)
- [ ] Engineering team พร้อม scale (hiring plan มีแล้ว)

---

## 🔗 References

- [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)
- [Google SRE Book — Chapter 17: Testing for Reliability](https://sre.google/sre-book/testing-reliability/)
- [Martin Fowler — Patterns of Distributed Systems](https://martinfowler.com/articles/patterns-of-distributed-systems/)
- [Designing Data-Intensive Applications — Martin Kleppmann](https://dataintensive.net/)
- [Twitter Engineering Blog](https://blog.twitter.com/engineering)
- [Facebook Engineering Blog](https://engineering.fb.com/)

---

*Part 091 | Road to 1,000,000 Users/Day | chuaikan.com*
