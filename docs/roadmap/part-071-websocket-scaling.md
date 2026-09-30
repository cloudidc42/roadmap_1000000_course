# Part 071: WebSocket Scaling ด้วย Redis Pub/Sub

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Advanced
> **Steps:** 701-710
> **เวลาโดยประมาณ:** 5 ชั่วโมง
> **Prerequisites:** Part 065 (Network Optimization), Part 070 (SLO)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ปัญหา sticky sessions ของ WebSocket
- Redis pub/sub สำหรับ Socket.io across multiple servers
- ตั้งค่า Socket.io Redis adapter
- Horizontal scaling: 10 Socket.io servers sharing state
- Room management at scale
- Connection count monitoring
- WebSocket load testing
- Graceful shutdown with connection draining

---

## 📖 ทฤษฎีและแนวคิด

### WebSocket Sticky Sessions Problem

```
Without Sticky Sessions / Redis Adapter:

User A connects → Server 1 (knows about room "sos-room-1")
User B connects → Server 2 (DIFFERENT room state!)
User A sends message to "sos-room-1"
Server 1 broadcasts to room locally
User B DOESN'T RECEIVE the message! ❌

With Redis Pub/Sub:

User A → Server 1 ─→ Redis ←─ Server 2 ← User B
         publishes          subscribes
         to channel         from channel
         
Server 1 sends to Redis → Redis broadcasts to all servers
→ Server 2 delivers to User B ✅
```

### Socket.io Redis Adapter Architecture

```
┌────────────────────────────────────────────────┐
│                    Client                       │
│            (Browser/Mobile App)                 │
└────────────┬──────────────────────┬────────────┘
             │                      │
             │ WebSocket            │ WebSocket
             ↓                      ↓
┌────────────────────┐  ┌────────────────────┐
│  Socket.io Server 1│  │  Socket.io Server 2│
│   (Pod 1)          │  │   (Pod 2)          │
└────────────┬───────┘  └────────────┬───────┘
             │                        │
             └────────────┬───────────┘
                          ↓
             ┌────────────────────────┐
             │    Redis Cluster       │
             │  (Pub/Sub backbone)    │
             │                        │
             │  Channels:             │
             │  - socket.io#/#room1#  │
             │  - socket.io#/#room2#  │
             │  - socket.io#/#sos-1#  │
             └────────────────────────┘
```

---

## ⚙️ Environment Setup

```bash
# สร้าง project
mkdir -p /home/user/websocket-scaling-demo
cd /home/user/websocket-scaling-demo
npm init -y

# ติดตั้ง dependencies
npm install \
  socket.io \
  @socket.io/redis-adapter \
  @socket.io/redis-streams-adapter \
  ioredis \
  express \
  prom-client

# Redis บน Docker (development)
docker run -d \
  --name redis-ws \
  -p 6379:6379 \
  redis:7-alpine \
  redis-server --appendonly yes

# ตรวจสอบ
redis-cli ping  # ควรได้ PONG
```

---

## 🛠️ Step-by-Step Implementation

### Step 701: Basic Socket.io Setup

```javascript
// server.js - Basic Socket.io setup (single server)
const express = require('express');
const { createServer } = require('http');
const { Server } = require('socket.io');

const app = express();
const httpServer = createServer(app);

const io = new Server(httpServer, {
  cors: {
    origin: process.env.FRONTEND_URL || '*',
    methods: ['GET', 'POST'],
  },
  // Connection settings
  pingTimeout: 60000,
  pingInterval: 25000,
  upgradeTimeout: 30000,
  transports: ['websocket', 'polling'],  // prefer websocket
  
  // Compression
  perMessageDeflate: {
    threshold: 1024,  // compress messages > 1KB
  },
  
  // Max payload
  maxHttpBufferSize: 1e6,  // 1MB
});

// Connection handler
io.on('connection', (socket) => {
  const userId = socket.handshake.auth.userId;
  const deviceType = socket.handshake.auth.deviceType || 'web';
  
  console.log(`User ${userId} connected (${socket.id}) from ${deviceType}`);
  
  // Join user's personal room
  socket.join(`user:${userId}`);
  
  // Handle SOS alerts
  socket.on('sos:create', async (data) => {
    const alert = await createSOSAlert({ ...data, userId });
    
    // Broadcast to nearby users (in same area room)
    const areaRoom = `area:${data.areaCode}`;
    io.to(areaRoom).emit('sos:new', alert);
    
    // Notify emergency services
    io.to('room:emergency-services').emit('sos:alert', alert);
  });
  
  socket.on('disconnect', (reason) => {
    console.log(`User ${userId} disconnected: ${reason}`);
  });
});

httpServer.listen(3000, () => {
  console.log('WebSocket server on port 3000');
});
```

