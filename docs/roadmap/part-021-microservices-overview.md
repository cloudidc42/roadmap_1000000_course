# Part 021: Microservices Architecture Overview
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 201-210
> **เวลาโดยประมาณ:** 8 ชั่วโมง
> **Prerequisites:** Part 001-020 (Monolith setup, PostgreSQL, Redis, Docker basics)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- เปรียบเทียบ Monolith vs Microservices สำหรับ chuaikan.com
- รู้จัก Strangler Fig Pattern สำหรับการ migrate อย่างปลอดภัย
- เข้าใจ Domain-Driven Design (DDD) และ Bounded Contexts
- ออกแบบ Service Communication แบบ sync และ async
- สร้าง Monorepo structure ด้วย Turborepo
- จัดการ Distributed Transactions ด้วย Saga Pattern
- ตั้งค่า Local Development ด้วย Docker Compose

---

## 📖 ทฤษฎีและแนวคิด

### Step 201: Monolith vs Microservices สำหรับ chuaikan.com

**Monolith Architecture** คือการที่ทุก feature อยู่ใน codebase เดียวกัน deploy พร้อมกัน

**ข้อดีของ Monolith (ในช่วงแรก)**
- Development ง่าย ไม่ต้องกังวลเรื่อง network latency ระหว่าง service
- Testing ง่าย ทดสอบได้ครบใน environment เดียว
- Deploy ง่าย deploy ที่เดียวจบ
- ไม่มี distributed transaction complexity
- เหมาะกับทีมขนาดเล็ก (1-5 คน)

**ข้อเสียของ Monolith (เมื่อ scale)**
- Deploy ช้า ต้อง deploy ทั้งระบบแม้แก้แค่ส่วนเดียว
- Scale ได้ยาก ต้อง scale ทั้งหมดแม้มีแค่บางส่วนที่ load สูง
- Code coupling สูง แก้ที่หนึ่งกระทบอีกที่
- Technology lock-in ทุก feature ต้องใช้ tech stack เดียวกัน
- Team ขยายยาก หลายคนแก้ file เดียวกันเกิด conflict บ่อย

**Microservices Architecture** คือการแบ่งระบบออกเป็น service ย่อยๆ แต่ละ service ทำงานอิสระ

**ข้อดีของ Microservices**
- Scale แต่ละ service ได้อิสระ เช่น Media service ต้องการ CPU สูง ก็ scale แยก
- Deploy อิสระ แก้ Auth service ไม่กระทบ Post service
- Technology freedom แต่ละ service ใช้ tech ได้ตามความเหมาะสม
- Fault isolation ถ้า Notification service ล่มไม่กระทบ core features
- Team ownership แต่ละทีม own service ของตัวเอง

**ข้อเสียของ Microservices**
- Operational complexity สูง ต้องดูแลหลาย service
- Network latency เพิ่มขึ้น service คุยกันผ่าน network
- Distributed transactions ยุ่งยาก
- ต้องการ DevOps knowledge สูง
- Debug ยากขึ้น request ผ่านหลาย service

**คำแนะนำสำหรับ chuaikan.com**

```
Users/Day | Architecture
----------+--------------
0-10,000  | Monolith (อย่าเพิ่ง split!)
10k-100k  | Modular Monolith (แยก modules ภายใน)
100k-500k | Selective Microservices (แยกเฉพาะ bottleneck)
500k+     | Full Microservices
```

### Step 202: ASCII Architecture Diagram

```
chuaikan.com Microservices Architecture
=========================================

                    ┌─────────────────────────────────┐
                    │         CLIENT LAYER             │
                    │  Web (Next.js) | iOS | Android   │
                    └─────────────────┬───────────────┘
                                      │ HTTPS
                    ┌─────────────────▼───────────────┐
                    │         API GATEWAY (Kong)       │
                    │  Rate Limit | Auth | Routing     │
                    └──┬──────┬──────┬──────┬─────────┘
                       │      │      │      │
           ┌───────────▼─┐  ┌─▼──────▼─┐  ┌▼──────────┐
           │ Auth Service│  │Post Service│  │SOS Service│
           │  :3001      │  │  :3002     │  │  :3003    │
           └──────┬──────┘  └─────┬──────┘  └─────┬─────┘
                  │               │                │
           ┌──────▼──────┐  ┌─────▼──────┐  ┌─────▼─────┐
           │  PostgreSQL  │  │ PostgreSQL │  │PostgreSQL │
           │  auth_db     │  │  post_db   │  │  sos_db   │
           └─────────────┘  └────────────┘  └───────────┘
                                      │
                    ┌─────────────────▼───────────────┐
                    │      MESSAGE BROKER (Redis/MQ)  │
                    │   Events: post.created,         │
                    │   sos.triggered, user.followed  │
                    └──────────────┬──────────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
   ┌──────────▼──────┐  ┌──────────▼──────┐  ┌─────────▼───────┐
   │Notification Svc │  │  Media Service  │  │  Search Service │
   │     :3004       │  │     :3005       │  │     :3006       │
   │ Email/SMS/Push  │  │  Upload/Resize  │  │  Elasticsearch  │
   └─────────────────┘  └─────────────────┘  └─────────────────┘


Shared Infrastructure:
┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐
│  Redis 7   │  │Cloudflare  │  │Prometheus  │  │  Consul    │
│  Cache     │  │    R2      │  │ + Grafana  │  │ Discovery  │
└────────────┘  └────────────┘  └────────────┘  └────────────┘
```

