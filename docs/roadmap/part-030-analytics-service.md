# Part 030: Analytics Service
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 291-300
> **เวลาโดยประมาณ:** 7 ชั่วโมง
> **Prerequisites:** Part 026-029 (Feed/SOS/Location/Search), Part 025 (Redis Cache)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ติดตั้ง ClickHouse เป็น OLAP Database บน Ubuntu 24.04
- ออกแบบ Event Tracking Schema (user_events table)
- สร้าง Frontend Analytics SDK (pageview, click, scroll depth)
- Server-side Events: API calls, feature usage
- ClickHouse Materialized Views สำหรับ Pre-aggregation
- Real-time Dashboard Queries: DAU, MAU, Retention
- Funnel Analysis: Registration → Profile → First Post → 30d Retention
- A/B Test Analysis
- Kafka Integration สำหรับ High-volume Event Ingestion
- Grafana Connection to ClickHouse
- PDPA Compliance: Data Retention Policy

---

## 📖 ทฤษฎีและแนวคิด

### Analytics Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                   ANALYTICS ARCHITECTURE                         │
│                                                                  │
│  Data Sources                                                    │
│  ────────────                                                    │
│  Frontend SDK ──┐                                               │
│  Mobile SDK  ───┤                                               │
│  Server APIs ───┼──► Kafka Topic: chuaikan.events               │
│  SOS Events  ───┤         │                                     │
│  Search Logs ───┘         │                                     │
│                           ▼                                     │
│                    ┌──────────────┐                             │
│                    │  ClickHouse  │◄── Materialized Views       │
│                    │  OLAP DB     │    (pre-aggregation)        │
│                    └──────┬───────┘                             │
│                           │                                     │
│              ┌────────────┼────────────┐                        │
│              ▼            ▼            ▼                        │
│           Grafana    Analytics    Real-time                     │
│           Dashboard   API        Dashboard                      │
│                                                                  │
│  Data Flow:                                                      │
│  Event → Kafka → ClickHouse Consumer → Materialized Views       │
│  Latency: < 30 seconds end-to-end                               │
└──────────────────────────────────────────────────────────────────┘
```

### ClickHouse vs PostgreSQL สำหรับ Analytics

```
Feature           ClickHouse          PostgreSQL
──────────────── ─────────────────── ──────────────────
Query Type        OLAP (analytics)    OLTP (transactions)
Compression       10:1 ratio          2:1 ratio
Query Speed       100x faster         Baseline
Insert Speed      1M rows/sec         50K rows/sec
Aggregation       Columnar (fast)     Row-based (slow)
JOINs             Avoid (denorm)      Good
Data Model        Columnar            Row-based
Use Case          Analytics, reports  Application DB

→ chuaikan.com ใช้ทั้งสอง:
  PostgreSQL: Application data
  ClickHouse: Analytics, reports, dashboards
```

### Event Taxonomy

```
Event Types สำหรับ chuaikan.com:

PAGE EVENTS
├── page_view          - เข้าชมหน้า
├── page_exit          - ออกจากหน้า
└── scroll_depth       - เลื่อนลงมาถึงกี่เปอร์เซ็นต์

USER EVENTS
├── user_signup        - สมัครสมาชิก
├── user_login         - เข้าสู่ระบบ
├── user_logout        - ออกจากระบบ
├── profile_complete   - ทำ profile ครบ
└── user_follow        - ติดตาม

POST EVENTS
├── post_create        - สร้างโพสต์
├── post_view          - ดูโพสต์
├── post_like          - กดถูกใจ
├── post_comment       - แสดงความคิดเห็น
├── post_share         - แชร์
└── post_save          - บันทึก

SOS EVENTS
├── sos_report         - รายงาน SOS
├── sos_view           - ดู SOS alert
├── sos_confirm_safe   - ยืนยันปลอดภัย
└── sos_verify         - ตรวจสอบโดย moderator

SEARCH EVENTS
├── search_query       - ค้นหา
├── search_click       - คลิก search result
└── search_zero_result - ค้นหาไม่เจอ
```

---

## ⚙️ Environment Setup

### Step 291: Install ClickHouse

```bash
# ติดตั้ง ClickHouse บน Ubuntu 24.04
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg

# เพิ่ม ClickHouse repository
curl -fsSL 'https://packages.clickhouse.com/rpm/lts/repodata/repomd.xml.key' | \
  sudo gpg --dearmor -o /usr/share/keyrings/clickhouse-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/clickhouse-keyring.gpg] \
  https://packages.clickhouse.com/deb stable main" | \
  sudo tee /etc/apt/sources.list.d/clickhouse.list

sudo apt-get update

# ติดตั้ง ClickHouse Server และ Client
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y \
  clickhouse-server \
  clickhouse-client

# Configure
sudo nano /etc/clickhouse-server/config.xml
# ตรวจสอบ: <listen_host>127.0.0.1</listen_host>

# Start service
sudo systemctl enable clickhouse-server
sudo systemctl start clickhouse-server
sudo systemctl status clickhouse-server

# ทดสอบ
clickhouse-client --query "SELECT version()"
# → 24.x.x.x

# สร้าง database
clickhouse-client --query "CREATE DATABASE IF NOT EXISTS chuaikan_analytics"
```

### Step 292: Install Kafka

```bash
# ติดตั้ง Kafka สำหรับ high-volume event ingestion
# ใช้ Kafka ใน Docker (สำหรับ development)
sudo apt-get install -y docker.io docker-compose

