# Part 007: Cloudflare — CDN & DDoS Protection
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Beginner
> **Steps:** 61-70
> **เวลาโดยประมาณ:** 4 ชั่วโมง
> **Prerequisites:** Part 001-006 (Linux, Git, Node.js, PostgreSQL, Redis, Docker)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

1. ตั้งค่า Cloudflare DNS สำหรับ chuaikan.com
2. เปิดใช้งาน SSL/TLS แบบ Full (Strict)
3. ตั้งค่า Page Rules สำหรับ cache static assets
4. สร้าง Firewall Rules เพื่อป้องกัน bots และ block by country
5. ตั้งค่า Rate Limiting สำหรับ API endpoints
6. ตั้งค่า Cache Rules สำหรับ Next.js static assets
7. เขียน Cloudflare Workers สำหรับ edge authentication
8. ตั้งค่า R2 Storage สำหรับ media files
9. ใช้งาน Cloudflare Tunnel สำหรับ secure origin connection
10. ตั้งค่า WAF และ Bot Fight Mode

---

## 📖 ทฤษฎีและแนวคิด

### Cloudflare คืออะไร?

Cloudflare เป็น CDN (Content Delivery Network) และ security platform ที่ทำหน้าที่เป็น reverse proxy อยู่ระหว่าง users กับ server ของเรา

```
┌──────────────────────────────────────────────────────────┐
│                  Request Flow                             │
│                                                          │
│  User ──→ Cloudflare Edge ──→ Origin Server              │
│  (Thailand)  (Singapore PoP)   (VPS/Cloud)               │
│                                                          │
│  Cloudflare ทำหน้าที่:                                   │
│  ✓ Cache static content (ไม่ forward request ถึง origin) │
│  ✓ DDoS protection (block bad traffic)                   │
│  ✓ SSL termination (HTTPS ถึง edge)                      │
│  ✓ WAF (Web Application Firewall)                        │
│  ✓ Rate limiting                                         │
│  ✓ Bot management                                        │
└──────────────────────────────────────────────────────────┘
```

### Cloudflare Proxy Modes

```
DNS Only (Grey Cloud):
  User ──→ Origin Server (Direct)
  ไม่มี protection, ไม่มี CDN

Proxied (Orange Cloud):
  User ──→ Cloudflare ──→ Origin Server
  มี DDoS protection, CDN, SSL, WAF

ควรใช้ Orange Cloud สำหรับ web servers เสมอ!
ยกเว้น: mail servers (MX), FTP servers
```

---

## ⚙️ Environment Setup

### Step 61: ตั้งค่า Cloudflare DNS

ไปที่ Cloudflare Dashboard → chuaikan.com → DNS → Records

#### DNS Records ที่ต้องตั้งค่า:

```
# A Records (IPv4)
Type  Name            Content          Proxy    TTL
A     @               203.0.113.10     ✓        Auto
A     www             203.0.113.10     ✓        Auto
A     api             203.0.113.10     ✓        Auto
A     staging         203.0.113.20     ✓        Auto

# CNAME Records
Type   Name      Content                  Proxy    TTL
CNAME  r2        pub-r2.chuaikan.com      ✓        Auto
CNAME  assets    chuaikan.r2.cloudflarestorage.com  ✓  Auto

# MX Records (Email - ไม่ proxy)
Type  Name    Content                    Priority  Proxy  TTL
MX    @       mail1.youremailprovider.com  10       ✗      Auto
MX    @       mail2.youremailprovider.com  20       ✗      Auto

# TXT Records
Type  Name    Content
TXT   @       "v=spf1 include:_spf.google.com ~all"
TXT   @       "google-site-verification=your_verification_code"
TXT   _dmarc  "v=DMARC1; p=quarantine; rua=mailto:dmarc@chuaikan.com"
```

ใช้ Cloudflare API เพื่อ manage DNS programmatically:

```bash
# ตั้งค่า environment variables
export CF_API_TOKEN="your_cloudflare_api_token"
export CF_ZONE_ID="your_zone_id"  # หาได้จาก Zone Overview page

# ดู DNS records ปัจจุบัน
curl -s -X GET \
    "https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/dns_records" \
    -H "Authorization: Bearer ${CF_API_TOKEN}" \
    -H "Content-Type: application/json" | jq '.result[] | {type, name, content, proxied}'

# สร้าง A record
curl -s -X POST \
    "https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/dns_records" \
    -H "Authorization: Bearer ${CF_API_TOKEN}" \
    -H "Content-Type: application/json" \
    --data '{
        "type": "A",
        "name": "api",
        "content": "203.0.113.10",
        "ttl": 1,
        "proxied": true
    }' | jq '.success, .result.id'

# สร้าง CNAME record
curl -s -X POST \
    "https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/dns_records" \
    -H "Authorization: Bearer ${CF_API_TOKEN}" \
    -H "Content-Type: application/json" \
    --data '{
        "type": "CNAME",
        "name": "www",
        "content": "chuaikan.com",
        "ttl": 1,
        "proxied": true
    }' | jq '.success'
```

---

## 🛠️ Step-by-Step Implementation

### Step 62: SSL/TLS Configuration

ไปที่ Cloudflare Dashboard → SSL/TLS → Overview

```
ตั้งค่า SSL/TLS encryption mode เป็น: Full (strict)

Flexible: HTTP ระหว่าง Cloudflare กับ Origin (ไม่ปลอดภัย!)
Full: HTTPS ระหว่าง Cloudflare กับ Origin (แต่ไม่ verify certificate)
Full (strict): HTTPS + verify valid SSL certificate (แนะนำ!)
```

เปิดใช้งาน options เหล่านี้:

```
SSL/TLS → Edge Certificates:
  ✓ Always Use HTTPS
  ✓ HTTP Strict Transport Security (HSTS)
    - Max Age Header: 12 months
    - Include subdomains: YES
    - Preload: YES
    - No-sniff header: YES
  ✓ Minimum TLS Version: TLS 1.2
  ✓ Opportunistic Encryption
  ✓ TLS 1.3
  ✓ Automatic HTTPS Rewrites
```

ติดตั้ง SSL Certificate บน origin server:

```bash
# ติดตั้ง Certbot บน Ubuntu 24.04
sudo apt-get update
sudo apt-get install -y certbot python3-certbot-nginx

# ขอ certificate สำหรับ chuaikan.com
sudo certbot --nginx \
    -d chuaikan.com \
    -d www.chuaikan.com \
    -d api.chuaikan.com \
    --non-interactive \
    --agree-tos \
    --email admin@chuaikan.com

# ดูว่า certificate ถูกสร้างแล้ว
sudo certbot certificates

# ตรวจสอบ auto-renewal
sudo certbot renew --dry-run

# Certificate อยู่ที่:
# /etc/letsencrypt/live/chuaikan.com/fullchain.pem
# /etc/letsencrypt/live/chuaikan.com/privkey.pem
```

### Step 63: Firewall Rules

ไปที่ Cloudflare Dashboard → Security → WAF → Firewall Rules

**Rule 1: Block Countries ที่ไม่ใช่กลุ่มเป้าหมาย**

```
Rule Name: Block Non-Target Countries
Expression:
  not (
    ip.geoip.country in {"TH" "MM" "LA" "KH" "VN" "MY" "SG"}
    or cf.client.bot
  )
  and not (
    http.request.uri.path contains "/api/webhook"
    or http.request.uri.path contains "/.well-known"
  )

Action: Block

หมายเหตุ: ปรับ country list ตาม target market ของคุณ
TH = ไทย, MM = พม่า, LA = ลาว, KH = กัมพูชา
VN = เวียดนาม, MY = มาเลเซีย, SG = สิงคโปร์
```

**Rule 2: Block Bad User Agents**

```
Rule Name: Block Bad Bots
Expression:
  (
    http.user_agent contains "zgrab"
    or http.user_agent contains "masscan"
    or http.user_agent contains "nikto"
    or http.user_agent contains "sqlmap"
    or http.user_agent contains "nmap"
    or http.user_agent eq ""
  )
  and not cf.client.bot

Action: Block
```

**Rule 3: Challenge Suspicious Requests**

