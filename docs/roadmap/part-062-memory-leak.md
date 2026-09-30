# Part 062: Memory Leak Detection & Fix

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Advanced
> **Steps:** 611-620
> **เวลาโดยประมาณ:** 5 ชั่วโมง
> **Prerequisites:** Part 061 (Node.js Profiling)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ระบุ symptoms ของ memory leak: growing heap, OOMKilled
- วิธีทำ Heap Snapshot ด้วย Chrome DevTools
- ใช้ heapdump module สำหรับ production analysis
- Memory leak patterns ที่พบบ่อยใน Node.js
- แก้ memory leak ใน Express.js middleware
- ใช้ WeakMap และ WeakRef สำหรับ cache
- ปรับ `--max-old-space-size`
- Monitor memory ด้วย Prometheus

---

## 📖 ทฤษฎีและแนวคิด

### Node.js Memory Structure

```
┌─────────────────────────────────────────────┐
│              Node.js Process Memory          │
├─────────────┬──────────────┬────────────────┤
│    Stack    │     Heap     │  External       │
│  (small,    │  (large,     │  (Buffer,       │
│  fast)      │  GC managed) │  native)        │
├─────────────┴──────────────┴────────────────┤
│  Heap = New Space + Old Space + Code Space  │
│  New Space: Young objects (short-lived)     │
│  Old Space: Survived 2 GC cycles (long-lived)│
└─────────────────────────────────────────────┘
```

### Memory Leak คืออะไร?

Memory leak เกิดขึ้นเมื่อ objects ถูก allocate แต่ไม่ถูก garbage collect แม้ว่าจะไม่ถูกใช้งานแล้ว สาเหตุหลักๆ:

1. **Global variables** ที่เก็บข้อมูลโดยไม่ล้าง
2. **Event listeners** ที่ไม่ถูก remove
3. **Closures** ที่ capture references โดยไม่ตั้งใจ
4. **Cache** ที่ไม่มี TTL หรือ size limit
5. **Timers** ที่ไม่ถูก clear

### Memory Leak Symptoms

```
Timeline ของ memory leak:
                                           OOMKilled!
Heap Usage                                     ↓
(MB) │                                    ╔══╗
     │                               ╔═══╝  ║
512  │                          ╔════╝       ║ process
     │                    ╔═════╝             exits
256  │              ╔══════╝
     │        ╔═════╝
128  │   ╔════╝
     │═══╝
      ───────────────────────────────────── time
```

---

## ⚙️ Environment Setup

```bash
# สร้าง project directory
mkdir -p /home/user/memory-leak-demo
cd /home/user/memory-leak-demo
npm init -y

# ติดตั้ง dependencies
npm install express heapdump prom-client node-schedule

# ติดตั้ง global tools
npm install -g clinic autocannon

# ตรวจสอบ memory ปัจจุบัน
node -e "console.log(process.memoryUsage())"
```

---

## 🛠️ Step-by-Step Implementation

### Step 611: Memory Leak Symptoms Detection

```javascript
// memory-monitor.js - Basic memory monitoring
function formatBytes(bytes) {
  return (bytes / 1024 / 1024).toFixed(2) + ' MB';
}

function checkMemory() {
  const mem = process.memoryUsage();
  const stats = {
    heapUsed: formatBytes(mem.heapUsed),
    heapTotal: formatBytes(mem.heapTotal),
    rss: formatBytes(mem.rss),            // Resident Set Size
    external: formatBytes(mem.external),   // C++ objects
    arrayBuffers: formatBytes(mem.arrayBuffers),
    heapUsedPercent: ((mem.heapUsed / mem.heapTotal) * 100).toFixed(1) + '%'
  };
  
  console.log(new Date().toISOString(), JSON.stringify(stats));
  
  // Warning ถ้า heap ใช้ > 80%
  if (mem.heapUsed / mem.heapTotal > 0.8) {
    console.error('⚠️  HIGH MEMORY USAGE:', stats.heapUsedPercent);
  }
}

// Monitor ทุก 5 วินาที
setInterval(checkMemory, 5000);
checkMemory(); // initial check
```

```bash
# รันพร้อม memory monitoring
node --max-old-space-size=512 memory-monitor.js
```

### Step 612: สร้าง Memory Leak ตัวอย่าง

