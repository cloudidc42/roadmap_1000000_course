# Part 005: Redis — Cache Layer
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 41-50
> **เวลาโดยประมาณ:** 10 ชั่วโมง
> **Prerequisites:** Part 001 (Linux Basics), Part 003 (Node.js), Part 004 (PostgreSQL)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

1. Redis data structures: String, Hash, List, Set, Sorted Set, Stream
2. Cache-aside pattern ใน Node.js
3. Write-through pattern
4. Session management ด้วย Redis
5. Rate limiting ด้วย sliding window algorithm (Lua script)
6. Pub/Sub สำหรับ real-time notifications
7. Redis Cluster setup (3 masters + 3 replicas)
8. Redis Sentinel สำหรับ High Availability
9. Eviction policies
10. Redis Streams สำหรับ event log

---

## 📖 ทฤษฎีและแนวคิด

### Redis Data Structures Overview

```
Redis Key-Value Store
│
├── String    → ค่าอะไรก็ได้ (text, number, binary, JSON)
│   ├── SET/GET/INCR/DECR/APPEND/STRLEN
│   └── ใช้: counter, cache, session token
│
├── Hash      → dictionary (field → value)
│   ├── HSET/HGET/HMSET/HGETALL/HDEL/HLEN
│   └── ใช้: user object, config, session data
│
├── List      → linked list (ordered by insertion)
│   ├── LPUSH/RPUSH/LPOP/RPOP/LRANGE/LLEN
│   └── ใช้: activity feed, message queue, recent items
│
├── Set       → unique, unordered collection
│   ├── SADD/SMEMBERS/SISMEMBER/SREM/SCARD/SUNION/SINTER
│   └── ใช้: tags, following/follower list, unique visitors
│
├── Sorted Set → unique + score (ordered by score)
│   ├── ZADD/ZRANGE/ZRANGEBYSCORE/ZRANK/ZSCORE/ZREM
│   └── ใช้: leaderboard, trending posts, rate limiting
│
└── Stream    → append-only log
    ├── XADD/XREAD/XREADGROUP/XACK/XLEN
    └── ใช้: event log, audit trail, message queue

Memory Model:
┌──────────────────────────────────────────────┐
│  Redis RAM: 8GB                              │
│  ┌─────────────────────────────────────────┐ │
│  │  Hot data (frequently accessed)         │ │
│  │  ├── Sessions: ~500MB                   │ │
│  │  ├── Cache: ~4GB                        │ │
│  │  ├── Rate limit: ~200MB                 │ │
│  │  └── Pub/Sub: ~100MB                    │ │
│  └─────────────────────────────────────────┘ │
└──────────────────────────────────────────────┘
```

### Cache Patterns

```
Cache-Aside (Lazy Loading):
─────────────────────────────
App → Cache? → Yes → Return data
              ↓ No
              DB → Store in Cache → Return data

Write-Through:
─────────────────────────────
App → Write to Cache → Write to DB → Return

Write-Behind (Write-Back):
─────────────────────────────
App → Write to Cache → Return (fast!)
Cache → Batch write to DB (async)

Read-Through:
─────────────────────────────
App → Cache? → Yes → Return
              ↓ No
              Cache fetches from DB → Return

chuaikan.com Strategy:
┌──────────────────────────────────────────────────────────┐
│  Data Type           │ Pattern      │ TTL               │
├──────────────────────────────────────────────────────────┤
│  User profile        │ Cache-aside  │ 300s              │
│  Social feed         │ Cache-aside  │ 60s               │
│  Post details        │ Cache-aside  │ 120s              │
│  SOS alerts nearby   │ Cache-aside  │ 10s               │
│  Session data        │ Write-through│ 86400s (24h)      │
│  Rate limit counters │ Sorted Set   │ 60s (window)      │
│  Online users        │ Set          │ 30s               │
└──────────────────────────────────────────────────────────┘
```

---

## ⚙️ Environment Setup

```bash
# ─── ติดตั้ง Redis 7 ─────────────────────────────────────
# เพิ่ม Redis repository
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor \
    -o /usr/share/keyrings/redis-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] \
    https://packages.redis.io/deb $(lsb_release -cs) main" | \
    sudo tee /etc/apt/sources.list.d/redis.list

sudo apt update
sudo apt install -y redis

# ตรวจสอบ version
redis-server --version
# Expected: Redis server v=7.x.x

redis-cli ping
# Expected: PONG

# ─── ติดตั้ง Node.js Redis client ────────────────────────
npm install ioredis
npm install --save-dev @types/ioredis

# ─── ติดตั้ง RedisInsight (GUI Monitor) ──────────────────
# Download จาก https://redis.com/redis-enterprise/redis-insight/
wget https://download.redis.io/redis-insight/redisinsight-linux64 \
    -O /usr/local/bin/redisinsight
chmod +x /usr/local/bin/redisinsight
```

---

## 🛠️ Step-by-Step Implementation

### Step 41: Redis Data Structures พร้อม Examples

