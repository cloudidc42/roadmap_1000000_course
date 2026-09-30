# Part 104: Real-time Analytics Dashboard

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** World Class
> **Steps:** 1031-1040
> **เวลาโดยประมาณ:** 5 ชั่วโมง
> **Prerequisites:** Part 092 (Distributed Systems), Part 094 (Distributed Tracing)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

Real-time Analytics คือ "ดวงตา" ของ chuaikan.com ที่ช่วยให้เห็นสิ่งที่เกิดขึ้น ณ ปัจจุบัน ใน Part นี้เราจะ:

- ติดตั้ง ClickHouse สำหรับ OLAP Queries ที่เร็วมาก
- สร้าง Kafka → ClickHouse Streaming Pipeline
- สร้าง Grafana Dashboard แบบ Real-time
- ระบบ Alert สำหรับ User Spike และ SOS Cluster
- Dashboard สำหรับ Emergency Responders

---

## 📖 ทฤษฎีและแนวคิด

### 1. OLTP vs OLAP

```
OLTP (Online Transaction Processing):
- PostgreSQL ที่เราใช้อยู่
- Optimized สำหรับ row-level operations (INSERT, UPDATE, SELECT by PK)
- ดีสำหรับ: บันทึก SOS, อัพเดท user profile
- ไม่ดีสำหรับ: "แสดง SOS ทั้งหมดใน 1 ชั่วโมงที่ผ่านมาแยกตาม region"

OLAP (Online Analytical Processing):
- ClickHouse ที่เราจะติดตั้ง
- Optimized สำหรับ aggregation queries บน columns
- ดีสำหรับ: Analytics, Dashboards, Reports
- Query ที่ใช้เวลาหลายนาทีบน PostgreSQL → หลายมิลลิวินาทีบน ClickHouse
```

### 2. ClickHouse ทำงานอย่างไร

```
ClickHouse เก็บข้อมูลแบบ Columnar:

PostgreSQL (Row Storage):
Row 1: [user_id=1, event=login, time=10:00, region=Bangkok]
Row 2: [user_id=2, event=post,  time=10:01, region=Chiang Mai]
Row 3: [user_id=3, event=sos,   time=10:02, region=Bangkok]

ClickHouse (Column Storage):
user_id:  [1, 2, 3, ...]
event:    [login, post, sos, ...]
time:     [10:00, 10:01, 10:02, ...]
region:   [Bangkok, Chiang Mai, Bangkok, ...]

เมื่อ query: SELECT COUNT(*) WHERE region='Bangkok' AND event='sos'
→ ClickHouse อ่านแค่ 2 columns (region, event) ไม่ต้องอ่าน user_id, time
→ เร็วกว่ามาก!
```

### 3. Kafka → ClickHouse Architecture

```
Events Flow:

App Services → Kafka Topics → ClickHouse (via Kafka Engine)
                                    ↓
                              Grafana Dashboards
                                    ↓  
                            Emergency Responders
```

---

## ⚙️ Environment Setup

### ติดตั้ง ClickHouse บน Kubernetes

```bash
# เพิ่ม Helm repo
helm repo add clickhouse-operator https://docs.altinity.com/clickhouse-operator/
helm repo update

# ติดตั้ง ClickHouse Operator
helm install clickhouse-operator clickhouse-operator/clickhouse-operator \
  --namespace clickhouse \
  --create-namespace
```

```yaml
# clickhouse-cluster.yaml
apiVersion: "clickhouse.altinity.com/v1"
kind: "ClickHouseInstallation"
metadata:
  name: chuaikan-analytics
  namespace: clickhouse
spec:
  configuration:
    zookeeper:
      nodes:
        - host: zookeeper.clickhouse
          port: 2181
    
    clusters:
      - name: "chuaikan"
        layout:
          shardsCount: 2     # 2 shards
          replicasCount: 2   # 2 replicas per shard
        
        templates:
          podTemplate: clickhouse-pod
          dataVolumeClaimTemplate: data-volume
  
  templates:
    podTemplates:
      - name: clickhouse-pod
        spec:
          containers:
          - name: clickhouse
            image: clickhouse/clickhouse-server:23.12
            resources:
              requests:
                memory: "4Gi"
                cpu: "2"
              limits:
                memory: "8Gi"
                cpu: "4"
    
    volumeClaimTemplates:
      - name: data-volume
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 500Gi
          storageClassName: gp3
```

