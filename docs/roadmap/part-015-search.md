# Part 015: Search ด้วย Elasticsearch / MeiliSearch

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 141-150
> **เวลาโดยประมาณ:** 8 ชั่วโมง
> **Prerequisites:** Part 014 (Notifications), Part 001-010 (Infrastructure)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

1. เปรียบเทียบ MeiliSearch vs Elasticsearch สำหรับ chuaikan.com
2. ติดตั้ง MeiliSearch บน Ubuntu 24.04 LTS
3. ตั้งค่า MeiliSearch index สำหรับ Posts, Users, SOS Alerts
4. ตั้งค่า Thai language search
5. สร้าง Full-text Search API พร้อม Highlighting
6. Faceted Search: กรองตาม category, location, date
7. Fuzzy Search และ Typo Tolerance
8. Search Ranking และ Relevance Tuning
9. Sync ข้อมูลจาก PostgreSQL ไปยัง MeiliSearch (CDC)
10. Elasticsearch Alternative, Autocomplete, Analytics

---

## 📖 ทฤษฎีและแนวคิด

### MeiliSearch vs Elasticsearch

```
┌──────────────────────────────────────────────────────────────┐
│          Search Engine Comparison                             │
├─────────────────┬─────────────────────┬──────────────────────┤
│ Feature         │ MeiliSearch         │ Elasticsearch        │
├─────────────────┼─────────────────────┼──────────────────────┤
│ Setup           │ ง่ายมาก (1 binary) │ ซับซ้อน (JVM)       │
│ RAM Usage       │ น้อย (~256MB)       │ มาก (~2GB+)          │
│ Search Speed    │ <50ms (fast)        │ <100ms               │
│ Thai Language   │ ต้องตั้งค่า tokenizer│ มี plugins          │
│ Fuzzy Search    │ Built-in            │ Manual config        │
│ Facets/Filters  │ Built-in            │ Aggregations         │
│ Typo Tolerance  │ Built-in (ดีมาก)   │ Manual levenshtein   │
│ Scaling         │ Single node (v1)    │ Cluster-ready        │
│ Cost (self)     │ ฟรี                 │ ฟรี (OSS)           │
│ Use Case        │ App search (<10M)   │ Big data, Analytics  │
├─────────────────┼─────────────────────┼──────────────────────┤
│ chuaikan สรุป   │ ✅ แนะนำ            │ ✅ เมื่อ scale ใหญ่   │
└─────────────────┴─────────────────────┴──────────────────────┘

สำหรับ chuaikan.com:
- ช่วงเริ่มต้น (<1M users): ใช้ MeiliSearch
- เมื่อโต (>1M users, >100M documents): พิจารณา Elasticsearch
```

### Search Architecture

```
User Query: "อุบัติเหตุ สาทร ด่วน"
                │
                ▼
        ┌───────────────┐
        │  Next.js API  │
        │  /api/search  │
        └───────┬───────┘
                │
       ┌────────▼─────────┐
       │   MeiliSearch    │
       │                  │
       │  Indexes:        │
       │  ├── posts       │
       │  ├── users       │
       │  └── sos_alerts  │
       └────────┬─────────┘
                │
     ┌──────────┼──────────┐
     │          │          │
  Results    Facets    Analytics
  (ranked)  (filters) (what users search)
```

### Thai Language Tokenization Challenge

```
ภาษาไทยไม่มีช่องว่างระหว่างคำ:
"อุบัติเหตุรถชนกันที่สาทร"
          ↓ ต้องตัดคำก่อน
["อุบัติเหตุ", "รถ", "ชน", "กัน", "ที่", "สาทร"]
          ↓ แล้วค้นหา
"อุบัติเหตุ สาทร" → match ได้
```

---

## ⚙️ Environment Setup

### Step 141: ติดตั้ง MeiliSearch บน Ubuntu 24.04

```bash
# ดาวน์โหลดและติดตั้ง MeiliSearch
curl -L https://install.meilisearch.com | sh

# ย้ายไปยัง PATH
sudo mv ./meilisearch /usr/local/bin/

# ตรวจสอบ version
meilisearch --version
# meilisearch 1.8.x

# สร้าง user สำหรับรัน MeiliSearch (security best practice)
sudo useradd -r -s /bin/false meilisearch
sudo mkdir -p /var/lib/meilisearch/data
sudo mkdir -p /var/lib/meilisearch/dumps
sudo mkdir -p /var/log/meilisearch
sudo chown -R meilisearch:meilisearch /var/lib/meilisearch /var/log/meilisearch

# สร้าง config file
sudo cat > /etc/meilisearch.toml << 'EOF'
# MeiliSearch Configuration
db_path = "/var/lib/meilisearch/data"
env = "production"
master_key = "your-super-secret-master-key-minimum-16-chars"
http_addr = "127.0.0.1:7700"
log_level = "INFO"
max_indexing_memory = "1 GiB"
max_indexing_threads = 4

# Dump settings
dump_dir = "/var/lib/meilisearch/dumps"
EOF

# สร้าง systemd service
sudo cat > /etc/systemd/system/meilisearch.service << 'EOF'
[Unit]
Description=MeiliSearch
After=network.target

[Service]
User=meilisearch
Group=meilisearch
ExecStart=/usr/local/bin/meilisearch --config-file-path /etc/meilisearch.toml
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable meilisearch
sudo systemctl start meilisearch

# ตรวจสอบสถานะ
sudo systemctl status meilisearch

# ทดสอบว่า MeiliSearch ทำงาน
curl http://localhost:7700/health
# {"status":"available"}

# ตรวจสอบด้วย master key
curl http://localhost:7700/version \
  -H 'Authorization: Bearer your-super-secret-master-key-minimum-16-chars'
```

