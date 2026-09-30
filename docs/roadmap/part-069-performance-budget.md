# Part 069: Performance Budget

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Advanced
> **Steps:** 681-690
> **เวลาโดยประมาณ:** 3 ชั่วโมง
> **Prerequisites:** Part 066-068

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- Performance budget คืออะไรและทำไมต้องมี
- Bundle size budget
- Lighthouse score targets
- Core Web Vitals budget
- API response time budget
- Database query budget
- Performance budget enforcement ใน CI
- Budget monitoring dashboard

---

## 📖 ทฤษฎีและแนวคิด

### Performance Budget คืออะไร?

Performance budget คือ "งบประมาณ" ที่กำหนดขีดจำกัดสำหรับ performance metrics ต่างๆ เช่นเดียวกับ financial budget มันช่วยให้ team มี shared standards และป้องกัน performance regression

```
Performance Budget Matrix for chuaikan.com:

┌─────────────────────────────────────────────────────┐
│ Category          │ Budget    │ Rationale             │
├───────────────────┼───────────┼───────────────────────┤
│ JS Bundle (total) │ 300KB gz  │ < 3s on 3G mobile     │
│ CSS (total)       │ 50KB gz   │ critical CSS inline   │
│ Image (per image) │ 200KB     │ WebP/AVIF compressed  │
│ Total Page Weight │ 1MB       │ 3G target             │
├───────────────────┼───────────┼───────────────────────┤
│ LCP               │ < 2.5s    │ Google's good range   │
│ INP               │ < 200ms   │ Google's good range   │
│ CLS               │ < 0.1     │ Google's good range   │
├───────────────────┼───────────┼───────────────────────┤
│ API P95 latency   │ < 200ms   │ User experience       │
│ API P99 latency   │ < 500ms   │ Tail latency          │
│ DB query max      │ 50ms      │ Database health       │
│ Cache hit rate    │ > 80%     │ Redis efficiency      │
└───────────────────┴───────────┴───────────────────────┘
```

---

## ⚙️ Environment Setup

```bash
# ติดตั้ง tools สำหรับ budget enforcement
npm install -D \
  bundlesize \
  @bundlesize/core \
  size-limit \
  @size-limit/preset-app \
  lighthouse \
  @lhci/cli

# ติดตั้ง webpack-bundle-analyzer
npm install -D webpack-bundle-analyzer

# สร้าง budget config files
mkdir -p .budgets
```

---

## 🛠️ Step-by-Step Implementation

### Step 681: Bundle Size Budget

```json
// bundlesize.config.json
{
  "files": [
    {
      "path": "./.next/static/chunks/framework-*.js",
      "maxSize": "120 kB",
      "compression": "gzip"
    },
    {
      "path": "./.next/static/chunks/main-*.js",
      "maxSize": "80 kB",
      "compression": "gzip"
    },
    {
      "path": "./.next/static/css/*.css",
      "maxSize": "50 kB",
      "compression": "gzip"
    },
    {
      "path": "./.next/static/chunks/pages/**/*.js",
      "maxSize": "50 kB",
      "compression": "gzip"
    }
  ]
}
```

```json
// .size-limit.json - Alternative: size-limit package
[
  {
    "name": "App JavaScript",
    "path": ".next/static/chunks/*.js",
    "limit": "300 kB",
    "gzip": true
  },
  {
    "name": "App CSS",
    "path": ".next/static/css/*.css",
    "limit": "50 kB",
    "gzip": true
  },
  {
    "name": "Critical chunk",
    "path": ".next/static/chunks/main*.js",
    "limit": "80 kB",
    "gzip": true
  }
]
```

```bash
# ตรวจสอบ bundle sizes
npm run build

# ใช้ bundlesize
npx bundlesize

# ใช้ size-limit
npx size-limit

# ดู bundle composition
ANALYZE=true npm run build
# เปิด .next/analyze/client.html
```

### Step 682: Lighthouse Score Targets

