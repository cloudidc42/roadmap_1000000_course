# Part 050: Backup & Disaster Recovery
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 491–500
> **เวลาโดยประมาณ:** 4 ชั่วโมง
> **Prerequisites:** Part 049 (Data Migration), PostgreSQL, Redis

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- กลยุทธ์ Backup: 3-2-1 Rule
- Continuous Archiving ด้วย pgBackRest (full, differential, incremental)
- Point-in-Time Recovery (PITR)
- Backup ไปยัง Cloudflare R2 (offsite)
- Redis Backup: RDB + AOF
- กำหนด RTO (Recovery Time Objective) และ RPO (Recovery Point Objective)
- เขียน Disaster Recovery Runbook
- ตั้งตาราง Backup Testing (restore drill รายเดือน)

---

## 📖 ทฤษฎีและแนวคิด

### Step 491 — 3-2-1 Backup Rule

```
3-2-1 Rule:
3 copies ของข้อมูล:
  1. Production database (live)
  2. Local backup (ใน datacenter เดียวกัน)
  3. Offsite backup (Cloudflare R2, ต่างประเทศ)

2 storage types:
  1. SSD/NVMe (production)
  2. Object Storage (R2/S3)

1 offsite copy:
  - ต้องอยู่นอก datacenter หลัก
  - ป้องกัน datacenter failure

สำหรับ chuaikan.com:
RTO (Recovery Time Objective): ≤ 1 ชั่วโมง
  → กลับมา online ภายใน 1 ชั่วโมงหลัง disaster

RPO (Recovery Point Objective): ≤ 5 นาที
  → เสียข้อมูลได้ไม่เกิน 5 นาทีล่าสุด

เพื่อให้ได้ RPO 5 นาที → ต้องทำ continuous WAL archiving
```

### Step 492 — pgBackRest vs pg_basebackup

```
pg_basebackup:
- ง่าย, built-in
- เหมาะสำหรับ setup replica
- ไม่มี incremental backup
- ไม่มี compression ที่ดี

pgBackRest:
- Full, Differential, Incremental backups
- Parallel backup/restore
- Built-in encryption (AES-256)
- Delta restore (เฉพาะ changed blocks)
- Multiple storage backends (S3, Azure, GCS, R2)
- Retention policies
- เหมาะสำหรับ production
```

---

## ⚙️ Environment Setup

### ติดตั้ง pgBackRest

```bash
# ติดตั้ง pgBackRest 2.x
sudo apt-get install -y pgbackrest

# ตรวจสอบ version
pgbackrest version

# สร้าง directories
sudo mkdir -p /var/lib/pgbackrest /var/log/pgbackrest
sudo chown postgres:postgres /var/lib/pgbackrest /var/log/pgbackrest
sudo chmod 750 /var/lib/pgbackrest /var/log/pgbackrest

# สร้าง config directory
sudo mkdir -p /etc/pgbackrest
sudo chown postgres:postgres /etc/pgbackrest
```

---

## 🛠️ Step-by-Step Implementation

### Step 493 — Configure pgBackRest

```ini
# /etc/pgbackrest/pgbackrest.conf

[global]
# Log settings
log-level-console=info
log-level-file=detail
log-path=/var/log/pgbackrest

# Repository (local)
repo1-path=/var/lib/pgbackrest

# Repository (Cloudflare R2 - offsite)
repo2-type=s3
repo2-s3-bucket=chuaikan-backups
repo2-s3-endpoint=<ACCOUNT_ID>.r2.cloudflarestorage.com
repo2-s3-region=auto
repo2-s3-key=<R2_ACCESS_KEY>
repo2-s3-key-secret=<R2_SECRET_KEY>
repo2-path=/backups/chuaikan

# Encryption (สำคัญมาก!)
repo1-cipher-type=aes-256-cbc
repo1-cipher-pass=<STRONG_RANDOM_PASSPHRASE>
repo2-cipher-type=aes-256-cbc
repo2-cipher-pass=<STRONG_RANDOM_PASSPHRASE>

# Retention
repo1-retention-full=4          # เก็บ full backups 4 ชุด (~1 เดือน)
repo1-retention-diff=14         # เก็บ diff backups 14 ชุด (~2 สัปดาห์)
repo2-retention-full=12         # เก็บ full backups 12 ชุด (~3 เดือน)

# Compression
compress-type=lz4               # เร็วกว่า gz, ratio ดีพอ
compress-level=6

[chuaikan]
pg1-path=/var/lib/postgresql/16/main
pg1-port=5432
pg1-socket-path=/var/run/postgresql

# Parallel workers
process-max=4
```

