# Part 026: Feed Service (Algorithm)
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 251-260
> **เวลาโดยประมาณ:** 6 ชั่วโมง
> **Prerequisites:** Part 021 (Post Service), Part 022 (Social Graph), Part 023 (Notification Service), Part 025 (Cache Layer)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- เข้าใจ Feed ประเภทต่างๆ: Home Feed, User Feed, Explore Feed, SOS Feed
- เปรียบเทียบ Pull vs Push vs Hybrid Feed Models
- ออกแบบ Hybrid Fan-out strategy ที่เหมาะกับ chuaikan.com
- สร้าง Feed Ranking Algorithm ด้วย Engagement Score
- ใช้ Redis Sorted Sets สำหรับ Feed Storage และ Pagination
- ทำ Cursor-based Pagination สำหรับ Infinite Scroll
- จัดการ Feed Freshness ด้วย TTL และ Cache Invalidation
- ทำ "5 new posts" notification
- Inject SOS Alert ให้อยู่ top ของ Feed เสมอ
- สร้าง A/B Test Framework สำหรับ Algorithm

---

## 📖 ทฤษฎีและแนวคิด

### Feed Types ของ chuaikan.com

```
┌─────────────────────────────────────────────────────────┐
│                    FEED TYPES                           │
├─────────────────────┬───────────────────────────────────┤
│  Home Feed          │ โพสต์จากคนที่ Follow + SOS Alerts │
│  User Feed          │ โพสต์ทั้งหมดของ User คนนั้น       │
│  Explore Feed       │ โพสต์แนะนำจาก Algorithm           │
│  SOS Feed           │ เฉพาะ Emergency Alerts รอบๆ ตัว   │
└─────────────────────┴───────────────────────────────────┘
```

### Pull vs Push vs Hybrid Model

```
PULL MODEL (Fan-out on Read)
─────────────────────────────
User A follows 500 people
↓
When A opens app:
  SELECT posts FROM posts
  WHERE author_id IN (500 user_ids)
  ORDER BY created_at DESC
  LIMIT 20

ข้อดี: Simple, Storage น้อย
ข้อเสีย: Query หนัก, Latency สูงเมื่อ Follow เยอะ


PUSH MODEL (Fan-out on Write)
──────────────────────────────
User B โพสต์รูปน้ำท่วม
↓ ทันที
Write to feed of ALL followers of B
feed:user:1001, feed:user:1002, ... feed:user:50000

ข้อดี: Read เร็วมาก O(1)
ข้อเสีย: Write amplification สูง (celebrity problem)
         Storage มาก, Staleness


HYBRID (chuaikan.com approach)
────────────────────────────────
Users ที่ follow < 1,000 คน → PUSH model
Users ที่ follow > 1,000 คน (Celebrities) → PULL model

When reading feed:
  1. Load pre-computed feed from Redis (PUSH users)
  2. Merge with recent posts from celebrities (PULL)
  3. Apply ranking algorithm
  4. Inject SOS alerts at top
```

### Engagement Scoring Formula

```
Score = (Likes × 1) + (Comments × 3) + (Shares × 5) + (Saves × 10)
      + (Views × 0.1) + (Click_through × 2)

Decay Factor = Score / (age_in_hours + 2)^gravity
where gravity = 1.8 (ค่า default ของ Hacker News)

Final Score = Decay Factor × Relationship_Strength × Recency_Boost

Relationship_Strength:
  - Close Friend: 3.0
  - Regular Follow: 1.0
  - Weak Tie: 0.5

Recency Boost:
  - < 1 hour: 2.0
  - 1-6 hours: 1.5
  - 6-24 hours: 1.0
  - > 24 hours: 0.8
```

### Redis Sorted Set Architecture

```
┌──────────────────────────────────────────────────────────┐
│                REDIS FEED STORAGE                        │
│                                                          │
│  Key: feed:home:{user_id}                                │
│  Type: Sorted Set                                        │
│  Member: post_id                                         │
│  Score: ranking_score (float)                            │
│                                                          │
│  ZRANGEBYSCORE feed:home:1001 -inf +inf                  │
│    LIMIT 0 20 WITHSCORES                                 │
│                                                          │
│  feed:home:1001 → [(post:9999, 98.5), (post:8888, 95.2)] │
│                                                          │
│  TTL: 24 hours (refresh on access)                       │
└──────────────────────────────────────────────────────────┘
```

---

## ⚙️ Environment Setup

### Step 251: Install Dependencies

```bash
cd /home/user/chuaikan/services/feed-service

npm init -y

npm install \
  express@4.18.2 \
  ioredis@5.3.2 \
  pg@8.11.3 \
  @types/pg@8.10.9 \
  axios@1.6.2 \
  zod@3.22.4 \
  winston@3.11.0 \
  bull@4.12.0 \
  socket.io@4.7.2 \
  geohash@1.1.0 \
  uuid@9.0.0

npm install -D \
  typescript@5.3.2 \
  @types/node@20.10.0 \
  @types/express@4.17.21 \
  @types/uuid@9.0.7 \
  ts-node@10.9.2 \
  nodemon@3.0.2

npx tsc --init --target ES2022 --module commonjs \
  --outDir dist --rootDir src --strict true \
  --esModuleInterop true --resolveJsonModule true
```

### Step 252: Database Schema

