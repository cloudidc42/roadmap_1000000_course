# Part 046: Caching Strategies Advanced
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 451–460
> **เวลาโดยประมาณ:** 4 ชั่วโมง
> **Prerequisites:** Part 045 (Query Optimization), Redis พื้นฐาน

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- Cache levels: L1 (in-process), L2 (Redis), L3 (CDN)
- Cache stampede prevention (probabilistic early expiration)
- Cache aside ด้วย distributed lock (ป้องกัน dog-pile)
- Write-through vs Write-behind เปรียบเทียบ
- Cache warming strategies
- Cache invalidation: TTL, event-based, tag-based
- Hierarchical cache สำหรับ feed data
- Redis pipeline และ multi-exec (atomic operations)
- Monitor cache hit rate

---

## 📖 ทฤษฎีและแนวคิด

### Step 451 — Cache Hierarchy

```
L1 Cache (In-Process Memory):
- ที่: RAM ของ process เอง (Node.js heap)
- ขนาด: 10-100 MB
- Latency: < 0.1ms
- Use case: Hot data ที่อ่านทุก request (config, provinces list)

L2 Cache (Redis):
- ที่: Redis server แยกต่างหาก
- ขนาด: 1-100 GB
- Latency: 0.5-2ms
- Use case: Session, flood status, recent SOS reports

L3 Cache (CDN - Cloudflare):
- ที่: Edge nodes ทั่วโลก
- ขนาด: ไม่จำกัด
- Latency: 1-20ms (local PoP)
- Use case: Static assets, API responses ที่ไม่ personal

Request Flow:
Browser → CDN (L3) → App Server → L1 (memory) → L2 (Redis) → DB
```

---

## ⚙️ Environment Setup

```bash
# ติดตั้ง Redis 7
sudo apt-get install -y redis-server

# แก้ไข redis.conf
sudo nano /etc/redis/redis.conf
# เพิ่ม:
# maxmemory 2gb
# maxmemory-policy allkeys-lru

sudo systemctl restart redis-server

# ทดสอบ
redis-cli ping  # PONG

# ติดตั้ง ioredis สำหรับ Node.js
npm install ioredis
npm install --save-dev @types/ioredis
```

---

## 🛠️ Step-by-Step Implementation

### Step 452 — L1 In-Process Cache

```typescript
// lib/cache/memory-cache.ts
import NodeCache from 'node-cache';

interface CacheConfig {
  stdTTL?: number;  // วินาที
  maxKeys?: number;
}

class MemoryCache {
  private cache: NodeCache;

  constructor(config: CacheConfig = {}) {
    this.cache = new NodeCache({
      stdTTL: config.stdTTL || 300,    // 5 นาที
      maxKeys: config.maxKeys || 1000,
      checkperiod: 60,
      useClones: false,  // เร็วกว่า แต่ mutate ระวัง
    });
  }

  get<T>(key: string): T | undefined {
    return this.cache.get<T>(key);
  }

  set<T>(key: string, value: T, ttl?: number): void {
    this.cache.set(key, value, ttl ?? 300);
  }

  del(key: string | string[]): void {
    this.cache.del(key);
  }

  // Cache-aside pattern
  async getOrSet<T>(
    key: string,
    fetcher: () => Promise<T>,
    ttl?: number
  ): Promise<T> {
    const cached = this.get<T>(key);
    if (cached !== undefined) return cached;

    const value = await fetcher();
    this.set(key, value, ttl);
    return value;
  }

  getStats() {
    return this.cache.getStats();
  }
}

// Singleton สำหรับ thai provinces (ไม่ค่อยเปลี่ยน)
export const provinceCache = new MemoryCache({ stdTTL: 3600, maxKeys: 100 });

// Singleton สำหรับ flood status (เปลี่ยนบ่อย)
export const floodStatusCache = new MemoryCache({ stdTTL: 60, maxKeys: 10000 });

// ใช้งาน
import { provinceCache } from './memory-cache';
import { prismaRead } from '../db/connection';

async function getProvinces() {
  return provinceCache.getOrSet(
    'provinces:all',
    () => prismaRead.thaiProvince.findMany({ orderBy: { nameTh: 'asc' } }),
    3600  // cache 1 ชั่วโมง
  );
}
```

