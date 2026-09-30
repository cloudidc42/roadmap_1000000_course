# Part 104: Real-time Analytics Dashboard
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** World Class
> **Steps:** 1031-1040
> **เวลาโดยประมาณ:** 10 ชั่วโมง
> **Prerequisites:** Part 072 (Kafka), Part 009 (Monitoring), Part 004 (PostgreSQL), Part 005 (Redis)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ClickHouse: OLAP database สำหรับ analytics
- Materialized views สำหรับ sub-second aggregations
- Kafka → ClickHouse streaming pipeline
- Grafana dashboard สำหรับ operations
- SOS command center สำหรับ emergency responders
- Real-time anomaly detection
- Grafana alerting ไปยัง Line Notify และ Discord

---

## 📖 ทฤษฎีและแนวคิด

### ทำไมต้องใช้ ClickHouse?

| Database | Query 1B rows | Write Speed | คำอธิบาย |
|----------|--------------|-------------|-----------|
| PostgreSQL | ~60 seconds | 100K rows/s | OLTP, ไม่เหมาะกับ analytics |
| ClickHouse | ~0.1 seconds | 1M rows/s | OLAP, เหมาะมากสำหรับ analytics |
| Redis | N/A | 1M ops/s | Cache, ไม่ใช่ database |

### Architecture

```
App Events (Node.js)
    ↓ Kafka (analytics-events topic)
    ↓ Kafka Table Engine
ClickHouse (analytics DB)
    ↓ Materialized Views
    ↓ HTTP API
Grafana Dashboards ← Query ClickHouse
    ↓ Alert rules
Line Notify / Discord
```

---

## ⚙️ Environment Setup

### Step 1031: ติดตั้ง ClickHouse

```bash
# ติดตั้ง ClickHouse บน Ubuntu 24.04
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
curl -fsSL 'https://packages.clickhouse.com/rpm/lts/repodata/repomd.xml.key' | sudo gpg --dearmour -o /usr/share/keyrings/clickhouse-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/clickhouse-keyring.gpg] https://packages.clickhouse.com/deb stable main" | \
  sudo tee /etc/apt/sources.list.d/clickhouse.list

sudo apt update
sudo apt install -y clickhouse-server clickhouse-client

sudo systemctl enable clickhouse-server
sudo systemctl start clickhouse-server
sudo systemctl status clickhouse-server
```

### Step 1032: ติดตั้ง Grafana

```bash
# Grafana OSS
sudo apt-get install -y apt-transport-https software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo gpg --dearmor -o /usr/share/keyrings/grafana.gpg
echo "deb [signed-by=/usr/share/keyrings/grafana.gpg] https://packages.grafana.com/oss/deb stable main" | \
  sudo tee -a /etc/apt/sources.list.d/grafana.list

sudo apt update
sudo apt install -y grafana

# ติดตั้ง ClickHouse datasource plugin
sudo grafana-cli plugins install grafana-clickhouse-datasource
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
# เปิด http://localhost:3001 (admin/admin)
```

---

## 🛠️ Step-by-Step Implementation

### Step 1033: ClickHouse Schema

```sql
-- สร้าง database
CREATE DATABASE IF NOT EXISTS chuaikan;

-- Events table (raw data)
CREATE TABLE chuaikan.events (
    event_id     String,
    user_id      String,
    action       LowCardinality(String),
    target_id    String,
    target_type  LowCardinality(String),
    metadata     String,  -- JSON
    province     LowCardinality(String),
    timestamp    DateTime64(3),  -- milliseconds precision
    date         Date MATERIALIZED toDate(timestamp)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (timestamp, user_id)
TTL date + INTERVAL 90 DAY;

-- SOS events table
CREATE TABLE chuaikan.sos_events (
    event_id     String,
    post_id      String,
    user_id      String,
    alert_type   LowCardinality(String),
    severity     LowCardinality(String),
    province     LowCardinality(String),
    lat          Float64,
    lng          Float64,
    description  String,
    status       LowCardinality(String) DEFAULT 'active',
    timestamp    DateTime64(3),
    date         Date MATERIALIZED toDate(timestamp)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (timestamp, province, severity)
TTL date + INTERVAL 365 DAY;

-- Push notification analytics
CREATE TABLE chuaikan.push_analytics (
    notification_id  String,
    user_id          String,
    token_platform   LowCardinality(String),
    status           LowCardinality(String),  -- delivered/opened/dismissed/failed
    timestamp        DateTime64(3),
    date             Date MATERIALIZED toDate(timestamp)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (timestamp, status);
```

