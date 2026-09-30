# Part 047: Event Sourcing & CQRS
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 461–470
> **เวลาโดยประมาณ:** 5 ชั่วโมง
> **Prerequisites:** Part 046 (Caching Advanced), PostgreSQL, TypeScript

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- Event Sourcing concepts กับตัวอย่าง SOS event log ของ chuaikan.com
- ออกแบบ Event Store ใน PostgreSQL
- Aggregate reconstitution จาก events
- CQRS: แยก Read Model ด้วย Projection
- Eventual Consistency implications
- Event Versioning and Migration
- Snapshotting เพื่อ performance
- EventStoreDB alternative
- Saga Pattern สำหรับ distributed transactions (SOS + Notification)

---

## 📖 ทฤษฎีและแนวคิด

### Step 461 — Event Sourcing คืออะไร?

```
Traditional CRUD:
users table: { id: 1, status: 'resolved', updated_at: ... }
→ ไม่รู้ว่า "ใครเปลี่ยน status" "เปลี่ยนเมื่อไหร่" "ผ่าน status อะไรมาก่อน"

Event Sourcing:
sos_events: [
  { type: 'SOS_REPORTED',  at: 10:00, userId: 1, lat: 13.7, lng: 100.5 }
  { type: 'SOS_VERIFIED',  at: 10:05, adminId: 99 }
  { type: 'SOS_DISPATCHED',at: 10:10, teamId: 5, eta: '10:30' }
  { type: 'SOS_RESOLVED',  at: 10:45, resolvedBy: 5 }
]
→ รู้ทุกอย่าง! rebuild สถานะปัจจุบันได้จาก events

ประโยชน์:
1. Audit log ฟรี
2. Replay events เพื่อ rebuild state
3. Time travel (ย้อนกลับไปดูสถานะ ณ เวลาใดก็ได้)
4. Event-driven integration (Pub/Sub)
```

### Step 462 — CQRS (Command Query Responsibility Segregation)

```
Without CQRS:
  API → DB Read/Write → Same table
  → Complex queries กับ write model เดียวกัน

With CQRS:
  Command Side (Write):
    API → Command Handler → Event Store → Publish Events

  Query Side (Read):
    Events → Projection → Read Model (materialized view)
    API → Read Model (denormalized, fast queries)

ตัวอย่าง chuaikan.com:
  Write: POST /sos → SOS_REPORTED event → event store
  Read:  GET /sos/nearby → query sos_read_model table (denormalized)
```

---

## ⚙️ Environment Setup

```bash
# ใช้ PostgreSQL ที่มีอยู่แล้ว
sudo -u postgres psql -d chuaikan_db

# ติดตั้ง dependencies
npm install uuid eventemitter3 @types/uuid
```

---

## 🛠️ Step-by-Step Implementation

### Step 463 — Event Store Design ใน PostgreSQL

```sql
-- สร้าง Event Store table
CREATE TABLE event_store (
    id              UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       VARCHAR(100) NOT NULL,  -- aggregate id (sos_id, user_id)
    stream_type     VARCHAR(50)  NOT NULL,  -- 'sos_report', 'user', 'sensor'
    event_type      VARCHAR(100) NOT NULL,  -- 'SOS_REPORTED', 'SOS_RESOLVED'
    event_version   INTEGER      NOT NULL,  -- version ของ event schema
    sequence_no     BIGINT       NOT NULL,  -- ลำดับ event ใน stream
    occurred_at     TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    correlation_id  UUID,                  -- trace requests
    causation_id    UUID,                  -- event ที่ทำให้เกิด event นี้
    metadata        JSONB        NOT NULL DEFAULT '{}',
    payload         JSONB        NOT NULL DEFAULT '{}',

    -- Unique constraint: ไม่ให้ duplicate sequence ใน stream
    UNIQUE (stream_id, sequence_no)
);

-- Index สำหรับ query by stream
CREATE INDEX idx_event_store_stream
    ON event_store(stream_id, sequence_no);

-- Index สำหรับ real-time projection (ตาม time)
CREATE INDEX idx_event_store_occurred
    ON event_store(occurred_at, stream_type);

-- Index สำหรับ event type queries
CREATE INDEX idx_event_store_type
    ON event_store(event_type, occurred_at DESC);

-- Snapshot table (ประสิทธิภาพ)
CREATE TABLE event_snapshots (
    id          UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id   VARCHAR(100) NOT NULL UNIQUE,
    stream_type VARCHAR(50)  NOT NULL,
    sequence_no BIGINT       NOT NULL,  -- snapshot taken at this sequence
    state       JSONB        NOT NULL,
    created_at  TIMESTAMPTZ  DEFAULT NOW()
);

-- Read Model: SOS reports (denormalized สำหรับ query)
CREATE TABLE sos_read_model (
    sos_id          VARCHAR(100) PRIMARY KEY,
    user_id         BIGINT,
    username        VARCHAR(50),
    description     TEXT,
    status          VARCHAR(30),
    province_id     INTEGER,
    province_name   VARCHAR(100),
    lat             DECIMAL(9, 6),
    lng             DECIMAL(9, 6),
    location        GEOGRAPHY(POINT, 4326),
    dispatched_team VARCHAR(100),
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ,
    event_sequence  BIGINT
);

CREATE INDEX idx_sos_read_status ON sos_read_model(status, created_at DESC);
CREATE INDEX idx_sos_read_location ON sos_read_model USING GIST (location);
```

