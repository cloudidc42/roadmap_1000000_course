# Part 068: Chaos Engineering

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Advanced
> **Steps:** 671-680
> **เวลาโดยประมาณ:** 5 ชั่วโมง
> **Prerequisites:** Part 067 (Load Testing)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- หลักการ Chaos Engineering (Netflix Chaos Monkey)
- ติดตั้ง Chaos Mesh บน Kubernetes
- Experiments: pod kill, network delay, network partition, disk failure
- ทดสอบ resilience ของ SOS service
- Database failover test
- Redis failover test
- Circuit breaker validation
- Chaos experiment runbook

---

## 📖 ทฤษฎีและแนวคิด

### Chaos Engineering คืออะไร?

> "Chaos Engineering is the discipline of experimenting on a system in order to build confidence in the system's capability to withstand turbulent conditions in production." — Netflix

หลักการ 5 ข้อของ Chaos Engineering:
1. **Define steady state** - รู้ว่า "ปกติ" คืออะไร
2. **Hypothesize steady state continues** - คาดว่าระบบจะ maintain steady state
3. **Introduce real-world events** - inject failures
4. **Observe system behavior** - ดูว่าระบบตอบสนองอย่างไร
5. **Automate experiments** - รัน chaos experiments อัตโนมัติ

### Chaos Mesh Architecture

```
┌───────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                   │
│                                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │              Chaos Mesh Control Plane             │  │
│  │  ┌──────────────┐  ┌──────────────────────────┐  │  │
│  │  │ Chaos Daemon │  │  Chaos Controller Manager  │  │  │
│  │  │  (DaemonSet) │  │  (Deploy in chaos-mesh ns) │  │  │
│  │  └──────────────┘  └──────────────────────────┘  │  │
│  └──────────────────────────────────────────────────┘  │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌────────────────────┐   │
│  │ API Pods │  │  DB Pods │  │  Redis Pods         │   │
│  │ (target) │  │ (target) │  │  (target)           │   │
│  └──────────┘  └──────────┘  └────────────────────┘   │
└───────────────────────────────────────────────────────┘
```

---

## ⚙️ Environment Setup

```bash
# ติดตั้ง Chaos Mesh บน Kubernetes
# ใช้ Helm (แนะนำ)
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm repo update

# ติดตั้งใน namespace chaos-mesh
kubectl create namespace chaos-mesh

helm install chaos-mesh chaos-mesh/chaos-mesh \
  --namespace chaos-mesh \
  --version 2.6.3 \
  --set chaosDaemon.runtime=containerd \
  --set chaosDaemon.socketPath=/run/containerd/containerd.sock \
  --set dashboard.securityMode=false  # ปิด auth สำหรับ development

# ตรวจสอบการติดตั้ง
kubectl get pods -n chaos-mesh

# เปิด Chaos Mesh Dashboard
kubectl port-forward svc/chaos-dashboard 2333:2333 -n chaos-mesh
# เปิด browser: http://localhost:2333
```

---

## 🛠️ Step-by-Step Implementation

### Step 671: Define Steady State

ก่อนทำ chaos experiment ต้องรู้ว่า "ปกติ" คืออะไรก่อน

```bash
# baseline-metrics.sh - วัด steady state
#!/bin/bash

echo "=== Steady State Baseline ==="
echo "Time: $(date)"

# 1. ตรวจสอบ pod health
echo ""
echo "Pod Status:"
kubectl get pods -n production -l app=chuaikan-api

# 2. Response time baseline
echo ""
echo "API Response Time (30 seconds):"
autocannon -d 30 -c 10 \
  --json https://api.chuaikan.com/health | \
  node -e "
    const d = JSON.parse(require('fs').readFileSync('/dev/stdin','utf8'));
    console.log('RPS:', d.requests.average);
    console.log('P50:', d.latency.p50, 'ms');
    console.log('P95:', d.latency.p95, 'ms');
    console.log('P99:', d.latency.p99, 'ms');
    console.log('Errors:', d.errors);
  "

# 3. Database connections
echo ""
echo "Database Connections:"
psql $DATABASE_URL -c "
  SELECT state, count(*) FROM pg_stat_activity 
  WHERE datname = 'chuaikan_db' 
  GROUP BY state;
"

# 4. Redis info
echo ""
echo "Redis Connections:"
redis-cli info clients | grep connected_clients

# 5. Save baseline
kubectl top pods -n production > /tmp/baseline-resources.txt
echo "Baseline saved to /tmp/baseline-resources.txt"
```

