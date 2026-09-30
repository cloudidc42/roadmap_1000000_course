# Part 064: Database Query Profiling

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Advanced
> **Steps:** 631-640
> **เวลาโดยประมาณ:** 5 ชั่วโมง
> **Prerequisites:** Part 061-063 (Node.js Profiling, Memory, CPU)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- Deep dive กับ `EXPLAIN (ANALYZE, BUFFERS)`
- ใช้ `auto_explain` สำหรับ slow query logging อัตโนมัติ
- `pg_stat_statements`: หา top queries by total_time
- Index usage stats ด้วย `pg_stat_user_indexes`
- ตรวจจับ Sequential Scan
- หา Missing Index Recommendations
- Prisma query optimization
- Connection pool monitoring

---

## 📖 ทฤษฎีและแนวคิด

### Query Execution Plan

PostgreSQL ใช้ Query Planner เพื่อเลือก execution plan ที่ดีที่สุด:

```
Query: SELECT * FROM posts WHERE user_id = 123 AND status = 'published'

Possible Plans:
1. Seq Scan (อ่านทุก row)
   Cost: High if table is large
   
2. Index Scan (ใช้ index)
   Cost: Low if index exists on user_id

3. Index Only Scan (ข้อมูลอยู่ใน index ทั้งหมด)
   Cost: Lowest

4. Bitmap Index Scan (combine หลาย indexes)
   Cost: Medium, good for multiple conditions
```

### EXPLAIN Output คืออะไร

```
EXPLAIN ANALYZE SELECT * FROM posts WHERE user_id = 123;

Seq Scan on posts  (cost=0.00..1250.00 rows=50 width=512) (actual time=0.042..89.221 rows=48 loops=1)
  Filter: (user_id = 123)
  Rows Removed by Filter: 99952
  Buffers: shared hit=752 read=0
Planning Time: 0.5 ms
Execution Time: 89.5 ms

└─ cost=0.00..1250.00   = estimated start cost .. total cost
   rows=50               = estimated rows returned
   actual time=0.042..89.221 = actual start..end time (ms)
   Rows Removed by Filter = scan ทุก row แต่ filter ออกเยอะมาก!
```

---

## ⚙️ Environment Setup

```bash
# ติดตั้ง PostgreSQL 16 (ถ้ายังไม่มี)
sudo apt-get install -y postgresql-16

# เปิดใช้ pg_stat_statements extension
sudo -u postgres psql -c "
  ALTER SYSTEM SET shared_preload_libraries = 'pg_stat_statements,auto_explain';
  SELECT pg_reload_conf();
"

# หรือแก้ไข postgresql.conf
sudo vim /etc/postgresql/16/main/postgresql.conf

# Restart PostgreSQL
sudo systemctl restart postgresql

# สร้าง extension ใน database
sudo -u postgres psql -d chuaikan_db -c "
  CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
"
```

---

## 🛠️ Step-by-Step Implementation

### Step 631: EXPLAIN (ANALYZE, BUFFERS) Deep Dive

```sql
-- basic EXPLAIN
EXPLAIN SELECT * FROM posts WHERE user_id = 123;

-- EXPLAIN ANALYZE: รัน query จริงๆ และวัดเวลา
EXPLAIN ANALYZE SELECT * FROM posts WHERE user_id = 123;

-- EXPLAIN (ANALYZE, BUFFERS): รวม buffer/cache statistics
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) 
SELECT * FROM posts WHERE user_id = 123;

-- ตัวอย่าง query ที่มีปัญหา
EXPLAIN (ANALYZE, BUFFERS)
SELECT 
  u.id,
  u.name,
  COUNT(p.id) as post_count,
  SUM(p.view_count) as total_views
FROM users u
LEFT JOIN posts p ON p.user_id = u.id
WHERE u.created_at > NOW() - INTERVAL '30 days'
GROUP BY u.id, u.name
ORDER BY total_views DESC
LIMIT 20;
```

