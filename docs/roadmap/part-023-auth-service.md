# Part 023: Auth Service (แยก Service)
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 221-230
> **เวลาโดยประมาณ:** 10 ชั่วโมง
> **Prerequisites:** Part 021 (Microservices), Part 022 (API Gateway), JWT basics

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ออกแบบ Auth Service ที่ครบครันสำหรับ chuaikan.com
- Implement JWT ด้วย RS256 (asymmetric keys)
- Refresh Token Rotation ที่ปลอดภัย
- OAuth 2.0: LINE Login และ Google
- Email verification และ Password reset flow
- 2FA ด้วย TOTP (Authenticator app) และ SMS OTP
- Rate limiting สำหรับ authentication endpoints
- Token introspection สำหรับ inter-service communication
- Unit tests และ Integration tests สำหรับ Auth Service

---

## 📖 ทฤษฎีและแนวคิด

### Step 221: Auth Service Responsibilities

**สิ่งที่ Auth Service รับผิดชอบ:**

```
Auth Service Responsibilities:
┌─────────────────────────────────────────────────────────────────┐
│                        AUTH SERVICE                             │
│                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐  │
│  │  Registration   │  │  Authentication │  │   OAuth 2.0  │  │
│  │  - Email/Pass   │  │  - Login        │  │  - LINE      │  │
│  │  - Validation   │  │  - JWT issuance │  │  - Google    │  │
│  │  - Email verify │  │  - Token refresh│  │              │  │
│  └─────────────────┘  └─────────────────┘  └──────────────┘  │
│                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐  │
│  │  Token Mgmt     │  │      2FA         │  │  Password    │  │
│  │  - Blacklist    │  │  - TOTP setup   │  │  - Hashing   │  │
│  │  - Rotation     │  │  - SMS OTP      │  │  - Reset     │  │
│  │  - Introspect   │  │  - Verify       │  │              │  │
│  └─────────────────┘  └─────────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────────────┘

สิ่งที่ Auth Service ไม่ทำ:
✗ ไม่จัดการ User Profile (ของ User Service)
✗ ไม่ส่ง notification (ของ Notification Service)
✗ ไม่ตรวจสอบ permissions (ของแต่ละ service เอง)
```

### Step 222: JWT RS256 Architecture

```
RS256 (Asymmetric) vs HS256 (Symmetric):

HS256 - shared secret:
Auth Service → signs with "my-secret" → JWT
Post Service → verifies with "my-secret" → OK
ปัญหา: ทุก service ต้องมี secret → secret leak ได้ง่าย

RS256 - asymmetric keys:
Auth Service → signs with PRIVATE key → JWT
Post Service → verifies with PUBLIC key (read-only) → OK
ดีกว่า: PUBLIC key แชร์ได้ปลอดภัย, PRIVATE key อยู่ที่ auth service เท่านั้น

JWT Structure:
┌──────────────┬────────────────────────┬──────────────────────┐
│   Header     │        Payload         │      Signature       │
│ {            │ {                      │ RS256(              │
│   "alg":"RS256"│  "sub": "user-123",  │   base64(header)+   │
│   "typ":"JWT"│  "email": "...",       │   "."+              │
│   "kid": "key-id"│  "iat": 1234567890,│   base64(payload),  │
│ }            │  "exp": 1234571490,    │   privateKey)        │
│              │  "jti": "unique-id"    │                     │
│              │ }                      │                     │
└──────────────┴────────────────────────┴──────────────────────┘
```

---

## ⚙️ Environment Setup

### Step 223: Auth Service Project Structure

```bash
# สร้าง auth service
cd ~/chuaikan-platform/apps
mkdir -p auth-service
cd auth-service

# สร้าง package.json
cat > package.json << 'EOF'
{
  "name": "@chuaikan/auth-service",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "lint": "eslint src --ext .ts",
    "db:migrate": "prisma migrate dev",
    "db:generate": "prisma generate"
  },
  "dependencies": {
    "@chuaikan/shared-types": "workspace:*",
    "@chuaikan/utils": "workspace:*",
    "@prisma/client": "^5.17.0",
    "bcryptjs": "^2.4.3",
    "express": "^4.19.0",
    "express-rate-limit": "^7.4.0",
    "helmet": "^7.1.0",
    "ioredis": "^5.4.1",
    "jsonwebtoken": "^9.0.2",
    "nodemailer": "^6.9.14",
    "passport": "^0.7.0",
    "passport-google-oauth20": "^2.0.0",
    "passport-line": "^1.1.1",
    "speakeasy": "^2.0.0",
    "twilio": "^5.2.0",
    "uuid": "^10.0.0",
    "winston": "^3.14.0",
    "zod": "^3.23.0"
  },
  "devDependencies": {
    "@types/bcryptjs": "^2.4.6",
    "@types/express": "^4.17.21",
    "@types/jsonwebtoken": "^9.0.6",
    "@types/node": "^22.0.0",
    "@types/passport-google-oauth20": "^2.0.14",
    "@types/speakeasy": "^2.0.7",
    "@types/uuid": "^10.0.0",
    "@vitest/coverage-v8": "^2.0.0",
    "nock": "^13.5.5",
    "prisma": "^5.17.0",
    "supertest": "^7.0.0",
    "tsx": "^4.16.0",
    "typescript": "^5.5.0",
    "vitest": "^2.0.0"
  }
}
EOF

# สร้าง directory structure
mkdir -p src/{config,controllers,middleware,models,routes,services,utils,__tests__}

# Generate RSA keys สำหรับ JWT
mkdir -p keys
openssl genrsa -out keys/private.pem 2048
openssl rsa -in keys/private.pem -pubout -out keys/public.pem
echo "Keys generated:"
ls -la keys/
```

---

## 🛠️ Step-by-Step Implementation