### Step 672: Pod Kill Experiment

```yaml
# chaos/pod-kill.yaml - Kill random API pods
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: api-pod-kill
  namespace: chaos-mesh
spec:
  action: pod-kill
  mode: one          # kill หนึ่ง pod ต่อครั้ง
  selector:
    namespaces:
      - production
    labelSelectors:
      app: chuaikan-api
  duration: "0"      # instant kill
  scheduler:
    cron: "@every 5m"  # ทุก 5 นาที (ใน experiment phase)
```

```bash
# Apply experiment
kubectl apply -f chaos/pod-kill.yaml

# Monitor ผลกระทบ
watch -n 2 kubectl get pods -n production -l app=chuaikan-api

# ตรวจสอบว่า traffic ไม่ drop
autocannon -d 60 -c 20 https://api.chuaikan.com/health

# ดู pod restart count
kubectl get pods -n production -l app=chuaikan-api \
  -o custom-columns='NAME:.metadata.name,RESTARTS:.status.containerStatuses[0].restartCount'

# หยุด experiment
kubectl delete -f chaos/pod-kill.yaml
```

### Step 673: Network Delay Experiment

```yaml
# chaos/network-delay.yaml - Inject network delay
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: api-network-delay
  namespace: chaos-mesh
spec:
  action: delay
  mode: all
  selector:
    namespaces:
      - production
    labelSelectors:
      app: chuaikan-api
  delay:
    latency: "200ms"    # เพิ่ม 200ms latency
    correlation: "25"   # 25% correlation (not completely random)
    jitter: "50ms"      # ± 50ms variation
  duration: "5m"        # รัน 5 นาที
  direction: to         # delay inbound traffic
```

```yaml
# chaos/network-delay-db.yaml - Simulate slow database connection
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: db-network-delay
  namespace: chaos-mesh
spec:
  action: delay
  mode: all
  selector:
    namespaces:
      - production
    labelSelectors:
      app: postgresql
  delay:
    latency: "100ms"
    jitter: "20ms"
  duration: "5m"
  direction: both
```

```bash
# Apply network delay
kubectl apply -f chaos/network-delay.yaml

# Monitor latency impact
while true; do
  LATENCY=$(curl -s -w "%{time_total}" -o /dev/null https://api.chuaikan.com/health)
  echo "$(date '+%H:%M:%S') API latency: ${LATENCY}s"
  sleep 5
done

# ตรวจสอบ circuit breaker triggered
kubectl logs -n production -l app=chuaikan-api | grep -i "circuit\|timeout\|retry"

# หยุด
kubectl delete -f chaos/network-delay.yaml
```

### Step 674: Network Partition Experiment

```yaml
# chaos/network-partition.yaml - Simulate network split
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: api-db-partition
  namespace: chaos-mesh
spec:
  action: partition
  mode: all
  selector:
    namespaces:
      - production
    labelSelectors:
      app: chuaikan-api
  direction: both
  target:
    mode: all
    selector:
      namespaces:
        - production
      labelSelectors:
        app: postgresql  # block traffic ระหว่าง API และ Database
  duration: "2m"         # 2 นาที
```

```bash
# Apply partition
kubectl apply -f chaos/network-partition.yaml

# ตรวจสอบว่า app handle database error อย่างไร
# ควร: return cached data หรือ 503 gracefully (ไม่ควร crash!)
curl -v https://api.chuaikan.com/api/posts
# Expected: 503 Service Unavailable with proper error message
# NOT: connection refused, 500 unhandled error

# ดู error logs
kubectl logs -n production -l app=chuaikan-api --since=2m | \
  grep -E "error|ERR|WARN"

# หยุด
kubectl delete -f chaos/network-partition.yaml
```

### Step 675: SOS Service Resilience Test

