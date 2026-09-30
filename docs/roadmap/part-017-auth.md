# Part 017: Authentication & Authorization (OAuth, RBAC)
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Beginner
> **Steps:** 161-170
> **เวลาโดยประมาณ:** 10 ชั่วโมง
> **Prerequisites:** Part 016 (API Design & REST Best Practices)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

1. เปรียบเทียบ JWT vs Session: ข้อดีข้อเสียและเมื่อไหร่ควรเลือกใช้อะไร
2. JWT implementation: access token (15 นาที) + refresh token (30 วัน) pattern
3. Refresh token rotation และการ revoke tokens
4. OAuth 2.0 Social Login: LINE Login (ยอดนิยมในไทย) และ Google
5. NextAuth.js v5 setup กับ custom providers
6. RBAC: roles (admin, moderator, user, sos_responder) และ permissions
7. Row-level security ใน PostgreSQL
8. 2FA ด้วย TOTP (Google Authenticator)
9. Rate limiting สำหรับ auth endpoints (ป้องกัน brute force)
10. Account lockout หลังจาก failed attempts
11. Password policy enforcement
12. Session management และ logout (token blacklisting ด้วย Redis)

---

## 📖 ทฤษฎีและแนวคิด

### JWT vs Session Comparison

```
┌─────────────────────────────────────────────────────────────────┐
│                    JWT vs Session                                │
├──────────────────┬───────────────────────┬──────────────────────┤
│ Feature          │ JWT                   │ Session              │
├──────────────────┼───────────────────────┼──────────────────────┤
│ Storage          │ Client (localStorage  │ Server (Redis/DB)    │
│                  │ or Cookie)            │                      │
├──────────────────┼───────────────────────┼──────────────────────┤
│ Stateless        │ Yes                   │ No                   │
├──────────────────┼───────────────────────┼──────────────────────┤
│ Revocation       │ Hard (need blacklist) │ Easy (delete from DB)│
├──────────────────┼───────────────────────┼──────────────────────┤
│ Scale            │ Easy (no shared state)│ Need sticky session  │
│                  │                       │ or Redis             │
├──────────────────┼───────────────────────┼──────────────────────┤
│ Size             │ ~500 bytes            │ ~32 bytes (ID only)  │
├──────────────────┼───────────────────────┼──────────────────────┤
│ Security         │ Payload exposed       │ Only ID exposed      │
│                  │ (not encrypted)       │                      │
├──────────────────┼───────────────────────┼──────────────────────┤
│ Use Case         │ API, Microservices    │ Web apps with        │
│                  │                       │ server-side rendering│
└──────────────────┴───────────────────────┴──────────────────────┘

สำหรับ chuaikan.com เลือก JWT เพราะ:
✓ Scale ได้ดีกับ multiple servers
✓ ใช้ร่วมกับ mobile app ได้
✓ Microservices รองรับ JWT ได้ง่าย
✓ ใช้ Redis สำหรับ blacklist ได้
```

### JWT Token Pattern

```
Access Token (15 minutes)
┌─────────────────────────────────────────────────────────┐
│ Header    │ Algorithm: HS256                             │
│           │ Type: JWT                                    │
├─────────────────────────────────────────────────────────┤
│ Payload   │ sub: user_id                                 │
│           │ email: user@example.com                     │
│           │ role: user                                   │
│           │ permissions: ["sos:create", "profile:read"] │
│           │ iat: issued at (timestamp)                  │
│           │ exp: expiry (iat + 15 minutes)              │
├─────────────────────────────────────────────────────────┤
│ Signature │ HMAC-SHA256(header.payload, secret)         │
└─────────────────────────────────────────────────────────┘

Refresh Token (30 days)
┌─────────────────────────────────────────────────────────┐
│ - Stored in HttpOnly Cookie                              │
│ - Stored in DB with user_id and device info             │
│ - เมื่อใช้ refresh: ออก token ใหม่ + rotate refresh     │
│ - เมื่อ logout: ลบจาก DB                               │
└─────────────────────────────────────────────────────────┘
```

### RBAC Permission Model

```
Roles & Permissions:
┌──────────────┬────────────────────────────────────────────┐
│ Role         │ Permissions                                 │
├──────────────┼────────────────────────────────────────────┤
│ admin        │ * (all permissions)                        │
│              │ user:manage, system:configure              │
├──────────────┼────────────────────────────────────────────┤
│ moderator    │ sos:manage, user:view, report:manage       │
│              │ location:view, notification:send           │
├──────────────┼────────────────────────────────────────────┤
│ sos_responder│ sos:respond, sos:update, location:track   │
│              │ notification:receive                       │
├──────────────┼────────────────────────────────────────────┤
│ user         │ sos:create, sos:view:own, profile:manage  │
│              │ notification:receive                       │
└──────────────┴────────────────────────────────────────────┘
```

---

## ⚙️ Environment Setup

