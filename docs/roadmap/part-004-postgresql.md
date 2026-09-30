# Part 004: PostgreSQL พื้นฐานสู่ขั้นสูง
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 31-40
> **เวลาโดยประมาณ:** 12 ชั่วโมง
> **Prerequisites:** Part 001 (Linux Basics), Part 003 (Node.js)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

1. PostgreSQL architecture (processes, shared memory, WAL)
2. Schema design สำหรับ chuaikan.com Social Media Platform
3. Index types: B-tree, GiST, GIN และการใช้งาน
4. EXPLAIN ANALYZE สำหรับ query optimization
5. PgBouncer connection pooling setup
6. VACUUM และ ANALYZE maintenance
7. pg_dump backup และ restore
8. postgresql.conf tuning สำหรับ production server
9. pg_stat_statements สำหรับ query monitoring
10. Common query patterns สำหรับ social feed และ SOS

---

## 📖 ทฤษฎีและแนวคิด

### PostgreSQL Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                    PostgreSQL Server                            │
│                                                                │
│  Client Connections                                            │
│  ────────────────────                                          │
│  App → postmaster → [fork] Backend Process (1 per connection)  │
│                                                                │
│  Shared Memory (Shared Buffers)                                │
│  ──────────────────────────────                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Shared Buffer Pool (25% of RAM = 4GB on 16GB server)   │  │
│  │  WAL Buffers (64MB)                                     │  │
│  │  Lock Table                                             │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
│  Background Processes                                          │
│  ─────────────────────                                         │
│  ├── checkpointer  → flush dirty pages to disk                 │
│  ├── background writer → write dirty buffers gradually        │
│  ├── WAL writer    → flush WAL buffers to disk                 │
│  ├── autovacuum    → cleanup dead tuples                       │
│  └── stats collector → gather statistics                      │
│                                                                │
│  Storage                                                       │
│  ──────────                                                    │
│  ├── Data files ($PGDATA/base/)                                │
│  ├── WAL files ($PGDATA/pg_wal/) → crash recovery              │
│  └── Config files ($PGDATA/postgresql.conf, pg_hba.conf)      │
│                                                                │
└────────────────────────────────────────────────────────────────┘

WAL (Write-Ahead Log) Flow:
Transaction → WAL buffer → WAL file (disk) → Data file
                                ↑
                          Durability: ถ้า crash → replay WAL
```

### MVCC (Multi-Version Concurrency Control)

```
Reader → ไม่ block Writer (ต่างจาก MySQL MyISAM)
Writer → ไม่ block Reader

ทุก row มี: xmin (created by txn), xmax (deleted by txn)
VACUUM → cleanup dead tuples (rows ที่ถูก update/delete)

ทำไม VACUUM สำคัญ:
┌─────────────────────────────────────────────────────────┐
│  UPDATE users SET name = 'Bob' WHERE id = 1;           │
│                                                        │
│  ก่อน UPDATE:                                          │
│  | id | name  | xmin | xmax |                          │
│  | 1  | Alice | 100  | 0    |  ← live tuple            │
│                                                        │
│  หลัง UPDATE:                                          │
│  | id | name  | xmin | xmax |                          │
│  | 1  | Alice | 100  | 200  |  ← dead tuple (waste!)   │
│  | 1  | Bob   | 200  | 0    |  ← live tuple            │
│                                                        │
│  VACUUM ลบ dead tuples → reclaim space                 │
└─────────────────────────────────────────────────────────┘
```

---

## ⚙️ Environment Setup

```bash
# ─── ติดตั้ง PostgreSQL 17 ────────────────────────────────
# เพิ่ม PostgreSQL APT repository
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh

# ติดตั้ง PostgreSQL 17
sudo apt install -y postgresql-17 postgresql-contrib-17

# ตรวจสอบ
psql --version
# Expected: psql (PostgreSQL) 17.x

# Status
sudo systemctl status postgresql

# ─── Initial Setup ────────────────────────────────────────
# เข้า psql ด้วย postgres superuser
sudo -u postgres psql

# สร้าง database และ user
sudo -u postgres psql << 'SQL'
-- สร้าง user
CREATE USER chuaikan_user WITH PASSWORD 'strong_password_here';

-- สร้าง database
CREATE DATABASE chuaikan_db
    WITH
    OWNER = chuaikan_user
    ENCODING = 'UTF8'
    LC_COLLATE = 'th_TH.UTF-8'
    LC_CTYPE = 'th_TH.UTF-8'
    TEMPLATE = template0;

-- ให้สิทธิ์
GRANT ALL PRIVILEGES ON DATABASE chuaikan_db TO chuaikan_user;
\q
SQL

# ─── ติดตั้ง PgBouncer ────────────────────────────────────
sudo apt install -y pgbouncer

# ─── ติดตั้ง pg_activity (monitor) ────────────────────────
pip3 install pg_activity
```

---

## 🛠️ Step-by-Step Implementation

### Step 31: Complete Schema Design สำหรับ chuaikan.com

```sql
-- schema.sql — Complete database schema สำหรับ chuaikan.com
-- PostgreSQL 17

-- ─── Extensions ──────────────────────────────────────────
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";        -- UUID generation
CREATE EXTENSION IF NOT EXISTS "pgcrypto";          -- Encryption
CREATE EXTENSION IF NOT EXISTS "pg_trgm";           -- Trigram (fuzzy search)
CREATE EXTENSION IF NOT EXISTS "unaccent";           -- Thai text search
CREATE EXTENSION IF NOT EXISTS postgis;              -- Geospatial (ถ้า install ได้)
CREATE EXTENSION IF NOT EXISTS "pg_stat_statements"; -- Query monitoring