```bash
# Script สำหรับ analyze slow queries
cat << 'EOF' > /tmp/analyze-query.sql
-- Enable timing
\timing on

-- ดู query plan
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT p.id, p.title, u.name as author_name, COUNT(c.id) as comment_count
FROM posts p
JOIN users u ON u.id = p.user_id
LEFT JOIN comments c ON c.post_id = p.id
WHERE p.status = 'published'
  AND p.created_at > NOW() - INTERVAL '7 days'
GROUP BY p.id, p.title, u.name
ORDER BY comment_count DESC
LIMIT 10;
EOF

psql -d chuaikan_db -f /tmp/analyze-query.sql
```

### Step 632: การอ่านและตีความ EXPLAIN Output

```sql
-- ตัวอย่าง EXPLAIN output ที่มีปัญหา:
/*
Hash Join  (cost=2500.00..15000.00 rows=1000 width=256)
  (actual time=250.345..1234.567 rows=987 loops=1)
  Hash Cond: (p.user_id = u.id)
  ->  Seq Scan on posts p                         ← ❌ Sequential scan!
        (cost=0.00..8500.00 rows=1000 width=128)
        (actual time=0.012..456.789 rows=987 loops=1)
        Filter: ((status = 'published') AND (created_at > ...))
        Rows Removed by Filter: 499013             ← ❌ scan 500K rows filter เหลือ 987!
        Buffers: shared hit=6250 read=2500         ← 2500 disk reads!
  ->  Hash  (cost=1250.00..1250.00 rows=10000 width=128)
        (actual time=150.234..150.234 rows=10000 loops=1)
        ->  Seq Scan on users u
              Buffers: shared hit=750
Planning Time: 2.3 ms
Execution Time: 1234.9 ms                         ← ❌ 1.2 seconds!
*/

-- สิ่งที่ต้องมอง:
-- 1. "Seq Scan" บน table ขนาดใหญ่ = ❌ ต้องการ index
-- 2. "Rows Removed by Filter" สูง = index ไม่ selective
-- 3. "read=xxxx" ใน Buffers = disk I/O สูง = ต้องการ work_mem หรือ index
-- 4. actual time >> estimated time = statistics ล้าสมัย (ต้อง ANALYZE)

-- Fix: สร้าง index
CREATE INDEX CONCURRENTLY idx_posts_status_created_at 
ON posts(status, created_at DESC)
WHERE status = 'published';  -- Partial index สำหรับ published posts เท่านั้น

-- ตรวจสอบ index ถูกใช้:
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM posts 
WHERE status = 'published' 
  AND created_at > NOW() - INTERVAL '7 days';
/*
Index Scan using idx_posts_status_created_at on posts
  (actual time=0.034..12.456 rows=987 loops=1)  ← ✅ 12ms แทน 456ms!
  Buffers: shared hit=45 read=0                  ← ✅ 0 disk reads!
*/
```

### Step 633: auto_explain สำหรับ Slow Query Logging

```sql
-- ใน postgresql.conf หรือ ALTER SYSTEM
ALTER SYSTEM SET auto_explain.log_min_duration = '100';  -- log queries > 100ms
ALTER SYSTEM SET auto_explain.log_analyze = on;
ALTER SYSTEM SET auto_explain.log_buffers = on;
ALTER SYSTEM SET auto_explain.log_format = 'json';
ALTER SYSTEM SET auto_explain.log_nested_statements = off;
ALTER SYSTEM SET auto_explain.sample_rate = 1.0;  -- log ทุก slow query

-- Reload config
SELECT pg_reload_conf();

-- ทดสอบ auto_explain
LOAD 'auto_explain';
SET auto_explain.log_min_duration = 0;  -- log ทุก query (test only!)
SELECT count(*) FROM posts;

-- ดู slow query log
tail -f /var/log/postgresql/postgresql-16-main.log | grep -A 20 "duration:"
```

```bash
# parse-slow-queries.sh - Parse auto_explain JSON logs
#!/bin/bash

LOG_FILE="/var/log/postgresql/postgresql-16-main.log"

# ดึง queries ที่ใช้ Seq Scan
grep -A 5 "Seq Scan" $LOG_FILE | \
  grep -E "(duration|relation|Filter)" | \
  head -50

# ดึง top slow queries
grep "duration:" $LOG_FILE | \
  awk '{print $NF, $0}' | \
  sort -rn | \
  head -20 | \
  awk '{$1=""; print $0}'
```

