# Part 063: CPU Optimization

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Advanced
> **Steps:** 621-630
> **เวลาโดยประมาณ:** 4 ชั่วโมง
> **Prerequisites:** Part 061-062 (Node.js Profiling, Memory Leaks)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ข้อจำกัดของ Node.js single-threaded model
- ใช้ Worker Threads สำหรับงาน CPU-intensive
- Cluster mode: fork processes เท่ากับจำนวน CPU cores
- ตั้งค่า PM2 cluster mode
- Offload งานไปที่ Lambda/Cloud Functions
- Cache computed results เพื่อลด CPU usage
- ผลกระทบของ Database query optimization ต่อ CPU

---

## 📖 ทฤษฎีและแนวคิด

### Node.js Single-Threaded Model

```
Traditional Multi-threaded Server:
┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
│Thread│ │Thread│ │Thread│ │Thread│
│  1   │ │  2   │ │  3   │ │  4   │
└──────┘ └──────┘ └──────┘ └──────┘
   1 request per thread (blocking)

Node.js Event-Driven (default):
┌─────────────────────────────────┐
│          Event Loop             │
│  req1 → req2 → req3 → req4 → │
│  (non-blocking I/O, fast!)      │
└─────────────────────────────────┘
  แต่ CPU-intensive task จะ block ทุก requests!

Node.js Cluster Mode:
┌──────────────────────────────────┐
│          Master Process          │
│    Load Balancer (Round-Robin)   │
└──┬──────┬──────┬──────┬──────┬──┘
   ↓      ↓      ↓      ↓
┌──────┐┌──────┐┌──────┐┌──────┐
│Worker││Worker││Worker││Worker│
│  1   ││  2   ││  3   ││  4   │
└──────┘└──────┘└──────┘└──────┘
  ใช้ทุก CPU core!
```

### CPU vs I/O Bound Tasks

- **I/O Bound**: Database query, HTTP call, file read/write → Node.js จัดการได้ดีมาก
- **CPU Bound**: Image processing, crypto, JSON parsing ขนาดใหญ่, ML inference → ต้องใช้ Worker Threads หรือ offload

---

## ⚙️ Environment Setup

```bash
# ตรวจสอบ CPU cores
nproc
cat /proc/cpuinfo | grep "model name" | head -1

# ติดตั้ง PM2
npm install -g pm2

# ติดตั้ง project dependencies
mkdir -p /home/user/cpu-optimization-demo
cd /home/user/cpu-optimization-demo
npm init -y
npm install express piscina sharp bcrypt

# Piscina = worker thread pool library
# Sharp = image processing
# Bcrypt = CPU-intensive hashing
```

---

## 🛠️ Step-by-Step Implementation

### Step 621: ทำความเข้าใจ Event Loop Blocking

```javascript
// blocking-demo.js - แสดงผลของ CPU blocking
const express = require('express');
const app = express();

// ❌ BAD: CPU-intensive task ที่ block event loop
function heavyCPUWork(iterations = 1e8) {
  let result = 0;
  for (let i = 0; i < iterations; i++) {
    result += Math.sqrt(i);
  }
  return result;
}

app.get('/api/heavy', (req, res) => {
  const start = Date.now();
  // ❌ นี่จะ block event loop ทั้งหมด!
  // ทุก request อื่นๆ จะต้องรอจนกว่า function นี้จะเสร็จ
  const result = heavyCPUWork();
  res.json({ result, duration: Date.now() - start });
});

app.get('/api/fast', (req, res) => {
  // request นี้จะถูก block ถ้ามีใคร call /api/heavy
  res.json({ message: 'This should be fast!' });
});

app.listen(3000, () => console.log('Server on port 3000'));
```

```bash
# ทดสอบการ blocking
node blocking-demo.js &

# Terminal 1: call heavy endpoint
curl http://localhost:3000/api/heavy &

# Terminal 2 (ทันที): call fast endpoint - จะถูก block!
time curl http://localhost:3000/api/fast
# จะใช้เวลาหลายวินาที แทนที่จะเป็น milliseconds
```

### Step 622: Worker Threads Solution

```javascript
// worker-pool-demo.js
const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');
const path = require('path');

// worker.js (inline สำหรับตัวอย่าง, ปกติ save เป็นไฟล์แยก)
const WORKER_CODE = `
const { parentPort, workerData } = require('worker_threads');

