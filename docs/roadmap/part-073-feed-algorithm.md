# Part 073: Real-time Feed Algorithm

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Advanced
> **Steps:** 721-730
> **เวลาโดยประมาณ:** 5 ชั่วโมง
> **Prerequisites:** Part 072 (Kafka), Part 064 (DB Profiling)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- Feed ranking factors (EdgeRank-inspired algorithm)
- Affinity score, weight by content type, time decay
- Pre-computing feed scores ด้วย background job
- Redis sorted set สำหรับ feed storage
- Collaborative filtering basics
- Fan-out on write vs fan-out on read
- Algorithm A/B testing

---

## 📖 ทฤษฎีและแนวคิด

### Feed Algorithm Overview

```
Feed Score = Affinity × ContentWeight × TimeDecay × BoostFactor

Affinity     = ความสัมพันธ์ระหว่าง user กับ author
               (likes, comments, shares, follows, profile views)

ContentWeight = ค่าน้ำหนักของ content type
               Photo: 1.0, Video: 1.5, SOS: 5.0, Text: 0.8

TimeDecay    = e^(-λ × hours_since_post)
               λ = 0.05 (posts หาย ~50% ใน 14 ชั่วโมง)

BoostFactor  = manual boosts (promoted, trending, SOS)
```

### Fan-out Strategy

```
Fan-out on Write (Push Model):
  User A posts → compute feed for ALL followers immediately
  ✅ Fast read (O(1) per user)
  ❌ Slow write (O(followers) per post)
  → ใช้สำหรับ users ที่มี <10,000 followers

Fan-out on Read (Pull Model):
  User reads feed → aggregate posts from followed users
  ✅ Fast write
  ❌ Slow read (O(following_count))
  → ใช้สำหรับ celebrities/influencers

chuaikan.com ใช้ Hybrid:
  - Normal users (<10k followers): fan-out on write
  - Influencers (>10k followers): fan-out on read
  - SOS alerts: real-time push to all area subscribers
```

---

## ⚙️ Environment Setup

### Step 721: ติดตั้ง Dependencies

```bash
npm install bull ioredis prom-client
```

```javascript
// config/redis.js
const Redis = require('ioredis');

const redis = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT || '6379'),
  password: process.env.REDIS_PASSWORD,
  maxRetriesPerRequest: 3,
  lazyConnect: false,
  enableReadyCheck: true,
});

module.exports = redis;
```

---

## 🛠️ Step-by-Step Implementation

### Step 722: Feed Score Calculation

