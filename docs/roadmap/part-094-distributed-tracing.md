# Part 094: Distributed Tracing ด้วย Jaeger

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** World Class
> **Steps:** 931-940
> **เวลาโดยประมาณ:** 4 ชั่วโมง
> **Prerequisites:** Part 031 (Kubernetes), Part 021 (Microservices Overview)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

เมื่อมี Microservices หลาย service การ debug "request ช้า" กลายเป็นฝันร้าย Distributed Tracing แก้ปัญหานี้ด้วยการ track request ข้ามทุก service ใน Part นี้เราจะ:

- ติดตั้ง Jaeger บน Kubernetes
- Auto-instrument Node.js services ด้วย OpenTelemetry
- Trace request ผ่าน API Gateway → Feed Service → DB
- ตั้ง Sampling Strategy ที่เหมาะสม
- Correlate traces กับ logs

---

## 📖 ทฤษฎีและแนวคิด

### 1. OpenTelemetry Concepts

**Trace:** เส้นทางทั้งหมดของ request ตั้งแต่ต้นจนจบ

```
User Request (trace_id: abc123)
│
├── API Gateway (span_id: 001, duration: 250ms)
│   ├── Auth check (span_id: 002, duration: 5ms)
│   └── Route to Feed Service (span_id: 003, duration: 240ms)
│       ├── Redis GET feed:user:456 (span_id: 004, duration: 2ms) MISS
│       ├── PostgreSQL SELECT posts (span_id: 005, duration: 150ms) ← SLOW!
│       └── Redis SET feed:user:456 (span_id: 006, duration: 3ms)
└── Response → User (total: 250ms)
```

**Span:** หน่วยงานที่เล็กที่สุดใน trace แต่ละ span แทน operation หนึ่ง

```javascript
// Span attributes ที่ควรมี
{
  name: "PostgreSQL SELECT posts",
  trace_id: "abc123def456",
  span_id: "001002003",
  parent_span_id: "001002",
  start_time: 1704067200000,
  end_time: 1704067200150,
  duration_ms: 150,
  attributes: {
    "db.system": "postgresql",
    "db.name": "chuaikan",
    "db.statement": "SELECT id, content FROM posts WHERE user_id = $1",
    "db.rows_affected": 20,
    "user_id": "456"
  },
  status: "OK"
}
```

**Baggage:** key-value pairs ที่ propagate ข้าม service boundaries

```javascript
// Baggage ใช้สำหรับ:
// - user_id (เพื่อ filter traces ของ user เฉพาะคน)
// - feature_flag (เพื่อดูว่า flag ไหนทำให้ช้า)
// - region (เพื่อ debug multi-region issues)
```

### 2. W3C TraceContext Standard

```
traceparent header format:
00-{trace_id}-{parent_span_id}-{flags}

ตัวอย่าง:
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
              ^^  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^  ^^^^^^^^^^^^^^^^  ^^
              version  trace_id (128-bit hex)       parent_span_id   flags
```

---

## ⚙️ Environment Setup

### ติดตั้ง Jaeger บน Kubernetes

```bash
# ติดตั้ง Jaeger Operator
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update

helm install jaeger-operator jaegertracing/jaeger-operator \
  --namespace observability \
  --create-namespace \
  --set rbac.clusterRole=true
```

```yaml
# jaeger-production.yaml
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-production
  namespace: observability
spec:
  strategy: production
  
  collector:
    maxReplicas: 5
    resources:
      limits:
        cpu: 1
        memory: 1Gi
    
  storage:
    type: elasticsearch
    elasticsearch:
      serverUrls: https://elasticsearch:9200
      secretName: jaeger-es-secret
      indexPrefix: jaeger
    
  query:
    replicas: 2
    resources:
      limits:
        cpu: 500m
        memory: 512Mi
        
  ingress:
    enabled: true
    annotations:
      nginx.ingress.kubernetes.io/auth-type: basic
      nginx.ingress.kubernetes.io/auth-secret: jaeger-auth
    hosts:
      - jaeger.internal.chuaikan.com
```