```javascript
// leaky-app.js - App ที่มี memory leaks หลายประเภท
const express = require('express');
const EventEmitter = require('events');
const app = express();

// ========================
// LEAK 1: Global variable cache without TTL
// ========================
const requestCache = new Map(); // ❌ ไม่มี TTL, ไม่มี size limit!

app.get('/api/cache-leak', (req, res) => {
  const key = `request-${Date.now()}-${Math.random()}`;
  // ❌ BAD: เพิ่ม cache เรื่อยๆ โดยไม่ล้าง
  requestCache.set(key, {
    timestamp: Date.now(),
    data: Buffer.alloc(1024), // 1KB per entry
    url: req.url,
    headers: req.headers  // หัว HTTP ก็ถูกเก็บไว้
  });
  
  res.json({ cacheSize: requestCache.size });
});

// ========================
// LEAK 2: Event listener leak
// ========================
class DataProcessor extends EventEmitter {
  process(data) {
    // ❌ BAD: เพิ่ม listener ทุกครั้งที่ call โดยไม่ remove
    process.on('data', (chunk) => {
      // handle data
    });
    this.emit('processed', data);
  }
}

const processor = new DataProcessor();

app.post('/api/process', express.json(), (req, res) => {
  processor.process(req.body);
  res.json({ ok: true });
});

// ========================
// LEAK 3: Closure capturing large object
// ========================
function createHandler(largeObject) {
  // ❌ BAD: closure จะ hold reference ไปยัง largeObject ตลอด
  return function handler(req, res) {
    // เข้าถึง largeObject ใน closure
    res.json({ length: largeObject.data.length });
  };
}

const bigData = { data: Buffer.alloc(10 * 1024 * 1024) }; // 10MB
app.get('/api/big-handler', createHandler(bigData));

// ========================
// LEAK 4: Timer not cleared
// ========================
const timers = [];
app.get('/api/start-timer', (req, res) => {
  // ❌ BAD: สร้าง timer ใหม่ทุกครั้ง โดยไม่ clear เก่า
  const timer = setInterval(() => {
    // do work
  }, 1000);
  timers.push(timer); // เก็บ reference แต่ไม่ clear
  
  res.json({ timersCount: timers.length });
});

const PORT = 3001;
app.listen(PORT, () => {
  console.log(`Leaky app on port ${PORT}`);
});
```

### Step 613: Heap Snapshot ด้วย Chrome DevTools

```bash
# รัน app ด้วย --inspect
node --inspect=0.0.0.0:9229 leaky-app.js
```

ขั้นตอนการทำ Heap Snapshot:
1. เปิด Chrome → `chrome://inspect`
2. คลิก "Open dedicated DevTools for Node"
3. ไปที่ **Memory** tab
4. เลือก "Heap snapshot" และคลิก **Take snapshot** (Snapshot 1 - baseline)
5. สร้าง load: `autocannon -d 30 -c 50 http://localhost:3001/api/cache-leak`
6. คลิก **Take snapshot** อีกครั้ง (Snapshot 2 - after load)
7. เลือก "Comparison" view เพื่อดู objects ที่เพิ่มขึ้น

```javascript
// heap-snapshot-programmatic.js - สำหรับ automation
const v8 = require('v8');
const fs = require('fs');
const path = require('path');

function takeHeapSnapshot(label = '') {
  const filename = `heap-${label}-${Date.now()}.heapsnapshot`;
  const filepath = path.join('/tmp', filename);
  
  // สร้าง heap snapshot
  const snapshotStream = v8.writeHeapSnapshot(filepath);
  console.log(`Heap snapshot written to: ${snapshotStream}`);
  
  return snapshotStream;
}

// Auto-snapshot ถ้า heap ใช้ > 500MB
setInterval(() => {
  const mem = process.memoryUsage();
  const heapMB = mem.heapUsed / 1024 / 1024;
  
  if (heapMB > 500) {
    console.warn(`High heap usage: ${heapMB.toFixed(0)}MB - taking snapshot`);
    takeHeapSnapshot('auto');
  }
}, 30000);
```

### Step 614: Heapdump Module สำหรับ Production

```bash
npm install heapdump
```