```javascript
// feed/scoring.js - Feed ranking algorithm

const CONTENT_WEIGHTS = {
  sos_alert:   5.0,   // SOS alerts มีความสำคัญสูงสุด
  video:       1.5,   // Videos มี engagement สูง
  photo:       1.0,   // Photos = baseline
  carousel:    1.2,   // Multiple images
  text:        0.8,   // Text-only posts
  share:       0.7,   // Shared content
};

const TIME_DECAY_LAMBDA = 0.05;  // ครึ่งชีวิต ~14 ชั่วโมง

/**
 * คำนวณ time decay factor
 * f(t) = e^(-λt) โดย t = ชั่วโมงตั้งแต่ post
 */
function calculateTimeDecay(createdAt) {
  const hoursSincePost = (Date.now() - new Date(createdAt).getTime()) / 3600000;
  return Math.exp(-TIME_DECAY_LAMBDA * hoursSincePost);
}

/**
 * คำนวณ affinity score ระหว่าง user กับ author
 * อ้างอิงจาก EdgeRank algorithm ของ Facebook
 */
async function calculateAffinity(userId, authorId, db) {
  const interactions = await db.query(`
    SELECT
      SUM(CASE WHEN type = 'like'    THEN 1 ELSE 0 END) * 1.0 AS likes,
      SUM(CASE WHEN type = 'comment' THEN 2 ELSE 0 END) * 2.0 AS comments,
      SUM(CASE WHEN type = 'share'   THEN 3 ELSE 0 END) * 3.0 AS shares,
      SUM(CASE WHEN type = 'view'    THEN 0.1 ELSE 0 END) AS views
    FROM user_interactions
    WHERE user_id = $1
      AND target_author_id = $2
      AND created_at > NOW() - INTERVAL '30 days'
  `, [userId, authorId]);

  const { likes = 0, comments = 0, shares = 0, views = 0 } = interactions.rows[0];
  const rawAffinity = likes + comments + shares + views;

  // Normalize ให้ได้ 0-1
  return Math.min(rawAffinity / 100, 1.0);
}

/**
 * คำนวณ final feed score สำหรับ post
 */
async function calculateFeedScore(userId, post, db) {
  const [affinity, timeDecay] = await Promise.all([
    calculateAffinity(userId, post.author_id, db),
    Promise.resolve(calculateTimeDecay(post.created_at)),
  ]);

  const contentWeight = CONTENT_WEIGHTS[post.type] || 1.0;

  // Base score
  let score = affinity * contentWeight * timeDecay;

  // SOS boost: SOS alerts ใน area เดียวกัน
  if (post.type === 'sos_alert' && post.area_code === await getUserAreaCode(userId, db)) {
    score *= 3.0;  // Triple boost for local SOS
  }

  // Engagement boost (viral content)
  const engagementRate = post.likes_count + post.comments_count * 2 + post.shares_count * 3;
  const engagementBoost = Math.log10(Math.max(engagementRate, 1)) * 0.1;
  score += engagementBoost;

  // Trending boost (trending ใน 1 ชั่วโมงที่ผ่านมา)
  if (post.is_trending) {
    score *= 1.5;
  }

  return score;
}

module.exports = { calculateFeedScore, calculateTimeDecay, calculateAffinity };
```

### Step 723: Redis Sorted Set สำหรับ Feed

```javascript
// feed/feed-store.js - Feed storage ด้วย Redis sorted set
const redis = require('../config/redis');

const FEED_TTL = 86400 * 7;  // Feed cache 7 วัน
const MAX_FEED_SIZE = 1000;   // เก็บ 1000 posts ต่อ user

/**
 * เพิ่ม post ลงใน user's feed
 * Redis ZADD feed:{userId} {score} {postId}
 */
async function addToFeed(userId, postId, score) {
  const feedKey = `feed:${userId}`;
  
  await redis.multi()
    .zadd(feedKey, score, postId.toString())
    .expire(feedKey, FEED_TTL)
    .exec();
  
  // ตัด feed ให้ไม่เกิน MAX_FEED_SIZE (ตัดคะแนนต่ำสุด)
  const feedSize = await redis.zcard(feedKey);
  if (feedSize > MAX_FEED_SIZE) {
    await redis.zremrangebyrank(feedKey, 0, feedSize - MAX_FEED_SIZE - 1);
  }
}

/**
 * ดึง feed สำหรับ user (paginated)
 */
async function getFeed(userId, cursor = 0, limit = 20) {
  const feedKey = `feed:${userId}`;
  
  // ZREVRANGEBYSCORE ดึง posts เรียงจากคะแนนสูงไปต่ำ
  const postIds = await redis.zrevrangebyscore(
    feedKey,
    '+inf',
    '-inf',
    'LIMIT',
    cursor,
    limit
  );
  
  return postIds;
}

/**
 * ลบ post ออกจาก feed (เช่น post ถูกลบ)
 */
async function removeFromFeed(userId, postId) {
  await redis.zrem(`feed:${userId}`, postId.toString());
}

/**
 * Fan-out on write: เพิ่ม post ลงใน feeds ของ followers ทั้งหมด
 */
async function fanOutToFollowers(post, db) {
  const { rows: followers } = await db.query(`
    SELECT follower_id FROM follows
    WHERE following_id = $1 AND is_active = TRUE
  `, [post.author_id]);

  if (followers.length === 0) return;

  // Compute scores แบบ batch สำหรับ followers
  const pipeline = redis.pipeline();
  
  for (const { follower_id } of followers) {
    // ใช้ approximate score สำหรับ fan-out (fast path)
    const timeDecay = calculateTimeDecaySync(post.created_at);
    const contentWeight = CONTENT_WEIGHTS[post.type] || 1.0;
    const approximateScore = timeDecay * contentWeight;
    
    const feedKey = `feed:${follower_id}`;
    pipeline.zadd(feedKey, approximateScore, post.id.toString());
    pipeline.expire(feedKey, FEED_TTL);
  }
  
  await pipeline.exec();
  
  console.log(`Fan-out post ${post.id} to ${followers.length} followers`);
}

function calculateTimeDecaySync(createdAt) {
  const hoursSincePost = (Date.now() - new Date(createdAt).getTime()) / 3600000;
  return Math.exp(-0.05 * hoursSincePost);
}

module.exports = { addToFeed, getFeed, removeFromFeed, fanOutToFollowers };
```

