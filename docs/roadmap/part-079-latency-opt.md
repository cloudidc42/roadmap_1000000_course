# Part 079: Latency Optimization (< 100ms Globally)
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Professional
> **Steps:** 781–790
> **เวลาโดยประมาณ:** 8 ชั่วโมง
> **Prerequisites:** Part 077 (Global LB), Part 078 (Data Replication)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- Latency budget: CDN edge + network + app + DB
- CDN cache optimization (cache-control headers, Vary header)
- Edge computing ด้วย Cloudflare Workers
- Database read replica ใน each region
- gRPC vs REST latency comparison
- Connection pooling impact
- Prerendering ด้วย Next.js
- P99 latency monitoring by region

---

## 📖 ทฤษฎีและแนวคิด

### Latency Budget สำหรับ chuaikan.com

```
Request จาก Bangkok → Response
────────────────────────────────
CDN Edge (Bangkok PoP)    : ~10ms  (cache hit)
Network (BKK → SG)        : ~30ms  (physical distance ~1,500km)
Application Processing    : ~50ms  (API logic, DB query)
Database Query            : ~10ms  (read replica + index)
────────────────────────────────
Total Target              : < 100ms (P99)
```

### ทำไม P99 ถึงสำคัญ?
- P50 (median) = ประสบการณ์ของ user ส่วนใหญ่
- P99 = 1 ใน 100 request ที่ช้าที่สุด → ยังส่งผลต่อ user ที่ active มากที่สุด
- P99.9 = 1 ใน 1,000 → ที่ 1M req/day = 1,000 requests ต่อวันที่ช้า

---

## 🛠️ Step-by-Step Implementation

### Step 781: CDN Cache Headers

```typescript
// src/middleware/cache-headers.ts
import { Request, Response, NextFunction } from 'express';

interface CacheOptions {
  maxAge?: number;        // seconds for browser cache
  sMaxAge?: number;       // seconds for CDN cache
  staleWhileRevalidate?: number;
  noStore?: boolean;
  private?: boolean;
}

export function setCacheHeaders(options: CacheOptions) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (options.noStore) {
      res.set('Cache-Control', 'no-store, no-cache, must-revalidate');
      res.set('Pragma', 'no-cache');
    } else if (options.private) {
      res.set('Cache-Control', `private, max-age=${options.maxAge || 0}`);
    } else {
      const directives = [
        'public',
        options.maxAge !== undefined ? `max-age=${options.maxAge}` : null,
        options.sMaxAge !== undefined ? `s-maxage=${options.sMaxAge}` : null,
        options.staleWhileRevalidate !== undefined
          ? `stale-while-revalidate=${options.staleWhileRevalidate}`
          : null,
      ].filter(Boolean);

      res.set('Cache-Control', directives.join(', '));
    }

    next();
  };
}

// ใช้กับ routes ต่างๆ
// routes/posts.ts
import express from 'express';
const router = express.Router();

// Post list: cache 60 วินาทีที่ CDN, 10 วินาทีที่ browser
router.get('/posts',
  setCacheHeaders({ sMaxAge: 60, maxAge: 10, staleWhileRevalidate: 120 }),
  async (req, res) => { /* ... */ }
);

// User profile: cache ที่ CDN 5 นาที
router.get('/users/:id',
  setCacheHeaders({ sMaxAge: 300, maxAge: 60 }),
  async (req, res) => { /* ... */ }
);

// Notifications: ห้าม cache (user-specific)
router.get('/notifications',
  setCacheHeaders({ private: true, maxAge: 0 }),
  async (req, res) => { /* ... */ }
);

// Static assets: cache 1 ปี (immutable)
router.get('/assets/*',
  (req, res, next) => {
    res.set('Cache-Control', 'public, max-age=31536000, immutable');
    next();
  }
);
```

```typescript
// src/middleware/vary-headers.ts
// Vary header บอก CDN ว่า response ต่างกันตาม header อะไร

export function setVaryHeaders(headers: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    // Vary: Accept-Encoding บอกว่า gzip/br versions ต่างกัน
    const varyHeaders = ['Accept-Encoding', ...headers];
    res.set('Vary', varyHeaders.join(', '));
    next();
  };
}

// API ที่ return ต่างกันตาม Accept-Language
router.get('/content',
  setVaryHeaders(['Accept-Language', 'X-App-Version']),
  async (req, res) => { /* ... */ }
);
```

### Step 782: Cloudflare Workers สำหรับ Edge Auth