### Step 1034: Materialized Views สำหรับ Real-time Metrics

```sql
-- Active users ใน 5 นาทีที่ผ่านมา
CREATE MATERIALIZED VIEW chuaikan.active_users_5min
ENGINE = AggregatingMergeTree()
ORDER BY (window_start)
POPULATE AS
SELECT
    toStartOfFiveMinutes(timestamp) as window_start,
    uniqState(user_id) as unique_users
FROM chuaikan.events
GROUP BY window_start;

-- Query: DAU (today)
-- SELECT uniqMerge(unique_users) FROM chuaikan.active_users_5min
-- WHERE window_start >= toStartOfDay(now());

-- Posts per minute (rolling window)
CREATE MATERIALIZED VIEW chuaikan.posts_per_minute
ENGINE = SummingMergeTree()
ORDER BY (minute)
POPULATE AS
SELECT
    toStartOfMinute(timestamp) as minute,
    countIf(action = 'post_created') as post_count,
    countIf(action = 'like') as like_count,
    countIf(action = 'comment') as comment_count
FROM chuaikan.events
GROUP BY minute;

-- SOS alerts by province (live)
CREATE MATERIALIZED VIEW chuaikan.sos_by_province_live
ENGINE = SummingMergeTree()
ORDER BY (province, alert_type, severity)
POPULATE AS
SELECT
    province,
    alert_type,
    severity,
    count() as alert_count,
    max(timestamp) as last_alert
FROM chuaikan.sos_events
WHERE status = 'active'
  AND timestamp >= now() - INTERVAL 24 HOUR
GROUP BY province, alert_type, severity;

-- Hourly user activity heatmap
CREATE MATERIALIZED VIEW chuaikan.hourly_activity
ENGINE = SummingMergeTree()
ORDER BY (date, hour)
POPULATE AS
SELECT
    toDate(timestamp) as date,
    toHour(timestamp) as hour,
    count() as event_count,
    uniq(user_id) as unique_users
FROM chuaikan.events
GROUP BY date, hour;
```

### Step 1035: Kafka → ClickHouse Pipeline

```sql
-- Kafka Table Engine (อ่านจาก Kafka โดยตรง)
CREATE TABLE chuaikan.kafka_events
(
    event_id   String,
    user_id    String,
    action     String,
    target_id  String,
    metadata   String,
    timestamp  String
)
ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'localhost:9092',
    kafka_topic_list = 'user-events',
    kafka_group_name = 'clickhouse-consumer',
    kafka_format = 'JSONEachRow',
    kafka_max_block_size = 65536;

-- Materialized view ที่เขียนจาก Kafka → events table
CREATE MATERIALIZED VIEW chuaikan.kafka_events_mv TO chuaikan.events AS
SELECT
    event_id,
    user_id,
    action,
    target_id,
    'post' AS target_type,
    metadata,
    JSONExtractString(metadata, 'province') as province,
    parseDateTimeBestEffort(timestamp) as timestamp
FROM chuaikan.kafka_events;

-- SOS Kafka pipeline
CREATE TABLE chuaikan.kafka_sos
(
    event_id    String,
    post_id     String,
    user_id     String,
    alert_type  String,
    severity    String,
    location    String,
    description String,
    timestamp   String
)
ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'localhost:9092',
    kafka_topic_list = 'sos-alerts',
    kafka_group_name = 'clickhouse-sos-consumer',
    kafka_format = 'JSONEachRow';

CREATE MATERIALIZED VIEW chuaikan.kafka_sos_mv TO chuaikan.sos_events AS
SELECT
    event_id,
    post_id,
    user_id,
    alert_type,
    severity,
    JSONExtractString(location, 'province') as province,
    toFloat64OrZero(JSONExtractString(location, 'lat')) as lat,
    toFloat64OrZero(JSONExtractString(location, 'lng')) as lng,
    description,
    'active' as status,
    parseDateTimeBestEffort(timestamp) as timestamp
FROM chuaikan.kafka_sos;
```

