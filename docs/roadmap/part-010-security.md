# Part 010: Security พื้นฐาน
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Beginner
> **Steps:** 91-100
> **เวลาโดยประมาณ:** 6 ชั่วโมง
> **Prerequisites:** Part 001-009 (Linux, Git, Node.js, PostgreSQL, Redis, Docker, Cloudflare, CI/CD, Monitoring)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

1. เข้าใจ OWASP Top 10 2021 กับตัวอย่างจริงจาก chuaikan.com
2. ตั้งค่า HTTPS ด้วย Let's Encrypt + Certbot
3. ตั้งค่า HTTP Security Headers
4. ใช้งาน Helmet.js สำหรับ Express/Node.js
5. ป้องกัน SQL Injection ด้วย Parameterized Queries (Prisma)
6. ป้องกัน XSS ด้วย DOMPurify และ sanitize-html
7. ป้องกัน CSRF attacks
8. JWT Best Practices
9. Password Hashing ด้วย argon2
10. จัดการ Environment Variables ด้วย zod validation

---

## 📖 ทฤษฎีและแนวคิด

### OWASP Top 10 2021

OWASP (Open Web Application Security Project) เป็นองค์กรที่รวบรวมช่องโหว่ที่พบบ่อยที่สุดในเว็บแอปพลิเคชัน อัพเดทล่าสุดปี 2021:

```
A01:2021 — Broken Access Control          (ขาดการควบคุมการเข้าถึง)
A02:2021 — Cryptographic Failures         (การเข้ารหัสผิดพลาด)
A03:2021 — Injection                      (SQL Injection, XSS, etc.)
A04:2021 — Insecure Design                (ออกแบบไม่ปลอดภัย)
A05:2021 — Security Misconfiguration      (ตั้งค่าผิด)
A06:2021 — Vulnerable and Outdated Components (ใช้ library เก่า)
A07:2021 — Identification & Authentication Failures (Auth ผิดพลาด)
A08:2021 — Software and Data Integrity Failures
A09:2021 — Security Logging Failures      (log ไม่ครบ)
A10:2021 — Server-Side Request Forgery    (SSRF)
```

**ตัวอย่างที่อาจเกิดกับ chuaikan.com:**

```
A01 - Broken Access Control:
  ❌ User A เข้าถึงโปรไฟล์ User B ได้ผ่าน /api/users/123
  ✅ ต้องตรวจสอบว่า request มาจาก user ที่มีสิทธิ์

A02 - Cryptographic Failures:
  ❌ เก็บ password เป็น MD5 หรือ plain text
  ✅ ใช้ argon2id สำหรับ password hashing

A03 - Injection:
  ❌ SELECT * FROM users WHERE email = '${email}'
  ✅ ใช้ parameterized queries / ORM

A05 - Security Misconfiguration:
  ❌ เปิด /api/admin ให้ทุกคน access ได้
  ✅ ป้องกันด้วย auth middleware + IP allowlist

A07 - Authentication Failures:
  ❌ ไม่มี rate limit สำหรับ login (brute force)
  ✅ Rate limit 10 attempts/min, lockout หลัง 5 ครั้ง
```

---

## ⚙️ Environment Setup

### Step 91: HTTPS Setup ด้วย Let's Encrypt

```bash
# ติดตั้ง Certbot และ Nginx plugin
sudo apt-get update
sudo apt-get install -y certbot python3-certbot-nginx

# ตรวจสอบว่า Nginx ทำงานอยู่
sudo systemctl status nginx

# ขอ certificate สำหรับ chuaikan.com
sudo certbot --nginx \
    -d chuaikan.com \
    -d www.chuaikan.com \
    -d api.chuaikan.com \
    --non-interactive \
    --agree-tos \
    --email security@chuaikan.com \
    --redirect \
    --hsts \
    --staple-ocsp

# ตรวจสอบ certificates
sudo certbot certificates

# Expected output:
# Found the following certs:
#   Certificate Name: chuaikan.com
#     Domains: chuaikan.com www.chuaikan.com api.chuaikan.com
#     Expiry Date: 2025-03-01 (VALID: 89 days)
#     Certificate Path: /etc/letsencrypt/live/chuaikan.com/fullchain.pem

# ทดสอบ auto-renewal
sudo certbot renew --dry-run
# Expected: Congratulations, all simulated renewals succeeded

# ตั้ง cron สำหรับ auto-renewal (certbot ตั้งให้อัตโนมัติ)
sudo cat /etc/cron.d/certbot
# หรือตรวจสอบ systemd timer
sudo systemctl status certbot.timer
```

ตั้งค่า Nginx สำหรับ HTTPS:

```nginx
# /etc/nginx/sites-available/chuaikan.com
server {
    listen 443 ssl http2;
    server_name chuaikan.com www.chuaikan.com;

    # SSL Certificates
    ssl_certificate /etc/letsencrypt/live/chuaikan.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/chuaikan.com/privkey.pem;
    
    # SSL Configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;
    ssl_stapling on;
    ssl_stapling_verify on;

    # ================================
    # Security Headers
    # ================================
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;

    # Content Security Policy (ปรับตาม app ของคุณ)
    add_header Content-Security-Policy "
        default-src 'self';
        script-src 'self' 'unsafe-inline' 'unsafe-eval' https://cdnjs.cloudflare.com;
        style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
        font-src 'self' https://fonts.gstatic.com;
        img-src 'self' data: https://assets.chuaikan.com;
        connect-src 'self' https://api.chuaikan.com wss://api.chuaikan.com;
        frame-ancestors 'none';
        base-uri 'self';
        form-action 'self';
    " always;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        
        # Timeouts
        proxy_connect_timeout 30s;
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;
    }
}

server {
    listen 80;
    server_name chuaikan.com www.chuaikan.com;
    return 301 https://$host$request_uri;
}
```

---

## 🛠️ Step-by-Step Implementation

### Step 92: Helmet.js สำหรับ Node.js/Express

```bash
npm install helmet
```

```typescript
// apps/api/src/middleware/security.ts
import helmet from 'helmet';
import { Express } from 'express';

export function setupSecurityMiddleware(app: Express): void {
  // Helmet ตั้งค่า security headers หลายตัวในครั้งเดียว
  app.use(helmet({
    // Content Security Policy
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'"],
        styleSrc: ["'self'", "https://fonts.googleapis.com"],
        fontSrc: ["'self'", "https://fonts.gstatic.com"],
        imgSrc: ["'self'", "data:", "https://assets.chuaikan.com"],
        connectSrc: ["'self'"],
        frameSrc: ["'none'"],
        objectSrc: ["'none'"],
        upgradeInsecureRequests: [],
      },
    },
    
    // HTTP Strict Transport Security
    hsts: {
      maxAge: 31536000,          // 1 ปี
      includeSubDomains: true,
      preload: true,
    },
    
    // ป้องกัน MIME type sniffing
    noSniff: true,
    
    // ป้องกัน clickjacking
    frameguard: { action: 'deny' },
    
    // ป้องกัน IE execute downloads
    ieNoOpen: true,
    
    // ซ่อน X-Powered-By header
    hidePoweredBy: true,
    
    // XSS Filter (legacy browsers)
    xssFilter: true,
    
    // Referrer Policy
    referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
    
    // Cross-Origin Resource Policy
    crossOriginResourcePolicy: { policy: 'same-site' },
    
    // Cross-Origin Embedder Policy
    crossOriginEmbedderPolicy: { policy: 'require-corp' },
    
    // Permissions Policy
    permissionsPolicy: {
      features: {
        geolocation: [],
        microphone: [],
        camera: [],
        payment: [],
      },
    },
  }));
}
```

```typescript
// apps/api/src/index.ts
import express from 'express';
import { setupSecurityMiddleware } from './middleware/security';
import cors from 'cors';
import rateLimit from 'express-rate-limit';

const app = express();

// 1. Security headers (first middleware)
setupSecurityMiddleware(app);

// 2. CORS Configuration
const corsOptions = {
  origin: (origin: string | undefined, callback: Function) => {
    const allowedOrigins = [
      'https://chuaikan.com',
      'https://www.chuaikan.com',
      process.env.NODE_ENV === 'development' ? 'http://localhost:3000' : null,
    ].filter(Boolean);
    
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-CSRF-Token'],
  maxAge: 86400,  // preflight cache 24 ชั่วโมง
};

app.use(cors(corsOptions));

// 3. Rate Limiting
const limiter = rateLimit({
  windowMs: 60 * 1000,  // 1 นาที
  max: 100,
  message: { error: 'Too many requests, please try again later.' },
  standardHeaders: true,
  legacyHeaders: false,
  skip: (req) => {
    // ไม่ rate limit สำหรับ health check
    return req.path === '/health';
  },
});

app.use('/api/', limiter);

// Stricter limit สำหรับ auth endpoints
const authLimiter = rateLimit({
  windowMs: 60 * 1000,
  max: 10,
  message: { error: 'Too many login attempts, please try again later.' },
  skipSuccessfulRequests: true,
});

app.use('/api/auth/', authLimiter);
```

### Step 93: SQL Injection Prevention ด้วย Prisma

```bash
npm install @prisma/client
npm install -D prisma
npx prisma init
```

