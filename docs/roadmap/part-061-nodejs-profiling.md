# Part 061: Profiling Node.js Applications

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Advanced
> **Steps:** 601-610
> **เวลาโดยประมาณ:** 4 ชั่วโมง
> **Prerequisites:** Part 001-060 (Linux, Docker, Kubernetes, Monitoring)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- วิธี Profile Node.js application ด้วย `--prof` flag และ `node --prof-process`
- ใช้ Clinic.js (Doctor, Flame, Bubbleprof) สำหรับ profiling แบบครอบคลุม
- Chrome DevTools remote debugging สำหรับ Node.js
- อ่านและตีความ Flame Graphs
- วัด Event Loop Lag และผลต่อ performance
- เปรียบเทียบ pino vs winston logger performance
- Benchmark ด้วย autocannon

---

## 📖 ทฤษฎีและแนวคิด

### ทำไมต้อง Profile?

เมื่อ application มี traffic 1,000,000 users/day ทุก millisecond มีความหมาย การ profiling ช่วยให้เราค้นหา "hotspot" หรือโค้ดส่วนที่กินเวลาและ CPU มากที่สุด

### Node.js Profiling Tools Overview

```
┌─────────────────────────────────────────────────┐
│           Node.js Profiling Toolchain            │
├───────────────┬─────────────────────────────────┤
│ Built-in      │ --prof, --inspect, perf_events  │
│ Clinic.js     │ Doctor, Flame, Bubbleprof        │
│ Chrome DevTools│ CPU Profile, Memory Timeline   │
│ APM           │ Datadog, New Relic, Sentry       │
└───────────────┴─────────────────────────────────┘
```

### Event Loop Architecture

Node.js ทำงานบน single thread ผ่าน Event Loop ซึ่งมี phases:
1. **timers** - setTimeout, setInterval callbacks
2. **I/O callbacks** - network, file system callbacks
3. **idle/prepare** - internal use
4. **poll** - retrieve new I/O events
5. **check** - setImmediate callbacks
6. **close callbacks** - socket.on('close') etc.

---

## ⚙️ Environment Setup

### ติดตั้ง Clinic.js และ tools ที่จำเป็น

```bash
# ติดตั้ง Clinic.js globally
npm install -g clinic autocannon

# ติดตั้ง dependencies สำหรับ project
mkdir -p /home/user/nodejs-profiling-demo
cd /home/user/nodejs-profiling-demo
npm init -y

# ติดตั้ง packages
npm install express pino winston clinic autocannon

# ตรวจสอบ Node.js version (ควรใช้ v20+)
node --version
npm --version
```

### สร้าง sample application สำหรับ profiling

```javascript
// app.js - Express app with intentional performance issues
const express = require('express');
const pino = require('pino');

const logger = pino({
  level: 'info',
  transport: {
    target: 'pino-pretty',
    options: { colorize: true }
  }
});

const app = express();
app.use(express.json());

// ❌ BAD: Synchronous heavy computation (blocks event loop)
function fibonacci(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}

// ❌ BAD: Unnecessary JSON stringify/parse in hot path
const config = { maxItems: 100, timeout: 5000 };

app.get('/api/users', (req, res) => {
  // simulate database query with sync blocking
  const users = [];
  for (let i = 0; i < 1000; i++) {
    users.push({
      id: i,
      name: `User ${i}`,
      // ❌ BAD: JSON.parse/stringify in loop
      config: JSON.parse(JSON.stringify(config))
    });
  }
  
  logger.info({ count: users.length }, 'Fetched users');
  res.json(users);
});

app.get('/api/fib/:n', (req, res) => {
  const n = parseInt(req.params.n) || 35;
  // ❌ BAD: CPU-intensive synchronous computation
  const result = fibonacci(n);
  res.json({ n, result });
});

app.get('/health', (req, res) => {
  res.json({ status: 'ok', uptime: process.uptime() });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  logger.info(`Server running on port ${PORT}`);
});

module.exports = app;
```

---

## 🛠️ Step-by-Step Implementation

### Step 601: Node.js Built-in Profiler (`--prof`)

`--prof` flag ใช้ V8's built-in sampling profiler ซึ่งบันทึก CPU samples ทุก 1ms

```bash
# รัน app ด้วย --prof flag
node --prof app.js &
APP_PID=$!

# สร้าง load ด้วย autocannon (30 วินาที, 10 concurrent)
autocannon -d 30 -c 10 http://localhost:3000/api/users

# หยุด app
kill $APP_PID

# ไฟล์ isolate-*.log จะถูกสร้างขึ้น
ls -la isolate-*.log
```