```bash
# ปรับ postgresql.conf สำหรับ WAL archiving
sudo -u postgres psql << 'EOF'
ALTER SYSTEM SET archive_mode = 'on';
ALTER SYSTEM SET archive_command =
  'pgbackrest --stanza=chuaikan archive-push %p';
ALTER SYSTEM SET archive_timeout = '300';  -- archive ทุก 5 นาที (RPO)
ALTER SYSTEM SET wal_level = 'replica';
SELECT pg_reload_conf();
EOF

# รีสตาร์ท PostgreSQL
sudo systemctl restart postgresql

# สร้าง stanza (เชื่อม pgBackRest กับ PostgreSQL)
sudo -u postgres pgbackrest --stanza=chuaikan stanza-create

# ทดสอบ configuration
sudo -u postgres pgbackrest --stanza=chuaikan check
```

### Step 494 — Full, Differential, Incremental Backup

```bash
# Full Backup (ทุกสัปดาห์)
sudo -u postgres pgbackrest \
  --stanza=chuaikan \
  --type=full \
  backup

# Differential Backup (ทุกวัน - เก็บ changes ตั้งแต่ full backup ล่าสุด)
sudo -u postgres pgbackrest \
  --stanza=chuaikan \
  --type=diff \
  backup

# Incremental Backup (ทุกชั่วโมง - เก็บ changes ตั้งแต่ backup ล่าสุด)
sudo -u postgres pgbackrest \
  --stanza=chuaikan \
  --type=incr \
  backup

# ดู backup catalog
sudo -u postgres pgbackrest \
  --stanza=chuaikan \
  info

# ตัวอย่าง output:
# stanza: chuaikan
#   status: ok
#   db: backup.repo1
#     wal archive min/max (16): 000000010000000000000001/0000000100000000000000FF
#   full backup: 20240115-020000F
#     timestamp start/stop: 2024-01-15 02:00:00+00 / 2024-01-15 02:45:23+00
#     wal start/stop: 000000010000000000000001 / 00000001000000000000000F
#     database size: 45.2GB, database backup size: 45.2GB
#   diff backup: 20240115-020000F_20240116-020000D
#     timestamp start/stop: 2024-01-16 02:00:00+00 / 2024-01-16 02:05:42+00
#     database backup size: 2.3GB (full: 45.2GB)
```

```bash
# Cron jobs สำหรับ backup schedule

# แก้ไข crontab สำหรับ postgres user
sudo -u postgres crontab -e

# เพิ่มบรรทัด:
# Full backup ทุกอาทิตย์ 02:00 น.
0 2 * * 0 pgbackrest --stanza=chuaikan --type=full backup

# Differential backup ทุกวัน 02:00 น. (ยกเว้นอาทิตย์)
0 2 * * 1-6 pgbackrest --stanza=chuaikan --type=diff backup

# Incremental backup ทุกชั่วโมง (ยกเว้น 02:00)
0 3-23 * * * pgbackrest --stanza=chuaikan --type=incr backup

# ตรวจสอบ backups ทุกวัน
30 6 * * * pgbackrest --stanza=chuaikan check
```

### Step 495 — Point-in-Time Recovery (PITR)

```bash
# สถานการณ์: มี bad SQL query ทำลายข้อมูล 14:30 น.

# 1. หา backup ที่ดีก่อน 14:30
sudo -u postgres pgbackrest --stanza=chuaikan info

# 2. Stop PostgreSQL
sudo systemctl stop postgresql

# 3. Restore ไปยัง 14:25 น. (ก่อน incident)
sudo -u postgres pgbackrest \
  --stanza=chuaikan \
  --delta \
  --type=time \
  --target="2024-01-15 14:25:00+07" \
  --target-action=promote \
  restore

# --delta: เฉพาะ changed blocks (เร็วกว่า full restore)
# --type=time: restore ถึงเวลาที่กำหนด
# --target-action=promote: เปิด database หลัง recovery

# 4. Start PostgreSQL
sudo systemctl start postgresql

# 5. ตรวจสอบว่า recovery ถึงเวลาที่ต้องการ
sudo -u postgres psql -c "SELECT pg_last_xact_replay_timestamp();"

# 6. ตรวจสอบข้อมูล
sudo -u postgres psql -d chuaikan_db -c \
  "SELECT COUNT(*) FROM users WHERE created_at < '2024-01-15 14:30:00';"
```