### Step 203: Domain-Driven Design (DDD) — Bounded Contexts

**Bounded Context** คือขอบเขตที่ชัดเจนของ domain model แต่ละ context มี ubiquitous language ของตัวเอง

**chuaikan.com Domains:**

```
┌─────────────────────────────────────────────────────────────┐
│                    User Domain                               │
│  Entities: User, Profile, Follow, Block                     │
│  Value Objects: Email, PhoneNumber, Location                │
│  Aggregates: UserAccount (root)                             │
│  Repository: UserRepository                                 │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    Post Domain                               │
│  Entities: Post, Comment, Like, Share, Tag                  │
│  Value Objects: Content, MediaRef, Coordinate               │
│  Aggregates: Post (root), Comment                           │
│  Repository: PostRepository                                 │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    SOS Domain                                │
│  Entities: SOSAlert, Responder, EmergencyContact            │
│  Value Objects: Location, AlertLevel, ResponseStatus        │
│  Aggregates: SOSAlert (root)                                │
│  Repository: SOSAlertRepository                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    Notification Domain                       │
│  Entities: Notification, NotificationPreference             │
│  Value Objects: NotificationContent, DeliveryChannel        │
│  Aggregates: NotificationBatch (root)                       │
│  Repository: NotificationRepository                         │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    Media Domain                              │
│  Entities: MediaFile, ProcessingJob, Variant                │
│  Value Objects: FileSize, MimeType, Dimensions              │
│  Aggregates: MediaFile (root)                               │
│  Repository: MediaFileRepository                            │
└─────────────────────────────────────────────────────────────┘
```

### Step 204: Service Communication Patterns

**Synchronous Communication (REST/gRPC)**
- ใช้เมื่อต้องการ response ทันที
- เหมาะกับ: authentication validation, data retrieval
- ข้อเสีย: coupling สูง, ถ้า downstream ล่ม upstream ก็ล่มด้วย

**Asynchronous Communication (Message Queue)**
- ใช้เมื่อไม่ต้องการ response ทันที
- เหมาะกับ: notifications, media processing, analytics
- ข้อดี: decoupled, resilient, can retry

```
Sync (Request-Response):
Client → API GW → Post Service → (sync) → Auth Service
                              ← token valid/invalid

Async (Event-Driven):
Post Service → publish "post.created" event → Message Queue
                                              → Notification Service (subscribe)
                                              → Search Service (subscribe)
                                              → Analytics Service (subscribe)
```

### Step 205: Strangler Fig Pattern

**แนวคิด:** ค่อยๆ ย้าย feature จาก monolith ไป microservice โดยไม่ต้อง rewrite ทั้งหมดพร้อมกัน

```
Phase 1: Monolith เต็มๆ
┌────────────────────────────────┐
│          Monolith              │
│  Auth | Posts | SOS | Media   │
└────────────────────────────────┘

Phase 2: แยก Media Service ก่อน (เพราะ CPU intensive)
┌──────────────────────────────┐    ┌─────────────┐
│ Monolith (Auth/Posts/SOS)    │───►│Media Service│
└──────────────────────────────┘    └─────────────┘

Phase 3: แยก Auth Service
┌────────────────────────┐    ┌─────────────┐    ┌─────────────┐
│ Monolith (Posts/SOS)   │    │Auth Service │    │Media Service│
└────────────────────────┘    └─────────────┘    └─────────────┘

Phase 4: แยก SOS Service (critical, ต้องการ HA สูง)
┌────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Monolith  │  │Auth Service │  │ SOS Service │  │Media Service│
│  (Posts)   │  └─────────────┘  └─────────────┘  └─────────────┘
└────────────┘

Phase 5: Full Microservices
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│Post Service │  │Auth Service │  │ SOS Service │
└─────────────┘  └─────────────┘  └─────────────┘
┌─────────────┐  ┌─────────────┐
│Media Service│  │Notif Service│
└─────────────┘  └─────────────┘
```

---

## ⚙️ Environment Setup

### Step 206: ติดตั้ง Turborepo Monorepo

```bash
# ใน Ubuntu 24.04 LTS
# ติดตั้ง Node.js 22
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs

# ตรวจสอบ version
node --version  # v22.x.x
npm --version   # 10.x.x

# ติดตั้ง pnpm (package manager ที่ Turborepo แนะนำ)
npm install -g pnpm@9
pnpm --version

# สร้าง monorepo project
mkdir chuaikan-platform
cd chuaikan-platform

# สร้าง Turborepo
npx create-turbo@latest . --package-manager pnpm

# Structure ที่จะได้:
# chuaikan-platform/
# ├── apps/
# │   ├── web/          (Next.js frontend)
# │   └── docs/         (Documentation)
# ├── packages/
# │   ├── eslint-config/
# │   ├── typescript-config/
# │   └── ui/
# ├── turbo.json
# ├── package.json
# └── pnpm-workspace.yaml
```

### Step 207: สร้าง Monorepo Structure สำหรับ chuaikan.com

