# Part 065: Network Optimization

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Advanced
> **Steps:** 641-650
> **เวลาโดยประมาณ:** 4 ชั่วโมง
> **Prerequisites:** Part 061-064

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ประโยชน์ของ HTTP/2 และ setup ใน Node.js
- HTTP/3 กับ QUIC ผ่าน Cloudflare proxy
- Response compression: Brotli vs gzip
- Keep-alive connections
- DNS prefetching และ preconnect
- TCP optimization ผ่าน kernel parameters
- Network latency measurement
- CDN cache hit rate optimization

---

## 📖 ทฤษฎีและแนวคิด

### HTTP Evolution

```
HTTP/1.1 (1997):
Client ──────────────── Server
  GET /api/users ───→
                    ←── 200 OK (body)
  GET /api/posts ───→  (ต้องรอ response ก่อน)
                    ←── 200 OK (body)

HTTP/2 (2015) - Multiplexing:
Client ══════════════════ Server
  Stream 1: GET /api/users ──→
  Stream 3: GET /api/posts ──→  (ส่งพร้อมกันได้!)
                           ←── Stream 1: users
                           ←── Stream 3: posts

HTTP/3 (2022) - QUIC (UDP-based):
  ✓ ไม่มี Head-of-line blocking
  ✓ 0-RTT connection resumption
  ✓ Better mobile performance (connection migration)
  ✓ Built-in encryption (TLS 1.3)
```

### Compression Comparison

```
Original JSON: 100KB
gzip compressed: ~30KB (70% reduction)
Brotli compressed: ~25KB (75% reduction)

Brotli ดีกว่า gzip ประมาณ 20-26% ใน text content
แต่ Brotli ใช้ CPU สูงกว่า (ใช้ pre-compression สำหรับ static files)
```

---

## ⚙️ Environment Setup

```bash
# ติดตั้ง tools
sudo apt-get install -y \
  nghttp2-client \
  brotli \
  apache2-utils \
  net-tools \
  tcpdump \
  mtr

# ติดตั้ง Node.js packages
mkdir -p /home/user/network-opt-demo
cd /home/user/network-opt-demo
npm init -y
npm install express compression http2 node-spdy

# ตรวจสอบ kernel version สำหรับ TCP optimization
uname -r
sysctl -a | grep net.ipv4 | head -20
```

---

## 🛠️ Step-by-Step Implementation

### Step 641: HTTP/2 ใน Node.js

```javascript
// http2-server.js
const http2 = require('http2');
const fs = require('fs');
const express = require('express');
const spdy = require('node-spdy');  // Express + HTTP/2

// Option 1: Native Node.js HTTP/2 (ต้องการ TLS)
function createHttp2Server() {
  const server = http2.createSecureServer({
    key: fs.readFileSync('./certs/key.pem'),
    cert: fs.readFileSync('./certs/cert.pem'),
    // HTTP/2 settings
    settings: {
      headerTableSize: 4096,
      enablePush: true,
      initialWindowSize: 65535,
      maxFrameSize: 16384,
      maxConcurrentStreams: 1000,
    }
  });
  
  server.on('stream', (stream, headers) => {
    const path = headers[':path'];
    const method = headers[':method'];
    
    if (path === '/api/users' && method === 'GET') {
      // HTTP/2 Server Push - ส่ง related resources ล่วงหน้า
      if (stream.pushAllowed) {
        // Push CSS ก่อนที่ client จะ request
        stream.pushStream({ ':path': '/styles/app.css' }, (err, pushStream) => {
          if (!err) {
            pushStream.respond({ 
              ':status': 200, 
              'content-type': 'text/css',
              'cache-control': 'max-age=86400'
            });
            pushStream.end(fs.readFileSync('./public/styles/app.css'));
          }
        });
      }
      
      stream.respond({
        ':status': 200,
        'content-type': 'application/json',
      });
      stream.end(JSON.stringify({ users: [] }));
    }
  });
  
  return server;
}

// Option 2: Express with spdy (HTTP/2 + HTTP/1.1 fallback)
const app = express();
app.use(express.json());

app.get('/api/users', (req, res) => {
  res.json({ users: [], protocol: req.httpVersion });
});

app.get('/api/posts', (req, res) => {
  res.json({ posts: [], protocol: req.httpVersion });
});

// สร้าง self-signed cert สำหรับ development
// openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes

const server = spdy.createServer(
  {
    key: fs.readFileSync('./certs/key.pem'),
    cert: fs.readFileSync('./certs/cert.pem'),
    spdy: {
      protocols: ['h2', 'spdy/3.1', 'http/1.1'],
      plain: false,
    }
  },
  app
);

server.listen(3443, () => {
  console.log('HTTP/2 server on https://localhost:3443');
});
```

