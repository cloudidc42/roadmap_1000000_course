# Part 020: Testing (Unit, Integration, E2E)
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Beginner
> **Steps:** 191-200
> **เวลาโดยประมาณ:** 12 ชั่วโมง
> **Prerequisites:** Part 019 (Database Migration & Schema Management)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

1. Testing pyramid: unit (70%) + integration (20%) + E2E (10%)
2. Vitest setup สำหรับ Node.js 22 และ Next.js 15
3. Unit testing: service functions, utility functions, API handlers
4. Integration testing: database tests กับ real PostgreSQL (testcontainers)
5. API integration tests ด้วย supertest
6. Mocking ด้วย vi.mock (database, external APIs, Redis)
7. E2E testing ด้วย Playwright บน Next.js
8. Test database seeding และ cleanup
9. Code coverage targets (80% minimum)
10. GitHub Actions CI integration
11. Testing SOS alert flow (critical path)
12. Performance testing ใน CI (regression detection)
13. Test data factories ด้วย @faker-js/faker

---

## 📖 ทฤษฎีและแนวคิด

### Testing Pyramid

```
                    ┌─────────┐
                    │   E2E   │  10% - Playwright
                    │  Tests  │  ทดสอบ user journey ทั้งหมด
                    └────┬────┘  ช้า, แพง, แต่ความมั่นใจสูง
                  ───────┴───────
                 │  Integration  │  20% - Vitest + Supertest
                 │    Tests      │  ทดสอบ API, DB, services ร่วมกัน
                 └──────┬────────┘
          ───────────────┴────────────────
         │           Unit Tests           │  70% - Vitest
         │  (functions, components, utils) │  เร็ว, ราคาถูก, isolated
         └─────────────────────────────────┘

Testing Rules:
✅ Unit tests: เร็ว < 100ms ต่อ test
✅ Integration tests: < 5 วินาที ต่อ test
✅ E2E tests: < 30 วินาที ต่อ scenario
✅ Coverage: minimum 80% overall, 90%+ สำหรับ critical paths

Critical Paths ที่ต้องทดสอบ 100%:
- SOS alert creation flow
- Authentication (login, refresh, logout)
- Payment processing
- Emergency notification dispatch
```

---

## ⚙️ Environment Setup

### Step 191: Vitest Setup

```bash
# ติดตั้ง testing dependencies
npm install -D vitest @vitest/coverage-v8 @vitest/ui
npm install -D @testing-library/react @testing-library/user-event @testing-library/jest-dom
npm install -D supertest @types/supertest
npm install -D testcontainers  # Real PostgreSQL for integration tests
npm install -D @faker-js/faker
npm install -D playwright @playwright/test

# Install Playwright browsers
npx playwright install --with-deps chromium firefox
```

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import tsconfigPaths from 'vite-tsconfig-paths';

export default defineConfig({
  plugins: [react(), tsconfigPaths()],
  
  test: {
    // Environment
    environment: 'node',
    
    // Global test setup
    globalSetup: './tests/global-setup.ts',
    setupFiles: ['./tests/setup.ts'],
    
    // Test file patterns
    include: ['**/__tests__/**/*.{test,spec}.{ts,tsx}', '**/*.{test,spec}.{ts,tsx}'],
    exclude: ['**/node_modules/**', '**/e2e/**', '**/*.playwright.ts'],
    
    // Coverage
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html', 'lcov'],
      reportsDirectory: './coverage',
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 80,
        statements: 80,
      },
      include: ['src/**/*.{ts,tsx}'],
      exclude: [
        'src/**/*.d.ts',
        'src/**/*.stories.tsx',
        'src/app/**/page.tsx',  // Page components ทดสอบด้วย E2E
        'src/app/**/layout.tsx',
      ],
    },
    
    // Timeouts
    testTimeout: 30000,
    hookTimeout: 60000,
    
    // Reporters
    reporters: process.env.CI ? ['verbose', 'junit'] : ['verbose'],
    outputFile: {
      junit: './test-results/junit.xml',
    },
    
    // Pool settings
    pool: 'threads',
    poolOptions: {
      threads: {
        singleThread: false,
        maxThreads: 4,
      },
    },
  },
});
```

```typescript
// vitest.config.integration.ts
// แยก config สำหรับ integration tests ที่ต้องการ DB
import { defineConfig } from 'vitest/config';
import tsconfigPaths from 'vite-tsconfig-paths';

export default defineConfig({
  plugins: [tsconfigPaths()],
  
  test: {
    environment: 'node',
    include: ['**/__tests__/integration/**/*.test.ts'],
    globalSetup: './tests/integration-global-setup.ts',
    setupFiles: ['./tests/integration-setup.ts'],
    testTimeout: 60000,
    hookTimeout: 120000,
    
    // Integration tests รันแบบ sequential (DB)
    poolOptions: {
      threads: {
        singleThread: true,
      },
    },
  },
});
```

```typescript
// tests/setup.ts
import '@testing-library/jest-dom';
import { vi, afterEach } from 'vitest';

// Auto-reset mocks after each test
afterEach(() => {
  vi.clearAllMocks();
  vi.restoreAllMocks();
});

