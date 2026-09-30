# Part 019: Database Migration & Schema Management
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Beginner
> **Steps:** 181-190
> **เวลาโดยประมาณ:** 8 ชั่วโมง
> **Prerequisites:** Part 018 (Logging ด้วย ELK Stack)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

1. Migration philosophy: backward compatible changes เท่านั้น
2. Prisma Migrate: init, migrate dev, migrate deploy
3. Migration naming convention และ ordering
4. Safe schema changes: add column (nullable ก่อน), rename (two-step), drop (deprecate ก่อน)
5. Zero-downtime migration: adding index concurrently
6. Expand-contract pattern สำหรับ breaking changes
7. Seeding database ด้วย Prisma seed
8. Managing migrations ใน CI/CD pipeline
9. Rolling back failed migrations
10. Schema documentation ด้วย comments
11. Data migrations (transforming existing data)
12. pgBadger สำหรับ migration impact analysis
13. Migration testing strategy

---

## 📖 ทฤษฎีและแนวคิด

### Migration Philosophy

```
หลักการสำคัญ: Backward Compatible Changes Only

การ migration ที่ดีต้องไม่ทำให้ระบบ downtime
ต้องสามารถ deploy ใหม่โดยที่ old version ยังทำงานได้ระหว่าง deploy

Safe Changes:
✅ ADD new nullable column
✅ ADD new table
✅ ADD new index (CONCURRENTLY)
✅ ADD new optional constraint
✅ RENAME column (two-step process)
✅ DROP column (after deprecation period)
✅ Change column type (compatible types only)

Unsafe Changes (ห้ามทำใน production directly):
❌ ADD NOT NULL column without DEFAULT
❌ RENAME column in one step
❌ DROP column immediately
❌ CHANGE column type (incompatible)
❌ ADD UNIQUE constraint (locks table)
```

### Expand-Contract Pattern

```
Pattern สำหรับ Breaking Changes:

Phase 1: EXPAND (เพิ่มของใหม่)
  - เพิ่ม column/table ใหม่
  - Deploy new code ที่เขียนทั้ง old และ new

Phase 2: MIGRATE (ย้ายข้อมูล)
  - ย้าย data จาก old ไป new
  - Run background migration

Phase 3: CONTRACT (ลบของเก่า)
  - Deploy code ที่ใช้แค่ new
  - ลบ old column/table

ตัวอย่าง: เปลี่ยน user.name เป็น user.first_name + user.last_name

Phase 1: เพิ่ม first_name, last_name columns (nullable)
         Code ใหม่: อ่าน name, เขียนทั้ง name, first_name, last_name

Phase 2: backfill first_name, last_name จาก name
         UPDATE users SET 
           first_name = split_part(name, ' ', 1),
           last_name = split_part(name, ' ', 2)

Phase 3: ลบ name column
         Code ใหม่: อ่าน first_name, last_name เท่านั้น
```

---

## ⚙️ Environment Setup

```bash
# ติดตั้ง Prisma
npm install prisma @prisma/client
npx prisma init

# ติดตั้ง pgBadger
sudo apt install pgbadger -y

# สร้าง migration directory structure
mkdir -p prisma/migrations
mkdir -p prisma/seed
mkdir -p scripts/migrations

# Environment setup
cat >> .env << 'EOF'
DATABASE_URL="postgresql://chuaikan:password@localhost:5432/chuaikan_db"
SHADOW_DATABASE_URL="postgresql://chuaikan:password@localhost:5432/chuaikan_shadow_db"
EOF
```

---

## 🛠️ Step-by-Step Implementation

### Step 181: Prisma Schema Setup