```
Rule Name: Challenge Suspicious Traffic
Expression:
  (
    http.request.uri.path contains "/wp-admin"
    or http.request.uri.path contains "/phpmyadmin"
    or http.request.uri.path contains "/.env"
    or http.request.uri.path contains "/config"
    or http.request.uri.path matches ".*\\.(php|asp|aspx|jsp)$"
  )

Action: Managed Challenge
```

ใช้ Cloudflare API สร้าง Firewall Rules:

```bash
# Rule: Block SQL injection attempts
curl -s -X POST \
    "https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/firewall/rules" \
    -H "Authorization: Bearer ${CF_API_TOKEN}" \
    -H "Content-Type: application/json" \
    --data '[{
        "filter": {
            "expression": "(http.request.uri.query contains \"UNION SELECT\" or http.request.uri.query contains \"OR 1=1\" or http.request.body contains \"DROP TABLE\")",
            "paused": false,
            "description": "Block SQL Injection"
        },
        "action": "block",
        "description": "Block SQL Injection Attempts",
        "paused": false
    }]' | jq '.result[].id'
```

### Step 64: Rate Limiting

ไปที่ Cloudflare Dashboard → Security → WAF → Rate limiting rules

```bash
# สร้าง Rate Limiting Rule สำหรับ API endpoints
curl -s -X POST \
    "https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/rulesets" \
    -H "Authorization: Bearer ${CF_API_TOKEN}" \
    -H "Content-Type: application/json" \
    --data '{
        "name": "API Rate Limiting",
        "description": "Limit API requests",
        "kind": "zone",
        "phase": "http_ratelimit",
        "rules": [
            {
                "description": "Limit API to 100 req/min per IP",
                "expression": "(http.request.uri.path matches \"/api/.*\")",
                "action": "block",
                "ratelimit": {
                    "characteristics": ["ip.src"],
                    "period": 60,
                    "requests_per_period": 100,
                    "mitigation_timeout": 600
                }
            },
            {
                "description": "Limit login to 10 req/min per IP",
                "expression": "(http.request.uri.path eq \"/api/auth/login\" and http.request.method eq \"POST\")",
                "action": "block",
                "ratelimit": {
                    "characteristics": ["ip.src"],
                    "period": 60,
                    "requests_per_period": 10,
                    "mitigation_timeout": 3600
                }
            }
        ]
    }' | jq '.result.id'
```

### Step 65: Cache Rules

ไปที่ Cloudflare Dashboard → Caching → Cache Rules

**Rule 1: Cache Next.js Static Assets (1 ปี)**

```
Rule Name: Cache Next.js Static Assets
If:
  Request URL path matches /_next/static/*

Then:
  Edge TTL: 1 year (31536000 seconds)
  Browser TTL: 1 year
  Cache Status: Cache Everything
  Origin Cache Control: Off
```

**Rule 2: Cache Public Images**

```
Rule Name: Cache Images
If:
  Request URL path matches /images/*
  OR Request URL path matches /assets/*

Then:
  Edge TTL: 30 days
  Browser TTL: 7 days
  Cache Status: Cache Everything
  Polish: Lossless (compress images)
```

**Rule 3: Bypass Cache for API**

```
Rule Name: Bypass API Cache
If:
  Request URL path matches /api/*

Then:
  Cache Status: Bypass
```

ใช้ Cloudflare API ตั้งค่า Cache:

```bash
# ดู cache settings ปัจจุบัน
curl -s -X GET \
    "https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/settings/cache_level" \
    -H "Authorization: Bearer ${CF_API_TOKEN}" | jq '.result'

# ตั้งค่า cache level เป็น aggressive
curl -s -X PATCH \
    "https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/settings/cache_level" \
    -H "Authorization: Bearer ${CF_API_TOKEN}" \
    -H "Content-Type: application/json" \
    --data '{"value": "aggressive"}' | jq '.result'

# เปิด Browser Cache TTL
curl -s -X PATCH \
    "https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/settings/browser_cache_ttl" \
    -H "Authorization: Bearer ${CF_API_TOKEN}" \
    -H "Content-Type: application/json" \
    --data '{"value": 14400}' | jq '.result'

# Purge cache เมื่อ deploy ใหม่
curl -s -X POST \
    "https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/purge_cache" \
    -H "Authorization: Bearer ${CF_API_TOKEN}" \
    -H "Content-Type: application/json" \
    --data '{"purge_everything": true}' | jq '.result'
```

