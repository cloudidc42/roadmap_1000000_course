# Part 012: WebSocket & Real-time Architecture

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 111-120
> **เวลาโดยประมาณ:** 8 ชั่วโมง
> **Prerequisites:** Part 011 (Load Testing), Part 001-010 (Infrastructure)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

1. เปรียบเทียบ WebSocket vs HTTP Polling vs Server-Sent Events
2. ติดตั้งและ setup Socket.io กับ Node.js 22
3. ใช้ Socket.io Rooms สำหรับ Private Messaging
4. สร้าง Authentication Middleware สำหรับ WebSocket (JWT)
5. จัดการ Reconnection และ Heartbeat
6. ใช้ Redis Adapter สำหรับ Socket.io แบบ Multi-instance
7. สร้าง Real-time Notifications (Post, Like, Comment, SOS)
8. แก้ปัญหา WebSocket Scaling ด้วย Sticky Sessions vs Redis Pub/Sub
9. เชื่อมต่อ Next.js Frontend กับ socket.io-client
10. จัดการ Connection Limits และ Graceful Shutdown

---

## 📖 ทฤษฎีและแนวคิด

### เปรียบเทียบ Real-time Techniques

```
┌─────────────────────────────────────────────────────────────┐
│          Real-time Communication Comparison                  │
├──────────────┬───────────────────────────────────────────── ┤
│ Technique    │ Description                                   │
├──────────────┼───────────────────────────────────────────── ┤
│ HTTP Polling │ Client ส่ง request ทุก N วินาที              │
│              │ ❌ Wasteful bandwidth (ส่วนใหญ่ข้อมูลว่าง)   │
│              │ ❌ Latency สูง (ขึ้นกับ interval)            │
│              │ ✓ Simple, ทำงานทุก browser                   │
├──────────────┼───────────────────────────────────────────── ┤
│ Long Polling │ Client ส่ง request, server hold จนมีข้อมูล  │
│              │ ❌ Connection overhead สูง                    │
│              │ ❌ Server ต้องจัดการ pending connections      │
│              │ ✓ ดีกว่า polling ปกติ                        │
├──────────────┼───────────────────────────────────────────── ┤
│ SSE          │ Server → Client one-way stream                │
│              │ ✓ Simple, native browser support              │
│              │ ✓ Auto-reconnect built-in                     │
│              │ ❌ One-direction only (Server → Client)       │
│              │ ❌ IE/Edge รุ่นเก่าไม่รองรับ                 │
├──────────────┼───────────────────────────────────────────── ┤
│ WebSocket    │ Full-duplex connection ผ่าน TCP               │
│              │ ✓ Real-time two-way communication             │
│              │ ✓ Low latency (~1-5ms)                        │
│              │ ✓ Binary data support                         │
│              │ ❌ Requires WebSocket-aware load balancer     │
│              │ ❌ Connection state ยาก scale                 │
└──────────────┴───────────────────────────────────────────── ┘
```

### WebSocket Protocol Flow

```
Client                                          Server
  │                                               │
  │  HTTP GET /socket.io/?EIO=4&transport=ws      │
  │──────────────────────────────────────────────►│
  │                                               │
  │  HTTP 101 Switching Protocols                 │
  │◄──────────────────────────────────────────────│
  │                                               │
  │  WebSocket Frames (bidirectional)             │
  │◄─────────────────────────────────────────────►│
  │                                               │
  │  Heartbeat PING                               │
  │──────────────────────────────────────────────►│
  │  Heartbeat PONG                               │
  │◄──────────────────────────────────────────────│
  │                                               │
  │  Event: "new_post" {data}                     │
  │◄──────────────────────────────────────────────│
  │                                               │
  │  Event: "like_post" {postId}                  │
  │──────────────────────────────────────────────►│
```

### Socket.io Scaling Architecture (chuaikan.com)

```
                     ┌─────────────────┐
                     │   Load Balancer  │
                     │  (Nginx/HAProxy) │
                     │  sticky sessions │
                     └────────┬────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
     ┌────────▼─────┐ ┌───────▼──────┐ ┌─────▼────────┐
     │ Node.js #1   │ │ Node.js #2   │ │ Node.js #3   │
     │ Socket.io    │ │ Socket.io    │ │ Socket.io    │
     │ (port 3001)  │ │ (port 3002)  │ │ (port 3003)  │
     └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
            │                │                │
            └────────────────┼────────────────┘
                             │
                    ┌────────▼────────┐
                    │   Redis 7       │
                    │  Pub/Sub Adapter│
                    │  (port 6379)    │
                    └─────────────────┘
```