### Step 702: Redis Adapter สำหรับ Multi-Server

```javascript
// server-redis.js - Socket.io + Redis adapter
const express = require('express');
const { createServer } = require('http');
const { Server } = require('socket.io');
const { createAdapter } = require('@socket.io/redis-adapter');
const { createClient } = require('ioredis');

const app = express();
const httpServer = createServer(app);

// สร้าง Redis clients (ต้องใช้ 2 clients: pub + sub)
const pubClient = createClient({
  host: process.env.REDIS_HOST || 'localhost',
  port: 6379,
  password: process.env.REDIS_PASSWORD,
  // Connection pool
  maxRetriesPerRequest: 3,
  lazyConnect: false,
  // Reconnect strategy
  retryStrategy: (times) => {
    const delay = Math.min(times * 50, 2000);
    return delay;
  },
});

const subClient = pubClient.duplicate();

// Handle Redis connection errors
pubClient.on('error', (err) => {
  console.error('Redis pub client error:', err);
});
subClient.on('error', (err) => {
  console.error('Redis sub client error:', err);
});

const io = new Server(httpServer, {
  cors: { origin: process.env.FRONTEND_URL || '*' },
  transports: ['websocket'],  // ใช้ websocket เท่านั้น (disable polling)
});

// ใช้ Redis adapter
io.adapter(createAdapter(pubClient, subClient));

// Wait for Redis connection ก่อน start server
async function startServer() {
  await Promise.all([pubClient.connect(), subClient.connect()]);
  console.log('Redis connected');
  
  io.on('connection', (socket) => {
    const userId = socket.handshake.auth.userId;
    socket.join(`user:${userId}`);
    
    // ===== Chat Room =====
    socket.on('room:join', (roomId) => {
      socket.join(`room:${roomId}`);
      socket.to(`room:${roomId}`).emit('user:joined', { userId, roomId });
    });
    
    socket.on('room:leave', (roomId) => {
      socket.leave(`room:${roomId}`);
      socket.to(`room:${roomId}`).emit('user:left', { userId, roomId });
    });
    
    socket.on('message:send', async ({ roomId, content }) => {
      const message = {
        id: crypto.randomUUID(),
        userId,
        roomId,
        content,
        timestamp: new Date().toISOString(),
      };
      
      // ✅ Redis adapter broadcasts ไปยัง server ทั้งหมด
      io.to(`room:${roomId}`).emit('message:new', message);
    });
    
    // ===== SOS Alerts =====
    socket.on('sos:create', async (data) => {
      const alert = {
        id: crypto.randomUUID(),
        ...data,
        userId,
        status: 'pending',
        createdAt: new Date().toISOString(),
      };
      
      // Save to database
      // await db.sosAlert.create({ data: alert });
      
      // Broadcast to all servers via Redis
      io.emit('sos:new', alert);  // broadcast to everyone
    });
  });
  
  httpServer.listen(process.env.PORT || 3000, () => {
    console.log(`WebSocket server ${process.env.PORT || 3000} connected to Redis`);
  });
}

startServer().catch(console.error);
```

### Step 703: Horizontal Scaling Configuration

