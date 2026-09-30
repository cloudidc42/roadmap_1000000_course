# Part 029: Search Service
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 281-290
> **เวลาโดยประมาณ:** 6 ชั่วโมง
> **Prerequisites:** Part 026 (Feed Service), Part 028 (Location Service), Part 025 (Redis Cache)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ติดตั้ง MeiliSearch เป็น Search Engine หลัก
- Configure indexes สำหรับ Thai language
- Search API: full-text, faceted search, geo-search
- Autocomplete ด้วย Redis Sorted Sets (prefix search)
- Trending Topics ด้วย Sliding Window Counter
- Hashtag Search และ Trending Hashtags
- People Search (ชื่อ, username, bio)
- Location-aware search "ค้นหาใกล้ฉัน"
- Search Analytics: popular queries, zero-result queries
- Sync service: PostgreSQL → MeiliSearch ด้วย Change Data Capture
- Search Relevance Tuning

---

## 📖 ทฤษฎีและแนวคิด

### Search Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     SEARCH ARCHITECTURE                         │
│                                                                 │
│  User Types                                                     │
│  ─────────                                                      │
│  "น้ำท่วม"                                                       │
│       │                                                         │
│       ▼                                                         │
│  ┌────────────┐    ┌──────────────┐    ┌─────────────────┐     │
│  │ Autocomplete│   │  Full-text   │    │  Geo Search     │     │
│  │ (Redis)    │   │  (MeiliSearch)│   │  (PostGIS)      │     │
│  └─────┬──────┘   └──────┬───────┘   └────────┬────────┘     │
│        │                 │                      │              │
│        └─────────────────┼──────────────────────┘              │
│                          ▼                                      │
│                   ┌─────────────┐                              │
│                   │  Result     │                              │
│                   │  Merger &   │                              │
│                   │  Ranker     │                              │
│                   └──────┬──────┘                              │
│                          │                                      │
│  Data Sources:           ▼                                      │
│  PostgreSQL ──CDC──► MeiliSearch Index                          │
│    posts               posts                                    │
│    users               users                                    │
│    hashtags            hashtags                                 │
│    sos_alerts          sos_alerts                               │
└─────────────────────────────────────────────────────────────────┘
```

### MeiliSearch vs Elasticsearch

```
Feature          MeiliSearch        Elasticsearch
──────────────── ──────────────── ──────────────────
Setup            Simple 1 binary   Complex cluster
Thai Support     Built-in ICU      Requires plugin
Typo Tolerance   Built-in          Needs config
Relevance Tuning  GUI              Manual
Memory Usage     Light (~500MB)    Heavy (~2GB+)
Scale            Up to 100M docs   Unlimited
Speed            Very fast         Fast
License          MIT               Apache/Commercial

→ chuaikan.com เลือก MeiliSearch:
  - ง่ายต่อการ setup
  - Built-in typo tolerance (สำคัญมากสำหรับ Thai)
  - Lightweight (เหมาะกับ startup)
  - Scale ได้ถึง 100M documents
```

### Sliding Window Counter สำหรับ Trending

```
Sliding Window: ช่วงเวลา 1 ชั่วโมงล่าสุด

เวลา:    12:00  12:15  12:30  12:45  13:00  13:15
#flood:    20     35     40     45     30     20

Window at 13:15 → sum(12:15 to 13:15) = 35+40+45+30+20 = 170

ใช้ Redis Sorted Set:
  Key: trending:hashtags
  Score: count ใน 1 ชั่วโมงล่าสุด (updated periodically)

ทุก 1 นาที:
  ZINCRBY trending:hashtags 1 "#flood"
  
ทุก 1 ชั่วโมง:
  Recalculate scores จาก actual counts
```

---

## ⚙️ Environment Setup

### Step 281: Install MeiliSearch

```bash
# ติดตั้ง MeiliSearch บน Ubuntu 24.04
curl -L https://install.meilisearch.com | sh

# Move to /usr/local/bin
sudo mv ./meilisearch /usr/local/bin/

# ตรวจสอบ
meilisearch --version

# สร้าง systemd service
sudo useradd -r -s /bin/false meilisearch
sudo mkdir -p /var/lib/meilisearch /etc/meilisearch

sudo tee /etc/meilisearch/config.toml << 'EOF'
env = "production"
master_key = "your-super-secret-master-key-32chars"
db_path = "/var/lib/meilisearch/data"
http_addr = "127.0.0.1:7700"
log_level = "WARN"

[indexer_options]
max_indexing_memory = "1 GiB"
max_indexing_threads = 4
EOF

sudo tee /etc/systemd/system/meilisearch.service << 'EOF'
[Unit]
Description=MeiliSearch
After=network.target

[Service]
User=meilisearch
Group=meilisearch
ExecStart=/usr/local/bin/meilisearch --config-file-path /etc/meilisearch/config.toml
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable meilisearch
sudo systemctl start meilisearch
sudo systemctl status meilisearch

# ทดสอบ
curl http://localhost:7700/health
# → {"status":"available"}
```

### Step 282: Node.js Dependencies

```bash
cd /home/user/chuaikan/services/search-service

npm install \
  meilisearch@0.40.0 \
  ioredis@5.3.2 \
  pg@8.11.3 \
  express@4.18.2 \
  zod@3.22.4 \
  winston@3.11.0 \
  axios@1.6.2

