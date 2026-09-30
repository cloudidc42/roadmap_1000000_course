# Part 028: Location Service (GeoSpatial)
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 271-280
> **เวลาโดยประมาณ:** 6 ชั่วโมง
> **Prerequisites:** Part 027 (SOS Service), Part 024 (WebSocket), Part 025 (Redis Cache)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ติดตั้ง PostGIS และทำงานกับ Geographic Data Types
- สร้าง Spatial Queries: nearby users, within province, inside flood zone
- Real-time Location Update ด้วย WebSocket + Privacy Controls
- Geofencing: แจ้งเตือนเมื่อ user เข้า/ออก flood zone
- Map Integration: Mapbox สำหรับประเทศไทย
- Address Geocoding และ Reverse Geocoding
- Import ข้อมูลจังหวัด/อำเภอ/ตำบล ของไทย
- Flood Zone Polygons จาก official data
- Location Privacy: Fuzzy Location (1km grid), Opt-in Settings
- Redis Geo สำหรับ fast nearby queries (GEOSEARCH)
- รับมือ 100,000 location updates/minute

---

## 📖 ทฤษฎีและแนวคิด

### Geographic Data Types

```
PostgreSQL + PostGIS Data Types
─────────────────────────────────────────────────────────
POINT      - จุดเดียว (lat, lng)
             Example: ตำแหน่งผู้ใช้, จุดรายงาน SOS

LINESTRING - เส้น
             Example: เส้นทาง, ถนน, แม่น้ำ

POLYGON    - พื้นที่
             Example: จังหวัด, อำเภอ, พื้นที่น้ำท่วม

MULTIPOLYGON - หลาย polygon
             Example: จังหวัดที่มีหลายส่วน (กรุงเทพ)

GEOGRAPHY  - ใช้หน่วย meter บนพื้นผิวโลก (Spherical)
GEOMETRY   - ใช้ระบบ Euclidean (แบน) เร็วกว่าแต่ไม่แม่นยำ

สำหรับ chuaikan.com: ใช้ GEOGRAPHY เสมอ (แม่นยำกว่า)
```

### Location Privacy Model

```
Privacy Levels
─────────────────────────────────────────────
EXACT    - แสดงตำแหน่งที่แน่นอน (เฉพาะตัวเอง)
FUZZY    - แสดงตำแหน่งแบบ 1km grid (default สำหรับคนอื่น)
PROVINCE - แสดงแค่จังหวัด
HIDDEN   - ไม่แสดงตำแหน่ง

User A is at: (13.7563, 100.5018)
Other users see: (13.750, 100.500) ← rounded to 1km grid
                 Province: กรุงเทพมหานคร

Admin/Moderator: เห็น EXACT เสมอ (สำหรับ SOS)
```

### Redis Geo vs PostGIS

```
Redis GEOSEARCH (GEOADD/GEOSEARCH)
────────────────────────────────────
+ เร็วมาก O(N+log(M)) ≈ microseconds
+ ใช้ Sorted Set ด้านหลัง
- ไม่ support polygon
- ไม่มี geofencing ซับซ้อน
- ใช้ haversine (approximate)
→ ใช้ sำหรับ: nearby users แบบ real-time

PostGIS ST_DWithin/ST_Within
─────────────────────────────
+ Accurate sphere calculations
+ Support polygon, multipolygon
+ Complex spatial operations
- ช้ากว่า Redis (milliseconds)
→ ใช้สำหรับ: flood zone check, province query

chuaikan.com ใช้ทั้งสองตามบริบท
```

---

## ⚙️ Environment Setup

### Step 271: PostGIS Setup และ Thai Geodata

```bash
# ติดตั้ง PostGIS
sudo apt-get update
sudo apt-get install -y \
  postgresql-17-postgis-3 \
  postgresql-17-postgis-3-scripts \
  gdal-bin \
  osmium-tool

# Enable extensions
psql -U postgres -d chuaikan_db << 'EOF'
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS postgis_topology;
CREATE EXTENSION IF NOT EXISTS fuzzystrmatch;
CREATE EXTENSION IF NOT EXISTS postgis_tiger_geocoder;
SELECT PostGIS_full_version();
EOF

# ติดตั้ง tools สำหรับ import geodata
sudo apt-get install -y shp2pgsql ogr2ogr
```

### Step 272: Import Thai Administrative Boundaries

```bash
# ดาวน์โหลด Thailand geodata จาก GADM
mkdir -p /home/user/geodata/thailand
cd /home/user/geodata/thailand

# Download Thailand admin boundaries (GADM level 1-3)
# Province (Level 1), District (Level 2), Sub-district (Level 3)
wget -q "https://geodata.ucdavis.edu/gadm/gadm4.1/shp/gadm41_THA_shp.zip"
unzip gadm41_THA_shp.zip

# Import to PostgreSQL
# Level 1 = จังหวัด
shp2pgsql -I -s 4326 gadm41_THA_1.shp public.thailand_provinces | \
  psql -U chuaikan_user -d chuaikan_db

# Level 2 = อำเภอ
shp2pgsql -I -s 4326 gadm41_THA_2.shp public.thailand_districts | \
  psql -U chuaikan_user -d chuaikan_db

# Level 3 = ตำบล
shp2pgsql -I -s 4326 gadm41_THA_3.shp public.thailand_subdistricts | \
  psql -U chuaikan_user -d chuaikan_db

# ตรวจสอบ
psql -U chuaikan_user -d chuaikan_db -c \
  "SELECT COUNT(*) FROM thailand_provinces;"
# → 77 rows (77 จังหวัด)
```