### Step 453 — L2 Redis Cache (ioredis)

```typescript
// lib/cache/redis-cache.ts
import Redis from 'ioredis';

const redis = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT || '6379'),
  password: process.env.REDIS_PASSWORD,
  retryStrategy: (times) => Math.min(times * 50, 2000),
  maxRetriesPerRequest: 3,
  lazyConnect: true,
  keyPrefix: 'ck:',  // chuaikan prefix
});

redis.on('error', (err) => console.error('[Redis]', err));

export class RedisCache {
  private redis: Redis;

  constructor(redis: Redis) {
    this.redis = redis;
  }

  async get<T>(key: string): Promise<T | null> {
    const raw = await this.redis.get(key);
    if (!raw) return null;
    return JSON.parse(raw) as T;
  }

  async set<T>(key: string, value: T, ttlSeconds: number): Promise<void> {
    await this.redis.setex(key, ttlSeconds, JSON.stringify(value));
  }

  async del(...keys: string[]): Promise<void> {
    if (keys.length > 0) {
      await this.redis.del(...keys);
    }
  }

  // Scan + delete by pattern (ไม่ใช้ KEYS ใน production!)
  async delPattern(pattern: string): Promise<number> {
    let cursor = '0';
    let deletedCount = 0;

    do {
      const [nextCursor, keys] = await this.redis.scan(
        cursor,
        'MATCH', pattern,
        'COUNT', 100
      );
      cursor = nextCursor;

      if (keys.length > 0) {
        await this.redis.del(...keys);
        deletedCount += keys.length;
      }
    } while (cursor !== '0');

    return deletedCount;
  }

  async getOrSet<T>(
    key: string,
    fetcher: () => Promise<T>,
    ttlSeconds: number
  ): Promise<T> {
    const cached = await this.get<T>(key);
    if (cached !== null) return cached;

    const value = await fetcher();
    await this.set(key, value, ttlSeconds);
    return value;
  }
}

export const cache = new RedisCache(redis);
export { redis };
```

### Step 454 — Cache Stampede Prevention

```typescript
// lib/cache/stampede-prevention.ts
import { redis, cache } from './redis-cache';

// วิธีที่ 1: Probabilistic Early Expiration (XFetch algorithm)
// ป้องกัน stampede โดย recompute ก่อน expire
async function probabilisticGet<T>(
  key: string,
  fetcher: () => Promise<T>,
  ttlSeconds: number,
  beta: number = 1.0  // ยิ่งสูง = recompute เร็วขึ้น (conservative)
): Promise<T | null> {
  const raw = await redis.get(key);
  if (!raw) return null;

  const { value, delta, expiry } = JSON.parse(raw);
  const currentTime = Date.now() / 1000;

  // XFetch: recompute ถ้า current_time - delta*beta*log(random()) > expiry
  if (currentTime - delta * beta * Math.log(Math.random()) > expiry) {
    // recompute in background โดยไม่ block
    const start = Date.now();
    const newValue = await fetcher();
    const newDelta = (Date.now() - start) / 1000;

    await redis.setex(
      key,
      ttlSeconds,
      JSON.stringify({
        value: newValue,
        delta: newDelta,
        expiry: Date.now() / 1000 + ttlSeconds,
      })
    );
    return newValue;
  }

  return value as T;
}

// วิธีที่ 2: Distributed Lock (Redis SETNX)
async function getOrSetWithLock<T>(
  key: string,
  fetcher: () => Promise<T>,
  ttlSeconds: number
): Promise<T> {
  // ลองอ่าน cache ก่อน
  const cached = await cache.get<T>(key);
  if (cached !== null) return cached;

  const lockKey = `lock:${key}`;
  const lockValue = Math.random().toString(36).slice(2);
  const LOCK_TTL = 10; // 10 วินาที

  // ลองล็อก
  const locked = await redis.set(
    lockKey,
    lockValue,
    'NX',     // Only set if not exists
    'EX',
    LOCK_TTL
  );

  if (!locked) {
    // มีคนอื่นกำลัง fetch อยู่ → รอแล้วอ่าน cache ใหม่
    await new Promise(resolve => setTimeout(resolve, 100));
    const retried = await cache.get<T>(key);
    if (retried !== null) return retried;
    // ถ้ายังไม่มี → fetch เองเลย
  }

  try {
    const value = await fetcher();
    await cache.set(key, value, ttlSeconds);
    return value;
  } finally {
    // ปล่อย lock ด้วย Lua script (atomic)
    const releaseLua = `
      if redis.call("GET", KEYS[1]) == ARGV[1] then
        return redis.call("DEL", KEYS[1])
      else
        return 0
      end
    `;
    await redis.eval(releaseLua, 1, lockKey, lockValue);
  }
}

export { probabilisticGet, getOrSetWithLock };
```

