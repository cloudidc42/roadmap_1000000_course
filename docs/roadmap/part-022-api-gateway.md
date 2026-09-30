# Part 022: Service Discovery & API Gateway
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 211-220
> **เวลาโดยประมาณ:** 6 ชั่วโมง
> **Prerequisites:** Part 021 (Microservices Overview, Docker Compose setup)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ทำความเข้าใจ API Gateway patterns และประโยชน์สำหรับ chuaikan.com
- ติดตั้งและตั้งค่า Kong Gateway บน Ubuntu 24.04 LTS
- กำหนด routes, plugins สำหรับทุก service
- จัดการ rate limiting, JWT validation, CORS ผ่าน Kong
- ทำความเข้าใจ Service Discovery ด้วย Consul
- ตั้งค่า Health Checking และ Circuit Breaker
- Monitor Kong ด้วย Prometheus + Grafana

---

## 📖 ทฤษฎีและแนวคิด

### Step 211: API Gateway คืออะไร?

**API Gateway** คือ reverse proxy ที่ทำหน้าที่เป็น single entry point สำหรับ clients ทุกตัว แทนที่ client จะต้องรู้ IP/Port ของแต่ละ service

```
Without API Gateway (ปัญหา):
Client → Post Service :3002 (ต้องรู้ IP:Port ตรงๆ)
Client → Auth Service :3001 (ต้องจัดการ JWT เอง)
Client → Media Service :3005 (ต้อง configure CORS เอง)

With API Gateway (แก้ปัญหา):
Client → Kong :8000 → /api/posts → Post Service :3002
                    → /api/auth  → Auth Service :3001
                    → /api/media → Media Service :3005
       (Kong จัดการ rate limit, JWT, CORS ให้ทุก service)
```

**API Gateway Patterns:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    Pattern 1: Routing                           │
│  Client ──► Gateway ──► Service A (based on path/header)       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    Pattern 2: Aggregation                       │
│  Client ──► Gateway ──► Call Service A                         │
│                      ──► Call Service B                         │
│                      ◄── Merge responses                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    Pattern 3: Backend for Frontend (BFF)        │
│  Mobile App ──► Mobile Gateway ──► optimized for mobile        │
│  Web App    ──► Web Gateway    ──► optimized for web            │
└─────────────────────────────────────────────────────────────────┘
```

### Step 212: Kong Architecture Diagram

```
chuaikan.com Kong Gateway Architecture
========================================

Internet Traffic
      │
      ▼
┌─────────────────────────────────────────────┐
│              Kong Gateway :8000              │
│                                             │
│  Plugins Pipeline (per request):            │
│  1. IP Restriction Check                    │
│  2. Rate Limiting (by IP/User)              │
│  3. JWT Authentication                      │
│  4. Request Size Limiting                   │
│  5. CORS Headers                            │
│  6. Request Logging                         │
│  7. Response Transform                      │
└───────────────────┬─────────────────────────┘
                    │ Route matching
        ┌───────────┼────────────────┐
        │           │                │
        ▼           ▼                ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Auth Service │ │ Post Service │ │ SOS Service  │
│    :3001     │ │    :3002     │ │    :3003     │
└──────────────┘ └──────────────┘ └──────────────┘
        │           │                │
        └───────────┴────────────────┘
                    │
                    ▼
        ┌──────────────────────┐
        │   Kong Admin API     │
        │       :8001          │
        │  (Management Only)   │
        └──────────────────────┘
                    │
                    ▼
        ┌──────────────────────┐
        │     Prometheus       │
        │   Kong Metrics       │
        └──────────────────────┘


Kong DB-less Mode: config อยู่ใน kong.yml (declarative)
ไม่ต้องใช้ database ทำให้ simpler และ immutable infrastructure
```

### Step 213: Service Discovery ด้วย Consul

**Service Discovery** แก้ปัญหา "service อยู่ที่ IP อะไร port อะไร?"

```
Problem: Services เปลี่ยน IP เมื่อ scale/restart