```typescript
// apps/api/src/services/user.service.ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient({
  log: ['error', 'warn'],
});

// ✅ SAFE: Prisma ใช้ parameterized queries อัตโนมัติ
export async function getUserByEmail(email: string) {
  // Prisma จะสร้าง SQL: SELECT * FROM users WHERE email = $1
  // และส่ง email เป็น parameter แยกต่างหาก (ไม่ใส่ใน SQL string)
  return await prisma.user.findUnique({
    where: { email },
    select: {
      id: true,
      email: true,
      name: true,
      role: true,
      // ไม่รวม passwordHash
    },
  });
}

// ✅ SAFE: Search ด้วย contains
export async function searchUsers(query: string) {
  return await prisma.user.findMany({
    where: {
      OR: [
        { name: { contains: query, mode: 'insensitive' } },
        { email: { contains: query, mode: 'insensitive' } },
      ],
    },
    take: 20,
  });
}

// ✅ SAFE: Raw query กับ parameterized inputs
export async function getUserStats(userId: string) {
  // ใช้ $queryRaw เมื่อต้องการ raw SQL แต่ยังปลอดภัย
  const stats = await prisma.$queryRaw<[{ count: bigint }]>`
    SELECT COUNT(*) as count
    FROM orders
    WHERE user_id = ${userId}
    AND created_at > NOW() - INTERVAL '30 days'
  `;
  return Number(stats[0].count);
}

// ❌ UNSAFE: อย่าทำแบบนี้!
export async function unsafeSearch(query: string) {
  // SQL Injection ได้! ถ้า query = "'; DROP TABLE users; --"
  const result = await prisma.$queryRawUnsafe(
    `SELECT * FROM users WHERE name LIKE '%${query}%'`  // อันตราย!
  );
  return result;
}
```

```typescript
// Input validation ด้วย zod
import { z } from 'zod';

const createUserSchema = z.object({
  email: z.string().email().max(255),
  name: z.string().min(2).max(100).regex(/^[\w\s฀-๿]+$/, 'Invalid characters'),
  password: z.string().min(8).max(128),
});

export async function createUser(input: unknown) {
  // Validate และ sanitize ก่อน ทุกครั้ง
  const data = createUserSchema.parse(input);  // throws ถ้า invalid
  
  return await prisma.user.create({
    data: {
      email: data.email.toLowerCase(),
      name: data.name.trim(),
      passwordHash: await hashPassword(data.password),
    },
  });
}
```

### Step 94: XSS Prevention

```bash
# Server-side: sanitize HTML input
npm install sanitize-html
npm install @types/sanitize-html --save-dev

# Client-side: sanitize HTML output
npm install dompurify
npm install @types/dompurify --save-dev
```

```typescript
// apps/api/src/utils/sanitize.ts
import sanitizeHtml from 'sanitize-html';
import { z } from 'zod';

// ตั้งค่า allowed HTML tags และ attributes
const sanitizeOptions: sanitizeHtml.IOptions = {
  allowedTags: [
    'p', 'br', 'strong', 'em', 'u', 'ul', 'ol', 'li',
    'h1', 'h2', 'h3', 'h4', 'a', 'blockquote', 'code', 'pre',
  ],
  allowedAttributes: {
    'a': ['href', 'target', 'rel'],
    '*': ['class'],
  },
  // ป้องกัน javascript: URLs
  allowedSchemes: ['http', 'https', 'mailto'],
  // เพิ่ม noopener noreferrer ให้ links อัตโนมัติ
  transformTags: {
    'a': sanitizeHtml.simpleTransform('a', {
      target: '_blank',
      rel: 'noopener noreferrer nofollow',
    }),
  },
  // ไม่อนุญาต inline styles
  allowedStyles: {},
};

export function sanitizeUserContent(html: string): string {
  return sanitizeHtml(html, sanitizeOptions);
}

// สำหรับ plain text (ไม่มี HTML เลย)
export function sanitizePlainText(text: string): string {
  return sanitizeHtml(text, { allowedTags: [], allowedAttributes: {} });
}

// ตัวอย่างการใช้งาน
export async function createPost(userId: string, input: {
  title: string;
  content: string;
}) {
  return await prisma.post.create({
    data: {
      userId,
      title: sanitizePlainText(input.title),    // title ไม่ควรมี HTML
      content: sanitizeUserContent(input.content), // content อนุญาต safe HTML
    },
  });
}
```

```tsx
// apps/web/src/components/SafeHtml.tsx (Client-side)
'use client';

import DOMPurify from 'dompurify';
import { useMemo } from 'react';

interface SafeHtmlProps {
  html: string;
  className?: string;
}

// DOMPurify config
const DOMPURIFY_CONFIG: DOMPurify.Config = {
  ALLOWED_TAGS: ['p', 'br', 'strong', 'em', 'u', 'ul', 'ol', 'li', 'a', 'h1', 'h2', 'h3'],
  ALLOWED_ATTR: ['href', 'target', 'rel', 'class'],
  ALLOW_DATA_ATTR: false,
  FORCE_BODY: true,
};

export function SafeHtml({ html, className }: SafeHtmlProps) {
  const sanitized = useMemo(() => {
    // DOMPurify ทำงานได้เฉพาะ browser (ไม่ใช่ SSR)
    if (typeof window === 'undefined') {
      return html.replace(/<[^>]*>/g, '');  // Strip tags ใน SSR
    }
    return DOMPurify.sanitize(html, DOMPURIFY_CONFIG);
  }, [html]);

  return (
    <div
      className={className}
      dangerouslySetInnerHTML={{ __html: sanitized }}
    />
  );
}
```

