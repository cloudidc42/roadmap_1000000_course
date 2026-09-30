# Part 105: Advanced Topics & Future Roadmap
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** World Class
> **Steps:** 1041-1050
> **เวลาโดยประมาณ:** 12 ชั่วโมง
> **Prerequisites:** Part 100-104 (ทุก World Class parts), Part 006 (Docker), Part 007 (Cloudflare)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- WebAssembly (WASM) สำหรับ compute-intensive frontend tasks
- Cloudflare Workers AI: LLM at the edge
- pgvector สำหรับ semantic search
- GraphQL Federation ด้วย Apollo Router
- Event-driven architecture maturity model
- DORA metrics และ engineering culture
- Architecture diagram สมบูรณ์: chuaikan.com ที่ 1M users/day
- Preview: Road to 10M users/day

---

## 📖 ทฤษฎีและแนวคิด

### Final Architecture Overview

```
                    chuaikan.com at 1M Users/Day
                    
Users (Thailand)
    ↓ HTTPS
Cloudflare (CDN + WAF + Workers AI)
    ↓
Nginx Load Balancer (multiple)
    ↓
Next.js App Servers (Node.js 22, auto-scaled)
    ↓ ↓ ↓ ↓
    PostgreSQL 17 (primary + replicas)
    Redis 7 (cluster mode)
    Kafka 3.7 (KRaft, 3-broker cluster)
    ClickHouse (analytics)
    ↓
Background Workers (BullMQ)
    - Feed Algorithm
    - Push Notifications (FCM)
    - NLP Processing (Python FastAPI)
    ↓
External Services
    - Cloudflare R2 (media storage)
    - Cloudflare Stream (video)
    - Firebase Cloud Messaging (push)
```

---

## ⚙️ Environment Setup

### Step 1041: WASM Setup

```bash
# ติดตั้ง @ffmpeg/ffmpeg สำหรับ client-side processing
npm install @ffmpeg/ffmpeg@0.12.x @ffmpeg/util@0.12.x
```

---

## 🛠️ Step-by-Step Implementation

### Step 1041: WebAssembly - Image Compression ใน Browser

```typescript
// src/lib/wasm/image-compressor.ts
'use client';

import { FFmpeg } from '@ffmpeg/ffmpeg';
import { fetchFile, toBlobURL } from '@ffmpeg/util';

let ffmpeg: FFmpeg | null = null;
let isLoaded = false;

async function loadFFmpeg(): Promise<FFmpeg> {
  if (ffmpeg && isLoaded) return ffmpeg;

  ffmpeg = new FFmpeg();

  // Load WASM จาก CDN
  const baseURL = 'https://unpkg.com/@ffmpeg/core@0.12.x/dist/umd';
  await ffmpeg.load({
    coreURL: await toBlobURL(`${baseURL}/ffmpeg-core.js`, 'text/javascript'),
    wasmURL: await toBlobURL(`${baseURL}/ffmpeg-core.wasm`, 'application/wasm'),
  });

  isLoaded = true;
  return ffmpeg;
}

export async function compressImage(file: File, maxSizeKB = 500): Promise<Blob> {
  const ff = await loadFFmpeg();

  const inputName = `input.${file.name.split('.').pop()}`;
  const outputName = 'output.jpg';

  // เขียนไฟล์เข้า WASM filesystem
  await ff.writeFile(inputName, await fetchFile(file));

  // Compress image ใน browser ไม่ต้องส่งไป server
  await ff.exec([
    '-i', inputName,
    '-vf', 'scale=iw*min(1\\,1920/iw):-1',  // max 1920px width
    '-q:v', '85',                              // JPEG quality
    '-f', 'image2',
    outputName,
  ]);

  const data = await ff.readFile(outputName);
  const blob = new Blob([data], { type: 'image/jpeg' });

  // ถ้ายังใหญ่เกินไป compress ต่อ
  if (blob.size > maxSizeKB * 1024) {
    await ff.exec(['-i', outputName, '-q:v', '70', 'output2.jpg']);
    const data2 = await ff.readFile('output2.jpg');
    return new Blob([data2], { type: 'image/jpeg' });
  }

  return blob;
}

// Video thumbnail generation ใน browser
export async function generateVideoThumbnail(videoFile: File): Promise<Blob> {
  const ff = await loadFFmpeg();

  await ff.writeFile('video.mp4', await fetchFile(videoFile));

  // Extract frame ที่ 1 วินาที
  await ff.exec([
    '-i', 'video.mp4',
    '-ss', '00:00:01.000',
    '-vframes', '1',
    '-vf', 'scale=640:-1',
    'thumbnail.jpg',
  ]);

  const data = await ff.readFile('thumbnail.jpg');
  return new Blob([data], { type: 'image/jpeg' });
}
```