### Step 66: Cloudflare Workers — Edge Authentication

สร้างไฟล์ `workers/auth-middleware/src/index.ts`:

```typescript
// Cloudflare Workers script สำหรับ edge authentication
// Deploy ด้วย: wrangler deploy

interface Env {
  JWT_SECRET: string;
  AUTH_BYPASS_PATHS: string;
}

interface JWTPayload {
  sub: string;
  iat: number;
  exp: number;
  role: string;
}

// Decode JWT without verification (verification ทำใน worker)
function decodeJWT(token: string): JWTPayload | null {
  try {
    const parts = token.split('.');
    if (parts.length !== 3) return null;
    
    const payload = JSON.parse(atob(parts[1].replace(/-/g, '+').replace(/_/g, '/')));
    return payload as JWTPayload;
  } catch {
    return null;
  }
}

// Verify JWT signature ด้วย Web Crypto API
async function verifyJWT(token: string, secret: string): Promise<JWTPayload | null> {
  try {
    const parts = token.split('.');
    if (parts.length !== 3) return null;

    const encoder = new TextEncoder();
    const keyData = encoder.encode(secret);
    
    const key = await crypto.subtle.importKey(
      'raw',
      keyData,
      { name: 'HMAC', hash: 'SHA-256' },
      false,
      ['verify']
    );

    const signatureInput = `${parts[0]}.${parts[1]}`;
    const signature = Uint8Array.from(
      atob(parts[2].replace(/-/g, '+').replace(/_/g, '/')),
      c => c.charCodeAt(0)
    );

    const isValid = await crypto.subtle.verify(
      'HMAC',
      key,
      signature,
      encoder.encode(signatureInput)
    );

    if (!isValid) return null;

    const payload = JSON.parse(atob(parts[1].replace(/-/g, '+').replace(/_/g, '/')));
    
    // ตรวจสอบ expiration
    if (payload.exp && payload.exp < Math.floor(Date.now() / 1000)) {
      return null;
    }

    return payload as JWTPayload;
  } catch {
    return null;
  }
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const pathname = url.pathname;

    // Bypass paths ที่ไม่ต้อง auth
    const bypassPaths = (env.AUTH_BYPASS_PATHS || '/,/api/auth/login,/api/auth/register,/api/health').split(',');
    
    if (bypassPaths.some(path => pathname === path || pathname.startsWith(path))) {
      return fetch(request);
    }

    // ตรวจสอบ protected API routes เท่านั้น
    if (!pathname.startsWith('/api/')) {
      return fetch(request);
    }

    // ดึง JWT token จาก Authorization header
    const authHeader = request.headers.get('Authorization');
    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      return new Response(JSON.stringify({ error: 'Unauthorized', code: 'NO_TOKEN' }), {
        status: 401,
        headers: { 'Content-Type': 'application/json' }
      });
    }

    const token = authHeader.substring(7);
    const payload = await verifyJWT(token, env.JWT_SECRET);

    if (!payload) {
      return new Response(JSON.stringify({ error: 'Unauthorized', code: 'INVALID_TOKEN' }), {
        status: 401,
        headers: { 'Content-Type': 'application/json' }
      });
    }

    // เพิ่ม user info ใน request headers ให้ origin server
    const modifiedRequest = new Request(request, {
      headers: {
        ...Object.fromEntries(request.headers),
        'X-User-Id': payload.sub,
        'X-User-Role': payload.role,
        'X-Auth-Verified': 'true',
      }
    });

    return fetch(modifiedRequest);
  }
};
```

ตั้งค่า `workers/auth-middleware/wrangler.toml`:

```toml
name = "chuaikan-auth"
main = "src/index.ts"
compatibility_date = "2024-01-01"
compatibility_flags = ["nodejs_compat"]

[vars]
AUTH_BYPASS_PATHS = "/,/api/auth/login,/api/auth/register,/api/health,/api/webhooks"

[[secrets]]
name = "JWT_SECRET"

[routes]
pattern = "chuaikan.com/api/*"
zone_name = "chuaikan.com"
```

```bash
# ติดตั้ง Wrangler CLI
npm install -g wrangler

# Login เข้า Cloudflare
wrangler login

# ตั้งค่า secret
wrangler secret put JWT_SECRET --name chuaikan-auth
# ป้อน JWT secret ของคุณ

# Deploy worker
cd workers/auth-middleware
wrangler deploy

# ทดสอบ
curl -H "Authorization: Bearer invalid_token" \
    https://chuaikan.com/api/user/profile
# Expected: {"error":"Unauthorized","code":"INVALID_TOKEN"}
```

### Step 67: R2 Storage Setup

```bash
# สร้าง R2 bucket ผ่าน Dashboard
# หรือใช้ Wrangler CLI

# สร้าง bucket
wrangler r2 bucket create chuaikan-media
wrangler r2 bucket create chuaikan-media-staging

# ดู buckets
wrangler r2 bucket list

# อัพโหลดไฟล์ทดสอบ
echo "Hello from R2" > test.txt
wrangler r2 object put chuaikan-media/test.txt --file test.txt

# ดูไฟล์ใน bucket
wrangler r2 object list chuaikan-media

# ลบไฟล์
wrangler r2 object delete chuaikan-media/test.txt
```

สร้าง API สำหรับ upload files ผ่าน R2:

```typescript
// apps/api/src/services/storage.service.ts
import { S3Client, PutObjectCommand, DeleteObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { randomUUID } from 'crypto';
import path from 'path';

const r2Client = new S3Client({
  region: 'auto',
  endpoint: `https://${process.env.CF_ACCOUNT_ID}.r2.cloudflarestorage.com`,
  credentials: {
    accessKeyId: process.env.R2_ACCESS_KEY_ID!,
    secretAccessKey: process.env.R2_SECRET_ACCESS_KEY!,
  },
});

const BUCKET_NAME = process.env.R2_BUCKET_NAME || 'chuaikan-media';
const PUBLIC_URL = process.env.R2_PUBLIC_URL || 'https://assets.chuaikan.com';

export async function uploadFile(
  file: Buffer,
  originalName: string,
  mimeType: string,
  folder: string = 'uploads'
): Promise<{ key: string; url: string }> {
  const ext = path.extname(originalName);
  const key = `${folder}/${randomUUID()}${ext}`;

  await r2Client.send(new PutObjectCommand({
    Bucket: BUCKET_NAME,
    Key: key,
    Body: file,
    ContentType: mimeType,
    // Cache for 1 year for immutable files
    CacheControl: 'public, max-age=31536000, immutable',
  }));

  return {
    key,
    url: `${PUBLIC_URL}/${key}`,
  };
}

export async function deleteFile(key: string): Promise<void> {
  await r2Client.send(new DeleteObjectCommand({
    Bucket: BUCKET_NAME,
    Key: key,
  }));
}

// สร้าง presigned URL สำหรับ direct upload จาก client
export async function createPresignedUploadUrl(
  filename: string,
  mimeType: string,
  folder: string = 'uploads'
): Promise<{ presignedUrl: string; key: string; publicUrl: string }> {
  const ext = path.extname(filename);
  const key = `${folder}/${randomUUID()}${ext}`;

  const presignedUrl = await getSignedUrl(
    r2Client,
    new PutObjectCommand({
      Bucket: BUCKET_NAME,
      Key: key,
      ContentType: mimeType,
    }),
    { expiresIn: 3600 } // 1 ชั่วโมง
  );

  return {
    presignedUrl,
    key,
    publicUrl: `${PUBLIC_URL}/${key}`,
  };
}
```

### Step 68: Cloudflare Tunnel

```bash
# ติดตั้ง cloudflared บน Ubuntu 24.04
curl -L --output cloudflared.deb \
    https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared.deb

# ตรวจสอบ version
cloudflared --version

