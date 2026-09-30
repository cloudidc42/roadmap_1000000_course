# Part 041: Database Sharding Strategy
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 401–410
> **เวลาโดยประมาณ:** 4 ชั่วโมง
> **Prerequisites:** Part 001–040 (PostgreSQL พื้นฐาน, Docker, Kubernetes)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- เข้าใจแนวคิด Horizontal Partitioning และ Database Sharding
- เลือก Sharding Key ที่เหมาะสมสำหรับ chuaikan.com (`user_id`)
- เปรียบเทียบ Range-based vs Hash-based Sharding
- ใช้งาน PostgreSQL Table Partitioning ด้วย `PARTITION BY HASH`
- แก้ปัญหา Cross-shard Queries
- ติดตั้งและใช้งาน Citus Extension สำหรับ Distributed Tables
- ทำ Shard Rebalancing

---

## 📖 ทฤษฎีและแนวคิด

### Step 401 — Sharding คืออะไร?

**Sharding** คือการแบ่งข้อมูลในตารางเดียวออกเป็นหลายส่วน (Shard) แล้วกระจายไปเก็บในหลาย Database Server เพื่อให้รับโหลดได้มากขึ้น

```
ก่อน Sharding:
┌────────────────────────────┐
│  DB Server (Single)        │
│  users: 10,000,000 rows    │ ← ช้ามาก, disk เต็ม
└────────────────────────────┘

หลัง Sharding (4 shards):
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Shard 0     │  │  Shard 1     │  │  Shard 2     │  │  Shard 3     │
│  users 0-24% │  │  users 25-49%│  │  users 50-74%│  │  users 75-99%│
│  2.5M rows   │  │  2.5M rows   │  │  2.5M rows   │  │  2.5M rows   │
└──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
```

### Step 402 — Sharding Key สำหรับ chuaikan.com

สำหรับ chuaikan.com เราเลือก `user_id` เป็น Sharding Key เพราะ:
- User เข้าถึงข้อมูลของตัวเองเป็นหลัก (locality)
- User ID มีการกระจายตัวสม่ำเสมอ (high cardinality)
- ป้องกัน hot spot ได้ดีกับ Consistent Hashing

### Step 403 — Range-based vs Hash-based Sharding

| แบบ | ข้อดี | ข้อเสีย |
|-----|-------|---------|
| Range-based | Query range ง่าย, rebalance ง่าย | Hot spot ถ้าข้อมูลใหม่กระจุกตัว |
| Hash-based | กระจายสม่ำเสมอ | Query range ยาก, rebalance ยากกว่า |
| Consistent Hash | Rebalance ดี, ย้ายข้อมูลน้อย | ซับซ้อนกว่า |

---

## ⚙️ Environment Setup

### ติดตั้ง PostgreSQL 16 + Citus บน Ubuntu 24.04

```bash
# Step 1: ติดตั้ง PostgreSQL 16
sudo apt-get update
sudo apt-get install -y postgresql-16 postgresql-client-16

# Step 2: เพิ่ม Citus repository
curl https://install.citusdata.com/community/deb.sh | sudo bash

# Step 3: ติดตั้ง Citus extension
sudo apt-get install -y postgresql-16-citus-12.1

# Step 4: เพิ่ม Citus ใน shared_preload_libraries
sudo -u postgres psql -c "ALTER SYSTEM SET shared_preload_libraries = 'citus';"

# Step 5: รีสตาร์ท PostgreSQL
sudo systemctl restart postgresql

# Step 6: ตรวจสอบ
sudo -u postgres psql -c "SHOW shared_preload_libraries;"
```

### Docker Compose สำหรับ Citus Cluster