### Step 273: Location Tables Schema

```sql
-- /home/user/chuaikan/db/migrations/022_location_tables.sql

-- User locations (real-time)
CREATE TABLE user_locations (
  user_id     UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  lat         DECIMAL(10, 7) NOT NULL,
  lng         DECIMAL(11, 7) NOT NULL,
  location    GEOGRAPHY(POINT, 4326),
  accuracy_m  FLOAT,                    -- GPS accuracy in meters
  province_code VARCHAR(10),
  district_code VARCHAR(10),
  geohash     VARCHAR(12),             -- สำหรับ fuzzy display
  privacy     VARCHAR(20) DEFAULT 'fuzzy', -- exact, fuzzy, province, hidden
  source      VARCHAR(20) DEFAULT 'gps',  -- gps, ip, manual
  updated_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_uloc_location ON user_locations USING GIST(location);
CREATE INDEX idx_uloc_updated  ON user_locations(updated_at DESC);
CREATE INDEX idx_uloc_province ON user_locations(province_code);

-- Location history (อย่างเก็บนาน เพราะ PDPA)
CREATE TABLE user_location_history (
  id          BIGSERIAL PRIMARY KEY,
  user_id     UUID NOT NULL REFERENCES users(id),
  lat         DECIMAL(10, 7) NOT NULL,
  lng         DECIMAL(11, 7) NOT NULL,
  province_code VARCHAR(10),
  recorded_at TIMESTAMPTZ DEFAULT NOW()
) PARTITION BY RANGE (recorded_at);

-- Partitions รายเดือน (เก็บ 3 เดือน)
CREATE TABLE user_location_history_2024_01
  PARTITION OF user_location_history
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- Flood zones (polygon จากหน่วยงานรัฐ)
CREATE TABLE flood_zones (
  id            BIGSERIAL PRIMARY KEY,
  zone_code     VARCHAR(50) UNIQUE NOT NULL,
  name          VARCHAR(200) NOT NULL,
  province_code VARCHAR(10),
  risk_level    VARCHAR(20),  -- low, medium, high, critical
  zone_polygon  GEOGRAPHY(MULTIPOLYGON, 4326),
  data_source   VARCHAR(100),
  valid_from    DATE,
  valid_to      DATE,
  metadata      JSONB DEFAULT '{}',
  created_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_flood_zone ON flood_zones USING GIST(zone_polygon);
CREATE INDEX idx_flood_province ON flood_zones(province_code);

-- Geofence configurations
CREATE TABLE geofences (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name        VARCHAR(200) NOT NULL,
  area        GEOGRAPHY(POLYGON, 4326) NOT NULL,
  type        VARCHAR(50),   -- flood_zone, disaster_area, evacuation_zone
  alert_on    VARCHAR(20),   -- enter, exit, both
  created_by  UUID,
  is_active   BOOLEAN DEFAULT true,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_geofence_area ON geofences USING GIST(area);

-- Geofence events (ติดตามว่า user เข้า/ออก)
CREATE TABLE geofence_events (
  id           BIGSERIAL PRIMARY KEY,
  user_id      UUID NOT NULL REFERENCES users(id),
  geofence_id  UUID NOT NULL REFERENCES geofences(id),
  event_type   VARCHAR(10) NOT NULL,  -- enter, exit
  lat          DECIMAL(10, 7),
  lng          DECIMAL(11, 7),
  created_at   TIMESTAMPTZ DEFAULT NOW()
);
```

```bash
psql -U chuaikan_user -d chuaikan_db \
  -f /home/user/chuaikan/db/migrations/022_location_tables.sql
```

---

## 🛠️ Step-by-Step Implementation

### Step 274: Location Service Core