# Login เข้า Cloudflare
cloudflared tunnel login
# จะเปิด browser ให้ authorize

# สร้าง tunnel
cloudflared tunnel create chuaikan-production
# จะได้ Tunnel ID เช่น: abc123...

# สร้าง config file
mkdir -p ~/.cloudflared
cat > ~/.cloudflared/config.yml <<'EOF'
tunnel: abc123def456  # ใส่ Tunnel ID ของคุณ
credentials-file: /root/.cloudflared/abc123def456.json

ingress:
  - hostname: chuaikan.com
    service: http://localhost:3000
  - hostname: api.chuaikan.com
    service: http://localhost:4000
  - hostname: www.chuaikan.com
    service: http://localhost:3000
  # Catch-all rule (จำเป็น)
  - service: http_status:404
EOF

# ทดสอบ config
cloudflared tunnel ingress validate

# Route DNS ไปยัง tunnel
cloudflared tunnel route dns chuaikan-production chuaikan.com
cloudflared tunnel route dns chuaikan-production api.chuaikan.com

# รัน tunnel
cloudflared tunnel run chuaikan-production

# ตั้งค่าเป็น systemd service
sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
sudo systemctl status cloudflared
```

### Step 69: WAF Managed Rules + Bot Fight Mode

ไปที่ Cloudflare Dashboard:

**WAF Managed Rules:**
```
Security → WAF → Managed rules → Deploy
เลือก:
  ✓ Cloudflare Managed Ruleset
  ✓ Cloudflare OWASP Core Ruleset (Paranoia Level: 2)
  ✓ Cloudflare Exposed Credentials Check
```

**Bot Fight Mode:**
```
Security → Bots → Bot Fight Mode: ON
Super Bot Fight Mode:
  ✓ Definitely Automated: Block
  ✓ Likely Automated: Managed Challenge
  ✓ Verified Bots: Allow (Google, Bing, etc.)
```

```bash
# เปิด Bot Fight Mode ผ่าน API
curl -s -X PUT \
    "https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/bot_management" \
    -H "Authorization: Bearer ${CF_API_TOKEN}" \
    -H "Content-Type: application/json" \
    --data '{
        "fight_mode": true,
        "session_score": false,
        "auto_update_model": true
    }' | jq '.result'

# ดู security events
curl -s -X GET \
    "https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/security/events?since=-86400&limit=10" \
    -H "Authorization: Bearer ${CF_API_TOKEN}" | jq '.result[] | {action, source, clientIP}'
```

### Step 70: Analytics Dashboard

```bash
# ดู traffic analytics ผ่าน API
# Total requests ใน 24 ชั่วโมงที่ผ่านมา
curl -s -X POST \
    "https://api.cloudflare.com/client/v4/graphql" \
    -H "Authorization: Bearer ${CF_API_TOKEN}" \
    -H "Content-Type: application/json" \
    --data '{
        "query": "{ viewer { zones(filter: { zoneTag: \"'${CF_ZONE_ID}'\" }) { httpRequests1dGroups(limit: 7, filter: { date_geq: \"2024-12-01\", date_leq: \"2024-12-07\" }) { sum { requests pageViews bytes } dimensions { date } } } } }"
    }' | jq '.data.viewer.zones[0].httpRequests1dGroups[]'

# Top countries
curl -s -X POST \
    "https://api.cloudflare.com/client/v4/graphql" \
    -H "Authorization: Bearer ${CF_API_TOKEN}" \
    -H "Content-Type: application/json" \
    --data '{
        "query": "{ viewer { zones(filter: { zoneTag: \"'${CF_ZONE_ID}'\" }) { httpRequests1hGroups(limit: 1, filter: { datetimeHour_geq: \"2024-12-07T00:00:00Z\" }) { sum { countryMap { clientCountryName requests } } } } } }"
    }' | jq '.data.viewer.zones[0].httpRequests1hGroups[0].sum.countryMap[] | select(.requests > 100)'