```bash
# สร้าง self-signed certificate สำหรับ development
mkdir -p certs
openssl req -x509 -newkey rsa:2048 \
  -keyout certs/key.pem \
  -out certs/cert.pem \
  -days 365 \
  -nodes \
  -subj "/C=TH/ST=Bangkok/O=chuaikan/CN=localhost"

# ทดสอบ HTTP/2
# ติดตั้ง nghttp2 client
sudo apt-get install -y nghttp2-client

# ทดสอบ HTTP/2 connection
nghttp -nv https://localhost:3443/api/users

# ดู protocol ที่ใช้
curl --http2 -k -v https://localhost:3443/api/users 2>&1 | grep "< HTTP"
```

### Step 642: Nginx HTTP/2 Configuration

```nginx
# /etc/nginx/sites-available/chuaikan.com
upstream nodejs_backend {
    server 127.0.0.1:3000;
    server 127.0.0.1:3001;
    keepalive 32;  # Keep 32 idle connections to backend
}

server {
    listen 80;
    server_name chuaikan.com www.chuaikan.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name chuaikan.com www.chuaikan.com;
    
    # SSL
    ssl_certificate /etc/letsencrypt/live/chuaikan.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/chuaikan.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    
    # HTTP/2 Push Preload (ใช้ Link header)
    http2_push_preload on;
    
    # Compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css application/json application/javascript 
               text/xml application/xml application/xml+rss text/javascript;
    
    # Brotli (ต้อง install nginx-module-brotli)
    brotli on;
    brotli_comp_level 6;
    brotli_types text/plain text/css application/json application/javascript
                 text/xml application/xml text/javascript;
    
    # Keep-alive
    keepalive_timeout 65;
    keepalive_requests 1000;
    
    location /api/ {
        proxy_pass http://nodejs_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        
        # Timeouts
        proxy_connect_timeout 5s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # Buffer settings
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
    }
    
    location / {
        root /var/www/chuaikan/public;
        try_files $uri $uri/ /index.html;
        
        # Cache static files
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
    }
}
```

```bash
# ติดตั้ง Nginx Brotli module
sudo apt-get install -y libnginx-mod-http-brotli

# หรือ compile จาก source
# https://github.com/google/ngx_brotli

# Test Nginx config
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx

# ทดสอบ Brotli compression
curl -H "Accept-Encoding: br" -I https://chuaikan.com/api/users
# ควรเห็น: Content-Encoding: br
```

### Step 643: Response Compression ใน Express