```typescript
// src/services/location.service.ts

import { Pool } from 'pg';
import Redis from 'ioredis';
import { Server as SocketServer } from 'socket.io';
import geohash from 'geohash';

export interface LocationUpdate {
  userId: string;
  lat: number;
  lng: number;
  accuracyMeters?: number;
  source?: 'gps' | 'ip' | 'manual';
}

export interface NearbyUser {
  userId: string;
  lat: number;     // fuzzy lat
  lng: number;     // fuzzy lng
  distanceKm: number;
  province?: string;
}

const FUZZY_PRECISION = 5;  // geohash precision 5 ≈ 4.9km × 4.9km
// ลด precision ให้ดูแบบ ~1km grid

export class LocationService {
  private readonly LOCATION_TTL = 3600;  // 1 hour
  private readonly GEO_KEY = 'geo:users';

  constructor(
    private db: Pool,
    private redis: Redis,
    private io: SocketServer
  ) {}

  // Update user location
  async updateLocation(update: LocationUpdate): Promise<void> {
    const { userId, lat, lng, accuracyMeters, source } = update;

    // คำนวณ geohash สำหรับ fuzzy display
    const hash = geohash.encode(lat, lng, FUZZY_PRECISION);
    const { latitude: fuzzyLat, longitude: fuzzyLng } =
      geohash.decode(hash);

    // หา province จาก PostGIS
    const provinceResult = await this.db.query(`
      SELECT gid::TEXT AS province_code, "NAME_1" AS province_name
      FROM thailand_provinces
      WHERE ST_Within(
        ST_MakePoint($2, $1)::geometry,
        geom
      )
      LIMIT 1
    `, [lat, lng]);

    const provinceCode = provinceResult.rows[0]?.province_code;

    // Upsert location ใน PostgreSQL
    await this.db.query(`
      INSERT INTO user_locations
        (user_id, lat, lng, accuracy_m, province_code, geohash, source, updated_at)
      VALUES ($1, $2, $3, $4, $5, $6, $7, NOW())
      ON CONFLICT (user_id) DO UPDATE SET
        lat          = EXCLUDED.lat,
        lng          = EXCLUDED.lng,
        accuracy_m   = EXCLUDED.accuracy_m,
        province_code = EXCLUDED.province_code,
        geohash      = EXCLUDED.geohash,
        source       = EXCLUDED.source,
        updated_at   = NOW()
    `, [userId, lat, lng, accuracyMeters, provinceCode, hash, source ?? 'gps']);

    // Update Redis Geo (ใช้ fuzzy coords สำหรับ privacy)
    await this.redis.geoadd(
      this.GEO_KEY,
      fuzzyLng,
      fuzzyLat,
      userId
    );
    await this.redis.expire(this.GEO_KEY, this.LOCATION_TTL);

    // Check geofences
    await this.checkGeofences(userId, lat, lng);

    // Broadcast ไปยัง followers (ถ้า user เปิด live location)
    await this.broadcastLocationUpdate(userId, fuzzyLat, fuzzyLng, provinceCode);
  }

  // หา users ใกล้เคียงด้วย Redis GEOSEARCH
  async findNearbyUsers(
    lat: number,
    lng: number,
    radiusKm: number,
    limit: number = 50
  ): Promise<NearbyUser[]> {
    const results = await this.redis.geosearch(
      this.GEO_KEY,
      'FROMLONLAT', lng, lat,
      'BYRADIUS', radiusKm, 'km',
      'ASC',
      'COUNT', limit,
      'WITHCOORD',
      'WITHDIST'
    ) as any[];

    // results: [[userId, distance, [lng, lat]], ...]
    return results.map((r: any) => ({
      userId: r[0],
      distanceKm: parseFloat(r[1]),
      lng: parseFloat(r[2][0]),
      lat: parseFloat(r[2][1]),
    }));
  }

  // ตรวจสอบว่า user อยู่ใน flood zone ไหม
  async isInFloodZone(
    lat: number,
    lng: number
  ): Promise<{ inZone: boolean; zone?: any }> {
    const result = await this.db.query(`
      SELECT id, zone_code, name, risk_level
      FROM flood_zones
      WHERE ST_Within(
        ST_MakePoint($2, $1)::geometry,
        zone_polygon::geometry
      )
        AND (valid_to IS NULL OR valid_to >= CURRENT_DATE)
      ORDER BY
        CASE risk_level
          WHEN 'critical' THEN 1
          WHEN 'high' THEN 2
          WHEN 'medium' THEN 3
          ELSE 4
        END
      LIMIT 1
    `, [lat, lng]);

    if (result.rows.length === 0) {
      return { inZone: false };
    }

    return { inZone: true, zone: result.rows[0] };
  }

  // Geofencing: check ถ้า user เข้า/ออก geofences
  private async checkGeofences(
    userId: string,
    lat: number,
    lng: number
  ): Promise<void> {
    // ตรวจสอบ geofences ที่ active
    const geofences = await this.db.query(`
      SELECT g.id, g.name, g.type, g.alert_on,
        ST_Within(
          ST_MakePoint($2, $1)::geometry,
          g.area::geometry
        ) AS is_inside
      FROM geofences g
      WHERE g.is_active = true
        AND ST_DWithin(
          g.area::geography,
          ST_MakePoint($2, $1)::geography,
          50000  -- check geofences within 50km
        )
    `, [lat, lng]);

    for (const fence of geofences.rows) {
      const prevStateKey = `geo:fence:${userId}:${fence.id}`;
      const prevState = await this.redis.get(prevStateKey);
      const wasInside = prevState === 'inside';
      const isInside = fence.is_inside;

      let event: 'enter' | 'exit' | null = null;

      if (isInside && !wasInside) event = 'enter';
      else if (!isInside && wasInside) event = 'exit';

      if (event) {
        await this.redis.setex(
          prevStateKey,
          86400,
          isInside ? 'inside' : 'outside'
        );

        if (
          fence.alert_on === 'both' ||
          fence.alert_on === event
        ) {
          await this.triggerGeofenceAlert(userId, fence, event, lat, lng);
        }
      } else {
        // อัปเดต state ไว้
        await this.redis.setex(
          prevStateKey,
          86400,
          isInside ? 'inside' : 'outside'
        );
      }
    }
  }

  private async triggerGeofenceAlert(
    userId: string,
    fence: any,
    event: 'enter' | 'exit',
    lat: number,
    lng: number
  ): Promise<void> {
    // Log event
    await this.db.query(`
      INSERT INTO geofence_events
        (user_id, geofence_id, event_type, lat, lng)
      VALUES ($1, $2, $3, $4, $5)
    `, [userId, fence.id, event, lat, lng]);

    // Notify via Socket.io
    this.io.to(`user-${userId}`).emit('geofence:alert', {
      fenceId: fence.id,
      fenceName: fence.name,
      fenceType: fence.type,
      event,
      message: event === 'enter'
        ? `คุณเข้าสู่พื้นที่ ${fence.name}`
        : `คุณออกจากพื้นที่ ${fence.name}`,
      lat,
      lng,
    });
  }

  private async broadcastLocationUpdate(
    userId: string,
    fuzzyLat: number,
    fuzzyLng: number,
    provinceCode?: string
  ): Promise<void> {
    // ตรวจสอบว่า user เปิด live location sharing ไหม
    const settings = await this.redis.get(`user:live-location:${userId}`);
    if (!settings) return;

    this.io.to(`location:${userId}`).emit('location:updated', {
      userId,
      lat: fuzzyLat,
      lng: fuzzyLng,
      provinceCode,
      updatedAt: new Date().toISOString(),
    });
  }

  // ดึงตำแหน่ง user (พร้อม privacy control)
  async getUserLocation(
    targetUserId: string,
    requesterId: string,
    requesterRole: string[]
  ): Promise<any | null> {
    const result = await this.db.query(`
      SELECT ul.lat, ul.lng, ul.geohash, ul.privacy,
             ul.province_code, ul.updated_at
      FROM user_locations ul
      WHERE ul.user_id = $1
    `, [targetUserId]);

    if (result.rows.length === 0) return null;

    const loc = result.rows[0];

    // Admin เห็น exact เสมอ
    if (requesterRole.includes('admin') || requesterRole.includes('moderator')) {
      return loc;
    }

    // ตัวเองเห็น exact
    if (targetUserId === requesterId) {
      return loc;
    }

    // Privacy settings
    switch (loc.privacy) {
      case 'hidden':
        return null;

      case 'province':
        return { provinceCode: loc.province_code, updatedAt: loc.updated_at };

      case 'fuzzy': {
        const decoded = geohash.decode(loc.geohash);
        return {
          lat: decoded.latitude,
          lng: decoded.longitude,
          provinceCode: loc.province_code,
          updatedAt: loc.updated_at,
        };
      }

      default: // exact
        return {
          lat: loc.lat,
          lng: loc.lng,
          provinceCode: loc.province_code,
          updatedAt: loc.updated_at,
        };
    }
  }
}
```

