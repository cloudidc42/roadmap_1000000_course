# Part 044: PostGIS สำหรับ Geospatial Data
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 431–440
> **เวลาโดยประมาณ:** 3 ชั่วโมง
> **Prerequisites:** Part 043 (TimescaleDB), PostgreSQL พื้นฐาน

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ติดตั้ง PostGIS extension
- เข้าใจ Geometry vs Geography types
- Import ข้อมูล Polygon จังหวัดไทย
- Query flood zone polygons (ST_Intersects, ST_Contains)
- Query users ใกล้เคียง (ST_DWithin)
- คำนวณระยะทาง (ST_Distance)
- Output เป็น GeoJSON (ST_AsGeoJSON)
- สร้าง Spatial Index (GIST)
- Integrate กับ SOS service

---

## 📖 ทฤษฎีและแนวคิด

### Step 431 — ทำไมต้อง PostGIS?

```
คำถามที่ต้องตอบใน chuaikan.com:
1. "เซ็นเซอร์ A อยู่ในจังหวัดอะไร?"
2. "มีพื้นที่น้ำท่วมอยู่ในรัศมี 10 กม. จากที่ user อยู่ไหม?"
3. "SOS report ใกล้ที่สุดอยู่ห่างแค่ไหน?"
4. "User ที่อยู่ในเขตน้ำท่วมมีกี่คน?"

ด้วย PostGIS:
- ST_DWithin: หา objects ในรัศมี (ใช้ spatial index)
- ST_Intersects: ตรวจว่า polygon ทับซ้อนกัน
- ST_Contains: ตรวจว่า point อยู่ใน polygon
- ST_Distance: คำนวณระยะทางจริง (บนผิวโลก)
```

### Step 432 — Geometry vs Geography

```
Geometry (Planar):
- คำนวณบนระนาบ 2D (Cartesian coordinates)
- ค่า unit = หน่วยของ projection (degree หรือ meter)
- เร็วกว่า แต่ไม่แม่นยำสำหรับระยะทางไกล
- ใช้เมื่อ: พื้นที่เล็ก (ในจังหวัดเดียว) หรือต้องการ performance

Geography (Spheroidal):
- คำนวณบนทรงกลม (WGS84 ellipsoid)
- ค่า unit = เมตรเสมอ
- ช้ากว่า แต่แม่นยำสำหรับระยะทางไกล
- ใช้เมื่อ: ต้องการระยะทางจริงบนผิวโลก

สำหรับ chuaikan.com:
- ใช้ Geography สำหรับ user location และ ST_DWithin (ต้องการ meter)
- ใช้ Geometry สำหรับ flood zone polygons (ต้องการ spatial operations)
```

---

## ⚙️ Environment Setup

### ติดตั้ง PostGIS บน Ubuntu 24.04

```bash
# ติดตั้ง PostGIS
sudo apt-get install -y \
  postgresql-16-postgis-3 \
  postgresql-16-postgis-3-scripts \
  postgis \
  gdal-bin \
  python3-gdal

# สร้าง extension ใน database
sudo -u postgres psql -d chuaikan_db << 'EOF'
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS postgis_topology;
CREATE EXTENSION IF NOT EXISTS fuzzystrmatch;
CREATE EXTENSION IF NOT EXISTS postgis_tiger_geocoder CASCADE;

-- ตรวจสอบ version
SELECT postgis_version();
SELECT postgis_full_version();
EOF
```

### Docker

```bash
docker run -d \
  --name postgis \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=gis_secret \
  -e POSTGRES_DB=chuaikan_db \
  -v postgis_data:/var/lib/postgresql/data \
  postgis/postgis:16-3.4

docker exec postgis psql -U postgres -d chuaikan_db -c \
  "SELECT postgis_version();"
```

---

## 🛠️ Step-by-Step Implementation

### Step 433 — สร้าง Tables พร้อม Spatial Columns

