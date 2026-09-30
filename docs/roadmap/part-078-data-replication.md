# Part 078: Data Replication Across Regions
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Professional
> **Steps:** 771–780
> **เวลาโดยประมาณ:** 8 ชั่วโมง
> **Prerequisites:** Part 076 (Multi-region), Part 041 (DB Sharding)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- PostgreSQL logical replication แบบ cross-region (Singapore → Tokyo)
- Conflict resolution ใน bidirectional replication
- Redis replication lag monitoring
- S3 bucket replication rules
- Kafka MirrorMaker 2 สำหรับ cross-region event streaming
- RPO/RTO targets สำหรับ chuaikan.com
- Replication monitoring dashboard

---

## 📖 ทฤษฎีและแนวคิด

### ประเภทของ Data Replication

| ประเภท | Consistency | Latency | Use Case |
|--------|-------------|---------|----------|
| Synchronous | Strong | สูง (รอ ACK) | Financial data |
| Asynchronous | Eventual | ต่ำ | Read replicas |
| Logical | Granular | ต่ำ | Cross-version replication |

### RPO/RTO สำหรับ chuaikan.com

- **RPO (Recovery Point Objective)**: 5 นาที
  - ข้อมูลสูญหายได้ไม่เกิน 5 นาที หาก primary ล้มเหลว
  - วัดจาก replication lag ที่ยอมรับได้
- **RTO (Recovery Time Objective)**: 30 นาที
  - ระบบต้องกลับมา serve traffic ได้ภายใน 30 นาที

---

## 🛠️ Step-by-Step Implementation

### Step 771: PostgreSQL Logical Replication Setup

```bash
# ===== PRIMARY SERVER (Singapore) =====

# แก้ไข postgresql.conf
sudo -u postgres cat >> /etc/postgresql/16/main/postgresql.conf << 'EOF'
wal_level = logical
max_replication_slots = 20
max_wal_senders = 20
wal_keep_size = 1024        # เก็บ WAL ไว้ 1GB กัน DR lag
max_slot_wal_keep_size = 2048  # สูงสุด 2GB per slot
EOF

# แก้ไข pg_hba.conf สำหรับ replication user
sudo -u postgres cat >> /etc/postgresql/16/main/pg_hba.conf << 'EOF'
# Replication from Tokyo DR
hostssl replication replicator 13.230.0.0/15 scram-sha-256
hostssl chuaikan   replicator 13.230.0.0/15 scram-sha-256
EOF

sudo systemctl reload postgresql@16-main
```

```sql
-- สร้าง tables สำหรับ track changes
-- primary server
CREATE TABLE IF NOT EXISTS replication_heartbeat (
    id          SERIAL PRIMARY KEY,
    region      VARCHAR(50) NOT NULL,
    beat_time   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    notes       TEXT
);

-- สร้าง user และ permissions
CREATE USER replicator WITH REPLICATION LOGIN ENCRYPTED PASSWORD 'Repl1c@tor_S3cure!';

-- Grant สำหรับ monitoring
GRANT CONNECT ON DATABASE chuaikan TO replicator;
GRANT USAGE ON SCHEMA public TO replicator;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO replicator;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO replicator;

-- Publication ครอบคลุมทุก tables
CREATE PUBLICATION chuaikan_full_pub
  FOR ALL TABLES
  WITH (publish = 'insert,update,delete,truncate');

-- หรือ publication เฉพาะ tables ที่สำคัญ
CREATE PUBLICATION chuaikan_critical_pub FOR TABLE
  users,
  posts,
  comments,
  likes,
  follows,
  notifications,
  sos_alerts,
  sos_responses,
  replication_heartbeat;
```

```bash
# ===== REPLICA SERVER (Tokyo) =====

# Dump schema จาก primary (ไม่ copy data)
pg_dump \
  --host=sg.db.chuaikan.com \
  --username=replicator \
  --schema-only \
  --no-owner \
  --no-acl \
  chuaikan > /tmp/schema_only.sql

# Import schema ไป DR
psql -U postgres -d chuaikan < /tmp/schema_only.sql

# สร้าง subscription
psql -U postgres -d chuaikan << 'EOF'
CREATE SUBSCRIPTION chuaikan_sub
  CONNECTION 'host=sg.db.chuaikan.com
              port=5432
              dbname=chuaikan
              user=replicator
              password=Repl1c@tor_S3cure!
              sslmode=require
              sslrootcert=/etc/postgresql/ssl/ca-cert.pem'
  PUBLICATION chuaikan_full_pub
  WITH (
    copy_data = true,
    create_slot = true,
    enabled = true,
    connect = true,
    slot_name = 'chuaikan_tokyo_slot'
  );

-- ตรวจสอบ
SELECT * FROM pg_subscription;
SELECT * FROM pg_stat_subscription;
EOF
```

