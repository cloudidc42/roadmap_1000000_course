# Part 048: Database per Service Pattern
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 471–480
> **เวลาโดยประมาณ:** 4 ชั่วโมง
> **Prerequisites:** Part 047 (Event Sourcing), Microservices concepts

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ทำไมต้อง Database per Service (loose coupling, independent scaling)
- แบ่ง chuaikan.com database: users DB, content DB, sos DB, analytics DB
- จัดการ data consistency ข้าม services (eventual consistency)
- API composition สำหรับ cross-service queries
- Distributed query ด้วย GraphQL Federation
- Data duplication strategy (denormalization)
- Database migration ข้าม services แบบ zero-downtime

---

## 📖 ทฤษฎีและแนวคิด

### Step 471 — ทำไมต้อง Database per Service?

```
Monolith DB (ปัญหา):
┌─────────────────────────────────────────────────┐
│  Single PostgreSQL                               │
│  users, posts, sos_reports, sensor_readings,    │
│  analytics, notifications, ... (50+ tables)     │
└─────────────────────────────────────────────────┘
ปัญหา:
- Schema changes ต้อง coordinate ทุก service
- Scale ทั้งหมดเมื่อแค่ SOS service traffic พุ่ง
- Technology lock-in (ทุกอย่างต้องเป็น PostgreSQL)
- One big deployment risk

Database per Service (ดีกว่า):
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Users DB     │  │ Content DB   │  │ SOS DB       │  │ Analytics DB │
│ PostgreSQL   │  │ PostgreSQL   │  │ PostgreSQL   │  │ TimescaleDB  │
│ users        │  │ posts        │  │ sos_reports  │  │ page_views   │
│ auth_tokens  │  │ comments     │  │ sensor_data  │  │ user_events  │
│ user_prefs   │  │ media        │  │ flood_zones  │  │ metrics      │
└──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
ข้อดี:
- Scale แต่ละ DB อิสระ
- Technology ต่างกันได้
- Team ต่างกันดูแลแยกกัน
- Schema changes ไม่กระทบกัน
```

---

## ⚙️ Environment Setup

```bash
# สร้าง 4 PostgreSQL instances ด้วย Docker Compose
cat > docker-compose-microdb.yml << 'EOF'
version: '3.8'

services:
  users-db:
    image: postgres:16
    container_name: chuaikan-users-db
    environment:
      POSTGRES_DB: chuaikan_users
      POSTGRES_PASSWORD: users_secret_2024
    ports:
      - "5432:5432"
    volumes:
      - users_db_data:/var/lib/postgresql/data

  content-db:
    image: postgres:16
    container_name: chuaikan-content-db
    environment:
      POSTGRES_DB: chuaikan_content
      POSTGRES_PASSWORD: content_secret_2024
    ports:
      - "5433:5432"
    volumes:
      - content_db_data:/var/lib/postgresql/data

  sos-db:
    image: postgis/postgis:16-3.4
    container_name: chuaikan-sos-db
    environment:
      POSTGRES_DB: chuaikan_sos
      POSTGRES_PASSWORD: sos_secret_2024
    ports:
      - "5434:5432"
    volumes:
      - sos_db_data:/var/lib/postgresql/data

  analytics-db:
    image: timescale/timescaledb:latest-pg16
    container_name: chuaikan-analytics-db
    environment:
      POSTGRES_DB: chuaikan_analytics
      POSTGRES_PASSWORD: analytics_secret_2024
    ports:
      - "5435:5432"
    volumes:
      - analytics_db_data:/var/lib/postgresql/data

volumes:
  users_db_data:
  content_db_data:
  sos_db_data:
  analytics_db_data:
EOF

docker-compose -f docker-compose-microdb.yml up -d
```

---

## 🛠️ Step-by-Step Implementation

### Step 472 — Schema Design ของแต่ละ Database

```sql
-- === USERS DB (Port 5432) ===
-- docker exec chuaikan-users-db psql -U postgres -d chuaikan_users

CREATE TABLE users (
    id              BIGSERIAL PRIMARY KEY,
    username        VARCHAR(50)  UNIQUE NOT NULL,
    email           VARCHAR(255) UNIQUE NOT NULL,
    password_hash   VARCHAR(255) NOT NULL,
    display_name    VARCHAR(100),
    avatar_url      VARCHAR(500),
    bio             TEXT,
    is_active       BOOLEAN DEFAULT true,
    is_verified     BOOLEAN DEFAULT false,
    province_id     INTEGER,
    flood_alerts    BOOLEAN DEFAULT true,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE auth_tokens (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     BIGINT REFERENCES users(id) ON DELETE CASCADE,
    token_hash  VARCHAR(255) NOT NULL,
    device_info JSONB,
    expires_at  TIMESTAMPTZ NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX ON auth_tokens(token_hash);
CREATE INDEX ON auth_tokens(user_id, expires_at);
```

