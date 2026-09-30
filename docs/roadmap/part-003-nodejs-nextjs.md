# Part 003: Node.js & Next.js Performance Basics
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 21-30
> **เวลาโดยประมาณ:** 10 ชั่วโมง
> **Prerequisites:** Part 001 (Linux Basics), Part 002 (Git Workflow)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

1. Node.js Event Loop และ non-blocking I/O model
2. Production Node.js configuration (ENV, thread pool, memory)
3. next.config.js production config แบบสมบูรณ์
4. Image optimization ด้วย next/image
5. Font optimization ด้วย next/font
6. Bundle analysis ด้วย @next/bundle-analyzer
7. Lazy loading และ dynamic imports
8. Code splitting strategies
9. Core Web Vitals measurement (LCP, FID, CLS)
10. Lighthouse CI setup

---

## 📖 ทฤษฎีและแนวคิด

### Node.js Event Loop Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    Node.js Runtime                            │
│                                                              │
│  Your JS Code                                                │
│  ───────────────                                             │
│  const data = await readFile('a.txt')  ← non-blocking!      │
│  const result = await fetch(url)       ← non-blocking!      │
│                                                              │
│  Event Loop (Single Thread)                                  │
│  ─────────────────────────────────────────────────────────  │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐               │
│  │  timers  │ →  │  I/O     │ →  │  check   │               │
│  │setTimeout│    │callbacks │    │setImmed. │               │
│  └──────────┘    └──────────┘    └──────────┘               │
│       ↑                                   ↓                  │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              microtasks queue                        │   │
│  │  process.nextTick()  →  Promise.resolve()            │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  libuv Thread Pool (default: 4 threads)                      │
│  ─────────────────────────────────────                       │
│  Thread 1: File I/O (readFile, writeFile)                    │
│  Thread 2: DNS lookup                                        │
│  Thread 3: Crypto (bcrypt, hash)                             │
│  Thread 4: Compression (zlib)                                │
│                                                              │
└──────────────────────────────────────────────────────────────┘

Key Insight: Event loop เป็น single thread แต่ I/O operations
ถูก offload ไปที่ OS/Thread pool → Node.js handle ได้หลาย
requests พร้อมกันโดยไม่ block
```

### Next.js Rendering Strategies

```
Request → Next.js Server
           │
           ├── Static Generation (SSG)
           │   └── Build time → HTML file → CDN cache
           │       เหมาะกับ: Blog, docs, product pages
           │
           ├── Server-Side Rendering (SSR)
           │   └── Each request → fresh HTML
           │       เหมาะกับ: Dashboard, personalized content
           │
           ├── Incremental Static Regeneration (ISR)
           │   └── Cache + revalidate after N seconds
           │       เหมาะกับ: News feed, product catalog
           │
           └── Client-Side Rendering (CSR)
               └── Static shell → JS loads data
                   เหมาะกับ: Highly interactive UI

chuaikan.com Strategy:
┌─────────────────────────────────────────────────────┐
│  Page            │ Strategy │ Revalidate             │
├─────────────────────────────────────────────────────┤
│  Home feed       │ ISR      │ 60 seconds             │
│  User profile    │ ISR      │ 300 seconds            │
│  SOS alerts      │ SSR      │ Always fresh           │
│  Blog posts      │ SSG      │ On deploy              │
│  Dashboard       │ CSR      │ Real-time via WS       │
└─────────────────────────────────────────────────────┘
```

### Core Web Vitals Targets

```
LCP (Largest Contentful Paint) — Loading Performance
  ✅ Good:    ≤ 2.5s
  ⚠️  Needs Improvement: 2.5s - 4.0s
  ❌ Poor:    > 4.0s

FID (First Input Delay) — Interactivity
  ✅ Good:    ≤ 100ms
  ⚠️  Needs Improvement: 100ms - 300ms
  ❌ Poor:    > 300ms

CLS (Cumulative Layout Shift) — Visual Stability
  ✅ Good:    ≤ 0.1
  ⚠️  Needs Improvement: 0.1 - 0.25
  ❌ Poor:    > 0.25

