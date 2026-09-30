# Part 027: SOS Service (Real-time Emergency)
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 261-270
> **เวลาโดยประมาณ:** 6 ชั่วโมง
> **Prerequisites:** Part 026 (Feed Service), Part 023 (Notification Service), Part 024 (WebSocket/Socket.io)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ออกแบบ SOS System Architecture สำหรับ chuaikan.com (น้ำท่วม/ไฟไหม้/อุบัติเหตุ)
- สร้าง SOS Alert Schema ที่ครบถ้วน
- Broadcast Real-time Alerts ด้วย Socket.io Rooms แบ่งตามพื้นที่
- ระบบ Escalation: User Report → Moderator Verify → Official Alert
- Geographic Routing ด้วย PostGIS ST_DWithin
- Push Notification Blast: 10,000 users ใน 5 วินาที
- SOS History และ Audit Log
- Severity Levels: INFO, WARNING, DANGER, CRITICAL
- SOS Confirmation System (ยืนยันว่าปลอดภัย)
- Integration กับ API ของหน่วยงานรัฐ (กรมอุตุ/กรมชลประทาน)
- รับมือ 1,000 SOS Alerts พร้อมกัน

---

## 📖 ทฤษฎีและแนวคิด

### SOS System Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                     SOS ALERT FLOW                               │
│                                                                  │
│  User Reports          Moderator           Official              │
│  ┌──────────┐          ┌────────┐          ┌─────────┐          │
│  │ User App │ ──POST─> │ Queue  │ ──verify─> │ Alert   │          │
│  └──────────┘          └────────┘          │ Active  │          │
│                        │                   └────┬────┘          │
│  Gov APIs              │ 3 reports              │               │
│  ┌──────────┐          │ in 10 min              │ Broadcast     │
│  │ HYDRO    │ ──auto──>└────────────────────────┤               │
│  │ METEO    │                                   │               │
│  └──────────┘                           ┌───────▼──────┐       │
│                                         │ Socket.io     │       │
│                                         │ Rooms by Area │       │
│                                         └───────┬───────┘       │
│                                                 │               │
│                                    ┌────────────┼────────────┐  │
│                                    ▼            ▼            ▼  │
│                               Bangkok      Chiang Mai    Phuket │
│                               Room         Room          Room   │
└──────────────────────────────────────────────────────────────────┘
```

### SOS Severity Levels

```
SEVERITY LEVELS
───────────────────────────────────────────────────────
CRITICAL  (สีแดง)   ชีวิตตกอยู่ในอันตราย ต้องการความช่วยเหลือทันที
                    Example: น้ำท่วมสูง >2m, ไฟไหม้ลุกลาม
                    Action:  Push notification ทุกคนในพื้นที่
                             Inject top of ALL feeds
                             SMS backup

DANGER    (สีส้ม)   สถานการณ์อันตราย ควรอพยพ
                    Example: น้ำเพิ่มขึ้นเร็ว, ถนนถูกตัด
                    Action:  Push notification คนในรัศมี 10km

WARNING   (สีเหลือง) ควรเฝ้าระวัง
                    Example: ฝนหนัก, น้ำเริ่มขึ้น
                    Action:  In-app notification เท่านั้น

INFO      (สีฟ้า)   ข้อมูลทั่วไป
                    Example: ถนนลื่น, จุดรับบริจาค
                    Action:  Feed injection เท่านั้น
```

### Escalation Pipeline

```
User Report (1 คน)
    │
    ▼
pending_review  ──── Moderator ────► verified
    │
    │  Auto-escalate: 3+ reports in 10 min
    ▼
auto_verified ──────────────────────► active (broadcast)
    │
    │  Government API confirmation
    ▼
official ────────────────────────────► CRITICAL broadcast
```

---

## ⚙️ Environment Setup

### Step 261: PostGIS Setup

```bash
# ติดตั้ง PostGIS บน Ubuntu 24.04
sudo apt-get install -y postgresql-17-postgis-3

# Enable PostGIS extension
psql -U postgres -d chuaikan_db -c "CREATE EXTENSION IF NOT EXISTS postgis;"
psql -U postgres -d chuaikan_db -c "CREATE EXTENSION IF NOT EXISTS postgis_topology;"

# ตรวจสอบ
psql -U postgres -d chuaikan_db -c "SELECT PostGIS_version();"
```

### Step 262: SOS Database Schema

```sql
-- /home/user/chuaikan/db/migrations/021_sos_tables.sql

-- ตาราง SOS Alerts หลัก
CREATE TABLE sos_alerts (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  alert_id        VARCHAR(20) UNIQUE NOT NULL,  -- human-readable: SOS-2024-001
  type            VARCHAR(30) NOT NULL,         -- flood, fire, accident, storm, other
  severity        VARCHAR(20) NOT NULL,         -- INFO, WARNING, DANGER, CRITICAL
  title           VARCHAR(200) NOT NULL,
  description     TEXT,
  lat             DECIMAL(10, 7) NOT NULL,
  lng             DECIMAL(11, 7) NOT NULL,
  location        GEOGRAPHY(POINT, 4326),       -- PostGIS point
  radius_km       DECIMAL(8, 2) DEFAULT 5.0,
  affected_area   GEOGRAPHY(POLYGON, 4326),     -- optional polygon
  province_code   VARCHAR(10),
  district_code   VARCHAR(10),
  status          VARCHAR(20) DEFAULT 'pending_review',
  -- pending_review, verified, active, official, resolved, false_alarm
  source          VARCHAR(20) DEFAULT 'user',   -- user, moderator, government, system
  reporter_id     UUID REFERENCES users(id),
  verified_by     UUID REFERENCES users(id),
  report_count    INT DEFAULT 1,
  confirm_safe_count INT DEFAULT 0,
  media_urls      TEXT[] DEFAULT '{}',
  external_ref    VARCHAR(100),    -- รหัสจากระบบภายนอก
  metadata        JSONB DEFAULT '{}',
  expires_at      TIMESTAMPTZ,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  resolved_at     TIMESTAMPTZ
);