```javascript
// compression-middleware.js
const express = require('express');
const compression = require('compression');
const zlib = require('zlib');

const app = express();

// ใช้ Brotli + gzip compression
app.use(compression({
  // Algorithm selection
  filter: (req, res) => {
    // Skip compression สำหรับ Server-Sent Events
    if (req.headers.accept && req.headers.accept.includes('text/event-stream')) {
      return false;
    }
    // Skip สำหรับ very small responses (< 1KB)
    return compression.filter(req, res);
  },
  
  // Compression level (1-9, default 6)
  level: 6,
  
  // Minimum response size to compress (bytes)
  threshold: 1024,  // Don't compress responses < 1KB
  
  // Memory usage (default 8)
  memLevel: 8,
}));

// Custom Brotli middleware (ดีกว่า gzip สำหรับ text)
function brotliMiddleware(req, res, next) {
  const acceptEncoding = req.headers['accept-encoding'] || '';
  
  if (!acceptEncoding.includes('br')) {
    return next();
  }
  
  const originalSend = res.send.bind(res);
  
  res.send = function(body) {
    if (typeof body === 'string' || Buffer.isBuffer(body)) {
      const shouldCompress = body.length > 1024;
      
      if (shouldCompress) {
        zlib.brotliCompress(body, {
          params: {
            [zlib.constants.BROTLI_PARAM_QUALITY]: 6,
          }
        }, (err, compressed) => {
          if (err) {
            return originalSend(body);
          }
          
          res.setHeader('Content-Encoding', 'br');
          res.setHeader('Content-Length', compressed.length);
          res.removeHeader('Transfer-Encoding');
          originalSend(compressed);
        });
        return;
      }
    }
    
    originalSend(body);
  };
  
  next();
}

// Pre-compress static assets
async function preCompressStaticFiles(directory) {
  const fs = require('fs').promises;
  const path = require('path');
  
  async function processDir(dir) {
    const files = await fs.readdir(dir, { withFileTypes: true });
    
    for (const file of files) {
      const filepath = path.join(dir, file.name);
      
      if (file.isDirectory()) {
        await processDir(filepath);
      } else if (/\.(js|css|html|json|svg|xml)$/.test(file.name)) {
        // Create .br file
        const content = await fs.readFile(filepath);
        const compressed = await new Promise((resolve, reject) => {
          zlib.brotliCompress(content, {
            params: { [zlib.constants.BROTLI_PARAM_QUALITY]: 11 } // max quality for static
          }, (err, result) => err ? reject(err) : resolve(result));
        });
        
        await fs.writeFile(`${filepath}.br`, compressed);
        
        // Create .gz file
        const gzipped = await new Promise((resolve, reject) => {
          zlib.gzip(content, { level: 9 }, (err, result) => 
            err ? reject(err) : resolve(result));
        });
        await fs.writeFile(`${filepath}.gz`, gzipped);
        
        const ratio = ((1 - compressed.length / content.length) * 100).toFixed(1);
        console.log(`${file.name}: ${content.length}B → ${compressed.length}B (${ratio}% reduction)`);
      }
    }
  }
  
  await processDir(directory);
}

app.listen(3000);
module.exports = { app, preCompressStaticFiles };
```

### Step 644: Keep-Alive Connections

```javascript
// keepalive-config.js
const http = require('http');
const https = require('https');
const express = require('express');

const app = express();

// HTTP Agent กับ keep-alive สำหรับ outbound requests
const httpAgent = new http.Agent({
  keepAlive: true,
  keepAliveMsecs: 30000,  // send keep-alive probes every 30s
  maxSockets: 50,          // max concurrent sockets
  maxFreeSockets: 10,      // max idle sockets in pool
  timeout: 60000,          // socket timeout
});

const httpsAgent = new https.Agent({
  keepAlive: true,
  keepAliveMsecs: 30000,
  maxSockets: 50,
  maxFreeSockets: 10,
  timeout: 60000,
  // TLS options
  rejectUnauthorized: process.env.NODE_ENV === 'production',
});

// ใช้ agent กับ fetch/axios
const axios = require('axios');
const axiosInstance = axios.create({
  httpAgent,
  httpsAgent,
  timeout: 10000,
});

// Express server keep-alive settings
const server = app.listen(3000);

// ตั้งค่า keep-alive timeout
server.keepAliveTimeout = 65000;  // 65 seconds (เกิน nginx 60s)
server.headersTimeout = 66000;    // ต้องเกิน keepAliveTimeout

// Monitor active connections
setInterval(() => {
  server.getConnections((err, count) => {
    if (!err) {
      console.log(`Active connections: ${count}`);
    }
  });
}, 30000);
```

### Step 645: DNS Prefetching สำหรับ Frontend

```html
<!-- ใน Next.js _document.tsx หรือ <head> -->
<head>
  <!-- DNS prefetch สำหรับ external domains ที่ใช้บ่อย -->
  <link rel="dns-prefetch" href="//api.chuaikan.com">
  <link rel="dns-prefetch" href="//cdn.chuaikan.com">
  <link rel="dns-prefetch" href="//fonts.googleapis.com">
  <link rel="dns-prefetch" href="//www.googletagmanager.com">
  
  <!-- Preconnect: DNS + TCP + TLS handshake ล่วงหน้า -->
  <!-- ใช้กับ critical third-party origins เท่านั้น -->
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  
  <!-- Prefetch resources ที่จะใช้ในหน้าถัดไป -->
  <link rel="prefetch" href="/dashboard" as="document">
  
  <!-- Preload critical resources -->
  <link rel="preload" href="/fonts/inter-regular.woff2" as="font" type="font/woff2" crossorigin>
  <link rel="preload" href="/styles/critical.css" as="style">
</head>
```