```bash
# ─── Redis CLI Examples ───────────────────────────────────
redis-cli

# ─── String ───────────────────────────────────────────────
SET user:1:name "สมชาย ใจดี"
GET user:1:name
# "สมชาย ใจดี"

SET page:views 0
INCR page:views      # → 1
INCR page:views      # → 2
INCRBY page:views 10 # → 12

# TTL (time to live)
SET session:abc123 "user_data" EX 3600     # expire in 3600s
TTL session:abc123                          # 3599
SET session:def456 "data" PX 60000         # expire in 60000ms
PERSIST session:abc123                      # ลบ TTL

# ─── Hash ─────────────────────────────────────────────────
HSET user:1 \
    username "somchai" \
    email "somchai@chuaikan.com" \
    display_name "สมชาย ใจดี" \
    follower_count 1250

HGET user:1 username
HMGET user:1 username email
HGETALL user:1
HLEN user:1
HINCRBY user:1 follower_count 1   # เพิ่ม follower_count ทีละ 1
HDEL user:1 email                  # ลบ field

# ─── List ─────────────────────────────────────────────────
RPUSH feed:user:1 "post:001" "post:002" "post:003"
LPUSH feed:user:1 "post:000"    # เพิ่มหัวสุด
LRANGE feed:user:1 0 -1         # ดูทั้งหมด
LRANGE feed:user:1 0 9          # 10 รายการแรก
LLEN feed:user:1

# Keep เฉพาะ 1000 รายการล่าสุด
LTRIM feed:user:1 0 999

# Queue operations
LPUSH queue:emails "email:001"
BRPOP queue:emails 0            # blocking pop (wait indefinitely)

# ─── Set ──────────────────────────────────────────────────
SADD following:user:1 "user:2" "user:3" "user:5"
SADD followers:user:1 "user:2" "user:4"
SMEMBERS following:user:1
SISMEMBER following:user:1 "user:2"   # 1 = yes
SCARD following:user:1                 # count
SREM following:user:1 "user:5"

# Set operations
SUNION following:user:1 following:user:2    # union
SINTER following:user:1 following:user:2   # intersection (mutual follows)
SDIFF following:user:1 following:user:2    # difference

# ─── Sorted Set ───────────────────────────────────────────
# score = timestamp (unix)
ZADD trending:posts $(date +%s) "post:001"
ZADD trending:posts $(date +%s) "post:002"
ZADD trending:posts 1000 "post:003"        # score = engagement score

ZRANGE trending:posts 0 -1 WITHSCORES     # lowest to highest
ZREVRANGE trending:posts 0 9 WITHSCORES   # top 10
ZRANGEBYSCORE trending:posts 1000 +inf    # posts with score >= 1000
ZRANK trending:posts "post:001"           # rank (0-indexed)
ZSCORE trending:posts "post:001"          # ดู score
ZINCRBY trending:posts 100 "post:001"     # เพิ่ม score
ZCARD trending:posts                       # count

# ─── Stream ───────────────────────────────────────────────
XADD events:sos * \
    type "alert_created" \
    alert_id "sos:001" \
    user_id "user:1" \
    severity "high"

XLEN events:sos
XRANGE events:sos - +             # ดูทุก events
XRANGE events:sos - + COUNT 10   # 10 events แรก
XREVRANGE events:sos + - COUNT 5  # 5 events ล่าสุด
```

### Step 42: Redis Configuration ใน Node.js (ioredis)

```typescript
// src/lib/redis.ts — Redis client configuration
import Redis from 'ioredis'

const redisOptions = {
  host: process.env.REDIS_HOST || '127.0.0.1',
  port: parseInt(process.env.REDIS_PORT || '6379'),
  password: process.env.REDIS_PASSWORD,
  db: 0,

  // Connection pooling
  maxRetriesPerRequest: 3,
  enableReadyCheck: true,

  // Reconnection
  retryStrategy(times: number): number | null {
    if (times > 10) return null  // หยุด retry หลัง 10 ครั้ง
    return Math.min(times * 50, 2000)  // exponential backoff
  },

  // Performance
  enableOfflineQueue: true,
  connectTimeout: 10000,
  commandTimeout: 5000,
  lazyConnect: false,
}

// Singleton pattern
let redisClient: Redis | null = null

export function getRedisClient(): Redis {
  if (!redisClient) {
    redisClient = new Redis(redisOptions)

    redisClient.on('connect', () => {
      console.log('[Redis] Connected')
    })

    redisClient.on('error', (err) => {
      console.error('[Redis] Error:', err.message)
    })

    redisClient.on('reconnecting', () => {
      console.warn('[Redis] Reconnecting...')
    })
  }
  return redisClient
}

// Pipeline helper (batch commands)
export async function redisPipeline(
  commands: (pipeline: ReturnType<Redis['pipeline']>) => void
): Promise<unknown[]> {
  const client = getRedisClient()
  const pipeline = client.pipeline()
  commands(pipeline)
  const results = await pipeline.exec()
  return results?.map(([err, result]) => {
    if (err) throw err
    return result
  }) || []
}

export default getRedisClient
```

### Step 43: Cache-Aside Pattern