npm install -D \
  typescript@5.3.2 \
  @types/node@20.10.0 \
  @types/express@4.17.21 \
  ts-node@10.9.2
```

---

## 🛠️ Step-by-Step Implementation

### Step 283: MeiliSearch Index Configuration

```typescript
// src/services/search-index.service.ts

import { MeiliSearch, Index } from 'meilisearch';

const client = new MeiliSearch({
  host: process.env.MEILISEARCH_URL ?? 'http://localhost:7700',
  apiKey: process.env.MEILISEARCH_KEY ?? 'your-master-key',
});

// ─── Posts Index ────────────────────────────────────────────────
export async function setupPostsIndex(): Promise<Index> {
  const index = client.index('posts');

  // Primary key
  await index.updateSettings({
    primaryKey: 'id',

    // Fields ที่ค้นหาได้
    searchableAttributes: [
      'content',
      'authorName',
      'hashtags',
      'locationName',
    ],

    // Fields สำหรับ filter/facet
    filterableAttributes: [
      'authorId',
      'type',
      'hasMedia',
      'provinceCode',
      'isSOS',
      'severity',
      'createdAt',
      'engagementScore',
      '_geo',
    ],

    // Fields สำหรับ sort
    sortableAttributes: [
      'createdAt',
      'engagementScore',
      'likesCount',
      '_geo',
    ],

    // Ranking Rules (ลำดับความสำคัญ)
    rankingRules: [
      'words',        // จำนวน match
      'typo',         // น้อย typo = ดีกว่า
      'proximity',    // คำใกล้กัน = ดีกว่า
      'attribute',    // field ที่อยู่ข้างบน = ดีกว่า
      'sort',         // custom sort
      'exactness',    // exact match = ดีกว่า
      'engagementScore:desc', // custom: engagement สูง = ดีกว่า
    ],

    // Typo tolerance สำหรับ Thai
    typoTolerance: {
      enabled: true,
      minWordSizeForTypos: {
        oneTypo: 4,
        twoTypos: 8,
      },
    },

    // Pagination
    pagination: {
      maxTotalHits: 10000,
    },

    // Distinct field (ไม่ซ้ำ author ถ้าจะทำ)
    // distinctAttribute: 'authorId',
  });

  return index;
}

// ─── Users Index ────────────────────────────────────────────────
export async function setupUsersIndex(): Promise<Index> {
  const index = client.index('users');

  await index.updateSettings({
    primaryKey: 'id',
    searchableAttributes: [
      'displayName',
      'username',
      'bio',
    ],
    filterableAttributes: [
      'provinceCode',
      'isVerified',
      'followerCount',
      '_geo',
    ],
    sortableAttributes: [
      'followerCount',
      'createdAt',
    ],
    rankingRules: [
      'words',
      'typo',
      'attribute',
      'sort',
      'exactness',
      'followerCount:desc',
    ],
  });

  return index;
}

// ─── Hashtags Index ──────────────────────────────────────────────
export async function setupHashtagsIndex(): Promise<Index> {
  const index = client.index('hashtags');

  await index.updateSettings({
    primaryKey: 'tag',
    searchableAttributes: ['tag', 'relatedTags'],
    filterableAttributes: ['postCount', 'trendScore'],
    sortableAttributes: ['postCount', 'trendScore', 'lastUsedAt'],
    rankingRules: [
      'words',
      'typo',
      'trendScore:desc',
      'postCount:desc',
    ],
  });

  return index;
}

// ─── SOS Alerts Index ────────────────────────────────────────────
export async function setupSOSIndex(): Promise<Index> {
  const index = client.index('sos_alerts');

  await index.updateSettings({
    primaryKey: 'id',
    searchableAttributes: ['title', 'description', 'locationName'],
    filterableAttributes: [
      'type',
      'severity',
      'status',
      'provinceCode',
      '_geo',
    ],
    sortableAttributes: ['createdAt', '_geo'],
    rankingRules: [
      'words',
      'sort',
      'severity:asc',  // CRITICAL = ขึ้นก่อน (alphabetically C < D < I < W)
      'createdAt:desc',
    ],
  });

  return index;
}
```

### Step 284: Search Service

```typescript
// src/services/search.service.ts

import { MeiliSearch, SearchParams, SearchResponse } from 'meilisearch';
import Redis from 'ioredis';
import { Pool } from 'pg';

export interface SearchOptions {
  query: string;
  type?: 'all' | 'posts' | 'users' | 'hashtags' | 'sos';
  filters?: string;
  page?: number;
  limit?: number;
  lat?: number;
  lng?: number;
  radiusKm?: number;
  provinceCode?: string;
  sortBy?: string;
  userId?: string; // สำหรับ personalization
}

export interface SearchResult {
  hits: any[];
  totalHits: number;
  page: number;
  totalPages: number;
  processingTimeMs: number;
  query: string;
}

export class SearchService {
  private client: MeiliSearch;
  private readonly ANALYTICS_TTL = 86400 * 30; // 30 days

  constructor(
    private redis: Redis,
    private db: Pool
  ) {
    this.client = new MeiliSearch({
      host: process.env.MEILISEARCH_URL ?? 'http://localhost:7700',
      apiKey: process.env.MEILISEARCH_KEY,
    });
  }