function heavyCPUWork(iterations) {
  let result = 0;
  for (let i = 0; i < iterations; i++) {
    result += Math.sqrt(i);
  }
  return result;
}

const result = heavyCPUWork(workerData.iterations);
parentPort.postMessage({ result, workerId: workerData.workerId });
`;

// Main thread
function runInWorker(data) {
  return new Promise((resolve, reject) => {
    const worker = new Worker(WORKER_CODE, {
      eval: true,
      workerData: data
    });
    
    worker.on('message', resolve);
    worker.on('error', reject);
    worker.on('exit', (code) => {
      if (code !== 0) reject(new Error(`Worker exited with code ${code}`));
    });
  });
}

const express = require('express');
const app = express();

// ✅ GOOD: CPU task รันใน worker thread
app.get('/api/heavy', async (req, res) => {
  const start = Date.now();
  
  try {
    const result = await runInWorker({
      iterations: 1e8,
      workerId: Math.random()
    });
    
    res.json({
      ...result,
      duration: Date.now() - start
    });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// ✅ Fast endpoint ไม่ถูก block แล้ว!
app.get('/api/fast', (req, res) => {
  res.json({ message: 'Fast response!', timestamp: Date.now() });
});

app.listen(3000);
```

### Step 623: Piscina - Worker Thread Pool

```bash
npm install piscina
```

```javascript
// workers/cpu-worker.js - Worker function file
module.exports = async function(data) {
  const { task, payload } = data;
  
  switch (task) {
    case 'fibonacci':
      return fibonacci(payload.n);
    
    case 'hash':
      const bcrypt = require('bcrypt');
      return bcrypt.hash(payload.password, payload.rounds);
    
    case 'imageResize':
      const sharp = require('sharp');
      const result = await sharp(payload.buffer)
        .resize(payload.width, payload.height)
        .webp({ quality: 80 })
        .toBuffer();
      return result;
    
    default:
      throw new Error(`Unknown task: ${task}`);
  }
};

function fibonacci(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}
```

```javascript
// server.js - ใช้ Piscina pool
const Piscina = require('piscina');
const path = require('path');
const os = require('os');
const express = require('express');

// สร้าง worker pool
const pool = new Piscina({
  filename: path.resolve(__dirname, 'workers/cpu-worker.js'),
  // ใช้ CPU cores - 1 (เก็บ 1 core ไว้สำหรับ event loop)
  maxThreads: Math.max(1, os.cpus().length - 1),
  minThreads: 2,
  // Timeout สำหรับ tasks ที่ค้างนาน
  taskTimeout: 30000,
  // Memory limit per worker
  resourceLimits: {
    maxOldGenerationSizeMb: 256
  }
});

const app = express();
app.use(express.json());

// Monitor pool stats
setInterval(() => {
  console.log('Worker Pool Stats:', {
    threads: pool.threads.length,
    queueSize: pool.queueSize,
    completed: pool.completed,
    runTime: pool.runTime
  });
}, 30000);

// CPU-intensive endpoints ใช้ pool
app.post('/api/hash', async (req, res) => {
  try {
    const hash = await pool.run({
      task: 'hash',
      payload: { password: req.body.password, rounds: 12 }
    });
    res.json({ hash });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.post('/api/fibonacci', async (req, res) => {
  try {
    const result = await pool.run({
      task: 'fibonacci',
      payload: { n: req.body.n || 40 }
    });
    res.json({ result });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.listen(3000, () => console.log('Piscina pool server on :3000'));
```

### Step 624: Node.js Cluster Mode

