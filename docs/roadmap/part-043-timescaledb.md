# Part 043: TimescaleDB สำหรับ Flood Sensor Data
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 421–430
> **เวลาโดยประมาณ:** 3 ชั่วโมง
> **Prerequisites:** Part 042 (Read Replica), PostgreSQL พื้นฐาน

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ติดตั้ง TimescaleDB extension บน PostgreSQL 16
- สร้าง Hypertable สำหรับ sensor_readings
- ใช้ Continuous Aggregates สำหรับสรุปข้อมูลรายชั่วโมง/รายวัน
- ตั้งค่า Data Retention Policy (ลบข้อมูลเก่ากว่า 1 ปีอัตโนมัติ)
- ตั้งค่า Compression Policy สำหรับข้อมูลเก่า
- เขียน Time-series queries: last N readings, moving average, anomaly detection
- Integrate กับ SOS service สำหรับ threshold alerts

---

## 📖 ทฤษฎีและแนวคิด

### Step 421 — ทำไมต้อง TimescaleDB?

```
Flood Sensor Data Pattern:
- เซ็นเซอร์ 10,000 จุดทั่วประเทศ
- อ่านค่าทุก 1 นาที = 10,000 × 60 × 24 = 14.4 ล้าน rows/วัน
- ต้องดูข้อมูล 7 วันย้อนหลัง = 100+ ล้าน rows ที่ query บ่อย

ปัญหากับ Regular PostgreSQL Table:
- INSERT ช้าลงเมื่อข้อมูลเยอะ (index B-tree เริ่มใหญ่)
- Range queries ช้า (ต้อง scan ข้ามหลาย pages)
- ลบข้อมูลเก่าช้ามาก (DELETE ทีละ row)

TimescaleDB แก้ปัญหาด้วย:
- Hypertable: แบ่ง data เป็น time-chunks อัตโนมัติ
- Chunk Pruning: query เฉพาะ chunks ที่เกี่ยวข้อง (ลด I/O 90%+)
- Compression: บีบอัด old chunks ประหยัด disk 90%+
- Continuous Aggregates: pre-computed rollups เร็วมาก
```

### Step 422 — TimescaleDB Architecture

```
Hypertable: sensor_readings
├── Chunk: 2024-01-01 ~ 2024-01-08 (1 week interval)
│   ├── Index: (time, sensor_id)
│   └── Data: 100M rows compressed
├── Chunk: 2024-01-08 ~ 2024-01-15
│   └── Data: 100M rows compressed
└── Chunk: 2024-01-15 ~ NOW (active, uncompressed)
    └── Data: 50M rows

Query: SELECT * WHERE time > NOW() - INTERVAL '3 days'
→ TimescaleDB ใช้แค่ 1 chunk (chunk pruning) แทนที่จะ scan ทั้งหมด
```

---

## ⚙️ Environment Setup

### ติดตั้ง TimescaleDB บน Ubuntu 24.04

```bash
# Step 1: เพิ่ม TimescaleDB repository
sudo apt-get install -y gnupg postgresql-common apt-transport-https lsb-release wget

# เพิ่ม GPG key
curl -fsSL https://packagecloud.io/timescale/timescaledb/gpgkey | \
  sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/timescaledb.gpg

# เพิ่ม repository
echo "deb https://packagecloud.io/timescale/timescaledb/ubuntu/ $(lsb_release -c -s) main" | \
  sudo tee /etc/apt/sources.list.d/timescaledb.list

# Step 2: ติดตั้ง
sudo apt-get update
sudo apt-get install -y timescaledb-2-postgresql-16

# Step 3: ปรับ PostgreSQL สำหรับ TimescaleDB
sudo timescaledb-tune --quiet --yes
# timescaledb-tune จะปรับ postgresql.conf อัตโนมัติ

# Step 4: เพิ่มใน shared_preload_libraries
sudo -u postgres psql -c \
  "ALTER SYSTEM SET shared_preload_libraries = 'timescaledb';"

# Step 5: รีสตาร์ท
sudo systemctl restart postgresql

# Step 6: ตรวจสอบ
sudo -u postgres psql -c "SHOW shared_preload_libraries;"
```

