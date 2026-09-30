# Part 042: Read Replica Configuration
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 411–420
> **เวลาโดยประมาณ:** 3 ชั่วโมง
> **Prerequisites:** Part 041 (Database Sharding), PostgreSQL พื้นฐาน

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ติดตั้ง PostgreSQL Streaming Replication (1 Primary + 2 Replicas)
- ปรับ `postgresql.conf` สำหรับ Primary
- ตั้งค่า `pg_hba.conf` สำหรับ replication user
- Setup Replica ด้วย `pg_basebackup`
- ใช้ PgBouncer routing: read → replicas, write → primary
- Monitor Replication Lag
- Promote Replica เป็น Primary (Failover)

---

## 📖 ทฤษฎีและแนวคิด

### Step 411 — ทำไมต้อง Read Replica?

```
Traffic Pattern chuaikan.com:
- Read  : 80% (ดู flood status, SOS map, user posts)
- Write : 20% (report SOS, update profile, add sensor data)

หากไม่มี Read Replica:
Primary DB ─── 100% traffic ──→ Primary DB
                              (bottleneck!)

หลังเพิ่ม Read Replica:
                ┌──────────────────────────────┐
                │ PgBouncer / HAProxy           │
                └──────────────────────────────┘
                       │                │
               Write 20%           Read 80%
                       │                │
               ┌───────┴───┐    ┌───────┴──────────────┐
               │  Primary  │    │ Replica 1  Replica 2  │
               │  (R/W)    │───▶│ (Read)     (Read)     │
               └───────────┘WAL └──────────────────────┘
```

### Step 412 — PostgreSQL Streaming Replication ทำงานอย่างไร?

```
1. Primary เขียน changes ลง WAL (Write-Ahead Log)
2. WAL Sender process ส่ง WAL records ไปยัง Replica
3. WAL Receiver ใน Replica รับและ apply changes
4. ข้อมูลใน Replica sync กับ Primary แบบ near real-time

Replication Modes:
- Asynchronous: Primary ไม่รอ Replica confirm (ค่าเริ่มต้น, เร็วกว่า)
- Synchronous: Primary รอ Replica ≥1 ตัว confirm (ปลอดภัยกว่า, ช้ากว่า)
```

---

## ⚙️ Environment Setup

### เตรียม 3 เครื่อง (หรือ Docker containers)

```bash
# สร้าง Docker network
docker network create pg-replication-net

# Primary PostgreSQL
docker run -d \
  --name pg-primary \
  --network pg-replication-net \
  -e POSTGRES_PASSWORD=primary_secret \
  -e POSTGRES_DB=chuaikan_db \
  -p 5432:5432 \
  -v pg_primary_data:/var/lib/postgresql/data \
  postgres:16

# Replica 1
docker run -d \
  --name pg-replica-1 \
  --network pg-replication-net \
  -e POSTGRES_PASSWORD=replica_secret \
  -p 5433:5432 \
  -v pg_replica1_data:/var/lib/postgresql/data \
  postgres:16

# Replica 2
docker run -d \
  --name pg-replica-2 \
  --network pg-replication-net \
  -e POSTGRES_PASSWORD=replica_secret \
  -p 5434:5432 \
  -v pg_replica2_data:/var/lib/postgresql/data \
  postgres:16
```

---

## 🛠️ Step-by-Step Implementation

### Step 413 — ปรับค่า Primary postgresql.conf

```bash
# เชื่อมต่อ Primary
docker exec -it pg-primary bash

# แก้ไข postgresql.conf
cat >> /var/lib/postgresql/data/postgresql.conf << 'EOF'

# === Replication Settings (Primary) ===
wal_level = replica              # ต้องเป็น replica หรือ logical
max_wal_senders = 10             # จำนวน replica ที่เชื่อมได้สูงสุด
wal_keep_size = 1GB              # เก็บ WAL ไว้ขนาดนี้เผื่อ replica ช้า
max_replication_slots = 10       # สำหรับ logical replication slots
hot_standby = on                 # ให้ replica รับ read queries ได้
synchronous_commit = on          # async mode (เร็วกว่า, ความเสี่ยงเล็กน้อย)

# === สำหรับ monitoring ===
track_commit_timestamp = on      # ติดตาม lag ได้แม่นขึ้น
EOF

# รีสตาร์ท Primary
docker restart pg-primary
```