// Mock environment variables
process.env.NODE_ENV = 'test';
process.env.DATABASE_URL = 'postgresql://test:test@localhost:5432/chuaikan_test';
process.env.REDIS_URL = 'redis://localhost:6379';
process.env.JWT_ACCESS_SECRET = 'test-access-secret-minimum-32-characters';
process.env.JWT_REFRESH_SECRET = 'test-refresh-secret-minimum-32-characters';
```

---

## 🛠️ Step-by-Step Implementation

### Step 192: Test Data Factories

```typescript
// tests/factories/user.factory.ts
import { faker } from '@faker-js/faker/locale/th';
import { PrismaClient, UserRole } from '@prisma/client';
import bcrypt from 'bcryptjs';

export interface CreateUserOptions {
  role?: UserRole;
  isActive?: boolean;
  email?: string;
  phone?: string;
  failedLoginAttempts?: number;
  lockedUntil?: Date | null;
  isTwoFactorEnabled?: boolean;
}

export function buildUserData(options: CreateUserOptions = {}) {
  return {
    email: options.email ?? faker.internet.email(),
    name: faker.person.fullName(),
    phone: options.phone ?? faker.phone.number('+669########'),
    role: options.role ?? 'USER' as UserRole,
    isActive: options.isActive ?? true,
    failedLoginAttempts: options.failedLoginAttempts ?? 0,
    lockedUntil: options.lockedUntil ?? null,
    isTwoFactorEnabled: options.isTwoFactorEnabled ?? false,
  };
}

export async function createUser(db: PrismaClient, options: CreateUserOptions = {}) {
  const password = 'Test@123456';
  const passwordHash = await bcrypt.hash(password, 10);
  
  return db.user.create({
    data: {
      ...buildUserData(options),
      passwordHash,
    },
  });
}

export async function createAdmin(db: PrismaClient) {
  return createUser(db, { role: 'ADMIN' });
}

export async function createSosResponder(db: PrismaClient) {
  return createUser(db, { role: 'SOS_RESPONDER' });
}
```

```typescript
// tests/factories/sos-alert.factory.ts
import { faker } from '@faker-js/faker/locale/th';
import { PrismaClient, AlertType, AlertSeverity, AlertStatus } from '@prisma/client';

export interface CreateSosAlertOptions {
  type?: AlertType;
  severity?: AlertSeverity;
  status?: AlertStatus;
  reportedById: string;
  assignedToId?: string;
}

export function buildSosAlertData(options: CreateSosAlertOptions) {
  return {
    type: options.type ?? 'MEDICAL' as AlertType,
    severity: options.severity ?? 'HIGH' as AlertSeverity,
    status: options.status ?? 'PENDING' as AlertStatus,
    description: faker.lorem.paragraph(),
    latitude: faker.location.latitude({ min: 13.0, max: 14.5 }), // Bangkok area
    longitude: faker.location.longitude({ min: 100.3, max: 101.0 }),
    location: faker.location.streetAddress(),
    reportedById: options.reportedById,
    assignedToId: options.assignedToId,
  };
}

export async function createSosAlert(db: PrismaClient, options: CreateSosAlertOptions) {
  return db.sosAlert.create({
    data: buildSosAlertData(options),
  });
}
```

### Step 193: Unit Testing

```typescript
// src/lib/auth/__tests__/password-policy.test.ts
import { describe, it, expect } from 'vitest';
import { validatePassword } from '@/lib/auth/password-policy';

describe('validatePassword', () => {
  describe('valid passwords', () => {
    it('should accept strong password', () => {
      const result = validatePassword('Chuaikan@2024!');
      expect(result.isValid).toBe(true);
      expect(result.errors).toHaveLength(0);
      expect(result.strength).toBe('very_strong');
    });

    it('should accept minimum valid password', () => {
      const result = validatePassword('Test@123');
      expect(result.isValid).toBe(true);
    });
  });

  describe('invalid passwords', () => {
    it('should reject password too short', () => {
      const result = validatePassword('Ab1!');
      expect(result.isValid).toBe(false);
      expect(result.errors).toContain(
        expect.stringContaining('ความยาวอย่างน้อย 8 ตัวอักษร')
      );
    });

    it('should reject password without uppercase', () => {
      const result = validatePassword('chuaikan@2024!');
      expect(result.isValid).toBe(false);
      expect(result.errors).toContain(
        expect.stringContaining('ตัวอักษรพิมพ์ใหญ่')
      );
    });

    it('should reject password without numbers', () => {
      const result = validatePassword('Chuaikan@Secure!');
      expect(result.isValid).toBe(false);
      expect(result.errors).toContain(expect.stringContaining('ตัวเลข'));
    });

    it('should reject common passwords', () => {
      const result = validatePassword('Password123!');
      expect(result.isValid).toBe(false);
    });
  });

  describe('password strength', () => {
    it('should rate short basic password as weak', () => {
      const result = validatePassword('abc');
      expect(result.strength).toBe('weak');
    });

    it('should rate long complex password as very_strong', () => {
      const result = validatePassword('ChuaiKan@Emergency2024!Response');
      expect(result.strength).toBe('very_strong');
    });
  });
});
```

```typescript
// src/lib/api/__tests__/pagination.test.ts
import { describe, it, expect } from 'vitest';
import { encodeCursor, decodeCursor } from '@/lib/api/pagination';