Solution - DNS-based Service Discovery:
┌────────────────────────────────────────────┐
│               Consul Server                │
│  Service Registry:                         │
│  auth-service    → 10.0.0.1:3001          │
│  post-service    → 10.0.0.2:3002          │
│  post-service    → 10.0.0.3:3002  (x2)   │
│  media-service   → 10.0.0.4:3005          │
└────────────────────────────────────────────┘
           │              │
           ▼              ▼
   New post-service   Kong Gateway
   registers itself   queries DNS:
   on startup         post-service.service.consul
```

---

## ⚙️ Environment Setup

### Step 214: ติดตั้ง Kong บน Ubuntu 24.04 LTS (Docker)

```bash
# Ubuntu 24.04 LTS - Docker installation
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
newgrp docker

# Verify
docker --version  # Docker version 26.x.x

# Pull Kong image
docker pull kong:3.7-ubuntu

# สร้าง directory สำหรับ Kong config
mkdir -p ~/chuaikan-platform/infrastructure/kong
cd ~/chuaikan-platform/infrastructure/kong

# Test Kong container
docker run --rm kong:3.7-ubuntu kong version
# Expected: Kong: 3.7.x
```

### Step 215: Kong Declarative Configuration

```bash
# สร้าง Kong config directory และ files
mkdir -p ~/chuaikan-platform/infrastructure/kong
cd ~/chuaikan-platform/infrastructure/kong
```

```yaml
# infrastructure/kong/kong.yml
_format_version: "3.0"
_transform: true

# Global plugins (apply to all services)
plugins:
  - name: prometheus
    config:
      status_code_metrics: true
      latency_metrics: true
      bandwidth_metrics: true
      upstream_health_metrics: true

  - name: request-transformer
    config:
      add:
        headers:
          - "X-Gateway: kong"
          - "X-Request-ID: $(uuid)"

# Services (upstream servers)
services:
  - name: auth-service
    url: http://auth-service:3001
    connect_timeout: 5000
    write_timeout: 10000
    read_timeout: 10000
    retries: 3
    tags:
      - chuaikan
      - auth

  - name: post-service
    url: http://post-service:3002
    connect_timeout: 5000
    write_timeout: 30000
    read_timeout: 30000
    retries: 3
    tags:
      - chuaikan
      - posts

  - name: sos-service
    url: http://sos-service:3003
    connect_timeout: 3000    # SOS ต้องการ latency ต่ำ
    write_timeout: 10000
    read_timeout: 10000
    retries: 5               # SOS retry มากกว่าปกติ
    tags:
      - chuaikan
      - sos
      - critical

  - name: notification-service
    url: http://notification-service:3004
    connect_timeout: 5000
    write_timeout: 15000
    read_timeout: 15000
    retries: 3
    tags:
      - chuaikan
      - notifications

  - name: media-service
    url: http://media-service:3005
    connect_timeout: 5000
    write_timeout: 120000    # Media upload ใช้เวลานาน
    read_timeout: 120000
    retries: 2
    tags:
      - chuaikan
      - media