```javascript
// production-heapdump.js
const heapdump = require('heapdump');
const path = require('path');
const os = require('os');

const DUMP_DIR = process.env.HEAP_DUMP_DIR || '/var/log/heapdumps';

// สร้าง directory ถ้ายังไม่มี
const fs = require('fs');
if (!fs.existsSync(DUMP_DIR)) {
  fs.mkdirSync(DUMP_DIR, { recursive: true });
}

// Signal-based heap dump (trigger จาก terminal)
// kill -USR2 <pid>  ← ส่ง signal SIGUSR2
process.on('SIGUSR2', () => {
  const filename = path.join(DUMP_DIR, `heapdump-${process.pid}-${Date.now()}.heapsnapshot`);
  heapdump.writeSnapshot(filename, (err, filename) => {
    if (err) {
      console.error('Failed to write heap snapshot:', err);
      return;
    }
    console.log(`Heap snapshot written to: ${filename}`);
  });
});

// HTTP endpoint สำหรับ trigger heap dump (ต้อง protect ด้วย auth!)
const express = require('express');
const app = express();

// ⚠️ IMPORTANT: ต้อง protect endpoint นี้ด้วย auth middleware!
app.get('/admin/heap-dump', requireAdminAuth, (req, res) => {
  const filename = path.join(DUMP_DIR, `heapdump-${Date.now()}.heapsnapshot`);
  heapdump.writeSnapshot(filename, (err, snapshotPath) => {
    if (err) {
      return res.status(500).json({ error: err.message });
    }
    const stats = fs.statSync(snapshotPath);
    res.json({
      message: 'Heap snapshot created',
      path: snapshotPath,
      size: (stats.size / 1024 / 1024).toFixed(2) + ' MB'
    });
  });
});

function requireAdminAuth(req, res, next) {
  const token = req.headers['x-admin-token'];
  if (token !== process.env.ADMIN_TOKEN) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  next();
}
```

```bash
# Trigger heap dump ใน production
kill -USR2 $(pgrep -f "node app.js")

# Download heap snapshot จาก production server
scp user@prod-server:/var/log/heapdumps/heapdump-*.heapsnapshot ./

# เปิดดูใน Chrome DevTools: Memory → Load profile
```

### Step 615: แก้ไข Memory Leaks

```javascript
// fixed-app.js - Version ที่แก้ memory leaks แล้ว

// ========================
// FIX 1: Cache with TTL and size limit
// ========================
class LRUCacheWithTTL {
  constructor({ maxSize = 1000, ttl = 60000 } = {}) {
    this.cache = new Map();
    this.maxSize = maxSize;
    this.ttl = ttl;
  }
  
  set(key, value) {
    // ลบ oldest entry ถ้า cache เต็ม
    if (this.cache.size >= this.maxSize) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    
    this.cache.set(key, {
      value,
      expiresAt: Date.now() + this.ttl
    });
  }
  
  get(key) {
    const entry = this.cache.get(key);
    if (!entry) return undefined;
    
    // ตรวจสอบ TTL
    if (Date.now() > entry.expiresAt) {
      this.cache.delete(key);
      return undefined;
    }
    
    return entry.value;
  }
  
  // Periodic cleanup
  startCleanup(interval = 60000) {
    this.cleanupTimer = setInterval(() => {
      const now = Date.now();
      for (const [key, entry] of this.cache) {
        if (now > entry.expiresAt) {
          this.cache.delete(key);
        }
      }
    }, interval);
    
    // อย่าให้ timer block process exit
    this.cleanupTimer.unref();
    return this;
  }
}

const requestCache = new LRUCacheWithTTL({ maxSize: 1000, ttl: 300000 })
  .startCleanup();

// ========================
// FIX 2: Properly remove event listeners
// ========================
class DataProcessor extends EventEmitter {
  process(data) {
    // ✅ GOOD: ใช้ once() แทน on() ถ้าต้องการ listen ครั้งเดียว
    // หรือ remove listener หลังใช้งาน
    const dataHandler = (chunk) => {
      // handle data
      process.removeListener('data', dataHandler); // ✅ clean up
    };
    process.on('data', dataHandler);
    this.emit('processed', data);
  }
}

// ========================
// FIX 3: WeakMap สำหรับ object-keyed cache
// ========================
// WeakMap ไม่ prevent garbage collection ของ keys
const handlerCache = new WeakMap();

function createOptimizedHandler(requestObj) {
  if (handlerCache.has(requestObj)) {
    return handlerCache.get(requestObj);
  }
  
  const handler = function(req, res) {
    res.json({ processed: true });
  };
  
  handlerCache.set(requestObj, handler);
  // เมื่อ requestObj ถูก GC, handler ใน WeakMap ก็จะถูก GC ด้วย
  return handler;
}

// ========================
// FIX 4: Timer management
// ========================
class TimerManager {
  constructor() {
    this.timers = new Set();
  }
  
  add(fn, interval) {
    const timer = setInterval(fn, interval);
    this.timers.add(timer);
    return timer;
  }
  
  remove(timer) {
    clearInterval(timer);
    this.timers.delete(timer);
  }
  
  clearAll() {
    for (const timer of this.timers) {
      clearInterval(timer);
    }
    this.timers.clear();
  }
}

const timerManager = new TimerManager();

// Cleanup on shutdown
process.on('SIGTERM', () => {
  timerManager.clearAll();
  process.exit(0);
});
```