### Step 142: ติดตั้ง Dependencies สำหรับ Node.js

```bash
cd /home/user/chuaikan-api

# MeiliSearch client
npm install meilisearch@0.41.0

# Thai word tokenizer (สำหรับ pre-processing)
npm install thai-tokenizer@1.0.1

# หรือใช้ budoux (Google's Thai tokenizer)
npm install budoux@0.6.5

# CDC (Change Data Capture) สำหรับ sync PostgreSQL → MeiliSearch
# ใช้ node-postgres สำหรับ LISTEN/NOTIFY
npm install pg@8.12.0 pg-listen@1.7.0

# Type definitions
npm install -D @types/pg@8.11.6

# เพิ่ม environment variables
cat >> /home/user/chuaikan-api/.env << 'EOF'
MEILISEARCH_HOST=http://localhost:7700
MEILISEARCH_MASTER_KEY=your-super-secret-master-key-minimum-16-chars
MEILISEARCH_SEARCH_KEY=your-search-only-key
EOF
```

---

## 🛠️ Step-by-Step Implementation

### Step 143: สร้าง MeiliSearch Client และ Index Configuration

สร้างไฟล์ `/home/user/chuaikan-api/src/search/meiliClient.ts`:

```typescript
import { MeiliSearch } from 'meilisearch';

export const meiliClient = new MeiliSearch({
  host: process.env.MEILISEARCH_HOST || 'http://localhost:7700',
  apiKey: process.env.MEILISEARCH_MASTER_KEY,
});

// Index names
export const INDEXES = {
  POSTS: 'posts',
  USERS: 'users',
  SOS_ALERTS: 'sos_alerts',
} as const;

// Types
export interface PostDocument {
  id: string;
  userId: string;
  username: string;
  displayName: string;
  content: string;
  contentTokenized?: string;  // Thai-tokenized content
  type: string;
  category: string;
  tags: string[];
  location?: {
    lat: number;
    lng: number;
    name: string;
    district?: string;
    province?: string;
  };
  likeCount: number;
  commentCount: number;
  shareCount: number;
  hasMedia: boolean;
  visibility: string;
  createdAt: number;  // Unix timestamp (MeiliSearch ใช้ number สำหรับ sort)
  updatedAt: number;
}

export interface UserDocument {
  id: string;
  username: string;
  displayName: string;
  bio?: string;
  location?: string;
  followerCount: number;
  followingCount: number;
  postCount: number;
  isVerified: boolean;
  createdAt: number;
}

export interface SOSDocument {
  id: string;
  userId: string;
  username: string;
  type: string;         // 'accident', 'flood', 'fire', etc.
  severity: string;     // 'low', 'medium', 'high', 'critical'
  description: string;
  descriptionTokenized?: string;
  location: {
    lat: number;
    lng: number;
    name: string;
    district: string;
    province: string;
  };
  status: string;       // 'active', 'responding', 'resolved'
  volunteerCount: number;
  createdAt: number;
  resolvedAt?: number;
}
```

### Step 144: Index Setup Script

สร้างไฟล์ `/home/user/chuaikan-api/src/search/setupIndexes.ts`:

```typescript
import { meiliClient, INDEXES } from './meiliClient';

export async function setupMeiliSearchIndexes(): Promise<void> {
  console.log('Setting up MeiliSearch indexes...');

  // ============================================
  // Posts Index
  // ============================================
  await meiliClient.createIndex(INDEXES.POSTS, { primaryKey: 'id' });
  const postsIndex = meiliClient.index(INDEXES.POSTS);

  await postsIndex.updateSettings({
    // Fields ที่ค้นหาได้ (เรียงตาม priority)
    searchableAttributes: [
      'content',
      'contentTokenized',   // Thai tokenized version
      'tags',
      'username',
      'displayName',
      'location.name',
      'location.district',
      'location.province',
      'category',
    ],

    // Fields ที่ใช้กรอง (Facets)
    filterableAttributes: [
      'category',
      'type',
      'visibility',
      'hasMedia',
      'userId',
      'location.district',
      'location.province',
      'createdAt',
      'likeCount',
    ],

    // Fields ที่ใช้ sort
    sortableAttributes: [
      'createdAt',
      'likeCount',
      'commentCount',
      'shareCount',
    ],

    // Fields ที่แสดงใน search results
    displayedAttributes: [
      'id', 'userId', 'username', 'displayName',
      'content', 'type', 'category', 'tags',
      'location', 'likeCount', 'commentCount',
      'hasMedia', 'createdAt',
    ],

    // Typo tolerance
    typoTolerance: {
      enabled: true,
      minWordSizeForTypos: {
        oneTypo: 4,    // คำที่ยาว >= 4 ตัวอักษรอนุญาต 1 typo
        twoTypos: 8,   // คำที่ยาว >= 8 ตัวอักษรอนุญาต 2 typos
      },
      disableOnWords: ['SOS', 'OTP'],  // คำที่ไม่อนุญาต typo
    },

    // Ranking rules (เรียงตาม priority)
    rankingRules: [
      'words',         // จำนวนคำที่ match
      'typo',          // น้อย typo ขึ้นก่อน
      'proximity',     // คำที่ match อยู่ใกล้กันขึ้นก่อน
      'attribute',     // ตาม searchableAttributes order
      'sort',          // ตาม sort parameter
      'exactness',     // exact match ขึ้นก่อน
      'likeCount:desc',  // custom: โพสต์ที่มี like มากขึ้นก่อน
    ],

    // Highlighting
    highlightPreTag: '<mark>',
    highlightPostTag: '</mark>',

    // Pagination
    pagination: {
      maxTotalHits: 10000,  // max results ที่ผู้ใช้จะเห็น
    },

    // Stop words สำหรับภาษาไทย
    stopWords: ['ที่', 'ใน', 'และ', 'หรือ', 'ของ', 'กับ', 'โดย', 'จาก', 'เป็น'],
  });

  console.log('Posts index configured');

  // ============================================
  // Users Index
  // ============================================
  await meiliClient.createIndex(INDEXES.USERS, { primaryKey: 'id' });
  const usersIndex = meiliClient.index(INDEXES.USERS);

  await usersIndex.updateSettings({
    searchableAttributes: [
      'username',
      'displayName',
      'bio',
      'location',
    ],
    filterableAttributes: [
      'isVerified',
      'location',
    ],
    sortableAttributes: [
      'followerCount',
      'postCount',
      'createdAt',
    ],
    typoTolerance: {
      enabled: true,
      minWordSizeForTypos: {
        oneTypo: 3,
        twoTypos: 6,
      },
    },
  });

  console.log('Users index configured');

  // ============================================
  // SOS Alerts Index
  // ============================================
  await meiliClient.createIndex(INDEXES.SOS_ALERTS, { primaryKey: 'id' });
  const sosIndex = meiliClient.index(INDEXES.SOS_ALERTS);

  await sosIndex.updateSettings({
    searchableAttributes: [
      'description',
      'descriptionTokenized',
      'type',
      'location.name',
      'location.district',
      'location.province',
    ],
    filterableAttributes: [
      'type',
      'severity',
      'status',
      'location.district',
      'location.province',
      'createdAt',
    ],
    sortableAttributes: [
      'createdAt',
      'severity',
    ],
    rankingRules: [
      'words',
      'typo',
      'proximity',
      'attribute',
      'sort',
      'exactness',
      'createdAt:desc',  // SOS ล่าสุดขึ้นก่อน
    ],
  });

  console.log('SOS Alerts index configured');
  console.log('All indexes setup complete!');
}

// รัน setup
setupMeiliSearchIndexes().catch(console.error);
```

### Step 145: Thai Language Tokenization

สร้างไฟล์ `/home/user/chuaikan-api/src/search/thaiTokenizer.ts`:

```typescript
// ใช้ budoux สำหรับ Thai word segmentation
// budoux เป็น ML-based tokenizer จาก Google
// npm install budoux

// Simple Thai tokenizer (fallback ถ้าไม่มี library)
// ใช้ regex-based segmentation

const THAI_STOP_WORDS = new Set([
  'ที่', 'ใน', 'และ', 'หรือ', 'ของ', 'กับ', 'โดย', 'จาก',
  'เป็น', 'ได้', 'มี', 'จะ', 'ไม่', 'แต่', 'ก็', 'ว่า',
  'นี้', 'นั้น', 'มา', 'ไป', 'คือ', 'อยู่', 'ให้', 'ถ้า',
]);

export function tokenizeThai(text: string): string {
  if (!text) return '';

  try {
    // ลอง import budoux
    const { loadThaiParser } = require('budoux');
    const parser = loadThaiParser();
    const chunks = parser.parse(text);
    return chunks
      .filter((chunk: string) => !THAI_STOP_WORDS.has(chunk.trim()))
      .join(' ');
  } catch {
    // Fallback: แค่เพิ่มช่องว่างระหว่างบล็อก Thai/English
    return simpleThaiTokenize(text);
  }
}

function simpleThaiTokenize(text: string): string {
  // เพิ่มช่องว่างระหว่าง Thai characters block และ non-Thai
  return text
    .replace(/([ก-๙])([a-zA-Z0-9])/g, '$1 $2')
    .replace(/([a-zA-Z0-9])([ก-๙])/g, '$1 $2')
    .replace(/\s+/g, ' ')
    .trim();
}

// Normalize text สำหรับ search
export function normalizeSearchText(text: string): string {
  return text
    .toLowerCase()
    .replace(/[.,\/#!$%\^&\*;:{}=\-_`~()]/g, ' ')
    .replace(/\s{2,}/g, ' ')
    .trim();
}

