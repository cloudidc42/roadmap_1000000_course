# Part 067: Load Testing at Scale (100k Concurrent)

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Advanced
> **Steps:** 661-670
> **เวลาโดยประมาณ:** 6 ชั่วโมง
> **Prerequisites:** Part 061-066

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- k6 สำหรับ 100k virtual users test
- Distributed k6 ด้วย k6 operator บน Kubernetes
- Test scenarios: ramp up, sustained, spike, soak
- วิเคราะห์ P50/P95/P99 latency, error rate, throughput
- กระบวนการหา bottleneck
- Database connection pool exhaustion test
- Redis connection test
- WebSocket scale test
- Infrastructure scaling ระหว่าง load test

---

## 📖 ทฤษฎีและแนวคิด

### Load Test Types

```
1. Smoke Test: น้อย VU, verify ว่า test script ทำงานได้
   ┌─────┐
   │     │
   ──────────────── time (1-5 min)

2. Load Test: simulate expected load
   ┌─────────────┐
   /             \
   ──────────────── time (30-60 min)

3. Stress Test: เกิน capacity เพื่อหา breaking point
         ┌────┐
        /     \
   ────/       ────── time

4. Spike Test: sudden traffic increase (viral content, SOS alert)
         ┌┐
   ──────┘└──────── time

5. Soak Test: ยาวนาน (4-8 ชั่วโมง) เพื่อหา memory leaks
   ┌──────────────────┐
   │                  │
   ──────────────────── time
```

### P50/P95/P99 Percentiles

```
P50 (median): 50% ของ requests ตอบได้ภายใน X ms
P95: 95% ของ requests ตอบได้ภายใน X ms
P99: 99% ของ requests ตอบได้ภายใน X ms

ตัวอย่าง:
P50: 50ms   ← most users experience this
P95: 200ms  ← 5% ของ users experience this or worse
P99: 500ms  ← 1% ของ users experience this or worse

target: P95 < 200ms สำหรับ API endpoints
```

---

## ⚙️ Environment Setup

```bash
# ติดตั้ง k6
sudo gpg -k
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update
sudo apt-get install k6

# ตรวจสอบ version
k6 version

# ติดตั้ง k6 extensions (optional)
# xk6-browser, xk6-sql, xk6-redis
# https://k6.io/docs/extensions/

# สร้าง project directory
mkdir -p /home/user/k6-tests
cd /home/user/k6-tests
```

---

## 🛠️ Step-by-Step Implementation

### Step 661: k6 Basics และ First Test

```javascript
// tests/smoke-test.js - เริ่มต้นด้วย smoke test
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend, Counter } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');
const apiDuration = new Trend('api_duration');
const totalRequests = new Counter('total_requests');

// Test configuration
export const options = {
  // Smoke test: 1 user, 1 minute
  vus: 1,
  duration: '1m',
  
  // Thresholds สำหรับ pass/fail
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],  // P95 < 500ms
    http_req_failed: ['rate<0.01'],                   // Error rate < 1%
    errors: ['rate<0.05'],
  },
};

// Setup: รัน once ก่อน test
export function setup() {
  // Login หรือสร้าง test data
  const loginRes = http.post('https://api.chuaikan.com/auth/login', JSON.stringify({
    email: 'test@chuaikan.com',
    password: 'test-password',
  }), {
    headers: { 'Content-Type': 'application/json' },
  });
  
  const token = loginRes.json('data.token');
  return { token };  // pass ไปให้ default function
}

// Default function: รันซ้ำสำหรับทุก VU iteration
export default function(data) {
  const { token } = data;
  const headers = {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json',
  };
  
  const BASE_URL = 'https://api.chuaikan.com';
  
  // Test 1: Homepage / Health check
  const healthRes = http.get(`${BASE_URL}/health`, { headers });
  check(healthRes, {
    'health status 200': (r) => r.status === 200,
    'health response time < 100ms': (r) => r.timings.duration < 100,
  });
  
  // Test 2: Get posts feed
  const feedRes = http.get(`${BASE_URL}/api/posts?page=1&limit=20`, { headers });
  check(feedRes, {
    'feed status 200': (r) => r.status === 200,
    'feed has posts': (r) => r.json('data.posts.length') > 0,
    'feed response time < 300ms': (r) => r.timings.duration < 300,
  });
  
  // Track metrics
  errorRate.add(feedRes.status !== 200);
  apiDuration.add(feedRes.timings.duration);
  totalRequests.add(1);
  
  // Test 3: Get user profile
  const profileRes = http.get(`${BASE_URL}/api/me`, { headers });
  check(profileRes, {
    'profile status 200': (r) => r.status === 200,
  });
  
  sleep(1);  // think time ระหว่าง requests
}

// Teardown: รัน once หลัง test เสร็จ
export function teardown(data) {
  console.log('Test completed. Token used:', data.token ? 'yes' : 'no');
}
```

