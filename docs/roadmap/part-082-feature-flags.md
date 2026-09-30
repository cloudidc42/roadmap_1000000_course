# Part 082: Feature Flags
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Professional
> **Steps:** 811–820
> **เวลาโดยประมาณ:** 6 ชั่วโมง
> **Prerequisites:** Part 081 (Canary), Part 006 (Docker)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- Feature flag use cases: gradual rollout, A/B testing, kill switch
- Unleash (open-source) installation และ setup
- Feature flag types: boolean, variant, gradual rollout
- Flag integration ใน Next.js (server และ client)
- Flag integration ใน Node.js API
- Flag targeting: user segment, geography, subscription tier
- Flag lifecycle: create → rollout → archive
- Flag performance impact (caching flags ใน Redis)

---

## 📖 ทฤษฎีและแนวคิด

### Feature Flag Use Cases สำหรับ chuaikan.com

| Use Case | Flag Type | ตัวอย่าง |
|----------|-----------|---------|
| Gradual rollout | Gradual | ปล่อย new feed algorithm ทีละ 10% |
| A/B testing | Variant | ทดสอบ SOS button สีแดง vs ส้ม |
| Kill switch | Boolean | ปิด feature ฉุกเฉิน |
| Beta users | User targeting | เปิดสำหรับ premium users เท่านั้น |
| Regional | Geography | เปิดเฉพาะในไทย |

---

## ⚙️ Environment Setup

### Step 811: ติดตั้ง Unleash Server

```yaml
# docker-compose.unleash.yml
version: '3.8'

services:
  unleash-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: unleash
      POSTGRES_PASSWORD: unleash_password
      POSTGRES_DB: unleash
    volumes:
      - unleash-db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U unleash"]
      interval: 10s
      timeout: 5s
      retries: 5

  unleash:
    image: unleashorg/unleash-server:6
    ports:
      - "4242:4242"
    environment:
      DATABASE_URL: postgresql://unleash:unleash_password@unleash-db:5432/unleash
      INIT_FRONTEND_API_TOKENS: "default:development.unleash-insecure-frontend-api-token"
      INIT_CLIENT_API_TOKENS: "default:development.unleash-insecure-api-token"
    depends_on:
      unleash-db:
        condition: service_healthy
    healthcheck:
      test: wget --no-verbose --tries=1 --spider http://localhost:4242/health
      interval: 30s
      timeout: 10s
      retries: 5
    restart: unless-stopped

volumes:
  unleash-db-data:
```

```bash
# Start Unleash
docker compose -f docker-compose.unleash.yml up -d

# รอ Unleash พร้อม
sleep 30
curl http://localhost:4242/health
# {"health":"GOOD"}

# Login: http://localhost:4242
# Username: admin
# Password: unleash4all
```

---

## 🛠️ Step-by-Step Implementation

### Step 812: สร้าง Feature Flags ผ่าน API

```bash
# ตั้งค่า API token
export UNLEASH_URL="http://localhost:4242"
export UNLEASH_API_TOKEN="*:development.unleash-insecure-api-token"

# สร้าง flag: new-feed-algorithm
curl -X POST "${UNLEASH_URL}/api/admin/projects/default/features" \
  -H "Authorization: ${UNLEASH_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "new-feed-algorithm",
    "description": "New ML-based feed ranking algorithm",
    "type": "release",
    "impressionData": true
  }'

# Enable flag ด้วย gradual rollout 10%
curl -X POST "${UNLEASH_URL}/api/admin/projects/default/features/new-feed-algorithm/environments/production/strategies" \
  -H "Authorization: ${UNLEASH_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "gradualRollout",
    "parameters": {
      "rollout": "10",
      "stickiness": "userId",
      "groupId": "new-feed-algorithm"
    }
  }'

# สร้าง kill switch: maintenance-mode
curl -X POST "${UNLEASH_URL}/api/admin/projects/default/features" \
  -H "Authorization: ${UNLEASH_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "maintenance-mode",
    "description": "Enable maintenance mode - blocks all API requests",
    "type": "kill-switch",
    "impressionData": false
  }'

# สร้าง variant flag: sos-button-color
curl -X POST "${UNLEASH_URL}/api/admin/projects/default/features" \
  -H "Authorization: ${UNLEASH_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "sos-button-color",
    "description": "A/B test SOS button color",
    "type": "experiment",
    "variants": [
      {"name": "red", "weight": 500, "weightType": "variable"},
      {"name": "orange", "weight": 500, "weightType": "variable"}
    ]
  }'
```

### Step 813: Node.js API Integration