```bash
kubectl apply -f clickhouse-cluster.yaml

# รอ cluster พร้อม
kubectl get clickhouseinstallation -n clickhouse
# NAME                 STATUS    CLUSTERS  SHARDS  HOSTS   AGE
# chuaikan-analytics   Complete  1         2       4       5m
```

---

## 🛠️ Step-by-Step Implementation

### Step 1031: สร้าง Schema ใน ClickHouse

```sql
-- ต่อ ClickHouse
kubectl exec -n clickhouse chi-chuaikan-analytics-chuaikan-0-0-0 -- \
  clickhouse-client --user default --password $CH_PASSWORD

-- สร้าง Database
CREATE DATABASE IF NOT EXISTS analytics ON CLUSTER chuaikan;

-- Events Table (Distributed)
CREATE TABLE IF NOT EXISTS analytics.events ON CLUSTER chuaikan
(
    event_id      UUID,
    event_type    LowCardinality(String),   -- post/like/comment/sos/login
    user_id       UInt64,
    session_id    String,
    
    -- Location
    region        LowCardinality(String),
    province      LowCardinality(String),
    lat           Float32,
    lng           Float32,
    
    -- Content
    sos_id        Nullable(UInt64),
    sos_severity  LowCardinality(Nullable(String)),  -- critical/high/medium/low
    
    -- Time
    event_time    DateTime,
    event_date    Date MATERIALIZED toDate(event_time),
    event_hour    UInt8 MATERIALIZED toHour(event_time),
    
    -- Device
    platform      LowCardinality(String),  -- ios/android/web
    app_version   String,
    
    -- Metadata
    processing_time_ms UInt32,
    created_at    DateTime DEFAULT now()
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/events', '{replica}')
PARTITION BY (event_date, region)
ORDER BY (event_time, event_type, user_id)
TTL event_date + INTERVAL 90 DAY DELETE  -- เก็บแค่ 90 วัน
SETTINGS index_granularity = 8192;

-- Distributed Table
CREATE TABLE analytics.events_dist ON CLUSTER chuaikan AS analytics.events
ENGINE = Distributed(chuaikan, analytics, events, rand());
```

```sql
-- Materialized View สำหรับ real-time aggregations
-- CalculateCounterทันทีเมื่อ data เข้า

-- Active users by region (updated every insert)
CREATE MATERIALIZED VIEW analytics.active_users_by_region_mv
ON CLUSTER chuaikan
ENGINE = SummingMergeTree()
PARTITION BY toDate(window_start)
ORDER BY (window_start, region)
POPULATE
AS SELECT
    toStartOfMinute(event_time) AS window_start,
    region,
    uniqState(user_id)           AS unique_users_state,
    count()                      AS events_count
FROM analytics.events
GROUP BY window_start, region;
```

### Step 1032: Kafka → ClickHouse Streaming

```sql
-- ClickHouse Kafka Engine — อ่าน messages จาก Kafka โดยตรง!

CREATE TABLE analytics.events_kafka_queue
(
    raw_data String
)
ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'kafka.kafka:9092',
    kafka_topic_list = 'app-events',
    kafka_group_name = 'clickhouse-consumer',
    kafka_format = 'JSONAsString',
    kafka_num_consumers = 4,
    kafka_max_block_size = 1000,
    kafka_poll_timeout_ms = 500;

-- Materialized View ที่แปลง raw Kafka message เป็น structured data
CREATE MATERIALIZED VIEW analytics.events_kafka_mv TO analytics.events_dist AS
SELECT
    toUUID(JSONExtractString(raw_data, 'event_id'))   AS event_id,
    JSONExtractString(raw_data, 'event_type')          AS event_type,
    JSONExtractUInt(raw_data, 'user_id')               AS user_id,
    JSONExtractString(raw_data, 'session_id')          AS session_id,
    JSONExtractString(raw_data, 'region')              AS region,
    JSONExtractString(raw_data, 'province')            AS province,
    JSONExtractFloat(raw_data, 'lat')                  AS lat,
    JSONExtractFloat(raw_data, 'lng')                  AS lng,
    JSONExtractUInt(raw_data, 'sos_id')                AS sos_id,
    JSONExtractString(raw_data, 'sos_severity')        AS sos_severity,
    toDateTime(JSONExtractString(raw_data, 'event_time')) AS event_time,
    JSONExtractString(raw_data, 'platform')            AS platform,
    JSONExtractString(raw_data, 'app_version')         AS app_version,
    JSONExtractUInt(raw_data, 'processing_time_ms')    AS processing_time_ms
FROM analytics.events_kafka_queue;
```