  // Full-text search across all content
  async search(options: SearchOptions): Promise<SearchResult> {
    const {
      query,
      type = 'all',
      filters,
      page = 1,
      limit = 20,
      lat,
      lng,
      radiusKm,
      provinceCode,
      userId,
    } = options;

    const offset = (page - 1) * limit;

    // Track search query สำหรับ analytics
    await this.trackSearchQuery(query, userId);

    const searchParams: SearchParams = {
      offset,
      limit,
      attributesToHighlight: ['content', 'displayName', 'title', 'bio'],
      highlightPreTag: '<mark>',
      highlightPostTag: '</mark>',
    };

    // Build filter
    const filterParts: string[] = [];
    if (filters) filterParts.push(filters);
    if (provinceCode) filterParts.push(`provinceCode = "${provinceCode}"`);

    // Geo filter
    if (lat !== undefined && lng !== undefined && radiusKm) {
      searchParams.sort = [`_geoPoint(${lat}, ${lng}):asc`];
      filterParts.push(
        `_geoRadius(${lat}, ${lng}, ${radiusKm * 1000})`
      );
    }

    if (filterParts.length > 0) {
      searchParams.filter = filterParts.join(' AND ');
    }

    let result: SearchResponse;

    if (type === 'all') {
      result = await this.multiSearch(query, searchParams);
    } else {
      const indexName = this.getIndexName(type);
      const index = this.client.index(indexName);
      result = await index.search(query, searchParams);
    }

    // Track zero-result queries
    if (result.hits.length === 0) {
      await this.trackZeroResult(query);
    }

    return {
      hits: result.hits,
      totalHits: result.totalHits ?? result.estimatedTotalHits ?? 0,
      page,
      totalPages: Math.ceil(
        (result.totalHits ?? result.estimatedTotalHits ?? 0) / limit
      ),
      processingTimeMs: result.processingTimeMs,
      query,
    };
  }

  // Multi-index search
  private async multiSearch(
    query: string,
    params: SearchParams
  ): Promise<any> {
    const results = await this.client.multiSearch({
      queries: [
        { indexUid: 'posts',      q: query, limit: 10, ...params },
        { indexUid: 'users',      q: query, limit: 5,  ...params },
        { indexUid: 'hashtags',   q: query, limit: 5,  ...params },
        { indexUid: 'sos_alerts', q: query, limit: 3,  ...params },
      ],
    });

    // Merge results
    const allHits: any[] = [];
    for (const r of results.results) {
      allHits.push(
        ...r.hits.map((h: any) => ({
          ...h,
          _type: r.indexUid,
        }))
      );
    }

    return {
      hits: allHits,
      totalHits: results.results.reduce(
        (sum, r) => sum + (r.estimatedTotalHits ?? 0), 0
      ),
      processingTimeMs: Math.max(
        ...results.results.map(r => r.processingTimeMs)
      ),
    };
  }

  // Geo-search: "ค้นหาใกล้ฉัน"
  async searchNearby(
    query: string,
    lat: number,
    lng: number,
    radiusKm: number = 10
  ): Promise<SearchResult> {
    return this.search({
      query,
      type: 'posts',
      lat,
      lng,
      radiusKm,
      sortBy: `_geoPoint(${lat}, ${lng}):asc`,
    });
  }

  // Track search analytics
  private async trackSearchQuery(
    query: string,
    userId?: string
  ): Promise<void> {
    if (!query.trim()) return;

    const normalized = query.toLowerCase().trim();

    // Increment popular queries counter
    await this.redis.zincrby('search:popular', 1, normalized);

    // Time-based tracking (hourly)
    const hour = Math.floor(Date.now() / 3600000);
    await this.redis.zincrby(`search:hourly:${hour}`, 1, normalized);
    await this.redis.expire(`search:hourly:${hour}`, 7200); // 2 hours

    // Log to DB async (ไม่รอ)
    this.db.query(
      'INSERT INTO search_analytics (query, user_id) VALUES ($1, $2)',
      [normalized, userId ?? null]
    ).catch(() => {});
  }

  private async trackZeroResult(query: string): Promise<void> {
    await this.redis.zincrby('search:zero-results', 1, query.toLowerCase());
  }

  private getIndexName(type: string): string {
    const map: Record<string, string> = {
      posts: 'posts',
      users: 'users',
      hashtags: 'hashtags',
      sos: 'sos_alerts',
    };
    return map[type] ?? 'posts';
  }
}
```

### Step 285: Autocomplete with Redis

```typescript
// src/services/autocomplete.service.ts

import Redis from 'ioredis';

export class AutocompleteService {
  private readonly PREFIX_KEY = 'autocomplete';
  private readonly MAX_SUGGESTIONS = 10;

  constructor(private redis: Redis) {}

  // เพิ่ม term ใน autocomplete index
  async addTerm(term: string, score: number = 0): Promise<void> {
    const normalized = term.toLowerCase().trim();
    if (!normalized || normalized.length < 2) return;

    // เพิ่ม prefixes ทั้งหมด
    // "flood" → "f", "fl", "flo", "floo", "flood"
    for (let i = 1; i <= normalized.length; i++) {
      const prefix = normalized.substring(0, i);
      await this.redis.zadd(
        `${this.PREFIX_KEY}:${prefix}`,
        score,
        normalized
      );
    }

    // ตั้ง expiry
    for (let i = 1; i <= normalized.length; i++) {
      const prefix = normalized.substring(0, i);
      await this.redis.expire(`${this.PREFIX_KEY}:${prefix}`, 86400);
    }
  }