```javascript
// next.config.js - Next.js DNS prefetch configuration
/** @type {import('next').NextConfig} */
const nextConfig = {
  // Enable HTTP/2 server push hints
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          {
            key: 'Link',
            value: [
              '<https://api.chuaikan.com>; rel=preconnect',
              '<https://fonts.googleapis.com>; rel=preconnect',
            ].join(', '),
          },
        ],
      },
    ];
  },
};

module.exports = nextConfig;
```

### Step 646: TCP Kernel Optimization

```bash
# /etc/sysctl.d/99-network-performance.conf
# TCP Optimization สำหรับ high-traffic server

cat > /etc/sysctl.d/99-network-performance.conf << 'EOF'
# TCP Buffer Sizes
net.core.rmem_max = 134217728          # 128MB receive buffer
net.core.wmem_max = 134217728          # 128MB send buffer
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864

# TCP Connections
net.core.somaxconn = 65535             # max pending connections
net.core.netdev_max_backlog = 65536    # packet queue size
net.ipv4.tcp_max_syn_backlog = 65536   # max SYN queue size

# Keep-alive settings
net.ipv4.tcp_keepalive_time = 60       # start keep-alive after 60s idle
net.ipv4.tcp_keepalive_intvl = 10      # retry every 10s
net.ipv4.tcp_keepalive_probes = 6      # 6 probes before giving up

# TIME_WAIT optimization
net.ipv4.tcp_tw_reuse = 1              # reuse TIME_WAIT sockets
net.ipv4.tcp_fin_timeout = 15          # reduce TIME_WAIT from 60s to 15s
net.ipv4.tcp_max_tw_buckets = 1440000  # max TIME_WAIT sockets

# SYN cookies (DDoS protection)
net.ipv4.tcp_syncookies = 1

# Congestion control (BBR is better for high-bandwidth)
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr

# File descriptors
fs.file-max = 2097152
EOF

# Apply settings
sysctl -p /etc/sysctl.d/99-network-performance.conf

# Verify BBR is active
sysctl net.ipv4.tcp_congestion_control
# Output: net.ipv4.tcp_congestion_control = bbr

# ตั้งค่า file descriptor limits
cat >> /etc/security/limits.conf << 'EOF'
* soft nofile 1048576
* hard nofile 1048576
EOF

# ตรวจสอบ current limits
ulimit -n
```

### Step 647: Network Latency Measurement

```javascript
// network-latency-monitor.js
const http = require('http');
const https = require('https');
const { performance } = require('perf_hooks');

class NetworkLatencyMonitor {
  constructor(endpoints) {
    this.endpoints = endpoints;
    this.results = new Map();
  }
  
  async measureLatency(url, iterations = 10) {
    const latencies = [];
    
    for (let i = 0; i < iterations; i++) {
      const start = performance.now();
      
      try {
        await new Promise((resolve, reject) => {
          const client = url.startsWith('https') ? https : http;
          const req = client.get(url, { timeout: 5000 }, (res) => {
            res.resume(); // consume response
            res.on('end', resolve);
          });
          req.on('error', reject);
          req.on('timeout', () => {
            req.destroy();
            reject(new Error('Timeout'));
          });
        });
        
        latencies.push(performance.now() - start);
      } catch (err) {
        latencies.push(-1); // error marker
      }
      
      // Small delay between requests
      await new Promise(r => setTimeout(r, 100));
    }
    
    const valid = latencies.filter(l => l > 0);
    const sorted = [...valid].sort((a, b) => a - b);
    
    return {
      url,
      samples: valid.length,
      errors: latencies.length - valid.length,
      min: sorted[0]?.toFixed(2),
      max: sorted[sorted.length - 1]?.toFixed(2),
      mean: (valid.reduce((a, b) => a + b, 0) / valid.length).toFixed(2),
      p50: sorted[Math.floor(sorted.length * 0.50)]?.toFixed(2),
      p95: sorted[Math.floor(sorted.length * 0.95)]?.toFixed(2),
      p99: sorted[Math.floor(sorted.length * 0.99)]?.toFixed(2),
    };
  }
  
  async runAll() {
    const results = await Promise.all(
      this.endpoints.map(url => this.measureLatency(url))
    );
    return results;
  }
}

// Usage
async function main() {
  const monitor = new NetworkLatencyMonitor([
    'https://api.chuaikan.com/health',
    'https://api.chuaikan.com/api/users',
    'https://cdn.chuaikan.com/images/logo.webp',
  ]);
  
  console.log('Measuring network latency...');
  const results = await monitor.runAll();
  
  console.table(results.map(r => ({
    URL: r.url.replace('https://', ''),
    'Min (ms)': r.min,
    'P50 (ms)': r.p50,
    'P95 (ms)': r.p95,
    'Errors': r.errors,
  })));
}

main();
```