### Step 616: WeakRef สำหรับ Optional Caching

```javascript
// weakref-cache.js - Node.js v14.6.0+
class WeakRefCache {
  constructor() {
    this.cache = new Map();
    // FinalizationRegistry แจ้งเมื่อ object ถูก GC
    this.registry = new FinalizationRegistry((key) => {
      const ref = this.cache.get(key);
      if (ref !== undefined && ref.deref() === undefined) {
        this.cache.delete(key);
        console.log(`Cache entry ${key} was garbage collected`);
      }
    });
  }
  
  set(key, value) {
    const ref = new WeakRef(value);
    this.cache.set(key, ref);
    this.registry.register(value, key);
  }
  
  get(key) {
    const ref = this.cache.get(key);
    if (!ref) return undefined;
    
    // deref() returns undefined ถ้าถูก GC แล้ว
    const value = ref.deref();
    if (value === undefined) {
      this.cache.delete(key);
      return undefined;
    }
    
    return value;
  }
}

// Usage example
const cache = new WeakRefCache();

// เหมาะสำหรับ cache ขนาดใหญ่ที่ยอมให้ GC ล้างได้
function processLargeRequest(requestId, data) {
  let processed = cache.get(requestId);
  
  if (!processed) {
    processed = { result: data.map(x => x * 2), requestId };
    cache.set(requestId, processed);
  }
  
  return processed;
}
```

### Step 617: --max-old-space-size Tuning

```bash
# Default V8 heap limits (Node.js)
# 32-bit: ~1.5GB
# 64-bit: ~4GB (Node.js v12+)

# ดู current heap limits
node -e "
const v8 = require('v8');
console.log('Heap Statistics:', v8.getHeapStatistics());
"

# Set max heap size
node --max-old-space-size=2048 app.js  # 2GB

# สำหรับ container ที่มี RAM จำกัด
# ถ้า container มี 1GB RAM:
node --max-old-space-size=768 app.js   # 75% ของ RAM

# ใน Dockerfile
ENV NODE_OPTIONS="--max-old-space-size=768"
```

```yaml
# kubernetes-deployment.yaml - Resource limits กับ heap size
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chuaikan-api
spec:
  template:
    spec:
      containers:
      - name: api
        image: chuaikan/api:latest
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        env:
        - name: NODE_OPTIONS
          # ตั้งค่า heap เป็น 75% ของ memory limit
          value: "--max-old-space-size=768"
        # Liveness probe ตรวจสอบว่า process ยังทำงานอยู่
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 15
          periodSeconds: 10
          failureThreshold: 3
```

### Step 618: Memory Monitoring ด้วย Prometheus

```bash
npm install prom-client
```

```javascript
// metrics.js - Prometheus metrics สำหรับ memory
const client = require('prom-client');

// Enable default metrics (CPU, memory, GC stats)
const collectDefaultMetrics = client.collectDefaultMetrics;
collectDefaultMetrics({ prefix: 'chuaikan_' });

// Custom memory metrics
const heapUsedGauge = new client.Gauge({
  name: 'nodejs_heap_used_bytes',
  help: 'V8 heap used bytes',
  labelNames: ['type']
});

const heapTotalGauge = new client.Gauge({
  name: 'nodejs_heap_total_bytes',
  help: 'V8 heap total bytes'
});

const gcPauseHistogram = new client.Histogram({
  name: 'nodejs_gc_pause_duration_seconds',
  help: 'GC pause duration in seconds',
  labelNames: ['type'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1]
});

// Update memory gauges
function updateMemoryMetrics() {
  const mem = process.memoryUsage();
  heapUsedGauge.set({ type: 'used' }, mem.heapUsed);
  heapUsedGauge.set({ type: 'external' }, mem.external);
  heapTotalGauge.set(mem.heapTotal);
}

setInterval(updateMemoryMetrics, 10000);

// GC monitoring (Node.js v16+)
try {
  const { PerformanceObserver } = require('perf_hooks');
  const obs = new PerformanceObserver((list) => {
    list.getEntries().forEach((entry) => {
      if (entry.entryType === 'gc') {
        gcPauseHistogram.observe(
          { type: entry.detail?.kind || 'unknown' },
          entry.duration / 1000
        );
      }
    });
  });
  obs.observe({ type: 'gc' });
} catch (e) {
  console.warn('GC metrics not available');
}

// Metrics endpoint
const express = require('express');
const app = express();

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});

module.exports = { app };
```