-- Trigger อัปเดต location จาก lat/lng
CREATE OR REPLACE FUNCTION update_sos_location()
RETURNS TRIGGER AS $$
BEGIN
  NEW.location = ST_MakePoint(NEW.lng, NEW.lat)::geography;
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sos_location
  BEFORE INSERT OR UPDATE ON sos_alerts
  FOR EACH ROW EXECUTE FUNCTION update_sos_location();

-- Audit Log
CREATE TABLE sos_audit_log (
  id          BIGSERIAL PRIMARY KEY,
  alert_id    UUID NOT NULL REFERENCES sos_alerts(id),
  action      VARCHAR(50) NOT NULL,
  actor_id    UUID,
  old_status  VARCHAR(20),
  new_status  VARCHAR(20),
  notes       TEXT,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- User SOS Reports (หลายคนรายงานเหตุการณ์เดียวกัน)
CREATE TABLE sos_reports (
  id          BIGSERIAL PRIMARY KEY,
  alert_id    UUID NOT NULL REFERENCES sos_alerts(id),
  user_id     UUID NOT NULL REFERENCES users(id),
  lat         DECIMAL(10, 7),
  lng         DECIMAL(11, 7),
  description TEXT,
  media_urls  TEXT[] DEFAULT '{}',
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(alert_id, user_id)
);

-- Safe Confirmations
CREATE TABLE sos_safe_confirmations (
  id          BIGSERIAL PRIMARY KEY,
  alert_id    UUID NOT NULL REFERENCES sos_alerts(id),
  user_id     UUID NOT NULL REFERENCES users(id),
  confirmed_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(alert_id, user_id)
);

-- Indexes
CREATE INDEX idx_sos_location    ON sos_alerts USING GIST(location);
CREATE INDEX idx_sos_status      ON sos_alerts(status, created_at DESC);
CREATE INDEX idx_sos_severity    ON sos_alerts(severity, status);
CREATE INDEX idx_sos_province    ON sos_alerts(province_code, status);
CREATE INDEX idx_sos_audit_alert ON sos_audit_log(alert_id, created_at DESC);
```

```bash
psql -U chuaikan_user -d chuaikan_db \
  -f /home/user/chuaikan/db/migrations/021_sos_tables.sql
```

---

## 🛠️ Step-by-Step Implementation

### Step 263: SOS Service Core Types

```typescript
// src/types/sos.types.ts

export type SOSType = 'flood' | 'fire' | 'accident' | 'storm' | 'other';
export type SOSSeverity = 'INFO' | 'WARNING' | 'DANGER' | 'CRITICAL';
export type SOSStatus =
  | 'pending_review'
  | 'verified'
  | 'active'
  | 'official'
  | 'resolved'
  | 'false_alarm';
export type SOSSource = 'user' | 'moderator' | 'government' | 'system';

export interface SOSAlert {
  id: string;
  alertId: string;      // Human-readable: SOS-2024-001
  type: SOSType;
  severity: SOSSeverity;
  title: string;
  description?: string;
  lat: number;
  lng: number;
  radiusKm: number;
  provinceCode?: string;
  districtCode?: string;
  status: SOSStatus;
  source: SOSSource;
  reporterId?: string;
  reportCount: number;
  confirmSafeCount: number;
  mediaUrls: string[];
  expiresAt?: Date;
  createdAt: Date;
  updatedAt: Date;
}

export interface CreateSOSRequest {
  type: SOSType;
  severity: SOSSeverity;
  title: string;
  description?: string;
  lat: number;
  lng: number;
  radiusKm?: number;
  mediaUrls?: string[];
}

export interface SOSBroadcastPayload {
  alertId: string;
  type: SOSType;
  severity: SOSSeverity;
  title: string;
  lat: number;
  lng: number;
  radiusKm: number;
  status: SOSStatus;
  timestamp: string;
}

export interface GOVFloodData {
  stationId: string;
  stationName: string;
  province: string;
  waterLevel: number;     // เมตร
  dangerLevel: number;    // ระดับอันตราย
  warningLevel: number;   // ระดับเตือน
  isOverflowing: boolean;
  lat: number;
  lng: number;
  recordedAt: Date;
}
```

### Step 264: SOS Alert Service

```typescript
// src/services/sos-alert.service.ts

import { Pool } from 'pg';
import Redis from 'ioredis';
import { Server as SocketServer } from 'socket.io';
import { v4 as uuidv4 } from 'uuid';
import {
  SOSAlert, CreateSOSRequest, SOSSeverity, SOSStatus
} from '../types/sos.types';

export class SOSAlertService {
  private readonly AUTO_ESCALATE_COUNT = 3;    // รายงาน 3 คน → auto verify
  private readonly AUTO_ESCALATE_MINUTES = 10; // ภายใน 10 นาที

  constructor(
    private db: Pool,
    private redis: Redis,
    private io: SocketServer
  ) {}

  // สร้าง SOS Alert ใหม่
  async createAlert(
    userId: string,
    data: CreateSOSRequest
  ): Promise<SOSAlert> {
    const alertId = await this.generateAlertId();

    const result = await this.db.query(`
      INSERT INTO sos_alerts
        (id, alert_id, type, severity, title, description,
         lat, lng, radius_km, status, source, reporter_id,
         media_urls, expires_at)
      VALUES
        ($1, $2, $3, $4, $5, $6, $7, $8, $9,
         'pending_review', 'user', $10, $11,
         NOW() + INTERVAL '24 hours')
      RETURNING *
    `, [
      uuidv4(), alertId,
      data.type, data.severity, data.title, data.description,
      data.lat, data.lng, data.radiusKm ?? 5.0,
      userId, data.mediaUrls ?? [],
    ]);

    const alert = this.mapRow(result.rows[0]);

    // Log audit
    await this.logAudit(alert.id, 'created', userId, null, 'pending_review');

    // ตรวจสอบว่า severity CRITICAL → broadcast ทันที
    if (data.severity === 'CRITICAL') {
      await this.escalateAlert(alert.id, userId, 'critical_severity');
    } else {
      // Check auto-escalation threshold
      await this.checkAutoEscalation(alert.id);
    }

    return alert;
  }

  // รายงาน SOS เพิ่มเติม (คนอื่นเห็นด้วย)
  async reportAlert(
    alertId: string,
    userId: string,
    lat: number,
    lng: number,
    description?: string
  ): Promise<void> {
    await this.db.query(`
      INSERT INTO sos_reports (alert_id, user_id, lat, lng, description)
      VALUES ($1, $2, $3, $4, $5)
      ON CONFLICT (alert_id, user_id) DO NOTHING
    `, [alertId, userId, lat, lng, description]);

    // Update report count
    await this.db.query(`
      UPDATE sos_alerts
      SET report_count = report_count + 1
      WHERE id = $1
    `, [alertId]);

    await this.checkAutoEscalation(alertId);
  }

  // Moderator verify alert
  async verifyAlert(
    alertId: string,
    moderatorId: string,
    notes?: string
  ): Promise<void> {
    const alert = await this.getAlertById(alertId);
    if (!alert) throw new Error('Alert not found');

    await this.db.query(`
      UPDATE sos_alerts
      SET status = 'verified', verified_by = $2
      WHERE id = $1
    `, [alertId, moderatorId]);

    await this.logAudit(alertId, 'verified', moderatorId,
      alert.status, 'verified', notes);

    // Broadcast verified alert
    await this.broadcastAlert(alertId);
  }

  // Escalate to active (broadcast)
  async escalateAlert(
    alertId: string,
    actorId: string,
    reason: string
  ): Promise<void> {
    const alert = await this.getAlertById(alertId);
    if (!alert) return;
    if (['resolved', 'false_alarm'].includes(alert.status)) return;

    const newStatus: SOSStatus = 'active';

    await this.db.query(`
      UPDATE sos_alerts SET status = $2 WHERE id = $1
    `, [alertId, newStatus]);

    await this.logAudit(alertId, 'escalated', actorId,
      alert.status, newStatus, reason);

    // Broadcast to nearby users
    await this.broadcastAlert(alertId);

    // Push notification blast สำหรับ DANGER/CRITICAL
    if (['DANGER', 'CRITICAL'].includes(alert.severity)) {
      await this.sendPushNotificationBlast(alertId);
    }

    // Inject ใน feed
    await this.redis.lpush('sos-feed-inject', alertId);
  }

  // User ยืนยันว่าปลอดภัย
  async confirmSafe(alertId: string, userId: string): Promise<void> {
    await this.db.query(`
      INSERT INTO sos_safe_confirmations (alert_id, user_id)
      VALUES ($1, $2)
      ON CONFLICT DO NOTHING
    `, [alertId, userId]);

    const result = await this.db.query(`
      UPDATE sos_alerts
      SET confirm_safe_count = confirm_safe_count + 1
      WHERE id = $1
      RETURNING confirm_safe_count
    `, [alertId]);

    const count = result.rows[0]?.confirm_safe_count;

    // Emit safe confirmation event
    this.io.to(`sos-${alertId}`).emit('sos:safe_confirmation', {
      alertId,
      confirmCount: count,
      userId,
    });
  }

  // Resolve alert
  async resolveAlert(
    alertId: string,
    actorId: string,
    notes?: string
  ): Promise<void> {
    const alert = await this.getAlertById(alertId);
    if (!alert) return;

    await this.db.query(`
      UPDATE sos_alerts
      SET status = 'resolved', resolved_at = NOW()
      WHERE id = $1
    `, [alertId]);

    await this.logAudit(alertId, 'resolved', actorId,
      alert.status, 'resolved', notes);

    // Notify subscribers
    this.io.to(`sos-area-${alert.provinceCode ?? 'TH'}`).emit('sos:resolved', {
      alertId,
      resolvedAt: new Date().toISOString(),
    });

    // Remove from feeds
    await this.redis.lpush('sos-feed-remove', alertId);
  }

  // Broadcast alert to nearby users via Socket.io
  private async broadcastAlert(alertId: string): Promise<void> {
    const alert = await this.getAlertById(alertId);
    if (!alert) return;

    const payload = {
      alertId: alert.alertId,
      id: alert.id,
      type: alert.type,
      severity: alert.severity,
      title: alert.title,
      lat: alert.lat,
      lng: alert.lng,
      radiusKm: alert.radiusKm,
      status: alert.status,
      timestamp: new Date().toISOString(),
    };

    // Broadcast to province room
    if (alert.provinceCode) {
      this.io
        .to(`sos-province-${alert.provinceCode}`)
        .emit('sos:new_alert', payload);
    }

    // Broadcast to national room (สำหรับ CRITICAL)
    if (alert.severity === 'CRITICAL') {
      this.io.to('sos-national').emit('sos:critical_alert', payload);
    }

    // Store in Redis for new socket connections
    await this.redis.setex(
      `sos:active:${alertId}`,
      86400,
      JSON.stringify(payload)
    );

    console.log(
      `Broadcast SOS ${alertId} (${alert.severity}) to rooms`
    );
  }

  // Push Notification Blast: 10,000 users in 5 seconds
  private async sendPushNotificationBlast(alertId: string): Promise<void> {
    const alert = await this.getAlertById(alertId);
    if (!alert) return;

    // หา users ใน radius
    const users = await this.db.query(`
      SELECT ul.user_id, u.push_token
      FROM user_locations ul
      JOIN users u ON u.id = ul.user_id
      WHERE ST_DWithin(
        ul.location::geography,
        ST_MakePoint($2, $1)::geography,
        $3 * 1000
      )
        AND u.push_token IS NOT NULL
        AND u.push_notifications_enabled = true
      LIMIT 10000
    `, [alert.lat, alert.lng, alert.radiusKm]);

    if (users.rows.length === 0) return;

    // Queue ไปยัง notification service (batch 500 tokens)
    const tokens: string[] = users.rows.map((r: any) => r.push_token);
    const batchSize = 500;

    for (let i = 0; i < tokens.length; i += batchSize) {
      const batch = tokens.slice(i, i + batchSize);
      await this.redis.lpush('push-notification-blast', JSON.stringify({
        tokens: batch,
        title: `🚨 แจ้งเตือน SOS: ${alert.title}`,
        body: alert.description ?? `${alert.type} ในพื้นที่ของคุณ`,
        data: {
          type: 'sos_alert',
          alertId: alert.id,
          severity: alert.severity,
        },
        priority: 'high',
      }));
    }

    console.log(
      `Queued push notification blast for ${tokens.length} users`
    );
  }

  // Check auto-escalation: 3 reports ใน 10 นาที
  private async checkAutoEscalation(alertId: string): Promise<void> {
    const result = await this.db.query(`
      SELECT COUNT(*) AS recent_reports
      FROM sos_reports
      WHERE alert_id = $1
        AND created_at > NOW() - INTERVAL '${this.AUTO_ESCALATE_MINUTES} minutes'
    `, [alertId]);

    const recentReports = parseInt(result.rows[0]?.recent_reports);

    if (recentReports >= this.AUTO_ESCALATE_COUNT) {
      const alert = await this.getAlertById(alertId);
      if (alert?.status === 'pending_review') {
        await this.escalateAlert(
          alertId,
          'system',
          `Auto-escalated: ${recentReports} reports in ${this.AUTO_ESCALATE_MINUTES} minutes`
        );
      }
    }
  }

  private async getAlertById(alertId: string): Promise<SOSAlert | null> {
    const result = await this.db.query(
      'SELECT * FROM sos_alerts WHERE id = $1',
      [alertId]
    );
    return result.rows.length > 0 ? this.mapRow(result.rows[0]) : null;
  }

  private async generateAlertId(): Promise<string> {
    const date = new Date();
    const year = date.getFullYear();
    const result = await this.db.query(
      "SELECT NEXTVAL('sos_alert_seq') AS seq"
    );
    const seq = result.rows[0].seq.toString().padStart(6, '0');
    return `SOS-${year}-${seq}`;
  }

  private async logAudit(
    alertId: string,
    action: string,
    actorId: string,
    oldStatus: string | null,
    newStatus: string,
    notes?: string
  ): Promise<void> {
    await this.db.query(`
      INSERT INTO sos_audit_log
        (alert_id, action, actor_id, old_status, new_status, notes)
      VALUES ($1, $2, $3, $4, $5, $6)
    `, [alertId, action, actorId, oldStatus, newStatus, notes]);
  }

  private mapRow(row: any): SOSAlert {
    return {
      id: row.id,
      alertId: row.alert_id,
      type: row.type,
      severity: row.severity,
      title: row.title,
      description: row.description,
      lat: parseFloat(row.lat),
      lng: parseFloat(row.lng),
      radiusKm: parseFloat(row.radius_km),
      provinceCode: row.province_code,
      districtCode: row.district_code,
      status: row.status,
      source: row.source,
      reporterId: row.reporter_id,
      reportCount: row.report_count,
      confirmSafeCount: row.confirm_safe_count,
      mediaUrls: row.media_urls ?? [],
      expiresAt: row.expires_at,
      createdAt: row.created_at,
      updatedAt: row.updated_at,
    };
  }
}
```

### Step 265: Geographic Alert Routing

```typescript
// src/services/geo-router.ts

import { Pool } from 'pg';
import { Server as SocketServer } from 'socket.io';

export class GeoRouter {
  constructor(
    private db: Pool,
    private io: SocketServer
  ) {}

  // หา users ที่อยู่ใน radius ของ alert (PostGIS)
  async findUsersInRadius(
    lat: number,
    lng: number,
    radiusKm: number,
    limit: number = 10000
  ): Promise<string[]> {
    const result = await this.db.query(`
      SELECT ul.user_id
      FROM user_locations ul
      WHERE ST_DWithin(
        ul.location::geography,
        ST_MakePoint($2, $1)::geography,
        $3 * 1000
      )
        AND ul.updated_at > NOW() - INTERVAL '1 hour'
      LIMIT $4
    `, [lat, lng, radiusKm, limit]);

    return result.rows.map((r: any) => r.user_id);
  }

  // Broadcast SOS ไปยัง users ตาม Socket.io rooms
  async broadcastToRoom(
    provinceCode: string,
    event: string,
    data: any
  ): Promise<void> {
    const roomName = `sos-province-${provinceCode}`;
    const socketsInRoom = await this.io.in(roomName).fetchSockets();

    this.io.to(roomName).emit(event, data);

    console.log(
      `Broadcast ${event} to room ${roomName}: ${socketsInRoom.length} connections`
    );
  }

  // Setup Socket.io room handlers
  setupRoomHandlers(): void {
    this.io.on('connection', (socket) => {
      const userId = socket.handshake.auth.userId;
      const provinceCode = socket.handshake.auth.provinceCode;

      if (provinceCode) {
        socket.join(`sos-province-${provinceCode}`);
      }

      // Join national room
      socket.join('sos-national');

      // Subscribe to specific alert
      socket.on('sos:subscribe', (alertId: string) => {
        socket.join(`sos-${alertId}`);
      });

      socket.on('sos:unsubscribe', (alertId: string) => {
        socket.leave(`sos-${alertId}`);
      });

      // Update location → update room membership
      socket.on('location:update', async ({ lat, lng, provinceCode: newProvince }) => {
        if (newProvince && newProvince !== provinceCode) {
          socket.leave(`sos-province-${provinceCode}`);
          socket.join(`sos-province-${newProvince}`);
        }
      });
    });
  }

  // Query alerts ใกล้ user
  async getAlertsNearUser(
    lat: number,
    lng: number,
    radiusKm: number = 50
  ): Promise<any[]> {
    const result = await this.db.query(`
      SELECT
        id, alert_id, type, severity, title,
        lat, lng, radius_km, status, created_at,
        ST_Distance(
          location::geography,
          ST_MakePoint($2, $1)::geography
        ) / 1000 AS distance_km
      FROM sos_alerts
      WHERE ST_DWithin(
        location::geography,
        ST_MakePoint($2, $1)::geography,
        $3 * 1000
      )
        AND status IN ('active', 'official', 'verified')
        AND (expires_at IS NULL OR expires_at > NOW())
      ORDER BY severity DESC, distance_km ASC
      LIMIT 50
    `, [lat, lng, radiusKm]);

    return result.rows;
  }
}
```

### Step 266: Government API Integration

```typescript
// src/services/gov-api.service.ts
// ดึงข้อมูลน้ำท่วมจากกรมชลประทานและกรมอุตุนิยมวิทยา

import axios from 'axios';
import { Pool } from 'pg';
import { SOSAlertService } from './sos-alert.service';
import { GOVFloodData } from '../types/sos.types';

const GOV_API_URLS = {
  HYDRO: process.env.HYDRO_API_URL ?? 'https://www.rid.go.th/thaiwrd/',
  METEO: process.env.METEO_API_URL ?? 'https://data.tmd.go.th/api/WeatherLocation',
  FLOOD_ZONE: process.env.FLOOD_API_URL ?? 'https://flood.gistda.or.th/api',
};

export class GovApiService {
  private readonly POLL_INTERVAL_MS = 5 * 60 * 1000; // 5 นาที

  constructor(
    private db: Pool,
    private sosService: SOSAlertService
  ) {}

  // เริ่ม polling government APIs
  startPolling(): void {
    this.pollHydroData();
    setInterval(() => this.pollHydroData(), this.POLL_INTERVAL_MS);
  }

  // ดึงข้อมูลระดับน้ำจากกรมชลประทาน
  async pollHydroData(): Promise<void> {
    try {
      const response = await axios.get(GOV_API_URLS.HYDRO, {
        params: {
          format: 'json',
          station: 'all',
        },
        timeout: 30000,
        headers: {
          'User-Agent': 'chuaikan-monitoring/1.0',
        },
      });

      const stations: GOVFloodData[] = this.parseHydroResponse(response.data);

      for (const station of stations) {
        await this.processHydroStation(station);
      }

      console.log(`Processed ${stations.length} hydro stations`);
    } catch (error) {
      console.error('Hydro API error:', error);
    }
  }

  // ตรวจสอบว่าระดับน้ำอันตรายหรือไม่ → สร้าง SOS alert
  private async processHydroStation(data: GOVFloodData): Promise<void> {
    // เช็คว่ามี alert อยู่แล้วหรือเปล่า
    const existing = await this.db.query(`
      SELECT id FROM sos_alerts
      WHERE external_ref = $1
        AND status NOT IN ('resolved', 'false_alarm')
        AND created_at > NOW() - INTERVAL '12 hours'
    `, [`hydro-${data.stationId}`]);

    if (data.isOverflowing && existing.rows.length === 0) {
      // สร้าง official SOS alert
      const severity = data.waterLevel > data.dangerLevel * 1.5
        ? 'CRITICAL' : 'DANGER';

      await this.sosService.createAlert('system', {
        type: 'flood',
        severity,
        title: `น้ำท่วมที่ ${data.stationName}`,
        description: `ระดับน้ำ ${data.waterLevel.toFixed(2)}m (ระดับอันตราย: ${data.dangerLevel}m)`,
        lat: data.lat,
        lng: data.lng,
        radiusKm: 20,
      });
    } else if (!data.isOverflowing && existing.rows.length > 0) {
      // Auto-resolve เมื่อน้ำลด
      await this.sosService.resolveAlert(
        existing.rows[0].id,
        'system',
        `Water level normalized: ${data.waterLevel.toFixed(2)}m`
      );
    }
  }

  private parseHydroResponse(data: any): GOVFloodData[] {
    // Parse ตาม format ของ API จริง
    // ตัวอย่าง mock:
    if (!Array.isArray(data?.stations)) return [];

    return data.stations.map((s: any) => ({
      stationId: s.id ?? s.station_id,
      stationName: s.name ?? s.station_name,
      province: s.province,
      waterLevel: parseFloat(s.water_level ?? 0),
      dangerLevel: parseFloat(s.danger_level ?? 999),
      warningLevel: parseFloat(s.warning_level ?? 999),
      isOverflowing: parseFloat(s.water_level ?? 0) >= parseFloat(s.danger_level ?? 999),
      lat: parseFloat(s.lat),
      lng: parseFloat(s.lng),
      recordedAt: new Date(s.recorded_at ?? Date.now()),
    }));
  }
}
```

### Step 267: SOS API Routes

```typescript
// src/routes/sos.routes.ts

import { Router, Request, Response } from 'express';
import { SOSAlertService } from '../services/sos-alert.service';
import { GeoRouter } from '../services/geo-router';
import { z } from 'zod';

const CreateSOSSchema = z.object({
  type: z.enum(['flood', 'fire', 'accident', 'storm', 'other']),
  severity: z.enum(['INFO', 'WARNING', 'DANGER', 'CRITICAL']),
  title: z.string().min(5).max(200),
  description: z.string().max(2000).optional(),
  lat: z.number().min(-90).max(90),
  lng: z.number().min(-180).max(180),
  radiusKm: z.number().min(0.1).max(100).optional(),
  mediaUrls: z.array(z.string().url()).max(5).optional(),
});

export function createSOSRouter(
  sosService: SOSAlertService,
  geoRouter: GeoRouter
): Router {
  const router = Router();

  // POST /api/v1/sos - รายงาน SOS ใหม่
  router.post('/', async (req: Request, res: Response) => {
    try {
      const data = CreateSOSSchema.parse(req.body);
      const userId = req.user!.id;

      const alert = await sosService.createAlert(userId, data);
      res.status(201).json(alert);
    } catch (error: any) {
      if (error.name === 'ZodError') {
        return res.status(400).json({ error: error.errors });
      }
      res.status(500).json({ error: 'Failed to create SOS alert' });
    }
  });

  // GET /api/v1/sos/nearby - SOS ใกล้ฉัน
  router.get('/nearby', async (req: Request, res: Response) => {
    const lat = parseFloat(req.query.lat as string);
    const lng = parseFloat(req.query.lng as string);
    const radius = parseFloat(req.query.radius as string) || 50;

    if (isNaN(lat) || isNaN(lng)) {
      return res.status(400).json({ error: 'Invalid lat/lng' });
    }

    const alerts = await geoRouter.getAlertsNearUser(lat, lng, radius);
    res.json({ alerts, count: alerts.length });
  });

  // POST /api/v1/sos/:id/confirm-safe
  router.post('/:id/confirm-safe', async (req: Request, res: Response) => {
    const { id } = req.params;
    const userId = req.user!.id;

    await sosService.confirmSafe(id, userId);
    res.json({ ok: true, message: 'ขอบคุณที่แจ้งว่าคุณปลอดภัย' });
  });

  // POST /api/v1/sos/:id/report - รายงานเหตุการณ์เดิม
  router.post('/:id/report', async (req: Request, res: Response) => {
    const { id } = req.params;
    const { lat, lng, description } = req.body;
    const userId = req.user!.id;

    await sosService.reportAlert(id, userId, lat, lng, description);
    res.json({ ok: true });
  });

  // PATCH /api/v1/sos/:id/verify (Moderator only)
  router.patch('/:id/verify', async (req: Request, res: Response) => {
    if (!req.user!.roles.includes('moderator')) {
      return res.status(403).json({ error: 'Forbidden' });
    }

    const { id } = req.params;
    const { notes } = req.body;
    const userId = req.user!.id;

    await sosService.verifyAlert(id, userId, notes);
    res.json({ ok: true });
  });

  // PATCH /api/v1/sos/:id/resolve
  router.patch('/:id/resolve', async (req: Request, res: Response) => {
    if (!req.user!.roles.includes('moderator')) {
      return res.status(403).json({ error: 'Forbidden' });
    }

    const { id } = req.params;
    const { notes } = req.body;
    const userId = req.user!.id;

    await sosService.resolveAlert(id, userId, notes);
    res.json({ ok: true });
  });

  // GET /api/v1/sos/:id/history - Audit log
  router.get('/:id/history', async (req: Request, res: Response) => {
    const { id } = req.params;

    const result = await req.db.query(`
      SELECT
        sal.action, sal.old_status, sal.new_status,
        sal.notes, sal.created_at,
        u.display_name AS actor_name
      FROM sos_audit_log sal
      LEFT JOIN users u ON u.id = sal.actor_id
      WHERE sal.alert_id = $1
      ORDER BY sal.created_at ASC
    `, [id]);

    res.json({ history: result.rows });
  });

  return router;
}
```

### Step 268: SOS Dashboard (Emergency Responders)

```typescript
// src/dashboard/sos-dashboard.service.ts

import { Pool } from 'pg';
import Redis from 'ioredis';

export interface SOSDashboardStats {
  activeAlerts: number;
  criticalAlerts: number;
  alertsByProvince: Array<{ province: string; count: number }>;
  alertsByType: Array<{ type: string; count: number }>;
  recentAlerts: any[];
  avgResolutionMinutes: number;
}

export class SOSDashboardService {
  constructor(
    private db: Pool,
    private redis: Redis
  ) {}

  async getStats(): Promise<SOSDashboardStats> {
    const [
      activeCounts,
      byProvince,
      byType,
      recentAlerts,
      avgResolution,
    ] = await Promise.all([
      this.getActiveCounts(),
      this.getAlertsByProvince(),
      this.getAlertsByType(),
      this.getRecentAlerts(),
      this.getAvgResolutionTime(),
    ]);

    return {
      activeAlerts: activeCounts.total,
      criticalAlerts: activeCounts.critical,
      alertsByProvince: byProvince,
      alertsByType: byType,
      recentAlerts,
      avgResolutionMinutes: avgResolution,
    };
  }

  private async getActiveCounts(): Promise<{ total: number; critical: number }> {
    const result = await this.db.query(`
      SELECT
        COUNT(*) AS total,
        COUNT(*) FILTER (WHERE severity = 'CRITICAL') AS critical
      FROM sos_alerts
      WHERE status IN ('active', 'official', 'verified')
        AND (expires_at IS NULL OR expires_at > NOW())
    `);

    return {
      total: parseInt(result.rows[0]?.total ?? 0),
      critical: parseInt(result.rows[0]?.critical ?? 0),
    };
  }

  private async getAlertsByProvince(): Promise<any[]> {
    const result = await this.db.query(`
      SELECT
        province_code,
        COUNT(*) AS alert_count,
        MAX(severity) AS max_severity
      FROM sos_alerts
      WHERE status IN ('active', 'official', 'verified')
        AND created_at > NOW() - INTERVAL '24 hours'
      GROUP BY province_code
      ORDER BY alert_count DESC
      LIMIT 20
    `);
    return result.rows;
  }

  private async getAlertsByType(): Promise<any[]> {
    const result = await this.db.query(`
      SELECT type, COUNT(*) AS count
      FROM sos_alerts
      WHERE created_at > NOW() - INTERVAL '24 hours'
      GROUP BY type
      ORDER BY count DESC
    `);
    return result.rows;
  }

  private async getRecentAlerts(): Promise<any[]> {
    const result = await this.db.query(`
      SELECT id, alert_id, type, severity, title, lat, lng, status, created_at
      FROM sos_alerts
      WHERE status IN ('active', 'official', 'verified', 'pending_review')
      ORDER BY
        CASE severity
          WHEN 'CRITICAL' THEN 1
          WHEN 'DANGER' THEN 2
          WHEN 'WARNING' THEN 3
          ELSE 4
        END,
        created_at DESC
      LIMIT 50
    `);
    return result.rows;
  }

  private async getAvgResolutionTime(): Promise<number> {
    const result = await this.db.query(`
      SELECT AVG(
        EXTRACT(EPOCH FROM (resolved_at - created_at)) / 60
      ) AS avg_minutes
      FROM sos_alerts
      WHERE resolved_at IS NOT NULL
        AND resolved_at > NOW() - INTERVAL '7 days'
    `);
    return Math.round(result.rows[0]?.avg_minutes ?? 0);
  }

  // Real-time stats สำหรับ dashboard WebSocket
  async streamStats(callback: (stats: SOSDashboardStats) => void): Promise<void> {
    const update = async () => {
      const stats = await this.getStats();
      callback(stats);
    };

    await update();
    setInterval(update, 5000); // refresh ทุก 5 วินาที
  }
}
```

---

## 🔧 Configuration Files

```bash
# /home/user/chuaikan/services/sos-service/.env

NODE_ENV=production
PORT=3005
DB_URL=postgresql://chuaikan_user:password@localhost:5432/chuaikan_db
REDIS_HOST=localhost
REDIS_PORT=6379
SOCKET_PATH=/var/run/chuaikan-sos.sock

# Government APIs
HYDRO_API_URL=https://www.rid.go.th/api
HYDRO_API_KEY=your_hydro_api_key
METEO_API_URL=https://data.tmd.go.th/api
METEO_API_KEY=your_meteo_api_key

# Push Notifications
FCM_SERVER_KEY=your_fcm_key
APNS_KEY=your_apns_key

# Auto-escalation
AUTO_ESCALATE_COUNT=3
AUTO_ESCALATE_MINUTES=10
```

---

## 🧪 Testing

### Step 269: Load Test - 1000 Simultaneous SOS Alerts

```typescript
// tests/sos-load.test.ts

import { describe, it, expect } from '@jest/globals';
import axios from 'axios';

describe('SOS Service Load Test', () => {
  it('should handle 1000 simultaneous SOS alerts', async () => {
    const BASE_URL = 'http://localhost:3005';
    const TOKEN = process.env.TEST_TOKEN!;

    const start = Date.now();

    // สร้าง 1000 SOS requests พร้อมกัน
    const requests = Array.from({ length: 1000 }, (_, i) => ({
      type: ['flood', 'fire', 'accident'][i % 3],
      severity: ['INFO', 'WARNING', 'DANGER'][i % 3],
      title: `Test SOS Alert ${i}`,
      lat: 13.7563 + (Math.random() - 0.5) * 2,
      lng: 100.5018 + (Math.random() - 0.5) * 2,
      radiusKm: 5,
    }));

    const results = await Promise.allSettled(
      requests.map(data =>
        axios.post(`${BASE_URL}/api/v1/sos`, data, {
          headers: { Authorization: `Bearer ${TOKEN}` },
          timeout: 10000,
        })
      )
    );

    const elapsed = Date.now() - start;
    const successful = results.filter(r => r.status === 'fulfilled').length;
    const failed = results.filter(r => r.status === 'rejected').length;

    console.log(`1000 alerts in ${elapsed}ms`);
    console.log(`Success: ${successful}, Failed: ${failed}`);

    expect(successful).toBeGreaterThan(950); // success rate > 95%
    expect(elapsed).toBeLessThan(30000);     // ต้องเสร็จใน 30 วินาที
  });
});
```

```bash
# รัน test
cd /home/user/chuaikan/services/sos-service
npx jest tests/sos-load.test.ts --testTimeout=60000

# ตรวจสอบ logs
journalctl -u chuaikan-sos -f --since "5 minutes ago"
```

### Step 270: Integration Test

```bash
# ทดสอบ full flow
# 1. สร้าง SOS alert
curl -X POST http://localhost:3005/api/v1/sos \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "type": "flood",
    "severity": "DANGER",
    "title": "น้ำท่วมสูง ถนนพระราม 2",
    "lat": 13.6833,
    "lng": 100.4833,
    "radiusKm": 10
  }'