### Docker สำหรับ Development

```bash
docker run -d \
  --name timescaledb \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=ts_secret \
  -e POSTGRES_DB=chuaikan_db \
  -v timescaledb_data:/var/lib/postgresql/data \
  timescale/timescaledb:latest-pg16

# เชื่อมต่อ
docker exec -it timescaledb psql -U postgres -d chuaikan_db
```

---

## 🛠️ Step-by-Step Implementation

### Step 423 — สร้าง Hypertable สำหรับ Sensor Readings

```sql
-- เชื่อมต่อกับ database
\c chuaikan_db

-- สร้าง TimescaleDB extension
CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;

-- สร้างตาราง sensor_readings
CREATE TABLE sensor_readings (
    time        TIMESTAMPTZ     NOT NULL,
    sensor_id   VARCHAR(50)     NOT NULL,
    water_level DECIMAL(8, 3),      -- ระดับน้ำ (เมตร)
    flow_rate   DECIMAL(10, 3),     -- อัตราการไหล (m³/s)
    temperature DECIMAL(5, 2),      -- อุณหภูมิ (°C)
    battery_pct SMALLINT,           -- แบตเตอรี่ (%)
    lat         DECIMAL(9, 6),      -- latitude
    lng         DECIMAL(9, 6),      -- longitude
    station_id  VARCHAR(50),        -- รหัสสถานี
    raw_data    JSONB               -- ข้อมูลดิบอื่นๆ
);

-- แปลงเป็น Hypertable (partition ทุก 1 สัปดาห์)
SELECT create_hypertable(
    'sensor_readings',
    'time',
    chunk_time_interval => INTERVAL '1 week',
    if_not_exists => TRUE
);

-- สร้าง index ที่เหมาะสม
CREATE INDEX ON sensor_readings (sensor_id, time DESC);
CREATE INDEX ON sensor_readings (station_id, time DESC);
CREATE INDEX ON sensor_readings USING GIST (
    gist_tsvector_ops(to_tsvector('thai', station_id))
);

-- ตรวจสอบ hypertable
SELECT * FROM timescaledb_information.hypertables
WHERE hypertable_name = 'sensor_readings';
```

### Step 424 — ใส่ข้อมูลทดสอบ

```sql
-- สร้าง function สำหรับ generate test data
CREATE OR REPLACE FUNCTION generate_sensor_data(
    p_days INTEGER DEFAULT 30,
    p_sensors INTEGER DEFAULT 100
)
RETURNS void AS $$
DECLARE
    v_sensor_id VARCHAR;
    v_time TIMESTAMPTZ;
    v_base_level DECIMAL;
BEGIN
    FOR i IN 1..p_sensors LOOP
        v_sensor_id := 'SENSOR_' || LPAD(i::text, 4, '0');
        v_base_level := 0.5 + random() * 2.0;  -- 0.5 - 2.5 เมตร

        -- เพิ่มข้อมูลทุก 1 นาที ย้อนหลัง p_days วัน
        INSERT INTO sensor_readings (
            time, sensor_id, water_level, temperature, battery_pct, lat, lng
        )
        SELECT
            generate_series AS time,
            v_sensor_id,
            v_base_level + sin(EXTRACT(EPOCH FROM generate_series) / 3600) * 0.3
              + random() * 0.1 AS water_level,
            25.0 + random() * 10 AS temperature,
            (80 + random() * 20)::smallint AS battery_pct,
            13.0 + random() * 5 AS lat,
            100.0 + random() * 5 AS lng
        FROM generate_series(
            NOW() - (p_days || ' days')::interval,
            NOW(),
            '1 minute'::interval
        );
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- Generate 30 วัน × 100 เซ็นเซอร์ (4.3 ล้าน rows)
SELECT generate_sensor_data(30, 100);

-- ตรวจสอบจำนวน rows
SELECT COUNT(*) FROM sensor_readings;

-- ดู chunks ที่ถูกสร้าง
SELECT chunk_name, range_start, range_end, is_compressed
FROM timescaledb_information.chunks
WHERE hypertable_name = 'sensor_readings'
ORDER BY range_start DESC;
```