describe('Cursor Pagination', () => {
  it('should encode and decode cursor correctly', () => {
    const id = 'user-123-abc';
    const createdAt = new Date('2024-01-15T10:30:00Z');
    
    const cursor = encodeCursor(id, createdAt);
    expect(cursor).toBeDefined();
    expect(typeof cursor).toBe('string');
    
    const decoded = decodeCursor(cursor);
    expect(decoded.id).toBe(id);
    expect(decoded.createdAt.toISOString()).toBe(createdAt.toISOString());
  });

  it('should produce URL-safe cursor', () => {
    const cursor = encodeCursor('test-id', new Date());
    // base64url characters: A-Z, a-z, 0-9, -, _
    expect(cursor).toMatch(/^[A-Za-z0-9_-]+$/);
  });

  it('should throw on invalid cursor', () => {
    expect(() => decodeCursor('invalid-cursor')).toThrow();
  });
});
```

```typescript
// src/lib/auth/__tests__/rbac.test.ts
import { describe, it, expect } from 'vitest';
import { hasPermission } from '@/lib/auth/rbac';
import type { Permission, UserRole } from '@/lib/auth/rbac';

describe('RBAC - hasPermission', () => {
  describe('ADMIN role', () => {
    it('should have all permissions', () => {
      expect(hasPermission('ADMIN', ['*'], 'sos:create')).toBe(true);
      expect(hasPermission('ADMIN', ['*'], 'user:manage')).toBe(true);
      expect(hasPermission('ADMIN', ['*'], 'system:configure')).toBe(true);
    });
  });

  describe('USER role', () => {
    const userPermissions: Permission[] = ['sos:create', 'sos:view:own', 'profile:manage'];
    
    it('should have user-level permissions', () => {
      expect(hasPermission('USER', userPermissions, 'sos:create')).toBe(true);
      expect(hasPermission('USER', userPermissions, 'profile:manage')).toBe(true);
    });

    it('should not have admin permissions', () => {
      expect(hasPermission('USER', userPermissions, 'user:manage')).toBe(false);
      expect(hasPermission('USER', userPermissions, 'sos:manage')).toBe(false);
      expect(hasPermission('USER', userPermissions, 'system:configure')).toBe(false);
    });
  });

  describe('SOS_RESPONDER role', () => {
    const responderPermissions: Permission[] = [
      'sos:respond', 'sos:update', 'sos:view',
      'location:track', 'notification:receive',
    ];

    it('should respond to SOS alerts', () => {
      expect(hasPermission('SOS_RESPONDER', responderPermissions, 'sos:respond')).toBe(true);
      expect(hasPermission('SOS_RESPONDER', responderPermissions, 'sos:update')).toBe(true);
    });

    it('should not manage users', () => {
      expect(hasPermission('SOS_RESPONDER', responderPermissions, 'user:manage')).toBe(false);
    });
  });
});
```

### Step 194: Mocking Database and External Services

```typescript
// src/lib/sos/__tests__/sos-service.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { createSosAlert, assignAlert } from '@/lib/sos/sos-service';

// Mock Prisma client
vi.mock('@/lib/db', () => ({
  db: {
    sosAlert: {
      create: vi.fn(),
      update: vi.fn(),
      findUnique: vi.fn(),
    },
    user: {
      findFirst: vi.fn(),
    },
  },
}));

// Mock Redis
vi.mock('ioredis', () => ({
  default: vi.fn(() => ({
    get: vi.fn(),
    set: vi.fn(),
    setex: vi.fn(),
    del: vi.fn(),
    publish: vi.fn(),
  })),
}));

// Mock notification service
vi.mock('@/lib/notifications/push', () => ({
  sendPushNotification: vi.fn().mockResolvedValue({ success: true }),
}));

import { db } from '@/lib/db';
import { sendPushNotification } from '@/lib/notifications/push';

describe('SOS Service', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  describe('createSosAlert', () => {
    it('should create SOS alert and notify nearby responders', async () => {
      const mockAlert = {
        id: 'alert-123',
        type: 'MEDICAL',
        severity: 'HIGH',
        status: 'PENDING',
        reportedById: 'user-456',
        latitude: 13.7563,
        longitude: 100.5018,
        createdAt: new Date(),
        updatedAt: new Date(),
      };

      const mockResponder = {
        id: 'responder-789',
        name: 'Responder 1',
        role: 'SOS_RESPONDER',
      };

      vi.mocked(db.sosAlert.create).mockResolvedValue(mockAlert as any);
      vi.mocked(db.user.findFirst).mockResolvedValue(mockResponder as any);

      const result = await createSosAlert({
        type: 'MEDICAL',
        severity: 'HIGH',
        description: 'Emergency situation',
        latitude: 13.7563,
        longitude: 100.5018,
        location: 'Bangkok',
        reportedById: 'user-456',
      });

      expect(result).toEqual(mockAlert);
      expect(db.sosAlert.create).toHaveBeenCalledOnce();
      expect(sendPushNotification).toHaveBeenCalled();
    });

    it('should throw when required fields missing', async () => {
      await expect(
        createSosAlert({
          type: 'MEDICAL',
          severity: 'HIGH',
          description: '',  // Empty description
          latitude: 13.7563,
          longitude: 100.5018,
          location: 'Bangkok',
          reportedById: 'user-456',
        })
      ).rejects.toThrow('Description is required');
    });
  });

  describe('assignAlert', () => {
    it('should assign alert to responder', async () => {
      const mockAlert = {
        id: 'alert-123',
        status: 'PENDING',
        assignedToId: null,
      };

      const updatedAlert = {
        ...mockAlert,
        status: 'ASSIGNED',
        assignedToId: 'responder-789',
      };

      vi.mocked(db.sosAlert.findUnique).mockResolvedValue(mockAlert as any);
      vi.mocked(db.sosAlert.update).mockResolvedValue(updatedAlert as any);

      const result = await assignAlert('alert-123', 'responder-789');

      expect(result.status).toBe('ASSIGNED');
      expect(result.assignedToId).toBe('responder-789');
    });

    it('should throw when alert already assigned', async () => {
      const mockAlert = {
        id: 'alert-123',
        status: 'ASSIGNED',
        assignedToId: 'another-responder',
      };

      vi.mocked(db.sosAlert.findUnique).mockResolvedValue(mockAlert as any);

      await expect(
        assignAlert('alert-123', 'responder-789')
      ).rejects.toThrow('Alert already assigned');
    });
  });
});
```

### Step 195: Integration Testing กับ Real PostgreSQL

```typescript
// tests/integration-global-setup.ts
import { GenericContainer, PostgreSqlContainer, Wait } from 'testcontainers';
import type { StartedPostgreSqlContainer } from 'testcontainers';
import { execSync } from 'child_process';