```bash
# Apply Jaeger deployment
kubectl apply -f jaeger-production.yaml

# ตรวจสอบ
kubectl get jaeger -n observability
kubectl get pods -n observability
```

---

## 🛠️ Step-by-Step Implementation

### Step 931: Auto-Instrumentation ใน Node.js

```bash
# ติดตั้ง OpenTelemetry packages
npm install \
  @opentelemetry/sdk-node \
  @opentelemetry/auto-instrumentations-node \
  @opentelemetry/exporter-jaeger \
  @opentelemetry/exporter-otlp-http \
  @opentelemetry/resources \
  @opentelemetry/semantic-conventions
```

```javascript
// tracing.js — ต้อง require ก่อน app code ทั้งหมด!
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-otlp-http');
const { Resource } = require('@opentelemetry/resources');
const { SemanticResourceAttributes } = require('@opentelemetry/semantic-conventions');

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: process.env.SERVICE_NAME || 'api-service',
    [SemanticResourceAttributes.SERVICE_VERSION]: process.env.APP_VERSION || '1.0.0',
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV || 'production',
    'chuaikan.region': process.env.AWS_REGION || 'ap-southeast-1',
  }),
  
  // ส่ง traces ไป Jaeger Collector
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT || 
         'http://jaeger-collector.observability:4318/v1/traces',
  }),
  
  // Auto-instrument ทุกอย่าง: HTTP, Express, pg, redis, mongoose, kafka
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-http': {
        ignoreIncomingPaths: ['/health', '/metrics'],
      },
      '@opentelemetry/instrumentation-express': {
        enabled: true,
      },
      '@opentelemetry/instrumentation-pg': {
        enhancedDatabaseReporting: true,
      },
      '@opentelemetry/instrumentation-ioredis': {
        enabled: true,
      },
    }),
  ],
});

// Start SDK
sdk.start();

// Graceful shutdown
process.on('SIGTERM', () => {
  sdk.shutdown()
    .then(() => process.exit(0))
    .catch((error) => {
      console.error('Error shutting down OpenTelemetry SDK', error);
      process.exit(1);
    });
});

module.exports = sdk;
```

```javascript
// server.js — require tracing FIRST
require('./tracing');  // ← ต้องเป็น line แรก!

const express = require('express');
const app = express();
// ... rest of app
```

```yaml
# kubernetes deployment — inject OTEL config via env vars
apiVersion: apps/v1
kind: Deployment
metadata:
  name: feed-service
spec:
  template:
    spec:
      containers:
      - name: feed-service
        image: chuaikan/feed-service:latest
        env:
        - name: SERVICE_NAME
          value: "feed-service"
        - name: OTEL_EXPORTER_OTLP_ENDPOINT
          value: "http://jaeger-collector.observability:4318"
        - name: OTEL_TRACES_SAMPLER
          value: "parentbased_traceidratio"
        - name: OTEL_TRACES_SAMPLER_ARG
          value: "0.01"  # Sample 1% ของ normal traffic
        - name: NODE_OPTIONS
          value: "--require ./tracing.js"
```

### Step 932: Custom Spans สำหรับ Business Logic