  // ดึง suggestions จาก prefix
  async getSuggestions(
    prefix: string,
    limit: number = 5
  ): Promise<string[]> {
    if (!prefix || prefix.length < 1) return [];

    const normalized = prefix.toLowerCase().trim();
    const key = `${this.PREFIX_KEY}:${normalized}`;

    // ZRANGEBYSCORE by score DESC
    const results = await this.redis.zrevrange(key, 0, limit - 1);
    return results;
  }

  // Batch add terms (เช่น จาก hashtags ยอดนิยม)
  async batchAddTerms(
    terms: Array<{ term: string; score: number }>
  ): Promise<void> {
    const pipeline = this.redis.pipeline();

    for (const { term, score } of terms) {
      const normalized = term.toLowerCase().trim();
      if (!normalized || normalized.length < 2) continue;

      for (let i = 1; i <= Math.min(normalized.length, 20); i++) {
        const prefix = normalized.substring(0, i);
        pipeline.zadd(
          `${this.PREFIX_KEY}:${prefix}`,
          score,
          normalized
        );
      }
    }

    await pipeline.exec();
  }

  // Autocomplete สำหรับ hashtags
  async getHashtagSuggestions(
    prefix: string,
    limit: number = 8
  ): Promise<Array<{ tag: string; count: number }>> {
    const suggestions = await this.getSuggestions(
      prefix.startsWith('#') ? prefix.slice(1) : prefix,
      limit
    );

    // ดึง count จาก Redis
    const counts = await Promise.all(
      suggestions.map(tag =>
        this.redis.zscore('trending:hashtags', `#${tag}`)
      )
    );

    return suggestions.map((tag, i) => ({
      tag: `#${tag}`,
      count: parseInt(counts[i] ?? '0'),
    }));
  }
}
```

### Step 286: Trending Topics Service

```typescript
// src/services/trending.service.ts

import Redis from 'ioredis';
import { Pool } from 'pg';

export interface TrendingTopic {
  topic: string;
  count: number;
  changePercent: number;  // เทียบกับชั่วโมงที่แล้ว
  isNew: boolean;         // เพิ่งขึ้น trending
}

export class TrendingService {
  private readonly WINDOW_HOURS = 1;
  private readonly TOP_N = 20;
  private readonly TREND_KEY = 'trending:hashtags';

  constructor(
    private redis: Redis,
    private db: Pool
  ) {}

  // Track hashtag usage (เรียกทุกครั้งที่มีโพสต์ใหม่)
  async trackHashtag(hashtag: string): Promise<void> {
    const normalized = hashtag.toLowerCase().trim();
    if (!normalized.startsWith('#')) return;

    // Global trending
    await this.redis.zincrby(this.TREND_KEY, 1, normalized);

    // Hourly bucket
    const hour = Math.floor(Date.now() / 3600000);
    await this.redis.zincrby(`trending:hourly:${hour}`, 1, normalized);
    await this.redis.expire(`trending:hourly:${hour}`, 7200);

    // Update autocomplete score
    await this.redis.zincrby(
      `autocomplete:${normalized.slice(1, 4)}`, // prefix แรก 3 chars
      1,
      normalized.slice(1) // ไม่มี #
    );
  }

  // ดึง trending topics ตอนนี้
  async getTrending(
    limit: number = 10,
    provinceCode?: string
  ): Promise<TrendingTopic[]> {
    let key = this.TREND_KEY;

    // Province-specific trending
    if (provinceCode) {
      key = `trending:province:${provinceCode}`;
    }

    const currentHour = Math.floor(Date.now() / 3600000);
    const prevHour = currentHour - 1;

    // ดึง current hour counts
    const currentTrends = await this.redis.zrevrangebyscore(
      `trending:hourly:${currentHour}`,
      '+inf',
      '-inf',
      'WITHSCORES',
      'LIMIT', 0, limit * 2
    );

    // ดึง previous hour counts
    const prevCounts = new Map<string, number>();
    const prevTrends = await this.redis.zrevrangebyscore(
      `trending:hourly:${prevHour}`,
      '+inf',
      '-inf',
      'WITHSCORES'
    );

    for (let i = 0; i < prevTrends.length - 1; i += 2) {
      prevCounts.set(prevTrends[i], parseFloat(prevTrends[i + 1]));
    }

    // Build trending topics
    const topics: TrendingTopic[] = [];
    for (let i = 0; i < currentTrends.length - 1; i += 2) {
      const topic = currentTrends[i];
      const count = parseFloat(currentTrends[i + 1]);
      const prevCount = prevCounts.get(topic) ?? 0;

      const changePercent = prevCount > 0
        ? ((count - prevCount) / prevCount) * 100
        : 100;

      topics.push({
        topic,
        count: Math.round(count),
        changePercent: Math.round(changePercent),
        isNew: prevCount === 0 && count > 5,
      });
    }

    return topics.slice(0, limit);
  }