### Step 634: pg_stat_statements: Top Queries

```sql
-- Enable pg_stat_statements
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- ดู top 10 queries โดย total_time
SELECT 
  round(total_exec_time::numeric, 2) AS total_time_ms,
  calls,
  round(mean_exec_time::numeric, 2) AS mean_time_ms,
  round(stddev_exec_time::numeric, 2) AS stddev_ms,
  round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) AS percentage,
  left(query, 100) AS query_snippet
FROM pg_stat_statements
WHERE query NOT LIKE '%pg_stat_statements%'
ORDER BY total_exec_time DESC
LIMIT 10;

-- ดู queries ที่ run บ่อยที่สุด
SELECT 
  calls,
  round(mean_exec_time::numeric, 2) AS mean_time_ms,
  round(total_exec_time::numeric, 2) AS total_time_ms,
  left(query, 100) AS query_snippet
FROM pg_stat_statements
WHERE calls > 1000
ORDER BY calls DESC
LIMIT 20;

-- ดู queries ที่ mean time สูง (อาจเป็น occasional slow queries)
SELECT 
  calls,
  round(mean_exec_time::numeric, 2) AS mean_ms,
  round(max_exec_time::numeric, 2) AS max_ms,
  round(stddev_exec_time::numeric, 2) AS stddev_ms,
  rows / calls AS avg_rows,
  left(query, 150) AS query
FROM pg_stat_statements
WHERE calls >= 10
  AND mean_exec_time > 50  -- mean > 50ms
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Reset statistics (ทำหลัง optimization)
SELECT pg_stat_statements_reset();
```

### Step 635: Index Usage Statistics

```sql
-- ดู index usage สำหรับ ทุก tables
SELECT 
  schemaname,
  tablename,
  indexname,
  idx_scan AS index_scans,
  idx_tup_read AS tuples_read,
  idx_tup_fetch AS tuples_fetched,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- หา indexes ที่ไม่ถูกใช้เลย (อาจลบได้)
SELECT 
  schemaname,
  tablename,
  indexname,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
  idx_scan AS times_used
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND schemaname NOT IN ('pg_catalog', 'pg_toast')
ORDER BY pg_relation_size(indexrelid) DESC;

-- หา tables ที่ Seq Scan มาก (ควรมี index)
SELECT 
  schemaname,
  relname AS tablename,
  seq_scan,
  idx_scan,
  n_live_tup AS row_count,
  CASE 
    WHEN seq_scan > 0 THEN round((idx_scan::float / (seq_scan + idx_scan) * 100)::numeric, 1)
    ELSE 100
  END AS index_usage_pct
FROM pg_stat_user_tables
WHERE seq_scan > 100  -- ทำ Seq Scan > 100 ครั้ง
ORDER BY seq_scan DESC;
```

### Step 636: Sequential Scan Detection