### Step 95: CSRF Protection

```bash
npm install csrf-csrf
```

```typescript
// apps/api/src/middleware/csrf.ts
import { doubleCsrf } from 'csrf-csrf';
import { Request, Response, NextFunction } from 'express';

const { generateToken, doubleCsrfProtection } = doubleCsrf({
  getSecret: (req) => process.env.CSRF_SECRET!,
  cookieName: '__Host-psifi.x-csrf-token',
  cookieOptions: {
    sameSite: 'strict',
    path: '/',
    secure: process.env.NODE_ENV === 'production',
    httpOnly: true,
  },
  size: 64,
  getTokenFromRequest: (req) => {
    return req.headers['x-csrf-token'] as string || req.body?._csrf;
  },
});

// Endpoint สำหรับ client ขอ CSRF token
export function getCsrfTokenHandler(req: Request, res: Response): void {
  const token = generateToken(req, res);
  res.json({ csrfToken: token });
}

// Middleware สำหรับ protect routes
export { doubleCsrfProtection };
```

```typescript
// ใน index.ts
import { getCsrfTokenHandler, doubleCsrfProtection } from './middleware/csrf';

// Route สำหรับดึง CSRF token
app.get('/api/auth/csrf', getCsrfTokenHandler);

// ป้องกัน mutation routes ด้วย CSRF
app.use('/api/', doubleCsrfProtection);

// ยกเว้น webhook endpoints
app.use('/api/webhooks/', (req, res, next) => {
  // webhooks ใช้ signature verification แทน CSRF
  next();
});
```

```typescript
// apps/web/src/hooks/useCsrf.ts (Client-side)
'use client';

import { useState, useEffect } from 'react';

export function useCsrf() {
  const [csrfToken, setCsrfToken] = useState<string>('');

  useEffect(() => {
    // ดึง CSRF token เมื่อ component mount
    fetch('/api/auth/csrf', { credentials: 'include' })
      .then(res => res.json())
      .then(data => setCsrfToken(data.csrfToken))
      .catch(console.error);
  }, []);

  // Helper สำหรับ fetch ที่มี CSRF token
  const fetchWithCsrf = (url: string, options: RequestInit = {}) => {
    return fetch(url, {
      ...options,
      credentials: 'include',
      headers: {
        ...options.headers,
        'X-CSRF-Token': csrfToken,
        'Content-Type': 'application/json',
      },
    });
  };

  return { csrfToken, fetchWithCsrf };
}
```

### Step 96: JWT Best Practices

```bash
npm install jsonwebtoken
npm install @types/jsonwebtoken --save-dev
```

```typescript
// apps/api/src/services/auth.service.ts
import jwt from 'jsonwebtoken';
import { prisma } from '../db';
import { redis } from '../redis';

const JWT_SECRET = process.env.JWT_SECRET!;
const JWT_REFRESH_SECRET = process.env.JWT_REFRESH_SECRET!;
const ACCESS_TOKEN_EXPIRES = '15m';   // Short-lived access token
const REFRESH_TOKEN_EXPIRES = '7d';  // Longer refresh token

interface TokenPayload {
  sub: string;      // user ID
  role: string;
  iat: number;
  exp: number;
  jti: string;      // JWT ID (สำหรับ revocation)
}

// สร้าง token pair
export async function generateTokens(userId: string, role: string) {
  const jti = crypto.randomUUID();
  
  const accessToken = jwt.sign(
    { sub: userId, role, jti: crypto.randomUUID() },
    JWT_SECRET,
    {
      expiresIn: ACCESS_TOKEN_EXPIRES,
      algorithm: 'HS256',
      issuer: 'chuaikan.com',
      audience: 'chuaikan-api',
    }
  );

  const refreshToken = jwt.sign(
    { sub: userId, jti },
    JWT_REFRESH_SECRET,
    {
      expiresIn: REFRESH_TOKEN_EXPIRES,
      algorithm: 'HS256',
      issuer: 'chuaikan.com',
      audience: 'chuaikan-refresh',
    }
  );

  // เก็บ refresh token ใน Redis (สำหรับ validation และ revocation)
  await redis.setEx(
    `refresh_token:${jti}`,
    7 * 24 * 60 * 60,  // 7 วัน
    JSON.stringify({ userId, role })
  );

  return { accessToken, refreshToken };
}

// Verify และ rotate refresh token
export async function refreshTokens(refreshToken: string) {
  let payload: any;
  
  try {
    payload = jwt.verify(refreshToken, JWT_REFRESH_SECRET, {
      issuer: 'chuaikan.com',
      audience: 'chuaikan-refresh',
    }) as TokenPayload;
  } catch (err) {
    throw new Error('Invalid refresh token');
  }

  // ตรวจสอบใน Redis (ป้องกัน reuse หลัง logout)
  const storedData = await redis.get(`refresh_token:${payload.jti}`);
  if (!storedData) {
    throw new Error('Refresh token has been revoked');
  }

  // Revoke old refresh token (Token Rotation)
  await redis.del(`refresh_token:${payload.jti}`);

  // สร้าง token ใหม่
  const { userId, role } = JSON.parse(storedData);
  return await generateTokens(userId, role);
}

// Logout: revoke refresh token
export async function logout(refreshToken: string) {
  try {
    const payload = jwt.decode(refreshToken) as TokenPayload;
    if (payload?.jti) {
      await redis.del(`refresh_token:${payload.jti}`);
    }
  } catch {
    // Ignore decode errors on logout
  }
}

// Verify access token middleware
export function verifyAccessToken(token: string): TokenPayload {
  return jwt.verify(token, JWT_SECRET, {
    issuer: 'chuaikan.com',
    audience: 'chuaikan-api',
  }) as TokenPayload;
}
```

