# Part 016: API Design & REST Best Practices
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Beginner
> **Steps:** 151-160
> **เวลาโดยประมาณ:** 8 ชั่วโมง
> **Prerequisites:** Part 001-015 (Infrastructure, Database, Caching)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

1. หลักการออกแบบ REST API ที่ดีสำหรับระบบขนาดใหญ่
2. API Versioning strategies (/v1, /v2) เพื่อ backward compatibility
3. การกำหนดมาตรฐาน Request/Response format ด้วย JSON API spec
4. Pagination แบบ cursor-based vs offset-based
5. Filtering, sorting, field selection สำหรับ REST API
6. Rate limiting headers และการป้องกัน API abuse
7. การทำ API Documentation ด้วย OpenAPI 3.1 / Swagger
8. Error response format มาตรฐาน
9. HATEOAS: เมื่อไหร่ควรใช้ เมื่อไหร่ไม่ควร
10. API Security: API keys, OAuth 2.0
11. Idempotency keys สำหรับ POST requests
12. Webhook design สำหรับ third-party integrations

---

## 📖 ทฤษฎีและแนวคิด

### REST API Design Principles

REST (Representational State Transfer) คือสถาปัตยกรรมสำหรับการออกแบบ web services ที่ใช้กันอย่างแพร่หลาย หลักการสำคัญมีดังนี้:

```
┌─────────────────────────────────────────────────────────────┐
│                    REST API Principles                       │
├─────────────────────────────────────────────────────────────┤
│  1. Stateless     - ทุก request มีข้อมูลครบในตัวเอง         │
│  2. Client-Server - แยก UI logic กับ data logic            │
│  3. Cacheable     - response บอกได้ว่า cache ได้หรือไม่     │
│  4. Uniform Interface - ใช้ HTTP methods มาตรฐาน           │
│  5. Layered System - middleware, proxy ได้                  │
│  6. Code on Demand - optional, ส่ง code มา execute ได้      │
└─────────────────────────────────────────────────────────────┘
```

### Resource Naming Convention

การตั้งชื่อ resource ที่ดีคือหัวใจของ REST API:

```
✅ GOOD - ใช้ noun, plural form
GET    /api/v1/users              # list users
GET    /api/v1/users/:id          # get user by id
POST   /api/v1/users              # create user
PUT    /api/v1/users/:id          # replace user
PATCH  /api/v1/users/:id          # update user partially
DELETE /api/v1/users/:id          # delete user

✅ Nested Resources
GET    /api/v1/users/:id/posts    # get posts of user
POST   /api/v1/users/:id/posts    # create post for user

❌ BAD - อย่าใช้ verb ใน URL
GET  /api/v1/getUsers             # ผิด
POST /api/v1/createUser           # ผิด
GET  /api/v1/user-list            # ผิด
```

### HTTP Methods และ Status Codes

```
HTTP Methods:
┌────────────┬────────────┬─────────────────────────────────┐
│ Method     │ Idempotent │ Use Case                        │
├────────────┼────────────┼─────────────────────────────────┤
│ GET        │ Yes        │ Read data                       │
│ POST       │ No         │ Create resource                 │
│ PUT        │ Yes        │ Replace resource completely     │
│ PATCH      │ No         │ Partial update                  │
│ DELETE     │ Yes        │ Delete resource                 │
│ HEAD       │ Yes        │ Check resource exists           │
│ OPTIONS    │ Yes        │ Get available methods (CORS)    │
└────────────┴────────────┴─────────────────────────────────┘

Status Codes ที่ใช้บ่อย:
2xx Success:
  200 OK           - GET, PUT, PATCH สำเร็จ
  201 Created      - POST สร้าง resource สำเร็จ
  204 No Content   - DELETE สำเร็จ (ไม่มี body)
  
4xx Client Error:
  400 Bad Request  - request body ผิดรูปแบบ
  401 Unauthorized - ยังไม่ได้ authenticate
  403 Forbidden    - authenticate แล้วแต่ไม่มีสิทธิ์
  404 Not Found    - resource ไม่พบ
  409 Conflict     - data conflict (เช่น email ซ้ำ)
  422 Unprocessable - validation error
  429 Too Many Requests - rate limit exceeded

5xx Server Error:
  500 Internal Server Error - server พัง
  502 Bad Gateway          - upstream service พัง
  503 Service Unavailable  - service หยุดชั่วคราว
  504 Gateway Timeout      - upstream timeout
```

### API Versioning Strategies

```
Strategy 1: URL Path (แนะนำ)
  /api/v1/users
  /api/v2/users

Strategy 2: Query Parameter
  /api/users?version=1
  /api/users?version=2

Strategy 3: Header
  Accept: application/vnd.chuaikan.v1+json
  Accept: application/vnd.chuaikan.v2+json

สำหรับ chuaikan.com เลือก URL Path เพราะ:
✓ ชัดเจน มองเห็นได้ใน URL
✓ ง่ายต่อการ cache
✓ ทดสอบง่ายใน browser
✓ Proxy/CDN รองรับได้ดี
```

---

## ⚙️ Environment Setup

### ติดตั้ง Dependencies

```bash
# Next.js 15 project
cd /home/user/chuaikan-app

# Install API documentation tools
npm install swagger-ui-express
npm install @asteasolutions/zod-to-openapi
npm install zod

# Install rate limiting
npm install @upstash/ratelimit
npm install ioredis

# Install validation
npm install zod
npm install zod-validation-error

# Install HTTP client for testing
npm install -D supertest
npm install -D @types/supertest
```

### โครงสร้างไดเรกทอรี

```bash
# สร้างโครงสร้างสำหรับ API
mkdir -p src/app/api/v1/{users,sos-alerts,locations,notifications}
mkdir -p src/lib/{api,validators,middleware}
mkdir -p src/types/api
```