### Step 425 — Continuous Aggregates

```sql
-- Continuous Aggregate: สรุปทุก 1 ชั่วโมง
CREATE MATERIALIZED VIEW sensor_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    sensor_id,
    station_id,
    AVG(water_level)    AS avg_water_level,
    MAX(water_level)    AS max_water_level,
    MIN(water_level)    AS min_water_level,
    AVG(temperature)    AS avg_temperature,
    MIN(battery_pct)    AS min_battery,
    COUNT(*)            AS reading_count
FROM sensor_readings
GROUP BY bucket, sensor_id, station_id
WITH NO DATA;  -- ไม่ backfill ทันที

-- Refresh ข้อมูลเก่าตั้งแต่ต้น
CALL refresh_continuous_aggregate('sensor_hourly',
    NOW() - INTERVAL '30 days',
    NOW()
);

-- เพิ่ม policy ให้ refresh อัตโนมัติ
SELECT add_continuous_aggregate_policy('sensor_hourly',
    start_offset => INTERVAL '3 hours',   -- refresh ข้อมูลใหม่กว่า 3 ชม.
    end_offset   => INTERVAL '10 minutes', -- ไม่ refresh ข้อมูลล่าสุด 10 นาที
    schedule_interval => INTERVAL '1 hour' -- รัน hourly
);

-- Continuous Aggregate: สรุปรายวัน (base on hourly aggregate!)
CREATE MATERIALIZED VIEW sensor_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', bucket) AS day,
    sensor_id,
    station_id,
    AVG(avg_water_level) AS avg_water_level,
    MAX(max_water_level) AS max_water_level,
    MIN(min_water_level) AS min_water_level,
    SUM(reading_count)   AS total_readings
FROM sensor_hourly
GROUP BY day, sensor_id, station_id
WITH NO DATA;

CALL refresh_continuous_aggregate('sensor_daily',
    NOW() - INTERVAL '30 days',
    NOW()
);

SELECT add_continuous_aggregate_policy('sensor_daily',
    start_offset => INTERVAL '3 days',
    end_offset   => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 day'
);

-- ตรวจสอบ policies
SELECT * FROM timescaledb_information.continuous_aggregate_policies;
```

### Step 426 — Data Retention Policy

```sql
-- ลบข้อมูล raw readings เก่ากว่า 1 ปีอัตโนมัติ
SELECT add_retention_policy('sensor_readings',
    drop_after => INTERVAL '1 year',
    if_not_exists => TRUE
);

-- ลบ hourly aggregate เก่ากว่า 3 ปี
SELECT add_retention_policy('sensor_hourly',
    drop_after => INTERVAL '3 years',
    if_not_exists => TRUE
);

-- ลบ daily aggregate เก่ากว่า 10 ปี
SELECT add_retention_policy('sensor_daily',
    drop_after => INTERVAL '10 years',
    if_not_exists => TRUE
);

-- ตรวจสอบ policies
SELECT * FROM timescaledb_information.jobs
WHERE proc_name = 'policy_retention';

-- Run manually เพื่อทดสอบ
SELECT run_job(job_id)
FROM timescaledb_information.jobs
WHERE proc_name = 'policy_retention'
LIMIT 1;
```

### Step 427 — Compression Policy