### Step 1036: ClickHouse Queries สำหรับ Grafana

```sql
-- Panel 1: Live DAU (ทุก 30 วินาที)
SELECT
    uniq(user_id) as dau
FROM chuaikan.events
WHERE date = today()
  AND timestamp >= toStartOfDay(now());

-- Panel 2: Posts per second (time series)
SELECT
    toStartOfMinute(timestamp) as time,
    countIf(action = 'post_created') / 60 as posts_per_second,
    countIf(action = 'like') / 60 as likes_per_second
FROM chuaikan.events
WHERE timestamp >= now() - INTERVAL 1 HOUR
GROUP BY time
ORDER BY time;

-- Panel 3: Active SOS by province (map)
SELECT
    province,
    count() as active_sos,
    countIf(severity = 'critical') as critical_count,
    max(timestamp) as last_alert
FROM chuaikan.sos_events
WHERE status = 'active'
  AND timestamp >= now() - INTERVAL 24 HOUR
GROUP BY province
ORDER BY active_sos DESC;

-- Panel 4: Error rate by service (ต้องมี error logs table)
SELECT
    service_name,
    countIf(level = 'error') as errors,
    count() as total,
    round(countIf(level = 'error') / count() * 100, 2) as error_rate_pct
FROM chuaikan.logs
WHERE timestamp >= now() - INTERVAL 5 MINUTE
GROUP BY service_name;

-- Panel 5: Queue depth (BullMQ jobs)
-- ดึงจาก Redis ผ่าน Node.js exporter
SELECT
    queue_name,
    waiting_jobs,
    active_jobs,
    failed_jobs
FROM chuaikan.queue_metrics
WHERE timestamp >= now() - INTERVAL 1 MINUTE
ORDER BY timestamp DESC
LIMIT 5;
```

### Step 1037: Anomaly Detection

```typescript
// src/lib/analytics/anomaly-detector.ts
import ClickHouse from '@clickhouse/client';
import { redis } from '../redis/redis-client';

const clickhouse = createClient({
  host: process.env.CLICKHOUSE_HOST || 'http://localhost:8123',
  database: 'chuaikan',
});

function createClient(config: any) {
  return new (require('@clickhouse/client').createClient)(config);
}

export async function checkDAUAnomaly(): Promise<void> {
  // เปรียบเทียบ DAU ปัจจุบันกับ 30 นาทีที่แล้ว
  const result = await clickhouse.query({
    query: `
      SELECT
        uniqIf(user_id, timestamp >= now() - INTERVAL 5 MINUTE) as dau_now,
        uniqIf(user_id, timestamp >= now() - INTERVAL 35 MINUTE
                      AND timestamp < now() - INTERVAL 30 MINUTE) as dau_30min_ago
      FROM chuaikan.events
      WHERE date = today()
    `,
    format: 'JSONEachRow',
  });

  const rows = await result.json() as any[];
  if (rows.length === 0) return;

  const { dau_now, dau_30min_ago } = rows[0];

  if (dau_30min_ago > 0) {
    const dropPercent = ((dau_30min_ago - dau_now) / dau_30min_ago) * 100;

    if (dropPercent > 20) {
      console.error(`[Anomaly] DAU dropped ${dropPercent.toFixed(1)}% in 5 minutes!`);
      await sendAlertToSlack({
        severity: 'critical',
        message: `DAU dropped ${dropPercent.toFixed(1)}% in 5 minutes (${dau_30min_ago} → ${dau_now})`,
        service: 'chuaikan-analytics',
      });
    }
  }
}

async function sendAlertToSlack(alert: any) {
  const webhookUrl = process.env.SLACK_WEBHOOK_URL;
  if (!webhookUrl) return;

  await fetch(webhookUrl, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: `🚨 *${alert.severity.toUpperCase()}*: ${alert.message}`,
      attachments: [{ color: 'danger', text: `Service: ${alert.service}` }],
    }),
  });
}

// Run ทุก 5 นาที
setInterval(checkDAUAnomaly, 5 * 60 * 1000);
```