### Step 1042: Cloudflare Workers AI

```javascript
// cloudflare-workers/content-moderation.js
// Deploy: wrangler deploy

export default {
  async fetch(request, env, ctx) {
    if (request.method !== 'POST') {
      return new Response('Method not allowed', { status: 405 });
    }

    const { text, context } = await request.json();

    // ใช้ Llama 3.1 สำหรับ content moderation ที่ edge
    const ai = new Ai(env.AI);

    const result = await ai.run('@cf/meta/llama-3.1-8b-instruct', {
      messages: [
        {
          role: 'system',
          content: `You are a content moderation AI for a Thai emergency response platform.
Analyze the following content and respond in JSON format only:
{"is_sos": boolean, "is_spam": boolean, "severity": "low|medium|high|critical", "category": "flood|fire|accident|other|normal", "confidence": 0.0-1.0}`,
        },
        {
          role: 'user',
          content: `Content to analyze: "${text}"`,
        },
      ],
      max_tokens: 100,
      temperature: 0.1,
    });

    let parsed;
    try {
      parsed = JSON.parse(result.response);
    } catch {
      parsed = { is_sos: false, is_spam: false, severity: 'low', category: 'normal', confidence: 0.5 };
    }

    return new Response(JSON.stringify(parsed), {
      headers: { 'Content-Type': 'application/json' },
    });
  },
};
```

```typescript
// src/lib/cloudflare/workers-ai-client.ts
export async function moderateContent(text: string): Promise<{
  is_sos: boolean;
  is_spam: boolean;
  severity: string;
  category: string;
  confidence: number;
}> {
  const response = await fetch(
    `https://content-moderation.chuaikan.workers.dev`,
    {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ text }),
    }
  );

  return response.json();
}
```

### Step 1043: pgvector สำหรับ Semantic Search

```sql
-- Enable pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- เพิ่ม embedding column ใน posts table
ALTER TABLE posts ADD COLUMN embedding vector(384);

-- สร้าง index สำหรับ fast similarity search
CREATE INDEX ON posts USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 100);

-- HNSW index (เร็วกว่า IVFFlat แต่ใช้ memory มากกว่า)
CREATE INDEX ON posts USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 64);
```

```typescript
// src/lib/search/vector-search.ts
import { db } from '../db/postgres';
import { getTextEmbedding } from '../nlp/thai-nlp-client';

export async function findSimilarSOSPosts(query: string, limit = 10): Promise<any[]> {
  // แปลง query เป็น embedding
  const embedding = await getTextEmbedding(query);

  // Vector similarity search
  const result = await db.query(
    `SELECT
        p.id, p.content, p.created_at,
        u.username,
        p.sos_category, p.sos_severity,
        1 - (p.embedding <=> $1::vector) as similarity
     FROM posts p
     JOIN users u ON u.id = p.user_id
     WHERE p.is_sos = true
       AND p.embedding IS NOT NULL
       AND 1 - (p.embedding <=> $1::vector) > 0.7  -- threshold
     ORDER BY p.embedding <=> $1::vector
     LIMIT $2`,
    [JSON.stringify(embedding), limit]
  );

  return result.rows;
}

// ตัวอย่าง: หา SOS posts คล้ายกัน
// const similar = await findSimilarSOSPosts("น้ำท่วมขัง ถนนปิด")
// จะหา posts ที่พูดถึง floods, flooded roads, etc.
```

### Step 1044: GraphQL Federation ด้วย Apollo Router

```yaml
# apollo-router/router.yaml
supergraph:
  listen: 0.0.0.0:4000
  path: /graphql

# Subgraphs (microservices)
subgraphs:
  users:
    routing_url: http://localhost:4001/graphql
    schema:
      file: ./schemas/users.graphql
  posts:
    routing_url: http://localhost:4002/graphql
    schema:
      file: ./schemas/posts.graphql
  sos:
    routing_url: http://localhost:4003/graphql
    schema:
      file: ./schemas/sos.graphql
  notifications:
    routing_url: http://localhost:4004/graphql
    schema:
      file: ./schemas/notifications.graphql