```bash
# ติดตั้ง NextAuth.js v5 และ dependencies
npm install next-auth@beta
npm install @auth/prisma-adapter
npm install jose  # JWT library
npm install otplib  # TOTP 2FA
npm install qrcode  # QR code for 2FA setup
npm install bcryptjs
npm install @types/bcryptjs -D

# สร้าง environment variables
cat >> .env.local << 'EOF'
# Auth
AUTH_SECRET="your-very-long-random-secret-minimum-32-chars"
AUTH_URL="http://localhost:3000"

# JWT
JWT_ACCESS_SECRET="access-token-secret-min-32-chars"
JWT_REFRESH_SECRET="refresh-token-secret-min-32-chars"

# LINE Login
LINE_CLIENT_ID="your-line-client-id"
LINE_CLIENT_SECRET="your-line-client-secret"

# Google
GOOGLE_CLIENT_ID="your-google-client-id"
GOOGLE_CLIENT_SECRET="your-google-client-secret"
EOF
```

### Database Schema

```sql
-- prisma/schema.prisma additions

model User {
  id                String    @id @default(uuid())
  email             String    @unique
  emailVerified     DateTime?
  phone             String?   @unique
  name              String
  image             String?
  passwordHash      String?
  role              UserRole  @default(USER)
  isActive          Boolean   @default(true)
  isTwoFactorEnabled Boolean  @default(false)
  twoFactorSecret   String?
  failedLoginAttempts Int     @default(0)
  lockedUntil       DateTime?
  lastLoginAt       DateTime?
  createdAt         DateTime  @default(now())
  updatedAt         DateTime  @updatedAt

  accounts          Account[]
  sessions          Session[]
  refreshTokens     RefreshToken[]
  sosAlerts         SosAlert[]   @relation("ReportedBy")
  assignedAlerts    SosAlert[]   @relation("AssignedTo")
}

model RefreshToken {
  id          String    @id @default(uuid())
  token       String    @unique
  userId      String
  deviceInfo  String?
  ipAddress   String?
  expiresAt   DateTime
  revokedAt   DateTime?
  createdAt   DateTime  @default(now())
  
  user        User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@index([userId])
  @@index([token])
}

enum UserRole {
  ADMIN
  MODERATOR
  USER
  SOS_RESPONDER
}
```

---

## 🛠️ Step-by-Step Implementation

### Step 161: JWT Service

```typescript
// src/lib/auth/jwt.ts
import { SignJWT, jwtVerify, type JWTPayload } from 'jose';
import { Redis } from 'ioredis';

const redis = new Redis(process.env.REDIS_URL!);

export interface AccessTokenPayload extends JWTPayload {
  sub: string;       // user id
  email: string;
  role: string;
  permissions: string[];
  sessionId: string;
}

const ACCESS_TOKEN_EXPIRY = '15m';
const REFRESH_TOKEN_EXPIRY = '30d';

const accessSecret = new TextEncoder().encode(process.env.JWT_ACCESS_SECRET!);
const refreshSecret = new TextEncoder().encode(process.env.JWT_REFRESH_SECRET!);

// Sign access token
export async function signAccessToken(payload: Omit<AccessTokenPayload, 'iat' | 'exp'>): Promise<string> {
  return new SignJWT(payload)
    .setProtectedHeader({ alg: 'HS256' })
    .setIssuedAt()
    .setExpirationTime(ACCESS_TOKEN_EXPIRY)
    .setIssuer('chuaikan.com')
    .setAudience('chuaikan-api')
    .sign(accessSecret);
}

// Verify access token
export async function verifyAccessToken(token: string): Promise<AccessTokenPayload> {
  // Check blacklist first
  const isBlacklisted = await redis.get(`blacklist:${token}`);
  if (isBlacklisted) {
    throw new Error('Token has been revoked');
  }

  const { payload } = await jwtVerify(token, accessSecret, {
    issuer: 'chuaikan.com',
    audience: 'chuaikan-api',
  });

  return payload as AccessTokenPayload;
}

// Sign refresh token (opaque token, stored in DB)
export function generateRefreshToken(): string {
  const buffer = new Uint8Array(32);
  crypto.getRandomValues(buffer);
  return Buffer.from(buffer).toString('hex');
}

// Blacklist access token (for logout)
export async function blacklistAccessToken(token: string, expiresIn: number): Promise<void> {
  await redis.setex(`blacklist:${token}`, expiresIn, '1');
}

// Get permissions for role
export function getPermissionsForRole(role: string): string[] {
  const permissions: Record<string, string[]> = {
    ADMIN: ['*'],
    MODERATOR: [
      'sos:manage', 'sos:view', 'user:view',
      'report:manage', 'location:view', 'notification:send',
    ],
    SOS_RESPONDER: [
      'sos:respond', 'sos:update', 'sos:view',
      'location:track', 'notification:receive',
    ],
    USER: [
      'sos:create', 'sos:view:own',
      'profile:manage', 'notification:receive',
    ],
  };
  
  return permissions[role] || permissions.USER;
}
```

### Step 162: Refresh Token Rotation