  // Recalculate trending scores (รัน hourly)
  async recalculateTrending(): Promise<void> {
    // รวม counts จาก 1 ชั่วโมงล่าสุด
    const currentHour = Math.floor(Date.now() / 3600000);
    const prevHour = currentHour - 1;

    // Merge hourly buckets
    await this.redis.zunionstore(
      this.TREND_KEY,
      2,
      `trending:hourly:${currentHour}`,
      `trending:hourly:${prevHour}`
    );

    // Trim to top 1000
    await this.redis.zremrangebyrank(this.TREND_KEY, 0, -1001);

    console.log('Trending recalculated');
  }

  // Trending hashtags ใน province
  async getProvinceTrending(
    provinceCode: string,
    limit: number = 10
  ): Promise<TrendingTopic[]> {
    const key = `trending:province:${provinceCode}`;
    const results = await this.redis.zrevrangebyscore(
      key,
      '+inf',
      '-inf',
      'WITHSCORES',
      'LIMIT', 0, limit
    );

    const topics: TrendingTopic[] = [];
    for (let i = 0; i < results.length - 1; i += 2) {
      topics.push({
        topic: results[i],
        count: Math.round(parseFloat(results[i + 1])),
        changePercent: 0,
        isNew: false,
      });
    }

    return topics;
  }
}
```

### Step 287: CDC Sync Service (PostgreSQL → MeiliSearch)

```typescript
// src/services/cdc-sync.service.ts
// Change Data Capture: sync PostgreSQL changes to MeiliSearch

import { Pool } from 'pg';
import { MeiliSearch } from 'meilisearch';
import { EventEmitter } from 'events';

export class CDCSyncService extends EventEmitter {
  private client: MeiliSearch;
  private isRunning = false;
  private lastSyncedAt: Date;

  constructor(
    private db: Pool
  ) {
    super();
    this.client = new MeiliSearch({
      host: process.env.MEILISEARCH_URL ?? 'http://localhost:7700',
      apiKey: process.env.MEILISEARCH_KEY,
    });
    this.lastSyncedAt = new Date(Date.now() - 60000); // เริ่มที่ 1 นาทีที่แล้ว
  }

  start(): void {
    this.isRunning = true;
    this.syncLoop();
  }

  stop(): void {
    this.isRunning = false;
  }

  // Polling-based CDC (ทำทุก 5 วินาที)
  private async syncLoop(): Promise<void> {
    while (this.isRunning) {
      try {
        await this.syncNewPosts();
        await this.syncUpdatedUsers();
        await this.syncHashtags();
        await this.syncSOSAlerts();

        this.lastSyncedAt = new Date();
      } catch (error) {
        console.error('CDC sync error:', error);
      }

      await new Promise(resolve => setTimeout(resolve, 5000));
    }
  }

  // Sync new/updated posts
  private async syncNewPosts(): Promise<void> {
    const result = await this.db.query(`
      SELECT
        p.id,
        p.content,
        p.author_id AS "authorId",
        u.display_name AS "authorName",
        p.hashtags,
        p.type,
        p.has_media AS "hasMedia",
        p.province_code AS "provinceCode",
        p.lat,
        p.lng,
        p.created_at AS "createdAt",
        p.updated_at AS "updatedAt",
        COALESCE(pes.raw_score, 0) AS "engagementScore",
        COALESCE(pes.likes_count, 0) AS "likesCount"
      FROM posts p
      JOIN users u ON u.id = p.author_id
      LEFT JOIN post_engagement_scores pes ON pes.post_id = p.id
      WHERE p.updated_at > $1
        AND p.status = 'published'
      LIMIT 1000
    `, [this.lastSyncedAt]);

    if (result.rows.length === 0) return;

    // Transform สำหรับ geo-search
    const docs = result.rows.map((row: any) => ({
      ...row,
      createdAt: new Date(row.createdAt).getTime(),
      // _geo field สำหรับ MeiliSearch geo filter
      _geo: row.lat && row.lng
        ? { lat: parseFloat(row.lat), lng: parseFloat(row.lng) }
        : undefined,
    }));

    // Batch add to MeiliSearch
    const task = await this.client.index('posts').addDocuments(docs, {
      primaryKey: 'id',
    });

    console.log(
      `Synced ${docs.length} posts to MeiliSearch (task: ${task.taskUid})`
    );
  }

  // Sync users
  private async syncUpdatedUsers(): Promise<void> {
    const result = await this.db.query(`
      SELECT
        u.id,
        u.display_name AS "displayName",
        u.username,
        u.bio,
        u.avatar_url AS "avatarUrl",
        u.is_verified AS "isVerified",
        ul.province_code AS "provinceCode",
        ul.lat,
        ul.lng,
        us.follower_count AS "followerCount",
        u.created_at AS "createdAt"
      FROM users u
      LEFT JOIN user_locations ul ON ul.user_id = u.id
      LEFT JOIN user_stats us ON us.user_id = u.id
      WHERE u.updated_at > $1
        AND u.status = 'active'
      LIMIT 500
    `, [this.lastSyncedAt]);

    if (result.rows.length === 0) return;

    const docs = result.rows.map((row: any) => ({
      ...row,
      _geo: row.lat && row.lng
        ? { lat: parseFloat(row.lat), lng: parseFloat(row.lng) }
        : undefined,
    }));

    await this.client.index('users').addDocuments(docs);
  }