```typescript
// src/lib/cache.ts — Cache-aside implementation
import { getRedisClient } from './redis'

const redis = getRedisClient()

// ─── Generic cache wrapper ────────────────────────────────
export async function withCache<T>(
  key: string,
  fetcher: () => Promise<T>,
  ttl: number = 300  // seconds
): Promise<T> {
  // 1. ลอง get จาก cache
  const cached = await redis.get(key)
  if (cached !== null) {
    return JSON.parse(cached) as T
  }

  // 2. Cache miss → fetch จาก database
  const data = await fetcher()

  // 3. Store ใน cache
  if (data !== null && data !== undefined) {
    await redis.setex(key, ttl, JSON.stringify(data))
  }

  return data
}

// ─── Invalidate cache ─────────────────────────────────────
export async function invalidateCache(pattern: string): Promise<void> {
  // ระวัง: KEYS command ช้าบน production! ใช้ SCAN แทน
  const stream = redis.scanStream({
    match: pattern,
    count: 100,
  })

  const pipeline = redis.pipeline()
  stream.on('data', (keys: string[]) => {
    keys.forEach((key) => pipeline.del(key))
  })

  await new Promise<void>((resolve, reject) => {
    stream.on('end', async () => {
      await pipeline.exec()
      resolve()
    })
    stream.on('error', reject)
  })
}

// ─── User profile cache ───────────────────────────────────
export const userCache = {
  key: (userId: string) => `user:${userId}:profile`,
  ttl: 300, // 5 minutes

  async get(userId: string): Promise<User | null> {
    return withCache(
      this.key(userId),
      () => db.users.findUnique({ where: { id: userId } }),
      this.ttl
    )
  },

  async invalidate(userId: string): Promise<void> {
    await redis.del(this.key(userId))
  },
}

// ─── Feed cache ───────────────────────────────────────────
export const feedCache = {
  key: (userId: string, page: number) => `feed:${userId}:page:${page}`,
  ttl: 60, // 1 minute

  async get(userId: string, page: number) {
    return withCache(
      this.key(userId, page),
      () => fetchFeedFromDB(userId, page),
      this.ttl
    )
  },

  async invalidate(userId: string): Promise<void> {
    // ลบ cache ทุก pages ของ user
    await invalidateCache(`feed:${userId}:page:*`)
  },
}

// ─── ตัวอย่างใช้งานใน API route ──────────────────────────
// src/app/api/users/[id]/route.ts
import { userCache } from '@/lib/cache'

export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  const user = await userCache.get(params.id)

  if (!user) {
    return Response.json({ error: 'User not found' }, { status: 404 })
  }

  return Response.json(user)
}
```

### Step 44: Write-Through Pattern

```typescript
// src/lib/cache-write-through.ts
import { getRedisClient } from './redis'
import { db } from './db'

const redis = getRedisClient()

// ─── Write-through: เขียน cache และ DB พร้อมกัน ──────────
export async function updateUserProfile(
  userId: string,
  data: Partial<User>
): Promise<User> {
  // 1. Update database
  const updatedUser = await db.users.update({
    where: { id: userId },
    data,
  })

  // 2. Update cache (synchronously)
  const cacheKey = `user:${userId}:profile`
  await redis.setex(cacheKey, 300, JSON.stringify(updatedUser))

  // 3. Invalidate related caches
  await redis.del(`feed:homepage:${userId}`)

  return updatedUser
}

// ─── Hash-based caching (เหมาะกับ partial updates) ────────
export const userHashCache = {
  key: (userId: string) => `user:hash:${userId}`,

  async set(userId: string, user: User): Promise<void> {
    await redis.hset(userHashCache.key(userId), {
      id: user.id,
      username: user.username,
      displayName: user.display_name,
      avatarUrl: user.avatar_url || '',
      followerCount: user.follower_count.toString(),
    })
    await redis.expire(userHashCache.key(userId), 300)
  },

  async incrementFollower(userId: string): Promise<void> {
    const key = userHashCache.key(userId)
    // Atomic increment - không cần lock
    await redis.hincrby(key, 'followerCount', 1)
    // Update DB async
    db.users.update({
      where: { id: userId },
      data: { follower_count: { increment: 1 } },
    }).catch(console.error)
  },

  async get(userId: string) {
    return redis.hgetall(userHashCache.key(userId))
  },
}
```

### Step 45: Session Management