```bash
# รัน smoke test
k6 run tests/smoke-test.js

# รัน กับ environment variables
k6 run \
  -e BASE_URL=https://staging.chuaikan.com \
  -e API_TOKEN=your-token \
  tests/smoke-test.js

# Output to JSON สำหรับ analysis
k6 run --out json=results.json tests/smoke-test.js
```

### Step 662: Load Test Scenarios

```javascript
// tests/load-test.js - Complete load test scenarios
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { SharedArray } from 'k6/data';
import { randomItem } from 'https://jslib.k6.io/k6-utils/1.4.0/index.js';

// Load test data (อ่านครั้งเดียว share กับทุก VU)
const users = new SharedArray('users', function() {
  return JSON.parse(open('./data/test-users.json'));
});

export const options = {
  scenarios: {
    // Scenario 1: Ramp up test
    ramp_up: {
      executor: 'ramping-vus',
      stages: [
        { duration: '5m', target: 100 },   // ramp up ใน 5 นาที
        { duration: '10m', target: 100 },  // stay at 100 VUs
        { duration: '5m', target: 0 },     // ramp down
      ],
      gracefulRampDown: '30s',
    },
    
    // Scenario 2: Spike test (run หลัง ramp_up)
    spike: {
      executor: 'ramping-vus',
      startTime: '20m',  // เริ่มหลัง ramp_up scenario
      stages: [
        { duration: '30s', target: 1000 }, // spike ขึ้น 1000 VUs ใน 30 วินาที!
        { duration: '2m', target: 1000 },  // stay
        { duration: '30s', target: 100 },  // back to normal
      ],
    },
  },
  
  thresholds: {
    // API endpoints ต้อง pass ทั้ง scenarios
    http_req_duration: [
      'p(50)<100',    // P50 < 100ms
      'p(95)<500',    // P95 < 500ms
      'p(99)<1000',   // P99 < 1 second
    ],
    'http_req_duration{endpoint:feed}': ['p(95)<300'],
    'http_req_duration{endpoint:search}': ['p(95)<500'],
    http_req_failed: ['rate<0.01'],  // < 1% error rate
  },
};

export default function() {
  const user = randomItem(users);
  const BASE_URL = __ENV.BASE_URL || 'https://api.chuaikan.com';
  
  const headers = {
    'Authorization': `Bearer ${user.token}`,
    'Content-Type': 'application/json',
  };
  
  group('User Journey: Browse Feed', function() {
    // Step 1: Load feed
    const feedRes = http.get(
      `${BASE_URL}/api/posts?page=1&limit=20`,
      { headers, tags: { endpoint: 'feed' } }
    );
    
    check(feedRes, {
      'feed OK': (r) => r.status === 200,
      'feed < 300ms': (r) => r.timings.duration < 300,
    });
    
    sleep(2);  // user reads feed
  });
  
  group('User Journey: Search', function() {
    const queries = ['ช่วยเหลือ', 'ข่าวสาร', 'SOS', 'emergency', 'อุบัติเหตุ'];
    const query = randomItem(queries);
    
    const searchRes = http.get(
      `${BASE_URL}/api/search?q=${encodeURIComponent(query)}&limit=10`,
      { headers, tags: { endpoint: 'search' } }
    );
    
    check(searchRes, {
      'search OK': (r) => r.status === 200,
      'search < 500ms': (r) => r.timings.duration < 500,
    });
    
    sleep(1);
  });
  
  group('User Journey: Create Post', function() {
    // 20% ของ users สร้าง post
    if (Math.random() < 0.2) {
      const postRes = http.post(
        `${BASE_URL}/api/posts`,
        JSON.stringify({
          content: 'Load test post content',
          type: 'text',
        }),
        { headers, tags: { endpoint: 'create_post' } }
      );
      
      check(postRes, {
        'create post OK': (r) => r.status === 201,
        'create post < 500ms': (r) => r.timings.duration < 500,
      });
    }
  });
  
  sleep(Math.random() * 3 + 1);  // 1-4 seconds think time
}
```