```sql
-- === CONTENT DB (Port 5433) ===
-- docker exec chuaikan-content-db psql -U postgres -d chuaikan_content

-- NOTE: ไม่มี FK ไปยัง users! ใช้ user_id เป็น reference เท่านั้น
CREATE TABLE posts (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT NOT NULL,        -- reference, no FK
    username    VARCHAR(50) NOT NULL,   -- denormalized!
    avatar_url  VARCHAR(500),           -- denormalized!
    title       VARCHAR(500),
    content     TEXT,
    tags        JSONB DEFAULT '[]',
    media_urls  JSONB DEFAULT '[]',
    like_count  INTEGER DEFAULT 0,
    view_count  INTEGER DEFAULT 0,
    is_published BOOLEAN DEFAULT true,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX ON posts(user_id, created_at DESC);
CREATE INDEX ON posts USING GIN(tags);

-- Denormalized user snapshot (สำหรับ display)
-- ไม่ต้อง query Users DB ทุกครั้ง
CREATE TABLE user_snapshots (
    user_id     BIGINT PRIMARY KEY,
    username    VARCHAR(50) NOT NULL,
    avatar_url  VARCHAR(500),
    display_name VARCHAR(100),
    synced_at   TIMESTAMPTZ DEFAULT NOW()
);
```

```sql
-- === SOS DB (Port 5434) ===
-- docker exec chuaikan-sos-db psql -U postgres -d chuaikan_sos

CREATE EXTENSION IF NOT EXISTS postgis;

CREATE TABLE sos_reports (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     BIGINT NOT NULL,         -- reference to users DB
    username    VARCHAR(50),             -- denormalized
    description TEXT,
    severity    VARCHAR(20),
    status      VARCHAR(30) DEFAULT 'pending',
    location    GEOGRAPHY(POINT, 4326),
    province_id INTEGER,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX ON sos_reports USING GIST(location);
CREATE INDEX ON sos_reports(status, created_at DESC);
```

```sql
-- === ANALYTICS DB (Port 5435) ===
-- docker exec chuaikan-analytics-db psql -U postgres -d chuaikan_analytics

CREATE EXTENSION IF NOT EXISTS timescaledb;

CREATE TABLE page_views (
    time        TIMESTAMPTZ NOT NULL,
    user_id     BIGINT,                  -- nullable (anonymous)
    session_id  VARCHAR(100),
    path        VARCHAR(500),
    referrer    VARCHAR(500),
    duration_ms INTEGER,
    device      VARCHAR(50),
    province_id INTEGER
);

SELECT create_hypertable('page_views', 'time',
    chunk_time_interval => INTERVAL '1 day');

CREATE TABLE user_events (
    time        TIMESTAMPTZ NOT NULL,
    user_id     BIGINT NOT NULL,
    event_type  VARCHAR(100),
    properties  JSONB DEFAULT '{}'
);

SELECT create_hypertable('user_events', 'time',
    chunk_time_interval => INTERVAL '1 day');
```

### Step 473 — Database Connection per Service

```typescript
// lib/db/databases.ts
import { PrismaClient } from '@prisma/client';

// แต่ละ service ใช้ Prisma client ของตัวเอง
// (หรือใช้ Knex / pg ก็ได้)

// Users Service DB
export const usersDb = new PrismaClient({
  datasources: {
    db: { url: process.env.USERS_DATABASE_URL },
  },
});

// Content Service DB
export const contentDb = new PrismaClient({
  datasources: {
    db: { url: process.env.CONTENT_DATABASE_URL },
  },
});

// SOS Service DB
export const sosDb = new PrismaClient({
  datasources: {
    db: { url: process.env.SOS_DATABASE_URL },
  },
});

// Analytics Service DB
export const analyticsDb = new PrismaClient({
  datasources: {
    db: { url: process.env.ANALYTICS_DATABASE_URL },
  },
});
```