```javascript
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      url: [
        'http://localhost:3000/',
        'http://localhost:3000/explore',
        'http://localhost:3000/sos',
      ],
      numberOfRuns: 3,
      settings: {
        // Mobile simulation
        formFactor: 'mobile',
        throttling: {
          rttMs: 40,
          throughputKbps: 10240,
          cpuSlowdownMultiplier: 4,
        },
        screenEmulation: {
          mobile: true,
          width: 390,
          height: 844,
          deviceScaleFactor: 3,
          disabled: false,
        },
      },
    },
    
    assert: {
      // Performance budgets per metric
      assertions: {
        // Category scores
        'categories:performance': ['error', { minScore: 0.85 }],       // 85+
        'categories:accessibility': ['error', { minScore: 0.95 }],     // 95+
        'categories:best-practices': ['warn', { minScore: 0.90 }],     // 90+
        'categories:seo': ['warn', { minScore: 0.90 }],                // 90+
        
        // Core Web Vitals
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-blocking-time': ['error', { maxNumericValue: 300 }],
        'first-contentful-paint': ['warn', { maxNumericValue: 1800 }],
        'interactive': ['warn', { maxNumericValue: 3800 }],
        
        // Bundle sizes
        'total-byte-weight': ['error', { maxNumericValue: 1048576 }],  // 1MB total
        'uses-optimized-images': 'off',       // handled by next/image
        'uses-responsive-images': 'off',      // handled by next/image
        
        // Performance opportunities
        'unused-javascript': ['warn', { maxNumericValue: 100000 }],    // < 100KB unused JS
        'unused-css-rules': ['warn', { maxNumericValue: 25000 }],      // < 25KB unused CSS
        
        // Resource counts
        'resource-summary:script:count': ['warn', { maxNumericValue: 20 }],  // max 20 JS files
        'resource-summary:stylesheet:count': ['warn', { maxNumericValue: 5 }],
      },
    },
  },
};
```

### Step 683: Core Web Vitals Budget Enforcement

```typescript
// src/lib/cwv-budget.ts - Real-time CWV monitoring กับ budget alerts
import { onCLS, onINP, onLCP, type Metric } from 'web-vitals';

const BUDGETS = {
  LCP: { good: 2500, needsImprovement: 4000 },
  INP: { good: 200, needsImprovement: 500 },
  CLS: { good: 0.1, needsImprovement: 0.25 },
  FCP: { good: 1800, needsImprovement: 3000 },
  TTFB: { good: 800, needsImprovement: 1800 },
} as const;

type VitalName = keyof typeof BUDGETS;

function evaluateBudget(metric: Metric): {
  within_budget: boolean;
  rating: 'good' | 'needs-improvement' | 'poor';
  budget: number;
  actual: number;
  overage_percent: number;
} {
  const budget = BUDGETS[metric.name as VitalName];
  if (!budget) return { within_budget: true, rating: 'good', budget: 0, actual: 0, overage_percent: 0 };
  
  const within_budget = metric.value <= budget.good;
  const overage_percent = within_budget 
    ? 0 
    : ((metric.value - budget.good) / budget.good * 100);
  
  return {
    within_budget,
    rating: metric.rating,
    budget: budget.good,
    actual: metric.value,
    overage_percent: Math.round(overage_percent),
  };
}

export function initCWVBudgetMonitor() {
  const reportVital = (metric: Metric) => {
    const evaluation = evaluateBudget(metric);
    
    // Send to analytics
    fetch('/api/vitals', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ metric, evaluation }),
      keepalive: true,
    }).catch(() => {});
    
    // Alert in development
    if (process.env.NODE_ENV === 'development' && !evaluation.within_budget) {
      console.warn(
        `🚨 CWV Budget Exceeded: ${metric.name}\n` +
        `  Budget: ${evaluation.budget}${metric.name === 'CLS' ? '' : 'ms'}\n` +
        `  Actual: ${evaluation.actual.toFixed(metric.name === 'CLS' ? 3 : 0)}${metric.name === 'CLS' ? '' : 'ms'}\n` +
        `  Over by: ${evaluation.overage_percent}%`
      );
    }
  };
  
  onCLS(reportVital, { reportAllChanges: false });
  onINP(reportVital, { reportAllChanges: true });
  onLCP(reportVital, { reportAllChanges: false });
}
```