# Performance
traffic_shaping:
  all:
    deduplicate_query: true

# Caching
apq:
  enabled: true
  storage:
    in_memory:
      limit: 512
```

```typescript
// src/graphql/subgraphs/sos/schema.ts
import { buildSubgraphSchema } from '@apollo/subgraph';
import { gql } from 'graphql-tag';

const typeDefs = gql`
  extend schema @link(url: "https://specs.apollo.dev/federation/v2.0", import: ["@key"])

  type SOSPost @key(fields: "id") {
    id: ID!
    content: String!
    alertType: String!
    severity: String!
    location: Location!
    status: String!
    responders: [User!]!
    createdAt: String!
  }

  type Location {
    lat: Float!
    lng: Float!
    province: String!
    address: String
  }

  type Query {
    activeSOS(province: String, severity: String): [SOSPost!]!
    sosById(id: ID!): SOSPost
  }

  type Mutation {
    createSOS(input: CreateSOSInput!): SOSPost!
    updateSOSStatus(id: ID!, status: String!): SOSPost!
  }

  input CreateSOSInput {
    content: String!
    alertType: String!
    location: LocationInput!
  }

  input LocationInput {
    lat: Float!
    lng: Float!
    province: String!
  }
`;
```

### Step 1045: Event-Driven Architecture Maturity Model

ระดับความสมบูรณ์ของ Event-Driven Architecture:

| ระดับ | ชื่อ | คำอธิบาย | chuaikan.com Status |
|-------|------|-----------|---------------------|
| 1 | **Basic Messaging** | ใช้ message queue อย่างง่าย | ✅ ผ่านแล้ว (Redis Queue) |
| 2 | **Event Streaming** | Kafka, topics, consumer groups | ✅ ผ่านแล้ว (Part 072) |
| 3 | **Event Sourcing** | เก็บ events เป็น source of truth | 🔄 กำลังทำ |
| 4 | **CQRS** | แยก read/write models | 🔄 กำลังทำ |
| 5 | **Reactive Systems** | ทุก service reactive, resilient | 🎯 เป้าหมาย |

### Step 1046: DORA Metrics

```typescript
// src/lib/metrics/dora-metrics.ts
// DORA = DevOps Research and Assessment

export const DORA_TARGETS = {
  // Deployment Frequency: "multiple per day" = Elite
  deploymentFrequency: 'multiple_per_day',

  // Lead Time: "< 1 hour" = Elite
  leadTimeForChanges: 60, // minutes

  // Change Failure Rate: "< 5%" = Elite
  changeFailureRate: 5, // percent

  // MTTR: "< 30 minutes" = Elite
  meanTimeToRestore: 30, // minutes
};

// Track deployment ด้วย GitHub Actions
export async function recordDeployment(commitSha: string, environment: string) {
  const now = new Date();

  await db.query(
    `INSERT INTO deployments (commit_sha, environment, deployed_at, deployed_by)
     VALUES ($1, $2, $3, $4)`,
    [commitSha, environment, now, process.env.DEPLOY_USER]
  );

  // Calculate lead time (commit time → deploy time)
  const commitTime = await getCommitTime(commitSha);
  const leadTimeMinutes = (now.getTime() - commitTime.getTime()) / 60000;

  console.log(`[DORA] Deployment: ${commitSha} | Lead time: ${leadTimeMinutes.toFixed(0)} min`);
}

async function getCommitTime(sha: string): Promise<Date> {
  // ดึงจาก GitHub API
  const response = await fetch(
    `https://api.github.com/repos/chuaikan/app/commits/${sha}`,
    { headers: { Authorization: `token ${process.env.GITHUB_TOKEN}` } }
  );
  const data = await response.json();
  return new Date(data.commit.author.date);
}
```

### Step 1047: Blameless Postmortem Template

```markdown
# Incident Postmortem: [Incident Title]

**Date:** YYYY-MM-DD
**Severity:** P1/P2/P3
**Duration:** X hours Y minutes
**Impact:** X% of users affected
**Prepared by:** [Name]

## Timeline (UTC+7)
| Time | Event |
|------|-------|
| HH:MM | Incident started |
| HH:MM | Alert triggered |
| HH:MM | Engineer paged |
| HH:MM | Mitigation applied |
| HH:MM | Incident resolved |

## Root Cause
[ไม่โทษคน แต่โทษระบบ/process]