### Step 455 — Write-Through vs Write-Behind

```typescript
// lib/cache/write-strategies.ts

// === Write-Through ===
// เขียน DB + Cache พร้อมกัน (consistent แต่ช้ากว่า)
async function updateSOSReportWriteThrough(
  id: string,
  status: string
) {
  // 1. เขียน DB ก่อน
  const updated = await prisma.sosReport.update({
    where: { id },
    data: { status, updatedAt: new Date() },
  });

  // 2. อัพเดท cache ทันที
  const cacheKey = `sos:${id}`;
  await cache.set(cacheKey, updated, 3600);

  // 3. Invalidate list cache
  await cache.delPattern('sos:list:*');

  return updated;
}

// === Write-Behind (Async Write) ===
// เขียน Cache ก่อน แล้ว queue การเขียน DB
import Bull from 'bull';
const dbWriteQueue = new Bull('db-writes', {
  redis: { host: process.env.REDIS_HOST }
});

async function updateSOSReportWriteBehind(
  id: string,
  status: string
) {
  const update = { id, status, updatedAt: new Date().toISOString() };

  // 1. เขียน cache ก่อน (เร็ว)
  const cacheKey = `sos:${id}`;
  const existing = await cache.get<any>(cacheKey);
  if (existing) {
    await cache.set(cacheKey, { ...existing, ...update }, 3600);
  }

  // 2. Queue การเขียน DB (async)
  await dbWriteQueue.add('update-sos', update, {
    attempts: 3,
    backoff: { type: 'exponential', delay: 1000 },
  });

  return update;
}

// Worker: ประมวลผล queue และเขียน DB
dbWriteQueue.process('update-sos', async (job) => {
  const { id, status, updatedAt } = job.data;
  await prisma.sosReport.update({
    where: { id },
    data: { status, updatedAt: new Date(updatedAt) },
  });
});
```

### Step 456 — Cache Warming

```typescript
// scripts/cache-warm.ts
// รัน cache warming เมื่อ start server หรือหลัง deploy

import { cache } from '../lib/cache/redis-cache';
import { prismaRead } from '../lib/db/connection';

async function warmCriticalData() {
  console.log('[Cache Warm] Starting...');
  const start = Date.now();

  // 1. จังหวัดทั้งหมด (อ่านบ่อยมาก)
  const provinces = await prismaRead.thaiProvince.findMany();
  await cache.set('provinces:all', provinces, 3600);
  console.log(`[Cache Warm] Provinces: ${provinces.length} items`);

  // 2. Flood zones ที่ active
  const floodZones = await prismaRead.floodZone.findMany({
    where: { isActive: true },
  });
  await cache.set('flood_zones:active', floodZones, 300);
  console.log(`[Cache Warm] Flood zones: ${floodZones.length} items`);

  // 3. Top 1000 users (homepage activity)
  const topUsers = await prismaRead.user.findMany({
    where: { isActive: true },
    orderBy: { postCount: 'desc' },
    take: 1000,
    select: { id: true, username: true, avatarUrl: true },
  });

  // Cache แต่ละ user แยก
  const pipeline = (await import('../lib/cache/redis-cache')).redis.pipeline();
  for (const user of topUsers) {
    pipeline.setex(
      `ck:user:${user.id}:profile`,
      600,
      JSON.stringify(user)
    );
  }
  await pipeline.exec();
  console.log(`[Cache Warm] User profiles: ${topUsers.length} items`);

  // 4. Latest flood sensor readings per province
  const latestReadings = await prismaRead.$queryRaw<any[]>`
    SELECT DISTINCT ON (province_id)
      province_id, sensor_id, water_level, time
    FROM sensor_readings sr
    JOIN sensor_stations ss ON ss.station_id = sr.sensor_id
    ORDER BY province_id, time DESC
  `;
  await cache.set('sensors:latest_by_province', latestReadings, 60);

  console.log(`[Cache Warm] Done in ${Date.now() - start}ms`);
}

// รัน warmup
warmCriticalData().catch(console.error);
```