```bash
# Recovery ไป specific LSN (แม่นยำกว่า time)
sudo -u postgres pgbackrest \
  --stanza=chuaikan \
  --delta \
  --type=lsn \
  --target="0/3F000000" \
  --target-action=promote \
  restore
```

### Step 496 — Cloudflare R2 Offsite Backup

```bash
# ติดตั้ง rclone สำหรับ R2
curl https://rclone.org/install.sh | sudo bash

# Configure rclone สำหรับ R2
rclone config create r2 s3 \
  provider Cloudflare \
  access_key_id <R2_ACCESS_KEY> \
  secret_access_key <R2_SECRET_KEY> \
  endpoint https://<ACCOUNT_ID>.r2.cloudflarestorage.com \
  region auto

# ทดสอบ connection
rclone ls r2:chuaikan-backups

# Sync local backups ไป R2
rclone sync \
  /var/lib/pgbackrest/ \
  r2:chuaikan-backups/pgbackrest/ \
  --transfers 8 \
  --checkers 16 \
  --progress

# pgBackRest upload โดยตรงไป R2 (ดีกว่า rclone sync)
# ตาม config ที่ตั้งไว้ใน pgbackrest.conf แล้ว (repo2-type=s3)
sudo -u postgres pgbackrest \
  --stanza=chuaikan \
  --repo=2 \
  --type=full \
  backup
```

### Step 497 — Redis Backup (RDB + AOF)

```bash
# === RDB Snapshot ===
# redis.conf
cat >> /etc/redis/redis.conf << 'EOF'
# RDB snapshots
save 900 1        # หลัง 15 นาทีถ้ามี 1 key เปลี่ยน
save 300 10       # หลัง 5 นาทีถ้ามี 10 keys เปลี่ยน
save 60 10000     # หลัง 1 นาทีถ้ามี 10000 keys เปลี่ยน

dbfilename dump.rdb
dir /var/lib/redis

# AOF (Append Only File)
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec         # flush ทุก 1 วินาที (trade-off: fast + safe)

# AOF rewrite เมื่อ AOF ใหญ่เกินไป
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# RDB + AOF hybrid mode (PostgreSQL 7.4+)
aof-use-rdb-preamble yes
EOF

# Manual RDB snapshot
redis-cli BGSAVE
redis-cli LASTSAVE  # Unix timestamp ของ snapshot ล่าสุด

# Manual AOF rewrite
redis-cli BGREWRITEAOF
```

```bash
# #!/bin/bash
# /usr/local/bin/backup-redis.sh

set -euo pipefail

DATE=$(date +%Y%m%d-%H%M%S)
REDIS_DIR="/var/lib/redis"
BACKUP_DIR="/var/backups/redis"
R2_BUCKET="r2:chuaikan-backups/redis"

mkdir -p "$BACKUP_DIR"

# Trigger RDB save
redis-cli BGSAVE
sleep 5  # รอ save เสร็จ

# Copy RDB
cp "$REDIS_DIR/dump.rdb" "$BACKUP_DIR/dump-$DATE.rdb"

# Copy AOF
if [ -f "$REDIS_DIR/appendonly.aof" ]; then
  cp "$REDIS_DIR/appendonly.aof" "$BACKUP_DIR/aof-$DATE.aof"
fi

# Upload to R2
rclone copy "$BACKUP_DIR/dump-$DATE.rdb" "$R2_BUCKET/" --progress

# Cleanup: เก็บแค่ 7 วันล่าสุด
find "$BACKUP_DIR" -mtime +7 -delete

echo "[$(date)] Redis backup complete: dump-$DATE.rdb"
```

```bash
# เพิ่ม cron
sudo crontab -e
# 0 * * * * /usr/local/bin/backup-redis.sh >> /var/log/redis-backup.log 2>&1
```

### Step 498 — RTO/RPO Targets และ Measurement