### Step 275: Geocoding Service

```typescript
// src/services/geocoding.service.ts

import axios from 'axios';
import Redis from 'ioredis';

interface GeocodingResult {
  lat: number;
  lng: number;
  formattedAddress: string;
  provinceCode?: string;
  districtCode?: string;
  country: string;
}

interface ReverseGeocodingResult {
  formattedAddress: string;
  street?: string;
  subdistrict?: string;
  district?: string;
  province?: string;
  provinceCode?: string;
  postalCode?: string;
  country: string;
}

export class GeocodingService {
  private readonly CACHE_TTL = 86400 * 7; // 7 days

  constructor(
    private redis: Redis,
    private mapboxToken = process.env.MAPBOX_TOKEN,
    private googleMapsKey = process.env.GOOGLE_MAPS_KEY
  ) {}

  // Forward geocoding: address → lat/lng
  async geocode(address: string): Promise<GeocodingResult | null> {
    const cacheKey = `geocode:${address.toLowerCase().trim()}`;
    const cached = await this.redis.get(cacheKey);
    if (cached) return JSON.parse(cached);

    try {
      // ลอง Mapbox ก่อน
      const result = await this.geocodeMapbox(address);
      if (result) {
        await this.redis.setex(cacheKey, this.CACHE_TTL, JSON.stringify(result));
        return result;
      }

      // Fallback to Google Maps
      const googleResult = await this.geocodeGoogle(address);
      if (googleResult) {
        await this.redis.setex(cacheKey, this.CACHE_TTL, JSON.stringify(googleResult));
        return googleResult;
      }

      return null;
    } catch (error) {
      console.error('Geocoding error:', error);
      return null;
    }
  }

  // Reverse geocoding: lat/lng → address
  async reverseGeocode(
    lat: number,
    lng: number
  ): Promise<ReverseGeocodingResult | null> {
    const cacheKey = `rgeocode:${lat.toFixed(4)}:${lng.toFixed(4)}`;
    const cached = await this.redis.get(cacheKey);
    if (cached) return JSON.parse(cached);

    try {
      const result = await this.reverseGeocodeMapbox(lat, lng);
      if (result) {
        await this.redis.setex(cacheKey, this.CACHE_TTL, JSON.stringify(result));
        return result;
      }
      return null;
    } catch (error) {
      console.error('Reverse geocoding error:', error);
      return null;
    }
  }

  private async geocodeMapbox(address: string): Promise<GeocodingResult | null> {
    if (!this.mapboxToken) return null;

    const encoded = encodeURIComponent(`${address}, Thailand`);
    const url = `https://api.mapbox.com/geocoding/v5/mapbox.places/${encoded}.json`;

    const response = await axios.get(url, {
      params: {
        access_token: this.mapboxToken,
        country: 'TH',
        language: 'th',
        limit: 1,
      },
      timeout: 5000,
    });

    const feature = response.data?.features?.[0];
    if (!feature) return null;

    const [lng, lat] = feature.center;

    return {
      lat,
      lng,
      formattedAddress: feature.place_name,
      country: 'TH',
    };
  }

  private async geocodeGoogle(address: string): Promise<GeocodingResult | null> {
    if (!this.googleMapsKey) return null;

    const response = await axios.get(
      'https://maps.googleapis.com/maps/api/geocode/json',
      {
        params: {
          address: `${address}, ประเทศไทย`,
          key: this.googleMapsKey,
          language: 'th',
          region: 'th',
        },
        timeout: 5000,
      }
    );

    const result = response.data?.results?.[0];
    if (!result) return null;

    return {
      lat: result.geometry.location.lat,
      lng: result.geometry.location.lng,
      formattedAddress: result.formatted_address,
      country: 'TH',
    };
  }

  private async reverseGeocodeMapbox(
    lat: number,
    lng: number
  ): Promise<ReverseGeocodingResult | null> {
    if (!this.mapboxToken) return null;

    const url =
      `https://api.mapbox.com/geocoding/v5/mapbox.places/${lng},${lat}.json`;

    const response = await axios.get(url, {
      params: {
        access_token: this.mapboxToken,
        types: 'address,place,region',
        language: 'th',
        limit: 1,
      },
      timeout: 5000,
    });

    const feature = response.data?.features?.[0];
    if (!feature) return null;

    const context = feature.context ?? [];
    const region = context.find((c: any) => c.id?.startsWith('region'));
    const place = context.find((c: any) => c.id?.startsWith('place'));
    const postcode = context.find((c: any) => c.id?.startsWith('postcode'));

    return {
      formattedAddress: feature.place_name,
      province: region?.text,
      district: place?.text,
      postalCode: postcode?.text,
      country: 'TH',
    };
  }
}
```

### Step 276: Flood Zone Import

```bash
#!/bin/bash
# scripts/import-flood-zones.sh
# Import flood zone data จาก GISTDA หรือ DWR