```

---

## 🔧 Configuration Files

### Nginx Configuration สำหรับ Origin Server

```nginx
# /etc/nginx/sites-available/chuaikan.com
server {
    listen 443 ssl http2;
    server_name chuaikan.com www.chuaikan.com;

    ssl_certificate /etc/letsencrypt/live/chuaikan.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/chuaikan.com/privkey.pem;
    
    # Allow only Cloudflare IPs (ป้องกัน bypass)
    # อัพเดท list ได้จาก https://www.cloudflare.com/ips/
    allow 173.245.48.0/20;
    allow 103.21.244.0/22;
    allow 103.22.200.0/22;
    allow 103.31.4.0/22;
    allow 141.101.64.0/18;
    allow 108.162.192.0/18;
    allow 190.93.240.0/20;
    allow 188.114.96.0/20;
    allow 197.234.240.0/22;
    allow 198.41.128.0/17;
    allow 162.158.0.0/15;
    allow 104.16.0.0/13;
    allow 104.24.0.0/14;
    allow 172.64.0.0/13;
    allow 131.0.72.0/22;
    # IPv6
    allow 2400:cb00::/32;
    allow 2606:4700::/32;
    allow 2803:f800::/32;
    allow 2405:b500::/32;
    allow 2405:8100::/32;
    allow 2a06:98c0::/29;
    allow 2c0f:f248::/32;
    # localhost for Cloudflare Tunnel
    allow 127.0.0.1;
    deny all;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $http_cf_connecting_ip;
        proxy_set_header X-Forwarded-For $http_cf_connecting_ip;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}

server {
    listen 80;
    server_name chuaikan.com www.chuaikan.com;
    return 301 https://$server_name$request_uri;
}
```

---

## 🧪 Testing

```bash
# Test 1: ตรวจสอบ DNS propagation
nslookup chuaikan.com
dig chuaikan.com +short
# ควรได้ IP ของ Cloudflare ไม่ใช่ origin server

# Test 2: ตรวจสอบ SSL
curl -vI https://chuaikan.com 2>&1 | grep -E "SSL|TLS|cipher|certificate"
# ตรวจ: TLS version, cipher suite, certificate issuer

# Test 3: ตรวจสอบ Security Headers
curl -sI https://chuaikan.com | grep -E "strict-transport|x-frame|x-content|content-security"

# Test 4: ทดสอบ Rate Limiting
for i in {1..110}; do
    STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://chuaikan.com/api/test)
    echo "Request $i: $STATUS"
    if [ "$STATUS" = "429" ]; then
        echo "Rate limit hit at request $i"
        break
    fi
done

# Test 5: ตรวจสอบ Cache
curl -sI https://chuaikan.com/_next/static/chunks/main.js | grep -E "cf-cache|age|cache-control"
# Expected: CF-Cache-Status: HIT, Age: ...

# Test 6: ทดสอบ Cloudflare Worker
curl -H "Authorization: Bearer invalid" \
    https://chuaikan.com/api/user/profile
# Expected: {"error":"Unauthorized","code":"INVALID_TOKEN"}

# Test 7: ตรวจสอบ Origin IP ไม่รั่ว
curl -s https://api64.ipify.org?format=json
# ต้องได้ IP ของ Cloudflare ไม่ใช่ origin server

# Test 8: ทดสอบ R2 upload
curl -X POST https://chuaikan.com/api/upload \
    -H "Authorization: Bearer $JWT_TOKEN" \
    -F "file=@test.jpg"
```

---

## ❌ Common Errors & Solutions

### Error 1: ERR_TOO_MANY_REDIRECTS

```
The page isn't redirecting properly (redirect loop)
```

**Solution:**
```
สาเหตุ: Cloudflare SSL mode เป็น "Flexible" แต่ origin server redirect HTTP → HTTPS
แก้ไข: เปลี่ยน SSL mode เป็น "Full" หรือ "Full (strict)"
Dashboard → SSL/TLS → Overview → Full (strict)
```

### Error 2: 521 Web Server Is Down

```
Error 521: Web Server Is Down
```

**Solution:**
```bash
# ตรวจสอบว่า origin server รันอยู่
sudo systemctl status nginx
sudo systemctl status your-app