```sql
-- /home/user/chuaikan/db/migrations/020_feed_tables.sql

-- Feed events table (ติดตาม engagement)
CREATE TABLE feed_events (
  id          BIGSERIAL PRIMARY KEY,
  user_id     UUID NOT NULL,
  post_id     UUID NOT NULL,
  event_type  VARCHAR(20) NOT NULL, -- 'view','like','comment','share','save'
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_feed_events_post ON feed_events(post_id, event_type);
CREATE INDEX idx_feed_events_user ON feed_events(user_id, created_at DESC);

-- Post engagement scores (materialized)
CREATE TABLE post_engagement_scores (
  post_id         UUID PRIMARY KEY,
  likes_count     INT DEFAULT 0,
  comments_count  INT DEFAULT 0,
  shares_count    INT DEFAULT 0,
  saves_count     INT DEFAULT 0,
  views_count     INT DEFAULT 0,
  raw_score       FLOAT DEFAULT 0,
  decay_score     FLOAT DEFAULT 0,
  updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- A/B test assignments
CREATE TABLE ab_test_assignments (
  user_id     UUID NOT NULL,
  test_name   VARCHAR(100) NOT NULL,
  variant     VARCHAR(50) NOT NULL,
  assigned_at TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (user_id, test_name)
);

-- Feed cache metadata
CREATE TABLE feed_cache_metadata (
  user_id         UUID PRIMARY KEY,
  last_generated  TIMESTAMPTZ,
  last_accessed   TIMESTAMPTZ,
  follower_count  INT DEFAULT 0,
  feed_strategy   VARCHAR(20) DEFAULT 'hybrid' -- 'push','pull','hybrid'
);
```

```bash
# รัน migration
psql -U chuaikan_user -d chuaikan_db -f /home/user/chuaikan/db/migrations/020_feed_tables.sql
```

---

## 🛠️ Step-by-Step Implementation

### Step 253: Feed Service Core Types

```typescript
// src/types/feed.types.ts

export interface FeedItem {
  postId: string;
  authorId: string;
  authorName: string;
  authorAvatar?: string;
  content: string;
  mediaUrls: string[];
  tags: string[];
  location?: {
    lat: number;
    lng: number;
    province?: string;
  };
  engagementScore: number;
  rankScore: number;
  createdAt: Date;
  isSOS: boolean;
  sosType?: 'flood' | 'fire' | 'accident' | 'other';
  sosSeverity?: 'INFO' | 'WARNING' | 'DANGER' | 'CRITICAL';
}

export interface FeedOptions {
  userId: string;
  cursor?: string;       // base64 encoded cursor
  limit: number;
  feedType: 'home' | 'user' | 'explore' | 'sos';
  targetUserId?: string; // สำหรับ user feed
  lat?: number;          // สำหรับ SOS feed
  lng?: number;
}

export interface FeedResponse {
  items: FeedItem[];
  nextCursor?: string;
  hasMore: boolean;
  newPostsCount?: number;   // "5 new posts"
  totalCount?: number;
}

export interface EngagementScore {
  postId: string;
  likes: number;
  comments: number;
  shares: number;
  saves: number;
  views: number;
  rawScore: number;
  decayScore: number;
}

export interface ABTestConfig {
  testName: string;
  variants: Array<{
    name: string;
    weight: number;       // 0-1, รวมกันต้อง = 1
    config: Record<string, any>;
  }>;
  startDate: Date;
  endDate: Date;
}
```

### Step 254: Engagement Score Calculator