```typescript
// src/lib/auth/token-service.ts
import { db } from '@/lib/db';
import { 
  signAccessToken, 
  generateRefreshToken, 
  blacklistAccessToken,
  getPermissionsForRole 
} from './jwt';
import { addDays, addMinutes } from 'date-fns';

export interface TokenPair {
  accessToken: string;
  refreshToken: string;
  expiresAt: Date;
}

export async function issueTokenPair(
  userId: string,
  deviceInfo?: string,
  ipAddress?: string
): Promise<TokenPair> {
  const user = await db.user.findUnique({
    where: { id: userId },
    select: { id: true, email: true, role: true },
  });

  if (!user) throw new Error('User not found');

  const sessionId = crypto.randomUUID();
  const permissions = getPermissionsForRole(user.role);

  // Sign access token
  const accessToken = await signAccessToken({
    sub: user.id,
    email: user.email,
    role: user.role,
    permissions,
    sessionId,
  });

  // Generate refresh token
  const refreshTokenValue = generateRefreshToken();
  const refreshTokenExpiry = addDays(new Date(), 30);

  // Store refresh token in database
  await db.refreshToken.create({
    data: {
      token: refreshTokenValue,
      userId: user.id,
      deviceInfo,
      ipAddress,
      expiresAt: refreshTokenExpiry,
    },
  });

  // Update last login
  await db.user.update({
    where: { id: userId },
    data: { lastLoginAt: new Date() },
  });

  return {
    accessToken,
    refreshToken: refreshTokenValue,
    expiresAt: addMinutes(new Date(), 15),
  };
}

// Rotate refresh token (use old token, get new pair)
export async function rotateRefreshToken(
  oldRefreshToken: string,
  deviceInfo?: string,
  ipAddress?: string
): Promise<TokenPair> {
  const storedToken = await db.refreshToken.findUnique({
    where: { token: oldRefreshToken },
    include: { user: true },
  });

  if (!storedToken) {
    throw new Error('Invalid refresh token');
  }

  if (storedToken.revokedAt) {
    // Token reuse detected! Revoke all tokens for this user (potential compromise)
    await db.refreshToken.updateMany({
      where: { userId: storedToken.userId, revokedAt: null },
      data: { revokedAt: new Date() },
    });
    throw new Error('Refresh token reuse detected. All sessions revoked.');
  }

  if (storedToken.expiresAt < new Date()) {
    await db.refreshToken.update({
      where: { id: storedToken.id },
      data: { revokedAt: new Date() },
    });
    throw new Error('Refresh token expired');
  }

  // Revoke old token
  await db.refreshToken.update({
    where: { id: storedToken.id },
    data: { revokedAt: new Date() },
  });

  // Issue new token pair
  return issueTokenPair(storedToken.userId, deviceInfo, ipAddress);
}

// Revoke all user sessions (logout from all devices)
export async function revokeAllUserSessions(userId: string): Promise<void> {
  await db.refreshToken.updateMany({
    where: { userId, revokedAt: null },
    data: { revokedAt: new Date() },
  });
}
```

### Step 163: NextAuth.js v5 Configuration

```typescript
// src/auth.ts
import NextAuth from 'next-auth';
import { PrismaAdapter } from '@auth/prisma-adapter';
import Credentials from 'next-auth/providers/credentials';
import Google from 'next-auth/providers/google';
import { db } from '@/lib/db';
import { issueTokenPair } from '@/lib/auth/token-service';
import bcrypt from 'bcryptjs';
import { z } from 'zod';

const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

export const { handlers, auth, signIn, signOut } = NextAuth({
  adapter: PrismaAdapter(db),
  session: { strategy: 'jwt' },
  
  providers: [
    Google({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
      profile(profile) {
        return {
          id: profile.sub,
          name: profile.name,
          email: profile.email,
          image: profile.picture,
          role: 'USER',
        };
      },
    }),

    // LINE Login (ยอดนิยมในไทย)
    {
      id: 'line',
      name: 'LINE',
      type: 'oauth',
      clientId: process.env.LINE_CLIENT_ID!,
      clientSecret: process.env.LINE_CLIENT_SECRET!,
      authorization: {
        url: 'https://access.line.me/oauth2/v2.1/authorize',
        params: {
          scope: 'profile openid email',
          nonce: crypto.randomUUID(),
        },
      },
      token: 'https://api.line.me/oauth2/v2.1/token',
      userinfo: 'https://api.line.me/v2/profile',
      profile(profile) {
        return {
          id: profile.userId,
          name: profile.displayName,
          email: profile.email || `line_${profile.userId}@line.user`,
          image: profile.pictureUrl,
          role: 'USER',
        };
      },
    },

    Credentials({
      credentials: {
        email: { label: 'Email', type: 'email' },
        password: { label: 'Password', type: 'password' },
      },
      async authorize(credentials) {
        const { email, password } = loginSchema.parse(credentials);

        const user = await db.user.findUnique({
          where: { email },
          select: {
            id: true,
            email: true,
            name: true,
            role: true,
            passwordHash: true,
            isActive: true,
            failedLoginAttempts: true,
            lockedUntil: true,
          },
        });

        if (!user || !user.passwordHash) return null;
        if (!user.isActive) throw new Error('Account is disabled');
        
        // Check account lockout
        if (user.lockedUntil && user.lockedUntil > new Date()) {
          const minutesLeft = Math.ceil(
            (user.lockedUntil.getTime() - Date.now()) / 60000
          );
          throw new Error(`Account locked. Try again in ${minutesLeft} minutes`);
        }

        const isValidPassword = await bcrypt.compare(password, user.passwordHash);

        if (!isValidPassword) {
          // Increment failed attempts
          const newAttempts = user.failedLoginAttempts + 1;
          const lockoutData = newAttempts >= 5 
            ? { lockedUntil: new Date(Date.now() + 30 * 60 * 1000) } // Lock 30 mins
            : {};

          await db.user.update({
            where: { id: user.id },
            data: {
              failedLoginAttempts: newAttempts,
              ...lockoutData,
            },
          });

          return null;
        }

        // Reset failed attempts on success
        await db.user.update({
          where: { id: user.id },
          data: { failedLoginAttempts: 0, lockedUntil: null },
        });

        return {
          id: user.id,
          email: user.email,
          name: user.name,
          role: user.role,
        };
      },
    }),
  ],

  callbacks: {
    async jwt({ token, user, account }) {
      if (user) {
        token.userId = user.id;
        token.role = (user as any).role;
        token.permissions = getPermissionsForRole((user as any).role);
      }
      return token;
    },
    
    async session({ session, token }) {
      session.user.id = token.userId as string;
      session.user.role = token.role as string;
      session.user.permissions = token.permissions as string[];
      return session;
    },
  },

  pages: {
    signIn: '/auth/login',
    error: '/auth/error',
  },
});
```