### Step 457 — Tag-based Cache Invalidation

```typescript
// lib/cache/tag-invalidation.ts
// ใช้ Redis Sets เก็บ cache keys ที่ associate กับ tag

const redis = /* redis instance */;

async function setWithTags(
  key: string,
  value: unknown,
  ttlSeconds: number,
  tags: string[]
): Promise<void> {
  const pipeline = redis.pipeline();

  // เก็บ value
  pipeline.setex(key, ttlSeconds, JSON.stringify(value));

  // เพิ่ม key ไปยัง tag sets
  for (const tag of tags) {
    const tagKey = `tag:${tag}`;
    pipeline.sadd(tagKey, key);
    pipeline.expire(tagKey, ttlSeconds + 60); // tag set expire หลัง value
  }

  await pipeline.exec();
}

async function invalidateByTag(tag: string): Promise<void> {
  const tagKey = `tag:${tag}`;
  const keys = await redis.smembers(tagKey);

  if (keys.length > 0) {
    const pipeline = redis.pipeline();
    pipeline.del(...keys);
    pipeline.del(tagKey);
    await pipeline.exec();
    console.log(`[Cache] Invalidated ${keys.length} keys for tag: ${tag}`);
  }
}

// ตัวอย่างการใช้งาน
// เก็บ flood zone data พร้อม tags
await setWithTags(
  'flood_zones:province:10',
  floodZonesData,
  300,
  ['flood_zones', 'province:10', 'flood_zones:province:10']
);

// เมื่อ flood zone เปลี่ยน → invalidate tag
async function onFloodZoneUpdate(provinceId: number) {
  await invalidateByTag(`province:${provinceId}`);
  // ลบ cache ทุกอย่างที่เกี่ยวกับ province นี้
}
```

### Step 458 — Redis Pipeline และ Multi-exec

```typescript
// lib/cache/atomic-operations.ts

// Pipeline: ส่ง commands หลายตัวพร้อมกัน (ลด round trips)
async function updateUserStats(userId: string, postId: string) {
  const pipeline = redis.pipeline();

  // ทำหลาย operations พร้อมกัน
  pipeline.incr(`user:${userId}:post_count`);
  pipeline.zadd(`user:${userId}:posts`, Date.now(), postId);
  pipeline.expire(`user:${userId}:posts`, 86400);
  pipeline.hincrby('global:stats', 'total_posts', 1);

  const results = await pipeline.exec();
  // results = [[null, 15], [null, 1], [null, 1], [null, 50001]]
  // [error, result] pairs

  return results;
}

// Multi/Exec: Atomic transaction (ทุก command หรือไม่มีเลย)
async function atomicTransfer(
  fromUserId: string,
  toUserId: string,
  amount: number
): Promise<boolean> {
  // Watch keys สำหรับ optimistic locking
  await redis.watch(`user:${fromUserId}:balance`, `user:${toUserId}:balance`);

  const fromBalance = parseInt(await redis.get(`user:${fromUserId}:balance`) || '0');

  if (fromBalance < amount) {
    await redis.unwatch();
    return false;
  }

  const multi = redis.multi();
  multi.decrby(`user:${fromUserId}:balance`, amount);
  multi.incrby(`user:${toUserId}:balance`, amount);
  multi.lpush('transfers:log',
    JSON.stringify({ from: fromUserId, to: toUserId, amount, at: Date.now() })
  );

  const results = await multi.exec();
  // ถ้า key ที่ watch เปลี่ยนระหว่างนี้ → exec return null (retry)

  return results !== null;
}

// Lua Script: atomic read-modify-write
const incrementIfBelow = `
  local current = tonumber(redis.call('GET', KEYS[1])) or 0
  local limit = tonumber(ARGV[1])
  if current < limit then
    return redis.call('INCR', KEYS[1])
  else
    return -1
  end
