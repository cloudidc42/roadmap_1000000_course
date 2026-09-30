# Part 049: Data Migration at Scale
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 481–490
> **เวลาโดยประมาณ:** 4 ชั่วโมง
> **Prerequisites:** Part 048 (DB per Service), PostgreSQL Advanced

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- Online schema changes: pg_repack, ALTER TABLE อย่างปลอดภัย
- เพิ่ม NOT NULL column ในตารางขนาดใหญ่ (multi-step)
- Large data migration ด้วย batching
- Zero-downtime index creation (CONCURRENTLY)
- Shadow table pattern สำหรับ table rebuild
- Migration ข้ามระหว่าง services
- Rollback strategy สำหรับ migration ที่ล้มเหลว
- Data validation หลัง migration

---

## 📖 ทฤษฎีและแนวคิด

### Step 481 — ปัญหาของ Schema Changes ใน Production

```
ปัญหาใน PostgreSQL:
ALTER TABLE users ADD COLUMN verified_at TIMESTAMPTZ NOT NULL DEFAULT NOW();

บน Production table 50M rows:
→ Lock table นาน 10-30 นาที!
→ ทุก INSERT/UPDATE/SELECT รอ lock
→ Application timeout ทั้งหมด

วิธีแก้: multi-step migration
Step 1: ADD COLUMN ก่อน (nullable, no default)
Step 2: backfill ด้วย batch UPDATE
Step 3: ADD DEFAULT ทีหลัง
Step 4: SET NOT NULL ทีหลัง (รอ pg 14+ CHECK CONSTRAINT)
```

---

## ⚙️ Environment Setup

```bash
# ติดตั้ง pg_repack
sudo apt-get install -y postgresql-16-repack

# ตรวจสอบ
pg_repack --version

# ติดตั้งใน database
sudo -u postgres psql -d chuaikan_db -c \
  "CREATE EXTENSION pg_repack;"
```

---

## 🛠️ Step-by-Step Implementation

### Step 482 — เพิ่ม NOT NULL Column ในตารางขนาดใหญ่

```sql
-- ❌ วิธีผิด: Lock table นาน
ALTER TABLE users
    ADD COLUMN email_verified_at TIMESTAMPTZ NOT NULL DEFAULT NOW();
-- Lock table ≈ 30 นาทีสำหรับ 50M rows!

-- ✅ วิธีถูก: Multi-step approach
-- สถานการณ์: เพิ่ม email_verified_at ใน users table (50M rows)

-- Step 1: ADD COLUMN แบบ nullable (เร็ว, ไม่ lock นาน)
ALTER TABLE users ADD COLUMN email_verified_at TIMESTAMPTZ;
-- ใช้เวลาไม่กี่วินาที เพราะ PostgreSQL เพิ่มแค่ metadata

-- Step 2: Backfill ทีละ batch (ไม่ lock ทั้งตาราง)
DO $$
DECLARE
    batch_size  INTEGER := 10000;
    last_id     BIGINT  := 0;
    max_id      BIGINT;
    affected    INTEGER;
BEGIN
    SELECT MAX(id) INTO max_id FROM users;

    LOOP
        UPDATE users
        SET email_verified_at = created_at  -- ค่า default เป็น created_at
        WHERE id > last_id
          AND id <= last_id + batch_size
          AND email_verified_at IS NULL;

        GET DIAGNOSTICS affected = ROW_COUNT;
        EXIT WHEN affected = 0;

        last_id := last_id + batch_size;
        RAISE NOTICE 'Backfilled up to id %, affected %', last_id, affected;

        -- หยุดพักไม่ให้โหลด DB สูงเกิน
        PERFORM pg_sleep(0.05);
    END LOOP;

    RAISE NOTICE 'Backfill complete';
END $$;

-- Step 3: เพิ่ม DEFAULT (PostgreSQL 11+: เร็ว ไม่ rewrite table)
ALTER TABLE users
    ALTER COLUMN email_verified_at SET DEFAULT NOW();

-- Step 4: เพิ่ม NOT NULL constraint (PostgreSQL 14+: ไม่ scan ทั้งตาราง)
-- ก่อน: ADD CHECK CONSTRAINT NOT VALID (เร็ว)
ALTER TABLE users
    ADD CONSTRAINT users_email_verified_at_not_null
    CHECK (email_verified_at IS NOT NULL) NOT VALID;

-- VALIDATE ต่างหาก (ใช้ Share Lock แทน Exclusive Lock)
ALTER TABLE users
    VALIDATE CONSTRAINT users_email_verified_at_not_null;

-- สุดท้าย: SET NOT NULL (อ้างอิง validated constraint)
ALTER TABLE users
    ALTER COLUMN email_verified_at SET NOT NULL;

-- ลบ CHECK CONSTRAINT ที่ไม่ต้องการแล้ว
ALTER TABLE users
    DROP CONSTRAINT users_email_verified_at_not_null;
```