```javascript
// cluster-app.js - Manual cluster setup
const cluster = require('cluster');
const os = require('os');
const express = require('express');

const NUM_CPUS = os.cpus().length;

if (cluster.isPrimary) {
  console.log(`Primary process ${process.pid} starting ${NUM_CPUS} workers`);
  
  // Fork workers
  for (let i = 0; i < NUM_CPUS; i++) {
    const worker = cluster.fork();
    console.log(`Worker ${worker.process.pid} started`);
  }
  
  // Restart crashed workers
  cluster.on('exit', (worker, code, signal) => {
    console.warn(`Worker ${worker.process.pid} died (${signal || code}). Restarting...`);
    cluster.fork();
  });
  
  // Graceful shutdown
  process.on('SIGTERM', () => {
    console.log('Primary received SIGTERM, shutting down workers...');
    
    for (const id in cluster.workers) {
      cluster.workers[id].send('shutdown');
    }
    
    setTimeout(() => {
      console.log('Force killing remaining workers');
      process.exit(0);
    }, 10000);
  });
  
} else {
  // Worker process
  const app = express();
  app.use(express.json());
  
  app.get('/health', (req, res) => {
    res.json({
      pid: process.pid,
      worker: cluster.worker.id,
      uptime: process.uptime()
    });
  });
  
  app.get('/api/users', async (req, res) => {
    // simulate database query
    await new Promise(resolve => setTimeout(resolve, 10));
    res.json({ users: [], workerId: cluster.worker.id });
  });
  
  const server = app.listen(3000, () => {
    console.log(`Worker ${process.pid} listening on port 3000`);
  });
  
  // Graceful shutdown for worker
  process.on('message', (msg) => {
    if (msg === 'shutdown') {
      server.close(() => {
        process.exit(0);
      });
    }
  });
  
  // Handle uncaught exceptions
  process.on('uncaughtException', (err) => {
    console.error(`Worker ${process.pid} uncaught exception:`, err);
    process.exit(1); // Let primary restart us
  });
}
```

### Step 625: PM2 Cluster Mode

```bash
# ติดตั้ง PM2
npm install -g pm2

# รัน app ด้วย cluster mode
pm2 start app.js -i max        # ใช้ทุก CPU cores
pm2 start app.js -i 4          # ใช้ 4 instances
pm2 start app.js -i -1         # ใช้ CPU - 1 cores

# ดู status
pm2 status
pm2 monit                       # real-time monitoring

# Reload ทีละ process (zero-downtime)
pm2 reload all
pm2 reload app

# Scale up/down
pm2 scale app +2                # เพิ่ม 2 instances
pm2 scale app 8                 # กำหนดเป็น 8 instances
```

```javascript
// ecosystem.config.js - PM2 configuration file
module.exports = {
  apps: [{
    name: 'chuaikan-api',
    script: './src/server.js',
    
    // Cluster mode
    instances: 'max',           // ใช้ทุก CPU cores
    exec_mode: 'cluster',
    
    // Environment
    env: {
      NODE_ENV: 'production',
      PORT: 3000,
      NODE_OPTIONS: '--max-old-space-size=512'
    },
    
    // Restart policy
    max_restarts: 10,
    min_uptime: '5s',
    restart_delay: 1000,
    
    // Crash recovery
    autorestart: true,
    watch: false,               // ปิดใน production
    
    // Logging
    log_file: '/var/log/pm2/combined.log',
    out_file: '/var/log/pm2/out.log',
    error_file: '/var/log/pm2/error.log',
    log_date_format: 'YYYY-MM-DD HH:mm:ss',
    
    // Memory management
    max_memory_restart: '1G',   // restart ถ้า memory > 1GB
    
    // Graceful shutdown
    kill_timeout: 5000,
    listen_timeout: 3000,
    
    // Health check
    health_check_url: 'http://localhost:3000/health',
    health_check_grace_period: 5000,
  }]
};
```

```bash
# Deploy ด้วย ecosystem config
pm2 start ecosystem.config.js

# Save config (รัน auto-start เมื่อ server reboot)
pm2 save
pm2 startup

# Monitor performance
pm2 monit

# Logs
pm2 logs --lines 100
```

### Step 626: Offloading ไปที่ Lambda/Cloud Functions

```javascript
// lambda-offload.js - Offload CPU work ไปที่ AWS Lambda
const { LambdaClient, InvokeCommand } = require('@aws-sdk/client-lambda');

const lambda = new LambdaClient({ region: process.env.AWS_REGION || 'ap-southeast-1' });

async function invokeWorkerLambda(task, payload) {
  const command = new InvokeCommand({
    FunctionName: process.env.WORKER_LAMBDA_NAME || 'chuaikan-cpu-worker',
    InvocationType: 'RequestResponse',
    Payload: Buffer.from(JSON.stringify({ task, payload })),
  });
  
  const response = await lambda.send(command);
  
  if (response.FunctionError) {
    const error = JSON.parse(Buffer.from(response.Payload).toString());
    throw new Error(`Lambda error: ${error.errorMessage}`);
  }
  
  return JSON.parse(Buffer.from(response.Payload).toString());
}

// Lambda function code (deploy แยก)
// lambda/cpu-worker/index.js
exports.handler = async (event) => {
  const { task, payload } = event;
  
  switch (task) {
    case 'processImage':
      // หนัก CPU มากๆ → รันใน Lambda ได้ (max 15 min, up to 10GB RAM)
      const sharp = require('sharp');
      const processed = await sharp(Buffer.from(payload.imageBase64, 'base64'))
        .resize(1200, 630, { fit: 'cover' })
        .webp({ quality: 85 })
        .toBuffer();
      return { imageBase64: processed.toString('base64') };
    
    case 'generatePDF':
      // PDF generation ก็ใช้ CPU สูง
      // const puppeteer = require('puppeteer-core');
      // ...
      return { pdfBase64: '...' };
    
    default:
      throw new Error(`Unknown task: ${task}`);
  }
};
```