---

## ⚙️ Environment Setup

### Step 111: ติดตั้ง Socket.io และ Dependencies

```bash
# สร้าง project directory
mkdir -p /home/user/chuaikan-ws
cd /home/user/chuaikan-ws

# ติดตั้ง Node.js 22 (ถ้ายังไม่มี)
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs

# ตรวจสอบ version
node --version   # v22.x.x
npm --version    # 10.x.x

# สร้าง package.json
npm init -y

# ติดตั้ง dependencies
npm install socket.io@4.7.5 \
  @socket.io/redis-adapter@8.3.0 \
  redis@4.6.14 \
  jsonwebtoken@9.0.2 \
  express@4.21.1 \
  cors@2.8.5

# ติดตั้ง TypeScript dev dependencies
npm install -D typescript@5.4.5 \
  @types/node@22.0.0 \
  @types/jsonwebtoken@9.0.6 \
  @types/cors@2.8.17 \
  ts-node@10.9.2 \
  nodemon@3.1.0

# สร้าง tsconfig.json
cat > tsconfig.json << 'EOF'
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
EOF
```

---

## 🛠️ Step-by-Step Implementation

### Step 112: สร้าง WebSocket Server หลัก

สร้างไฟล์ `/home/user/chuaikan-ws/src/server.ts`:

```typescript
import express from 'express';
import { createServer } from 'http';
import { Server, Socket } from 'socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';
import cors from 'cors';
import { verifyToken, AuthPayload } from './middleware/auth';
import { setupNotificationHandlers } from './handlers/notifications';
import { setupChatHandlers } from './handlers/chat';
import { setupSOSHandlers } from './handlers/sos';
import { ConnectionManager } from './utils/connectionManager';

// Types
interface AuthenticatedSocket extends Socket {
  userId: string;
  user: AuthPayload;
}

const app = express();
const httpServer = createServer(app);

// CORS configuration
app.use(cors({
  origin: [
    'https://chuaikan.com',
    'https://www.chuaikan.com',
    process.env.NODE_ENV === 'development' ? 'http://localhost:3000' : '',
  ].filter(Boolean),
  credentials: true,
}));

// Socket.io Server configuration
const io = new Server(httpServer, {
  cors: {
    origin: [
      'https://chuaikan.com',
      'https://www.chuaikan.com',
      process.env.NODE_ENV === 'development' ? 'http://localhost:3000' : '',
    ].filter(Boolean),
    methods: ['GET', 'POST'],
    credentials: true,
  },
  // Connection settings
  pingTimeout: 60000,       // 60 วินาที ถ้าไม่มี pong → disconnect
  pingInterval: 25000,      // ส่ง ping ทุก 25 วินาที
  transports: ['websocket', 'polling'], // WebSocket ก่อน, fallback เป็น polling
  maxHttpBufferSize: 1e6,   // 1MB max message size
  allowEIO3: true,          // รองรับ Socket.io v2 clients
});

// Connection Manager สำหรับนับ connections
const connectionManager = new ConnectionManager();

// ============================================
// Step 113: Redis Adapter Setup
// ============================================
async function setupRedisAdapter() {
  const pubClient = createClient({
    url: process.env.REDIS_URL || 'redis://localhost:6379',
    socket: {
      reconnectStrategy: (retries) => Math.min(retries * 100, 3000),
    },
  });

  const subClient = pubClient.duplicate();

  // Error handlers
  pubClient.on('error', (err) => console.error('Redis pub error:', err));
  subClient.on('error', (err) => console.error('Redis sub error:', err));

  await Promise.all([pubClient.connect(), subClient.connect()]);
  console.log('Redis adapter connected');

  io.adapter(createAdapter(pubClient, subClient));
  return { pubClient, subClient };
}

// ============================================
// Step 114: JWT Authentication Middleware
// ============================================
io.use(async (socket: Socket, next) => {
  try {
    // ดึง token จาก handshake (หลายวิธี)
    const token =
      socket.handshake.auth.token ||
      socket.handshake.headers.authorization?.replace('Bearer ', '') ||
      (socket.handshake.query.token as string);

    if (!token) {
      return next(new Error('Authentication token missing'));
    }

    // Verify JWT
    const payload = await verifyToken(token);
    if (!payload) {
      return next(new Error('Invalid or expired token'));
    }

    // Attach user info to socket
    (socket as AuthenticatedSocket).userId = payload.sub;
    (socket as AuthenticatedSocket).user = payload;

    next();
  } catch (error) {
    console.error('Auth middleware error:', error);
    next(new Error('Authentication failed'));
  }
});

// ============================================
// Main Connection Handler
// ============================================
io.on('connection', (socket: Socket) => {
  const authSocket = socket as AuthenticatedSocket;
  const userId = authSocket.userId;

  console.log(`User ${userId} connected - Socket: ${socket.id}`);
  connectionManager.addConnection(userId, socket.id);

  // Join user's personal room (สำหรับ private notifications)
  socket.join(`user:${userId}`);

  // Emit connection acknowledgment
  socket.emit('connected', {
    socketId: socket.id,
    userId,
    timestamp: new Date().toISOString(),
  });

  // Update user online status ใน database
  updateUserOnlineStatus(userId, true);

  // ========================================
  // Setup Event Handlers
  // ========================================
  setupNotificationHandlers(io, socket, authSocket.user);
  setupChatHandlers(io, socket, authSocket.user);
  setupSOSHandlers(io, socket, authSocket.user);

  // ========================================
  // Step 115: Heartbeat Handler
  // ========================================
  socket.on('ping', () => {
    socket.emit('pong', { timestamp: Date.now() });
  });

  // ========================================
  // Disconnect Handler
  // ========================================
  socket.on('disconnect', (reason) => {
    console.log(`User ${userId} disconnected - Reason: ${reason}`);
    connectionManager.removeConnection(userId, socket.id);

    // ถ้าไม่มี connection อื่นของ user นี้เหลือ → set offline
    if (!connectionManager.hasConnections(userId)) {
      updateUserOnlineStatus(userId, false);
    }
  });

  // ========================================
  // Error Handler
  // ========================================
  socket.on('error', (error) => {
    console.error(`Socket error for user ${userId}:`, error);
  });
});

// Health check endpoint
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    connections: connectionManager.getTotalConnections(),
    uptime: process.uptime(),
  });
});

// Start server
const PORT = parseInt(process.env.PORT || '3001');

async function start() {
  const redisClients = await setupRedisAdapter();

  httpServer.listen(PORT, () => {
    console.log(`WebSocket server running on port ${PORT}`);
  });

  // ========================================
  // Step 120: Graceful Shutdown
  // ========================================
  async function gracefulShutdown(signal: string) {
    console.log(`${signal} received. Shutting down gracefully...`);

    // หยุดรับ connection ใหม่
    httpServer.close(async () => {
      console.log('HTTP server closed');

      // แจ้ง clients ให้ reconnect
      io.emit('server_restart', {
        message: 'Server is restarting. Please reconnect.',
        reconnectDelay: 3000,
      });

      // ปิด all socket connections
      io.close();

      // ปิด Redis connections
      await redisClients.pubClient.quit();
      await redisClients.subClient.quit();

      console.log('Graceful shutdown complete');
      process.exit(0);
    });

    // Force shutdown หลัง 30 วินาที
    setTimeout(() => {
      console.error('Could not close connections in time. Forcing shutdown.');
      process.exit(1);
    }, 30000);
  }

  process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
  process.on('SIGINT', () => gracefulShutdown('SIGINT'));
}

start().catch(console.error);

async function updateUserOnlineStatus(userId: string, isOnline: boolean) {
  // TODO: อัปเดต online status ใน PostgreSQL และ/หรือ Redis
}

export { io };
```

### Step 116: Authentication Middleware

สร้างไฟล์ `/home/user/chuaikan-ws/src/middleware/auth.ts`:

```typescript
import jwt from 'jsonwebtoken';

export interface AuthPayload {
  sub: string;       // userId
  email: string;
  username: string;
  role: 'user' | 'moderator' | 'admin';
  iat: number;
  exp: number;
}

const JWT_SECRET = process.env.JWT_SECRET || 'your-secret-key-change-in-production';

export async function verifyToken(token: string): Promise<AuthPayload | null> {
  try {
    const payload = jwt.verify(token, JWT_SECRET) as AuthPayload;
    
    // ตรวจสอบว่า token ยังไม่หมดอายุ
    if (payload.exp < Math.floor(Date.now() / 1000)) {
      return null;
    }
    
    return payload;
  } catch (error) {
    return null;
  }
}

export function generateTestToken(userId: string): string {
  return jwt.sign(
    {
      sub: userId,
      email: `${userId}@test.com`,
      username: `user_${userId}`,
      role: 'user',
    },
    JWT_SECRET,
    { expiresIn: '24h' }
  );
}
```