### Step 1033: Real-time Queries

```sql
-- Query 1: DAU (Daily Active Users) Real-time
SELECT
    toDate(event_time) AS date,
    uniq(user_id)      AS dau
FROM analytics.events_dist
WHERE event_date >= today() - 7
GROUP BY date
ORDER BY date;

-- Query 2: Events per second (เพื่อดู spike)
SELECT
    toStartOfSecond(event_time) AS second,
    count()                      AS events_per_second
FROM analytics.events_dist
WHERE event_time >= now() - INTERVAL 60 SECOND
GROUP BY second
ORDER BY second;

-- Query 3: SOS Hotspots (สำหรับ emergency dashboard)
SELECT
    province,
    count()                    AS sos_count,
    countIf(sos_severity='critical') AS critical_count,
    min(event_time)            AS first_sos,
    max(event_time)            AS last_sos
FROM analytics.events_dist
WHERE event_type = 'sos'
  AND event_time >= now() - INTERVAL 6 HOUR
GROUP BY province
ORDER BY sos_count DESC
LIMIT 20;

-- Query 4: User Spike Detection (>2x normal)
SELECT
    toStartOfMinute(event_time) AS minute,
    count()                      AS events,
    -- ค่าเฉลี่ย 7 วันที่ผ่านมาใน timeframe เดียวกัน
    avg(count()) OVER (
        ORDER BY minute
        ROWS BETWEEN 10080 PRECEDING AND 1 PRECEDING  -- 7 days * 60 * 24
    ) AS expected_events,
    count() / avg(count()) OVER (
        ORDER BY minute
        ROWS BETWEEN 10080 PRECEDING AND 1 PRECEDING
    ) AS spike_ratio
FROM analytics.events_dist
WHERE event_time >= now() - INTERVAL 2 HOUR
GROUP BY minute
ORDER BY minute DESC
HAVING spike_ratio > 2.0;  -- Spike = >2x normal
```

### Step 1034: Grafana Dashboard Configuration

```json
{
  "dashboard": {
    "title": "chuaikan.com Operations Dashboard",
    "tags": ["operations", "realtime"],
    "refresh": "5s",
    
    "panels": [
      {
        "id": 1,
        "title": "Live DAU (Today)",
        "type": "stat",
        "gridPos": {"h": 4, "w": 4, "x": 0, "y": 0},
        "targets": [{
          "datasource": "ClickHouse",
          "rawSql": "SELECT uniq(user_id) FROM analytics.events_dist WHERE event_date = today()",
          "format": "table"
        }],
        "options": {
          "colorMode": "background",
          "thresholds": {
            "steps": [
              {"color": "green", "value": null},
              {"color": "yellow", "value": 800000},
              {"color": "red", "value": 950000}
            ]
          }
        }
      },
      
      {
        "id": 2,
        "title": "Events/Second (Live)",
        "type": "timeseries",
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 4},
        "targets": [{
          "datasource": "ClickHouse",
          "rawSql": "SELECT toStartOfSecond(event_time) AS t, count() AS v FROM analytics.events_dist WHERE event_time >= now() - INTERVAL 5 MINUTE GROUP BY t ORDER BY t",
          "format": "time_series"
        }]
      },
      
      {
        "id": 3,
        "title": "Active SOS Alerts Map",
        "type": "geomap",
        "gridPos": {"h": 12, "w": 12, "x": 12, "y": 0},
        "targets": [{
          "datasource": "ClickHouse",
          "rawSql": "SELECT lat, lng, sos_severity, count() AS count FROM analytics.events_dist WHERE event_type = 'sos' AND event_time >= now() - INTERVAL 6 HOUR GROUP BY lat, lng, sos_severity"
        }],
        "options": {
          "layers": [{
            "type": "heatmap",
            "config": {
              "weight": { "field": "count" }
            }
          }]
        }
      }
    ]
  }
}
```