### Step 97: Password Hashing ด้วย argon2

```bash
npm install argon2
```

```typescript
// apps/api/src/utils/password.ts
import argon2 from 'argon2';

// argon2id parameters แนะนำสำหรับ production (2024)
const ARGON2_OPTIONS: argon2.Options = {
  type: argon2.argon2id,      // ใช้ argon2id (ปลอดภัยที่สุด)
  memoryCost: 65536,           // 64 MB memory
  timeCost: 3,                 // 3 iterations
  parallelism: 4,              // 4 parallel threads
  hashLength: 32,              // 32 bytes output
  saltLength: 16,              // 16 bytes salt (auto-generated)
};

export async function hashPassword(password: string): Promise<string> {
  // Validate password strength ก่อน hash
  if (password.length < 8) {
    throw new Error('Password must be at least 8 characters');
  }
  if (password.length > 128) {
    throw new Error('Password must not exceed 128 characters');
  }
  
  return await argon2.hash(password, ARGON2_OPTIONS);
}

export async function verifyPassword(
  hash: string,
  password: string
): Promise<boolean> {
  try {
    const isValid = await argon2.verify(hash, password);
    
    // ตรวจสอบว่า hash ต้องการ rehash (เมื่ออัพเดท parameters)
    if (isValid && argon2.needsRehash(hash, ARGON2_OPTIONS)) {
      // rehash ใน background (อัพเดทใน DB ด้วย)
      return isValid; // return true ก่อน แล้วค่อย rehash async
    }
    
    return isValid;
  } catch {
    return false;
  }
}

// ตัวอย่าง timing ที่ควรคาดหวัง
// hashPassword: ~100-300ms (ตั้งใจทำให้ช้า เพื่อป้องกัน brute force)
```

### Step 98: Environment Variable Validation ด้วย zod

```bash
npm install zod
```

```typescript
// apps/api/src/config/env.ts
import { z } from 'zod';

// Define schema สำหรับ environment variables
const envSchema = z.object({
  // App
  NODE_ENV: z.enum(['development', 'staging', 'production', 'test']),
  PORT: z.string().regex(/^\d+$/).transform(Number).pipe(z.number().min(1024).max(65535)),
  APP_URL: z.string().url(),
  
  // Database
  DATABASE_URL: z.string().url().startsWith('postgresql://'),
  DATABASE_POOL_MIN: z.string().optional().transform(v => v ? Number(v) : 2),
  DATABASE_POOL_MAX: z.string().optional().transform(v => v ? Number(v) : 10),
  
  // Redis
  REDIS_URL: z.string().url().startsWith('redis'),
  
  // JWT
  JWT_SECRET: z.string().min(32, 'JWT_SECRET must be at least 32 characters'),
  JWT_REFRESH_SECRET: z.string().min(32),
  
  // Security
  CSRF_SECRET: z.string().min(32),
  METRICS_SECRET: z.string().min(16).optional(),
  
  // External Services
  R2_ACCESS_KEY_ID: z.string().optional(),
  R2_SECRET_ACCESS_KEY: z.string().optional(),
  R2_BUCKET_NAME: z.string().optional(),
  CF_ACCOUNT_ID: z.string().optional(),
  
  // Optional
  SENTRY_DSN: z.string().url().optional(),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
});

// Validate และ export
function validateEnv() {
  const result = envSchema.safeParse(process.env);
  
  if (!result.success) {
    console.error('❌ Invalid environment variables:');
    result.error.errors.forEach(err => {
      console.error(`  ${err.path.join('.')}: ${err.message}`);
    });
    process.exit(1);
  }
  
  return result.data;
}

export const env = validateEnv();
export type Env = typeof env;

// ใช้งาน:
// import { env } from './config/env';
// const { DATABASE_URL, JWT_SECRET } = env;
```

