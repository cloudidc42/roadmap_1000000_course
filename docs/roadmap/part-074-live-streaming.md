# Part 074: Live Streaming Architecture

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Advanced
> **Steps:** 731-740
> **เวลาโดยประมาณ:** 6 ชั่วโมง
> **Prerequisites:** Part 071 (WebSocket Scaling), Part 065 (Network Optimization)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- RTMP ingest และ FFmpeg transcoding
- HLS delivery สำหรับ live streaming
- mediamtx media server
- Cloudflare Stream สำหรับ managed live streaming
- Low-latency HLS (LL-HLS, 2-3s delay)
- Chat alongside live stream ด้วย WebSocket
- SOS live stream สำหรับ emergency reporting

---

## 📖 ทฤษฎีและแนวคิด

### Live Streaming Flow

```
Broadcaster (OBS/Mobile App)
         │
         │  RTMP (port 1935)
         ▼
┌─────────────────────────────────────────────────────┐
│          Media Ingest Server (mediamtx)             │
│                                                     │
│   RTMP → [mediamtx] → FFmpeg transcoding           │
│                              │                      │
│                    ┌─────────┴─────────┐            │
│                    │                   │            │
│             360p/30fps          720p/30fps           │
│             1080p/30fps         (Adaptive)           │
└────────────────────┬──────────────────┘             
                     │ HLS segments (.m3u8 + .ts)
                     ▼
              CDN (Cloudflare)
                     │ HTTP/2
          ┌──────────┼──────────┐
          ▼          ▼          ▼
       Viewer 1  Viewer 2  ...Viewer 100,000

Latency: Standard HLS = 10-30s, LL-HLS = 2-3s
```

### SOS Live Stream

```
Emergency Situation
         │
         │  User taps "SOS Live"
         ▼
Mobile App starts RTMP stream
         │
         │  + Location + Emergency type
         ▼
chuaikan.com API → marks stream as SOS
         │
         ▼
Broadcast to area subscribers (push notification)
"🚨 SOS Live nearby! Tap to watch"
         │
         ▼
Nearby users join live stream
         │ (1000s of viewers instantly)
         ▼
Emergency services also watch the stream
```

---

## ⚙️ Environment Setup

### Step 731: ติดตั้ง mediamtx

```bash
# Download mediamtx (formerly rtsp-simple-server)
MEDIAMTX_VERSION="v1.8.0"
wget https://github.com/bluenviron/mediamtx/releases/download/${MEDIAMTX_VERSION}/mediamtx_${MEDIAMTX_VERSION}_linux_amd64.tar.gz
tar -xzf mediamtx_${MEDIAMTX_VERSION}_linux_amd64.tar.gz
sudo mv mediamtx /usr/local/bin/
sudo mv mediamtx.yml /etc/mediamtx/

# ติดตั้ง FFmpeg
sudo apt-get install -y ffmpeg

# ตรวจสอบ
ffmpeg -version
mediamtx --version
```

---

## 🛠️ Step-by-Step Implementation

### Step 732: mediamtx Configuration