### Step 483 — Zero-downtime Index Creation

```sql
-- ❌ วิธีผิด: Lock table ขณะสร้าง index
CREATE INDEX ON posts(user_id);
-- ใน production: lock ทั้งตาราง ทำให้ read/write รอ

-- ✅ วิธีถูก: CONCURRENTLY
CREATE INDEX CONCURRENTLY idx_posts_user_id
    ON posts(user_id);
-- ใช้เวลานานกว่า แต่ไม่ lock table

-- สำหรับ Unique Index
CREATE UNIQUE INDEX CONCURRENTLY idx_posts_slug_unique
    ON posts(slug)
    WHERE is_published = true;

-- ตรวจสอบ progress ขณะสร้าง index
SELECT
    phase,
    blocks_done,
    blocks_total,
    ROUND(100.0 * blocks_done / NULLIF(blocks_total, 0), 1) AS pct_done
FROM pg_stat_progress_create_index
WHERE relid = 'posts'::regclass;

-- กรณีที่ CONCURRENTLY index สร้างไม่สำเร็จ (invalid index)
SELECT indexname, indisvalid
FROM pg_indexes pi
JOIN pg_index i ON i.indexrelid = pi.indexname::regclass
WHERE tablename = 'posts';

-- ลบ invalid index แล้วสร้างใหม่
DROP INDEX CONCURRENTLY IF EXISTS idx_posts_user_id;
```

### Step 484 — Large Data Migration ด้วย Batching

```typescript
// scripts/migrations/backfill-province-id.ts
// Migration: คำนวณ province_id สำหรับทุก user จาก lat/lng

import { prisma } from '../../lib/db/client';

const BATCH_SIZE = 1000;
const DELAY_MS = 50;

async function backfillProvinceIds() {
  console.log('[Migration] Starting province_id backfill...');
  const startTime = Date.now();

  let processed = 0;
  let cursor = 0n; // BIGINT cursor
  let hasMore = true;

  while (hasMore) {
    // Fetch batch
    const users = await prisma.$queryRaw<{ id: bigint; lat: number; lng: number }[]>`
      SELECT id, lat, lng
      FROM users
      WHERE id > ${cursor}
        AND province_id IS NULL
        AND lat IS NOT NULL
        AND lng IS NOT NULL
      ORDER BY id ASC
      LIMIT ${BATCH_SIZE}
    `;

    if (users.length === 0) {
      hasMore = false;
      break;
    }

    // Process batch: ใช้ PostGIS หา province
    await prisma.$executeRaw`
      UPDATE users u
      SET province_id = p.id
      FROM thai_provinces p
      WHERE ST_Contains(p.boundary, ST_MakePoint(u.lng, u.lat))
        AND u.id = ANY(${users.map(u => u.id)}::bigint[])
        AND u.province_id IS NULL
    `;

    cursor = users[users.length - 1].id;
    processed += users.length;

    // Progress report
    if (processed % 10000 === 0) {
      const elapsed = ((Date.now() - startTime) / 1000).toFixed(1);
      console.log(`[Migration] Processed ${processed} users in ${elapsed}s (last id: ${cursor})`);
    }

    // Throttle ไม่ให้ database load สูงเกิน
    await new Promise(resolve => setTimeout(resolve, DELAY_MS));
  }

  const elapsed = ((Date.now() - startTime) / 1000).toFixed(1);
  console.log(`[Migration] Complete: ${processed} users in ${elapsed}s`);
}

backfillProvinceIds().catch(console.error);
```