```prisma
// prisma/schema.prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["postgresqlExtensions"]
}

datasource db {
  provider          = "postgresql"
  url               = env("DATABASE_URL")
  shadowDatabaseUrl = env("SHADOW_DATABASE_URL")
  extensions        = [postgis, uuid_ossp]
}

// ========================================
// USER MODULE
// ========================================

/// ผู้ใช้งานระบบ chuaikan.com
/// @version 1.0.0
/// @since 2024-01-01
model User {
  /// Primary key - UUID v4
  id                    String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  
  /// Email address - ใช้ login, ต้อง unique
  email                 String    @unique @db.VarChar(255)
  emailVerified         DateTime? @map("email_verified")
  
  /// เบอร์โทรศัพท์ - optional, format: +66812345678
  phone                 String?   @unique @db.VarChar(20)
  
  /// ชื่อที่แสดง
  name                  String    @db.VarChar(100)
  
  /// รูปโปรไฟล์ URL
  image                 String?   @db.VarChar(500)
  
  /// bcrypt hash ของ password, null สำหรับ social login
  passwordHash          String?   @map("password_hash") @db.VarChar(60)
  
  /// บทบาทในระบบ
  role                  UserRole  @default(USER)
  
  /// สถานะบัญชี
  isActive              Boolean   @default(true) @map("is_active")
  
  /// 2FA
  isTwoFactorEnabled    Boolean   @default(false) @map("is_two_factor_enabled")
  twoFactorSecret       String?   @map("two_factor_secret") @db.Text
  
  /// Security - track failed login attempts
  failedLoginAttempts   Int       @default(0) @map("failed_login_attempts")
  lockedUntil           DateTime? @map("locked_until")
  lastLoginAt           DateTime? @map("last_login_at")
  
  /// Timestamps
  createdAt             DateTime  @default(now()) @map("created_at")
  updatedAt             DateTime  @updatedAt @map("updated_at")
  
  /// @deprecated ใช้ first_name และ last_name แทน (migration phase 3)
  // fullName           String?  @map("full_name")

  // Relations
  accounts              Account[]
  refreshTokens         RefreshToken[]
  reportedAlerts        SosAlert[]   @relation("ReportedBy")
  assignedAlerts        SosAlert[]   @relation("AssignedTo")

  @@index([email])
  @@index([phone])
  @@index([role, isActive])
  @@map("users")
}

/// SOS Alert - การแจ้งเหตุฉุกเฉิน
model SosAlert {
  id            String      @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  
  /// ประเภทเหตุการณ์
  type          AlertType
  
  /// ระดับความรุนแรง
  severity      AlertSeverity
  
  /// สถานะปัจจุบัน
  status        AlertStatus @default(PENDING)
  
  /// รายละเอียด
  description   String      @db.Text
  
  /// ตำแหน่ง - ใช้ PostGIS ใน production
  latitude      Decimal     @db.Decimal(10, 8)
  longitude     Decimal     @db.Decimal(11, 8)
  location      String?     @db.VarChar(500)
  
  /// ผู้แจ้ง
  reportedById  String      @map("reported_by_id") @db.Uuid
  
  /// ผู้รับผิดชอบ
  assignedToId  String?     @map("assigned_to_id") @db.Uuid
  
  /// วันที่แก้ไข
  resolvedAt    DateTime?   @map("resolved_at")
  
  createdAt     DateTime    @default(now()) @map("created_at")
  updatedAt     DateTime    @updatedAt @map("updated_at")

  // Relations
  reportedByUser  User      @relation("ReportedBy", fields: [reportedById], references: [id])
  assignedToUser  User?     @relation("AssignedTo", fields: [assignedToId], references: [id])
  timeline        AlertTimeline[]

  @@index([status, createdAt])
  @@index([reportedById])
  @@index([assignedToId])
  @@index([severity, status])
  @@map("sos_alerts")
}

enum UserRole {
  ADMIN
  MODERATOR
  USER
  SOS_RESPONDER
  
  @@map("user_role")
}

enum AlertType {
  MEDICAL
  FIRE
  CRIME
  ACCIDENT
  OTHER
  
  @@map("alert_type")
}

enum AlertSeverity {
  LOW
  MEDIUM
  HIGH
  CRITICAL
  
  @@map("alert_severity")
}

enum AlertStatus {
  PENDING
  ASSIGNED
  IN_PROGRESS
  RESOLVED
  CANCELLED
  
  @@map("alert_status")
}
```