# Routes (how to match incoming requests)
routes:
  # Auth routes (public - ไม่ต้อง JWT)
  - name: auth-login
    service: auth-service
    paths:
      - /api/v1/auth/login
      - /api/v1/auth/register
      - /api/v1/auth/refresh
      - /api/v1/auth/forgot-password
      - /api/v1/auth/reset-password
      - /api/v1/auth/verify-email
    methods:
      - POST
      - GET
    strip_path: false

  # Auth routes (OAuth callbacks)
  - name: auth-oauth
    service: auth-service
    paths:
      - /api/v1/auth/line
      - /api/v1/auth/google
    methods:
      - GET
      - POST
    strip_path: false

  # Protected auth routes
  - name: auth-protected
    service: auth-service
    paths:
      - /api/v1/auth/me
      - /api/v1/auth/logout
      - /api/v1/auth/devices
    methods:
      - GET
      - POST
      - DELETE
    strip_path: false
    plugins:
      - name: jwt
        config:
          secret_is_base64: false
          claims_to_verify:
            - exp
          key_claim_name: kid

  # Post routes
  - name: posts-public
    service: post-service
    paths:
      - /api/v1/posts
      - /api/v1/users
    methods:
      - GET
    strip_path: false

  - name: posts-protected
    service: post-service
    paths:
      - /api/v1/posts
      - /api/v1/comments
      - /api/v1/likes
      - /api/v1/feed
    methods:
      - POST
      - PUT
      - PATCH
      - DELETE
    strip_path: false
    plugins:
      - name: jwt
        config:
          secret_is_base64: false

  # SOS routes (critical - separate rate limits)
  - name: sos-routes
    service: sos-service
    paths:
      - /api/v1/sos
    methods:
      - GET
      - POST
      - PUT
    strip_path: false
    plugins:
      - name: jwt
        config:
          secret_is_base64: false
      - name: rate-limiting
        config:
          minute: 20       # SOS อนุญาต trigger บ่อยกว่า
          hour: 100
          policy: local
          hide_client_headers: false

  # Notification routes
  - name: notification-routes
    service: notification-service
    paths:
      - /api/v1/notifications
    methods:
      - GET
      - POST
      - PATCH
    strip_path: false
    plugins:
      - name: jwt
        config:
          secret_is_base64: false

  # Media routes
  - name: media-upload
    service: media-service
    paths:
      - /api/v1/media
    methods:
      - GET
      - POST
      - DELETE
    strip_path: false
    plugins:
      - name: jwt
        config:
          secret_is_base64: false
      - name: request-size-limiting
        config:
          allowed_payload_size: 50     # 50MB max upload
          size_unit: megabytes
          require_content_length: false

# Consumers (users ที่ใช้ API)
consumers:
  - username: mobile-app-ios
    tags:
      - mobile
      - ios
  - username: mobile-app-android
    tags:
      - mobile
      - android
  - username: web-app
    tags:
      - web

# Global plugins ที่ apply ทุก route
plugins:
  - name: cors
    config:
      origins:
        - https://chuaikan.com
        - https://www.chuaikan.com
        - https://app.chuaikan.com
        - http://localhost:3000     # dev only
      methods:
        - GET
        - POST
        - PUT
        - PATCH
        - DELETE
        - OPTIONS
      headers:
        - Authorization
        - Content-Type
        - X-Request-ID
        - X-API-Version
      exposed_headers:
        - X-Request-ID
        - X-RateLimit-Remaining
      credentials: true
      max_age: 3600
      preflight_continue: false

  - name: rate-limiting
    config:
      minute: 60         # 60 requests/minute global default
      hour: 1000         # 1000 requests/hour
      day: 10000         # 10000 requests/day
      policy: local      # ใช้ redis ใน production
      limit_by: consumer  # rate limit per authenticated user
      fault_tolerant: true
      hide_client_headers: false
      error_code: 429
      error_message: "Rate limit exceeded. Please slow down."

  - name: ip-restriction
    config:
      allow:
        - 0.0.0.0/0      # Allow all in dev; restrict in production
      deny: []

  - name: response-ratelimiting
    config:
      limits:
        video_upload:
          minute: 5      # Max 5 video uploads/minute

# Upstreams (for load balancing)
upstreams:
  - name: post-service-upstream
    algorithm: round-robin
    healthchecks:
      active:
        type: http
        http_path: /health
        interval: 10
        healthy:
          successes: 2
        unhealthy:
          http_failures: 3
          interval: 5
    targets:
      - target: post-service-1:3002
        weight: 100
      - target: post-service-2:3002
        weight: 100
```

---

## 🛠️ Step-by-Step Implementation

### Step 216: Kong Admin API Usage

```bash
# Start Kong (DB-less mode)
docker run -d --name kong \
  -e KONG_DATABASE=off \
  -e KONG_DECLARATIVE_CONFIG=/kong/kong.yml \
  -e KONG_PROXY_ACCESS_LOG=/dev/stdout \
  -e KONG_ADMIN_ACCESS_LOG=/dev/stdout \
  -e KONG_PROXY_ERROR_LOG=/dev/stderr \
  -e KONG_ADMIN_ERROR_LOG=/dev/stderr \
  -e KONG_ADMIN_LISTEN="0.0.0.0:8001" \
  -v $(pwd)/infrastructure/kong:/kong \
  -p 8000:8000 \
  -p 8001:8001 \
  kong:3.7-ubuntu