### Step 684: API Response Time Budget

```javascript
// middleware/response-time-budget.js
const client = require('prom-client');

// Budget thresholds
const API_BUDGETS = {
  '/api/health': { p95: 50, p99: 100 },
  '/api/posts': { p95: 200, p99: 500 },
  '/api/users': { p95: 150, p99: 300 },
  '/api/search': { p95: 500, p99: 1000 },
  '/api/sos': { p95: 300, p99: 600 },    // SOS เผื่อเวลา process มากกว่า
  'default': { p95: 200, p99: 500 },
};

// Track response times per endpoint
const endpointTimings = new Map();

// Prometheus histogram สำหรับ budget monitoring
const apiResponseTime = new client.Histogram({
  name: 'api_response_time_ms',
  help: 'API response time in milliseconds',
  labelNames: ['method', 'path', 'status_code'],
  buckets: [10, 25, 50, 100, 200, 500, 1000, 2000, 5000],
});

const budgetViolations = new client.Counter({
  name: 'api_budget_violations_total',
  help: 'Number of API response time budget violations',
  labelNames: ['path', 'budget_type'],
});

function responseBudgetMiddleware(req, res, next) {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = Date.now() - start;
    const path = req.route?.path || req.path.replace(/\/\d+/g, '/:id');
    
    // Record metrics
    apiResponseTime.observe(
      { method: req.method, path, status_code: res.statusCode },
      duration
    );
    
    // Track per-endpoint timing window (sliding window)
    if (!endpointTimings.has(path)) {
      endpointTimings.set(path, []);
    }
    const timings = endpointTimings.get(path);
    timings.push(duration);
    
    // Keep only last 1000 samples
    if (timings.length > 1000) {
      timings.shift();
    }
    
    // Check budget violation
    const budget = API_BUDGETS[path] || API_BUDGETS.default;
    
    // Calculate current percentiles
    const sorted = [...timings].sort((a, b) => a - b);
    const p95 = sorted[Math.floor(sorted.length * 0.95)];
    const p99 = sorted[Math.floor(sorted.length * 0.99)];
    
    if (p95 > budget.p95 && timings.length >= 100) {
      budgetViolations.inc({ path, budget_type: 'p95' });
      console.warn(`Budget violation: ${path} P95=${p95}ms > budget ${budget.p95}ms`);
    }
    
    if (p99 > budget.p99 && timings.length >= 100) {
      budgetViolations.inc({ path, budget_type: 'p99' });
      console.warn(`Budget violation: ${path} P99=${p99}ms > budget ${budget.p99}ms`);
    }
  });
  
  next();
}

module.exports = responseBudgetMiddleware;
```

### Step 685: Database Query Budget