### Step 164: RBAC Implementation

```typescript
// src/lib/auth/rbac.ts
import { auth } from '@/auth';
import { NextRequest } from 'next/server';

export type Permission = 
  | '*'
  | 'sos:create' | 'sos:view' | 'sos:view:own' | 'sos:manage' | 'sos:respond' | 'sos:update'
  | 'user:view' | 'user:manage'
  | 'profile:manage'
  | 'report:manage'
  | 'location:view' | 'location:track'
  | 'notification:send' | 'notification:receive'
  | 'system:configure';

export type UserRole = 'ADMIN' | 'MODERATOR' | 'SOS_RESPONDER' | 'USER';

const ROLE_PERMISSIONS: Record<UserRole, Permission[]> = {
  ADMIN: ['*'],
  MODERATOR: [
    'sos:manage', 'sos:view', 'user:view',
    'report:manage', 'location:view', 'notification:send',
  ],
  SOS_RESPONDER: [
    'sos:respond', 'sos:update', 'sos:view',
    'location:track', 'notification:receive',
  ],
  USER: [
    'sos:create', 'sos:view:own',
    'profile:manage', 'notification:receive',
  ],
};

export function hasPermission(
  userRole: UserRole,
  userPermissions: Permission[],
  required: Permission
): boolean {
  // Admin has all permissions
  if (userRole === 'ADMIN' || userPermissions.includes('*')) return true;
  
  // Check exact permission
  if (userPermissions.includes(required)) return true;
  
  // Check wildcard (sos:* covers sos:create, sos:view, etc.)
  const [category] = required.split(':');
  if (userPermissions.some(p => p === `${category}:*`)) return true;
  
  return false;
}

// Middleware for protecting routes
export function requirePermission(permission: Permission) {
  return async function middleware(req: NextRequest) {
    const session = await auth();
    
    if (!session?.user) {
      return Response.json(
        { success: false, error: { code: 'UNAUTHORIZED', message: 'กรุณาเข้าสู่ระบบก่อน' } },
        { status: 401 }
      );
    }

    const can = hasPermission(
      session.user.role as UserRole,
      session.user.permissions as Permission[],
      permission
    );

    if (!can) {
      return Response.json(
        { success: false, error: { code: 'FORBIDDEN', message: 'คุณไม่มีสิทธิ์ดำเนินการนี้' } },
        { status: 403 }
      );
    }

    return null; // Allow request to proceed
  };
}

// React hook for checking permissions on client
export function usePermissions() {
  // ใช้ร่วมกับ useSession ของ NextAuth
  return {
    can: (permission: Permission, userRole: UserRole, userPermissions: Permission[]) =>
      hasPermission(userRole, userPermissions, permission),
  };
}
```

### Step 165: Row-Level Security ใน PostgreSQL

```sql
-- database/migrations/enable_rls.sql
-- เปิดใช้งาน Row-Level Security สำหรับ sos_alerts

-- Enable RLS
ALTER TABLE sos_alerts ENABLE ROW LEVEL SECURITY;

-- Policy: Users can only see their own alerts (unless admin/moderator)
CREATE POLICY sos_alerts_select_policy ON sos_alerts
  FOR SELECT
  USING (
    -- Admin and moderator see all
    current_setting('app.user_role') IN ('ADMIN', 'MODERATOR', 'SOS_RESPONDER')
    OR
    -- Users see their own alerts
    reported_by_id = current_setting('app.user_id')::uuid
    OR
    -- Assigned responders see their alerts
    assigned_to_id = current_setting('app.user_id')::uuid
  );

-- Policy: Only authenticated users can create alerts
CREATE POLICY sos_alerts_insert_policy ON sos_alerts
  FOR INSERT
  WITH CHECK (
    current_setting('app.user_id') IS NOT NULL
    AND reported_by_id = current_setting('app.user_id')::uuid
  );

-- Policy: Only admins and responders can update alerts
CREATE POLICY sos_alerts_update_policy ON sos_alerts
  FOR UPDATE
  USING (
    current_setting('app.user_role') IN ('ADMIN', 'MODERATOR', 'SOS_RESPONDER')
  );

-- Set session variables when connecting (in Prisma middleware)
-- SET app.user_id = 'user-uuid-here';
-- SET app.user_role = 'USER';
```