### Step 464 — Domain Events Definition

```typescript
// domain/events/sos-events.ts

// Base event type
interface DomainEvent {
  eventType: string;
  eventVersion: number;
  streamId: string;
  streamType: string;
  occurredAt: Date;
  correlationId?: string;
  causationId?: string;
  metadata: Record<string, unknown>;
}

// SOS Events
export interface SOSReportedEvent extends DomainEvent {
  eventType: 'SOS_REPORTED';
  eventVersion: 1;
  payload: {
    userId: string;
    username: string;
    description: string;
    lat: number;
    lng: number;
    provinceId: number;
    provinceName: string;
    severity: 'low' | 'medium' | 'high' | 'critical';
    photoUrls: string[];
  };
}

export interface SOSVerifiedEvent extends DomainEvent {
  eventType: 'SOS_VERIFIED';
  eventVersion: 1;
  payload: {
    verifiedByAdminId: string;
    notes: string;
  };
}

export interface SOSDispatchedEvent extends DomainEvent {
  eventType: 'SOS_DISPATCHED';
  eventVersion: 1;
  payload: {
    teamId: string;
    teamName: string;
    eta: string;
    dispatchedAt: string;
  };
}

export interface SOSResolvedEvent extends DomainEvent {
  eventType: 'SOS_RESOLVED';
  eventVersion: 1;
  payload: {
    resolvedByTeamId: string;
    resolutionNotes: string;
    resolvedAt: string;
  };
}

export type SOSEvent =
  | SOSReportedEvent
  | SOSVerifiedEvent
  | SOSDispatchedEvent
  | SOSResolvedEvent;
```

### Step 465 — Event Store Repository

```typescript
// infrastructure/event-store.ts
import { prisma } from '../lib/db/client';
import { v4 as uuidv4 } from 'uuid';

export class EventStore {
  async appendToStream(
    streamId: string,
    streamType: string,
    events: Array<{
      eventType: string;
      eventVersion: number;
      payload: Record<string, unknown>;
      metadata?: Record<string, unknown>;
      correlationId?: string;
      causationId?: string;
    }>,
    expectedSequence?: number  // Optimistic concurrency
  ): Promise<void> {
    // Transaction: append events atomically
    await prisma.$transaction(async (tx) => {
      // ตรวจสอบ sequence ล่าสุด
      const lastEvent = await tx.$queryRaw<{ sequence_no: number }[]>`
        SELECT sequence_no FROM event_store
        WHERE stream_id = ${streamId}
        ORDER BY sequence_no DESC
        LIMIT 1
      `;

      const currentSequence = lastEvent[0]?.sequence_no ?? 0;

      // Optimistic concurrency check
      if (expectedSequence !== undefined && currentSequence !== expectedSequence) {
        throw new Error(
          `Concurrency conflict: expected ${expectedSequence}, ` +
          `but got ${currentSequence}`
        );
      }

      // Append events
      let sequence = currentSequence;
      for (const event of events) {
        sequence++;
        await tx.$executeRaw`
          INSERT INTO event_store (
            id, stream_id, stream_type, event_type, event_version,
            sequence_no, occurred_at, correlation_id, causation_id,
            metadata, payload
          ) VALUES (
            ${uuidv4()},
            ${streamId},
            ${streamType},
            ${event.eventType},
            ${event.eventVersion},
            ${sequence},
            NOW(),
            ${event.correlationId ?? null},
            ${event.causationId ?? null},
            ${JSON.stringify(event.metadata ?? {})}::jsonb,
            ${JSON.stringify(event.payload)}::jsonb
          )
        `;
      }
    });
  }

  async loadStream(
    streamId: string,
    fromSequence: number = 0
  ): Promise<any[]> {
    return prisma.$queryRaw`
      SELECT * FROM event_store
      WHERE stream_id = ${streamId}
        AND sequence_no > ${fromSequence}
      ORDER BY sequence_no ASC
    `;
  }

  async loadStreamByType(
    streamType: string,
    fromTime?: Date
  ): Promise<any[]> {
    if (fromTime) {
      return prisma.$queryRaw`
        SELECT * FROM event_store
        WHERE stream_type = ${streamType}
          AND occurred_at > ${fromTime}
        ORDER BY occurred_at ASC
      `;
    }

    return prisma.$queryRaw`
      SELECT * FROM event_store
      WHERE stream_type = ${streamType}
      ORDER BY occurred_at ASC
    `;
  }
}

export const eventStore = new EventStore();
```