### Step 224: Core Auth Service Files

```typescript
// apps/auth-service/src/config/env.ts

import { z } from 'zod';

const EnvSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  PORT: z.coerce.number().default(3001),
  
  // Database
  DATABASE_URL: z.string().url(),
  
  // Redis
  REDIS_URL: z.string().url(),
  
  // JWT Keys
  JWT_PRIVATE_KEY_PATH: z.string().default('./keys/private.pem'),
  JWT_PUBLIC_KEY_PATH: z.string().default('./keys/public.pem'),
  JWT_ACCESS_TOKEN_TTL: z.string().default('15m'),
  JWT_REFRESH_TOKEN_TTL: z.string().default('30d'),
  JWT_KEY_ID: z.string().default('chuaikan-key-v1'),
  
  // OAuth - LINE
  LINE_CHANNEL_ID: z.string(),
  LINE_CHANNEL_SECRET: z.string(),
  LINE_CALLBACK_URL: z.string().url(),
  
  // OAuth - Google
  GOOGLE_CLIENT_ID: z.string(),
  GOOGLE_CLIENT_SECRET: z.string(),
  GOOGLE_CALLBACK_URL: z.string().url(),
  
  // Twilio (SMS)
  TWILIO_ACCOUNT_SID: z.string().optional(),
  TWILIO_AUTH_TOKEN: z.string().optional(),
  TWILIO_PHONE_NUMBER: z.string().optional(),
  
  // Email
  SMTP_HOST: z.string().default('smtp.gmail.com'),
  SMTP_PORT: z.coerce.number().default(587),
  SMTP_USER: z.string().email(),
  SMTP_PASS: z.string(),
  EMAIL_FROM: z.string().default('noreply@chuaikan.com'),
  
  // App
  APP_URL: z.string().url().default('https://chuaikan.com'),
  SERVICE_ID: z.string().default('auth-service'),
});

export type Env = z.infer<typeof EnvSchema>;

function loadEnv(): Env {
  const result = EnvSchema.safeParse(process.env);
  if (!result.success) {
    console.error('Invalid environment variables:', result.error.format());
    process.exit(1);
  }
  return result.data;
}

export const env = loadEnv();
```

```typescript
// apps/auth-service/src/config/jwt.ts

import fs from 'fs';
import jwt from 'jsonwebtoken';
import { env } from './env';
import type { User } from '@chuaikan/shared-types';

const privateKey = fs.readFileSync(env.JWT_PRIVATE_KEY_PATH, 'utf8');
const publicKey = fs.readFileSync(env.JWT_PUBLIC_KEY_PATH, 'utf8');

export interface AccessTokenPayload {
  sub: string;      // user ID
  email: string;
  username: string;
  isVerified: boolean;
  jti: string;      // JWT ID (unique per token)
  type: 'access';
}

export interface RefreshTokenPayload {
  sub: string;
  jti: string;
  tokenFamily: string;  // สำหรับ refresh token rotation
  type: 'refresh';
}

export function signAccessToken(user: Pick<User, 'id' | 'email' | 'username' | 'isVerified'>): string {
  const payload: Omit<AccessTokenPayload, 'iat' | 'exp'> = {
    sub: user.id,
    email: user.email,
    username: user.username,
    isVerified: user.isVerified,
    jti: `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
    type: 'access',
  };

  return jwt.sign(payload, privateKey, {
    algorithm: 'RS256',
    expiresIn: env.JWT_ACCESS_TOKEN_TTL,
    issuer: 'chuaikan-auth',
    audience: 'chuaikan-services',
    keyid: env.JWT_KEY_ID,
  });
}

export function signRefreshToken(userId: string, tokenFamily?: string): string {
  const family = tokenFamily || `family-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  
  const payload: Omit<RefreshTokenPayload, 'iat' | 'exp'> = {
    sub: userId,
    jti: `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
    tokenFamily: family,
    type: 'refresh',
  };

  return jwt.sign(payload, privateKey, {
    algorithm: 'RS256',
    expiresIn: env.JWT_REFRESH_TOKEN_TTL,
    issuer: 'chuaikan-auth',
  });
}

export function verifyToken(token: string): AccessTokenPayload | RefreshTokenPayload {
  return jwt.verify(token, publicKey, {
    algorithms: ['RS256'],
    issuer: 'chuaikan-auth',
  }) as AccessTokenPayload | RefreshTokenPayload;
}

export function getPublicKey(): string {
  return publicKey;
}
```

### Step 225: Prisma Schema และ Database

```prisma
// apps/auth-service/prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id              String   @id @default(cuid())
  email           String   @unique
  username        String   @unique
  passwordHash    String?  // null สำหรับ OAuth users
  displayName     String
  avatarUrl       String?
  isVerified      Boolean  @default(false)
  isActive        Boolean  @default(true)
  
  // Email verification
  emailVerifyToken    String?   @unique
  emailVerifyExpiry   DateTime?
  
  // Password reset
  passwordResetToken  String?   @unique
  passwordResetExpiry DateTime?
  
  // 2FA
  twoFactorSecret   String?
  twoFactorEnabled  Boolean @default(false)
  backupCodes       String[] // hashed backup codes
  
  // Login tracking
  lastLoginAt     DateTime?
  loginAttempts   Int       @default(0)
  lockedUntil     DateTime?
  
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  refreshTokens RefreshToken[]
  oauthAccounts OAuthAccount[]
  
  @@index([email])
  @@index([username])
}

model RefreshToken {
  id           String   @id @default(cuid())
  userId       String
  tokenHash    String   @unique  // เก็บ hash ไม่เก็บ token จริง
  tokenFamily  String   // สำหรับ rotation detection
  deviceInfo   String?
  ipAddress    String?
  expiresAt    DateTime
  revokedAt    DateTime?
  createdAt    DateTime @default(now())
  
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@index([userId])
  @@index([tokenFamily])
}