mkdir -p /home/user/chuaikan/docker/kafka
cat > /home/user/chuaikan/docker/kafka/docker-compose.yml << 'EOF'
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    restart: unless-stopped

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'true'
      KAFKA_NUM_PARTITIONS: 8
    restart: unless-stopped
EOF

cd /home/user/chuaikan/docker/kafka
sudo docker-compose up -d

# ตรวจสอบ
sudo docker-compose ps
# → kafka running, zookeeper running

# สร้าง topic
sudo docker exec -it kafka-kafka-1 kafka-topics \
  --create --topic chuaikan.events \
  --bootstrap-server localhost:9092 \
  --partitions 8 \
  --replication-factor 1

sudo docker exec -it kafka-kafka-1 kafka-topics \
  --list --bootstrap-server localhost:9092
```

### Step 293: Node.js Dependencies

```bash
cd /home/user/chuaikan/services/analytics-service

npm install \
  @clickhouse/client@0.3.0 \
  kafkajs@2.2.4 \
  express@4.18.2 \
  ioredis@5.3.2 \
  zod@3.22.4 \
  winston@3.11.0 \
  uuid@9.0.0

npm install -D \
  typescript@5.3.2 \
  @types/node@20.10.0 \
  @types/express@4.17.21 \
  ts-node@10.9.2 \
  nodemon@3.0.2
```

---

## 🛠️ Step-by-Step Implementation

### Step 294: ClickHouse Schema

```sql
-- /home/user/chuaikan/clickhouse/schema.sql

-- หลัก: user_events table
CREATE TABLE IF NOT EXISTS chuaikan_analytics.user_events
(
    -- Partition key
    event_date      Date DEFAULT toDate(timestamp),
    timestamp       DateTime64(3) DEFAULT now64(),

    -- Event identity
    event_id        UUID DEFAULT generateUUIDv4(),
    event_type      LowCardinality(String),
    event_category  LowCardinality(String),  -- page, user, post, sos, search

    -- User info
    user_id         Nullable(UUID),
    session_id      String,
    anonymous_id    String,

    -- Content
    object_type     LowCardinality(String),  -- post, user, sos, hashtag
    object_id       Nullable(String),

    -- Properties (flexible JSON)
    properties      String DEFAULT '{}',     -- JSON blob

    -- Context
    platform        LowCardinality(String),  -- web, ios, android
    app_version     LowCardinality(String),
    os              LowCardinality(String),
    browser         LowCardinality(String),
    device_type     LowCardinality(String),  -- desktop, mobile, tablet

    -- Location
    country         LowCardinality(String) DEFAULT 'TH',
    province_code   LowCardinality(String),
    lat             Nullable(Float32),
    lng             Nullable(Float32),

    -- UTM
    utm_source      LowCardinality(String),
    utm_medium      LowCardinality(String),
    utm_campaign    LowCardinality(String),

    -- AB Test
    ab_variant      LowCardinality(String),

    -- Performance
    page_load_ms    Nullable(UInt32),
    api_response_ms Nullable(UInt32)
)
ENGINE = MergeTree
PARTITION BY (event_date)
ORDER BY (event_type, user_id, timestamp)
SETTINGS index_granularity = 8192;

-- TTL: ลบข้อมูลเก่ากว่า 2 ปี (PDPA compliance)
ALTER TABLE chuaikan_analytics.user_events
  MODIFY TTL event_date + INTERVAL 2 YEAR;


-- ─── Materialized Views ────────────────────────────────────────

-- Daily Active Users
CREATE MATERIALIZED VIEW IF NOT EXISTS chuaikan_analytics.mv_daily_active_users
ENGINE = AggregatingMergeTree
PARTITION BY event_date
ORDER BY (event_date, province_code)
AS
SELECT
    event_date,
    province_code,
    uniqState(user_id)     AS dau_state,
    uniqState(session_id)  AS sessions_state,
    countState()           AS events_state
FROM chuaikan_analytics.user_events
WHERE user_id IS NOT NULL
GROUP BY event_date, province_code;

-- Hourly event counts (สำหรับ real-time monitoring)
CREATE MATERIALIZED VIEW IF NOT EXISTS chuaikan_analytics.mv_hourly_events
ENGINE = SummingMergeTree
PARTITION BY toDate(hour)
ORDER BY (hour, event_type, platform)
AS
SELECT
    toStartOfHour(timestamp) AS hour,
    event_type,
    platform,
    count()                  AS event_count
FROM chuaikan_analytics.user_events
GROUP BY hour, event_type, platform;

-- Post engagement summary
CREATE MATERIALIZED VIEW IF NOT EXISTS chuaikan_analytics.mv_post_engagement
ENGINE = AggregatingMergeTree
PARTITION BY event_date
ORDER BY (event_date, object_id)
AS
SELECT
    event_date,
    object_id AS post_id,
    countStateIf(event_type = 'post_view')    AS views_state,
    countStateIf(event_type = 'post_like')    AS likes_state,
    countStateIf(event_type = 'post_comment') AS comments_state,
    countStateIf(event_type = 'post_share')   AS shares_state,
    uniqState(user_id)                        AS unique_users_state