```javascript
// prisma-query-budget.js - Monitor database query execution times
const { PrismaClient } = require('@prisma/client');

const DB_QUERY_BUDGET_MS = 50; // max 50ms per query

function createBudgetAwarePrisma() {
  const prisma = new PrismaClient({
    log: [
      { level: 'query', emit: 'event' },
      { level: 'warn', emit: 'stdout' },
      { level: 'error', emit: 'stdout' },
    ],
  });
  
  // Monitor query execution times
  prisma.$on('query', (event) => {
    const duration = event.duration;
    
    // Log slow queries
    if (duration > DB_QUERY_BUDGET_MS) {
      console.warn(`🐌 SLOW QUERY (${duration}ms):\n  ${event.query}`);
      
      // Increment violation counter
      slowQueryCounter.inc({
        duration_bucket: duration > 500 ? '>500ms' : 
                         duration > 200 ? '200-500ms' : 
                         '50-200ms'
      });
    }
    
    // Track for percentile calculation
    queryDurations.push(duration);
    if (queryDurations.length > 10000) queryDurations.shift();
  });
  
  return prisma;
}

// Prometheus metrics
const client = require('prom-client');

const slowQueryCounter = new client.Counter({
  name: 'db_slow_queries_total',
  help: 'Number of slow database queries',
  labelNames: ['duration_bucket'],
});

const queryDurationHistogram = new client.Histogram({
  name: 'db_query_duration_ms',
  help: 'Database query duration in milliseconds',
  buckets: [5, 10, 25, 50, 100, 250, 500, 1000],
});

let queryDurations = [];

// Periodic report
setInterval(() => {
  if (queryDurations.length < 10) return;
  
  const sorted = [...queryDurations].sort((a, b) => a - b);
  const n = sorted.length;
  
  const report = {
    count: n,
    p50: sorted[Math.floor(n * 0.50)],
    p95: sorted[Math.floor(n * 0.95)],
    p99: sorted[Math.floor(n * 0.99)],
    over_budget: sorted.filter(d => d > DB_QUERY_BUDGET_MS).length,
    budget_compliance: `${((1 - sorted.filter(d => d > DB_QUERY_BUDGET_MS).length / n) * 100).toFixed(1)}%`,
  };
  
  if (report.p95 > DB_QUERY_BUDGET_MS) {
    console.warn('DB Query Budget Status:', report);
  }
  
  queryDurations = [];  // Reset for next window
}, 60000);  // Report every minute
```

### Step 686: Performance Budget ใน CI/CD

```yaml
# .github/workflows/performance-budget.yml
name: Performance Budget Check

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  bundle-size-check:
    runs-on: ubuntu-latest
    name: Bundle Size Budget
    
    steps:
    - uses: actions/checkout@v4
    
    - uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'
    
    - run: npm ci
    
    - name: Build
      run: npm run build
      
    - name: Check bundle sizes
      run: npx bundlesize
      # Fails if any bundle exceeds budget
    
    - name: Size limit check
      run: npx size-limit
    
    - name: Upload bundle stats
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: bundle-stats
        path: .next/analyze/

  lighthouse-budget:
    runs-on: ubuntu-latest
    name: Lighthouse Budget
    needs: bundle-size-check
    
    steps:
    - uses: actions/checkout@v4
    
    - uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'
    
    - run: npm ci && npm run build
    
    - name: Start server
      run: |
        npm run start &
        npx wait-on http://localhost:3000 --timeout 30000
    
    - name: Lighthouse Budget Check
      run: npx lhci autorun
      env:
        LHCI_GITHUB_APP_TOKEN: ${{ secrets.LHCI_GITHUB_APP_TOKEN }}
    
    - name: Upload Lighthouse results
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: lighthouse-budget-results
        path: .lighthouseci/

  api-performance-budget:
    runs-on: ubuntu-latest
    name: API Performance Budget
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v4
    
    - uses: actions/setup-node@v4
      with:
        node-version: '20'
    
    - run: npm install -g autocannon
    
    - name: API Response Time Budget Check
      run: |
        # Run quick performance test against staging
        autocannon \
          --connections 10 \
          --duration 30 \
          --json \
          https://staging.chuaikan.com/api/health > /tmp/perf-result.json
        
        # Check P95 is within budget
        node -e "
          const result = require('/tmp/perf-result.json');
          const p95 = result.latency.p95;
          const BUDGET = 200;
          console.log('API P95 Latency:', p95 + 'ms (budget: ' + BUDGET + 'ms)');
          if (p95 > BUDGET) {
            console.error('BUDGET EXCEEDED: ' + p95 + 'ms > ' + BUDGET + 'ms');
            process.exit(1);
          }
          console.log('Budget check passed!');
        "
```