```yaml
# prometheus-alert-rules.yaml
groups:
- name: nodejs_memory
  rules:
  - alert: HighHeapUsage
    expr: |
      (nodejs_heap_used_bytes / nodejs_heap_total_bytes) > 0.85
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High Node.js heap usage"
      description: "Heap usage is {{ $value | humanizePercentage }} for {{ $labels.instance }}"
  
  - alert: PossibleMemoryLeak
    expr: |
      rate(nodejs_heap_used_bytes[30m]) > 1048576
    for: 10m
    labels:
      severity: critical
    annotations:
      summary: "Possible memory leak detected"
      description: "Heap growing at {{ $value | humanize }}B/s for 10+ minutes"
```

### Step 619: Automated Memory Leak Detection in CI

```javascript
// memory-leak-test.js - Jest test สำหรับ detect memory leak
const { execSync } = require('child_process');

describe('Memory Leak Tests', () => {
  let server;
  
  beforeAll(async () => {
    server = require('./app'); // start server
    await new Promise(resolve => setTimeout(resolve, 1000));
  });
  
  afterAll(async () => {
    await server.close();
  });
  
  test('Memory should not grow unboundedly under load', async () => {
    const initialMem = process.memoryUsage().heapUsed;
    
    // สร้าง requests จำนวนมาก
    const requests = Array.from({ length: 1000 }, (_, i) => 
      fetch(`http://localhost:3000/api/users?page=${i}`)
        .then(r => r.json())
        .catch(() => null)
    );
    
    await Promise.all(requests);
    
    // Force GC (ต้องรัน Node.js ด้วย --expose-gc)
    if (global.gc) global.gc();
    
    await new Promise(resolve => setTimeout(resolve, 2000));
    
    const finalMem = process.memoryUsage().heapUsed;
    const growthMB = (finalMem - initialMem) / 1024 / 1024;
    
    // Memory should not grow more than 50MB for 1000 requests
    expect(growthMB).toBeLessThan(50);
  });
});
```

```bash
# รัน memory leak test ด้วย --expose-gc
node --expose-gc node_modules/.bin/jest memory-leak-test.js
```

### Step 620: Common Memory Leak Patterns Summary

```javascript
// memory-patterns.js - สรุป patterns และ fixes

// Pattern 1: Accumulating event listeners
const emitter = require('events').EventEmitter;
const ee = new emitter();

// ❌ BAD
function attachBadListener() {
  ee.on('data', (d) => console.log(d)); // adds new listener every call!
}

// ✅ GOOD
const dataHandler = (d) => console.log(d);
ee.on('data', dataHandler);
// cleanup when done:
// ee.removeListener('data', dataHandler);

// Pattern 2: Holding DOM-like references in Node.js
// ❌ BAD: large object held in closure
function badClosure() {
  const largeBuffer = Buffer.alloc(100 * 1024 * 1024); // 100MB
  return () => {
    return largeBuffer.length; // closure holds entire buffer
  };
}

// ✅ GOOD: Extract only what's needed
function goodClosure() {
  const bufferLength = Buffer.alloc(100 * 1024 * 1024).length; // ✅ only store length
  return () => bufferLength;
}

// Pattern 3: Cache poisoning
// ❌ BAD: Unbounded Map
const unboundedCache = new Map();