```bash
# รัน migration แบบมี progress tracking
npx ts-node scripts/migrations/backfill-province-id.ts 2>&1 | \
  tee /var/log/migration-$(date +%Y%m%d-%H%M%S).log

# Monitor อีก terminal
watch -n 5 "psql -U postgres -d chuaikan_db -c \
  \"SELECT COUNT(*) FROM users WHERE province_id IS NULL;\""
```

### Step 485 — Shadow Table Pattern (Zero-downtime Table Rebuild)

```sql
-- ใช้กรณี: ต้อง rewrite table ทั้งหมด เช่น เปลี่ยน data type

-- ตัวอย่าง: เปลี่ยน user_id จาก INTEGER เป็น BIGINT
-- (ทำบนตาราง posts)

-- Step 1: สร้าง shadow table ใหม่
CREATE TABLE posts_v2 (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT NOT NULL,          -- เปลี่ยนจาก INTEGER เป็น BIGINT
    title       VARCHAR(500),
    content     TEXT,
    created_at  TIMESTAMPTZ DEFAULT NOW()
    -- ... columns อื่นๆ
);

-- Step 2: Copy ข้อมูลจาก posts ไป posts_v2 (batch)
INSERT INTO posts_v2
SELECT id, user_id::bigint, title, content, created_at
FROM posts
WHERE id <= 1000000;  -- batch แรก

-- Step 3: สร้าง trigger ใน posts เพื่อ sync changes ไป posts_v2
CREATE OR REPLACE FUNCTION sync_posts_to_v2()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO posts_v2 VALUES (NEW.id, NEW.user_id::bigint, NEW.title, NEW.content, NEW.created_at);
    ELSIF TG_OP = 'UPDATE' THEN
        UPDATE posts_v2 SET
            user_id = NEW.user_id::bigint,
            title = NEW.title,
            content = NEW.content
        WHERE id = NEW.id;
    ELSIF TG_OP = 'DELETE' THEN
        DELETE FROM posts_v2 WHERE id = OLD.id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_posts_to_v2
    AFTER INSERT OR UPDATE OR DELETE ON posts
    FOR EACH ROW EXECUTE FUNCTION sync_posts_to_v2();

-- Step 4: Copy ข้อมูลที่เหลือ (ระหว่างนี้ trigger sync changes ใหม่)
INSERT INTO posts_v2
SELECT id, user_id::bigint, title, content, created_at
FROM posts
WHERE id > 1000000;

-- Step 5: ตรวจสอบว่าข้อมูล consistent
SELECT
    (SELECT COUNT(*) FROM posts) AS original_count,
    (SELECT COUNT(*) FROM posts_v2) AS v2_count,
    (SELECT MAX(id) FROM posts) AS max_original,
    (SELECT MAX(id) FROM posts_v2) AS max_v2;

-- Step 6: Rename tables (เร็วมาก)
BEGIN;
ALTER TABLE posts RENAME TO posts_old;
ALTER TABLE posts_v2 RENAME TO posts;
DROP TRIGGER IF EXISTS trg_sync_posts_to_v2 ON posts_old;
COMMIT;

-- Step 7: ลบตารางเก่า (หลังจากมั่นใจ 100%)
-- DROP TABLE posts_old;
```

### Step 486 — pg_repack: Rebuild Table Online

```bash
# pg_repack สำหรับ reclaim space และ reorder table
# ทำงานโดยไม่ lock table (ยกเว้น ACCESS EXCLUSIVE lock สั้นๆ ตอนสุดท้าย)

# ดู table bloat ก่อน
sudo -u postgres psql -d chuaikan_db << 'EOF'
SELECT
    tablename,
    pg_size_pretty(pg_total_relation_size(tablename::regclass)) AS total_size,
    pg_size_pretty(pg_relation_size(tablename::regclass)) AS table_size,
    n_dead_tup,
    n_live_tup,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup, 0), 1) AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 100000
ORDER BY n_dead_tup DESC;
EOF

# Repack table ที่ bloated
pg_repack \
  --host localhost \
  --username postgres \
  --dbname chuaikan_db \
  --table posts \
  --no-kill-backend \
  --wait-timeout 60

# Repack ทุก table ใน database
pg_repack \
  --host localhost \
  --username postgres \
  --dbname chuaikan_db \
  --no-kill-backend

# ตรวจสอบขนาดหลัง repack
sudo -u postgres psql -d chuaikan_db -c \
  "SELECT pg_size_pretty(pg_total_relation_size('posts'::regclass));"
```