```javascript
// feed-service.js
const { trace, context, SpanStatusCode } = require('@opentelemetry/api');
const tracer = trace.getTracer('feed-service', '1.0.0');

async function getUserFeed(userId, options = {}) {
  // สร้าง custom span สำหรับ business operation
  return tracer.startActiveSpan('get_user_feed', async (span) => {
    try {
      span.setAttributes({
        'user.id': userId,
        'feed.page': options.page || 1,
        'feed.limit': options.limit || 20,
      });
      
      // Check cache
      const cached = await tracer.startActiveSpan('redis.get_feed', async (cacheSpan) => {
        try {
          const result = await redis.get(`feed:${userId}`);
          cacheSpan.setAttributes({
            'cache.hit': result !== null,
            'cache.key': `feed:${userId}`,
          });
          return result;
        } finally {
          cacheSpan.end();
        }
      });
      
      if (cached) {
        span.setAttributes({ 'feed.source': 'cache' });
        span.setStatus({ code: SpanStatusCode.OK });
        return JSON.parse(cached);
      }
      
      // Query DB
      const feed = await tracer.startActiveSpan('db.get_feed', async (dbSpan) => {
        try {
          const result = await db.query(
            'SELECT id, content, created_at FROM posts WHERE user_id = ANY($1) ORDER BY created_at DESC LIMIT $2',
            [followingIds, options.limit || 20]
          );
          dbSpan.setAttributes({
            'db.rows_returned': result.rows.length,
          });
          return result.rows;
        } catch (err) {
          dbSpan.setStatus({
            code: SpanStatusCode.ERROR,
            message: err.message,
          });
          throw err;
        } finally {
          dbSpan.end();
        }
      });
      
      // Cache result
      await redis.setex(`feed:${userId}`, 60, JSON.stringify(feed));
      
      span.setAttributes({ 'feed.source': 'database', 'feed.count': feed.length });
      span.setStatus({ code: SpanStatusCode.OK });
      return feed;
      
    } catch (error) {
      span.setStatus({
        code: SpanStatusCode.ERROR,
        message: error.message,
      });
      span.recordException(error);
      throw error;
    } finally {
      span.end();
    }
  });
}
```

### Step 933: Trace Propagation ข้าม Services

```javascript
// middleware/trace-propagation.js
const { propagation, context, trace } = require('@opentelemetry/api');

// Middleware สำหรับ extract trace context จาก incoming requests
function traceMiddleware(req, res, next) {
  // Extract W3C TraceContext จาก headers
  const activeContext = propagation.extract(context.active(), req.headers);
  
  // Add span attributes จาก request
  const span = trace.getActiveSpan();
  if (span) {
    span.setAttributes({
      'http.method': req.method,
      'http.url': req.url,
      'http.user_agent': req.headers['user-agent'],
      'user.id': req.user?.id,
    });
  }
  
  // Continue with extracted context
  context.with(activeContext, next);
}

module.exports = traceMiddleware;
```

```javascript
// http-client.js — inject trace context ใน outgoing requests
const axios = require('axios');
const { propagation, context } = require('@opentelemetry/api');

function createTracedHttpClient() {
  const client = axios.create();
  
  // Inject trace headers ทุก outgoing request
  client.interceptors.request.use((config) => {
    const headers = {};
    propagation.inject(context.active(), headers);
    
    config.headers = {
      ...config.headers,
      ...headers,
    };
    
    return config;
  });
  
  return client;
}

module.exports = { createTracedHttpClient };
```

### Step 934: Sampling Strategies

```javascript
// sampling.js
const { ParentBasedSampler, TraceIdRatioBasedSampler } = require('@opentelemetry/sdk-trace-base');

// Custom Sampler: sample 100% สำหรับ SOS, 1% สำหรับ normal traffic
class ChuaikanSampler {
  shouldSample(context, traceId, spanName, spanKind, attributes) {
    // Sample 100% สำหรับ SOS operations
    if (spanName.includes('sos') || attributes?.['sos.critical'] === true) {
      return { decision: SamplingDecision.RECORD_AND_SAMPLED };
    }
    
    // Sample 100% สำหรับ errors
    if (attributes?.['http.status_code'] >= 400) {
      return { decision: SamplingDecision.RECORD_AND_SAMPLED };
    }
    
    // Sample 100% สำหรับ slow requests (> 500ms)
    // (ต้องใช้ tail sampling — Jaeger Collector supports this)
    
    // Sample 1% สำหรับ normal traffic
    const samplingRate = parseFloat(process.env.TRACE_SAMPLING_RATE || '0.01');
    const random = parseInt(traceId.substring(0, 8), 16) / 0xffffffff;
    
    if (random < samplingRate) {
      return { decision: SamplingDecision.RECORD_AND_SAMPLED };
    }
    
    return { decision: SamplingDecision.NOT_RECORD };
  }
  
  toString() {
    return 'ChuaikanSampler';
  }
}
```