```sql
-- Real-time monitor สำหรับ Seq Scans
CREATE OR REPLACE VIEW v_seq_scan_alert AS
SELECT 
  relname AS table_name,
  seq_scan,
  seq_tup_read AS rows_read_by_seqscan,
  n_live_tup AS total_rows,
  CASE 
    WHEN n_live_tup > 0 
    THEN round((seq_tup_read::float / n_live_tup)::numeric, 2)
    ELSE 0 
  END AS seqscan_to_table_ratio
FROM pg_stat_user_tables
WHERE seq_scan > 50
  AND n_live_tup > 10000  -- เฉพาะ tables ขนาดใหญ่
ORDER BY seq_tup_read DESC;

SELECT * FROM v_seq_scan_alert;

-- ดู current running queries
SELECT 
  pid,
  now() - query_start AS duration,
  state,
  wait_event_type,
  wait_event,
  left(query, 100) AS query
FROM pg_stat_activity
WHERE state != 'idle'
  AND now() - query_start > INTERVAL '1 second'
ORDER BY duration DESC;

-- ดู lock contention
SELECT 
  blocked_locks.pid AS blocked_pid,
  blocked_activity.usename AS blocked_user,
  blocking_locks.pid AS blocking_pid,
  blocking_activity.usename AS blocking_user,
  blocked_activity.query AS blocked_statement,
  blocking_activity.query AS current_statement_in_blocking_process
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks 
  ON blocking_locks.locktype = blocked_locks.locktype
  AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
  AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
  AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
  AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
  AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
  AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
  AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
  AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
  AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
  AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

### Step 637: Missing Index Recommendation

```sql
-- Script หา indexes ที่ควรสร้าง
-- อ้างอิงจาก Seq Scans และ filter conditions
CREATE OR REPLACE FUNCTION suggest_missing_indexes()
RETURNS TABLE(
  table_name TEXT,
  seq_scans BIGINT,
  rows_per_scan NUMERIC,
  suggestion TEXT
) AS $$
BEGIN
  RETURN QUERY
  SELECT 
    t.relname::TEXT,
    t.seq_scan,
    ROUND((t.seq_tup_read::NUMERIC / NULLIF(t.seq_scan, 0)), 0),
    'Consider adding index. High seq scan rate: ' || t.seq_scan || ' scans on ' || t.n_live_tup || ' rows'
  FROM pg_stat_user_tables t
  WHERE t.seq_scan > 100
    AND t.n_live_tup > 5000
    AND NOT EXISTS (
      SELECT 1 FROM pg_stat_user_indexes i 
      WHERE i.relname = t.relname 
        AND i.idx_scan > 0
    )
  ORDER BY t.seq_tup_read DESC;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM suggest_missing_indexes();

-- ดู index bloat (indexes ที่ใหญ่กว่าที่ควร)
SELECT
  current_database(),
  schemaname || '.' || tablename AS table,
  pg_size_pretty(bs * (relpages)::bigint) AS "real_size",
  pg_size_pretty(bs * ceil(reltuples / ((bs - pa - 8) / (4 + nulldatahdrwid + ma * ceil((nulldatahdrwid + 1) / ma::float) + datahdrwid))) ) AS "estimated_size",
  tablename,
  indexname
FROM (
  SELECT
    ma, bs, schemaname, tablename, indexname,
    relpages, reltuples, pa,
    nulldatahdrwid,
    datahdrwid
  FROM (
    SELECT
      current_setting('block_size')::numeric AS bs,
      23 AS hdrsize,
      4 AS ma,
      1 AS pa,
      s.schemaname,
      s.tablename,
      s.indexname,
      c.relpages,
      c.reltuples,
      8 AS nulldatahdrwid,
      6 AS datahdrwid
    FROM pg_stat_user_indexes s
    JOIN pg_class c ON c.relname = s.indexname
    WHERE c.relpages > 100
  ) AS foo
) AS bar
LIMIT 20;
```

### Step 638: Prisma Query Optimization

```typescript
// prisma-optimization.ts

import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient({
  log: [
    { level: 'query', emit: 'event' },
    { level: 'slow', emit: 'event' },  // Prisma 5.x
    { level: 'warn', emit: 'stdout' },
    { level: 'error', emit: 'stdout' },
  ],
});

// Log slow queries
prisma.$on('query', (event) => {
  if (event.duration > 100) {  // queries > 100ms
    console.warn('SLOW QUERY:', {
      query: event.query,
      params: event.params,
      duration: event.duration,
    });
  }
});

// ❌ BAD: SELECT * (loads all columns)
async function getPostsBad() {
  return prisma.post.findMany({
    where: { status: 'published' },
    include: {
      user: true,     // loads entire user object
      comments: true, // loads all comments!
    },
  });
}

// ✅ GOOD: Select only needed fields
async function getPostsGood() {
  return prisma.post.findMany({
    where: { 
      status: 'published',
      createdAt: {
        gte: new Date(Date.now() - 7 * 24 * 60 * 60 * 1000)
      }
    },
    select: {
      id: true,
      title: true,
      summary: true,
      createdAt: true,
      user: {
        select: {
          id: true,
          name: true,
          avatarUrl: true,
        }
      },
      _count: {
        select: { comments: true }  // COUNT แทนการ load ทุก comments
      }
    },
    orderBy: { createdAt: 'desc' },
    take: 20,
    skip: 0,
  });
}