### Step 99: npm audit และ Snyk Security Scanning

```bash
# ตรวจสอบ vulnerabilities ด้วย npm audit
npm audit

# แก้ไขอัตโนมัติ (ระวัง: อาจ break things)
npm audit fix

# แก้ไขแบบ major version (ต้องทดสอบก่อน)
npm audit fix --force

# ดู audit report แบบ JSON
npm audit --json > audit-report.json

# ตรวจสอบเฉพาะ production dependencies
npm audit --omit=dev
```

ติดตั้ง Snyk สำหรับ scanning ที่ละเอียดกว่า:

```bash
# ติดตั้ง Snyk CLI
npm install -g snyk

# Login
snyk auth

# Scan project
snyk test

# Scan พร้อม report
snyk test --json > snyk-report.json

# Monitor dependencies (continuous monitoring)
snyk monitor --project-name=chuaikan-api

# Scan Docker image
snyk container test ghcr.io/chuaikan/api:latest

# Fix vulnerabilities automatically
snyk wizard

# ตั้งค่า Snyk ใน CI/CD (.github/workflows/ci.yml)
```

เพิ่ม Security Scanning ใน CI:

```yaml
# ใน .github/workflows/ci.yml
  security-scan:
    name: Security Scan
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      
      - name: Run npm audit
        run: npm audit --audit-level=high
        
      - name: Run Snyk
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
          
      - name: Scan Docker image
        uses: snyk/actions/docker@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          image: ghcr.io/${{ github.repository }}/api:latest
          args: --severity-threshold=high
```

### Step 100: Content Security Policy สำหรับ Next.js

```typescript
// apps/web/next.config.js
const ContentSecurityPolicy = `
  default-src 'self';
  script-src 'self' 'unsafe-inline' 'unsafe-eval'
    https://cdn.jsdelivr.net
    https://cdnjs.cloudflare.com;
  style-src 'self' 'unsafe-inline'
    https://fonts.googleapis.com;
  font-src 'self'
    https://fonts.gstatic.com;
  img-src 'self' data: blob:
    https://assets.chuaikan.com
    https://r2.chuaikan.com;
  media-src 'self'
    https://assets.chuaikan.com;
  connect-src 'self'
    https://api.chuaikan.com
    wss://api.chuaikan.com;
  frame-src 'none';
  frame-ancestors 'none';
  object-src 'none';
  base-uri 'self';
  form-action 'self';
  upgrade-insecure-requests;
`.replace(/\s{2,}/g, ' ').trim();

const securityHeaders = [
  {
    key: 'Content-Security-Policy',
    value: ContentSecurityPolicy,
  },
  {
    key: 'X-DNS-Prefetch-Control',
    value: 'on',
  },
  {
    key: 'Strict-Transport-Security',
    value: 'max-age=31536000; includeSubDomains; preload',
  },
  {
    key: 'X-Frame-Options',
    value: 'DENY',
  },
  {
    key: 'X-Content-Type-Options',
    value: 'nosniff',
  },
  {
    key: 'Referrer-Policy',
    value: 'strict-origin-when-cross-origin',
  },
  {
    key: 'Permissions-Policy',
    value: 'geolocation=(), microphone=(), camera=(), payment=()',
  },
];

/** @type {import('next').NextConfig} */
const nextConfig = {
  output: 'standalone',
  
  async headers() {
    return [
      {
        // Apply to all routes
        source: '/(.*)',
        headers: securityHeaders,
      },
      {
        // Cache static assets
        source: '/_next/static/(.*)',
        headers: [
          { key: 'Cache-Control', value: 'public, max-age=31536000, immutable' },
        ],
      },
    ];
  },
  
  // ป้องกัน server component secrets leak
  serverExternalPackages: [],
};

module.exports = nextConfig;
```

---

## 🔧 Configuration Files

### Security Middleware Stack