```typescript
// src/lib/session.ts — Redis session management
import { getRedisClient } from './redis'
import crypto from 'crypto'

const redis = getRedisClient()

const SESSION_PREFIX = 'session:'
const SESSION_TTL = 60 * 60 * 24  // 24 hours

interface SessionData {
  userId: string
  email: string
  role: string
  createdAt: string
  lastActiveAt: string
}

export const sessionManager = {
  // สร้าง session token
  generateToken(): string {
    return crypto.randomBytes(32).toString('hex')
  },

  // สร้าง session ใหม่
  async create(userId: string, data: Omit<SessionData, 'createdAt' | 'lastActiveAt'>): Promise<string> {
    const token = this.generateToken()
    const key = `${SESSION_PREFIX}${token}`
    const sessionData: SessionData = {
      ...data,
      userId,
      createdAt: new Date().toISOString(),
      lastActiveAt: new Date().toISOString(),
    }

    // เก็บใน Redis Hash
    await redis.hset(key, sessionData as unknown as Record<string, string>)
    await redis.expire(key, SESSION_TTL)

    // Track user's sessions (สำหรับ logout all devices)
    await redis.sadd(`user:${userId}:sessions`, token)
    await redis.expire(`user:${userId}:sessions`, SESSION_TTL)

    return token
  },

  // ดึง session
  async get(token: string): Promise<SessionData | null> {
    const key = `${SESSION_PREFIX}${token}`
    const data = await redis.hgetall(key)

    if (!data || Object.keys(data).length === 0) return null

    // Extend TTL บน access (sliding expiration)
    await redis.expire(key, SESSION_TTL)
    await redis.hset(key, 'lastActiveAt', new Date().toISOString())

    return data as unknown as SessionData
  },

  // ลบ session (logout)
  async destroy(token: string): Promise<void> {
    const session = await this.get(token)
    if (session) {
      await redis.srem(`user:${session.userId}:sessions`, token)
    }
    await redis.del(`${SESSION_PREFIX}${token}`)
  },

  // Logout ทุก devices
  async destroyAll(userId: string): Promise<void> {
    const sessionsKey = `user:${userId}:sessions`
    const tokens = await redis.smembers(sessionsKey)

    if (tokens.length > 0) {
      const pipeline = redis.pipeline()
      tokens.forEach((token) => {
        pipeline.del(`${SESSION_PREFIX}${token}`)
      })
      pipeline.del(sessionsKey)
      await pipeline.exec()
    }
  },
}
```

### Step 46: Rate Limiting ด้วย Sliding Window

```typescript
// src/lib/rate-limit.ts — Sliding Window Rate Limiter

import { getRedisClient } from './redis'

const redis = getRedisClient()

// ─── Lua script สำหรับ atomic rate limiting ───────────────
// Sliding window algorithm: นับ requests ใน 60 วินาทีที่ผ่านมา
const rateLimitScript = `
local key = KEYS[1]
local window = tonumber(ARGV[1])   -- window size in seconds
local limit = tonumber(ARGV[2])    -- max requests
local now = tonumber(ARGV[3])      -- current timestamp (ms)
local window_start = now - (window * 1000)

-- ลบ entries เก่ากว่า window
redis.call('ZREMRANGEBYSCORE', key, 0, window_start)

-- นับ requests ใน window
local count = redis.call('ZCARD', key)

if count >= limit then
    -- Rate limit exceeded
    local oldest = redis.call('ZRANGE', key, 0, 0, 'WITHSCORES')
    local reset_at = oldest[2] and math.ceil((tonumber(oldest[2]) + window * 1000) / 1000) or 0
    return {0, count, reset_at}
end

-- เพิ่ม request ปัจจุบัน
redis.call('ZADD', key, now, now .. '-' .. math.random(10000))
redis.call('EXPIRE', key, window)

return {1, count + 1, 0}
`

interface RateLimitResult {
  allowed: boolean
  count: number
  resetAt: number
  remaining: number
}

export async function checkRateLimit(
  identifier: string,        // IP หรือ userId
  resource: string,          // 'api', 'login', 'sos'
  limit: number,             // max requests
  windowSeconds: number      // time window
): Promise<RateLimitResult> {
  const key = `ratelimit:${resource}:${identifier}`
  const now = Date.now()

  const result = await redis.eval(
    rateLimitScript,
    1,                        // number of keys
    key,                      // KEYS[1]
    windowSeconds.toString(), // ARGV[1]
    limit.toString(),         // ARGV[2]
    now.toString()            // ARGV[3]
  ) as [number, number, number]

  const [allowed, count, resetAt] = result

  return {
    allowed: allowed === 1,
    count,
    resetAt,
    remaining: Math.max(0, limit - count),
  }
}

// ─── Rate limit configurations ────────────────────────────
export const rateLimits = {
  // API calls: 100 requests per minute
  api: (ip: string) =>
    checkRateLimit(ip, 'api', 100, 60),

  // Login attempts: 5 per 15 minutes
  login: (ip: string) =>
    checkRateLimit(ip, 'login', 5, 900),

  // SOS alerts: 3 per hour (ป้องกัน spam)
  sos: (userId: string) =>
    checkRateLimit(userId, 'sos', 3, 3600),

  // Post creation: 20 per hour
  post: (userId: string) =>
    checkRateLimit(userId, 'post', 20, 3600),
}

// ─── Middleware สำหรับ Next.js ────────────────────────────
// src/middleware.ts
import { NextRequest, NextResponse } from 'next/server'
import { rateLimits } from '@/lib/rate-limit'

export async function middleware(request: NextRequest) {
  const ip = request.headers.get('x-forwarded-for') ||
             request.headers.get('x-real-ip') ||
             '127.0.0.1'

  const result = await rateLimits.api(ip)

  if (!result.allowed) {
    return NextResponse.json(
      {
        error: 'Too Many Requests',
        message: 'Rate limit exceeded. Please try again later.',
        resetAt: result.resetAt,
      },
      {
        status: 429,
        headers: {
          'X-RateLimit-Limit': '100',
          'X-RateLimit-Remaining': result.remaining.toString(),
          'X-RateLimit-Reset': result.resetAt.toString(),
          'Retry-After': Math.max(0, result.resetAt - Math.floor(Date.now() / 1000)).toString(),
        },
      }
    )
  }

  const response = NextResponse.next()
  response.headers.set('X-RateLimit-Limit', '100')
  response.headers.set('X-RateLimit-Remaining', result.remaining.toString())
  return response
}
```