```javascript
// cloudflare-workers/auth-edge.js
// ทำ JWT validation ที่ Edge แทนที่จะส่งไป origin

export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);

    // Skip auth สำหรับ public endpoints
    const publicPaths = ['/health', '/v1/posts', '/v1/users/'];
    const isPublic = publicPaths.some(p => url.pathname.startsWith(p));

    if (!isPublic || request.method !== 'GET') {
      // ตรวจสอบ JWT ที่ edge
      const authResult = await verifyJWT(request, env);

      if (!authResult.valid) {
        return new Response(
          JSON.stringify({ error: 'Unauthorized', code: 'INVALID_TOKEN' }),
          {
            status: 401,
            headers: {
              'Content-Type': 'application/json',
              'X-Edge-Auth': 'failed'
            }
          }
        );
      }

      // เพิ่ม user info ใน header ส่งไป origin
      const modifiedRequest = new Request(request, {
        headers: {
          ...Object.fromEntries(request.headers),
          'X-User-ID': authResult.userId,
          'X-User-Role': authResult.role,
          'X-Edge-Verified': 'true'
        }
      });

      return fetch(modifiedRequest);
    }

    return fetch(request);
  }
};

async function verifyJWT(request, env) {
  const authHeader = request.headers.get('Authorization');
  if (!authHeader?.startsWith('Bearer ')) {
    return { valid: false };
  }

  const token = authHeader.slice(7);

  try {
    // ใช้ Web Crypto API (available ใน Workers)
    const [headerB64, payloadB64, signatureB64] = token.split('.');

    const payload = JSON.parse(atob(payloadB64));

    // ตรวจสอบ expiry
    if (payload.exp < Date.now() / 1000) {
      return { valid: false, reason: 'expired' };
    }

    // ตรวจสอบ signature
    const encoder = new TextEncoder();
    const data = encoder.encode(`${headerB64}.${payloadB64}`);
    const signature = base64UrlDecode(signatureB64);

    const key = await crypto.subtle.importKey(
      'raw',
      encoder.encode(env.JWT_SECRET),
      { name: 'HMAC', hash: 'SHA-256' },
      false,
      ['verify']
    );

    const isValid = await crypto.subtle.verify('HMAC', key, signature, data);

    return {
      valid: isValid,
      userId: payload.sub,
      role: payload.role
    };
  } catch (e) {
    return { valid: false, reason: 'parse_error' };
  }
}

function base64UrlDecode(str) {
  const base64 = str.replace(/-/g, '+').replace(/_/g, '/');
  const padded = base64.padEnd(base64.length + (4 - base64.length % 4) % 4, '=');
  const binary = atob(padded);
  return Uint8Array.from(binary, c => c.charCodeAt(0));
}
```

```javascript
// cloudflare-workers/ab-testing-edge.js
// A/B Testing ที่ Edge (ไม่ต้องรอ origin)

export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);

    // A/B test สำหรับ feed page
    if (url.pathname === '/feed') {
      const userId = getUserId(request);
      const bucket = getBucket(userId);

      // เพิ่ม header บอก variant
      const modifiedRequest = new Request(request, {
        headers: {
          ...Object.fromEntries(request.headers),
          'X-AB-Variant': bucket,
          'X-AB-Experiment': 'feed-algorithm-v2'
        }
      });

      const response = await fetch(modifiedRequest);

      // Track experiment ไป Analytics Engine
      ctx.waitUntil(trackExperiment(env, userId, bucket));

      return response;
    }

    return fetch(request);
  }
};

function getBucket(userId) {
  // Hash-based bucketing: deterministic สำหรับ user เดิม
  let hash = 0;
  for (let i = 0; i < userId.length; i++) {
    hash = ((hash << 5) - hash) + userId.charCodeAt(i);
    hash |= 0;
  }

  const percentage = Math.abs(hash) % 100;

  if (percentage < 10) return 'v2-feed';        // 10% ได้ new algorithm
  return 'v1-feed';                              // 90% ได้ stable
}

async function trackExperiment(env, userId, bucket) {
  await env.ANALYTICS.writeDataPoint({
    indexes: [userId],
    blobs: [bucket, 'feed-algorithm-v2'],
    doubles: [Date.now()]
  });
}
```

### Step 783: gRPC vs REST Benchmark