```bash
cd chuaikan-platform

# สร้าง services directories
mkdir -p apps/{web,auth-service,post-service,sos-service,notification-service,media-service}
mkdir -p packages/{shared-types,utils,database,logger,config}

# สร้าง root package.json
cat > package.json << 'EOF'
{
  "name": "chuaikan-platform",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "build": "turbo run build",
    "dev": "turbo run dev",
    "lint": "turbo run lint",
    "test": "turbo run test",
    "clean": "turbo run clean && rm -rf node_modules",
    "format": "prettier --write \"**/*.{ts,tsx,md}\"",
    "docker:up": "docker-compose up -d",
    "docker:down": "docker-compose down"
  },
  "devDependencies": {
    "turbo": "^2.0.0",
    "prettier": "^3.3.0",
    "@types/node": "^22.0.0",
    "typescript": "^5.5.0"
  },
  "engines": {
    "node": ">=22.0.0",
    "pnpm": ">=9.0.0"
  },
  "packageManager": "pnpm@9.0.0"
}
EOF

# สร้าง pnpm-workspace.yaml
cat > pnpm-workspace.yaml << 'EOF'
packages:
  - "apps/*"
  - "packages/*"
EOF

# สร้าง turbo.json
cat > turbo.json << 'EOF'
{
  "$schema": "https://turbo.build/schema.json",
  "ui": "tui",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "inputs": ["$TURBO_DEFAULT$", ".env*"],
      "outputs": [".next/**", "!.next/cache/**", "dist/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "lint": {
      "dependsOn": ["^lint"]
    },
    "test": {
      "dependsOn": ["^build"],
      "outputs": ["coverage/**"]
    },
    "clean": {
      "cache": false
    }
  }
}
EOF
```

### Step 208: Shared Libraries

```bash
# สร้าง @chuaikan/shared-types package
cd packages/shared-types
cat > package.json << 'EOF'
{
  "name": "@chuaikan/shared-types",
  "version": "0.0.1",
  "main": "./src/index.ts",
  "types": "./src/index.ts",
  "scripts": {
    "lint": "tsc --noEmit",
    "clean": "rm -rf dist"
  },
  "devDependencies": {
    "typescript": "^5.5.0",
    "@chuaikan/typescript-config": "workspace:*"
  }
}
EOF

mkdir -p src
cat > src/index.ts << 'EOF'
// User types
export interface User {
  id: string;
  email: string;
  username: string;
  displayName: string;
  avatarUrl?: string;
  bio?: string;
  isVerified: boolean;
  createdAt: Date;
  updatedAt: Date;
}

export interface UserProfile extends User {
  followersCount: number;
  followingCount: number;
  postsCount: number;
}

// Post types
export interface Post {
  id: string;
  authorId: string;
  content: string;
  mediaUrls: string[];
  location?: GeoLocation;
  tags: string[];
  likesCount: number;
  commentsCount: number;
  sharesCount: number;
  createdAt: Date;
  updatedAt: Date;
}

// SOS types
export type AlertLevel = 'low' | 'medium' | 'high' | 'critical';

export interface SOSAlert {
  id: string;
  userId: string;
  location: GeoLocation;
  alertLevel: AlertLevel;
  description: string;
  status: 'active' | 'resolved' | 'cancelled';
  respondersCount: number;
  createdAt: Date;
}

// Notification types
export type NotificationChannel = 'push' | 'sms' | 'email';
export type NotificationPriority = 'low' | 'medium' | 'high' | 'critical';

export interface Notification {
  id: string;
  userId: string;
  type: string;
  channel: NotificationChannel;
  priority: NotificationPriority;
  title: string;
  body: string;
  data?: Record<string, unknown>;
  readAt?: Date;
  sentAt?: Date;
  createdAt: Date;
}

// Media types
export type MediaType = 'image' | 'video' | 'audio';

export interface MediaFile {
  id: string;
  userId: string;
  type: MediaType;
  originalUrl: string;
  processedVariants: MediaVariant[];
  mimeType: string;
  fileSize: number;
  width?: number;
  height?: number;
  duration?: number;
  blurHash?: string;
  status: 'pending' | 'processing' | 'ready' | 'failed';
  createdAt: Date;
}

export interface MediaVariant {
  name: string;  // 'thumbnail' | 'medium' | 'large'
  url: string;
  width: number;
  height: number;
  fileSize: number;
  format: string;
}

// Common types
export interface GeoLocation {
  lat: number;
  lng: number;
  address?: string;
  city?: string;
  country?: string;
}

export interface PaginatedResponse<T> {
  data: T[];
  total: number;
  page: number;
  limit: number;
  hasMore: boolean;
}

export interface ApiResponse<T = void> {
  success: boolean;
  data?: T;
  error?: {
    code: string;
    message: string;
    details?: unknown;
  };
}

// Service events (for message queue)
export interface DomainEvent {
  eventId: string;
  eventType: string;
  aggregateId: string;
  aggregateType: string;
  payload: Record<string, unknown>;
  occurredAt: Date;
  version: number;
}

export interface UserCreatedEvent extends DomainEvent {
  eventType: 'user.created';
  payload: {
    userId: string;
    email: string;
    username: string;
  };
}

export interface PostCreatedEvent extends DomainEvent {
  eventType: 'post.created';
  payload: {
    postId: string;
    authorId: string;
    content: string;
    mediaIds: string[];
    mentionedUserIds: string[];
  };
}

export interface SOSTriggeredEvent extends DomainEvent {
  eventType: 'sos.triggered';
  payload: {
    alertId: string;
    userId: string;
    location: GeoLocation;
    alertLevel: AlertLevel;
  };
}
EOF
```