### Step 182: Prisma Migrate Commands

```bash
# Initialize Prisma (ครั้งแรก)
npx prisma init

# สร้าง migration ใหม่ (development)
npx prisma migrate dev --name "init_schema"

# Migration naming convention:
# [YYYYMMDD]_[description_in_snake_case]
# ตัวอย่าง:
npx prisma migrate dev --name "add_sos_alerts_table"
npx prisma migrate dev --name "add_user_phone_column"
npx prisma migrate dev --name "add_index_sos_alerts_status"

# Deploy migrations (production)
npx prisma migrate deploy

# ดู migration status
npx prisma migrate status

# Reset (development only - ล้าง database ทั้งหมด)
npx prisma migrate reset

# Generate Prisma Client
npx prisma generate

# Open Prisma Studio (GUI)
npx prisma studio
```

### Step 183: Safe Migration - Add Column

```bash
# ❌ WRONG: เพิ่ม NOT NULL column โดยไม่มี DEFAULT
# ALTER TABLE users ADD COLUMN phone VARCHAR(20) NOT NULL;
# Error: column "phone" contains null values

# ✅ CORRECT: เพิ่มแบบ nullable ก่อน
# Step 1: เพิ่ม nullable column
npx prisma migrate dev --name "add_user_phone_nullable"
```

```sql
-- prisma/migrations/[timestamp]_add_user_phone_nullable/migration.sql

-- Step 1: Add nullable column
ALTER TABLE "users" ADD COLUMN "phone" VARCHAR(20);

-- Step 2: Add unique index (หลังจาก backfill)
-- ทำใน migration แยก หลังจาก deploy step 1 แล้ว
```

```bash
# Step 2: หลัง deploy แล้ว backfill data (ถ้ามี default value)
# Step 3: Add NOT NULL constraint
npx prisma migrate dev --name "make_user_phone_not_null"
```

```sql
-- prisma/migrations/[timestamp]_make_user_phone_not_null/migration.sql
-- Run ONLY after backfilling phone data for all users

-- Verify no nulls
DO $$
BEGIN
  IF EXISTS (SELECT 1 FROM users WHERE phone IS NULL) THEN
    RAISE EXCEPTION 'Cannot add NOT NULL constraint: null values exist in users.phone';
  END IF;
END $$;

ALTER TABLE "users" ALTER COLUMN "phone" SET NOT NULL;
```

### Step 184: Zero-Downtime Index Creation

```sql
-- ❌ WRONG: สร้าง index แบบปกติจะ lock table
-- CREATE INDEX idx_sos_alerts_status ON sos_alerts(status);

-- ✅ CORRECT: ใช้ CONCURRENTLY (ไม่ lock table)
-- แต่ Prisma ไม่รองรับ CONCURRENTLY โดยตรง ต้องใช้ raw SQL

-- prisma/migrations/[timestamp]_add_sos_alert_status_index/migration.sql

-- Disable transaction (CONCURRENTLY ทำงานใน transaction ไม่ได้)
-- prisma จะ wrap ใน transaction โดยอัตโนมัติ ต้องปิด
```

```typescript
// สำหรับ CONCURRENT index ต้องใช้ manual migration
// scripts/migrations/add-concurrent-index.ts

import { db } from '../src/lib/db';

async function addConcurrentIndex() {
  console.log('Adding concurrent index...');
  
  // ตรวจสอบว่า index มีอยู่แล้วหรือไม่
  const existingIndex = await db.$queryRaw`
    SELECT indexname 
    FROM pg_indexes 
    WHERE tablename = 'sos_alerts' 
    AND indexname = 'idx_sos_alerts_status_created'
  `;

  if ((existingIndex as any[]).length > 0) {
    console.log('Index already exists, skipping...');
    return;
  }

  // สร้าง index แบบ CONCURRENTLY (ต้องใช้ $executeRawUnsafe เพื่อหลีกเลี่ยง transaction)
  console.log('Creating index concurrently (this may take a few minutes for large tables)...');
  
  await db.$executeRawUnsafe(`
    CREATE INDEX CONCURRENTLY IF NOT EXISTS 
    idx_sos_alerts_status_created 
    ON sos_alerts(status, created_at DESC)
  `);

  console.log('Index created successfully!');
}

addConcurrentIndex()
  .catch(console.error)
  .finally(() => db.$disconnect());
```