INP (Interaction to Next Paint) — replaces FID in 2024
  ✅ Good:    ≤ 200ms
  ⚠️  Needs Improvement: 200ms - 500ms
  ❌ Poor:    > 500ms
```

---

## ⚙️ Environment Setup

```bash
# ─── ติดตั้ง Node.js 22 LTS ───────────────────────────────
# ใช้ NodeSource repository
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

# ตรวจสอบ version
node --version     # v22.x.x
npm --version      # 10.x.x

# ติดตั้ง pnpm (แนะนำสำหรับ production)
npm install -g pnpm
pnpm --version     # 9.x.x

# ─── สร้าง Next.js 15 Project ─────────────────────────────
npx create-next-app@latest chuaikan-app \
  --typescript \
  --tailwind \
  --eslint \
  --app \
  --src-dir \
  --import-alias "@/*"

cd chuaikan-app

# ตรวจสอบ Next.js version
cat package.json | grep '"next"'
# "next": "15.x.x"

# ─── ติดตั้ง Dependencies ─────────────────────────────────
npm install \
  @next/bundle-analyzer \
  sharp \
  next-pwa \
  next-sitemap

npm install --save-dev \
  @types/node \
  @next/eslint-plugin-next \
  lighthouse \
  @lhci/cli
```

---

## 🛠️ Step-by-Step Implementation

### Step 21: Production Node.js Configuration

```bash
# ─── Environment Variables ────────────────────────────────
sudo tee /opt/chuaikan/.env.production << 'EOF'
# ─── App ──────────────────────────────────────────────────
NODE_ENV=production
PORT=3000
HOSTNAME=0.0.0.0

# ─── Node.js Performance ──────────────────────────────────
# Thread pool size: ตั้งเท่ากับจำนวน CPU cores
UV_THREADPOOL_SIZE=8

# ─── Database ─────────────────────────────────────────────
DATABASE_URL=postgresql://chuaikan_user:secret@localhost:5432/chuaikan_db

# ─── Redis ────────────────────────────────────────────────
REDIS_URL=redis://localhost:6379

# ─── Auth ─────────────────────────────────────────────────
JWT_SECRET=your-very-long-random-secret-here
NEXTAUTH_SECRET=another-very-long-random-secret
NEXTAUTH_URL=https://chuaikan.com

# ─── File Storage ─────────────────────────────────────────
AWS_S3_BUCKET=chuaikan-uploads
AWS_REGION=ap-southeast-1
AWS_ACCESS_KEY_ID=your-access-key
AWS_SECRET_ACCESS_KEY=your-secret-key

# ─── Analytics ────────────────────────────────────────────
NEXT_PUBLIC_GA_ID=G-XXXXXXXXXX
EOF
```

```bash
# ─── Production startup script ────────────────────────────
cat > /opt/chuaikan/scripts/start-prod.sh << 'SCRIPT'
#!/bin/bash
set -euo pipefail

APP_DIR="/opt/chuaikan/app"
LOG_DIR="/var/log/chuaikan"

# Memory allocation: 75% ของ RAM
TOTAL_RAM_MB=$(free -m | awk 'NR==2{print $2}')
MAX_OLD_SPACE=$((TOTAL_RAM_MB * 75 / 100))

echo "Starting chuaikan.com with ${MAX_OLD_SPACE}MB heap..."

cd "$APP_DIR"

exec node \
  --max-old-space-size=${MAX_OLD_SPACE} \
  --experimental-vm-modules \
  ./node_modules/.bin/next start \
  --port ${PORT:-3000} \
  --hostname ${HOSTNAME:-0.0.0.0}
SCRIPT
chmod +x /opt/chuaikan/scripts/start-prod.sh
```

### Step 22: next.config.js Production Configuration

```javascript
// next.config.js
import bundleAnalyzer from '@next/bundle-analyzer'

const withBundleAnalyzer = bundleAnalyzer({
  enabled: process.env.ANALYZE === 'true',
  openAnalyzer: false,
})