### Step 117: Real-time Notification Handlers

สร้างไฟล์ `/home/user/chuaikan-ws/src/handlers/notifications.ts`:

```typescript
import { Server, Socket } from 'socket.io';
import { AuthPayload } from '../middleware/auth';

// Notification types สำหรับ chuaikan.com
export type NotificationType =
  | 'new_post'
  | 'new_like'
  | 'new_comment'
  | 'new_follower'
  | 'sos_alert'
  | 'sos_nearby'
  | 'mention'
  | 'system';

export interface Notification {
  id: string;
  type: NotificationType;
  fromUserId?: string;
  fromUsername?: string;
  targetId?: string;        // postId, commentId, sosId
  message: string;
  data?: Record<string, unknown>;
  createdAt: string;
}

export function setupNotificationHandlers(
  io: Server,
  socket: Socket,
  user: AuthPayload
) {
  const userId = user.sub;

  // ============================================
  // Subscribe to location-based SOS alerts
  // ============================================
  socket.on('subscribe_location', (data: {
    lat: number;
    lng: number;
    radius: number; // km
  }) => {
    // ใช้ geohash สำหรับ room naming
    const geohash = encodeGeohash(data.lat, data.lng, 5); // precision 5 = ~5km
    const roomName = `geo:${geohash}`;
    
    // Leave previous location rooms
    socket.rooms.forEach((room) => {
      if (room.startsWith('geo:')) {
        socket.leave(room);
      }
    });
    
    socket.join(roomName);
    socket.emit('subscribed_location', { room: roomName, geohash });
    
    console.log(`User ${userId} subscribed to location room: ${roomName}`);
  });

  // ============================================
  // Subscribe to specific post updates
  // ============================================
  socket.on('watch_post', (postId: string) => {
    socket.join(`post:${postId}`);
  });

  socket.on('unwatch_post', (postId: string) => {
    socket.leave(`post:${postId}`);
  });

  // ============================================
  // Mark notifications as read
  // ============================================
  socket.on('mark_notification_read', async (notificationId: string) => {
    // TODO: อัปเดต database
    socket.emit('notification_marked_read', { notificationId });
  });
}

// ============================================
// Functions สำหรับส่ง notifications จาก API
// ============================================

// ส่ง notification ไปหา user คนใดคนหนึ่ง
export async function sendNotificationToUser(
  io: Server,
  targetUserId: string,
  notification: Notification
) {
  io.to(`user:${targetUserId}`).emit('notification', notification);
}

// ส่ง SOS alert ไปยังพื้นที่ใกล้เคียง
export async function broadcastSOSToArea(
  io: Server,
  sosData: {
    sosId: string;
    type: string;
    severity: 'low' | 'medium' | 'high' | 'critical';
    lat: number;
    lng: number;
    address: string;
    fromUserId: string;
  }
) {
  const geohash = encodeGeohash(sosData.lat, sosData.lng, 5);
  const nearbyGeohashes = getNearbyGeohashes(geohash);

  const notification: Notification = {
    id: `sos_${sosData.sosId}`,
    type: 'sos_nearby',
    fromUserId: sosData.fromUserId,
    targetId: sosData.sosId,
    message: `SOS Alert: ${sosData.type} near ${sosData.address}`,
    data: sosData,
    createdAt: new Date().toISOString(),
  };

  // Broadcast ไปยัง geohash rooms ทั้งหมดในรัศมี
  for (const hash of nearbyGeohashes) {
    io.to(`geo:${hash}`).emit('sos_alert', notification);
  }
}

// ส่ง real-time update เมื่อมี like
export async function broadcastLikeUpdate(
  io: Server,
  postId: string,
  data: { userId: string; username: string; likeCount: number }
) {
  io.to(`post:${postId}`).emit('like_update', {
    postId,
    ...data,
    timestamp: new Date().toISOString(),
  });
}

// Geohash utilities (simplified)
function encodeGeohash(lat: number, lng: number, precision: number): string {
  const base32 = '0123456789bcdefghjkmnpqrstuvwxyz';
  let minLat = -90, maxLat = 90;
  let minLng = -180, maxLng = 180;
  let hash = '';
  let bits = 0;
  let bitsTotal = 0;
  let hashValue = 0;
  let isEven = true;

  while (hash.length < precision) {
    if (isEven) {
      const mid = (minLng + maxLng) / 2;
      if (lng >= mid) { hashValue = (hashValue << 1) + 1; minLng = mid; }
      else { hashValue = (hashValue << 1) + 0; maxLng = mid; }
    } else {
      const mid = (minLat + maxLat) / 2;
      if (lat >= mid) { hashValue = (hashValue << 1) + 1; minLat = mid; }
      else { hashValue = (hashValue << 1) + 0; maxLat = mid; }
    }
    isEven = !isEven;
    if (++bits === 5) {
      hash += base32[hashValue];
      bits = 0;
      hashValue = 0;
    }
    bitsTotal++;
  }
  return hash;
}

function getNearbyGeohashes(geohash: string): string[] {
  // คืน geohash เอง + geohashes รอบข้าง (simplified)
  return [geohash];
}
```

