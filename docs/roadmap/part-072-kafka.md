# Part 072: Apache Kafka สำหรับ Event Streaming
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Advanced
> **Steps:** 711-720
> **เวลาโดยประมาณ:** 8 ชั่วโมง
> **Prerequisites:** Part 005 (Redis), Part 006 (Docker), Part 008 (CI/CD), Part 012 (WebSocket)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- Kafka architecture และ concepts หลักทั้งหมด
- ติดตั้ง Kafka 3.7 ด้วย KRaft mode (ไม่ต้องใช้ Zookeeper) บน Ubuntu 24.04
- สร้าง Topics สำหรับ chuaikan.com แต่ละ service
- เขียน Producer และ Consumer ด้วย KafkaJS 2.x + TypeScript
- Exactly-once semantics สำหรับ SOS alerts
- Kafka Connect + Debezium สำหรับ PostgreSQL CDC
- Monitoring ด้วย Kafka UI

---

## 📖 ทฤษฎีและแนวคิด

### Kafka คืออะไร?

Kafka คือ distributed event streaming platform ที่ออกแบบมาสำหรับ high-throughput, fault-tolerant messaging ระหว่าง microservices

**Core Concepts:**

| Concept | คำอธิบาย |
|---------|-----------|
| **Topic** | ช่องทางสำหรับ events แต่ละประเภท (เหมือน table ใน database) |
| **Partition** | Topic แต่ละอันแบ่งเป็น partitions เพื่อ parallel processing |
| **Offset** | ตำแหน่งของ message ใน partition (เริ่มจาก 0) |
| **Consumer Group** | กลุ่ม consumers ที่แบ่งงานกันอ่าน partitions |
| **Lag** | จำนวน messages ที่ยังไม่ได้ consume (ถ้าสูง = มีปัญหา) |
| **Broker** | Kafka server แต่ละตัว |
| **Replication** | backup ข้อมูลข้าม brokers เพื่อ fault tolerance |

### KRaft Mode คืออะไร?

ตั้งแต่ Kafka 3.3+ สามารถรันได้โดยไม่ต้องใช้ Zookeeper ผ่าน **KRaft** (Kafka Raft) protocol ซึ่ง:
- ลด complexity ในการ deploy
- เร็วกว่า recovery
- ง่ายต่อการดูแล

### Topics สำหรับ chuaikan.com

```
user-events         - click, view, like, share ทุก action
sos-alerts          - SOS posts และ status updates (CRITICAL)
notifications       - push notifications ที่รอส่ง
analytics-events    - data สำหรับ analytics pipeline
feed-updates        - trigger recalculate feed เมื่อมี new post
```

---

## ⚙️ Environment Setup

### Step 711: ติดตั้ง Java (Kafka ต้องการ)

```bash
sudo apt update
sudo apt install -y openjdk-21-jdk
java -version
# openjdk version "21.0.x"
```

### Step 712: Download และติดตั้ง Kafka 3.7.0

```bash
cd /opt
sudo wget https://downloads.apache.org/kafka/3.7.0/kafka_2.13-3.7.0.tgz
sudo tar -xzf kafka_2.13-3.7.0.tgz
sudo ln -s kafka_2.13-3.7.0 kafka
sudo chown -R $USER:$USER /opt/kafka
```

### Step 713: กำหนดค่า KRaft Mode

สร้าง cluster ID:
```bash
KAFKA_CLUSTER_ID=$(/opt/kafka/bin/kafka-storage.sh random-uuid)
echo "Cluster ID: $KAFKA_CLUSTER_ID"
```

สร้าง config สำหรับ KRaft:
```bash
cat > /opt/kafka/config/kraft/server.properties << 'EOF'
# KRaft mode - no Zookeeper needed
process.roles=broker,controller
node.id=1
controller.quorum.voters=1@localhost:9093

# Listeners
listeners=PLAINTEXT://localhost:9092,CONTROLLER://localhost:9093
inter.broker.listener.name=PLAINTEXT
advertised.listeners=PLAINTEXT://localhost:9092
controller.listener.names=CONTROLLER
listener.security.protocol.map=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT

# Storage
log.dirs=/var/kafka/logs

# Topic defaults
num.partitions=6
default.replication.factor=1
min.insync.replicas=1
log.retention.hours=168
log.retention.bytes=10737418240

# Performance
num.network.threads=8
num.io.threads=16
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600

# Log cleanup
log.cleanup.policy=delete
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
EOF
```