```env
# .env
USERS_DATABASE_URL="postgresql://postgres:users_secret_2024@localhost:5432/chuaikan_users"
CONTENT_DATABASE_URL="postgresql://postgres:content_secret_2024@localhost:5433/chuaikan_content"
SOS_DATABASE_URL="postgresql://postgres:sos_secret_2024@localhost:5434/chuaikan_sos"
ANALYTICS_DATABASE_URL="postgresql://postgres:analytics_secret_2024@localhost:5435/chuaikan_analytics"
```

### Step 474 — Data Consistency: Event-driven Sync

```typescript
// ปัญหา: เมื่อ user update profile ใน Users DB
// → Content DB, SOS DB ต้องรู้ด้วย (denormalized data)

// Solution: Publish event ผ่าน message queue

// services/users/handlers/update-profile.handler.ts
import { usersDb } from '../../lib/db/databases';
import { publishEvent } from '../../lib/events/publisher';

export async function updateUserProfile(
  userId: string,
  data: { displayName?: string; avatarUrl?: string }
) {
  // 1. อัพเดท Users DB
  const updated = await usersDb.user.update({
    where: { id: BigInt(userId) },
    data,
  });

  // 2. Publish event (async, non-blocking)
  await publishEvent('user.profile_updated', {
    userId,
    username: updated.username,
    displayName: updated.displayName,
    avatarUrl: updated.avatarUrl,
    updatedAt: new Date().toISOString(),
  });

  return updated;
}

// services/content/handlers/user-events.handler.ts
// รับ event แล้วอัพเดท user_snapshots ใน Content DB
import { contentDb } from '../../lib/db/databases';

export async function handleUserProfileUpdated(event: {
  userId: string;
  username: string;
  displayName: string;
  avatarUrl: string;
}) {
  // อัพเดท user snapshot ใน Content DB
  await contentDb.$executeRaw`
    INSERT INTO user_snapshots (user_id, username, avatar_url, display_name, synced_at)
    VALUES (${BigInt(event.userId)}, ${event.username}, ${event.avatarUrl},
            ${event.displayName}, NOW())
    ON CONFLICT (user_id) DO UPDATE SET
      username = EXCLUDED.username,
      avatar_url = EXCLUDED.avatar_url,
      display_name = EXCLUDED.display_name,
      synced_at = NOW()
  `;
}
```

### Step 475 — API Composition (BFF Pattern)

```typescript
// api/bff/user-profile.ts
// Backend For Frontend: compose data จากหลาย services

import { usersDb } from '../../lib/db/databases';
import { contentDb } from '../../lib/db/databases';
import { sosDb } from '../../lib/db/databases';

interface UserProfileResponse {
  user: {
    id: string;
    username: string;
    displayName: string;
    avatarUrl: string;
    bio: string;
    province: string;
  };
  posts: {
    total: number;
    recent: Array<{ id: string; title: string; createdAt: string }>;
  };
  sosStats: {
    reported: number;
    resolved: number;
  };
}

export async function getUserProfile(userId: string): Promise<UserProfileResponse> {
  // Parallel fetch จากหลาย databases
  const [user, posts, sosStats] = await Promise.all([
    // 1. Users DB
    usersDb.user.findUnique({
      where: { id: BigInt(userId) },
      select: { id: true, username: true, displayName: true, avatarUrl: true, bio: true },
    }),

    // 2. Content DB
    contentDb.$queryRaw<any[]>`
      SELECT
        COUNT(*) OVER() AS total,
        id, title, created_at
      FROM posts
      WHERE user_id = ${BigInt(userId)}
        AND is_published = true
      ORDER BY created_at DESC
      LIMIT 5
    `,

    // 3. SOS DB
    sosDb.$queryRaw<any[]>`
      SELECT
        COUNT(*) FILTER (WHERE TRUE) AS reported,
        COUNT(*) FILTER (WHERE status = 'resolved') AS resolved
      FROM sos_reports
      WHERE user_id = ${BigInt(userId)}
    `,
  ]);

  if (!user) throw new Error('User not found');

  return {
    user: {
      id: user.id.toString(),
      username: user.username,
      displayName: user.displayName || user.username,
      avatarUrl: user.avatarUrl || '',
      bio: user.bio || '',
      province: '',
    },
    posts: {
      total: parseInt(posts[0]?.total ?? '0'),
      recent: posts.map(p => ({
        id: p.id.toString(),
        title: p.title,
        createdAt: p.created_at,
      })),
    },
    sosStats: {
      reported: parseInt(sosStats[0]?.reported ?? '0'),
      resolved: parseInt(sosStats[0]?.resolved ?? '0'),
    },
  };
}
```