### Step 687: Budget Monitoring Dashboard

```javascript
// budget-dashboard.js - API สำหรับ budget monitoring
const express = require('express');
const client = require('prom-client');

const app = express();

// Budget definitions (same as thresholds)
const BUDGETS = {
  api: {
    p95_ms: 200,
    p99_ms: 500,
    error_rate_percent: 1,
  },
  database: {
    max_query_ms: 50,
    slow_query_threshold_ms: 100,
    connection_pool_usage_percent: 80,
  },
  cache: {
    hit_rate_percent: 80,
    max_latency_ms: 5,
  },
  frontend: {
    lcp_ms: 2500,
    inp_ms: 200,
    cls: 0.1,
    bundle_size_kb: 300,
  },
};

// Budget status endpoint
app.get('/api/budget-status', async (req, res) => {
  // Get current metrics
  const metrics = await getCurrentMetrics();
  
  const status = {
    timestamp: new Date().toISOString(),
    overall: 'green',
    budgets: {
      api: {
        p95_ms: {
          budget: BUDGETS.api.p95_ms,
          current: metrics.api.p95,
          status: metrics.api.p95 <= BUDGETS.api.p95_ms ? 'green' : 'red',
        },
        error_rate: {
          budget: BUDGETS.api.error_rate_percent,
          current: metrics.api.errorRate,
          status: metrics.api.errorRate <= BUDGETS.api.error_rate_percent ? 'green' : 'red',
        },
      },
      database: {
        slow_queries: {
          budget: 5,  // max 5 slow queries per minute
          current: metrics.db.slowQueriesPerMinute,
          status: metrics.db.slowQueriesPerMinute <= 5 ? 'green' : 'yellow',
        },
      },
      cache: {
        hit_rate: {
          budget: BUDGETS.cache.hit_rate_percent,
          current: metrics.cache.hitRate,
          status: metrics.cache.hitRate >= BUDGETS.cache.hit_rate_percent ? 'green' : 'yellow',
        },
      },
    },
  };
  
  // Set overall status
  const hasRed = Object.values(status.budgets).some(category =>
    Object.values(category).some(m => m.status === 'red')
  );
  const hasYellow = Object.values(status.budgets).some(category =>
    Object.values(category).some(m => m.status === 'yellow')
  );
  
  status.overall = hasRed ? 'red' : hasYellow ? 'yellow' : 'green';
  
  res.json(status);
});

async function getCurrentMetrics() {
  // In production: query from Prometheus/Grafana
  // Mock data for example
  return {
    api: { p95: 185, errorRate: 0.3 },
    db: { slowQueriesPerMinute: 2 },
    cache: { hitRate: 85 },
  };
}

app.listen(3001);
```

```yaml
# grafana/performance-budget-dashboard.json
# Grafana dashboard สำหรับ budget monitoring
{
  "title": "Performance Budget - chuaikan.com",
  "panels": [
    {
      "title": "API P95 vs Budget",
      "type": "stat",
      "targets": [{
        "expr": "histogram_quantile(0.95, rate(api_response_time_ms_bucket[5m]))"
      }],
      "thresholds": {
        "mode": "absolute",
        "steps": [
          {"color": "green", "value": null},
          {"color": "yellow", "value": 150},
          {"color": "red", "value": 200}
        ]
      }
    },
    {
      "title": "DB Slow Queries/min",
      "type": "graph",
      "targets": [{
        "expr": "rate(db_slow_queries_total[1m]) * 60"
      }]
    }
  ]
}
```

### Step 688: Bundle Budget Enforcement Script