```sql
-- เปิด compression สำหรับ sensor_readings
ALTER TABLE sensor_readings SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'sensor_id',  -- segment by sensor
    timescaledb.compress_orderby = 'time DESC'      -- order within segment
);

-- Auto-compress chunks เก่ากว่า 7 วัน
SELECT add_compression_policy('sensor_readings',
    compress_after => INTERVAL '7 days',
    if_not_exists => TRUE
);

-- Compress manually เพื่อดูผล
SELECT compress_chunk(chunk_name)
FROM show_chunks('sensor_readings',
    older_than => NOW() - INTERVAL '7 days'
);

-- ดูประสิทธิภาพ compression
SELECT
    chunk_name,
    pg_size_pretty(before_compression_total_bytes) AS before,
    pg_size_pretty(after_compression_total_bytes) AS after,
    ROUND(100.0 * (1 - after_compression_total_bytes::float
        / NULLIF(before_compression_total_bytes, 0)), 1) AS compression_pct
FROM chunk_compression_stats('sensor_readings')
WHERE after_compression_total_bytes IS NOT NULL
ORDER BY range_start DESC
LIMIT 10;
```

### Step 428 — Time-series Queries

```sql
-- === Query 1: Last N Readings สำหรับ Sensor หนึ่ง ===
SELECT time, water_level, temperature, battery_pct
FROM sensor_readings
WHERE sensor_id = 'SENSOR_0001'
  AND time > NOW() - INTERVAL '24 hours'
ORDER BY time DESC
LIMIT 100;

-- === Query 2: Moving Average (24 ชั่วโมง) ===
SELECT
    time,
    sensor_id,
    water_level,
    AVG(water_level) OVER (
        PARTITION BY sensor_id
        ORDER BY time
        ROWS BETWEEN 1440 PRECEDING AND CURRENT ROW  -- 1440 นาที = 24 ชม.
    ) AS moving_avg_24h
FROM sensor_readings
WHERE sensor_id = 'SENSOR_0001'
  AND time > NOW() - INTERVAL '7 days'
ORDER BY time DESC;

-- === Query 3: Anomaly Detection (Z-score) ===
WITH stats AS (
    SELECT
        sensor_id,
        AVG(water_level)    AS mean_level,
        STDDEV(water_level) AS std_level
    FROM sensor_readings
    WHERE time > NOW() - INTERVAL '7 days'
    GROUP BY sensor_id
)
SELECT
    r.time,
    r.sensor_id,
    r.water_level,
    s.mean_level,
    s.std_level,
    ABS(r.water_level - s.mean_level) / NULLIF(s.std_level, 0) AS z_score
FROM sensor_readings r
JOIN stats s ON s.sensor_id = r.sensor_id
WHERE r.time > NOW() - INTERVAL '1 hour'
  AND ABS(r.water_level - s.mean_level) / NULLIF(s.std_level, 0) > 3
  -- Z-score > 3 = anomaly (ระดับน้ำผิดปกติมาก)
ORDER BY z_score DESC;

-- === Query 4: ใช้ Continuous Aggregate (เร็วกว่า raw มาก) ===
SELECT bucket, sensor_id, avg_water_level, max_water_level
FROM sensor_hourly
WHERE bucket > NOW() - INTERVAL '30 days'
  AND sensor_id = 'SENSOR_0001'
ORDER BY bucket DESC;

-- === Query 5: Top 10 sensors ที่มี water_level สูงสุดวันนี้ ===
SELECT
    sensor_id,
    station_id,
    MAX(max_water_level) AS peak_today,
    AVG(avg_water_level) AS avg_today
FROM sensor_daily
WHERE day = CURRENT_DATE
ORDER BY peak_today DESC
LIMIT 10;
```

### Step 429 — Integration กับ SOS Service