```yaml
# serverless.yml - Deploy Lambda worker
service: chuaikan-cpu-worker

provider:
  name: aws
  runtime: nodejs20.x
  region: ap-southeast-1
  memorySize: 1024     # 1GB RAM สำหรับ image processing
  timeout: 30          # 30 seconds max

functions:
  cpu-worker:
    handler: lambda/cpu-worker/index.handler
    environment:
      NODE_ENV: production
    layers:
      - arn:aws:lambda:ap-southeast-1:...  # sharp layer
```

### Step 627: Caching Computed Results

```javascript
// computed-cache.js - Cache CPU-intensive computations
const { LRUCache } = require('lru-cache');
const crypto = require('crypto');

// Cache สำหรับ expensive computations
const computedCache = new LRUCache({
  max: 500,
  ttl: 1000 * 60 * 60, // 1 hour TTL
  updateAgeOnGet: true,
  updateAgeOnHas: true,
  // Calculate size based on result
  sizeCalculation: (value) => {
    return Buffer.byteLength(JSON.stringify(value));
  },
  maxSize: 50 * 1024 * 1024, // 50MB max cache size
});

function computeHeavy(input) {
  // Create deterministic cache key
  const cacheKey = crypto
    .createHash('sha256')
    .update(JSON.stringify(input))
    .digest('hex');
  
  // Check cache first
  const cached = computedCache.get(cacheKey);
  if (cached !== undefined) {
    return { result: cached, fromCache: true };
  }
  
  // Expensive computation
  const result = performExpensiveComputation(input);
  
  // Cache the result
  computedCache.set(cacheKey, result);
  
  return { result, fromCache: false };
}

function performExpensiveComputation(input) {
  // Simulate expensive computation (e.g., complex analytics)
  let result = 0;
  for (let i = 0; i < input.iterations; i++) {
    result += Math.sin(i) * Math.cos(i) * input.factor;
  }
  return result;
}

// Express endpoint
const express = require('express');
const app = express();
app.use(express.json());

app.post('/api/compute', (req, res) => {
  const start = Date.now();
  const { result, fromCache } = computeHeavy(req.body);
  
  res.json({
    result,
    fromCache,
    duration: Date.now() - start,
    cacheStats: {
      size: computedCache.size,
      calculatedSize: computedCache.calculatedSize,
    }
  });
});
```

### Step 628: Database Query CPU Impact

```javascript
// db-cpu-impact.js - Database queries มีผลต่อ CPU มาก
const { Pool } = require('pg');

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,
});

// ❌ BAD: N+1 Query ทำให้ CPU สูงจากการ process หลาย results
async function getUsersWithPostsBad() {
  const users = await pool.query('SELECT * FROM users LIMIT 100');
  
  // N+1 problem: 100 users = 100 separate queries!
  const usersWithPosts = await Promise.all(
    users.rows.map(async (user) => {
      const posts = await pool.query(
        'SELECT * FROM posts WHERE user_id = $1',
        [user.id]
      );
      return { ...user, posts: posts.rows };
    })
  );
  
  return usersWithPosts;
}

// ✅ GOOD: Single JOIN query - ลด CPU และ network overhead
async function getUsersWithPostsGood() {
  const result = await pool.query(`
    SELECT 
      u.id,
      u.name,
      u.email,
      json_agg(
        json_build_object(
          'id', p.id,
          'title', p.title,
          'created_at', p.created_at
        ) ORDER BY p.created_at DESC
      ) FILTER (WHERE p.id IS NOT NULL) AS posts
    FROM users u
    LEFT JOIN posts p ON p.user_id = u.id
    WHERE u.created_at > NOW() - INTERVAL '30 days'
    GROUP BY u.id, u.name, u.email
    LIMIT 100
  `);
  
  return result.rows;
}

// ✅ GOOD: Use materialized view สำหรับ expensive aggregations
async function createDailyStatsMaterializedView() {
  await pool.query(`
    CREATE MATERIALIZED VIEW IF NOT EXISTS daily_stats AS
    SELECT 
      DATE(created_at) as date,
      COUNT(*) as total_posts,
      COUNT(DISTINCT user_id) as unique_users,
      AVG(view_count) as avg_views
    FROM posts
    GROUP BY DATE(created_at)
    ORDER BY date DESC;
    
    CREATE UNIQUE INDEX IF NOT EXISTS daily_stats_date_idx ON daily_stats(date);
  `);
}

// Refresh materialized view ตอน low traffic
async function refreshDailyStats() {
  await pool.query('REFRESH MATERIALIZED VIEW CONCURRENTLY daily_stats');
}
```