// ✅ GOOD: Cursor-based pagination (ดีกว่า offset สำหรับ large datasets)
async function getPostsCursorPagination(cursor?: string, take = 20) {
  return prisma.post.findMany({
    where: { status: 'published' },
    select: {
      id: true,
      title: true,
      createdAt: true,
    },
    take,
    skip: cursor ? 1 : 0,  // skip cursor itself
    cursor: cursor ? { id: cursor } : undefined,
    orderBy: { createdAt: 'desc' },
  });
}

// ✅ GOOD: Batch requests ด้วย $transaction
async function createPostWithNotification(data: {
  title: string;
  content: string;
  userId: string;
}) {
  // ทั้งหมดใน transaction เดียว
  return prisma.$transaction(async (tx) => {
    const post = await tx.post.create({
      data: {
        title: data.title,
        content: data.content,
        userId: data.userId,
        status: 'published',
      },
    });
    
    await tx.notification.create({
      data: {
        type: 'NEW_POST',
        userId: data.userId,
        postId: post.id,
      },
    });
    
    return post;
  });
}

// ✅ GOOD: Raw query สำหรับ complex analytics
async function getTopAuthors() {
  return prisma.$queryRaw`
    SELECT 
      u.id,
      u.name,
      COUNT(p.id)::int AS post_count,
      SUM(p.view_count)::int AS total_views,
      AVG(p.view_count)::float AS avg_views
    FROM users u
    JOIN posts p ON p.user_id = u.id
    WHERE p.status = 'published'
      AND p.created_at > NOW() - INTERVAL '30 days'
    GROUP BY u.id, u.name
    HAVING COUNT(p.id) >= 3
    ORDER BY total_views DESC
    LIMIT 10
  `;
}
```

### Step 639: Connection Pool Monitoring

```javascript
// connection-pool-monitor.js
const { Pool } = require('pg');
const client = require('prom-client');

// Prometheus metrics สำหรับ connection pool
const poolTotalConnections = new client.Gauge({
  name: 'pg_pool_total_connections',
  help: 'Total connections in pool',
  labelNames: ['pool_name']
});

const poolIdleConnections = new client.Gauge({
  name: 'pg_pool_idle_connections',
  help: 'Idle connections in pool',
  labelNames: ['pool_name']
});

const poolWaitingCount = new client.Gauge({
  name: 'pg_pool_waiting_count',
  help: 'Queries waiting for connection',
  labelNames: ['pool_name']
});

// สร้าง monitored pool
function createMonitoredPool(name, config) {
  const pool = new Pool({
    host: process.env.DB_HOST || 'localhost',
    port: 5432,
    database: process.env.DB_NAME || 'chuaikan_db',
    user: process.env.DB_USER || 'postgres',
    password: process.env.DB_PASSWORD,
    max: config.max || 20,          // maximum connections
    min: config.min || 2,           // minimum idle connections
    idleTimeoutMillis: 30000,       // close idle connections after 30s
    connectionTimeoutMillis: 5000,  // fail if can't connect in 5s
    ...config
  });
  
  pool.on('connect', (client) => {
    console.log(`[${name}] New connection established`);
  });
  
  pool.on('acquire', (client) => {
    // connection was acquired from pool
  });
  
  pool.on('remove', (client) => {
    console.log(`[${name}] Connection removed from pool`);
  });
  
  pool.on('error', (err, client) => {
    console.error(`[${name}] Pool error:`, err.message);
  });
  
  // Update metrics
  setInterval(() => {
    poolTotalConnections.set({ pool_name: name }, pool.totalCount);
    poolIdleConnections.set({ pool_name: name }, pool.idleCount);
    poolWaitingCount.set({ pool_name: name }, pool.waitingCount);
  }, 5000);
  
  return pool;
}