### Step 118: Chat Handler (Private Messaging)

สร้างไฟล์ `/home/user/chuaikan-ws/src/handlers/chat.ts`:

```typescript
import { Server, Socket } from 'socket.io';
import { AuthPayload } from '../middleware/auth';

interface ChatMessage {
  id: string;
  conversationId: string;
  senderId: string;
  senderUsername: string;
  content: string;
  type: 'text' | 'image' | 'location';
  mediaUrl?: string;
  createdAt: string;
}

export function setupChatHandlers(
  io: Server,
  socket: Socket,
  user: AuthPayload
) {
  const senderId = user.sub;

  // เข้า conversation room
  socket.on('join_conversation', (conversationId: string) => {
    // TODO: ตรวจสอบว่า user เป็นสมาชิกของ conversation นี้
    socket.join(`conversation:${conversationId}`);
    socket.emit('joined_conversation', { conversationId });
  });

  // ออกจาก conversation room
  socket.on('leave_conversation', (conversationId: string) => {
    socket.leave(`conversation:${conversationId}`);
  });

  // ส่งข้อความ
  socket.on('send_message', async (data: {
    conversationId: string;
    content: string;
    type: 'text' | 'image' | 'location';
    mediaUrl?: string;
    tempId: string; // client-generated temp ID สำหรับ optimistic UI
  }) => {
    try {
      // TODO: บันทึกลง database
      const message: ChatMessage = {
        id: generateMessageId(),
        conversationId: data.conversationId,
        senderId,
        senderUsername: user.username,
        content: data.content,
        type: data.type,
        mediaUrl: data.mediaUrl,
        createdAt: new Date().toISOString(),
      };

      // ส่งไปยังทุกคนใน conversation (รวมผู้ส่ง)
      io.to(`conversation:${data.conversationId}`).emit('new_message', message);

      // ยืนยันกับผู้ส่ง (พร้อม tempId เพื่อ match กับ optimistic message)
      socket.emit('message_sent', {
        tempId: data.tempId,
        message,
      });

    } catch (error) {
      socket.emit('message_error', {
        tempId: data.tempId,
        error: 'Failed to send message',
      });
    }
  });

  // Typing indicator
  socket.on('typing_start', (conversationId: string) => {
    socket.to(`conversation:${conversationId}`).emit('user_typing', {
      userId: senderId,
      username: user.username,
    });
  });

  socket.on('typing_stop', (conversationId: string) => {
    socket.to(`conversation:${conversationId}`).emit('user_stop_typing', {
      userId: senderId,
    });
  });

  // Message read receipt
  socket.on('mark_read', (data: {
    conversationId: string;
    lastReadMessageId: string;
  }) => {
    socket.to(`conversation:${data.conversationId}`).emit('messages_read', {
      userId: senderId,
      lastReadMessageId: data.lastReadMessageId,
      readAt: new Date().toISOString(),
    });
  });
}

function generateMessageId(): string {
  return `msg_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
}
```

### Step 119: Connection Manager Utility

สร้างไฟล์ `/home/user/chuaikan-ws/src/utils/connectionManager.ts`:

```typescript
import { createClient } from 'redis';

export class ConnectionManager {
  // In-memory สำหรับ connections ของ server นี้
  private connections = new Map<string, Set<string>>();
  private redisClient: ReturnType<typeof createClient> | null = null;