```yaml
# /etc/mediamtx/mediamtx.yml - Media server configuration

# Logging
logLevel: info
logDestinations: [stdout]

# API สำหรับ control (ดู streams, kick users)
api: yes
apiAddress: 127.0.0.1:9997

# Metrics สำหรับ Prometheus
metrics: yes
metricsAddress: 127.0.0.1:9998

# RTMP ingest (Broadcaster → Server)
rtmp: yes
rtmpAddress: :1935
rtmpEncryption: no  # ใช้ yes + certificates ใน production

# HLS delivery (Server → Viewers)
hls: yes
hlsAddress: :8888
hlsAlwaysRemux: no
hlsVariant: lowLatency    # LL-HLS!
hlsSegmentCount: 7
hlsSegmentDuration: 1s    # 1 second segments สำหรับ LL-HLS
hlsPartDuration: 200ms    # Partial segments ทุก 200ms (LL-HLS key setting)
hlsSegmentMaxSize: 50MB
hlsAllowOrigin: '*'
hlsEncryption: no

# Path configuration
paths:
  # Default path สำหรับ live streams
  live/{streamKey}:
    # Authenticate ด้วย stream key
    publishUser: ""
    publishPass: ""
    
    # Run hook เมื่อ stream start/end
    runOnReady: >
      curl -s -X POST http://localhost:3000/api/internal/stream/start
      -H "Content-Type: application/json"
      -d '{"streamKey":"$MTX_PATH","address":"$MTX_QUERY"}'
    runOnReadyRestart: yes
    
    runOnNotReady: >
      curl -s -X POST http://localhost:3000/api/internal/stream/end
      -H "Content-Type: application/json"
      -d '{"streamKey":"$MTX_PATH"}'
    
    # FFmpeg transcoding: สร้าง multiple quality tracks
    runOnReady: >
      ffmpeg -i rtmp://localhost/$MTX_PATH
      -c:v libx264 -preset veryfast -tune zerolatency
      -map 0:v -s 1920x1080 -b:v 4000k -maxrate 4000k -bufsize 8000k
      -map 0:v -s 1280x720  -b:v 2000k -maxrate 2000k -bufsize 4000k
      -map 0:v -s 640x360   -b:v 800k  -maxrate 800k  -bufsize 1600k
      -c:a aac -b:a 128k -ar 44100
      -f hls
      -hls_time 1
      -hls_list_size 10
      -hls_flags delete_segments+independent_segments+program_date_time
      -hls_segment_type mpegts
      -master_pl_name master.m3u8
      -var_stream_map "v:0,a:0 v:1,a:1 v:2,a:2"
      /var/www/hls/$MTX_PATH_%v/index.m3u8
    
    # Max concurrent publishers (1 stream key = 1 publisher)
    maxReaders: 10000
```

```bash
# สร้าง directory สำหรับ HLS segments
sudo mkdir -p /var/www/hls
sudo chown -R www-data:www-data /var/www/hls

# สร้าง systemd service สำหรับ mediamtx
sudo tee /etc/systemd/system/mediamtx.service << 'EOF'
[Unit]
Description=mediamtx media server
After=network.target

[Service]
Type=simple
User=www-data
ExecStart=/usr/local/bin/mediamtx /etc/mediamtx/mediamtx.yml
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable mediamtx
sudo systemctl start mediamtx
```

### Step 733: Node.js Live Stream Management API

```javascript
// routes/streams.js - Live stream management
const express = require('express');
const router = express.Router();
const crypto = require('crypto');
const db = require('../config/db');
const redis = require('../config/redis');

/**
 * POST /api/streams/start
 * User ขอ stream key เพื่อเริ่ม live stream
 */
router.post('/start', async (req, res) => {
  const userId = req.user.id;
  const { title, type = 'general', areaCode } = req.body;
  
  // สร้าง unique stream key
  const streamKey = crypto.randomBytes(16).toString('hex');
  
  // บันทึกใน database
  const { rows: [stream] } = await db.query(`
    INSERT INTO live_streams (user_id, stream_key, title, type, area_code, status)
    VALUES ($1, $2, $3, $4, $5, 'waiting')
    RETURNING id, stream_key, rtmp_url, hls_url
  `, [userId, streamKey, title, type, areaCode]);
  
  // Cache stream info ใน Redis
  await redis.setex(`stream:${streamKey}`, 3600, JSON.stringify({
    id: stream.id,
    userId,
    type,
    areaCode,
  }));
  
  res.json({
    streamId: stream.id,
    streamKey: stream.stream_key,
    rtmpUrl: `rtmp://stream.chuaikan.com/live/${stream.stream_key}`,
    hlsUrl: `https://stream.chuaikan.com/hls/live/${stream.stream_key}/master.m3u8`,
  });
});

/**
 * POST /api/internal/stream/start - Called by mediamtx hook
 * เมื่อ broadcaster เชื่อมต่อมา
 */