### Step 1038: SOS Command Center

```typescript
// src/app/api/admin/sos-command/route.ts
import { NextResponse } from 'next/server';
import { clickhouseClient } from '@/lib/analytics/clickhouse-client';

export async function GET() {
  // Active SOS by severity and location
  const [activeSOS, heatmapData, responseTeams] = await Promise.all([
    clickhouseClient.query({
      query: `
        SELECT
            post_id, user_id, alert_type, severity,
            province, lat, lng, description,
            timestamp,
            dateDiff('minute', timestamp, now()) as minutes_ago
        FROM chuaikan.sos_events
        WHERE status = 'active'
          AND timestamp >= now() - INTERVAL 48 HOUR
        ORDER BY severity DESC, timestamp DESC
        LIMIT 100
      `,
      format: 'JSONEachRow',
    }).then(r => r.json()),

    clickhouseClient.query({
      query: `
        SELECT province, count() as count,
               countIf(severity='critical') as critical,
               countIf(severity='high') as high
        FROM chuaikan.sos_events
        WHERE status = 'active'
          AND timestamp >= now() - INTERVAL 24 HOUR
        GROUP BY province
        ORDER BY count DESC
      `,
      format: 'JSONEachRow',
    }).then(r => r.json()),

    // Response teams จาก PostgreSQL
    fetch('/api/admin/response-teams').then(r => r.json()),
  ]);

  return NextResponse.json({
    activeSOS,
    heatmapData,
    responseTeams,
    generatedAt: new Date().toISOString(),
  });
}
```

### Step 1039: Grafana Alerting

```yaml
# grafana/alerting/line-notify.yml
apiVersion: 1
contactPoints:
  - orgId: 1
    name: Line Notify
    receivers:
      - uid: line-notify-uid
        type: webhook
        settings:
          url: https://notify-api.line.me/api/notify
          httpMethod: POST
          headers:
            Authorization: "Bearer ${LINE_NOTIFY_TOKEN}"
          message: "🚨 chuaikan.com Alert: {{ .Message }}"

  - orgId: 1
    name: Discord
    receivers:
      - uid: discord-uid
        type: discord
        settings:
          url: "${DISCORD_WEBHOOK_URL}"
          message: "🚨 **chuaikan.com Alert**\n{{ .Message }}"
```

```yaml
# grafana/alerting/rules.yml
apiVersion: 1
groups:
  - orgId: 1
    name: chuaikan-critical
    folder: chuaikan
    interval: 1m
    rules:
      - uid: dau-drop-alert
        title: DAU Drop >20%
        condition: C
        data:
          - refId: A
            queryType: ""
            model:
              rawSql: |
                SELECT
                  uniqIf(user_id, timestamp >= now() - INTERVAL 5 MINUTE) as dau_now,
                  uniqIf(user_id, timestamp BETWEEN now() - INTERVAL 35 MINUTE AND now() - INTERVAL 30 MINUTE) as dau_prev
                FROM chuaikan.events WHERE date = today()
        noDataState: OK
        execErrState: Alerting
        annotations:
          summary: "DAU dropped significantly"
          description: "Current DAU is {{ $values.A.dau_now }}, was {{ $values.A.dau_prev }}"
        labels:
          severity: critical
          team: engineering
```