```bash
# สร้าง @chuaikan/utils package
cd ../../packages/utils

cat > package.json << 'EOF'
{
  "name": "@chuaikan/utils",
  "version": "0.0.1",
  "main": "./src/index.ts",
  "types": "./src/index.ts",
  "dependencies": {
    "date-fns": "^3.6.0",
    "zod": "^3.23.0"
  },
  "devDependencies": {
    "@chuaikan/shared-types": "workspace:*",
    "typescript": "^5.5.0"
  }
}
EOF

mkdir -p src
cat > src/index.ts << 'EOF'
import { z } from 'zod';

// Pagination helpers
export function getPaginationParams(
  page: number = 1,
  limit: number = 20,
  maxLimit: number = 100
): { skip: number; take: number; page: number; limit: number } {
  const safePage = Math.max(1, page);
  const safeLimit = Math.min(Math.max(1, limit), maxLimit);
  return {
    skip: (safePage - 1) * safeLimit,
    take: safeLimit,
    page: safePage,
    limit: safeLimit,
  };
}

// ID generation
export function generateId(): string {
  return `${Date.now().toString(36)}-${Math.random().toString(36).substr(2, 9)}`;
}

// Sleep utility
export function sleep(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

// Retry with exponential backoff
export async function retry<T>(
  fn: () => Promise<T>,
  options: {
    maxRetries?: number;
    initialDelay?: number;
    maxDelay?: number;
    onRetry?: (error: Error, attempt: number) => void;
  } = {}
): Promise<T> {
  const { maxRetries = 3, initialDelay = 1000, maxDelay = 30000, onRetry } = options;
  let lastError: Error;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error instanceof Error ? error : new Error(String(error));
      if (attempt < maxRetries) {
        const delay = Math.min(initialDelay * Math.pow(2, attempt), maxDelay);
        onRetry?.(lastError, attempt + 1);
        await sleep(delay);
      }
    }
  }
  throw lastError!;
}

// Validation schemas
export const PhoneNumberSchema = z
  .string()
  .regex(/^(\+66|0)[0-9]{8,9}$/, 'Invalid Thai phone number format');

export const EmailSchema = z
  .string()
  .email('Invalid email format')
  .toLowerCase();

export const PasswordSchema = z
  .string()
  .min(8, 'Password must be at least 8 characters')
  .regex(/[A-Z]/, 'Must contain uppercase letter')
  .regex(/[0-9]/, 'Must contain number')
  .regex(/[!@#$%^&*]/, 'Must contain special character');

// Format Thai date
export function formatThaiDate(date: Date): string {
  const thaiYear = date.getFullYear() + 543;
  const months = [
    'ม.ค.', 'ก.พ.', 'มี.ค.', 'เม.ย.', 'พ.ค.', 'มิ.ย.',
    'ก.ค.', 'ส.ค.', 'ก.ย.', 'ต.ค.', 'พ.ย.', 'ธ.ค.'
  ];
  return `${date.getDate()} ${months[date.getMonth()]} ${thaiYear}`;
}

// Hash password
export async function hashPassword(password: string): Promise<string> {
  const bcrypt = await import('bcryptjs');
  return bcrypt.hash(password, 12);
}

export async function comparePassword(
  password: string,
  hash: string
): Promise<boolean> {
  const bcrypt = await import('bcryptjs');
  return bcrypt.compare(password, hash);
}

// Sanitize user input
export function sanitizeString(input: string, maxLength: number = 500): string {
  return input
    .trim()
    .replace(/<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>/gi, '')
    .replace(/[<>]/g, '')
    .substring(0, maxLength);
}

// Calculate distance between two coordinates (Haversine formula)
export function calculateDistance(
  lat1: number,
  lon1: number,
  lat2: number,
  lon2: number
): number {
  const R = 6371; // Earth's radius in km
  const dLat = ((lat2 - lat1) * Math.PI) / 180;
  const dLon = ((lon2 - lon1) * Math.PI) / 180;
  const a =
    Math.sin(dLat / 2) * Math.sin(dLat / 2) +
    Math.cos((lat1 * Math.PI) / 180) *
      Math.cos((lat2 * Math.PI) / 180) *
      Math.sin(dLon / 2) *
      Math.sin(dLon / 2);
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
  return R * c; // Distance in km
}
EOF
```

---

## 🛠️ Step-by-Step Implementation

### Step 209: Inter-service Authentication (Service Tokens)

**แนวคิด:** เมื่อ service คุยกัน เราไม่สามารถใช้ user JWT ได้ ต้องมี service-to-service authentication