FROM chuaikan_analytics.user_events
WHERE object_type = 'post'
GROUP BY event_date, object_id;

-- Funnel steps
CREATE MATERIALIZED VIEW IF NOT EXISTS chuaikan_analytics.mv_funnel_steps
ENGINE = AggregatingMergeTree
PARTITION BY event_date
ORDER BY (event_date, user_id)
AS
SELECT
    event_date,
    user_id,
    groupArrayState(event_type) AS steps_state,
    minState(timestamp)         AS first_event_state,
    maxState(timestamp)         AS last_event_state
FROM chuaikan_analytics.user_events
WHERE event_type IN (
    'user_signup', 'profile_complete',
    'post_create', 'user_login'
)
  AND user_id IS NOT NULL
GROUP BY event_date, user_id;
```

```bash
# รัน schema
clickhouse-client --database=chuaikan_analytics \
  --queries-file /home/user/chuaikan/clickhouse/schema.sql

# ตรวจสอบ
clickhouse-client --query "SHOW TABLES FROM chuaikan_analytics"
```

### Step 295: Analytics SDK (Frontend)

```typescript
// packages/analytics-sdk/src/index.ts
// SDK สำหรับ Frontend (Next.js)

interface EventProperties {
  [key: string]: string | number | boolean | null | undefined;
}

interface TrackerConfig {
  endpoint: string;
  appVersion: string;
  userId?: string;
  batchSize?: number;
  flushInterval?: number;  // ms
}

class ChuaikanAnalytics {
  private config: Required<TrackerConfig>;
  private queue: any[] = [];
  private sessionId: string;
  private anonymousId: string;
  private flushTimer?: NodeJS.Timeout;
  private pageEnterTime?: number;

  constructor(config: TrackerConfig) {
    this.config = {
      batchSize: 20,
      flushInterval: 5000,
      ...config,
    };

    this.sessionId = this.getOrCreateSessionId();
    this.anonymousId = this.getOrCreateAnonymousId();
    this.startAutoFlush();
  }

  // Track page view
  pageView(path: string, properties: EventProperties = {}): void {
    this.pageEnterTime = Date.now();

    this.track('page_view', 'page', {
      path,
      title: document.title,
      referrer: document.referrer,
      ...properties,
    });

    // ติดตาม scroll depth
    this.setupScrollTracking(path);
  }

  // Track page exit (time on page)
  pageExit(path: string): void {
    const timeOnPage = this.pageEnterTime
      ? Date.now() - this.pageEnterTime
      : 0;

    this.track('page_exit', 'page', {
      path,
      time_on_page_ms: timeOnPage,
    });
  }

  // Track click
  click(element: string, properties: EventProperties = {}): void {
    this.track('click', 'page', {
      element,
      ...properties,
    });
  }

  // Track custom event
  track(
    eventType: string,
    category: string,
    properties: EventProperties = {}
  ): void {
    const event = {
      event_type: eventType,
      event_category: category,
      user_id: this.config.userId,
      session_id: this.sessionId,
      anonymous_id: this.anonymousId,
      properties: JSON.stringify(properties),
      platform: 'web',
      app_version: this.config.appVersion,
      timestamp: new Date().toISOString(),
      browser: this.getBrowser(),
      device_type: this.getDeviceType(),
      url: window.location.href,
      utm_source: this.getUTM('utm_source'),
      utm_medium: this.getUTM('utm_medium'),
      utm_campaign: this.getUTM('utm_campaign'),
    };

    this.queue.push(event);

    if (this.queue.length >= this.config.batchSize) {
      this.flush();
    }
  }

  // Scroll depth tracking
  private setupScrollTracking(path: string): void {
    const depths = [25, 50, 75, 100];
    const tracked = new Set<number>();

    const handleScroll = () => {
      const scrolled =
        (window.scrollY + window.innerHeight) / document.body.scrollHeight * 100;

      for (const depth of depths) {
        if (scrolled >= depth && !tracked.has(depth)) {
          tracked.add(depth);
          this.track('scroll_depth', 'page', { path, depth });
        }
      }
    };

    window.addEventListener('scroll', handleScroll, { passive: true });

    // Cleanup
    window.addEventListener('beforeunload', () => {
      window.removeEventListener('scroll', handleScroll);
    }, { once: true });
  }