// ✅ GOOD: Bounded cache with LRU eviction
const { LRUCache } = require('lru-cache');
const boundedCache = new LRUCache({
  max: 1000,
  ttl: 1000 * 60 * 5, // 5 minutes
  updateAgeOnGet: true,
});
```

---

## 🔧 Configuration Files

### .env สำหรับ memory settings

```bash
# .env
NODE_OPTIONS=--max-old-space-size=1024
HEAP_DUMP_DIR=/var/log/heapdumps
ADMIN_TOKEN=change-this-in-production
MEMORY_ALERT_THRESHOLD_MB=800
```

### grafana-memory-dashboard.json (excerpt)

```json
{
  "panels": [
    {
      "title": "Node.js Heap Usage",
      "type": "graph",
      "targets": [
        {
          "expr": "nodejs_heap_used_bytes{job=\"chuaikan-api\"}",
          "legendFormat": "Heap Used"
        },
        {
          "expr": "nodejs_heap_total_bytes{job=\"chuaikan-api\"}",
          "legendFormat": "Heap Total"
        }
      ],
      "yaxes": [{ "format": "bytes" }]
    }
  ]
}
```

---

## 🧪 Testing

```bash
# Test 1: ตรวจสอบ memory leak ด้วย clinic
clinic doctor -- node leaky-app.js &
autocannon -d 60 -c 100 http://localhost:3001/api/cache-leak
# ดูว่า memory เพิ่มขึ้นเรื่อยๆ หรือไม่

# Test 2: รัน load test และ monitor memory
node --max-old-space-size=256 leaky-app.js &
watch -n 2 'ps aux | grep node | grep -v grep | awk "{print \$6/1024\" MB\"}"'

# Test 3: Take heap snapshot ก่อนและหลัง load
node --inspect leaky-app.js &
# ทำ heap snapshot ด้วย Chrome DevTools
# รัน load: autocannon -d 30 -c 50 http://localhost:3001/api/cache-leak
# ทำ heap snapshot อีกครั้ง
# Compare snapshots
```

---

## ❌ Common Errors & Solutions

### Error 1: FATAL ERROR: CALL_AND_RETRY_LAST Allocation failed

```bash
# ปัญหา: Out of memory
FATAL ERROR: CALL_AND_RETRY_LAST Allocation failed - JavaScript heap out of memory

# แก้ไข 1: เพิ่ม heap size
node --max-old-space-size=4096 app.js

# แก้ไข 2: หาและแก้ memory leak (แนะนำ)
# ใช้ heap snapshot เพื่อหา root cause
```

### Error 2: MaxListenersExceededWarning

```bash
# Warning: Possible EventEmitter memory leak detected
# 11 data listeners added to [EventEmitter]

# แก้ไข: เพิ่ม maxListeners หรือ remove listeners
emitter.setMaxListeners(20);

# หรือหาว่า listener ไหน leak และ remove มัน
emitter.removeAllListeners('data');
```

### Error 3: heapdump ทำให้ process หยุดชั่วคราว

```bash
# ปัญหา: Heap dump ขนาดใหญ่ทำให้ process หยุดนานมาก (stop-the-world GC)
# แก้ไข: ทำ heap dump นอก peak hours
# หรือใช้ streaming heap dump (Node.js v18+)
const { createHeapSnapshot } = require('v8');
# ส่งผ่าน stream แทน write ทั้งหมดพร้อมกัน
```

---

## ✅ Checklist

- [ ] ติดตั้ง memory monitoring (Prometheus + prom-client)
- [ ] ตั้งค่า `--max-old-space-size` ให้เหมาะสมกับ container RAM
- [ ] ตั้งค่า `NODE_OPTIONS` ใน Kubernetes deployment
- [ ] สร้าง `/admin/heap-dump` endpoint (พร้อม auth protection)
- [ ] ตั้งค่า Prometheus alert สำหรับ memory growth
- [ ] Review code สำหรับ unbounded Map/Array (ต้องมี size limit และ TTL)
- [ ] ตรวจสอบ event listeners ทุกที่ (ต้องมี removeListener)
- [ ] ย้าย cache ที่เหมาะสมไปใช้ LRU Cache (lru-cache package)
- [ ] เพิ่ม memory leak test ใน CI pipeline
- [ ] สร้าง Grafana dashboard สำหรับ memory metrics

---

## 🔗 References

- [Node.js Memory Management](https://nodejs.org/en/docs/guides/diagnostics/memory/)
- [V8 Heap Snapshots](https://v8.dev/docs/memory-leak-investigation)
- [heapdump npm package](https://www.npmjs.com/package/heapdump)
- [lru-cache npm package](https://www.npmjs.com/package/lru-cache)
- [WeakRef MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakRef)
- [prom-client Node.js](https://github.com/siimon/prom-client)

---

*Part 062 | Road to 1,000,000 Users/Day | chuaikan.com*