### Step 47: Pub/Sub สำหรับ Real-time Notifications

```typescript
// src/lib/pubsub.ts — Redis Pub/Sub implementation
import Redis from 'ioredis'

// ต้องสร้าง connection แยก (pub/sub connection ไม่สามารถใช้ commands อื่นได้)
const publisher = new Redis({
  host: process.env.REDIS_HOST || '127.0.0.1',
  port: parseInt(process.env.REDIS_PORT || '6379'),
})

const subscriber = new Redis({
  host: process.env.REDIS_HOST || '127.0.0.1',
  port: parseInt(process.env.REDIS_PORT || '6379'),
})

// ─── Channel naming convention ────────────────────────────
export const channels = {
  userNotification: (userId: string) => `notifications:user:${userId}`,
  sosAlert: (regionCode: string) => `sos:region:${regionCode}`,
  systemBroadcast: 'system:broadcast',
}

// ─── Publisher ────────────────────────────────────────────
export interface NotificationEvent {
  type: 'like' | 'comment' | 'follow' | 'sos_nearby' | 'system'
  userId: string
  data: Record<string, unknown>
  timestamp: string
}

export async function publishNotification(
  userId: string,
  event: Omit<NotificationEvent, 'timestamp'>
): Promise<void> {
  const channel = channels.userNotification(userId)
  const message: NotificationEvent = {
    ...event,
    timestamp: new Date().toISOString(),
  }

  await publisher.publish(channel, JSON.stringify(message))
}

export async function publishSosAlert(
  regionCode: string,
  alertData: Record<string, unknown>
): Promise<void> {
  const channel = channels.sosAlert(regionCode)
  await publisher.publish(channel, JSON.stringify({
    type: 'sos_alert',
    ...alertData,
    timestamp: new Date().toISOString(),
  }))
}

// ─── Subscriber (Server-Sent Events หรือ WebSocket) ───────
export function subscribeToUserNotifications(
  userId: string,
  onMessage: (event: NotificationEvent) => void
): () => void {
  const channel = channels.userNotification(userId)

  subscriber.subscribe(channel, (err) => {
    if (err) console.error('[PubSub] Subscribe error:', err)
  })

  subscriber.on('message', (ch, message) => {
    if (ch === channel) {
      try {
        const event = JSON.parse(message) as NotificationEvent
        onMessage(event)
      } catch (err) {
        console.error('[PubSub] Parse error:', err)
      }
    }
  })

  // Unsubscribe function
  return () => {
    subscriber.unsubscribe(channel)
  }
}

// ─── Server-Sent Events endpoint ─────────────────────────
// src/app/api/notifications/stream/route.ts
export async function GET(
  request: Request,
  { params }: { params: { userId: string } }
) {
  const userId = params.userId  // ควร verify from session จริงๆ

  const stream = new ReadableStream({
    start(controller) {
      const unsubscribe = subscribeToUserNotifications(
        userId,
        (event) => {
          const data = `data: ${JSON.stringify(event)}\n\n`
          controller.enqueue(new TextEncoder().encode(data))
        }
      )

      // Cleanup เมื่อ client disconnect
      request.signal.addEventListener('abort', () => {
        unsubscribe()
        controller.close()
      })
    },
  })

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    },
  })
}
```

### Step 48: Redis Streams สำหรับ Event Log