### Step 772: Monitoring Replication Lag

```sql
-- บน PRIMARY: ตรวจสอบ replication slots
SELECT
    slot_name,
    active,
    active_pid,
    restart_lsn,
    confirmed_flush_lsn,
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)
    ) AS lag_bytes,
    CASE
        WHEN confirmed_flush_lsn IS NULL THEN NULL
        ELSE EXTRACT(EPOCH FROM (NOW() - pg_last_xact_replay_timestamp()))
    END AS estimated_lag_seconds
FROM pg_replication_slots
WHERE slot_type = 'logical';
```

```python
# scripts/replication_monitor.py
"""
Monitor PostgreSQL replication lag and send alerts
"""

import psycopg2
import boto3
import time
import logging
from dataclasses import dataclass
from typing import Optional

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@dataclass
class ReplicationStatus:
    slot_name: str
    active: bool
    lag_bytes: int
    lag_seconds: Optional[float]
    replica_host: str

def get_replication_status(primary_dsn: str) -> list[ReplicationStatus]:
    """ดึงสถานะ replication จาก primary"""
    conn = psycopg2.connect(primary_dsn)
    cur = conn.cursor()

    cur.execute("""
        SELECT
            s.slot_name,
            s.active,
            COALESCE(
                pg_wal_lsn_diff(pg_current_wal_lsn(), s.confirmed_flush_lsn),
                0
            ) AS lag_bytes,
            EXTRACT(EPOCH FROM (NOW() - ps.reply_time)) AS lag_seconds,
            COALESCE(ps.client_addr::text, 'disconnected') AS replica_host
        FROM pg_replication_slots s
        LEFT JOIN pg_stat_replication ps ON ps.sent_lsn IS NOT NULL
        WHERE s.slot_type = 'logical'
        ORDER BY lag_bytes DESC;
    """)

    results = []
    for row in cur.fetchall():
        results.append(ReplicationStatus(
            slot_name=row[0],
            active=row[1],
            lag_bytes=row[2],
            lag_seconds=float(row[3]) if row[3] else None,
            replica_host=row[4]
        ))

    cur.close()
    conn.close()
    return results

def publish_metrics(status_list: list[ReplicationStatus]):
    """ส่ง metrics ไป CloudWatch"""
    cw = boto3.client('cloudwatch', region_name='ap-southeast-1')
    metric_data = []

    for status in status_list:
        metric_data.extend([
            {
                'MetricName': 'ReplicationLagBytes',
                'Dimensions': [{'Name': 'SlotName', 'Value': status.slot_name}],
                'Value': float(status.lag_bytes),
                'Unit': 'Bytes'
            },
            {
                'MetricName': 'ReplicationActive',
                'Dimensions': [{'Name': 'SlotName', 'Value': status.slot_name}],
                'Value': 1.0 if status.active else 0.0,
                'Unit': 'Count'
            }
        ])

        if status.lag_seconds is not None:
            metric_data.append({
                'MetricName': 'ReplicationLagSeconds',
                'Dimensions': [{'Name': 'SlotName', 'Value': status.slot_name}],
                'Value': status.lag_seconds,
                'Unit': 'Seconds'
            })

    cw.put_metric_data(
        Namespace='chuaikan/PostgreSQL/Replication',
        MetricData=metric_data
    )

def check_alerts(status_list: list[ReplicationStatus]):
    """ตรวจสอบและส่ง alert"""
    for status in status_list:
        # Alert: Replication ขาด
        if not status.active:
            logger.error(f"CRITICAL: Replication slot {status.slot_name} is NOT ACTIVE!")
            send_pagerduty_alert(
                severity="critical",
                summary=f"PostgreSQL replication slot {status.slot_name} is disconnected",
                details={"slot": status.slot_name, "host": status.replica_host}
            )

        # Alert: Lag > 5 นาที (RPO breach)
        elif status.lag_seconds and status.lag_seconds > 300:
            logger.warning(
                f"WARNING: Replication lag {status.lag_seconds:.0f}s "
                f"exceeds RPO of 300s on slot {status.slot_name}"
            )
            send_pagerduty_alert(
                severity="warning",
                summary=f"Replication lag exceeds RPO: {status.lag_seconds:.0f}s",
                details={"slot": status.slot_name, "lag_seconds": status.lag_seconds}
            )

def send_pagerduty_alert(severity: str, summary: str, details: dict):
    """ส่ง PagerDuty alert (placeholder)"""
    logger.info(f"[{severity.upper()}] {summary}: {details}")
    # ใส่ PagerDuty integration key จริง

def main():
    primary_dsn = "host=sg.db.chuaikan.com dbname=chuaikan user=monitor password=monitor_pass"

    while True:
        try:
            statuses = get_replication_status(primary_dsn)
            publish_metrics(statuses)
            check_alerts(statuses)

            for s in statuses:
                logger.info(
                    f"Slot: {s.slot_name} | Active: {s.active} | "
                    f"Lag: {s.lag_bytes/1024:.1f}KB | Replica: {s.replica_host}"
                )
        except Exception as e:
            logger.error(f"Error checking replication: {e}")

        time.sleep(30)

if __name__ == "__main__":
    main()
```