### Step 487 — Migration ข้าม Services

```typescript
// scripts/migrations/move-flood-zones-to-content-db.ts

import { sosDb, contentDb } from '../../lib/db/databases';

async function migrateFloodZones() {
  console.log('[Migration] Moving flood_zones from sos_db to content_db');

  // Step 1: Export จาก SOS DB
  const floodZones = await sosDb.$queryRaw<any[]>`
    SELECT
      id, name, risk_level,
      ST_AsGeoJSON(boundary) AS boundary_geojson,
      created_at
    FROM flood_zones
    ORDER BY id
  `;

  console.log(`Found ${floodZones.length} flood zones to migrate`);

  // Step 2: Import ไปยัง Content DB
  let migrated = 0;
  const BATCH_SIZE = 100;

  for (let i = 0; i < floodZones.length; i += BATCH_SIZE) {
    const batch = floodZones.slice(i, i + BATCH_SIZE);

    await contentDb.$transaction(
      batch.map(zone =>
        contentDb.$executeRaw`
          INSERT INTO flood_zones (id, name, risk_level, boundary, created_at)
          VALUES (
            ${zone.id},
            ${zone.name},
            ${zone.risk_level},
            ST_GeomFromGeoJSON(${zone.boundary_geojson}),
            ${zone.created_at}
          )
          ON CONFLICT (id) DO UPDATE SET
            name = EXCLUDED.name,
            risk_level = EXCLUDED.risk_level,
            boundary = EXCLUDED.boundary
        `
      )
    );

    migrated += batch.length;
    console.log(`Migrated ${migrated}/${floodZones.length}`);
  }

  // Step 3: Validate
  const [sosCount, contentCount] = await Promise.all([
    sosDb.$queryRaw<{ count: string }[]>`SELECT COUNT(*) FROM flood_zones`,
    contentDb.$queryRaw<{ count: string }[]>`SELECT COUNT(*) FROM flood_zones`,
  ]);

  const sosTotal = parseInt(sosCount[0].count);
  const contentTotal = parseInt(contentCount[0].count);

  if (sosTotal !== contentTotal) {
    throw new Error(`Count mismatch: sos=${sosTotal}, content=${contentTotal}`);
  }

  console.log(`[Migration] Success: ${contentTotal} flood zones migrated`);
}

migrateFloodZones().catch(console.error);
```

### Step 488 — Rollback Strategy

```sql
-- === Rollback Strategy ===

-- 1. Before-migration snapshot (สำหรับ small tables)
CREATE TABLE posts_backup_20240115 AS
TABLE posts;
-- หรือ
pg_dump -t posts chuaikan_db > /backup/posts_20240115.sql

-- 2. Transaction-based rollback (DDL ใน transaction)
BEGIN;

ALTER TABLE users ADD COLUMN tier VARCHAR(20) DEFAULT 'free';

-- ทดสอบ
UPDATE users SET tier = 'premium' WHERE id IN (1, 2, 3);
SELECT tier FROM users WHERE id IN (1, 2, 3);

-- ถ้าพบปัญหา:
ROLLBACK;
-- ถ้าโอเค:
-- COMMIT;

-- 3. Feature flag สำหรับ application-level rollback
```

```typescript
// lib/feature-flags.ts
const FEATURES = {
  USE_NEW_TIER_SYSTEM: process.env.FEATURE_NEW_TIER === 'true',
  USE_BIGINT_USER_ID: process.env.FEATURE_BIGINT_ID === 'true',
};

// ใช้ feature flag ใน code
export async function getUserPosts(userId: string) {
  if (FEATURES.USE_BIGINT_USER_ID) {
    return prisma.$queryRaw`
      SELECT * FROM posts WHERE user_id = ${BigInt(userId)}
    `;
  } else {
    return prisma.$queryRaw`
      SELECT * FROM posts WHERE user_id = ${parseInt(userId)}
    `;
  }
}

// Rollback = เปลี่ยน ENV variable แล้ว restart
// FEATURE_BIGINT_ID=false → กลับไปใช้ integer
```

### Step 489 — Data Validation หลัง Migration