Format storage:
```bash
sudo mkdir -p /var/kafka/logs
sudo chown -R $USER:$USER /var/kafka

/opt/kafka/bin/kafka-storage.sh format \
  -t $KAFKA_CLUSTER_ID \
  -c /opt/kafka/config/kraft/server.properties
```

### Step 714: สร้าง Systemd Service

```bash
sudo tee /etc/systemd/system/kafka.service << 'EOF'
[Unit]
Description=Apache Kafka Server (KRaft mode)
After=network.target

[Service]
Type=simple
User=ubuntu
Environment="JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64"
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/kraft/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable kafka
sudo systemctl start kafka
sudo systemctl status kafka
```

---

## 🛠️ Step-by-Step Implementation

### Step 715: สร้าง Topics

```bash
KAFKA_BIN=/opt/kafka/bin

# user-events: high volume, 6 partitions, 7 days retention
$KAFKA_BIN/kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --partitions 6 \
  --replication-factor 1 \
  --config retention.ms=604800000 \
  --config compression.type=lz4

# sos-alerts: critical, 6 partitions, 30 days retention
$KAFKA_BIN/kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --topic sos-alerts \
  --partitions 6 \
  --replication-factor 1 \
  --config retention.ms=2592000000 \
  --config min.insync.replicas=1 \
  --config unclean.leader.election.enable=false

# notifications: medium volume
$KAFKA_BIN/kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --topic notifications \
  --partitions 6 \
  --replication-factor 1 \
  --config retention.ms=604800000

# analytics-events
$KAFKA_BIN/kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --topic analytics-events \
  --partitions 6 \
  --replication-factor 1 \
  --config retention.ms=604800000 \
  --config compression.type=snappy

# feed-updates
$KAFKA_BIN/kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --topic feed-updates \
  --partitions 6 \
  --replication-factor 1 \
  --config retention.ms=86400000

# List all topics
$KAFKA_BIN/kafka-topics.sh --list --bootstrap-server localhost:9092
```

### Step 716: Kafka Producer ด้วย KafkaJS

```bash
cd /home/user/chuaikan-app
npm install kafkajs@2.2.4
```

```typescript
// src/lib/kafka/kafka-client.ts
import { Kafka, CompressionTypes, logLevel } from 'kafkajs';

export const kafka = new Kafka({
  clientId: 'chuaikan-app',
  brokers: [process.env.KAFKA_BROKER || 'localhost:9092'],
  logLevel: logLevel.WARN,
  retry: {
    initialRetryTime: 100,
    retries: 8,
    maxRetryTime: 30000,
    factor: 0.2,
    multiplier: 2,
  },
});

export const producer = kafka.producer({
  idempotent: true,          // exactly-once guarantee
  maxInFlightRequests: 5,
  transactionalId: undefined, // set per-transaction
});

export type SOSAlertEvent = {
  eventId: string;
  postId: string;
  userId: string;
  alertType: 'flood' | 'fire' | 'accident' | 'other';
  severity: 'low' | 'medium' | 'high' | 'critical';
  location: { lat: number; lng: number; province: string };
  description: string;
  timestamp: string;
};

export type UserEvent = {
  eventId: string;
  userId: string;
  action: 'view' | 'like' | 'comment' | 'share' | 'save' | 'sos_view';
  targetId: string;
  targetType: 'post' | 'user' | 'story';
  metadata: Record<string, unknown>;
  timestamp: string;
};
```