`;

async function rateLimitedIncrement(key: string, limit: number) {
  return redis.eval(incrementIfBelow, 1, key, limit);
}
```

### Step 459 — Hierarchical Cache สำหรับ Feed

```typescript
// lib/cache/feed-cache.ts
import { redis, cache } from './redis-cache';
import { floodStatusCache } from './memory-cache'; // L1

interface FeedItem {
  id: string;
  type: 'sos' | 'sensor' | 'post';
  data: unknown;
  timestamp: number;
}

// Feed caching strategy:
// L1 (memory, 30s) → L2 (Redis, 5min) → DB

async function getUserFeed(
  userId: string,
  page: number = 0
): Promise<FeedItem[]> {
  const cacheKey = `feed:${userId}:${page}`;
  const L1_TTL = 30;    // วินาที
  const L2_TTL = 300;   // วินาที

  // L1: in-memory cache
  const l1Hit = floodStatusCache.get<FeedItem[]>(cacheKey);
  if (l1Hit) return l1Hit;

  // L2: Redis
  const l2Hit = await cache.get<FeedItem[]>(cacheKey);
  if (l2Hit) {
    floodStatusCache.set(cacheKey, l2Hit, L1_TTL);
    return l2Hit;
  }

  // DB: fetch
  const feed = await fetchFeedFromDB(userId, page);

  // Store in both caches
  await cache.set(cacheKey, feed, L2_TTL);
  floodStatusCache.set(cacheKey, feed, L1_TTL);

  return feed;
}

// Redis Sorted Set สำหรับ timeline
async function addToUserTimeline(userId: string, item: FeedItem) {
  const timelineKey = `timeline:${userId}`;
  const TIMELINE_MAX = 200; // เก็บแค่ 200 items ล่าสุด

  const pipeline = redis.pipeline();
  pipeline.zadd(timelineKey, item.timestamp, JSON.stringify(item));
  pipeline.zremrangebyrank(timelineKey, 0, -(TIMELINE_MAX + 1)); // trim
  pipeline.expire(timelineKey, 3600);
  await pipeline.exec();
}

async function getTimeline(userId: string, offset = 0, limit = 20) {
  const timelineKey = `timeline:${userId}`;
  const items = await redis.zrevrange(
    timelineKey,
    offset,
    offset + limit - 1,
    'WITHSCORES'
  );

  // Parse items
  const result: FeedItem[] = [];
  for (let i = 0; i < items.length; i += 2) {
    result.push(JSON.parse(items[i]));
  }
  return result;
}
```

### Step 460 — Cache Hit Rate Monitoring

```typescript
// lib/monitoring/cache-metrics.ts
import { Counter, Gauge, register } from 'prom-client';

const cacheHits = new Counter({
  name: 'cache_hits_total',
  help: 'Total cache hits',
  labelNames: ['level', 'key_pattern'],
});

const cacheMisses = new Counter({
  name: 'cache_misses_total',
  help: 'Total cache misses',
  labelNames: ['level', 'key_pattern'],
});

export function recordCacheHit(level: 'L1' | 'L2' | 'L3', pattern: string) {
  cacheHits.inc({ level, key_pattern: pattern });
}

export function recordCacheMiss(level: 'L1' | 'L2' | 'L3', pattern: string) {
  cacheMisses.inc({ level, key_pattern: pattern });
}

// Redis INFO stats
export async function getRedisCacheStats() {
  const info = await redis.info('stats');
  const lines = info.split('\r\n');

  const stats: Record<string, string> = {};
  for (const line of lines) {
    const [key, value] = line.split(':');
    if (key && value) stats[key.trim()] = value.trim();
  }

  const hits = parseInt(stats['keyspace_hits'] || '0');
  const misses = parseInt(stats['keyspace_misses'] || '0');
  const hitRate = hits / (hits + misses);

  return {
    hits,
    misses,
    hitRate: isNaN(hitRate) ? 0 : hitRate,
    hitRatePct: isNaN(hitRate) ? '0%' : `${(hitRate * 100).toFixed(1)}%`,
    connectedClients: parseInt(stats['connected_clients'] || '0'),
    usedMemoryHuman: stats['used_memory_human'],
    evictedKeys: parseInt(stats['evicted_keys'] || '0'),
  };
}

// API endpoint
// GET /api/admin/cache-stats
router.get('/cache-stats', async (req, res) => {
  const stats = await getRedisCacheStats();
  res.json(stats);
});
```