  // Flush events to server
  async flush(): Promise<void> {
    if (this.queue.length === 0) return;

    const events = [...this.queue];
    this.queue = [];

    try {
      // ใช้ sendBeacon สำหรับ reliability (รองรับ page unload)
      if (navigator.sendBeacon) {
        navigator.sendBeacon(
          `${this.config.endpoint}/batch`,
          JSON.stringify({ events })
        );
      } else {
        await fetch(`${this.config.endpoint}/batch`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ events }),
          keepalive: true,
        });
      }
    } catch {
      // Restore events on failure
      this.queue = [...events, ...this.queue];
    }
  }

  // Auto flush
  private startAutoFlush(): void {
    this.flushTimer = setInterval(
      () => this.flush(),
      this.config.flushInterval
    );

    // Flush on page unload
    window.addEventListener('beforeunload', () => this.flush());
    window.addEventListener('visibilitychange', () => {
      if (document.hidden) this.flush();
    });
  }

  private getOrCreateSessionId(): string {
    try {
      let id = sessionStorage.getItem('chuaikan_session');
      if (!id) {
        id = crypto.randomUUID();
        sessionStorage.setItem('chuaikan_session', id);
      }
      return id;
    } catch {
      return crypto.randomUUID();
    }
  }

  private getOrCreateAnonymousId(): string {
    try {
      let id = localStorage.getItem('chuaikan_anon_id');
      if (!id) {
        id = crypto.randomUUID();
        localStorage.setItem('chuaikan_anon_id', id);
      }
      return id;
    } catch {
      return crypto.randomUUID();
    }
  }

  private getBrowser(): string {
    const ua = navigator.userAgent;
    if (ua.includes('Chrome')) return 'chrome';
    if (ua.includes('Firefox')) return 'firefox';
    if (ua.includes('Safari')) return 'safari';
    return 'other';
  }

  private getDeviceType(): string {
    if (/mobile/i.test(navigator.userAgent)) return 'mobile';
    if (/tablet/i.test(navigator.userAgent)) return 'tablet';
    return 'desktop';
  }

  private getUTM(param: string): string {
    return new URLSearchParams(window.location.search).get(param) ?? '';
  }
}

// Next.js integration
export function useAnalytics() {
  // ใน Next.js App Router:
  // const analytics = new ChuaikanAnalytics({...})
  // เรียกใน layout.tsx

  return {
    track: (event: string, props?: EventProperties) =>
      window.__analytics?.track(event, 'custom', props ?? {}),
    pageView: (path: string) =>
      window.__analytics?.pageView(path),
  };
}

export default ChuaikanAnalytics;
```

### Step 296: Kafka Producer & Consumer

```typescript
// src/services/kafka-producer.ts

import { Kafka, Producer, CompressionTypes } from 'kafkajs';

const kafka = new Kafka({
  clientId: 'analytics-service',
  brokers: (process.env.KAFKA_BROKERS ?? 'localhost:9092').split(','),
});

export class EventProducer {
  private producer: Producer;
  private isConnected = false;
  private buffer: any[] = [];
  private readonly BUFFER_MAX = 1000;
  private readonly FLUSH_INTERVAL = 1000; // 1 second

  constructor() {
    this.producer = kafka.producer({
      maxInFlightRequests: 5,
      idempotent: true,
      compression: CompressionTypes.GZIP,
    });

    this.startFlushTimer();
  }

  async connect(): Promise<void> {
    await this.producer.connect();
    this.isConnected = true;
    console.log('Kafka producer connected');
  }

  async sendEvent(event: any): Promise<void> {
    this.buffer.push(event);

    if (this.buffer.length >= this.BUFFER_MAX) {
      await this.flush();
    }
  }

  async flush(): Promise<void> {
    if (this.buffer.length === 0 || !this.isConnected) return;

    const events = [...this.buffer];
    this.buffer = [];

    try {
      await this.producer.send({
        topic: 'chuaikan.events',
        messages: events.map(event => ({
          key: event.user_id ?? event.anonymous_id,
          value: JSON.stringify(event),
          timestamp: String(Date.now()),
        })),
        acks: 1, // Leader ack (performance over durability)
      });
    } catch (error) {
      // Restore buffer
      this.buffer = [...events, ...this.buffer];
      throw error;
    }
  }

  private startFlushTimer(): void {
    setInterval(() => this.flush(), this.FLUSH_INTERVAL);
  }

  async disconnect(): Promise<void> {
    await this.flush(); // flush before disconnect
    await this.producer.disconnect();
  }
}

// ─── Kafka Consumer (ClickHouse Ingestor) ────────────────────────
// src/services/kafka-consumer.ts

import { Kafka, Consumer, EachBatchPayload } from 'kafkajs';
import { createClient } from '@clickhouse/client';

const clickhouse = createClient({
  host: process.env.CLICKHOUSE_URL ?? 'http://localhost:8123',
  username: process.env.CLICKHOUSE_USER ?? 'default',
  password: process.env.CLICKHOUSE_PASSWORD ?? '',
  database: 'chuaikan_analytics',
});

export class ClickHouseIngestor {
  private consumer: Consumer;
  private batchBuffer: any[] = [];
  private readonly BATCH_SIZE = 5000;

  constructor() {
    const kafka = new Kafka({
      clientId: 'clickhouse-ingestor',
      brokers: (process.env.KAFKA_BROKERS ?? 'localhost:9092').split(','),
    });

    this.consumer = kafka.consumer({
      groupId: 'clickhouse-analytics',
      maxBytesPerPartition: 10 * 1024 * 1024, // 10MB
    });
  }

  async start(): Promise<void> {
    await this.consumer.connect();
    await this.consumer.subscribe({
      topic: 'chuaikan.events',
      fromBeginning: false,
    });

    await this.consumer.run({
      eachBatch: async (payload: EachBatchPayload) => {
        await this.processBatch(payload);
      },
    });

    console.log('ClickHouse ingestor started');
  }

  private async processBatch(
    { batch, resolveOffset, heartbeat }: EachBatchPayload
  ): Promise<void> {
    const events: any[] = [];

    for (const message of batch.messages) {
      if (!message.value) continue;

      try {
        const event = JSON.parse(message.value.toString());
        events.push(this.transformEvent(event));
      } catch {
        // Skip malformed events
      }
    }

    if (events.length > 0) {
      await this.insertToClickHouse(events);
    }

    // Commit offset
    resolveOffset(batch.messages[batch.messages.length - 1].offset);
    await heartbeat();
  }