```yaml
# docker-compose-citus.yml
version: '3.8'

services:
  citus-coordinator:
    image: citusdata/citus:12.1
    container_name: citus-coordinator
    ports:
      - "5432:5432"
    environment:
      POSTGRES_PASSWORD: chuaikan_secret
      POSTGRES_DB: chuaikan_db
    volumes:
      - citus_coordinator_data:/var/lib/postgresql/data
    networks:
      - citus-net

  citus-worker-1:
    image: citusdata/citus:12.1
    container_name: citus-worker-1
    environment:
      POSTGRES_PASSWORD: chuaikan_secret
      POSTGRES_DB: chuaikan_db
    volumes:
      - citus_worker1_data:/var/lib/postgresql/data
    networks:
      - citus-net

  citus-worker-2:
    image: citusdata/citus:12.1
    container_name: citus-worker-2
    environment:
      POSTGRES_PASSWORD: chuaikan_secret
      POSTGRES_DB: chuaikan_db
    volumes:
      - citus_worker2_data:/var/lib/postgresql/data
    networks:
      - citus-net

  citus-worker-3:
    image: citusdata/citus:12.1
    container_name: citus-worker-3
    environment:
      POSTGRES_PASSWORD: chuaikan_secret
      POSTGRES_DB: chuaikan_db
    volumes:
      - citus_worker3_data:/var/lib/postgresql/data
    networks:
      - citus-net

volumes:
  citus_coordinator_data:
  citus_worker1_data:
  citus_worker2_data:
  citus_worker3_data:

networks:
  citus-net:
    driver: bridge
```

```bash
# รัน Citus cluster
docker-compose -f docker-compose-citus.yml up -d

# รอให้ทุก container พร้อม
sleep 10

# เพิ่ม worker nodes
docker exec citus-coordinator psql -U postgres -c \
  "SELECT citus_add_node('citus-worker-1', 5432);"
docker exec citus-coordinator psql -U postgres -c \
  "SELECT citus_add_node('citus-worker-2', 5432);"
docker exec citus-coordinator psql -U postgres -c \
  "SELECT citus_add_node('citus-worker-3', 5432);"

# ตรวจสอบ workers
docker exec citus-coordinator psql -U postgres -c \
  "SELECT * FROM citus_get_active_worker_nodes();"
```

---

## 🛠️ Step-by-Step Implementation

### Step 404 — PostgreSQL Native Partitioning (PARTITION BY HASH)

```sql
-- เชื่อมต่อกับ PostgreSQL
sudo -u postgres psql -d chuaikan_db

-- สร้าง partitioned table
CREATE TABLE users (
    id          BIGSERIAL,
    username    VARCHAR(50)  NOT NULL,
    email       VARCHAR(255) NOT NULL UNIQUE,
    created_at  TIMESTAMPTZ  DEFAULT NOW(),
    updated_at  TIMESTAMPTZ  DEFAULT NOW(),
    is_active   BOOLEAN      DEFAULT true,
    PRIMARY KEY (id)
) PARTITION BY HASH (id);

-- สร้าง 4 partitions
CREATE TABLE users_0 PARTITION OF users
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);

CREATE TABLE users_1 PARTITION OF users
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);

CREATE TABLE users_2 PARTITION OF users
    FOR VALUES WITH (MODULUS 4, REMAINDER 2);

CREATE TABLE users_3 PARTITION OF users
    FOR VALUES WITH (MODULUS 4, REMAINDER 3);

-- สร้าง index ใน partitions
CREATE INDEX CONCURRENTLY ON users_0 (username);
CREATE INDEX CONCURRENTLY ON users_1 (username);
CREATE INDEX CONCURRENTLY ON users_2 (username);
CREATE INDEX CONCURRENTLY ON users_3 (username);

-- ใส่ข้อมูลทดสอบ
INSERT INTO users (username, email)
SELECT
    'user_' || i,
    'user_' || i || '@chuaikan.com'
FROM generate_series(1, 1000000) AS i;

-- ตรวจสอบการกระจายข้อมูล
SELECT
    tableoid::regclass AS partition,
    COUNT(*) AS row_count
FROM users
GROUP BY tableoid
ORDER BY partition;
```

### Step 405 — Consistent Hash Sharding ด้วย Citus

```sql
-- เชื่อมต่อผ่าน coordinator
docker exec -it citus-coordinator psql -U postgres -d chuaikan_db

-- สร้าง extension
CREATE EXTENSION IF NOT EXISTS citus;

-- สร้างตาราง users
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    username    VARCHAR(50)  NOT NULL,
    email       VARCHAR(255) NOT NULL,
    created_at  TIMESTAMPTZ  DEFAULT NOW(),
    profile_data JSONB       DEFAULT '{}'
);

-- แจกจ่ายตารางด้วย user_id เป็น distribution key
SELECT create_distributed_table('users', 'id');

-- ตรวจสอบ shards
SELECT * FROM citus_shards WHERE logicalrelid = 'users'::regclass;

-- สร้างตาราง posts (co-located กับ users)
CREATE TABLE posts (
    id          BIGSERIAL,
    user_id     BIGINT       NOT NULL,
    title       VARCHAR(500),
    content     TEXT,
    created_at  TIMESTAMPTZ  DEFAULT NOW(),
    PRIMARY KEY (id, user_id)
);

-- Co-locate posts กับ users (shard key เดียวกัน = join ไม่ข้าม shard)
SELECT create_distributed_table('posts', 'user_id',
    colocate_with => 'users');

-- ตรวจสอบว่า co-located
SELECT logicalrelid, colocationid
FROM pg_dist_partition
WHERE logicalrelid IN ('users'::regclass, 'posts'::regclass);
```