### Step 724: Background Job สำหรับ Feed Pre-computation

```javascript
// feed/feed-worker.js - Background job ด้วย BullMQ
const { Queue, Worker } = require('bullmq');
const { calculateFeedScore } = require('./scoring');
const { addToFeed, fanOutToFollowers } = require('./feed-store');
const db = require('../config/db');

const connection = {
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT || '6379'),
};

// Feed update queue
const feedQueue = new Queue('feed-updates', { connection });

// Worker สำหรับ process feed updates
const feedWorker = new Worker('feed-updates', async (job) => {
  const { type, data } = job.data;
  
  switch (type) {
    case 'NEW_POST':
      await handleNewPost(data.post);
      break;
    
    case 'POST_DELETED':
      await handlePostDeleted(data.postId, data.authorId);
      break;
    
    case 'RECOMPUTE_SCORES':
      await recomputeUserFeedScores(data.userId);
      break;
  }
}, {
  connection,
  concurrency: 20,
  limiter: {
    max: 1000,
    duration: 1000,  // 1000 jobs/second
  },
});

async function handleNewPost(post) {
  // Fan-out ไปยัง followers (push model)
  await fanOutToFollowers(post, db);
  
  // สำหรับ SOS alerts: push ทันที ไม่รอ fan-out
  if (post.type === 'sos_alert') {
    await pushSOSAlertToAreaSubscribers(post);
  }
}

async function pushSOSAlertToAreaSubscribers(alert) {
  const { rows: subscribers } = await db.query(`
    SELECT user_id FROM area_subscriptions
    WHERE area_code = $1 AND is_active = TRUE
  `, [alert.area_code]);
  
  const pipeline = require('../config/redis').pipeline();
  
  for (const { user_id } of subscribers) {
    // SOS score สูงมาก (> 10) เพื่อให้อยู่บนสุดของ feed
    const sosScore = 100 + (Date.now() / 1000000);
    pipeline.zadd(`feed:${user_id}`, sosScore, alert.id.toString());
  }
  
  await pipeline.exec();
  console.log(`SOS alert ${alert.id} pushed to ${subscribers.length} area subscribers`);
}

// Periodic recompute: update scores ทุก 15 นาที
async function scheduleScoreRecomputation() {
  const { rows: activeUsers } = await db.query(`
    SELECT id FROM users
    WHERE last_active_at > NOW() - INTERVAL '7 days'
    ORDER BY last_active_at DESC
    LIMIT 100000
  `);
  
  for (const user of activeUsers) {
    await feedQueue.add('recompute', {
      type: 'RECOMPUTE_SCORES',
      data: { userId: user.id },
    }, {
      delay: Math.random() * 900000,  // spread over 15 minutes
      removeOnComplete: 100,
    });
  }
}

// รัน recomputation ทุก 15 นาที
setInterval(scheduleScoreRecomputation, 900000);

module.exports = { feedQueue, feedWorker };
```

### Step 725: Feed API Endpoint