```javascript
// tests/sos-resilience-test.js
// ทดสอบว่า SOS service ทำงานได้แม้ notification service ล่ม

import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  vus: 50,
  duration: '10m',
  thresholds: {
    // SOS endpoint ต้อง work เสมอ แม้มี failures
    'http_req_duration{endpoint:sos}': ['p(99)<2000'],  // ยอมรับ P99 สูงกว่าปกติ
    'checks{endpoint:sos}': ['rate>0.99'],  // 99% success rate
  },
};

export default function() {
  const BASE_URL = 'https://api.chuaikan.com';
  
  // Test 1: Create SOS alert (critical path)
  const sosRes = http.post(
    `${BASE_URL}/api/sos`,
    JSON.stringify({
      type: 'emergency',
      location: { lat: 13.7563, lng: 100.5018 },
      message: 'Load test SOS',
      priority: 'high',
    }),
    {
      headers: {
        'Content-Type': 'application/json',
        'Authorization': 'Bearer test-token',
      },
      tags: { endpoint: 'sos' },
    }
  );
  
  check(sosRes, {
    'SOS created (even during failures)': (r) => r.status === 201 || r.status === 202,
    'SOS has alert ID': (r) => r.json('data.alertId') !== undefined,
  });
  
  // Test 2: Check alert status (should be persisted to DB)
  if (sosRes.status === 201) {
    const alertId = sosRes.json('data.alertId');
    const statusRes = http.get(
      `${BASE_URL}/api/sos/${alertId}`,
      {
        headers: { 'Authorization': 'Bearer test-token' },
        tags: { endpoint: 'sos_status' },
      }
    );
    
    check(statusRes, {
      'SOS alert persisted': (r) => r.status === 200,
      'SOS alert has correct status': (r) => 
        ['pending', 'processing', 'sent'].includes(r.json('data.status')),
    });
  }
  
  sleep(2);
}
```

```yaml
# chaos/notification-service-kill.yaml - Kill notification service ขณะ SOS active
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: kill-notification-service
  namespace: chaos-mesh
spec:
  action: pod-kill
  mode: all  # kill ALL notification pods
  selector:
    namespaces:
      - production
    labelSelectors:
      app: notification-service
  duration: "5m"
```

```bash
# Chaos test: SOS ยังทำงานได้แม้ notification service ล่ม
# Step 1: เริ่ม SOS load test
k6 run tests/sos-resilience-test.js &
K6_PID=$!

# Step 2: Inject failure หลัง 2 นาที
sleep 120
kubectl apply -f chaos/notification-service-kill.yaml

# Step 3: Monitor
echo "SOS test running... notification service is DOWN"
sleep 300

# Step 4: Restore
kubectl delete -f chaos/notification-service-kill.yaml
wait $K6_PID
echo "Test complete. Check if SOS alerts were saved to database."
```

### Step 676: Database Failover Test

```yaml
# chaos/postgresql-kill.yaml - Test database failover
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: postgresql-primary-kill
  namespace: chaos-mesh
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces:
      - production
    labelSelectors:
      app: postgresql
      role: primary  # kill primary หรือ master pod
  duration: "0"
```

```bash
# Database failover test procedure
echo "=== Database Failover Test ==="

# Step 1: Check current primary
PGPRIMARY=$(kubectl get pods -n production -l app=postgresql,role=primary --no-headers -o name)
echo "Current primary: $PGPRIMARY"

# Step 2: เริ่ม write test
cat << 'EOF' > /tmp/db-write-test.sh
#!/bin/bash
for i in $(seq 1 1000); do
  curl -s -X POST https://api.chuaikan.com/api/posts \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer test-token" \
    -d '{"content":"failover test post","type":"text"}' \
    | jq -r '.data.id // "FAILED"' &
  sleep 0.1
done
wait
EOF

bash /tmp/db-write-test.sh > /tmp/write-results.txt &

# Step 3: Kill primary (หลัง 30 วินาที)
sleep 30
echo "Killing primary database pod..."
kubectl apply -f chaos/postgresql-kill.yaml

# Step 4: Monitor failover
echo "Waiting for failover..."
sleep 30

# ตรวจสอบว่ามี primary ใหม่ขึ้นมา
kubectl get pods -n production -l app=postgresql
kubectl logs -n production -l app=postgresql | grep "promoted to primary\|failover"

# Step 5: Check data integrity
echo "Checking for missing writes..."
EXPECTED=1000
ACTUAL=$(psql $DATABASE_URL -t -c "SELECT count(*) FROM posts WHERE content='failover test post' AND created_at > NOW() - INTERVAL '5 minutes'")
echo "Expected writes: $EXPECTED"
echo "Actual in DB: $ACTUAL"

# Step 6: Measure RTO (Recovery Time Objective)
echo "Failover test complete."
```

### Step 677: Redis Failover Test