```typescript
// src/services/engagement-scorer.ts

import { Pool } from 'pg';
import Redis from 'ioredis';

interface ScoreWeights {
  like: number;
  comment: number;
  share: number;
  save: number;
  view: number;
  clickThrough: number;
}

const DEFAULT_WEIGHTS: ScoreWeights = {
  like: 1,
  comment: 3,
  share: 5,
  save: 10,
  view: 0.1,
  clickThrough: 2,
};

export class EngagementScorer {
  private readonly GRAVITY = 1.8;  // Hacker News decay gravity

  constructor(
    private db: Pool,
    private redis: Redis,
    private weights: ScoreWeights = DEFAULT_WEIGHTS
  ) {}

  // คำนวณ Raw Score จาก engagement counts
  calculateRawScore(
    likes: number,
    comments: number,
    shares: number,
    saves: number,
    views: number,
    clickThroughs: number = 0
  ): number {
    return (
      likes * this.weights.like +
      comments * this.weights.comment +
      shares * this.weights.share +
      saves * this.weights.save +
      views * this.weights.view +
      clickThroughs * this.weights.clickThrough
    );
  }

  // คำนวณ Time Decay Score (Hacker News formula)
  calculateDecayScore(rawScore: number, createdAt: Date): number {
    const ageInHours = (Date.now() - createdAt.getTime()) / (1000 * 60 * 60);
    return rawScore / Math.pow(ageInHours + 2, this.GRAVITY);
  }

  // Recency Boost Multiplier
  getRecencyBoost(createdAt: Date): number {
    const ageInHours = (Date.now() - createdAt.getTime()) / (1000 * 60 * 60);
    if (ageInHours < 1) return 2.0;
    if (ageInHours < 6) return 1.5;
    if (ageInHours < 24) return 1.0;
    return 0.8;
  }

  // Relationship Strength Multiplier
  async getRelationshipStrength(viewerId: string, authorId: string): Promise<number> {
    const cacheKey = `rel:strength:${viewerId}:${authorId}`;
    const cached = await this.redis.get(cacheKey);
    if (cached) return parseFloat(cached);

    const result = await this.db.query(`
      SELECT
        CASE
          WHEN close_friend = true THEN 3.0
          WHEN interaction_count > 10 THEN 1.5
          ELSE 1.0
        END as strength
      FROM social_relationships
      WHERE follower_id = $1 AND following_id = $2
    `, [viewerId, authorId]);

    const strength = result.rows[0]?.strength ?? 0.5;
    await this.redis.setex(cacheKey, 3600, strength.toString());
    return strength;
  }

  // คำนวณ Final Rank Score
  async calculateFinalScore(
    post: {
      postId: string;
      authorId: string;
      likes: number;
      comments: number;
      shares: number;
      saves: number;
      views: number;
      createdAt: Date;
    },
    viewerId: string
  ): Promise<number> {
    const rawScore = this.calculateRawScore(
      post.likes,
      post.comments,
      post.shares,
      post.saves,
      post.views
    );

    const decayScore = this.calculateDecayScore(rawScore, post.createdAt);
    const recencyBoost = this.getRecencyBoost(post.createdAt);
    const relationshipStrength = await this.getRelationshipStrength(
      viewerId,
      post.authorId
    );

    return decayScore * recencyBoost * relationshipStrength;
  }

  // Batch update scores ใน DB
  async updateBatchScores(postIds: string[]): Promise<void> {
    if (postIds.length === 0) return;

    await this.db.query(`
      INSERT INTO post_engagement_scores
        (post_id, likes_count, comments_count, shares_count,
         saves_count, views_count, raw_score, decay_score, updated_at)
      SELECT
        p.id,
        COUNT(CASE WHEN fe.event_type = 'like' THEN 1 END)    AS likes_count,
        COUNT(CASE WHEN fe.event_type = 'comment' THEN 1 END) AS comments_count,
        COUNT(CASE WHEN fe.event_type = 'share' THEN 1 END)   AS shares_count,
        COUNT(CASE WHEN fe.event_type = 'save' THEN 1 END)    AS saves_count,
        COUNT(CASE WHEN fe.event_type = 'view' THEN 1 END)    AS views_count,
        (
          COUNT(CASE WHEN fe.event_type = 'like' THEN 1 END) * ${this.weights.like} +
          COUNT(CASE WHEN fe.event_type = 'comment' THEN 1 END) * ${this.weights.comment} +
          COUNT(CASE WHEN fe.event_type = 'share' THEN 1 END) * ${this.weights.share} +
          COUNT(CASE WHEN fe.event_type = 'save' THEN 1 END) * ${this.weights.save} +
          COUNT(CASE WHEN fe.event_type = 'view' THEN 1 END) * ${this.weights.view}
        ) AS raw_score,
        0 AS decay_score,
        NOW()
      FROM posts p
      LEFT JOIN feed_events fe ON fe.post_id = p.id
      WHERE p.id = ANY($1::uuid[])
      GROUP BY p.id
      ON CONFLICT (post_id) DO UPDATE SET
        likes_count    = EXCLUDED.likes_count,
        comments_count = EXCLUDED.comments_count,
        shares_count   = EXCLUDED.shares_count,
        saves_count    = EXCLUDED.saves_count,
        views_count    = EXCLUDED.views_count,
        raw_score      = EXCLUDED.raw_score,
        updated_at     = NOW()
    `, [postIds]);
  }
}
```

### Step 255: Redis Feed Store