```javascript
// routes/feed.js
const express = require('express');
const router = express.Router();
const { getFeed } = require('../feed/feed-store');
const db = require('../config/db');
const redis = require('../config/redis');

/**
 * GET /api/feed
 * ดึง personalized feed ของ user
 */
router.get('/', async (req, res) => {
  const userId = req.user.id;
  const { cursor = 0, limit = 20 } = req.query;
  
  try {
    // ดึง post IDs จาก Redis sorted set
    const postIds = await getFeed(userId, parseInt(cursor), parseInt(limit));
    
    if (postIds.length === 0) {
      // Feed empty: generate fallback feed
      const fallbackPosts = await generateFallbackFeed(userId, parseInt(limit));
      return res.json({
        posts: fallbackPosts,
        nextCursor: null,
        source: 'fallback',
      });
    }
    
    // ดึง post details จาก PostgreSQL
    const { rows: posts } = await db.query(`
      SELECT
        p.*,
        u.username, u.display_name, u.avatar_url, u.is_verified,
        COALESCE(pl.liked, FALSE) AS user_has_liked,
        COALESCE(ps.saved, FALSE) AS user_has_saved
      FROM posts p
      JOIN users u ON u.id = p.author_id
      LEFT JOIN LATERAL (
        SELECT TRUE AS liked
        FROM post_likes
        WHERE post_id = p.id AND user_id = $1
      ) pl ON TRUE
      LEFT JOIN LATERAL (
        SELECT TRUE AS saved
        FROM post_saves
        WHERE post_id = p.id AND user_id = $1
      ) ps ON TRUE
      WHERE p.id = ANY($2)
        AND p.is_deleted = FALSE
        AND p.is_published = TRUE
      ORDER BY array_position($2::text[], p.id::text)
    `, [userId, postIds]);
    
    res.json({
      posts,
      nextCursor: postIds.length >= parseInt(limit) ? parseInt(cursor) + parseInt(limit) : null,
      source: 'personalized',
    });
    
  } catch (err) {
    console.error('Feed error:', err);
    res.status(500).json({ error: 'Failed to load feed' });
  }
});

/**
 * Fallback feed: trending posts เมื่อ feed ว่าง
 */
async function generateFallbackFeed(userId, limit) {
  const { rows } = await db.query(`
    SELECT p.*, u.username, u.display_name, u.avatar_url
    FROM posts p
    JOIN users u ON u.id = p.author_id
    WHERE p.created_at > NOW() - INTERVAL '24 hours'
      AND p.is_deleted = FALSE
      AND p.is_published = TRUE
    ORDER BY (p.likes_count + p.comments_count * 2 + p.shares_count * 3) DESC
    LIMIT $1
  `, [limit]);
  
  return rows;
}

module.exports = router;
```

### Step 726: Collaborative Filtering Basics

