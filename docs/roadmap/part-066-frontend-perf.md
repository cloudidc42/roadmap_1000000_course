# Part 066: Frontend Performance (Core Web Vitals)

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Advanced
> **Steps:** 651-660
> **เวลาโดยประมาณ:** 5 ชั่วโมง
> **Prerequisites:** Part 065 (Network Optimization)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- Core Web Vitals 2024: LCP, INP, CLS
- Next.js 15 performance features
- Image optimization strategy
- JavaScript bundle size audit
- Critical CSS extraction
- Font loading optimization
- Third-party script management
- Real User Monitoring (RUM)
- Lighthouse CI ใน GitHub Actions

---

## 📖 ทฤษฎีและแนวคิด

### Core Web Vitals 2024

```
┌────────────────────────────────────────────────────────┐
│                  Core Web Vitals                        │
├──────────────┬─────────────────┬───────────────────────┤
│   Metric     │    Good         │    Needs Improvement  │
├──────────────┼─────────────────┼───────────────────────┤
│ LCP          │ ≤ 2.5s          │ 2.5s - 4.0s           │
│ (Loading)    │ Largest element │                        │
├──────────────┼─────────────────┼───────────────────────┤
│ INP          │ ≤ 200ms         │ 200ms - 500ms          │
│ (Interaction)│ Replaced FID    │                        │
├──────────────┼─────────────────┼───────────────────────┤
│ CLS          │ ≤ 0.1           │ 0.1 - 0.25            │
│ (Stability)  │ Layout shift    │                        │
└──────────────┴─────────────────┴───────────────────────┘

INP (Interaction to Next Paint) แทนที่ FID ตั้งแต่ March 2024
วัด: Keyboard, pointer, touch interactions
```

---

## ⚙️ Environment Setup

```bash
# ติดตั้ง Next.js 15 project
npx create-next-app@15 chuaikan-web \
  --typescript \
  --tailwind \
  --eslint \
  --app \
  --src-dir \
  --import-alias "@/*"

cd chuaikan-web

# ติดตั้ง tools
npm install -D \
  @next/bundle-analyzer \
  lighthouse \
  @lhci/cli \
  web-vitals \
  @vercel/analytics

# ตรวจสอบ dependencies
npm list --depth=0
```

---

## 🛠️ Step-by-Step Implementation

### Step 651: Core Web Vitals Measurement

```typescript
// src/lib/web-vitals.ts
import { onCLS, onINP, onLCP, onFCP, onTTFB, type Metric } from 'web-vitals';

interface VitalsReport {
  name: string;
  value: number;
  rating: 'good' | 'needs-improvement' | 'poor';
  delta: number;
  id: string;
}

function sendToAnalytics(metric: Metric) {
  const report: VitalsReport = {
    name: metric.name,
    value: metric.value,
    rating: metric.rating,
    delta: metric.delta,
    id: metric.id,
  };
  
  // Send to our analytics endpoint
  if (navigator.sendBeacon) {
    navigator.sendBeacon('/api/vitals', JSON.stringify(report));
  } else {
    fetch('/api/vitals', {
      method: 'POST',
      body: JSON.stringify(report),
      headers: { 'Content-Type': 'application/json' },
      keepalive: true,
    });
  }
  
  // Log in development
  if (process.env.NODE_ENV === 'development') {
    console.log(`[Web Vital] ${metric.name}:`, {
      value: metric.value.toFixed(2),
      rating: metric.rating,
    });
  }
}

export function initWebVitals() {
  onCLS(sendToAnalytics);
  onINP(sendToAnalytics);   // Interaction to Next Paint
  onLCP(sendToAnalytics);
  onFCP(sendToAnalytics);
  onTTFB(sendToAnalytics);
}
```

```typescript
// src/app/layout.tsx - Initialize Web Vitals
'use client';
import { useEffect } from 'react';
import { initWebVitals } from '@/lib/web-vitals';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  useEffect(() => {
    initWebVitals();
  }, []);
  
  return (
    <html lang="th">
      <body>{children}</body>
    </html>
  );
}
```