```typescript
// src/services/feed-store.ts

import Redis from 'ioredis';

export class FeedStore {
  private readonly FEED_TTL = 86400;          // 24 hours
  private readonly MAX_FEED_SIZE = 1000;       // เก็บไม่เกิน 1000 items
  private readonly SOS_SCORE_BOOST = 999999;   // SOS ต้องอยู่ top เสมอ

  constructor(private redis: Redis) {}

  // Key patterns
  private homeFeedKey(userId: string): string {
    return `feed:home:${userId}`;
  }

  private newPostsKey(userId: string): string {
    return `feed:new:${userId}`;
  }

  // เพิ่ม posts เข้า feed ของ user
  async addToFeed(
    userId: string,
    posts: Array<{ postId: string; score: number }>
  ): Promise<void> {
    const key = this.homeFeedKey(userId);
    const pipeline = this.redis.pipeline();

    for (const { postId, score } of posts) {
      pipeline.zadd(key, score, postId);
    }

    // Trim เพื่อไม่ให้ feed ใหญ่เกินไป
    pipeline.zremrangebyrank(key, 0, -(this.MAX_FEED_SIZE + 1));
    pipeline.expire(key, this.FEED_TTL);

    await pipeline.exec();
  }

  // เพิ่ม SOS alert เข้า feed (top priority)
  async injectSOSAlert(userId: string, alertId: string): Promise<void> {
    const key = this.homeFeedKey(userId);
    // SOS ใช้ score สูงมาก + timestamp เพื่อ sort ภายใน SOS
    const sosScore = this.SOS_SCORE_BOOST + (Date.now() / 1000);
    await this.redis.zadd(key, sosScore, `sos:${alertId}`);
    await this.redis.expire(key, this.FEED_TTL);
  }

  // ดึง feed ด้วย cursor-based pagination
  async getFeed(
    userId: string,
    cursor?: string,
    limit: number = 20
  ): Promise<{ items: string[]; nextCursor?: string; hasMore: boolean }> {
    const key = this.homeFeedKey(userId);

    // Decode cursor (score ของ item สุดท้าย)
    let maxScore = '+inf';
    if (cursor) {
      try {
        const decoded = Buffer.from(cursor, 'base64').toString('utf8');
        const cursorData = JSON.parse(decoded);
        maxScore = `(${cursorData.score}`; // ( = exclusive
      } catch {
        maxScore = '+inf';
      }
    }

    // ดึง limit+1 เพื่อเช็ค hasMore
    const results = await this.redis.zrangebyscore(
      key,
      '-inf',
      maxScore,
      'WITHSCORES',
      'LIMIT', 0, limit + 1
    );

    // Parse results (สลับกัน: member, score, member, score...)
    const items: string[] = [];
    let lastScore: string | undefined;

    for (let i = results.length - 2; i >= 0; i -= 2) {
      if (items.length < limit) {
        items.push(results[i]);
        lastScore = results[i + 1];
      }
    }

    const hasMore = results.length / 2 > limit;

    let nextCursor: string | undefined;
    if (hasMore && lastScore) {
      nextCursor = Buffer.from(
        JSON.stringify({ score: parseFloat(lastScore) })
      ).toString('base64');
    }

    return { items, nextCursor, hasMore };
  }

  // ติดตาม new posts count
  async trackNewPost(userId: string, postId: string): Promise<void> {
    const key = this.newPostsKey(userId);
    await this.redis.zadd(key, Date.now(), postId);
    await this.redis.expire(key, 3600); // 1 hour
  }

  // ดึงจำนวน new posts ตั้งแต่ last seen
  async getNewPostsCount(userId: string, lastSeenAt: Date): Promise<number> {
    const key = this.newPostsKey(userId);
    return await this.redis.zcount(key, lastSeenAt.getTime(), '+inf');
  }

  // Mark feed as seen (reset new posts counter)
  async markFeedSeen(userId: string): Promise<void> {
    const key = this.newPostsKey(userId);
    await this.redis.del(key);
  }

  // ลบ post ออกจาก feed (เมื่อถูก delete)
  async removeFromFeed(userId: string, postId: string): Promise<void> {
    await this.redis.zrem(this.homeFeedKey(userId), postId);
  }

  // เช็คว่า feed ถูก generate แล้วหรือยัง
  async isFeedCached(userId: string): Promise<boolean> {
    return (await this.redis.exists(this.homeFeedKey(userId))) === 1;
  }

  // Invalidate feed (บังคับ regenerate)
  async invalidateFeed(userId: string): Promise<void> {
    await this.redis.del(this.homeFeedKey(userId));
  }
}
```

### Step 256: Feed Generator (Fan-out Logic)

```typescript
// src/services/feed-generator.ts

import { Pool } from 'pg';
import Redis from 'ioredis';
import Queue from 'bull';
import { FeedStore } from './feed-store';
import { EngagementScorer } from './engagement-scorer';

const CELEBRITY_THRESHOLD = 1000; // followers > 1000 = celebrity

export class FeedGenerator {
  private fanoutQueue: Queue.Queue;

  constructor(
    private db: Pool,
    private redis: Redis,
    private feedStore: FeedStore,
    private scorer: EngagementScorer
  ) {
    this.fanoutQueue = new Queue('feed-fanout', {
      redis: { host: process.env.REDIS_HOST ?? 'localhost', port: 6379 },
    });

    this.setupWorker();
  }

  // เมื่อมี post ใหม่ → fan-out ไปหา followers
  async onNewPost(postId: string, authorId: string): Promise<void> {
    // ดู follower count ของ author
    const result = await this.db.query(
      'SELECT follower_count FROM user_stats WHERE user_id = $1',
      [authorId]
    );
    const followerCount = result.rows[0]?.follower_count ?? 0;

    if (followerCount < CELEBRITY_THRESHOLD) {
      // PUSH model: fan-out ทันที
      await this.fanoutQueue.add('fanout-post', {
        postId,
        authorId,
        strategy: 'push',
      }, {
        priority: 1,
        attempts: 3,
        backoff: { type: 'exponential', delay: 1000 },
      });
    } else {
      // Celebrity: ไม่ push แต่ update score
      await this.fanoutQueue.add('update-celebrity-post', {
        postId,
        authorId,
        strategy: 'pull',
      }, { priority: 5 });
    }
  }

  // Worker สำหรับ fan-out
  private setupWorker(): void {
    this.fanoutQueue.process('fanout-post', 10, async (job) => {
      const { postId, authorId } = job.data;

      // ดึง followers ทั้งหมด (batch 1000)
      let offset = 0;
      const batchSize = 1000;

      while (true) {
        const followers = await this.db.query(`
          SELECT follower_id
          FROM social_follows
          WHERE following_id = $1
          LIMIT $2 OFFSET $3
        `, [authorId, batchSize, offset]);

        if (followers.rows.length === 0) break;

        // คำนวณ score ของ post นี้
        const post = await this.getPostWithEngagement(postId);
        if (!post) break;

        // Update feed ของแต่ละ follower
        const pipeline = this.redis.pipeline();
        for (const { follower_id } of followers.rows) {
          const feedKey = `feed:home:${follower_id}`;
          const score = await this.scorer.calculateFinalScore(post, follower_id);
          pipeline.zadd(feedKey, score, postId);
          pipeline.zremrangebyrank(feedKey, 0, -1001); // trim to 1000
          pipeline.expire(feedKey, 86400);

          // Track as new post
          await this.feedStore.trackNewPost(follower_id, postId);
        }

        await pipeline.exec();
        offset += batchSize;

        // Rate limit: ไม่ทำหนักเกินไป
        if (followers.rows.length === batchSize) {
          await new Promise(resolve => setTimeout(resolve, 10));
        }
      }
    });
  }

  // Generate feed สำหรับ user ที่ยังไม่มี feed cache
  async generateFeedOnDemand(userId: string): Promise<void> {
    // ดึง following list
    const following = await this.db.query(`
      SELECT following_id FROM social_follows
      WHERE follower_id = $1
    `, [userId]);

    if (following.rows.length === 0) {
      // ยังไม่ follow ใคร → ใช้ explore feed
      await this.generateExploreFeed(userId);
      return;
    }

    const followingIds = following.rows.map((r: any) => r.following_id);

    // ดึง posts จาก following (24 ชั่วโมงล่าสุด) ด้วย window function
    const posts = await this.db.query(`
      SELECT
        p.id         AS post_id,
        p.author_id,
        p.created_at,
        COALESCE(pes.likes_count, 0)    AS likes,
        COALESCE(pes.comments_count, 0) AS comments,
        COALESCE(pes.shares_count, 0)   AS shares,
        COALESCE(pes.saves_count, 0)    AS saves,
        COALESCE(pes.views_count, 0)    AS views,
        ROW_NUMBER() OVER (
          PARTITION BY p.author_id
          ORDER BY p.created_at DESC
        ) AS author_rank
      FROM posts p
      LEFT JOIN post_engagement_scores pes ON pes.post_id = p.id
      WHERE p.author_id = ANY($1::uuid[])
        AND p.created_at > NOW() - INTERVAL '24 hours'
        AND p.status = 'published'
      ORDER BY p.created_at DESC
      LIMIT 500
    `, [followingIds]);

    // คำนวณ score และเพิ่มใน feed
    const feedItems: Array<{ postId: string; score: number }> = [];

    for (const row of posts.rows) {
      const score = await this.scorer.calculateFinalScore({
        postId: row.post_id,
        authorId: row.author_id,
        likes: row.likes,
        comments: row.comments,
        shares: row.shares,
        saves: row.saves,
        views: row.views,
        createdAt: row.created_at,
      }, userId);

      feedItems.push({ postId: row.post_id, score });
    }

    await this.feedStore.addToFeed(userId, feedItems);
  }

  // Generate explore feed (สำหรับ user ใหม่หรือที่ไม่มี following)
  async generateExploreFeed(userId: string): Promise<void> {
    const posts = await this.db.query(`
      SELECT
        p.id AS post_id,
        p.author_id,
        p.created_at,
        COALESCE(pes.raw_score, 0) AS raw_score
      FROM posts p
      LEFT JOIN post_engagement_scores pes ON pes.post_id = p.id
      WHERE p.created_at > NOW() - INTERVAL '48 hours'
        AND p.status = 'published'
        AND p.author_id != $1
      ORDER BY COALESCE(pes.raw_score, 0) DESC
      LIMIT 100
    `, [userId]);

    const feedItems = posts.rows.map((row: any) => ({
      postId: row.post_id,
      score: row.raw_score,
    }));

    await this.feedStore.addToFeed(userId, feedItems);
  }

  private async getPostWithEngagement(postId: string): Promise<any> {
    const result = await this.db.query(`
      SELECT
        p.id AS post_id, p.author_id, p.created_at,
        COALESCE(pes.likes_count, 0) AS likes,
        COALESCE(pes.comments_count, 0) AS comments,
        COALESCE(pes.shares_count, 0) AS shares,
        COALESCE(pes.saves_count, 0) AS saves,
        COALESCE(pes.views_count, 0) AS views
      FROM posts p
      LEFT JOIN post_engagement_scores pes ON pes.post_id = p.id
      WHERE p.id = $1
    `, [postId]);

    return result.rows[0] ?? null;
  }
}
```

### Step 257: Feed API Endpoints

```typescript
// src/routes/feed.routes.ts

import { Router, Request, Response } from 'express';
import { FeedStore } from '../services/feed-store';
import { FeedGenerator } from '../services/feed-generator';
import { Pool } from 'pg';
import Redis from 'ioredis';

export function createFeedRouter(
  db: Pool,
  redis: Redis,
  feedStore: FeedStore,
  feedGenerator: FeedGenerator
): Router {
  const router = Router();

  // GET /api/v1/feed/home
  router.get('/home', async (req: Request, res: Response) => {
    try {
      const userId = req.user!.id; // จาก auth middleware
      const cursor = req.query.cursor as string | undefined;
      const limit = Math.min(parseInt(req.query.limit as string) || 20, 50);

      // เช็คว่ามี feed cache ไหม
      const isCached = await feedStore.isFeedCached(userId);
      if (!isCached) {
        await feedGenerator.generateFeedOnDemand(userId);
      }

      // ดึง feed จาก Redis
      const { items: postIds, nextCursor, hasMore } = await feedStore.getFeed(
        userId,
        cursor,
        limit
      );

      // แยก SOS และ regular posts
      const sosIds = postIds.filter(id => id.startsWith('sos:'));
      const regularIds = postIds.filter(id => !id.startsWith('sos:'));

      // ดึง post details
      const posts = regularIds.length > 0
        ? await fetchPostDetails(db, regularIds, userId)
        : [];

      // ดึง SOS details
      const sosAlerts = sosIds.length > 0
        ? await fetchSOSDetails(db, sosIds.map(id => id.replace('sos:', '')))
        : [];

      // New posts count
      const lastSeenAt = req.cookies?.lastFeedSeen
        ? new Date(req.cookies.lastFeedSeen)
        : new Date(Date.now() - 3600000);
      const newPostsCount = await feedStore.getNewPostsCount(userId, lastSeenAt);

      // Mark as seen
      res.cookie('lastFeedSeen', new Date().toISOString(), {
        httpOnly: true,
        maxAge: 86400000,
      });
      await feedStore.markFeedSeen(userId);

      res.json({
        items: [...sosAlerts, ...posts],
        nextCursor,
        hasMore,
        newPostsCount,
        meta: {
          generatedAt: new Date().toISOString(),
          strategy: isCached ? 'cache' : 'generated',
        },
      });
    } catch (error) {
      console.error('Feed error:', error);
      res.status(500).json({ error: 'Failed to load feed' });
    }
  });

  // POST /api/v1/feed/event (track engagement)
  router.post('/event', async (req: Request, res: Response) => {
    const { postId, eventType } = req.body;
    const userId = req.user!.id;

    const validEvents = ['view', 'like', 'comment', 'share', 'save'];
    if (!validEvents.includes(eventType)) {
      return res.status(400).json({ error: 'Invalid event type' });
    }

    await db.query(
      'INSERT INTO feed_events (user_id, post_id, event_type) VALUES ($1, $2, $3)',
      [userId, postId, eventType]
    );

    // Queue score update
    await redis.lpush('score-update-queue', postId);

    res.json({ ok: true });
  });

  // GET /api/v1/feed/new-count (polling สำหรับ "5 new posts")
  router.get('/new-count', async (req: Request, res: Response) => {
    const userId = req.user!.id;
    const since = req.query.since
      ? new Date(req.query.since as string)
      : new Date(Date.now() - 3600000);

    const count = await feedStore.getNewPostsCount(userId, since);
    res.json({ count, since: since.toISOString() });
  });

  return router;
}

async function fetchPostDetails(
  db: Pool,
  postIds: string[],
  viewerId: string
): Promise<any[]> {
  if (postIds.length === 0) return [];

  const result = await db.query(`
    SELECT
      p.id, p.content, p.media_urls, p.tags,
      p.created_at, p.author_id,
      u.display_name AS author_name,
      u.avatar_url   AS author_avatar,
      COALESCE(pes.likes_count, 0)    AS likes_count,
      COALESCE(pes.comments_count, 0) AS comments_count,
      COALESCE(pes.shares_count, 0)   AS shares_count,
      EXISTS(
        SELECT 1 FROM feed_events
        WHERE user_id = $2 AND post_id = p.id AND event_type = 'like'
      ) AS user_liked
    FROM posts p
    JOIN users u ON u.id = p.author_id
    LEFT JOIN post_engagement_scores pes ON pes.post_id = p.id
    WHERE p.id = ANY($1::uuid[])
      AND p.status = 'published'
  `, [postIds, viewerId]);

  // รักษาลำดับตาม postIds
  const postMap = new Map(result.rows.map((r: any) => [r.id, r]));
  return postIds
    .filter(id => postMap.has(id))
    .map(id => postMap.get(id));
}

async function fetchSOSDetails(db: Pool, alertIds: string[]): Promise<any[]> {
  if (alertIds.length === 0) return [];

  const result = await db.query(`
    SELECT id, type, severity, title, description,
           lat, lng, radius_km, status, created_at
    FROM sos_alerts
    WHERE id = ANY($1::uuid[])
      AND status IN ('active', 'verified')
    ORDER BY severity DESC, created_at DESC
  `, [alertIds]);

  return result.rows.map((r: any) => ({ ...r, isSOS: true }));
}
```

### Step 258: A/B Test Framework

```typescript
// src/services/ab-test.ts

import { Pool } from 'pg';
import crypto from 'crypto';

export interface Variant {
  name: string;
  weight: number;
  config: Record<string, any>;
}

export interface Test {
  testName: string;
  variants: Variant[];
  startDate: Date;
  endDate: Date;
  isActive: boolean;
}

export class ABTestFramework {
  private testsCache = new Map<string, Test>();

  constructor(private db: Pool) {}

  // Assign user to variant (deterministic: same user → same variant)
  async getVariant(userId: string, testName: string): Promise<Variant | null> {
    const test = await this.getTest(testName);
    if (!test || !test.isActive) return null;

    // เช็ค assignment ที่มีอยู่แล้ว
    const existing = await this.db.query(`
      SELECT variant FROM ab_test_assignments
      WHERE user_id = $1 AND test_name = $2
    `, [userId, testName]);

    if (existing.rows.length > 0) {
      const variantName = existing.rows[0].variant;
      return test.variants.find(v => v.name === variantName) ?? null;
    }

    // Assign ใหม่ (deterministic hash)
    const hash = crypto
      .createHash('sha256')
      .update(`${userId}:${testName}`)
      .digest('hex');

    const hashValue = parseInt(hash.slice(0, 8), 16) / 0xffffffff; // 0-1

    let cumulative = 0;
    let assignedVariant: Variant | null = null;
    for (const variant of test.variants) {
      cumulative += variant.weight;
      if (hashValue <= cumulative) {
        assignedVariant = variant;
        break;
      }
    }

    if (!assignedVariant) {
      assignedVariant = test.variants[test.variants.length - 1];
    }

    // Save assignment
    await this.db.query(`
      INSERT INTO ab_test_assignments (user_id, test_name, variant)
      VALUES ($1, $2, $3)
      ON CONFLICT (user_id, test_name) DO NOTHING
    `, [userId, testName, assignedVariant.name]);

    return assignedVariant;
  }

  // สร้าง test ใหม่
  async createTest(test: Omit<Test, 'isActive'>): Promise<void> {
    const totalWeight = test.variants.reduce((sum, v) => sum + v.weight, 0);
    if (Math.abs(totalWeight - 1.0) > 0.001) {
      throw new Error(`Variant weights must sum to 1.0, got ${totalWeight}`);
    }

    await this.db.query(`
      INSERT INTO ab_tests (test_name, variants, start_date, end_date, is_active)
      VALUES ($1, $2::jsonb, $3, $4, true)
    `, [test.testName, JSON.stringify(test.variants), test.startDate, test.endDate]);

    this.testsCache.delete(test.testName);
  }

  private async getTest(testName: string): Promise<Test | null> {
    if (this.testsCache.has(testName)) {
      return this.testsCache.get(testName)!;
    }

    const result = await this.db.query(`
      SELECT * FROM ab_tests
      WHERE test_name = $1
        AND is_active = true
        AND start_date <= NOW()
        AND end_date >= NOW()
    `, [testName]);

    if (result.rows.length === 0) return null;

    const test: Test = {
      testName: result.rows[0].test_name,
      variants: result.rows[0].variants,
      startDate: result.rows[0].start_date,
      endDate: result.rows[0].end_date,
      isActive: result.rows[0].is_active,
    };

    this.testsCache.set(testName, test);
    setTimeout(() => this.testsCache.delete(testName), 300000); // cache 5 min
    return test;
  }
}
```

### Step 259: SOS Priority Injection

```typescript
// src/services/sos-injector.ts

import { Pool } from 'pg';
import Redis from 'ioredis';
import { FeedStore } from './feed-store';

export class SOSInjector {
  constructor(
    private db: Pool,
    private redis: Redis,
    private feedStore: FeedStore
  ) {}

  // เมื่อมี SOS alert ใหม่ → inject เข้า feed ของ users รอบๆ
  async injectSOSToNearbyFeeds(alertId: string): Promise<void> {
    const alert = await this.db.query(`
      SELECT id, lat, lng, radius_km, severity
      FROM sos_alerts
      WHERE id = $1
    `, [alertId]);

    if (alert.rows.length === 0) return;

    const { lat, lng, radius_km } = alert.rows[0];

    // หา users ในรัศมี (ใช้ PostGIS)
    const nearbyUsers = await this.db.query(`
      SELECT user_id
      FROM user_locations
      WHERE ST_DWithin(
        location::geography,
        ST_MakePoint($2, $1)::geography,
        $3 * 1000  -- km to meters
      )
      LIMIT 10000
    `, [lat, lng, radius_km]);

    // Inject SOS ไปใน feed ของ users ทุกคน
    const pipeline = this.redis.pipeline();
    for (const { user_id } of nearbyUsers.rows) {
      const feedKey = `feed:home:${user_id}`;
      const sosScore = 999999 + (Date.now() / 1000);
      pipeline.zadd(feedKey, sosScore, `sos:${alertId}`);
      pipeline.expire(feedKey, 86400);
    }
    await pipeline.exec();

    console.log(
      `Injected SOS ${alertId} into ${nearbyUsers.rows.length} feeds`
    );
  }

  // Remove expired SOS from feeds
  async removeExpiredSOS(alertId: string): Promise<void> {
    // หา users ที่อาจมี SOS นี้ในฟีด
    const users = await this.db.query(`
      SELECT DISTINCT user_id
      FROM feed_cache_metadata
      WHERE last_generated > NOW() - INTERVAL '24 hours'
    `);

    const pipeline = this.redis.pipeline();
    for (const { user_id } of users.rows) {
      pipeline.zrem(`feed:home:${user_id}`, `sos:${alertId}`);
    }
    await pipeline.exec();
  }
}
```

---

## 🔧 Configuration Files

```typescript
// src/config/feed.config.ts

export const FEED_CONFIG = {
  // Feed sizes
  MAX_FEED_SIZE: 1000,
  DEFAULT_PAGE_SIZE: 20,
  MAX_PAGE_SIZE: 50,

  // TTLs
  FEED_TTL_SECONDS: 86400,       // 24 hours
  NEW_POSTS_TTL_SECONDS: 3600,   // 1 hour
  SCORE_CACHE_TTL: 3600,

  // Fan-out
  CELEBRITY_THRESHOLD: 1000,     // followers > 1000 = celebrity
  FANOUT_BATCH_SIZE: 1000,
  FANOUT_CONCURRENCY: 10,

  // Scoring
  ENGAGEMENT_WEIGHTS: {
    like: 1,
    comment: 3,
    share: 5,
    save: 10,
    view: 0.1,
    clickThrough: 2,
  },
  GRAVITY: 1.8,

  // SOS
  SOS_SCORE_BOOST: 999999,

  // A/B Tests
  AB_TESTS: {
    FEED_ALGORITHM_V2: {
      testName: 'feed_algorithm_v2',
      variants: [
        { name: 'control', weight: 0.5, config: { gravity: 1.8 } },
        { name: 'treatment', weight: 0.5, config: { gravity: 2.2 } },
      ],
    },
  },
};
```

---

## 🧪 Testing

### Step 260: Integration Tests

```typescript
// tests/feed.test.ts

import { describe, it, expect, beforeAll, afterAll } from '@jest/globals';
import { Pool } from 'pg';
import Redis from 'ioredis';
import { FeedStore } from '../src/services/feed-store';
import { EngagementScorer } from '../src/services/engagement-scorer';

describe('FeedStore', () => {
  let redis: Redis;
  let feedStore: FeedStore;

  beforeAll(async () => {
    redis = new Redis({ host: 'localhost', port: 6379 });
    feedStore = new FeedStore(redis);
    await redis.flushdb(); // clean test DB
  });

  afterAll(async () => {
    await redis.quit();
  });

  it('should add posts to feed', async () => {
    const userId = 'test-user-001';
    await feedStore.addToFeed(userId, [
      { postId: 'post-001', score: 100 },
      { postId: 'post-002', score: 200 },
      { postId: 'post-003', score: 50 },
    ]);

    const { items } = await feedStore.getFeed(userId);
    expect(items).toContain('post-002'); // highest score first
    expect(items.length).toBe(3);
  });

  it('should inject SOS at top of feed', async () => {
    const userId = 'test-user-002';
    await feedStore.addToFeed(userId, [
      { postId: 'post-100', score: 1000 },
    ]);
    await feedStore.injectSOSAlert(userId, 'alert-001');

    const { items } = await feedStore.getFeed(userId);
    expect(items[0]).toBe('sos:alert-001'); // SOS ต้องอยู่ top
  });

  it('should paginate with cursor', async () => {
    const userId = 'test-user-003';
    const posts = Array.from({ length: 50 }, (_, i) => ({
      postId: `post-${i}`,
      score: i * 10,
    }));
    await feedStore.addToFeed(userId, posts);

    const page1 = await feedStore.getFeed(userId, undefined, 20);
    expect(page1.items.length).toBe(20);
    expect(page1.hasMore).toBe(true);

    const page2 = await feedStore.getFeed(userId, page1.nextCursor, 20);
    expect(page2.items.length).toBe(20);

    // ไม่ซ้ำกับ page1
    const overlap = page1.items.filter(id => page2.items.includes(id));
    expect(overlap.length).toBe(0);
  });
});

describe('EngagementScorer', () => {
  it('should calculate correct raw score', () => {
    const scorer = new EngagementScorer({} as Pool, {} as Redis);

    const score = scorer.calculateRawScore(
      10, // likes × 1 = 10
      5,  // comments × 3 = 15
      2,  // shares × 5 = 10
      1,  // saves × 10 = 10
      100, // views × 0.1 = 10
      0
    );

    expect(score).toBe(55); // 10 + 15 + 10 + 10 + 10
  });

  it('should decay score over time', () => {
    const scorer = new EngagementScorer({} as Pool, {} as Redis);
    const rawScore = 100;

    const now = new Date();
    const oneHourAgo = new Date(now.getTime() - 3600000);
    const oneDayAgo = new Date(now.getTime() - 86400000);

    const recentScore = scorer.calculateDecayScore(rawScore, now);
    const oldScore = scorer.calculateDecayScore(rawScore, oneDayAgo);

    expect(recentScore).toBeGreaterThan(oldScore);
  });
});
```

```bash
# รัน tests
cd /home/user/chuaikan/services/feed-service
npm test

# Load test: simulate 10,000 feed reads
npm install -D artillery

cat > artillery-feed.yml << 'EOF'
config:
  target: "http://localhost:3004"
  phases:
    - duration: 60
      arrivalRate: 167  # ~10,000 req/min
  defaults:
    headers:
      Authorization: "Bearer test-token"

scenarios:
  - name: "Load Feed"
    flow:
      - get:
          url: "/api/v1/feed/home"
          qs:
            limit: 20
EOF

npx artillery run artillery-feed.yml
```

---

## ❌ Common Errors & Solutions

### Error 1: Redis Memory Full

```
Error: OOM command not allowed when used memory > 'maxmemory'
```

```bash
# แก้: ตั้ง maxmemory policy
redis-cli CONFIG SET maxmemory 2gb
redis-cli CONFIG SET maxmemory-policy allkeys-lru

# หรือใน redis.conf
echo "maxmemory 2gb" >> /etc/redis/redis.conf
echo "maxmemory-policy allkeys-lru" >> /etc/redis/redis.conf
sudo systemctl restart redis
```

### Error 2: Fan-out Too Slow for Large Following

```
Bull job timeout: fanout-post exceeded 30s
```

```typescript
// แก้: เพิ่ม concurrency และ batch size
this.fanoutQueue.process('fanout-post', 50, async (job) => {
  // เพิ่ม concurrency จาก 10 → 50
});

// หรือใช้ Redis Pipeline แบบ batch ใหญ่ขึ้น
const FANOUT_BATCH = 5000; // จาก 1000 → 5000
```

### Error 3: Cursor Decode Error

```
SyntaxError: Unexpected token in JSON at position 0
```

```typescript
// แก้: ตรวจสอบ cursor ก่อน decode
function decodeCursor(cursor: string): { score: number } | null {
  try {
    const decoded = Buffer.from(cursor, 'base64url').toString('utf8');
    return JSON.parse(decoded);
  } catch {
    return null; // Invalid cursor → start from beginning
  }
}
```

---

## ✅ Checklist

- [ ] **Step 251**: ติดตั้ง dependencies และ configure TypeScript
- [ ] **Step 252**: รัน migration สร้าง feed_events, post_engagement_scores tables
- [ ] **Step 253**: สร้าง feed.types.ts
- [ ] **Step 254**: Implement EngagementScorer พร้อม decay formula
- [ ] **Step 255**: Implement FeedStore ด้วย Redis Sorted Sets
- [ ] **Step 256**: Implement FeedGenerator ด้วย Hybrid fan-out
- [ ] **Step 257**: สร้าง Feed API endpoints (GET /home, POST /event, GET /new-count)
- [ ] **Step 258**: Implement A/B Test Framework
- [ ] **Step 259**: Implement SOS Priority Injection
- [ ] **Step 260**: รัน Integration Tests และ Load Tests
- [ ] Verify cursor-based pagination ทำงานถูกต้อง
- [ ] Verify SOS items อยู่ top ของ feed เสมอ
- [ ] Verify feed TTL 24 ชั่วโมง
- [ ] Verify "new posts" counter reset เมื่อ open feed
- [ ] Load test ผ่าน: 10,000 feed reads/min ที่ p99 < 100ms

---

## 🔗 References

- [Redis Sorted Sets Documentation](https://redis.io/docs/data-types/sorted-sets/)
- [Bull Queue Documentation](https://github.com/OptimalBits/bull)
- [Hacker News Ranking Algorithm](https://news.ycombinator.com/item?id=1781013)
- [Instagram Feed Engineering](https://instagram-engineering.com/feed-ranking-and-personalization)
- [Twitter Timeline Architecture](https://www.infoq.com/presentations/Twitter-Timeline-Scalability/)

---
*Part 026 | Road to 1,000,000 Users/Day | chuaikan.com*