```bash
# Tools สำหรับ network latency measurement
# 1. MTR (My Traceroute) - เห็น latency ทุก hop
mtr --report --report-cycles 10 api.chuaikan.com

# 2. curl verbose timing
curl -w "\n\nDNS: %{time_namelookup}s\nConnect: %{time_connect}s\nSSL: %{time_appconnect}s\nTTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n" \
  -o /dev/null -s https://api.chuaikan.com/health

# 3. httpstat (curl wrapper with nice output)
npm install -g httpstat
httpstat https://api.chuaikan.com/health
```

### Step 648: CDN Cache Hit Rate Optimization

```javascript
// cache-headers.js - Optimal cache header configuration
const express = require('express');
const crypto = require('crypto');
const app = express();

// Helper: Generate ETag
function generateETag(content) {
  return crypto
    .createHash('md5')
    .update(content)
    .digest('hex');
}

// Cache strategy middleware
function cacheStrategy(options = {}) {
  return (req, res, next) => {
    const {
      maxAge = 0,
      sMaxAge = 0,       // CDN cache duration
      staleWhileRevalidate = 0,
      staleIfError = 0,
      immutable = false,
      private: isPrivate = false,
    } = options;
    
    const directives = [];
    
    if (isPrivate) {
      directives.push('private');
    } else {
      directives.push('public');
    }
    
    if (maxAge > 0) directives.push(`max-age=${maxAge}`);
    if (sMaxAge > 0) directives.push(`s-maxage=${sMaxAge}`);
    if (staleWhileRevalidate > 0) directives.push(`stale-while-revalidate=${staleWhileRevalidate}`);
    if (staleIfError > 0) directives.push(`stale-if-error=${staleIfError}`);
    if (immutable) directives.push('immutable');
    
    res.setHeader('Cache-Control', directives.join(', '));
    next();
  };
}

// Static assets: cache forever (content-hash in filename)
app.use('/static', 
  cacheStrategy({ maxAge: 31536000, sMaxAge: 31536000, immutable: true }),
  express.static('./public/static')
);

// API responses: short cache with stale-while-revalidate
app.get('/api/trending', 
  cacheStrategy({ 
    sMaxAge: 60,                    // CDN caches 60s
    staleWhileRevalidate: 600,       // serve stale content for 10 min while revalidating
    staleIfError: 3600               // serve stale for 1 hour on error
  }),
  async (req, res) => {
    const posts = await getTrendingPosts();
    const content = JSON.stringify({ posts });
    
    // ETag for conditional requests
    const etag = generateETag(content);
    res.setHeader('ETag', `"${etag}"`);
    res.setHeader('Last-Modified', new Date().toUTCString());
    
    // Conditional GET support
    if (req.headers['if-none-match'] === `"${etag}"`) {
      return res.status(304).end();
    }
    
    res.json({ posts });
  }
);

// User-specific data: private, no CDN cache
app.get('/api/profile',
  cacheStrategy({ maxAge: 0, private: true }),
  async (req, res) => {
    res.json({ user: {} });
  }
);

// Cloudflare Cache Rules (via API)
async function configureCloudflareCacheRules() {
  const CF_API = 'https://api.cloudflare.com/client/v4';
  const headers = {
    'Authorization': `Bearer ${process.env.CF_API_TOKEN}`,
    'Content-Type': 'application/json'
  };
  
  // Create cache rule for API endpoints
  const rule = {
    action: {
      type: 'set_cache_settings',
      value: {
        cache: true,
        edge_ttl: {
          mode: 'override_origin',
          default: 60
        },
        browser_ttl: {
          mode: 'respect_origin'
        }
      }
    },
    enabled: true,
    expression: '(http.host eq "api.chuaikan.com") and (http.request.uri.path matches "^/api/trending")',
    description: 'Cache trending API for 60s'
  };
  
  // POST to Cloudflare Cache Rules API
  console.log('Configure Cloudflare Cache Rules:', JSON.stringify(rule, null, 2));
}
```