```typescript
// src/app/api/vitals/route.ts - Store vitals
import { NextRequest, NextResponse } from 'next/server';

export async function POST(request: NextRequest) {
  try {
    const vital = await request.json();
    
    // Store in database for analysis
    // await db.webVitals.create({ data: vital });
    
    // Or send to monitoring service
    console.log('[Vital]', vital);
    
    return NextResponse.json({ ok: true });
  } catch (error) {
    return NextResponse.json({ error: 'Failed' }, { status: 500 });
  }
}
```

### Step 652: Next.js 15 Performance Features

```typescript
// next.config.ts
import type { NextConfig } from 'next';
import BundleAnalyzer from '@next/bundle-analyzer';

const withBundleAnalyzer = BundleAnalyzer({
  enabled: process.env.ANALYZE === 'true',
});

const nextConfig: NextConfig = {
  // Experimental features (Next.js 15)
  experimental: {
    // Partial Prerendering: static shell + dynamic holes
    ppr: true,
    
    // React 19 features
    reactCompiler: true,
    
    // Optimize package imports (tree shaking)
    optimizePackageImports: [
      'lodash',
      'date-fns',
      '@heroicons/react',
      'lucide-react',
    ],
    
    // Turbopack (stable in Next.js 15)
    turbo: {
      rules: {
        '*.svg': {
          loaders: ['@svgr/webpack'],
          as: '*.js',
        },
      },
    },
  },
  
  // Image optimization
  images: {
    formats: ['image/avif', 'image/webp'],
    deviceSizes: [640, 750, 828, 1080, 1200, 1920],
    imageSizes: [16, 32, 48, 64, 96, 128, 256, 384],
    minimumCacheTTL: 86400,  // 24 hours
    domains: ['cdn.chuaikan.com'],
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'cdn.chuaikan.com',
        pathname: '/images/**',
      },
    ],
  },
  
  // Headers for performance
  async headers() {
    return [
      {
        source: '/_next/static/:path*',
        headers: [
          { key: 'Cache-Control', value: 'public, max-age=31536000, immutable' },
        ],
      },
    ];
  },
  
  // Compression
  compress: true,
  
  // Minimize in production
  swcMinify: true,
  
  // Remove console.log in production
  compiler: {
    removeConsole: process.env.NODE_ENV === 'production',
  },
};

export default withBundleAnalyzer(nextConfig);
```

```typescript
// src/app/page.tsx - Partial Prerendering (PPR) example
import { Suspense } from 'react';
import { unstable_noStore as noStore } from 'next/cache';

// Static shell (prerendered at build time)
function StaticHeader() {
  return (
    <header>
      <h1>chuaikan.com</h1>
      <nav>...</nav>
    </header>
  );
}

// Dynamic content (rendered per request)
async function DynamicFeed() {
  noStore(); // Mark as dynamic
  const posts = await fetchLatestPosts();
  return <PostList posts={posts} />;
}

export default function HomePage() {
  return (
    <>
      <StaticHeader />  {/* Static: served from CDN */}
      
      <Suspense fallback={<FeedSkeleton />}>
        <DynamicFeed />  {/* Dynamic: streamed from server */}
      </Suspense>
    </>
  );
}
```

### Step 653: Image Optimization

```typescript
// src/components/OptimizedImage.tsx
import Image from 'next/image';
import type { ImageProps } from 'next/image';

interface SmartImageProps extends Omit<ImageProps, 'src'> {
  src: string;
  priority?: boolean;
  aspectRatio?: '1:1' | '16:9' | '4:3' | '3:2';
}

export function SmartImage({ 
  src, 
  alt, 
  priority = false,
  aspectRatio = '16:9',
  ...props 
}: SmartImageProps) {
  const ratioMap = {
    '1:1': { width: 1, height: 1 },
    '16:9': { width: 16, height: 9 },
    '4:3': { width: 4, height: 3 },
    '3:2': { width: 3, height: 2 },
  };
  
  const { width, height } = ratioMap[aspectRatio];
  
  return (
    <div style={{ aspectRatio: `${width}/${height}` }} className="relative overflow-hidden">
      <Image
        src={src}
        alt={alt}
        fill
        sizes="(max-width: 640px) 100vw, (max-width: 1024px) 50vw, 33vw"
        priority={priority}
        quality={85}
        // placeholder="blur" สำหรับ local images
        style={{ objectFit: 'cover' }}
        {...props}
      />
    </div>
  );
}

// Hero image ที่เป็น LCP element
export function HeroImage({ src, alt }: { src: string; alt: string }) {
  return (
    <Image
      src={src}
      alt={alt}
      width={1920}
      height={1080}
      priority={true}     // ✅ preload LCP image
      fetchPriority="high" // ✅ browser hint
      quality={90}
      sizes="100vw"
      style={{ width: '100%', height: 'auto' }}
    />
  );
}
```