model OAuthAccount {
  id           String   @id @default(cuid())
  userId       String
  provider     String   // 'google', 'line'
  providerId   String   // user ID จาก provider
  accessToken  String?
  refreshToken String?
  expiresAt    DateTime?
  profileData  Json?
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt
  
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@unique([provider, providerId])
  @@index([userId])
}

model RevokedToken {
  jti       String   @id  // JWT ID
  expiresAt DateTime
  revokedAt DateTime @default(now())
  reason    String?
  
  @@index([expiresAt])  // สำหรับ cleanup job
}
```

```bash
# สร้าง migration
cd apps/auth-service
npx prisma migrate dev --name "initial_auth_schema"
npx prisma generate
```

### Step 226: Auth Controllers

```typescript
// apps/auth-service/src/controllers/auth.controller.ts

import { Request, Response, NextFunction } from 'express';
import { z } from 'zod';
import { AuthService } from '../services/auth.service';
import { ApiResponse } from '@chuaikan/shared-types';

const RegisterSchema = z.object({
  email: z.string().email(),
  username: z.string().min(3).max(30).regex(/^[a-zA-Z0-9_]+$/),
  password: z.string().min(8),
  displayName: z.string().min(1).max(50),
});

const LoginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(1),
  deviceInfo: z.string().optional(),
  totpCode: z.string().length(6).optional(),
});

const RefreshSchema = z.object({
  refreshToken: z.string().min(1),
});

export class AuthController {
  constructor(private readonly authService: AuthService) {}

  register = async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    try {
      const data = RegisterSchema.parse(req.body);
      const result = await this.authService.register(data);
      
      res.status(201).json({
        success: true,
        data: result,
      } satisfies ApiResponse<typeof result>);
    } catch (error) {
      next(error);
    }
  };

  login = async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    try {
      const data = LoginSchema.parse(req.body);
      const ipAddress = req.ip || req.socket.remoteAddress || 'unknown';
      
      const result = await this.authService.login({
        ...data,
        ipAddress,
      });

      // Set refresh token as httpOnly cookie
      res.cookie('refreshToken', result.refreshToken, {
        httpOnly: true,
        secure: process.env.NODE_ENV === 'production',
        sameSite: 'strict',
        maxAge: 30 * 24 * 60 * 60 * 1000, // 30 days
        path: '/api/v1/auth/refresh',
      });

      res.json({
        success: true,
        data: {
          accessToken: result.accessToken,
          user: result.user,
          requiresTwoFactor: result.requiresTwoFactor,
        },
      });
    } catch (error) {
      next(error);
    }
  };

  refresh = async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    try {
      // รับจาก cookie หรือ body
      const refreshToken = req.cookies?.refreshToken || RefreshSchema.parse(req.body).refreshToken;
      const ipAddress = req.ip || 'unknown';
      
      const result = await this.authService.refreshTokens(refreshToken, ipAddress);

      // Update cookie
      res.cookie('refreshToken', result.refreshToken, {
        httpOnly: true,
        secure: process.env.NODE_ENV === 'production',
        sameSite: 'strict',
        maxAge: 30 * 24 * 60 * 60 * 1000,
        path: '/api/v1/auth/refresh',
      });

      res.json({
        success: true,
        data: { accessToken: result.accessToken },
      });
    } catch (error) {
      next(error);
    }
  };

  logout = async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    try {
      const refreshToken = req.cookies?.refreshToken;
      const userId = (req as any).user?.sub;
      
      if (refreshToken && userId) {
        await this.authService.logout(userId, refreshToken);
      }

      // Clear cookie
      res.clearCookie('refreshToken', { path: '/api/v1/auth/refresh' });
      res.json({ success: true });
    } catch (error) {
      next(error);
    }
  };

  // Token introspection endpoint สำหรับ other services
  introspect = async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    try {
      const { token } = req.body;
      const result = await this.authService.introspectToken(token);
      res.json({ success: true, data: result });
    } catch (error) {
      next(error);
    }
  };

  // 2FA setup
  setup2FA = async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    try {
      const userId = (req as any).user.sub;
      const result = await this.authService.setup2FA(userId);
      res.json({ success: true, data: result });
    } catch (error) {
      next(error);
    }
  };

  verify2FA = async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    try {
      const userId = (req as any).user.sub;
      const { totpCode } = z.object({ totpCode: z.string().length(6) }).parse(req.body);
      
      const result = await this.authService.verify2FA(userId, totpCode);
      res.json({ success: true, data: result });
    } catch (error) {
      next(error);
    }
  };
}
```

### Step 227: Auth Service (Business Logic)

```typescript
// apps/auth-service/src/services/auth.service.ts

import bcrypt from 'bcryptjs';
import speakeasy from 'speakeasy';
import crypto from 'crypto';
import { PrismaClient } from '@prisma/client';
import { Redis } from 'ioredis';
import { signAccessToken, signRefreshToken, verifyToken } from '../config/jwt';
import { EmailService } from './email.service';
import { SmsService } from './sms.service';
import type { AccessTokenPayload } from '../config/jwt';

export class AuthService {
  constructor(
    private readonly prisma: PrismaClient,
    private readonly redis: Redis,
    private readonly emailService: EmailService,
    private readonly smsService: SmsService
  ) {}