```typescript
// src/lib/kafka/sos-producer.ts
import { producer, kafka, SOSAlertEvent } from './kafka-client';
import { v4 as uuidv4 } from 'uuid';

let isConnected = false;

export async function connectProducer() {
  if (!isConnected) {
    await producer.connect();
    isConnected = true;
    console.log('[Kafka] Producer connected');
  }
}

export async function publishSOSAlert(data: Omit<SOSAlertEvent, 'eventId' | 'timestamp'>) {
  await connectProducer();

  const event: SOSAlertEvent = {
    ...data,
    eventId: uuidv4(),
    timestamp: new Date().toISOString(),
  };

  // Use postId as partition key เพื่อให้ events ของ post เดียวกันอยู่ใน partition เดียวกัน
  await producer.send({
    topic: 'sos-alerts',
    compression: CompressionTypes.GZIP,
    messages: [
      {
        key: event.postId,
        value: JSON.stringify(event),
        headers: {
          'event-type': 'SOS_ALERT_CREATED',
          'severity': event.severity,
          'source': 'chuaikan-app',
        },
      },
    ],
  });

  console.log(`[Kafka] SOS Alert published: ${event.eventId} (${event.severity})`);
  return event.eventId;
}

export async function publishUserEvent(data: Omit<UserEvent, 'eventId' | 'timestamp'>) {
  await connectProducer();

  const event: UserEvent = {
    ...data,
    eventId: uuidv4(),
    timestamp: new Date().toISOString(),
  };

  await producer.send({
    topic: 'user-events',
    compression: CompressionTypes.LZ4,
    messages: [
      {
        key: event.userId,  // same user → same partition
        value: JSON.stringify(event),
        headers: { 'event-type': event.action },
      },
    ],
  });
}

// Graceful shutdown
process.on('SIGTERM', async () => {
  await producer.disconnect();
  console.log('[Kafka] Producer disconnected');
});
```

### Step 717: Kafka Consumer with Consumer Group

```typescript
// src/services/notification-consumer.ts
import { kafka } from '../lib/kafka/kafka-client';
import { EachMessagePayload } from 'kafkajs';

const consumer = kafka.consumer({
  groupId: 'notification-service',
  maxWaitTimeInMs: 50,
  minBytes: 1,
  maxBytes: 1048576, // 1MB
});

async function processSOSAlert(payload: EachMessagePayload) {
  const { topic, partition, message } = payload;
  const event = JSON.parse(message.value!.toString());

  console.log(`[NotificationConsumer] Processing SOS: ${event.eventId} from partition ${partition}`);

  if (event.severity === 'critical') {
    // ส่ง push notification ด่วนไปยังผู้ใช้ในรัศมี 50km
    await sendEmergencyPush({
      province: event.location.province,
      lat: event.location.lat,
      lng: event.location.lng,
      alertType: event.alertType,
      description: event.description,
      postId: event.postId,
    });
  }

  // Mark offset เพื่อป้องกัน duplicate processing
  await consumer.commitOffsets([
    { topic, partition, offset: (BigInt(message.offset) + 1n).toString() },
  ]);
}

async function sendEmergencyPush(data: any) {
  // Implementation ใน Part 075
  console.log('[Push] Emergency push sent for province:', data.province);
}

export async function startNotificationConsumer() {
  await consumer.connect();
  await consumer.subscribe({ topics: ['sos-alerts', 'notifications'] });

  await consumer.run({
    autoCommit: false,  // manual commit for exactly-once
    eachBatchAutoResolve: false,
    eachMessage: processSOSAlert,
  });

  console.log('[Kafka] Notification consumer started');
}

// Analytics Consumer (separate group ทำให้อ่าน messages เดิมได้อีกครั้ง)
const analyticsConsumer = kafka.consumer({
  groupId: 'analytics-service',
  maxWaitTimeInMs: 500,
});

export async function startAnalyticsConsumer() {
  await analyticsConsumer.connect();
  await analyticsConsumer.subscribe({ topics: ['user-events', 'sos-alerts', 'analytics-events'] });

  await analyticsConsumer.run({
    autoCommit: true,
    autoCommitInterval: 5000,
    eachMessage: async ({ message }) => {
      const event = JSON.parse(message.value!.toString());
      // Insert into ClickHouse for analytics (Part 104)
      await insertAnalyticsEvent(event);
    },
  });
}

async function insertAnalyticsEvent(event: any) {
  // Batch insert to ClickHouse ทุก 1000 events
  console.log('[Analytics] Inserting event:', event.eventId);
}
```

### Step 718: Exactly-Once Semantics สำหรับ SOS