### Step 773: Conflict Resolution

```sql
-- สำหรับ bidirectional replication (ถ้าต้องการ Active-Active)
-- ใช้ timestamp-based conflict resolution

-- สร้าง function จัดการ conflict
CREATE OR REPLACE FUNCTION resolve_conflict()
RETURNS TRIGGER AS $$
BEGIN
    -- ถ้า updated_at ของ NEW น้อยกว่า current → ไม่ update
    IF TG_OP = 'UPDATE' AND NEW.updated_at < OLD.updated_at THEN
        RETURN OLD;  -- เก็บ version ที่ใหม่กว่า
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply ให้ทุก table ที่มี updated_at
CREATE TRIGGER conflict_resolution
    BEFORE UPDATE ON posts
    FOR EACH ROW EXECUTE FUNCTION resolve_conflict();
```

### Step 774: Redis Replication Monitoring

```python
# scripts/redis_replication_monitor.py

import redis
import boto3
import time
import logging

logger = logging.getLogger(__name__)

def get_redis_replication_info(host: str, port: int = 6379, password: str = None):
    """ดึง replication info จาก Redis"""
    r = redis.Redis(
        host=host,
        port=port,
        password=password,
        ssl=True,
        decode_responses=True
    )

    info = r.info('replication')
    return info

def check_redis_lag(primary_host: str, replica_host: str, password: str):
    """ตรวจสอบ Redis replication lag"""
    primary_info = get_redis_replication_info(primary_host, password=password)
    replica_info = get_redis_replication_info(replica_host, password=password)

    primary_offset = int(primary_info.get('master_repl_offset', 0))
    replica_offset = int(replica_info.get('master_repl_offset', 0))
    lag_bytes = primary_offset - replica_offset

    connected_slaves = int(primary_info.get('connected_slaves', 0))

    # ตรวจสอบ slave-specific lag
    for i in range(connected_slaves):
        slave_info = primary_info.get(f'slave{i}', '')
        if slave_info:
            # Format: ip=x.x.x.x,port=6379,state=online,offset=123,lag=0
            parts = dict(p.split('=') for p in slave_info.split(','))
            slave_lag = int(parts.get('lag', -1))
            logger.info(f"Slave {i}: offset={parts.get('offset')}, lag={slave_lag}s")

            if slave_lag > 10:
                logger.warning(f"Redis slave {i} lag: {slave_lag}s > threshold 10s")

    return lag_bytes

def monitor_redis_global_datastore():
    """Monitor ElastiCache Global Datastore"""
    ec_client = boto3.client('elasticache', region_name='ap-southeast-1')

    response = ec_client.describe_global_replication_groups(
        GlobalReplicationGroupId='chuaikan-global',
        ShowMemberInfo=True
    )

    for group in response['GlobalReplicationGroups']:
        for member in group.get('Members', []):
            print(f"Region: {member['ReplicationGroupRegion']}")
            print(f"Status: {member['Status']}")
            print(f"Role: {member['Role']}")
            print("---")

if __name__ == "__main__":
    # Check replication lag ทุก 30 วินาที
    while True:
        lag = check_redis_lag(
            primary_host="chuaikan-redis.cluster.sg.cache.amazonaws.com",
            replica_host="chuaikan-redis.cluster.jp.cache.amazonaws.com",
            password="redis_auth_token_here"
        )
        logger.info(f"Redis replication lag: {lag} bytes")
        time.sleep(30)
```

### Step 775: Kafka MirrorMaker 2