# 2. ตรวจสอบ alerts ใกล้เคียง
curl "http://localhost:3005/api/v1/sos/nearby?lat=13.69&lng=100.48&radius=15" \
  -H "Authorization: Bearer $TOKEN"

# 3. Confirm safe
curl -X POST http://localhost:3005/api/v1/sos/{ALERT_ID}/confirm-safe \
  -H "Authorization: Bearer $TOKEN"

# 4. ดู audit history
curl http://localhost:3005/api/v1/sos/{ALERT_ID}/history \
  -H "Authorization: Bearer $TOKEN"
```

---

## ❌ Common Errors & Solutions

### Error 1: PostGIS Extension Not Found

```
ERROR: function st_dwithin(geography, geography, integer) does not exist
```

```bash
# แก้: ต้องใส่ type cast ::geography
# และ ติดตั้ง PostGIS ให้ถูกต้อง
sudo apt-get install -y postgresql-17-postgis-3

psql -U postgres -c "CREATE EXTENSION postgis;" chuaikan_db
psql -U postgres -c "\dx" chuaikan_db  # ตรวจสอบ
```

### Error 2: Socket.io Room Not Receiving Events

```
Client not receiving sos:new_alert
```

```typescript
// แก้: ตรวจสอบ room join
// Client ต้อง join room ก่อน
socket.emit('join:province', { provinceCode: '10' }); // Bangkok