### Step 466 — Aggregate Reconstitution

```typescript
// domain/aggregates/sos-report.aggregate.ts

interface SOSState {
  id: string;
  status: 'pending' | 'verified' | 'dispatched' | 'resolved' | 'cancelled';
  userId: string;
  description: string;
  lat: number;
  lng: number;
  provinceId: number;
  severity: string;
  dispatchedTeam?: string;
  resolvedAt?: Date;
  version: number;  // sequence number
}

export class SOSReportAggregate {
  private state: SOSState | null = null;
  private uncommittedEvents: any[] = [];

  get id() { return this.state?.id; }
  get status() { return this.state?.status; }
  get version() { return this.state?.version ?? 0; }

  // Reconstitute state from events
  static fromEvents(events: any[]): SOSReportAggregate {
    const aggregate = new SOSReportAggregate();
    for (const event of events) {
      aggregate.apply(event);
    }
    return aggregate;
  }

  private apply(event: any) {
    this.state = this.state ?? { version: 0 } as any;

    switch (event.event_type) {
      case 'SOS_REPORTED':
        this.state = {
          ...this.state!,
          id: event.stream_id,
          status: 'pending',
          userId: event.payload.userId,
          description: event.payload.description,
          lat: event.payload.lat,
          lng: event.payload.lng,
          provinceId: event.payload.provinceId,
          severity: event.payload.severity,
          version: event.sequence_no,
        };
        break;

      case 'SOS_VERIFIED':
        this.state = { ...this.state!, status: 'verified', version: event.sequence_no };
        break;

      case 'SOS_DISPATCHED':
        this.state = {
          ...this.state!,
          status: 'dispatched',
          dispatchedTeam: event.payload.teamName,
          version: event.sequence_no,
        };
        break;

      case 'SOS_RESOLVED':
        this.state = {
          ...this.state!,
          status: 'resolved',
          resolvedAt: new Date(event.payload.resolvedAt),
          version: event.sequence_no,
        };
        break;
    }
  }

  // Command: Report SOS
  reportSOS(command: {
    id: string;
    userId: string;
    description: string;
    lat: number;
    lng: number;
    provinceId: number;
    severity: string;
  }) {
    if (this.state) throw new Error('SOS already exists');

    const event = {
      eventType: 'SOS_REPORTED',
      eventVersion: 1,
      payload: { ...command },
    };

    this.uncommittedEvents.push(event);
    this.apply({ event_type: 'SOS_REPORTED', payload: command, sequence_no: 1, stream_id: command.id });

    return this;
  }

  // Command: Resolve
  resolve(command: { resolvedByTeamId: string; notes: string }) {
    if (this.state?.status === 'resolved') throw new Error('Already resolved');
    if (this.state?.status !== 'dispatched') throw new Error('Must be dispatched first');

    const resolvedAt = new Date().toISOString();
    const event = {
      eventType: 'SOS_RESOLVED',
      eventVersion: 1,
      payload: { ...command, resolvedAt },
    };

    this.uncommittedEvents.push(event);
    this.apply({ event_type: 'SOS_RESOLVED', payload: { resolvedAt }, sequence_no: this.version + 1 });

    return this;
  }

  popUncommittedEvents() {
    const events = [...this.uncommittedEvents];
    this.uncommittedEvents = [];
    return events;
  }
}
```