```javascript
// feed/collaborative-filter.js
// User-based collaborative filtering (simplified)

/**
 * หา similar users โดยใช้ Jaccard similarity
 * Users ที่ like/follow posts คล้ายกัน = similar users
 */
async function findSimilarUsers(userId, db, limit = 20) {
  const { rows } = await db.query(`
    WITH user_posts AS (
      SELECT DISTINCT post_id
      FROM post_likes
      WHERE user_id = $1
      ORDER BY post_id
      LIMIT 1000
    ),
    candidate_users AS (
      SELECT DISTINCT pl.user_id
      FROM post_likes pl
      JOIN user_posts up ON up.post_id = pl.post_id
      WHERE pl.user_id != $1
      LIMIT 5000
    ),
    similarities AS (
      SELECT
        cu.user_id,
        COUNT(DISTINCT CASE WHEN pl2.post_id = any(array(SELECT post_id FROM user_posts)) THEN pl2.post_id END) AS intersection,
        COUNT(DISTINCT pl2.post_id) + (SELECT COUNT(*) FROM user_posts) - 
          COUNT(DISTINCT CASE WHEN pl2.post_id = any(array(SELECT post_id FROM user_posts)) THEN pl2.post_id END) AS union_count
      FROM candidate_users cu
      JOIN post_likes pl2 ON pl2.user_id = cu.user_id
      GROUP BY cu.user_id
    )
    SELECT
      user_id,
      intersection::float / NULLIF(union_count, 0) AS jaccard_similarity
    FROM similarities
    WHERE intersection > 5
    ORDER BY jaccard_similarity DESC
    LIMIT $2
  `, [userId, limit]);
  
  return rows;
}

/**
 * Recommend posts จาก similar users
 */
async function getCollaborativeRecommendations(userId, db, limit = 20) {
  const similarUsers = await findSimilarUsers(userId, db);
  
  if (similarUsers.length === 0) return [];
  
  const similarUserIds = similarUsers.map(u => u.user_id);
  
  // ดึง posts ที่ similar users like แต่ current user ยังไม่เห็น
  const { rows } = await db.query(`
    SELECT DISTINCT
      p.id,
      p.type,
      p.created_at,
      COUNT(pl.user_id) AS similar_user_likes
    FROM posts p
    JOIN post_likes pl ON pl.post_id = p.id
    WHERE pl.user_id = ANY($1)
      AND p.author_id != $2
      AND NOT EXISTS (
        SELECT 1 FROM post_likes ul
        WHERE ul.post_id = p.id AND ul.user_id = $2
      )
      AND NOT EXISTS (
        SELECT 1 FROM feed_seen fs
        WHERE fs.post_id = p.id AND fs.user_id = $2
      )
      AND p.created_at > NOW() - INTERVAL '3 days'
      AND p.is_deleted = FALSE
    GROUP BY p.id, p.type, p.created_at
    ORDER BY similar_user_likes DESC
    LIMIT $3
  `, [similarUserIds, userId, limit]);
  
  return rows;
}

module.exports = { findSimilarUsers, getCollaborativeRecommendations };
```

### Step 727: A/B Testing สำหรับ Feed Algorithm

```javascript
// feed/ab-testing.js
const crypto = require('crypto');

const EXPERIMENTS = {
  'feed-ranking-v2': {
    id: 'feed-ranking-v2',
    buckets: {
      control: 0.5,    // 50% ใช้ algorithm เดิม
      treatment: 0.5,  // 50% ใช้ algorithm ใหม่
    },
    metrics: ['feed_ctr', 'session_duration', 'posts_viewed'],
  },
};

/**
 * กำหนด user ให้อยู่ใน A/B bucket
 * Deterministic: user เดียวกันได้ bucket เดียวกันเสมอ
 */
function assignExperimentBucket(userId, experimentId) {
  const hash = crypto.createHash('md5')
    .update(`${userId}:${experimentId}`)
    .digest('hex');
  
  const hashValue = parseInt(hash.substring(0, 8), 16) / 0xFFFFFFFF;
  
  const experiment = EXPERIMENTS[experimentId];
  if (!experiment) return 'control';
  
  let cumulative = 0;
  for (const [bucket, ratio] of Object.entries(experiment.buckets)) {
    cumulative += ratio;
    if (hashValue < cumulative) return bucket;
  }
  
  return 'control';
}

/**
 * Track experiment metric
 */
async function trackExperimentEvent(userId, experimentId, event, value = 1) {
  const bucket = assignExperimentBucket(userId, experimentId);
  
  // Log to analytics topic
  await publishToKafka('analytics', {
    type: 'experiment_event',
    userId,
    experimentId,
    bucket,
    event,
    value,
    timestamp: new Date().toISOString(),
  });
}

/**
 * Feed route ด้วย A/B testing
 */
async function getPersonalizedFeed(userId, options = {}) {
  const bucket = assignExperimentBucket(userId, 'feed-ranking-v2');
  
  if (bucket === 'treatment') {
    // Algorithm ใหม่: รวม collaborative filtering
    return await getFeedWithCollaborativeFiltering(userId, options);
  } else {
    // Algorithm เดิม: affinity + time decay เท่านั้น
    return await getStandardFeed(userId, options);
  }
}

module.exports = { assignExperimentBucket, trackExperimentEvent, getPersonalizedFeed };
```