### Step 476 — GraphQL Federation

```typescript
// services/users/graphql/schema.ts
// Users subgraph
import { buildSubgraphSchema } from '@apollo/subgraph';
import { gql } from 'apollo-server';

const typeDefs = gql`
  extend schema @link(url: "https://specs.apollo.dev/federation/v2.0", import: ["@key"])

  type User @key(fields: "id") {
    id: ID!
    username: String!
    displayName: String
    avatarUrl: String
    bio: String
  }

  type Query {
    user(id: ID!): User
    me: User
  }
`;

// services/content/graphql/schema.ts
// Content subgraph
const contentTypeDefs = gql`
  extend schema @link(url: "https://specs.apollo.dev/federation/v2.0", import: ["@key", "@external"])

  type User @key(fields: "id") {
    id: ID! @external
    posts: [Post!]!          # extend User type จาก Users service
    postCount: Int!
  }

  type Post {
    id: ID!
    title: String!
    content: String!
    author: User!
    createdAt: String!
  }

  type Query {
    post(id: ID!): Post
    posts(userId: ID, limit: Int): [Post!]!
  }
`;

// Gateway: Apollo Router config
// router.yaml
```

```yaml
# router.yaml (Apollo Router)
supergraph:
  listen: 0.0.0.0:4000

subgraphs:
  users:
    routing_url: http://users-service:4001/graphql
  content:
    routing_url: http://content-service:4002/graphql
  sos:
    routing_url: http://sos-service:4003/graphql
```

```bash
# รัน Apollo Router
docker run -d \
  --name apollo-router \
  -p 4000:4000 \
  -v $(pwd)/router.yaml:/dist/config/router.yaml \
  ghcr.io/apollographql/router:latest

# ทดสอบ federated query
curl -X POST http://localhost:4000/graphql \
  -H "Content-Type: application/json" \
  -d '{
    "query": "{ user(id: \"1\") { id username posts { title } } }"
  }'
```

### Step 477 — Data Duplication Strategy

```typescript
// lib/sync/user-sync.service.ts
// Sync user data ไปยัง services ที่ต้องการ

import { EventEmitter } from 'events';
import { usersDb, contentDb, sosDb } from '../db/databases';

export class UserSyncService {
  private eventEmitter: EventEmitter;

  constructor() {
    this.eventEmitter = new EventEmitter();
    this.setupListeners();
  }

  private setupListeners() {
    // Listen for user changes
    this.eventEmitter.on('user:created', this.syncUserToAllServices.bind(this));
    this.eventEmitter.on('user:updated', this.syncUserToAllServices.bind(this));
    this.eventEmitter.on('user:deleted', this.deleteUserFromAllServices.bind(this));
  }

  async syncUserToAllServices(userId: string) {
    // ดึงข้อมูลจาก Users DB
    const user = await usersDb.user.findUnique({
      where: { id: BigInt(userId) },
      select: {
        id: true,
        username: true,
        displayName: true,
        avatarUrl: true,
      },
    });

    if (!user) return;

    const userData = {
      userId: user.id.toString(),
      username: user.username,
      displayName: user.displayName,
      avatarUrl: user.avatarUrl,
    };

    // Sync พร้อมกัน (fire-and-forget, ไม่รอผล)
    await Promise.allSettled([
      this.syncToContentDb(userData),
      this.syncToSosDb(userData),
    ]);
  }

  private async syncToContentDb(user: any) {
    await contentDb.$executeRaw`
      INSERT INTO user_snapshots (user_id, username, display_name, avatar_url, synced_at)
      VALUES (${BigInt(user.userId)}, ${user.username}, ${user.displayName},
              ${user.avatarUrl}, NOW())
      ON CONFLICT (user_id) DO UPDATE SET
        username = EXCLUDED.username,
        display_name = EXCLUDED.display_name,
        avatar_url = EXCLUDED.avatar_url,
        synced_at = NOW()
    `;
  }

  private async syncToSosDb(user: any) {
    await sosDb.$executeRaw`
      UPDATE sos_reports
      SET username = ${user.username}
      WHERE user_id = ${BigInt(user.userId)}
        AND status NOT IN ('resolved', 'cancelled')
    `;
  }

  private async deleteUserFromAllServices(userId: string) {
    await Promise.allSettled([
      contentDb.$executeRaw`DELETE FROM user_snapshots WHERE user_id = ${BigInt(userId)}`,
      // Anonymize แทนลบ สำหรับ sos_reports (เก็บ audit)
      sosDb.$executeRaw`
        UPDATE sos_reports
        SET username = '[deleted]', user_id = NULL
        WHERE user_id = ${BigInt(userId)}
      `,
    ]);
  }
}
```