router.post('/internal/stream/start', async (req, res) => {
  const { streamKey } = req.body;
  
  const streamInfo = JSON.parse(await redis.get(`stream:${streamKey}`));
  if (!streamInfo) {
    return res.status(404).json({ error: 'Invalid stream key' });
  }
  
  // Update stream status
  await db.query(
    'UPDATE live_streams SET status = $1, started_at = NOW() WHERE stream_key = $2',
    ['live', streamKey]
  );
  
  // SOS streams: notify area subscribers immediately
  if (streamInfo.type === 'sos') {
    await notifySOSStreamToArea(streamInfo);
  }
  
  res.json({ ok: true });
});

async function notifySOSStreamToArea(streamInfo) {
  const { id: streamId, userId, areaCode } = streamInfo;
  
  // ดึง subscribers ใน area
  const { rows: subscribers } = await db.query(`
    SELECT u.fcm_token
    FROM area_subscriptions asub
    JOIN users u ON u.id = asub.user_id
    WHERE asub.area_code = $1
      AND asub.is_active = TRUE
      AND u.fcm_token IS NOT NULL
      AND u.id != $2
    LIMIT 10000
  `, [areaCode, userId]);
  
  // Send FCM notifications
  const tokens = subscribers.map(s => s.fcm_token);
  await sendFCMBatch(tokens, {
    title: '🚨 SOS Live Stream ใกล้คุณ',
    body: 'มีเหตุฉุกเฉิน - แตะเพื่อดู live',
    data: {
      type: 'SOS_LIVE',
      streamId: streamId.toString(),
    },
    priority: 'high',
  });
  
  console.log(`SOS live stream ${streamId} notified to ${tokens.length} area subscribers`);
}

module.exports = router;
```

### Step 734: Low-Latency HLS (LL-HLS) Setup

```nginx
# /etc/nginx/sites-available/streaming.chuaikan.com
server {
    listen 443 ssl http2;
    server_name stream.chuaikan.com;
    
    ssl_certificate /etc/letsencrypt/live/stream.chuaikan.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/stream.chuaikan.com/privkey.pem;
    
    # HLS segments directory
    location /hls/ {
        alias /var/www/hls/;
        
        # LL-HLS requires these headers
        add_header 'Cache-Control' 'no-cache';
        add_header 'Access-Control-Allow-Origin' '*';
        add_header 'Access-Control-Allow-Methods' 'GET, OPTIONS';
        add_header 'Access-Control-Expose-Headers' 'Content-Length';
        
        # MIME types
        types {
            application/vnd.apple.mpegurl m3u8;
            video/mp2t ts;
            application/x-mpegURL m3u8;
        }
        
        # Cache playlist ไม่นาน (เปลี่ยนบ่อย)
        location ~* \.m3u8$ {
            add_header 'Cache-Control' 'no-cache, no-store, must-revalidate';
            expires -1;
        }
        
        # Cache .ts segments นานกว่า (ไม่เปลี่ยน)
        location ~* \.ts$ {
            add_header 'Cache-Control' 'public, max-age=600';
        }
    }
    
    # Proxy ไปยัง mediamtx HLS
    location /live/ {
        proxy_pass http://127.0.0.1:8888/;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_buffering off;
        
        # LL-HLS requires chunked transfer for partial segments
        proxy_set_header Transfer-Encoding chunked;
    }
}
```

### Step 735: HLS.js Player Integration (Frontend)

```javascript
// frontend/components/LivePlayer.jsx
import React, { useEffect, useRef, useState } from 'react';
import Hls from 'hls.js';