  private transformEvent(event: any): any {
    return {
      event_date: new Date(event.timestamp).toISOString().split('T')[0],
      timestamp: event.timestamp,
      event_type: event.event_type ?? 'unknown',
      event_category: event.event_category ?? 'unknown',
      user_id: event.user_id ?? null,
      session_id: event.session_id ?? '',
      anonymous_id: event.anonymous_id ?? '',
      object_type: event.object_type ?? '',
      object_id: event.object_id ?? null,
      properties: typeof event.properties === 'object'
        ? JSON.stringify(event.properties)
        : event.properties ?? '{}',
      platform: event.platform ?? 'unknown',
      app_version: event.app_version ?? '',
      province_code: event.province_code ?? '',
      lat: event.lat ?? null,
      lng: event.lng ?? null,
      utm_source: event.utm_source ?? '',
      utm_medium: event.utm_medium ?? '',
      utm_campaign: event.utm_campaign ?? '',
      ab_variant: event.ab_variant ?? '',
    };
  }

  private async insertToClickHouse(events: any[]): Promise<void> {
    await clickhouse.insert({
      table: 'user_events',
      values: events,
      format: 'JSONEachRow',
    });

    console.log(`Inserted ${events.length} events to ClickHouse`);
  }
}
```

### Step 297: Analytics Query Service

```typescript
// src/services/analytics-query.service.ts

import { createClient, ClickHouseClient } from '@clickhouse/client';

export interface DAUResult {
  date: string;
  dau: number;
  sessions: number;
  events: number;
}

export interface RetentionResult {
  cohort_date: string;
  day_1: number;
  day_7: number;
  day_30: number;
  cohort_size: number;
}

export interface FunnelResult {
  step: string;
  users: number;
  conversion_rate: number;
}

export class AnalyticsQueryService {
  private client: ClickHouseClient;

  constructor() {
    this.client = createClient({
      host: process.env.CLICKHOUSE_URL ?? 'http://localhost:8123',
      username: process.env.CLICKHOUSE_USER ?? 'default',
      password: process.env.CLICKHOUSE_PASSWORD ?? '',
      database: 'chuaikan_analytics',
    });
  }

  // DAU (Daily Active Users) - 30 วันล่าสุด
  async getDAU(days: number = 30): Promise<DAUResult[]> {
    const result = await this.client.query({
      query: `
        SELECT
          event_date,
          uniqMerge(dau_state)      AS dau,
          uniqMerge(sessions_state) AS sessions,
          countMerge(events_state)  AS events
        FROM mv_daily_active_users
        WHERE event_date >= today() - ${days}
        GROUP BY event_date
        ORDER BY event_date ASC
      `,
      format: 'JSONEachRow',
    });

    const data = await result.json();
    return (data as any[]).map(r => ({
      date: r.event_date,
      dau: parseInt(r.dau),
      sessions: parseInt(r.sessions),
      events: parseInt(r.events),
    }));
  }

  // MAU (Monthly Active Users)
  async getMAU(months: number = 6): Promise<any[]> {
    const result = await this.client.query({
      query: `
        SELECT
          toStartOfMonth(event_date) AS month,
          uniq(user_id)             AS mau,
          countDistinct(session_id) AS total_sessions
        FROM user_events
        WHERE event_date >= toDate(now() - INTERVAL ${months} MONTH)
          AND user_id IS NOT NULL
        GROUP BY month
        ORDER BY month ASC
      `,
      format: 'JSONEachRow',
    });

    return result.json() as any;
  }

  // User Retention (Cohort Analysis)
  async getRetention(): Promise<RetentionResult[]> {
    const result = await this.client.query({
      query: `
        WITH cohorts AS (
          SELECT
            user_id,
            min(event_date)       AS cohort_date,
            min(timestamp)        AS first_seen
          FROM user_events
          WHERE event_type = 'user_signup'
            AND event_date >= today() - 60
            AND user_id IS NOT NULL
          GROUP BY user_id
        ),
        activity AS (
          SELECT
            user_id,
            event_date
          FROM user_events
          WHERE user_id IS NOT NULL
            AND event_date >= today() - 60
          GROUP BY user_id, event_date
        )
        SELECT
          c.cohort_date,
          count(DISTINCT c.user_id)                   AS cohort_size,
          countDistinctIf(
            a.user_id,
            dateDiff('day', c.cohort_date, a.event_date) BETWEEN 1 AND 1
          )                                           AS day_1,
          countDistinctIf(
            a.user_id,
            dateDiff('day', c.cohort_date, a.event_date) BETWEEN 6 AND 8
          )                                           AS day_7,
          countDistinctIf(
            a.user_id,
            dateDiff('day', c.cohort_date, a.event_date) BETWEEN 28 AND 32
          )                                           AS day_30
        FROM cohorts c
        LEFT JOIN activity a ON a.user_id = c.user_id
        GROUP BY c.cohort_date
        ORDER BY c.cohort_date ASC
      `,
      format: 'JSONEachRow',
    });

    const data = await result.json() as any[];
    return data.map(r => ({
      cohort_date: r.cohort_date,
      cohort_size: parseInt(r.cohort_size),
      day_1: parseInt(r.day_1),
      day_7: parseInt(r.day_7),
      day_30: parseInt(r.day_30),
    }));
  }