```sql
-- เชื่อมต่อ database
\c chuaikan_db

-- ตารางสถานีเซ็นเซอร์
CREATE TABLE sensor_stations (
    id          SERIAL PRIMARY KEY,
    station_id  VARCHAR(50) UNIQUE NOT NULL,
    name        VARCHAR(200),
    province_id INTEGER,
    location    GEOGRAPHY(POINT, 4326) NOT NULL,  -- WGS84
    altitude_m  DECIMAL(8, 2),
    installed_at TIMESTAMPTZ DEFAULT NOW()
);

-- GIST index สำหรับ spatial queries
CREATE INDEX idx_stations_location
    ON sensor_stations USING GIST (location);

-- ตาราง flood zones (พื้นที่เสี่ยงน้ำท่วม)
CREATE TABLE flood_zones (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(200),
    risk_level  VARCHAR(20)     -- 'low', 'medium', 'high', 'critical'
                CHECK (risk_level IN ('low', 'medium', 'high', 'critical')),
    province_id INTEGER,
    boundary    GEOMETRY(POLYGON, 4326) NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    updated_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_flood_zones_boundary
    ON flood_zones USING GIST (boundary);

-- ตาราง Thai provinces
CREATE TABLE thai_provinces (
    id          SERIAL PRIMARY KEY,
    code        VARCHAR(10) UNIQUE,
    name_th     VARCHAR(100) NOT NULL,
    name_en     VARCHAR(100),
    region      VARCHAR(50),
    boundary    GEOMETRY(MULTIPOLYGON, 4326),
    centroid    GEOGRAPHY(POINT, 4326) GENERATED ALWAYS AS (
                    ST_GeogFromWKB(ST_AsEWKB(ST_Centroid(boundary)))
                ) STORED
);

CREATE INDEX idx_provinces_boundary
    ON thai_provinces USING GIST (boundary);

-- ตาราง users พร้อม location
ALTER TABLE users ADD COLUMN IF NOT EXISTS
    location GEOGRAPHY(POINT, 4326);

ALTER TABLE users ADD COLUMN IF NOT EXISTS
    province_id INTEGER REFERENCES thai_provinces(id);

CREATE INDEX idx_users_location
    ON users USING GIST (location);
```

### Step 434 — Import ข้อมูลจังหวัดไทย

```bash
# ดาวน์โหลด shapefile จังหวัดไทย
wget https://data.humdata.org/dataset/tha-administrative-divisions-shapefiles/resource/\
  tha_adm1_rtsd_itos_20210121_SHP.zip \
  -O /tmp/thai_provinces.zip

cd /tmp && unzip thai_provinces.zip

# ตรวจสอบ shapefile
ogrinfo -al -so /tmp/tha_adm1_rtsd_itos_20210121.shp | head -30

# Import ด้วย shp2pgsql
shp2pgsql \
  -s 4326 \           # SRID = WGS84
  -I \                # สร้าง spatial index อัตโนมัติ
  -W UTF-8 \          # encoding
  /tmp/tha_adm1_rtsd_itos_20210121.shp \
  thai_provinces_raw | \
  psql -U postgres -d chuaikan_db

# หรือใช้ ogr2ogr (ยืดหยุ่นกว่า)
ogr2ogr \
  -f "PostgreSQL" \
  PG:"host=localhost dbname=chuaikan_db user=postgres password=secret" \
  /tmp/tha_adm1_rtsd_itos_20210121.shp \
  -nln thai_provinces_import \
  -t_srs EPSG:4326 \
  -overwrite
```

```sql
-- แปลงข้อมูลจาก import table ไป thai_provinces
INSERT INTO thai_provinces (code, name_th, name_en, boundary)
SELECT
    adm1_pcode AS code,
    adm1_th AS name_th,
    adm1_en AS name_en,
    ST_Multi(wkb_geometry)::GEOMETRY(MULTIPOLYGON, 4326) AS boundary
FROM thai_provinces_import;

-- ตรวจสอบ
SELECT code, name_th, name_en,
    ST_AsText(ST_Centroid(boundary)) AS centroid_wkt
FROM thai_provinces
LIMIT 5;

-- ใส่ province_id ให้ sensor_stations
UPDATE sensor_stations ss
SET province_id = p.id
FROM thai_provinces p
WHERE ST_Within(
    ss.location::geometry,
    p.boundary
);
```

### Step 435 — Flood Zone Polygon Queries

