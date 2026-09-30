# Part 045: Database Query Optimization Advanced
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 441–450
> **เวลาโดยประมาณ:** 4 ชั่วโมง
> **Prerequisites:** Part 044 (PostGIS), PostgreSQL พื้นฐาน, Prisma ORM

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- อ่าน EXPLAIN ANALYZE output: cost, rows, actual time
- เข้าใจ Index Scan vs Sequential Scan decisions
- ใช้ Covering Indexes (INCLUDE columns)
- ใช้ Partial Indexes (WHERE clause)
- Index on expression (function-based index)
- ตรวจหาและแก้ N+1 query ใน Prisma
- เขียน Query ที่ดีขึ้น: CTEs vs subqueries vs joins
- ปรับ PostgreSQL planner statistics (ANALYZE, VACUUM)
- ใช้ pg_stat_statements หา 10 slow queries

---

## 📖 ทฤษฎีและแนวคิด

### Step 441 — EXPLAIN ANALYZE Deep Dive

```sql
-- ตัวอย่าง output และการอ่านความหมาย
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM users WHERE username = 'john_doe';

/*
Output:
Index Scan using idx_users_username on users
  (cost=0.43..8.45 rows=1 width=200)     ← estimated
  (actual time=0.123..0.124 rows=1 loops=1)  ← actual
  Index Cond: ((username)::text = 'john_doe'::text)
  Buffers: shared hit=4 read=0           ← buffer cache hits
Planning Time: 0.123 ms
Execution Time: 0.145 ms

อ่านค่า:
cost=0.43..8.45:
  - 0.43 = startup cost (overhead ก่อนดึง row แรก)
  - 8.45 = total cost (หน่วยเป็น "cost units" ไม่ใช่วินาที)

rows=1: planner คาดว่าจะได้ 1 row
width=200: ขนาดเฉลี่ยต่อ row (bytes)
actual time=0.123..0.124: เวลาจริง (ms)
loops=1: node นี้ run กี่รอบ

Buffers: shared hit=4
  - hit = อ่านจาก cache (RAM) เร็ว
  - read = อ่านจาก disk ช้า
  ← อยากเห็น hit สูง, read ต่ำ
*/
```

---

## ⚙️ Environment Setup

```bash
# ติดตั้ง pg_stat_statements
sudo -u postgres psql -d chuaikan_db << 'EOF'
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
ALTER SYSTEM SET shared_preload_libraries = 'pg_stat_statements,timescaledb';
EOF

sudo systemctl restart postgresql

# ตั้งค่า pg_stat_statements
sudo -u postgres psql << 'EOF'
ALTER SYSTEM SET pg_stat_statements.track = 'all';
ALTER SYSTEM SET pg_stat_statements.max = 10000;
SELECT pg_reload_conf();
EOF
```

---

## 🛠️ Step-by-Step Implementation

### Step 442 — Index Scan vs Sequential Scan

```sql
-- สร้างตารางทดสอบ
CREATE TABLE test_optimization AS
SELECT
    i AS id,
    'user_' || (i % 10000) AS username,
    (i % 5) AS status,
    NOW() - (random() * INTERVAL '365 days') AS created_at
FROM generate_series(1, 1000000) i;

-- Case 1: Sequential Scan (ไม่มี index, หรือ selectivity ต่ำ)
EXPLAIN ANALYZE SELECT * FROM test_optimization WHERE status = 1;
/*
Seq Scan on test_optimization (cost=0..21692 rows=200000 width=48)
→ 20% ของข้อมูล = index ไม่คุ้ม
*/

-- Case 2: Index ช่วยได้ (high selectivity)
CREATE INDEX ON test_optimization(username);
EXPLAIN ANALYZE SELECT * FROM test_optimization WHERE username = 'user_1';
/*
Index Scan using test_optimization_username_idx
→ ดึงแค่ไม่กี่ rows = index คุ้มมาก
*/

-- Rule of thumb:
-- < 5% rows → Index Scan เร็วกว่า
-- > 20% rows → Seq Scan เร็วกว่า (avoid random I/O)
-- 5-20% → ขึ้นกับ data layout และ cache

-- บังคับ planner เพื่อทดสอบ
SET enable_seqscan = off;
EXPLAIN ANALYZE SELECT * FROM test_optimization WHERE status = 1;
SET enable_seqscan = on;
```

### Step 443 — Covering Indexes (INCLUDE)