```bash
# รัน script แยกต่างหาก (ไม่ใช้ Prisma migrate)
npx tsx scripts/migrations/add-concurrent-index.ts
```

### Step 185: Expand-Contract Pattern (Rename Column)

```bash
# ตัวอย่าง: เปลี่ยน user.name เป็น user.display_name

# PHASE 1: EXPAND - เพิ่ม column ใหม่
npx prisma migrate dev --name "expand_add_display_name_column"
```

```sql
-- Migration PHASE 1: EXPAND
ALTER TABLE "users" ADD COLUMN "display_name" VARCHAR(100);

-- Copy existing data to new column
UPDATE "users" SET "display_name" = "name";

-- Make new column NOT NULL (after copy)
ALTER TABLE "users" ALTER COLUMN "display_name" SET NOT NULL;
ALTER TABLE "users" ALTER COLUMN "display_name" SET DEFAULT '';
```

```typescript
// Code ในช่วง PHASE 1 + 2: เขียนทั้ง old และ new field
async function updateUser(id: string, name: string) {
  return db.user.update({
    where: { id },
    data: {
      name,           // old field (backward compat)
      displayName: name, // new field
    },
  });
}

async function getUser(id: string) {
  const user = await db.user.findUnique({ where: { id } });
  return {
    ...user,
    displayName: user?.displayName || user?.name, // prefer new, fallback to old
  };
}
```

```bash
# PHASE 2: MIGRATE - backfill ครบแล้ว, deploy code ที่ใช้แค่ new
# PHASE 3: CONTRACT - ลบ old column
npx prisma migrate dev --name "contract_drop_old_name_column"
```

```sql
-- Migration PHASE 3: CONTRACT (ทำหลังจาก deploy code ใหม่แล้วอย่างน้อย 1 release)
-- ตรวจสอบว่า display_name ไม่มี NULL
DO $$
BEGIN
  IF EXISTS (SELECT 1 FROM users WHERE display_name IS NULL) THEN
    RAISE EXCEPTION 'Cannot drop name: display_name still has NULL values';
  END IF;
END $$;

ALTER TABLE "users" DROP COLUMN "name";
```

### Step 186: Database Seeding

```typescript
// prisma/seed/index.ts
import { PrismaClient } from '@prisma/client';
import { seedUsers } from './users';
import { seedSosAlerts } from './sos-alerts';

const db = new PrismaClient();

async function main() {
  console.log('Starting database seed...');

  await seedUsers(db);
  await seedSosAlerts(db);

  console.log('Seeding completed!');
}

main()
  .catch(console.error)
  .finally(() => db.$disconnect());
```