---

## 🔧 Configuration Files

### ClickHouse config สำหรับ Performance

```xml
<!-- /etc/clickhouse-server/config.d/custom.xml -->
<clickhouse>
  <max_memory_usage>8000000000</max_memory_usage>
  <max_concurrent_queries>100</max_concurrent_queries>
  
  <!-- Kafka integration -->
  <kafka>
    <auto_offset_reset>latest</auto_offset_reset>
    <max_poll_interval_ms>300000</max_poll_interval_ms>
  </kafka>
</clickhouse>
```

---

## 🧪 Testing

### Test ClickHouse Query Speed

```bash
# ทดสอบ query speed บน 1 ล้าน rows
clickhouse-client --query "
  INSERT INTO chuaikan.events
  SELECT
    generateUUIDv4() as event_id,
    concat('user-', toString(rand() % 100000)) as user_id,
    ['view','like','comment','share'][rand() % 4 + 1] as action,
    generateUUIDv4() as target_id,
    'post' as target_type,
    '{}' as metadata,
    ['Bangkok','Chiang Mai','Ayutthaya'][rand() % 3 + 1] as province,
    now() - toIntervalSecond(rand() % 86400) as timestamp
  FROM numbers(1000000)
"

# Query ความเร็ว
time clickhouse-client --query "SELECT count(), uniq(user_id) FROM chuaikan.events WHERE date = today()"
# คาดว่า: < 100ms
```

---

## ❌ Common Errors & Solutions

### Error 1: Kafka Consumer Lag ใน ClickHouse

**ตรวจสอบ:**
```sql
SELECT * FROM system.kafka_consumers;
```

**แก้ไข:**
```sql
-- Detach และ re-attach table
DETACH TABLE chuaikan.kafka_events;
ATTACH TABLE chuaikan.kafka_events;
```

### Error 2: ClickHouse Memory เกิน

**แก้ไข:**
```sql
-- ลด max memory per query
SET max_memory_usage = 4000000000;

-- หรือใช้ LIMIT ใน query
SELECT * FROM chuaikan.events LIMIT 100000;
```

---

## ✅ Checklist

- [ ] **Step 1031:** ClickHouse ติดตั้งสำเร็จ, รันที่ port 8123
- [ ] **Step 1032:** Grafana รันที่ port 3001, ClickHouse datasource เชื่อมต่อได้
- [ ] **Step 1033:** ClickHouse schema สร้างครบ (events, sos_events, push_analytics)
- [ ] **Step 1034:** Materialized views ทำงาน (active_users_5min, posts_per_minute)
- [ ] **Step 1035:** Kafka → ClickHouse pipeline streaming events ได้
- [ ] **Step 1036:** Grafana panels แสดงข้อมูล real-time
- [ ] **Step 1037:** Anomaly detection แจ้งเตือนเมื่อ DAU drop >20%
- [ ] **Step 1038:** SOS Command Center แสดง active alerts
- [ ] **Step 1039:** Grafana alerting ส่ง Line Notify และ Discord
- [ ] **Step 1040:** Dashboard refresh ทุก 30 วินาที performance ดี

---

## 🔗 References

- [ClickHouse Documentation](https://clickhouse.com/docs/)
- [ClickHouse Kafka Engine](https://clickhouse.com/docs/en/engines/table-engines/integrations/kafka)
- [Grafana ClickHouse Plugin](https://grafana.com/grafana/plugins/grafana-clickhouse-datasource/)
- [Line Notify API](https://notify-bot.line.me/doc/en/)

---
*Part 104 | Road to 1,000,000 Users/Day | chuaikan.com*