  constructor() {
    this.setupRedis();
  }

  private async setupRedis() {
    this.redisClient = createClient({
      url: process.env.REDIS_URL || 'redis://localhost:6379',
    });
    await this.redisClient.connect();
  }

  // เพิ่ม connection
  addConnection(userId: string, socketId: string) {
    if (!this.connections.has(userId)) {
      this.connections.set(userId, new Set());
    }
    this.connections.get(userId)!.add(socketId);

    // บันทึกใน Redis สำหรับ cross-server tracking
    if (this.redisClient) {
      this.redisClient.sAdd(`ws:user:${userId}`, socketId);
      this.redisClient.expire(`ws:user:${userId}`, 3600); // expire 1 ชั่วโมง
    }
  }

  // ลบ connection
  removeConnection(userId: string, socketId: string) {
    const userConnections = this.connections.get(userId);
    if (userConnections) {
      userConnections.delete(socketId);
      if (userConnections.size === 0) {
        this.connections.delete(userId);
      }
    }

    if (this.redisClient) {
      this.redisClient.sRem(`ws:user:${userId}`, socketId);
    }
  }

  // ตรวจสอบว่า user มี connection อยู่หรือเปล่า
  hasConnections(userId: string): boolean {
    const local = (this.connections.get(userId)?.size ?? 0) > 0;
    return local;
  }

  // นับ total connections ของ server นี้
  getTotalConnections(): number {
    let total = 0;
    this.connections.forEach((sockets) => {
      total += sockets.size;
    });
    return total;
  }

  // ตรวจสอบว่า user online อยู่หรือเปล่า (across all servers)
  async isUserOnline(userId: string): Promise<boolean> {
    if (!this.redisClient) return this.hasConnections(userId);
    const count = await this.redisClient.sCard(`ws:user:${userId}`);
    return count > 0;
  }
}
```

---

## 🔧 Configuration Files

### SOS Handler

สร้างไฟล์ `/home/user/chuaikan-ws/src/handlers/sos.ts`:

```typescript
import { Server, Socket } from 'socket.io';
import { AuthPayload } from '../middleware/auth';
import { broadcastSOSToArea } from './notifications';

export function setupSOSHandlers(
  io: Server,
  socket: Socket,
  user: AuthPayload
) {
  const userId = user.sub;

  // สร้าง SOS alert ผ่าน WebSocket (สำหรับ real-time)
  socket.on('create_sos', async (data: {
    type: string;
    severity: 'low' | 'medium' | 'high' | 'critical';
    lat: number;
    lng: number;
    address: string;
    description?: string;
  }) => {
    try {
      // TODO: บันทึกลง database ผ่าน REST API
      const sosId = `sos_${Date.now()}`;

      // Broadcast ไปยังพื้นที่ใกล้เคียง
      await broadcastSOSToArea(io, {
        sosId,
        type: data.type,
        severity: data.severity,
        lat: data.lat,
        lng: data.lng,
        address: data.address,
        fromUserId: userId,
      });

      socket.emit('sos_created', { sosId });

    } catch (error) {
      socket.emit('sos_error', { error: 'Failed to create SOS' });
    }
  });

  // อัปเดตสถานะ SOS
  socket.on('update_sos_status', (data: {
    sosId: string;
    status: 'active' | 'responding' | 'resolved';
  }) => {
    // Broadcast status update ไปยังทุกคนที่ subscribe
    io.to(`sos:${data.sosId}`).emit('sos_status_updated', {
      sosId: data.sosId,
      status: data.status,
      updatedBy: userId,
      updatedAt: new Date().toISOString(),
    });
  });

  // Volunteer เสนอตัวช่วย SOS
  socket.on('volunteer_sos', (sosId: string) => {
    socket.join(`sos:${sosId}`);
    io.to(`sos:${sosId}`).emit('volunteer_joined', {
      sosId,
      volunteerId: userId,
      username: user.username,
      joinedAt: new Date().toISOString(),
    });
  });
}
```

### Nginx Configuration สำหรับ WebSocket

สร้างไฟล์ `/etc/nginx/sites-available/chuaikan-ws`:

```nginx
upstream ws_backend {
    # Sticky sessions โดยใช้ IP hash
    ip_hash;
    
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
    
    keepalive 64;
}