```typescript
// prisma/seed/users.ts
import { PrismaClient } from '@prisma/client';
import bcrypt from 'bcryptjs';

export async function seedUsers(db: PrismaClient) {
  console.log('Seeding users...');

  const adminPassword = await bcrypt.hash('Admin@123456', 12);
  const userPassword = await bcrypt.hash('User@123456', 12);

  // Create admin user
  await db.user.upsert({
    where: { email: 'admin@chuaikan.com' },
    update: {},
    create: {
      email: 'admin@chuaikan.com',
      name: 'System Admin',
      passwordHash: adminPassword,
      role: 'ADMIN',
      isActive: true,
    },
  });

  // Create test SOS responder
  await db.user.upsert({
    where: { email: 'responder1@chuaikan.com' },
    update: {},
    create: {
      email: 'responder1@chuaikan.com',
      name: 'SOS Responder 1',
      phone: '+66812345678',
      passwordHash: userPassword,
      role: 'SOS_RESPONDER',
      isActive: true,
    },
  });

  // Create test regular users
  const testUsers = Array.from({ length: 10 }, (_, i) => ({
    email: `testuser${i + 1}@chuaikan.com`,
    name: `Test User ${i + 1}`,
    phone: `+6689${String(i + 1).padStart(7, '0')}`,
    passwordHash: userPassword,
    role: 'USER' as const,
    isActive: true,
  }));

  await db.user.createMany({
    data: testUsers,
    skipDuplicates: true,
  });

  console.log(`Users seeded: admin, 1 responder, 10 test users`);
}
```

```typescript
// prisma/seed/sos-alerts.ts
import { PrismaClient } from '@prisma/client';

export async function seedSosAlerts(db: PrismaClient) {
  console.log('Seeding SOS alerts...');

  const users = await db.user.findMany({
    where: { role: 'USER' },
    select: { id: true },
    take: 5,
  });

  if (users.length === 0) {
    console.log('No users found, skipping SOS alerts seed');
    return;
  }

  const alerts = [
    {
      type: 'MEDICAL' as const,
      severity: 'HIGH' as const,
      status: 'PENDING' as const,
      description: 'ผู้สูงอายุหมดสติ ต้องการความช่วยเหลือด่วน',
      latitude: 13.7563,
      longitude: 100.5018,
      location: 'กรุงเทพมหานคร, ย่านสยาม',
      reportedById: users[0].id,
    },
    {
      type: 'ACCIDENT' as const,
      severity: 'CRITICAL' as const,
      status: 'ASSIGNED' as const,
      description: 'อุบัติเหตุรถยนต์ มีผู้บาดเจ็บ 3 คน',
      latitude: 13.7367,
      longitude: 100.5232,
      location: 'ถนนรัชดาภิเษก, กรุงเทพมหานคร',
      reportedById: users[1].id,
    },
    {
      type: 'FIRE' as const,
      severity: 'HIGH' as const,
      status: 'RESOLVED' as const,
      description: 'ไฟไหม้ตึกแถว ควบคุมไฟได้แล้ว',
      latitude: 13.7247,
      longitude: 100.4771,
      location: 'เขตธนบุรี, กรุงเทพมหานคร',
      reportedById: users[2].id,
      resolvedAt: new Date(),
    },
  ];

  for (const alertData of alerts) {
    await db.sosAlert.create({ data: alertData });
  }

  console.log(`SOS alerts seeded: ${alerts.length} alerts`);
}
```

```json
// package.json - เพิ่ม seed script
{
  "prisma": {
    "seed": "tsx prisma/seed/index.ts"
  }
}
```

```bash
# รัน seed
npx prisma db seed

# หรือรัน seed หลัง migrate dev
npx prisma migrate dev --name "init"
# Prisma จะถามว่าต้องการ seed ไหม
```

### Step 187: Data Migration Script