```typescript
// src/lib/event-stream.ts — Redis Streams
import { getRedisClient } from './redis'

const redis = getRedisClient()

export interface AppEvent {
  type: string
  userId?: string
  data: Record<string, string>
}

const STREAM_KEY = 'events:app'
const STREAM_MAX_LENGTH = 100000  // เก็บแค่ 100K events

// ─── Publish event ────────────────────────────────────────
export async function publishEvent(event: AppEvent): Promise<string> {
  const id = await redis.xadd(
    STREAM_KEY,
    'MAXLEN', '~', STREAM_MAX_LENGTH.toString(),  // ~ = approximate trimming
    '*',  // auto-generate ID
    'type', event.type,
    'userId', event.userId || '',
    'data', JSON.stringify(event.data),
    'timestamp', new Date().toISOString()
  )

  return id as string
}

// ─── Consumer Group สำหรับ processing ────────────────────
const CONSUMER_GROUP = 'event-processors'

export async function setupConsumerGroup(): Promise<void> {
  try {
    await redis.xgroup('CREATE', STREAM_KEY, CONSUMER_GROUP, '$', 'MKSTREAM')
    console.log('[Stream] Consumer group created')
  } catch (err: unknown) {
    if (err instanceof Error && err.message.includes('BUSYGROUP')) {
      console.log('[Stream] Consumer group already exists')
    } else {
      throw err
    }
  }
}

// ─── Event processor ─────────────────────────────────────
export async function processEvents(
  consumerName: string,
  handler: (event: AppEvent & { id: string }) => Promise<void>
): Promise<void> {
  while (true) {
    const results = await redis.xreadgroup(
      'GROUP', CONSUMER_GROUP, consumerName,
      'COUNT', '10',
      'BLOCK', '5000',   // block 5 seconds waiting for events
      'STREAMS', STREAM_KEY, '>'
    ) as Array<[string, Array<[string, string[]]>]> | null

    if (!results) continue  // timeout, retry

    for (const [, entries] of results) {
      for (const [id, fields] of entries) {
        const event: AppEvent & { id: string } = {
          id,
          type: '',
          data: {},
        }

        // Parse fields (ioredis returns flat array)
        for (let i = 0; i < fields.length; i += 2) {
          const key = fields[i]
          const value = fields[i + 1]
          if (key === 'type') event.type = value
          else if (key === 'userId') event.userId = value
          else if (key === 'data') event.data = JSON.parse(value)
        }

        try {
          await handler(event)
          // ACK เมื่อ process สำเร็จ
          await redis.xack(STREAM_KEY, CONSUMER_GROUP, id)
        } catch (err) {
          console.error(`[Stream] Failed to process event ${id}:`, err)
          // ไม่ ACK → ถูก redelivered ครั้งต่อไป
        }
      }
    }
  }
}

// ─── ตัวอย่างการใช้ ────────────────────────────────────────
// ใน API route:
await publishEvent({
  type: 'sos.created',
  userId: session.userId,
  data: {
    alertId: newAlert.id,
    severity: newAlert.severity,
    lat: newAlert.latitude.toString(),
    lng: newAlert.longitude.toString(),
  },
})

// ใน background worker:
await setupConsumerGroup()
await processEvents('worker-1', async (event) => {
  if (event.type === 'sos.created') {
    await notifyNearbyUsers(event.data)
  }
})
```

### Step 49: Redis Cluster Setup

```bash
# ─── Redis Cluster (3 masters + 3 replicas) ───────────────
# Ports:
# Masters: 7000, 7001, 7002
# Replicas: 7003, 7004, 7005

# สร้าง directories
sudo mkdir -p /opt/redis-cluster/{7000,7001,7002,7003,7004,7005}

# สร้าง config ให้แต่ละ node
for PORT in 7000 7001 7002 7003 7004 7005; do
sudo tee /opt/redis-cluster/${PORT}/redis.conf << EOF
port ${PORT}
cluster-enabled yes
cluster-config-file /opt/redis-cluster/${PORT}/nodes.conf
cluster-node-timeout 5000
appendonly yes
appendfsync everysec
dir /opt/redis-cluster/${PORT}
logfile /var/log/redis/redis-cluster-${PORT}.log
save 900 1
save 300 10
save 60 10000
maxmemory 2gb
maxmemory-policy allkeys-lru
requirepass your_redis_password
masterauth your_redis_password
EOF
done

# สร้าง systemd services
for PORT in 7000 7001 7002 7003 7004 7005; do
sudo tee /etc/systemd/system/redis-cluster-${PORT}.service << EOF
[Unit]
Description=Redis Cluster Node ${PORT}
After=network.target

[Service]
User=redis
Group=redis
ExecStart=/usr/bin/redis-server /opt/redis-cluster/${PORT}/redis.conf
ExecStop=/usr/bin/redis-cli -p ${PORT} -a your_redis_password shutdown
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
done

# Enable และ start ทุก nodes
for PORT in 7000 7001 7002 7003 7004 7005; do
    sudo systemctl enable redis-cluster-${PORT}
    sudo systemctl start redis-cluster-${PORT}
done

# ─── สร้าง cluster ────────────────────────────────────────
redis-cli --cluster create \
    127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 \
    127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
    --cluster-replicas 1 \
    -a your_redis_password

# ตอบ yes เมื่อถาม

# ─── ทดสอบ cluster ────────────────────────────────────────
redis-cli -c -p 7000 -a your_redis_password cluster info
redis-cli -c -p 7000 -a your_redis_password cluster nodes

# ─── ioredis Cluster client ───────────────────────────────
# src/lib/redis-cluster.ts
```

```typescript
// src/lib/redis-cluster.ts
import Redis from 'ioredis'

export const redisCluster = new Redis.Cluster(
  [
    { port: 7000, host: '127.0.0.1' },
    { port: 7001, host: '127.0.0.1' },
    { port: 7002, host: '127.0.0.1' },
  ],
  {
    redisOptions: {
      password: process.env.REDIS_PASSWORD,
      connectTimeout: 10000,
    },
    clusterRetryStrategy(times) {
      if (times > 5) return null
      return Math.min(times * 100, 3000)
    },
    enableReadyCheck: true,
    scaleReads: 'slave',  // อ่านจาก replicas
  }
)
```

### Step 50: Redis Configuration (redis.conf)