### Step 414 — ตั้งค่า pg_hba.conf สำหรับ Replication

```bash
# สร้าง replication user ก่อน
docker exec -it pg-primary psql -U postgres << 'EOF'
-- สร้าง replication user
CREATE ROLE replicator
  WITH REPLICATION LOGIN PASSWORD 'repl_secret_2024';

-- ตรวจสอบ
\du replicator
EOF

# เพิ่มกฎใน pg_hba.conf
docker exec pg-primary bash -c "cat >> /var/lib/postgresql/data/pg_hba.conf << 'EOF'

# Allow replication connections from replica containers
# TYPE  DATABASE        USER        ADDRESS         METHOD
host    replication     replicator  172.0.0.0/8     scram-sha-256
host    replication     replicator  10.0.0.0/8      scram-sha-256
EOF"

# Reload config (ไม่ต้อง restart)
docker exec pg-primary psql -U postgres -c "SELECT pg_reload_conf();"

# ตรวจสอบ pg_hba.conf มีผล
docker exec pg-primary psql -U postgres -c \
  "SELECT * FROM pg_hba_file_rules WHERE database = '{replication}';"
```

### Step 415 — Setup Replica ด้วย pg_basebackup

```bash
# === Setup Replica 1 ===

# หยุด PostgreSQL ใน replica-1 ก่อน (เราจะแทนที่ data directory)
docker exec pg-replica-1 pg_ctlcluster 16 main stop 2>/dev/null || true

# ลบ data directory เดิม
docker exec pg-replica-1 rm -rf /var/lib/postgresql/data/*

# ทำ base backup จาก Primary
docker exec pg-replica-1 pg_basebackup \
  --host=pg-primary \
  --username=replicator \
  --pgdata=/var/lib/postgresql/data \
  --wal-method=stream \
  --checkpoint=fast \
  --progress \
  --verbose \
  --no-password \
  --no-sync

# หมายเหตุ: ใส่ password ผ่าน .pgpass
docker exec pg-replica-1 bash -c \
  "echo 'pg-primary:5432:replication:replicator:repl_secret_2024' \
   > ~/.pgpass && chmod 600 ~/.pgpass"

# สร้าง recovery signal (บอกว่านี่คือ standby)
docker exec pg-replica-1 touch /var/lib/postgresql/data/standby.signal

# เพิ่ม replication settings ใน postgresql.conf ของ Replica
docker exec pg-replica-1 bash -c "cat >> /var/lib/postgresql/data/postgresql.conf << 'EOF'

# === Replica Settings ===
hot_standby = on
primary_conninfo = 'host=pg-primary port=5432 user=replicator password=repl_secret_2024 application_name=replica1'
primary_slot_name = 'replica1_slot'
recovery_min_apply_delay = 0    # 0 = sync เร็วที่สุด
EOF"

# สร้าง replication slot บน Primary
docker exec pg-primary psql -U postgres -c \
  "SELECT pg_create_physical_replication_slot('replica1_slot');"

# รีสตาร์ท Replica 1
docker restart pg-replica-1

# ทำซ้ำสำหรับ Replica 2
docker exec pg-replica-2 rm -rf /var/lib/postgresql/data/*
docker exec pg-replica-2 bash -c \
  "echo 'pg-primary:5432:replication:replicator:repl_secret_2024' \
   > ~/.pgpass && chmod 600 ~/.pgpass"

docker exec pg-replica-2 pg_basebackup \
  --host=pg-primary \
  --username=replicator \
  --pgdata=/var/lib/postgresql/data \
  --wal-method=stream \
  --checkpoint=fast \
  --progress --verbose --no-password --no-sync

docker exec pg-replica-2 touch /var/lib/postgresql/data/standby.signal

docker exec pg-replica-2 bash -c "cat >> /var/lib/postgresql/data/postgresql.conf << 'EOF'

hot_standby = on
primary_conninfo = 'host=pg-primary port=5432 user=replicator password=repl_secret_2024 application_name=replica2'
primary_slot_name = 'replica2_slot'
EOF"

docker exec pg-primary psql -U postgres -c \
  "SELECT pg_create_physical_replication_slot('replica2_slot');"

docker restart pg-replica-2

# ตรวจสอบ replication status
docker exec pg-primary psql -U postgres -c \
  "SELECT client_addr, application_name, state, sent_lsn, write_lsn,
          flush_lsn, replay_lsn, sync_state
   FROM pg_stat_replication;"
```