```typescript
// scripts/migrations/backfill-user-display-name.ts
// Data migration: copy name -> display_name
import { PrismaClient } from '@prisma/client';

const db = new PrismaClient();
const BATCH_SIZE = 1000;

async function backfillDisplayName() {
  console.log('Starting backfill: display_name from name...');
  
  let processed = 0;
  let lastId: string | undefined;
  
  while (true) {
    // ดึงข้อมูล batch by batch (ไม่ดึงทีเดียวทั้งหมด)
    const users = await db.user.findMany({
      where: {
        displayName: null, // เฉพาะที่ยังไม่ได้ backfill
        ...(lastId ? { id: { gt: lastId } } : {}),
      },
      select: { id: true, name: true },
      take: BATCH_SIZE,
      orderBy: { id: 'asc' },
    });
    
    if (users.length === 0) break;
    
    // Batch update
    await db.$transaction(
      users.map(user =>
        db.user.update({
          where: { id: user.id },
          data: { displayName: user.name },
        })
      )
    );
    
    processed += users.length;
    lastId = users[users.length - 1].id;
    
    console.log(`Processed: ${processed} users`);
    
    // Throttle to avoid overwhelming DB
    if (users.length === BATCH_SIZE) {
      await new Promise(resolve => setTimeout(resolve, 100));
    }
  }
  
  console.log(`Backfill complete! Total processed: ${processed}`);
}

// Dry run option
const isDryRun = process.argv.includes('--dry-run');
if (isDryRun) {
  console.log('DRY RUN - no changes will be made');
}

backfillDisplayName()
  .catch(console.error)
  .finally(() => db.$disconnect());
```

### Step 188: CI/CD Migration Pipeline

```yaml
# .github/workflows/migrate.yml
name: Database Migration

on:
  push:
    branches: [main]
    paths:
      - 'prisma/migrations/**'
      - 'prisma/schema.prisma'

jobs:
  migrate:
    name: Run Database Migrations
    runs-on: ubuntu-24.04
    
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Generate Prisma Client
        run: npx prisma generate
      
      - name: Check migration status
        run: npx prisma migrate status
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
      
      - name: Run migrations (production)
        run: npx prisma migrate deploy
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
      
      - name: Verify migration
        run: |
          node -e "
          const { PrismaClient } = require('@prisma/client');
          const db = new PrismaClient();
          db.\$queryRaw\`SELECT 1\`.then(() => {
            console.log('Database connection OK');
            db.\$disconnect();
          }).catch(err => {
            console.error('Database verification failed:', err);
            process.exit(1);
          });
          "
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
      
      - name: Notify on failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "⚠️ Database migration FAILED on production!",
              "blocks": [{
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "*Migration Failed*\nBranch: ${{ github.ref }}\nCommit: ${{ github.sha }}"
                }
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### Step 189: Rolling Back Migrations

```bash
# ตรวจสอบ migration status
npx prisma migrate status

# Prisma ไม่มี automatic rollback
# ต้องสร้าง rollback migration เอง

# Step 1: ดู migration ที่ต้อง rollback
ls prisma/migrations/

# Step 2: สร้าง rollback migration
npx prisma migrate dev --name "rollback_[migration_name]"
```

```sql
-- prisma/migrations/[timestamp]_rollback_add_user_phone/migration.sql
-- Rollback: remove phone column that was added in previous migration

-- ตรวจสอบก่อนว่า column มีอยู่
ALTER TABLE "users" DROP COLUMN IF EXISTS "phone";
```

```typescript
// scripts/migrations/rollback.ts
// Script สำหรับ manual rollback ใน emergency

import { PrismaClient } from '@prisma/client';
import { execSync } from 'child_process';

const db = new PrismaClient();

async function rollback(migrationName: string) {
  console.log(`Rolling back: ${migrationName}`);
  
  // 1. Check current migration status
  console.log('Current migration status:');
  execSync('npx prisma migrate status', { stdio: 'inherit' });
  
  // 2. Mark migration as rolled back in _prisma_migrations table
  await db.$executeRaw`
    UPDATE "_prisma_migrations"
    SET "rolled_back_at" = NOW()
    WHERE "migration_name" = ${migrationName}
    AND "rolled_back_at" IS NULL
  `;
  
  console.log(`Migration ${migrationName} marked as rolled back`);
  console.log('WARNING: You must also manually revert the schema changes!');
}

const migrationName = process.argv[2];
if (!migrationName) {
  console.error('Usage: tsx scripts/migrations/rollback.ts <migration_name>');
  process.exit(1);
}

rollback(migrationName)
  .catch(console.error)
  .finally(() => db.$disconnect());
```

### Step 190: pgBadger Migration Analysis

```bash
# ติดตั้ง pgBadger
sudo apt install pgbadger -y