```typescript
// apps/api/src/middleware/index.ts
import express, { Express } from 'express';
import helmet from 'helmet';
import cors from 'cors';
import { rateLimit } from 'express-rate-limit';
import { doubleCsrfProtection } from './csrf';
import { requestLogger } from './logger';

export function applyMiddleware(app: Express): void {
  // 1. Request logging (before everything)
  app.use(requestLogger);
  
  // 2. Security headers
  app.use(helmet({ /* ... */ }));
  
  // 3. CORS
  app.use(cors({ /* ... */ }));
  
  // 4. Parse JSON (limit size to prevent DoS)
  app.use(express.json({ limit: '10kb' }));
  app.use(express.urlencoded({ extended: true, limit: '10kb' }));
  
  // 5. Rate limiting
  app.use('/api/', rateLimit({ /* ... */ }));
  
  // 6. CSRF protection (after JSON parsing)
  app.use('/api/', doubleCsrfProtection);
  
  // 7. Input sanitization
  app.use((req, _res, next) => {
    if (req.body) {
      // Remove null bytes
      const sanitize = (obj: any): any => {
        if (typeof obj === 'string') {
          return obj.replace(/\0/g, '');
        }
        if (typeof obj === 'object' && obj !== null) {
          return Object.fromEntries(
            Object.entries(obj).map(([k, v]) => [k, sanitize(v)])
          );
        }
        return obj;
      };
      req.body = sanitize(req.body);
    }
    next();
  });
}
```

---

## 🧪 Testing

```bash
# Test 1: ตรวจสอบ Security Headers
curl -sI https://chuaikan.com | grep -iE \
    "strict-transport|x-frame|x-content|content-security|referrer"

# Test 2: SSL Rating
# ไปที่ https://www.ssllabs.com/ssltest/analyze.html?d=chuaikan.com
# Target: Grade A หรือ A+

# Test 3: Security Headers Score
# ไปที่ https://securityheaders.com/?q=chuaikan.com
# Target: Grade A

# Test 4: ทดสอบ SQL Injection (ต้องไม่ผ่าน!)
curl -s "https://api.chuaikan.com/api/users?email=test@test.com' OR '1'='1" | jq
# Expected: error response, NOT user data

# Test 5: ทดสอบ XSS (ต้องไม่ผ่าน!)
curl -s -X POST https://api.chuaikan.com/api/posts \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $TOKEN" \
    --data '{"title": "<script>alert(1)</script>", "content": "test"}' | jq
# Expected: script tags ถูก strip ออก

# Test 6: ทดสอบ Rate Limiting
for i in {1..15}; do
    curl -s -o /dev/null -w "%{http_code}\n" \
        -X POST https://api.chuaikan.com/api/auth/login \
        -H "Content-Type: application/json" \
        --data '{"email":"test@test.com","password":"wrong"}'
done
# Expected: 429 Too Many Requests หลัง attempt ที่ 10

# Test 7: ตรวจสอบ CSRF
curl -s -X POST https://api.chuaikan.com/api/users/profile \
    -H "Content-Type: application/json" \
    --data '{"name": "hack"}'
# Expected: 403 Forbidden (ไม่มี CSRF token)

# Test 8: Password hashing
node -e "
const argon2 = require('argon2');
const start = Date.now();
argon2.hash('test_password', {type: argon2.argon2id, memoryCost: 65536}).then(hash => {
    console.log('Hash time:', Date.now() - start, 'ms');
    console.log('Hash:', hash.substring(0, 50) + '...');
});
"
# Expected: 100-300ms (not instant)

# Test 9: npm audit
npm audit --audit-level=moderate
# Expected: found 0 vulnerabilities

# Test 10: ทดสอบ env validation
node -e "require('./dist/config/env')"
# ถ้า env ครบ: ไม่มี error
# ถ้า env ขาด: error message บอกว่าขาดตัวไหน
```

---

## ❌ Common Errors & Solutions

### Error 1: CORS Error ใน Browser

```
Access to fetch at 'https://api.chuaikan.com/api/users' from origin
'https://chuaikan.com' has been blocked by CORS policy
```

**Solution:**
```typescript
// ตรวจสอบ CORS config ใน API
const corsOptions = {
  origin: ['https://chuaikan.com', 'https://www.chuaikan.com'],
  credentials: true,  // จำเป็นสำหรับ cookies
};

// ตรวจสอบว่า frontend ส่ง credentials: 'include'
fetch('/api/users', { credentials: 'include' });
```

### Error 2: CSRF Token Mismatch

```
ForbiddenError: invalid csrf token
```

**Solution:**
```typescript
// ต้องดึง CSRF token ก่อน POST
const { csrfToken } = await fetch('/api/auth/csrf').then(r => r.json());

// ส่ง CSRF token ใน header
fetch('/api/users', {
  method: 'POST',
  headers: {
    'X-CSRF-Token': csrfToken,
    'Content-Type': 'application/json',
  },
  credentials: 'include',
  body: JSON.stringify(data),
});
```

### Error 3: JWT Token Expired

```
JsonWebTokenError: jwt expired
```