### Step 629: CPU Metrics และ Monitoring

```javascript
// cpu-metrics.js - Monitor CPU usage
const os = require('os');
const client = require('prom-client');

// CPU usage gauge
const cpuUsageGauge = new client.Gauge({
  name: 'process_cpu_percent',
  help: 'Process CPU usage percentage',
});

const cpuUserGauge = new client.Gauge({
  name: 'process_cpu_user_seconds_total',
  help: 'Total user CPU time',
});

// Worker thread pool stats
const workerThreadsGauge = new client.Gauge({
  name: 'worker_threads_active',
  help: 'Number of active worker threads',
  labelNames: ['pool']
});

// Track CPU usage over time
let lastCpuUsage = process.cpuUsage();
let lastTime = Date.now();

setInterval(() => {
  const currentCpuUsage = process.cpuUsage();
  const currentTime = Date.now();
  
  const elapsedMs = currentTime - lastTime;
  const userDiff = currentCpuUsage.user - lastCpuUsage.user; // microseconds
  const systemDiff = currentCpuUsage.system - lastCpuUsage.system;
  
  // CPU percentage = (CPU time / elapsed time) * 100
  const cpuPercent = ((userDiff + systemDiff) / (elapsedMs * 1000)) * 100;
  
  cpuUsageGauge.set(Math.min(cpuPercent, 100));
  cpuUserGauge.set(currentCpuUsage.user / 1e6);
  
  lastCpuUsage = currentCpuUsage;
  lastTime = currentTime;
}, 5000);

// System-level CPU
function getSystemCpuPercent() {
  const cpus = os.cpus();
  let totalIdle = 0;
  let totalTick = 0;
  
  cpus.forEach(cpu => {
    for (const type in cpu.times) {
      totalTick += cpu.times[type];
    }
    totalIdle += cpu.times.idle;
  });
  
  return ((1 - totalIdle / totalTick) * 100).toFixed(2);
}
```

### Step 630: Complete CPU Optimization Checklist

```bash
# check-cpu-optimization.sh
#!/bin/bash

echo "=== CPU Optimization Check ==="

# Check 1: Verify cluster mode
echo ""
echo "1. PM2 Status:"
pm2 status 2>/dev/null || echo "PM2 not running"

# Check 2: Count worker processes
echo ""
echo "2. Node.js processes:"
ps aux | grep node | grep -v grep | wc -l
echo "CPU cores: $(nproc)"

# Check 3: Check for synchronous operations in hot path
echo ""
echo "3. Checking for sync operations in src/:"
grep -rn "readFileSync\|execSync\|spawnSync" ./src/ --include="*.js" | grep -v test | grep -v comment

# Check 4: CPU usage per process
echo ""
echo "4. Current CPU usage per process:"
ps aux --sort=-%cpu | head -10

# Check 5: Check worker thread configuration
echo ""
echo "5. Worker pool configuration:"
grep -rn "new Piscina\|new Worker\|cluster.fork" ./src/ --include="*.js"
```

---

## 🔧 Configuration Files

### ecosystem.config.js (Production)

```javascript
module.exports = {
  apps: [{
    name: 'chuaikan-api',
    script: './dist/server.js',
    instances: 'max',
    exec_mode: 'cluster',
    env_production: {
      NODE_ENV: 'production',
      PORT: 3000,
      NODE_OPTIONS: '--max-old-space-size=768',
    },
    max_memory_restart: '1G',
    kill_timeout: 10000,
    listen_timeout: 5000,
    autorestart: true,
    max_restarts: 20,
    min_uptime: '10s',
  }]
};
```