```yaml
# chaos/redis-kill.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: redis-master-kill
  namespace: chaos-mesh
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces:
      - production
    labelSelectors:
      app: redis
      role: master
  duration: "0"
```

```javascript
// tests/redis-failover.js - ทดสอบว่า app ทำงานได้เมื่อ Redis ล่ม
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },
    { duration: '5m', target: 100 },  // Redis จะถูก kill ใน phase นี้
    { duration: '2m', target: 0 },
  ],
  thresholds: {
    // เมื่อ Redis ล่ม: ยอมรับ latency สูงขึ้นแต่ error ต้องไม่เกิน 5%
    http_req_duration: ['p(95)<2000'],  // ผ่อนปรนจาก 500ms เป็น 2000ms
    http_req_failed: ['rate<0.05'],     // ยอมรับ 5% (แทนที่จะเป็น 1%)
  },
};

export default function() {
  const BASE_URL = 'https://api.chuaikan.com';
  
  // Test: API ควรยัง work แม้ cache ไม่มี (fallback to DB)
  const res = http.get(`${BASE_URL}/api/posts?page=1`, {
    headers: { 'Authorization': 'Bearer test-token' },
  });
  
  check(res, {
    'API works without Redis': (r) => r.status === 200,
    'Response has data': (r) => Array.isArray(r.json('data.posts')),
  });
  
  // Session validation test (session stored in Redis)
  const authRes = http.get(`${BASE_URL}/api/me`, {
    headers: { 'Authorization': 'Bearer test-token' },
  });
  
  // เมื่อ Redis ล่ม session validation อาจ fail
  // แต่ต้องเป็น graceful fail (401 ไม่ใช่ 500)
  check(authRes, {
    'Auth fail gracefully': (r) => [200, 401].includes(r.status),
    'No 500 errors': (r) => r.status !== 500,
  });
  
  sleep(1);
}
```

### Step 678: Circuit Breaker Validation

```javascript
// circuit-breaker.js - Implement และ test circuit breaker
const CircuitBreaker = require('opossum');

// สร้าง circuit breaker สำหรับ external service calls
function createCircuitBreaker(fn, options = {}) {
  const breaker = new CircuitBreaker(fn, {
    timeout: 3000,              // request timeout 3s
    errorThresholdPercentage: 50, // open circuit ถ้า error > 50%
    resetTimeout: 30000,        // ลอง close circuit หลัง 30s
    rollingCountTimeout: 10000, // window สำหรับนับ errors
    rollingCountBuckets: 10,    // แบ่งเป็น 10 buckets
    name: options.name || 'unknown',
    volumeThreshold: 5,         // ต้องมี min 5 requests ก่อน circuit เปิด
    ...options,
  });
  
  // Events
  breaker.on('open', () => {
    console.warn(`Circuit OPEN: ${breaker.name}`);
    // Alert ทาง Slack/PagerDuty
  });
  
  breaker.on('halfOpen', () => {
    console.info(`Circuit HALF-OPEN: ${breaker.name} - probing...`);
  });
  
  breaker.on('close', () => {
    console.info(`Circuit CLOSED: ${breaker.name} - service recovered`);
  });
  
  breaker.on('fallback', (result) => {
    console.info(`Circuit FALLBACK: ${breaker.name}`, result);
  });
  
  // Metrics for Prometheus
  const client = require('prom-client');
  const circuitStateGauge = new client.Gauge({
    name: 'circuit_breaker_state',
    help: 'Circuit breaker state (0=closed, 1=open, 2=half-open)',
    labelNames: ['name'],
  });
  
  breaker.on('open', () => circuitStateGauge.set({ name: breaker.name }, 1));
  breaker.on('close', () => circuitStateGauge.set({ name: breaker.name }, 0));
  breaker.on('halfOpen', () => circuitStateGauge.set({ name: breaker.name }, 2));
  
  return breaker;
}

// ใช้ circuit breaker กับ notification service
async function sendNotificationRaw(userId, message) {
  const res = await fetch(`${process.env.NOTIFICATION_SERVICE_URL}/send`, {
    method: 'POST',
    body: JSON.stringify({ userId, message }),
    headers: { 'Content-Type': 'application/json' },
    signal: AbortSignal.timeout(3000),
  });
  
  if (!res.ok) throw new Error(`Notification failed: ${res.status}`);
  return res.json();
}

const notificationBreaker = createCircuitBreaker(sendNotificationRaw, {
  name: 'notification-service',
});

// Fallback: queue notification สำหรับ retry ทีหลัง
notificationBreaker.fallback(async (userId, message) => {
  await queueNotificationForRetry(userId, message);
  return { queued: true };
});

async function sendNotification(userId, message) {
  return notificationBreaker.fire(userId, message);
}
```