---

## 🛠️ Step-by-Step Implementation

### Step 151: สร้าง Standard Response Format

```typescript
// src/types/api/response.ts
export interface ApiResponse<T = unknown> {
  success: boolean;
  data?: T;
  error?: ApiError;
  meta?: ApiMeta;
}

export interface ApiError {
  code: string;
  message: string;
  details?: Record<string, string[]>;
  traceId?: string;
}

export interface ApiMeta {
  pagination?: PaginationMeta;
  timestamp: string;
  version: string;
}

export interface PaginationMeta {
  cursor?: string;
  nextCursor?: string;
  prevCursor?: string;
  hasMore: boolean;
  total?: number;
  limit: number;
}

// Helper functions
export function successResponse<T>(
  data: T,
  meta?: Partial<ApiMeta>
): ApiResponse<T> {
  return {
    success: true,
    data,
    meta: {
      timestamp: new Date().toISOString(),
      version: '1.0.0',
      ...meta,
    },
  };
}

export function errorResponse(
  code: string,
  message: string,
  details?: Record<string, string[]>,
  traceId?: string
): ApiResponse<never> {
  return {
    success: false,
    error: {
      code,
      message,
      details,
      traceId,
    },
    meta: {
      timestamp: new Date().toISOString(),
      version: '1.0.0',
    },
  };
}
```

### Step 152: สร้าง API Middleware

```typescript
// src/lib/middleware/api-middleware.ts
import { NextRequest, NextResponse } from 'next/server';
import { ZodError } from 'zod';
import { errorResponse } from '@/types/api/response';
import { generateTraceId } from '@/lib/utils/trace';

export type ApiHandler = (
  req: NextRequest,
  context: { params: Record<string, string> }
) => Promise<NextResponse>;

export function withApiMiddleware(handler: ApiHandler): ApiHandler {
  return async (req, context) => {
    const traceId = generateTraceId();
    
    try {
      // Add trace ID to request headers
      const requestWithTrace = new NextRequest(req.url, {
        headers: {
          ...Object.fromEntries(req.headers.entries()),
          'x-trace-id': traceId,
        },
        method: req.method,
        body: req.body,
      });

      const response = await handler(requestWithTrace, context);
      
      // Add standard headers to response
      response.headers.set('X-Trace-Id', traceId);
      response.headers.set('X-API-Version', 'v1');
      response.headers.set('X-Content-Type-Options', 'nosniff');
      
      return response;
    } catch (error) {
      console.error('[API Error]', { traceId, error });
      
      if (error instanceof ZodError) {
        const details: Record<string, string[]> = {};
        error.errors.forEach((err) => {
          const key = err.path.join('.');
          if (!details[key]) details[key] = [];
          details[key].push(err.message);
        });
        
        return NextResponse.json(
          errorResponse('VALIDATION_ERROR', 'Validation failed', details, traceId),
          { status: 422 }
        );
      }
      
      if (error instanceof ApiException) {
        return NextResponse.json(
          errorResponse(error.code, error.message, undefined, traceId),
          { status: error.statusCode }
        );
      }
      
      return NextResponse.json(
        errorResponse('INTERNAL_ERROR', 'An unexpected error occurred', undefined, traceId),
        { status: 500 }
      );
    }
  };
}

export class ApiException extends Error {
  constructor(
    public readonly code: string,
    public readonly message: string,
    public readonly statusCode: number
  ) {
    super(message);
    this.name = 'ApiException';
  }
}

// Common exceptions
export const Exceptions = {
  notFound: (resource: string) =>
    new ApiException('NOT_FOUND', `${resource} not found`, 404),
  unauthorized: () =>
    new ApiException('UNAUTHORIZED', 'Authentication required', 401),
  forbidden: () =>
    new ApiException('FORBIDDEN', 'Insufficient permissions', 403),
  conflict: (message: string) =>
    new ApiException('CONFLICT', message, 409),
  badRequest: (message: string) =>
    new ApiException('BAD_REQUEST', message, 400),
  tooManyRequests: () =>
    new ApiException('RATE_LIMIT_EXCEEDED', 'Too many requests', 429),
};
```

### Step 153: Cursor-Based Pagination

```typescript
// src/lib/api/pagination.ts
import { z } from 'zod';

// Cursor pagination schema
export const cursorPaginationSchema = z.object({
  cursor: z.string().optional(),
  limit: z.coerce.number().min(1).max(100).default(20),
  direction: z.enum(['next', 'prev']).default('next'),
});

export type CursorPagination = z.infer<typeof cursorPaginationSchema>;

// Encode cursor (base64 of id + timestamp)
export function encodeCursor(id: string, createdAt: Date): string {
  const payload = JSON.stringify({ id, createdAt: createdAt.toISOString() });
  return Buffer.from(payload).toString('base64url');
}

// Decode cursor
export function decodeCursor(cursor: string): { id: string; createdAt: Date } {
  const payload = JSON.parse(Buffer.from(cursor, 'base64url').toString());
  return {
    id: payload.id,
    createdAt: new Date(payload.createdAt),
  };
}

// Build cursor query for Prisma
export function buildCursorQuery(
  cursor?: string,
  direction: 'next' | 'prev' = 'next'
) {
  if (!cursor) return {};
  
  const { id, createdAt } = decodeCursor(cursor);
  
  return {
    cursor: { id },
    skip: 1, // Skip the cursor item itself
    where: direction === 'next'
      ? { createdAt: { lte: createdAt } }
      : { createdAt: { gte: createdAt } },
  };
}

// Usage in API route
// src/app/api/v1/sos-alerts/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { db } from '@/lib/db';
import { cursorPaginationSchema, encodeCursor } from '@/lib/api/pagination';
import { successResponse } from '@/types/api/response';

export async function GET(req: NextRequest) {
  const searchParams = req.nextUrl.searchParams;
  const { cursor, limit, direction } = cursorPaginationSchema.parse({
    cursor: searchParams.get('cursor'),
    limit: searchParams.get('limit'),
    direction: searchParams.get('direction'),
  });

  const alerts = await db.sosAlert.findMany({
    take: limit + 1, // Take one extra to check if there's more
    orderBy: { createdAt: 'desc' },
    ...(cursor ? {
      cursor: { id: decodeCursor(cursor).id },
      skip: 1,
    } : {}),
  });

  const hasMore = alerts.length > limit;
  const items = hasMore ? alerts.slice(0, limit) : alerts;
  
  const nextCursor = hasMore
    ? encodeCursor(items[items.length - 1].id, items[items.length - 1].createdAt)
    : undefined;

  return NextResponse.json(
    successResponse(items, {
      pagination: {
        cursor,
        nextCursor,
        hasMore,
        limit,
      },
    })
  );
}
```