```typescript
// next.config.ts - Image optimization pipeline
// สร้าง blur placeholder สำหรับทุก images

// scripts/generate-blurdata.ts
import { getPlaiceholder } from 'plaiceholder';
import fs from 'fs';

async function generateBlurData(imagePaths: string[]) {
  const results: Record<string, string> = {};
  
  for (const path of imagePaths) {
    const buffer = fs.readFileSync(path);
    const { base64 } = await getPlaiceholder(buffer, { size: 10 });
    results[path] = base64;
  }
  
  fs.writeFileSync('./src/data/blur-data.json', JSON.stringify(results, null, 2));
  console.log('Generated blur data for', imagePaths.length, 'images');
}

generateBlurData(['./public/hero.jpg', './public/logo.png']);
```

### Step 654: Bundle Size Audit

```bash
# วิเคราะห์ bundle size
ANALYZE=true npm run build

# ดู bundle stats ใน browser
# เปิด .next/analyze/client.html

# Check specific packages
npx bundlephobia -p lodash
npx bundlephobia -p moment  # ❌ huge! ใช้ date-fns แทน
npx bundlephobia -p date-fns
```

```typescript
// ❌ BAD: Import ทั้ง library
import _ from 'lodash';
const sorted = _.sortBy(items, 'name');

// ✅ GOOD: Import เฉพาะ function ที่ใช้
import sortBy from 'lodash/sortBy';
const sorted = sortBy(items, 'name');

// ✅ BETTER: ใช้ native JavaScript
const sorted = [...items].sort((a, b) => a.name.localeCompare(b.name));

// ❌ BAD: Import ทั้ง icon library
import * as Icons from '@heroicons/react/24/solid';

// ✅ GOOD: Import เฉพาะ icon ที่ใช้
import { CheckIcon, XMarkIcon } from '@heroicons/react/24/solid';
```

```typescript
// Dynamic imports สำหรับ heavy components
import dynamic from 'next/dynamic';

// ✅ Load rich text editor เฉพาะเมื่อ user เปิด edit mode
const RichTextEditor = dynamic(() => import('@/components/RichTextEditor'), {
  loading: () => <div className="animate-pulse h-32 bg-gray-100 rounded" />,
  ssr: false,  // ปิด SSR สำหรับ editor ที่ต้องการ DOM
});

// ✅ Load map component เฉพาะเมื่อจำเป็น
const MapView = dynamic(() => import('@/components/MapView'), {
  loading: () => <MapSkeleton />,
  ssr: false,
});

// ✅ Dynamic import ด้วย intersection observer
function LazyChart({ data }: { data: any[] }) {
  const [show, setShow] = useState(false);
  const ref = useRef<HTMLDivElement>(null);
  
  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => { if (entry.isIntersecting) setShow(true); },
      { rootMargin: '200px' }
    );
    if (ref.current) observer.observe(ref.current);
    return () => observer.disconnect();
  }, []);
  
  return (
    <div ref={ref}>
      {show ? <HeavyChartComponent data={data} /> : <ChartSkeleton />}
    </div>
  );
}
```

### Step 655: Critical CSS Extraction

```typescript
// next.config.ts - ใช้ CSS optimization
const nextConfig = {
  // Next.js จัดการ CSS optimization อัตโนมัติด้วย SWC
  // แต่เราสามารถ optimize เพิ่มเติมได้

  // Purge unused CSS ด้วย Tailwind
  // tailwind.config.ts
};
```