```sql
-- ใส่ตัวอย่าง flood zones
INSERT INTO flood_zones (name, risk_level, boundary)
VALUES
  -- พื้นที่น้ำท่วมบางปะกง (ตัวอย่าง)
  (
    'น้ำท่วมบางปะกง 2024',
    'critical',
    ST_GeomFromText(
      'POLYGON((101.0 13.5, 101.5 13.5, 101.5 14.0, 101.0 14.0, 101.0 13.5))',
      4326
    )
  ),
  -- พื้นที่เสี่ยงเชียงราย
  (
    'เสี่ยงน้ำท่วมเชียงราย',
    'high',
    ST_GeomFromText(
      'POLYGON((99.5 19.5, 100.5 19.5, 100.5 20.5, 99.5 20.5, 99.5 19.5))',
      4326
    )
  );

-- Query 1: เซ็นเซอร์ที่อยู่ในพื้นที่น้ำท่วม
SELECT
    ss.station_id,
    ss.name,
    fz.name AS flood_zone,
    fz.risk_level
FROM sensor_stations ss
JOIN flood_zones fz ON ST_Intersects(
    ss.location::geometry,
    fz.boundary
)
WHERE fz.risk_level IN ('critical', 'high');

-- Query 2: พื้นที่น้ำท่วมที่ครอบคลุมจังหวัดใด
SELECT
    fz.name AS flood_zone,
    fz.risk_level,
    p.name_th AS province,
    ROUND(
        ST_Area(ST_Intersection(fz.boundary, p.boundary)::geography) / 1e6,
        2
    ) AS overlap_km2
FROM flood_zones fz
JOIN thai_provinces p ON ST_Intersects(fz.boundary, p.boundary)
ORDER BY overlap_km2 DESC;

-- Query 3: Users ที่อยู่ในพื้นที่น้ำท่วม critical
SELECT
    u.id,
    u.username,
    fz.name AS danger_zone,
    ST_Distance(u.location, ST_Centroid(fz.boundary)::geography) AS dist_to_center_m
FROM users u
JOIN flood_zones fz ON ST_Intersects(
    u.location::geometry,
    fz.boundary
)
WHERE fz.risk_level = 'critical'
ORDER BY dist_to_center_m;
```

### Step 436 — Nearby Users Query (ST_DWithin)

```sql
-- หา users ในรัศมี 5 กม. จาก lat/lng ที่กำหนด
-- ใช้ Geography เพื่อความแม่นยำ (หน่วยเป็นเมตร)
SELECT
    u.id,
    u.username,
    ROUND(
        ST_Distance(
            u.location,
            ST_MakePoint(100.5018, 13.7563)::geography  -- กรุงเทพฯ
        )::numeric,
        2
    ) AS distance_m
FROM users u
WHERE ST_DWithin(
    u.location,
    ST_MakePoint(100.5018, 13.7563)::geography,
    5000  -- 5,000 เมตร = 5 กม.
)
ORDER BY distance_m
LIMIT 50;

-- หา SOS reports ใกล้ sensor ที่ระดับน้ำสูง
SELECT
    sr.id,
    sr.user_id,
    sr.description,
    sr.created_at,
    ROUND(
        ST_Distance(
            sr.location,
            ss.location
        )::numeric,
        2
    ) AS dist_from_sensor_m
FROM sos_reports sr
CROSS JOIN sensor_stations ss
WHERE ss.station_id = 'SENSOR_0001'
  AND ST_DWithin(sr.location, ss.location, 10000)  -- 10 กม.
  AND sr.created_at > NOW() - INTERVAL '24 hours'
ORDER BY dist_from_sensor_m;
```

### Step 437 — Distance Calculation (ST_Distance)

```sql
-- ระยะทางระหว่าง 2 จุด (เมตร)
SELECT ST_Distance(
    ST_MakePoint(100.5018, 13.7563)::geography,  -- กรุงเทพ
    ST_MakePoint(98.9853, 18.7883)::geography    -- เชียงใหม่
) AS distance_meters;
-- ≈ 696,000 เมตร = 696 กม.

-- ระยะทางจาก road/linestring
CREATE TABLE evacuation_routes (
    id      SERIAL PRIMARY KEY,
    name    VARCHAR(200),
    route   GEOMETRY(LINESTRING, 4326)
);

-- หา evacuation route ที่ใกล้ user มากที่สุด
SELECT
    er.name,
    ST_Distance(
        u.location,
        er.route::geography
    ) AS dist_to_route_m,
    ST_ClosestPoint(er.route, u.location::geometry) AS nearest_point
FROM users u
CROSS JOIN evacuation_routes er
WHERE u.id = 42
ORDER BY dist_to_route_m
LIMIT 3;
```

### Step 438 — GeoJSON Output (ST_AsGeoJSON)

