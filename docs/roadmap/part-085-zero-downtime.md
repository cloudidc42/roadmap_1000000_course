# Part 085: Zero-downtime Migration
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Professional
> **Steps:** 841–850
> **เวลาโดยประมาณ:** 8 ชั่วโมง
> **Prerequisites:** Part 080 (Blue-Green), Part 082 (Feature Flags)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- Zero-downtime checklist สำหรับทุก migration
- Backward-compatible schema changes (expand-contract)
- Rolling deployment strategy
- Database migration ด้วย dual-read/write period
- Feature flag สำหรับ new code path
- Smoke tests ระหว่าง migration
- Runbook สำหรับ zero-downtime major version upgrade
- PostgreSQL major version upgrade ด้วย pg_upgrade
- Node.js version upgrade strategy

---

## 📖 ทฤษฎีและแนวคิด

### Zero-downtime Principles

1. **No breaking changes**: Code ใหม่ต้องทำงานร่วมกับ code เก่าได้
2. **Expand-Contract**: เพิ่ม feature → migrate data → ลบ old code
3. **Feature flags**: เปิด code path ใหม่เมื่อพร้อม
4. **Blue-green**: switch traffic อย่างรวดเร็ว
5. **Rollback plan**: ทุก step ต้องมี rollback

---

## 🛠️ Step-by-Step Implementation

### Step 841: Zero-downtime Checklist

```bash
#!/bin/bash
# scripts/zero-downtime-checklist.sh
# เช็ค checklist ก่อนทำ migration ใหญ่

echo "=== Zero-downtime Migration Checklist ==="
echo ""

check() {
    local item=$1
    read -p "[ ] ${item} (y/n): " response
    if [ "$response" != "y" ]; then
        echo "BLOCKED: ${item} not ready!"
        exit 1
    fi
    echo "[x] ${item}"
}

# Pre-migration checks
check "Database backup เสร็จแล้วและทดสอบ restore แล้ว"
check "Schema migration ผ่าน compatibility check"
check "Staging environment ทดสอบ migration แล้ว"
check "Rollback plan เขียนแล้วและทดสอบแล้ว"
check "Monitoring dashboard เปิดอยู่"
check "On-call engineer พร้อม"
check "Customer support aware (ถ้า migration ใหญ่)"
check "Feature flag migration path เปิดอยู่"
check "Dual-write code deploy แล้ว"
check "Load test บน staging ผ่านแล้ว"

echo ""
echo "=== ALL CHECKS PASSED. READY TO MIGRATE ==="
```

### Step 842: Expand-Contract Pattern

```sql
-- ===== SCENARIO: เปลี่ยน user.name เป็น first_name + last_name =====

-- ===== PHASE 1: EXPAND =====
-- เพิ่ม columns ใหม่ (nullable)
ALTER TABLE users
  ADD COLUMN first_name VARCHAR(100),
  ADD COLUMN last_name VARCHAR(100);

-- ทั้ง v1 (ใช้ name) และ v2 (ใช้ first_name+last_name) ทำงานได้

-- ===== PHASE 2: MIGRATE (ทำ background) =====
-- เริ่ม populate new columns จาก old column
-- ทำทีละ batch ไม่ lock table
DO $$
DECLARE
    batch_size INT := 1000;
    offset_val INT := 0;
    rows_updated INT;
BEGIN
    LOOP
        UPDATE users u
        SET
            first_name = split_part(u.name, ' ', 1),
            last_name = NULLIF(substring(u.name from position(' ' in u.name) + 1), '')
        WHERE u.id IN (
            SELECT id FROM users
            WHERE first_name IS NULL
            ORDER BY id
            LIMIT batch_size
        );

        GET DIAGNOSTICS rows_updated = ROW_COUNT;
        EXIT WHEN rows_updated = 0;

        PERFORM pg_sleep(0.1);  -- เว้นว่างให้ DB หายใจ
    END LOOP;
END $$;

-- ===== PHASE 3: CONTRACT (หลัง v2 stable 2 สัปดาห์) =====
-- เพิ่ม NOT NULL หลัง backfill เสร็จ
ALTER TABLE users
  ALTER COLUMN first_name SET NOT NULL,
  ALTER COLUMN last_name SET DEFAULT '';

-- สร้าง computed column หรือ view สำหรับ backward compatibility
ALTER TABLE users
  ADD COLUMN name_computed VARCHAR(200)
  GENERATED ALWAYS AS (
    TRIM(first_name || ' ' || COALESCE(last_name, ''))
  ) STORED;

-- DropColumn name เมื่อ v1 code ถูก retire แล้ว
-- ALTER TABLE users DROP COLUMN name;
```