  // Funnel Analysis
  async getFunnelAnalysis(
    startDate: string,
    endDate: string
  ): Promise<FunnelResult[]> {
    const result = await this.client.query({
      query: `
        WITH
          signup_users AS (
            SELECT DISTINCT user_id
            FROM user_events
            WHERE event_type = 'user_signup'
              AND event_date BETWEEN '${startDate}' AND '${endDate}'
              AND user_id IS NOT NULL
          ),
          profile_users AS (
            SELECT DISTINCT e.user_id
            FROM user_events e
            INNER JOIN signup_users s ON s.user_id = e.user_id
            WHERE e.event_type = 'profile_complete'
              AND e.event_date BETWEEN '${startDate}' AND '${endDate}'
          ),
          first_post_users AS (
            SELECT DISTINCT e.user_id
            FROM user_events e
            INNER JOIN profile_users p ON p.user_id = e.user_id
            WHERE e.event_type = 'post_create'
              AND e.event_date BETWEEN '${startDate}' AND '${endDate}'
          )
        SELECT
          step,
          users,
          round(users * 100.0 / max(users) OVER (), 2) AS conversion_rate
        FROM (
          SELECT 1 AS ord, 'signup'     AS step, count() AS users FROM signup_users
          UNION ALL
          SELECT 2, 'profile_complete', count() FROM profile_users
          UNION ALL
          SELECT 3, 'first_post',       count() FROM first_post_users
        )
        ORDER BY ord
      `,
      format: 'JSONEachRow',
    });

    return result.json() as any;
  }

  // A/B Test Analysis
  async getABTestResults(testName: string): Promise<any[]> {
    const result = await this.client.query({
      query: `
        SELECT
          ab_variant,
          count(DISTINCT user_id)                         AS users,
          countIf(event_type = 'post_create')             AS posts_created,
          countIf(event_type = 'post_like')               AS likes_given,
          countIf(event_type = 'user_signup')             AS signups,
          round(
            countIf(event_type = 'post_create') * 1.0 /
            count(DISTINCT user_id), 4
          )                                               AS post_creation_rate
        FROM user_events
        WHERE event_date >= today() - 30
          AND ab_variant LIKE '${testName}:%'
          AND user_id IS NOT NULL
        GROUP BY ab_variant
        ORDER BY ab_variant
      `,
      format: 'JSONEachRow',
    });

    return result.json() as any;
  }

  // Real-time: events ใน 5 นาทีล่าสุด
  async getRealtimeEvents(): Promise<any[]> {
    const result = await this.client.query({
      query: `
        SELECT
          event_type,
          count() AS count,
          uniq(user_id) AS unique_users
        FROM user_events
        WHERE timestamp >= now() - INTERVAL 5 MINUTE
        GROUP BY event_type
        ORDER BY count DESC
        LIMIT 20
      `,
      format: 'JSONEachRow',
    });

    return result.json() as any;
  }

  // SOS Analytics
  async getSOSAnalytics(days: number = 7): Promise<any> {
    const result = await this.client.query({
      query: `
        SELECT
          toDate(timestamp) AS date,
          JSONExtractString(properties, 'type') AS sos_type,
          count()                               AS alerts,
          uniq(user_id)                         AS reporters
        FROM user_events
        WHERE event_type = 'sos_report'
          AND event_date >= today() - ${days}
        GROUP BY date, sos_type
        ORDER BY date DESC, alerts DESC
      `,
      format: 'JSONEachRow',
    });

    return result.json();
  }
}
```

### Step 298: Analytics API Routes

```typescript
// src/routes/analytics.routes.ts

import { Router, Request, Response } from 'express';
import { AnalyticsQueryService } from '../services/analytics-query.service';
import { EventProducer } from '../services/kafka-producer';