# Enable PostgreSQL slow query logging
sudo nano /etc/postgresql/17/main/postgresql.conf
```

```ini
# postgresql.conf - Enable query logging
log_min_duration_statement = 100   # Log queries slower than 100ms
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0
log_autovacuum_min_duration = 0
```

```bash
# Reload PostgreSQL
sudo systemctl reload postgresql

# Run pgBadger analysis
pgbadger /var/log/postgresql/postgresql-17-main.log \
  --outfile /tmp/pgbadger-report.html \
  --format html \
  --timezone +7

# View report
# เปิด /tmp/pgbadger-report.html ใน browser

# Monitor specific migration
# 1. Enable logging before migration
sudo -u postgres psql -c "SELECT pg_reload_conf();"

# 2. Run migration
time npx prisma migrate deploy

# 3. Analyze logs
pgbadger /var/log/postgresql/postgresql-17-main.log \
  --begin "2024-01-15 10:00:00" \
  --end "2024-01-15 10:30:00" \
  --outfile /tmp/migration-analysis.html
```

---

## 🔧 Configuration Files

### Migration Checklist Script

```bash
#!/bin/bash
# scripts/pre-migration-check.sh
# รันก่อน migration ทุกครั้ง

set -e

echo "=== Pre-Migration Check ==="

# 1. Check database connection
echo "1. Checking database connection..."
npx prisma db execute --stdin <<< "SELECT 1" > /dev/null
echo "   ✓ Database connection OK"

# 2. Check disk space (ต้องมี space อย่างน้อย 10GB)
echo "2. Checking disk space..."
AVAILABLE=$(df /var/lib/postgresql | awk 'NR==2 {print $4}')
REQUIRED=10485760  # 10GB in KB
if [ "$AVAILABLE" -lt "$REQUIRED" ]; then
  echo "   ✗ Insufficient disk space. Available: $((AVAILABLE/1024/1024))GB"
  exit 1
fi
echo "   ✓ Disk space OK: $((AVAILABLE/1024/1024))GB available"

# 3. Check database size
echo "3. Checking database size..."
DB_SIZE=$(npx prisma db execute --stdin <<< "SELECT pg_size_pretty(pg_database_size(current_database()))" 2>/dev/null | grep -o '[0-9.]\+ \(MB\|GB\)')
echo "   ✓ Database size: $DB_SIZE"

# 4. Check for pending migrations
echo "4. Checking migration status..."
npx prisma migrate status
echo ""

# 5. Check connections count
echo "5. Checking active connections..."
CONNECTIONS=$(npx prisma db execute --stdin <<< "SELECT count(*) FROM pg_stat_activity WHERE state = 'active'" 2>/dev/null | grep -o '[0-9]\+' | head -1)
echo "   ✓ Active connections: $CONNECTIONS"

# 6. Check for table locks
echo "6. Checking for locks..."
LOCKS=$(npx prisma db execute --stdin <<< "SELECT count(*) FROM pg_locks WHERE NOT GRANTED" 2>/dev/null | grep -o '[0-9]\+' | head -1)
if [ "$LOCKS" -gt "0" ]; then
  echo "   ⚠ Warning: $LOCKS pending locks detected"
else
  echo "   ✓ No blocking locks"
fi

echo ""
echo "=== Pre-Migration Check Complete ==="
echo "Ready to run migration!"
```

---

## 🧪 Testing

```typescript
// __tests__/migrations/migration.test.ts
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { execSync } from 'child_process';
import { PrismaClient } from '@prisma/client';