### Step 416 — ติดตั้งและ Configure PgBouncer

```bash
# ติดตั้ง PgBouncer
sudo apt-get install -y pgbouncer

# หรือ Docker
docker run -d \
  --name pgbouncer \
  --network pg-replication-net \
  -p 6432:6432 \
  -v /etc/pgbouncer:/etc/pgbouncer \
  edoburu/pgbouncer
```

```ini
# /etc/pgbouncer/pgbouncer.ini

[databases]
; Write connections → Primary
chuaikan_primary = host=pg-primary port=5432 dbname=chuaikan_db
; Read connections → Replica 1 (load balanced)
chuaikan_replica = host=pg-replica-1,pg-replica-2 port=5432 dbname=chuaikan_db

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

; Connection pooling mode
pool_mode = transaction       ; transaction-level pooling (เหมาะสำหรับ web app)
max_client_conn = 1000        ; สูงสุด 1000 client connections
default_pool_size = 25        ; 25 connections ต่อ database-user pair

; Timeouts
server_idle_timeout = 600
client_idle_timeout = 0
connect_timeout = 15

logfile = /var/log/pgbouncer/pgbouncer.log
pidfile = /var/run/pgbouncer/pgbouncer.pid

; Stats
stats_period = 60
```

```bash
# สร้าง userlist.txt
echo '"app_user" "md5_hash_of_password"' > /etc/pgbouncer/userlist.txt

# สร้าง MD5 hash: md5(password + username)
echo -n "app_password_2024app_user" | md5sum
# ตัวอย่าง: "md5" + hash

# รีสตาร์ท PgBouncer
sudo systemctl restart pgbouncer
sudo systemctl status pgbouncer
```

### Step 417 — Application Routing: Write vs Read

```typescript
// lib/db/connection.ts
import { PrismaClient } from '@prisma/client';
import { Pool } from 'pg';

// Write: ไปที่ Primary เสมอ
const primaryPool = new Pool({
  host: process.env.DB_PRIMARY_HOST || 'localhost',
  port: 6432,  // PgBouncer port
  database: 'chuaikan_primary',  // PgBouncer database alias
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  max: 20,
});

// Read: ไปที่ Replica (round-robin โดย PgBouncer)
const replicaPool = new Pool({
  host: process.env.DB_BOUNCER_HOST || 'localhost',
  port: 6432,
  database: 'chuaikan_replica',  // PgBouncer database alias
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  max: 50,  // อนุญาตมากกว่าเพราะ read traffic เยอะกว่า
});

// Prisma client สำหรับ Primary
export const prismaWrite = new PrismaClient({
  datasources: {
    db: {
      url: `postgresql://${process.env.DB_USER}:${process.env.DB_PASSWORD}@${process.env.DB_PRIMARY_HOST}:6432/chuaikan_primary`,
    },
  },
});