### Step 154: Filtering, Sorting, Field Selection

```typescript
// src/lib/api/query-builder.ts
import { z } from 'zod';

// Filter schema builder
export function createFilterSchema<T extends Record<string, z.ZodType>>(
  fields: T
) {
  return z.object(fields).partial();
}

// Sort schema
export const sortSchema = z.object({
  sortBy: z.string().optional(),
  sortOrder: z.enum(['asc', 'desc']).default('desc'),
});

// Field selection schema
export const fieldSelectionSchema = z.object({
  fields: z.string()
    .transform((val) => val.split(',').map((f) => f.trim()))
    .optional(),
});

// SOS Alert filters
export const sosAlertFilterSchema = z.object({
  status: z.enum(['PENDING', 'ASSIGNED', 'RESOLVED', 'CANCELLED']).optional(),
  type: z.enum(['MEDICAL', 'FIRE', 'CRIME', 'ACCIDENT', 'OTHER']).optional(),
  severity: z.enum(['LOW', 'MEDIUM', 'HIGH', 'CRITICAL']).optional(),
  assignedTo: z.string().uuid().optional(),
  lat: z.coerce.number().optional(),
  lng: z.coerce.number().optional(),
  radius: z.coerce.number().positive().max(100).optional(), // km
  fromDate: z.string().datetime().optional(),
  toDate: z.string().datetime().optional(),
  search: z.string().max(100).optional(),
});

// Build Prisma where clause from filters
export function buildFilterWhere(filters: z.infer<typeof sosAlertFilterSchema>) {
  const where: Record<string, unknown> = {};

  if (filters.status) where.status = filters.status;
  if (filters.type) where.type = filters.type;
  if (filters.severity) where.severity = filters.severity;
  if (filters.assignedTo) where.assignedToId = filters.assignedTo;
  
  if (filters.fromDate || filters.toDate) {
    where.createdAt = {
      ...(filters.fromDate ? { gte: new Date(filters.fromDate) } : {}),
      ...(filters.toDate ? { lte: new Date(filters.toDate) } : {}),
    };
  }
  
  if (filters.search) {
    where.OR = [
      { description: { contains: filters.search, mode: 'insensitive' } },
      { location: { contains: filters.search, mode: 'insensitive' } },
    ];
  }

  // Geospatial filter
  if (filters.lat && filters.lng && filters.radius) {
    // Use PostGIS for real geo queries
    // This is simplified - use raw query in production
    where.latitude = {
      gte: filters.lat - (filters.radius / 111),
      lte: filters.lat + (filters.radius / 111),
    };
  }

  return where;
}

// Build Prisma orderBy from sort params
export function buildSortOrderBy(sortBy?: string, sortOrder: 'asc' | 'desc' = 'desc') {
  const allowedFields = ['createdAt', 'updatedAt', 'severity', 'status'];
  
  if (!sortBy || !allowedFields.includes(sortBy)) {
    return { createdAt: 'desc' as const };
  }
  
  return { [sortBy]: sortOrder };
}

// Field selection - return only requested fields
export function selectFields<T extends Record<string, unknown>>(
  data: T,
  fields?: string[]
): Partial<T> {
  if (!fields || fields.length === 0) return data;
  
  return fields.reduce((acc, field) => {
    if (field in data) {
      acc[field as keyof T] = data[field as keyof T];
    }
    return acc;
  }, {} as Partial<T>);
}
```

### Step 155: Rate Limiting Headers

```typescript
// src/lib/middleware/rate-limit.ts
import { NextRequest, NextResponse } from 'next/server';
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from 'ioredis';
import { errorResponse } from '@/types/api/response';

const redis = new Redis(process.env.REDIS_URL!);

// Create rate limiters for different endpoints
export const rateLimiters = {
  // General API: 100 requests per minute
  api: new Ratelimit({
    redis,
    limiter: Ratelimit.slidingWindow(100, '1 m'),
    prefix: 'rl:api',
  }),
  
  // SOS alerts: 10 per minute (prevent spam)
  sos: new Ratelimit({
    redis,
    limiter: Ratelimit.slidingWindow(10, '1 m'),
    prefix: 'rl:sos',
  }),
  
  // Auth endpoints: 5 per minute (prevent brute force)
  auth: new Ratelimit({
    redis,
    limiter: Ratelimit.slidingWindow(5, '1 m'),
    prefix: 'rl:auth',
  }),
};

export async function withRateLimit(
  req: NextRequest,
  limiter: Ratelimit,
  identifier?: string
): Promise<{ success: boolean; response?: NextResponse }> {
  const id = identifier || 
    req.headers.get('x-forwarded-for') || 
    req.headers.get('x-real-ip') || 
    'anonymous';

  const { success, limit, remaining, reset } = await limiter.limit(id);

  const headers = {
    'X-RateLimit-Limit': limit.toString(),
    'X-RateLimit-Remaining': remaining.toString(),
    'X-RateLimit-Reset': reset.toString(),
    'X-RateLimit-Policy': '100;w=60',
  };

  if (!success) {
    const retryAfter = Math.ceil((reset - Date.now()) / 1000);
    return {
      success: false,
      response: NextResponse.json(
        errorResponse(
          'RATE_LIMIT_EXCEEDED',
          `Rate limit exceeded. Retry after ${retryAfter} seconds`
        ),
        {
          status: 429,
          headers: {
            ...headers,
            'Retry-After': retryAfter.toString(),
          },
        }
      ),
    };
  }

  return { success: true };
}

// Middleware wrapper
export function withRateLimitMiddleware(
  handler: Function,
  limiterKey: keyof typeof rateLimiters = 'api'
) {
  return async (req: NextRequest, context: unknown) => {
    const { success, response } = await withRateLimit(req, rateLimiters[limiterKey]);
    if (!success) return response;
    return handler(req, context);
  };
}
```