# ตรวจสอบ Kong status
curl -s http://localhost:8001/ | jq '.version'

# ดู services ทั้งหมด
curl -s http://localhost:8001/services | jq '.data[].name'

# ดู routes
curl -s http://localhost:8001/routes | jq '.data[] | {name: .name, paths: .paths}'

# ดู plugins
curl -s http://localhost:8001/plugins | jq '.data[] | {name: .name, service: .service}'

# ทดสอบ rate limiting
for i in {1..70}; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8000/api/v1/posts)
  echo "Request $i: HTTP $STATUS"
done

# คาดหวัง: requests 61-70 ได้รับ HTTP 429

# ดู rate limit headers
curl -v http://localhost:8000/api/v1/posts 2>&1 | grep -i "ratelimit"
# X-RateLimit-Limit-Minute: 60
# X-RateLimit-Remaining-Minute: 59

# เพิ่ม consumer
curl -s -X POST http://localhost:8001/consumers \
  -H "Content-Type: application/json" \
  -d '{"username": "test-user-123", "tags": ["mobile", "ios"]}' | jq

# เพิ่ม JWT credential สำหรับ consumer
curl -s -X POST http://localhost:8001/consumers/test-user-123/jwt \
  -H "Content-Type: application/json" \
  -d '{"algorithm": "RS256", "key": "user-123"}' | jq

# Live reload config (DB-less mode)
curl -s -X POST http://localhost:8001/config \
  -F config=@./infrastructure/kong/kong.yml | jq '.services | length'
```

### Step 217: Kong deck (Declarative Config Management)

```bash
# ติดตั้ง deck (Kong's declarative config tool)
curl -sL https://github.com/kong/deck/releases/download/v1.38.0/deck_1.38.0_linux_amd64.tar.gz | tar xz
sudo mv deck /usr/local/bin/
deck version

# Validate config ก่อน apply
deck gateway validate ./infrastructure/kong/kong.yml

# Sync config ไปยัง Kong
deck gateway sync ./infrastructure/kong/kong.yml \
  --kong-addr http://localhost:8001

# Diff - ดูว่ามีอะไรเปลี่ยนแปลงบ้าง
deck gateway diff ./infrastructure/kong/kong.yml \
  --kong-addr http://localhost:8001

# Dump current config เป็น file
deck gateway dump --kong-addr http://localhost:8001 \
  -o ./infrastructure/kong/kong-backup.yml

# Reset Kong config
deck gateway reset --kong-addr http://localhost:8001 --force
```

### Step 218: Circuit Breaker ใน Kong

```bash
# Kong ใช้ passive health checks เป็น circuit breaker
# เมื่อ upstream failures เกิน threshold → Kong หยุดส่ง traffic ไป

# ตั้งค่า circuit breaker ผ่าน Admin API
curl -X POST http://localhost:8001/upstreams \
  -H "Content-Type: application/json" \
  -d '{
    "name": "post-service-cb",
    "healthchecks": {
      "active": {
        "type": "http",
        "http_path": "/health",
        "interval": 10,
        "healthy": {
          "successes": 2,
          "interval": 5
        },
        "unhealthy": {
          "http_failures": 3,
          "interval": 5,
          "tcp_failures": 2
        }
      },
      "passive": {
        "type": "http",
        "healthy": {
          "successes": 5,
          "http_statuses": [200, 201, 202, 204]
        },
        "unhealthy": {
          "http_failures": 5,
          "http_statuses": [429, 500, 502, 503, 504],
          "tcp_failures": 2
        }
      }
    }
  }'

# ตรวจสอบ health status ของ upstream
curl -s http://localhost:8001/upstreams/post-service-cb/health | jq
```

### Step 219: Load Balancing Algorithms

```yaml
# infrastructure/kong/kong.yml - Load balancing strategies