```yaml
# chaos/circuit-breaker-test.yaml - Test circuit breaker behavior
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: notification-service-errors
  namespace: chaos-mesh
spec:
  action: loss  # packet loss ทำให้ connection timeout
  mode: all
  selector:
    namespaces:
      - production
    labelSelectors:
      app: notification-service
  loss:
    loss: "80"   # 80% packet loss
    correlation: "25"
  duration: "3m"
```

### Step 679: Disk Failure Experiment

```yaml
# chaos/disk-stress.yaml - Stress disk I/O
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: disk-io-stress
  namespace: chaos-mesh
spec:
  mode: one
  selector:
    namespaces:
      - production
    labelSelectors:
      app: postgresql
  stressors:
    io:
      workers: 4         # 4 workers เขียน I/O
      size: "10GB"       # เขียนข้อมูลทั้งหมด 10GB
      path: "/var/lib/postgresql/data"
  duration: "5m"
```

```yaml
# chaos/cpu-stress.yaml - CPU pressure test
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: api-cpu-stress
  namespace: chaos-mesh
spec:
  mode: all
  selector:
    namespaces:
      - production
    labelSelectors:
      app: chuaikan-api
  stressors:
    cpu:
      workers: 4   # 4 CPU workers ที่ load 100%
      load: 80     # 80% CPU utilization
  duration: "5m"
```

### Step 680: Chaos Experiment Runbook

```markdown
# Chaos Experiment Runbook: chuaikan.com

## Pre-Experiment Checklist

- [ ] แจ้ง team ก่อนรัน experiment
- [ ] ตรวจสอบ monitoring dashboards ทำงานปกติ
- [ ] มี rollback plan พร้อม
- [ ] กำหนด steady state metrics
- [ ] ตรวจสอบ on-call engineer พร้อม
- [ ] เลือก experiment ที่ minimum blast radius ก่อน

## During Experiment

- [ ] Monitor Grafana dashboard ตลอด
- [ ] Monitor k6/autocannon load test
- [ ] บันทึก observations
- [ ] พร้อม abort ถ้า user impact เกิน threshold

## Abort Criteria

หยุด experiment ทันทีถ้า:
- Error rate > 5%
- P99 latency > 5 seconds
- SOS alerts ไม่สามารถสร้างได้
- Data loss detected
- On-call ต้องการ abort
```

```bash
#!/bin/bash
# run-chaos-experiment.sh - Automated chaos runbook

EXPERIMENT_NAME="${1:-pod-kill}"
DURATION="${2:-5m}"
MONITOR_INTERVAL=15

echo "=== Chaos Experiment: $EXPERIMENT_NAME ==="
echo "Duration: $DURATION"
echo ""

# Pre-flight check
echo "1. Pre-flight check..."
kubectl get pods -n production -l app=chuaikan-api --no-headers | grep -v Running && \
  echo "ERROR: Some pods not running!" && exit 1
echo "   All pods healthy ✓"

# Get baseline
BASELINE_RPS=$(autocannon -d 10 -c 10 --json https://api.chuaikan.com/health | \
  python3 -c "import sys,json; d=json.load(sys.stdin); print(d['requests']['average'])")
echo "   Baseline RPS: $BASELINE_RPS ✓"

# Start load test
echo ""
echo "2. Starting load test..."
autocannon -d $(($(echo $DURATION | sed 's/m//')*60+60)) -c 50 \
  --json https://api.chuaikan.com/health > /tmp/chaos-load-results.json &
AUTOCANNON_PID=$!

sleep 5  # wait for load test to warm up

# Apply chaos
echo ""
echo "3. Applying chaos: $EXPERIMENT_NAME..."
kubectl apply -f chaos/${EXPERIMENT_NAME}.yaml
CHAOS_START=$(date +%s)

# Monitor
echo ""
echo "4. Monitoring (${MONITOR_INTERVAL}s intervals)..."
ABORT=false

while [ $(($(date +%s) - CHAOS_START)) -lt $(($(echo $DURATION | sed 's/m//')*60)) ]; do
  CURRENT_RPS=$(kubectl top pods -n production -l app=chuaikan-api 2>/dev/null | \
    awk 'NR>1{sum+=$3} END{print sum}')
  
  ERROR_RATE=$(kubectl logs --since=1m -n production -l app=chuaikan-api | \
    grep -c "ERROR\|500\|503" || true)
  
  echo "$(date '+%H:%M:%S') | RPS: $CURRENT_RPS | Errors (1m): $ERROR_RATE"
  
  # Abort criteria
  if [ "$ERROR_RATE" -gt 50 ]; then
    echo "⚠️  HIGH ERROR RATE: $ERROR_RATE errors in last minute!"
    ABORT=true
    break
  fi
  
  sleep $MONITOR_INTERVAL
done

# Cleanup
echo ""
echo "5. Cleaning up..."
kubectl delete -f chaos/${EXPERIMENT_NAME}.yaml
kill $AUTOCANNON_PID 2>/dev/null

if [ "$ABORT" = true ]; then
  echo "❌ Experiment ABORTED due to high error rate"
  exit 1
else
  echo "✅ Experiment completed successfully"
fi
```