-- ─── Enums ───────────────────────────────────────────────
CREATE TYPE user_role AS ENUM ('user', 'moderator', 'admin');
CREATE TYPE user_status AS ENUM ('active', 'inactive', 'suspended', 'deleted');
CREATE TYPE post_type AS ENUM ('text', 'image', 'video', 'story');
CREATE TYPE post_status AS ENUM ('draft', 'published', 'archived', 'deleted');
CREATE TYPE alert_severity AS ENUM ('low', 'medium', 'high', 'critical');
CREATE TYPE alert_status AS ENUM ('active', 'resolved', 'false_alarm', 'expired');
CREATE TYPE notification_type AS ENUM (
    'like', 'comment', 'follow', 'sos_nearby',
    'sos_response', 'mention', 'system'
);
CREATE TYPE media_type AS ENUM ('image', 'video', 'audio', 'document');

-- ─── Users Table ─────────────────────────────────────────
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    username        VARCHAR(50) NOT NULL,
    email           VARCHAR(255) NOT NULL,
    phone           VARCHAR(20),
    password_hash   VARCHAR(255) NOT NULL,
    display_name    VARCHAR(100) NOT NULL,
    bio             TEXT,
    avatar_url      VARCHAR(500),
    cover_url       VARCHAR(500),
    role            user_role NOT NULL DEFAULT 'user',
    status          user_status NOT NULL DEFAULT 'active',
    is_verified     BOOLEAN NOT NULL DEFAULT FALSE,
    is_private      BOOLEAN NOT NULL DEFAULT FALSE,

    -- Geolocation (last known location)
    latitude        DECIMAL(10, 8),
    longitude       DECIMAL(11, 8),
    location_updated_at TIMESTAMPTZ,

    -- Settings
    push_enabled    BOOLEAN NOT NULL DEFAULT TRUE,
    email_enabled   BOOLEAN NOT NULL DEFAULT TRUE,
    sos_radius_km   INTEGER NOT NULL DEFAULT 10,

    -- Stats (denormalized for performance)
    post_count      INTEGER NOT NULL DEFAULT 0,
    follower_count  INTEGER NOT NULL DEFAULT 0,
    following_count INTEGER NOT NULL DEFAULT 0,

    -- Timestamps
    last_seen_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ,

    -- Constraints
    CONSTRAINT users_username_check CHECK (
        username ~ '^[a-zA-Z0-9_]{3,50}$'
    ),
    CONSTRAINT users_email_check CHECK (
        email ~ '^[^@]+@[^@]+\.[^@]+$'
    )
);

CREATE UNIQUE INDEX idx_users_username ON users(LOWER(username))
    WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX idx_users_email ON users(LOWER(email))
    WHERE deleted_at IS NULL;
CREATE INDEX idx_users_status ON users(status) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_created_at ON users(created_at DESC);