### Step 156: Idempotency Keys

```typescript
// src/lib/middleware/idempotency.ts
import { NextRequest, NextResponse } from 'next/server';
import { Redis } from 'ioredis';

const redis = new Redis(process.env.REDIS_URL!);
const IDEMPOTENCY_TTL = 86400; // 24 hours

export async function withIdempotency(
  req: NextRequest,
  handler: (req: NextRequest) => Promise<NextResponse>
): Promise<NextResponse> {
  const idempotencyKey = req.headers.get('Idempotency-Key');
  
  if (!idempotencyKey) {
    return handler(req);
  }
  
  // Validate idempotency key format (UUID v4)
  const uuidRegex = /^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;
  if (!uuidRegex.test(idempotencyKey)) {
    return NextResponse.json(
      { success: false, error: { code: 'INVALID_IDEMPOTENCY_KEY', message: 'Invalid idempotency key format' } },
      { status: 400 }
    );
  }
  
  const cacheKey = `idempotency:${idempotencyKey}`;
  
  // Check if we have a cached response
  const cached = await redis.get(cacheKey);
  if (cached) {
    const cachedResponse = JSON.parse(cached);
    return NextResponse.json(cachedResponse.body, {
      status: cachedResponse.status,
      headers: {
        ...cachedResponse.headers,
        'Idempotency-Replayed': 'true',
      },
    });
  }
  
  // Mark as processing (prevent concurrent duplicate requests)
  const lock = await redis.set(
    `${cacheKey}:lock`,
    '1',
    'EX',
    30,
    'NX'
  );
  
  if (!lock) {
    return NextResponse.json(
      { success: false, error: { code: 'CONCURRENT_REQUEST', message: 'Duplicate request in progress' } },
      { status: 409 }
    );
  }
  
  try {
    const response = await handler(req);
    const body = await response.json();
    
    // Cache the response
    await redis.setex(
      cacheKey,
      IDEMPOTENCY_TTL,
      JSON.stringify({
        body,
        status: response.status,
        headers: Object.fromEntries(response.headers.entries()),
      })
    );
    
    return NextResponse.json(body, { status: response.status });
  } finally {
    await redis.del(`${cacheKey}:lock`);
  }
}
```

### Step 157: Webhook Design

```typescript
// src/lib/webhooks/webhook-dispatcher.ts
import crypto from 'crypto';
import { db } from '@/lib/db';

interface WebhookPayload {
  id: string;
  event: string;
  data: Record<string, unknown>;
  timestamp: string;
  version: string;
}

// Sign webhook payload with HMAC-SHA256
export function signWebhookPayload(payload: string, secret: string): string {
  return `sha256=${crypto
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex')}`;
}

// Verify incoming webhook signature
export function verifyWebhookSignature(
  payload: string,
  signature: string,
  secret: string
): boolean {
  const expectedSignature = signWebhookPayload(payload, secret);
  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expectedSignature)
  );
}

// Dispatch webhook to registered endpoints
export async function dispatchWebhook(
  event: string,
  data: Record<string, unknown>,
  userId?: string
): Promise<void> {
  const webhooks = await db.webhook.findMany({
    where: {
      events: { has: event },
      active: true,
      ...(userId ? { userId } : {}),
    },
  });

  const payload: WebhookPayload = {
    id: crypto.randomUUID(),
    event,
    data,
    timestamp: new Date().toISOString(),
    version: '1.0',
  };

  const payloadString = JSON.stringify(payload);

  await Promise.allSettled(
    webhooks.map(async (webhook) => {
      const signature = signWebhookPayload(payloadString, webhook.secret);
      
      try {
        const response = await fetch(webhook.url, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'X-Webhook-Signature': signature,
            'X-Webhook-Event': event,
            'X-Webhook-Id': payload.id,
          },
          body: payloadString,
          signal: AbortSignal.timeout(10000), // 10 second timeout
        });

        await db.webhookDelivery.create({
          data: {
            webhookId: webhook.id,
            event,
            payloadId: payload.id,
            status: response.ok ? 'SUCCESS' : 'FAILED',
            statusCode: response.status,
            responseBody: await response.text(),
          },
        });
      } catch (error) {
        await db.webhookDelivery.create({
          data: {
            webhookId: webhook.id,
            event,
            payloadId: payload.id,
            status: 'FAILED',
            error: error instanceof Error ? error.message : 'Unknown error',
          },
        });
        
        // Schedule retry with exponential backoff
        await scheduleWebhookRetry(webhook.id, payloadString, signature);
      }
    })
  );
}