```bash
# แปลง profiling data เป็น readable format
node --prof-process isolate-0x*.log > processed-profile.txt

# ดู top functions ที่ใช้ CPU มากที่สุด
cat processed-profile.txt | head -100
```

ผลลัพธ์ที่ได้จะมีลักษณะนี้:
```
Statistical profiling result from isolate-0x...log
 [Shared libraries]:
   ticks  total  nonlib   name
   1234   45.2%         /lib/x86_64-linux-gnu/libc.so.6

 [JavaScript]:
   ticks  total  nonlib   name
    456   16.7%   30.4%  LazyCompile: *fibonacci app.js:8:18
    234    8.6%   15.6%  Builtin: JsonStringify
```

### Step 602: Clinic.js Doctor - Overall Health Check

```bash
# รัน Clinic Doctor (วิเคราะห์ event loop, CPU, memory ทั้งหมด)
clinic doctor -- node app.js &
CLINIC_PID=$!

# สร้าง load
autocannon -d 20 -c 50 http://localhost:3000/api/users

# หยุด process (CTRL+C หรือ kill)
kill $CLINIC_PID

# Doctor จะ generate HTML report อัตโนมัติ
# เปิดดูใน browser: .clinic/*/doctor_*.html
ls -la .clinic/
```

Doctor จะระบุปัญหาต่างๆ เช่น:
- **I/O bottleneck**: Event loop เร็ว แต่ CPU ต่ำ
- **CPU bottleneck**: Event loop ช้า, CPU สูง
- **Memory leak**: Heap เพิ่มขึ้นเรื่อยๆ
- **Event loop delay**: High latency ใน event processing

### Step 603: Clinic.js Flame - Flame Graph

```bash
# รัน Clinic Flame สำหรับ CPU flame graph
clinic flame -- node app.js &

# สร้าง load ที่ CPU-intensive endpoint
autocannon -d 30 -c 20 http://localhost:3000/api/fib/35

# Flame graph จะ show ว่า function ไหนกิน CPU มากที่สุด
ls -la .clinic/
# เปิด .clinic/*/flame_*.html
```

### Step 604: Clinic.js Bubbleprof - Async Profiling

Bubbleprof วิเคราะห์ async operations และ delays ใน I/O

```bash
# รัน Clinic Bubbleprof
clinic bubbleprof -- node app.js &

# สร้าง load ที่มี async operations
autocannon -d 20 -c 30 http://localhost:3000/api/users

# ดู bubble diagram
ls -la .clinic/
# เปิด .clinic/*/bubbleprof_*.html
```

### Step 605: Chrome DevTools Remote Debugging

```bash
# รัน Node.js ด้วย --inspect flag
node --inspect=0.0.0.0:9229 app.js

# หรือใช้ --inspect-brk เพื่อ break ทันที
node --inspect-brk=0.0.0.0:9229 app.js
```

```bash
# ถ้ารันบน remote server ให้ทำ SSH tunnel
ssh -L 9229:localhost:9229 user@your-server.com

# จากนั้นเปิด Chrome และไปที่:
# chrome://inspect
# คลิก "Open dedicated DevTools for Node"
```

การใช้ Chrome DevTools CPU Profiler:
1. เปิด Chrome DevTools สำหรับ Node.js
2. ไปที่ **Profiler** tab
3. คลิก **Start** เพื่อเริ่ม recording
4. สร้าง load ด้วย autocannon
5. คลิก **Stop** เพื่อหยุด
6. วิเคราะห์ flame chart และ heavy (bottom up) view

### Step 606: อ่าน Flame Graph

```
Flame Graph อ่านจาก bottom to top:
┌────────────────────────────────────────┐
│  fibonacci (wide = กิน CPU มาก!)       │ ← TOP: function ที่กำลังทำงาน
├────────────────────────────────────────┤
│  fibonacci                              │
├────────────────────────────────────────┤
│  /api/fib/:n handler                   │
├────────────────────────────────────────┤
│  Layer#handle                          │
├────────────────────────────────────────┤
│  process events (Node.js internals)    │ ← BOTTOM: event loop
└────────────────────────────────────────┘
ความกว้างของแต่ละ bar = % ของ CPU time
```

### Step 607: Event Loop Lag Measurement