```yaml
# kafka/mirrormaker2-config.yaml
# MirrorMaker 2 สำหรับ cross-region event streaming

apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-mirrormaker2
  namespace: kafka
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mirrormaker2
  template:
    metadata:
      labels:
        app: mirrormaker2
    spec:
      containers:
        - name: mirrormaker2
          image: confluentinc/cp-kafka:7.6.0
          command:
            - /bin/connect-distributed
            - /etc/kafka/mirrormaker2.properties
          env:
            - name: KAFKA_HEAP_OPTS
              value: "-Xms512m -Xmx2g"
          volumeMounts:
            - name: config
              mountPath: /etc/kafka
          resources:
            requests:
              memory: "1Gi"
              cpu: "500m"
            limits:
              memory: "2Gi"
              cpu: "2000m"
      volumes:
        - name: config
          configMap:
            name: mirrormaker2-config
```

```properties
# kafka/mirrormaker2.properties

# MirrorMaker 2 Connector Configuration
# Source: Singapore, Target: Tokyo

# Cluster aliases
clusters = sg, jp

# Singapore cluster (source)
sg.bootstrap.servers = kafka-sg.chuaikan.com:9092
sg.security.protocol = SSL
sg.ssl.truststore.location = /etc/kafka/ssl/truststore.jks
sg.ssl.truststore.password = truststore_password

# Tokyo cluster (target)
jp.bootstrap.servers = kafka-jp.chuaikan.com:9092
jp.security.protocol = SSL
jp.ssl.truststore.location = /etc/kafka/ssl/truststore.jks
jp.ssl.truststore.password = truststore_password

# Mirror Maker configuration
sg->jp.enabled = true
sg->jp.topics = chuaikan.posts, chuaikan.notifications, chuaikan.sos-alerts
sg->jp.groups = chuaikan-consumers
sg->jp.emit.heartbeats.enabled = true
sg->jp.emit.checkpoints.enabled = true
sg->jp.sync.group.offsets.enabled = true

# Replication factor for mirrored topics
replication.factor = 2

# Offset translation (สำหรับ consumer failover)
sg->jp.emit.checkpoints.interval.seconds = 60

# Performance tuning
tasks.max = 4
producer.override.acks = 1
producer.override.compression.type = lz4
```

```bash
# ตรวจสอบ MirrorMaker 2 status
kubectl exec -n kafka deploy/kafka-mirrormaker2 -- \
  kafka-consumer-groups.sh \
  --bootstrap-server kafka-jp.chuaikan.com:9092 \
  --group mirrormaker2 \
  --describe

# ดู mirror topics
kubectl exec -n kafka deploy/kafka-mirrormaker2 -- \
  kafka-topics.sh \
  --bootstrap-server kafka-jp.chuaikan.com:9092 \
  --list | grep "sg\."
# ควรเห็น: sg.chuaikan.posts, sg.chuaikan.notifications, etc.
```

### Step 776: S3 Replication Rules

```python
# scripts/setup_s3_replication.py

import boto3

def setup_s3_replication():
    """ตั้งค่า S3 Cross-Region Replication"""
    s3_sg = boto3.client('s3', region_name='ap-southeast-1')

    replication_config = {
        'Role': 'arn:aws:iam::123456789012:role/s3-replication-role',
        'Rules': [
            {
                'ID': 'replicate-user-uploads',
                'Status': 'Enabled',
                'Priority': 1,
                'Filter': {
                    'Prefix': 'uploads/'
                },
                'Destination': {
                    'Bucket': 'arn:aws:s3:::chuaikan-assets-dr',
                    'StorageClass': 'STANDARD_IA',
                    'ReplicationTime': {
                        'Status': 'Enabled',
                        'Time': {'Minutes': 15}
                    },
                    'Metrics': {
                        'Status': 'Enabled',
                        'EventThreshold': {'Minutes': 15}
                    }
                },
                'DeleteMarkerReplication': {'Status': 'Enabled'}
            },
            {
                'ID': 'replicate-profile-images',
                'Status': 'Enabled',
                'Priority': 2,
                'Filter': {
                    'Prefix': 'profiles/'
                },
                'Destination': {
                    'Bucket': 'arn:aws:s3:::chuaikan-assets-dr',
                    'StorageClass': 'STANDARD'
                },
                'DeleteMarkerReplication': {'Status': 'Enabled'}
            }
        ]
    }

    s3_sg.put_bucket_replication(
        Bucket='chuaikan-assets-primary',
        ReplicationConfiguration=replication_config
    )

    print("S3 replication configured successfully!")

def monitor_s3_replication():
    """ตรวจสอบ S3 Replication Metrics"""
    cw = boto3.client('cloudwatch', region_name='ap-southeast-1')

    response = cw.get_metric_statistics(
        Namespace='AWS/S3',
        MetricName='ReplicationLatency',
        Dimensions=[
            {'Name': 'SourceBucket', 'Value': 'chuaikan-assets-primary'},
            {'Name': 'DestinationBucket', 'Value': 'chuaikan-assets-dr'},
            {'Name': 'RuleId', 'Value': 'replicate-user-uploads'}
        ],
        StartTime='2024-01-01T00:00:00Z',
        EndTime='2024-01-01T01:00:00Z',
        Period=300,
        Statistics=['Average', 'Maximum']
    )

    for dp in response['Datapoints']:
        print(f"Replication Latency - Avg: {dp['Average']:.1f}s, Max: {dp['Maximum']:.1f}s")

if __name__ == "__main__":
    setup_s3_replication()
    monitor_s3_replication()
```