```bash
# ─── Production redis.conf ────────────────────────────────
sudo tee /etc/redis/redis.conf << 'EOF'
# ─── Network ──────────────────────────────────────────────
bind 127.0.0.1
port 6379
protected-mode yes

# Unix socket (faster than TCP for local connections)
unixsocket /var/run/redis/redis.sock
unixsocketperm 770

# ─── General ──────────────────────────────────────────────
daemonize no
supervised systemd
pidfile /var/run/redis/redis.pid
loglevel notice
logfile /var/log/redis/redis.log
databases 16

# ─── Security ─────────────────────────────────────────────
requirepass your_strong_redis_password_here
rename-command FLUSHALL ""         # ปิด dangerous commands
rename-command FLUSHDB ""
rename-command DEBUG ""
rename-command CONFIG "REDIS_CONFIG_CMD"

# ─── Memory ───────────────────────────────────────────────
maxmemory 4gb

# Eviction Policy
# allkeys-lru: ลบ key ที่ไม่ได้ใช้นานที่สุด (แนะนำสำหรับ cache)
maxmemory-policy allkeys-lru

# ─── Eviction Policy Options ─────────────────────────────
# noeviction      → error เมื่อ memory เต็ม (สำหรับ session)
# allkeys-lru     → LRU จากทุก keys (สำหรับ cache)
# volatile-lru    → LRU เฉพาะ keys ที่มี expire
# allkeys-lfu     → LFU (Least Frequently Used)
# volatile-lfu    → LFU เฉพาะ keys ที่มี expire
# allkeys-random  → random จากทุก keys
# volatile-random → random จาก keys ที่มี expire
# volatile-ttl    → ลบ keys ที่ TTL น้อยที่สุดก่อน

# ─── Persistence ──────────────────────────────────────────
# RDB Snapshots
save 900 1      # save ทุก 900s ถ้ามี 1+ key เปลี่ยน
save 300 10     # save ทุก 300s ถ้ามี 10+ keys เปลี่ยน
save 60 10000   # save ทุก 60s ถ้ามี 10000+ keys เปลี่ยน
rdbcompression yes
rdbfilename dump.rdb
dir /var/lib/redis

# AOF (Append Only File) - durability ดีกว่า RDB
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec   # sync ทุกวินาที (balance between perf/safety)
                       # always = safe แต่ช้า
                       # no = เร็ว แต่อาจ lose data
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# ─── Performance ──────────────────────────────────────────
# Lazy freeing (ลบ objects แบบ async ใน background thread)
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
replica-lazy-flush no

# Hash optimization (เก็บ compact เมื่อ hash เล็ก)
hash-max-listpack-entries 128
hash-max-listpack-value 64

# List optimization
list-max-listpack-size -2          # 8kb per listpack node
list-compress-depth 0

# Set optimization
set-max-intset-entries 512

# Sorted Set optimization
zset-max-listpack-entries 128
zset-max-listpack-value 64

# ─── Connections ──────────────────────────────────────────
maxclients 10000
tcp-backlog 511
tcp-keepalive 300
timeout 0

# ─── Slow Log ─────────────────────────────────────────────
slowlog-log-slower-than 10000    # log commands ที่ใช้ > 10ms
slowlog-max-len 128

# ─── Latency Monitor ──────────────────────────────────────
latency-monitor-threshold 100    # track events ที่ > 100ms
EOF

sudo systemctl restart redis
sudo systemctl status redis
```

### Redis Sentinel (High Availability)

```bash
# ─── Redis Sentinel Setup ─────────────────────────────────
# Sentinel ดูแล Redis master-replica pair
# ถ้า master ล้ม → promote replica เป็น master อัตโนมัติ

# ─── sentinel.conf ────────────────────────────────────────
sudo tee /etc/redis/sentinel.conf << 'EOF'
port 26379
daemonize no
logfile /var/log/redis/sentinel.log

# Monitor master ชื่อ "mymaster" ที่ 127.0.0.1:6379
# 2 = จำนวน sentinels ที่ต้องตกลงกันก่อน failover
sentinel monitor mymaster 127.0.0.1 6379 2

sentinel auth-pass mymaster your_strong_redis_password_here

# Failover settings
sentinel down-after-milliseconds mymaster 5000   # ถือว่า down หลัง 5s
sentinel parallel-syncs mymaster 1               # sync ทีละ 1 replica
sentinel failover-timeout mymaster 60000         # timeout 60s
EOF

sudo systemctl enable redis-sentinel
sudo systemctl start redis-sentinel
```

---

## 🔧 Configuration Files

### RedisInsight Setup

```bash
# ─── ติดตั้ง RedisInsight ─────────────────────────────────
# Download และติดตั้ง
wget "https://download.redis.io/redis-insight/latest" \
    -O /tmp/redisinsight.deb
sudo dpkg -i /tmp/redisinsight.deb

# หรือ run แบบ Docker
docker run -d \
    --name redisinsight \
    -p 8001:5540 \
    -v redisinsight:/data \
    redis/redisinsight:latest

# เปิด browser: http://localhost:8001
# Add Redis database:
# Host: 127.0.0.1
# Port: 6379
# Password: your_redis_password
```

---

## 🧪 Testing