### Step 478 — Eventual Consistency: Handling Inconsistency

```typescript
// lib/sync/consistency-checker.ts
// ตรวจหาและซ่อมแซม inconsistency ระหว่าง databases

export async function checkAndRepairUserSync() {
  // ดู users ที่ snapshot ไม่ sync ใน Content DB
  const outdatedSnapshots = await contentDb.$queryRaw<any[]>`
    SELECT us.user_id, us.username, us.synced_at
    FROM user_snapshots us
    WHERE us.synced_at < NOW() - INTERVAL '1 hour'
    ORDER BY us.synced_at ASC
    LIMIT 100
  `;

  console.log(`Found ${outdatedSnapshots.length} outdated snapshots`);

  for (const snapshot of outdatedSnapshots) {
    const freshUser = await usersDb.user.findUnique({
      where: { id: snapshot.user_id },
      select: { id: true, username: true, displayName: true, avatarUrl: true },
    });

    if (!freshUser) {
      // User ถูกลบไปแล้ว → ลบ snapshot
      await contentDb.$executeRaw`
        DELETE FROM user_snapshots WHERE user_id = ${snapshot.user_id}
      `;
    } else if (freshUser.username !== snapshot.username) {
      // username เปลี่ยน → sync
      await contentDb.$executeRaw`
        UPDATE user_snapshots
        SET username = ${freshUser.username},
            avatar_url = ${freshUser.avatarUrl},
            synced_at = NOW()
        WHERE user_id = ${snapshot.user_id}
      `;
    }
  }

  return outdatedSnapshots.length;
}

// รัน checker ทุก 15 นาที
import cron from 'node-cron';
cron.schedule('*/15 * * * *', () => {
  checkAndRepairUserSync().catch(console.error);
});
```

### Step 479 — Database Migration ข้าม Services

```bash
# === Zero-downtime migration strategy ===

# สถานการณ์: ย้าย flood_zones จาก SOS DB ไปยัง Content DB

# Phase 1: สร้างตารางใน Content DB (ยังไม่ใช้งาน)
docker exec chuaikan-content-db psql -U postgres -d chuaikan_content << 'EOF'
CREATE TABLE flood_zones (
  id          SERIAL PRIMARY KEY,
  name        VARCHAR(200),
  risk_level  VARCHAR(20),
  boundary    GEOMETRY(POLYGON, 4326),
  created_at  TIMESTAMPTZ DEFAULT NOW()
);
CREATE EXTENSION IF NOT EXISTS postgis;
EOF

# Phase 2: Copy data จาก SOS DB ไป Content DB
pg_dump \
  -h localhost -p 5434 -U postgres -d chuaikan_sos \
  -t flood_zones --data-only | \
  psql -h localhost -p 5433 -U postgres -d chuaikan_content

# Phase 3: เปิด dual-write (เขียนทั้ง 2 databases)
# แก้โค้ดใน flood zone service ให้เขียนทั้ง SOS DB และ Content DB

# Phase 4: ย้าย read ไปที่ Content DB แล้วตรวจสอบ

# Phase 5: ปิด write ไป SOS DB

# Phase 6: Drop flood_zones จาก SOS DB
docker exec chuaikan-sos-db psql -U postgres -d chuaikan_sos \
  -c "DROP TABLE IF EXISTS flood_zones;"
```

### Step 480 — Service Communication Patterns