```typescript
// src/lib/kafka/transactional-producer.ts
import { kafka } from './kafka-client';
import { CompressionTypes } from 'kafkajs';

// Transactional producer สำหรับ SOS ที่ต้องการ exactly-once
const transactionalProducer = kafka.producer({
  transactionalId: 'sos-transactional-producer',
  idempotent: true,
  maxInFlightRequests: 1, // ต้องเป็น 1 สำหรับ transactional
});

export async function publishSOSWithTransaction(sosData: any, dbClient: any) {
  await transactionalProducer.connect();
  const transaction = await transactionalProducer.transaction();

  try {
    // 1. บันทึกลง PostgreSQL
    await dbClient.query(
      'INSERT INTO sos_events (id, data, status) VALUES ($1, $2, $3)',
      [sosData.eventId, JSON.stringify(sosData), 'published']
    );

    // 2. Publish to Kafka ใน transaction เดียวกัน
    await transaction.send({
      topic: 'sos-alerts',
      compression: CompressionTypes.GZIP,
      messages: [{ key: sosData.postId, value: JSON.stringify(sosData) }],
    });

    // 3. Commit transaction (ทั้ง DB และ Kafka)
    await transaction.commit();
    console.log('[Kafka] SOS transaction committed:', sosData.eventId);

  } catch (error) {
    await transaction.abort();
    console.error('[Kafka] Transaction aborted:', error);
    throw error;
  }
}
```

### Step 719: Kafka Connect + Debezium CDC

```bash
# Download Kafka Connect Debezium PostgreSQL connector
mkdir -p /opt/kafka/plugins/debezium-postgres
cd /opt/kafka/plugins/debezium-postgres
wget https://repo1.maven.org/maven2/io/debezium/debezium-connector-postgres/2.7.0.Final/debezium-connector-postgres-2.7.0.Final-plugin.tar.gz
tar -xzf debezium-connector-postgres-2.7.0.Final-plugin.tar.gz
```

```json
// config/debezium-postgres-connector.json
{
  "name": "chuaikan-postgres-cdc",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "localhost",
    "database.port": "5432",
    "database.user": "chuaikan_cdc",
    "database.password": "${env:POSTGRES_CDC_PASSWORD}",
    "database.dbname": "chuaikan",
    "database.server.name": "chuaikan-pg",
    "table.include.list": "public.posts,public.users,public.sos_posts",
    "plugin.name": "pgoutput",
    "slot.name": "debezium_slot",
    "publication.name": "debezium_publication",
    "topic.prefix": "cdc",
    "transforms": "route",
    "transforms.route.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter"
  }
}
```

```sql
-- PostgreSQL: Enable logical replication สำหรับ CDC
ALTER SYSTEM SET wal_level = logical;
SELECT pg_reload_conf();

-- สร้าง replication user
CREATE USER chuaikan_cdc WITH REPLICATION LOGIN PASSWORD 'secure_password';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO chuaikan_cdc;

-- สร้าง publication
CREATE PUBLICATION debezium_publication FOR TABLE posts, users, sos_posts;
```

---

## 🔧 Configuration Files

### Monitoring: Kafka UI ด้วย Docker Compose

```yaml
# docker-compose.kafka-ui.yml
version: '3.8'
services:
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: chuaikan-kafka
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: host.docker.internal:9092
      KAFKA_CLUSTERS_0_METRICS_PORT: 9997
      AUTH_TYPE: LOGIN_FORM
      SPRING_SECURITY_USER_NAME: admin
      SPRING_SECURITY_USER_PASSWORD: ${KAFKA_UI_PASSWORD}
    extra_hosts:
      - "host.docker.internal:host-gateway"
    restart: unless-stopped
```

```bash
docker compose -f docker-compose.kafka-ui.yml up -d
# เปิด http://localhost:8080
```

### JMX Metrics Export (สำหรับ Prometheus)

```bash
# เพิ่มใน kafka startup script
export KAFKA_JMX_OPTS="-Dcom.sun.management.jmxremote \
  -Dcom.sun.management.jmxremote.authenticate=false \
  -Dcom.sun.management.jmxremote.ssl=false \
  -Dcom.sun.management.jmxremote.port=9997"
```

---

## 🧪 Testing

### Test Producer ด้วย CLI

```bash
# ส่ง test message
echo '{"postId":"test-123","severity":"critical","alertType":"flood"}' | \
  /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic sos-alerts \
  --property parse.key=false

# อ่าน messages
/opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic sos-alerts \
  --from-beginning \
  --max-messages 10
```