upstreams:
  # Round Robin (default) - ส่ง request แบบ rotate
  - name: post-service-round-robin
    algorithm: round-robin
    targets:
      - target: post-service-1:3002
        weight: 100
      - target: post-service-2:3002
        weight: 100

  # Least Connections - ส่งไป server ที่มี connections น้อยที่สุด
  - name: post-service-least-conn
    algorithm: least-connections
    targets:
      - target: post-service-1:3002
        weight: 100
      - target: post-service-2:3002
        weight: 100

  # Consistent Hashing - ส่ง request จาก same user ไป same server (sticky session)
  - name: post-service-consistent-hash
    algorithm: consistent-hashing
    hash_on: header
    hash_on_header: X-User-ID    # route based on user ID
    targets:
      - target: post-service-1:3002
        weight: 100
      - target: post-service-2:3002
        weight: 100

  # Weighted - ส่ง traffic ตาม weight (canary deployment)
  - name: post-service-weighted
    algorithm: round-robin
    targets:
      - target: post-service-v1:3002
        weight: 90    # 90% traffic to v1
      - target: post-service-v2:3002
        weight: 10    # 10% traffic to v2 (canary)
```

### Step 220: Monitoring Kong ด้วย Prometheus

```yaml
# infrastructure/docker-compose.monitoring.yml
version: '3.9'

services:
  prometheus:
    image: prom/prometheus:v2.53.0
    container_name: chuaikan-prometheus
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'

  grafana:
    image: grafana/grafana:11.1.0
    container_name: chuaikan-grafana
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin123
      GF_USERS_ALLOW_SIGN_UP: "false"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources
    ports:
      - "3000:3000"
    depends_on:
      - prometheus

volumes:
  prometheus-data:
  grafana-data:
```

```yaml
# infrastructure/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  # Kong metrics
  - job_name: 'kong'
    static_configs:
      - targets: ['kong:8001']
    metrics_path: '/metrics'

  # Node exporter (server metrics)
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  # Application services
  - job_name: 'auth-service'
    static_configs:
      - targets: ['auth-service:3001']
    metrics_path: '/metrics'

  - job_name: 'post-service'
    static_configs:
      - targets: ['post-service:3002']
    metrics_path: '/metrics'
```

```javascript
// Prometheus metrics ใน Express.js services
// apps/post-service/src/middleware/metrics.ts

import { Registry, Counter, Histogram, Gauge, collectDefaultMetrics } from 'prom-client';

const register = new Registry();
collectDefaultMetrics({ register });

export const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code', 'service'],
  registers: [register],
});

export const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'service'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5],
  registers: [register],
});

export const activeConnections = new Gauge({
  name: 'active_connections',
  help: 'Number of active connections',
  labelNames: ['service'],
  registers: [register],
});

export function metricsMiddleware(serviceName: string) {
  return (req: any, res: any, next: any) => {
    const start = Date.now();
    activeConnections.labels(serviceName).inc();

    res.on('finish', () => {
      const duration = (Date.now() - start) / 1000;
      const route = req.route?.path || req.path;

      httpRequestsTotal.labels(req.method, route, String(res.statusCode), serviceName).inc();
      httpRequestDuration.labels(req.method, route, serviceName).observe(duration);
      activeConnections.labels(serviceName).dec();
    });

    next();
  };
}

export function metricsEndpoint() {
  return async (req: any, res: any) => {
    res.set('Content-Type', register.contentType);
    res.send(await register.metrics());
  };
}
```

---

## 🔧 Configuration Files

### Nginx เป็น Simple API Gateway (Alternative)

```nginx
# infrastructure/nginx/nginx.conf
# Nginx เป็น lightweight alternative ถ้าไม่อยาก run Kong

events {
    worker_connections 1024;
}