let postgresContainer: StartedPostgreSqlContainer;

export async function setup() {
  console.log('Starting PostgreSQL container...');
  
  postgresContainer = await new PostgreSqlContainer('postgres:17-alpine')
    .withDatabase('chuaikan_test')
    .withUsername('test')
    .withPassword('test')
    .withWaitStrategy(Wait.forLogMessage('database system is ready to accept connections'))
    .start();

  const connectionString = postgresContainer.getConnectionUri();
  process.env.DATABASE_URL = connectionString;
  
  // Run migrations on test database
  console.log('Running migrations...');
  execSync('npx prisma migrate deploy', {
    env: { ...process.env, DATABASE_URL: connectionString },
    stdio: 'inherit',
  });
  
  console.log('Test database ready!');
  
  // Store container reference for teardown
  (global as any).__POSTGRES_CONTAINER__ = postgresContainer;
}

export async function teardown() {
  console.log('Stopping PostgreSQL container...');
  const container = (global as any).__POSTGRES_CONTAINER__;
  if (container) {
    await container.stop();
  }
}
```

```typescript
// tests/integration-setup.ts
import { PrismaClient } from '@prisma/client';
import { beforeEach, afterAll } from 'vitest';

export const testDb = new PrismaClient({
  datasources: {
    db: { url: process.env.DATABASE_URL },
  },
});

// Clean database before each test
beforeEach(async () => {
  // Delete in correct order to avoid foreign key violations
  await testDb.alertTimeline.deleteMany();
  await testDb.sosAlert.deleteMany();
  await testDb.refreshToken.deleteMany();
  await testDb.account.deleteMany();
  await testDb.session.deleteMany();
  await testDb.user.deleteMany();
});

afterAll(async () => {
  await testDb.$disconnect();
});
```

```typescript
// __tests__/integration/sos-alert-api.test.ts
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import request from 'supertest';
import { createServer } from '@/server';
import { testDb } from '../../tests/integration-setup';
import { createUser, createAdmin } from '../../tests/factories/user.factory';
import { signAccessToken, getPermissionsForRole } from '@/lib/auth/jwt';