### Step 663: 100k VU Distributed Test

```javascript
// tests/distributed-100k.js - สำหรับ 100k VUs ด้วย distributed execution
import http from 'k6/http';
import { check, sleep } from 'k6';
import { SharedArray } from 'k6/data';

export const options = {
  scenarios: {
    // Gradual ramp to 100k VUs
    massive_load: {
      executor: 'ramping-arrival-rate',
      // arrival-rate ดีกว่า ramping-vus สำหรับ scale ใหญ่
      preAllocatedVUs: 1000,
      maxVUs: 110000,
      stages: [
        { duration: '10m', target: 10000 },   // 10k req/s
        { duration: '10m', target: 50000 },   // 50k req/s
        { duration: '10m', target: 100000 },  // 100k req/s
        { duration: '10m', target: 100000 },  // sustain
        { duration: '5m', target: 0 },         // ramp down
      ],
    },
  },
  
  thresholds: {
    http_req_duration: ['p(95)<2000'],  // ผ่อนปรน threshold ที่ 100k load
    http_req_failed: ['rate<0.05'],     // ยอมรับ 5% error ที่ extreme load
  },
};

export default function() {
  // Lightweight test script สำหรับ 100k test
  // Focus บน health check และ critical paths
  const res = http.get('https://api.chuaikan.com/health', {
    timeout: '10s',
  });
  
  check(res, {
    'status 200': (r) => r.status === 200,
  });
  
  sleep(0.1);  // minimal think time
}
```

```yaml
# k8s/k6-operator-test.yaml - Distributed test บน Kubernetes
apiVersion: k6.io/v1alpha1
kind: TestRun
metadata:
  name: chuaikan-100k-test
  namespace: k6-tests
spec:
  parallelism: 20  # 20 k6 runners
  
  script:
    configMap:
      name: k6-test-script
      file: distributed-100k.js
  
  arguments: |
    --out influxdb=http://influxdb:8086/k6
    --tag testid=100k-test
  
  runner:
    image: grafana/k6:latest
    resources:
      requests:
        cpu: "2"
        memory: "4Gi"
      limits:
        cpu: "4"
        memory: "8Gi"
    
    env:
    - name: BASE_URL
      value: "https://api.chuaikan.com"
    
  # ใช้ nodeSelector เพื่อรัน k6 บน dedicated nodes
  nodeSelector:
    node-role: load-test
```

```bash
# Install k6 operator บน Kubernetes
helm repo add grafana https://grafana.github.io/helm-charts
helm install k6-operator grafana/k6-operator \
  --namespace k6-operator \
  --create-namespace

# สร้าง ConfigMap จาก test script
kubectl create configmap k6-test-script \
  --from-file=distributed-100k.js=tests/distributed-100k.js \
  -n k6-tests

# รัน distributed test
kubectl apply -f k8s/k6-operator-test.yaml

# ดู progress
kubectl get testruns -n k6-tests
kubectl logs -f -l k6_cr=chuaikan-100k-test -n k6-tests
```

### Step 664: Database Connection Pool Test

```javascript
// tests/db-pool-exhaustion.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  scenarios: {
    // Gradually increase until we exhaust the connection pool
    pool_exhaustion: {
      executor: 'ramping-arrival-rate',
      preAllocatedVUs: 100,
      maxVUs: 500,
      stages: [
        { duration: '2m', target: 50 },
        { duration: '2m', target: 100 },
        { duration: '2m', target: 200 },  // จะเริ่มเห็น pool exhaustion
        { duration: '2m', target: 300 },
        { duration: '2m', target: 0 },
      ],
    },
  },
  thresholds: {
    'http_req_duration{endpoint:db_heavy}': ['p(95)<500'],
    http_req_failed: ['rate<0.01'],
  },
};

export default function() {
  // Test endpoint ที่ทำ database query
  const res = http.get(
    'https://api.chuaikan.com/api/users?include=posts,comments',
    {
      tags: { endpoint: 'db_heavy' },
      timeout: '30s',
    }
  );
  
  check(res, {
    'status 200': (r) => r.status === 200,
    'no pool error': (r) => !r.body.includes('pool is exhausted'),
    'no timeout': (r) => r.status !== 503,
  });
  
  // ดู database pool metrics ใน Grafana ระหว่าง test
  sleep(0.5);
}
```