```go
// benchmarks/grpc_vs_rest_test.go
package benchmarks

import (
	"context"
	"net/http"
	"testing"
	"time"

	pb "github.com/chuaikan/proto/gen"
	"google.golang.org/grpc"
)

func BenchmarkRESTGetPost(b *testing.B) {
	client := &http.Client{
		Transport: &http.Transport{
			MaxIdleConns:       100,
			IdleConnTimeout:    90 * time.Second,
			DisableCompression: false,
		},
	}

	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		resp, err := client.Get("https://api.chuaikan.com/v1/posts/123")
		if err != nil {
			b.Fatal(err)
		}
		resp.Body.Close()
	}
}

func BenchmarkGRPCGetPost(b *testing.B) {
	conn, err := grpc.Dial(
		"api.chuaikan.com:9090",
		grpc.WithTransportCredentials(insecure.NewCredentials()),
		grpc.WithBlock(),
	)
	if err != nil {
		b.Fatal(err)
	}
	defer conn.Close()

	client := pb.NewPostServiceClient(conn)

	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		_, err := client.GetPost(context.Background(), &pb.GetPostRequest{
			PostId: "123",
		})
		if err != nil {
			b.Fatal(err)
		}
	}
}
```

```bash
# รัน benchmark
go test -bench=. -benchtime=10s -benchmem ./benchmarks/

# ผลลัพธ์ตัวอย่าง:
# BenchmarkRESTGetPost-8     5000    250423 ns/op   4096 B/op    42 allocs/op
# BenchmarkGRPCGetPost-8    10000    120891 ns/op   2048 B/op    18 allocs/op
# gRPC เร็วกว่า ~2x เพราะ:
# - Binary protocol (protobuf) แทน JSON
# - HTTP/2 multiplexing
# - Smaller payload
```

### Step 784: Connection Pooling

```typescript
// src/db/pool.ts
import { Pool, PoolConfig } from 'pg';

const poolConfig: PoolConfig = {
  host: process.env.DB_HOST,
  port: 5432,
  database: 'chuaikan',
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,

  // Connection pool settings
  min: 10,          // Minimum connections ที่ keep ไว้
  max: 50,          // Maximum connections
  idleTimeoutMillis: 30000,    // Close connections ที่ idle > 30s
  connectionTimeoutMillis: 5000, // Timeout ถ้า pool เต็ม

  // SSL
  ssl: {
    rejectUnauthorized: true,
    ca: process.env.DB_SSL_CA
  }
};

// Read pool → Read Replica (Tokyo)
const readPool = new Pool({
  ...poolConfig,
  host: process.env.DB_READ_REPLICA_HOST,
  min: 20,   // Read traffic สูงกว่า
  max: 100
});

// Write pool → Primary (Singapore)
const writePool = new Pool(poolConfig);

// Monitor pool health
setInterval(() => {
  const writeStats = {
    total: writePool.totalCount,
    idle: writePool.idleCount,
    waiting: writePool.waitingCount
  };

  const readStats = {
    total: readPool.totalCount,
    idle: readPool.idleCount,
    waiting: readPool.waitingCount
  };

  console.log('Write pool:', writeStats);
  console.log('Read pool:', readStats);

  // Alert ถ้า waiting > 10
  if (writeStats.waiting > 10 || readStats.waiting > 10) {
    console.error('ALERT: Connection pool exhausted!');
  }
}, 10000);

export { readPool, writePool };
```

```bash
# PgBouncer สำหรับ connection pooling ที่ database layer
# /etc/pgbouncer/pgbouncer.ini

[databases]
chuaikan = host=127.0.0.1 port=5432 dbname=chuaikan

[pgbouncer]
listen_port = 6432
listen_addr = *
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction   # transaction mode: เหมาะกับ stateless API
max_client_conn = 5000    # Node.js connections ทั้งหมด
default_pool_size = 50    # PostgreSQL connections จริง
min_pool_size = 10
reserve_pool_size = 10
reserve_pool_timeout = 5
server_idle_timeout = 600
log_connections = 0       # ปิด log ลด overhead
log_disconnections = 0
```

### Step 785: Next.js Prerendering

```typescript
// app/posts/[id]/page.tsx
// Incremental Static Regeneration (ISR)

import { notFound } from 'next/navigation';

interface Props {
  params: { id: string };
}

// ISR: revalidate ทุก 60 วินาที
export const revalidate = 60;

// Static params สำหรับ popular posts (top 1000)
export async function generateStaticParams() {
  const posts = await fetch('https://api.chuaikan.com/v1/posts/popular?limit=1000')
    .then(r => r.json());

  return posts.map((post: { id: string }) => ({
    id: post.id
  }));
}

export default async function PostPage({ params }: Props) {
  const post = await fetch(
    `https://api.chuaikan.com/v1/posts/${params.id}`,
    {
      next: { revalidate: 60 },  // Cache ไว้ 60 วินาที
      headers: { 'X-Server-Component': 'true' }
    }
  );

  if (!post.ok) {
    if (post.status === 404) notFound();
    throw new Error(`Failed to fetch post: ${post.status}`);
  }

  const data = await post.json();

  return (
    <article>
      <h1>{data.title}</h1>
      <p>{data.content}</p>
    </article>
  );
}
```

### Step 786: P99 Latency Monitoring

```yaml
# k8s/monitoring/latency-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: latency-sla
  namespace: monitoring