export function createAnalyticsRouter(
  queryService: AnalyticsQueryService,
  producer: EventProducer
): Router {
  const router = Router();

  // POST /api/v1/analytics/events - Track events from frontend
  router.post('/events', async (req: Request, res: Response) => {
    const { events } = req.body;

    if (!Array.isArray(events) || events.length === 0) {
      return res.status(400).json({ error: 'events array required' });
    }

    // Enrich events with server-side data
    const enriched = events.map((event: any) => ({
      ...event,
      user_id: req.user?.id ?? event.user_id,
      server_timestamp: new Date().toISOString(),
      ip: req.ip,
    }));

    // Send to Kafka
    await Promise.all(enriched.map(e => producer.sendEvent(e)));

    res.json({ ok: true, count: enriched.length });
  });

  // GET /api/v1/analytics/dau (Admin only)
  router.get('/dau', async (req: Request, res: Response) => {
    if (!req.user?.roles.includes('admin')) {
      return res.status(403).json({ error: 'Forbidden' });
    }

    const days = parseInt(req.query.days as string) || 30;
    const data = await queryService.getDAU(days);
    res.json(data);
  });

  // GET /api/v1/analytics/mau
  router.get('/mau', async (req: Request, res: Response) => {
    if (!req.user?.roles.includes('admin')) {
      return res.status(403).json({ error: 'Forbidden' });
    }

    const data = await queryService.getMAU();
    res.json(data);
  });

  // GET /api/v1/analytics/retention
  router.get('/retention', async (req: Request, res: Response) => {
    if (!req.user?.roles.includes('admin')) {
      return res.status(403).json({ error: 'Forbidden' });
    }

    const data = await queryService.getRetention();
    res.json(data);
  });

  // GET /api/v1/analytics/funnel
  router.get('/funnel', async (req: Request, res: Response) => {
    if (!req.user?.roles.includes('admin')) {
      return res.status(403).json({ error: 'Forbidden' });
    }

    const startDate = (req.query.start as string) ||
      new Date(Date.now() - 30 * 86400000).toISOString().split('T')[0];
    const endDate = (req.query.end as string) ||
      new Date().toISOString().split('T')[0];

    const data = await queryService.getFunnelAnalysis(startDate, endDate);
    res.json(data);
  });

  // GET /api/v1/analytics/ab-test/:testName
  router.get('/ab-test/:testName', async (req: Request, res: Response) => {
    if (!req.user?.roles.includes('admin')) {
      return res.status(403).json({ error: 'Forbidden' });
    }

    const data = await queryService.getABTestResults(req.params.testName);
    res.json(data);
  });

  // GET /api/v1/analytics/realtime
  router.get('/realtime', async (req: Request, res: Response) => {
    if (!req.user?.roles.includes('admin')) {
      return res.status(403).json({ error: 'Forbidden' });
    }

    const data = await queryService.getRealtimeEvents();
    res.json({ events: data, timestamp: new Date().toISOString() });
  });

  return router;
}
```

### Step 299: Grafana Setup

```bash
# ติดตั้ง Grafana บน Ubuntu 24.04
sudo apt-get install -y apt-transport-https software-properties-common

wget -q -O - https://packages.grafana.com/gpg.key | \
  sudo gpg --dearmor -o /usr/share/keyrings/grafana.gpg

echo "deb [signed-by=/usr/share/keyrings/grafana.gpg] \
  https://packages.grafana.com/oss/deb stable main" | \
  sudo tee /etc/apt/sources.list.d/grafana.list

sudo apt-get update
sudo apt-get install -y grafana

sudo systemctl enable grafana-server
sudo systemctl start grafana-server

# ติดตั้ง ClickHouse datasource plugin
sudo grafana-cli plugins install grafana-clickhouse-datasource
sudo systemctl restart grafana-server

echo "Grafana running at http://localhost:3000"
echo "Default login: admin / admin"
```

```json
// grafana/datasource.json
// ตั้งค่า ClickHouse datasource ใน Grafana

{
  "name": "ClickHouse Analytics",
  "type": "grafana-clickhouse-datasource",
  "url": "http://localhost:8123",
  "jsonData": {
    "defaultDatabase": "chuaikan_analytics",
    "username": "default",
    "tlsSkipVerify": true
  }
}
```

```sql
-- Grafana Dashboard Queries

-- Panel 1: DAU/MAU
SELECT
  event_date,
  uniqMerge(dau_state) AS dau
FROM mv_daily_active_users
WHERE event_date >= today() - 30
GROUP BY event_date
ORDER BY event_date;

-- Panel 2: Events per minute (real-time)
SELECT
  toStartOfMinute(timestamp) AS time,
  count() AS events
FROM user_events
WHERE timestamp >= now() - INTERVAL 1 HOUR
GROUP BY time
ORDER BY time;

-- Panel 3: Top Events
SELECT
  event_type,
  count() AS count
FROM user_events
WHERE event_date = today()
GROUP BY event_type
ORDER BY count DESC
LIMIT 10;

-- Panel 4: SOS Alerts by Province
SELECT
  province_code,
  count() AS alerts,
  JSONExtractString(properties, 'severity') AS severity
FROM user_events
WHERE event_type = 'sos_report'
  AND event_date >= today() - 7
GROUP BY province_code, severity
ORDER BY alerts DESC;
```

---

## 🔧 Configuration Files

```bash
# /etc/clickhouse-server/users.d/chuaikan.xml
<yandex>
  <users>
    <chuaikan_analytics>
      <password>your-secure-password</password>
      <networks>
        <ip>::1</ip>
        <ip>127.0.0.1</ip>
      </networks>
      <profile>default</profile>
      <quota>default</quota>
      <databases>
        <chuaikan_analytics/>
      </databases>
    </chuaikan_analytics>
  </users>
</yandex>
```

---

## 🧪 Testing

### Step 300: Analytics Tests & PDPA Compliance

```typescript
// tests/analytics.test.ts

import { describe, it, expect } from '@jest/globals';
import { AnalyticsQueryService } from '../src/services/analytics-query.service';

describe('ClickHouse Analytics', () => {
  const service = new AnalyticsQueryService();

  it('should return DAU data', async () => {
    const dau = await service.getDAU(7);
    expect(Array.isArray(dau)).toBe(true);
    expect(dau.length).toBeLessThanOrEqual(7);
  });

  it('should return funnel analysis', async () => {
    const startDate = '2024-01-01';
    const endDate = '2024-01-31';
    const funnel = await service.getFunnelAnalysis(startDate, endDate);

    expect(funnel.length).toBeGreaterThan(0);

    // Signup ต้องมี conversion_rate = 100
    const signup = funnel.find(f => f.step === 'signup');
    expect(signup?.conversion_rate).toBe(100);

    // ขั้นต่อไปต้องน้อยกว่าหรือเท่ากับ 100
    for (const step of funnel) {
      expect(parseFloat(step.conversion_rate)).toBeLessThanOrEqual(100);
    }
  });
});
```

```bash
# ทดสอบ PDPA data retention
# TTL ต้องลบข้อมูลอายุ > 2 ปี