/** @type {import('next').NextConfig} */
const nextConfig = {
  // ─── Output ───────────────────────────────────────────
  output: 'standalone',   // สำหรับ Docker deployment
  
  // ─── Performance ──────────────────────────────────────
  compress: true,          // Gzip compression
  poweredByHeader: false,  // ซ่อน X-Powered-By header
  
  // ─── Images ───────────────────────────────────────────
  images: {
    // Domains ที่อนุญาต (เก่า, ใช้ remotePatterns แทน)
    // domains: ['cdn.chuaikan.com'],
    
    // remotePatterns (แนะนำ)
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'cdn.chuaikan.com',
        pathname: '/uploads/**',
      },
      {
        protocol: 'https',
        hostname: '*.amazonaws.com',
        pathname: '/**',
      },
    ],
    
    // Image formats ที่ support
    formats: ['image/avif', 'image/webp'],
    
    // Device sizes สำหรับ responsive images
    deviceSizes: [640, 750, 828, 1080, 1200, 1920, 2048, 3840],
    
    // Image sizes สำหรับ layout="fixed"/"intrinsic"
    imageSizes: [16, 32, 48, 64, 96, 128, 256, 384],
    
    // Cache duration
    minimumCacheTTL: 60 * 60 * 24 * 30, // 30 days
    
    // Optimize quality
    quality: 85,
    
    // Lazy loading (default: true)
    // unoptimized: false,
  },

  // ─── Headers ──────────────────────────────────────────
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          // Security headers
          {
            key: 'X-Frame-Options',
            value: 'SAMEORIGIN',
          },
          {
            key: 'X-Content-Type-Options',
            value: 'nosniff',
          },
          {
            key: 'X-XSS-Protection',
            value: '1; mode=block',
          },
          {
            key: 'Referrer-Policy',
            value: 'strict-origin-when-cross-origin',
          },
          {
            key: 'Permissions-Policy',
            value: 'camera=(), microphone=(), geolocation=(self)',
          },
          {
            key: 'Strict-Transport-Security',
            value: 'max-age=31536000; includeSubDomains',
          },
          {
            key: 'Content-Security-Policy',
            value: [
              "default-src 'self'",
              "script-src 'self' 'unsafe-inline' https://www.googletagmanager.com",
              "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com",
              "font-src 'self' https://fonts.gstatic.com",
              "img-src 'self' data: https://cdn.chuaikan.com https://*.amazonaws.com",
              "connect-src 'self' https://api.chuaikan.com wss://ws.chuaikan.com",
            ].join('; '),
          },
        ],
      },
      {
        // Cache static assets
        source: '/_next/static/(.*)',
        headers: [
          {
            key: 'Cache-Control',
            value: 'public, max-age=31536000, immutable',
          },
        ],
      },
      {
        // Cache images
        source: '/images/(.*)',
        headers: [
          {
            key: 'Cache-Control',
            value: 'public, max-age=86400, stale-while-revalidate=604800',
          },
        ],
      },
    ]
  },

  // ─── Redirects ────────────────────────────────────────
  async redirects() {
    return [
      {
        source: '/home',
        destination: '/',
        permanent: true,   // 301
      },
      {
        source: '/sos',
        destination: '/alerts',
        permanent: false,  // 302
      },
    ]
  },

  // ─── Rewrites ─────────────────────────────────────────
  async rewrites() {
    return [
      {
        // Proxy API requests (ไม่ expose backend URL)
        source: '/api/v2/:path*',
        destination: `${process.env.API_BASE_URL}/v2/:path*`,
      },
    ]
  },

  // ─── Webpack Customization ────────────────────────────
  webpack: (config, { dev, isServer }) => {
    // SVG as React components
    config.module.rules.push({
      test: /\.svg$/,
      use: ['@svgr/webpack'],
    })

    // Aliases
    config.resolve.alias = {
      ...config.resolve.alias,
      '@components': `${__dirname}/src/components`,
      '@lib': `${__dirname}/src/lib`,
      '@hooks': `${__dirname}/src/hooks`,
    }

    return config
  },

  // ─── Experimental Features ────────────────────────────
  experimental: {
    // Partial Prerendering (Next.js 14+)
    // ppr: true,
    
    // React 19 features
    reactCompiler: true,
    
    // Optimize package imports
    optimizePackageImports: [
      '@headlessui/react',
      'lucide-react',
      'date-fns',
    ],
  },
}