```bash
#!/bin/bash
# check-performance-budget.sh - CI budget check script

set -e
echo "=== Performance Budget Check ==="

FAILURES=0

# 1. Check bundle sizes
echo ""
echo "1. Bundle Size Check:"
JS_SIZE=$(du -sk .next/static/chunks/*.js 2>/dev/null | awk '{sum+=$1} END{print sum}')
CSS_SIZE=$(du -sk .next/static/css/*.css 2>/dev/null | awk '{sum+=$1} END{print sum}')
JS_BUDGET=350    # KB
CSS_BUDGET=55    # KB

echo "   JS: ${JS_SIZE}KB (budget: ${JS_BUDGET}KB)"
echo "   CSS: ${CSS_SIZE}KB (budget: ${CSS_BUDGET}KB)"

if [ "$JS_SIZE" -gt "$JS_BUDGET" ]; then
  echo "   ❌ JS bundle exceeds budget by $((JS_SIZE - JS_BUDGET))KB!"
  FAILURES=$((FAILURES + 1))
else
  echo "   ✅ JS bundle within budget"
fi

if [ "$CSS_SIZE" -gt "$CSS_BUDGET" ]; then
  echo "   ❌ CSS bundle exceeds budget by $((CSS_SIZE - CSS_BUDGET))KB!"
  FAILURES=$((FAILURES + 1))
else
  echo "   ✅ CSS bundle within budget"
fi

# 2. Check Lighthouse scores (ต้องมี server running)
echo ""
echo "2. Lighthouse Budget Check:"
if command -v lighthouse &> /dev/null; then
  LIGHTHOUSE_RESULT=$(lighthouse http://localhost:3000 \
    --output json \
    --quiet \
    --only-categories performance,accessibility 2>/dev/null)
  
  PERF_SCORE=$(echo $LIGHTHOUSE_RESULT | \
    python3 -c "import sys,json; d=json.load(sys.stdin); print(int(d['categories']['performance']['score']*100))")
  A11Y_SCORE=$(echo $LIGHTHOUSE_RESULT | \
    python3 -c "import sys,json; d=json.load(sys.stdin); print(int(d['categories']['accessibility']['score']*100))")
  
  PERF_BUDGET=85
  A11Y_BUDGET=95
  
  echo "   Performance: ${PERF_SCORE} (budget: ${PERF_BUDGET})"
  echo "   Accessibility: ${A11Y_SCORE} (budget: ${A11Y_BUDGET})"
  
  if [ "$PERF_SCORE" -lt "$PERF_BUDGET" ]; then
    echo "   ❌ Performance score below budget!"
    FAILURES=$((FAILURES + 1))
  else
    echo "   ✅ Performance score within budget"
  fi
fi

# Summary
echo ""
if [ "$FAILURES" -gt 0 ]; then
  echo "❌ $FAILURES budget check(s) failed!"
  exit 1
else
  echo "✅ All performance budgets met!"
  exit 0
fi
```

### Step 689: Automated Budget Alerts