  async register(data: {
    email: string;
    username: string;
    password: string;
    displayName: string;
  }) {
    // ตรวจสอบ duplicate
    const existing = await this.prisma.user.findFirst({
      where: { OR: [{ email: data.email }, { username: data.username }] },
    });

    if (existing) {
      if (existing.email === data.email) {
        throw new Error('EMAIL_ALREADY_EXISTS');
      }
      throw new Error('USERNAME_ALREADY_EXISTS');
    }

    // Hash password
    const passwordHash = await bcrypt.hash(data.password, 12);

    // สร้าง email verification token
    const emailVerifyToken = crypto.randomBytes(32).toString('hex');
    const emailVerifyExpiry = new Date(Date.now() + 24 * 60 * 60 * 1000); // 24h

    // สร้าง user
    const user = await this.prisma.user.create({
      data: {
        email: data.email.toLowerCase(),
        username: data.username.toLowerCase(),
        passwordHash,
        displayName: data.displayName,
        emailVerifyToken,
        emailVerifyExpiry,
      },
    });

    // ส่ง verification email (async - ไม่ block response)
    this.emailService.sendVerificationEmail(user.email, emailVerifyToken).catch(console.error);

    return {
      userId: user.id,
      email: user.email,
      message: 'Registration successful. Please check your email to verify your account.',
    };
  }

  async login(data: {
    email: string;
    password: string;
    ipAddress: string;
    deviceInfo?: string;
    totpCode?: string;
  }) {
    const user = await this.prisma.user.findUnique({
      where: { email: data.email.toLowerCase() },
    });

    // ตรวจสอบ account lock
    if (user?.lockedUntil && user.lockedUntil > new Date()) {
      const remainingSeconds = Math.ceil((user.lockedUntil.getTime() - Date.now()) / 1000);
      throw new Error(`ACCOUNT_LOCKED:${remainingSeconds}`);
    }

    // ตรวจสอบ password
    if (!user || !user.passwordHash || !(await bcrypt.compare(data.password, user.passwordHash))) {
      // Track failed attempts
      if (user) {
        const newAttempts = user.loginAttempts + 1;
        const lockUntil = newAttempts >= 5 ? new Date(Date.now() + 15 * 60 * 1000) : null;
        
        await this.prisma.user.update({
          where: { id: user.id },
          data: {
            loginAttempts: newAttempts,
            lockedUntil: lockUntil,
          },
        });
      }
      throw new Error('INVALID_CREDENTIALS');
    }

    if (!user.isActive) {
      throw new Error('ACCOUNT_DISABLED');
    }

    // 2FA check
    if (user.twoFactorEnabled) {
      if (!data.totpCode) {
        return { requiresTwoFactor: true, userId: user.id };
      }

      const isValidTotp = speakeasy.totp.verify({
        secret: user.twoFactorSecret!,
        encoding: 'base32',
        token: data.totpCode,
        window: 2,
      });

      if (!isValidTotp) {
        throw new Error('INVALID_2FA_CODE');
      }
    }

    // Reset login attempts on success
    await this.prisma.user.update({
      where: { id: user.id },
      data: {
        loginAttempts: 0,
        lockedUntil: null,
        lastLoginAt: new Date(),
      },
    });

    // Issue tokens
    const accessToken = signAccessToken({
      id: user.id,
      email: user.email,
      username: user.username,
      isVerified: user.isVerified,
    });

    const refreshTokenValue = signRefreshToken(user.id);
    
    // Store refresh token hash (ไม่เก็บ token จริง)
    const tokenHash = crypto.createHash('sha256').update(refreshTokenValue).digest('hex');
    const decoded = verifyToken(refreshTokenValue) as any;
    
    await this.prisma.refreshToken.create({
      data: {
        userId: user.id,
        tokenHash,
        tokenFamily: decoded.tokenFamily,
        deviceInfo: data.deviceInfo,
        ipAddress: data.ipAddress,
        expiresAt: new Date(decoded.exp * 1000),
      },
    });

    return {
      accessToken,
      refreshToken: refreshTokenValue,
      requiresTwoFactor: false,
      user: {
        id: user.id,
        email: user.email,
        username: user.username,
        displayName: user.displayName,
        isVerified: user.isVerified,
      },
    };
  }

  async refreshTokens(refreshToken: string, ipAddress: string) {
    let decoded: any;
    try {
      decoded = verifyToken(refreshToken);
    } catch {
      throw new Error('INVALID_REFRESH_TOKEN');
    }

    if (decoded.type !== 'refresh') {
      throw new Error('INVALID_TOKEN_TYPE');
    }

    const tokenHash = crypto.createHash('sha256').update(refreshToken).digest('hex');
    
    // ตรวจสอบว่า token มีอยู่ใน DB
    const storedToken = await this.prisma.refreshToken.findUnique({
      where: { tokenHash },
      include: { user: true },
    });

    if (!storedToken || storedToken.revokedAt || storedToken.expiresAt < new Date()) {
      // Token reuse detected! Revoke ทั้ง family
      if (storedToken) {
        await this.prisma.refreshToken.updateMany({
          where: { tokenFamily: decoded.tokenFamily },
          data: { revokedAt: new Date() },
        });
      }
      throw new Error('REFRESH_TOKEN_REUSE_DETECTED');
    }

    // Revoke old token
    await this.prisma.refreshToken.update({
      where: { id: storedToken.id },
      data: { revokedAt: new Date() },
    });

    // Issue new tokens (same family)
    const newAccessToken = signAccessToken({
      id: storedToken.user.id,
      email: storedToken.user.email,
      username: storedToken.user.username,
      isVerified: storedToken.user.isVerified,
    });

    const newRefreshToken = signRefreshToken(storedToken.userId, decoded.tokenFamily);
    const newTokenHash = crypto.createHash('sha256').update(newRefreshToken).digest('hex');
    const newDecoded = verifyToken(newRefreshToken) as any;

    await this.prisma.refreshToken.create({
      data: {
        userId: storedToken.userId,
        tokenHash: newTokenHash,
        tokenFamily: decoded.tokenFamily,
        ipAddress,
        expiresAt: new Date(newDecoded.exp * 1000),
      },
    });

    return {
      accessToken: newAccessToken,
      refreshToken: newRefreshToken,
    };
  }