```yaml
# kubernetes/websocket-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chuaikan-websocket
  namespace: production
spec:
  replicas: 10  # 10 Socket.io instances
  selector:
    matchLabels:
      app: chuaikan-websocket
  template:
    metadata:
      labels:
        app: chuaikan-websocket
    spec:
      containers:
      - name: websocket
        image: chuaikan/websocket:latest
        ports:
        - containerPort: 3000
        env:
        - name: REDIS_HOST
          valueFrom:
            secretKeyRef:
              name: redis-credentials
              key: host
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-credentials
              key: password
        - name: PORT
          value: "3000"
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "2000m"
            memory: "1Gi"
        # Graceful shutdown
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 30"]
        terminationGracePeriodSeconds: 60
---
# Service (ไม่ต้องใช้ sticky sessions เพราะมี Redis adapter)
apiVersion: v1
kind: Service
metadata:
  name: chuaikan-websocket-svc
  namespace: production
spec:
  selector:
    app: chuaikan-websocket
  ports:
  - protocol: TCP
    port: 3000
    targetPort: 3000
  type: ClusterIP
---
# Nginx Ingress for WebSocket
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: websocket-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-body-size: "8m"
    nginx.ingress.kubernetes.io/upstream-hash-by: "$http_x_forwarded_for"  # optional sticky
spec:
  rules:
  - host: ws.chuaikan.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: chuaikan-websocket-svc
            port:
              number: 3000
```

### Step 704: Room Management at Scale

```javascript
// room-manager.js - Efficient room management
const { createClient } = require('ioredis');

class RoomManager {
  constructor(redis) {
    this.redis = redis;
    this.ROOM_EXPIRY = 3600;  // rooms expire after 1 hour of inactivity
  }
  
  // Track room membership ใน Redis (persistent across server restarts)
  async joinRoom(roomId, userId, socketId) {
    const pipe = this.redis.pipeline();
    
    // Set membership
    pipe.sadd(`room:${roomId}:members`, userId);
    
    // Track user's socket
    pipe.hset(`user:${userId}:sockets`, socketId, Date.now());
    
    // Track which room socket is in
    pipe.hset(`socket:${socketId}`, 'roomId', roomId, 'userId', userId);
    
    // Update room activity
    pipe.expire(`room:${roomId}:members`, this.ROOM_EXPIRY);
    
    // Increment room count
    pipe.incr(`room:${roomId}:count`);
    
    await pipe.exec();
    
    return this.getRoomInfo(roomId);
  }
  
  async leaveRoom(roomId, userId, socketId) {
    const pipe = this.redis.pipeline();
    
    pipe.srem(`room:${roomId}:members`, userId);
    pipe.hdel(`user:${userId}:sockets`, socketId);
    pipe.del(`socket:${socketId}`);
    pipe.decr(`room:${roomId}:count`);
    
    await pipe.exec();
    
    // ลบ room ถ้าว่างเปล่า
    const count = await this.redis.get(`room:${roomId}:count`);
    if (parseInt(count) <= 0) {
      await this.redis.del(`room:${roomId}:members`, `room:${roomId}:count`);
    }
  }
  
  async getRoomInfo(roomId) {
    const [members, count] = await Promise.all([
      this.redis.smembers(`room:${roomId}:members`),
      this.redis.get(`room:${roomId}:count`),
    ]);
    
    return {
      roomId,
      memberCount: parseInt(count) || members.length,
      members,
    };
  }
  
  async getRoomCount() {
    // SCAN สำหรับ count rooms (ไม่ใช้ KEYS ใน production!)
    let cursor = '0';
    let count = 0;
    
    do {
      const [newCursor, keys] = await this.redis.scan(
        cursor,
        'MATCH', 'room:*:count',
        'COUNT', 100
      );
      cursor = newCursor;
      count += keys.length;
    } while (cursor !== '0');
    
    return count;
  }
  
  // Rate limit: ป้องกัน join room spam
  async checkJoinRateLimit(userId) {
    const key = `ratelimit:join:${userId}`;
    const count = await this.redis.incr(key);
    
    if (count === 1) {
      await this.redis.expire(key, 60);  // reset ทุก 60 วินาที
    }
    
    return count <= 10;  // max 10 room joins per minute
  }
  
  // Broadcast to SOS area (geographic rooms)
  async broadcastToArea(areaCode, event, data) {
    const areaRoom = `area:${areaCode}`;
    const members = await this.getRoomInfo(areaRoom);
    
    // Use Socket.io's Redis adapter for broadcast
    // This is handled automatically by io.to(areaRoom).emit()
    return members;
  }
}

module.exports = RoomManager;
```