```yaml
# jaeger-collector-sampling.yaml — Tail Sampling Policy
# Sample 100% ของ traces ที่มี error หรือ latency สูง

apiVersion: v1
kind: ConfigMap
metadata:
  name: jaeger-sampling-config
data:
  sampling.json: |
    {
      "default_strategy": {
        "type": "probabilistic",
        "param": 0.01
      },
      "per_operation_strategies": [
        {
          "operation": "POST /sos",
          "probabilistic": { "sampling_rate": 1.0 }
        },
        {
          "operation": "GET /health",
          "probabilistic": { "sampling_rate": 0.001 }
        }
      ]
    }
```

### Step 935: Correlate Traces กับ Logs

```javascript
// logger.js — inject trace_id ใน every log line
const winston = require('winston');
const { trace, context } = require('@opentelemetry/api');

function getTraceContext() {
  const span = trace.getActiveSpan();
  if (!span) return {};
  
  const spanContext = span.spanContext();
  return {
    trace_id: spanContext.traceId,
    span_id: spanContext.spanId,
    trace_flags: spanContext.traceFlags,
  };
}

const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json(),
    winston.format((info) => {
      // Inject trace context ใน every log
      return { ...info, ...getTraceContext() };
    })()
  ),
  transports: [
    new winston.transports.Console(),
  ],
});

module.exports = logger;
```

```json
// Log output with trace context:
{
  "timestamp": "2024-01-01T12:00:00Z",
  "level": "info",
  "message": "Processing feed request",
  "user_id": "456",
  "trace_id": "0af7651916cd43dd8448eb211c80319c",
  "span_id": "b7ad6b7169203331",
  "trace_flags": 1
}
```

```
Grafana query — find logs by trace_id:
{app="feed-service"} | json | trace_id="0af7651916cd43dd8448eb211c80319c"
```

---

## 🔧 Configuration Files

### OpenTelemetry Collector Pipeline

```yaml
# otel-collector.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
  namespace: observability
spec:
  template:
    spec:
      containers:
      - name: otel-collector
        image: otel/opentelemetry-collector-contrib:latest
        args: ["--config=/etc/otel/config.yaml"]
        volumeMounts:
        - name: config
          mountPath: /etc/otel
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          http:
            endpoint: 0.0.0.0:4318
          grpc:
            endpoint: 0.0.0.0:4317
    
    processors:
      batch:
        timeout: 1s
        send_batch_size: 1024
      
      # กรอง sensitive data ออก
      attributes:
        actions:
          - key: user.password
            action: delete
          - key: authorization
            action: delete
      
      # Tail sampling: 100% errors, 1% normal
      tail_sampling:
        decision_wait: 10s
        policies:
          - name: errors
            type: status_code
            status_code: { status_codes: [ERROR] }
          - name: slow-traces
            type: latency
            latency: { threshold_ms: 500 }
          - name: sos-operations
            type: string_attribute
            string_attribute:
              key: service.name
              values: [sos-service]
          - name: default-1pct
            type: probabilistic
            probabilistic: { sampling_percentage: 1 }
    
    exporters:
      jaeger:
        endpoint: jaeger-collector.observability:14250
        tls:
          insecure: true
    
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch, attributes, tail_sampling]
          exporters: [jaeger]
```

---

## 🧪 Testing

### Test Trace Propagation