http {
    # Upstream definitions
    upstream auth_service {
        least_conn;
        server auth-service:3001;
        keepalive 32;
    }

    upstream post_service {
        least_conn;
        server post-service:3002;
        server post-service-2:3002 backup;
        keepalive 32;
    }

    upstream media_service {
        server media-service:3005;
        keepalive 16;
    }

    # Rate limiting zones
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=60r/m;
    limit_req_zone $binary_remote_addr zone=login_limit:10m rate=5r/m;
    limit_req_zone $binary_remote_addr zone=upload_limit:10m rate=10r/m;

    # Main server block
    server {
        listen 80;
        server_name api.chuaikan.com;

        # Security headers
        add_header X-Frame-Options DENY;
        add_header X-Content-Type-Options nosniff;
        add_header X-XSS-Protection "1; mode=block";
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";

        # CORS headers
        add_header Access-Control-Allow-Origin $http_origin always;
        add_header Access-Control-Allow-Methods "GET, POST, PUT, PATCH, DELETE, OPTIONS" always;
        add_header Access-Control-Allow-Headers "Authorization, Content-Type, X-Request-ID" always;
        add_header Access-Control-Allow-Credentials true always;

        # Handle preflight
        if ($request_method = OPTIONS) {
            return 204;
        }

        # Auth service routes (public)
        location ~ ^/api/v1/auth/(login|register|refresh|forgot-password|reset-password) {
            limit_req zone=login_limit burst=3 nodelay;
            proxy_pass http://auth_service;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        # Post service routes
        location /api/v1/posts {
            limit_req zone=api_limit burst=20 nodelay;
            proxy_pass http://post_service;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_read_timeout 30s;
        }

        # Media upload routes
        location /api/v1/media {
            limit_req zone=upload_limit burst=5 nodelay;
            client_max_body_size 50M;
            proxy_pass http://media_service;
            proxy_set_header Host $host;
            proxy_read_timeout 120s;
            proxy_send_timeout 120s;
        }

        # Health check endpoint
        location /health {
            return 200 '{"status":"ok","gateway":"nginx"}';
            add_header Content-Type application/json;
        }
    }
}
```

### Consul Service Discovery Setup

```bash
# ติดตั้ง Consul
docker pull consul:1.19

# Run Consul server (development mode)
docker run -d \
  --name=consul-server \
  -p 8500:8500 \
  -p 8600:8600/udp \
  consul:1.19 \
  agent -server -bootstrap -ui -client=0.0.0.0

# Register service manually (via API)
curl -X PUT http://localhost:8500/v1/agent/service/register \
  -H "Content-Type: application/json" \
  -d '{
    "ID": "post-service-1",
    "Name": "post-service",
    "Address": "10.0.0.2",
    "Port": 3002,
    "Tags": ["v1", "production"],
    "Check": {
      "HTTP": "http://10.0.0.2:3002/health",
      "Interval": "10s",
      "Timeout": "5s",
      "DeregisterCriticalServiceAfter": "30s"
    }
  }'

# Query services
curl -s http://localhost:8500/v1/catalog/service/post-service | jq

# DNS query (services register as <name>.service.consul)
dig @127.0.0.1 -p 8600 post-service.service.consul SRV
```

```typescript
// apps/post-service/src/consul-registration.ts
// Auto-registration เมื่อ service เริ่มทำงาน

import Consul from 'consul';
import { env } from './config';

const consul = new Consul({
  host: env.CONSUL_HOST || 'consul-server',
  port: 8500,
});

const SERVICE_ID = `post-service-${process.env.HOSTNAME || 'local'}-${env.PORT}`;

export async function registerService(): Promise<void> {
  await consul.agent.service.register({
    id: SERVICE_ID,
    name: 'post-service',
    address: env.SERVICE_HOST,
    port: env.PORT,
    tags: [`v${env.API_VERSION}`, env.NODE_ENV],
    check: {
      http: `http://${env.SERVICE_HOST}:${env.PORT}/health`,
      interval: '10s',
      timeout: '5s',
      deregistercriticalserviceafter: '30s',
    },
  });

  console.log(`Service registered with Consul: ${SERVICE_ID}`);
}

export async function deregisterService(): Promise<void> {
  await consul.agent.service.deregister(SERVICE_ID);
  console.log(`Service deregistered from Consul: ${SERVICE_ID}`);
}