### Step 1035: SOS Cluster Detection Alert

```python
# sos-cluster-detector.py
# ตรวจจับ SOS alerts ที่กระจุกตัวในพื้นที่เดียวกัน = disaster!

from sklearn.cluster import DBSCAN
import numpy as np
import clickhouse_driver
import json
from datetime import datetime, timedelta

class SOSClusterDetector:
    def __init__(self):
        self.ch_client = clickhouse_driver.Client(
            host='clickhouse.clickhouse',
            user='analytics',
            password=os.environ['CH_PASSWORD'],
            database='analytics'
        )
    
    def detect_clusters(self, time_window_hours=1, min_cluster_size=5):
        """ตรวจจับ SOS cluster ที่อาจเป็น disaster event"""
        
        # ดึง SOS locations ใน time window
        query = """
        SELECT lat, lng, sos_severity, user_id, event_time
        FROM analytics.events_dist
        WHERE event_type = 'sos'
          AND event_time >= now() - INTERVAL %s HOUR
          AND lat != 0 AND lng != 0
        """
        
        rows = self.ch_client.execute(query, (time_window_hours,))
        
        if len(rows) < min_cluster_size:
            return []
        
        # Convert to numpy array
        coords = np.array([[row[0], row[1]] for row in rows])
        
        # DBSCAN clustering
        # eps=0.05 ≈ 5km (1 degree lat/lng ≈ 111km)
        dbscan = DBSCAN(eps=0.05, min_samples=min_cluster_size, algorithm='ball_tree', metric='haversine')
        labels = dbscan.fit_predict(np.radians(coords))
        
        clusters = []
        for cluster_id in set(labels):
            if cluster_id == -1:  # Noise points
                continue
            
            cluster_mask = labels == cluster_id
            cluster_points = [rows[i] for i in range(len(rows)) if cluster_mask[i]]
            cluster_coords = coords[cluster_mask]
            
            # คำนวณ cluster stats
            center_lat = np.mean(cluster_coords[:, 0])
            center_lng = np.mean(cluster_coords[:, 1])
            
            severities = [row[2] for row in cluster_points]
            critical_count = sum(1 for s in severities if s == 'critical')
            
            clusters.append({
                'cluster_id': int(cluster_id),
                'center_lat': float(center_lat),
                'center_lng': float(center_lng),
                'sos_count': len(cluster_points),
                'critical_count': critical_count,
                'severity': 'critical' if critical_count > 0 else 'high',
                'first_sos': min(row[4] for row in cluster_points).isoformat(),
                'user_count': len(set(row[3] for row in cluster_points)),
            })
        
        return clusters
    
    def notify_if_new_cluster(self, clusters):
        """แจ้งเตือนถ้าพบ cluster ใหม่"""
        for cluster in clusters:
            cache_key = f"sos_cluster:{cluster['cluster_id']}"
            
            if not redis.exists(cache_key):
                # Cluster ใหม่! แจ้งเตือน
                self.send_cluster_alert(cluster)
                redis.setex(cache_key, 3600, json.dumps(cluster))  # cache 1 hour
    
    def send_cluster_alert(self, cluster):
        """ส่ง alert ไปทีม emergency response"""
        message = {
            'alert_type': 'sos_cluster_detected',
            'severity': cluster['severity'],
            'location': {
                'lat': cluster['center_lat'],
                'lng': cluster['center_lng'],
            },
            'stats': {
                'sos_count': cluster['sos_count'],
                'critical_count': cluster['critical_count'],
                'affected_users': cluster['user_count'],
            },
            'timestamp': datetime.utcnow().isoformat(),
        }
        
        # ส่งไป PagerDuty
        requests.post(
            'https://events.pagerduty.com/v2/enqueue',
            json={
                'routing_key': os.environ['PAGERDUTY_KEY'],
                'event_action': 'trigger',
                'payload': {
                    'summary': f"SOS Cluster Detected: {cluster['sos_count']} alerts near {cluster['center_lat']:.4f},{cluster['center_lng']:.4f}",
                    'severity': cluster['severity'],
                    'custom_details': message,
                }
            }
        )
```