spec:
  groups:
    - name: latency.rules
      rules:
        # P99 latency > 100ms เป็น alert
        - alert: HighLatencyP99
          expr: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket[5m])) by (le, route, region)
            ) > 0.1
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "P99 latency > 100ms on {{ $labels.route }}"
            description: "P99: {{ $value | humanizeDuration }} on {{ $labels.region }}"

        # P99 > 500ms เป็น critical
        - alert: CriticalLatencyP99
          expr: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket[5m])) by (le, route, region)
            ) > 0.5
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "CRITICAL: P99 latency > 500ms on {{ $labels.route }}"
```

```python
# scripts/latency_report.py
"""สร้าง latency report แยกตาม region"""

import boto3
import json
from datetime import datetime, timedelta

def get_p99_latency_by_region(period_minutes: int = 60):
    """ดึง P99 latency จาก CloudWatch แยกตาม region"""
    cw = boto3.client('cloudwatch', region_name='us-east-1')

    regions = ['ap-southeast-1', 'ap-northeast-1']
    results = {}

    for region in regions:
        response = cw.get_metric_statistics(
            Namespace='chuaikan/API',
            MetricName='ResponseTime',
            Dimensions=[{'Name': 'Region', 'Value': region}],
            StartTime=datetime.utcnow() - timedelta(minutes=period_minutes),
            EndTime=datetime.utcnow(),
            Period=period_minutes * 60,
            Statistics=['p99'],
            ExtendedStatistics=['p99', 'p95', 'p50']
        )

        if response['Datapoints']:
            dp = response['Datapoints'][0]
            results[region] = {
                'p50': dp.get('ExtendedStatistics', {}).get('p50', 0),
                'p95': dp.get('ExtendedStatistics', {}).get('p95', 0),
                'p99': dp.get('ExtendedStatistics', {}).get('p99', 0)
            }

    return results

if __name__ == "__main__":
    report = get_p99_latency_by_region()
    print(json.dumps(report, indent=2))
    # {"ap-southeast-1": {"p50": 45.2, "p95": 78.5, "p99": 98.3}, ...}
```

---

## 🧪 Testing

```bash
# ทดสอบ Cache Headers
curl -sv "https://api.chuaikan.com/v1/posts?limit=10" 2>&1 | grep -i "cache-control\|age\|x-cache"

# ทดสอบ Cloudflare Workers Auth
curl -sv "https://api.chuaikan.com/v1/me" \
  -H "Authorization: Bearer invalid_token" \
  | jq .
# ควรได้ 401 ทันที (ไม่ต้องรอ origin)

# วัด latency จาก multiple locations ด้วย httpstat
npm install -g httpstat
httpstat https://api.chuaikan.com/v1/posts

# Load test และ monitor P99
k6 run --vus=100 --duration=5m scripts/load-test.js
```

---

## ✅ Checklist

- [ ] Cache-Control headers ตั้งถูกต้องทุก endpoint
- [ ] CDN cache hit rate > 80% สำหรับ public content
- [ ] Cloudflare Workers validate JWT ที่ edge (ไม่ส่งไป origin)
- [ ] Read replicas ใน Singapore ทุก AZ
- [ ] Connection pooling: PgBouncer หน้า PostgreSQL
- [ ] gRPC ใช้สำหรับ internal service-to-service communication
- [ ] ISR ตั้งค่าสำหรับ popular content pages
- [ ] P99 latency < 100ms สำหรับ Thai users
- [ ] Prometheus alerts สำหรับ P99 > 100ms
- [ ] Latency dashboard แยก region

---

## 🔗 References

- [Cloudflare Workers](https://developers.cloudflare.com/workers/)
- [Next.js ISR](https://nextjs.org/docs/app/building-your-application/data-fetching/incremental-static-regeneration)
- [PgBouncer Configuration](https://www.pgbouncer.org/config.html)

---
*Part 079 | Road to 1,000,000 Users/Day | chuaikan.com*