## What Went Well
- การ detect ปัญหาเร็ว
- Team communication ดี

## What Went Wrong
- [ระบุปัญหาเชิงระบบ]

## Action Items
| Action | Owner | Due Date | Status |
|--------|-------|----------|--------|
| Add circuit breaker to X | Engineer A | 2024-XX-XX | TODO |
| Improve alerting for Y | Engineer B | 2024-XX-XX | TODO |

## 5 Whys
1. Why? Service crashed → Because memory OOM
2. Why OOM? → Because cache not configured with TTL
3. Why no TTL? → Because code review missed it
4. Why missed? → Because no automated check
5. Root: ขาด automated memory leak detection
```

### Step 1048: Open Source Contributions

```bash
# Contributing to PyThaiNLP
git clone https://github.com/PyThaiNLP/pythainlp
cd pythainlp

# เพิ่ม Thai SOS vocabulary
# pythainlp/corpus/words/thai-sos-vocab.txt
echo "น้ำท่วม
ไฟไหม้
อุบัติเหตุ
ดินถล่ม
วาตภัย
แผ่นดินไหว
ขอความช่วยเหลือ
ฉุกเฉิน" >> pythainlp/corpus/words/thai-sos-vocab.txt

# Run tests
python -m pytest tests/

# Submit PR พร้อม description เป็นภาษาอังกฤษ
```

### Step 1049: Final Architecture Diagram (ASCII)

```
╔══════════════════════════════════════════════════════════════════════╗
║           chuaikan.com — 1,000,000 Users/Day Architecture            ║
╚══════════════════════════════════════════════════════════════════════╝

[Users/Devices] ──HTTPS──▶ [Cloudflare]
                                │
               ┌────────────────┼────────────────┐
               ▼                ▼                ▼
        [CDN Edge]    [Workers AI]     [DDoS/WAF]
               │                │
               └────────┬───────┘
                        ▼
              [Nginx Load Balancer]
                        │
         ┌──────────────┼──────────────┐
         ▼              ▼              ▼
    [Next.js 1]    [Next.js 2]    [Next.js N]
    Node.js 22     Node.js 22     Node.js 22
         │              │              │
         └──────────────┼──────────────┘
                        │
    ┌───────────────────┼───────────────────┐
    ▼                   ▼                   ▼
[PostgreSQL 17]    [Redis 7]         [Kafka 3.7]
Primary+Replicas   Cluster Mode      KRaft 3-broker
    │                   │                   │
    │              [BullMQ Workers]          │
    │              Feed/Push/NLP             │
    │                                        │
    └──────────────────▼────────────────────┘
                [ClickHouse]
                Analytics DB
                        │
                [Grafana]
                Dashboard + Alerts
                        │
                [Line/Discord]
                Alerts

EXTERNAL SERVICES:
  Cloudflare R2 ──── Media Storage (images, videos)
  Cloudflare Stream ─ Live Video Delivery  
  Firebase FCM ────── Push Notifications (10M devices)
  Python FastAPI ───── Thai NLP Service (port 8001)
  mediamtx ──────────── RTMP/HLS Live Streaming
```

### Step 1050: Preview — Road to 10M Users/Day

```typescript
// สิ่งที่ต้องทำเพิ่มเพื่อ scale จาก 1M → 10M users/day

const roadTo10M = {
  database: {
    current: 'PostgreSQL primary + 2 replicas',
    next: 'CockroachDB / Vitess (horizontal sharding)',
    when: '> 2M concurrent users',
  },

  cache: {
    current: 'Redis Cluster (3 nodes)',
    next: 'Redis Cluster (10+ nodes) + Regional caches',
    when: '> 5M cache ops/sec',
  },

  kafka: {
    current: '3-broker KRaft cluster',
    next: '10+ broker cluster, cross-region replication',
    when: '> 10M events/day',
  },

  infrastructure: {
    current: 'Single datacenter (TH)',
    next: 'Multi-region (TH + SG + HK), Kubernetes',
    when: 'International expansion',
  },

  ai: {
    current: 'Claude API + Workers AI',
    next: 'Fine-tuned Thai LLM on-premise',
    when: 'Cost optimization needed',
  },

  team: {
    current: '5-10 engineers',
    next: '20-50 engineers, platform team',
    when: '> 100 services',
  },
};