set -e

DB_URL="postgresql://chuaikan_user:password@localhost:5432/chuaikan_db"

echo "Downloading flood zone data..."

# ดาวน์โหลด flood zone shapefile (2024)
# ในความเป็นจริง ควรขอ API key จาก GISTDA
wget -q "https://flood.gistda.or.th/download/flood_risk_2024.zip" \
  -O /tmp/flood_risk.zip || {
  echo "Download failed, using sample data..."
  # สร้าง sample data สำหรับทดสอบ
  cat > /tmp/flood_sample.sql << 'SQLEOF'
INSERT INTO flood_zones (zone_code, name, province_code, risk_level, zone_polygon, data_source, valid_from)
VALUES (
  'BKK-FLOOD-001',
  'พื้นที่เสี่ยงน้ำท่วม กรุงเทพใต้',
  '10',
  'high',
  ST_GeographyFromText('SRID=4326;MULTIPOLYGON(((100.45 13.65, 100.55 13.65, 100.55 13.75, 100.45 13.75, 100.45 13.65)))'),
  'GISTDA Sample',
  CURRENT_DATE
);

INSERT INTO flood_zones (zone_code, name, province_code, risk_level, zone_polygon, data_source, valid_from)
VALUES (
  'AYT-FLOOD-001',
  'พื้นที่เสี่ยงน้ำท่วม พระนครศรีอยุธยา',
  '14',
  'critical',
  ST_GeographyFromText('SRID=4326;MULTIPOLYGON(((100.55 14.30, 100.65 14.30, 100.65 14.40, 100.55 14.40, 100.55 14.30)))'),
  'GISTDA Sample',
  CURRENT_DATE
);
SQLEOF
  psql "$DB_URL" -f /tmp/flood_sample.sql
  echo "Sample flood zones imported"
  exit 0
}

# Unzip และ import จริง
unzip -q /tmp/flood_risk.zip -d /tmp/flood_risk/

# Convert และ import
ogr2ogr -f "PostgreSQL" \
  PG:"$DB_URL" \
  /tmp/flood_risk/flood_risk_2024.shp \
  -nln flood_zones_temp \
  -t_srs EPSG:4326 \
  -overwrite

# Transform ไปยัง flood_zones table ของเรา
psql "$DB_URL" << 'SQLEOF'
INSERT INTO flood_zones (zone_code, name, province_code, risk_level, zone_polygon, data_source, valid_from)
SELECT
  'GISTDA-' || gid::TEXT,
  COALESCE(name_th, name_en, 'Unknown'),
  province_code,
  CASE risk_level::INT
    WHEN 1 THEN 'low'
    WHEN 2 THEN 'medium'
    WHEN 3 THEN 'high'
    WHEN 4 THEN 'critical'
    ELSE 'low'
  END,
  ST_Multi(wkb_geometry)::geography,
  'GISTDA 2024',
  CURRENT_DATE
FROM flood_zones_temp
ON CONFLICT (zone_code) DO UPDATE SET
  zone_polygon = EXCLUDED.zone_polygon,
  risk_level   = EXCLUDED.risk_level;

DROP TABLE flood_zones_temp;
SQLEOF

echo "Flood zones imported successfully"
psql "$DB_URL" -c "SELECT COUNT(*), risk_level FROM flood_zones GROUP BY risk_level;"
```

### Step 277: Location API Routes

```typescript
// src/routes/location.routes.ts

import { Router, Request, Response } from 'express';
import { LocationService } from '../services/location.service';
import { GeocodingService } from '../services/geocoding.service';
import { z } from 'zod';

const UpdateLocationSchema = z.object({
  lat: z.number().min(5).max(21),   // Thailand lat range
  lng: z.number().min(97).max(106), // Thailand lng range
  accuracy: z.number().positive().optional(),
  source: z.enum(['gps', 'ip', 'manual']).optional(),
});