// Retry failed webhook with exponential backoff
async function scheduleWebhookRetry(
  webhookId: string,
  payload: string,
  signature: string,
  attempt = 1
): Promise<void> {
  if (attempt > 5) return; // Max 5 retries
  
  const delay = Math.pow(2, attempt) * 1000; // 2s, 4s, 8s, 16s, 32s
  
  // In production, use a job queue like Bull/BullMQ
  setTimeout(async () => {
    const webhook = await db.webhook.findUnique({ where: { id: webhookId } });
    if (!webhook?.active) return;
    
    try {
      await fetch(webhook.url, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'X-Webhook-Signature': signature,
          'X-Webhook-Retry-Attempt': attempt.toString(),
        },
        body: payload,
      });
    } catch {
      await scheduleWebhookRetry(webhookId, payload, signature, attempt + 1);
    }
  }, delay);
}
```

### Step 158: OpenAPI 3.1 Documentation

```typescript
// src/lib/api/openapi.ts
import { OpenAPIRegistry, OpenApiGeneratorV31 } from '@asteasolutions/zod-to-openapi';
import { z } from 'zod';

const registry = new OpenAPIRegistry();

// Register schemas
const UserSchema = registry.register('User', z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  name: z.string(),
  role: z.enum(['admin', 'moderator', 'user', 'sos_responder']),
  createdAt: z.string().datetime(),
  updatedAt: z.string().datetime(),
}));

const SosAlertSchema = registry.register('SosAlert', z.object({
  id: z.string().uuid(),
  type: z.enum(['MEDICAL', 'FIRE', 'CRIME', 'ACCIDENT', 'OTHER']),
  severity: z.enum(['LOW', 'MEDIUM', 'HIGH', 'CRITICAL']),
  status: z.enum(['PENDING', 'ASSIGNED', 'RESOLVED', 'CANCELLED']),
  description: z.string(),
  latitude: z.number(),
  longitude: z.number(),
  location: z.string(),
  reportedBy: z.string().uuid(),
  assignedTo: z.string().uuid().optional(),
  createdAt: z.string().datetime(),
  updatedAt: z.string().datetime(),
}));

const ApiErrorSchema = registry.register('ApiError', z.object({
  success: z.literal(false),
  error: z.object({
    code: z.string(),
    message: z.string(),
    details: z.record(z.array(z.string())).optional(),
    traceId: z.string().optional(),
  }),
}));

// Register paths
registry.registerPath({
  method: 'get',
  path: '/api/v1/sos-alerts',
  summary: 'List SOS alerts',
  tags: ['SOS Alerts'],
  security: [{ bearerAuth: [] }],
  request: {
    query: z.object({
      cursor: z.string().optional().describe('Pagination cursor'),
      limit: z.coerce.number().min(1).max(100).default(20),
      status: z.enum(['PENDING', 'ASSIGNED', 'RESOLVED', 'CANCELLED']).optional(),
      severity: z.enum(['LOW', 'MEDIUM', 'HIGH', 'CRITICAL']).optional(),
    }),
  },
  responses: {
    200: {
      description: 'List of SOS alerts',
      content: {
        'application/json': {
          schema: z.object({
            success: z.literal(true),
            data: z.array(SosAlertSchema),
            meta: z.object({
              pagination: z.object({
                nextCursor: z.string().optional(),
                hasMore: z.boolean(),
                limit: z.number(),
              }),
              timestamp: z.string(),
              version: z.string(),
            }),
          }),
        },
      },
    },
    401: {
      description: 'Unauthorized',
      content: { 'application/json': { schema: ApiErrorSchema } },
    },
  },
});

registry.registerPath({
  method: 'post',
  path: '/api/v1/sos-alerts',
  summary: 'Create SOS alert',
  tags: ['SOS Alerts'],
  security: [{ bearerAuth: [] }],
  request: {
    headers: z.object({
      'Idempotency-Key': z.string().uuid().describe('Prevent duplicate submissions'),
    }),
    body: {
      content: {
        'application/json': {
          schema: z.object({
            type: z.enum(['MEDICAL', 'FIRE', 'CRIME', 'ACCIDENT', 'OTHER']),
            severity: z.enum(['LOW', 'MEDIUM', 'HIGH', 'CRITICAL']),
            description: z.string().min(10).max(1000),
            latitude: z.number().min(-90).max(90),
            longitude: z.number().min(-180).max(180),
            location: z.string().max(500),
          }),
        },
      },
    },
  },
  responses: {
    201: {
      description: 'SOS alert created',
      content: {
        'application/json': {
          schema: z.object({
            success: z.literal(true),
            data: SosAlertSchema,
          }),
        },
      },
    },
  },
});

// Generate OpenAPI document
export function generateOpenApiDocument() {
  const generator = new OpenApiGeneratorV31(registry.definitions);
  
  return generator.generateDocument({
    openapi: '3.1.0',
    info: {
      title: 'Chuaikan API',
      version: '1.0.0',
      description: 'API for Chuaikan emergency response platform',
      contact: {
        name: 'Chuaikan Support',
        email: 'api@chuaikan.com',
      },
    },
    servers: [
      {
        url: 'https://api.chuaikan.com',
        description: 'Production',
      },
      {
        url: 'https://staging-api.chuaikan.com',
        description: 'Staging',
      },
      {
        url: 'http://localhost:3000',
        description: 'Development',
      },
    ],
    components: {
      securitySchemes: {
        bearerAuth: {
          type: 'http',
          scheme: 'bearer',
          bearerFormat: 'JWT',
        },
        apiKey: {
          type: 'apiKey',
          in: 'header',
          name: 'X-API-Key',
        },
      },
    },
  });
}
```

```typescript
// src/app/api/docs/route.ts
import { NextResponse } from 'next/server';
import { generateOpenApiDocument } from '@/lib/api/openapi';