  // Sync hashtags
  private async syncHashtags(): Promise<void> {
    const result = await this.db.query(`
      SELECT
        tag,
        post_count AS "postCount",
        trend_score AS "trendScore",
        last_used_at AS "lastUsedAt"
      FROM hashtags
      WHERE updated_at > $1
      LIMIT 500
    `, [this.lastSyncedAt]);

    if (result.rows.length === 0) return;
    await this.client.index('hashtags').addDocuments(result.rows);
  }

  // Sync SOS alerts
  private async syncSOSAlerts(): Promise<void> {
    const result = await this.db.query(`
      SELECT
        id,
        alert_id AS "alertId",
        type,
        severity,
        title,
        description,
        lat,
        lng,
        province_code AS "provinceCode",
        status,
        created_at AS "createdAt"
      FROM sos_alerts
      WHERE updated_at > $1
        AND status IN ('active', 'official', 'verified')
      LIMIT 200
    `, [this.lastSyncedAt]);

    if (result.rows.length === 0) return;

    const docs = result.rows.map((row: any) => ({
      ...row,
      _geo: row.lat && row.lng
        ? { lat: parseFloat(row.lat), lng: parseFloat(row.lng) }
        : undefined,
    }));

    await this.client.index('sos_alerts').addDocuments(docs);
  }

  // Full re-index (รันตอน maintenance)
  async fullReindex(indexName: string): Promise<void> {
    console.log(`Starting full re-index of ${indexName}...`);

    const index = this.client.index(indexName);
    await index.deleteAllDocuments();

    let offset = 0;
    const batchSize = 1000;

    while (true) {
      const result = await this.fetchBatch(indexName, offset, batchSize);
      if (result.length === 0) break;

      await index.addDocuments(result);
      offset += batchSize;
      console.log(`Re-indexed ${offset} documents in ${indexName}`);
    }

    console.log(`Full re-index of ${indexName} complete`);
  }

  private async fetchBatch(
    indexName: string,
    offset: number,
    limit: number
  ): Promise<any[]> {
    // ดึง batch ตาม index name
    const queries: Record<string, string> = {
      posts: `
        SELECT p.id, p.content, p.author_id AS "authorId",
               u.display_name AS "authorName", p.hashtags,
               p.lat, p.lng, p.created_at AS "createdAt"
        FROM posts p
        JOIN users u ON u.id = p.author_id
        WHERE p.status = 'published'
        ORDER BY p.created_at DESC
        LIMIT $1 OFFSET $2
      `,
      users: `
        SELECT id, display_name AS "displayName", username, bio
        FROM users WHERE status = 'active'
        ORDER BY created_at DESC
        LIMIT $1 OFFSET $2
      `,
    };

    const query = queries[indexName];
    if (!query) return [];

    const result = await this.db.query(query, [limit, offset]);
    return result.rows;
  }
}
```

### Step 288: Search API Routes

```typescript
// src/routes/search.routes.ts

import { Router, Request, Response } from 'express';
import { SearchService } from '../services/search.service';
import { AutocompleteService } from '../services/autocomplete.service';
import { TrendingService } from '../services/trending.service';