```javascript
// event-loop-monitor.js
const { monitorEventLoopDelay } = require('perf_hooks');

// Method 1: monitorEventLoopDelay (Node.js v12+)
const histogram = monitorEventLoopDelay({ resolution: 10 });
histogram.enable();

setInterval(() => {
  const stats = {
    min: (histogram.min / 1e6).toFixed(2),     // nanoseconds to ms
    max: (histogram.max / 1e6).toFixed(2),
    mean: (histogram.mean / 1e6).toFixed(2),
    p50: (histogram.percentile(50) / 1e6).toFixed(2),
    p99: (histogram.percentile(99) / 1e6).toFixed(2),
  };
  
  console.log('Event Loop Delay (ms):', stats);
  
  // Alert ถ้า p99 > 100ms (หมายความว่า event loop ถูก block)
  if (parseFloat(stats.p99) > 100) {
    console.error('⚠️  High event loop lag detected!', stats.p99, 'ms');
  }
  
  histogram.reset();
}, 5000);

// Method 2: Simple interval-based measurement
let lastCheck = Date.now();
const EXPECTED_INTERVAL = 1000; // 1 second

setInterval(() => {
  const now = Date.now();
  const lag = now - lastCheck - EXPECTED_INTERVAL;
  lastCheck = now;
  
  if (lag > 50) {
    console.warn(`Event loop lag: ${lag}ms`);
  }
}, EXPECTED_INTERVAL);
```

```bash
# รัน event loop monitor พร้อม load test
node event-loop-monitor.js &

# สร้าง CPU-intensive load เพื่อดู lag
autocannon -d 10 -c 100 http://localhost:3000/api/fib/40
```

### Step 608: Pino vs Winston Performance Benchmark

```javascript
// logger-benchmark.js
const pino = require('pino');
const winston = require('winston');
const { performance } = require('perf_hooks');

// Setup pino
const pinoLogger = pino({
  level: 'info',
  // ใช้ async transport ใน production
  transport: process.env.NODE_ENV !== 'production' ? {
    target: 'pino-pretty'
  } : undefined
});

// Setup winston
const winstonLogger = winston.createLogger({
  level: 'info',
  format: winston.format.json(),
  transports: [
    new winston.transports.File({ filename: '/dev/null' }) // discard output
  ]
});

// Benchmark function
function benchmark(name, fn, iterations = 100000) {
  const start = performance.now();
  for (let i = 0; i < iterations; i++) {
    fn(i);
  }
  const end = performance.now();
  const duration = end - start;
  console.log(`${name}: ${duration.toFixed(2)}ms for ${iterations} iterations`);
  console.log(`  Avg: ${(duration / iterations * 1000).toFixed(2)}µs per log`);
  return duration;
}

// Run benchmarks
console.log('=== Logger Performance Benchmark ===\n');

const pinoTime = benchmark('Pino', (i) => {
  pinoLogger.info({ userId: i, action: 'login', ip: '192.168.1.1' }, 'User logged in');
});

const winstonTime = benchmark('Winston', (i) => {
  winstonLogger.info('User logged in', { userId: i, action: 'login', ip: '192.168.1.1' });
});

console.log(`\nPino is ${(winstonTime / pinoTime).toFixed(1)}x faster than Winston`);
```

```bash
node logger-benchmark.js
# ผลลัพธ์ที่คาดหวัง:
# Pino: ~800ms for 100,000 iterations
# Winston: ~3200ms for 100,000 iterations
# Pino is ~4x faster than Winston
```

### Step 609: Autocannon Benchmark

```bash
# Basic benchmark
autocannon http://localhost:3000/api/users

# ปรับแต่ง parameters
autocannon \
  --connections 100 \    # concurrent connections
  --duration 30 \        # test duration (seconds)
  --pipelining 1 \       # HTTP pipelining
  --timeout 10 \         # request timeout
  http://localhost:3000/api/users

# Compare endpoints
autocannon -d 10 -c 50 http://localhost:3000/api/users > users.txt
autocannon -d 10 -c 50 http://localhost:3000/api/fib/30 > fib.txt
```

```javascript
// autocannon-programmatic.js - ใช้ autocannon ใน code
const autocannon = require('autocannon');

async function runBenchmark(url, options = {}) {
  const result = await autocannon({
    url,
    connections: options.connections || 10,
    duration: options.duration || 10,
    ...options
  });
  
  return {
    url,
    requests: result.requests.average,
    latency: {
      p50: result.latency.p50,
      p95: result.latency.p95,
      p99: result.latency.p99,
    },
    throughput: (result.throughput.average / 1024 / 1024).toFixed(2) + ' MB/s',
    errors: result.errors,
    timeouts: result.timeouts,
  };
}

async function main() {
  console.log('Running benchmarks...\n');
  
  const endpoints = [
    { url: 'http://localhost:3000/api/users', connections: 50 },
    { url: 'http://localhost:3000/health', connections: 100 },
  ];
  
  for (const endpoint of endpoints) {
    const result = await runBenchmark(endpoint.url, { connections: endpoint.connections });
    console.log(`Endpoint: ${result.url}`);
    console.log(`  Requests/sec: ${result.requests}`);
    console.log(`  Latency P50: ${result.latency.p50}ms, P95: ${result.latency.p95}ms, P99: ${result.latency.p99}ms`);
    console.log(`  Throughput: ${result.throughput}`);
    console.log(`  Errors: ${result.errors}, Timeouts: ${result.timeouts}\n`);
  }
}

main().catch(console.error);
```