export async function GET() {
  const document = generateOpenApiDocument();
  return NextResponse.json(document);
}
```

```typescript
// src/app/api-docs/page.tsx
'use client';
import dynamic from 'next/dynamic';
import 'swagger-ui-react/swagger-ui.css';

const SwaggerUI = dynamic(
  () => import('swagger-ui-react'),
  { ssr: false }
);

export default function ApiDocsPage() {
  return (
    <div className="api-docs">
      <SwaggerUI url="/api/docs" />
    </div>
  );
}
```

### Step 159: Error Response Format Standard

```typescript
// src/lib/api/errors.ts

// Error codes enum
export enum ErrorCode {
  // Auth errors
  UNAUTHORIZED = 'UNAUTHORIZED',
  FORBIDDEN = 'FORBIDDEN',
  TOKEN_EXPIRED = 'TOKEN_EXPIRED',
  INVALID_TOKEN = 'INVALID_TOKEN',
  
  // Validation errors
  VALIDATION_ERROR = 'VALIDATION_ERROR',
  INVALID_INPUT = 'INVALID_INPUT',
  
  // Resource errors
  NOT_FOUND = 'NOT_FOUND',
  ALREADY_EXISTS = 'ALREADY_EXISTS',
  CONFLICT = 'CONFLICT',
  
  // Rate limit errors
  RATE_LIMIT_EXCEEDED = 'RATE_LIMIT_EXCEEDED',
  
  // Business logic errors
  SOS_ALREADY_ASSIGNED = 'SOS_ALREADY_ASSIGNED',
  SOS_ALREADY_RESOLVED = 'SOS_ALREADY_RESOLVED',
  INSUFFICIENT_PERMISSIONS = 'INSUFFICIENT_PERMISSIONS',
  
  // Server errors
  INTERNAL_ERROR = 'INTERNAL_ERROR',
  SERVICE_UNAVAILABLE = 'SERVICE_UNAVAILABLE',
  DATABASE_ERROR = 'DATABASE_ERROR',
}

// Map error codes to HTTP status codes
export const ERROR_STATUS_MAP: Record<ErrorCode, number> = {
  [ErrorCode.UNAUTHORIZED]: 401,
  [ErrorCode.FORBIDDEN]: 403,
  [ErrorCode.TOKEN_EXPIRED]: 401,
  [ErrorCode.INVALID_TOKEN]: 401,
  [ErrorCode.VALIDATION_ERROR]: 422,
  [ErrorCode.INVALID_INPUT]: 400,
  [ErrorCode.NOT_FOUND]: 404,
  [ErrorCode.ALREADY_EXISTS]: 409,
  [ErrorCode.CONFLICT]: 409,
  [ErrorCode.RATE_LIMIT_EXCEEDED]: 429,
  [ErrorCode.SOS_ALREADY_ASSIGNED]: 409,
  [ErrorCode.SOS_ALREADY_RESOLVED]: 409,
  [ErrorCode.INSUFFICIENT_PERMISSIONS]: 403,
  [ErrorCode.INTERNAL_ERROR]: 500,
  [ErrorCode.SERVICE_UNAVAILABLE]: 503,
  [ErrorCode.DATABASE_ERROR]: 500,
};

// Error messages in Thai and English
export const ERROR_MESSAGES: Record<ErrorCode, string> = {
  [ErrorCode.UNAUTHORIZED]: 'กรุณาเข้าสู่ระบบก่อน',
  [ErrorCode.FORBIDDEN]: 'คุณไม่มีสิทธิ์เข้าถึงส่วนนี้',
  [ErrorCode.TOKEN_EXPIRED]: 'เซสชันหมดอายุ กรุณาเข้าสู่ระบบใหม่',
  [ErrorCode.INVALID_TOKEN]: 'Token ไม่ถูกต้อง',
  [ErrorCode.VALIDATION_ERROR]: 'ข้อมูลที่ส่งมาไม่ถูกต้อง',
  [ErrorCode.INVALID_INPUT]: 'ข้อมูล input ไม่ถูกต้อง',
  [ErrorCode.NOT_FOUND]: 'ไม่พบข้อมูลที่ต้องการ',
  [ErrorCode.ALREADY_EXISTS]: 'ข้อมูลนี้มีอยู่แล้ว',
  [ErrorCode.CONFLICT]: 'ข้อมูลขัดแย้งกัน',
  [ErrorCode.RATE_LIMIT_EXCEEDED]: 'ส่งคำขอมากเกินไป กรุณารอสักครู่',
  [ErrorCode.SOS_ALREADY_ASSIGNED]: 'การแจ้งเหตุนี้ถูกรับมอบหมายแล้ว',
  [ErrorCode.SOS_ALREADY_RESOLVED]: 'การแจ้งเหตุนี้ถูกแก้ไขแล้ว',
  [ErrorCode.INSUFFICIENT_PERMISSIONS]: 'ไม่มีสิทธิ์ดำเนินการ',
  [ErrorCode.INTERNAL_ERROR]: 'เกิดข้อผิดพลาดภายในระบบ',
  [ErrorCode.SERVICE_UNAVAILABLE]: 'ระบบไม่พร้อมให้บริการชั่วคราว',
  [ErrorCode.DATABASE_ERROR]: 'เกิดข้อผิดพลาดกับฐานข้อมูล',
};
```

### Step 160: Complete API Routes for chuaikan.com

```typescript
// src/app/api/v1/sos-alerts/[id]/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { z } from 'zod';
import { db } from '@/lib/db';
import { successResponse, errorResponse } from '@/types/api/response';
import { withApiMiddleware, Exceptions } from '@/lib/middleware/api-middleware';
import { withRateLimitMiddleware } from '@/lib/middleware/rate-limit';
import { withIdempotency } from '@/lib/middleware/idempotency';
import { dispatchWebhook } from '@/lib/webhooks/webhook-dispatcher';
import { getAuthUser } from '@/lib/auth/get-auth-user';
import { ErrorCode } from '@/lib/api/errors';