```typescript
// scripts/validate-migration.ts

interface ValidationResult {
  check: string;
  passed: boolean;
  details: string;
}

async function validateMigration(): Promise<void> {
  const results: ValidationResult[] = [];

  // Check 1: Row counts match
  const [originalCount, migratedCount] = await Promise.all([
    prisma.$queryRaw<{ count: bigint }[]>`
      SELECT COUNT(*) AS count FROM posts_old
    `,
    prisma.$queryRaw<{ count: bigint }[]>`
      SELECT COUNT(*) AS count FROM posts
    `,
  ]);

  const countMatch = originalCount[0].count === migratedCount[0].count;
  results.push({
    check: 'Row count matches',
    passed: countMatch,
    details: `original: ${originalCount[0].count}, migrated: ${migratedCount[0].count}`,
  });

  // Check 2: No null values in NOT NULL columns
  const nullCheck = await prisma.$queryRaw<{ null_count: bigint }[]>`
    SELECT COUNT(*) AS null_count FROM posts
    WHERE user_id IS NULL OR title IS NULL OR created_at IS NULL
  `;

  const noNulls = nullCheck[0].null_count === 0n;
  results.push({
    check: 'No nulls in required columns',
    passed: noNulls,
    details: `null rows: ${nullCheck[0].null_count}`,
  });

  // Check 3: Spot check random rows
  const spotCheck = await prisma.$queryRaw<any[]>`
    SELECT p.id, p.user_id, p.title, p.created_at,
           o.user_id AS old_user_id, o.title AS old_title
    FROM posts p
    JOIN posts_old o ON o.id = p.id
    WHERE p.user_id != o.user_id::bigint
       OR p.title != o.title
       OR p.created_at != o.created_at
    LIMIT 10
  `;

  const dataIntegrity = spotCheck.length === 0;
  results.push({
    check: 'Data integrity (spot check)',
    passed: dataIntegrity,
    details: dataIntegrity ? 'All spot checks passed' :
      `Found ${spotCheck.length} mismatches`,
  });

  // Check 4: Indexes exist
  const indexes = await prisma.$queryRaw<{ indexname: string }[]>`
    SELECT indexname FROM pg_indexes
    WHERE tablename = 'posts'
      AND indexname IN ('idx_posts_user_id', 'posts_pkey')
  `;

  const hasIndexes = indexes.length >= 2;
  results.push({
    check: 'Required indexes exist',
    passed: hasIndexes,
    details: `Found indexes: ${indexes.map(i => i.indexname).join(', ')}`,
  });

  // Report
  console.log('\n=== Migration Validation Report ===');
  let allPassed = true;
  for (const result of results) {
    const status = result.passed ? '✅ PASS' : '❌ FAIL';
    console.log(`${status} | ${result.check}: ${result.details}`);
    if (!result.passed) allPassed = false;
  }

  if (!allPassed) {
    console.error('\n⚠️  VALIDATION FAILED - Consider rollback');
    process.exit(1);
  } else {
    console.log('\n✅ All validations passed');
  }
}

validateMigration().catch(console.error);
```

### Step 490 — Automated Migration Pipeline

```bash
#!/bin/bash
# scripts/run-migration.sh
# Complete migration pipeline with validation

set -euo pipefail

MIGRATION_NAME="${1:-unknown}"
DB_URL="${DATABASE_URL}"
LOG_FILE="/var/log/migrations/$(date +%Y%m%d-%H%M%S)-${MIGRATION_NAME}.log"

mkdir -p /var/log/migrations

echo "[$(date)] Starting migration: ${MIGRATION_NAME}" | tee -a "$LOG_FILE"

# 1. Pre-migration backup
echo "[$(date)] Creating backup..." | tee -a "$LOG_FILE"
pg_dump \
  --host="${DB_HOST}" \
  --username="${DB_USER}" \
  --dbname="${DB_NAME}" \
  --schema-only \
  --file="/backup/schema_before_${MIGRATION_NAME}_$(date +%Y%m%d).sql"

# 2. Run migration
echo "[$(date)] Running migration..." | tee -a "$LOG_FILE"
npx prisma migrate deploy 2>&1 | tee -a "$LOG_FILE"

# 3. Run backfill if needed
if [ -f "scripts/migrations/${MIGRATION_NAME}-backfill.ts" ]; then
  echo "[$(date)] Running backfill..." | tee -a "$LOG_FILE"
  npx ts-node "scripts/migrations/${MIGRATION_NAME}-backfill.ts" 2>&1 | tee -a "$LOG_FILE"
fi

# 4. Validate
echo "[$(date)] Validating..." | tee -a "$LOG_FILE"
npx ts-node scripts/validate-migration.ts 2>&1 | tee -a "$LOG_FILE"

echo "[$(date)] Migration complete!" | tee -a "$LOG_FILE"
```