describe('SOS Alert API Integration Tests', () => {
  let server: ReturnType<typeof createServer>;
  let userToken: string;
  let adminToken: string;
  let userId: string;
  let adminId: string;

  beforeAll(async () => {
    server = createServer();
    
    // Create test users
    const user = await createUser(testDb);
    const admin = await createAdmin(testDb);
    userId = user.id;
    adminId = admin.id;
    
    // Generate tokens
    userToken = await signAccessToken({
      sub: user.id,
      email: user.email,
      role: user.role,
      permissions: getPermissionsForRole(user.role),
      sessionId: 'test-session',
    });
    
    adminToken = await signAccessToken({
      sub: admin.id,
      email: admin.email,
      role: admin.role,
      permissions: getPermissionsForRole(admin.role),
      sessionId: 'test-session-admin',
    });
  });

  afterAll(async () => {
    await server.close();
  });

  describe('POST /api/v1/sos-alerts', () => {
    it('should create SOS alert successfully', async () => {
      const alertData = {
        type: 'MEDICAL',
        severity: 'HIGH',
        description: 'Emergency: person collapsed needs immediate help',
        latitude: 13.7563,
        longitude: 100.5018,
        location: 'Siam, Bangkok',
      };

      const response = await request(server)
        .post('/api/v1/sos-alerts')
        .set('Authorization', `Bearer ${userToken}`)
        .set('Idempotency-Key', crypto.randomUUID())
        .send(alertData);

      expect(response.status).toBe(201);
      expect(response.body.success).toBe(true);
      expect(response.body.data.id).toBeDefined();
      expect(response.body.data.type).toBe('MEDICAL');
      expect(response.body.data.status).toBe('PENDING');
      expect(response.body.data.reportedById).toBe(userId);
      
      // Verify in database
      const alertInDb = await testDb.sosAlert.findUnique({
        where: { id: response.body.data.id },
      });
      expect(alertInDb).not.toBeNull();
      expect(alertInDb?.type).toBe('MEDICAL');
    });

    it('should return 422 for invalid data', async () => {
      const response = await request(server)
        .post('/api/v1/sos-alerts')
        .set('Authorization', `Bearer ${userToken}`)
        .send({
          type: 'INVALID_TYPE',  // ไม่ valid
          severity: 'HIGH',
          description: '',  // ว่างเปล่า
          latitude: 200,  // เกิน 90
          longitude: 100,
          location: 'Bangkok',
        });

      expect(response.status).toBe(422);
      expect(response.body.success).toBe(false);
      expect(response.body.error.code).toBe('VALIDATION_ERROR');
      expect(response.body.error.details).toBeDefined();
    });

    it('should prevent duplicate with idempotency key', async () => {
      const idempotencyKey = crypto.randomUUID();
      const alertData = {
        type: 'FIRE',
        severity: 'CRITICAL',
        description: 'Building fire in central Bangkok area',
        latitude: 13.7466,
        longitude: 100.5347,
        location: 'Central Bangkok',
      };

      // First request
      const response1 = await request(server)
        .post('/api/v1/sos-alerts')
        .set('Authorization', `Bearer ${userToken}`)
        .set('Idempotency-Key', idempotencyKey)
        .send(alertData);

      // Second request (duplicate)
      const response2 = await request(server)
        .post('/api/v1/sos-alerts')
        .set('Authorization', `Bearer ${userToken}`)
        .set('Idempotency-Key', idempotencyKey)
        .send(alertData);

      expect(response1.status).toBe(201);
      expect(response2.status).toBe(201);
      expect(response2.headers['idempotency-replayed']).toBe('true');
      expect(response1.body.data.id).toBe(response2.body.data.id);
      
      // Should only be one alert in DB
      const alerts = await testDb.sosAlert.findMany({
        where: { reportedById: userId, type: 'FIRE' },
      });
      expect(alerts).toHaveLength(1);
    });
  });

  describe('GET /api/v1/sos-alerts', () => {
    it('should return user own alerts only (regular user)', async () => {
      // Create alerts for userId
      await testDb.sosAlert.createMany({
        data: [
          {
            type: 'MEDICAL',
            severity: 'HIGH',
            status: 'PENDING',
            description: 'Test alert 1',
            latitude: 13.7563,
            longitude: 100.5018,
            location: 'Bangkok',
            reportedById: userId,
          },
          {
            type: 'ACCIDENT',
            severity: 'MEDIUM',
            status: 'PENDING',
            description: 'Test alert 2',
            latitude: 13.7563,
            longitude: 100.5018,
            location: 'Bangkok',
            reportedById: adminId,  // Different user
          },
        ],
      });

      const response = await request(server)
        .get('/api/v1/sos-alerts')
        .set('Authorization', `Bearer ${userToken}`);

      expect(response.status).toBe(200);
      
      // User should only see their own alerts
      response.body.data.forEach((alert: any) => {
        expect(alert.reportedById).toBe(userId);
      });
    });

    it('should support cursor pagination', async () => {
      // Create 25 alerts
      for (let i = 0; i < 25; i++) {
        await testDb.sosAlert.create({
          data: {
            type: 'OTHER',
            severity: 'LOW',
            status: 'PENDING',
            description: `Test alert ${i + 1}`,
            latitude: 13.7563 + i * 0.001,
            longitude: 100.5018,
            location: 'Bangkok',
            reportedById: userId,
          },
        });
      }

      // First page
      const page1 = await request(server)
        .get('/api/v1/sos-alerts')
        .set('Authorization', `Bearer ${userToken}`)
        .query({ limit: 10 });

      expect(page1.status).toBe(200);
      expect(page1.body.data).toHaveLength(10);
      expect(page1.body.meta.pagination.hasMore).toBe(true);
      expect(page1.body.meta.pagination.nextCursor).toBeDefined();

      // Second page
      const page2 = await request(server)
        .get('/api/v1/sos-alerts')
        .set('Authorization', `Bearer ${userToken}`)
        .query({
          limit: 10,
          cursor: page1.body.meta.pagination.nextCursor,
        });

      expect(page2.status).toBe(200);
      expect(page2.body.data).toHaveLength(10);
      
      // No overlap between pages
      const page1Ids = page1.body.data.map((a: any) => a.id);
      const page2Ids = page2.body.data.map((a: any) => a.id);
      const overlap = page1Ids.filter((id: string) => page2Ids.includes(id));
      expect(overlap).toHaveLength(0);
    });
  });
});
```

### Step 196: Testing SOS Alert Critical Flow

```typescript
// __tests__/integration/sos-critical-path.test.ts
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import request from 'supertest';
import { testDb } from '../../tests/integration-setup';
import { createUser, createSosResponder } from '../../tests/factories/user.factory';
import { signAccessToken, getPermissionsForRole } from '@/lib/auth/jwt';