```sql
-- Output เป็น GeoJSON Feature
SELECT json_build_object(
    'type', 'Feature',
    'geometry', ST_AsGeoJSON(location)::json,
    'properties', json_build_object(
        'id', id,
        'name', station_id,
        'altitude', altitude_m
    )
) AS feature
FROM sensor_stations
LIMIT 5;

-- Output เป็น GeoJSON FeatureCollection
SELECT json_build_object(
    'type', 'FeatureCollection',
    'features', json_agg(
        json_build_object(
            'type', 'Feature',
            'geometry', ST_AsGeoJSON(boundary)::json,
            'properties', json_build_object(
                'name', name,
                'risk_level', risk_level
            )
        )
    )
) AS geojson
FROM flood_zones
WHERE risk_level IN ('high', 'critical');
```

### Step 439 — API Integration สำหรับ SOS Service

```typescript
// api/routes/sos.ts
import { Router } from 'express';
import { prisma } from '../db/client';

const router = Router();

// GET /api/sos/nearby - SOS reports ในรัศมี
router.get('/nearby', async (req, res) => {
  const { lat, lng, radius = 5000 } = req.query;

  if (!lat || !lng) {
    return res.status(400).json({ error: 'lat and lng required' });
  }

  try {
    const reports = await prisma.$queryRaw<any[]>`
      SELECT
        sr.id,
        sr.description,
        sr.created_at,
        sr.status,
        ST_AsGeoJSON(sr.location)::json AS location,
        ROUND(ST_Distance(
          sr.location,
          ST_MakePoint(${parseFloat(lng as string)}, ${parseFloat(lat as string)})::geography
        )::numeric, 2) AS distance_m
      FROM sos_reports sr
      WHERE ST_DWithin(
        sr.location,
        ST_MakePoint(${parseFloat(lng as string)}, ${parseFloat(lat as string)})::geography,
        ${parseInt(radius as string)}
      )
      AND sr.created_at > NOW() - INTERVAL '24 hours'
      ORDER BY distance_m
      LIMIT 50
    `;

    res.json({
      type: 'FeatureCollection',
      features: reports.map(r => ({
        type: 'Feature',
        geometry: r.location,
        properties: {
          id: r.id,
          description: r.description,
          createdAt: r.created_at,
          status: r.status,
          distanceM: r.distance_m,
        },
      })),
    });
  } catch (error) {
    res.status(500).json({ error: String(error) });
  }
});

// GET /api/flood-zones/check - ตรวจสอบว่า location อยู่ใน flood zone
router.get('/flood-zones/check', async (req, res) => {
  const { lat, lng } = req.query;

  const zones = await prisma.$queryRaw<any[]>`
    SELECT
      id,
      name,
      risk_level,
      ST_AsGeoJSON(boundary)::json AS geometry
    FROM flood_zones
    WHERE ST_Contains(
      boundary,
      ST_MakePoint(${parseFloat(lng as string)}, ${parseFloat(lat as string)})
    )
    ORDER BY
      CASE risk_level
        WHEN 'critical' THEN 1
        WHEN 'high' THEN 2
        WHEN 'medium' THEN 3
        ELSE 4
      END
  `;

  res.json({
    inFloodZone: zones.length > 0,
    zones,
    highestRisk: zones[0]?.risk_level || null,
  });
});

export default router;
```

### Step 440 — Spatial Index Optimization

```sql
-- ตรวจสอบว่า query ใช้ spatial index
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users
WHERE ST_DWithin(
    location,
    ST_MakePoint(100.5, 13.7)::geography,
    5000
);
-- ควรเห็น "Index Scan using idx_users_location"

-- Cluster table ตาม spatial index (เพิ่ม performance)
CLUSTER users USING idx_users_location;

-- ตรวจสอบ index usage
SELECT
    indexrelname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE relname IN ('users', 'sensor_stations', 'flood_zones')
ORDER BY idx_scan DESC;

-- VACUUM ANALYZE เพื่อ update statistics
VACUUM ANALYZE users;
VACUUM ANALYZE sensor_stations;
VACUUM ANALYZE flood_zones;

-- ดู table statistics
SELECT
    relname AS table,
    n_live_tup AS live_rows,
    n_dead_tup AS dead_rows,
    last_vacuum,
    last_autoanalyze
FROM pg_stat_user_tables
WHERE relname IN ('users', 'sensor_stations', 'flood_zones');
```

---

## 🔧 Configuration Files