const updateSosAlertSchema = z.object({
  status: z.enum(['ASSIGNED', 'RESOLVED', 'CANCELLED']).optional(),
  assignedTo: z.string().uuid().optional(),
  notes: z.string().max(2000).optional(),
  resolvedAt: z.string().datetime().optional(),
});

// GET /api/v1/sos-alerts/:id
export const GET = withApiMiddleware(async (req, { params }) => {
  const user = await getAuthUser(req);
  if (!user) throw Exceptions.unauthorized();

  const alert = await db.sosAlert.findUnique({
    where: { id: params.id },
    include: {
      reportedByUser: {
        select: { id: true, name: true, phone: true },
      },
      assignedToUser: {
        select: { id: true, name: true, phone: true },
      },
      timeline: {
        orderBy: { createdAt: 'asc' },
      },
    },
  });

  if (!alert) throw Exceptions.notFound('SOS Alert');

  // Check permissions - users can only see their own alerts unless admin/responder
  if (
    user.role === 'user' &&
    alert.reportedById !== user.id &&
    alert.assignedToId !== user.id
  ) {
    throw Exceptions.forbidden();
  }

  return NextResponse.json(successResponse(alert));
});

// PATCH /api/v1/sos-alerts/:id
export const PATCH = withRateLimitMiddleware(
  withApiMiddleware(async (req, { params }) => {
    const user = await getAuthUser(req);
    if (!user) throw Exceptions.unauthorized();
    
    // Only admins and responders can update alerts
    if (!['admin', 'moderator', 'sos_responder'].includes(user.role)) {
      throw Exceptions.forbidden();
    }

    const body = await req.json();
    const data = updateSosAlertSchema.parse(body);

    const alert = await db.sosAlert.findUnique({
      where: { id: params.id },
    });

    if (!alert) throw Exceptions.notFound('SOS Alert');
    
    if (alert.status === 'RESOLVED' || alert.status === 'CANCELLED') {
      return NextResponse.json(
        errorResponse(
          ErrorCode.SOS_ALREADY_RESOLVED,
          'Cannot update a resolved or cancelled alert'
        ),
        { status: 409 }
      );
    }

    const updated = await db.sosAlert.update({
      where: { id: params.id },
      data: {
        ...data,
        updatedAt: new Date(),
        timeline: {
          create: {
            action: `Status updated to ${data.status}`,
            userId: user.id,
          },
        },
      },
    });

    // Dispatch webhook event
    await dispatchWebhook('sos_alert.updated', {
      alertId: updated.id,
      previousStatus: alert.status,
      newStatus: updated.status,
      updatedBy: user.id,
    });

    return NextResponse.json(successResponse(updated));
  }),
  'sos'
);
```

---

## 🔧 Configuration Files

```yaml
# openapi.yaml - สำหรับ chuaikan.com
openapi: 3.1.0
info:
  title: Chuaikan Emergency Response API
  version: 1.0.0
  description: |
    API สำหรับระบบแจ้งเหตุฉุกเฉิน chuaikan.com
    
    ## Authentication
    API นี้ใช้ JWT Bearer token สำหรับ authentication
    
    ## Rate Limiting
    - General API: 100 requests/minute
    - SOS Endpoints: 10 requests/minute
    - Auth Endpoints: 5 requests/minute
    
  contact:
    name: Chuaikan API Support
    email: api-support@chuaikan.com
    url: https://docs.chuaikan.com

servers:
  - url: https://api.chuaikan.com/api/v1
    description: Production
  - url: http://localhost:3000/api/v1
    description: Development

security:
  - bearerAuth: []

tags:
  - name: SOS Alerts
    description: จัดการการแจ้งเหตุฉุกเฉิน
  - name: Users
    description: จัดการข้อมูลผู้ใช้
  - name: Locations
    description: ข้อมูลตำแหน่ง
  - name: Notifications
    description: การแจ้งเตือน
  - name: Webhooks
    description: จัดการ webhook endpoints

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
    apiKey:
      type: apiKey
      in: header
      name: X-API-Key
```

---

## 🧪 Testing

```typescript
// __tests__/api/sos-alerts.test.ts
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import request from 'supertest';
import { createTestServer } from '../helpers/test-server';
import { createTestUser, createTestSosAlert } from '../helpers/factories';
import { generateTestToken } from '../helpers/auth';