describe('SOS Alert Critical Path - Full Flow', () => {
  let server: any;
  let userToken: string;
  let responderToken: string;
  let userId: string;
  let responderId: string;
  let alertId: string;

  beforeAll(async () => {
    server = createServer();
    
    const user = await createUser(testDb);
    const responder = await createSosResponder(testDb);
    userId = user.id;
    responderId = responder.id;
    
    userToken = await signAccessToken({
      sub: user.id, email: user.email, role: user.role,
      permissions: getPermissionsForRole(user.role), sessionId: 'u-session',
    });
    
    responderToken = await signAccessToken({
      sub: responder.id, email: responder.email, role: responder.role,
      permissions: getPermissionsForRole(responder.role), sessionId: 'r-session',
    });
  });

  it('STEP 1: User creates SOS alert', async () => {
    const response = await request(server)
      .post('/api/v1/sos-alerts')
      .set('Authorization', `Bearer ${userToken}`)
      .set('Idempotency-Key', crypto.randomUUID())
      .send({
        type: 'MEDICAL',
        severity: 'CRITICAL',
        description: 'ผู้ป่วยหัวใจล้มเหลวต้องการความช่วยเหลือทันที',
        latitude: 13.7563,
        longitude: 100.5018,
        location: 'สยามสแควร์, กรุงเทพมหานคร',
      });

    expect(response.status).toBe(201);
    expect(response.body.data.status).toBe('PENDING');
    alertId = response.body.data.id;
    
    // Verify timeline entry created
    const timeline = await testDb.alertTimeline.findMany({
      where: { alertId },
    });
    expect(timeline).toHaveLength(1);
    expect(timeline[0].action).toContain('created');
  });

  it('STEP 2: Responder views pending alerts', async () => {
    const response = await request(server)
      .get('/api/v1/sos-alerts')
      .set('Authorization', `Bearer ${responderToken}`)
      .query({ status: 'PENDING', severity: 'CRITICAL' });

    expect(response.status).toBe(200);
    const alertInList = response.body.data.find((a: any) => a.id === alertId);
    expect(alertInList).toBeDefined();
  });

  it('STEP 3: Responder assigns themselves', async () => {
    const response = await request(server)
      .patch(`/api/v1/sos-alerts/${alertId}`)
      .set('Authorization', `Bearer ${responderToken}`)
      .send({
        status: 'ASSIGNED',
        assignedTo: responderId,
      });

    expect(response.status).toBe(200);
    expect(response.body.data.status).toBe('ASSIGNED');
    expect(response.body.data.assignedToId).toBe(responderId);
  });

  it('STEP 4: Responder updates status to IN_PROGRESS', async () => {
    const response = await request(server)
      .patch(`/api/v1/sos-alerts/${alertId}`)
      .set('Authorization', `Bearer ${responderToken}`)
      .send({
        status: 'IN_PROGRESS',
        notes: 'กำลังเดินทางถึงที่เกิดเหตุ ETA 5 นาที',
      });

    expect(response.status).toBe(200);
    expect(response.body.data.status).toBe('IN_PROGRESS');
  });

  it('STEP 5: Responder resolves the alert', async () => {
    const response = await request(server)
      .patch(`/api/v1/sos-alerts/${alertId}`)
      .set('Authorization', `Bearer ${responderToken}`)
      .send({
        status: 'RESOLVED',
        notes: 'ผู้ป่วยได้รับการช่วยเหลือแล้ว ส่งโรงพยาบาลเรียบร้อย',
        resolvedAt: new Date().toISOString(),
      });

    expect(response.status).toBe(200);
    expect(response.body.data.status).toBe('RESOLVED');
    expect(response.body.data.resolvedAt).toBeDefined();
  });

  it('STEP 6: Cannot update resolved alert', async () => {
    const response = await request(server)
      .patch(`/api/v1/sos-alerts/${alertId}`)
      .set('Authorization', `Bearer ${responderToken}`)
      .send({ notes: 'Trying to update resolved alert' });

    expect(response.status).toBe(409);
    expect(response.body.error.code).toBe('SOS_ALREADY_RESOLVED');
  });

  it('STEP 7: Full timeline should have 4 entries', async () => {
    const timeline = await testDb.alertTimeline.findMany({
      where: { alertId },
      orderBy: { createdAt: 'asc' },
    });
    
    expect(timeline).toHaveLength(4); // created, assigned, in_progress, resolved
  });
});
```

### Step 197: E2E Testing ด้วย Playwright

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  
  reporter: [
    ['html', { outputFolder: 'playwright-report' }],
    ['junit', { outputFile: 'test-results/playwright-junit.xml' }],
  ],
  
  use: {
    baseURL: process.env.PLAYWRIGHT_BASE_URL || 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'on-first-retry',
  },

  projects: [
    { name: 'setup', testMatch: /.*\.setup\.ts/ },
    
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
      dependencies: ['setup'],
    },
    
    {
      name: 'Mobile Safari',
      use: { ...devices['iPhone 13'] },
      dependencies: ['setup'],
    },
  ],

  webServer: {
    command: 'npm run build && npm run start',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
    timeout: 120000,
  },
});
```

```typescript
// e2e/auth.setup.ts
import { test as setup, expect } from '@playwright/test';
import path from 'path';

const authFile = path.join(__dirname, '../playwright/.auth/user.json');

setup('authenticate', async ({ page }) => {
  await page.goto('/auth/login');
  
  await page.getByLabel('อีเมล').fill('testuser1@chuaikan.com');
  await page.getByLabel('รหัสผ่าน').fill('User@123456');
  await page.getByRole('button', { name: 'เข้าสู่ระบบ' }).click();
  
  // Wait for redirect to dashboard
  await page.waitForURL('/dashboard');
  await expect(page.getByRole('heading', { name: 'แดชบอร์ด' })).toBeVisible();
  
  await page.context().storageState({ path: authFile });
});
```