### Step 467 — CQRS Command Handler และ Projection

```typescript
// application/commands/report-sos.handler.ts
import { eventStore } from '../../infrastructure/event-store';
import { SOSReportAggregate } from '../../domain/aggregates/sos-report.aggregate';
import { v4 as uuidv4 } from 'uuid';

export async function handleReportSOS(command: {
  userId: string;
  description: string;
  lat: number;
  lng: number;
  provinceId: number;
  severity: string;
}) {
  const sosId = uuidv4();
  const aggregate = new SOSReportAggregate();

  aggregate.reportSOS({ id: sosId, ...command });

  const events = aggregate.popUncommittedEvents();
  await eventStore.appendToStream(sosId, 'sos_report', events);

  return sosId;
}

// application/projections/sos-read-model.projection.ts
import { prisma } from '../../lib/db/client';

export async function projectSOSEvent(event: any) {
  switch (event.event_type) {
    case 'SOS_REPORTED':
      await prisma.$executeRaw`
        INSERT INTO sos_read_model (
          sos_id, user_id, username, description, status,
          province_id, province_name, lat, lng, location,
          created_at, updated_at, event_sequence
        ) VALUES (
          ${event.stream_id},
          ${event.payload.userId},
          ${event.payload.username},
          ${event.payload.description},
          'pending',
          ${event.payload.provinceId},
          ${event.payload.provinceName},
          ${event.payload.lat},
          ${event.payload.lng},
          ST_MakePoint(${event.payload.lng}, ${event.payload.lat})::geography,
          ${event.occurred_at},
          ${event.occurred_at},
          ${event.sequence_no}
        )
        ON CONFLICT (sos_id) DO NOTHING
      `;
      break;

    case 'SOS_DISPATCHED':
      await prisma.$executeRaw`
        UPDATE sos_read_model
        SET status = 'dispatched',
            dispatched_team = ${event.payload.teamName},
            updated_at = ${event.occurred_at},
            event_sequence = ${event.sequence_no}
        WHERE sos_id = ${event.stream_id}
          AND event_sequence < ${event.sequence_no}
      `;
      break;

    case 'SOS_RESOLVED':
      await prisma.$executeRaw`
        UPDATE sos_read_model
        SET status = 'resolved',
            resolved_at = ${event.payload.resolvedAt},
            updated_at = ${event.occurred_at},
            event_sequence = ${event.sequence_no}
        WHERE sos_id = ${event.stream_id}
          AND event_sequence < ${event.sequence_no}
      `;
      break;
  }
}

// Rebuild projection จาก scratch (ถ้า read model เสียหาย)
export async function rebuildSOSProjection() {
  await prisma.$executeRaw`TRUNCATE sos_read_model`;

  const events = await prisma.$queryRaw<any[]>`
    SELECT * FROM event_store
    WHERE stream_type = 'sos_report'
    ORDER BY occurred_at ASC, sequence_no ASC
  `;

  for (const event of events) {
    await projectSOSEvent(event);
  }

  console.log(`Rebuilt sos_read_model from ${events.length} events`);
}
```

### Step 468 — Snapshotting

```typescript
// infrastructure/snapshot-store.ts

const SNAPSHOT_THRESHOLD = 50; // snapshot ทุก 50 events

export async function takeSnapshotIfNeeded(
  streamId: string,
  aggregate: SOSReportAggregate
) {
  const lastSnapshot = await prisma.$queryRaw<any[]>`
    SELECT sequence_no FROM event_snapshots
    WHERE stream_id = ${streamId}
  `;

  const snapshotAt = lastSnapshot[0]?.sequence_no ?? 0;
  const currentVersion = aggregate.version;

  if (currentVersion - snapshotAt >= SNAPSHOT_THRESHOLD) {
    await prisma.$executeRaw`
      INSERT INTO event_snapshots (stream_id, stream_type, sequence_no, state)
      VALUES (${streamId}, 'sos_report', ${currentVersion}, ${JSON.stringify(aggregate)}::jsonb)
      ON CONFLICT (stream_id) DO UPDATE
        SET sequence_no = ${currentVersion},
            state = ${JSON.stringify(aggregate)}::jsonb,
            created_at = NOW()
    `;
  }
}

// Load aggregate ด้วย snapshot (เร็วกว่า replay ทั้งหมด)
export async function loadAggregateWithSnapshot(
  streamId: string
): Promise<SOSReportAggregate> {
  const snapshot = await prisma.$queryRaw<any[]>`
    SELECT * FROM event_snapshots WHERE stream_id = ${streamId}
  `;

  const fromSequence = snapshot[0]?.sequence_no ?? 0;
  const state = snapshot[0]?.state ?? null;

  // โหลด events หลัง snapshot เท่านั้น
  const events = await eventStore.loadStream(streamId, fromSequence);

  const aggregate = state
    ? Object.assign(new SOSReportAggregate(), { state, version: fromSequence })
    : new SOSReportAggregate();

  return SOSReportAggregate.fromEvents(
    state ? events : events
  );
}
```