### Step 665: Redis Connection Test

```javascript
// tests/redis-stress.js - Test Redis under high load
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  vus: 1000,
  duration: '5m',
  thresholds: {
    'http_req_duration{cache:hit}': ['p(95)<50'],   // cache hit < 50ms
    'http_req_duration{cache:miss}': ['p(95)<200'],  // cache miss < 200ms
    http_req_failed: ['rate<0.001'],
  },
};

export default function() {
  const userId = Math.floor(Math.random() * 10000);
  
  // Test cached endpoint (should hit Redis)
  const cachedRes = http.get(
    `https://api.chuaikan.com/api/users/${userId}/profile`,
    { tags: { cache: 'hit' } }
  );
  
  check(cachedRes, {
    'cached profile OK': (r) => r.status === 200,
    'cache header present': (r) => r.headers['X-Cache'] !== undefined,
  });
  
  // Test session validation (heavy Redis usage)
  const sessionRes = http.get(
    'https://api.chuaikan.com/api/me',
    {
      headers: { 'Authorization': 'Bearer test-token' },
      tags: { cache: 'session' },
    }
  );
  
  sleep(0.1);
}
```

### Step 666: WebSocket Scale Test

```javascript
// tests/websocket-scale.js - Test WebSocket connections at scale
import ws from 'k6/ws';
import { check, sleep } from 'k6';
import { Counter, Trend } from 'k6/metrics';

const wsMessages = new Counter('ws_messages_sent');
const wsLatency = new Trend('ws_message_latency');

export const options = {
  vus: 10000,  // 10k concurrent WebSocket connections
  duration: '10m',
  
  thresholds: {
    'ws_connecting': ['p(95)<5000'],           // WS connect < 5s
    'ws_session_duration': ['p(95)<600000'],   // session < 10 min
    ws_messages_sent: ['count>100000'],        // at least 100k messages sent
  },
};

export default function() {
  const userId = __VU;  // virtual user ID
  const WS_URL = `wss://api.chuaikan.com/ws?userId=${userId}`;
  
  const res = ws.connect(WS_URL, { 
    headers: { 'Authorization': 'Bearer test-token' },
    tags: { connection_type: 'ws' },
  }, function(socket) {
    socket.on('open', () => {
      // Join a room
      socket.send(JSON.stringify({
        event: 'join',
        room: `room-${Math.floor(userId / 100)}`,  // 100 users per room
      }));
      
      wsMessages.add(1);
    });
    
    socket.on('message', (msg) => {
      const data = JSON.parse(msg);
      
      if (data.event === 'pong') {
        const latency = Date.now() - data.sentAt;
        wsLatency.add(latency);
      }
    });
    
    socket.on('error', (e) => {
      console.error('WS Error:', e.error());
    });
    
    // Send periodic pings
    socket.setInterval(function() {
      socket.send(JSON.stringify({
        event: 'ping',
        sentAt: Date.now(),
      }));
      wsMessages.add(1);
    }, 30000);  // ping every 30 seconds
    
    // ปิด connection หลัง random duration (simulate real users)
    socket.setTimeout(function() {
      socket.close();
    }, Math.random() * 300000 + 60000);  // 1-6 minutes
  });
  
  check(res, {
    'ws connected successfully': (r) => r && r.status === 101,
  });
}
```

### Step 667: Bottleneck Identification

```javascript
// tests/bottleneck-finder.js - Systematic bottleneck identification
import http from 'k6/http';
import { check, group, sleep } from 'k6';
import { Trend } from 'k6/metrics';

// Separate metrics สำหรับแต่ละ component
const dbMetric = new Trend('component_database');
const cacheMetric = new Trend('component_cache');
const computeMetric = new Trend('component_compute');
const networkMetric = new Trend('component_network');

export const options = {
  stages: [
    { duration: '2m', target: 50 },
    { duration: '5m', target: 200 },
    { duration: '2m', target: 0 },
  ],
  thresholds: {
    component_database: ['p(95)<100'],
    component_cache: ['p(95)<10'],
    component_compute: ['p(95)<50'],
  },
};