```bash
# ทดสอบ RTO: วัดเวลา restore จริง
time sudo -u postgres pgbackrest \
  --stanza=chuaikan \
  --delta \
  restore

# บันทึกผล
echo "Restore time: $SECONDS seconds (RTO target: 3600s)"

# ทดสอบ RPO: ตรวจสอบว่าเสียข้อมูลแค่ไหน
sudo -u postgres psql << 'EOF'
-- หลัง restore ตรวจสอบว่า transaction ล่าสุดคือเมื่อไหร่
SELECT
  pg_last_xact_replay_timestamp() AS last_transaction,
  NOW() - pg_last_xact_replay_timestamp() AS data_loss_window;
EOF
```

```yaml
# docs/rto-rpo-targets.yaml
# RTO/RPO Targets สำหรับ chuaikan.com

services:
  users_db:
    rto: "1 hour"      # กลับมา online ภายใน 1 ชั่วโมง
    rpo: "5 minutes"   # เสียข้อมูลได้ไม่เกิน 5 นาที
    strategy: "PITR with pgBackRest + WAL archiving every 5 min"

  content_db:
    rto: "2 hours"
    rpo: "15 minutes"
    strategy: "pgBackRest + hourly incremental"

  sos_db:
    rto: "30 minutes"  # SOS critical! ต้องเร็วที่สุด
    rpo: "1 minute"    # เสียข้อมูลได้ไม่เกิน 1 นาที
    strategy: "Synchronous replica + pgBackRest + WAL streaming"

  redis:
    rto: "15 minutes"
    rpo: "1 second"    # AOF everysec
    strategy: "Redis AOF + RDB snapshots hourly"

  analytics_db:
    rto: "4 hours"     # ไม่ critical
    rpo: "1 hour"
    strategy: "pgBackRest daily full + hourly incremental"
```

### Step 499 — Disaster Recovery Runbook

```markdown
# Disaster Recovery Runbook — chuaikan.com
## Version: 1.0 | Updated: 2024-01-15

## 1. Database Server Failure

### Symptoms:
- API ตอบ 500 errors
- Grafana แสดง DB connection errors
- PgBouncer log: "connection refused"

### Steps:
1. ตรวจสอบสถานะ database server
   ```bash
   ping db-primary.chuaikan.internal
   ssh admin@db-primary.chuaikan.internal
   sudo systemctl status postgresql
   ```

2. ถ้า DB server down → Failover ไป replica
   ```bash
   # Promote replica-1
   docker exec pg-replica-1 psql -U postgres -c "SELECT pg_promote();"
   # อัพเดท PgBouncer
   sed -i 's/host=pg-primary/host=pg-replica-1/' /etc/pgbouncer/pgbouncer.ini
   kill -HUP $(cat /var/run/pgbouncer/pgbouncer.pid)
   ```

3. ถ้าทุก replica down → Restore จาก backup
   ```bash
   # 1. สร้าง instance ใหม่ (Hetzner/AWS)
   # 2. ติดตั้ง PostgreSQL 16
   # 3. Restore ด้วย pgBackRest
   pgbackrest --stanza=chuaikan --delta restore
   # 4. Start PostgreSQL
   sudo systemctl start postgresql
   ```

4. แจ้ง stakeholders ผ่าน Slack #incidents

## 2. Data Corruption (Bad Query)

### Steps:
1. ระบุเวลาที่เกิด corruption จาก logs
   ```bash
   grep "ERROR\|FATAL" /var/log/postgresql/postgresql-*.log | tail -50
   ```

2. PITR restore ไปก่อน corruption
   ```bash
   sudo systemctl stop postgresql
   pgbackrest --stanza=chuaikan --delta \
     --type=time --target="<TIME_BEFORE_INCIDENT>" \
     --target-action=promote restore
   sudo systemctl start postgresql
   ```

3. Validate ข้อมูล
   ```bash
   npx ts-node scripts/validate-migration.ts
   ```

## 3. Datacenter Failure

### Steps:
1. ตรวจสอบ R2 offsite backup
   ```bash
   pgbackrest --stanza=chuaikan --repo=2 info
   ```

2. Deploy infrastructure ใหม่ใน region ต่างกัน

3. Restore จาก R2
   ```bash
   pgbackrest --stanza=chuaikan --repo=2 --delta restore
   ```

## 4. Redis Failure

### Steps:
1. รีสตาร์ท Redis
   ```bash
   sudo systemctl restart redis-server
   ```

2. ถ้าข้อมูลหาย → restore จาก backup
   ```bash
   sudo systemctl stop redis-server
   cp /var/backups/redis/dump-latest.rdb /var/lib/redis/dump.rdb
   sudo systemctl start redis-server
   ```

3. ทำ cache warming
   ```bash
   npx ts-node scripts/cache-warm.ts
   ```
```