```typescript
// services/flood-alert/alert-checker.ts
import { prisma } from '../db/client';

interface AlertConfig {
  sensorId: string;
  warningThreshold: number;    // เมตร
  criticalThreshold: number;   // เมตร
  dangerThreshold: number;     // เมตร
}

interface SensorAlert {
  sensorId: string;
  currentLevel: number;
  alertLevel: 'warning' | 'critical' | 'danger';
  trend: 'rising' | 'falling' | 'stable';
  lat: number;
  lng: number;
}

export async function checkFloodAlerts(
  configs: AlertConfig[]
): Promise<SensorAlert[]> {
  const alerts: SensorAlert[] = [];

  for (const config of configs) {
    // ดูค่าล่าสุดและ trend
    const recentReadings = await prisma.$queryRaw<any[]>`
      SELECT
        sensor_id,
        time,
        water_level,
        lat,
        lng,
        -- trend: เปรียบเทียบ 30 นาทีล่าสุด กับ 30 นาทีก่อนหน้า
        water_level - LAG(water_level, 30) OVER (
          PARTITION BY sensor_id ORDER BY time
        ) AS level_change_30m
      FROM sensor_readings
      WHERE sensor_id = ${config.sensorId}
        AND time > NOW() - INTERVAL '1 hour'
      ORDER BY time DESC
      LIMIT 1
    `;

    if (recentReadings.length === 0) continue;

    const latest = recentReadings[0];
    const level = parseFloat(latest.water_level);
    const change = parseFloat(latest.level_change_30m || '0');

    // ตรวจสอบ threshold
    let alertLevel: SensorAlert['alertLevel'] | null = null;
    if (level >= config.dangerThreshold) alertLevel = 'danger';
    else if (level >= config.criticalThreshold) alertLevel = 'critical';
    else if (level >= config.warningThreshold) alertLevel = 'warning';

    if (alertLevel) {
      alerts.push({
        sensorId: config.sensorId,
        currentLevel: level,
        alertLevel,
        trend: change > 0.1 ? 'rising' : change < -0.1 ? 'falling' : 'stable',
        lat: parseFloat(latest.lat),
        lng: parseFloat(latest.lng),
      });
    }
  }

  return alerts;
}

// ส่ง SOS alert ไปยัง user บริเวณใกล้เคียง
export async function notifyNearbyUsers(alert: SensorAlert) {
  const RADIUS_KM = 5;

  // ดู users ที่อยู่ในรัศมี 5 กม. (ต้องใช้ PostGIS - Part 044)
  const nearbyUsers = await prisma.$queryRaw<any[]>`
    SELECT u.id, u.push_token
    FROM users u
    WHERE ST_DWithin(
      u.location::geography,
      ST_MakePoint(${alert.lng}, ${alert.lat})::geography,
      ${RADIUS_KM * 1000}
    )
    AND u.flood_alerts_enabled = true
  `;

  console.log(
    `Sending ${alert.alertLevel} alert to ${nearbyUsers.length} users ` +
    `near sensor ${alert.sensorId}`
  );

  // ส่ง push notification (implement ใน Part 060+)
  return nearbyUsers.length;
}
```

### Step 430 — Query Performance Comparison

```sql
-- เปรียบเทียบ: raw table vs continuous aggregate

-- Raw query (ช้ากว่า)
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT
    DATE_TRUNC('hour', time) AS hour,
    sensor_id,
    AVG(water_level)
FROM sensor_readings
WHERE time > NOW() - INTERVAL '30 days'
GROUP BY DATE_TRUNC('hour', time), sensor_id;

-- Continuous Aggregate (เร็วกว่ามาก)
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT bucket, sensor_id, avg_water_level
FROM sensor_hourly
WHERE bucket > NOW() - INTERVAL '30 days';

-- ดู chunk pruning ใน action
EXPLAIN SELECT COUNT(*)
FROM sensor_readings
WHERE time > NOW() - INTERVAL '7 days';
-- สังเกต: "Chunks excluded due to constraints: X"
```

---

## 🔧 Configuration Files