```typescript
// src/lib/db/rls-client.ts
// Prisma middleware to set RLS session variables
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

export async function withRLS<T>(
  userId: string,
  userRole: string,
  fn: (tx: PrismaClient) => Promise<T>
): Promise<T> {
  return prisma.$transaction(async (tx) => {
    // Set session variables for RLS
    await tx.$executeRaw`SELECT set_config('app.user_id', ${userId}, true)`;
    await tx.$executeRaw`SELECT set_config('app.user_role', ${userRole}, true)`;
    
    return fn(tx as unknown as PrismaClient);
  });
}

// Usage in API routes:
// const alerts = await withRLS(user.id, user.role, async (db) => {
//   return db.sosAlert.findMany();
// });
```

### Step 166: 2FA ด้วย TOTP

```typescript
// src/lib/auth/totp.ts
import { authenticator } from 'otplib';
import QRCode from 'qrcode';
import { db } from '@/lib/db';
import { encrypt, decrypt } from '@/lib/crypto';

// Configure TOTP
authenticator.options = {
  window: 1, // Allow 1 step before/after for clock drift
  step: 30,  // 30 second intervals
};

// Setup 2FA for user
export async function setup2FA(userId: string): Promise<{
  secret: string;
  qrCodeUrl: string;
  backupCodes: string[];
}> {
  const user = await db.user.findUnique({
    where: { id: userId },
    select: { email: true, name: true },
  });

  if (!user) throw new Error('User not found');

  // Generate secret
  const secret = authenticator.generateSecret(32);
  const encryptedSecret = encrypt(secret);

  // Generate backup codes
  const backupCodes = Array.from({ length: 8 }, () => 
    Math.random().toString(36).substring(2, 10).toUpperCase()
  );

  // Store encrypted secret (not enabled yet, until verified)
  await db.user.update({
    where: { id: userId },
    data: {
      twoFactorSecret: encryptedSecret,
      // Don't enable yet - enable after verification
    },
  });

  // Generate QR code
  const otpAuthUrl = authenticator.keyuri(
    user.email,
    'Chuaikan',
    secret
  );
  const qrCodeUrl = await QRCode.toDataURL(otpAuthUrl);

  return {
    secret,
    qrCodeUrl,
    backupCodes,
  };
}

// Verify 2FA token and enable
export async function verify2FA(userId: string, token: string): Promise<boolean> {
  const user = await db.user.findUnique({
    where: { id: userId },
    select: { twoFactorSecret: true },
  });

  if (!user?.twoFactorSecret) throw new Error('2FA not setup');

  const secret = decrypt(user.twoFactorSecret);
  const isValid = authenticator.check(token, secret);

  if (isValid) {
    await db.user.update({
      where: { id: userId },
      data: { isTwoFactorEnabled: true },
    });
  }

  return isValid;
}

// Validate 2FA token during login
export async function validate2FAToken(
  userId: string, 
  token: string
): Promise<boolean> {
  const user = await db.user.findUnique({
    where: { id: userId },
    select: { twoFactorSecret: true, isTwoFactorEnabled: true },
  });

  if (!user?.isTwoFactorEnabled || !user?.twoFactorSecret) {
    return true; // 2FA not enabled, allow
  }

  const secret = decrypt(user.twoFactorSecret);
  return authenticator.check(token, secret);
}
```

### Step 167: Rate Limiting สำหรับ Auth Endpoints