// สร้าง search-friendly version ของ document
export function prepareDocumentText(content: string): {
  original: string;
  tokenized: string;
  normalized: string;
} {
  const original = content;
  const tokenized = tokenizeThai(content);
  const normalized = normalizeSearchText(content);

  return { original, tokenized, normalized };
}
```

### Step 146: Search API

สร้างไฟล์ `/home/user/chuaikan-api/src/search/searchService.ts`:

```typescript
import { meiliClient, INDEXES, PostDocument, UserDocument, SOSDocument } from './meiliClient';
import { tokenizeThai, normalizeSearchText } from './thaiTokenizer';

// ============================================
// Full-text Search สำหรับ Posts
// ============================================
export interface SearchPostsParams {
  query: string;
  page?: number;
  hitsPerPage?: number;
  // Facet filters
  category?: string;
  district?: string;
  province?: string;
  hasMedia?: boolean;
  // Date range
  fromDate?: Date;
  toDate?: Date;
  // Sort
  sortBy?: 'relevance' | 'latest' | 'popular';
}

export interface SearchResult<T> {
  hits: Array<T & {
    _formatted?: Record<string, string>;  // Highlighted version
    _matchesPosition?: Record<string, Array<{ start: number; length: number }>>;
  }>;
  totalHits: number;
  page: number;
  hitsPerPage: number;
  totalPages: number;
  processingTimeMs: number;
  facetDistribution?: Record<string, Record<string, number>>;
  query: string;
}

export async function searchPosts(
  params: SearchPostsParams
): Promise<SearchResult<PostDocument>> {
  const {
    query,
    page = 1,
    hitsPerPage = 20,
    category,
    district,
    province,
    hasMedia,
    fromDate,
    toDate,
    sortBy = 'relevance',
  } = params;

  const postsIndex = meiliClient.index(INDEXES.POSTS);

  // Tokenize Thai query
  const tokenizedQuery = tokenizeThai(query);
  const searchQuery = tokenizedQuery || query;

  // สร้าง filter array
  const filters: string[] = [
    "visibility = 'public'",  // แสดงเฉพาะ public posts
  ];

  if (category) filters.push(`category = '${category}'`);
  if (district) filters.push(`location.district = '${district}'`);
  if (province) filters.push(`location.province = '${province}'`);
  if (hasMedia !== undefined) filters.push(`hasMedia = ${hasMedia}`);
  if (fromDate) filters.push(`createdAt >= ${Math.floor(fromDate.getTime() / 1000)}`);
  if (toDate) filters.push(`createdAt <= ${Math.floor(toDate.getTime() / 1000)}`);

  // Sort settings
  const sort: string[] = [];
  if (sortBy === 'latest') sort.push('createdAt:desc');
  if (sortBy === 'popular') sort.push('likeCount:desc', 'commentCount:desc');
  // relevance: ไม่ต้องกำหนด sort (ใช้ ranking rules)

  const searchOptions = {
    filter: filters.join(' AND '),
    sort,
    offset: (page - 1) * hitsPerPage,
    limit: hitsPerPage,
    // Highlighting
    attributesToHighlight: ['content', 'contentTokenized'],
    highlightPreTag: '<mark class="search-highlight">',
    highlightPostTag: '</mark>',
    // Crop (ตัดข้อความแสดงเฉพาะส่วนที่ match)
    attributesToCrop: ['content'],
    cropLength: 200,
    // Facets
    facets: ['category', 'location.province', 'hasMedia'],
    // Match positions
    attributesToRetrieve: [
      'id', 'userId', 'username', 'displayName',
      'content', 'type', 'category', 'tags',
      'location', 'likeCount', 'commentCount',
      'hasMedia', 'createdAt',
    ],
  };

  const result = await postsIndex.search<PostDocument>(searchQuery, searchOptions);

  return {
    hits: result.hits,
    totalHits: result.estimatedTotalHits || 0,
    page,
    hitsPerPage,
    totalPages: Math.ceil((result.estimatedTotalHits || 0) / hitsPerPage),
    processingTimeMs: result.processingTimeMs,
    facetDistribution: result.facetDistribution as Record<string, Record<string, number>>,
    query: result.query,
  };
}

// ============================================
// Autocomplete / Typeahead
// ============================================
export async function autocomplete(
  query: string,
  indexName: typeof INDEXES[keyof typeof INDEXES] = INDEXES.POSTS
): Promise<string[]> {
  if (!query || query.length < 2) return [];

  const index = meiliClient.index(indexName);
  
  const result = await index.search(query, {
    limit: 8,
    attributesToRetrieve: ['content', 'username', 'displayName'],
    attributesToHighlight: [],
  });

  // ดึง unique suggestions จาก results
  const suggestions = new Set<string>();

  result.hits.forEach((hit: any) => {
    // แยกคำจาก content และนำคำที่ match กับ query มาเสนอ
    const words = (hit.content || hit.displayName || '').split(/\s+/);
    words.forEach((word: string) => {
      if (
        word.length > 2 &&
        word.toLowerCase().includes(query.toLowerCase())
      ) {
        suggestions.add(word);
      }
    });
  });

  return Array.from(suggestions).slice(0, 8);
}