### Test Producer ด้วย Node.js

```typescript
// scripts/test-kafka-producer.ts
import { publishSOSAlert, connectProducer } from '../src/lib/kafka/sos-producer';

async function main() {
  await connectProducer();

  for (let i = 0; i < 5; i++) {
    const eventId = await publishSOSAlert({
      postId: `post-${i}`,
      userId: `user-${i % 100}`,
      alertType: 'flood',
      severity: 'critical',
      location: { lat: 14.3500, lng: 100.5600, province: 'Ayutthaya' },
      description: 'น้ำท่วมหนักมาก ต้องการความช่วยเหลือด่วน',
    });
    console.log(`Published event: ${eventId}`);
  }

  process.exit(0);
}

main().catch(console.error);
```

```bash
npx ts-node scripts/test-kafka-producer.ts
```

### ตรวจสอบ Consumer Lag

```bash
/opt/kafka/bin/kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group notification-service
```

---

## ❌ Common Errors & Solutions

### Error 1: Consumer Rebalancing บ่อยเกินไป

```
[Kafka] Consumer rebalance triggered
```

**สาเหตุ:** `session.timeout.ms` น้อยเกินไป หรือ processing ช้า

**แก้ไข:**
```typescript
const consumer = kafka.consumer({
  groupId: 'notification-service',
  sessionTimeout: 30000,    // เพิ่มจาก default 30s → 60s
  heartbeatInterval: 3000,
  maxPollIntervalMs: 300000, // ให้เวลา process 5 นาที
});
```

### Error 2: Lag สะสมมากขึ้นเรื่อยๆ

**สาเหตุ:** Consumer ช้ากว่า Producer

**แก้ไข:**
```typescript
// เพิ่ม partition และ scale consumers
// consumers จะต้อง <= partitions
await consumer.run({
  partitionsConsumedConcurrently: 3, // process หลาย partitions พร้อมกัน
  eachMessage: async (payload) => {
    // เพิ่ม batch processing
  },
});
```

### Error 3: Offset Out of Range

```
OffsetOutOfRange: The requested offset is not within the range of offsets maintained by the server
```

**แก้ไข:**
```typescript
const consumer = kafka.consumer({
  groupId: 'my-group',
  // เมื่อ offset หมดอายุ ให้เริ่มจาก earliest
});

consumer.on('consumer.group_join', (event) => {
  console.log('[Kafka] Consumer joined group', event.payload.groupId);
});
```

### Error 4: Message Too Large

```
MessageSizeTooLarge: The message is too large for the server
```

**แก้ไข:**
```bash
# เพิ่มใน server.properties
message.max.bytes=10485760    # 10MB
replica.fetch.max.bytes=10485760
```

---

## ✅ Checklist

- [ ] **Step 711:** Java 21 ติดตั้งสำเร็จ, `java -version` แสดงผลถูกต้อง
- [ ] **Step 712:** Kafka 3.7.0 download และ extract ไปที่ `/opt/kafka`
- [ ] **Step 713:** สร้าง KRaft config และ format storage directory
- [ ] **Step 714:** Kafka systemd service รันอยู่ (`systemctl status kafka`)
- [ ] **Step 715:** สร้างครบ 5 topics: user-events, sos-alerts, notifications, analytics-events, feed-updates
- [ ] **Step 716:** KafkaJS producer ทำงานได้ ส่ง SOS alert สำเร็จ
- [ ] **Step 717:** Consumer groups ทั้ง 2 (notification-service, analytics-service) รันได้
- [ ] **Step 718:** Exactly-once transaction ทำงานได้
- [ ] **Step 719:** Debezium connector setup สำหรับ PostgreSQL CDC
- [ ] **Step 720:** Kafka UI เปิดได้ที่ port 8080 แสดง topics และ consumer groups

---

## 🔗 References

- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [KafkaJS Documentation](https://kafka.js.org/)
- [KRaft Mode Migration Guide](https://kafka.apache.org/documentation/#kraft)
- [Debezium PostgreSQL Connector](https://debezium.io/documentation/reference/connectors/postgresql.html)
- [Kafka UI GitHub](https://github.com/provectus/kafka-ui)

---
*Part 072 | Road to 1,000,000 Users/Day | chuaikan.com*