```typescript
// tailwind.config.ts
import type { Config } from 'tailwindcss';

const config: Config = {
  content: [
    './src/pages/**/*.{js,ts,jsx,tsx,mdx}',
    './src/components/**/*.{js,ts,jsx,tsx,mdx}',
    './src/app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {
      // ใช้ CSS variables สำหรับ theme colors
      colors: {
        primary: 'hsl(var(--primary))',
        secondary: 'hsl(var(--secondary))',
      },
      animation: {
        // Prefers-reduced-motion สำหรับ accessibility
        'spin-slow': 'spin 3s linear infinite',
      },
    },
  },
  plugins: [],
  // Safelist classes ที่ dynamic generate (ไม่ purge)
  safelist: [
    'text-red-500',
    'text-green-500',
    { pattern: /bg-(red|green|blue)-(100|500|900)/ },
  ],
};

export default config;
```

```html
<!-- Critical CSS inline สำหรับ above-the-fold content -->
<!-- ใน src/app/layout.tsx -->
<head>
  <style dangerouslySetInnerHTML={{ __html: `
    /* Critical styles สำหรับ initial render */
    :root { font-family: var(--font-inter), sans-serif; }
    body { margin: 0; background: #fff; }
    .header { height: 64px; background: #fff; border-bottom: 1px solid #e5e7eb; }
    /* ... minimal above-the-fold styles */
  `}} />
  
  {/* Non-critical CSS loaded asynchronously */}
  <link
    rel="stylesheet"
    href="/styles/non-critical.css"
    media="print"
    onLoad="this.media='all'"
  />
</head>
```

### Step 656: Font Loading Optimization

```typescript
// src/app/layout.tsx - Next.js font optimization
import { Inter, Sarabun } from 'next/font/google';

// ✅ ดีที่สุด: next/font โหลด font ที่ build time
// ไม่มี external network request ใน production
const inter = Inter({
  subsets: ['latin'],
  display: 'swap',
  variable: '--font-inter',
  // Preload ทั้ง weights ที่ใช้
  weight: ['400', '500', '600', '700'],
  // Preload Latin subset เท่านั้น (ลด download)
});

// Thai font สำหรับ chuaikan.com
const sarabun = Sarabun({
  subsets: ['thai', 'latin'],
  display: 'swap',
  variable: '--font-sarabun',
  weight: ['300', '400', '500', '600', '700'],
  // Font display swap: ใช้ fallback font ก่อน แล้ว swap เมื่อโหลดเสร็จ
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="th" className={`${inter.variable} ${sarabun.variable}`}>
      <body>{children}</body>
    </html>
  );
}
```

```css
/* globals.css */
:root {
  --font-inter: 'Inter', sans-serif;
  --font-sarabun: 'Sarabun', sans-serif;
}

body {
  font-family: var(--font-sarabun);
}

/* ใช้ Inter สำหรับ headings */
h1, h2, h3 {
  font-family: var(--font-inter);
}

/* Font size clamp สำหรับ responsive typography */
h1 {
  font-size: clamp(1.75rem, 4vw, 3rem);
}
```

### Step 657: Third-Party Script Management

```typescript
// src/components/ThirdPartyScripts.tsx
import Script from 'next/script';

export function ThirdPartyScripts() {
  return (
    <>
      {/* Google Analytics - defer loading */}
      <Script
        src={`https://www.googletagmanager.com/gtag/js?id=${process.env.NEXT_PUBLIC_GA_ID}`}
        strategy="afterInteractive"  // ✅ โหลดหลัง page interactive
      />
      <Script id="google-analytics" strategy="afterInteractive">
        {`
          window.dataLayer = window.dataLayer || [];
          function gtag(){dataLayer.push(arguments);}
          gtag('js', new Date());
          gtag('config', '${process.env.NEXT_PUBLIC_GA_ID}', {
            page_path: window.location.pathname,
          });
        `}
      </Script>
      
      {/* Intercom chat - lazyOnload */}
      <Script
        id="intercom"
        strategy="lazyOnload"  // ✅ โหลดหลัง page fully loaded
        dangerouslySetInnerHTML={{
          __html: `
            window.intercomSettings = {
              api_base: "https://api-iam.intercom.io",
              app_id: "${process.env.NEXT_PUBLIC_INTERCOM_APP_ID}"
            };
          `
        }}
      />
    </>
  );
}

// Script loading strategies:
// "beforeInteractive" - โหลดก่อน hydration (critical scripts only!)
// "afterInteractive"  - โหลดหลัง page interactive (default สำหรับ analytics)
// "lazyOnload"        - โหลดหลัง page fully loaded (non-critical)
// "worker"            - รันใน Web Worker (experimental)
```

```typescript
// src/hooks/useGTM.ts - Google Tag Manager ด้วย useEffect
import { useEffect } from 'react';
import { usePathname, useSearchParams } from 'next/navigation';