```typescript
// packages/shared-types/src/service-auth.ts

import * as jwt from 'jsonwebtoken';

export interface ServiceTokenPayload {
  serviceId: string;
  serviceName: string;
  permissions: string[];
  iat: number;
  exp: number;
}

export class ServiceAuthClient {
  private readonly serviceId: string;
  private readonly serviceName: string;
  private readonly privateKey: string;
  private cachedToken: string | null = null;
  private tokenExpiry: number = 0;

  constructor(config: {
    serviceId: string;
    serviceName: string;
    privateKey: string;
  }) {
    this.serviceId = config.serviceId;
    this.serviceName = config.serviceName;
    this.privateKey = config.privateKey;
  }

  generateServiceToken(permissions: string[] = []): string {
    const now = Math.floor(Date.now() / 1000);
    
    // Cache token สำหรับ 4 นาที (expire 5 นาที)
    if (this.cachedToken && this.tokenExpiry > now + 60) {
      return this.cachedToken;
    }

    const payload: Omit<ServiceTokenPayload, 'iat' | 'exp'> = {
      serviceId: this.serviceId,
      serviceName: this.serviceName,
      permissions,
    };

    this.cachedToken = jwt.sign(payload, this.privateKey, {
      algorithm: 'RS256',
      expiresIn: '5m',
      issuer: 'chuaikan-services',
    });
    this.tokenExpiry = now + 300; // 5 minutes
    return this.cachedToken;
  }

  // Axios interceptor for automatic service auth headers
  createAxiosInterceptor() {
    return {
      request: (config: any) => {
        config.headers['X-Service-Token'] = this.generateServiceToken();
        config.headers['X-Service-ID'] = this.serviceId;
        return config;
      },
    };
  }
}

// Middleware to validate incoming service tokens
export function validateServiceToken(publicKey: string) {
  return (req: any, res: any, next: any) => {
    const token = req.headers['x-service-token'];
    
    if (!token) {
      return res.status(401).json({
        success: false,
        error: { code: 'NO_SERVICE_TOKEN', message: 'Service token required' },
      });
    }

    try {
      const payload = jwt.verify(token, publicKey, {
        algorithms: ['RS256'],
        issuer: 'chuaikan-services',
      }) as ServiceTokenPayload;

      req.serviceContext = payload;
      next();
    } catch (error) {
      return res.status(401).json({
        success: false,
        error: { code: 'INVALID_SERVICE_TOKEN', message: 'Invalid service token' },
      });
    }
  };
}
```

### Step 210: Saga Pattern สำหรับ Distributed Transactions

**ตัวอย่าง:** เมื่อ user สร้าง post ที่มี media (ต้องทำหลาย service พร้อมกัน)

```typescript
// apps/post-service/src/sagas/create-post-with-media.saga.ts

import { EventEmitter } from 'events';

interface CreatePostWithMediaCommand {
  userId: string;
  content: string;
  tempMediaIds: string[];
  location?: { lat: number; lng: number };
}

interface SagaStep {
  name: string;
  execute: () => Promise<void>;
  compensate: () => Promise<void>;
}

class CreatePostWithMediaSaga {
  private executedSteps: SagaStep[] = [];
  private postId: string | null = null;
  private confirmedMediaIds: string[] = [];

  constructor(
    private readonly command: CreatePostWithMediaCommand,
    private readonly postRepo: any,
    private readonly mediaClient: any,
    private readonly eventBus: EventEmitter
  ) {}

  async execute(): Promise<{ postId: string }> {
    const steps: SagaStep[] = [
      {
        name: 'ConfirmMediaOwnership',
        execute: async () => {
          // ตรวจสอบว่า media เป็นของ user จริงๆ
          this.confirmedMediaIds = await this.mediaClient.confirmOwnership(
            this.command.tempMediaIds,
            this.command.userId
          );
        },
        compensate: async () => {
          // ไม่ต้อง compensate - ไม่มีการเปลี่ยนแปลง
        },
      },
      {
        name: 'CreatePost',
        execute: async () => {
          const post = await this.postRepo.create({
            authorId: this.command.userId,
            content: this.command.content,
            mediaIds: this.confirmedMediaIds,
            location: this.command.location,
            status: 'draft', // สร้างเป็น draft ก่อน
          });
          this.postId = post.id;
        },
        compensate: async () => {
          if (this.postId) {
            await this.postRepo.delete(this.postId);
          }
        },
      },
      {
        name: 'AttachMediaToPost',
        execute: async () => {
          await this.mediaClient.attachToPost(this.confirmedMediaIds, this.postId!);
        },
        compensate: async () => {
          await this.mediaClient.detachFromPost(this.confirmedMediaIds, this.postId!);
        },
      },
      {
        name: 'PublishPost',
        execute: async () => {
          await this.postRepo.publish(this.postId!);
          
          // Publish event to message queue
          this.eventBus.emit('post.created', {
            eventType: 'post.created',
            aggregateId: this.postId,
            payload: {
              postId: this.postId,
              authorId: this.command.userId,
              content: this.command.content,
              mediaIds: this.confirmedMediaIds,
            },
            occurredAt: new Date(),
          });
        },
        compensate: async () => {
          await this.postRepo.updateStatus(this.postId!, 'failed');
        },
      },
    ];

    // Execute steps ทีละขั้น
    for (const step of steps) {
      try {
        await step.execute();
        this.executedSteps.push(step);
      } catch (error) {
        console.error(`Saga step ${step.name} failed:`, error);
        // Compensate in reverse order
        await this.compensate();
        throw new Error(`Saga failed at step ${step.name}: ${error}`);
      }
    }

    return { postId: this.postId! };
  }

  private async compensate(): Promise<void> {
    // Execute compensations in reverse order
    const stepsToCompensate = [...this.executedSteps].reverse();
    
    for (const step of stepsToCompensate) {
      try {
        await step.compensate();
        console.log(`Compensated step: ${step.name}`);
      } catch (error) {
        console.error(`Failed to compensate step ${step.name}:`, error);
        // Log to dead letter queue for manual review
      }
    }
  }
}

export { CreatePostWithMediaSaga, CreatePostWithMediaCommand };
```