// Prisma client สำหรับ Replica
export const prismaRead = new PrismaClient({
  datasources: {
    db: {
      url: `postgresql://${process.env.DB_USER}:${process.env.DB_PASSWORD}@${process.env.DB_BOUNCER_HOST}:6432/chuaikan_replica`,
    },
  },
});

// Helper function
export function getDb(isWrite: boolean = false) {
  return isWrite ? prismaWrite : prismaRead;
}

// ตัวอย่างการใช้งาน
export async function createSosReport(data: {
  userId: string;
  lat: number;
  lng: number;
  description: string;
}) {
  // Write: ใช้ Primary
  return prismaWrite.sosReport.create({ data });
}

export async function getNearbySOSReports(lat: number, lng: number) {
  // Read: ใช้ Replica
  return prismaRead.$queryRaw`
    SELECT * FROM sos_reports
    WHERE created_at > NOW() - INTERVAL '24 hours'
    ORDER BY created_at DESC
    LIMIT 50
  `;
}
```

### Step 418 — Monitor Replication Lag

```sql
-- บน Primary: ดู replication lag
SELECT
    client_addr,
    application_name,
    state,
    pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn) AS send_lag_bytes,
    pg_wal_lsn_diff(sent_lsn, write_lsn) AS write_lag_bytes,
    pg_wal_lsn_diff(write_lsn, flush_lsn) AS flush_lag_bytes,
    pg_wal_lsn_diff(flush_lsn, replay_lsn) AS replay_lag_bytes,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;

-- บน Replica: ดู lag ของตัวเอง
SELECT
    now() - pg_last_xact_replay_timestamp() AS replication_lag,
    pg_is_in_recovery() AS is_replica,
    pg_last_wal_replay_lsn() AS last_replay_lsn;
```

```typescript
// lib/monitoring/replication-lag.ts
import { prismaWrite } from '../db/connection';

interface ReplicationStats {
  applicationName: string;
  replayLagSeconds: number;
  isHealthy: boolean;
}

export async function checkReplicationLag(): Promise<ReplicationStats[]> {
  const result = await prismaWrite.$queryRaw<any[]>`
    SELECT
      application_name,
      EXTRACT(EPOCH FROM replay_lag)::float AS replay_lag_seconds,
      state
    FROM pg_stat_replication
  `;

  return result.map(row => ({
    applicationName: row.application_name,
    replayLagSeconds: row.replay_lag_seconds || 0,
    isHealthy: (row.replay_lag_seconds || 0) < 30, // Alert ถ้า lag > 30 วินาที
  }));
}

// ส่ง alert ถ้า lag สูงเกิน threshold
export async function alertOnHighLag(threshold: number = 30) {
  const stats = await checkReplicationLag();

  for (const stat of stats) {
    if (!stat.isHealthy) {
      console.error(
        `[ALERT] Replica ${stat.applicationName} lag: ` +
        `${stat.replayLagSeconds.toFixed(1)}s (threshold: ${threshold}s)`
      );
      // ส่งไป Slack/PagerDuty ที่นี่
    }
  }
}
```

### Step 419 — ตั้งค่า Prometheus + Grafana สำหรับ Monitoring

```bash
# ติดตั้ง postgres_exporter
wget https://github.com/prometheus-community/postgres_exporter/releases/download/v0.15.0/postgres_exporter-0.15.0.linux-amd64.tar.gz
tar xzf postgres_exporter-0.15.0.linux-amd64.tar.gz
sudo mv postgres_exporter-0.15.0.linux-amd64/postgres_exporter /usr/local/bin/

# สร้าง user สำหรับ monitoring
sudo -u postgres psql << 'EOF'
CREATE USER postgres_exporter PASSWORD 'exporter_pass';
GRANT pg_monitor TO postgres_exporter;
EOF