### Step 705: Connection Monitoring

```javascript
// ws-metrics.js - WebSocket metrics สำหรับ Prometheus
const client = require('prom-client');

// Metrics
const wsConnections = new client.Gauge({
  name: 'websocket_connections_total',
  help: 'Total WebSocket connections',
  labelNames: ['server_id'],
});

const wsConnectionsPerUser = new client.Histogram({
  name: 'websocket_connections_per_user',
  help: 'WebSocket connections per user',
  buckets: [1, 2, 3, 5, 10],
});

const wsMessagesTotal = new client.Counter({
  name: 'websocket_messages_total',
  help: 'Total WebSocket messages',
  labelNames: ['event_type', 'direction'],
});

const wsRoomSizes = new client.Histogram({
  name: 'websocket_room_size',
  help: 'WebSocket room sizes',
  buckets: [1, 5, 10, 50, 100, 500, 1000, 5000],
});

const wsLatency = new client.Histogram({
  name: 'websocket_message_latency_ms',
  help: 'WebSocket message latency in milliseconds',
  buckets: [1, 5, 10, 25, 50, 100, 250, 500, 1000],
  labelNames: ['event_type'],
});

const SERVER_ID = process.env.HOSTNAME || require('os').hostname();

function setupWebSocketMetrics(io) {
  let connectionCount = 0;
  
  io.on('connection', (socket) => {
    connectionCount++;
    wsConnections.set({ server_id: SERVER_ID }, connectionCount);
    wsMessagesTotal.inc({ event_type: 'connection', direction: 'in' });
    
    // Track message metrics
    const originalEmit = socket.emit.bind(socket);
    socket.emit = function(event, ...args) {
      wsMessagesTotal.inc({ event_type: event, direction: 'out' });
      return originalEmit(event, ...args);
    };
    
    socket.onAny((event) => {
      wsMessagesTotal.inc({ event_type: event, direction: 'in' });
    });
    
    socket.on('disconnect', () => {
      connectionCount--;
      wsConnections.set({ server_id: SERVER_ID }, connectionCount);
    });
  });
  
  // Report room sizes periodically
  setInterval(async () => {
    const rooms = io.sockets.adapter.rooms;
    for (const [roomId, sockets] of rooms) {
      if (roomId.startsWith('room:')) {
        wsRoomSizes.observe(sockets.size);
      }
    }
    
    // Report to Redis for global count
    await pubClient.set(
      `ws:server:${SERVER_ID}:connections`,
      connectionCount,
      'EX', 60
    );
  }, 15000);
  
  return {
    getConnectionCount: () => connectionCount,
    getGlobalConnectionCount: async () => {
      let total = 0;
      const keys = await pubClient.keys('ws:server:*:connections');
      for (const key of keys) {
        total += parseInt(await pubClient.get(key)) || 0;
      }
      return total;
    },
  };
}

module.exports = { setupWebSocketMetrics };
```

### Step 706: WebSocket Load Testing