```sql
-- ปัญหา: query ต้องการ id, username, email แต่ index มีแค่ username
-- → ต้อง "heap fetch" เพื่อดึง email (เพิ่ม I/O)

-- ❌ ไม่มี covering index
CREATE INDEX idx_users_username ON users(username);
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, username, email FROM users WHERE username = 'user_1';
/*
Index Scan using idx_users_username on users
Heap Fetches: 1    ← ต้องไปดึง email จาก heap
*/

-- ✅ Covering index (INCLUDE columns)
DROP INDEX idx_users_username;
CREATE INDEX idx_users_username_covering
    ON users(username)
    INCLUDE (id, email);  -- INCLUDE = เก็บค่าใน index แต่ไม่ sort by

EXPLAIN (ANALYZE, BUFFERS)
SELECT id, username, email FROM users WHERE username = 'user_1';
/*
Index Only Scan using idx_users_username_covering
Heap Fetches: 0    ← ดึงจาก index ได้เลย ไม่ต้องไป heap
*/

-- ตัวอย่าง covering index สำหรับ chuaikan.com
-- Query: GET /api/posts?user_id=X (ดู posts ของ user)
CREATE INDEX idx_posts_user_covering
    ON posts(user_id, created_at DESC)
    INCLUDE (id, title, thumbnail_url, like_count);

-- Query นี้ใช้ Index Only Scan ได้เลย
SELECT id, title, thumbnail_url, like_count
FROM posts
WHERE user_id = 42
ORDER BY created_at DESC
LIMIT 20;
```

### Step 444 — Partial Indexes (WHERE clause)

```sql
-- Partial Index: index เฉพาะ subset ของข้อมูล
-- ประหยัด space และเร็วกว่า full index

-- ตัวอย่าง: index เฉพาะ active users
CREATE INDEX idx_users_active_email
    ON users(email)
    WHERE is_active = true;

-- Query นี้จะใช้ partial index (เร็วและ index เล็กกว่า)
SELECT * FROM users WHERE email = 'user@example.com' AND is_active = true;

-- ตัวอย่าง: index SOS reports เฉพาะที่ pending
CREATE INDEX idx_sos_pending
    ON sos_reports(created_at DESC, province_id)
    WHERE status = 'pending';

-- Dashboard query เร็วมาก (scan แค่ pending records)
SELECT COUNT(*), province_id
FROM sos_reports
WHERE status = 'pending'
GROUP BY province_id;

-- ตัวอย่าง: index flood sensors ที่ battery ต่ำ
CREATE INDEX idx_sensors_low_battery
    ON sensor_stations(last_battery_pct)
    WHERE last_battery_pct < 20;

-- Alert query
SELECT station_id, last_battery_pct
FROM sensor_stations
WHERE last_battery_pct < 20
ORDER BY last_battery_pct;
```

### Step 445 — Index on Expression

```sql
-- Expression index: index บน computed value

-- ตัวอย่าง: case-insensitive search
CREATE INDEX idx_users_username_lower
    ON users(LOWER(username));

-- Query ต้องใช้ expression เดียวกับ index
SELECT * FROM users WHERE LOWER(username) = LOWER('JohnDoe');
-- ✅ ใช้ index ได้

-- ตัวอย่าง: index บน JSONB field
CREATE INDEX idx_posts_tags_gin
    ON posts USING GIN (tags);  -- tags เป็น JSONB array

-- Query
SELECT * FROM posts WHERE tags @> '["flood"]';  -- contains 'flood'

-- ตัวอย่าง: index บน computed column
CREATE INDEX idx_sensor_hour
    ON sensor_readings(DATE_TRUNC('hour', time), sensor_id);

-- Query
SELECT AVG(water_level)
FROM sensor_readings
WHERE DATE_TRUNC('hour', time) = '2024-01-15 10:00:00'
  AND sensor_id = 'SENSOR_0001';

-- ตัวอย่าง: index on COALESCE
CREATE INDEX idx_posts_published_at
    ON posts(COALESCE(published_at, created_at) DESC);

SELECT * FROM posts
ORDER BY COALESCE(published_at, created_at) DESC
LIMIT 20;
```

### Step 446 — N+1 Query Detection และแก้ด้วย Prisma