export function LivePlayer({ streamId, isSOSStream = false }) {
  const videoRef = useRef(null);
  const hlsRef = useRef(null);
  const [viewerCount, setViewerCount] = useState(0);
  const [latency, setLatency] = useState(null);
  const [quality, setQuality] = useState('auto');
  
  useEffect(() => {
    const video = videoRef.current;
    if (!video) return;
    
    const hlsUrl = `https://stream.chuaikan.com/hls/live/${streamId}/master.m3u8`;
    
    if (Hls.isSupported()) {
      const hls = new Hls({
        // LL-HLS settings
        lowLatencyMode: true,
        backBufferLength: 30,
        maxLiveSyncPlaybackRate: 1.5,    // Speed up เพื่อ catch up
        liveSyncDurationCount: 3,
        liveMaxLatencyDurationCount: 5,
        
        // Quality settings
        capLevelToPlayerSize: true,
        startLevel: -1,  // Auto quality selection
        
        // Retry settings
        manifestLoadingMaxRetry: 5,
        levelLoadingMaxRetry: 5,
      });
      
      hls.loadSource(hlsUrl);
      hls.attachMedia(video);
      
      hls.on(Hls.Events.MANIFEST_PARSED, () => {
        video.play().catch(console.error);
      });
      
      // Monitor latency
      hls.on(Hls.Events.FRAG_CHANGED, (event, data) => {
        const latencyMs = hls.latency * 1000;
        setLatency(Math.round(latencyMs));
      });
      
      hlsRef.current = hls;
      
    } else if (video.canPlayType('application/vnd.apple.mpegurl')) {
      // Safari native HLS support
      video.src = hlsUrl;
      video.play().catch(console.error);
    }
    
    return () => {
      if (hlsRef.current) {
        hlsRef.current.destroy();
      }
    };
  }, [streamId]);
  
  return (
    <div className={`relative ${isSOSStream ? 'ring-4 ring-red-500' : ''}`}>
      {isSOSStream && (
        <div className="absolute top-2 left-2 bg-red-600 text-white px-2 py-1 rounded text-sm font-bold z-10">
          🚨 SOS LIVE
        </div>
      )}
      
      <video
        ref={videoRef}
        controls
        autoPlay
        muted
        playsInline
        className="w-full aspect-video bg-black"
      />
      
      <div className="flex items-center gap-4 p-2 bg-gray-900 text-white text-sm">
        <span>👁️ {viewerCount.toLocaleString()} คน</span>
        {latency && <span>📡 {latency}ms</span>}
      </div>
    </div>
  );
}
```

### Step 736: Live Chat ด้วย Socket.io

```javascript
// websocket/live-chat.js - Live stream chat
const { Server } = require('socket.io');
const { createAdapter } = require('@socket.io/redis-adapter');

function setupLiveChat(server, pubClient, subClient) {
  const io = new Server(server, {
    cors: { origin: process.env.FRONTEND_URL },
  });
  
  io.adapter(createAdapter(pubClient, subClient));
  
  io.of('/live-chat').on('connection', (socket) => {
    const { streamId } = socket.handshake.query;
    const userId = socket.user?.id;
    
    if (!streamId) {
      socket.disconnect();
      return;
    }
    
    // Join stream room
    socket.join(`stream:${streamId}`);
    
    // Update viewer count
    updateViewerCount(streamId, +1);
    
    // ส่ง viewer count update ไปยังทุก viewer
    io.of('/live-chat').to(`stream:${streamId}`).emit('viewer_count', {
      count: await getViewerCount(streamId),
    });
    
    socket.on('message', async (data) => {
      if (!userId) return;  // Anonymous users cannot chat
      
      const message = {
        id: crypto.randomUUID(),
        userId,
        username: socket.user.username,
        text: data.text.slice(0, 200),
        timestamp: new Date().toISOString(),
        type: data.type || 'chat',
      };
      
      // Rate limit: 5 messages per 10 seconds per user
      const rateLimitKey = `chat-rate:${userId}:${streamId}`;
      const count = await redis.incr(rateLimitKey);
      await redis.expire(rateLimitKey, 10);
      
      if (count > 5) {
        socket.emit('error', { message: 'Rate limit exceeded' });
        return;
      }
      
      // Broadcast ไปยังทุกคนใน stream room
      io.of('/live-chat').to(`stream:${streamId}`).emit('message', message);
      
      // บันทึก chat log (async, ไม่ block)
      saveChatMessage(streamId, message).catch(console.error);
    });
    
    socket.on('disconnect', () => {
      updateViewerCount(streamId, -1);
    });
  });
}