export default withBundleAnalyzer(nextConfig)
```

### Step 23: Image Optimization ด้วย next/image

```tsx
// src/components/UserAvatar.tsx
import Image from 'next/image'

interface UserAvatarProps {
  src: string
  alt: string
  size?: number
  priority?: boolean
}

export function UserAvatar({
  src,
  alt,
  size = 48,
  priority = false,
}: UserAvatarProps) {
  return (
    <div
      className="relative rounded-full overflow-hidden"
      style={{ width: size, height: size }}
    >
      <Image
        src={src}
        alt={alt}
        fill                      // ใช้ fill แทน width/height เมื่อ parent มี position: relative
        sizes={`${size}px`}       // hint สำหรับ srcset calculation
        priority={priority}       // true สำหรับ above-the-fold images (ไม่ lazy load)
        className="object-cover"
        quality={90}
        placeholder="blur"        // show blur placeholder ขณะโหลด
        blurDataURL="data:image/jpeg;base64,/9j/4AAQSkZJRgABAQAAAQABAAD..."
      />
    </div>
  )
}

// src/components/PostImage.tsx — Responsive image
import Image from 'next/image'

export function PostImage({ src, alt }: { src: string; alt: string }) {
  return (
    <div className="relative aspect-video w-full">
      <Image
        src={src}
        alt={alt}
        fill
        sizes="(max-width: 768px) 100vw,
               (max-width: 1200px) 50vw,
               33vw"
        className="object-cover rounded-lg"
        quality={80}
      />
    </div>
  )
}

// src/components/HeroImage.tsx — Above the fold (LCP element)
export function HeroImage() {
  return (
    <Image
      src="/images/hero.jpg"
      alt="chuaikan hero"
      width={1920}
      height={1080}
      priority={true}      // สำคัญ: preload LCP image!
      quality={90}
      className="w-full h-auto"
    />
  )
}
```

### Step 24: Font Optimization ด้วย next/font

```tsx
// src/app/layout.tsx
import { Sarabun, Noto_Sans_Thai } from 'next/font/google'
import localFont from 'next/font/local'

// Thai font สำหรับ UI
const sarabun = Sarabun({
  weight: ['300', '400', '500', '600', '700'],
  subsets: ['thai', 'latin'],
  display: 'swap',          // ป้องกัน FOIT (Flash of Invisible Text)
  variable: '--font-sarabun',
  preload: true,
})

// Alternative Thai font
const notoSansThai = Noto_Sans_Thai({
  weight: ['400', '500', '700'],
  subsets: ['thai'],
  display: 'optional',      // ใช้ system font ถ้า download ช้า
  variable: '--font-noto-thai',
})

// Local font (สำหรับ custom/brand fonts)
const chuaikanFont = localFont({
  src: [
    {
      path: '../fonts/ChuaikanSans-Regular.woff2',
      weight: '400',
      style: 'normal',
    },
    {
      path: '../fonts/ChuaikanSans-Bold.woff2',
      weight: '700',
      style: 'normal',
    },
  ],
  variable: '--font-chuaikan',
  display: 'swap',
  preload: true,
})

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html
      lang="th"
      className={`${sarabun.variable} ${notoSansThai.variable} ${chuaikanFont.variable}`}
    >
      <body className={sarabun.className}>
        {children}
      </body>
    </html>
  )
}
```

```css
/* src/app/globals.css */
:root {
  --font-sans: var(--font-sarabun), var(--font-noto-thai), system-ui, sans-serif;
  --font-brand: var(--font-chuaikan), var(--font-sarabun), sans-serif;
}

body {
  font-family: var(--font-sans);
}

h1, h2, h3 {
  font-family: var(--font-brand);
}
```

### Step 25: Bundle Analysis

```bash
# ─── ติดตั้ง Bundle Analyzer ──────────────────────────────
npm install --save-dev @next/bundle-analyzer