---

## 🔧 Configuration Files

```sql
-- Migration tracking table
CREATE TABLE IF NOT EXISTS migration_log (
    id              SERIAL PRIMARY KEY,
    migration_name  VARCHAR(200) NOT NULL,
    started_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at    TIMESTAMPTZ,
    status          VARCHAR(20) DEFAULT 'running',
    rows_affected   BIGINT DEFAULT 0,
    notes           TEXT
);

-- ใช้ใน migration scripts
INSERT INTO migration_log (migration_name) VALUES ('backfill_province_id')
RETURNING id;
-- บันทึก id แล้วใช้ UPDATE เมื่อเสร็จ
UPDATE migration_log SET status = 'done', completed_at = NOW(), rows_affected = 50000
WHERE id = 1;
```

---

## 🧪 Testing

```bash
# ทดสอบ batch migration บน dev database
sudo -u postgres psql -d chuaikan_db_test << 'EOF'
-- สร้าง test data
INSERT INTO users (username, email)
SELECT 'u' || i, 'u' || i || '@test.com'
FROM generate_series(1, 100000) i;

-- ทดสอบ migration performance
EXPLAIN (ANALYZE, BUFFERS)
UPDATE users SET province_id = 10
WHERE id BETWEEN 1 AND 10000
  AND province_id IS NULL;
EOF
```

---

## ❌ Common Errors & Solutions

**Lock timeout ขณะทำ ALTER TABLE**
```sql
-- ตั้ง lock_timeout เพื่อ fail fast แทนรอนาน
SET lock_timeout = '5s';
ALTER TABLE users ADD COLUMN new_col VARCHAR(50);
RESET lock_timeout;
-- ถ้า timeout → retry ใน maintenance window
```

**CONCURRENTLY index ค้างอยู่ใน invalid state**
```sql
-- ตรวจสอบ
SELECT indexname, pg_get_indexdef(indexrelid) AS def
FROM pg_indexes pi
JOIN pg_index i ON i.indexrelid = pi.indexname::regclass
WHERE tablename = 'posts' AND NOT i.indisvalid;

-- ลบ invalid index แล้วสร้างใหม่
DROP INDEX CONCURRENTLY idx_posts_user_id_invalid;
CREATE INDEX CONCURRENTLY idx_posts_user_id ON posts(user_id);
```

---

## ✅ Checklist

- [ ] **Step 481** — เข้าใจว่า DDL ธรรมดาทำไม lock table นาน
- [ ] **Step 482** — เพิ่ม NOT NULL column ด้วย multi-step approach
- [ ] **Step 483** — สร้าง index ด้วย `CONCURRENTLY` บน table จริง
- [ ] **Step 484** — เขียน TypeScript batch backfill script พร้อม progress tracking
- [ ] **Step 485** — ใช้ shadow table pattern สำหรับ table rebuild
- [ ] **Step 486** — รัน pg_repack บน bloated table
- [ ] **Step 487** — Migrate data ระหว่าง databases พร้อม validation
- [ ] **Step 488** — เตรียม rollback strategy ก่อน migration ทุกครั้ง
- [ ] **Step 489** — รัน validation script หลัง migration สำเร็จ
- [ ] **Step 490** — สร้าง automated migration pipeline script

---

## 🔗 References

- [pg_repack Documentation](https://reorg.github.io/pg_repack/)
- [Safe ALTER TABLE on Large Tables](https://postgres.ai/blog/20210923-zero-downtime-postgres-schema-migrations-lock-timeout-and-retries)
- [Zero-Downtime Migrations](https://medium.com/doctolib/zero-downtime-migrations-19e8c6d1b4d9)
- [PostgreSQL NOT VALID constraint](https://www.postgresql.org/docs/current/sql-altertable.html)

---
*Part 049 | Road to 1,000,000 Users/Day | chuaikan.com*