async function updateViewerCount(streamId, delta) {
  const key = `viewers:${streamId}`;
  await redis.incrby(key, delta);
  await redis.expire(key, 3600);
}

async function getViewerCount(streamId) {
  return parseInt(await redis.get(`viewers:${streamId}`) || '0');
}
```

### Step 737: Cloudflare Stream Integration

```javascript
// services/cloudflare-stream.js - Managed live streaming

const CLOUDFLARE_ACCOUNT_ID = process.env.CLOUDFLARE_ACCOUNT_ID;
const CLOUDFLARE_API_TOKEN = process.env.CLOUDFLARE_API_TOKEN;

const cfHeaders = {
  'Authorization': `Bearer ${CLOUDFLARE_API_TOKEN}`,
  'Content-Type': 'application/json',
};

/**
 * สร้าง live stream ด้วย Cloudflare Stream
 * ใช้สำหรับ high-profile streams (SOS alerts, events)
 */
async function createCloudflareStream(options) {
  const { title, userId, type } = options;
  
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${CLOUDFLARE_ACCOUNT_ID}/stream/live_inputs`,
    {
      method: 'POST',
      headers: cfHeaders,
      body: JSON.stringify({
        meta: { name: title },
        recording: {
          mode: 'automatic',         // Auto-record ทุก stream
          timeoutSeconds: 3600,      // Record นาน 1 ชั่วโมง
        },
        defaultCreator: userId,
      }),
    }
  );
  
  const data = await response.json();
  
  if (!data.success) {
    throw new Error(`Cloudflare Stream error: ${JSON.stringify(data.errors)}`);
  }
  
  return {
    liveInputId: data.result.uid,
    rtmpsUrl: data.result.rtmps.url,    // RTMPS ingest URL
    rtmpsKey: data.result.rtmps.streamKey,
    hlsUrl: data.result.playback.hls,   // HLS playback URL
    dashUrl: data.result.playback.dash,
  };
}

/**
 * ดูสถานะ live stream
 */
async function getLiveStreamStatus(liveInputId) {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${CLOUDFLARE_ACCOUNT_ID}/stream/live_inputs/${liveInputId}`,
    { headers: cfHeaders }
  );
  
  const data = await response.json();
  return data.result?.status || 'unknown';
}

/**
 * ปิด live stream
 */
async function endCloudflareStream(liveInputId) {
  await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${CLOUDFLARE_ACCOUNT_ID}/stream/live_inputs/${liveInputId}`,
    {
      method: 'DELETE',
      headers: cfHeaders,
    }
  );
}

module.exports = { createCloudflareStream, getLiveStreamStatus, endCloudflareStream };
```

### Step 738: SOS Live Stream Feature

```javascript
// routes/sos-stream.js - SOS Emergency Live Streaming

router.post('/sos-live/start', async (req, res) => {
  const userId = req.user.id;
  const { latitude, longitude, emergencyType, description } = req.body;
  
  try {
    // สร้าง SOS alert ก่อน
    const { rows: [alert] } = await db.query(`
      INSERT INTO sos_alerts (
        user_id, latitude, longitude, emergency_type, 
        description, status, has_live_stream
      )
      VALUES ($1, $2, $3, $4, $5, 'active', TRUE)
      RETURNING id, area_code
    `, [userId, latitude, longitude, emergencyType, description]);
    
    // สร้าง Cloudflare Stream (ใช้ managed service สำหรับ SOS)
    const stream = await createCloudflareStream({
      title: `SOS: ${emergencyType} - ${new Date().toISOString()}`,
      userId: userId.toString(),
      type: 'sos',
    });
    
    // บันทึก stream info ใน database
    await db.query(`
      UPDATE sos_alerts
      SET stream_key = $1,
          live_stream_url = $2,
          rtmps_url = $3,
          rtmps_key = $4
      WHERE id = $5
    `, [stream.liveInputId, stream.hlsUrl, stream.rtmpsUrl, stream.rtmpsKey, alert.id]);
    
    // Notify area subscribers
    await publishSOSAlert({
      id: alert.id,
      type: 'sos_live',
      userId,
      location: { lat: latitude, lng: longitude },
      severity: 'critical',
      streamUrl: stream.hlsUrl,
      areaCode: alert.area_code,
    });
    
    res.json({
      alertId: alert.id,
      rtmpsUrl: stream.rtmpsUrl,       // Broadcaster ใช้ URL นี้
      rtmpsKey: stream.rtmpsKey,
      watchUrl: `https://chuaikan.com/sos-live/${alert.id}`,
    });
    
  } catch (err) {
    console.error('SOS live stream error:', err);
    res.status(500).json({ error: 'Failed to start SOS live stream' });
  }
});