# Config file
cat > /etc/postgres_exporter/queries.yaml << 'EOF'
pg_replication:
  query: |
    SELECT
      CASE WHEN pg_is_in_recovery() THEN 1 ELSE 0 END AS is_replica,
      EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp()))::float
        AS replication_lag_seconds
  metrics:
    - is_replica:
        usage: GAUGE
        description: "1 if this is a replica"
    - replication_lag_seconds:
        usage: GAUGE
        description: "Replication lag in seconds"
EOF

# systemd service
cat > /etc/systemd/system/postgres_exporter.service << 'EOF'
[Unit]
Description=Prometheus PostgreSQL Exporter
After=network.target

[Service]
User=postgres
Environment="DATA_SOURCE_NAME=postgresql://postgres_exporter:exporter_pass@localhost:5432/postgres?sslmode=disable"
ExecStart=/usr/local/bin/postgres_exporter \
  --extend.query-path=/etc/postgres_exporter/queries.yaml \
  --web.listen-address=:9187
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now postgres_exporter
```

### Step 420 — Failover: Promote Replica เป็น Primary

```bash
# === กรณีฉุกเฉิน: Primary ล้มเหลว ===

# Step 1: ตรวจสอบสถานะ Primary
pg_isready -h pg-primary -p 5432
# ถ้าได้ "no response" = Primary ล้มเหลว

# Step 2: เลือก Replica ที่มี lag น้อยที่สุด
docker exec pg-replica-1 psql -U postgres -c \
  "SELECT pg_last_wal_replay_lsn();"
docker exec pg-replica-2 psql -U postgres -c \
  "SELECT pg_last_wal_replay_lsn();"
# เลือกตัวที่ LSN สูงกว่า

# Step 3: Promote Replica-1 เป็น Primary
docker exec pg-replica-1 pg_ctl promote \
  -D /var/lib/postgresql/data

# หรือใช้ SQL command
docker exec pg-replica-1 psql -U postgres -c \
  "SELECT pg_promote();"

# Step 4: ตรวจสอบ Promotion สำเร็จ
docker exec pg-replica-1 psql -U postgres -c \
  "SELECT pg_is_in_recovery();"  -- ควรได้ false

# Step 5: อัพเดท PgBouncer ให้ชี้ Primary ใหม่
# แก้ pgbouncer.ini
sed -i 's/host=pg-primary/host=pg-replica-1/' /etc/pgbouncer/pgbouncer.ini
# Reload PgBouncer โดยไม่ drop connections
kill -HUP $(cat /var/run/pgbouncer/pgbouncer.pid)

# Step 6: ตั้งค่า Replica-2 ให้ follow Primary ใหม่
docker exec pg-replica-2 bash -c "
cat > /var/lib/postgresql/data/postgresql.conf.d/recovery.conf << 'EOF'
primary_conninfo = 'host=pg-replica-1 port=5432 user=replicator password=repl_secret_2024'
primary_slot_name = 'replica2_slot_new'
EOF
"
docker restart pg-replica-2
```

---

## 🔧 Configuration Files

```ini
# /etc/postgresql/16/main/postgresql.conf — Primary settings
wal_level = replica
max_wal_senders = 10
wal_keep_size = 1024          # 1GB
synchronous_commit = on
track_commit_timestamp = on
max_replication_slots = 10
hot_standby_feedback = on     # replica บอก primary ว่ากำลัง query อะไร
                               # ป้องกัน primary ลบ dead rows ที่ replica ยังใช้อยู่
```

```bash
# /etc/pgbouncer/pgbouncer.ini (สรุป)
[databases]
chuaikan_primary = host=10.0.0.1 port=5432 dbname=chuaikan_db
chuaikan_replica = host=10.0.0.2,10.0.0.3 port=5432 dbname=chuaikan_db \
  load_balance_hosts=random