// Server:
socket.on('join:province', ({ provinceCode }) => {
  socket.join(`sos-province-${provinceCode}`);
  console.log(`${socket.id} joined sos-province-${provinceCode}`);
});
```

### Error 3: Push Notification Blast Too Slow

```
Sending 10,000 push notifications took > 60 seconds
```

```bash
# แก้: ใช้ FCM batch API (500 per request)
# และรัน parallel requests

# ตั้ง concurrency ใน worker
PUSH_BLAST_CONCURRENCY=20  # 20 parallel FCM requests × 500 = 10,000/request
```

---

## ✅ Checklist

- [ ] **Step 261**: ติดตั้ง PostGIS และ enable extension
- [ ] **Step 262**: รัน migration สร้าง sos_alerts, sos_reports, sos_safe_confirmations
- [ ] **Step 263**: สร้าง SOS types ครบถ้วน
- [ ] **Step 264**: Implement SOSAlertService พร้อม escalation logic
- [ ] **Step 265**: Implement GeoRouter ด้วย PostGIS ST_DWithin
- [ ] **Step 266**: Implement GovApiService เชื่อมต่อกรมชลประทาน
- [ ] **Step 267**: สร้าง SOS API Routes ครบ
- [ ] **Step 268**: Implement SOSDashboard สำหรับ emergency responders
- [ ] **Step 269**: Load test 1000 simultaneous alerts
- [ ] **Step 270**: Integration test full flow
- [ ] Verify auto-escalation ทำงาน (3 reports ใน 10 นาที)
- [ ] Verify SOS injected into nearby feeds
- [ ] Verify Push notification blast < 5 seconds สำหรับ 10,000 users
- [ ] Verify audit log ทุก action
- [ ] Verify PostGIS queries ใช้ GIST index (EXPLAIN ANALYZE)

---

## 🔗 References

- [PostGIS Documentation](https://postgis.net/docs/)
- [Socket.io Rooms](https://socket.io/docs/v4/rooms/)
- [Firebase Cloud Messaging Batch](https://firebase.google.com/docs/cloud-messaging/send-message#send-messages-to-multiple-devices)
- [กรมชลประทาน API](https://www.rid.go.th)
- [GISTDA Flood Data](https://www.gistda.or.th)

---
*Part 027 | Road to 1,000,000 Users/Day | chuaikan.com*