  async logout(userId: string, refreshToken: string) {
    const tokenHash = crypto.createHash('sha256').update(refreshToken).digest('hex');
    
    await this.prisma.refreshToken.updateMany({
      where: { userId, tokenHash },
      data: { revokedAt: new Date() },
    });
  }

  async introspectToken(token: string) {
    // ตรวจสอบ blacklist ใน Redis ก่อน
    const jti = (verifyToken(token) as AccessTokenPayload).jti;
    const isBlacklisted = await this.redis.exists(`blacklist:${jti}`);
    
    if (isBlacklisted) {
      return { isValid: false, reason: 'TOKEN_REVOKED' };
    }

    const payload = verifyToken(token) as AccessTokenPayload;
    
    if (payload.type !== 'access') {
      return { isValid: false, reason: 'WRONG_TOKEN_TYPE' };
    }

    return {
      isValid: true,
      userId: payload.sub,
      email: payload.email,
      username: payload.username,
      isVerified: payload.isVerified,
    };
  }

  async setup2FA(userId: string) {
    const secret = speakeasy.generateSecret({
      name: `chuaikan.com (${userId})`,
      length: 20,
    });

    // เก็บ secret ชั่วคราวใน Redis (user ต้อง verify ก่อนจะ activate)
    await this.redis.setex(
      `2fa:setup:${userId}`,
      600, // 10 minutes
      secret.base32
    );

    return {
      secret: secret.base32,
      qrCodeUrl: secret.otpauth_url,
      manualEntry: secret.base32,
    };
  }

  async verify2FA(userId: string, totpCode: string) {
    const secret = await this.redis.get(`2fa:setup:${userId}`);
    
    if (!secret) {
      throw new Error('2FA_SETUP_EXPIRED');
    }

    const isValid = speakeasy.totp.verify({
      secret,
      encoding: 'base32',
      token: totpCode,
      window: 2,
    });

    if (!isValid) {
      throw new Error('INVALID_TOTP_CODE');
    }

    // Generate backup codes
    const backupCodes = Array.from({ length: 8 }, () =>
      crypto.randomBytes(4).toString('hex').toUpperCase()
    );
    const hashedBackupCodes = await Promise.all(
      backupCodes.map((code) => bcrypt.hash(code, 10))
    );

    // Activate 2FA
    await this.prisma.user.update({
      where: { id: userId },
      data: {
        twoFactorSecret: secret,
        twoFactorEnabled: true,
        backupCodes: hashedBackupCodes,
      },
    });

    // Cleanup Redis
    await this.redis.del(`2fa:setup:${userId}`);

    return {
      message: '2FA enabled successfully',
      backupCodes, // แสดงให้ user เห็นครั้งเดียว
    };
  }

  async sendSmsOtp(userId: string, phoneNumber: string) {
    const otp = Math.floor(100000 + Math.random() * 900000).toString();
    
    // เก็บ OTP ใน Redis (5 นาที)
    await this.redis.setex(`sms:otp:${userId}`, 300, otp);
    
    await this.smsService.send(phoneNumber, `รหัส OTP chuaikan.com: ${otp} (หมดอายุใน 5 นาที)`);
    
    return { message: 'OTP sent successfully' };
  }

  async verifyEmailToken(token: string) {
    const user = await this.prisma.user.findFirst({
      where: {
        emailVerifyToken: token,
        emailVerifyExpiry: { gt: new Date() },
      },
    });

    if (!user) {
      throw new Error('INVALID_OR_EXPIRED_TOKEN');
    }

    await this.prisma.user.update({
      where: { id: user.id },
      data: {
        isVerified: true,
        emailVerifyToken: null,
        emailVerifyExpiry: null,
      },
    });

    return { message: 'Email verified successfully' };
  }

  async requestPasswordReset(email: string) {
    const user = await this.prisma.user.findUnique({ where: { email } });
    
    // ไม่บอกว่า email มีอยู่หรือไม่ (security)
    if (!user) {
      return { message: 'If that email exists, a reset link has been sent.' };
    }

    const resetToken = crypto.randomBytes(32).toString('hex');
    const resetExpiry = new Date(Date.now() + 60 * 60 * 1000); // 1 hour

    await this.prisma.user.update({
      where: { id: user.id },
      data: {
        passwordResetToken: resetToken,
        passwordResetExpiry: resetExpiry,
      },
    });

    await this.emailService.sendPasswordResetEmail(user.email, resetToken);
    
    return { message: 'If that email exists, a reset link has been sent.' };
  }
}
```

### Step 228: OAuth 2.0 - LINE Login

```typescript
// apps/auth-service/src/config/passport.ts

import passport from 'passport';
import { Strategy as GoogleStrategy } from 'passport-google-oauth20';
import { PrismaClient } from '@prisma/client';
import { env } from './env';
import axios from 'axios';