```sql
-- ตรวจสอบ PostGIS version และ SRID
SELECT PostGIS_Version();
SELECT * FROM spatial_ref_sys WHERE srid = 4326;

-- สร้าง composite index สำหรับ query บ่อย
CREATE INDEX idx_sos_location_time
    ON sos_reports USING GIST (location)
    WHERE created_at > NOW() - INTERVAL '7 days';
-- Partial spatial index เฉพาะข้อมูลใหม่

-- Function helper สำหรับ bounding box query (เร็วกว่า ST_DWithin สำหรับ approx)
CREATE OR REPLACE FUNCTION nearby_approx(
    p_lat FLOAT,
    p_lng FLOAT,
    p_radius_km FLOAT
)
RETURNS TABLE (
    user_id BIGINT,
    distance_m FLOAT
) AS $$
  SELECT
      id AS user_id,
      ST_Distance(location, ST_MakePoint(p_lng, p_lat)::geography) AS distance_m
  FROM users
  WHERE location && ST_Expand(
      ST_MakePoint(p_lng, p_lat)::geography,
      p_radius_km * 1000
  )
  AND ST_DWithin(location, ST_MakePoint(p_lng, p_lat)::geography, p_radius_km * 1000)
$$ LANGUAGE SQL STABLE;
```

---

## 🧪 Testing

```bash
# ทดสอบ ST_DWithin performance
psql -U postgres -d chuaikan_db << 'EOF'
-- Insert 1M users กระจายทั่วไทย
INSERT INTO users (username, email, location)
SELECT
    'user_' || i,
    'u' || i || '@test.com',
    ST_MakePoint(
        97.0 + random() * 8.0,   -- longitude 97-105
        5.0 + random() * 20.0    -- latitude 5-25
    )::geography
FROM generate_series(1, 1000000) i;

-- Test query
EXPLAIN ANALYZE
SELECT COUNT(*) FROM users
WHERE ST_DWithin(
    location,
    ST_MakePoint(100.5018, 13.7563)::geography,
    5000
);
EOF
```

---

## ❌ Common Errors & Solutions

**Error: `function st_dwithin(geography, ...) does not exist`**
```sql
-- PostGIS ยังไม่ได้ติดตั้ง
CREATE EXTENSION postgis;
-- ตรวจสอบ
SELECT postgis_version();
```

**Error: `ERROR: Geometry type (MultiPolygon) does not match column type (Polygon)`**
```sql
-- ใช้ GEOMETRY แทน GEOMETRY(POLYGON, 4326)
ALTER TABLE thai_provinces
    ALTER COLUMN boundary TYPE GEOMETRY(GEOMETRY, 4326)
    USING ST_Multi(boundary)::GEOMETRY(MULTIPOLYGON, 4326);
```

**ST_DWithin ช้าแม้มี index**
```sql
-- ตรวจสอบว่า statistics เก่า
ANALYZE users;
-- ตรวจสอบว่า index ถูกใช้
SET enable_seqscan = off;  -- force index
EXPLAIN SELECT ...;
SET enable_seqscan = on;
```

---

## ✅ Checklist

- [ ] **Step 431** — อธิบาย use cases ของ PostGIS สำหรับ chuaikan.com
- [ ] **Step 432** — เข้าใจความต่าง Geometry vs Geography
- [ ] **Step 433** — สร้าง tables พร้อม spatial columns และ GIST index
- [ ] **Step 434** — Import ข้อมูล polygon จังหวัดไทยสำเร็จ
- [ ] **Step 435** — ใช้ ST_Intersects และ ST_Contains ได้
- [ ] **Step 436** — Query users ใกล้เคียงด้วย ST_DWithin
- [ ] **Step 437** — คำนวณระยะทางด้วย ST_Distance (ได้หน่วยเมตร)
- [ ] **Step 438** — Output GeoJSON FeatureCollection ได้
- [ ] **Step 439** — สร้าง API endpoint /sos/nearby และ /flood-zones/check
- [ ] **Step 440** — ยืนยันว่า GIST index ถูกใช้ใน EXPLAIN ANALYZE

---

## 🔗 References

- [PostGIS Documentation](https://postgis.net/documentation/)
- [PostGIS Introduction](https://postgis.net/workshops/postgis-intro/)
- [Thailand GIS Data](https://data.humdata.org/dataset/tha-administrative-divisions-shapefiles)
- [Spatial Indexing with PostGIS](https://postgis.net/workshops/postgis-intro/indexing.html)

---
*Part 044 | Road to 1,000,000 Users/Day | chuaikan.com*