// ============================================
// Search SOS Alerts
// ============================================
export async function searchSOSAlerts(
  query: string,
  options: {
    status?: string;
    severity?: string;
    province?: string;
    page?: number;
    hitsPerPage?: number;
  } = {}
): Promise<SearchResult<SOSDocument>> {
  const { status, severity, province, page = 1, hitsPerPage = 20 } = options;

  const sosIndex = meiliClient.index(INDEXES.SOS_ALERTS);

  const filters: string[] = [];
  if (status) filters.push(`status = '${status}'`);
  if (severity) filters.push(`severity = '${severity}'`);
  if (province) filters.push(`location.province = '${province}'`);

  const tokenizedQuery = tokenizeThai(query);

  const result = await sosIndex.search<SOSDocument>(tokenizedQuery || query, {
    filter: filters.length > 0 ? filters.join(' AND ') : undefined,
    sort: ['createdAt:desc'],
    offset: (page - 1) * hitsPerPage,
    limit: hitsPerPage,
    attributesToHighlight: ['description', 'location.name'],
    facets: ['type', 'severity', 'status', 'location.province'],
  });

  return {
    hits: result.hits,
    totalHits: result.estimatedTotalHits || 0,
    page,
    hitsPerPage,
    totalPages: Math.ceil((result.estimatedTotalHits || 0) / hitsPerPage),
    processingTimeMs: result.processingTimeMs,
    facetDistribution: result.facetDistribution as Record<string, Record<string, number>>,
    query: result.query,
  };
}

// ============================================
// Search Analytics: บันทึกสิ่งที่ user ค้นหา
// ============================================
export async function trackSearchQuery(
  query: string,
  userId?: string,
  resultsCount?: number
): Promise<void> {
  // TODO: บันทึกลง PostgreSQL หรือ ClickHouse
  // สำหรับ analytics และ improving search quality
  console.log(`Search tracked: "${query}" by ${userId || 'anonymous'} → ${resultsCount} results`);
}
```

### Step 147: PostgreSQL → MeiliSearch Sync (CDC)

สร้างไฟล์ `/home/user/chuaikan-api/src/search/cdcSyncer.ts`:

```typescript
import { Pool } from 'pg';
import pgListen from 'pg-listen';
import { meiliClient, INDEXES, PostDocument, UserDocument } from './meiliClient';
import { tokenizeThai } from './thaiTokenizer';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 5,
});

// ============================================
// Setup PostgreSQL NOTIFY triggers
// ============================================
export async function setupCDCTriggers(): Promise<void> {
  const client = await pool.connect();

  try {
    // สร้าง function สำหรับ notify
    await client.query(`
      CREATE OR REPLACE FUNCTION notify_search_sync()
      RETURNS TRIGGER AS $$
      DECLARE
        payload JSONB;
      BEGIN
        IF (TG_OP = 'DELETE') THEN
          payload = jsonb_build_object(
            'operation', TG_OP,
            'table', TG_TABLE_NAME,
            'id', OLD.id::text
          );
        ELSE
          payload = jsonb_build_object(
            'operation', TG_OP,
            'table', TG_TABLE_NAME,
            'id', NEW.id::text
          );
        END IF;
        
        PERFORM pg_notify('search_sync', payload::text);
        RETURN NEW;
      END;
      $$ LANGUAGE plpgsql;
    `);

    // สร้าง trigger บน posts table
    await client.query(`
      DROP TRIGGER IF EXISTS posts_search_sync ON posts;
      CREATE TRIGGER posts_search_sync
        AFTER INSERT OR UPDATE OR DELETE ON posts
        FOR EACH ROW
        EXECUTE FUNCTION notify_search_sync();
    `);

    // สร้าง trigger บน users table
    await client.query(`
      DROP TRIGGER IF EXISTS users_search_sync ON users;
      CREATE TRIGGER users_search_sync
        AFTER INSERT OR UPDATE OR DELETE ON users
        FOR EACH ROW
        EXECUTE FUNCTION notify_search_sync();
    `);

    // สร้าง trigger บน sos_alerts table
    await client.query(`
      DROP TRIGGER IF EXISTS sos_alerts_search_sync ON sos_alerts;
      CREATE TRIGGER sos_alerts_search_sync
        AFTER INSERT OR UPDATE OR DELETE ON sos_alerts
        FOR EACH ROW
        EXECUTE FUNCTION notify_search_sync();
    `);

    console.log('CDC triggers setup complete');
  } finally {
    client.release();
  }
}