### Step 649: TCP/HTTP Performance Testing

```bash
# test-network-performance.sh

echo "=== Network Performance Test ==="
HOST="${1:-api.chuaikan.com}"

# Test 1: HTTP/2 support
echo ""
echo "1. HTTP/2 Support:"
curl -svo /dev/null --http2 https://$HOST/health 2>&1 | grep "HTTP/"

# Test 2: Compression
echo ""
echo "2. Compression Support:"
echo -n "  gzip: "
curl -sI -H "Accept-Encoding: gzip" https://$HOST/api/users | grep -i "content-encoding" || echo "not supported"
echo -n "  brotli: "
curl -sI -H "Accept-Encoding: br" https://$HOST/api/users | grep -i "content-encoding" || echo "not supported"

# Test 3: Keep-alive
echo ""
echo "3. Keep-alive:"
curl -sI https://$HOST/health | grep -i "connection"

# Test 4: Response time
echo ""
echo "4. Response Timing:"
curl -w "DNS: %{time_namelookup}s | Connect: %{time_connect}s | TLS: %{time_appconnect}s | TTFB: %{time_starttransfer}s | Total: %{time_total}s\n" \
  -o /dev/null -s https://$HOST/health

# Test 5: Cache headers
echo ""
echo "5. Cache Headers:"
curl -sI https://$HOST/api/trending | grep -i "cache-control\|etag\|cf-cache"
```

### Step 650: CDN Cache Hit Rate Monitoring

```javascript
// cdn-cache-monitor.js - Monitor CDN cache performance
const client = require('prom-client');
const express = require('express');

// Cloudflare cache status metrics
const cacheHitCounter = new client.Counter({
  name: 'cdn_cache_hit_total',
  help: 'Total CDN cache hits',
  labelNames: ['status', 'path_group']
});

// Middleware to track cache performance
function cacheMetricsMiddleware(req, res, next) {
  res.on('finish', () => {
    // Cloudflare adds CF-Cache-Status header
    const cfCacheStatus = res.getHeader('CF-Cache-Status');
    
    if (cfCacheStatus) {
      // Group paths for metrics (avoid high cardinality)
      const pathGroup = req.path.match(/^\/api\/(\w+)/)?.[1] || 'other';
      cacheHitCounter.inc({ 
        status: cfCacheStatus.toLowerCase(),
        path_group: pathGroup
      });
    }
  });
  next();
}

// Cloudflare Analytics API
async function getCloudflareCacheStats(zoneId, hours = 24) {
  const query = `
    query {
      viewer {
        zones(filter: { zoneTag: "${zoneId}" }) {
          httpRequests1hGroups(
            limit: ${hours}
            filter: { datetime_gt: "${new Date(Date.now() - hours * 3600000).toISOString()}" }
            orderBy: [datetime_ASC]
          ) {
            sum {
              requests
              cachedRequests
              bytes
              cachedBytes
            }
            dimensions {
              datetime
            }
          }
        }
      }
    }
  `;
  
  const response = await fetch('https://api.cloudflare.com/client/v4/graphql', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.CF_API_TOKEN}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ query }),
  });
  
  const data = await response.json();
  const groups = data.data?.viewer?.zones[0]?.httpRequests1hGroups || [];
  
  return groups.map(g => ({
    time: g.dimensions.datetime,
    requests: g.sum.requests,
    cachedRequests: g.sum.cachedRequests,
    cacheHitRate: g.sum.requests > 0 
      ? (g.sum.cachedRequests / g.sum.requests * 100).toFixed(1) + '%'
      : '0%',
    bandwidthSaved: ((g.sum.cachedBytes / g.sum.bytes) * 100).toFixed(1) + '%',
  }));
}
```

---

## 🔧 Configuration Files

### nginx-compression-test.sh