// ดู connection pool stats จาก PostgreSQL
async function getPoolStats(pool) {
  const result = await pool.query(`
    SELECT 
      count(*) FILTER (WHERE state = 'active') AS active,
      count(*) FILTER (WHERE state = 'idle') AS idle,
      count(*) FILTER (WHERE state = 'idle in transaction') AS idle_in_transaction,
      count(*) FILTER (WHERE wait_event_type = 'Lock') AS waiting_on_lock,
      count(*) AS total
    FROM pg_stat_activity
    WHERE datname = current_database()
      AND pid != pg_backend_pid()
  `);
  
  return result.rows[0];
}

// Alert ถ้า pool เต็ม
async function checkPoolHealth(pool, maxConnections) {
  const stats = await getPoolStats(pool);
  const usage = (parseInt(stats.total) / maxConnections) * 100;
  
  if (usage > 80) {
    console.error(`⚠️  Connection pool high usage: ${usage.toFixed(1)}%`, stats);
  }
  
  if (parseInt(stats.idle_in_transaction) > 5) {
    console.warn('⚠️  Too many idle-in-transaction connections!', stats);
  }
}
```

### Step 640: Automated Query Performance Report

```bash
#!/bin/bash
# db-performance-report.sh - สร้าง performance report

DB_NAME="${DB_NAME:-chuaikan_db}"
REPORT_FILE="/tmp/db-report-$(date +%Y%m%d).txt"

echo "=== Database Performance Report ===" > $REPORT_FILE
echo "Generated: $(date)" >> $REPORT_FILE
echo "" >> $REPORT_FILE

# 1. Top 10 slow queries
echo "### TOP 10 SLOW QUERIES (by total time) ###" >> $REPORT_FILE
psql -d $DB_NAME -c "
SELECT 
  round(total_exec_time::numeric/1000, 2) AS total_sec,
  calls,
  round(mean_exec_time::numeric, 2) AS mean_ms,
  round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 1) AS pct,
  left(query, 80) AS query
FROM pg_stat_statements
WHERE query NOT LIKE '%pg_stat%'
ORDER BY total_exec_time DESC
LIMIT 10;
" >> $REPORT_FILE 2>&1

# 2. Tables with high Seq Scan
echo "" >> $REPORT_FILE
echo "### TABLES WITH HIGH SEQ SCAN ###" >> $REPORT_FILE
psql -d $DB_NAME -c "
SELECT relname, seq_scan, n_live_tup, 
  pg_size_pretty(pg_total_relation_size(relid)) as size
FROM pg_stat_user_tables
WHERE seq_scan > 100 AND n_live_tup > 10000
ORDER BY seq_scan DESC LIMIT 10;
" >> $REPORT_FILE 2>&1

# 3. Unused indexes
echo "" >> $REPORT_FILE
echo "### UNUSED INDEXES (0 scans) ###" >> $REPORT_FILE
psql -d $DB_NAME -c "
SELECT tablename, indexname,
  pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC LIMIT 10;
" >> $REPORT_FILE 2>&1

# ส่ง report ทาง email หรือ Slack
cat $REPORT_FILE

# Slack notification
if [ -n "$SLACK_WEBHOOK_URL" ]; then
  curl -X POST -H 'Content-type: application/json' \
    --data "{\"text\": \"DB Performance Report ready: \`\`\`$(head -30 $REPORT_FILE)\`\`\`\"}" \
    $SLACK_WEBHOOK_URL
fi
```

---

## 🔧 Configuration Files

### postgresql.conf (Performance Tuning)

```ini
# postgresql.conf - Performance settings

# Memory
shared_buffers = 4GB              # 25% of RAM
effective_cache_size = 12GB       # 75% of RAM  
work_mem = 64MB                   # per sort/hash operation
maintenance_work_mem = 1GB        # for VACUUM, CREATE INDEX

# Planner
random_page_cost = 1.1            # SSD: 1.1, HDD: 4.0
effective_io_concurrency = 200    # SSD: 200, HDD: 2

# WAL & Checkpoints
wal_buffers = 64MB
checkpoint_completion_target = 0.9
max_wal_size = 4GB