```javascript
// trace-propagation.test.js
const request = require('supertest');
const { trace } = require('@opentelemetry/api');

describe('Distributed Tracing', () => {
  it('should propagate trace context across services', async () => {
    // ส่ง request พร้อม trace context
    const response = await request(app)
      .get('/api/v1/feed')
      .set('traceparent', '00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01')
      .set('Authorization', `Bearer ${testToken}`)
      .expect(200);
    
    // Response ต้องมี trace info กลับมา
    expect(response.headers['trace-id']).toBeDefined();
    
    // ตรวจว่า trace ถูก recorded ใน Jaeger
    await waitFor(async () => {
      const traces = await jaegerClient.getTraces({
        service: 'feed-service',
        traceId: '0af7651916cd43dd8448eb211c80319c',
      });
      return traces.length > 0;
    }, { timeout: 5000 });
  });
  
  it('should create child spans for DB queries', async () => {
    let capturedSpans = [];
    
    // Mock exporter to capture spans
    const mockExporter = {
      export: (spans) => { capturedSpans.push(...spans); },
      shutdown: () => Promise.resolve(),
    };
    
    await request(app).get('/api/v1/feed').set('Authorization', `Bearer ${testToken}`);
    
    const dbSpans = capturedSpans.filter(s => s.name.includes('pg.query'));
    expect(dbSpans.length).toBeGreaterThan(0);
    expect(dbSpans[0].attributes['db.system']).toBe('postgresql');
  });
});
```

---

## ❌ Common Errors & Solutions

### Error 1: Traces ไม่ปรากฎใน Jaeger

```
ตรวจ 3 จุด:
1. Application → Collector (OTLP)
2. Collector → Jaeger (gRPC)
3. Jaeger → Elasticsearch (storage)
```

```bash
# 1. ตรวจว่า app ส่ง traces ถึง collector
kubectl logs -n observability otel-collector-xxx | grep "Received spans"

# 2. ตรวจ Jaeger Collector
kubectl logs -n observability jaeger-collector-xxx | grep -i "error"

# 3. ทดสอบโดยตรง
kubectl exec -it app-pod -- curl -X POST \
  http://otel-collector.observability:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d '{"resourceSpans": []}' 
```

### Error 2: Too many traces (cost too high)

```javascript
// ลด sampling rate สำหรับ high-traffic endpoints
process.env.OTEL_TRACES_SAMPLER_ARG = '0.001'; // 0.1% แทน 1%

// หรือ exclude health check endpoints
instrumentations: [
  getNodeAutoInstrumentations({
    '@opentelemetry/instrumentation-http': {
      ignoreIncomingPaths: ['/health', '/metrics', '/favicon.ico'],
    },
  }),
],
```

---

## ✅ Checklist

- [ ] Jaeger cluster ติดตั้งบน Kubernetes แล้ว
- [ ] OpenTelemetry SDK ใส่ใน services ทุกตัวแล้ว
- [ ] Trace propagation ทำงานข้าม 3 services ขึ้นไปได้
- [ ] trace_id ปรากฎใน logs ทุก service
- [ ] Sampling: 1% normal, 100% errors/SOS
- [ ] Service Dependency Graph ใน Jaeger แสดงถูกต้อง
- [ ] Grafana → Jaeger integration ตั้งค่าแล้ว (click trace_id from logs)
- [ ] Slow query alerts ใช้ trace data
- [ ] Jaeger data retention policy ตั้งค่าแล้ว (7 days)
- [ ] Runbook: วิธี debug slow request ด้วย Jaeger

---

## 🔗 References

- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [Jaeger Documentation](https://www.jaegertracing.io/docs/)
- [W3C TraceContext Specification](https://www.w3.org/TR/trace-context/)
- [OpenTelemetry Node.js](https://opentelemetry.io/docs/instrumentation/js/)
- [Distributed Tracing with Jaeger — Yuri Shkuro](https://www.jaegertracing.io/docs/1.21/getting-started/)

---

*Part 094 | Road to 1,000,000 Users/Day | chuaikan.com*