-- ─── Posts Table ─────────────────────────────────────────
CREATE TABLE posts (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    parent_id       UUID REFERENCES posts(id) ON DELETE SET NULL,
    type            post_type NOT NULL DEFAULT 'text',
    status          post_status NOT NULL DEFAULT 'published',
    content         TEXT,
    content_html    TEXT,
    location_name   VARCHAR(255),
    latitude        DECIMAL(10, 8),
    longitude       DECIMAL(11, 8),
    is_pinned       BOOLEAN NOT NULL DEFAULT FALSE,
    view_count      INTEGER NOT NULL DEFAULT 0,
    like_count      INTEGER NOT NULL DEFAULT 0,
    comment_count   INTEGER NOT NULL DEFAULT 0,
    share_count     INTEGER NOT NULL DEFAULT 0,

    -- Search
    search_vector   TSVECTOR,

    -- Timestamps
    published_at    TIMESTAMPTZ DEFAULT NOW(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ,

    CONSTRAINT posts_content_check CHECK (
        content IS NOT NULL OR type != 'text'
    )
);

CREATE INDEX idx_posts_user_id ON posts(user_id, created_at DESC)
    WHERE deleted_at IS NULL AND status = 'published';
CREATE INDEX idx_posts_created_at ON posts(created_at DESC)
    WHERE deleted_at IS NULL AND status = 'published';
CREATE INDEX idx_posts_parent_id ON posts(parent_id)
    WHERE parent_id IS NOT NULL AND deleted_at IS NULL;
CREATE INDEX idx_posts_search ON posts USING GIN(search_vector);
CREATE INDEX idx_posts_location ON posts USING GIST(
    ll_to_earth(latitude, longitude)
) WHERE latitude IS NOT NULL AND longitude IS NOT NULL;

-- Auto-update search_vector
CREATE OR REPLACE FUNCTION update_post_search_vector()
RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector = to_tsvector('thai',
        COALESCE(NEW.content, '') || ' ' ||
        COALESCE(NEW.location_name, '')
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER posts_search_vector_update
    BEFORE INSERT OR UPDATE ON posts
    FOR EACH ROW EXECUTE FUNCTION update_post_search_vector();

-- ─── Media Table ─────────────────────────────────────────
CREATE TABLE media (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    post_id         UUID REFERENCES posts(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    type            media_type NOT NULL,
    url             VARCHAR(500) NOT NULL,
    thumbnail_url   VARCHAR(500),
    width           INTEGER,
    height          INTEGER,
    duration        INTEGER,     -- seconds (video/audio)
    file_size       BIGINT,      -- bytes
    mime_type       VARCHAR(100),
    sort_order      SMALLINT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_media_post_id ON media(post_id, sort_order);

-- ─── Comments Table ──────────────────────────────────────
CREATE TABLE comments (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    post_id         UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    parent_id       UUID REFERENCES comments(id) ON DELETE CASCADE,
    content         TEXT NOT NULL,
    like_count      INTEGER NOT NULL DEFAULT 0,
    reply_count     INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ,

    CONSTRAINT comments_content_length CHECK (
        char_length(content) BETWEEN 1 AND 2000
    )
);

CREATE INDEX idx_comments_post_id ON comments(post_id, created_at ASC)
    WHERE deleted_at IS NULL;
CREATE INDEX idx_comments_user_id ON comments(user_id)
    WHERE deleted_at IS NULL;
CREATE INDEX idx_comments_parent_id ON comments(parent_id)
    WHERE parent_id IS NOT NULL;

-- ─── Likes Table ─────────────────────────────────────────
CREATE TABLE likes (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    post_id         UUID REFERENCES posts(id) ON DELETE CASCADE,
    comment_id      UUID REFERENCES comments(id) ON DELETE CASCADE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT likes_target_check CHECK (
        (post_id IS NOT NULL AND comment_id IS NULL) OR
        (post_id IS NULL AND comment_id IS NOT NULL)
    ),
    CONSTRAINT likes_unique_post UNIQUE (user_id, post_id),
    CONSTRAINT likes_unique_comment UNIQUE (user_id, comment_id)
);

CREATE INDEX idx_likes_post_id ON likes(post_id) WHERE post_id IS NOT NULL;
CREATE INDEX idx_likes_comment_id ON likes(comment_id) WHERE comment_id IS NOT NULL;

-- ─── Follows Table ───────────────────────────────────────
CREATE TABLE follows (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    follower_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    following_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    is_approved     BOOLEAN NOT NULL DEFAULT TRUE, -- FALSE สำหรับ private accounts
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT follows_no_self_follow CHECK (follower_id != following_id),
    CONSTRAINT follows_unique UNIQUE (follower_id, following_id)
);

CREATE INDEX idx_follows_follower ON follows(follower_id, created_at DESC);
CREATE INDEX idx_follows_following ON follows(following_id, created_at DESC);

-- ─── SOS Alerts Table ────────────────────────────────────
CREATE TABLE sos_alerts (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    severity        alert_severity NOT NULL DEFAULT 'medium',
    status          alert_status NOT NULL DEFAULT 'active',
    title           VARCHAR(200) NOT NULL,
    description     TEXT,
    latitude        DECIMAL(10, 8) NOT NULL,
    longitude       DECIMAL(11, 8) NOT NULL,
    address         VARCHAR(500),
    radius_km       DECIMAL(5, 2) NOT NULL DEFAULT 10.0,
    response_count  INTEGER NOT NULL DEFAULT 0,
    view_count      INTEGER NOT NULL DEFAULT 0,
    contact_phone   VARCHAR(20),
    is_anonymous    BOOLEAN NOT NULL DEFAULT FALSE,

    -- Timestamps
    resolved_at     TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ DEFAULT (NOW() + INTERVAL '24 hours'),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sos_alerts_status ON sos_alerts(status, created_at DESC)
    WHERE status = 'active';
CREATE INDEX idx_sos_alerts_location ON sos_alerts USING GIST(
    ll_to_earth(latitude, longitude)
) WHERE status = 'active';
CREATE INDEX idx_sos_alerts_user_id ON sos_alerts(user_id, created_at DESC);

-- ─── SOS Responses Table ─────────────────────────────────
CREATE TABLE sos_responses (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    alert_id        UUID NOT NULL REFERENCES sos_alerts(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    message         TEXT,
    is_on_the_way   BOOLEAN NOT NULL DEFAULT FALSE,
    latitude        DECIMAL(10, 8),
    longitude       DECIMAL(11, 8),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT sos_responses_unique UNIQUE (alert_id, user_id)
);

CREATE INDEX idx_sos_responses_alert ON sos_responses(alert_id, created_at DESC);

-- ─── Notifications Table ─────────────────────────────────
CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    type            notification_type NOT NULL,
    title           VARCHAR(200) NOT NULL,
    body            TEXT,
    data            JSONB,
    is_read         BOOLEAN NOT NULL DEFAULT FALSE,
    read_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notifications_user_id ON notifications(user_id, created_at DESC)
    WHERE is_read = FALSE;
CREATE INDEX idx_notifications_created_at ON notifications(created_at DESC);

-- ─── Updated At Trigger ──────────────────────────────────
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply ไปทุก table ที่มี updated_at
DO $$
DECLARE
    t TEXT;
BEGIN
    FOREACH t IN ARRAY ARRAY['users', 'posts', 'comments', 'sos_alerts'] LOOP
        EXECUTE format('
            CREATE TRIGGER %I_updated_at
                BEFORE UPDATE ON %I
                FOR EACH ROW EXECUTE FUNCTION update_updated_at()',
            t, t);
    END LOOP;
END;
$$;
```

### Step 32: Index Types และ Usage

```sql
-- ─── B-tree Index (default) ───────────────────────────────
-- เหมาะกับ: equality, range queries, sorting
-- ตัวอย่างใน chuaikan.com

-- Composite index สำหรับ social feed
CREATE INDEX idx_posts_feed ON posts(user_id, published_at DESC)
    WHERE deleted_at IS NULL AND status = 'published';
-- Query ที่ได้ประโยชน์:
-- SELECT * FROM posts WHERE user_id = $1 ORDER BY published_at DESC LIMIT 20;

-- Partial index (เฉพาะ active SOS alerts)
CREATE INDEX idx_sos_active ON sos_alerts(created_at DESC)
    WHERE status = 'active' AND expires_at > NOW();

-- Expression index
CREATE INDEX idx_users_lower_email ON users(LOWER(email));
-- Query: WHERE LOWER(email) = LOWER($1)

-- ─── GiST Index (Generalized Search Tree) ────────────────
-- เหมาะกับ: geometric types, full-text search, ranges
-- ใช้สำหรับ geospatial query ใน SOS

-- สร้าง earth_distance index (ต้องการ earthdistance extension)
CREATE EXTENSION IF NOT EXISTS earthdistance;
CREATE EXTENSION IF NOT EXISTS cube;

CREATE INDEX idx_sos_gist_location ON sos_alerts
    USING GIST(ll_to_earth(latitude, longitude))
    WHERE status = 'active';

-- Query หา SOS alerts ใกล้เคียง (ใน 10km)
SELECT
    id, title, severity,
    earth_distance(
        ll_to_earth(latitude, longitude),
        ll_to_earth($1, $2)    -- user's lat, lng
    ) / 1000.0 AS distance_km
FROM sos_alerts
WHERE
    status = 'active'
    AND earth_box(ll_to_earth($1, $2), 10000) @> ll_to_earth(latitude, longitude)
ORDER BY distance_km ASC
LIMIT 50;

-- ─── GIN Index (Generalized Inverted Index) ───────────────
-- เหมาะกับ: arrays, JSONB, full-text search, trigrams
-- ใช้สำหรับ full-text search

-- Full-text search index
CREATE INDEX idx_posts_gin_search ON posts USING GIN(search_vector);

-- JSONB index
CREATE INDEX idx_notifications_data ON notifications USING GIN(data);
-- Query: WHERE data @> '{"alert_id": "xxx"}'

-- Trigram index สำหรับ LIKE query
CREATE INDEX idx_users_trgm_username ON users
    USING GIN(username gin_trgm_ops);
-- Query: WHERE username ILIKE '%john%'

-- Array index
-- CREATE INDEX idx_posts_tags ON posts USING GIN(tags);
-- Query: WHERE tags @> ARRAY['emergency', 'sos']
```

### Step 33: EXPLAIN ANALYZE และ Query Optimization

```sql
-- ─── EXPLAIN ANALYZE ──────────────────────────────────────
-- วิธีอ่าน EXPLAIN output:
-- Seq Scan → full table scan (ช้า)
-- Index Scan → ใช้ index (ดี)
-- Index Only Scan → ดึงข้อมูลจาก index เท่านั้น (ดีที่สุด)
-- Nested Loop / Hash Join / Merge Join → join strategies
-- cost=startup..total rows=estimate width=bytes
-- actual time=startup..total rows=actual loops=

-- ─── ตัวอย่าง: Social Feed Query ────────────────────────
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT
    p.id,
    p.content,
    p.like_count,
    p.comment_count,
    p.created_at,
    u.id AS author_id,
    u.username,
    u.display_name,
    u.avatar_url
FROM posts p
JOIN users u ON p.user_id = u.id
WHERE
    p.user_id IN (
        SELECT following_id
        FROM follows
        WHERE follower_id = 'user-uuid-here'
          AND is_approved = TRUE
    )
    AND p.status = 'published'
    AND p.deleted_at IS NULL
    AND p.created_at > NOW() - INTERVAL '7 days'
ORDER BY p.created_at DESC
LIMIT 20
OFFSET 0;

-- ─── อ่านผล EXPLAIN ──────────────────────────────────────
-- ก่อน optimize:
-- Seq Scan on posts (cost=0..50000 rows=50000 width=200)
--   actual time=0.050..1200 rows=1000 loops=1
-- → ช้ามาก! ต้อง scan ทั้ง table

-- หลัง add index:
-- Index Scan using idx_posts_feed on posts
--   actual time=0.050..5.2 rows=20 loops=1
-- → เร็วขึ้น 230x!

-- ─── Optimization Techniques ──────────────────────────────
-- 1. ใช้ CTE สำหรับ complex queries
WITH follower_ids AS MATERIALIZED (
    SELECT following_id
    FROM follows
    WHERE follower_id = $1 AND is_approved = TRUE
)
SELECT p.*, u.username, u.avatar_url
FROM posts p
JOIN users u ON p.user_id = u.id
WHERE p.user_id = ANY(SELECT following_id FROM follower_ids)
  AND p.status = 'published'
  AND p.deleted_at IS NULL
ORDER BY p.created_at DESC
LIMIT 20;

-- 2. ใช้ EXISTS แทน IN (เร็วกว่าเมื่อ subquery ใหญ่)
SELECT p.*
FROM posts p
WHERE EXISTS (
    SELECT 1 FROM follows f
    WHERE f.follower_id = $1
      AND f.following_id = p.user_id
      AND f.is_approved = TRUE
)
AND p.status = 'published'
ORDER BY p.created_at DESC
LIMIT 20;

-- 3. Pagination ด้วย cursor (เร็วกว่า OFFSET)
-- ช้า: OFFSET 1000 ต้อง skip 1000 rows
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 1000;

-- เร็ว: cursor-based pagination
SELECT * FROM posts
WHERE created_at < $1  -- last_cursor = created_at ของ item สุดท้าย
ORDER BY created_at DESC
LIMIT 20;
```

### Step 34: PgBouncer Setup

```bash
# ─── ติดตั้งและ configure PgBouncer ──────────────────────
sudo apt install -y pgbouncer

# ─── pgbouncer.ini configuration ─────────────────────────
sudo tee /etc/pgbouncer/pgbouncer.ini << 'EOF'
[databases]
; Database alias = connection string
chuaikan_db = host=127.0.0.1 port=5432 dbname=chuaikan_db

[pgbouncer]
; ─── Connection Settings ────────────────────────────────
listen_addr = 127.0.0.1
listen_port = 6432
unix_socket_dir = /var/run/postgresql

; ─── Pool Mode ──────────────────────────────────────────
; transaction: คืน connection กลับหลังจบ transaction (แนะนำ)
; session: คืน connection เมื่อ client disconnect
; statement: คืนหลังทุก statement (ไม่รองรับ prepared statements)
pool_mode = transaction

; ─── Pool Sizes ─────────────────────────────────────────
; Max connections ไป PostgreSQL (ควรน้อยกว่า max_connections)
max_client_conn = 1000          ; max clients ที่ต่อเข้า PgBouncer ได้
default_pool_size = 25          ; connections ต่อ database+user pair
min_pool_size = 5               ; minimum connections ไว้ warm
reserve_pool_size = 5           ; extra connections เมื่อ pool เต็ม
reserve_pool_timeout = 5        ; seconds ก่อน error ถ้า reserve ไม่พอ

; ─── Authentication ─────────────────────────────────────
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

; ─── Timeouts ───────────────────────────────────────────
server_idle_timeout = 600       ; ปิด idle server connections หลัง 10 min
client_idle_timeout = 0         ; 0 = no timeout สำหรับ clients
server_connect_timeout = 15
server_login_retry = 15
query_timeout = 0               ; 0 = no timeout

; ─── Logging ────────────────────────────────────────────
log_connections = 0
log_disconnections = 0
log_pooler_errors = 1
stats_period = 60

; ─── Admin ──────────────────────────────────────────────
admin_users = pgbouncer_admin
stats_users = pgbouncer_stats

; ─── Performance ────────────────────────────────────────
server_reset_query = DISCARD ALL
server_check_query = SELECT 1
server_check_delay = 30
ignore_startup_parameters = extra_float_digits,application_name
EOF

# ─── สร้าง userlist.txt ───────────────────────────────────
# ต้องใช้ password hash (ไม่ใช่ plain text)
# สร้าง hash ด้วย Python:
python3 -c "
import hashlib
import hmac
import base64
import os

password = 'strong_password_here'
username = 'chuaikan_user'

# SCRAM-SHA-256 hash
salt = os.urandom(16)
salt_b64 = base64.b64encode(salt).decode()
print(f'\"chuaikan_user\" \"SCRAM-SHA-256\${salt_b64}\"')
print('(Use pgbouncer auth_query for production)')
"

# วิธีง่ายกว่า: ดึง hash จาก PostgreSQL
sudo -u postgres psql -c "SELECT usename, passwd FROM pg_shadow WHERE usename = 'chuaikan_user';"
# คัดลอก hash มาใส่ใน userlist.txt:
sudo tee /etc/pgbouncer/userlist.txt << 'EOF'
"chuaikan_user" "SCRAM-SHA-256$4096:salt_here$hash_here"
EOF

sudo chmod 640 /etc/pgbouncer/userlist.txt
sudo chown postgres:postgres /etc/pgbouncer/userlist.txt

# Start PgBouncer
sudo systemctl enable pgbouncer
sudo systemctl start pgbouncer

# ทดสอบ connection ผ่าน PgBouncer
psql -h 127.0.0.1 -p 6432 -U chuaikan_user -d chuaikan_db -c "SELECT version();"

# ดู stats
psql -h 127.0.0.1 -p 6432 -U pgbouncer_admin pgbouncer -c "SHOW POOLS;"
psql -h 127.0.0.1 -p 6432 -U pgbouncer_admin pgbouncer -c "SHOW STATS;"
```

### Step 35: VACUUM และ ANALYZE Maintenance

```sql
-- ─── Manual VACUUM ───────────────────────────────────────
-- VACUUM: cleanup dead tuples, ไม่คืน space ให้ OS
VACUUM VERBOSE posts;

-- VACUUM FULL: คืน space ให้ OS (lock table! ใช้ maintenance window)
VACUUM FULL posts;

-- VACUUM ANALYZE: cleanup + update statistics
VACUUM ANALYZE posts;
VACUUM ANALYZE;     -- ทุก tables

-- ─── ANALYZE ─────────────────────────────────────────────
-- Update query planner statistics
ANALYZE posts;
ANALYZE;            -- ทุก tables

-- ─── ตรวจสอบ bloat ───────────────────────────────────────
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
    n_dead_tup,
    n_live_tup,
    ROUND(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_pct,
    last_vacuum,
    last_autovacuum,
    last_analyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

```bash
# ─── ตั้งค่า autovacuum ใน postgresql.conf ────────────────
sudo tee -a /etc/postgresql/17/main/conf.d/autovacuum.conf << 'EOF'
# Autovacuum settings
autovacuum = on
autovacuum_max_workers = 4
autovacuum_naptime = 30s
autovacuum_vacuum_threshold = 50
autovacuum_vacuum_scale_factor = 0.02   # 2% ของ table
autovacuum_analyze_threshold = 50
autovacuum_analyze_scale_factor = 0.01  # 1% ของ table
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = 400
EOF
```

### Step 36: postgresql.conf Tuning

```bash
# ─── postgresql.conf สำหรับ 8 CPU / 16GB RAM Server ──────
sudo tee /etc/postgresql/17/main/conf.d/tuning.conf << 'EOF'
# ═══════════════════════════════════════════════════════════
# PostgreSQL 17 Production Tuning
# Server: 8 CPU cores, 16GB RAM, SSD storage
# Role: Primary OLTP (chuaikan.com social media)
# Generated: 2024-09-30
# ═══════════════════════════════════════════════════════════

# ─── Memory ─────────────────────────────────────────────
# shared_buffers: 25% of RAM = 4GB
shared_buffers = 4GB

# effective_cache_size: 75% of RAM (hint for query planner)
effective_cache_size = 12GB

# work_mem: RAM per sort/hash operation
# Formula: (RAM - shared_buffers) / (max_connections * 2)
# (16GB - 4GB) / (200 * 2) = 30MB
work_mem = 32MB

# maintenance_work_mem: RAM for VACUUM, CREATE INDEX
maintenance_work_mem = 1GB

# huge_pages: ใช้ huge pages ถ้า OS support
huge_pages = try

# ─── WAL / Checkpoints ───────────────────────────────────
# wal_buffers: WAL buffer size (auto ถ้า -1)
wal_buffers = 64MB

# checkpoint_completion_target: spread checkpoint I/O
checkpoint_completion_target = 0.9

# max_wal_size: ขนาด WAL ก่อน trigger checkpoint
max_wal_size = 4GB
min_wal_size = 1GB

# ─── Connections ─────────────────────────────────────────
# max_connections: ใช้ PgBouncer จะลด connections ได้
max_connections = 200

# superuser_reserved_connections: สำรองไว้สำหรับ admin
superuser_reserved_connections = 3

# ─── Query Planner ──────────────────────────────────────
# สำหรับ SSD ลด random_page_cost
random_page_cost = 1.1
seq_page_cost = 1.0

# เพิ่ม parallelism
max_parallel_workers_per_gather = 4
max_parallel_workers = 8
max_worker_processes = 16

# ─── I/O ────────────────────────────────────────────────
effective_io_concurrency = 200  # SSD

# ─── Logging ────────────────────────────────────────────
log_destination = 'csvlog'
logging_collector = on
log_directory = '/var/log/postgresql'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = 1d
log_rotation_size = 100MB
log_min_duration_statement = 1000  # log queries > 1 second
log_checkpoints = on
log_connections = off
log_disconnections = off
log_lock_waits = on
log_temp_files = 10MB
log_autovacuum_min_duration = 250ms

# ─── Statistics ─────────────────────────────────────────
track_activities = on
track_counts = on
track_io_timing = on
track_functions = all
track_activity_query_size = 2048

# ─── Extensions ─────────────────────────────────────────
shared_preload_libraries = 'pg_stat_statements,auto_explain'

# pg_stat_statements
pg_stat_statements.max = 5000
pg_stat_statements.track = all

# auto_explain (log slow queries with EXPLAIN)
auto_explain.log_min_duration = 1000
auto_explain.log_analyze = on
auto_explain.log_buffers = on
auto_explain.log_nested_statements = on
EOF

# Reload PostgreSQL
sudo systemctl reload postgresql
sudo -u postgres psql -c "SELECT pg_reload_conf();"
```

### Step 37: pg_dump Backup และ Restore

```bash
# ─── pg_dump Backup ───────────────────────────────────────
# Custom format (-Fc) — แนะนำสุด
pg_dump \
    -U chuaikan_user \
    -h localhost \
    -d chuaikan_db \
    --format=custom \
    --compress=9 \
    --verbose \
    --file=/var/backups/chuaikan/db/chuaikan_$(date +%Y%m%d_%H%M%S).dump

# Plain SQL format (human-readable, ใหญ่กว่า)
pg_dump -U chuaikan_user chuaikan_db > /tmp/backup.sql

# Gzipped SQL
pg_dump -U chuaikan_user chuaikan_db | gzip > /tmp/backup.sql.gz

# ─── Backup เฉพาะ Schema ──────────────────────────────────
pg_dump -U chuaikan_user -d chuaikan_db \
    --schema-only \
    --file=/var/backups/chuaikan/schema_$(date +%Y%m%d).sql

# ─── Backup เฉพาะ Table ───────────────────────────────────
pg_dump -U chuaikan_user -d chuaikan_db \
    --table=users --table=posts \
    --format=custom \
    --file=/var/backups/chuaikan/users_posts.dump

# ─── Restore ──────────────────────────────────────────────
# Restore custom format
pg_restore \
    -U chuaikan_user \
    -h localhost \
    -d chuaikan_db \
    --verbose \
    --clean \                    # drop objects before create
    --if-exists \               # ไม่ error ถ้าไม่มี object
    /var/backups/chuaikan/db/chuaikan_20240930.dump

# Restore เฉพาะ table
pg_restore \
    -U chuaikan_user \
    -d chuaikan_db \
    --table=users \
    /var/backups/chuaikan/db/chuaikan_20240930.dump

# Restore SQL format
psql -U chuaikan_user -d chuaikan_db < /tmp/backup.sql

# ─── pg_basebackup (full cluster backup) ─────────────────
sudo -u postgres pg_basebackup \
    --pgdata=/var/backups/chuaikan/basebackup \
    --format=tar \
    --gzip \
    --compress=9 \
    --progress \
    --verbose
```

### Step 38: pg_stat_statements

```sql
-- ─── Enable pg_stat_statements ───────────────────────────
-- (ต้องเพิ่มใน shared_preload_libraries ก่อน)
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- ─── Top 10 slowest queries ──────────────────────────────
SELECT
    LEFT(query, 100) AS query_snippet,
    calls,
    ROUND(total_exec_time::numeric, 2) AS total_ms,
    ROUND(mean_exec_time::numeric, 2) AS mean_ms,
    ROUND(stddev_exec_time::numeric, 2) AS stddev_ms,
    rows,
    ROUND(rows::numeric / calls, 2) AS rows_per_call,
    100.0 * shared_blks_hit /
        NULLIF(shared_blks_hit + shared_blks_read, 0) AS cache_hit_pct
FROM pg_stat_statements
WHERE calls > 100                     -- execute อย่างน้อย 100 ครั้ง
ORDER BY mean_exec_time DESC
LIMIT 10;

-- ─── Queries ที่ใช้ CPU มากที่สุด ─────────────────────────
SELECT
    LEFT(query, 100) AS query_snippet,
    calls,
    ROUND(total_exec_time::numeric / 1000, 2) AS total_seconds,
    ROUND((total_exec_time / SUM(total_exec_time) OVER ()) * 100, 2) AS pct_total
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- ─── Reset statistics ─────────────────────────────────────
SELECT pg_stat_statements_reset();

-- ─── Cache hit rate ──────────────────────────────────────
SELECT
    sum(heap_blks_read) AS heap_read,
    sum(heap_blks_hit) AS heap_hit,
    ROUND(100.0 * sum(heap_blks_hit) /
        NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0), 2) AS cache_hit_rate
FROM pg_statio_user_tables;
-- ควร > 99% สำหรับ production
```

### Step 39: Common Query Patterns

```sql
-- ─── Social Feed Query (Optimized) ───────────────────────
-- ดึง feed ของ user ที่ follow คนอื่น
CREATE OR REPLACE FUNCTION get_user_feed(
    p_user_id UUID,
    p_cursor TIMESTAMPTZ DEFAULT NOW(),
    p_limit INTEGER DEFAULT 20
)
RETURNS TABLE (
    post_id UUID,
    content TEXT,
    like_count INTEGER,
    comment_count INTEGER,
    created_at TIMESTAMPTZ,
    author_id UUID,
    author_username VARCHAR,
    author_avatar VARCHAR
) AS $$
BEGIN
    RETURN QUERY
    SELECT
        p.id,
        p.content,
        p.like_count,
        p.comment_count,
        p.created_at,
        u.id,
        u.username,
        u.avatar_url
    FROM posts p
    JOIN users u ON p.user_id = u.id
    WHERE p.user_id = ANY(
        SELECT following_id FROM follows
        WHERE follower_id = p_user_id AND is_approved = TRUE
    )
    AND p.status = 'published'
    AND p.deleted_at IS NULL
    AND p.created_at < p_cursor
    ORDER BY p.created_at DESC
    LIMIT p_limit;
END;
$$ LANGUAGE plpgsql STABLE;

-- ใช้:
SELECT * FROM get_user_feed('user-uuid', NOW(), 20);

-- ─── SOS Nearby Query ─────────────────────────────────────
CREATE OR REPLACE FUNCTION get_nearby_sos(
    p_lat DECIMAL,
    p_lng DECIMAL,
    p_radius_km DECIMAL DEFAULT 10.0
)
RETURNS TABLE (
    alert_id UUID,
    title VARCHAR,
    severity alert_severity,
    distance_km DECIMAL,
    created_at TIMESTAMPTZ
) AS $$
BEGIN
    RETURN QUERY
    SELECT
        id,
        title,
        severity,
        ROUND(
            earth_distance(
                ll_to_earth(latitude, longitude),
                ll_to_earth(p_lat, p_lng)
            ) / 1000.0
        , 2)::DECIMAL,
        created_at
    FROM sos_alerts
    WHERE
        status = 'active'
        AND expires_at > NOW()
        AND earth_box(ll_to_earth(p_lat, p_lng), p_radius_km * 1000)
            @> ll_to_earth(latitude, longitude)
    ORDER BY
        earth_distance(
            ll_to_earth(latitude, longitude),
            ll_to_earth(p_lat, p_lng)
        )
    LIMIT 50;
END;
$$ LANGUAGE plpgsql STABLE;
```

---

## 🔧 Configuration Files

### pg_hba.conf (Authentication)

```bash
sudo tee /etc/postgresql/17/main/pg_hba.conf << 'EOF'
# TYPE  DATABASE        USER            ADDRESS         METHOD
# Local connections
local   all             postgres                        peer
local   all             all                             scram-sha-256
# IPv4 local connections
host    all             all             127.0.0.1/32    scram-sha-256
# IPv6 local connections
host    all             all             ::1/128         scram-sha-256
# Reject remote connections (ใช้ PgBouncer แทน)
# host  chuaikan_db     chuaikan_user   10.0.0.0/8      scram-sha-256
EOF

sudo systemctl reload postgresql
```

---

## 🧪 Testing

```bash
# ─── ทดสอบ Connection ─────────────────────────────────────
psql -h localhost -U chuaikan_user -d chuaikan_db -c "SELECT version();"

# ─── ทดสอบ Schema ─────────────────────────────────────────
psql -U chuaikan_user -d chuaikan_db << 'SQL'
-- ทดสอบ insert
INSERT INTO users (username, email, password_hash, display_name)
VALUES ('testuser', 'test@chuaikan.com', 'hash', 'Test User')
RETURNING id, username;

-- ทดสอบ index usage
EXPLAIN (ANALYZE, FORMAT TEXT)
SELECT * FROM users WHERE username = 'testuser';
-- Expected: Index Scan using idx_users_username

-- ทดสอบ pg_stat_statements
SELECT query, calls FROM pg_stat_statements
WHERE query LIKE '%users%' LIMIT 5;
SQL

# ─── ทดสอบ PgBouncer ──────────────────────────────────────
psql -h 127.0.0.1 -p 6432 -U chuaikan_user -d chuaikan_db \
  -c "SELECT pg_backend_pid();" -c "SELECT pg_backend_pid();"
# ถ้าใช้ transaction mode: PID ต่างกัน = connection reuse

# ─── ทดสอบ Backup ─────────────────────────────────────────
pg_dump -U chuaikan_user -d chuaikan_db --format=custom \
  --file=/tmp/test_backup.dump
pg_restore --list /tmp/test_backup.dump | head -20
```

---

## ❌ Common Errors & Solutions

### Error 1: Too many connections

```
❌ Error: FATAL: remaining connection slots are reserved for
non-replication superuser connections

✅ Solution:
# ตรวจสอบ connections
sudo -u postgres psql -c "SELECT count(*), state FROM pg_stat_activity GROUP BY state;"

# เพิ่ม max_connections หรือ (แนะนำ) ใช้ PgBouncer
# PgBouncer: max_client_conn = 1000 → PostgreSQL: max_connections = 100
```

### Error 2: Slow queries หลังจาก data เยอะขึ้น

```
❌ Error: Query ช้าลงเรื่อยๆ

✅ Solution:
-- Step 1: หา slow queries
SELECT mean_exec_time, query FROM pg_stat_statements
ORDER BY mean_exec_time DESC LIMIT 5;

-- Step 2: EXPLAIN ANALYZE query นั้น
EXPLAIN (ANALYZE, BUFFERS) <your query>;

-- Step 3: ดูว่า Seq Scan มีไหม → add index
-- Step 4: Run ANALYZE
ANALYZE;
```

### Error 3: Disk เต็มจาก WAL files

```
❌ Error: PANIC: could not write to file "pg_wal/xxx"
No space left on device

✅ Solution:
# ตรวจสอบ disk
df -h /var/lib/postgresql

# ลด WAL size ชั่วคราว
sudo -u postgres psql -c "CHECKPOINT;"
sudo -u postgres psql -c "SELECT pg_switch_wal();"

# ระยะยาว: เพิ่ม disk หรือ archive WAL
max_wal_size = 2GB  # ลด ถ้า disk ไม่พอ
```

---

## 📊 Performance Benchmark

| ตัวชี้วัด | Default Config | Tuned Config |
|---------|---------------|-------------|
| shared_buffers | 128MB | 4GB |
| Cache hit rate | 75% | 99%+ |
| Connections (PgBouncer) | 100 | 1000 |
| Feed query time | 850ms | 12ms |
| SOS nearby query | 1200ms | 8ms |
| VACUUM speed | slow | 4x faster |

---

## ✅ Checklist

- [ ] ติดตั้ง PostgreSQL 17
- [ ] สร้าง database และ user
- [ ] ตั้งค่า pg_hba.conf
- [ ] Run schema.sql (tables, indexes, triggers)
- [ ] Enable extensions (uuid-ossp, pg_trgm, earthdistance)
- [ ] ตั้งค่า postgresql.conf (memory, WAL, logging)
- [ ] ติดตั้งและ configure PgBouncer
- [ ] Enable pg_stat_statements
- [ ] ทดสอบ EXPLAIN ANALYZE บน queries หลัก
- [ ] ตั้ง cron สำหรับ backup ด้วย pg_dump
- [ ] ทดสอบ restore จาก backup
- [ ] ตรวจสอบ cache hit rate > 99%
- [ ] ตั้งค่า autovacuum
- [ ] ทดสอบ geospatial query (SOS nearby)

---

## 🔗 References

- [PostgreSQL 17 Documentation](https://www.postgresql.org/docs/17/)
- [PgBouncer Documentation](https://www.pgbouncer.org/config.html)
- [pgTune Calculator](https://pgtune.leopard.in.ua/)
- [EXPLAIN Visualizer](https://explain.dalibo.com/)
- [PostgreSQL Indexes](https://www.postgresql.org/docs/current/indexes.html)
- [pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html)

---
*Part 004 | Road to 1,000,000 Users/Day | chuaikan.com*