```javascript
// tests/websocket-load-test.js - k6 WebSocket test
import ws from 'k6/ws';
import { check, sleep } from 'k6';
import { Counter, Trend, Rate } from 'k6/metrics';

const messagesSent = new Counter('ws_messages_sent');
const messageLatency = new Trend('ws_message_latency_ms');
const connectionErrors = new Rate('ws_connection_errors');

export const options = {
  stages: [
    { duration: '1m', target: 100 },    // 100 concurrent WS connections
    { duration: '5m', target: 1000 },   // scale to 1000
    { duration: '5m', target: 5000 },   // scale to 5000
    { duration: '10m', target: 5000 },  // sustain 5000
    { duration: '2m', target: 0 },      // ramp down
  ],
  thresholds: {
    'ws_message_latency_ms': ['p(95)<100', 'p(99)<500'],
    'ws_connection_errors': ['rate<0.01'],
  },
};

export default function() {
  const userId = __VU;
  const WS_URL = `wss://ws.chuaikan.com?userId=${userId}&token=test-token`;
  
  const res = ws.connect(WS_URL, {
    headers: { 'Authorization': 'Bearer test-token' },
  }, function(socket) {
    socket.on('open', () => {
      // Join a test room
      const roomId = `test-room-${Math.floor(userId / 100)}`;
      socket.send(JSON.stringify({
        event: 'room:join',
        roomId,
      }));
      messagesSent.add(1);
    });
    
    socket.on('message', (data) => {
      try {
        const msg = JSON.parse(data);
        
        if (msg.event === 'pong') {
          const latency = Date.now() - msg.sentAt;
          messageLatency.add(latency);
        }
      } catch (e) {}
    });
    
    socket.on('error', (e) => {
      connectionErrors.add(1);
    });
    
    // Send periodic messages
    socket.setInterval(() => {
      const sentAt = Date.now();
      socket.send(JSON.stringify({
        event: 'ping',
        sentAt,
      }));
      messagesSent.add(1);
    }, 10000);  // ping every 10s
    
    // Simulate user activity
    socket.setInterval(() => {
      if (Math.random() < 0.1) {  // 10% send a message
        socket.send(JSON.stringify({
          event: 'message:send',
          roomId: `test-room-${Math.floor(userId / 100)}`,
          content: 'Load test message',
        }));
        messagesSent.add(1);
      }
    }, 5000);
    
    // Close connection after random time
    socket.setTimeout(() => {
      socket.close();
    }, Math.random() * 300000 + 60000);  // 1-6 minutes
  });
  
  check(res, {
    'WebSocket connected': (r) => r && r.status === 101,
  });
  connectionErrors.add(res === null || res.status !== 101 ? 1 : 0);
}
```

### Step 707: Graceful Shutdown

```javascript
// graceful-shutdown.js - Connection draining
async function gracefulShutdown(io, httpServer) {
  console.log('Starting graceful shutdown...');
  
  // 1. หยุดรับ connections ใหม่
  httpServer.close(() => {
    console.log('HTTP server closed (no new connections)');
  });
  
  // 2. แจ้ง clients ว่าจะ shutdown
  io.emit('server:maintenance', {
    message: 'Server is restarting, please reconnect in 30 seconds',
    reconnectDelay: 30000,
  });
  
  // 3. รอให้ clients มีเวลา disconnect
  await new Promise(resolve => setTimeout(resolve, 5000));
  
  // 4. ดู remaining connections
  const sockets = await io.fetchSockets();
  console.log(`Remaining connections: ${sockets.length}`);
  
  // 5. Disconnect remaining clients
  if (sockets.length > 0) {
    console.log(`Force disconnecting ${sockets.length} remaining sockets`);
    
    await Promise.all(
      sockets.map(socket => {
        return new Promise(resolve => {
          socket.disconnect(true);
          resolve();
        });
      })
    );
  }
  
  // 6. Close Redis connections
  await Promise.all([
    pubClient.quit(),
    subClient.quit(),
  ]);
  console.log('Redis connections closed');
  
  console.log('Graceful shutdown complete');
  process.exit(0);
}

// Handle shutdown signals
process.on('SIGTERM', () => gracefulShutdown(io, httpServer));
process.on('SIGINT', () => gracefulShutdown(io, httpServer));

// Handle uncaught errors (prevent crash without cleanup)
process.on('uncaughtException', (err) => {
  console.error('Uncaught exception:', err);
  gracefulShutdown(io, httpServer);
});
```

### Step 708: SOS Real-time Features

```javascript
// sos-realtime.js - Real-time SOS features
class SOSRealtimeService {
  constructor(io, redis) {
    this.io = io;
    this.redis = redis;
  }
  