```typescript
// src/lib/feature-flags.ts
import { initialize, isEnabled, getVariant } from 'unleash-client';
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

// Initialize Unleash client
export const unleash = initialize({
  url: process.env.UNLEASH_URL || 'http://unleash.internal:4242/api',
  appName: 'chuaikan-api',
  customHeaders: {
    Authorization: process.env.UNLEASH_API_TOKEN
  },
  // Cache flags locally สำหรับ performance
  refreshInterval: 15,    // sync ทุก 15 วินาที
  metricsInterval: 60     // ส่ง metrics ทุก 60 วินาที
});

unleash.on('ready', () => {
  console.log('Unleash client is ready');
});

unleash.on('error', (err) => {
  console.error('Unleash error:', err);
});

// Helper สำหรับ context ที่มี user info
export function createContext(userId: string, sessionId?: string, properties?: Record<string, string>) {
  return {
    userId,
    sessionId,
    properties: {
      ...properties,
      appName: 'chuaikan-api'
    }
  };
}

// Cache flag values ใน Redis สำหรับ performance
const FLAG_CACHE_TTL = 30; // 30 วินาที

export async function isFeatureEnabled(
  flagName: string,
  userId: string,
  properties?: Record<string, string>
): Promise<boolean> {
  const cacheKey = `flag:${flagName}:${userId}`;

  // ลอง cache ก่อน
  try {
    const cached = await redis.get(cacheKey);
    if (cached !== null) {
      return cached === '1';
    }
  } catch (e) {
    // Redis unavailable: fallback ไป Unleash
  }

  const context = createContext(userId, undefined, properties);
  const enabled = unleash.isEnabled(flagName, context);

  // Cache ไว้
  try {
    await redis.setex(cacheKey, FLAG_CACHE_TTL, enabled ? '1' : '0');
  } catch (e) {
    // ไม่ cache ได้ ก็ยังทำงานต่อได้
  }

  return enabled;
}

export async function getFeatureVariant(
  flagName: string,
  userId: string
): Promise<string> {
  const context = createContext(userId);
  const variant = unleash.getVariant(flagName, context);
  return variant.name || 'control';
}
```

```typescript
// src/middleware/feature-flag.middleware.ts
import { Request, Response, NextFunction } from 'express';
import { isFeatureEnabled } from '../lib/feature-flags';

// Middleware: ตรวจสอบ maintenance mode
export async function checkMaintenanceMode(
  req: Request,
  res: Response,
  next: NextFunction
) {
  const inMaintenance = await isFeatureEnabled('maintenance-mode', 'system');

  if (inMaintenance) {
    return res.status(503).json({
      error: 'Service temporarily unavailable',
      message: 'ระบบอยู่ในช่วงบำรุงรักษา กรุณาลองใหม่ในภายหลัง',
      retryAfter: 3600
    });
  }

  next();
}

// Middleware: เพิ่ม feature flag context ให้ request
export async function attachFeatureFlags(
  req: Request,
  res: Response,
  next: NextFunction
) {
  if (req.user) {
    const [newFeed, sosV2] = await Promise.all([
      isFeatureEnabled('new-feed-algorithm', req.user.id),
      isFeatureEnabled('sos-alert-v2', req.user.id, {
        subscriptionTier: req.user.subscriptionTier
      })
    ]);

    req.features = {
      newFeedAlgorithm: newFeed,
      sosAlertV2: sosV2
    };
  }

  next();
}
```

### Step 814: Next.js Integration

```typescript
// src/lib/server-flags.ts
// Server-side feature flags (ใน Next.js Server Components)

import { initialize, isEnabled } from 'unleash-client';

let unleashClient: ReturnType<typeof initialize> | null = null;

export function getUnleashClient() {
  if (!unleashClient) {
    unleashClient = initialize({
      url: process.env.UNLEASH_URL!,
      appName: 'chuaikan-web',
      customHeaders: {
        Authorization: process.env.UNLEASH_SERVER_API_TOKEN!
      }
    });
  }
  return unleashClient;
}

// ใช้ใน Server Component
export async function serverIsEnabled(
  flagName: string,
  userId?: string
): Promise<boolean> {
  const client = getUnleashClient();
  const context = userId ? { userId } : {};
  return client.isEnabled(flagName, context);
}
```

```typescript
// app/feed/page.tsx
// Next.js App Router - Server Component
import { cookies } from 'next/headers';
import { serverIsEnabled } from '@/lib/server-flags';
import { FeedV1 } from '@/components/FeedV1';
import { FeedV2 } from '@/components/FeedV2';

export default async function FeedPage() {
  const cookieStore = cookies();
  const userId = cookieStore.get('user_id')?.value;

  const useNewFeed = await serverIsEnabled('new-feed-algorithm', userId);

  if (useNewFeed) {
    return <FeedV2 />;
  }

  return <FeedV1 />;
}
```