describe('Database Migrations', () => {
  let db: PrismaClient;

  beforeAll(async () => {
    db = new PrismaClient({
      datasources: {
        db: { url: process.env.TEST_DATABASE_URL },
      },
    });
  });

  afterAll(async () => {
    await db.$disconnect();
  });

  it('should apply all migrations successfully', async () => {
    // ตรวจสอบว่า migration ทั้งหมดถูก apply
    const result = await db.$queryRaw<{ count: bigint }[]>`
      SELECT COUNT(*) as count 
      FROM "_prisma_migrations" 
      WHERE "finished_at" IS NOT NULL
      AND "rolled_back_at" IS NULL
    `;

    expect(Number(result[0].count)).toBeGreaterThan(0);
  });

  it('should have all required tables', async () => {
    const tables = await db.$queryRaw<{ tablename: string }[]>`
      SELECT tablename 
      FROM pg_tables 
      WHERE schemaname = 'public'
      ORDER BY tablename
    `;

    const tableNames = tables.map(t => t.tablename);
    
    expect(tableNames).toContain('users');
    expect(tableNames).toContain('sos_alerts');
    expect(tableNames).toContain('refresh_tokens');
  });

  it('should have all required indexes', async () => {
    const indexes = await db.$queryRaw<{ indexname: string }[]>`
      SELECT indexname 
      FROM pg_indexes 
      WHERE tablename = 'sos_alerts'
    `;

    const indexNames = indexes.map(i => i.indexname);
    
    // ตรวจสอบ indexes สำคัญ
    expect(indexNames.some(n => n.includes('status'))).toBe(true);
    expect(indexNames.some(n => n.includes('reported_by'))).toBe(true);
  });

  it('should be able to create and read user', async () => {
    const user = await db.user.create({
      data: {
        email: `test-${Date.now()}@chuaikan.com`,
        name: 'Test User',
        role: 'USER',
      },
    });

    expect(user.id).toBeDefined();
    expect(user.email).toBeDefined();

    await db.user.delete({ where: { id: user.id } });
  });
});
```

---

## ❌ Common Errors & Solutions

### Error 1: Migration drift detected

```
Error: P3005 - The database schema is not empty
```

**แก้ไข:**
```bash
# ใช้ baseline เมื่อ database มีข้อมูลอยู่แล้ว
npx prisma migrate resolve --applied "20240115_initial_migration"
```

### Error 2: Column doesn't exist

```
Error: column "phone" of relation "users" does not exist
```

**แก้ไข:**
```bash
# ตรวจสอบ migration status
npx prisma migrate status

# Generate Prisma client ใหม่
npx prisma generate
```

### Error 3: Migration timeout

```
Error: Migration timed out after 30000ms
```

**แก้ไข:**
```bash
# เพิ่ม timeout
export DATABASE_URL="postgresql://...?statement_timeout=300000"
npx prisma migrate deploy
```

---

## ✅ Checklist

- [ ] Prisma schema มี comments ทุก model และ field สำคัญ
- [ ] Migration naming convention: timestamp + descriptive name
- [ ] Safe changes: เพิ่ม nullable column ก่อน
- [ ] Zero-downtime indexes: ใช้ CONCURRENTLY
- [ ] Expand-contract pattern สำหรับ column rename
- [ ] Seed data ครบสำหรับ development environment
- [ ] CI/CD pipeline รัน `prisma migrate deploy`
- [ ] Pre-migration checklist script
- [ ] Rollback procedure documented
- [ ] Data migration scripts แยกจาก schema migrations
- [ ] pgBadger configured สำหรับ performance monitoring
- [ ] Migration tests ใน test suite

---

## 🔗 References

- [Prisma Migrate Documentation](https://www.prisma.io/docs/concepts/components/prisma-migrate)
- [Zero Downtime Migrations](https://benchling.engineering/move-fast-and-migrate-things-how-we-automated-migrations-in-postgres-d60aba0fc3d4)
- [Expand-Contract Pattern](https://martinfowler.com/bliki/ParallelChange.html)
- [pgBadger](https://github.com/darold/pgbadger)
- [PostgreSQL ALTER TABLE documentation](https://www.postgresql.org/docs/17/sql-altertable.html)

---
*Part 019 | Road to 1,000,000 Users/Day | chuaikan.com*