---

## 🔧 Configuration Files

### Docker Compose สำหรับ Local Development

```yaml
# docker-compose.yml (ที่ root ของ monorepo)
version: '3.9'

services:
  # Infrastructure
  postgres-auth:
    image: postgres:17-alpine
    container_name: chuaikan-postgres-auth
    environment:
      POSTGRES_DB: chuaikan_auth
      POSTGRES_USER: chuaikan
      POSTGRES_PASSWORD: chuaikan_dev_password
    ports:
      - "5432:5432"
    volumes:
      - postgres-auth-data:/var/lib/postgresql/data
      - ./infrastructure/postgres/init-auth.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U chuaikan -d chuaikan_auth"]
      interval: 10s
      timeout: 5s
      retries: 5

  postgres-post:
    image: postgres:17-alpine
    container_name: chuaikan-postgres-post
    environment:
      POSTGRES_DB: chuaikan_post
      POSTGRES_USER: chuaikan
      POSTGRES_PASSWORD: chuaikan_dev_password
    ports:
      - "5433:5432"
    volumes:
      - postgres-post-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U chuaikan -d chuaikan_post"]
      interval: 10s
      timeout: 5s
      retries: 5

  postgres-sos:
    image: postgres:17-alpine
    container_name: chuaikan-postgres-sos
    environment:
      POSTGRES_DB: chuaikan_sos
      POSTGRES_USER: chuaikan
      POSTGRES_PASSWORD: chuaikan_dev_password
    ports:
      - "5434:5432"
    volumes:
      - postgres-sos-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U chuaikan -d chuaikan_sos"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: chuaikan-redis
    command: redis-server --appendonly yes --requirepass chuaikan_redis_dev
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "chuaikan_redis_dev", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Services
  auth-service:
    build:
      context: ./apps/auth-service
      dockerfile: Dockerfile.dev
    container_name: chuaikan-auth-service
    ports:
      - "3001:3001"
    environment:
      - NODE_ENV=development
      - PORT=3001
      - DATABASE_URL=postgresql://chuaikan:chuaikan_dev_password@postgres-auth:5432/chuaikan_auth
      - REDIS_URL=redis://:chuaikan_redis_dev@redis:6379
      - JWT_PRIVATE_KEY_PATH=/app/keys/private.pem
      - JWT_PUBLIC_KEY_PATH=/app/keys/public.pem
    volumes:
      - ./apps/auth-service:/app
      - ./keys:/app/keys:ro
      - /app/node_modules
    depends_on:
      postgres-auth:
        condition: service_healthy
      redis:
        condition: service_healthy
    command: pnpm dev

  post-service:
    build:
      context: ./apps/post-service
      dockerfile: Dockerfile.dev
    container_name: chuaikan-post-service
    ports:
      - "3002:3002"
    environment:
      - NODE_ENV=development
      - PORT=3002
      - DATABASE_URL=postgresql://chuaikan:chuaikan_dev_password@postgres-post:5432/chuaikan_post
      - REDIS_URL=redis://:chuaikan_redis_dev@redis:6379
      - AUTH_SERVICE_URL=http://auth-service:3001
    volumes:
      - ./apps/post-service:/app
      - /app/node_modules
    depends_on:
      - auth-service
    command: pnpm dev

  notification-service:
    build:
      context: ./apps/notification-service
      dockerfile: Dockerfile.dev
    container_name: chuaikan-notification-service
    ports:
      - "3004:3004"
    environment:
      - NODE_ENV=development
      - PORT=3004
      - REDIS_URL=redis://:chuaikan_redis_dev@redis:6379
      - RESEND_API_KEY=${RESEND_API_KEY}
      - TWILIO_ACCOUNT_SID=${TWILIO_ACCOUNT_SID}
      - TWILIO_AUTH_TOKEN=${TWILIO_AUTH_TOKEN}
      - FCM_SERVER_KEY=${FCM_SERVER_KEY}
    volumes:
      - ./apps/notification-service:/app
      - /app/node_modules
    depends_on:
      - redis
    command: pnpm dev

  media-service:
    build:
      context: ./apps/media-service
      dockerfile: Dockerfile.dev
    container_name: chuaikan-media-service
    ports:
      - "3005:3005"
    environment:
      - NODE_ENV=development
      - PORT=3005
      - DATABASE_URL=postgresql://chuaikan:chuaikan_dev_password@postgres-post:5432/chuaikan_post
      - REDIS_URL=redis://:chuaikan_redis_dev@redis:6379
      - R2_ACCOUNT_ID=${R2_ACCOUNT_ID}
      - R2_ACCESS_KEY_ID=${R2_ACCESS_KEY_ID}
      - R2_SECRET_ACCESS_KEY=${R2_SECRET_ACCESS_KEY}
      - R2_BUCKET_NAME=chuaikan-media-dev
    volumes:
      - ./apps/media-service:/app
      - /app/node_modules
    depends_on:
      - redis
    command: pnpm dev

  # API Gateway (Kong)
  kong:
    image: kong:3.7-ubuntu
    container_name: chuaikan-kong
    environment:
      KONG_DATABASE: "off"  # DB-less mode
      KONG_DECLARATIVE_CONFIG: /kong/kong.yml
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_ADMIN_LISTEN: "0.0.0.0:8001"
    ports:
      - "8000:8000"   # Kong proxy
      - "8001:8001"   # Kong admin
    volumes:
      - ./infrastructure/kong:/kong
    depends_on:
      - auth-service
      - post-service

volumes:
  postgres-auth-data:
  postgres-post-data:
  postgres-sos-data:
  redis-data:

networks:
  default:
    name: chuaikan-network
```