# Statistics
track_io_timing = on
track_functions = all

# pg_stat_statements
shared_preload_libraries = 'pg_stat_statements,auto_explain'
pg_stat_statements.max = 10000
pg_stat_statements.track = all

# auto_explain
auto_explain.log_min_duration = 200   # log queries > 200ms
auto_explain.log_analyze = on
auto_explain.log_buffers = on
auto_explain.log_format = 'json'
auto_explain.sample_rate = 1.0
```

---

## 🧪 Testing

```bash
# Test 1: เปรียบเทียบ query ก่อน/หลัง index
# ก่อนสร้าง index
EXPLAIN ANALYZE SELECT * FROM posts WHERE user_id = 100;
# บันทึก execution time

# สร้าง index
CREATE INDEX CONCURRENTLY idx_posts_user_id ON posts(user_id);

# หลังสร้าง index
EXPLAIN ANALYZE SELECT * FROM posts WHERE user_id = 100;
# ควรเร็วขึ้นมาก

# Test 2: pg_stat_statements reset และ measure
SELECT pg_stat_statements_reset();
# รัน load test
autocannon -d 60 -c 50 http://localhost:3000/api/posts
# ดู top queries
SELECT left(query,80), calls, round(mean_exec_time::numeric,2) as mean_ms
FROM pg_stat_statements
ORDER BY total_exec_time DESC LIMIT 10;
```

---

## ❌ Common Errors & Solutions

### Error 1: pg_stat_statements ไม่มีข้อมูล

```sql
-- ตรวจสอบว่า extension ถูก enable
SELECT * FROM pg_extension WHERE extname = 'pg_stat_statements';

-- ถ้าไม่มี ให้ create
CREATE EXTENSION pg_stat_statements;

-- ตรวจสอบ shared_preload_libraries
SHOW shared_preload_libraries;
-- ต้องมี 'pg_stat_statements' - ถ้าไม่มีต้อง restart PostgreSQL
```

### Error 2: Index ไม่ถูกใช้แม้สร้างแล้ว

```sql
-- PostgreSQL อาจเลือก Seq Scan ถ้า table เล็กมาก
-- หรือถ้า statistics ล้าสมัย

-- อัพเดท statistics
ANALYZE posts;

-- บังคับ planner ใช้ index (test เท่านั้น!)
SET enable_seqscan = off;
EXPLAIN SELECT * FROM posts WHERE user_id = 100;
SET enable_seqscan = on;

-- ถ้า query เลือก Seq Scan แม้มี index อาจเป็นเพราะ:
-- 1. Index ไม่ selective พอ (column มีค่าซ้ำกันเยอะ)
-- 2. Table เล็กมาก Seq Scan เร็วกว่า
-- 3. Query ดึงข้อมูลมากกว่า 20% ของ table
```

---

## ✅ Checklist

- [ ] Enable `pg_stat_statements` extension
- [ ] ตั้งค่า `auto_explain` สำหรับ slow query logging (> 200ms)
- [ ] รัน weekly slow query report script
- [ ] ตรวจสอบ indexes ที่ไม่ถูกใช้ (idx_scan = 0) และพิจารณาลบ
- [ ] ตรวจสอบ tables ที่มี high Seq Scan และสร้าง index
- [ ] ปรับ `postgresql.conf` ตาม RAM ของ server
- [ ] ตั้งค่า connection pool monitoring (Prometheus)
- [ ] แก้ N+1 queries ทั้งหมดใน codebase
- [ ] ใช้ cursor-based pagination แทน offset
- [ ] ตั้งค่า alert สำหรับ slow queries > 500ms

---

## 🔗 References

- [PostgreSQL EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
- [pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html)
- [auto_explain](https://www.postgresql.org/docs/current/auto-explain.html)
- [Prisma Performance](https://www.prisma.io/docs/guides/performance-and-optimization)
- [pganalyze](https://pganalyze.com/) - PostgreSQL performance monitoring
- [Use The Index, Luke!](https://use-the-index-luke.com/)

---

*Part 064 | Road to 1,000,000 Users/Day | chuaikan.com*