### Step 469 — Saga Pattern (SOS + Notification)

```typescript
// application/sagas/sos-notification.saga.ts
// Saga: เมื่อ SOS ถูก reported → ส่ง notification ไป nearby users

interface SagaState {
  sosId: string;
  step: 'idle' | 'finding_users' | 'sending_notifications' | 'completed' | 'failed';
  nearbyUserIds: string[];
  notificationsSent: number;
}

export class SOSNotificationSaga {
  private state: SagaState;

  constructor(sosId: string) {
    this.state = {
      sosId,
      step: 'idle',
      nearbyUserIds: [],
      notificationsSent: 0,
    };
  }

  async handle(event: any) {
    switch (event.event_type) {
      case 'SOS_REPORTED':
        await this.onSOSReported(event);
        break;
      case 'NEARBY_USERS_FOUND':
        await this.onNearbyUsersFound(event);
        break;
      case 'NOTIFICATIONS_SENT':
        await this.onNotificationsSent(event);
        break;
    }
  }

  private async onSOSReported(event: any) {
    this.state.step = 'finding_users';

    try {
      // Step 1: Find nearby users
      const { lat, lng } = event.payload;
      const users = await prisma.$queryRaw<{ id: string }[]>`
        SELECT id FROM users
        WHERE ST_DWithin(
          location,
          ST_MakePoint(${lng}, ${lat})::geography,
          5000
        )
        AND flood_alerts_enabled = true
        LIMIT 1000
      `;

      this.state.nearbyUserIds = users.map(u => u.id);

      // Publish event ต่อไป
      await eventStore.appendToStream(this.state.sosId, 'sos_saga', [{
        eventType: 'NEARBY_USERS_FOUND',
        eventVersion: 1,
        payload: { userIds: this.state.nearbyUserIds, count: users.length },
      }]);

    } catch (error) {
      this.state.step = 'failed';
      console.error('[Saga] Failed to find nearby users:', error);
    }
  }

  private async onNearbyUsersFound(event: any) {
    this.state.step = 'sending_notifications';

    // Step 2: Send push notifications (batch)
    const { userIds } = event.payload;

    try {
      // ส่งเป็น batch ของ 100
      const batchSize = 100;
      for (let i = 0; i < userIds.length; i += batchSize) {
        const batch = userIds.slice(i, i + batchSize);
        await sendPushNotifications(batch, {
          title: '⚠️ แจ้งเตือนน้ำท่วมใกล้คุณ',
          body: 'มีรายงาน SOS ในระยะ 5 กม.',
          data: { sosId: this.state.sosId },
        });
        this.state.notificationsSent += batch.length;
      }

      this.state.step = 'completed';
    } catch (error) {
      this.state.step = 'failed';
    }
  }
}
```

### Step 470 — Event Versioning

```typescript
// infrastructure/event-upcaster.ts
// Upcasting: แปลง old event version ไป new version

type EventUpcaster = (oldPayload: Record<string, unknown>) => Record<string, unknown>;

const upcasters: Record<string, Record<number, EventUpcaster>> = {
  'SOS_REPORTED': {
    // Version 1 → 2: เพิ่ม severity field (default 'medium')
    1: (payload) => ({
      ...payload,
      severity: payload.severity ?? 'medium',
    }),
    // Version 2 → 3: lat/lng เปลี่ยนเป็น location object
    2: (payload) => ({
      ...payload,
      location: {
        lat: payload.lat,
        lng: payload.lng,
      },
    }),
  },
};

export function upcastEvent(event: any): any {
  const eventUpcasters = upcasters[event.event_type];
  if (!eventUpcasters) return event;

  let payload = { ...event.payload };
  let version = event.event_version;
  const latestVersion = Math.max(...Object.keys(eventUpcasters).map(Number)) + 1;

  while (version < latestVersion) {
    const upcaster = eventUpcasters[version];
    if (upcaster) {
      payload = upcaster(payload);
    }
    version++;
  }

  return { ...event, payload, event_version: latestVersion };
}

// ใช้งานใน aggregate loading
export async function loadAggregate(streamId: string) {
  const rawEvents = await eventStore.loadStream(streamId);
  const upcasted = rawEvents.map(upcastEvent);
  return SOSReportAggregate.fromEvents(upcasted);
}
```