[pgbouncer]
pool_mode = transaction
max_client_conn = 2000
default_pool_size = 30
min_pool_size = 5
reserve_pool_size = 5
reserve_pool_timeout = 3
```

---

## 🧪 Testing

```bash
# ทดสอบ replication ทำงาน
# 1. สร้างข้อมูลบน Primary
docker exec pg-primary psql -U postgres -d chuaikan_db -c \
  "INSERT INTO users(username, email) VALUES ('test_repl', 'test@chuaikan.com');"

# 2. ตรวจสอบว่าข้อมูลมาถึง Replica ภายใน 1 วินาที
sleep 1
docker exec pg-replica-1 psql -U postgres -d chuaikan_db -c \
  "SELECT * FROM users WHERE username = 'test_repl';"

# ทดสอบ PgBouncer routing
# Write ผ่าน primary alias
psql -h localhost -p 6432 -U app_user -d chuaikan_primary \
  -c "INSERT INTO test_table VALUES (1, 'write test');"

# Read ผ่าน replica alias
psql -h localhost -p 6432 -U app_user -d chuaikan_replica \
  -c "SELECT * FROM test_table LIMIT 5;"

# Load test (pgbench)
pgbench -h localhost -p 6432 -U app_user \
  -d chuaikan_replica \
  -c 100 -j 8 -T 60 -S  # -S = SELECT only
```

---

## ❌ Common Errors & Solutions

**Error: `FATAL: could not connect to the primary server`**
```bash
# ตรวจสอบ pg_hba.conf บน Primary อนุญาต replica IP หรือยัง
docker exec pg-primary cat /var/lib/postgresql/data/pg_hba.conf | grep replication

# ตรวจสอบ firewall
sudo ufw status
sudo ufw allow from 172.0.0.0/8 to any port 5432
```

**Error: `replication slot already exists`**
```sql
-- ดู slots ที่มีอยู่
SELECT * FROM pg_replication_slots;

-- ลบ slot ที่ไม่ใช้
SELECT pg_drop_replication_slot('replica1_slot');
-- แล้วสร้างใหม่
SELECT pg_create_physical_replication_slot('replica1_slot');
```

**Error: PgBouncer `Auth failed` แม้ password ถูก**
```bash
# ตรวจสอบว่า auth_type ตรงกัน
# PostgreSQL 14+ ใช้ scram-sha-256 เป็นค่าเริ่มต้น
# PgBouncer รุ่นเก่า (< 1.17) ไม่รองรับ scram-sha-256

# แก้: อัพเกรด PgBouncer หรือเปลี่ยน pg_hba.conf เป็น md5
host  all  app_user  0.0.0.0/0  md5
```

---

## ✅ Checklist

- [ ] **Step 411** — อธิบาย Streaming Replication architecture ได้
- [ ] **Step 412** — เข้าใจ Async vs Sync replication trade-offs
- [ ] **Step 413** — ตั้งค่า `postgresql.conf` Primary: `wal_level = replica`
- [ ] **Step 414** — สร้าง replication user และตั้งค่า `pg_hba.conf`
- [ ] **Step 415** — Setup Replica สำเร็จด้วย `pg_basebackup`
- [ ] **Step 416** — ติดตั้งและ configure PgBouncer routing
- [ ] **Step 417** — แยก read/write ใน application code ได้
- [ ] **Step 418** — Monitor replication lag ด้วย `pg_stat_replication`
- [ ] **Step 419** — ตั้งค่า postgres_exporter + Grafana dashboard
- [ ] **Step 420** — ทำ manual failover: promote replica เป็น primary สำเร็จ

---

## 🔗 References

- [PostgreSQL Streaming Replication](https://www.postgresql.org/docs/current/warm-standby.html)
- [pg_basebackup Documentation](https://www.postgresql.org/docs/current/app-pgbasebackup.html)
- [PgBouncer Configuration](https://www.pgbouncer.org/config.html)
- [Patroni — Automatic Failover](https://github.com/zalando/patroni)

---
*Part 042 | Road to 1,000,000 Users/Day | chuaikan.com*