### Step 843: Dual-Read/Write Pattern

```typescript
// src/repositories/user.repository.ts
// Dual-write pattern: เขียนทั้ง old และ new structure ใน phase 2

import { isFeatureEnabled } from '../lib/feature-flags';

export class UserRepository {
  async updateUserName(
    userId: string,
    fullName: string
  ): Promise<void> {
    const useNewSchema = await isFeatureEnabled('new-user-schema', userId);

    if (useNewSchema) {
      // Phase 2+: เขียน new schema หลัก, เขียน old schema backup
      const [firstName, ...restParts] = fullName.split(' ');
      const lastName = restParts.join(' ');

      await writePool.query(
        `UPDATE users
         SET first_name = $1, last_name = $2, name = $3
         WHERE id = $4`,
        [firstName, lastName, fullName, userId]
      );
    } else {
      // Phase 1: เขียน old schema หลัก, sync ไป new schema
      const [firstName, ...restParts] = fullName.split(' ');
      const lastName = restParts.join(' ');

      await writePool.query(
        `UPDATE users
         SET name = $1, first_name = $2, last_name = $3
         WHERE id = $4`,
        [fullName, firstName, lastName, userId]
      );
    }
  }

  async getUserName(userId: string): Promise<string> {
    const useNewSchema = await isFeatureEnabled('new-user-schema', userId);

    if (useNewSchema) {
      // อ่านจาก new schema
      const result = await readPool.query(
        `SELECT first_name, last_name FROM users WHERE id = $1`,
        [userId]
      );
      const { first_name, last_name } = result.rows[0];
      return [first_name, last_name].filter(Boolean).join(' ');
    } else {
      // อ่านจาก old schema
      const result = await readPool.query(
        `SELECT name FROM users WHERE id = $1`,
        [userId]
      );
      return result.rows[0].name;
    }
  }
}
```

### Step 844: PostgreSQL Major Version Upgrade (14 → 16)