### Step 777: Replication Monitoring Dashboard

```yaml
# k8s/grafana/replication-dashboard.yaml
# Grafana Dashboard สำหรับ Replication

apiVersion: v1
kind: ConfigMap
metadata:
  name: replication-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "true"
data:
  replication-dashboard.json: |
    {
      "title": "Data Replication - chuaikan.com",
      "panels": [
        {
          "title": "PostgreSQL Replication Lag (seconds)",
          "type": "timeseries",
          "targets": [{
            "expr": "aws_cloudwatch_metric{namespace='chuaikan/PostgreSQL/Replication', metric_name='ReplicationLagSeconds'}",
            "legendFormat": "{{slot_name}}"
          }],
          "alert": {
            "conditions": [{
              "query": {"params": ["A", "5m", "now"]},
              "reducer": {"params": [], "type": "last"},
              "evaluator": {"params": [300], "type": "gt"}
            }],
            "name": "Replication lag > 5 minutes",
            "frequency": "1m",
            "for": "5m",
            "notifications": [{"id": 1}]
          }
        },
        {
          "title": "Redis Global Datastore Lag",
          "type": "stat",
          "targets": [{
            "expr": "redis_connected_slaves",
            "legendFormat": "Connected Replicas"
          }]
        },
        {
          "title": "S3 Replication Latency",
          "type": "gauge",
          "targets": [{
            "expr": "aws_s3_replication_latency_average",
            "legendFormat": "Avg Latency"
          }],
          "fieldConfig": {
            "defaults": {
              "max": 900,
              "thresholds": {
                "steps": [
                  {"color": "green", "value": null},
                  {"color": "yellow", "value": 300},
                  {"color": "red", "value": 600}
                ]
              }
            }
          }
        }
      ]
    }
```

---

## 🧪 Testing

```bash
# ทดสอบ PostgreSQL replication
# 1. Insert ข้อมูลบน primary
psql -h sg.db.chuaikan.com -U postgres -d chuaikan -c \
  "INSERT INTO replication_heartbeat (region, notes) VALUES ('singapore', 'test-$(date +%s)');"

# 2. ตรวจสอบบน replica ภายใน 5 วินาที
sleep 5
psql -h jp.db.chuaikan.com -U postgres -d chuaikan -c \
  "SELECT * FROM replication_heartbeat ORDER BY beat_time DESC LIMIT 1;"

# ทดสอบ Redis replication
redis-cli -h primary.redis.chuaikan.com SET test_key "hello_$(date +%s)"
sleep 2
redis-cli -h replica.redis.chuaikan.com GET test_key
# ควรได้ค่าเดียวกัน

# ทดสอบ S3 replication
aws s3 cp test-file.txt s3://chuaikan-assets-primary/test/test-file.txt
sleep 30
aws s3 ls s3://chuaikan-assets-dr/test/ --region ap-northeast-1
# ควรเห็น test-file.txt
```

---

## ✅ Checklist

- [ ] PostgreSQL logical replication ทำงาน (lag < 30 วินาที ปกติ, < 5 นาที worst case)
- [ ] Replication slot ไม่มี orphan slots (ลบ inactive slots)
- [ ] `pg_replication_slots` monitor อัตโนมัติ
- [ ] Redis Global Datastore connected ระหว่าง regions
- [ ] Redis lag < 5 วินาที
- [ ] S3 Cross-Region Replication เปิด + Replication Time Control (15 นาที)
- [ ] Kafka MirrorMaker 2 replicating critical topics
- [ ] Consumer group offsets sync (สำหรับ failover consumers)
- [ ] Grafana dashboard แสดง all replication metrics
- [ ] CloudWatch alarms สำหรับ lag > RPO threshold
- [ ] Failover drill ทำทุก quarter (ทดสอบ RPO จริง)

---

## 🔗 References

- [PostgreSQL Logical Replication](https://www.postgresql.org/docs/current/logical-replication.html)
- [Kafka MirrorMaker 2](https://kafka.apache.org/documentation/#georeplication)
- [S3 Replication](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html)

---
*Part 078 | Road to 1,000,000 Users/Day | chuaikan.com*