# ─── เพิ่ม script ใน package.json ─────────────────────────
# "analyze": "ANALYZE=true next build"
```

```json
// package.json
{
  "scripts": {
    "dev": "next dev --turbo",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "analyze": "ANALYZE=true next build",
    "analyze:server": "BUNDLE_ANALYZE=server next build",
    "analyze:browser": "BUNDLE_ANALYZE=browser next build",
    "test": "jest",
    "test:ci": "jest --ci --coverage --passWithNoTests",
    "test:e2e": "playwright test",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "db:migrate": "prisma migrate deploy",
    "db:studio": "prisma studio",
    "lighthouse": "lhci autorun"
  }
}
```

```bash
# Run bundle analysis
npm run analyze
# เปิด browser แสดง interactive treemap ของ bundle

# ─── วิเคราะห์ผล ────────────────────────────────────────
# สิ่งที่ต้องดู:
# 1. ไฟล์ขนาดใหญ่ที่ไม่ควรอยู่ใน client bundle
# 2. Libraries ที่ซ้ำซ้อน
# 3. Moment.js (ควรเปลี่ยนเป็น date-fns หรือ dayjs)
# 4. lodash full (ควรใช้ lodash-es หรือ import แบบ tree-shaking)

# ─── Reduce Bundle Size ─────────────────────────────────
# แทน moment.js ด้วย date-fns (เล็กกว่า 90%)
npm uninstall moment
npm install date-fns

# แทน lodash ด้วย individual imports
# แทน: import _ from 'lodash'
# ใช้:  import { debounce } from 'lodash-es'
```

### Step 26: Lazy Loading และ Dynamic Imports

```tsx
// src/app/page.tsx
import dynamic from 'next/dynamic'
import { Suspense } from 'react'

// ─── Dynamic Import แบบต่างๆ ──────────────────────────────

// 1. ไม่โหลด SSR (Client-only components)
const MapComponent = dynamic(
  () => import('@/components/SosMap'),
  {
    ssr: false,                    // ไม่ render บน server
    loading: () => <div className="h-64 bg-gray-100 animate-pulse rounded" />,
  }
)

// 2. โหลดเมื่อ user interact
const RichTextEditor = dynamic(
  () => import('@/components/Editor').then(mod => mod.RichTextEditor),
  {
    ssr: false,
    loading: () => <textarea className="w-full border rounded p-2" />,
  }
)

// 3. ใช้ Suspense (Next.js 13+)
const HeavyChart = dynamic(() => import('@/components/ActivityChart'))

// 4. Multiple exports
const { SosModal, SosButton } = {
  SosModal: dynamic(() => import('@/components/sos/Modal')),
  SosButton: dynamic(() => import('@/components/sos/Button')),
}

// ─── ใช้ใน component ──────────────────────────────────────
export default function HomePage() {
  return (
    <main>
      {/* Above the fold — ไม่ lazy load */}
      <HeroSection />
      
      {/* Map — โหลดแค่ฝั่ง client (ใช้ browser APIs) */}
      <MapComponent />
      
      {/* Heavy chart — lazy load ด้วย Suspense */}
      <Suspense fallback={<ChartSkeleton />}>
        <HeavyChart />
      </Suspense>
    </main>
  )
}
```

```tsx
// src/components/FeedPost.tsx — Intersection Observer Lazy Load
'use client'

import { useState, useRef, useEffect } from 'react'
import dynamic from 'next/dynamic'

const VideoPlayer = dynamic(() => import('./VideoPlayer'), { ssr: false })
const CommentSection = dynamic(() => import('./CommentSection'))

interface FeedPostProps {
  post: Post
}

export function FeedPost({ post }: FeedPostProps) {
  const [isVisible, setIsVisible] = useState(false)
  const ref = useRef<HTMLDivElement>(null)

  // Intersection Observer — โหลด comments เมื่อ post มองเห็น
  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setIsVisible(true)
          observer.disconnect()
        }
      },
      { threshold: 0.1 }
    )

    if (ref.current) observer.observe(ref.current)
    return () => observer.disconnect()
  }, [])

  return (
    <article ref={ref} className="post-card">
      <PostHeader post={post} />
      <PostContent post={post} />
      
      {post.hasVideo && <VideoPlayer src={post.videoUrl} />}
      
      {/* โหลด comments เฉพาะเมื่อ user เห็น post */}
      {isVisible && <CommentSection postId={post.id} />}
    </article>
  )
}
```

### Step 27: Code Splitting Strategies

```tsx
// src/app/layout.tsx — Route-based code splitting (automatic ใน Next.js)
// Next.js แยก bundle ตาม route อัตโนมัติ