// ============================================
// Listen และ sync ข้อมูล
// ============================================
export async function startCDCListener(): Promise<void> {
  const subscriber = pgListen({
    connectionString: process.env.DATABASE_URL!,
  });

  subscriber.notifications.on('search_sync', async (payload: string) => {
    try {
      const event = JSON.parse(payload) as {
        operation: 'INSERT' | 'UPDATE' | 'DELETE';
        table: string;
        id: string;
      };

      console.log(`CDC event: ${event.operation} on ${event.table} id=${event.id}`);

      switch (event.table) {
        case 'posts':
          await syncPost(event.id, event.operation);
          break;
        case 'users':
          await syncUser(event.id, event.operation);
          break;
        case 'sos_alerts':
          await syncSOSAlert(event.id, event.operation);
          break;
      }
    } catch (error) {
      console.error('CDC sync error:', error);
    }
  });

  subscriber.events.on('error', (err) => {
    console.error('CDC listener error:', err);
  });

  await subscriber.connect();
  await subscriber.listenTo('search_sync');

  console.log('CDC listener started');
}

// Sync post ไปยัง MeiliSearch
async function syncPost(
  postId: string,
  operation: 'INSERT' | 'UPDATE' | 'DELETE'
): Promise<void> {
  const postsIndex = meiliClient.index(INDEXES.POSTS);

  if (operation === 'DELETE') {
    await postsIndex.deleteDocument(postId);
    return;
  }

  // ดึงข้อมูล post จาก PostgreSQL
  const result = await pool.query(
    `SELECT
      p.id,
      p.user_id as "userId",
      u.username,
      u.display_name as "displayName",
      p.content,
      p.type,
      p.category,
      p.tags,
      p.location_lat as lat,
      p.location_lng as lng,
      p.location_name as "locationName",
      p.location_district as "locationDistrict",
      p.location_province as "locationProvince",
      p.like_count as "likeCount",
      p.comment_count as "commentCount",
      p.share_count as "shareCount",
      p.has_media as "hasMedia",
      p.visibility,
      EXTRACT(EPOCH FROM p.created_at)::bigint as "createdAt",
      EXTRACT(EPOCH FROM p.updated_at)::bigint as "updatedAt"
    FROM posts p
    JOIN users u ON p.user_id = u.id
    WHERE p.id = $1 AND p.deleted_at IS NULL`,
    [postId]
  );

  if (result.rows.length === 0) return;

  const row = result.rows[0];

  const document: PostDocument = {
    id: row.id,
    userId: row.userId,
    username: row.username,
    displayName: row.displayName,
    content: row.content,
    contentTokenized: tokenizeThai(row.content),  // Thai tokenization
    type: row.type,
    category: row.category,
    tags: row.tags || [],
    location: row.lat
      ? {
          lat: parseFloat(row.lat),
          lng: parseFloat(row.lng),
          name: row.locationName,
          district: row.locationDistrict,
          province: row.locationProvince,
        }
      : undefined,
    likeCount: parseInt(row.likeCount || 0),
    commentCount: parseInt(row.commentCount || 0),
    shareCount: parseInt(row.shareCount || 0),
    hasMedia: row.hasMedia,
    visibility: row.visibility,
    createdAt: row.createdAt,
    updatedAt: row.updatedAt,
  };

  await postsIndex.addDocuments([document], { primaryKey: 'id' });
}

async function syncUser(
  userId: string,
  operation: 'INSERT' | 'UPDATE' | 'DELETE'
): Promise<void> {
  const usersIndex = meiliClient.index(INDEXES.USERS);

  if (operation === 'DELETE') {
    await usersIndex.deleteDocument(userId);
    return;
  }

  const result = await pool.query(
    `SELECT
      id, username,
      display_name as "displayName",
      bio, location,
      follower_count as "followerCount",
      following_count as "followingCount",
      post_count as "postCount",
      is_verified as "isVerified",
      EXTRACT(EPOCH FROM created_at)::bigint as "createdAt"
    FROM users
    WHERE id = $1 AND deleted_at IS NULL`,
    [userId]
  );

  if (result.rows.length === 0) return;
  await usersIndex.addDocuments([result.rows[0]], { primaryKey: 'id' });
}

async function syncSOSAlert(sosId: string, operation: 'INSERT' | 'UPDATE' | 'DELETE'): Promise<void> {
  const sosIndex = meiliClient.index(INDEXES.SOS_ALERTS);

  if (operation === 'DELETE') {
    await sosIndex.deleteDocument(sosId);
    return;
  }

  const result = await pool.query(
    `SELECT
      s.id, s.user_id as "userId", u.username,
      s.type, s.severity, s.description,
      s.location_lat as lat, s.location_lng as lng,
      s.location_name as "locationName",
      s.location_district as "locationDistrict",
      s.location_province as "locationProvince",
      s.status, s.volunteer_count as "volunteerCount",
      EXTRACT(EPOCH FROM s.created_at)::bigint as "createdAt",
      EXTRACT(EPOCH FROM s.resolved_at)::bigint as "resolvedAt"
    FROM sos_alerts s JOIN users u ON s.user_id = u.id
    WHERE s.id = $1`,
    [sosId]
  );

  if (result.rows.length === 0) return;
  const row = result.rows[0];

  const doc = {
    id: row.id,
    userId: row.userId,
    username: row.username,
    type: row.type,
    severity: row.severity,
    description: row.description,
    descriptionTokenized: tokenizeThai(row.description),
    location: {
      lat: parseFloat(row.lat),
      lng: parseFloat(row.lng),
      name: row.locationName,
      district: row.locationDistrict,
      province: row.locationProvince,
    },
    status: row.status,
    volunteerCount: parseInt(row.volunteerCount || 0),
    createdAt: row.createdAt,
    resolvedAt: row.resolvedAt,
  };

  await sosIndex.addDocuments([doc], { primaryKey: 'id' });
}