// Register on startup, deregister on shutdown
process.on('SIGTERM', async () => {
  await deregisterService();
  process.exit(0);
});
```

---

## 🧪 Testing

### Kong Integration Tests

```bash
#!/bin/bash
# scripts/test-kong.sh - Kong gateway integration tests

BASE_URL="http://localhost:8000"
PASS=0
FAIL=0

test_case() {
  local name=$1
  local expected=$2
  local actual=$3
  
  if [ "$actual" = "$expected" ]; then
    echo "✓ PASS: $name"
    ((PASS++))
  else
    echo "✗ FAIL: $name (expected: $expected, got: $actual)"
    ((FAIL++))
  fi
}

echo "=== Kong Gateway Tests ==="

# Test 1: Health check
STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$BASE_URL/health")
test_case "Health check returns 200" "200" "$STATUS"

# Test 2: CORS headers present
CORS=$(curl -s -I -X OPTIONS "$BASE_URL/api/v1/posts" \
  -H "Origin: https://chuaikan.com" | grep -c "Access-Control-Allow-Origin")
test_case "CORS headers present" "1" "$CORS"

# Test 3: Rate limiting headers present
RATE_HEADER=$(curl -s -I "$BASE_URL/api/v1/posts" | grep -c "X-RateLimit")
test_case "Rate limit headers present" "1" "$RATE_HEADER"

# Test 4: Protected route requires auth
AUTH_STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
  -X POST "$BASE_URL/api/v1/posts")
test_case "Protected POST /posts requires auth (401)" "401" "$AUTH_STATUS"

# Test 5: Request size limit
LARGE_PAYLOAD=$(python3 -c "print('x' * 60000000)")  # 60MB
SIZE_STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
  -X POST "$BASE_URL/api/v1/media" \
  -H "Content-Type: application/json" \
  -d "{\"data\": \"$LARGE_PAYLOAD\"}")
test_case "Large payload rejected (413)" "413" "$SIZE_STATUS"

# Test 6: Kong admin not exposed to public
ADMIN_STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
  "http://localhost:8001/services" 2>/dev/null || echo "000")
# Kong admin port should not be accessible from outside (only 000 = connection refused)

echo ""
echo "=== Results: $PASS passed, $FAIL failed ==="
```

### Load Testing Kong

```bash
# ติดตั้ง k6 สำหรับ load testing
sudo gpg -k
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
  --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | \
  sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update && sudo apt-get install k6

# สร้าง k6 test script
cat > /tmp/kong-load-test.js << 'EOF'
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  stages: [
    { duration: '30s', target: 50 },   // Ramp up
    { duration: '60s', target: 100 },  // Stay at 100 users
    { duration: '30s', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],   // 95% requests < 500ms
    http_req_failed: ['rate<0.01'],     // Error rate < 1%
  },
};

export default function () {
  const res = http.get('http://localhost:8000/api/v1/posts?page=1&limit=20');
  
  check(res, {
    'status is 200 or 401': (r) => r.status === 200 || r.status === 401,
    'response time < 500ms': (r) => r.timings.duration < 500,
    'has request ID header': (r) => r.headers['X-Request-Id'] !== undefined,
  });
  
  sleep(1);
}
EOF

# Run load test
k6 run /tmp/kong-load-test.js
```

---

## ❌ Common Errors & Solutions

### Error 1: "No route matched" - 404 จาก Kong

```bash
# Debug: ดู routes ที่ลงทะเบียนไว้
curl -s http://localhost:8001/routes | jq '.data[] | {name, paths, methods}'

# ตรวจสอบว่า request path ตรงกับ route ไหม
curl -v http://localhost:8000/api/v1/posts 2>&1 | grep "< HTTP"

# แก้ไข: ตรวจสอบ path ใน kong.yml ให้ถูกต้อง
# หมายเหตุ: Kong path matching เป็น prefix-based
# /api/v1/posts จะ match /api/v1/posts/123 ด้วย
```

### Error 2: JWT validation fails แม้ token ถูกต้อง

```bash
# ตรวจสอบ JWT key ที่ register กับ consumer
curl -s http://localhost:8001/consumers/test-user/jwt | jq