### Step 610: Optimization ตามผล Profiling

```javascript
// app-optimized.js - หลังจาก profiling และ fix

const express = require('express');
const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');
const pino = require('pino');

const logger = pino({ level: 'info' });
const app = express();
app.use(express.json());

// ✅ GOOD: Cache config object (ไม่ต้อง stringify/parse ซ้ำๆ)
const config = Object.freeze({ maxItems: 100, timeout: 5000 });

// ✅ GOOD: Pre-generate user data แทนการสร้างในทุก request
const cachedUsers = Array.from({ length: 1000 }, (_, i) => ({
  id: i,
  name: `User ${i}`,
  config  // reference แทน copy
}));

app.get('/api/users', (req, res) => {
  // ✅ GOOD: Return cached data
  logger.info({ count: cachedUsers.length }, 'Fetched users');
  res.json(cachedUsers);
});

// ✅ GOOD: Fibonacci ใน Worker Thread (ไม่ block event loop)
function fibWorker(n) {
  return new Promise((resolve, reject) => {
    const worker = new Worker(`
      const { parentPort, workerData } = require('worker_threads');
      function fib(n) {
        if (n <= 1) return n;
        return fib(n-1) + fib(n-2);
      }
      parentPort.postMessage(fib(workerData.n));
    `, { eval: true, workerData: { n } });
    
    worker.on('message', resolve);
    worker.on('error', reject);
    worker.on('exit', (code) => {
      if (code !== 0) reject(new Error(`Worker stopped with exit code ${code}`));
    });
  });
}

app.get('/api/fib/:n', async (req, res) => {
  const n = parseInt(req.params.n) || 35;
  // ✅ GOOD: รัน CPU-intensive task ใน worker thread
  const result = await fibWorker(n);
  res.json({ n, result });
});

app.get('/health', (req, res) => {
  res.json({ status: 'ok', uptime: process.uptime() });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  logger.info(`Optimized server running on port ${PORT}`);
});
```

---

## 🔧 Configuration Files

### package.json

```json
{
  "name": "nodejs-profiling-demo",
  "version": "1.0.0",
  "scripts": {
    "start": "node app.js",
    "start:profile": "node --prof app.js",
    "start:inspect": "node --inspect=0.0.0.0:9229 app.js",
    "start:clinic-doctor": "clinic doctor -- node app.js",
    "start:clinic-flame": "clinic flame -- node app.js",
    "start:clinic-bubble": "clinic bubbleprof -- node app.js",
    "bench": "autocannon -d 30 -c 100 http://localhost:3000/api/users",
    "bench:fib": "autocannon -d 30 -c 50 http://localhost:3000/api/fib/35"
  },
  "dependencies": {
    "express": "^4.18.2",
    "pino": "^8.17.2",
    "pino-pretty": "^10.3.1",
    "winston": "^3.11.0"
  },
  "devDependencies": {
    "autocannon": "^7.14.0",
    "clinic": "^13.0.0"
  }
}
```

### .clinic/.gitignore

```gitignore
# อย่า commit profiling data
*.clinic/
isolate-*.log
*.cpuprofile
*.heapsnapshot
```

---

## 🧪 Testing

### Performance Regression Test