```typescript
// ❌ N+1 Problem
// Query 1: ดึง 10 posts
const posts = await prisma.post.findMany({ take: 10 });

// Query 2-11: ดึง author ของแต่ละ post (10 queries!)
for (const post of posts) {
  const author = await prisma.user.findUnique({
    where: { id: post.userId }
  });
  console.log(`${post.title} by ${author?.username}`);
}
// Total: 11 queries!

// ✅ แก้ด้วย include (JOIN)
const postsWithAuthors = await prisma.post.findMany({
  take: 10,
  include: {
    user: {
      select: { id: true, username: true, avatarUrl: true }
    }
  }
});
// Total: 1 query!

// ✅ แก้ด้วย select + nested (ยืดหยุ่นกว่า)
const postsData = await prisma.post.findMany({
  take: 10,
  select: {
    id: true,
    title: true,
    createdAt: true,
    user: {
      select: { username: true }
    },
    _count: {
      select: { likes: true, comments: true }
    }
  }
});

// ✅ แก้ด้วย raw query เมื่อต้องการ performance สูงสุด
const rawPosts = await prisma.$queryRaw`
  SELECT
    p.id,
    p.title,
    p.created_at,
    u.username,
    u.avatar_url,
    COUNT(DISTINCT l.id) AS like_count,
    COUNT(DISTINCT c.id) AS comment_count
  FROM posts p
  JOIN users u ON u.id = p.user_id
  LEFT JOIN likes l ON l.post_id = p.id
  LEFT JOIN comments c ON c.post_id = p.id
  WHERE p.is_published = true
  GROUP BY p.id, u.id
  ORDER BY p.created_at DESC
  LIMIT 10
`;
```

```bash
# ตรวจหา N+1 ด้วย Prisma logging
# .env
DATABASE_URL="..."
PRISMA_QUERY_ENGINE_LOG=query
```

```typescript
// prisma/client.ts — เปิด query logging
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient({
  log: [
    { level: 'query', emit: 'event' },
    { level: 'warn', emit: 'stdout' },
    { level: 'error', emit: 'stdout' },
  ],
});

// Count queries per request
let queryCount = 0;
prisma.$on('query', (e) => {
  queryCount++;
  if (process.env.NODE_ENV === 'development') {
    console.log(`[Query ${queryCount}] ${e.duration}ms: ${e.query}`);
  }
});

// Middleware เพื่อ warn ถ้า > 5 queries ต่อ request
export function resetQueryCount() { queryCount = 0; }
export function getQueryCount() { return queryCount; }
```

### Step 447 — CTEs vs Subqueries vs Joins

```sql
-- ตัวอย่าง: ดู users ที่มี SOS reports active ในพื้นที่น้ำท่วม

-- ❌ Nested Subquery (อ่านยาก, อาจ re-execute หลายครั้ง)
SELECT * FROM users
WHERE id IN (
    SELECT DISTINCT user_id FROM sos_reports
    WHERE status = 'pending'
    AND created_at > NOW() - INTERVAL '24 hours'
    AND id IN (
        SELECT sos_id FROM sos_in_flood_zone
        WHERE flood_zone_id IN (
            SELECT id FROM flood_zones WHERE risk_level = 'critical'
        )
    )
);

-- ✅ JOIN (มักเร็วที่สุด, planner optimize ได้ดี)
SELECT DISTINCT u.*
FROM users u
JOIN sos_reports sr ON sr.user_id = u.id
JOIN sos_in_flood_zone sfz ON sfz.sos_id = sr.id
JOIN flood_zones fz ON fz.id = sfz.flood_zone_id
WHERE sr.status = 'pending'
  AND sr.created_at > NOW() - INTERVAL '24 hours'
  AND fz.risk_level = 'critical';

-- ✅ CTE (อ่านง่าย, planner สามารถ optimize ใน pg14+)
WITH critical_zones AS (
    SELECT id FROM flood_zones WHERE risk_level = 'critical'
),
recent_pending_sos AS (
    SELECT sr.id, sr.user_id
    FROM sos_reports sr
    WHERE sr.status = 'pending'
      AND sr.created_at > NOW() - INTERVAL '24 hours'
),
sos_in_critical AS (
    SELECT DISTINCT rps.user_id
    FROM recent_pending_sos rps
    JOIN sos_in_flood_zone sfz ON sfz.sos_id = rps.id
    WHERE sfz.flood_zone_id IN (SELECT id FROM critical_zones)
)
SELECT u.*
FROM users u
WHERE u.id IN (SELECT user_id FROM sos_in_critical);

-- Note: ใน PostgreSQL 12+ CTE ไม่ใช่ "optimization fence" แล้ว
-- ยกเว้น MATERIALIZED keyword
WITH MATERIALIZED expensive_calc AS (
    SELECT ... FROM large_table  -- force CTE เป็น temp table
)
SELECT * FROM expensive_calc WHERE ...;
```