### Step 406 — ใส่ข้อมูลและทดสอบ Distribution

```sql
-- ใส่ข้อมูลทดสอบ
INSERT INTO users (username, email)
SELECT
    'user_' || i,
    'user_' || i || '@chuaikan.com'
FROM generate_series(1, 100000) AS i;

-- Query ที่ efficient (filter by distribution key)
EXPLAIN (ANALYZE, COSTS, VERBOSE)
SELECT u.id, u.username, COUNT(p.id) AS post_count
FROM users u
LEFT JOIN posts p ON p.user_id = u.id
WHERE u.id = 42
GROUP BY u.id, u.username;
-- ควรเห็น "Custom Scan (Citus Router)" = ไปที่ shard เดียว

-- Query แบบ aggregate ข้าม shards
EXPLAIN (ANALYZE, COSTS, VERBOSE)
SELECT DATE_TRUNC('day', created_at) AS day, COUNT(*)
FROM users
GROUP BY day
ORDER BY day DESC
LIMIT 30;
-- เห็น "Custom Scan (Citus Real-Time)" = ทำงานใน parallel ทุก shard
```

### Step 407 — Cross-Shard Queries และวิธีแก้

```sql
-- ❌ ปัญหา: JOIN ข้าม distribution key ต่างกัน (ช้ามาก!)
-- users กระจายด้วย id, sos_requests กระจายด้วย area_id
-- JOIN แบบนี้ต้องดึงข้อมูลข้าม shard

-- ✅ วิธีแก้ 1: Reference Table (ข้อมูลน้อย, copy ทุก shard)
CREATE TABLE thai_provinces (
    id    SERIAL PRIMARY KEY,
    name  VARCHAR(100) NOT NULL,
    code  VARCHAR(10)
);

-- reference table มีสำเนาอยู่ทุก shard
SELECT create_reference_table('thai_provinces');

INSERT INTO thai_provinces (name, code) VALUES
    ('กรุงเทพมหานคร', 'BKK'),
    ('เชียงใหม่', 'CNX'),
    ('ภูเก็ต', 'HKT');

-- JOIN กับ reference table ทำงานได้ปกติใน shard เดียว
SELECT u.username, p.name AS province
FROM users u
JOIN thai_provinces p ON p.code = u.profile_data->>'province_code'
WHERE u.id = 42;

-- ✅ วิธีแก้ 2: Re-design schema ให้ใช้ distribution key เดียวกัน
-- ย้าย area_id เป็น shard ผ่าน user_id แทน

-- ✅ วิธีแก้ 3: Application-level join (N+1 ที่ควบคุม)
-- ดึงจาก shard A แล้ว lookup ใน shard B แยกกัน
```

### Step 408 — Shard Rebalancing

```bash
# ดูสถานะ shard ปัจจุบัน
docker exec citus-coordinator psql -U postgres -d chuaikan_db -c \
  "SELECT nodename, COUNT(*) AS shard_count,
          pg_size_pretty(SUM(shard_size)) AS total_size
   FROM citus_shards
   GROUP BY nodename;"

# เพิ่ม worker node ใหม่
docker exec citus-coordinator psql -U postgres -d chuaikan_db -c \
  "SELECT citus_add_node('citus-worker-4', 5432);"

# Rebalance shards (เริ่ม background job)
docker exec citus-coordinator psql -U postgres -d chuaikan_db -c \
  "SELECT citus_rebalance_start();"

# ติดตาม progress
docker exec citus-coordinator psql -U postgres -d chuaikan_db -c \
  "SELECT * FROM citus_rebalance_status();"

# รอให้เสร็จ (อาจใช้เวลาหลายนาทีถ้าข้อมูลเยอะ)
watch -n5 "docker exec citus-coordinator psql -U postgres -d chuaikan_db \
  -c \"SELECT * FROM citus_rebalance_status();\""
```