export function createSearchRouter(
  searchService: SearchService,
  autocompleteService: AutocompleteService,
  trendingService: TrendingService
): Router {
  const router = Router();

  // GET /api/v1/search?q=...&type=...
  router.get('/', async (req: Request, res: Response) => {
    const query = req.query.q as string;
    const type = (req.query.type as string) || 'all';
    const page = parseInt(req.query.page as string) || 1;
    const limit = parseInt(req.query.limit as string) || 20;
    const lat = req.query.lat ? parseFloat(req.query.lat as string) : undefined;
    const lng = req.query.lng ? parseFloat(req.query.lng as string) : undefined;
    const radius = req.query.radius ? parseFloat(req.query.radius as string) : undefined;
    const province = req.query.province as string | undefined;

    if (!query || query.trim().length === 0) {
      return res.status(400).json({ error: 'Query required' });
    }

    const result = await searchService.search({
      query: query.trim(),
      type: type as any,
      page,
      limit,
      lat,
      lng,
      radiusKm: radius,
      provinceCode: province,
      userId: req.user?.id,
    });

    res.json(result);
  });

  // GET /api/v1/search/autocomplete?q=...
  router.get('/autocomplete', async (req: Request, res: Response) => {
    const prefix = req.query.q as string;
    const type = (req.query.type as string) || 'all';

    if (!prefix || prefix.length < 1) {
      return res.json({ suggestions: [] });
    }

    let suggestions: any[];

    if (type === 'hashtag') {
      suggestions = await autocompleteService.getHashtagSuggestions(
        prefix, 8
      );
    } else {
      const terms = await autocompleteService.getSuggestions(prefix, 8);
      suggestions = terms.map(t => ({ text: t }));
    }

    res.json({ suggestions, query: prefix });
  });

  // GET /api/v1/search/trending
  router.get('/trending', async (req: Request, res: Response) => {
    const province = req.query.province as string | undefined;
    const limit = parseInt(req.query.limit as string) || 10;

    const trending = province
      ? await trendingService.getProvinceTrending(province, limit)
      : await trendingService.getTrending(limit);

    res.json({ trending, updatedAt: new Date().toISOString() });
  });

  // GET /api/v1/search/hashtag/:tag
  router.get('/hashtag/:tag', async (req: Request, res: Response) => {
    const tag = req.params.tag.startsWith('#')
      ? req.params.tag
      : `#${req.params.tag}`;
    const page = parseInt(req.query.page as string) || 1;

    const result = await searchService.search({
      query: tag,
      type: 'posts',
      page,
      limit: 20,
      filters: `hashtags = "${tag}"`,
    });

    res.json(result);
  });

  // GET /api/v1/search/people?q=...
  router.get('/people', async (req: Request, res: Response) => {
    const query = req.query.q as string;
    const page = parseInt(req.query.page as string) || 1;

    if (!query) return res.status(400).json({ error: 'Query required' });

    const result = await searchService.search({
      query,
      type: 'users',
      page,
      limit: 20,
    });

    res.json(result);
  });

  // GET /api/v1/search/nearby?q=...&lat=...&lng=...
  router.get('/nearby', async (req: Request, res: Response) => {
    const query = req.query.q as string || '';
    const lat = parseFloat(req.query.lat as string);
    const lng = parseFloat(req.query.lng as string);
    const radius = parseFloat(req.query.radius as string) || 10;

    if (isNaN(lat) || isNaN(lng)) {
      return res.status(400).json({ error: 'Invalid coordinates' });
    }

    const result = await searchService.searchNearby(query, lat, lng, radius);
    res.json(result);
  });

  // GET /api/v1/search/analytics (admin only)
  router.get('/analytics', async (req: Request, res: Response) => {
    if (!req.user?.roles.includes('admin')) {
      return res.status(403).json({ error: 'Forbidden' });
    }

    const [popular, zeroResults] = await Promise.all([
      req.redis.zrevrangebyscore(
        'search:popular', '+inf', '-inf', 'WITHSCORES', 'LIMIT', 0, 20
      ),
      req.redis.zrevrangebyscore(
        'search:zero-results', '+inf', '-inf', 'WITHSCORES', 'LIMIT', 0, 20
      ),
    ]);

    // Parse results
    const parseScored = (arr: string[]) => {
      const result: Array<{ query: string; count: number }> = [];
      for (let i = 0; i < arr.length - 1; i += 2) {
        result.push({ query: arr[i], count: parseInt(arr[i + 1]) });
      }
      return result;
    };

    res.json({
      popularQueries: parseScored(popular),
      zeroResultQueries: parseScored(zeroResults),
    });
  });

  return router;
}
```

### Step 289: Search Relevance Tuning

```typescript
// src/scripts/tune-search-relevance.ts
// รัน script นี้หลังจาก collect search data

import { MeiliSearch } from 'meilisearch';

const client = new MeiliSearch({
  host: process.env.MEILISEARCH_URL!,
  apiKey: process.env.MEILISEARCH_KEY!,
});

async function tunePostsIndex(): Promise<void> {
  const index = client.index('posts');

  // Boost โพสต์ที่มี engagement สูง
  // และ recency bonus สำหรับโพสต์ใหม่ๆ
  await index.updateSettings({
    rankingRules: [
      'words',
      'typo',
      'proximity',
      'attribute',
      'sort',
      'exactness',
    ],

    // Attributes ที่อยู่ข้างบนมีน้ำหนักมากกว่า
    searchableAttributes: [
      'hashtags',      // weight: สูงสุด
      'title',
      'content',
      'authorName',
      'locationName',  // weight: ต่ำสุด
    ],

    // Distinct: ไม่แสดงโพสต์ซ้ำจาก author เดียวกัน
    // (uncomment ถ้าต้องการ)
    // distinctAttribute: 'authorId',
  });

  // Update stop words สำหรับภาษาไทย (คำที่ไม่ค้นหา)
  await index.updateStopWords([
    'และ', 'หรือ', 'ของ', 'ใน', 'บน', 'ที่', 'ก็', 'แต่',
    'is', 'are', 'the', 'a', 'an', 'in', 'on', 'at',
  ]);

  // Synonyms สำหรับคำที่เกี่ยวข้อง
  await index.updateSynonyms({
    'น้ำท่วม': ['flood', 'อุทกภัย', 'น้ำ'],
    'ไฟไหม้': ['fire', 'เพลิงไหม้', 'ไฟ'],
    'อุบัติเหตุ': ['accident', 'รถชน', 'ชน'],
    'bkk': ['กรุงเทพ', 'bangkok', 'กทม'],
    'cm': ['เชียงใหม่', 'chiang mai'],
  });

  console.log('Posts index tuned successfully');
}

async function tuneUsersIndex(): Promise<void> {
  const index = client.index('users');

  await index.updateSettings({
    rankingRules: [
      'words',
      'typo',
      'attribute',
      'sort',
      'exactness',
      'followerCount:desc',
    ],

    // Username exact match สำคัญกว่า bio
    searchableAttributes: [
      'username',
      'displayName',
      'bio',
    ],
  });
}

// รัน
(async () => {
  await tunePostsIndex();
  await tuneUsersIndex();
  console.log('All indexes tuned');
  process.exit(0);
})();
```

---

## 🔧 Configuration Files

```bash
# /etc/meilisearch/config.toml
env = "production"
master_key = "your-super-secret-master-key-minimum-16-chars"
db_path = "/var/lib/meilisearch/data"
http_addr = "127.0.0.1:7700"
log_level = "WARN"

[indexer_options]
max_indexing_memory = "1 GiB"
max_indexing_threads = 4