```bash
#!/bin/bash
# scripts/pg-major-upgrade.sh
# Zero-downtime PostgreSQL major version upgrade
# Strategy: Setup new version, sync data, switch

set -euo pipefail

OLD_VERSION="14"
NEW_VERSION="16"
OLD_PORT="5432"
NEW_PORT="5433"
DB_NAME="chuaikan"

echo "=== PostgreSQL Major Version Upgrade: ${OLD_VERSION} → ${NEW_VERSION} ==="

# Step 1: ติดตั้ง PostgreSQL 16 parallel
echo "Step 1: Installing PostgreSQL ${NEW_VERSION}..."
sudo apt install -y postgresql-${NEW_VERSION} postgresql-${NEW_VERSION}-postgis-3

# Step 2: Initialize new cluster บน port ใหม่
sudo -u postgres /usr/lib/postgresql/${NEW_VERSION}/bin/initdb \
  -D /var/lib/postgresql/${NEW_VERSION}/main

# แก้ port ใน postgresql.conf
sudo sed -i "s/port = 5432/port = ${NEW_PORT}/" \
  /etc/postgresql/${NEW_VERSION}/main/postgresql.conf

# เพิ่ม logical replication settings
sudo -u postgres cat >> /etc/postgresql/${NEW_VERSION}/main/postgresql.conf << 'EOF'
wal_level = logical
max_replication_slots = 20
max_wal_senders = 20
EOF

sudo systemctl start postgresql@${NEW_VERSION}-main

# Step 3: pg_upgrade check
echo "Step 3: pg_upgrade check..."
sudo -u postgres /usr/lib/postgresql/${NEW_VERSION}/bin/pg_upgrade \
  --old-datadir=/var/lib/postgresql/${OLD_VERSION}/main \
  --new-datadir=/var/lib/postgresql/${NEW_VERSION}/main \
  --old-bindir=/usr/lib/postgresql/${OLD_VERSION}/bin \
  --new-bindir=/usr/lib/postgresql/${NEW_VERSION}/bin \
  --check  # dry-run ก่อน

echo "pg_upgrade check: PASSED"

# Step 4: Setup logical replication จาก 14 → 16
echo "Step 4: Setting up logical replication from ${OLD_VERSION} to ${NEW_VERSION}..."

# บน PostgreSQL 14 (old): สร้าง publication
sudo -u postgres psql -p ${OLD_PORT} -d ${DB_NAME} << 'EOF'
CREATE PUBLICATION upgrade_pub FOR ALL TABLES;
EOF

# บน PostgreSQL 16 (new): dump schema และ subscribe
sudo -u postgres pg_dump \
  -p ${OLD_PORT} \
  --schema-only \
  --no-owner \
  ${DB_NAME} | sudo -u postgres psql -p ${NEW_PORT} -d ${DB_NAME}

sudo -u postgres psql -p ${NEW_PORT} -d ${DB_NAME} << EOF
CREATE SUBSCRIPTION upgrade_sub
  CONNECTION 'host=localhost port=${OLD_PORT} dbname=${DB_NAME} user=postgres'
  PUBLICATION upgrade_pub
  WITH (copy_data = true, create_slot = true);
EOF

# Step 5: Monitor replication lag
echo "Step 5: Monitoring replication lag..."
while true; do
  LAG=$(sudo -u postgres psql -p ${OLD_PORT} -tAc \
    "SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn) FROM pg_replication_slots WHERE slot_name = 'upgrade_sub';")
  echo "Replication lag: ${LAG} bytes"
  [ "${LAG}" -lt 1024 ] && echo "Lag is minimal! Ready to switch." && break
  sleep 5
done

# Step 6: Switch (ต้องทำอย่างรวดเร็ว)
echo "Step 6: Switching to new PostgreSQL version..."

# หยุด writes บน app (flip feature flag)
# ... (manual step)

read -p "Stop app writes? (yes): "

# ตรวจสอบ lag เป็น 0
FINAL_LAG=$(sudo -u postgres psql -p ${OLD_PORT} -tAc \
  "SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn) FROM pg_replication_slots WHERE slot_name = 'upgrade_sub';")
echo "Final lag: ${FINAL_LAG}"

# Switch port: 16 เปลี่ยนไปใช้ port 5432
sudo sed -i "s/port = ${NEW_PORT}/port = ${OLD_PORT}/" \
  /etc/postgresql/${NEW_VERSION}/main/postgresql.conf
sudo sed -i "s/port = ${OLD_PORT}/port = 5500/" \
  /etc/postgresql/${OLD_VERSION}/main/postgresql.conf

sudo systemctl restart postgresql@${NEW_VERSION}-main
sudo systemctl restart postgresql@${OLD_VERSION}-main

echo "=== SWITCH COMPLETE ==="
echo "PostgreSQL ${NEW_VERSION} now on port ${OLD_PORT}"
echo "Test connectivity: psql -p ${OLD_PORT} -d ${DB_NAME}"
```

### Step 845: Node.js Version Upgrade