---

## 🔧 Configuration Files

```sql
-- Notification index สำหรับ event store
CREATE INDEX idx_event_store_notification
    ON event_store(stream_type, event_type, occurred_at DESC)
    WHERE event_type IN ('SOS_REPORTED', 'SOS_DISPATCHED', 'SOS_RESOLVED');

-- Function สำหรับ clean old events (เก็บไว้ 2 ปี)
CREATE OR REPLACE FUNCTION archive_old_events()
RETURNS void AS $$
BEGIN
  INSERT INTO event_store_archive
  SELECT * FROM event_store
  WHERE occurred_at < NOW() - INTERVAL '2 years';

  DELETE FROM event_store
  WHERE occurred_at < NOW() - INTERVAL '2 years';
END;
$$ LANGUAGE plpgsql;
```

---

## 🧪 Testing

```bash
# ทดสอบ event sourcing workflow
npx ts-node << 'EOF'
import { handleReportSOS } from './application/commands/report-sos.handler';
import { loadAggregate } from './infrastructure/event-upcaster';

async function test() {
  // 1. Report SOS
  const sosId = await handleReportSOS({
    userId: '1',
    description: 'น้ำท่วมสูง 2 เมตร',
    lat: 13.7563,
    lng: 100.5018,
    provinceId: 10,
    severity: 'critical',
  });
  console.log('SOS ID:', sosId);

  // 2. Reload aggregate
  const aggregate = await loadAggregate(sosId);
  console.log('Status:', aggregate.status); // 'pending'
}

test().catch(console.error);
EOF
```

---

## ❌ Common Errors & Solutions

**Concurrency conflict ใน event append**
```typescript
// Retry logic สำหรับ optimistic concurrency
async function appendWithRetry(streamId: string, events: any[], maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      const aggregate = await loadAggregate(streamId);
      await eventStore.appendToStream(streamId, 'sos_report', events, aggregate.version);
      return;
    } catch (error) {
      if (!String(error).includes('Concurrency conflict') || i === maxRetries - 1) throw error;
      await new Promise(r => setTimeout(r, 100 * (i + 1)));
    }
  }
}
```

**Projection ไม่ sync กับ event store**
```bash
# Rebuild projection
npx ts-node -e "require('./application/projections/sos-read-model.projection').rebuildSOSProjection()"
```

---

## ✅ Checklist

- [ ] **Step 461** — อธิบาย Event Sourcing ด้วยตัวอย่าง SOS lifecycle ได้
- [ ] **Step 462** — เข้าใจ CQRS pattern: write side vs read side
- [ ] **Step 463** — สร้าง event_store table ที่ถูกต้องใน PostgreSQL
- [ ] **Step 464** — Define SOS domain events ครบทุก lifecycle
- [ ] **Step 465** — เขียน EventStore.appendToStream() พร้อม optimistic concurrency
- [ ] **Step 466** — Reconstitute SOSReportAggregate จาก events ได้
- [ ] **Step 467** — สร้าง projection อัพเดท sos_read_model จาก events
- [ ] **Step 468** — Implement snapshotting ทุก 50 events
- [ ] **Step 469** — Implement SOSNotificationSaga ด้วย step-by-step pattern
- [ ] **Step 470** — Event upcasting รองรับ schema migration ได้

---

## 🔗 References

- [Event Sourcing Pattern](https://martinfowler.com/eaaDev/EventSourcing.html)
- [CQRS Pattern](https://martinfowler.com/bliki/CQRS.html)
- [EventStoreDB](https://eventstore.com/)
- [Saga Pattern](https://microservices.io/patterns/data/saga.html)

---
*Part 047 | Road to 1,000,000 Users/Day | chuaikan.com*