### Step 728: Feed Cache Warm-up

```javascript
// feed/cache-warmer.js
// Pre-warm feed cache สำหรับ active users

const { Queue, Worker } = require('bullmq');
const { fanOutToFollowers } = require('./feed-store');

async function warmFeedCacheForNewUser(userId) {
  // ดึง posts จาก users ที่ user ใหม่ follow
  const { rows: followedPosts } = await db.query(`
    SELECT p.*
    FROM posts p
    JOIN follows f ON f.following_id = p.author_id
    WHERE f.follower_id = $1
      AND f.is_active = TRUE
      AND p.created_at > NOW() - INTERVAL '3 days'
      AND p.is_deleted = FALSE
    ORDER BY p.created_at DESC
    LIMIT 200
  `, [userId]);
  
  // Compute scores และใส่ใน Redis
  const pipeline = redis.pipeline();
  
  for (const post of followedPosts) {
    const timeDecay = calculateTimeDecaySync(post.created_at);
    const contentWeight = CONTENT_WEIGHTS[post.type] || 1.0;
    const score = timeDecay * contentWeight;
    
    pipeline.zadd(`feed:${userId}`, score, post.id.toString());
  }
  
  pipeline.expire(`feed:${userId}`, FEED_TTL);
  await pipeline.exec();
  
  console.log(`Warmed feed cache for user ${userId} with ${followedPosts.length} posts`);
}
```

### Step 729: Feed Metrics

```javascript
// feed/metrics.js
const client = require('prom-client');

const feedGenerationDuration = new client.Histogram({
  name: 'feed_generation_duration_seconds',
  help: 'Time to generate feed for a user',
  labelNames: ['source'],
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.5, 1],
});

const feedCacheHitRatio = new client.Gauge({
  name: 'feed_cache_hit_ratio',
  help: 'Ratio of feed requests served from cache',
});

const fanOutDuration = new client.Histogram({
  name: 'feed_fanout_duration_seconds',
  help: 'Time to fan-out post to all followers',
  buckets: [0.1, 0.5, 1, 5, 10, 30],
});

const feedQueueDepth = new client.Gauge({
  name: 'feed_queue_depth',
  help: 'Number of feed update jobs in queue',
  labelNames: ['queue'],
});

module.exports = { feedGenerationDuration, feedCacheHitRatio, fanOutDuration, feedQueueDepth };
```

### Step 730: SOS Alert Feed Priority

```javascript
// feed/sos-feed.js
// SOS alerts ต้องอยู่บน feed เสมอ

async function injectSOSAlertsIntoFeed(userId, existingPostIds) {
  // ดึง active SOS alerts ใน area ของ user
  const userAreaCode = await getUserAreaCode(userId);
  
  const { rows: sosAlerts } = await db.query(`
    SELECT id, type, created_at, area_code, severity
    FROM sos_alerts
    WHERE area_code = $1
      AND status IN ('active', 'responding')
      AND created_at > NOW() - INTERVAL '2 hours'
      AND is_deleted = FALSE
    ORDER BY severity DESC, created_at DESC
    LIMIT 3
  `, [userAreaCode]);
  
  // Prepend SOS alerts ไปยัง feed (top of feed)
  const sosIds = sosAlerts.map(a => a.id.toString());
  const regularIds = existingPostIds.filter(id => !sosIds.includes(id));
  
  return [...sosIds, ...regularIds];
}

module.exports = { injectSOSAlertsIntoFeed };
```

---

## 🔧 Configuration Files

```javascript
// config/feed.js - Feed configuration
module.exports = {
  // Scoring weights
  contentWeights: {
    sos_alert: 5.0,
    video: 1.5,
    photo: 1.0,
    text: 0.8,
  },
  
  // Time decay
  timeDecayLambda: 0.05,
  
  // Cache settings
  feedTTL: 86400 * 7,
  maxFeedSize: 1000,
  
  // Fan-out threshold
  influencerThreshold: 10000,
  
  // A/B testing
  experiments: {
    'feed-ranking-v2': {
      control: 0.5,
      treatment: 0.5,
    },
  },
};
```