server {
    listen 443 ssl http2;
    server_name ws.chuaikan.com;

    ssl_certificate /etc/ssl/certs/chuaikan.com.crt;
    ssl_certificate_key /etc/ssl/private/chuaikan.com.key;

    # WebSocket upgrade headers
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # WebSocket timeout settings
    proxy_read_timeout 86400s;   # 24 ชั่วโมง
    proxy_send_timeout 86400s;
    proxy_connect_timeout 75s;

    location /socket.io/ {
        proxy_pass http://ws_backend;
        proxy_buffering off;
    }

    location /health {
        proxy_pass http://ws_backend;
    }
}
```

### PM2 Configuration สำหรับรัน Multiple Instances

สร้างไฟล์ `/home/user/chuaikan-ws/ecosystem.config.js`:

```javascript
module.exports = {
  apps: [
    {
      name: 'chuaikan-ws-1',
      script: './dist/server.js',
      instances: 1,
      env: {
        NODE_ENV: 'production',
        PORT: 3001,
        REDIS_URL: 'redis://localhost:6379',
        JWT_SECRET: process.env.JWT_SECRET,
      },
      max_memory_restart: '1G',
      error_file: './logs/ws-1-error.log',
      out_file: './logs/ws-1-out.log',
    },
    {
      name: 'chuaikan-ws-2',
      script: './dist/server.js',
      instances: 1,
      env: {
        NODE_ENV: 'production',
        PORT: 3002,
        REDIS_URL: 'redis://localhost:6379',
        JWT_SECRET: process.env.JWT_SECRET,
      },
    },
    {
      name: 'chuaikan-ws-3',
      script: './dist/server.js',
      instances: 1,
      env: {
        NODE_ENV: 'production',
        PORT: 3003,
        REDIS_URL: 'redis://localhost:6379',
        JWT_SECRET: process.env.JWT_SECRET,
      },
    },
  ],
};
```

---

## 🧪 Testing

### Step 120: Frontend Next.js Integration

สร้างไฟล์ `/home/user/chuaikan-web/src/hooks/useSocket.ts`:

```typescript
'use client';

import { useEffect, useRef, useState, useCallback } from 'react';
import { io, Socket } from 'socket.io-client';
import { useSession } from 'next-auth/react';

const WS_URL = process.env.NEXT_PUBLIC_WS_URL || 'wss://ws.chuaikan.com';

interface UseSocketReturn {
  socket: Socket | null;
  isConnected: boolean;
  connectionError: string | null;
  emit: (event: string, data?: unknown) => void;
}

export function useSocket(): UseSocketReturn {
  const { data: session } = useSession();
  const socketRef = useRef<Socket | null>(null);
  const [isConnected, setIsConnected] = useState(false);
  const [connectionError, setConnectionError] = useState<string | null>(null);

  useEffect(() => {
    if (!session?.accessToken) return;

    // สร้าง socket connection
    const socket = io(WS_URL, {
      auth: { token: session.accessToken },
      transports: ['websocket', 'polling'],
      reconnection: true,
      reconnectionAttempts: 5,
      reconnectionDelay: 1000,
      reconnectionDelayMax: 5000,
      timeout: 10000,
    });

    socketRef.current = socket;

    // Event handlers
    socket.on('connect', () => {
      console.log('Socket connected:', socket.id);
      setIsConnected(true);
      setConnectionError(null);
    });

    socket.on('disconnect', (reason) => {
      console.log('Socket disconnected:', reason);
      setIsConnected(false);

      if (reason === 'io server disconnect') {
        // Server ปิด connection → reconnect manually
        socket.connect();
      }
    });

    socket.on('connect_error', (error) => {
      console.error('Socket connection error:', error.message);
      setConnectionError(error.message);
      setIsConnected(false);
    });

    // Handle server restart notification
    socket.on('server_restart', ({ reconnectDelay }) => {
      console.log('Server restarting...');
      setTimeout(() => {
        socket.connect();
      }, reconnectDelay);
    });

    return () => {
      socket.disconnect();
      socketRef.current = null;
    };
  }, [session?.accessToken]);

  const emit = useCallback((event: string, data?: unknown) => {
    if (socketRef.current?.connected) {
      socketRef.current.emit(event, data);
    } else {
      console.warn('Socket not connected, cannot emit:', event);
    }
  }, []);

  return {
    socket: socketRef.current,
    isConnected,
    connectionError,
    emit,
  };
}