export function configurePassport(prisma: PrismaClient) {
  // Google OAuth
  passport.use(
    new GoogleStrategy(
      {
        clientID: env.GOOGLE_CLIENT_ID,
        clientSecret: env.GOOGLE_CLIENT_SECRET,
        callbackURL: env.GOOGLE_CALLBACK_URL,
      },
      async (accessToken, refreshToken, profile, done) => {
        try {
          const email = profile.emails?.[0]?.value;
          if (!email) {
            return done(new Error('No email from Google'));
          }

          // ค้นหา existing OAuth account
          let oauthAccount = await prisma.oAuthAccount.findUnique({
            where: { provider_providerId: { provider: 'google', providerId: profile.id } },
            include: { user: true },
          });

          if (oauthAccount) {
            return done(null, oauthAccount.user);
          }

          // ค้นหา user จาก email
          let user = await prisma.user.findUnique({ where: { email } });

          if (!user) {
            // สร้าง user ใหม่
            const username = `${profile.displayName?.replace(/\s+/g, '').toLowerCase()}_${Date.now().toString(36)}`;
            user = await prisma.user.create({
              data: {
                email,
                username,
                displayName: profile.displayName || username,
                avatarUrl: profile.photos?.[0]?.value,
                isVerified: true, // Google verified email
              },
            });
          }

          // สร้าง OAuth account record
          await prisma.oAuthAccount.create({
            data: {
              userId: user.id,
              provider: 'google',
              providerId: profile.id,
              accessToken,
              refreshToken,
              profileData: profile._json,
            },
          });

          return done(null, user);
        } catch (error) {
          return done(error as Error);
        }
      }
    )
  );

  return passport;
}

// LINE OAuth (manual implementation - passport-line ไม่ active maintain)
export async function exchangeLineCode(code: string, redirectUri: string) {
  // Exchange code for tokens
  const tokenResponse = await axios.post(
    'https://api.line.me/oauth2/v2.1/token',
    new URLSearchParams({
      grant_type: 'authorization_code',
      code,
      redirect_uri: redirectUri,
      client_id: env.LINE_CHANNEL_ID,
      client_secret: env.LINE_CHANNEL_SECRET,
    }),
    { headers: { 'Content-Type': 'application/x-www-form-urlencoded' } }
  );

  const { access_token, id_token } = tokenResponse.data;

  // Get user profile
  const profileResponse = await axios.get('https://api.line.me/v2/profile', {
    headers: { Authorization: `Bearer ${access_token}` },
  });

  return {
    lineUserId: profileResponse.data.userId,
    displayName: profileResponse.data.displayName,
    pictureUrl: profileResponse.data.pictureUrl,
    email: null, // LINE ไม่ return email ใน free tier
    accessToken: access_token,
  };
}
```

### Step 229: Routes และ Rate Limiting

```typescript
// apps/auth-service/src/routes/auth.routes.ts

import { Router } from 'express';
import rateLimit from 'express-rate-limit';
import { AuthController } from '../controllers/auth.controller';
import { authenticate } from '../middleware/authenticate';

export function createAuthRouter(controller: AuthController): Router {
  const router = Router();

  // Rate limiters
  const loginLimiter = rateLimit({
    windowMs: 15 * 60 * 1000,  // 15 minutes
    limit: 5,                   // 5 attempts per 15 min
    message: {
      success: false,
      error: {
        code: 'TOO_MANY_LOGIN_ATTEMPTS',
        message: 'Too many login attempts. Please try again after 15 minutes.',
        retryAfter: 900,
      },
    },
    standardHeaders: true,
    legacyHeaders: false,
    keyGenerator: (req) => `${req.ip}-${req.body?.email || 'unknown'}`,
  });

  const registerLimiter = rateLimit({
    windowMs: 60 * 60 * 1000,  // 1 hour
    limit: 3,                   // 3 registrations per hour per IP
    message: {
      success: false,
      error: { code: 'TOO_MANY_REGISTRATIONS', message: 'Too many registration attempts.' },
    },
  });

  const forgotPasswordLimiter = rateLimit({
    windowMs: 60 * 60 * 1000,
    limit: 3,
    message: {
      success: false,
      error: { code: 'TOO_MANY_RESET_REQUESTS', message: 'Too many password reset requests.' },
    },
  });

  // Public routes
  router.post('/register', registerLimiter, controller.register);
  router.post('/login', loginLimiter, controller.login);
  router.post('/refresh', controller.refresh);
  router.post('/forgot-password', forgotPasswordLimiter, controller.requestPasswordReset);
  router.post('/reset-password', controller.resetPassword);
  router.get('/verify-email/:token', controller.verifyEmail);

  // OAuth routes
  router.get('/google', passport.authenticate('google', { scope: ['profile', 'email'] }));
  router.get('/google/callback', passport.authenticate('google'), controller.oauthCallback);
  
  router.get('/line', controller.lineOAuthRedirect);
  router.get('/line/callback', controller.lineOAuthCallback);

  // Protected routes (require valid JWT)
  router.post('/logout', authenticate, controller.logout);
  router.get('/me', authenticate, controller.getMe);
  router.post('/2fa/setup', authenticate, controller.setup2FA);
  router.post('/2fa/verify', authenticate, controller.verify2FA);
  router.delete('/2fa', authenticate, controller.disable2FA);
  router.get('/devices', authenticate, controller.getDevices);
  router.delete('/devices/:deviceId', authenticate, controller.revokeDevice);

  // Internal routes (service-to-service only)
  router.post('/internal/introspect', validateServiceToken, controller.introspect);
  router.get('/internal/public-key', controller.getPublicKey);

  return router;
}
```

---

## 🔧 Configuration Files

### Dockerfile สำหรับ Auth Service

```dockerfile
# apps/auth-service/Dockerfile
FROM node:22-alpine AS base

# Install pnpm
RUN npm install -g pnpm@9

# Builder stage
FROM base AS builder
WORKDIR /app

# Copy package files
COPY pnpm-workspace.yaml ./
COPY package.json pnpm-lock.yaml ./
COPY apps/auth-service/package.json ./apps/auth-service/
COPY packages/shared-types/package.json ./packages/shared-types/
COPY packages/utils/package.json ./packages/utils/

# Install dependencies
RUN pnpm install --frozen-lockfile

# Copy source
COPY packages/shared-types ./packages/shared-types
COPY packages/utils ./packages/utils
COPY apps/auth-service ./apps/auth-service

# Generate Prisma client
WORKDIR /app/apps/auth-service
RUN npx prisma generate

# Build
RUN pnpm --filter=@chuaikan/auth-service build