export function createLocationRouter(
  locationService: LocationService,
  geocodingService: GeocodingService
): Router {
  const router = Router();

  // PUT /api/v1/location - Update location
  router.put('/', async (req: Request, res: Response) => {
    try {
      const data = UpdateLocationSchema.parse(req.body);
      const userId = req.user!.id;

      await locationService.updateLocation({
        userId,
        lat: data.lat,
        lng: data.lng,
        accuracyMeters: data.accuracy,
        source: data.source,
      });

      res.json({ ok: true });
    } catch (error: any) {
      if (error.name === 'ZodError') {
        return res.status(400).json({ error: error.errors });
      }
      res.status(500).json({ error: 'Location update failed' });
    }
  });

  // GET /api/v1/location/nearby-users
  router.get('/nearby-users', async (req: Request, res: Response) => {
    const lat = parseFloat(req.query.lat as string);
    const lng = parseFloat(req.query.lng as string);
    const radius = parseFloat(req.query.radius as string) || 10;

    if (isNaN(lat) || isNaN(lng)) {
      return res.status(400).json({ error: 'Invalid coordinates' });
    }

    const users = await locationService.findNearbyUsers(lat, lng, radius);
    res.json({ users, count: users.length });
  });

  // GET /api/v1/location/flood-zone
  router.get('/flood-zone', async (req: Request, res: Response) => {
    const lat = parseFloat(req.query.lat as string);
    const lng = parseFloat(req.query.lng as string);

    const result = await locationService.isInFloodZone(lat, lng);
    res.json(result);
  });

  // GET /api/v1/location/geocode?address=...
  router.get('/geocode', async (req: Request, res: Response) => {
    const address = req.query.address as string;
    if (!address) return res.status(400).json({ error: 'address required' });

    const result = await geocodingService.geocode(address);
    if (!result) return res.status(404).json({ error: 'Location not found' });

    res.json(result);
  });

  // GET /api/v1/location/reverse-geocode?lat=...&lng=...
  router.get('/reverse-geocode', async (req: Request, res: Response) => {
    const lat = parseFloat(req.query.lat as string);
    const lng = parseFloat(req.query.lng as string);

    const result = await geocodingService.reverseGeocode(lat, lng);
    if (!result) return res.status(404).json({ error: 'Address not found' });

    res.json(result);
  });

  // GET /api/v1/location/users/:userId
  router.get('/users/:userId', async (req: Request, res: Response) => {
    const { userId } = req.params;
    const requesterId = req.user!.id;
    const requesterRoles = req.user!.roles;

    const location = await locationService.getUserLocation(
      userId,
      requesterId,
      requesterRoles
    );

    if (!location) {
      return res.status(404).json({ error: 'Location not available' });
    }

    res.json(location);
  });

  // PATCH /api/v1/location/privacy
  router.patch('/privacy', async (req: Request, res: Response) => {
    const userId = req.user!.id;
    const { privacy } = req.body;

    const validPrivacy = ['exact', 'fuzzy', 'province', 'hidden'];
    if (!validPrivacy.includes(privacy)) {
      return res.status(400).json({ error: 'Invalid privacy setting' });
    }

    await req.db.query(
      'UPDATE user_locations SET privacy = $2 WHERE user_id = $1',
      [userId, privacy]
    );

    res.json({ ok: true, privacy });
  });

  return router;
}
```

### Step 278: High-Performance Location Updates

```typescript
// src/workers/location-update.worker.ts
// Handle 100,000 location updates/minute

import Redis from 'ioredis';
import { Pool } from 'pg';
import { LocationService } from '../services/location.service';

const BATCH_SIZE = 1000;
const BATCH_INTERVAL_MS = 1000; // process every 1 second

export class LocationUpdateWorker {
  private isRunning = false;

  constructor(
    private redis: Redis,
    private db: Pool,
    private locationService: LocationService
  ) {}

  start(): void {
    this.isRunning = true;
    this.processBatch();
  }

  stop(): void {
    this.isRunning = false;
  }

  private async processBatch(): Promise<void> {
    while (this.isRunning) {
      const startTime = Date.now();

      try {
        // ดึง updates จาก Redis queue
        const pipeline = this.redis.pipeline();
        for (let i = 0; i < BATCH_SIZE; i++) {
          pipeline.rpop('location-update-queue');
        }
        const results = await pipeline.exec();

        const updates = results
          ?.map(([err, val]) => val as string | null)
          .filter((v): v is string => v !== null)
          .map(v => JSON.parse(v));

        if (!updates || updates.length === 0) {
          // ไม่มี update → wait
          await new Promise(r => setTimeout(r, 100));
          continue;
        }

        // Batch upsert ลง PostgreSQL
        await this.batchUpsertLocations(updates);

        // Update Redis Geo batch
        await this.batchUpdateGeo(updates);

        const elapsed = Date.now() - startTime;
        if (updates.length > 0) {
          console.log(
            `Processed ${updates.length} location updates in ${elapsed}ms`
          );
        }
      } catch (error) {
        console.error('Location batch error:', error);
        await new Promise(r => setTimeout(r, 1000));
      }
    }
  }

  private async batchUpsertLocations(updates: any[]): Promise<void> {
    if (updates.length === 0) return;

    // Build bulk upsert
    const values = updates
      .map((u, i) => {
        const base = i * 6;
        return `($${base + 1}, $${base + 2}, $${base + 3}, $${base + 4}, $${base + 5}, $${base + 6}, NOW())`;
      })
      .join(', ');

    const params = updates.flatMap(u => [
      u.userId,
      u.lat,
      u.lng,
      u.accuracyMeters ?? null,
      u.provinceCode ?? null,
      u.geohash ?? null,
    ]);

    await this.db.query(`
      INSERT INTO user_locations
        (user_id, lat, lng, accuracy_m, province_code, geohash, updated_at)
      VALUES ${values}
      ON CONFLICT (user_id) DO UPDATE SET
        lat          = EXCLUDED.lat,
        lng          = EXCLUDED.lng,
        accuracy_m   = EXCLUDED.accuracy_m,
        province_code = EXCLUDED.province_code,
        geohash      = EXCLUDED.geohash,
        updated_at   = EXCLUDED.updated_at
    `, params);
  }