```typescript
// src/app/api/v1/auth/login/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { z } from 'zod';
import { db } from '@/lib/db';
import { signIn } from '@/auth';
import { issueTokenPair } from '@/lib/auth/token-service';
import { validate2FAToken } from '@/lib/auth/totp';
import { rateLimiters, withRateLimit } from '@/lib/middleware/rate-limit';
import bcrypt from 'bcryptjs';

const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(1),
  totpCode: z.string().length(6).optional(),
  rememberDevice: z.boolean().default(false),
});

export async function POST(req: NextRequest) {
  // Rate limit by IP: 5 attempts per minute
  const ip = req.headers.get('x-forwarded-for') || 'unknown';
  const { success, response } = await withRateLimit(req, rateLimiters.auth, `login:${ip}`);
  if (!success) return response;

  try {
    const body = await req.json();
    const { email, password, totpCode } = loginSchema.parse(body);

    const user = await db.user.findUnique({
      where: { email },
      select: {
        id: true,
        email: true,
        name: true,
        role: true,
        passwordHash: true,
        isActive: true,
        failedLoginAttempts: true,
        lockedUntil: true,
        isTwoFactorEnabled: true,
      },
    });

    // Generic error to prevent user enumeration
    if (!user || !user.passwordHash) {
      return NextResponse.json(
        { success: false, error: { code: 'INVALID_CREDENTIALS', message: 'อีเมลหรือรหัสผ่านไม่ถูกต้อง' } },
        { status: 401 }
      );
    }

    // Check lockout
    if (user.lockedUntil && user.lockedUntil > new Date()) {
      const minutesLeft = Math.ceil((user.lockedUntil.getTime() - Date.now()) / 60000);
      return NextResponse.json(
        { 
          success: false, 
          error: { 
            code: 'ACCOUNT_LOCKED', 
            message: `บัญชีถูกล็อก กรุณารออีก ${minutesLeft} นาที` 
          } 
        },
        { status: 423 }
      );
    }

    const isValidPassword = await bcrypt.compare(password, user.passwordHash);

    if (!isValidPassword) {
      const newAttempts = user.failedLoginAttempts + 1;
      
      await db.user.update({
        where: { id: user.id },
        data: {
          failedLoginAttempts: newAttempts,
          ...(newAttempts >= 5 ? {
            lockedUntil: new Date(Date.now() + 30 * 60 * 1000),
          } : {}),
        },
      });

      const remaining = Math.max(0, 5 - newAttempts);
      return NextResponse.json(
        { 
          success: false, 
          error: { 
            code: 'INVALID_CREDENTIALS', 
            message: `รหัสผ่านไม่ถูกต้อง ${remaining > 0 ? `(เหลืออีก ${remaining} ครั้ง)` : 'บัญชีถูกล็อก'}`
          } 
        },
        { status: 401 }
      );
    }

    // Validate 2FA if enabled
    if (user.isTwoFactorEnabled) {
      if (!totpCode) {
        return NextResponse.json(
          { success: false, error: { code: 'TOTP_REQUIRED', message: 'กรุณาใส่รหัส 2FA' } },
          { status: 400 }
        );
      }

      const isValidTotp = await validate2FAToken(user.id, totpCode);
      if (!isValidTotp) {
        return NextResponse.json(
          { success: false, error: { code: 'INVALID_TOTP', message: 'รหัส 2FA ไม่ถูกต้อง' } },
          { status: 401 }
        );
      }
    }

    // Issue tokens
    const deviceInfo = req.headers.get('user-agent') || undefined;
    const { accessToken, refreshToken, expiresAt } = await issueTokenPair(
      user.id,
      deviceInfo,
      ip
    );

    // Reset failed attempts
    await db.user.update({
      where: { id: user.id },
      data: { failedLoginAttempts: 0, lockedUntil: null },
    });

    // Create response
    const res = NextResponse.json({
      success: true,
      data: {
        accessToken,
        expiresAt: expiresAt.toISOString(),
        user: {
          id: user.id,
          email: user.email,
          name: user.name,
          role: user.role,
        },
      },
    });

    // Set refresh token in HttpOnly cookie
    res.cookies.set('refresh_token', refreshToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict',
      maxAge: 30 * 24 * 60 * 60, // 30 days
      path: '/api/v1/auth',
    });

    return res;
  } catch (error) {
    console.error('[Login Error]', error);
    return NextResponse.json(
      { success: false, error: { code: 'INTERNAL_ERROR', message: 'เกิดข้อผิดพลาด' } },
      { status: 500 }
    );
  }
}
```

### Step 168: Password Policy