  // Broadcast SOS alert to nearby users
  async broadcastSOSAlert(alert) {
    const { areaCode, severity, location } = alert;
    
    // Broadcast ไปยัง area subscribers
    this.io.to(`area:${areaCode}`).emit('sos:alert', {
      id: alert.id,
      type: alert.type,
      severity,
      location,
      timestamp: alert.createdAt,
      // ไม่ส่ง sensitive data ไปยัง all users
    });
    
    // Broadcast ไปยัง emergency services
    this.io.to('room:emergency-services').emit('sos:priority', alert);
    
    // Store in Redis สำหรับ users ที่ connect ทีหลัง
    await this.redis.setex(
      `sos:active:${alert.id}`,
      3600,  // active for 1 hour
      JSON.stringify(alert)
    );
    
    // Add to area's active alerts
    await this.redis.lpush(`area:${areaCode}:active-sos`, alert.id);
    await this.redis.ltrim(`area:${areaCode}:active-sos`, 0, 99);  // keep last 100
    
    console.log(`SOS Alert ${alert.id} broadcast to area ${areaCode}`);
    return { sent: true, alertId: alert.id };
  }
  
  // Send active SOS alerts เมื่อ user เพิ่ง connect
  async sendActiveAlerts(socket, areaCode) {
    const alertIds = await this.redis.lrange(`area:${areaCode}:active-sos`, 0, 9);
    
    const alerts = await Promise.all(
      alertIds.map(id => 
        this.redis.get(`sos:active:${id}`)
          .then(data => data ? JSON.parse(data) : null)
      )
    );
    
    const activeAlerts = alerts.filter(Boolean);
    
    if (activeAlerts.length > 0) {
      socket.emit('sos:active-alerts', activeAlerts);
    }
  }
  
  // Update SOS status (resolved, in-progress, etc.)
  async updateSOSStatus(alertId, status, updatedBy) {
    await this.redis.hset(`sos:${alertId}:status`, {
      status,
      updatedBy,
      updatedAt: Date.now(),
    });
    
    // Notify all subscribers
    this.io.emit('sos:status-update', {
      alertId,
      status,
      updatedBy,
      timestamp: new Date().toISOString(),
    });
  }
}

module.exports = SOSRealtimeService;
```

### Step 709: Redis Cluster สำหรับ High Availability

```javascript
// redis-cluster-config.js - Redis Cluster สำหรับ production
const { Cluster } = require('ioredis');

// Redis Cluster (production)
const redisCluster = new Cluster([
  { host: 'redis-node-1.internal', port: 7000 },
  { host: 'redis-node-2.internal', port: 7001 },
  { host: 'redis-node-3.internal', port: 7002 },
  { host: 'redis-node-4.internal', port: 7003 },
  { host: 'redis-node-5.internal', port: 7004 },
  { host: 'redis-node-6.internal', port: 7005 },
], {
  redisOptions: {
    password: process.env.REDIS_PASSWORD,
    tls: process.env.NODE_ENV === 'production' ? {} : undefined,
  },
  // Socket.io Redis adapter ต้องใช้ @socket.io/redis-streams-adapter 
  // กับ Redis Cluster เพราะ pub/sub ทำงานบน single node
  enableOfflineQueue: true,
  maxRedirections: 16,
  retryDelayOnClusterDown: 300,
  retryDelayOnFailover: 1000,
  retryDelayOnTryAgain: 100,
});

// สำหรับ Socket.io + Redis Cluster
// ใช้ Redis Streams adapter แทน pub/sub
const { createAdapter: createStreamsAdapter } = require('@socket.io/redis-streams-adapter');

io.adapter(createStreamsAdapter(redisCluster));
```

### Step 710: Monitoring Dashboard

```javascript
// ws-dashboard.js - WebSocket metrics API
const express = require('express');
const router = express.Router();

// Real-time connection stats
router.get('/connections', async (req, res) => {
  const [localConnections, servers] = await Promise.all([
    Promise.resolve(io.engine.clientsCount),
    pubClient.keys('ws:server:*:connections'),
  ]);
  
  const serverStats = await Promise.all(
    servers.map(async (key) => ({
      server: key.replace('ws:server:', '').replace(':connections', ''),
      connections: parseInt(await pubClient.get(key)) || 0,
    }))
  );
  
  const totalConnections = serverStats.reduce((sum, s) => sum + s.connections, 0);
  
  res.json({
    total_connections: totalConnections,
    servers: serverStats,
    local_connections: localConnections,
  });
});