```typescript
// hooks/useFeatureFlag.ts
// Client-side feature flags (React Hook)
'use client';

import { useEffect, useState } from 'react';
import { UnleashClient } from 'unleash-proxy-client';

const unleashClient = new UnleashClient({
  url: process.env.NEXT_PUBLIC_UNLEASH_PROXY_URL!,
  clientKey: process.env.NEXT_PUBLIC_UNLEASH_CLIENT_KEY!,
  appName: 'chuaikan-web',
  refreshInterval: 30
});

let initialized = false;

export function useFeatureFlag(flagName: string, userId?: string): boolean {
  const [enabled, setEnabled] = useState(false);

  useEffect(() => {
    async function init() {
      if (!initialized) {
        if (userId) {
          unleashClient.updateContext({ userId });
        }
        await unleashClient.start();
        initialized = true;
      }

      setEnabled(unleashClient.isEnabled(flagName));

      unleashClient.on('update', () => {
        setEnabled(unleashClient.isEnabled(flagName));
      });
    }

    init();
  }, [flagName, userId]);

  return enabled;
}

// ใช้ใน Component
// const showNewSOS = useFeatureFlag('sos-alert-v2', currentUser.id);
```

### Step 815: Flag Lifecycle Management

```bash
# scripts/flag-lifecycle.sh
# จัดการ lifecycle ของ feature flags

# 1. สร้าง flag ใหม่ (เริ่ม disabled)
create_flag() {
    local name=$1
    local description=$2

    curl -X POST "${UNLEASH_URL}/api/admin/projects/default/features" \
      -H "Authorization: ${UNLEASH_API_TOKEN}" \
      -H "Content-Type: application/json" \
      -d "{
        \"name\": \"${name}\",
        \"description\": \"${description}\",
        \"type\": \"release\"
      }"

    echo "Flag '${name}' created (disabled)"
}

# 2. เพิ่ม rollout percentage
increase_rollout() {
    local name=$1
    local percentage=$2

    curl -X PATCH "${UNLEASH_URL}/api/admin/projects/default/features/${name}/environments/production/strategies/STRATEGY_ID" \
      -H "Authorization: ${UNLEASH_API_TOKEN}" \
      -H "Content-Type: application/json" \
      -d "{\"parameters\": {\"rollout\": \"${percentage}\"}}"

    echo "Flag '${name}' rollout increased to ${percentage}%"
}

# 3. Archive flag (หลังจาก 100% rollout เสร็จ)
archive_flag() {
    local name=$1

    curl -X DELETE "${UNLEASH_URL}/api/admin/projects/default/features/${name}/environments/production/strategies/STRATEGY_ID" \
      -H "Authorization: ${UNLEASH_API_TOKEN}"

    curl -X POST "${UNLEASH_URL}/api/admin/archive/features/${name}" \
      -H "Authorization: ${UNLEASH_API_TOKEN}"

    echo "Flag '${name}' archived"
}

# Usage
# create_flag "new-payment-flow" "New payment UI with Apple Pay support"
# increase_rollout "new-payment-flow" 10
# increase_rollout "new-payment-flow" 50
# increase_rollout "new-payment-flow" 100
# archive_flag "new-payment-flow"
```

---

## 🧪 Testing

```typescript
// tests/feature-flags.test.ts
import { isFeatureEnabled } from '../src/lib/feature-flags';

describe('Feature Flags', () => {
  it('should return false for disabled flag', async () => {
    const enabled = await isFeatureEnabled('disabled-feature', 'user123');
    expect(enabled).toBe(false);
  });

  it('should be deterministic for same userId', async () => {
    const result1 = await isFeatureEnabled('gradual-rollout-flag', 'user_12345');
    const result2 = await isFeatureEnabled('gradual-rollout-flag', 'user_12345');
    expect(result1).toBe(result2);
  });

  it('should respect kill switch', async () => {
    // Enable maintenance mode
    process.env.MAINTENANCE_MODE = 'true';
    const enabled = await isFeatureEnabled('maintenance-mode', 'any_user');
    expect(enabled).toBe(true);
  });
});
```

---

## ✅ Checklist

- [ ] Unleash server ทำงานอยู่ (ใน K8s cluster)
- [ ] Unleash Proxy ตั้งค่าสำหรับ client-side SDK
- [ ] Node.js SDK initialize ถูกต้อง
- [ ] Next.js server component ใช้ server-side flag ได้
- [ ] Client-side hook ใช้งานได้ใน React
- [ ] Redis caching ลด latency ของ flag evaluation
- [ ] Kill switch ทดสอบแล้ว (maintenance mode)
- [ ] Gradual rollout ทดสอบกับ user IDs จริง
- [ ] Variant flags ทดสอบแล้ว
- [ ] Flag archive workflow เข้าใจและทำตาม

---

## 🔗 References

- [Unleash Documentation](https://docs.getunleash.io/)
- [Unleash Node.js SDK](https://docs.getunleash.io/reference/sdks/node)
- [Feature Flags Best Practices](https://martinfowler.com/articles/feature-toggles.html)

---
*Part 082 | Road to 1,000,000 Users/Day | chuaikan.com*