clickhouse-client --query "
  SELECT
    min(event_date) AS oldest_date,
    max(event_date) AS newest_date,
    count()         AS total_events
  FROM chuaikan_analytics.user_events;
"

# ตรวจสอบ TTL ว่า configure ถูกต้อง
clickhouse-client --query "
  SHOW CREATE TABLE chuaikan_analytics.user_events
" | grep TTL

# Load test: insert 1M events
cat > /tmp/test-events.sql << 'EOF'
INSERT INTO chuaikan_analytics.user_events
  (event_type, event_category, session_id, anonymous_id, platform, timestamp)
SELECT
  ['page_view','post_like','user_login','post_create'][rand() % 4 + 1],
  'test',
  generateUUIDv4(),
  generateUUIDv4(),
  'web',
  now() - rand() % 86400
FROM numbers(1000000);
EOF

time clickhouse-client --queries-file /tmp/test-events.sql
# ควรเสร็จใน < 10 วินาที

# ตรวจสอบ query performance
clickhouse-client --query "
  EXPLAIN SELECT event_type, count()
  FROM chuaikan_analytics.user_events
  WHERE event_date = today()
  GROUP BY event_type
"
```

---

## ❌ Common Errors & Solutions

### Error 1: ClickHouse Connection Refused

```
Error: connect ECONNREFUSED 127.0.0.1:8123
```

```bash
# แก้: ตรวจสอบ ClickHouse service
sudo systemctl status clickhouse-server
sudo journalctl -u clickhouse-server -n 50

# ตรวจสอบ port
sudo netstat -tlnp | grep 8123

# ถ้า listen ที่ wrong interface
sudo nano /etc/clickhouse-server/config.xml
# <listen_host>0.0.0.0</listen_host>  ← เปลี่ยนเป็น 127.0.0.1 เพื่อ security
```

### Error 2: Kafka Consumer Lag

```
Consumer group clickhouse-analytics is 500,000 messages behind
```

```bash
# ตรวจสอบ consumer lag
sudo docker exec kafka-kafka-1 kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --describe --group clickhouse-analytics

# แก้: เพิ่ม consumer instances
# เพิ่ม CONSUMER_INSTANCES=4 ใน env
# และ rebalance

# Scale up ClickHouse insert batch size
CLICKHOUSE_INSERT_BATCH_SIZE=10000  # จาก 5000 → 10000
```

### Error 3: Materialized View Out of Sync

```
mv_daily_active_users shows 0 for yesterday
```

```sql
-- แก้: ตรวจสอบ materialized view
SELECT count() FROM mv_daily_active_users
WHERE event_date = yesterday();

-- ถ้า = 0 แต่ user_events มีข้อมูล → rebuild
DROP TABLE mv_daily_active_users;
-- สร้างใหม่ตาม schema ด้านบน

-- หรือ populate manually
INSERT INTO mv_daily_active_users
SELECT
  event_date,
  province_code,
  uniqState(user_id),
  uniqState(session_id),
  countState()
FROM user_events
WHERE event_date >= today() - 7
  AND user_id IS NOT NULL
GROUP BY event_date, province_code;
```

---

## ✅ Checklist

- [ ] **Step 291**: ติดตั้ง ClickHouse 24.x บน Ubuntu 24.04
- [ ] **Step 292**: ติดตั้ง Kafka ด้วย Docker Compose
- [ ] **Step 293**: ติดตั้ง Node.js dependencies
- [ ] **Step 294**: สร้าง user_events table และ Materialized Views
- [ ] **Step 295**: Implement Frontend Analytics SDK (pageview, click, scroll)
- [ ] **Step 296**: Implement Kafka Producer และ ClickHouse Consumer
- [ ] **Step 297**: Implement Analytics Query Service (DAU, MAU, Retention, Funnel)
- [ ] **Step 298**: สร้าง Analytics API Routes
- [ ] **Step 299**: Setup Grafana + ClickHouse datasource
- [ ] **Step 300**: Analytics tests + PDPA TTL verification
- [ ] Verify TTL configured (2 years)
- [ ] Verify Materialized Views populated
- [ ] Load test: 1M events insert < 10 seconds
- [ ] Verify Funnel Analysis query returns correct data
- [ ] Verify A/B Test analysis works
- [ ] Grafana dashboard แสดง DAU/MAU ถูกต้อง
- [ ] Kafka consumer lag < 30 seconds

---

## 🔗 References

- [ClickHouse Documentation](https://clickhouse.com/docs)
- [ClickHouse MergeTree Engines](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family)
- [KafkaJS Documentation](https://kafka.js.org/)
- [Grafana ClickHouse Plugin](https://grafana.com/grafana/plugins/grafana-clickhouse-datasource/)
- [PDPA Thailand](https://www.pdpa.pro/)

---
*Part 030 | Road to 1,000,000 Users/Day | chuaikan.com*