### Step 448 — PostgreSQL Query Planner Statistics

```sql
-- ANALYZE อัพเดท statistics ของ planner
ANALYZE users;             -- specific table
ANALYZE;                   -- ทุก table

-- ดู statistics ปัจจุบัน
SELECT
    attname AS column,
    n_distinct,
    correlation,           -- 1.0 = เรียงตามลำดับ, 0 = random
    most_common_vals,
    most_common_freqs
FROM pg_stats
WHERE tablename = 'users'
  AND attname IN ('status', 'province_id', 'is_active');

-- ปรับ statistics target (ค่าเริ่มต้น = 100)
-- สูงกว่า = แม่นยำกว่า แต่ ANALYZE ช้ากว่า
ALTER TABLE users ALTER COLUMN province_id
    SET STATISTICS 500;   -- สำหรับ column ที่ใช้ใน WHERE บ่อย

-- VACUUM อัพเดท dead rows และ visibility map
VACUUM ANALYZE users;

-- ดู bloat (dead rows)
SELECT
    relname,
    n_live_tup,
    n_dead_tup,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;

-- ตั้งค่า autovacuum ให้ aggressive กว่าสำหรับ table ที่ update บ่อย
ALTER TABLE sensor_readings SET (
    autovacuum_vacuum_scale_factor = 0.01,  -- vacuum ที่ 1% dead rows
    autovacuum_analyze_scale_factor = 0.005  -- analyze ที่ 0.5%
);
```

### Step 449 — pg_stat_statements: Top 10 Slow Queries

```sql
-- Enable pg_stat_statements (ต้องทำก่อน, ใน setup แล้ว)
-- ดู top 10 slowest queries (total execution time)
SELECT
    ROUND(total_exec_time::numeric, 2) AS total_ms,
    calls,
    ROUND(mean_exec_time::numeric, 2) AS avg_ms,
    ROUND(stddev_exec_time::numeric, 2) AS stddev_ms,
    ROUND((100 * total_exec_time / SUM(total_exec_time) OVER ())::numeric, 2) AS pct,
    LEFT(query, 100) AS query_preview
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- ดู queries ที่มี avg time สูงสุด (ช้าต่อครั้ง)
SELECT
    calls,
    ROUND(mean_exec_time::numeric, 2) AS avg_ms,
    ROUND(max_exec_time::numeric, 2) AS max_ms,
    rows / calls AS avg_rows,
    LEFT(query, 150) AS query
FROM pg_stat_statements
WHERE calls > 100  -- ถูกเรียกบ่อยพอสมควร
ORDER BY mean_exec_time DESC
LIMIT 10;

-- ดู queries ที่เรียกบ่อยที่สุด
SELECT
    calls,
    ROUND(total_exec_time::numeric, 2) AS total_ms,
    ROUND(mean_exec_time::numeric, 2) AS avg_ms,
    LEFT(query, 150) AS query
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;

-- Reset stats เพื่อเริ่มนับใหม่
SELECT pg_stat_statements_reset();
```

### Step 450 — Automated Slow Query Detection

```typescript
// scripts/analyze-slow-queries.ts
import { prismaWrite } from '../lib/db/connection';

interface SlowQuery {
  totalMs: number;
  calls: number;
  avgMs: number;
  query: string;
}

export async function getTopSlowQueries(
  limit = 10,
  minAvgMs = 100
): Promise<SlowQuery[]> {
  const rows = await prismaWrite.$queryRaw<any[]>`
    SELECT
      ROUND(total_exec_time::numeric, 2) AS total_ms,
      calls,
      ROUND(mean_exec_time::numeric, 2) AS avg_ms,
      LEFT(query, 300) AS query
    FROM pg_stat_statements
    WHERE mean_exec_time > ${minAvgMs}
      AND calls > 10
    ORDER BY total_exec_time DESC
    LIMIT ${limit}
  `;

  return rows.map(r => ({
    totalMs: Number(r.total_ms),
    calls: Number(r.calls),
    avgMs: Number(r.avg_ms),
    query: r.query,
  }));
}

// Run และแสดงผล
async function main() {
  const slow = await getTopSlowQueries(10, 50);

  console.log('\n=== TOP 10 SLOW QUERIES ===\n');
  slow.forEach((q, i) => {
    console.log(`[${i + 1}] avg: ${q.avgMs}ms | calls: ${q.calls} | total: ${q.totalMs}ms`);
    console.log(`    ${q.query}\n`);
  });
}

main().catch(console.error);
```