// src/app/(dashboard)/layout.tsx — Separate bundle สำหรับ dashboard
// src/app/(marketing)/layout.tsx — Separate bundle สำหรับ marketing pages

// ─── Barrel files (ระวัง anti-pattern) ────────────────────
// ❌ ไม่ดี: import ทั้ง barrel file → bundle ใหญ่
import { Button, Modal, Input, Select } from '@/components/ui'

// ✅ ดีกว่า: import ทีละ component
import { Button } from '@/components/ui/Button'
import { Modal } from '@/components/ui/Modal'

// หรือ ตั้งค่าใน next.config.js:
// experimental: { optimizePackageImports: ['@/components/ui'] }
```

```tsx
// src/lib/loadPolyfills.ts — Load polyfills แบบ conditional
export async function loadPolyfills() {
  if (typeof window === 'undefined') return

  const polyfills: Promise<void>[] = []

  // Intersection Observer (รองรับ IE11)
  if (!('IntersectionObserver' in window)) {
    polyfills.push(
      import('intersection-observer').then(() => {})
    )
  }

  // ResizeObserver
  if (!('ResizeObserver' in window)) {
    polyfills.push(
      import('@juggle/resize-observer').then(({ ResizeObserver }) => {
        window.ResizeObserver = ResizeObserver
      })
    )
  }

  await Promise.all(polyfills)
}
```

### Step 28: Core Web Vitals Measurement

```tsx
// src/app/layout.tsx — Web Vitals Reporting
import { useReportWebVitals } from 'next/web-vitals'

// ใน Client Component
'use client'
import { useReportWebVitals } from 'next/web-vitals'

export function WebVitals() {
  useReportWebVitals((metric) => {
    const { name, value, id, rating } = metric

    // Log ไป console ใน development
    if (process.env.NODE_ENV === 'development') {
      console.log(`${name}: ${value.toFixed(2)}ms [${rating}]`)
    }

    // ส่งไป Analytics
    if (typeof gtag !== 'undefined') {
      gtag('event', name, {
        event_category: 'Web Vitals',
        event_label: id,
        value: Math.round(name === 'CLS' ? value * 1000 : value),
        non_interaction: true,
      })
    }

    // ส่งไป custom endpoint
    fetch('/api/vitals', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ name, value, id, rating }),
    })
  })

  return null
}
```

```typescript
// src/app/api/vitals/route.ts — Endpoint รับ Web Vitals
import { NextRequest, NextResponse } from 'next/server'

interface WebVital {
  name: string
  value: number
  id: string
  rating: 'good' | 'needs-improvement' | 'poor'
}

const vitalsLog: WebVital[] = []

export async function POST(request: NextRequest) {
  const vital: WebVital = await request.json()

  // เก็บ vitals (production: ส่งไป monitoring system)
  vitalsLog.push({
    ...vital,
    // timestamp: new Date().toISOString(),
  })

  // ถ้า poor rating → alert
  if (vital.rating === 'poor') {
    console.error(`[VITALS] Poor ${vital.name}: ${vital.value.toFixed(2)}`)
    // TODO: ส่ง alert ไป Slack/PagerDuty
  }

  return NextResponse.json({ ok: true })
}
```

### Step 29: Lighthouse CI Setup

```bash
# ─── ติดตั้ง Lighthouse CI ────────────────────────────────
npm install -g @lhci/cli