# ตรวจสอบว่า firewall ไม่ block Cloudflare IPs
sudo ufw status
# ต้อง allow port 443 จาก Cloudflare IPs

# ตรวจสอบ origin server logs
sudo journalctl -u nginx -n 50
```

### Error 3: 1020 Access Denied (Firewall Rule)

```
Error 1020: Access Denied
```

**Solution:**
```
ตรวจสอบ Firewall Rules ที่ตั้งค่าไว้
Dashboard → Security → WAF → Firewall rules
ดู Events ว่า request ถูก block ด้วย rule ไหน
แก้ไข rule หรือ whitelist IP นั้น
```

### Error 4: Cloudflare Worker Error

```
Error: Script exceeded CPU time limit
```

**Solution:**
```typescript
// Worker ใช้เวลา CPU มากเกินไป
// ใช้ async/await แทน synchronous operations
// หลีกเลี่ยง heavy computation ใน worker
// ใช้ KV storage แทนการคำนวณซ้ำ

// ❌ หลีกเลี่ยง
function heavyComputation() {
    // loops ขนาดใหญ่, regular expressions ซับซ้อน
}

// ✅ ใช้แทน
async function cachedResult(key: string, env: Env) {
    const cached = await env.KV.get(key);
    if (cached) return JSON.parse(cached);
    // compute และ cache ผล
}
```

### Error 5: Cache MISS ตลอด

```
CF-Cache-Status: MISS (ไม่ cache)
```

**Solution:**
```bash
# ตรวจสอบ Cache-Control header จาก origin
curl -sI https://chuaikan.com/images/logo.png | grep cache-control

# Origin ต้องส่ง Cache-Control header ที่ถูกต้อง
# เพิ่มใน Next.js:
# next.config.js
async headers() {
    return [{
        source: '/images/:path*',
        headers: [{ key: 'Cache-Control', value: 'public, max-age=31536000' }]
    }]
}

# หรือตั้ง Cache Rule ใน Cloudflare ให้ Override origin settings
```

---

## ✅ Checklist

### DNS Setup
- [ ] A record สำหรับ @ และ www ชี้ไปที่ origin server
- [ ] A record สำหรับ api subdomain
- [ ] MX records ตั้งค่าแล้วสำหรับ email
- [ ] DMARC, SPF, DKIM records ตั้งค่าแล้ว
- [ ] ทุก web records ใช้ Orange Cloud (Proxied)

### SSL/TLS
- [ ] SSL mode เป็น Full (strict)
- [ ] Always Use HTTPS เปิดอยู่
- [ ] HSTS เปิดอยู่ (12 months, include subdomains)
- [ ] Minimum TLS 1.2
- [ ] Origin server มี valid SSL certificate

### Security
- [ ] Firewall Rules: Block non-target countries
- [ ] Firewall Rules: Block bad user agents
- [ ] Rate Limiting: 100 req/min สำหรับ API
- [ ] Rate Limiting: 10 req/min สำหรับ login
- [ ] WAF Managed Rules เปิดอยู่
- [ ] Bot Fight Mode เปิดอยู่

### Caching
- [ ] Cache Rules สำหรับ /_next/static/* (1 year)
- [ ] Cache Rules สำหรับ /images/* (30 days)
- [ ] API requests ไม่ถูก cache
- [ ] Tested: CF-Cache-Status: HIT สำหรับ static files

### Workers & Storage
- [ ] Auth Worker deploy แล้ว
- [ ] R2 bucket สร้างแล้ว
- [ ] R2 public URL ตั้งค่าแล้ว
- [ ] Cloudflare Tunnel ตั้งค่าแล้ว (optional)

---

## 🔗 References

- [Cloudflare Documentation](https://developers.cloudflare.com/)
- [Cloudflare Workers Docs](https://developers.cloudflare.com/workers/)
- [Cloudflare R2 Documentation](https://developers.cloudflare.com/r2/)
- [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- [Cloudflare API Reference](https://api.cloudflare.com/)
- [WAF Managed Rules](https://developers.cloudflare.com/waf/managed-rules/)

---
*Part 007 | Road to 1,000,000 Users/Day | chuaikan.com*