**Solution:**
```typescript
// ใน API client (apps/web)
async function fetchWithAuth(url: string, options: RequestInit = {}) {
  let response = await fetch(url, {
    ...options,
    headers: { ...options.headers, Authorization: `Bearer ${accessToken}` },
  });
  
  // ถ้า 401 ให้ refresh token
  if (response.status === 401) {
    const { accessToken: newToken } = await refreshTokens();
    setAccessToken(newToken);
    
    response = await fetch(url, {
      ...options,
      headers: { ...options.headers, Authorization: `Bearer ${newToken}` },
    });
  }
  
  return response;
}
```

### Error 4: argon2 ช้าเกินไปใน Test

```
# Test timeout เพราะ argon2 ช้า
```

**Solution:**
```typescript
// ใน test environment ใช้ parameters ที่เร็วกว่า
const hashOptions = process.env.NODE_ENV === 'test'
  ? { type: argon2.argon2id, memoryCost: 1024, timeCost: 1 }  // Fast for tests
  : { type: argon2.argon2id, memoryCost: 65536, timeCost: 3 }; // Secure for production
```

### Error 5: CSP Blocking Inline Scripts

```
Refused to execute inline script because it violates Content Security Policy
```

**Solution:**
```typescript
// ใช้ nonce-based CSP แทน 'unsafe-inline'
// ใน Next.js Middleware
import { NextResponse } from 'next/server';
import crypto from 'crypto';

export function middleware() {
  const nonce = crypto.randomBytes(16).toString('base64');
  const response = NextResponse.next();
  
  response.headers.set(
    'Content-Security-Policy',
    `script-src 'nonce-${nonce}' 'strict-dynamic'`
  );
  response.headers.set('x-nonce', nonce);
  
  return response;
}
```

---

## ✅ Checklist

### HTTPS & TLS
- [ ] Let's Encrypt certificate ออกแล้ว
- [ ] Auto-renewal ตั้งค่าแล้ว
- [ ] SSL Labs Grade A+ สำหรับ chuaikan.com
- [ ] HSTS header ตั้งค่าแล้ว (max-age=31536000)
- [ ] TLS 1.2+ เท่านั้น

### Security Headers
- [ ] Content-Security-Policy ตั้งค่าแล้ว
- [ ] X-Frame-Options: DENY
- [ ] X-Content-Type-Options: nosniff
- [ ] Strict-Transport-Security
- [ ] Referrer-Policy: strict-origin-when-cross-origin
- [ ] Permissions-Policy ตั้งค่าแล้ว
- [ ] SecurityHeaders.com Score: A+

### Application Security
- [ ] Helmet.js ติดตั้งใน API
- [ ] Rate limiting ตั้งค่า (100/min general, 10/min auth)
- [ ] CORS config ถูกต้อง
- [ ] CSRF protection เปิดใช้งาน
- [ ] Input validation ด้วย zod ทุก endpoint
- [ ] SQL: ใช้ Prisma (parameterized queries เท่านั้น)
- [ ] XSS: sanitize-html สำหรับ user content
- [ ] DOMPurify ใน client components

### Authentication
- [ ] JWT access token: 15 minutes expiry
- [ ] JWT refresh token: 7 days expiry
- [ ] Refresh token rotation ทำงาน
- [ ] Logout revokes refresh token (Redis)
- [ ] Password: argon2id, 64MB memory, 3 iterations
- [ ] No password stored in plain text

### Dependency Security
- [ ] npm audit: 0 high/critical vulnerabilities
- [ ] Snyk ติดตั้งและ scan แล้ว
- [ ] Snyk ใน CI/CD pipeline
- [ ] Dependencies อัพเดทเป็น version ล่าสุด

### Environment
- [ ] .env validation ด้วย zod
- [ ] ไม่มี secrets ใน codebase
- [ ] .env files อยู่ใน .gitignore
- [ ] Production secrets ใช้ Docker secrets / GitHub Secrets

---

## 🔗 References

- [OWASP Top 10 2021](https://owasp.org/Top10/)
- [Let's Encrypt Documentation](https://letsencrypt.org/docs/)
- [Helmet.js](https://helmetjs.github.io/)
- [CSRF-CSRF Library](https://github.com/Psifi-Solutions/csrf-csrf)
- [argon2 npm package](https://github.com/ranisalt/node-argon2)
- [JWT Best Practices](https://datatracker.ietf.org/doc/html/rfc8725)
- [DOMPurify](https://github.com/cure53/DOMPurify)
- [sanitize-html](https://github.com/apostrophecms/sanitize-html)
- [Mozilla Security Guidelines](https://infosec.mozilla.org/guidelines/web_security)
- [SecurityHeaders.com](https://securityheaders.com/)
- [SSL Labs Test](https://www.ssllabs.com/ssltest/)

---
*Part 010 | Road to 1,000,000 Users/Day | chuaikan.com*