// Room statistics
router.get('/rooms', async (req, res) => {
  const rooms = io.sockets.adapter.rooms;
  const roomStats = [];
  
  for (const [roomId, sockets] of rooms) {
    if (roomId.startsWith('room:') || roomId.startsWith('area:')) {
      roomStats.push({
        roomId,
        size: sockets.size,
        type: roomId.startsWith('area:') ? 'area' : 'room',
      });
    }
  }
  
  roomStats.sort((a, b) => b.size - a.size);
  
  res.json({
    total_rooms: roomStats.length,
    top_rooms: roomStats.slice(0, 10),
    large_rooms: roomStats.filter(r => r.size > 100).length,
  });
});

module.exports = router;
```

---

## 🔧 Configuration Files

```yaml
# redis-values.yaml - Redis Helm chart values
replica:
  replicaCount: 3

sentinel:
  enabled: true
  quorum: 2
  
auth:
  enabled: true
  password: ""  # Set via secret
  
persistence:
  enabled: true
  size: 10Gi
```

---

## 🧪 Testing

```bash
# Test 1: WebSocket connection
wscat -c wss://ws.chuaikan.com \
  -H "Authorization: Bearer test-token"

# Test 2: Load test
k6 run tests/websocket-load-test.js

# Test 3: Verify Redis pub/sub
# Terminal 1: Monitor Redis
redis-cli monitor | grep "socket.io"

# Terminal 2: Connect to server 1
wscat -c ws://localhost:3001

# Terminal 3: Connect to server 2
wscat -c ws://localhost:3002

# ส่ง message จาก server 1 ควร appear ใน server 2's client
```

---

## ❌ Common Errors & Solutions

### Error 1: Messages not delivered across servers

```javascript
// ปัญหา: Redis adapter ไม่ได้ connect
// ตรวจสอบ Redis connection
const adapter = io.of('/').adapter;
console.log('Adapter type:', adapter.constructor.name);
// ควรเห็น: RedisAdapter หรือ RedisStreamsAdapter

// ตรวจสอบ Redis pub/sub channels
redis-cli subscribe "socket.io#/#"
```

### Error 2: Memory leak จาก rooms ที่ไม่ได้ clean up

```javascript
// แก้ไข: ตั้งค่า room cleanup
io.on('disconnect', (socket) => {
  // Socket.io จัดการ room cleanup อัตโนมัติเมื่อ disconnect
  // แต่ Redis state ต้องล้างเอง
  const userId = socket.handshake.auth.userId;
  roomManager.handleDisconnect(userId, socket.id);
});
```

---

## ✅ Checklist

- [ ] ติดตั้ง @socket.io/redis-adapter
- [ ] สร้าง pub/sub Redis clients แยกกัน
- [ ] ตั้งค่า Socket.io ใช้ websocket transport เท่านั้น
- [ ] Deploy หลาย instances ใน Kubernetes
- [ ] ตั้งค่า Nginx ingress สำหรับ WebSocket proxy
- [ ] Implement room management ใน Redis
- [ ] ตั้งค่า graceful shutdown
- [ ] สร้าง WebSocket metrics
- [ ] รัน load test สำหรับ concurrent connections
- [ ] Monitor Redis memory สำหรับ pub/sub channels

---

## 🔗 References

- [Socket.io Redis Adapter](https://socket.io/docs/v4/redis-adapter/)
- [Socket.io Scaling](https://socket.io/docs/v4/using-multiple-nodes/)
- [Redis Pub/Sub](https://redis.io/docs/manual/pubsub/)
- [k6 WebSocket Testing](https://k6.io/docs/using-k6/protocols/websockets/)

---

*Part 071 | Road to 1,000,000 Users/Day | chuaikan.com*