export function useGTM() {
  const pathname = usePathname();
  const searchParams = useSearchParams();
  
  useEffect(() => {
    // Track page views on navigation
    if (typeof window !== 'undefined' && window.gtag) {
      window.gtag('config', process.env.NEXT_PUBLIC_GA_ID!, {
        page_path: pathname + (searchParams.toString() ? `?${searchParams}` : ''),
      });
    }
  }, [pathname, searchParams]);
}
```

### Step 658: Real User Monitoring (RUM)

```typescript
// src/lib/rum.ts - Custom RUM implementation
interface RUMEvent {
  type: 'navigation' | 'resource' | 'paint' | 'longtask' | 'layout-shift';
  name: string;
  value: number;
  url: string;
  connection?: string;
  deviceType?: string;
  sessionId: string;
}

class RealUserMonitor {
  private sessionId: string;
  private queue: RUMEvent[] = [];
  private observer: PerformanceObserver | null = null;
  
  constructor() {
    this.sessionId = this.generateSessionId();
    this.setupObservers();
    this.scheduleFlush();
  }
  
  private generateSessionId(): string {
    return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
  
  private getDeviceType(): string {
    const ua = navigator.userAgent;
    if (/mobile/i.test(ua)) return 'mobile';
    if (/tablet/i.test(ua)) return 'tablet';
    return 'desktop';
  }
  
  private setupObservers() {
    // Monitor Long Tasks (blocks main thread > 50ms)
    if ('PerformanceObserver' in window) {
      try {
        this.observer = new PerformanceObserver((list) => {
          for (const entry of list.getEntries()) {
            if (entry.entryType === 'longtask') {
              this.track({
                type: 'longtask',
                name: 'long-task',
                value: entry.duration,
              });
            }
          }
        });
        
        this.observer.observe({ 
          type: 'longtask',
          buffered: true 
        });
      } catch (e) {
        // longtask observer not supported
      }
    }
  }
  
  track(event: Omit<RUMEvent, 'url' | 'sessionId' | 'deviceType' | 'connection'>) {
    this.queue.push({
      ...event,
      url: window.location.pathname,
      sessionId: this.sessionId,
      deviceType: this.getDeviceType(),
      connection: (navigator as any).connection?.effectiveType,
    });
  }
  
  private scheduleFlush() {
    // Flush ทุก 5 วินาที หรือเมื่อ user ออกจากหน้า
    setInterval(() => this.flush(), 5000);
    
    window.addEventListener('visibilitychange', () => {
      if (document.visibilityState === 'hidden') {
        this.flush();
      }
    });
  }
  
  private flush() {
    if (this.queue.length === 0) return;
    
    const events = [...this.queue];
    this.queue = [];
    
    if (navigator.sendBeacon) {
      navigator.sendBeacon(
        '/api/rum',
        JSON.stringify({ events })
      );
    }
  }
}

export const rum = typeof window !== 'undefined' 
  ? new RealUserMonitor() 
  : null;
```

```typescript
// src/app/api/rum/route.ts - RUM data collection
import { NextRequest, NextResponse } from 'next/server';

export async function POST(request: NextRequest) {
  try {
    const { events } = await request.json();
    
    // Process and store RUM events
    // In production: send to ClickHouse/TimescaleDB for analytics
    const processed = events.map((event: any) => ({
      ...event,
      timestamp: new Date().toISOString(),
      ip: request.ip || 'unknown',
      userAgent: request.headers.get('user-agent'),
    }));
    
    // Batch insert to database
    // await clickhouse.insert({ table: 'rum_events', values: processed });
    
    return NextResponse.json({ received: processed.length });
  } catch (error) {
    return NextResponse.json({ error: 'Failed' }, { status: 500 });
  }
}
```

### Step 659: Lighthouse CI ใน GitHub Actions

```yaml
# .github/workflows/lighthouse-ci.yml
name: Lighthouse CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  lighthouse:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Build Next.js app
      run: npm run build
      env:
        NEXT_PUBLIC_API_URL: ${{ secrets.NEXT_PUBLIC_API_URL }}
    