```typescript
// src/lib/auth/password-policy.ts
import bcrypt from 'bcryptjs';

export interface PasswordValidationResult {
  isValid: boolean;
  errors: string[];
  strength: 'weak' | 'fair' | 'strong' | 'very_strong';
}

const PASSWORD_POLICY = {
  minLength: 8,
  maxLength: 128,
  requireUppercase: true,
  requireLowercase: true,
  requireNumbers: true,
  requireSpecialChars: true,
  specialChars: '!@#$%^&*()_+-=[]{}|;:,.<>?',
  preventCommonPasswords: true,
};

const COMMON_PASSWORDS = new Set([
  'password', 'password123', '123456789', 'qwerty123',
  'admin123', 'letmein', 'welcome1', 'monkey123',
  'dragon123', 'master123',
]);

export function validatePassword(password: string): PasswordValidationResult {
  const errors: string[] = [];
  let score = 0;

  if (password.length < PASSWORD_POLICY.minLength) {
    errors.push(`รหัสผ่านต้องมีความยาวอย่างน้อย ${PASSWORD_POLICY.minLength} ตัวอักษร`);
  } else {
    score += 1;
  }

  if (password.length > PASSWORD_POLICY.maxLength) {
    errors.push(`รหัสผ่านต้องมีความยาวไม่เกิน ${PASSWORD_POLICY.maxLength} ตัวอักษร`);
  }

  if (PASSWORD_POLICY.requireUppercase && !/[A-Z]/.test(password)) {
    errors.push('รหัสผ่านต้องมีตัวอักษรพิมพ์ใหญ่อย่างน้อย 1 ตัว');
  } else if (/[A-Z]/.test(password)) {
    score += 1;
  }

  if (PASSWORD_POLICY.requireLowercase && !/[a-z]/.test(password)) {
    errors.push('รหัสผ่านต้องมีตัวอักษรพิมพ์เล็กอย่างน้อย 1 ตัว');
  } else if (/[a-z]/.test(password)) {
    score += 1;
  }

  if (PASSWORD_POLICY.requireNumbers && !/\d/.test(password)) {
    errors.push('รหัสผ่านต้องมีตัวเลขอย่างน้อย 1 ตัว');
  } else if (/\d/.test(password)) {
    score += 1;
  }

  const specialCharsRegex = new RegExp(
    `[${PASSWORD_POLICY.specialChars.replace(/[-[\]{}()*+?.,\\^$|#\s]/g, '\\$&')}]`
  );
  if (PASSWORD_POLICY.requireSpecialChars && !specialCharsRegex.test(password)) {
    errors.push('รหัสผ่านต้องมีอักขระพิเศษอย่างน้อย 1 ตัว (!@#$%...)');
  } else if (specialCharsRegex.test(password)) {
    score += 1;
  }

  if (COMMON_PASSWORDS.has(password.toLowerCase())) {
    errors.push('รหัสผ่านนี้ใช้บ่อยเกินไป กรุณาเลือกรหัสผ่านอื่น');
    score = 0;
  }

  if (password.length >= 12) score += 1;
  if (password.length >= 16) score += 1;

  const strength: PasswordValidationResult['strength'] = 
    score <= 2 ? 'weak' :
    score <= 4 ? 'fair' :
    score <= 5 ? 'strong' : 'very_strong';

  return {
    isValid: errors.length === 0,
    errors,
    strength,
  };
}

export async function hashPassword(password: string): Promise<string> {
  const { isValid, errors } = validatePassword(password);
  if (!isValid) {
    throw new Error(`Password validation failed: ${errors.join(', ')}`);
  }
  return bcrypt.hash(password, 12);
}
```

### Step 169: Next.js Middleware

```typescript
// middleware.ts (root level)
import { NextRequest, NextResponse } from 'next/server';
import { verifyAccessToken } from '@/lib/auth/jwt';
import { hasPermission } from '@/lib/auth/rbac';

// Routes that require authentication
const PROTECTED_ROUTES = [
  '/dashboard',
  '/profile',
  '/admin',
];

// Routes that require specific permissions
const PERMISSION_ROUTES: Record<string, string> = {
  '/admin': '*',
  '/admin/users': 'user:manage',
  '/admin/reports': 'report:manage',
};

// Public routes (skip auth check)
const PUBLIC_ROUTES = [
  '/auth/login',
  '/auth/register',
  '/auth/forgot-password',
  '/',
  '/about',
];

export async function middleware(req: NextRequest) {
  const pathname = req.nextUrl.pathname;

  // Skip public routes
  if (PUBLIC_ROUTES.some(route => pathname.startsWith(route))) {
    return NextResponse.next();
  }

  // Check if route needs protection
  const isProtected = PROTECTED_ROUTES.some(route => pathname.startsWith(route));
  if (!isProtected) return NextResponse.next();

  // Get access token
  const authHeader = req.headers.get('Authorization');
  const token = authHeader?.startsWith('Bearer ') 
    ? authHeader.slice(7)
    : req.cookies.get('access_token')?.value;

  if (!token) {
    if (req.nextUrl.pathname.startsWith('/api/')) {
      return NextResponse.json(
        { success: false, error: { code: 'UNAUTHORIZED', message: 'กรุณาเข้าสู่ระบบก่อน' } },
        { status: 401 }
      );
    }
    return NextResponse.redirect(new URL('/auth/login', req.url));
  }

  try {
    const payload = await verifyAccessToken(token);

    // Check permission for specific routes
    const requiredPermission = PERMISSION_ROUTES[pathname];
    if (requiredPermission) {
      const hasAccess = hasPermission(
        payload.role as any,
        payload.permissions as any[],
        requiredPermission as any
      );

      if (!hasAccess) {
        return NextResponse.json(
          { success: false, error: { code: 'FORBIDDEN', message: 'คุณไม่มีสิทธิ์เข้าถึงส่วนนี้' } },
          { status: 403 }
        );
      }
    }

    // Add user info to request headers for downstream use
    const requestHeaders = new Headers(req.headers);
    requestHeaders.set('X-User-Id', payload.sub!);
    requestHeaders.set('X-User-Role', payload.role);
    requestHeaders.set('X-User-Permissions', payload.permissions.join(','));

    return NextResponse.next({ request: { headers: requestHeaders } });
  } catch {
    if (req.nextUrl.pathname.startsWith('/api/')) {
      return NextResponse.json(
        { success: false, error: { code: 'INVALID_TOKEN', message: 'Token ไม่ถูกต้องหรือหมดอายุ' } },
        { status: 401 }
      );
    }
    return NextResponse.redirect(new URL('/auth/login', req.url));
  }
}

export const config = {
  matcher: [
    '/((?!_next/static|_next/image|favicon.ico|public).*)',
  ],
};
```

### Step 170: Logout และ Token Blacklisting

```typescript
// src/app/api/v1/auth/logout/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { verifyAccessToken, blacklistAccessToken } from '@/lib/auth/jwt';
import { db } from '@/lib/db';

export async function POST(req: NextRequest) {
  const authHeader = req.headers.get('Authorization');
  const accessToken = authHeader?.startsWith('Bearer ') ? authHeader.slice(7) : null;
  const refreshToken = req.cookies.get('refresh_token')?.value;
  const logoutAll = req.nextUrl.searchParams.get('all') === 'true';

  try {
    if (accessToken) {
      try {
        const payload = await verifyAccessToken(accessToken);
        const expiresIn = Math.max(0, (payload.exp! - Math.floor(Date.now() / 1000)));
        
        // Blacklist access token
        await blacklistAccessToken(accessToken, expiresIn);

        // Revoke refresh token or all refresh tokens
        if (logoutAll) {
          await db.refreshToken.updateMany({
            where: { userId: payload.sub!, revokedAt: null },
            data: { revokedAt: new Date() },
          });
        } else if (refreshToken) {
          await db.refreshToken.updateMany({
            where: { token: refreshToken, revokedAt: null },
            data: { revokedAt: new Date() },
          });
        }
      } catch {
        // Token already expired/invalid, still clear cookies
      }
    }

    const response = NextResponse.json({
      success: true,
      data: { message: 'ออกจากระบบเรียบร้อยแล้ว' },
    });

    // Clear cookies
    response.cookies.delete('refresh_token');
    response.cookies.delete('access_token');

    return response;
  } catch (error) {
    return NextResponse.json(
      { success: false, error: { code: 'INTERNAL_ERROR', message: 'เกิดข้อผิดพลาด' } },
      { status: 500 }
    );
  }
}
```

---

## 🔧 Configuration Files

```typescript
// src/types/next-auth.d.ts
import { DefaultSession, DefaultUser } from 'next-auth';
import { JWT } from 'next-auth/jwt';

declare module 'next-auth' {
  interface Session {
    user: DefaultSession['user'] & {
      id: string;
      role: string;
      permissions: string[];
    };
  }

  interface User extends DefaultUser {
    role: string;
    permissions?: string[];
  }
}

declare module 'next-auth/jwt' {
  interface JWT {
    userId: string;
    role: string;
    permissions: string[];
  }
}
```

---

## 🧪 Testing

```typescript
// __tests__/auth/jwt.test.ts
import { describe, it, expect } from 'vitest';
import { signAccessToken, verifyAccessToken } from '@/lib/auth/jwt';

describe('JWT Service', () => {
  it('should sign and verify access token', async () => {
    const payload = {
      sub: 'user-123',
      email: 'test@example.com',
      role: 'USER',
      permissions: ['sos:create', 'profile:manage'],
      sessionId: 'session-abc',
    };

    const token = await signAccessToken(payload);
    expect(token).toBeDefined();
    expect(token.split('.')).toHaveLength(3);

    const decoded = await verifyAccessToken(token);
    expect(decoded.sub).toBe(payload.sub);
    expect(decoded.email).toBe(payload.email);
    expect(decoded.role).toBe(payload.role);
  });

  it('should reject expired token', async () => {
    // Create token with past expiry
    const { SignJWT } = await import('jose');
    const secret = new TextEncoder().encode(process.env.JWT_ACCESS_SECRET!);
    const expiredToken = await new SignJWT({ sub: 'user-123' })
      .setProtectedHeader({ alg: 'HS256' })
      .setExpirationTime('1s')
      .sign(secret);

    await new Promise(resolve => setTimeout(resolve, 2000));

    await expect(verifyAccessToken(expiredToken)).rejects.toThrow();
  });
});

// __tests__/auth/rbac.test.ts
describe('RBAC', () => {
  it('admin should have all permissions', () => {
    expect(hasPermission('ADMIN', ['*'], 'sos:create')).toBe(true);
    expect(hasPermission('ADMIN', ['*'], 'user:manage')).toBe(true);
    expect(hasPermission('ADMIN', ['*'], 'system:configure')).toBe(true);
  });

  it('user should not have admin permissions', () => {
    const userPermissions: Permission[] = ['sos:create', 'sos:view:own', 'profile:manage'];
    expect(hasPermission('USER', userPermissions, 'user:manage')).toBe(false);
    expect(hasPermission('USER', userPermissions, 'sos:manage')).toBe(false);
  });
});
```

---

## ❌ Common Errors & Solutions

### Error 1: JWT Secret Not Set

```
Error: JWT_ACCESS_SECRET is not defined
```
**แก้ไข:** ตรวจสอบ .env.local มีค่า JWT secrets ครบ

### Error 2: LINE Login Callback Error

```
Error: Invalid redirect_uri
```
**แก้ไข:** เพิ่ม callback URL ใน LINE Developer Console: `https://yourdomain.com/api/auth/callback/line`

### Error 3: Cookie Not Set (HTTPS Required)

```
Warning: Secure cookie requires HTTPS
```
**แก้ไข:** ใน development ตั้ง `NODE_ENV=development` ให้ cookie ทำงานได้แบบ non-secure

---

## ✅ Checklist

- [ ] JWT access token อายุ 15 นาที
- [ ] Refresh token อายุ 30 วัน เก็บใน HttpOnly cookie
- [ ] Refresh token rotation ทุกครั้งที่ใช้งาน
- [ ] Token reuse detection และ revoke all sessions
- [ ] LINE Login integration
- [ ] Google OAuth integration
- [ ] RBAC roles ครบ: admin, moderator, user, sos_responder
- [ ] Row-level security ใน PostgreSQL
- [ ] 2FA TOTP setup และ verification
- [ ] Rate limiting: 5 attempts/minute สำหรับ login
- [ ] Account lockout 30 นาที หลัง 5 failed attempts
- [ ] Password policy: uppercase, lowercase, number, special char
- [ ] Token blacklisting ด้วย Redis สำหรับ logout
- [ ] NextAuth.js v5 middleware ป้องกัน routes
- [ ] Audit log สำหรับ auth events

---

## 🔗 References

- [NextAuth.js v5 Documentation](https://authjs.dev/)
- [JWT Best Practices](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)
- [LINE Login Documentation](https://developers.line.biz/en/docs/line-login/)
- [OWASP Authentication Cheatsheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [TOTP RFC 6238](https://tools.ietf.org/html/rfc6238)

---
*Part 017 | Road to 1,000,000 Users/Day | chuaikan.com*