```typescript
// e2e/sos-alert.spec.ts
import { test, expect } from '@playwright/test';

test.use({ storageState: 'playwright/.auth/user.json' });

test.describe('SOS Alert Flow', () => {
  test('user can create SOS alert', async ({ page }) => {
    await page.goto('/dashboard');
    
    // Click SOS button
    await page.getByRole('button', { name: 'แจ้งเหตุด่วน' }).click();
    
    // Fill SOS form
    await page.getByLabel('ประเภทเหตุการณ์').selectOption('MEDICAL');
    await page.getByLabel('ระดับความรุนแรง').selectOption('HIGH');
    await page.getByLabel('รายละเอียด').fill(
      'ผู้ป่วยหมดสติต้องการความช่วยเหลือด่วน'
    );
    
    // Location should be auto-filled from GPS
    await expect(page.getByLabel('ตำแหน่ง')).not.toBeEmpty();
    
    // Submit
    await page.getByRole('button', { name: 'ส่งการแจ้งเหตุ' }).click();
    
    // Should show success
    await expect(page.getByText('ส่งการแจ้งเหตุเรียบร้อยแล้ว')).toBeVisible();
    
    // Should redirect to alert detail page
    await page.waitForURL(/\/sos-alerts\/.+/);
  });

  test('should show validation errors', async ({ page }) => {
    await page.goto('/sos-alerts/new');
    
    // Submit without filling required fields
    await page.getByRole('button', { name: 'ส่งการแจ้งเหตุ' }).click();
    
    await expect(page.getByText('กรุณาเลือกประเภทเหตุการณ์')).toBeVisible();
    await expect(page.getByText('กรุณากรอกรายละเอียด')).toBeVisible();
  });

  test('should show real-time alert status updates', async ({ page, context }) => {
    // Create alert first via API
    const alertId = 'test-alert-id';
    
    await page.goto(`/sos-alerts/${alertId}`);
    
    // Status should be PENDING initially
    await expect(page.getByText('รอดำเนินการ')).toBeVisible();
    
    // Simulate status update via WebSocket/SSE
    // In a real test, this would be done by a second browser session
    
    // Wait for real-time update
    await expect(page.getByText('กำลังดำเนินการ')).toBeVisible({ timeout: 10000 });
  });
});
```

### Step 198: Performance Testing

```typescript
// __tests__/performance/api-performance.test.ts
import { describe, it, expect } from 'vitest';
import request from 'supertest';
import { createServer } from '@/server';

const PERFORMANCE_THRESHOLDS = {
  p50: 100,   // 50th percentile: < 100ms
  p95: 500,   // 95th percentile: < 500ms
  p99: 1000,  // 99th percentile: < 1000ms
};

describe('API Performance', () => {
  let server: any;
  let authToken: string;

  it('GET /api/v1/sos-alerts should respond in < 100ms (p95)', async () => {
    const durations: number[] = [];
    const requests = 50; // 50 requests

    for (let i = 0; i < requests; i++) {
      const start = Date.now();
      
      await request(server)
        .get('/api/v1/sos-alerts')
        .set('Authorization', `Bearer ${authToken}`)
        .query({ limit: 20 });
      
      durations.push(Date.now() - start);
    }

    durations.sort((a, b) => a - b);
    
    const p50 = durations[Math.floor(requests * 0.5)];
    const p95 = durations[Math.floor(requests * 0.95)];
    const p99 = durations[Math.floor(requests * 0.99)];

    console.log(`Performance Results:
      P50: ${p50}ms (threshold: ${PERFORMANCE_THRESHOLDS.p50}ms)
      P95: ${p95}ms (threshold: ${PERFORMANCE_THRESHOLDS.p95}ms)
      P99: ${p99}ms (threshold: ${PERFORMANCE_THRESHOLDS.p99}ms)
    `);

    expect(p95).toBeLessThan(PERFORMANCE_THRESHOLDS.p95);
    expect(p99).toBeLessThan(PERFORMANCE_THRESHOLDS.p99);
  });
});
```

### Step 199: GitHub Actions CI Integration

```yaml
# .github/workflows/test.yml
name: Test Suite

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-24.04
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
      
      - run: npm ci
      
      - name: Run Unit Tests
        run: npx vitest run --coverage
        env:
          NODE_ENV: test
          DATABASE_URL: postgresql://test:test@localhost:5432/test
          JWT_ACCESS_SECRET: test-secret-minimum-32-characters-long
          JWT_REFRESH_SECRET: test-refresh-minimum-32-characters-long
      
      - name: Upload Coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
          fail_ci_if_error: true
          minimum_coverage: 80

  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-24.04
    
    services:
      postgres:
        image: postgres:17-alpine
        env:
          POSTGRES_DB: chuaikan_test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
      
      - run: npm ci
      
      - name: Run Database Migrations
        run: npx prisma migrate deploy
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/chuaikan_test
      
      - name: Run Integration Tests
        run: npx vitest run --config vitest.config.integration.ts
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/chuaikan_test
          REDIS_URL: redis://localhost:6379
          JWT_ACCESS_SECRET: test-secret-minimum-32-characters-long
          JWT_REFRESH_SECRET: test-refresh-minimum-32-characters-long

  e2e-tests:
    name: E2E Tests (Playwright)
    runs-on: ubuntu-24.04
    needs: [unit-tests, integration-tests]
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
      
      - run: npm ci
      
      - name: Install Playwright Browsers
        run: npx playwright install --with-deps chromium
      
      - name: Build Application
        run: npm run build
        env:
          DATABASE_URL: ${{ secrets.TEST_DATABASE_URL }}
      
      - name: Run E2E Tests
        run: npx playwright test
        env:
          PLAYWRIGHT_BASE_URL: http://localhost:3000
          DATABASE_URL: ${{ secrets.TEST_DATABASE_URL }}
      
      - uses: actions/upload-artifact@v3
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30
```