### Step 409 — Shard Monitoring

```sql
-- ดูขนาด shards แต่ละ node
SELECT
    n.nodename,
    n.nodeport,
    s.logicalrelid::text AS table_name,
    COUNT(*) AS shard_count,
    pg_size_pretty(SUM(s.shard_size)) AS total_size
FROM citus_shards s
JOIN pg_dist_node n ON n.nodeid = s.nodeid
GROUP BY n.nodename, n.nodeport, s.logicalrelid
ORDER BY SUM(s.shard_size) DESC;

-- ตรวจสอบ imbalance
SELECT
    nodename,
    COUNT(*) AS shard_count,
    pg_size_pretty(SUM(shard_size)) AS total_size,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 2) AS pct
FROM citus_shards
GROUP BY nodename
ORDER BY nodename;

-- Query performance ข้าม shards
SELECT
    query,
    calls,
    ROUND(mean_exec_time::numeric, 2) AS avg_ms,
    ROUND(total_exec_time::numeric, 2) AS total_ms
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

### Step 410 — Application-level Sharding (ไม่ใช้ Citus)

```typescript
// lib/db/shard-router.ts
// สำหรับกรณีที่ใช้ Native PostgreSQL Partitioning

import { Pool } from 'pg';

interface ShardConfig {
  id: number;
  host: string;
  port: number;
  database: string;
}

const SHARD_CONFIGS: ShardConfig[] = [
  { id: 0, host: 'db-shard-0', port: 5432, database: 'chuaikan_shard0' },
  { id: 1, host: 'db-shard-1', port: 5432, database: 'chuaikan_shard1' },
  { id: 2, host: 'db-shard-2', port: 5432, database: 'chuaikan_shard2' },
  { id: 3, host: 'db-shard-3', port: 5432, database: 'chuaikan_shard3' },
];

class ShardRouter {
  private pools: Map<number, Pool> = new Map();
  private readonly NUM_SHARDS = 4;

  constructor() {
    SHARD_CONFIGS.forEach(config => {
      this.pools.set(config.id, new Pool({
        host: config.host,
        port: config.port,
        database: config.database,
        user: process.env.DB_USER,
        password: process.env.DB_PASSWORD,
        max: 20,
      }));
    });
  }

  // Consistent hash ด้วย user_id
  getShardId(userId: bigint): number {
    // FNV-1a hash สำหรับกระจายสม่ำเสมอ
    const hash = this.fnv1a(userId.toString());
    return Number(hash % BigInt(this.NUM_SHARDS));
  }

  private fnv1a(input: string): bigint {
    let hash = 14695981039346656037n;
    const FNV_PRIME = 1099511628211n;
    for (const char of input) {
      hash ^= BigInt(char.charCodeAt(0));
      hash = (hash * FNV_PRIME) & 0xFFFFFFFFFFFFFFFFn;
    }
    return hash;
  }

  getPool(userId: bigint): Pool {
    const shardId = this.getShardId(userId);
    const pool = this.pools.get(shardId);
    if (!pool) throw new Error(`No pool for shard ${shardId}`);
    return pool;
  }

  // Fan-out query ไปทุก shard แล้ว merge
  async queryAllShards<T>(
    sql: string,
    params: unknown[] = []
  ): Promise<T[]> {
    const promises = Array.from(this.pools.values()).map(pool =>
      pool.query<T>(sql, params).then(r => r.rows)
    );
    const results = await Promise.all(promises);
    return results.flat();
  }
}

export const shardRouter = new ShardRouter();

// ตัวอย่างการใช้งาน
async function getUserById(userId: bigint) {
  const pool = shardRouter.getPool(userId);
  const result = await pool.query(
    'SELECT * FROM users WHERE id = $1',
    [userId]
  );
  return result.rows[0];
}