    - name: Start server
      run: |
        npm run start &
        sleep 5
        curl --retry 5 --retry-delay 2 http://localhost:3000
    
    - name: Run Lighthouse CI
      run: npx lhci autorun
      env:
        LHCI_GITHUB_APP_TOKEN: ${{ secrets.LHCI_GITHUB_APP_TOKEN }}
    
    - name: Upload Lighthouse results
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: lighthouse-results
        path: .lighthouseci/
```

```javascript
// lighthouserc.js - Lighthouse CI configuration
module.exports = {
  ci: {
    collect: {
      url: [
        'http://localhost:3000/',
        'http://localhost:3000/explore',
        'http://localhost:3000/sos',
      ],
      numberOfRuns: 3,  // รัน 3 ครั้ง เอา median
      settings: {
        // Simulate 4G mobile connection
        throttlingMethod: 'simulate',
        throttling: {
          rttMs: 40,
          throughputKbps: 10240,
          cpuSlowdownMultiplier: 4,
        },
        emulatedFormFactor: 'mobile',
      },
    },
    
    assert: {
      // ✅ Fail CI ถ้า scores ต่ำกว่า threshold
      assertions: {
        'categories:performance': ['error', { minScore: 0.85 }],
        'categories:accessibility': ['error', { minScore: 0.95 }],
        'categories:best-practices': ['warn', { minScore: 0.90 }],
        'categories:seo': ['warn', { minScore: 0.90 }],
        
        // Core Web Vitals
        'first-contentful-paint': ['error', { maxNumericValue: 2000 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-blocking-time': ['warn', { maxNumericValue: 300 }],
        
        // Bundle size
        'total-byte-weight': ['warn', { maxNumericValue: 512000 }],  // 500KB
      },
    },
    
    upload: {
      target: 'lhci',
      serverBaseUrl: process.env.LHCI_SERVER_URL || 'http://localhost:9001',
      token: process.env.LHCI_TOKEN,
    },
  },
};
```

### Step 660: Performance Monitoring Dashboard

```typescript
// src/components/PerformanceDashboard.tsx - Dev tool
'use client';
import { useEffect, useState } from 'react';

interface PerfMetrics {
  lcp: number | null;
  inp: number | null;
  cls: number | null;
  ttfb: number | null;
  fcp: number | null;
}

export function PerformanceDashboard() {
  const [metrics, setMetrics] = useState<PerfMetrics>({
    lcp: null, inp: null, cls: null, ttfb: null, fcp: null
  });
  
  useEffect(() => {
    if (process.env.NODE_ENV !== 'development') return;
    
    import('web-vitals').then(({ onLCP, onINP, onCLS, onTTFB, onFCP }) => {
      onLCP((m) => setMetrics(prev => ({ ...prev, lcp: m.value })));
      onINP((m) => setMetrics(prev => ({ ...prev, inp: m.value })));
      onCLS((m) => setMetrics(prev => ({ ...prev, cls: m.value })));
      onTTFB((m) => setMetrics(prev => ({ ...prev, ttfb: m.value })));
      onFCP((m) => setMetrics(prev => ({ ...prev, fcp: m.value })));
    });
  }, []);
  
  if (process.env.NODE_ENV !== 'development') return null;
  
  const getRating = (name: string, value: number | null) => {
    if (!value) return 'gray';
    const thresholds: Record<string, [number, number]> = {
      lcp: [2500, 4000], inp: [200, 500], cls: [0.1, 0.25],
      ttfb: [800, 1800], fcp: [1800, 3000],
    };
    const [good, poor] = thresholds[name] || [0, 0];
    if (value <= good) return 'green';
    if (value <= poor) return 'yellow';
    return 'red';
  };
  
  return (
    <div style={{
      position: 'fixed', bottom: 16, right: 16,
      background: '#1f2937', color: 'white',
      padding: '12px', borderRadius: '8px',
      fontSize: '12px', fontFamily: 'monospace',
      zIndex: 9999, minWidth: '200px'
    }}>
      <div style={{ fontWeight: 'bold', marginBottom: '8px' }}>⚡ Web Vitals</div>
      {Object.entries(metrics).map(([name, value]) => (
        <div key={name} style={{ display: 'flex', justifyContent: 'space-between', padding: '2px 0' }}>
          <span style={{ textTransform: 'uppercase' }}>{name}:</span>
          <span style={{ color: { green: '#10b981', yellow: '#f59e0b', red: '#ef4444', gray: '#9ca3af' }[getRating(name, value)] }}>
            {value !== null ? `${name === 'cls' ? value.toFixed(3) : Math.round(value) + 'ms'}` : '...'}
          </span>
        </div>
      ))}
    </div>
  );
}
```

---

## 🔧 Configuration Files

```json
// .lighthouse-budgets.json
[
  {
    "path": "/*",
    "timings": [
      { "metric": "first-contentful-paint", "budget": 2000 },
      { "metric": "largest-contentful-paint", "budget": 2500 },
      { "metric": "time-to-interactive", "budget": 4000 }
    ],
    "resourceSizes": [
      { "resourceType": "script", "budget": 300 },
      { "resourceType": "stylesheet", "budget": 50 },
      { "resourceType": "image", "budget": 500 },
      { "resourceType": "total", "budget": 1000 }
    ],
    "resourceCounts": [
      { "resourceType": "third-party", "budget": 5 }
    ]
  }
]
```

---

## 🧪 Testing

```bash
# รัน Lighthouse ใน CLI
npx lighthouse https://chuaikan.com \
  --output html \
  --output-path ./lighthouse-report.html \
  --only-categories performance,accessibility,best-practices,seo

# Batch test สำหรับ multiple pages
for path in "/" "/explore" "/sos" "/profile"; do
  npx lighthouse "https://chuaikan.com${path}" \
    --output json \
    --output-path "lighthouse-${path//\//-}.json" \
    --quiet
done

# ดู Performance score
cat lighthouse-report.json | \
  node -e "const d=JSON.parse(require('fs').readFileSync('/dev/stdin','utf8')); console.log('Performance:', d.categories.performance.score * 100)"
```

---

## ❌ Common Errors & Solutions

### Error 1: LCP สูงเกิน 2.5s

```bash
# ตรวจสอบ LCP element
# ใน Chrome DevTools → Performance → Record → Find LCP
# หรือใช้ Lighthouse "Opportunities" section

# Solutions:
# 1. เพิ่ม priority={true} บน LCP image
# 2. Preload LCP image ด้วย <link rel="preload">
# 3. เพิ่ม fetchPriority="high" บน LCP image
# 4. Avoid lazy loading LCP image
```

### Error 2: CLS สูงเกิน 0.1

```typescript
// ปัญหา: Image ไม่มี dimensions กำหนด → layout shift เมื่อ load
// แก้ไข: ใช้ aspect-ratio หรือ กำหนด width/height

// ❌ BAD: ไม่มี dimensions
<img src="/hero.jpg" alt="hero" />

// ✅ GOOD: กำหนด dimensions
<Image src="/hero.jpg" alt="hero" width={1920} height={1080} />
// หรือ
<div style={{ aspectRatio: '16/9', position: 'relative' }}>
  <Image src="/hero.jpg" alt="hero" fill />
</div>
```

---

## ✅ Checklist

- [ ] ตั้งค่า web-vitals measurement ใน production
- [ ] LCP < 2.5s บน mobile 4G
- [ ] INP < 200ms สำหรับ all interactions
- [ ] CLS < 0.1 บน all pages
- [ ] กำหนด width/height บน all images
- [ ] ใช้ `next/image` สำหรับ all images
- [ ] ใช้ `next/font` สำหรับ all fonts
- [ ] Dynamic import สำหรับ heavy components
- [ ] ตั้งค่า Lighthouse CI ใน GitHub Actions
- [ ] Performance score > 85 ใน Lighthouse

---

## 🔗 References

- [Core Web Vitals](https://web.dev/vitals/)
- [Next.js Performance](https://nextjs.org/docs/app/building-your-application/optimizing)
- [web-vitals library](https://github.com/GoogleChrome/web-vitals)
- [Lighthouse CI](https://github.com/GoogleChrome/lighthouse-ci)
- [next/image](https://nextjs.org/docs/app/api-reference/components/image)
- [next/font](https://nextjs.org/docs/app/api-reference/components/font)

---

*Part 066 | Road to 1,000,000 Users/Day | chuaikan.com*