```typescript
// lib/events/publisher.ts
// Event publisher ด้วย Redis Pub/Sub หรือ RabbitMQ

import { redis } from '../cache/redis-cache';

interface DomainEvent {
  type: string;
  payload: Record<string, unknown>;
  timestamp: string;
  correlationId: string;
}

export async function publishEvent(
  channel: string,
  payload: Record<string, unknown>
) {
  const event: DomainEvent = {
    type: channel,
    payload,
    timestamp: new Date().toISOString(),
    correlationId: crypto.randomUUID(),
  };

  // Publish ไปยัง Redis channel
  await redis.publish(channel, JSON.stringify(event));

  // Store ใน event log (for replay)
  await redis.lpush('event_log', JSON.stringify(event));
  await redis.ltrim('event_log', 0, 9999); // เก็บ 10000 events ล่าสุด
}

// lib/events/subscriber.ts
import Redis from 'ioredis';

const subscriber = new Redis({ host: process.env.REDIS_HOST });

export function subscribeToEvents(
  channels: string[],
  handler: (channel: string, event: DomainEvent) => Promise<void>
) {
  subscriber.subscribe(...channels);

  subscriber.on('message', async (channel, message) => {
    try {
      const event = JSON.parse(message) as DomainEvent;
      await handler(channel, event);
    } catch (error) {
      console.error(`[Event] Error handling ${channel}:`, error);
    }
  });
}

// ตัวอย่างการใช้งานใน Content Service
subscribeToEvents(['user.profile_updated', 'user.deleted'], async (channel, event) => {
  if (channel === 'user.profile_updated') {
    await handleUserProfileUpdated(event.payload as any);
  } else if (channel === 'user.deleted') {
    await handleUserDeleted(event.payload.userId as string);
  }
});
```

---

## 🔧 Configuration Files

```yaml
# k8s/configmap-db-urls.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
  namespace: chuaikan
data:
  USERS_DB_HOST: "users-db-service"
  CONTENT_DB_HOST: "content-db-service"
  SOS_DB_HOST: "sos-db-service"
  ANALYTICS_DB_HOST: "analytics-db-service"

---
apiVersion: v1
kind: Secret
metadata:
  name: db-secrets
  namespace: chuaikan
type: Opaque
stringData:
  USERS_DATABASE_URL: "postgresql://postgres:users_secret@users-db-service:5432/chuaikan_users"
  CONTENT_DATABASE_URL: "postgresql://postgres:content_secret@content-db-service:5432/chuaikan_content"
```

---

## 🧪 Testing

```bash
# ทดสอบ cross-service query performance
time curl -s http://localhost:3000/api/users/1/profile | jq '.user.username'

# ทดสอบ consistency
# 1. Update user ใน Users DB
curl -X PATCH http://localhost:3001/api/users/1 \
  -H "Content-Type: application/json" \
  -d '{"displayName": "Updated Name"}'

# 2. รอ event propagation (~100ms)
sleep 1

# 3. ตรวจสอบ Content DB ได้รับ update
curl http://localhost:3002/api/posts?userId=1 | jq '.[0].author.displayName'
```

---

## ❌ Common Errors & Solutions

**Event ไม่ถึง subscriber**
```bash
# ตรวจสอบ Redis pub/sub
redis-cli SUBSCRIBE user.profile_updated
# แล้ว publish test event จากอีก terminal

# ตรวจสอบ event log
redis-cli LRANGE event_log 0 10 | python3 -m json.tool
```

**Stale data ใน Content DB**
```bash
# Manual sync
curl -X POST http://localhost:3002/api/admin/sync-users \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

---

## ✅ Checklist

- [ ] **Step 471** — อธิบาย Database per Service ข้อดี/ข้อเสียได้
- [ ] **Step 472** — สร้าง 4 databases แยกกัน: users, content, sos, analytics
- [ ] **Step 473** — Configure Prisma client แยกสำหรับแต่ละ service
- [ ] **Step 474** — Implement event-driven sync ด้วย publish/subscribe
- [ ] **Step 475** — สร้าง BFF API Composition สำหรับ user profile
- [ ] **Step 476** — Setup Apollo Federation gateway ด้วย 2+ subgraphs
- [ ] **Step 477** — Implement user snapshot denormalization ใน Content DB
- [ ] **Step 478** — สร้าง consistency checker รัน background job
- [ ] **Step 479** — ทำ zero-downtime migration ข้าม databases ได้
- [ ] **Step 480** — Implement Redis pub/sub event bus

---

## 🔗 References

- [Database per Service Pattern](https://microservices.io/patterns/data/database-per-service.html)
- [Apollo Federation](https://www.apollographql.com/docs/federation/)
- [Eventual Consistency](https://www.allthingsdistributed.com/2007/12/eventually_consistent.html)
- [Saga Pattern](https://microservices.io/patterns/data/saga.html)

---
*Part 048 | Road to 1,000,000 Users/Day | chuaikan.com*
