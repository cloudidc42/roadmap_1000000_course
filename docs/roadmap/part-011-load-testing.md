# Part 011: Load Testing ด้วย k6 / Artillery

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 101-110
> **เวลาโดยประมาณ:** 6 ชั่วโมง
> **Prerequisites:** Part 001-010 (Infrastructure Setup, Docker, CI/CD พื้นฐาน)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

1. ทำความเข้าใจ Load Testing Concepts: RPS, VU, Throughput, Latency Percentiles
2. ติดตั้งและใช้งาน k6 บน Ubuntu 24.04 LTS
3. เขียน k6 script สำหรับ chuaikan.com (Homepage, API, WebSocket)
4. กำหนด k6 Stages: Ramp Up, Sustain, Ramp Down
5. ตั้งค่า k6 Thresholds และ Checks
6. ติดตั้งและใช้งาน Artillery
7. เขียน Artillery scenario: User Registration → Login → Post flow
8. เปรียบเทียบ k6 Cloud vs Self-hosted Results
9. ตั้งค่า Grafana + InfluxDB สำหรับ k6 results
10. วิเคราะห์ผลลัพธ์และหา Bottlenecks

---

## 📖 ทฤษฎีและแนวคิด

### Load Testing คืออะไร?

Load Testing คือการจำลองการใช้งานจริงของผู้ใช้จำนวนมากพร้อมกันบนระบบ เพื่อตรวจสอบว่าระบบรองรับได้มากแค่ไหนก่อนที่จะเริ่มมีปัญหา

```
                    ┌─────────────────────────────────┐
                    │         Load Test Types          │
                    └─────────────────────────────────┘

  Load Test         ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
  (ทดสอบปกติ)       0────────────────────────────────▶ time

  Stress Test       ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓██████
  (ทดสอบขีดจำกัด)   0────────────────────────────────▶ time

  Spike Test        ▓▓▓▓▓▓▓▓▓██████████▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
  (ทดสอบ traffic พุ่ง) 0──────────────────────────────▶ time

  Soak Test         ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
  (ทดสอบระยะยาว)    0────────────────────────────────▶ 24h
```

### คำศัพท์สำคัญ

**RPS (Requests Per Second)** — จำนวน HTTP requests ที่ระบบรับได้ต่อวินาที
- chuaikan.com target: **11,574 RPS** (= 1,000,000 users/day ÷ 86,400 seconds)
- ในทางปฏิบัติต้องรองรับ peak traffic ที่ **3-5x** = ~50,000 RPS

**VU (Virtual Users)** — จำนวนผู้ใช้จำลองที่ส่ง request พร้อมกัน
- VU ≠ RPS: VU 100 คน ส่ง request เฉลี่ยคนละ 1 req/s = 100 RPS

**Throughput** — ปริมาณข้อมูลที่ผ่านระบบได้จริงต่อหน่วยเวลา (bytes/sec)

**Latency Percentiles** — การวัด response time ในรูปแบบ percentile:
```
p50  = 50th percentile  → 50% ของ requests เร็วกว่านี้ (ค่ากลาง)
p90  = 90th percentile  → 90% ของ requests เร็วกว่านี้
p95  = 95th percentile  → 95% ของ requests เร็วกว่านี้
p99  = 99th percentile  → 99% ของ requests เร็วกว่านี้
```

**ตัวอย่าง:** ถ้า p95 = 500ms แปลว่า 95% ของผู้ใช้ได้รับ response ภายใน 500ms

### Load Testing Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    k6 Load Test Runner                   │
│                                                          │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐             │
│  │  VU 1   │    │  VU 2   │    │  VU N   │             │
│  └────┬────┘    └────┬────┘    └────┬────┘             │
└───────┼──────────────┼──────────────┼────────────────── ┘
        │              │              │
        ▼              ▼              ▼
┌───────────────────────────────────────────────┐
│              chuaikan.com API                  │
│  ┌──────────┐  ┌──────────┐  ┌────────────┐  │
│  │ Next.js  │  │ Node.js  │  │ PostgreSQL  │  │
│  │   15     │  │  API     │  │    17      │  │
│  └──────────┘  └──────────┘  └────────────┘  │
└───────────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────────────┐
│           Metrics Storage                      │
│   InfluxDB  ◄───────────────► Grafana         │
└───────────────────────────────────────────────┘
```

---

## ⚙️ Environment Setup

### Step 101: ติดตั้ง k6 บน Ubuntu 24.04 LTS

```bash
# เพิ่ม k6 GPG key และ repository
sudo gpg -k
sudo gpg --no-default-keyring \
  --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
  --keyserver hkp://keyserver.ubuntu.com:80 \
  --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69

echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] \
  https://dl.k6.io/deb stable main" | \
  sudo tee /etc/apt/sources.list.d/k6.list

sudo apt-get update
sudo apt-get install k6 -y

# ตรวจสอบการติดตั้ง
k6 version
# k6 v0.54.0 (go1.22.x, linux/amd64)
```

### Step 102: ติดตั้ง InfluxDB 2.x สำหรับเก็บ metrics

```bash
# ติดตั้ง InfluxDB 2.x
wget -q https://repos.influxdata.com/influxdata-archive_compat.key
echo '393e8779c89ac8d958f81f942f9ad7fb82a25e133faddaf92e15b16e6ac9ce4c \
  influxdata-archive_compat.key' | sha256sum -c
cat influxdata-archive_compat.key | gpg --dearmor | \
  sudo tee /etc/apt/trusted.gpg.d/influxdata-archive_compat.gpg > /dev/null

echo 'deb [signed-by=/etc/apt/trusted.gpg.d/influxdata-archive_compat.gpg] \
  https://repos.influxdata.com/debian stable main' | \
  sudo tee /etc/apt/sources.list.d/influxdata.list

sudo apt-get update && sudo apt-get install influxdb2 -y
sudo systemctl enable influxdb
sudo systemctl start influxdb

# Initial setup InfluxDB
influx setup \
  --username admin \
  --password SecurePassword123 \
  --org chuaikan \
  --bucket k6-metrics \
  --retention 30d \
  --force
```

### Step 103: ติดตั้ง Grafana

```bash
# ติดตั้ง Grafana OSS
sudo apt-get install -y apt-transport-https software-properties-common wget
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | \
  gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] \
  https://apt.grafana.com stable main" | \
  sudo tee /etc/apt/sources.list.d/grafana.list

sudo apt-get update
sudo apt-get install grafana -y
sudo systemctl daemon-reload
sudo systemctl enable grafana-server
sudo systemctl start grafana-server

# เข้าใช้งาน Grafana ที่ http://localhost:3000
# Default login: admin/admin
```

### Step 104: ติดตั้ง Artillery

```bash
# ต้องมี Node.js 22 ก่อน
node --version  # v22.x.x

# ติดตั้ง Artillery แบบ global
npm install -g artillery@latest

# ตรวจสอบการติดตั้ง
artillery version
# Artillery: 2.0.x

# ติดตั้ง Artillery plugins ที่จำเป็น
npm install -g artillery-plugin-expect
npm install -g artillery-plugin-metrics-by-endpoint
```

---

## 🛠️ Step-by-Step Implementation

### Step 105: k6 Script สำหรับ Homepage Load Test

สร้างไฟล์ `/home/user/chuaikan-loadtest/k6/homepage-test.js`:

```javascript
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Rate, Trend, Counter } from 'k6/metrics';

// Custom metrics สำหรับ chuaikan.com
const errorRate = new Rate('errors');
const homepageDuration = new Trend('homepage_duration');
const apiDuration = new Trend('api_duration');
const failedRequests = new Counter('failed_requests');

// Base URL - เปลี่ยนตาม environment
const BASE_URL = __ENV.BASE_URL || 'https://chuaikan.com';

// k6 Options: กำหนด load pattern
export const options = {
  // Stages: Ramp Up → Sustain → Ramp Down
  stages: [
    { duration: '2m', target: 100 },   // Ramp up: 0 → 100 VUs ใน 2 นาที
    { duration: '5m', target: 100 },   // Sustain: คง 100 VUs 5 นาที
    { duration: '2m', target: 500 },   // Ramp up: 100 → 500 VUs
    { duration: '5m', target: 500 },   // Sustain: คง 500 VUs 5 นาที
    { duration: '2m', target: 1000 },  // Ramp up: 500 → 1000 VUs
    { duration: '10m', target: 1000 }, // Peak load: คง 1000 VUs 10 นาที
    { duration: '3m', target: 0 },     // Ramp down: 1000 → 0 VUs
  ],

  // Thresholds: กำหนดเกณฑ์ที่ยอมรับได้
  thresholds: {
    // 95% ของ requests ต้องเสร็จภายใน 500ms
    'http_req_duration': ['p(95)<500', 'p(99)<1000'],
    // Error rate ต้องน้อยกว่า 1%
    'errors': ['rate<0.01'],
    // Homepage ต้องเร็วกว่า 300ms (p95)
    'homepage_duration': ['p(95)<300'],
    // API ต้องเร็วกว่า 200ms (p95)
    'api_duration': ['p(95)<200'],
    // HTTP failure rate < 0.5%
    'http_req_failed': ['rate<0.005'],
  },
};