### Kubernetes HPA (Horizontal Pod Autoscaler)

```yaml
# hpa.yaml - Auto-scale based on CPU
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: chuaikan-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: chuaikan-api
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70   # scale up ถ้า CPU > 70%
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60        # เพิ่มได้สูงสุด 2 pods ต่อนาที
    scaleDown:
      stabilizationWindowSeconds: 300  # รอ 5 นาทีก่อน scale down
```

---

## 🧪 Testing

```bash
# Test 1: เปรียบเทียบ single process vs cluster
# Single process
node app.js &
autocannon -d 10 -c 100 http://localhost:3000/api/heavy > single.txt

# Kill and restart with cluster
pkill -f "node app.js"
pm2 start ecosystem.config.js
autocannon -d 10 -c 100 http://localhost:3000/api/heavy > cluster.txt

# Compare results
echo "=== Single Process ===" && cat single.txt | grep "Req/Sec"
echo "=== Cluster Mode ===" && cat cluster.txt | grep "Req/Sec"

# Test 2: Worker threads vs blocking
node --expose-gc blocking-demo.js &
autocannon -d 10 -c 100 http://localhost:3000/api/heavy &
# While that runs, test fast endpoint:
autocannon -d 5 -c 10 http://localhost:3000/api/fast
# Observe that /api/fast is also slow when /api/heavy is running
```

---

## ❌ Common Errors & Solutions

### Error 1: EADDRINUSE ใน cluster mode

```javascript
// ปัญหา: ทุก worker พยายาม bind port เดียวกัน
// แก้ไข: Node.js cluster module จัดการ port sharing อัตโนมัติ
// แต่ถ้าใช้ uWebSockets.js หรือ native addons ต้องระวัง

// ✅ Correct: cluster ทำ port sharing ให้อัตโนมัติ
if (cluster.isWorker) {
  app.listen(3000); // ทุก worker listen port เดียวกันได้
}
```

### Error 2: Worker thread communication overhead

```javascript
// ปัญหา: ส่ง large objects ระหว่าง threads ช้าเพราะ serialization
// แก้ไข: ใช้ SharedArrayBuffer สำหรับ large binary data

// ✅ ส่ง buffer ผ่าน transferable objects (zero-copy)
const buffer = new ArrayBuffer(1024 * 1024); // 1MB
worker.postMessage({ buffer }, [buffer]); // transfer ownership
// buffer ใน main thread จะ detached (neutered) หลัง transfer
```

### Error 3: PM2 cluster ไม่ใช้ทุก cores

```bash
# ตรวจสอบว่า PM2 เห็น CPU cores ถูกต้อง
pm2 info app

# ถ้า container มี CPU limit, Node.js จะเห็น cores น้อยกว่า host
# แก้ไข: กำหนด instances แบบ manual
pm2 start app.js -i 4  # กำหนด 4 instances ตรงๆ
```

---

## ✅ Checklist

- [ ] วัด CPU usage baseline ก่อน optimization
- [ ] ระบุ CPU-intensive operations ด้วย Clinic.js Flame
- [ ] ย้าย CPU-intensive tasks ไปใช้ Worker Threads (Piscina pool)
- [ ] เปิดใช้ PM2 cluster mode ใน production
- [ ] ตั้งค่า HPA ใน Kubernetes สำหรับ auto-scaling
- [ ] Implement caching สำหรับ expensive computations (LRU cache)
- [ ] แก้ N+1 query problems ใน database layer
- [ ] ตั้งค่า Lambda/Cloud Functions สำหรับ async CPU tasks
- [ ] Monitor CPU usage ด้วย Prometheus + Grafana
- [ ] ทดสอบ cluster mode performance เทียบกับ single process

---

## 🔗 References

- [Node.js Cluster Module](https://nodejs.org/api/cluster.html)
- [Worker Threads](https://nodejs.org/api/worker_threads.html)
- [Piscina Worker Pool](https://github.com/piscinajs/piscina)
- [PM2 Cluster Mode](https://pm2.keymetrics.io/docs/usage/cluster-mode/)
- [AWS Lambda for Offloading](https://docs.aws.amazon.com/lambda/)
- [Kubernetes HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)

---

*Part 063 | Road to 1,000,000 Users/Day | chuaikan.com*