### Step 200: Code Coverage Report

```typescript
// scripts/check-coverage.ts
import { readFileSync } from 'fs';

interface CoverageThreshold {
  lines: number;
  functions: number;
  branches: number;
  statements: number;
}

const THRESHOLDS: CoverageThreshold = {
  lines: 80,
  functions: 80,
  branches: 75,
  statements: 80,
};

// Critical modules ต้องการ coverage สูงกว่า
const CRITICAL_THRESHOLDS: Record<string, CoverageThreshold> = {
  'src/lib/auth': { lines: 90, functions: 90, branches: 85, statements: 90 },
  'src/lib/sos': { lines: 95, functions: 95, branches: 90, statements: 95 },
  'src/lib/payments': { lines: 95, functions: 95, branches: 90, statements: 95 },
};

async function checkCoverage() {
  const summaryPath = './coverage/coverage-summary.json';
  
  try {
    const summary = JSON.parse(readFileSync(summaryPath, 'utf-8'));
    const total = summary.total;
    
    let failed = false;
    
    console.log('\n=== Coverage Report ===\n');
    
    for (const [metric, threshold] of Object.entries(THRESHOLDS)) {
      const actual = total[metric as keyof CoverageThreshold].pct;
      const passed = actual >= threshold;
      
      console.log(
        `${passed ? '✓' : '✗'} ${metric}: ${actual.toFixed(1)}% (min: ${threshold}%)`
      );
      
      if (!passed) failed = true;
    }
    
    if (failed) {
      console.error('\n❌ Coverage thresholds not met!');
      process.exit(1);
    } else {
      console.log('\n✓ All coverage thresholds met!');
    }
  } catch (error) {
    console.error('Failed to read coverage summary:', error);
    process.exit(1);
  }
}

checkCoverage();
```

---

## 🔧 Configuration Files

```json
// package.json - test scripts
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "test:integration": "vitest run --config vitest.config.integration.ts",
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui",
    "test:all": "npm run test && npm run test:integration",
    "test:ci": "npm run test:coverage && npm run test:integration"
  }
}
```

---

## 🧪 Running Tests

```bash
# Unit tests
npm run test

# Watch mode (development)
npm run test:watch

# With coverage
npm run test:coverage

# Integration tests only
npm run test:integration

# E2E tests
npm run test:e2e

# E2E with UI
npm run test:e2e:ui

# Run specific test file
npx vitest run src/lib/auth/__tests__/jwt.test.ts

# Run tests matching pattern
npx vitest run --grep "SOS Alert"

# Debug failing test
npx vitest run --reporter=verbose src/lib/sos/__tests__/
```

---

## ❌ Common Errors & Solutions

### Error 1: Testcontainers timeout

```
Error: Container failed to start within 60000ms
```

**แก้ไข:**
```bash
# เพิ่ม timeout
export TESTCONTAINERS_PULL_POLICY=missing
export RYUK_DISABLED=true

# หรือ pull image ก่อน
docker pull postgres:17-alpine
```

### Error 2: Vitest cannot find module

```
Error: Cannot find module '@/lib/db'
```

**แก้ไข:**
```typescript
// vitest.config.ts - ตรวจสอบ plugin
plugins: [tsconfigPaths()],
```

### Error 3: Playwright browser not found

```
Error: browserType.launch: Executable doesn't exist
```

**แก้ไข:**
```bash
npx playwright install chromium
```

---

## ✅ Checklist

### Unit Tests
- [ ] ทุก utility function มี unit tests
- [ ] ทุก service function มี unit tests
- [ ] Mocking: database, Redis, external APIs
- [ ] Edge cases covered (null, empty, invalid input)
- [ ] Error scenarios tested

### Integration Tests
- [ ] API endpoints ทั้งหมดมี integration tests
- [ ] Database operations tested กับ real PostgreSQL
- [ ] Authentication flow tested
- [ ] Pagination tested
- [ ] Idempotency tested
- [ ] Rate limiting tested

### E2E Tests
- [ ] User login/logout flow
- [ ] SOS alert creation flow (critical path)
- [ ] Admin operations
- [ ] Mobile responsive

### Coverage
- [ ] Overall coverage >= 80%
- [ ] Auth module >= 90%
- [ ] SOS critical path >= 95%
- [ ] Coverage report ใน CI

### CI/CD
- [ ] Unit tests รันทุก PR
- [ ] Integration tests รันทุก PR
- [ ] E2E tests รันก่อน merge to main
- [ ] Coverage check ใน CI
- [ ] Test results uploaded as artifacts

---

## 🔗 References

- [Vitest Documentation](https://vitest.dev/)
- [Playwright Documentation](https://playwright.dev/)
- [Testing Library](https://testing-library.com/)
- [Testcontainers for Node.js](https://node.testcontainers.org/)
- [Faker.js](https://fakerjs.dev/)
- [Supertest](https://github.com/ladjs/supertest)

---
*Part 020 | Road to 1,000,000 Users/Day | chuaikan.com*