### Dockerfile.dev สำหรับ Services

```dockerfile
# apps/auth-service/Dockerfile.dev
FROM node:22-alpine

RUN npm install -g pnpm@9

WORKDIR /app

# Install dependencies
COPY package.json pnpm-lock.yaml* ./
RUN pnpm install

# Copy source
COPY . .

EXPOSE 3001

CMD ["pnpm", "dev"]
```

### Service Versioning Strategy

```typescript
// packages/shared-types/src/versioning.ts

// API Version header: X-API-Version: 2024-01-01
// ใช้ date-based versioning แบบ Stripe

export const API_VERSIONS = {
  V1: '2024-01-01',
  V2: '2024-06-01',
  CURRENT: '2024-06-01',
} as const;

export type ApiVersion = (typeof API_VERSIONS)[keyof typeof API_VERSIONS];

// Express middleware สำหรับ version routing
export function versionMiddleware(req: any, res: any, next: any) {
  const requestedVersion = req.headers['x-api-version'] || API_VERSIONS.CURRENT;
  
  // Validate version
  const validVersions = Object.values(API_VERSIONS);
  if (!validVersions.includes(requestedVersion as ApiVersion)) {
    return res.status(400).json({
      success: false,
      error: {
        code: 'INVALID_API_VERSION',
        message: `Unsupported API version. Valid versions: ${validVersions.join(', ')}`,
      },
    });
  }

  req.apiVersion = requestedVersion;
  res.setHeader('X-API-Version', requestedVersion);
  next();
}
```

---

## 🧪 Testing

### Integration Test สำหรับ Service Communication

```typescript
// apps/post-service/src/__tests__/service-communication.test.ts

import request from 'supertest';
import { createApp } from '../app';
import nock from 'nock';

describe('Post Service - Service Communication', () => {
  let app: any;

  beforeAll(async () => {
    app = await createApp();
  });

  afterEach(() => {
    nock.cleanAll();
  });

  it('should validate user token with auth service', async () => {
    // Mock auth service
    nock('http://auth-service:3001')
      .post('/internal/introspect')
      .reply(200, {
        success: true,
        data: {
          userId: 'user-123',
          email: 'test@chuaikan.com',
          isValid: true,
        },
      });

    const response = await request(app)
      .post('/posts')
      .set('Authorization', 'Bearer valid-token')
      .send({
        content: 'Test post content',
      });

    expect(response.status).toBe(201);
    expect(response.body.success).toBe(true);
  });

  it('should return 401 when auth service rejects token', async () => {
    nock('http://auth-service:3001')
      .post('/internal/introspect')
      .reply(401, {
        success: false,
        error: { code: 'INVALID_TOKEN', message: 'Token expired' },
      });

    const response = await request(app)
      .post('/posts')
      .set('Authorization', 'Bearer expired-token')
      .send({ content: 'Test post' });

    expect(response.status).toBe(401);
  });

  it('should handle auth service timeout gracefully', async () => {
    nock('http://auth-service:3001')
      .post('/internal/introspect')
      .delayConnection(5000) // Simulate timeout
      .reply(200, {});

    const response = await request(app)
      .post('/posts')
      .set('Authorization', 'Bearer some-token')
      .send({ content: 'Test post' });

    expect(response.status).toBe(503);
    expect(response.body.error.code).toBe('AUTH_SERVICE_UNAVAILABLE');
  });
});
```

### Migration Script