// Setup function: ทำงานครั้งเดียวก่อน test เริ่ม
export function setup() {
  console.log(`Starting load test for: ${BASE_URL}`);
  
  // ตรวจสอบว่า server พร้อมทำงาน
  const healthCheck = http.get(`${BASE_URL}/api/health`);
  if (healthCheck.status !== 200) {
    throw new Error(`Server not ready: ${healthCheck.status}`);
  }
  
  return { baseUrl: BASE_URL };
}

// Default function: ทำงานซ้ำในแต่ละ VU
export default function (data) {
  const baseUrl = data.baseUrl;

  // จำลอง user browsing pattern ของ chuaikan.com
  group('Homepage Flow', function () {
    // 1. โหลด Homepage
    const homeResponse = http.get(baseUrl, {
      headers: { 'Accept': 'text/html,application/xhtml+xml' },
      tags: { name: 'homepage' },
    });

    const homeCheck = check(homeResponse, {
      'homepage status 200': (r) => r.status === 200,
      'homepage has content': (r) => r.body.includes('chuaikan'),
      'homepage loads fast': (r) => r.timings.duration < 500,
    });

    homepageDuration.add(homeResponse.timings.duration);
    errorRate.add(!homeCheck);
    if (!homeCheck) failedRequests.add(1);

    sleep(1); // Think time: 1 วินาที

    // 2. โหลด Posts Feed API
    const feedResponse = http.get(`${baseUrl}/api/posts/feed?page=1&limit=20`, {
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
      },
      tags: { name: 'api_feed' },
    });

    const feedCheck = check(feedResponse, {
      'feed status 200': (r) => r.status === 200,
      'feed returns JSON': (r) => {
        try {
          const body = JSON.parse(r.body);
          return Array.isArray(body.posts);
        } catch (e) {
          return false;
        }
      },
      'feed response time OK': (r) => r.timings.duration < 200,
    });

    apiDuration.add(feedResponse.timings.duration);
    errorRate.add(!feedCheck);

    sleep(2); // Think time: 2 วินาที

    // 3. โหลด Post Detail
    const postId = Math.floor(Math.random() * 10000) + 1;
    const postResponse = http.get(`${baseUrl}/api/posts/${postId}`, {
      tags: { name: 'api_post_detail' },
    });

    check(postResponse, {
      'post detail status ok': (r) => r.status === 200 || r.status === 404,
    });

    sleep(1);
  });

  group('SOS Alert Feed', function () {
    // โหลด SOS alerts - feature หลักของ chuaikan.com
    const sosResponse = http.get(`${baseUrl}/api/sos/active?lat=13.7563&lng=100.5018&radius=10`, {
      headers: { 'Accept': 'application/json' },
      tags: { name: 'api_sos_feed' },
    });

    check(sosResponse, {
      'SOS feed status 200': (r) => r.status === 200,
      'SOS feed response time OK': (r) => r.timings.duration < 300,
    });

    apiDuration.add(sosResponse.timings.duration);
    sleep(1);
  });
}

// Teardown function: ทำงานครั้งเดียวหลัง test จบ
export function teardown(data) {
  console.log('Load test completed for: ' + data.baseUrl);
}
```

### Step 106: k6 Script สำหรับ API Load Test (Authenticated)

สร้างไฟล์ `/home/user/chuaikan-loadtest/k6/api-auth-test.js`:

```javascript
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { SharedArray } from 'k6/data';
import { Rate } from 'k6/metrics';

const BASE_URL = __ENV.BASE_URL || 'https://api.chuaikan.com';
const errorRate = new Rate('errors');

// โหลด test users จากไฟล์ JSON (สร้างไว้ล่วงหน้า)
const testUsers = new SharedArray('users', function () {
  return JSON.parse(open('./data/test-users.json'));
});

export const options = {
  stages: [
    { duration: '1m', target: 50 },
    { duration: '5m', target: 200 },
    { duration: '10m', target: 500 },
    { duration: '2m', target: 0 },
  ],
  thresholds: {
    'http_req_duration{name:login}': ['p(95)<300'],
    'http_req_duration{name:create_post}': ['p(95)<500'],
    'http_req_duration{name:get_feed}': ['p(95)<200'],
    'errors': ['rate<0.02'],
  },
};