```javascript
// budget-alert.js - Send alerts when budgets are exceeded
const axios = require('axios');

const SLACK_WEBHOOK = process.env.SLACK_WEBHOOK_URL;

async function sendBudgetAlert(violation) {
  const emoji = violation.severity === 'critical' ? '🚨' : '⚠️';
  const color = violation.severity === 'critical' ? '#ff0000' : '#ff9900';
  
  const message = {
    attachments: [{
      color,
      title: `${emoji} Performance Budget Violation: ${violation.metric}`,
      fields: [
        { title: 'Budget', value: `${violation.budget}${violation.unit}`, short: true },
        { title: 'Actual', value: `${violation.actual}${violation.unit}`, short: true },
        { title: 'Over by', value: `${violation.overagePercent}%`, short: true },
        { title: 'Endpoint', value: violation.endpoint || 'N/A', short: true },
        { title: 'Environment', value: process.env.NODE_ENV, short: true },
        { title: 'Time', value: new Date().toISOString(), short: true },
      ],
      footer: 'chuaikan.com Performance Budget Monitor',
    }],
  };
  
  if (SLACK_WEBHOOK) {
    await axios.post(SLACK_WEBHOOK, message);
  }
  
  // Also log to console
  console.error(`Budget violation: ${violation.metric} = ${violation.actual}${violation.unit} (budget: ${violation.budget}${violation.unit})`);
}

// Check budgets every minute
async function checkBudgets() {
  const budgets = [
    {
      metric: 'API P95 Latency',
      query: 'histogram_quantile(0.95, rate(api_response_time_ms_bucket[5m]))',
      budget: 200,
      unit: 'ms',
      severity: 'critical',
    },
    {
      metric: 'Error Rate',
      query: 'rate(http_requests_total{status_code=~"5.."}[5m]) / rate(http_requests_total[5m]) * 100',
      budget: 1,
      unit: '%',
      severity: 'critical',
    },
    {
      metric: 'Cache Hit Rate',
      query: 'redis_hit_rate_percent',
      budget: 80,
      unit: '%',
      severity: 'warning',
      checkFn: (actual, budget) => actual < budget,  // violation if BELOW budget
    },
  ];
  
  for (const budget of budgets) {
    // Query Prometheus
    const result = await queryPrometheus(budget.query);
    const actual = parseFloat(result);
    
    const isViolation = budget.checkFn 
      ? budget.checkFn(actual, budget.budget)
      : actual > budget.budget;
    
    if (isViolation) {
      const overagePercent = budget.checkFn
        ? ((budget.budget - actual) / budget.budget * 100).toFixed(1)
        : ((actual - budget.budget) / budget.budget * 100).toFixed(1);
      
      await sendBudgetAlert({
        ...budget,
        actual: actual.toFixed(2),
        overagePercent,
      });
    }
  }
}

async function queryPrometheus(query) {
  const res = await axios.get(`${process.env.PROMETHEUS_URL}/api/v1/query`, {
    params: { query, time: Date.now() / 1000 },
  });
  
  const results = res.data.data.result;
  if (results.length === 0) return 0;
  return parseFloat(results[0].value[1]);
}

setInterval(checkBudgets, 60000);
checkBudgets();
```

### Step 690: Performance Budget Report

```javascript
// generate-budget-report.js - Weekly budget report
const fs = require('fs');

function generateBudgetReport(metrics) {
  const sections = [
    '# Performance Budget Report',
    `Generated: ${new Date().toISOString()}`,
    '',
    '## Summary',
    '',
  ];
  
  const categories = [
    {
      name: 'Bundle Sizes',
      items: [
        { name: 'JavaScript (gzip)', actual: metrics.js_kb, budget: 300, unit: 'KB' },
        { name: 'CSS (gzip)', actual: metrics.css_kb, budget: 50, unit: 'KB' },
        { name: 'Total page weight', actual: metrics.total_kb, budget: 1000, unit: 'KB' },
      ],
    },
    {
      name: 'Core Web Vitals (P75)',
      items: [
        { name: 'LCP', actual: metrics.lcp_ms, budget: 2500, unit: 'ms' },
        { name: 'INP', actual: metrics.inp_ms, budget: 200, unit: 'ms' },
        { name: 'CLS', actual: metrics.cls, budget: 0.1, unit: '' },
      ],
    },
    {
      name: 'API Performance',
      items: [
        { name: 'P95 Latency', actual: metrics.api_p95, budget: 200, unit: 'ms' },
        { name: 'P99 Latency', actual: metrics.api_p99, budget: 500, unit: 'ms' },
        { name: 'Error Rate', actual: metrics.error_rate, budget: 1, unit: '%' },
      ],
    },
    {
      name: 'Database',
      items: [
        { name: 'Avg Query Time', actual: metrics.db_avg_ms, budget: 50, unit: 'ms' },
        { name: 'Slow Queries/hr', actual: metrics.slow_queries_per_hour, budget: 10, unit: '' },
      ],
    },
  ];
  
  for (const category of categories) {
    sections.push(`### ${category.name}`);
    sections.push('');
    sections.push('| Metric | Budget | Actual | Status |');
    sections.push('|--------|--------|--------|--------|');
    
    for (const item of category.items) {
      const isOk = item.actual <= item.budget;
      const status = isOk ? '✅ Pass' : '❌ FAIL';
      sections.push(`| ${item.name} | ${item.budget}${item.unit} | ${item.actual}${item.unit} | ${status} |`);
    }
    sections.push('');
  }
  
  return sections.join('\n');
}