# ── Nginx proxy สำหรับ MeiliSearch ──
# /etc/nginx/sites-available/meilisearch
server {
    listen 7701;
    location / {
        proxy_pass http://127.0.0.1:7700;
        # ห้าม access จาก public (internal only)
        allow 127.0.0.1;
        deny all;
    }
}
```

---

## 🧪 Testing

### Step 290: Search Tests

```bash
# ทดสอบ full-text search
curl "http://localhost:3006/api/v1/search?q=น้ำท่วม&type=posts" \
  -H "Authorization: Bearer $TOKEN" | jq '.hits | length'

# ทดสอบ autocomplete
curl "http://localhost:3006/api/v1/search/autocomplete?q=น้ำ&type=hashtag" \
  -H "Authorization: Bearer $TOKEN"

# ทดสอบ geo-search
curl "http://localhost:3006/api/v1/search/nearby?q=ช่วยเหลือ&lat=13.75&lng=100.50&radius=20" \
  -H "Authorization: Bearer $TOKEN"

# ทดสอบ trending
curl "http://localhost:3006/api/v1/search/trending?province=10" \
  -H "Authorization: Bearer $TOKEN"
```

```typescript
// tests/search.test.ts
import { describe, it, expect } from '@jest/globals';

describe('MeiliSearch Integration', () => {
  it('should return Thai language results', async () => {
    const result = await fetch(
      'http://localhost:7700/indexes/posts/search',
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${process.env.MEILISEARCH_KEY}`,
        },
        body: JSON.stringify({ q: 'น้ำท่วม', limit: 5 }),
      }
    );

    const data = await result.json();
    expect(data.hits.length).toBeGreaterThanOrEqual(0);
    expect(data.processingTimeMs).toBeLessThan(100); // < 100ms
  });

  it('should handle typos in Thai', async () => {
    // ผลลัพธ์ควรยังหาเจอแม้พิมพ์ผิด
    const result = await fetch(
      'http://localhost:7700/indexes/posts/search',
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${process.env.MEILISEARCH_KEY}`,
        },
        body: JSON.stringify({ q: 'flood', limit: 5 }),
      }
    );
    const data = await result.json();
    // ควรหาเจอ synonym 'น้ำท่วม'
    expect(data.hits).toBeDefined();
  });
});
```

---

## ❌ Common Errors & Solutions

### Error 1: MeiliSearch Out of Memory

```
Error: OOM - memory limit reached
```

```bash
# แก้: ลด max_indexing_memory ใน config
sudo nano /etc/meilisearch/config.toml
# เปลี่ยน max_indexing_memory = "512 MiB"

# หรือเพิ่ม swap
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

### Error 2: CDC Sync Out of Sync

```
MeiliSearch has 50,000 posts but PostgreSQL has 75,000
```

```bash
# แก้: รัน full re-index
npm run reindex -- --index=posts

# หรือผ่าน API
curl -X POST http://localhost:3006/admin/reindex/posts \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

### Error 3: Thai Text Not Tokenized Correctly

```
ค้นหา "น้ำท่วม" ไม่เจอโพสต์ "น้ำท่วมกรุงเทพ"
```

```typescript
// แก้: MeiliSearch ใช้ Unicode segmentation
// แต่สำหรับ Thai อาจต้องเพิ่ม synonyms
await index.updateSynonyms({
  'น้ำท่วม': ['น้ำท่วมกรุงเทพ', 'น้ำท่วมเชียงใหม่', 'อุทกภัย'],
});

// หรือ index ด้วย pre-tokenized text
// แยก "น้ำท่วมกรุงเทพ" → "น้ำท่วม กรุงเทพ"
```

---

## ✅ Checklist

- [ ] **Step 281**: ติดตั้ง MeiliSearch และ configure systemd service
- [ ] **Step 282**: ติดตั้ง Node.js dependencies
- [ ] **Step 283**: Configure MeiliSearch indexes (posts, users, hashtags, sos)
- [ ] **Step 284**: Implement SearchService ด้วย multi-index support
- [ ] **Step 285**: Implement AutocompleteService ด้วย Redis prefix
- [ ] **Step 286**: Implement TrendingService ด้วย sliding window
- [ ] **Step 287**: Implement CDCSyncService (PostgreSQL → MeiliSearch)
- [ ] **Step 288**: สร้าง Search API routes ทั้งหมด
- [ ] **Step 289**: Tune search relevance (synonyms, stop words, ranking)
- [ ] **Step 290**: Integration tests ผ่านทั้งหมด
- [ ] Verify Thai text search ทำงานถูกต้อง
- [ ] Verify typo tolerance ทำงาน
- [ ] Verify geo-search ใน MeiliSearch
- [ ] Verify CDC sync ทำงาน real-time (< 10 seconds delay)
- [ ] Load test: 1000 search requests/sec ที่ p99 < 50ms

---

## 🔗 References

- [MeiliSearch Documentation](https://www.meilisearch.com/docs)
- [MeiliSearch Ranking Rules](https://www.meilisearch.com/docs/learn/core_concepts/relevancy)
- [Redis Sorted Sets](https://redis.io/docs/data-types/sorted-sets/)
- [Change Data Capture Pattern](https://debezium.io/documentation/reference/stable/connectors/postgresql.html)

---
*Part 029 | Road to 1,000,000 Users/Day | chuaikan.com*