```bash
# ─── ทดสอบ Redis Basics ───────────────────────────────────
redis-cli -a your_redis_password << 'EOF'
PING
SET test:key "hello"
GET test:key
INCR test:counter
EXPIRE test:key 10
TTL test:key
DEL test:key test:counter
EOF

# ─── ทดสอบ Rate Limiting ──────────────────────────────────
# ส่ง requests เร็วๆ เกิน limit
for i in $(seq 1 10); do
    curl -s http://localhost:3000/api/test \
        -H "X-Forwarded-For: 1.2.3.4" | \
        python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('error','OK'))"
done
# Expected: OK x5, then "Too Many Requests"

# ─── ทดสอบ Pub/Sub ────────────────────────────────────────
# Terminal 1 (subscriber)
redis-cli -a your_password SUBSCRIBE notifications:user:user-uuid-123

# Terminal 2 (publisher)
redis-cli -a your_password PUBLISH notifications:user:user-uuid-123 \
    '{"type":"like","userId":"user-uuid-123","data":{"postId":"post-001"}}'

# ─── ทดสอบ Performance ────────────────────────────────────
redis-benchmark \
    -h 127.0.0.1 \
    -p 6379 \
    -a your_password \
    -n 100000 \
    -q
# Expected:
# SET: 100000+ req/s
# GET: 100000+ req/s

# ─── ดู Slow Log ─────────────────────────────────────────
redis-cli -a your_password SLOWLOG GET 10

# ─── Memory Info ──────────────────────────────────────────
redis-cli -a your_password INFO memory
# ดูที่:
# used_memory_human: actual memory ที่ใช้
# mem_fragmentation_ratio: ควร > 1.0 แต่ < 2.0
```

---

## ❌ Common Errors & Solutions

### Error 1: Connection refused

```
❌ Error: Error: connect ECONNREFUSED 127.0.0.1:6379

✅ Solution:
sudo systemctl status redis
sudo systemctl start redis

# ตรวจสอบ redis config
redis-cli ping
# ถ้า PONG = OK

# ตรวจสอบ bind address ใน redis.conf
grep "^bind" /etc/redis/redis.conf
# ควรเห็น: bind 127.0.0.1
```

### Error 2: OOM (Out of Memory)

```
❌ Error: OOM command not allowed when used memory > 'maxmemory'

✅ Solution:
# 1. เพิ่ม maxmemory ใน redis.conf
maxmemory 8gb

# 2. หรือตั้ง eviction policy
maxmemory-policy allkeys-lru

# 3. ดูว่า key อะไรใช้ memory มาก
redis-cli -a your_pass --memkeys  # top 10 keys by size
redis-cli -a your_pass memory usage key:name
```

### Error 3: Pipeline ไม่ได้ performance ที่คาดหวัง

```
❌ Problem: ทำ 1000 operations ด้วย loop แต่ช้ามาก

✅ Solution:
// แทนที่ loop:
for (const id of ids) {
    await redis.set(`key:${id}`, value)  // ❌ 1000 round trips
}

// ใช้ Pipeline:
const pipeline = redis.pipeline()
for (const id of ids) {
    pipeline.set(`key:${id}`, value)    // queue commands
}
await pipeline.exec()                   // ✅ 1 round trip
```

---

## 📊 Performance Benchmark

| Operation | Without Redis | With Redis | Improvement |
|-----------|--------------|------------|-------------|
| User profile API | 85ms | 2ms | 42x faster |
| Social feed query | 320ms | 8ms | 40x faster |
| Rate limit check | 45ms (DB) | 0.5ms | 90x faster |
| Session validation | 35ms (DB) | 0.3ms | 116x faster |
| SOS nearby (10km) | 180ms | 12ms | 15x faster |
| Concurrent users | 500 | 10,000+ | 20x more |

---

## ✅ Checklist

- [ ] ติดตั้ง Redis 7
- [ ] ตั้งค่า redis.conf (memory, persistence, security)
- [ ] ตั้ง maxmemory และ eviction policy (allkeys-lru)
- [ ] Implement cache-aside pattern
- [ ] Implement write-through สำหรับ critical data
- [ ] ตั้งค่า session management ด้วย Redis
- [ ] Implement rate limiting ด้วย Lua script
- [ ] ตั้งค่า Pub/Sub สำหรับ notifications
- [ ] Implement Redis Streams สำหรับ event log
- [ ] ทดสอบ cache hit rate > 95%
- [ ] ทดสอบ rate limiting ทำงานถูกต้อง
- [ ] ทดสอบ Pub/Sub latency < 10ms
- [ ] ติดตั้ง RedisInsight สำหรับ monitoring
- [ ] ตั้งค่า Redis Sentinel (สำหรับ HA)
- [ ] ทดสอบ redis-benchmark

---

## 🔗 References

- [Redis Documentation](https://redis.io/docs/)
- [Redis Data Types](https://redis.io/docs/data-types/)
- [ioredis Documentation](https://github.com/redis/ioredis)
- [Redis Best Practices](https://redis.io/docs/manual/patterns/)
- [Redis Cluster Setup](https://redis.io/docs/management/scaling/)
- [Redis Sentinel](https://redis.io/docs/management/sentinel/)
- [RedisInsight](https://redis.com/redis-enterprise/redis-insight/)
- [Lua Scripting in Redis](https://redis.io/docs/manual/programmability/eval-intro/)

---
*Part 005 | Road to 1,000,000 Users/Day | chuaikan.com*