```bash
# ดู Redis stats ผ่าน CLI
redis-cli INFO stats | grep -E "keyspace_hits|keyspace_misses|connected_clients|used_memory_human|evicted_keys"

# Monitor realtime (ดู commands ที่รันอยู่)
redis-cli MONITOR | grep -E "GET|SET|DEL"

# ดู hit rate
redis-cli INFO keyspace
redis-cli INFO stats | grep keyspace
```

---

## 🔧 Configuration Files

```ini
# /etc/redis/redis.conf
maxmemory 4gb
maxmemory-policy allkeys-lru    # LRU eviction ทั่วไป

# ถ้า data มี TTL ทุกตัว
# maxmemory-policy volatile-lru

# Persistence (RDB snapshot)
save 900 1      # บันทึกถ้ามีการเปลี่ยนแปลง 1 key ใน 15 นาที
save 300 10     # บันทึกถ้ามีการเปลี่ยนแปลง 10 keys ใน 5 นาที
save 60 10000   # บันทึกถ้ามีการเปลี่ยนแปลง 10000 keys ใน 1 นาที

# AOF (append-only log)
appendonly yes
appendfsync everysec    # flush ทุก 1 วินาที

# Lazy freeing (ไม่ block main thread)
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
```

---

## 🧪 Testing

```bash
# ทดสอบ pipeline performance
redis-cli --pipe << 'EOF'
SET test:1 "hello"
SET test:2 "world"
GET test:1
GET test:2
EOF

# Benchmark
redis-benchmark -h localhost -p 6379 \
  -c 50 -n 100000 -q \
  -t get,set,lpush,lrange

# ดู cache hit rate หลัง warm up
redis-cli INFO stats | grep -E "hits|misses"
```

---

## ❌ Common Errors & Solutions

**Cache hit rate ต่ำ (< 80%)**
```bash
# 1. TTL สั้นเกินไป → เพิ่ม TTL
# 2. Key pattern หลากหลายเกินไป → normalize keys
# 3. ข้อมูล invalidate บ่อยเกินไป → ตรวจ invalidation logic
redis-cli DEBUG SLEEP 0  # ทดสอบว่า Redis ตอบสนอง
```

**Memory เต็ม (OOM)**
```bash
redis-cli CONFIG GET maxmemory
redis-cli INFO memory | grep used_memory_human
# แก้: เพิ่ม maxmemory หรือใช้ volatile-lru policy
```

**Dog-pile ยังเกิดอยู่**
```typescript
// ตรวจว่า lock ทำงานถูกต้อง
// เพิ่ม logging ใน getOrSetWithLock
console.log(`Lock acquired: ${locked}, key: ${key}`);
```

---

## ✅ Checklist

- [ ] **Step 451** — อธิบาย L1/L2/L3 cache levels ให้ทีมฟังได้
- [ ] **Step 452** — ติดตั้ง node-cache สำหรับ in-process cache
- [ ] **Step 453** — เขียน RedisCache class ด้วย ioredis
- [ ] **Step 454** — ป้องกัน cache stampede ด้วย distributed lock
- [ ] **Step 455** — เปรียบเทียบ write-through vs write-behind ในโค้ดจริง
- [ ] **Step 456** — รัน cache warming script หลัง deploy
- [ ] **Step 457** — ใช้ tag-based invalidation สำหรับ flood zone cache
- [ ] **Step 458** — ใช้ Redis pipeline ลด round trips
- [ ] **Step 459** — สร้าง hierarchical cache สำหรับ user feed
- [ ] **Step 460** — Monitor cache hit rate ผ่าน Prometheus + Grafana

---

## 🔗 References

- [Redis Documentation](https://redis.io/documentation)
- [Cache Stampede Prevention (XFetch)](https://cseweb.ucsd.edu/~avattani/papers/cache_stampede.pdf)
- [ioredis Documentation](https://github.com/redis/ioredis)
- [Cache Invalidation Strategies](https://martinfowler.com/bliki/TwoHardThings.html)

---
*Part 046 | Road to 1,000,000 Users/Day | chuaikan.com*