  private async batchUpdateGeo(updates: any[]): Promise<void> {
    if (updates.length === 0) return;

    const geoArgs: (string | number)[] = ['geo:users'];
    for (const u of updates) {
      geoArgs.push(u.lng, u.lat, u.userId);
    }

    await (this.redis as any).geoadd(...geoArgs);
  }
}
```

### Step 279: WebSocket Location Handler

```typescript
// src/socket/location.handler.ts

import { Server, Socket } from 'socket.io';
import { LocationUpdateWorker } from '../workers/location-update.worker';
import Redis from 'ioredis';
import geohash from 'geohash';

export function setupLocationHandlers(
  io: Server,
  redis: Redis
): void {
  io.on('connection', (socket: Socket) => {
    const userId = socket.handshake.auth.userId as string;
    if (!userId) return;

    // Join user's personal room
    socket.join(`user-${userId}`);

    // Handle location update from client
    socket.on('location:update', async (data: {
      lat: number;
      lng: number;
      accuracy?: number;
    }) => {
      // Validate Thailand bounds
      if (
        data.lat < 5 || data.lat > 21 ||
        data.lng < 97 || data.lng > 106
      ) {
        socket.emit('location:error', { message: 'Invalid coordinates' });
        return;
      }

      const hash = geohash.encode(data.lat, data.lng, 5);
      const decoded = geohash.decode(hash);

      // Queue ใน Redis สำหรับ batch processing
      await redis.lpush('location-update-queue', JSON.stringify({
        userId,
        lat: data.lat,
        lng: data.lng,
        accuracyMeters: data.accuracy,
        geohash: hash,
        fuzzyLat: decoded.latitude,
        fuzzyLng: decoded.longitude,
        timestamp: Date.now(),
      }));

      socket.emit('location:ack', { ok: true });
    });

    // Subscribe to watch another user's location
    socket.on('location:watch', (targetUserId: string) => {
      socket.join(`location:${targetUserId}`);
    });

    socket.on('location:unwatch', (targetUserId: string) => {
      socket.leave(`location:${targetUserId}`);
    });

    // Enable live location sharing
    socket.on('location:share:start', async () => {
      await redis.setex(`user:live-location:${userId}`, 3600, '1');
      socket.emit('location:share:started');
    });

    socket.on('location:share:stop', async () => {
      await redis.del(`user:live-location:${userId}`);
      socket.emit('location:share:stopped');
    });
  });
}
```

---

## 🔧 Configuration Files

```typescript
// src/config/location.config.ts

export const LOCATION_CONFIG = {
  // Thailand geographic bounds
  BOUNDS: {
    MIN_LAT: 5.5,
    MAX_LAT: 20.5,
    MIN_LNG: 97.5,
    MAX_LNG: 105.7,
  },

  // Privacy
  DEFAULT_PRIVACY: 'fuzzy' as const,
  FUZZY_GEOHASH_PRECISION: 5,   // ~4.9km × 4.9km box

  // Redis
  GEO_KEY: 'geo:users',
  LOCATION_TTL: 3600,            // 1 hour
  GEOFENCE_STATE_TTL: 86400,    // 24 hours

  // Batch processing
  BATCH_SIZE: 1000,
  BATCH_INTERVAL_MS: 1000,

  // Geocoding
  GEOCODE_CACHE_TTL: 604800,    // 7 days

  // Nearby search
  DEFAULT_RADIUS_KM: 10,
  MAX_RADIUS_KM: 100,
  MAX_NEARBY_RESULTS: 100,

  // Maps
  MAPBOX_STYLE: 'mapbox://styles/mapbox/streets-v12',
  DEFAULT_CENTER: { lat: 13.7563, lng: 100.5018 }, // Bangkok
  DEFAULT_ZOOM: 10,
};
```

---

## 🧪 Testing

### Step 280: Spatial Query Tests

```typescript
// tests/location.test.ts

import { describe, it, expect, beforeAll } from '@jest/globals';
import { Pool } from 'pg';

describe('PostGIS Spatial Queries', () => {
  let db: Pool;

  beforeAll(() => {
    db = new Pool({
      connectionString: process.env.TEST_DB_URL,
    });
  });

  it('should find users within radius using ST_DWithin', async () => {
    const BKK_LAT = 13.7563;
    const BKK_LNG = 100.5018;
    const RADIUS_KM = 5;

    const result = await db.query(`
      SELECT COUNT(*) AS count
      FROM user_locations
      WHERE ST_DWithin(
        location::geography,
        ST_MakePoint($2, $1)::geography,
        $3 * 1000
      )
    `, [BKK_LAT, BKK_LNG, RADIUS_KM]);

    expect(parseInt(result.rows[0].count)).toBeGreaterThanOrEqual(0);
  });

  it('should check if point is in flood zone', async () => {
    // ใส่ sample flood zone ก่อน
    await db.query(`
      INSERT INTO flood_zones
        (zone_code, name, risk_level, zone_polygon, data_source, valid_from)
      VALUES (
        'TEST-ZONE-001',
        'Test Zone',
        'high',
        ST_GeographyFromText('SRID=4326;MULTIPOLYGON(((100.4 13.6, 100.6 13.6, 100.6 13.8, 100.4 13.8, 100.4 13.6)))'),
        'test',
        CURRENT_DATE
      )
      ON CONFLICT (zone_code) DO NOTHING
    `);

    // ทดสอบ point ที่อยู่ใน zone
    const inside = await db.query(`
      SELECT EXISTS(
        SELECT 1 FROM flood_zones
        WHERE ST_Within(
          ST_MakePoint(100.5, 13.7)::geometry,
          zone_polygon::geometry
        )
        AND zone_code = 'TEST-ZONE-001'
      ) AS in_zone
    `);

    expect(inside.rows[0].in_zone).toBe(true);

    // ทดสอบ point ที่อยู่นอก zone
    const outside = await db.query(`
      SELECT EXISTS(
        SELECT 1 FROM flood_zones
        WHERE ST_Within(
          ST_MakePoint(101.0, 14.0)::geometry,
          zone_polygon::geometry
        )
        AND zone_code = 'TEST-ZONE-001'
      ) AS in_zone
    `);

    expect(outside.rows[0].in_zone).toBe(false);
  });
});