// Hook สำหรับ real-time notifications
export function useNotifications() {
  const { socket } = useSocket();
  const [notifications, setNotifications] = useState<Notification[]>([]);
  const [unreadCount, setUnreadCount] = useState(0);

  useEffect(() => {
    if (!socket) return;

    socket.on('notification', (notification: Notification) => {
      setNotifications((prev) => [notification, ...prev].slice(0, 50));
      setUnreadCount((prev) => prev + 1);
    });

    socket.on('sos_alert', (alert: Notification) => {
      // แสดง SOS alert แบบ priority (ใช้ toast หรือ modal)
      console.log('SOS Alert received:', alert);
      setNotifications((prev) => [alert, ...prev]);
      setUnreadCount((prev) => prev + 1);
    });

    return () => {
      socket.off('notification');
      socket.off('sos_alert');
    };
  }, [socket]);

  return { notifications, unreadCount, setUnreadCount };
}
```

### Build และ Deploy

```bash
# Build TypeScript
cd /home/user/chuaikan-ws
npm run build  # tsc

# รัน ด้วย PM2
pm2 start ecosystem.config.js
pm2 save
pm2 startup

# ตรวจสอบสถานะ
pm2 status
pm2 logs chuaikan-ws-1

# ทดสอบ WebSocket connection ด้วย wscat
npm install -g wscat
TOKEN=$(node -e "const jwt = require('jsonwebtoken'); console.log(jwt.sign({sub:'user1',email:'test@test.com',username:'test',role:'user'}, 'secret', {expiresIn:'1h'}))")
wscat -c "ws://localhost:3001/socket.io/?EIO=4&transport=websocket&token=$TOKEN"
```

---

## ❌ Common Errors & Solutions

### Error 1: "Cross-site WebSocket handshake was denied"

```bash
# ตรวจสอบ CORS settings ใน server.ts
# ต้องเพิ่ม origin ที่ถูกต้อง
const io = new Server(httpServer, {
  cors: {
    origin: ['https://chuaikan.com'],  # ต้องตรงกับ frontend URL
    credentials: true,
  }
});
```

### Error 2: Socket.io ไม่ทำงานข้าม servers (Redis adapter ปัญหา)

```bash
# ตรวจสอบ Redis connection
redis-cli ping  # ต้องได้ PONG

# ตรวจสอบ Redis pub/sub
redis-cli monitor  # ดู real-time commands

# ทดสอบ pub/sub manually
redis-cli subscribe socket.io
# ใน terminal อื่น:
redis-cli publish socket.io "test message"
```

### Error 3: Memory leak - Socket connections ไม่ถูก cleanup

```typescript
// เพิ่ม cleanup ใน disconnect handler
socket.on('disconnect', () => {
  // Remove all listeners เพื่อป้องกัน memory leak
  socket.removeAllListeners();
  
  // Leave all rooms
  socket.rooms.forEach((room) => {
    socket.leave(room);
  });
});
```

---

## ✅ Checklist

- [ ] Socket.io server ทำงานบน Node.js 22
- [ ] JWT authentication middleware ทำงานถูกต้อง
- [ ] Redis adapter configured สำหรับ multi-instance
- [ ] Rooms setup: user:${userId}, conversation:${id}, geo:${hash}, sos:${id}
- [ ] Heartbeat/ping-pong ทำงาน (pingInterval: 25s, pingTimeout: 60s)
- [ ] Reconnection logic ทำงาน (client auto-reconnect)
- [ ] Graceful shutdown ส่ง event แจ้ง clients ก่อน shutdown
- [ ] Nginx configured ด้วย ip_hash sticky sessions
- [ ] Connection limits monitored
- [ ] Next.js useSocket hook ทำงาน
- [ ] SOS real-time broadcast ทำงาน
- [ ] Private messaging rooms ทำงาน
- [ ] PM2 ecosystem.config.js deploy ได้
- [ ] Load test WebSocket ด้วย k6 (Part 011)

---

## 🔗 References

- [Socket.io Documentation](https://socket.io/docs/v4/)
- [Socket.io Redis Adapter](https://socket.io/docs/v4/redis-adapter/)
- [Socket.io Scaling](https://socket.io/docs/v4/using-multiple-nodes/)
- [WebSocket Protocol RFC 6455](https://datatracker.ietf.org/doc/html/rfc6455)

---

*Part 012 | Road to 1,000,000 Users/Day | chuaikan.com*