export default function() {
  group('Component: Database', function() {
    const res = http.get('https://api.chuaikan.com/debug/db-query', {
      tags: { component: 'database' },
    });
    dbMetric.add(res.timings.duration);
    check(res, { 'db ok': (r) => r.status === 200 });
  });
  
  group('Component: Cache', function() {
    const res = http.get('https://api.chuaikan.com/debug/cache-get', {
      tags: { component: 'cache' },
    });
    cacheMetric.add(res.timings.duration);
    check(res, { 'cache ok': (r) => r.status === 200 });
  });
  
  group('Component: Full API', function() {
    const start = Date.now();
    const res = http.get('https://api.chuaikan.com/api/posts?page=1', {
      tags: { component: 'full_api' },
    });
    
    // Parse timing from response header
    const serverTiming = res.headers['Server-Timing'];
    if (serverTiming) {
      // Parse: db;dur=45.2, cache;dur=2.1, render;dur=12.3
      const timings = serverTiming.split(',').reduce((acc, t) => {
        const [name, dur] = t.trim().split(';dur=');
        acc[name.trim()] = parseFloat(dur);
        return acc;
      }, {});
      
      if (timings.db) dbMetric.add(timings.db);
      if (timings.cache) cacheMetric.add(timings.cache);
    }
  });
  
  sleep(1);
}
```

### Step 668: Infrastructure Auto-Scaling Test

```bash
#!/bin/bash
# test-autoscaling.sh - ทดสอบ auto-scaling ระหว่าง load test

NAMESPACE="production"
DEPLOYMENT="chuaikan-api"

echo "=== Infrastructure Auto-Scaling Test ==="
echo "Initial pod count:"
kubectl get pods -n $NAMESPACE -l app=$DEPLOYMENT --no-headers | wc -l

# เริ่ม load test ใน background
k6 run \
  --vus 0 \
  --stage 0s:10,5m:500,10m:500,15m:0 \
  tests/load-test.js &
K6_PID=$!

# Monitor scaling
echo "Monitoring HPA and pod count every 30 seconds..."
while kill -0 $K6_PID 2>/dev/null; do
  PODS=$(kubectl get pods -n $NAMESPACE -l app=$DEPLOYMENT --no-headers | grep Running | wc -l)
  CPU=$(kubectl top pods -n $NAMESPACE -l app=$DEPLOYMENT 2>/dev/null | awk 'NR>1{sum+=$2} END{print sum}')
  HPA=$(kubectl get hpa -n $NAMESPACE chuaikan-api-hpa --no-headers 2>/dev/null)
  
  echo "$(date '+%H:%M:%S') | Pods: $PODS | CPU: ${CPU}m | HPA: $HPA"
  sleep 30
done

echo "Load test completed."
echo "Final pod count: $(kubectl get pods -n $NAMESPACE -l app=$DEPLOYMENT --no-headers | wc -l)"
```

### Step 669: Result Analysis

```javascript
// analyze-results.js - วิเคราะห์ k6 JSON output
const fs = require('fs');

function analyzeK6Results(jsonFile) {
  const lines = fs.readFileSync(jsonFile, 'utf8').trim().split('\n');
  
  const metrics = {
    http_req_duration: [],
    http_req_failed: 0,
    total_requests: 0,
  };
  
  for (const line of lines) {
    try {
      const point = JSON.parse(line);
      
      if (point.metric === 'http_req_duration') {
        metrics.http_req_duration.push(point.data.value);
      }
      if (point.metric === 'http_req_failed') {
        metrics.http_req_failed += point.data.value;
        metrics.total_requests++;
      }
    } catch (e) {
      // skip non-JSON lines
    }
  }
  
  const sorted = [...metrics.http_req_duration].sort((a, b) => a - b);
  const n = sorted.length;
  
  const percentile = (p) => sorted[Math.floor(n * p / 100)];
  
  console.log('=== Load Test Results ===');
  console.log(`Total Requests: ${n.toLocaleString()}`);
  console.log(`Error Rate: ${((metrics.http_req_failed / metrics.total_requests) * 100).toFixed(2)}%`);
  console.log('\nLatency Distribution:');
  console.log(`  P50: ${percentile(50)?.toFixed(2)}ms`);
  console.log(`  P75: ${percentile(75)?.toFixed(2)}ms`);
  console.log(`  P90: ${percentile(90)?.toFixed(2)}ms`);
  console.log(`  P95: ${percentile(95)?.toFixed(2)}ms`);
  console.log(`  P99: ${percentile(99)?.toFixed(2)}ms`);
  console.log(`  Max: ${sorted[n-1]?.toFixed(2)}ms`);
  
  // Throughput (requests per second)
  // ต้องมี startTime/endTime จาก test
  
  return { sorted, percentile };
}