### Step 500 — Backup Testing Schedule

```bash
#!/bin/bash
# /usr/local/bin/monthly-restore-drill.sh
# รัน restore drill ทุกเดือน บน test environment

set -euo pipefail

DRILL_DATE=$(date +%Y%m)
LOG_FILE="/var/log/restore-drill-$DRILL_DATE.log"
TEST_DATA_DIR="/tmp/restore-drill-$DRILL_DATE"

echo "[$(date)] === MONTHLY RESTORE DRILL ===" | tee "$LOG_FILE"

# Step 1: สร้าง test instance
mkdir -p "$TEST_DATA_DIR"
export PGDATA="$TEST_DATA_DIR/pgdata"

echo "[$(date)] Starting restore on test instance..." | tee -a "$LOG_FILE"
START_TIME=$(date +%s)

# Step 2: Restore จาก backup ล่าสุด
sudo -u postgres pgbackrest \
  --stanza=chuaikan \
  --pg1-path="$TEST_DATA_DIR/pgdata" \
  --pg1-port=5499 \
  --delta \
  restore 2>&1 | tee -a "$LOG_FILE"

RESTORE_TIME=$(($(date +%s) - START_TIME))
echo "[$(date)] Restore completed in ${RESTORE_TIME}s" | tee -a "$LOG_FILE"

# Step 3: Start test PostgreSQL
sudo -u postgres pg_ctl \
  -D "$TEST_DATA_DIR/pgdata" \
  -l "$TEST_DATA_DIR/pg.log" \
  -o "-p 5499" \
  start

sleep 5

# Step 4: ทดสอบว่าข้อมูลอยู่ครบ
echo "[$(date)] Validating restored data..." | tee -a "$LOG_FILE"

RESULTS=$(psql -h localhost -p 5499 -U postgres -d chuaikan_db << 'EOSQL'
SELECT
  (SELECT COUNT(*) FROM users) AS user_count,
  (SELECT COUNT(*) FROM posts) AS post_count,
  (SELECT COUNT(*) FROM sos_reports) AS sos_count,
  (SELECT MAX(created_at) FROM users) AS latest_user;
EOSQL
)
echo "$RESULTS" | tee -a "$LOG_FILE"

# Step 5: RTO check
if [ $RESTORE_TIME -lt 3600 ]; then
  echo "[$(date)] ✅ RTO PASS: ${RESTORE_TIME}s < 3600s target" | tee -a "$LOG_FILE"
else
  echo "[$(date)] ❌ RTO FAIL: ${RESTORE_TIME}s > 3600s target" | tee -a "$LOG_FILE"
fi

# Step 6: หยุด test instance
sudo -u postgres pg_ctl -D "$TEST_DATA_DIR/pgdata" stop

# Step 7: Cleanup
rm -rf "$TEST_DATA_DIR"

# Step 8: ส่ง report ไป Slack
curl -s -X POST "$SLACK_WEBHOOK_URL" \
  -H 'Content-type: application/json' \
  -d "{
    \"text\": \"📊 Monthly Restore Drill Result\n\`\`\`$(cat $LOG_FILE | tail -20)\`\`\`\"
  }"

echo "[$(date)] Drill complete!" | tee -a "$LOG_FILE"
```

```bash
# ตั้ง cron สำหรับ monthly drill
sudo crontab -e
# 0 3 1 * * /usr/local/bin/monthly-restore-drill.sh

# ทดสอบ drill ครั้งแรก
sudo /usr/local/bin/monthly-restore-drill.sh
```

---

## 🔧 Configuration Files