---

## 🧪 Testing

```javascript
// tests/feed.test.js
const { calculateTimeDecay, calculateFeedScore } = require('../feed/scoring');

test('SOS alerts get highest priority score', async () => {
  const sosPost = {
    id: '1',
    type: 'sos_alert',
    author_id: 'author-1',
    created_at: new Date().toISOString(),
    is_trending: false,
    area_code: 'BKK',
    likes_count: 0,
    comments_count: 0,
    shares_count: 0,
  };
  
  const photoPost = { ...sosPost, type: 'photo' };
  
  const sosScore = await calculateFeedScore('user-1', sosPost, mockDb);
  const photoScore = await calculateFeedScore('user-1', photoPost, mockDb);
  
  expect(sosScore).toBeGreaterThan(photoScore);
});

test('Older posts have lower time decay', () => {
  const newPost = new Date().toISOString();
  const oldPost = new Date(Date.now() - 24 * 3600000).toISOString();
  
  const newDecay = calculateTimeDecay(newPost);
  const oldDecay = calculateTimeDecay(oldPost);
  
  expect(newDecay).toBeGreaterThan(oldDecay);
  expect(newDecay).toBeCloseTo(1.0, 1);
  expect(oldDecay).toBeLessThan(0.5);
});
```

```bash
# รัน tests
npm test -- --testPathPattern=feed

# Monitor feed queue
npx bull-board
# เปิด http://localhost:3000/admin/queues
```

---

## ❌ Common Errors & Solutions

### Error 1: Fan-out ช้ามากสำหรับ influencer

```javascript
// ปัญหา: User มี 100k followers, fan-out ใช้เวลา > 10 วินาที
// แก้ไข: ใช้ fan-out on read สำหรับ influencers

async function smartFanOut(post, followerCount) {
  if (followerCount > 10000) {
    // Influencer: ไม่ทำ fan-out, ให้ read-time pull แทน
    await redis.sadd(`influencer-posts:${post.author_id}`, post.id);
    await redis.expire(`influencer-posts:${post.author_id}`, FEED_TTL);
  } else {
    await fanOutToFollowers(post, db);
  }
}
```

### Error 2: Feed ว่างสำหรับ new user

```javascript
// ปัญหา: User ใหม่ยังไม่ follow ใคร feed เลยว่าง
// แก้ไข: ใช้ fallback feed จาก trending posts

async function getFeedWithFallback(userId) {
  const feed = await getFeed(userId);
  
  if (feed.length < 10) {
    const trending = await getTrendingPosts(20 - feed.length);
    return [...feed, ...trending.map(p => p.id)];
  }
  
  return feed;
}
```

---

## ✅ Checklist

- [ ] Implement feed scoring algorithm (affinity × weight × time decay)
- [ ] ใช้ Redis sorted set สำหรับ feed storage
- [ ] Fan-out on write สำหรับ normal users
- [ ] Fan-out on read สำหรับ influencers (>10k followers)
- [ ] SOS alerts inject ไปที่ top of feed เสมอ
- [ ] Background job สำหรับ feed score recomputation
- [ ] A/B testing framework สำหรับ algorithm
- [ ] Feed cache warm-up สำหรับ new users
- [ ] Monitor fan-out duration ด้วย Prometheus
- [ ] ทดสอบ feed correctness ด้วย unit tests

---

## 🔗 References

- [EdgeRank Algorithm](https://en.wikipedia.org/wiki/EdgeRank)
- [Instagram Explore Algorithm](https://ai.facebook.com/blog/powered-by-ai-instagrams-explore-recommender-system/)
- [Redis Sorted Sets](https://redis.io/docs/data-types/sorted-sets/)
- [BullMQ Documentation](https://docs.bullmq.io/)
- [Collaborative Filtering](https://developers.google.com/machine-learning/recommendation/collaborative/basics)

---

*Part 073 | Road to 1,000,000 Users/Day | chuaikan.com*