analyzeK6Results('./results.json');
```

### Step 670: Soak Test สำหรับ Memory Leak Detection

```javascript
// tests/soak-test.js - 8-hour soak test
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  // 8-hour soak test ด้วย moderate load
  stages: [
    { duration: '30m', target: 100 },   // ramp up
    { duration: '7h', target: 100 },    // soak at 100 VUs for 7 hours
    { duration: '30m', target: 0 },     // ramp down
  ],
  
  thresholds: {
    // Thresholds ต้อง consistent ตลอด test
    // memory leak จะทำให้ latency เพิ่มขึ้นเรื่อยๆ
    http_req_duration: ['p(95)<500'],
    http_req_failed: ['rate<0.01'],
  },
};

let startTime = Date.now();

export default function() {
  const elapsed = Math.floor((Date.now() - startTime) / 60000);
  
  const res = http.get('https://api.chuaikan.com/api/posts');
  
  check(res, {
    'status 200': (r) => r.status === 200,
    // บันทึก latency แต่ละ hour
    [`hour ${Math.floor(elapsed / 60)}: < 500ms`]: (r) => r.timings.duration < 500,
  });
  
  sleep(Math.random() * 2 + 1);
}
```

---

## 🔧 Configuration Files

```yaml
# k8s/k6-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: k6-tests
---
# Dedicated nodes สำหรับ load testing
apiVersion: v1
kind: Node
metadata:
  labels:
    node-role: load-test
```

```bash
# .env.test
BASE_URL=https://staging.chuaikan.com
API_TOKEN=load-test-token
DB_MAX_CONNECTIONS=100
REDIS_MAX_CONNECTIONS=50
```

---

## 🧪 Testing

```bash
# Quick smoke test
k6 run tests/smoke-test.js

# Full load test พร้อม InfluxDB output
k6 run \
  --out influxdb=http://localhost:8086/k6 \
  tests/load-test.js

# ดู results ใน Grafana
open http://localhost:3001

# Distributed test บน K8s
kubectl apply -f k8s/k6-operator-test.yaml
kubectl get testruns -n k6-tests --watch
```

---

## ❌ Common Errors & Solutions

### Error 1: Too many open files

```bash
# ปัญหา: k6 runner เปิด file descriptors เกิน limit
WARN[0001] Request Failed  error="dial tcp: lookup api.chuaikan.com: too many open files"

# แก้ไข: เพิ่ม file descriptor limit
ulimit -n 65536
# หรือใน /etc/security/limits.conf
echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 65536" >> /etc/security/limits.conf
```

### Error 2: High P99 latency ใน distributed test

```bash
# ปัญหา: Load balancer timeout ทำให้ P99 สูง
# ตรวจสอบ load balancer timeout settings

# AWS ALB: default idle timeout = 60s
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn $ALB_ARN \
  --attributes Key=idle_timeout.timeout_seconds,Value=120
```

### Error 3: k6 operator pod OOMKilled

```yaml
# แก้ไข: เพิ่ม memory ให้ k6 runner
runner:
  resources:
    requests:
      memory: "8Gi"
    limits:
      memory: "16Gi"
```

---

## ✅ Checklist

- [ ] รัน smoke test ก่อน load test เสมอ
- [ ] ตั้งค่า thresholds ก่อนรัน (P95 < 200ms, error < 1%)
- [ ] Monitor Kubernetes pod scaling ระหว่าง test
- [ ] Monitor database connection pool ระหว่าง test
- [ ] Monitor Redis connection count ระหว่าง test
- [ ] รัน soak test อย่างน้อย 4 ชั่วโมง (เพื่อหา memory leaks)
- [ ] ตั้งค่า k6 output → Grafana dashboard
- [ ] รัน distributed test บน dedicated nodes
- [ ] Document bottlenecks ที่พบและ fixes
- [ ] Automate performance regression tests ใน CI

---

## 🔗 References

- [k6 Documentation](https://k6.io/docs/)
- [k6 Operator for Kubernetes](https://k6.io/docs/k6-oss/k6-operator/)
- [k6 Output Formats](https://k6.io/docs/results-output/)
- [Load Testing Patterns](https://k6.io/docs/test-types/)
- [k6 Scenarios](https://k6.io/docs/using-k6/scenarios/)

---

*Part 067 | Road to 1,000,000 Users/Day | chuaikan.com*