---

## 🔧 Configuration Files

```yaml
# chaos/schedule-weekly.yaml - Weekly automated chaos test
apiVersion: chaos-mesh.org/v1alpha1
kind: Schedule
metadata:
  name: weekly-chaos
  namespace: chaos-mesh
spec:
  schedule: "@weekly"  # ทุกอาทิตย์
  type: PodChaos
  historyLimit: 5
  podChaos:
    action: pod-kill
    mode: one
    selector:
      namespaces:
        - production
      labelSelectors:
        app: chuaikan-api
```

---

## 🧪 Testing

```bash
# Quick smoke test: pod kill
kubectl apply -f chaos/pod-kill.yaml
sleep 30
kubectl get pods -n production -l app=chuaikan-api
curl -f https://api.chuaikan.com/health || echo "FAIL: API down!"
kubectl delete -f chaos/pod-kill.yaml

# Full test suite
for experiment in pod-kill network-delay network-partition; do
  echo "Running: $experiment"
  bash run-chaos-experiment.sh $experiment 2m
  echo "Waiting 2 minutes before next experiment..."
  sleep 120
done
```

---

## ❌ Common Errors & Solutions

### Error 1: Chaos Mesh daemon not ready

```bash
# ตรวจสอบ Chaos Mesh status
kubectl get pods -n chaos-mesh

# ถ้า chaos-daemon ไม่ running ให้ตรวจ socket path
kubectl describe pod -n chaos-mesh -l app.kubernetes.io/component=chaos-daemon
# ตรวจสอบ container runtime: containerd หรือ docker?
kubectl get nodes -o wide
```

### Error 2: Experiment ไม่มีผลกระทบ

```bash
# ตรวจสอบ selector ถูกต้อง
kubectl get pods -n production -l app=chuaikan-api
# ต้องมี label app=chuaikan-api บน pods

# ตรวจสอบ RBAC
kubectl auth can-i '*' '*' --as=system:serviceaccount:chaos-mesh:chaos-controller-manager

# ดู Chaos Mesh logs
kubectl logs -n chaos-mesh -l app.kubernetes.io/component=chaos-controller-manager
```

---

## ✅ Checklist

- [ ] ติดตั้ง Chaos Mesh บน Kubernetes cluster
- [ ] กำหนด steady state metrics (RPS, P95, error rate)
- [ ] รัน pod-kill test และยืนยันว่า app ยัง available
- [ ] รัน network-delay test และยืนยัน circuit breaker ทำงาน
- [ ] รัน network-partition test และยืนยัน graceful degradation
- [ ] รัน SOS resilience test (notification service down)
- [ ] รัน database failover test
- [ ] รัน Redis failover test
- [ ] สร้าง chaos runbook สำหรับ team
- [ ] ตั้งค่า weekly automated chaos test

---

## 🔗 References

- [Chaos Engineering Book](https://www.oreilly.com/library/view/chaos-engineering/9781492043850/)
- [Chaos Mesh Documentation](https://chaos-mesh.org/docs/)
- [Netflix Chaos Engineering](https://netflixtechblog.com/chaos-engineering-upgraded-878d341f15fa)
- [opossum Circuit Breaker](https://github.com/nodeshift/opossum)
- [Principles of Chaos Engineering](https://principlesofchaos.org/)

---

*Part 068 | Road to 1,000,000 Users/Day | chuaikan.com*