# Production stage
FROM node:22-alpine AS production
WORKDIR /app

RUN addgroup -g 1001 -S nodejs && adduser -S nodeapp -u 1001
RUN apk add --no-cache openssl  # สำหรับ Prisma

COPY --from=builder --chown=nodeapp:nodejs /app/apps/auth-service/dist ./dist
COPY --from=builder --chown=nodeapp:nodejs /app/apps/auth-service/node_modules ./node_modules
COPY --from=builder --chown=nodeapp:nodejs /app/apps/auth-service/prisma ./prisma

USER nodeapp

EXPOSE 3001

HEALTHCHECK --interval=10s --timeout=5s --start-period=30s --retries=3 \
  CMD wget -q --spider http://localhost:3001/health || exit 1

CMD ["node", "dist/index.js"]
```

---

## 🧪 Testing

### Unit Tests สำหรับ Auth Service

```typescript
// apps/auth-service/src/__tests__/auth.service.test.ts

import { describe, it, expect, vi, beforeEach } from 'vitest';
import { AuthService } from '../services/auth.service';
import bcrypt from 'bcryptjs';

// Mock dependencies
const mockPrisma = {
  user: {
    findFirst: vi.fn(),
    findUnique: vi.fn(),
    create: vi.fn(),
    update: vi.fn(),
    updateMany: vi.fn(),
  },
  refreshToken: {
    create: vi.fn(),
    findUnique: vi.fn(),
    update: vi.fn(),
    updateMany: vi.fn(),
  },
};

const mockRedis = {
  setex: vi.fn().mockResolvedValue('OK'),
  get: vi.fn(),
  del: vi.fn().mockResolvedValue(1),
  exists: vi.fn().mockResolvedValue(0),
};

const mockEmailService = {
  sendVerificationEmail: vi.fn().mockResolvedValue(true),
  sendPasswordResetEmail: vi.fn().mockResolvedValue(true),
};

const mockSmsService = {
  send: vi.fn().mockResolvedValue(true),
};

describe('AuthService', () => {
  let authService: AuthService;

  beforeEach(() => {
    vi.clearAllMocks();
    authService = new AuthService(
      mockPrisma as any,
      mockRedis as any,
      mockEmailService as any,
      mockSmsService as any
    );
  });

  describe('register', () => {
    it('should create user and send verification email', async () => {
      mockPrisma.user.findFirst.mockResolvedValue(null);
      mockPrisma.user.create.mockResolvedValue({
        id: 'user-123',
        email: 'test@chuaikan.com',
        username: 'testuser',
      });

      const result = await authService.register({
        email: 'Test@Chuaikan.com',
        username: 'testuser',
        password: 'SecurePass123!',
        displayName: 'Test User',
      });

      expect(result.userId).toBe('user-123');
      expect(mockEmailService.sendVerificationEmail).toHaveBeenCalledOnce();
    });

    it('should throw EMAIL_ALREADY_EXISTS when email is taken', async () => {
      mockPrisma.user.findFirst.mockResolvedValue({
        id: 'existing-user',
        email: 'test@chuaikan.com',
      });

      await expect(
        authService.register({
          email: 'test@chuaikan.com',
          username: 'newuser',
          password: 'SecurePass123!',
          displayName: 'New User',
        })
      ).rejects.toThrow('EMAIL_ALREADY_EXISTS');
    });
  });

  describe('login', () => {
    it('should return tokens on successful login', async () => {
      const passwordHash = await bcrypt.hash('correct-password', 12);
      
      mockPrisma.user.findUnique.mockResolvedValue({
        id: 'user-123',
        email: 'test@chuaikan.com',
        username: 'testuser',
        passwordHash,
        isVerified: true,
        isActive: true,
        loginAttempts: 0,
        lockedUntil: null,
        twoFactorEnabled: false,
        displayName: 'Test User',
      });
      mockPrisma.user.update.mockResolvedValue({});
      mockPrisma.refreshToken.create.mockResolvedValue({});

      const result = await authService.login({
        email: 'test@chuaikan.com',
        password: 'correct-password',
        ipAddress: '127.0.0.1',
      });

      expect(result.accessToken).toBeDefined();
      expect(result.refreshToken).toBeDefined();
      expect(result.requiresTwoFactor).toBe(false);
    });

    it('should lock account after 5 failed attempts', async () => {
      mockPrisma.user.findUnique.mockResolvedValue({
        id: 'user-123',
        email: 'test@chuaikan.com',
        passwordHash: 'wrong-hash',
        isActive: true,
        loginAttempts: 4,  // 4 previous failures
        lockedUntil: null,
        twoFactorEnabled: false,
      });
      mockPrisma.user.update.mockResolvedValue({});

      await expect(
        authService.login({
          email: 'test@chuaikan.com',
          password: 'wrong-password',
          ipAddress: '127.0.0.1',
        })
      ).rejects.toThrow('INVALID_CREDENTIALS');

      // Should lock account (5th attempt)
      expect(mockPrisma.user.update).toHaveBeenCalledWith({
        where: { id: 'user-123' },
        data: expect.objectContaining({
          loginAttempts: 5,
          lockedUntil: expect.any(Date),
        }),
      });
    });

    it('should require 2FA code when enabled', async () => {
      const passwordHash = await bcrypt.hash('correct-password', 12);
      
      mockPrisma.user.findUnique.mockResolvedValue({
        id: 'user-123',
        email: 'test@chuaikan.com',
        passwordHash,
        isActive: true,
        loginAttempts: 0,
        lockedUntil: null,
        twoFactorEnabled: true,
        twoFactorSecret: 'JBSWY3DPEHPK3PXP',
      });

      const result = await authService.login({
        email: 'test@chuaikan.com',
        password: 'correct-password',
        ipAddress: '127.0.0.1',
      });

      expect(result.requiresTwoFactor).toBe(true);
    });
  });

  describe('refreshTokens', () => {
    it('should detect token reuse and revoke family', async () => {
      // ทดสอบ refresh token rotation security
      const { signRefreshToken } = await import('../config/jwt');
      const validToken = signRefreshToken('user-123');
      
      // Token ถูก revoke แล้ว (จำลองการ reuse)
      mockPrisma.refreshToken.findUnique.mockResolvedValue({
        id: 'token-id',
        userId: 'user-123',
        tokenFamily: 'family-123',
        revokedAt: new Date(), // Already revoked!
        expiresAt: new Date(Date.now() + 86400000),
        user: {},
      });
      mockPrisma.refreshToken.updateMany.mockResolvedValue({});

      await expect(
        authService.refreshTokens(validToken, '127.0.0.1')
      ).rejects.toThrow('REFRESH_TOKEN_REUSE_DETECTED');

      // ต้อง revoke ทั้ง family
      expect(mockPrisma.refreshToken.updateMany).toHaveBeenCalled();
    });
  });
});
```

---

## ❌ Common Errors & Solutions

### Error 1: "JWT malformed" หรือ "invalid signature"

```bash
# ตรวจสอบ public/private key pair
openssl rsa -in keys/private.pem -pubout -out /tmp/test-public.pem
diff keys/public.pem /tmp/test-public.pem
# ถ้า diff มีผล = keys ไม่ match