```javascript
// perf-regression-test.js
const autocannon = require('autocannon');
const assert = require('assert');

const PERFORMANCE_THRESHOLDS = {
  '/api/users': {
    minRPS: 500,        // minimum requests per second
    maxP95: 100,        // P95 latency ต้องไม่เกิน 100ms
    maxErrors: 0,       // ไม่ยอมรับ errors
  },
  '/health': {
    minRPS: 5000,
    maxP95: 10,
    maxErrors: 0,
  }
};

async function runPerfTest(endpoint, threshold) {
  const result = await autocannon({
    url: `http://localhost:3000${endpoint}`,
    connections: 50,
    duration: 15,
  });
  
  const rps = result.requests.average;
  const p95 = result.latency.p95;
  const errors = result.errors;
  
  console.log(`\n${endpoint}:`);
  console.log(`  RPS: ${rps} (min: ${threshold.minRPS})`);
  console.log(`  P95: ${p95}ms (max: ${threshold.maxP95}ms)`);
  console.log(`  Errors: ${errors}`);
  
  // Assert thresholds
  assert(rps >= threshold.minRPS, 
    `RPS ${rps} below minimum ${threshold.minRPS}`);
  assert(p95 <= threshold.maxP95,
    `P95 ${p95}ms exceeds maximum ${threshold.maxP95}ms`);
  assert(errors <= threshold.maxErrors,
    `Errors ${errors} exceeds maximum ${threshold.maxErrors}`);
  
  console.log(`  ✅ PASSED`);
}

async function main() {
  console.log('=== Performance Regression Tests ===');
  
  for (const [endpoint, threshold] of Object.entries(PERFORMANCE_THRESHOLDS)) {
    await runPerfTest(endpoint, threshold);
  }
  
  console.log('\n✅ All performance tests passed!');
}

main().catch((err) => {
  console.error('\n❌ Performance test FAILED:', err.message);
  process.exit(1);
});
```

```bash
# รัน performance regression test
node app.js &
sleep 2
node perf-regression-test.js
```

---

## ❌ Common Errors & Solutions

### Error 1: `clinic` command not found

```bash
# ปัญหา: clinic ไม่ได้ติดตั้ง globally
$ clinic doctor -- node app.js
bash: clinic: command not found

# แก้ไข:
npm install -g clinic
# หรือใช้ npx
npx clinic doctor -- node app.js
```

### Error 2: Inspector port already in use

```bash
# ปัญหา: port 9229 ถูกใช้งานอยู่แล้ว
$ node --inspect=9229 app.js
Starting inspector on 127.0.0.1:9229 failed: address already in use

# แก้ไข: ใช้ port อื่น
node --inspect=9230 app.js

# หรือ kill process ที่ใช้ port 9229
lsof -ti:9229 | xargs kill
```

### Error 3: Flame graph ว่างเปล่า

```bash
# ปัญหา: profile time สั้นเกินไป / load ไม่พอ
# แก้ไข: เพิ่ม duration และ connections
clinic flame -- node app.js &
autocannon -d 60 -c 100 http://localhost:3000/api/users  # เพิ่มเป็น 60s
```

### Error 4: Worker threads ทำงานช้ากว่า expected

```javascript
// ปัญหา: Worker thread overhead สูงเกินไปสำหรับ small tasks
// Worker threads เหมาะกับ CPU tasks ที่ใช้เวลา > 5ms เท่านั้น

// แก้ไข: ใช้ Worker Pool แทนการสร้าง worker ใหม่ทุกครั้ง
const { Piscina } = require('piscina');

const pool = new Piscina({
  filename: path.resolve(__dirname, 'worker.js'),
  maxThreads: require('os').cpus().length
});

app.get('/api/fib/:n', async (req, res) => {
  const n = parseInt(req.params.n) || 35;
  const result = await pool.run({ n });
  res.json({ n, result });
});
```

---

## ✅ Checklist

- [ ] ติดตั้ง Clinic.js และ autocannon
- [ ] รัน `clinic doctor` และวิเคราะห์ผล (CPU/Memory/IO bottleneck)
- [ ] รัน `clinic flame` และอ่าน flame graph
- [ ] รัน `clinic bubbleprof` และวิเคราะห์ async issues
- [ ] วัด event loop lag ด้วย `monitorEventLoopDelay`
- [ ] ย้ายจาก winston ไป pino (ถ้ายังใช้ winston อยู่)
- [ ] เปลี่ยน CPU-intensive operations ไปใช้ Worker Threads
- [ ] ตั้งค่า Chrome DevTools remote debugging สำหรับ production debugging
- [ ] สร้าง performance baseline ด้วย autocannon
- [ ] เขียน performance regression tests ใน CI/CD pipeline

---

## 🔗 References

- [Node.js Profiling Guide](https://nodejs.org/en/docs/guides/simple-profiling)
- [Clinic.js Documentation](https://clinicjs.org/documentation/)
- [Node.js Worker Threads](https://nodejs.org/api/worker_threads.html)
- [Pino Logger](https://getpino.io/)
- [Autocannon](https://github.com/mcollina/autocannon)
- [Flame Graph Interpretation](https://www.brendangregg.com/flamegraphs.html)

---

*Part 061 | Road to 1,000,000 Users/Day | chuaikan.com*