```bash
#!/bin/bash
# scripts/migrate-monolith-to-microservices.sh
# Migration strategy script

set -e

PHASE=${1:-"check"}
ENV=${2:-"staging"}

echo "=== chuaikan.com Migration Script ==="
echo "Phase: $PHASE | Environment: $ENV"

check_prerequisites() {
  echo "Checking prerequisites..."
  command -v docker >/dev/null 2>&1 || { echo "Docker required"; exit 1; }
  command -v kubectl >/dev/null 2>&1 || { echo "kubectl required"; exit 1; }
  command -v pnpm >/dev/null 2>&1 || { echo "pnpm required"; exit 1; }
  echo "All prerequisites met"
}

phase_1_media_service() {
  echo "Phase 1: Extracting Media Service..."
  
  # 1. Deploy new media service
  docker build -t chuaikan/media-service:v1 ./apps/media-service
  docker push chuaikan/media-service:v1
  
  # 2. Update API gateway to route /api/media to new service
  # (while keeping monolith as fallback)
  kubectl apply -f ./k8s/media-service.yaml
  
  # 3. Run migration to copy media records
  pnpm --filter=media-service run migrate:from-monolith
  
  # 4. Update monolith to use media service via HTTP
  # (strangler fig: monolith calls new service)
  
  echo "Phase 1 complete: Media service extracted"
}

phase_2_auth_service() {
  echo "Phase 2: Extracting Auth Service..."
  
  # 1. Generate RSA keys for JWT
  mkdir -p ./keys
  openssl genrsa -out ./keys/private.pem 2048
  openssl rsa -in ./keys/private.pem -pubout -out ./keys/public.pem
  
  # 2. Deploy auth service
  kubectl create secret generic jwt-keys \
    --from-file=private.pem=./keys/private.pem \
    --from-file=public.pem=./keys/public.pem
  
  kubectl apply -f ./k8s/auth-service.yaml
  
  # 3. Migrate users table
  pnpm --filter=auth-service run migrate:from-monolith
  
  echo "Phase 2 complete: Auth service extracted"
}

case $PHASE in
  "check") check_prerequisites ;;
  "phase1") phase_1_media_service ;;
  "phase2") phase_2_auth_service ;;
  *) echo "Unknown phase: $PHASE. Use: check, phase1, phase2" ;;
esac
```

---

## ❌ Common Errors & Solutions

### Error 1: "Cannot resolve module @chuaikan/shared-types"

```bash
# สาเหตุ: pnpm workspace links ไม่ถูก setup
# แก้ไข:
cd chuaikan-platform
pnpm install  # ต้องรันที่ root เพื่อสร้าง workspace links

# ตรวจสอบ
ls node_modules/@chuaikan/  # ควรเห็น shared-types, utils
```

### Error 2: "ECONNREFUSED" ระหว่าง services

```bash
# สาเหตุ: service ยังไม่ขึ้นหรือ healthcheck ยังไม่ผ่าน
# ตรวจสอบ:
docker-compose ps
docker-compose logs auth-service

# แก้ไข: เพิ่ม retry logic ใน service client
# ใช้ packages/utils/src/retry function
```

### Error 3: "Distributed transaction inconsistency"

```bash
# สาเหตุ: Saga compensation ทำงานไม่ครบ
# วิธีป้องกัน: ทุก saga step ต้องเป็น idempotent

# ตรวจสอบ saga execution log
docker-compose logs post-service | grep "Saga"

# ตรวจสอบ dead letter queue
redis-cli -a chuaikan_redis_dev LLEN saga:failed
```

### Error 4: Turborepo cache ไม่ทำงาน

```bash
# ตรวจสอบ turbo.json inputs
cat turbo.json

# Force rebuild โดยไม่ใช้ cache
pnpm turbo build --force

# Clear turbo cache
pnpm turbo clean
rm -rf .turbo
```

---

## ✅ Checklist

### Step 201-202: Architecture Design
- [ ] เข้าใจข้อดีข้อเสียของ Monolith vs Microservices
- [ ] วาด ASCII diagram ของ chuaikan.com architecture
- [ ] กำหนด Bounded Contexts ทั้ง 5 domains
- [ ] ตัดสินใจว่า service ไหนจะ split ก่อน

### Step 203-204: Design Patterns
- [ ] เข้าใจ Strangler Fig Pattern
- [ ] กำหนด sync vs async communication ของแต่ละ service
- [ ] ออกแบบ Domain Events ที่จำเป็น

### Step 205-206: Monorepo Setup
- [ ] ติดตั้ง Turborepo และ pnpm workspace
- [ ] สร้าง monorepo structure ตามที่กำหนด
- [ ] `pnpm install` สำเร็จที่ root

### Step 207-208: Shared Libraries
- [ ] สร้าง `@chuaikan/shared-types` พร้อม types ครบถ้วน
- [ ] สร้าง `@chuaikan/utils` พร้อม helper functions
- [ ] ทดสอบ import shared packages จาก service อื่น

### Step 209: Service Authentication
- [ ] Generate RSA key pair (private/public)
- [ ] Implement `ServiceAuthClient`
- [ ] Implement `validateServiceToken` middleware
- [ ] ทดสอบ service-to-service authentication

### Step 210: Distributed Transactions
- [ ] เข้าใจ Saga Pattern (choreography vs orchestration)
- [ ] Implement `CreatePostWithMediaSaga`
- [ ] ทดสอบ happy path
- [ ] ทดสอบ compensation เมื่อ step fail
- [ ] Docker Compose ขึ้นสำเร็จทุก service

---

## 🔗 References

- [Turborepo Documentation](https://turbo.build/repo/docs)
- [Domain-Driven Design by Eric Evans](https://www.domainlanguage.com/ddd/)
- [Microservices Patterns by Chris Richardson](https://microservices.io/patterns/)
- [Saga Pattern](https://microservices.io/patterns/data/saga.html)
- [Strangler Fig Pattern](https://martinfowler.com/bliki/StranglerFigApplication.html)
- [pnpm Workspaces](https://pnpm.io/workspaces)

---
*Part 021 | Road to 1,000,000 Users/Day | chuaikan.com*