```bash
# รัน slow query analyzer
npx ts-node scripts/analyze-slow-queries.ts

# หรือตั้ง cron job รันทุกคืน
crontab -e
# 0 2 * * * /usr/bin/npx ts-node /app/scripts/analyze-slow-queries.ts >> /var/log/slow-queries.log 2>&1
```

---

## 🔧 Configuration Files

```ini
# postgresql.conf — Query optimization settings
# planner cost settings (ปรับให้ match hardware)
random_page_cost = 1.1          # SSD: ลด default จาก 4.0 เป็น 1.1
seq_page_cost = 1.0
effective_cache_size = 8GB      # RAM ที่ OS ใช้ cache ไว้
work_mem = 64MB                 # สำหรับ sort/hash (ต่อ operation)

# parallel query
max_parallel_workers_per_gather = 4
parallel_tuple_cost = 0.1
parallel_setup_cost = 1000

# pg_stat_statements
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = 'all'
pg_stat_statements.max = 10000
pg_stat_statements.save = on
```

---

## 🧪 Testing

```bash
# สร้าง test data
sudo -u postgres psql -d chuaikan_db << 'EOF'
INSERT INTO posts (user_id, title, content, created_at)
SELECT
    (random() * 10000 + 1)::bigint,
    'Post ' || i,
    repeat('content ', 100),
    NOW() - (random() * INTERVAL '365 days')
FROM generate_series(1, 500000) i;

-- ทดสอบ query ก่อนและหลัง index
EXPLAIN ANALYZE
SELECT u.username, COUNT(p.id)
FROM users u
LEFT JOIN posts p ON p.user_id = u.id
WHERE u.is_active = true
GROUP BY u.username
ORDER BY COUNT(p.id) DESC
LIMIT 100;
EOF
```

---

## ❌ Common Errors & Solutions

**Planner เลือก Seq Scan แม้มี Index**
```sql
-- 1. ANALYZE เพื่ออัพเดท statistics
ANALYZE users;

-- 2. ตรวจ correlation (ถ้าต่ำ = random I/O, seq scan เร็วกว่า)
SELECT attname, correlation FROM pg_stats WHERE tablename = 'users';

-- 3. CLUSTER table เรียง physical order ตาม index
CLUSTER users USING idx_users_created_at;
-- (lock table ชั่วคราว, ใช้ production ระวัง)
```

**N+1 ยังเกิดอยู่แม้ใช้ include**
```typescript
// ตรวจว่า Prisma generated query ถูกต้อง
const prisma = new PrismaClient({ log: ['query'] });
// ดู console log ว่ามีกี่ queries จริงๆ
```

---

## ✅ Checklist

- [ ] **Step 441** — อ่าน EXPLAIN ANALYZE ได้: cost, actual time, buffers
- [ ] **Step 442** — เข้าใจว่า planner เลือก index หรือ seq scan อย่างไร
- [ ] **Step 443** — สร้าง Covering Index ด้วย INCLUDE และ verify Index Only Scan
- [ ] **Step 444** — สร้าง Partial Index สำหรับ active records
- [ ] **Step 445** — สร้าง Expression Index สำหรับ LOWER() search
- [ ] **Step 446** — ตรวจหา N+1 ใน Prisma และแก้ด้วย include
- [ ] **Step 447** — เปรียบเทียบ CTE vs subquery vs JOIN performance
- [ ] **Step 448** — ANALYZE table และปรับ statistics target
- [ ] **Step 449** — ใช้ pg_stat_statements หา top 10 slow queries
- [ ] **Step 450** — เขียน automated slow query analyzer script

---

## 🔗 References

- [PostgreSQL EXPLAIN Documentation](https://www.postgresql.org/docs/current/sql-explain.html)
- [pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html)
- [Use The Index, Luke!](https://use-the-index-luke.com/)
- [Prisma Performance Guide](https://www.prisma.io/docs/guides/performance-and-optimization)

---
*Part 045 | Road to 1,000,000 Users/Day | chuaikan.com*