```ini
# /etc/pgbackrest/pgbackrest.conf (สรุปสมบูรณ์)
[global]
log-level-console=info
log-level-file=detail
log-path=/var/log/pgbackrest
log-timestamp=y

# Local repository
repo1-path=/var/lib/pgbackrest
repo1-cipher-type=aes-256-cbc
repo1-cipher-pass=<GENERATE_WITH: openssl rand -base64 32>
repo1-retention-full=4
repo1-retention-diff=14
repo1-retention-archive=14

# Cloudflare R2 repository
repo2-type=s3
repo2-s3-bucket=chuaikan-db-backups
repo2-s3-endpoint=<ACCOUNT_ID>.r2.cloudflarestorage.com
repo2-s3-region=auto
repo2-s3-key=<R2_ACCESS_KEY_ID>
repo2-s3-key-secret=<R2_SECRET_KEY>
repo2-path=/
repo2-cipher-type=aes-256-cbc
repo2-cipher-pass=<GENERATE_WITH: openssl rand -base64 32>
repo2-retention-full=12

compress-type=lz4
compress-level=6
process-max=4
start-fast=y

[chuaikan]
pg1-path=/var/lib/postgresql/16/main
pg1-port=5432
pg1-socket-path=/var/run/postgresql
```

---

## 🧪 Testing

```bash
# ทดสอบ backup integrity
sudo -u postgres pgbackrest --stanza=chuaikan verify

# ทดสอบ restore ไป tmp directory
sudo -u postgres pgbackrest \
  --stanza=chuaikan \
  --pg1-path=/tmp/test-restore \
  --pg1-port=5499 \
  --delta \
  restore

# ตรวจสอบข้อมูล
pg_ctl -D /tmp/test-restore -o "-p 5499" start
psql -p 5499 -U postgres -c "SELECT COUNT(*) FROM users;"
pg_ctl -D /tmp/test-restore stop
rm -rf /tmp/test-restore
```

---

## ❌ Common Errors & Solutions

**Error: `archive_command failed with return code 1`**
```bash
# ตรวจสอบ permissions
ls -la /var/lib/pgbackrest
sudo chown -R postgres:postgres /var/lib/pgbackrest

# ทดสอบ archive manually
sudo -u postgres pgbackrest --stanza=chuaikan \
  archive-push /var/lib/postgresql/16/main/pg_wal/000000010000000000000001
```

**Error: `R2 connection refused / 403`**
```bash
# ทดสอบ R2 credentials
aws s3 ls s3://chuaikan-db-backups/ \
  --endpoint-url https://<ACCOUNT_ID>.r2.cloudflarestorage.com

# ตรวจสอบ bucket permissions ใน Cloudflare dashboard
```

**Restore ช้ามาก (> 2 ชั่วโมง)**
```bash
# ใช้ --delta flag (restore เฉพาะ changed blocks)
pgbackrest --stanza=chuaikan --delta restore

# เพิ่ม parallel workers
pgbackrest --stanza=chuaikan --process-max=8 restore
```

---

## ✅ Checklist

- [ ] **Step 491** — อธิบาย 3-2-1 backup rule และ RTO/RPO ได้
- [ ] **Step 492** — เข้าใจความต่าง pgBackRest vs pg_basebackup
- [ ] **Step 493** — Configure pgBackRest กับทั้ง local และ R2 repositories
- [ ] **Step 494** — ตั้งค่า cron jobs: full weekly, diff daily, incr hourly
- [ ] **Step 495** — ทดสอบ PITR restore ไปยังเวลาที่กำหนดสำเร็จ
- [ ] **Step 496** — ทดสอบ restore จาก Cloudflare R2 สำเร็จ
- [ ] **Step 497** — Configure Redis RDB + AOF และทดสอบ restore
- [ ] **Step 498** — กำหนด RTO/RPO targets และวัดผล restore drill
- [ ] **Step 499** — เขียน Disaster Recovery Runbook ครบทุก scenario
- [ ] **Step 500** — ตั้ง monthly restore drill cron job และรัน drill ครั้งแรก

---

## 🔗 References

- [pgBackRest Documentation](https://pgbackrest.org/)
- [Cloudflare R2 + PostgreSQL Backup](https://blog.cloudflare.com/r2-open-beta/)
- [Point-in-Time Recovery Guide](https://www.postgresql.org/docs/current/continuous-archiving.html)
- [Redis Persistence Guide](https://redis.io/docs/manual/persistence/)
- [Disaster Recovery Planning](https://aws.amazon.com/blogs/database/disaster-recovery-strategies-for-amazon-rds/)

---
*Part 050 | Road to 1,000,000 Users/Day | chuaikan.com*