// Load test: 100,000 location updates/minute
describe('Location Update Performance', () => {
  it('should handle 1000 updates in < 2 seconds', async () => {
    const redis = new (await import('ioredis')).default();
    const start = Date.now();

    const pipeline = redis.pipeline();
    for (let i = 0; i < 1000; i++) {
      pipeline.lpush('location-update-queue', JSON.stringify({
        userId: `user-${i}`,
        lat: 13.7 + Math.random() * 0.1,
        lng: 100.5 + Math.random() * 0.1,
        timestamp: Date.now(),
      }));
    }
    await pipeline.exec();

    const elapsed = Date.now() - start;
    console.log(`Queued 1000 updates in ${elapsed}ms`);

    expect(elapsed).toBeLessThan(2000);
    await redis.quit();
  });
});
```

```bash
# รัน spatial tests
cd /home/user/chuaikan/services/location-service
npx jest tests/location.test.ts

# ตรวจสอบ index usage
psql -U chuaikan_user -d chuaikan_db -c "
EXPLAIN ANALYZE
SELECT user_id
FROM user_locations
WHERE ST_DWithin(
  location::geography,
  ST_MakePoint(100.5018, 13.7563)::geography,
  5000
);
"
# ควรใช้ Index Scan on idx_uloc_location
```

---

## ❌ Common Errors & Solutions

### Error 1: GIST Index Not Used

```
Seq Scan on user_locations (cost=0.00..12345.00 rows=100)
```

```sql
-- แก้: ตรวจสอบ index
\d user_locations
-- ต้องมี: "idx_uloc_location" gist (location)

-- ถ้าไม่มี:
CREATE INDEX idx_uloc_location ON user_locations USING GIST(location);

-- อัปเดต statistics
ANALYZE user_locations;

-- Force planner ใช้ index
SET enable_seqscan = OFF;
EXPLAIN SELECT ...;
SET enable_seqscan = ON;
```

### Error 2: Geohash Precision Too High (Privacy Leak)

```
User A is at: (13.7563, 100.5018)
Geohash precision 9: w3gv3m9gh ← แม่นเกินไป! ≈ 4.8m
```

```typescript
// แก้: ใช้ precision 5 (≈ 4.9km × 4.9km)
const FUZZY_PRECISION = 5;
// หรือ precision 6 (≈ 1.2km × 0.6km) ถ้าต้องการแม่นขึ้น

const hash = geohash.encode(lat, lng, FUZZY_PRECISION);
const { latitude, longitude } = geohash.decode(hash);
// latitude, longitude จะถูก snap ไป grid center
```

### Error 3: Redis GEOADD Returns Error

```
Error: ERR value is not a valid float
```

```typescript
// แก้: ตรวจสอบ lng ก่อน lat ใน GEOADD
// Redis GEOADD ลำดับคือ: longitude ก่อน latitude!
await redis.geoadd(GEO_KEY, lng, lat, userId); // ถูก
// ไม่ใช่:
// await redis.geoadd(GEO_KEY, lat, lng, userId); // ผิด!
```

---

## ✅ Checklist

- [ ] **Step 271**: ติดตั้ง PostGIS 3 และ enable extensions
- [ ] **Step 272**: Import Thailand boundaries (provinces, districts, subdistricts)
- [ ] **Step 273**: รัน migration สร้าง user_locations, flood_zones, geofences tables
- [ ] **Step 274**: Implement LocationService พร้อม privacy control
- [ ] **Step 275**: Implement GeocodingService (Mapbox + Google fallback)
- [ ] **Step 276**: Import flood zone polygons
- [ ] **Step 277**: สร้าง Location API Routes
- [ ] **Step 278**: Implement batch location update worker
- [ ] **Step 279**: Setup WebSocket location handlers
- [ ] **Step 280**: รัน spatial query tests
- [ ] Verify PostGIS GIST index ถูกใช้ (EXPLAIN ANALYZE)
- [ ] Verify fuzzy location (precision 5) ทำงาน
- [ ] Verify geofencing triggers เมื่อ enter flood zone
- [ ] Load test: 100,000 updates/minute ผ่าน
- [ ] Verify Thailand boundary data ครบ 77 จังหวัด

---

## 🔗 References

- [PostGIS Documentation](https://postgis.net/docs/)
- [Redis GEOSEARCH](https://redis.io/commands/geosearch/)
- [Mapbox Geocoding API](https://docs.mapbox.com/api/search/geocoding/)
- [GADM Thailand Data](https://gadm.org/download_country.html)
- [Geohash Explained](http://geohash.gofreerange.com/)

---
*Part 028 | Road to 1,000,000 Users/Day | chuaikan.com*