/**
 * GET /api/sos-live/:alertId - Get live stream info for viewers
 */
router.get('/sos-live/:alertId', async (req, res) => {
  const { alertId } = req.params;
  
  const { rows: [alert] } = await db.query(`
    SELECT sa.*, u.username, u.avatar_url
    FROM sos_alerts sa
    JOIN users u ON u.id = sa.user_id
    WHERE sa.id = $1 AND sa.has_live_stream = TRUE
  `, [alertId]);
  
  if (!alert) {
    return res.status(404).json({ error: 'SOS stream not found' });
  }
  
  res.json({
    alertId: alert.id,
    streamerName: alert.username,
    streamerAvatar: alert.avatar_url,
    emergencyType: alert.emergency_type,
    description: alert.description,
    status: alert.status,
    hlsUrl: alert.live_stream_url,
    viewerCount: await getViewerCount(alertId),
    startedAt: alert.started_at,
  });
});
```

### Step 739: Stream Recording และ VOD

```javascript
// services/vod.js - Video on Demand after live stream

/**
 * เมื่อ live stream จบ: สร้าง VOD recording
 */
async function processStreamRecording(streamId, liveInputId) {
  // Wait for Cloudflare ประมวลผล recording
  let retries = 0;
  while (retries < 10) {
    const { rows: [videos] } = await db.query(`
      SELECT cloudflare_video_id
      FROM live_streams
      WHERE id = $1 AND cloudflare_video_id IS NOT NULL
    `, [streamId]);
    
    if (videos) break;
    
    await new Promise(resolve => setTimeout(resolve, 30000));
    retries++;
  }
  
  // ดึง recording จาก Cloudflare
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${CLOUDFLARE_ACCOUNT_ID}/stream/live_inputs/${liveInputId}/videos`,
    { headers: cfHeaders }
  );
  
  const data = await response.json();
  const recording = data.result?.[0];
  
  if (!recording) return;
  
  // บันทึก VOD info ใน database
  await db.query(`
    UPDATE live_streams
    SET vod_url = $1,
        vod_thumbnail = $2,
        duration = $3,
        status = 'ended'
    WHERE id = $4
  `, [
    recording.playback.hls,
    recording.thumbnail,
    recording.duration,
    streamId,
  ]);
  
  console.log(`VOD created for stream ${streamId}: ${recording.playback.hls}`);
}
```

### Step 740: Load Testing สำหรับ Streaming

```javascript
// tests/stream-load-test.js - k6 สำหรับ HLS streaming load test
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '1m', target: 100 },    // 100 concurrent viewers
    { duration: '5m', target: 1000 },   // 1000 concurrent viewers
    { duration: '5m', target: 5000 },   // 5000 concurrent viewers
    { duration: '2m', target: 0 },
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],   // HLS segments load < 500ms
    http_req_failed: ['rate<0.01'],     // Error rate < 1%
  },
};

const STREAM_URL = 'https://stream.chuaikan.com/hls/live/test-stream';

export default function() {
  // Simulate HLS player fetching playlist
  const masterPlaylist = http.get(`${STREAM_URL}/master.m3u8`);
  check(masterPlaylist, {
    'master playlist 200': (r) => r.status === 200,
    'has HLS content': (r) => r.body.includes('#EXTM3U'),
  });
  
  if (masterPlaylist.status !== 200) return;
  
  // Parse and fetch 720p playlist
  const videoPlaylist = http.get(`${STREAM_URL}/720p/index.m3u8`);
  check(videoPlaylist, {
    'video playlist 200': (r) => r.status === 200,
  });
  
  // Simulate segment fetch (every ~2 seconds)
  sleep(2);
  
  // Fetch latest segment
  const segment = http.get(`${STREAM_URL}/720p/seg001.ts`);
  check(segment, {
    'segment 200': (r) => r.status === 200,
    'segment not empty': (r) => r.body.length > 0,
  });
  
  sleep(1);
}
```

---

## 🔧 Configuration Files

```bash
# /etc/sysctl.d/99-streaming.conf - Kernel settings สำหรับ streaming
# Handle thousands of concurrent connections

net.core.somaxconn = 65535
net.core.netdev_max_backlog = 32768
net.ipv4.tcp_max_syn_backlog = 16384

# Network buffers สำหรับ video streaming
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# Apply
sysctl -p /etc/sysctl.d/99-streaming.conf
```

---

## 🧪 Testing

```bash
# Test RTMP ingest ด้วย FFmpeg
ffmpeg -re -i test-video.mp4 \
  -c:v libx264 -preset veryfast -tune zerolatency \
  -c:a aac -b:a 128k \
  -f flv rtmp://localhost/live/test-stream-key

# Monitor HLS output
watch -n 1 'ls -la /var/www/hls/live/test-stream-key/ | tail -20'

# Test HLS playback
ffplay https://stream.chuaikan.com/hls/live/test-stream-key/master.m3u8

# Test viewer count
curl http://localhost:9997/v3/paths/list | jq '.items[].readers'
```

---

## ❌ Common Errors & Solutions

### Error 1: High latency (>10 seconds)

```yaml
# ปัญหา: Standard HLS latency สูง
# แก้ไข: ใช้ LL-HLS settings
hlsVariant: lowLatency
hlsSegmentDuration: 1s
hlsPartDuration: 200ms
```

### Error 2: 404 on HLS segments

```bash
# ปัญหา: Nginx ไม่สามารถ serve .ts files
# แก้ไข: เพิ่ม MIME types
sudo tee -a /etc/nginx/mime.types << 'EOF'
video/mp2t ts;
application/vnd.apple.mpegurl m3u8;
EOF
sudo nginx -t && sudo nginx -s reload
```

---

## ✅ Checklist

- [ ] ติดตั้ง mediamtx สำหรับ RTMP ingest
- [ ] ตั้งค่า FFmpeg transcoding (1080p/720p/360p)
- [ ] Enable LL-HLS สำหรับ low latency (2-3s)
- [ ] Nginx proxy สำหรับ HLS delivery
- [ ] Cloudflare Stream integration สำหรับ SOS streams
- [ ] Live chat ด้วย Socket.io + Redis adapter
- [ ] Stream recording → VOD pipeline
- [ ] SOS live stream feature (instant notify area subscribers)
- [ ] Load test 5000 concurrent viewers
- [ ] Monitor CDN bandwidth usage

---

## 🔗 References

- [mediamtx GitHub](https://github.com/bluenviron/mediamtx)
- [LL-HLS Specification](https://developer.apple.com/documentation/http-live-streaming/enabling-low-latency-hls)
- [HLS.js](https://github.com/video-dev/hls.js)
- [Cloudflare Stream](https://developers.cloudflare.com/stream/)
- [FFmpeg Documentation](https://ffmpeg.org/documentation.html)

---

*Part 074 | Road to 1,000,000 Users/Day | chuaikan.com*