// ============================================
// Full Reindex (สำหรับ initial sync หรือ rebuild)
// ============================================
export async function reindexAll(): Promise<void> {
  console.log('Starting full reindex...');

  // Reindex Posts
  let offset = 0;
  const batchSize = 1000;

  while (true) {
    const result = await pool.query(
      `SELECT p.id FROM posts p
       WHERE p.deleted_at IS NULL AND p.visibility = 'public'
       ORDER BY p.created_at DESC
       LIMIT $1 OFFSET $2`,
      [batchSize, offset]
    );

    if (result.rows.length === 0) break;

    await Promise.all(result.rows.map((row) => syncPost(row.id, 'INSERT')));
    console.log(`Reindexed posts batch: ${offset} - ${offset + result.rows.length}`);
    offset += batchSize;
  }

  console.log('Full reindex complete');
}
```

---

## 🔧 Configuration Files

### Step 148: Search API Routes

สร้างไฟล์ `/home/user/chuaikan-api/src/routes/search.ts`:

```typescript
import { Router, Request, Response } from 'express';
import { searchPosts, searchSOSAlerts, autocomplete, trackSearchQuery } from '../search/searchService';
import { optionalAuthMiddleware } from '../middleware/auth';

const router = Router();

// GET /api/search/posts?q=...&page=1&category=...
router.get('/posts', optionalAuthMiddleware, async (req: Request, res: Response) => {
  try {
    const {
      q = '',
      page = '1',
      hitsPerPage = '20',
      category,
      district,
      province,
      hasMedia,
      fromDate,
      toDate,
      sortBy = 'relevance',
    } = req.query as Record<string, string>;

    if (!q.trim()) {
      return res.json({
        hits: [],
        totalHits: 0,
        page: 1,
        hitsPerPage: 20,
        totalPages: 0,
        processingTimeMs: 0,
        query: '',
      });
    }

    const results = await searchPosts({
      query: q,
      page: parseInt(page),
      hitsPerPage: Math.min(parseInt(hitsPerPage), 100),
      category,
      district,
      province,
      hasMedia: hasMedia !== undefined ? hasMedia === 'true' : undefined,
      fromDate: fromDate ? new Date(fromDate) : undefined,
      toDate: toDate ? new Date(toDate) : undefined,
      sortBy: sortBy as 'relevance' | 'latest' | 'popular',
    });

    // Track search analytics
    const userId = (req as any).user?.sub;
    await trackSearchQuery(q, userId, results.totalHits);

    res.json(results);
  } catch (error) {
    console.error('Search error:', error);
    res.status(500).json({ error: 'Search failed' });
  }
});

// GET /api/search/sos?q=...&status=active
router.get('/sos', async (req: Request, res: Response) => {
  try {
    const { q = '', status, severity, province, page = '1' } = req.query as Record<string, string>;

    const results = await searchSOSAlerts(q, {
      status,
      severity,
      province,
      page: parseInt(page),
    });

    res.json(results);
  } catch (error) {
    res.status(500).json({ error: 'SOS search failed' });
  }
});

// GET /api/search/autocomplete?q=...
router.get('/autocomplete', async (req: Request, res: Response) => {
  try {
    const { q = '', index = 'posts' } = req.query as Record<string, string>;

    if (q.length < 2) {
      return res.json({ suggestions: [] });
    }

    const suggestions = await autocomplete(q, index as any);
    res.json({ suggestions });
  } catch (error) {
    res.status(500).json({ suggestions: [] });
  }
});

export default router;
```

### Elasticsearch Alternative Setup

```bash
# ติดตั้ง Elasticsearch 8.x (สำหรับ scale ใหญ่)
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | \
  sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] \
  https://artifacts.elastic.co/packages/8.x/apt stable main" | \
  sudo tee /etc/apt/sources.list.d/elastic-8.x.list

sudo apt-get update && sudo apt-get install elasticsearch -y

# ปรับ JVM heap size (แนะนำ 4GB สำหรับ production)
sudo nano /etc/elasticsearch/jvm.options
# -Xms4g
# -Xmx4g

sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch

# สร้าง index mapping สำหรับ chuaikan posts
curl -X PUT "localhost:9200/posts" \
  -H 'Content-Type: application/json' \
  -d '{
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1,
      "analysis": {
        "analyzer": {
          "thai_analyzer": {
            "type": "custom",
            "tokenizer": "thai",
            "filter": ["lowercase", "thai_stop"]
          }
        },
        "filter": {
          "thai_stop": {
            "type": "stop",
            "stopwords": ["ที่", "ใน", "และ", "หรือ", "ของ"]
          }
        }
      }
    },
    "mappings": {
      "properties": {
        "content": {
          "type": "text",
          "analyzer": "thai_analyzer",
          "search_analyzer": "thai_analyzer"
        },
        "username": { "type": "keyword" },
        "category": { "type": "keyword" },
        "location": { "type": "geo_point" },
        "createdAt": { "type": "date" },
        "likeCount": { "type": "integer" }
      }
    }
  }'