// Fan-out: ดึง active users ทุก shard
async function getActiveUserCount() {
  const counts = await shardRouter.queryAllShards<{ count: string }>(
    'SELECT COUNT(*) as count FROM users WHERE is_active = true'
  );
  return counts.reduce((sum, row) => sum + parseInt(row.count), 0);
}
```

---

## 🔧 Configuration Files

```ini
# /etc/postgresql/16/main/postgresql.conf
# Citus configuration
shared_preload_libraries = 'citus,pg_stat_statements'
citus.shard_count = 32          # จำนวน shards ทั้งหมด
citus.shard_replication_factor = 1  # 1 = ไม่มี replica ของ shard
citus.node_connection_timeout = 5000  # 5 วินาที
citus.multi_shard_modify_mode = 'sequential'  # ปลอดภัยกว่า parallel

# Performance
max_connections = 200
shared_buffers = 4GB
work_mem = 64MB
maintenance_work_mem = 1GB
effective_cache_size = 12GB
```

---

## 🧪 Testing

```bash
# ทดสอบ distribution evenness
docker exec citus-coordinator psql -U postgres -d chuaikan_db << 'EOF'
-- insert 1 million rows
INSERT INTO users (username, email)
SELECT 'u' || i, 'u' || i || '@test.com'
FROM generate_series(1, 1000000) i;

-- check distribution
SELECT
  nodename,
  COUNT(*) shard_count,
  pg_size_pretty(SUM(shard_size)) total_size,
  ROUND(100.0 * SUM(shard_size) / SUM(SUM(shard_size)) OVER (), 2) AS pct
FROM citus_shards
WHERE logicalrelid = 'users'::regclass
GROUP BY nodename
ORDER BY nodename;
EOF

# ทดสอบ query performance
docker exec citus-coordinator pgbench -U postgres -d chuaikan_db \
  -c 50 -j 4 -T 60 \
  -f /tmp/shard_test.sql
```

```sql
-- /tmp/shard_test.sql (pgbench script)
\set uid random(1, 1000000)
SELECT id, username, email FROM users WHERE id = :uid;
```

---

## ❌ Common Errors & Solutions

**Error: `could not connect to server: Connection refused`**
```bash
# ตรวจสอบว่า worker nodes ทำงาน
docker ps | grep citus-worker
# แก้: รีสตาร์ท worker ที่ล้มเหลว
docker restart citus-worker-1
```

**Error: `relation "users" does not exist on worker`**
```sql
-- ต้องสร้างตารางผ่าน coordinator เท่านั้น (ไม่ใช่ worker โดยตรง)
-- coordinator จะ sync schema ไปยัง workers อัตโนมัติ
SELECT run_command_on_workers($$ CREATE INDEX ON users(username) $$);
```

**Error: `cannot use subquery or CTE on a distributed table`**
```sql
-- ❌ ไม่ work
SELECT * FROM users WHERE id IN (SELECT user_id FROM banned_users);

-- ✅ ใช้ JOIN แทน
SELECT u.* FROM users u
JOIN banned_users b ON b.user_id = u.id;

-- หรือ pull ไปที่ coordinator ก่อน
SELECT user_id FROM banned_users;  -- run in app
-- แล้วค่อย
SELECT * FROM users WHERE id = ANY($1::bigint[]);
```

---

## ✅ Checklist

- [ ] **Step 401** — เข้าใจความแตกต่างระหว่าง Partitioning และ Sharding
- [ ] **Step 402** — เลือก `user_id` เป็น Sharding Key สำหรับ chuaikan.com
- [ ] **Step 403** — อธิบาย Range-based vs Hash-based ได้
- [ ] **Step 404** — สร้าง `PARTITION BY HASH` ใน PostgreSQL ได้
- [ ] **Step 405** — ติดตั้ง Citus และสร้าง distributed table ได้
- [ ] **Step 406** — ตรวจสอบการกระจายข้อมูลสม่ำเสมอด้วย `citus_shards`
- [ ] **Step 407** — แก้ปัญหา Cross-shard query ด้วย reference table
- [ ] **Step 408** — Rebalance shards หลังเพิ่ม worker node ใหม่
- [ ] **Step 409** — Monitor shard size และ imbalance
- [ ] **Step 410** — เขียน Application-level shard router ใน TypeScript

---

## 🔗 References

- [Citus Documentation](https://docs.citusdata.com/)
- [PostgreSQL Table Partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html)
- [Consistent Hashing Explained](https://highscalability.com/consistent-hashing-algorithm/)
- [Distributed Systems Patterns — Sharding](https://martinfowler.com/articles/patterns-of-distributed-systems/sharding.html)

---
*Part 041 | Road to 1,000,000 Users/Day | chuaikan.com*