# สร้าง keys ใหม่
openssl genrsa -out keys/private.pem 2048
openssl rsa -in keys/private.pem -pubout -out keys/public.pem

# ทดสอบ sign/verify
node -e "
const jwt = require('jsonwebtoken');
const fs = require('fs');
const priv = fs.readFileSync('keys/private.pem');
const pub = fs.readFileSync('keys/public.pem');
const token = jwt.sign({sub: 'test'}, priv, {algorithm: 'RS256'});
const decoded = jwt.verify(token, pub);
console.log('SUCCESS:', decoded);
"
```

### Error 2: "Too many Redis connections"

```bash
# ตรวจสอบ Redis connections
redis-cli -a chuaikan_redis_dev CLIENT LIST | wc -l

# แก้ไข: ใช้ connection pool
# ใน auth service config ให้ใช้ IORedis cluster หรือ pool
# อย่าสร้าง new Redis() ใน request handler

# ตรวจสอบ connection leak
redis-cli -a chuaikan_redis_dev INFO clients
```

### Error 3: Prisma "P2025 Record not found" ใน refresh token

```bash
# สาเหตุ: Race condition เมื่อ multiple requests ส่ง refresh token พร้อมกัน
# แก้ไข: ใช้ database transaction

# ใน refreshTokens method:
await prisma.$transaction(async (tx) => {
  const token = await tx.refreshToken.findUnique({ where: { tokenHash } });
  if (!token) throw new Error('INVALID_REFRESH_TOKEN');
  await tx.refreshToken.update({ where: { id: token.id }, data: { revokedAt: new Date() } });
  // ... create new token
});
```

### Error 4: LINE OAuth callback error

```bash
# LINE OAuth ต้องการ exact callback URL match
# ตรวจสอบใน LINE Developers Console:
# Callback URL ต้องตรงกับ env.LINE_CALLBACK_URL

# Debug:
curl -v "https://api.line.me/oauth2/v2.1/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=authorization_code&code=TEST_CODE&redirect_uri=http://localhost:3001/api/v1/auth/line/callback&client_id=YOUR_CHANNEL_ID&client_secret=YOUR_CHANNEL_SECRET"
```

---

## ✅ Checklist

### Step 221-222: Architecture
- [ ] เข้าใจ responsibilities ของ Auth Service
- [ ] เข้าใจ RS256 vs HS256
- [ ] สร้าง RSA key pair สำเร็จ

### Step 223-224: Project Setup
- [ ] สร้าง auth-service directory structure
- [ ] ติดตั้ง dependencies สำเร็จ
- [ ] Prisma schema ครบถ้วน

### Step 225: Database
- [ ] `prisma migrate dev` สำเร็จ
- [ ] Tables ครบ: User, RefreshToken, OAuthAccount, RevokedToken
- [ ] Indexes ถูกสร้าง

### Step 226-227: Core Logic
- [ ] Register สร้าง user และส่ง email verification
- [ ] Login ตรวจสอบ password และออก JWT
- [ ] Account lock หลัง 5 attempts ผิด
- [ ] Refresh token rotation ทำงาน
- [ ] Token reuse detection ทำงาน

### Step 228: OAuth
- [ ] Google OAuth callback ทำงาน
- [ ] LINE OAuth exchange code ทำงาน
- [ ] OAuth user สร้าง account ได้

### Step 229: Security Features
- [ ] Rate limiting: 5 login attempts/15 min
- [ ] 2FA TOTP setup และ verify
- [ ] Email verification flow
- [ ] Password reset flow

### Step 230: Testing & Deployment
- [ ] Unit tests ผ่านทั้งหมด
- [ ] Integration tests ผ่าน
- [ ] Dockerfile build สำเร็จ
- [ ] Health check endpoint `/health` ตอบสนอง
- [ ] Token introspection endpoint ทำงาน

---

## 🔗 References

- [JWT Best Practices (RFC 8725)](https://www.rfc-editor.org/rfc/rfc8725)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [speakeasy TOTP](https://github.com/speakeasyjs/speakeasy)
- [LINE Login Documentation](https://developers.line.biz/en/docs/line-login/)
- [Prisma Documentation](https://www.prisma.io/docs)
- [Refresh Token Rotation](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation)

---
*Part 023 | Road to 1,000,000 Users/Day | chuaikan.com*