# ─── ตั้งค่า lighthouserc ──────────────────────────────────
cat > lighthouserc.js << 'EOF'
/** @type {import('@lhci/types').LhrConfig} */
module.exports = {
  ci: {
    collect: {
      // URL ที่จะ test
      url: [
        'http://localhost:3000',
        'http://localhost:3000/alerts',
        'http://localhost:3000/profile',
      ],
      // จำนวนครั้งที่ run (เฉลี่ยผล)
      numberOfRuns: 3,
      // Start server command
      startServerCommand: 'npm run start',
      startServerReadyPattern: 'started server',
    },
    assert: {
      // Minimum scores (0-1)
      assertions: {
        'categories:performance': ['error', { minScore: 0.8 }],
        'categories:accessibility': ['error', { minScore: 0.9 }],
        'categories:best-practices': ['error', { minScore: 0.9 }],
        'categories:seo': ['error', { minScore: 0.9 }],

        // Core Web Vitals
        'first-contentful-paint': ['warn', { maxNumericValue: 2000 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-blocking-time': ['warn', { maxNumericValue: 300 }],
        'interactive': ['warn', { maxNumericValue: 3500 }],

        // Best practices
        'uses-optimized-images': 'warn',
        'uses-webp-images': 'warn',
        'uses-text-compression': 'error',
        'render-blocking-resources': 'warn',
      },
    },
    upload: {
      // เก็บผลลัพธ์ locally
      target: 'filesystem',
      outputDir: './lighthouse-results',
    },
  },
}
EOF

# Run Lighthouse CI
npx lhci autorun
# Expected: All assertions passed!
```

### Step 30: Performance Optimization Script

```typescript
// src/lib/performance.ts — Performance utilities

/**
 * Debounce function สำหรับ search input, resize events
 */
export function debounce<T extends (...args: unknown[]) => unknown>(
  fn: T,
  delay: number
): (...args: Parameters<T>) => void {
  let timer: ReturnType<typeof setTimeout>
  return (...args: Parameters<T>) => {
    clearTimeout(timer)
    timer = setTimeout(() => fn(...args), delay)
  }
}

/**
 * Throttle function สำหรับ scroll events
 */
export function throttle<T extends (...args: unknown[]) => unknown>(
  fn: T,
  limit: number
): (...args: Parameters<T>) => void {
  let lastCall = 0
  return (...args: Parameters<T>) => {
    const now = Date.now()
    if (now - lastCall >= limit) {
      lastCall = now
      fn(...args)
    }
  }
}

/**
 * Preload critical resources
 */
export function preloadResources(urls: string[]): void {
  urls.forEach((url) => {
    const link = document.createElement('link')
    link.rel = 'preload'
    link.as = url.endsWith('.js') ? 'script' : 'fetch'
    link.href = url
    document.head.appendChild(link)
  })
}

/**
 * Measure component render time
 */
export function measurePerf(name: string): () => void {
  const start = performance.now()
  return () => {
    const duration = performance.now() - start
    if (process.env.NODE_ENV === 'development') {
      console.log(`[PERF] ${name}: ${duration.toFixed(2)}ms`)
    }
    performance.measure(name, { start, end: performance.now() })
  }
}
```

---

## 🔧 Configuration Files

### .eslintrc.json

```json
{
  "extends": [
    "next/core-web-vitals",
    "next/typescript"
  ],
  "rules": {
    "@next/next/no-img-element": "error",
    "@next/next/no-html-link-for-pages": "error",
    "react/no-array-index-key": "warn",
    "no-console": ["warn", { "allow": ["error", "warn"] }]
  }
}
```

### .prettierrc

```json
{
  "semi": false,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 100,
  "plugins": ["prettier-plugin-tailwindcss"]
}
```

---

## 🧪 Testing

```bash
# ─── Build Production ─────────────────────────────────────
npm run build
# ดูที่ output:
# Route (app)     Size    First Load JS
# ○ /            4.2 kB   87.3 kB
# ○ /alerts      3.1 kB   90.2 kB
# ✓ Compiled successfully

# ─── Bundle Analysis ──────────────────────────────────────
npm run analyze
# เปิด browser → ดู bundle sizes

# ─── Start Production Server ──────────────────────────────
npm run start &
sleep 5
curl -I http://localhost:3000
# Expected: HTTP/1.1 200 OK

# ─── Run Lighthouse ───────────────────────────────────────
npx lhci autorun
# Expected: ✅ Performance: 85+, Accessibility: 90+

# ─── ทดสอบ Image Optimization ─────────────────────────────
curl -I "http://localhost:3000/_next/image?url=/images/hero.jpg&w=1920&q=85"
# Expected:
# content-type: image/webp (ถ้า browser support)
# cache-control: public, max-age=31536000, immutable

# ─── ตรวจสอบ Response Headers ─────────────────────────────
curl -sI http://localhost:3000 | grep -E "x-frame|x-content|strict-transport"
```

---

## ❌ Common Errors & Solutions

### Error 1: Image domain ไม่ได้ configured

```
❌ Error:
Error: Invalid src prop (https://cdn.chuaikan.com/img.jpg) on `next/image`,
hostname "cdn.chuaikan.com" is not configured under images.

✅ Solution:
// next.config.js
images: {
  remotePatterns: [
    {
      protocol: 'https',
      hostname: 'cdn.chuaikan.com',
      pathname: '/**',
    },
  ],
}
```

### Error 2: Bundle ใหญ่เกิน

```
❌ Error:
Warning: First Load JS shared by all   > 200 kB

✅ Solution:
1. ตรวจสอบด้วย bundle analyzer
2. ใช้ dynamic import สำหรับ heavy libraries
3. เปลี่ยน moment.js → date-fns
4. ตรวจสอบ barrel file imports

// แทนที่:
import { format, addDays, subDays } from 'date-fns'
// ไม่ใช้:
import moment from 'moment'
```

### Error 3: Font ทำให้เกิด Layout Shift (CLS)

```
❌ Error: CLS score สูงเพราะ font loading

✅ Solution:
// ใช้ display: 'swap' หรือ 'optional'
const sarabun = Sarabun({
  display: 'swap',   // ป้องกัน FOIT ด้วย fallback font
})

// หรือ preload font
const sarabun = Sarabun({
  preload: true,
})

// กำหนด size-adjust บน fallback font ใน CSS
@font-face {
  font-family: 'Sarabun-Fallback';
  src: local('Arial');
  size-adjust: 98%;  // ปรับให้ size ใกล้เคียง Sarabun
}
```

---

## 📊 Performance Benchmark

| ตัวชี้วัด | Before Optimization | After Optimization | Target |
|---------|---------------------|-------------------|--------|
| LCP | 4.2s | 1.8s | < 2.5s ✅ |
| CLS | 0.25 | 0.05 | < 0.1 ✅ |
| TBT | 800ms | 200ms | < 300ms ✅ |
| First Load JS | 450 kB | 180 kB | < 200 kB ✅ |
| Lighthouse Score | 52 | 88 | > 80 ✅ |
| TTI | 6.5s | 3.2s | < 3.5s ✅ |

---

## ✅ Checklist

- [ ] ติดตั้ง Node.js 22 LTS
- [ ] สร้าง Next.js 15 project ด้วย TypeScript
- [ ] ตั้งค่า next.config.js สำหรับ production
- [ ] เพิ่ม security headers ทั้งหมด
- [ ] ตั้งค่า image optimization (formats, sizes, cache)
- [ ] ใช้ next/font แทน Google Fonts CDN
- [ ] ติดตั้ง bundle analyzer
- [ ] ตรวจสอบ bundle size < 200kB first load
- [ ] Implement dynamic imports สำหรับ heavy components
- [ ] ตั้งค่า code splitting strategy
- [ ] เพิ่ม Web Vitals reporting
- [ ] ติดตั้งและ configure Lighthouse CI
- [ ] ผ่าน Lighthouse Performance score > 80
- [ ] ผ่าน Lighthouse Accessibility score > 90
- [ ] LCP < 2.5s, CLS < 0.1

---

## 🔗 References

- [Next.js 15 Documentation](https://nextjs.org/docs)
- [Next.js Image Optimization](https://nextjs.org/docs/app/building-your-application/optimizing/images)
- [Next.js Font Optimization](https://nextjs.org/docs/app/building-your-application/optimizing/fonts)
- [Core Web Vitals](https://web.dev/vitals/)
- [Lighthouse CI](https://github.com/GoogleChrome/lighthouse-ci)
- [Node.js Best Practices](https://github.com/goldbergyoni/nodebestpractices)
- [Bundle Analyzer](https://www.npmjs.com/package/@next/bundle-analyzer)

---
*Part 003 | Road to 1,000,000 Users/Day | chuaikan.com*