const metrics = {
  js_kb: 285,
  css_kb: 42,
  total_kb: 850,
  lcp_ms: 2200,
  inp_ms: 175,
  cls: 0.08,
  api_p95: 185,
  api_p99: 420,
  error_rate: 0.2,
  db_avg_ms: 35,
  slow_queries_per_hour: 3,
};

const report = generateBudgetReport(metrics);
fs.writeFileSync('./budget-report.md', report);
console.log(report);
```

---

## 🔧 Configuration Files

```json
// package.json scripts
{
  "scripts": {
    "perf:budget": "npm run build && npm run perf:bundle && npm run perf:lighthouse",
    "perf:bundle": "bundlesize",
    "perf:lighthouse": "lhci autorun",
    "perf:report": "node scripts/generate-budget-report.js"
  }
}
```

---

## 🧪 Testing

```bash
# รัน full budget check pipeline
npm run build

# Check bundle sizes
npx bundlesize

# Start server และรัน Lighthouse
npm start &
sleep 5
npx lhci autorun

# Check API performance (ต้องมี production/staging environment)
autocannon -d 30 -c 50 --json https://staging.chuaikan.com/api/health > /tmp/api-perf.json
node -e "
  const r = require('/tmp/api-perf.json');
  const budget = { p95: 200, errors: 0.01 };
  console.log('P95:', r.latency.p95 + 'ms (budget: ' + budget.p95 + 'ms)');
  if (r.latency.p95 > budget.p95) process.exit(1);
"
```

---

## ❌ Common Errors & Solutions

### Error 1: Bundle size ใหญ่ขึ้นเรื่อยๆ

```bash
# ตรวจสอบ dependency ที่เพิ่มขึ้น
git diff HEAD~1 package-lock.json | grep '"resolved"' | wc -l

# ตรวจสอบว่า package ไหนใหญ่ขึ้น
ANALYZE=true npm run build
# เปิด .next/analyze/client.html และ filter ตาม size
```

### Error 2: Lighthouse score ต่ำแต่หาสาเหตุไม่เจอ

```bash
# ใช้ Lighthouse ด้วย --view flag เพื่อดู full report
lighthouse https://chuaikan.com \
  --view \
  --output html \
  --output-path lighthouse-detailed.html
open lighthouse-detailed.html
# ดู "Opportunities" และ "Diagnostics" sections
```

---

## ✅ Checklist

- [ ] กำหนด bundle size budgets ใน bundlesize.config.json
- [ ] ตั้งค่า Lighthouse CI ด้วย performance thresholds
- [ ] เพิ่ม budget checks ใน GitHub Actions CI
- [ ] ตั้งค่า Prometheus alerts สำหรับ API response time budget
- [ ] Monitor database slow query rate
- [ ] สร้าง weekly budget report
- [ ] ตั้งค่า Slack notifications สำหรับ budget violations
- [ ] Document performance budgets ใน team wiki
- [ ] Review budgets ทุก quarter (ปรับตาม user growth)
- [ ] สร้าง Grafana dashboard สำหรับ budget monitoring

---

## 🔗 References

- [Performance Budgets](https://web.dev/performance-budgets-101/)
- [bundlesize](https://github.com/siddharthkp/bundlesize)
- [size-limit](https://github.com/ai/size-limit)
- [Lighthouse CI](https://github.com/GoogleChrome/lighthouse-ci)
- [Performance Budget Calculator](https://www.performancebudget.io/)

---

*Part 069 | Road to 1,000,000 Users/Day | chuaikan.com*