```

---

## 🧪 Testing

### Step 149: ทดสอบ Search

```bash
# ทดสอบ MeiliSearch ทำงาน
curl -X GET "http://localhost:7700/indexes" \
  -H "Authorization: Bearer your-master-key"

# ทดสอบ search ด้วย curl
curl -X POST "http://localhost:7700/indexes/posts/search" \
  -H "Authorization: Bearer your-search-only-key" \
  -H "Content-Type: application/json" \
  -d '{
    "q": "อุบัติเหตุ สาทร",
    "limit": 5,
    "filter": "visibility = '\''public'\''",
    "facets": ["category", "location.province"],
    "attributesToHighlight": ["content"],
    "highlightPreTag": "<em>",
    "highlightPostTag": "</em>"
  }'

# ทดสอบ reindex ทั้งหมด
cd /home/user/chuaikan-api
npx ts-node -e "
const { reindexAll } = require('./src/search/cdcSyncer');
reindexAll().then(() => console.log('Done')).catch(console.error);
"

# ทดสอบ API endpoint
curl "http://localhost:3000/api/search/posts?q=อุบัติเหตุ&sortBy=latest"
curl "http://localhost:3000/api/search/autocomplete?q=อุบัติ"
curl "http://localhost:3000/api/search/sos?q=น้ำท่วม&status=active&province=กรุงเทพ"
```

---

## ❌ Common Errors & Solutions

### Error 1: MeiliSearch "Invalid API key"

```bash
# ตรวจสอบ master key
curl http://localhost:7700/keys \
  -H "Authorization: Bearer your-master-key"

# สร้าง search-only API key (สำหรับ frontend ใช้)
curl -X POST http://localhost:7700/keys \
  -H "Authorization: Bearer your-master-key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Search API Key",
    "description": "For frontend search",
    "actions": ["search"],
    "indexes": ["posts", "users", "sos_alerts"],
    "expiresAt": null
  }'
```

### Error 2: Thai search ไม่ match

```bash
# ปัญหา: ค้นหา "อุบัติเหตุ" ไม่เจอ "อุบัติเหตุรถชน"
# สาเหตุ: Thai ไม่มี word boundaries

# วิธีแก้: ใช้ tokenized field
# ใน document ให้มี 2 fields:
# content: "อุบัติเหตุรถชนกันที่สาทร"  
# contentTokenized: "อุบัติเหตุ รถ ชน กัน ที่ สาทร"

# ใน searchableAttributes ให้ใส่ทั้ง 2 fields:
# ["content", "contentTokenized", ...]
```

### Error 3: MeiliSearch ใช้ RAM มากเกินไป

```bash
# ปรับ config
sudo nano /etc/meilisearch.toml

# ลด indexing memory
max_indexing_memory = "512 MiB"  # ลดจาก 1GiB
max_indexing_threads = 2          # ลด threads

# Restart
sudo systemctl restart meilisearch

# Monitor memory usage
watch -n 5 "ps aux | grep meilisearch | grep -v grep"
```

---

## ✅ Checklist

- [ ] MeiliSearch ติดตั้งและรันเป็น systemd service
- [ ] สร้าง master key ที่ปลอดภัย (min 16 chars)
- [ ] Index setup: posts, users, sos_alerts
- [ ] searchableAttributes, filterableAttributes, sortableAttributes ตั้งค่าแล้ว
- [ ] Thai tokenization ทำงาน (budoux หรือ custom)
- [ ] Stop words ภาษาไทยตั้งค่าแล้ว
- [ ] Typo tolerance configured
- [ ] Highlighting ทำงาน
- [ ] Faceted search ทำงาน (category, province, date range)
- [ ] CDC triggers บน PostgreSQL สร้างแล้ว
- [ ] pgListen daemon ทำงาน (auto sync)
- [ ] Full reindex script พร้อมใช้
- [ ] Search API routes: /search/posts, /search/sos, /search/autocomplete
- [ ] Search analytics tracking
- [ ] Search-only API key สำหรับ frontend
- [ ] Load test search endpoint (Part 011)

---

## 🔗 References

- [MeiliSearch Documentation](https://www.meilisearch.com/docs)
- [MeiliSearch Node.js Client](https://github.com/meilisearch/meilisearch-js)
- [MeiliSearch Thai Language](https://www.meilisearch.com/docs/learn/configuration/language)
- [budoux Thai Tokenizer](https://github.com/google/budoux)
- [Elasticsearch Thai Language](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-lang-analyzer.html#thai-analyzer)
- [PostgreSQL LISTEN/NOTIFY](https://www.postgresql.org/docs/current/sql-listen.html)

---

*Part 015 | Road to 1,000,000 Users/Day | chuaikan.com*