```bash
#!/bin/bash
# ทดสอบ compression ของ Nginx

URL="${1:-https://chuaikan.com/api/users}"

echo "=== Compression Test for $URL ==="

echo ""
echo "No compression:"
SIZE_UNCOMPRESSED=$(curl -so /dev/null -w "%{size_download}" $URL)
echo "  Size: $SIZE_UNCOMPRESSED bytes"

echo ""
echo "gzip compression:"
SIZE_GZIP=$(curl -so /dev/null -w "%{size_download}" -H "Accept-Encoding: gzip" $URL)
echo "  Size: $SIZE_GZIP bytes"
echo "  Ratio: $(echo "scale=1; (1 - $SIZE_GZIP/$SIZE_UNCOMPRESSED) * 100" | bc)%"

echo ""
echo "brotli compression:"
SIZE_BROTLI=$(curl -so /dev/null -w "%{size_download}" -H "Accept-Encoding: br" $URL)
echo "  Size: $SIZE_BROTLI bytes"
echo "  Ratio: $(echo "scale=1; (1 - $SIZE_BROTLI/$SIZE_UNCOMPRESSED) * 100" | bc)%"
```

---

## 🧪 Testing

```bash
# Test HTTP/2 performance vs HTTP/1.1
echo "=== HTTP/1.1 vs HTTP/2 Performance ==="

# HTTP/1.1
autocannon --http1 -d 10 -c 100 https://api.chuaikan.com/api/users

# HTTP/2
autocannon --http2 -d 10 -c 100 https://api.chuaikan.com/api/users

# Test compression savings
curl -w "Size without compression: %{size_download} bytes\n" \
  -o /dev/null -s https://api.chuaikan.com/api/users

curl -w "Size with brotli: %{size_download} bytes\n" \
  -H "Accept-Encoding: br" \
  -o /dev/null -s https://api.chuaikan.com/api/users
```

---

## ❌ Common Errors & Solutions

### Error 1: HTTP/2 ไม่ทำงานใน Production

```bash
# ตรวจสอบว่า Nginx compiled กับ http_v2_module
nginx -V 2>&1 | grep http_v2

# ตรวจสอบ SSL (HTTP/2 ต้องการ TLS)
curl -k --http2 -v https://localhost/health 2>&1 | grep "ALPN\|HTTP/"

# ถ้าใช้ AWS ELB/ALB ต้องเปิด HTTP/2 ใน listener settings
```

### Error 2: Brotli ใช้ CPU สูงใน production

```nginx
# แก้ไข: ใช้ pre-compressed files แทน on-the-fly compression
# ให้ build process สร้าง .br files ไว้ล่วงหน้า
brotli_static on;  # Nginx จะ serve .br files ถ้ามี

# และ disable on-the-fly brotli สำหรับ dynamic responses
brotli off;
```

### Error 3: keepAliveTimeout ทำให้ connection drop

```javascript
// Express ต้องมี keepAliveTimeout > Nginx keepalive_timeout
// Nginx default keepalive_timeout = 75s
// ดังนั้น Node.js ต้องตั้งค่า >= 76s

server.keepAliveTimeout = 76000;  // 76 seconds
server.headersTimeout = 77000;    // ต้องเกิน keepAliveTimeout
```

---

## ✅ Checklist

- [ ] Enable HTTP/2 ใน Nginx
- [ ] Enable Brotli compression (หรือ gzip ถ้า Brotli ไม่พร้อม)
- [ ] ตั้งค่า `keep-alive` connections สำหรับทั้ง Nginx และ Node.js
- [ ] ปรับ TCP kernel parameters บน production server
- [ ] ตั้งค่า DNS prefetch/preconnect ใน HTML head
- [ ] ตั้งค่า proper Cache-Control headers สำหรับทุก endpoint
- [ ] ตรวจสอบ CDN cache hit rate > 80%
- [ ] Monitor network latency ด้วย Prometheus
- [ ] เปิดใช้ BBR congestion control
- [ ] ทดสอบ HTTP/2 multiplexing performance

---

## 🔗 References

- [HTTP/2 RFC 7540](https://tools.ietf.org/html/rfc7540)
- [Brotli Compression](https://github.com/google/brotli)
- [TCP BBR Congestion Control](https://cloud.google.com/blog/products/networking/tcp-bbr-congestion-control-comes-to-gcp-your-internet-just-got-faster)
- [Cloudflare Cache Rules](https://developers.cloudflare.com/cache/about/cache-rules/)
- [web.dev - HTTP caching](https://web.dev/http-cache/)

---

*Part 065 | Road to 1,000,000 Users/Day | chuaikan.com*