---

## 🔧 Configuration Files

### ClickHouse Backup

```bash
#!/bin/bash
# clickhouse-backup.sh

DATE=$(date +%Y%m%d)
BACKUP_DIR="s3://chuaikan-backups/clickhouse/$DATE"

# ใช้ clickhouse-backup tool
clickhouse-backup create --config /etc/clickhouse-backup/config.yaml "backup-$DATE"
clickhouse-backup upload --config /etc/clickhouse-backup/config.yaml "backup-$DATE"

echo "Backup completed: $BACKUP_DIR"
```

---

## 🧪 Testing

### Dashboard Performance Test

```python
# test-analytics-queries.py

def test_dau_query_performance():
    """DAU query ต้องเร็วกว่า 100ms"""
    import time
    
    start = time.time()
    result = ch_client.execute(
        "SELECT uniq(user_id) FROM analytics.events_dist WHERE event_date = today()"
    )
    duration = (time.time() - start) * 1000
    
    assert duration < 100, f"DAU query too slow: {duration:.0f}ms"
    print(f"DAU query: {duration:.0f}ms ✅")

def test_sos_cluster_detection():
    """Cluster detection ต้องทำงานได้"""
    detector = SOSClusterDetector()
    clusters = detector.detect_clusters(time_window_hours=24)
    
    # ถ้ามี data ต้องได้ list (อาจ empty)
    assert isinstance(clusters, list)
    print(f"Detected {len(clusters)} SOS clusters")
```

---

## ❌ Common Errors & Solutions

### Error 1: ClickHouse Disk Full

```bash
# ตรวจ disk usage
SELECT
    database,
    table,
    formatReadableSize(sum(bytes)) AS size,
    sum(rows) AS rows
FROM system.parts
WHERE active
GROUP BY database, table
ORDER BY sum(bytes) DESC;

# ถ้า TTL ไม่ทำงาน → Force TTL execution
ALTER TABLE analytics.events ON CLUSTER chuaikan
    MATERIALIZE TTL;
```

### Error 2: Kafka Consumer Lag

```bash
# ตรวจ consumer group lag
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --describe --group clickhouse-consumer

# ถ้า lag สูง → เพิ่ม consumer หรือ batch size
ALTER TABLE analytics.events_kafka_queue
    MODIFY SETTING kafka_num_consumers = 8;
```

---

## ✅ Checklist

- [ ] ClickHouse cluster ติดตั้งแล้ว (2 shards × 2 replicas)
- [ ] Schema สร้างแล้วพร้อม TTL policy
- [ ] Kafka → ClickHouse pipeline ทำงาน
- [ ] Grafana Dashboard แสดง DAU, Events/sec, SOS map
- [ ] Dashboard refresh rate: 5 วินาที
- [ ] SOS Cluster Detection algorithm ทำงาน
- [ ] Emergency Responder Dashboard แยกต่างหาก
- [ ] Query performance: DAU < 100ms, SOS hotspot < 500ms
- [ ] ClickHouse backup daily ไป S3
- [ ] Anomaly detection alerts ตั้งค่าแล้ว

---

## 🔗 References

- [ClickHouse Documentation](https://clickhouse.com/docs)
- [ClickHouse + Kafka Integration](https://clickhouse.com/docs/en/integrations/kafka)
- [Grafana ClickHouse Plugin](https://grafana.com/grafana/plugins/grafana-clickhouse-datasource/)
- [DBSCAN Algorithm](https://scikit-learn.org/stable/modules/clustering.html#dbscan)
- [ClickHouse Operator](https://github.com/Altinity/clickhouse-operator)

---

*Part 104 | Road to 1,000,000 Users/Day | chuaikan.com*