# ปัญหา: RS256 ต้องการ public key ไม่ใช่ secret
# แก้ไข: สร้าง JWT credential พร้อม rsa_public_key

curl -X POST http://localhost:8001/consumers/test-user/jwt \
  -H "Content-Type: application/json" \
  -d "{
    \"algorithm\": \"RS256\",
    \"key\": \"user-key-id\",
    \"rsa_public_key\": \"$(cat keys/public.pem)\"
  }"
```

### Error 3: CORS error ใน browser แม้ตั้งค่าแล้ว

```bash
# ตรวจสอบว่า CORS plugin active
curl -s http://localhost:8001/plugins | jq '.data[] | select(.name == "cors")'

# Kong CORS plugin ต้องตั้งค่า origins ให้ครบ
# ถ้าใช้ credentials: true ห้ามใช้ * เป็น origin
# ต้องระบุ origin ตรงๆ เช่น https://chuaikan.com

# Debug ด้วย curl (simulate browser)
curl -v -X OPTIONS http://localhost:8000/api/v1/posts \
  -H "Origin: https://chuaikan.com" \
  -H "Access-Control-Request-Method: POST" 2>&1 | grep -i "access-control"
```

### Error 4: Kong ไม่ start เมื่อ config ผิด

```bash
# ตรวจสอบ config syntax
docker run --rm \
  -v $(pwd)/infrastructure/kong:/kong \
  kong:3.7-ubuntu \
  kong config parse /kong/kong.yml

# ถ้า error: "schema violation"
# ตรวจสอบ _format_version ใน kong.yml ต้องเป็น "3.0"
# ตรวจสอบ indentation (YAML sensitive)
```

---

## ✅ Checklist

### Step 211-212: API Gateway Design
- [ ] เข้าใจประโยชน์ของ API Gateway
- [ ] วาด architecture diagram พร้อม Kong
- [ ] กำหนด services และ routes ทั้งหมด

### Step 213: Service Discovery
- [ ] เข้าใจ Consul service registry
- [ ] Consul container ขึ้นสำเร็จ
- [ ] ทดสอบ register/deregister service

### Step 214-215: Kong Setup
- [ ] ติดตั้ง Kong บน Docker สำเร็จ
- [ ] สร้าง kong.yml (declarative config)
- [ ] `deck gateway validate` ผ่าน

### Step 216: Kong Admin API
- [ ] ดู services ผ่าน Admin API ได้
- [ ] เพิ่ม consumer และ JWT credential ได้
- [ ] Reload config ด้วย deck sync ได้

### Step 217-218: Plugins & Circuit Breaker
- [ ] Rate limiting ทำงาน (429 เมื่อ exceed)
- [ ] CORS headers ถูกต้อง
- [ ] JWT plugin ปกป้อง routes ที่กำหนด
- [ ] Request size limiting ทำงาน
- [ ] Circuit breaker (passive health check) ตั้งค่าแล้ว

### Step 219: Load Balancing
- [ ] Round-robin load balancing ทำงาน
- [ ] Health check active ตั้งค่าแล้ว
- [ ] Weighted routing สำหรับ canary deployment พร้อม

### Step 220: Monitoring
- [ ] Prometheus container ขึ้นสำเร็จ
- [ ] Kong metrics scraping ทำงาน
- [ ] Grafana dashboard เห็น Kong metrics
- [ ] Load test ด้วย k6 ผ่าน threshold

---

## 🔗 References

- [Kong Gateway Documentation](https://docs.konghq.com/)
- [Kong DB-less Mode](https://docs.konghq.com/gateway/latest/production/deployment-topologies/db-less-and-declarative-config/)
- [deck CLI](https://docs.konghq.com/deck/)
- [Consul Documentation](https://developer.hashicorp.com/consul/docs)
- [Kong Prometheus Plugin](https://docs.konghq.com/hub/kong-inc/prometheus/)
- [k6 Load Testing](https://k6.io/docs/)

---
*Part 022 | Road to 1,000,000 Users/Day | chuaikan.com*