describe('SOS Alerts API', () => {
  let server: any;
  let authToken: string;
  let adminToken: string;

  beforeAll(async () => {
    server = await createTestServer();
    const user = await createTestUser({ role: 'user' });
    const admin = await createTestUser({ role: 'admin' });
    authToken = generateTestToken(user);
    adminToken = generateTestToken(admin);
  });

  afterAll(async () => {
    await server.close();
  });

  describe('GET /api/v1/sos-alerts', () => {
    it('should return paginated list of alerts', async () => {
      const response = await request(server)
        .get('/api/v1/sos-alerts')
        .set('Authorization', `Bearer ${authToken}`)
        .query({ limit: 10 });

      expect(response.status).toBe(200);
      expect(response.body.success).toBe(true);
      expect(response.body.meta.pagination).toBeDefined();
      expect(response.body.meta.pagination.limit).toBe(10);
    });

    it('should filter by status', async () => {
      const response = await request(server)
        .get('/api/v1/sos-alerts')
        .set('Authorization', `Bearer ${authToken}`)
        .query({ status: 'PENDING' });

      expect(response.status).toBe(200);
      response.body.data.forEach((alert: any) => {
        expect(alert.status).toBe('PENDING');
      });
    });

    it('should return 401 without auth token', async () => {
      const response = await request(server)
        .get('/api/v1/sos-alerts');

      expect(response.status).toBe(401);
      expect(response.body.success).toBe(false);
      expect(response.body.error.code).toBe('UNAUTHORIZED');
    });
  });

  describe('POST /api/v1/sos-alerts', () => {
    it('should create SOS alert with idempotency key', async () => {
      const idempotencyKey = crypto.randomUUID();
      const alertData = {
        type: 'MEDICAL',
        severity: 'HIGH',
        description: 'Emergency medical situation requires immediate assistance',
        latitude: 13.7563,
        longitude: 100.5018,
        location: 'Bangkok, Thailand',
      };

      const response1 = await request(server)
        .post('/api/v1/sos-alerts')
        .set('Authorization', `Bearer ${authToken}`)
        .set('Idempotency-Key', idempotencyKey)
        .send(alertData);

      const response2 = await request(server)
        .post('/api/v1/sos-alerts')
        .set('Authorization', `Bearer ${authToken}`)
        .set('Idempotency-Key', idempotencyKey)
        .send(alertData);

      expect(response1.status).toBe(201);
      expect(response2.status).toBe(201);
      expect(response2.headers['idempotency-replayed']).toBe('true');
      // Same data returned for both requests
      expect(response1.body.data.id).toBe(response2.body.data.id);
    });

    it('should rate limit excessive SOS creation', async () => {
      const requests = Array(11).fill(null).map(() =>
        request(server)
          .post('/api/v1/sos-alerts')
          .set('Authorization', `Bearer ${authToken}`)
          .send({
            type: 'OTHER',
            severity: 'LOW',
            description: 'Test alert for rate limiting purposes',
            latitude: 13.7563,
            longitude: 100.5018,
            location: 'Test Location',
          })
      );

      const responses = await Promise.all(requests);
      const rateLimited = responses.filter(r => r.status === 429);
      expect(rateLimited.length).toBeGreaterThan(0);
    });
  });
});
```

---

## ❌ Common Errors & Solutions

### Error 1: CORS Issues

```
Error: CORS policy blocked request from https://app.chuaikan.com
```

**แก้ไข:**
```typescript
// next.config.ts
const nextConfig = {
  async headers() {
    return [
      {
        source: '/api/:path*',
        headers: [
          { key: 'Access-Control-Allow-Origin', value: process.env.ALLOWED_ORIGIN! },
          { key: 'Access-Control-Allow-Methods', value: 'GET, POST, PUT, PATCH, DELETE, OPTIONS' },
          { key: 'Access-Control-Allow-Headers', value: 'Content-Type, Authorization, X-API-Key, Idempotency-Key' },
          { key: 'Access-Control-Max-Age', value: '86400' },
        ],
      },
    ];
  },
};
```

### Error 2: Rate Limit Redis Connection

```
Error: Redis connection failed
```

**แก้ไข:**
```bash
# ตรวจสอบ Redis connection
redis-cli -h localhost -p 6379 ping

# ตรวจสอบ environment variable
echo $REDIS_URL

# ถ้าใช้ Redis Cluster
export REDIS_URL="redis://localhost:6379"
```

### Error 3: Pagination Cursor Invalid

```
Error: Invalid cursor format
```

**แก้ไข:**
```typescript
// เพิ่ม validation สำหรับ cursor
function safeDecodeCursor(cursor: string) {
  try {
    return decodeCursor(cursor);
  } catch {
    throw new ApiException('INVALID_CURSOR', 'Invalid pagination cursor', 400);
  }
}
```

---

## ✅ Checklist

### REST Design
- [ ] ใช้ nouns ใน URL ไม่ใช่ verbs
- [ ] ใช้ plural form สำหรับ resource names
- [ ] HTTP methods ถูกต้องตาม semantics
- [ ] Status codes ถูกต้องสำหรับแต่ละ operation
- [ ] Nested resources ไม่เกิน 2 levels

### Versioning
- [ ] API version อยู่ใน URL path (/api/v1/)
- [ ] มี deprecation notice สำหรับ old versions
- [ ] Changelog สำหรับแต่ละ version

### Pagination
- [ ] ใช้ cursor-based pagination สำหรับ feeds
- [ ] Limit มี maximum value (100)
- [ ] Pagination meta อยู่ใน response

### Rate Limiting
- [ ] Headers: X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset
- [ ] 429 status code เมื่อ exceed limit
- [ ] Retry-After header

### Documentation
- [ ] OpenAPI 3.1 spec ครบทุก endpoint
- [ ] Swagger UI accessible ที่ /api-docs
- [ ] Error codes ทุกตัวมีคำอธิบาย
- [ ] Request/Response examples ทุก endpoint

### Security
- [ ] API keys สำหรับ third-party access
- [ ] JWT authentication สำหรับ user actions
- [ ] Idempotency keys สำหรับ POST requests
- [ ] Webhook signatures ตรวจสอบได้

### Error Handling
- [ ] Standard error format ทุก endpoint
- [ ] Trace IDs ใน error responses
- [ ] Validation errors มี field-level details
- [ ] ไม่ expose internal error details ใน production

---

## 🔗 References

- [REST API Design Best Practices](https://restfulapi.net/)
- [OpenAPI 3.1 Specification](https://spec.openapis.org/oas/v3.1.0)
- [JSON API Specification](https://jsonapi.org/)
- [Zod to OpenAPI](https://github.com/asteasolutions/zod-to-openapi)
- [Upstash Rate Limiting](https://upstash.com/docs/redis/sdks/ratelimit-ts/overview)

---
*Part 016 | Road to 1,000,000 Users/Day | chuaikan.com*