// Helper: Login และดึง JWT token
function loginUser(user) {
  const loginPayload = JSON.stringify({
    email: user.email,
    password: user.password,
  });

  const loginResponse = http.post(
    `${BASE_URL}/auth/login`,
    loginPayload,
    {
      headers: { 'Content-Type': 'application/json' },
      tags: { name: 'login' },
    }
  );

  const loginSuccess = check(loginResponse, {
    'login status 200': (r) => r.status === 200,
    'login returns token': (r) => {
      try {
        const body = JSON.parse(r.body);
        return body.accessToken && body.accessToken.length > 0;
      } catch (e) {
        return false;
      }
    },
  });

  errorRate.add(!loginSuccess);

  if (loginResponse.status === 200) {
    return JSON.parse(loginResponse.body).accessToken;
  }
  return null;
}

export default function () {
  // สุ่มเลือก test user
  const user = testUsers[Math.floor(Math.random() * testUsers.length)];
  
  let token = loginUser(user);
  if (!token) {
    sleep(1);
    return;
  }

  const authHeaders = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`,
  };

  sleep(0.5);

  group('Authenticated User Flow', function () {
    // ดู Profile ตัวเอง
    const profileResponse = http.get(`${BASE_URL}/users/me`, {
      headers: authHeaders,
      tags: { name: 'get_profile' },
    });
    check(profileResponse, { 'get profile 200': (r) => r.status === 200 });

    sleep(1);

    // สร้าง Post ใหม่
    const postPayload = JSON.stringify({
      content: `Test post from load test at ${new Date().toISOString()}`,
      type: 'text',
      visibility: 'public',
      location: { lat: 13.7563, lng: 100.5018, name: 'Bangkok' },
    });

    const createPostResponse = http.post(
      `${BASE_URL}/posts`,
      postPayload,
      {
        headers: authHeaders,
        tags: { name: 'create_post' },
      }
    );

    check(createPostResponse, {
      'create post 201': (r) => r.status === 201,
      'post has id': (r) => {
        try {
          return JSON.parse(r.body).id !== undefined;
        } catch (e) {
          return false;
        }
      },
    });

    sleep(2);

    // Like โพสต์แบบสุ่ม
    const randomPostId = Math.floor(Math.random() * 5000) + 1;
    const likeResponse = http.post(
      `${BASE_URL}/posts/${randomPostId}/like`,
      null,
      {
        headers: authHeaders,
        tags: { name: 'like_post' },
      }
    );
    check(likeResponse, {
      'like status ok': (r) => r.status === 200 || r.status === 409,
    });

    sleep(1);
  });
}
```

### Step 107: k6 WebSocket Test

สร้างไฟล์ `/home/user/chuaikan-loadtest/k6/websocket-test.js`:

```javascript
import ws from 'k6/ws';
import { check, sleep } from 'k6';
import { Counter, Rate } from 'k6/metrics';

const WS_URL = __ENV.WS_URL || 'wss://ws.chuaikan.com';
const wsErrors = new Counter('ws_errors');
const wsConnectSuccess = new Rate('ws_connect_success');

export const options = {
  vus: 200,
  duration: '10m',
  thresholds: {
    'ws_connect_success': ['rate>0.95'],  // 95% ของ WebSocket ต้อง connect ได้
    'ws_errors': ['count<100'],
    'ws_session_duration': ['p(95)<30000'],
  },
};

export default function () {
  // ใช้ JWT token จาก environment variable
  const token = __ENV.TEST_TOKEN || 'test-jwt-token';

  const wsResponse = ws.connect(
    `${WS_URL}/socket.io/?EIO=4&transport=websocket&token=${token}`,
    {},
    function (socket) {
      // Connection event
      socket.on('open', () => {
        wsConnectSuccess.add(true);
        console.log('WebSocket connected');

        // Join room สำหรับ notifications
        socket.send(JSON.stringify({
          type: 'join',
          room: 'notifications',
        }));

        // Subscribe to SOS alerts ในพื้นที่ Bangkok
        socket.send(JSON.stringify({
          type: 'subscribe_sos',
          lat: 13.7563,
          lng: 100.5018,
          radius: 10,
        }));
      });

      // Message event
      socket.on('message', (data) => {
        try {
          const message = JSON.parse(data);
          check(message, {
            'valid message format': (m) => m.type !== undefined,
          });

          // จำลองการ respond ต่อ notification
          if (message.type === 'new_post') {
            socket.send(JSON.stringify({
              type: 'mark_seen',
              postId: message.data.id,
            }));
          }
        } catch (e) {
          wsErrors.add(1);
        }
      });

      // Error event
      socket.on('error', (e) => {
        wsConnectSuccess.add(false);
        wsErrors.add(1);
        console.error('WebSocket error:', e.error());
      });

      // Close event
      socket.on('close', () => {
        console.log('WebSocket closed');
      });

      // คง connection ไว้ 30 วินาที พร้อม heartbeat ทุก 5 วินาที
      socket.setInterval(() => {
        socket.send(JSON.stringify({ type: 'ping' }));
      }, 5000);

      sleep(30);
    }
  );

  check(wsResponse, {
    'WebSocket connected successfully': (r) => r && r.status === 101,
  });
}
```

### Step 108: Artillery Configuration

สร้างไฟล์ `/home/user/chuaikan-loadtest/artillery/user-journey.yml`:

```yaml
# Artillery scenario สำหรับ chuaikan.com user journey
config:
  target: "https://api.chuaikan.com"
  http:
    timeout: 30
    pool: 50
  phases:
    # Warm up phase
    - name: "Warm up"
      duration: 60
      arrivalRate: 5
    # Ramp up phase
    - name: "Ramp up"
      duration: 120
      arrivalRate: 5
      rampTo: 50
    # Sustained load
    - name: "Sustained load"
      duration: 300
      arrivalRate: 50
    # Peak load
    - name: "Peak load"
      duration: 180
      arrivalRate: 100
    # Cool down
    - name: "Cool down"
      duration: 60
      arrivalRate: 10

  # Default headers สำหรับทุก request
  defaults:
    headers:
      Content-Type: "application/json"
      Accept: "application/json"
      X-Platform: "artillery-test"

  # Variables
  variables:
    testEmail: "loadtest_{{ $randomString() }}@test.com"

  # Plugin settings
  plugins:
    expect: {}
    metrics-by-endpoint:
      useOnlyRequestNames: true

  # Metrics thresholds
  ensure:
    p95: 500
    p99: 1000
    maxErrorRate: 1

scenarios:
  # Scenario 1: New User Registration Flow (30% ของ traffic)
  - name: "User Registration and First Post"
    weight: 30
    flow:
      # Step 1: Register
      - post:
          url: "/auth/register"
          name: "register"
          json:
            email: "{{ testEmail }}"
            password: "TestPass123!"
            username: "user_{{ $randomString(6) }}"
            displayName: "Test User"
          expect:
            - statusCode: 201
            - hasProperty: "userId"
          capture:
            - json: "$.accessToken"
              as: "authToken"
            - json: "$.userId"
              as: "userId"

      - think: 1

      # Step 2: Complete profile
      - put:
          url: "/users/{{ userId }}/profile"
          name: "update_profile"
          headers:
            Authorization: "Bearer {{ authToken }}"
          json:
            bio: "Test user for load testing"
            location: "Bangkok, Thailand"
          expect:
            - statusCode: 200

      - think: 2

      # Step 3: Create first post
      - post:
          url: "/posts"
          name: "create_first_post"
          headers:
            Authorization: "Bearer {{ authToken }}"
          json:
            content: "Hello chuaikan! This is my first post."
            type: "text"
            visibility: "public"
          expect:
            - statusCode: 201
            - hasProperty: "id"
          capture:
            - json: "$.id"
              as: "postId"

      - think: 1

      # Step 4: ดู feed หลังสร้าง post
      - get:
          url: "/posts/feed?page=1"
          name: "check_feed"
          headers:
            Authorization: "Bearer {{ authToken }}"
          expect:
            - statusCode: 200

  # Scenario 2: Existing User Login and Browse (50% ของ traffic)
  - name: "Login and Browse"
    weight: 50
    flow:
      - post:
          url: "/auth/login"
          name: "login"
          json:
            email: "testuser@chuaikan.com"
            password: "TestPass123!"
          expect:
            - statusCode: 200
            - hasProperty: "accessToken"
          capture:
            - json: "$.accessToken"
              as: "token"

      - think: 1

      - get:
          url: "/posts/feed?page=1&limit=20"
          name: "browse_feed"
          headers:
            Authorization: "Bearer {{ token }}"
          expect:
            - statusCode: 200

      - think: 3

      - get:
          url: "/sos/active?lat=13.7563&lng=100.5018&radius=10"
          name: "check_sos"
          headers:
            Authorization: "Bearer {{ token }}"
          expect:
            - statusCode: 200

      - think: 2

      # Like โพสต์
      - post:
          url: "/posts/{{ $randomInt(1, 10000) }}/like"
          name: "like_post"
          headers:
            Authorization: "Bearer {{ token }}"

      - think: 1

  # Scenario 3: SOS Emergency Flow (20% ของ traffic)
  - name: "SOS Emergency Alert"
    weight: 20
    flow:
      - post:
          url: "/auth/login"
          name: "sos_login"
          json:
            email: "sos_tester@chuaikan.com"
            password: "TestPass123!"
          capture:
            - json: "$.accessToken"
              as: "sosToken"

      - think: 0.5

      # สร้าง SOS alert
      - post:
          url: "/sos"
          name: "create_sos"
          headers:
            Authorization: "Bearer {{ sosToken }}"
          json:
            type: "accident"
            severity: "high"
            description: "Load test SOS alert"
            location:
              lat: 13.7563
              lng: 100.5018
              address: "Bangkok CBD"
          expect:
            - statusCode: 201
          capture:
            - json: "$.sosId"
              as: "sosId"

      - think: 2

      # อัปเดตสถานะ SOS
      - put:
          url: "/sos/{{ sosId }}/status"
          name: "update_sos_status"
          headers:
            Authorization: "Bearer {{ sosToken }}"
          json:
            status: "resolved"
          expect:
            - statusCode: 200
```

### Step 109: รัน k6 พร้อม InfluxDB Output

```bash
# สร้าง directory สำหรับ test data
mkdir -p /home/user/chuaikan-loadtest/{k6,artillery,data,results}
cd /home/user/chuaikan-loadtest

# สร้าง test users data file
cat > data/test-users.json << 'EOF'
[
  {"email": "testuser1@chuaikan.com", "password": "TestPass123!"},
  {"email": "testuser2@chuaikan.com", "password": "TestPass123!"},
  {"email": "testuser3@chuaikan.com", "password": "TestPass123!"},
  {"email": "testuser4@chuaikan.com", "password": "TestPass123!"},
  {"email": "testuser5@chuaikan.com", "password": "TestPass123!"}
]
EOF

# รัน k6 Homepage test พร้อมส่ง metrics ไป InfluxDB
k6 run \
  --out influxdb=http://localhost:8086/k6-metrics \
  --env BASE_URL=https://chuaikan.com \
  k6/homepage-test.js

# รัน k6 API auth test
k6 run \
  --out influxdb=http://localhost:8086/k6-metrics \
  --env BASE_URL=https://api.chuaikan.com \
  k6/api-auth-test.js \
  --vus 100 \
  --duration 5m

# รัน k6 WebSocket test
k6 run \
  --out influxdb=http://localhost:8086/k6-metrics \
  --env WS_URL=wss://ws.chuaikan.com \
  --env TEST_TOKEN=your_test_jwt_token \
  k6/websocket-test.js

# รัน Artillery user journey test
artillery run \
  --output results/artillery-$(date +%Y%m%d-%H%M%S).json \
  artillery/user-journey.yml

# สร้าง Artillery HTML report
artillery report \
  --output results/report.html \
  results/artillery-*.json
```

---

## 🔧 Configuration Files

### Grafana Dashboard สำหรับ k6

สร้างไฟล์ `/home/user/chuaikan-loadtest/grafana/k6-dashboard.json`:

```json
{
  "title": "k6 Load Test - chuaikan.com",
  "panels": [
    {
      "title": "HTTP Requests per Second",
      "type": "graph",
      "datasource": "InfluxDB",
      "targets": [
        {
          "query": "SELECT non_negative_derivative(mean(\"value\"), 1s) FROM \"http_reqs\" WHERE $timeFilter GROUP BY time($__interval)"
        }
      ]
    },
    {
      "title": "Response Time Percentiles",
      "type": "graph",
      "targets": [
        {
          "query": "SELECT percentile(\"value\", 95) as \"p95\", percentile(\"value\", 99) as \"p99\" FROM \"http_req_duration\" WHERE $timeFilter GROUP BY time($__interval)"
        }
      ]
    },
    {
      "title": "Virtual Users",
      "type": "graph",
      "targets": [
        {
          "query": "SELECT mean(\"value\") FROM \"vus\" WHERE $timeFilter GROUP BY time($__interval)"
        }
      ]
    },
    {
      "title": "Error Rate",
      "type": "singlestat",
      "targets": [
        {
          "query": "SELECT mean(\"value\") FROM \"errors\" WHERE $timeFilter"
        }
      ]
    }
  ]
}
```

### Docker Compose สำหรับ Monitoring Stack

สร้างไฟล์ `/home/user/chuaikan-loadtest/docker-compose.monitoring.yml`:

```yaml
version: '3.8'

services:
  influxdb:
    image: influxdb:2.7
    ports:
      - "8086:8086"
    environment:
      DOCKER_INFLUXDB_INIT_MODE: setup
      DOCKER_INFLUXDB_INIT_USERNAME: admin
      DOCKER_INFLUXDB_INIT_PASSWORD: SecurePassword123
      DOCKER_INFLUXDB_INIT_ORG: chuaikan
      DOCKER_INFLUXDB_INIT_BUCKET: k6-metrics
      DOCKER_INFLUXDB_INIT_RETENTION: 30d
    volumes:
      - influxdb-data:/var/lib/influxdb2
    healthcheck:
      test: ["CMD", "influx", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  grafana:
    image: grafana/grafana:10.4.0
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: admin123
      GF_INSTALL_PLUGINS: grafana-clock-panel
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources
    depends_on:
      influxdb:
        condition: service_healthy

volumes:
  influxdb-data:
  grafana-data:
```

---

## 🧪 Testing

### Step 110: วิเคราะห์ผลลัพธ์และหา Bottlenecks

หลังจากรัน load test จะได้ผลลัพธ์ดังนี้:

```
          /\      |‾‾| /‾‾/   /‾‾/
     /\  /  \     |  |/  /   /  /
    /  \/    \    |     (   /   ‾‾\
   /          \   |  |\  \ |  (‾)  |
  / __________ \  |__| \__\ \_____/ .io

     execution: local
        script: k6/homepage-test.js
        output: InfluxDB (http://localhost:8086)

     scenarios: (100.00%) 1 scenario, 1000 max VUs
              : default: Up to 1000 looping VUs for 29m0s

✓ homepage status 200
✓ homepage has content
✗ homepage loads fast
  ↳  78% — ✓ 156000 / ✗ 44000

✓ feed status 200
✗ feed returns JSON
  ↳  95% — ✓ 95000 / ✗ 5000

checks.........................: 92.50% ✓ 412500 ✗ 33500
data_received..................: 2.3 GB 1.3 MB/s
data_sent......................: 145 MB 83 kB/s
errors.........................: 7.50%  ✓ 0 ✗ 33500   [THRESHOLD FAILED]
homepage_duration p(95)........: 742ms            [THRESHOLD FAILED - target: 300ms]
http_req_blocked...............: avg=1.21ms  min=0s    med=0s    max=3.98s
http_req_connecting............: avg=0.31ms  min=0s    med=0s    max=1.23s
http_req_duration..............: avg=312ms   min=12ms  med=245ms max=8.41s
  { expected_response:true }...: avg=285ms   min=12ms  med=223ms max=4.12s
http_req_failed................: 3.12%  ✓ 6240 ✗ 193760
http_req_receiving.............: avg=45.2ms  min=0s    med=12ms  max=2.34s
http_req_sending...............: avg=0.87ms  min=0s    med=0s    max=134ms
http_reqs......................: 200000 115/s
vus............................: 234    min=0   max=1000
```

### วิเคราะห์ผลลัพธ์ที่ได้:

```
📊 Bottleneck Analysis สำหรับ chuaikan.com

❌ FAILED Thresholds:
   1. homepage_duration p(95) = 742ms (target: <300ms)
      → สาเหตุ: Database query ช้า, ไม่ได้ cache
      → แก้ไข: เพิ่ม Redis cache สำหรับ homepage data

   2. errors rate = 7.5% (target: <1%)
      → สาเหตุ: Connection pool exhaustion ที่ PostgreSQL
      → แก้ไข: เพิ่ม pool size, เพิ่ม PgBouncer

📈 Metrics Summary:
   - Peak RPS: 115 req/s (ยังไม่ถึง target 11,574 RPS)
   - p50 latency: 245ms ✓
   - p95 latency: 742ms ✗ (เกิน 500ms threshold)
   - p99 latency: 2.1s ✗ (เกิน 1000ms threshold)

🔧 Recommended Actions (priority order):
   1. เพิ่ม Redis caching สำหรับ API responses
   2. ปรับ PostgreSQL connection pool settings
   3. เพิ่ม Database read replicas
   4. ใช้ CDN สำหรับ static assets
   5. ปรับ Next.js ISR settings
```

---

## ❌ Common Errors & Solutions

### Error 1: Connection Pool Exhaustion

```
FATAL: remaining connection slots are reserved for 
non-replication superuser connections
```

**วิธีแก้:** ปรับ PostgreSQL และ PgBouncer settings

```bash
# แก้ไข /etc/postgresql/17/main/postgresql.conf
max_connections = 500          # เพิ่มจาก default 100

# ติดตั้งและ config PgBouncer
sudo apt-get install pgbouncer -y

# /etc/pgbouncer/pgbouncer.ini
cat > /etc/pgbouncer/pgbouncer.ini << 'EOF'
[databases]
chuaikan = host=localhost port=5432 dbname=chuaikan

[pgbouncer]
listen_port = 6432
listen_addr = localhost
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 5000
default_pool_size = 100
min_pool_size = 10
server_reset_query = DISCARD ALL
EOF

sudo systemctl restart pgbouncer
```

### Error 2: Redis Connection Timeout

```
Error: Redis connection to localhost:6379 failed - connect ETIMEDOUT
```

**วิธีแก้:** ปรับ Redis และ connection settings

```bash
# แก้ไข /etc/redis/redis.conf
timeout 0
tcp-keepalive 300
maxmemory 4gb
maxmemory-policy allkeys-lru
tcp-backlog 511

sudo systemctl restart redis

# ใน Node.js application
const redis = new Redis({
  host: 'localhost',
  port: 6379,
  retryDelayOnFailover: 100,
  enableReadyCheck: true,
  maxRetriesPerRequest: 3,
  connectTimeout: 5000,
  commandTimeout: 3000,
  lazyConnect: false,
  // Connection pool
  connectionName: 'chuaikan-app',
  db: 0,
});
```

### Error 3: k6 Script Memory Issue

```
ERRO[0045] GoError: runtime error: index out of range
```

**วิธีแก้:** จัดการ SharedArray ให้ถูกต้อง

```javascript
// แทนที่
const users = JSON.parse(open('./data/users.json'));  // ❌ โหลดซ้ำทุก VU

// ใช้ SharedArray แทน
const users = new SharedArray('users', function() {  // ✓ โหลดครั้งเดียว
  return JSON.parse(open('./data/users.json'));
});
```

---

## ✅ Checklist

### Pre-Load Test Checklist

- [ ] สร้าง test users ใน database (อย่างน้อย 1,000 accounts)
- [ ] ตรวจสอบว่า staging environment มี data เพียงพอ (seed data)
- [ ] ตั้งค่า monitoring: Grafana + InfluxDB
- [ ] ตรวจสอบ server resources: CPU, RAM, Disk ก่อนเริ่ม test
- [ ] ตรวจสอบ database connection pool settings
- [ ] ตรวจสอบ Redis connection limits
- [ ] เตรียม rollback plan หาก test ทำให้ระบบล่ม
- [ ] แจ้งทีมว่ากำลัง load test (ไม่ใช่ production outage)
- [ ] Set up alerting ใน Grafana (CPU > 80%, Error rate > 5%)

### During Load Test Checklist

- [ ] Monitor CPU usage (target: <70%)
- [ ] Monitor Memory usage (target: <80%)
- [ ] Monitor Database connections (target: <80% of max)
- [ ] Monitor Redis memory (target: <80%)
- [ ] Monitor error rate (target: <1%)
- [ ] Monitor response time p95 (target: <500ms)

### Post-Load Test Checklist

- [ ] Export Grafana dashboard screenshots
- [ ] Export k6 summary JSON: `k6 run --out json=results.json ...`
- [ ] วิเคราะห์ slow queries ใน PostgreSQL logs
- [ ] ดู Redis slow log: `redis-cli SLOWLOG GET 10`
- [ ] บันทึก findings และ action items
- [ ] สร้าง ticket สำหรับแก้ไข bottlenecks ที่พบ

---

## 🔗 References

- [k6 Documentation](https://k6.io/docs/)
- [Artillery Documentation](https://www.artillery.io/docs)
- [k6 InfluxDB Output](https://k6.io/docs/results-output/real-time/influxdb/)
- [Grafana k6 Dashboard](https://grafana.com/grafana/dashboards/2587-k6-load-testing-results/)
- [PostgreSQL Performance Tuning](https://www.postgresql.org/docs/17/performance-tips.html)

---

*Part 011 | Road to 1,000,000 Users/Day | chuaikan.com*