```ini
# postgresql.conf — TimescaleDB optimized
shared_preload_libraries = 'timescaledb,pg_stat_statements'
timescaledb.max_background_workers = 8  # workers สำหรับ background jobs
max_worker_processes = 16               # ต้องมากกว่า background_workers

# Memory สำหรับ time-series
work_mem = 128MB
maintenance_work_mem = 1GB
effective_cache_size = 8GB
```

---

## 🧪 Testing

```bash
# ทดสอบ query performance
time psql -U postgres -d chuaikan_db -c \
  "SELECT bucket, AVG(avg_water_level)
   FROM sensor_hourly
   WHERE bucket > NOW() - INTERVAL '30 days'
   GROUP BY bucket ORDER BY bucket DESC LIMIT 720;"

# ทดสอบ compression ratio
psql -U postgres -d chuaikan_db -c \
  "SELECT
     pg_size_pretty(SUM(before_compression_total_bytes)) AS before,
     pg_size_pretty(SUM(after_compression_total_bytes)) AS after
   FROM chunk_compression_stats('sensor_readings')
   WHERE after_compression_total_bytes IS NOT NULL;"

# ทดสอบ retention policy
psql -U postgres -d chuaikan_db -c \
  "SELECT show_chunks('sensor_readings',
     older_than => NOW() - INTERVAL '1 year');"
```

---

## ❌ Common Errors & Solutions

**Error: `ERROR: could not create unique index`**
```sql
-- TimescaleDB ต้องการ time column ใน primary key ของ hypertable
-- ❌ ผิด
ALTER TABLE sensor_readings ADD PRIMARY KEY (id);
-- ✅ ถูก
ALTER TABLE sensor_readings ADD PRIMARY KEY (id, time);
```

**Error: `continuous aggregate refresh is taking too long`**
```sql
-- แบ่ง refresh เป็นช่วงเล็กๆ
CALL refresh_continuous_aggregate('sensor_hourly',
    NOW() - INTERVAL '7 days',
    NOW() - INTERVAL '6 days'
);
-- ทำซ้ำสำหรับแต่ละสัปดาห์
```

**Compression ทำให้ UPDATE ไม่ได้**
```sql
-- ต้อง decompress chunk ก่อน UPDATE
SELECT decompress_chunk(chunk_name)
FROM show_chunks('sensor_readings', older_than => '7 days')
LIMIT 1;

-- Update แล้ว compress ใหม่
UPDATE sensor_readings SET water_level = 1.5 WHERE ...;

SELECT compress_chunk(chunk_name)
FROM show_chunks('sensor_readings', older_than => '7 days')
LIMIT 1;
```

---

## ✅ Checklist

- [ ] **Step 421** — อธิบาย Hypertable และ chunk pruning ได้
- [ ] **Step 422** — เข้าใจว่า TimescaleDB ดีกว่า regular table อย่างไร
- [ ] **Step 423** — สร้าง `sensor_readings` hypertable ได้
- [ ] **Step 424** — Insert test data 1M+ rows ได้
- [ ] **Step 425** — สร้าง Continuous Aggregate hourly และ daily ได้
- [ ] **Step 426** — ตั้ง retention policy ลบข้อมูลเก่ากว่า 1 ปี
- [ ] **Step 427** — Compress old chunks และวัด compression ratio
- [ ] **Step 428** — เขียน moving average และ anomaly detection queries
- [ ] **Step 429** — Integrate flood alert checker กับ SOS service
- [ ] **Step 430** — ยืนยันว่า continuous aggregate เร็วกว่า raw query

---

## 🔗 References

- [TimescaleDB Documentation](https://docs.timescale.com/)
- [Continuous Aggregates Guide](https://docs.timescale.com/use-timescale/continuous-aggregates/)
- [Compression Guide](https://docs.timescale.com/use-timescale/compression/)
- [Time-series Data Patterns](https://www.timescale.com/blog/time-series-data/)

---
*Part 043 | Road to 1,000,000 Users/Day | chuaikan.com*