```bash
#!/bin/bash
# scripts/nodejs-upgrade.sh
# Node.js 20 → 22 zero-downtime upgrade

# Strategy: Rolling upgrade บน Kubernetes

OLD_VERSION="20"
NEW_VERSION="22"
NAMESPACE="chuaikan"
DEPLOYMENT="chuaikan-api"

echo "=== Node.js Upgrade: ${OLD_VERSION} → ${NEW_VERSION} ==="

# Step 1: ทดสอบบน staging
echo "Testing on staging..."
kubectl set image deployment/${DEPLOYMENT} \
  api=registry.chuaikan.com/api:node${NEW_VERSION}-latest \
  -n chuaikan-staging

# รอ staging stable
kubectl rollout status deployment/${DEPLOYMENT} -n chuaikan-staging

# ทดสอบ
curl -sf "https://staging.chuaikan.com/health" || {
  echo "Staging test failed!"
  kubectl rollout undo deployment/${DEPLOYMENT} -n chuaikan-staging
  exit 1
}

echo "Staging: PASSED"

# Step 2: Build และ tag image สำหรับ production
# Dockerfile ใช้ multi-stage build

cat > Dockerfile.nodeupgrade << 'EOF'
# Stage 1: Build
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:22-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package.json ./

# ตรวจสอบ Node.js version
RUN node --version

USER node
CMD ["node", "dist/index.js"]
EOF

# Build image
docker build -f Dockerfile.nodeupgrade \
  -t registry.chuaikan.com/api:node22-v1.5.0 .

docker push registry.chuaikan.com/api:node22-v1.5.0

# Step 3: Rolling upgrade production (zero-downtime)
echo "Starting rolling upgrade production..."
kubectl set image deployment/${DEPLOYMENT} \
  api=registry.chuaikan.com/api:node22-v1.5.0 \
  -n ${NAMESPACE}

# ดู rolling update progress
kubectl rollout status deployment/${DEPLOYMENT} -n ${NAMESPACE} --timeout=300s

echo "Node.js upgrade complete!"
node_version=$(kubectl exec -n ${NAMESPACE} \
  $(kubectl get pod -n ${NAMESPACE} -l app=chuaikan-api -o name | head -1) \
  -- node --version)
echo "Running version: ${node_version}"
```

### Step 846: Smoke Tests ระหว่าง Migration

```typescript
// tests/smoke/migration-smoke-tests.ts
import axios from 'axios';

const BASE_URL = process.env.SMOKE_TEST_URL || 'https://api.chuaikan.com';

const tests = [
  {
    name: 'Health check',
    fn: async () => {
      const res = await axios.get(`${BASE_URL}/health`);
      if (res.status !== 200) throw new Error(`Status: ${res.status}`);
      if (!res.data.status === 'ok') throw new Error('Status not ok');
    }
  },
  {
    name: 'List posts',
    fn: async () => {
      const res = await axios.get(`${BASE_URL}/v1/posts?limit=1`);
      if (res.status !== 200) throw new Error(`Status: ${res.status}`);
      if (!Array.isArray(res.data.data)) throw new Error('Not array');
    }
  },
  {
    name: 'Auth endpoint returns 401',
    fn: async () => {
      try {
        await axios.get(`${BASE_URL}/v1/me`);
        throw new Error('Should have returned 401');
      } catch (e: any) {
        if (e.response?.status !== 401) throw e;
      }
    }
  },
  {
    name: 'Database connectivity',
    fn: async () => {
      const res = await axios.get(`${BASE_URL}/health/db`);
      if (res.data.database !== 'connected') throw new Error('DB not connected');
    }
  }
];

async function runSmokeTests(): Promise<boolean> {
  let passed = 0;
  let failed = 0;

  for (const test of tests) {
    try {
      await test.fn();
      console.log(`PASS: ${test.name}`);
      passed++;
    } catch (e: any) {
      console.error(`FAIL: ${test.name}: ${e.message}`);
      failed++;
    }
  }

  console.log(`\nResults: ${passed}/${tests.length} passed`);
  return failed === 0;
}

if (require.main === module) {
  runSmokeTests().then(success => {
    process.exit(success ? 0 : 1);
  });
}
```

---

## ✅ Checklist

- [ ] Expand-contract pattern ใช้กับทุก schema change
- [ ] Dual-write period ก่อน drop old code/columns
- [ ] Feature flag ควบคุม code path ใหม่
- [ ] pg_upgrade --check ผ่านก่อน upgrade จริง
- [ ] Logical replication lag ≈ 0 ก่อน switch
- [ ] Node.js upgrade ทดสอบบน staging ก่อน
- [ ] Smoke tests ผ่านหลัง migration
- [ ] Rollback plan ทดสอบแล้ว
- [ ] Monitoring dashboards เปิดระหว่าง migration
- [ ] MTTR (Mean Time to Recovery) < 5 นาที

---

## 🔗 References

- [Expand-Contract Migration](https://www.martinfowler.com/bliki/ParallelChange.html)
- [PostgreSQL pg_upgrade](https://www.postgresql.org/docs/current/pgupgrade.html)
- [Zero-downtime Database Migrations](https://www.depesz.com/2020/01/02/how-to-safely-change-column-types/)

---
*Part 085 | Road to 1,000,000 Users/Day | chuaikan.com*