console.log('Next milestone: 10M Users/Day');
console.log('Key investments:', Object.keys(roadTo10M));
```

---

## 🔧 Configuration Files

### Wrangler.toml สำหรับ Cloudflare Workers

```toml
# cloudflare-workers/wrangler.toml
name = "chuaikan-content-moderation"
main = "src/index.js"
compatibility_date = "2024-09-01"
account_id = "your_cloudflare_account_id"

[ai]
binding = "AI"

[[routes]]
pattern = "content-moderation.chuaikan.workers.dev"
zone_name = "chuaikan.workers.dev"
```

### pgvector Index สำหรับ Production

```sql
-- สร้าง HNSW index (ดีกว่า IVFFlat สำหรับ recall)
CREATE INDEX CONCURRENTLY posts_embedding_hnsw_idx
ON posts USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 200);

-- ตรวจสอบ index
SELECT schemaname, tablename, indexname, indexdef
FROM pg_indexes
WHERE tablename = 'posts' AND indexname LIKE '%embedding%';

-- Query performance
EXPLAIN ANALYZE
SELECT id, 1 - (embedding <=> '[0.1, 0.2, ...]'::vector) as similarity
FROM posts
ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector
LIMIT 10;
```

---

## 🧪 Testing

### Test WASM Image Compression

```typescript
// tests/wasm/image-compression.test.ts
import { compressImage } from '../../src/lib/wasm/image-compressor';

test('should compress large image', async () => {
  // สร้าง test image
  const response = await fetch('https://picsum.photos/3840/2160');
  const blob = await response.blob();
  const file = new File([blob], 'test.jpg', { type: 'image/jpeg' });

  console.log(`Original size: ${(file.size / 1024).toFixed(0)}KB`);

  const compressed = await compressImage(file, 500);

  console.log(`Compressed size: ${(compressed.size / 1024).toFixed(0)}KB`);
  expect(compressed.size).toBeLessThan(500 * 1024);
}, 30000); // 30s timeout สำหรับ WASM loading
```

---

## ❌ Common Errors & Solutions

### Error 1: WASM Loading ช้า (> 5 วินาที)

**แก้ไข:** Preload WASM ตอน app startup
```typescript
// ใน layout.tsx
useEffect(() => {
  loadFFmpeg(); // load in background
}, []);
```

### Error 2: pgvector Index Build ช้า

**แก้ไข:**
```sql
-- Build index ด้วย more workers
SET max_parallel_maintenance_workers = 4;
CREATE INDEX CONCURRENTLY ...;
```

### Error 3: Apollo Router GraphQL N+1

**แก้ไข:** ใช้ DataLoader ใน resolvers
```typescript
import DataLoader from 'dataloader';

const userLoader = new DataLoader(async (ids: string[]) => {
  const users = await db.query('SELECT * FROM users WHERE id = ANY($1)', [ids]);
  return ids.map((id) => users.rows.find((u: any) => u.id === id));
});
```

---

## ✅ Checklist

- [ ] **Step 1041:** WASM image compression ทำงานใน browser, < 500KB output
- [ ] **Step 1042:** Cloudflare Workers AI deploy สำเร็จ, moderation ทำงาน
- [ ] **Step 1043:** pgvector extension enable, HNSW index สร้างแล้ว
- [ ] **Step 1044:** Apollo Router รัน GraphQL Federation ได้
- [ ] **Step 1045:** Event-driven maturity model ประเมินตำแหน่งปัจจุบัน
- [ ] **Step 1046:** DORA metrics tracking ทำงาน (deployment frequency, lead time)
- [ ] **Step 1047:** Postmortem template ใช้หลังทุก P1/P2 incident
- [ ] **Step 1048:** Open source contribution plan เตรียมไว้
- [ ] **Step 1049:** Architecture diagram อัพเดทสมบูรณ์
- [ ] **Step 1050:** Road to 10M users/day technical plan เสร็จ

---

## 🔗 References

- [WebAssembly.org](https://webassembly.org/)
- [FFmpeg.wasm](https://ffmpegwasm.netlify.app/)
- [Cloudflare Workers AI](https://developers.cloudflare.com/workers-ai/)
- [pgvector GitHub](https://github.com/pgvector/pgvector)
- [Apollo Federation](https://www.apollographql.com/docs/federation/)
- [DORA Metrics](https://dora.dev/guides/dora-metrics-four-keys/)
- [Google SRE Book — Postmortems](https://sre.google/sre-book/postmortem-culture/)

---
*Part 105 | Road to 1,000,000 Users/Day | chuaikan.com*
