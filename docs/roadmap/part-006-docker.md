# Part 006: Docker — Containerization
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Beginner
> **Steps:** 51-60
> **เวลาโดยประมาณ:** 6 ชั่วโมง
> **Prerequisites:** Part 001-005 (Linux, Git, Node.js, PostgreSQL, Redis)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

1. เข้าใจแนวคิด Docker: images, containers, layers, registry
2. เขียน Dockerfile สำหรับ Next.js 15 แบบ multi-stage build
3. เขียน Dockerfile สำหรับ Node.js 22 API แบบ production-optimized
4. สร้าง docker-compose.yml สำหรับ local development
5. จัดการ Docker networking (bridge, host, overlay)
6. จัดการ volumes สำหรับ persistent data
7. ใช้งาน Docker secrets
8. ตั้งค่า health checks
9. Optimize Docker build ด้วย layer caching และ BuildKit
10. แก้ปัญหาที่พบบ่อยใน Docker

---

## 📖 ทฤษฎีและแนวคิด

### Docker คืออะไร?

Docker คือ platform สำหรับ containerization ที่ช่วยให้เราสามารถ package แอปพลิเคชันพร้อมกับ dependencies ทั้งหมดไว้ใน container เดียว ทำให้ "run ที่ไหนก็ได้เหมือนกัน"

```
┌─────────────────────────────────────────────┐
│              Docker Architecture             │
├─────────────────────────────────────────────┤
│                                             │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐ │
│   │Container │  │Container │  │Container │ │
│   │ Next.js  │  │   API    │  │PostgreSQL│ │
│   └────┬─────┘  └────┬─────┘  └────┬─────┘ │
│        │              │              │       │
│   ─────┴──────────────┴──────────────┴───── │
│                 Docker Engine               │
│   ─────────────────────────────────────── │
│                Ubuntu 24.04 LTS             │
└─────────────────────────────────────────────┘
```

### Image vs Container

```
Image (แม่แบบ / Blueprint)
  └── Container 1 (instance ที่กำลัง run อยู่)
  └── Container 2 (instance อีกตัว)
  └── Container 3 (instance อีกตัว)
```

- **Image**: ไฟล์ที่ไม่เปลี่ยนแปลง (immutable) ประกอบด้วย layers ซ้อนกัน
- **Container**: Image ที่กำลัง run อยู่ มี writable layer อยู่ด้านบน

### Docker Layers

```
┌──────────────────────────┐  ← Writable Layer (Container)
├──────────────────────────┤
│     COPY . .             │  ← Layer 4 (application code)
├──────────────────────────┤
│     RUN npm install      │  ← Layer 3 (dependencies)
├──────────────────────────┤
│     WORKDIR /app         │  ← Layer 2
├──────────────────────────┤
│   FROM node:22-alpine    │  ← Layer 1 (base image)
└──────────────────────────┘
```

แต่ละ instruction ใน Dockerfile สร้าง layer ใหม่ Docker จะ cache layers ที่ไม่เปลี่ยนแปลง ทำให้ build ครั้งต่อไปเร็วขึ้น

### Multi-stage Build

เป็นเทคนิคที่ใช้หลาย `FROM` ใน Dockerfile เดียว ช่วยลดขนาด image สุดท้ายโดยไม่รวม dev dependencies

```
Stage 1: builder
  - ติดตั้ง all dependencies
  - build application
  
Stage 2: runner
  - copy เฉพาะ production files
  - ไม่มี build tools, dev dependencies
  - image เล็กลงมาก
```

---

## ⚙️ Environment Setup

### Step 51: ติดตั้ง Docker บน Ubuntu 24.04 LTS

```bash
# ลบ old versions ถ้ามี
sudo apt-get remove -y docker docker-engine docker.io containerd runc

# ติดตั้ง dependencies
sudo apt-get update
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# เพิ่ม Docker GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# เพิ่ม Docker repository
echo \
  "deb [arch="$(dpkg --print-architecture)" \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# ติดตั้ง Docker Engine
sudo apt-get update
sudo apt-get install -y \
    docker-ce \
    docker-ce-cli \
    containerd.io \
    docker-buildx-plugin \
    docker-compose-plugin

# เพิ่ม user เข้า docker group (ไม่ต้องใช้ sudo)
sudo usermod -aG docker $USER
newgrp docker

# ตรวจสอบ version
docker --version
# Docker version 27.x.x, build xxxxxxx

docker compose version
# Docker Compose version v2.x.x

# ทดสอบ
docker run hello-world
```

### เปิดใช้งาน BuildKit (แนะนำ)

```bash
# ตั้งค่า BuildKit เป็น default
echo 'export DOCKER_BUILDKIT=1' >> ~/.bashrc
echo 'export COMPOSE_DOCKER_CLI_BUILD=1' >> ~/.bashrc
source ~/.bashrc

# หรือตั้งค่าใน /etc/docker/daemon.json
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "features": {
    "buildkit": true
  },
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF

sudo systemctl restart docker
```

---

## 🛠️ Step-by-Step Implementation

### Step 52: Dockerfile สำหรับ Next.js 15

สร้างไฟล์ `apps/web/Dockerfile`:

```dockerfile
# ============================================
# Stage 1: Dependencies
# ============================================
FROM node:22-alpine AS deps

# ติดตั้ง libc6-compat สำหรับ alpine compatibility
RUN apk add --no-cache libc6-compat

WORKDIR /app

# Copy package files ก่อน (layer caching)
COPY package.json package-lock.json* ./
COPY apps/web/package.json ./apps/web/

# ติดตั้ง dependencies
RUN npm ci --workspace=apps/web

# ============================================
# Stage 2: Builder
# ============================================
FROM node:22-alpine AS builder

WORKDIR /app

# Copy dependencies จาก stage แรก
COPY --from=deps /app/node_modules ./node_modules
COPY --from=deps /app/apps/web/node_modules ./apps/web/node_modules

# Copy source code
COPY apps/web ./apps/web
COPY packages ./packages

# ตั้งค่า environment สำหรับ build
ENV NEXT_TELEMETRY_DISABLED=1
ENV NODE_ENV=production

WORKDIR /app/apps/web

# Build Next.js application
RUN npm run build

# ============================================
# Stage 3: Runner (Production image)
# ============================================
FROM node:22-alpine AS runner

WORKDIR /app

ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

# สร้าง non-root user เพื่อความปลอดภัย
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs

# Copy public files
COPY --from=builder /app/apps/web/public ./public

# Copy standalone output (ต้องตั้งค่า output: 'standalone' ใน next.config.js)
COPY --from=builder --chown=nextjs:nodejs /app/apps/web/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/apps/web/.next/static ./.next/static

# เปลี่ยนไปใช้ non-root user
USER nextjs

# Expose port
EXPOSE 3000

ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:3000/api/health || exit 1

# Start application
CMD ["node", "server.js"]
```

ตั้งค่า `next.config.js` ให้ใช้ standalone output:

```javascript
// apps/web/next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: 'standalone',
  
  // Optimize images
  images: {
    domains: ['chuaikan.com', 'r2.chuaikan.com'],
    formats: ['image/avif', 'image/webp'],
  },
  
  // Security headers
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          { key: 'X-Content-Type-Options', value: 'nosniff' },
          { key: 'X-Frame-Options', value: 'DENY' },
          { key: 'X-XSS-Protection', value: '1; mode=block' },
        ],
      },
    ]
  },
}

module.exports = nextConfig
```

### Step 53: Dockerfile สำหรับ Node.js 22 API

สร้างไฟล์ `apps/api/Dockerfile`:

```dockerfile
# ============================================
# Stage 1: Base image with dependencies
# ============================================
FROM node:22-alpine AS base

# ติดตั้ง security updates
RUN apk update && apk upgrade && \
    apk add --no-cache \
        dumb-init \
        curl

WORKDIR /app

# ============================================
# Stage 2: Install dependencies
# ============================================
FROM base AS deps

COPY package.json package-lock.json* ./
COPY apps/api/package.json ./apps/api/

# ติดตั้ง production dependencies เท่านั้น
RUN npm ci --workspace=apps/api --omit=dev

# ============================================
# Stage 3: Build TypeScript
# ============================================
FROM base AS build

COPY package.json package-lock.json* ./
COPY apps/api/package.json ./apps/api/

# ติดตั้ง all dependencies (รวม dev)
RUN npm ci --workspace=apps/api

# Copy source code
COPY apps/api ./apps/api
COPY packages ./packages

WORKDIR /app/apps/api

# Build TypeScript
RUN npm run build

# ============================================
# Stage 4: Production runner
# ============================================
FROM base AS runner

# สร้าง non-root user
RUN addgroup --system --gid 1001 apigroup && \
    adduser --system --uid 1001 apiuser

WORKDIR /app

# Copy production dependencies
COPY --from=deps --chown=apiuser:apigroup /app/node_modules ./node_modules

# Copy built application
COPY --from=build --chown=apiuser:apigroup /app/apps/api/dist ./dist
COPY --from=build --chown=apiuser:apigroup /app/apps/api/package.json ./

# ใช้ dumb-init เป็น PID 1 เพื่อจัดการ signals อย่างถูกต้อง
USER apiuser

EXPOSE 4000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=20s --retries=3 \
    CMD curl -f http://localhost:4000/health || exit 1

# Start with dumb-init
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "dist/index.js"]
```

### Step 54: .dockerignore Files

สร้างไฟล์ `apps/web/.dockerignore`:

```dockerignore
# Dependencies
node_modules
.npm
.pnpm-store

# Build outputs
.next
out
dist
build

# Development files
.env
.env.local
.env.development
.env.test
*.env

# Version control
.git
.gitignore
.gitattributes

# Documentation
README.md
CHANGELOG.md
docs/

# Tests
__tests__
*.test.ts
*.test.tsx
*.spec.ts
*.spec.tsx
coverage/
.nyc_output

# IDE files
.vscode
.idea
*.swp
*.swo
.DS_Store
Thumbs.db

# Logs
logs/
*.log
npm-debug.log*

# Docker files (ไม่ต้องรวมใน image)
Dockerfile
.dockerignore
docker-compose*.yml

# CI/CD
.github/
.gitlab-ci.yml

# TypeScript cache
*.tsbuildinfo
tsconfig.tsbuildinfo
```

สร้างไฟล์ `apps/api/.dockerignore` (เหมือนกัน + เพิ่ม):

```dockerignore
# Dependencies
node_modules
.npm

# Build outputs
dist
build

# Development files
.env*
!.env.example

# Version control
.git
.gitignore

# Tests
__tests__
*.test.ts
*.spec.ts
coverage/

# IDE files
.vscode
.idea
*.swp
.DS_Store

# Logs
logs/
*.log

# Docker files
Dockerfile
.dockerignore
docker-compose*.yml

# Documentation
README.md
docs/

# TypeScript
*.tsbuildinfo
src/
```

### Step 55: docker-compose.yml สำหรับ Local Development

```yaml
# docker-compose.yml (root directory)
version: '3.9'

# =====================================
# Named Volumes
# =====================================
volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
  pgadmin_data:
    driver: local

# =====================================
# Networks
# =====================================
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
  database:
    driver: bridge

# =====================================
# Services
# =====================================
services:

  # ----------------------------------
  # PostgreSQL 17
  # ----------------------------------
  postgres:
    image: postgres:17-alpine
    container_name: chuaikan_postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: chuaikan_dev
      POSTGRES_USER: chuaikan_user
      POSTGRES_PASSWORD_FILE: /run/secrets/postgres_password
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/01-init.sql:ro
    networks:
      - database
    secrets:
      - postgres_password
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U chuaikan_user -d chuaikan_dev"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    deploy:
      resources:
        limits:
          memory: 512M

  # ----------------------------------
  # Redis 7
  # ----------------------------------
  redis:
    image: redis:7-alpine
    container_name: chuaikan_redis
    restart: unless-stopped
    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD:-dev_redis_pass}
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru
      --appendonly yes
      --appendfsync everysec
    volumes:
      - redis_data:/data
    networks:
      - backend
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "--no-auth-warning", "-a", "${REDIS_PASSWORD:-dev_redis_pass}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 10s

  # ----------------------------------
  # Node.js API
  # ----------------------------------
  api:
    build:
      context: .
      dockerfile: apps/api/Dockerfile
      target: runner
      cache_from:
        - type=local,src=/tmp/.buildx-cache
      args:
        - BUILDKIT_INLINE_CACHE=1
    container_name: chuaikan_api
    restart: unless-stopped
    environment:
      NODE_ENV: development
      PORT: 4000
      DATABASE_URL: postgresql://chuaikan_user:${POSTGRES_PASSWORD:-dev_pass}@postgres:5432/chuaikan_dev
      REDIS_URL: redis://:${REDIS_PASSWORD:-dev_redis_pass}@redis:6379
      JWT_SECRET_FILE: /run/secrets/jwt_secret
    volumes:
      - ./apps/api/src:/app/src:ro  # hot reload ใน dev
    networks:
      - backend
      - database
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    secrets:
      - jwt_secret
    ports:
      - "4000:4000"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:4000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 20s

  # ----------------------------------
  # Next.js 15
  # ----------------------------------
  web:
    build:
      context: .
      dockerfile: apps/web/Dockerfile
      target: runner
      args:
        - BUILDKIT_INLINE_CACHE=1
    container_name: chuaikan_web
    restart: unless-stopped
    environment:
      NODE_ENV: development
      NEXT_PUBLIC_API_URL: http://api:4000
      INTERNAL_API_URL: http://api:4000
    networks:
      - frontend
      - backend
    depends_on:
      api:
        condition: service_healthy
    ports:
      - "3000:3000"
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  # ----------------------------------
  # pgAdmin (Development only)
  # ----------------------------------
  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: chuaikan_pgadmin
    restart: unless-stopped
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@chuaikan.com
      PGADMIN_DEFAULT_PASSWORD_FILE: /run/secrets/pgadmin_password
      PGADMIN_CONFIG_SERVER_MODE: 'False'
    volumes:
      - pgadmin_data:/var/lib/pgadmin
    networks:
      - database
    depends_on:
      postgres:
        condition: service_healthy
    secrets:
      - pgadmin_password
    ports:
      - "5050:80"
    profiles:
      - dev-tools  # ใช้งานเฉพาะเมื่อต้องการ

# =====================================
# Secrets
# =====================================
secrets:
  postgres_password:
    file: ./secrets/postgres_password.txt
  jwt_secret:
    file: ./secrets/jwt_secret.txt
  pgadmin_password:
    file: ./secrets/pgadmin_password.txt
```

สร้าง secrets files:

```bash
# สร้าง directory สำหรับ secrets
mkdir -p secrets

# สร้าง strong passwords
openssl rand -base64 32 > secrets/postgres_password.txt
openssl rand -base64 64 > secrets/jwt_secret.txt
echo "pgadmin_dev_password" > secrets/pgadmin_password.txt

# ป้องกัน secrets ไม่ให้ push ขึ้น git
echo "secrets/" >> .gitignore

# ตรวจสอบ
ls -la secrets/
```

### Step 56: Docker Networking

```bash
# ดู networks ทั้งหมด
docker network ls

# ดูรายละเอียด network
docker network inspect chuaikan_backend

# สร้าง custom network
docker network create \
    --driver bridge \
    --subnet 172.20.0.0/16 \
    --gateway 172.20.0.1 \
    chuaikan_production

# ทดสอบการเชื่อมต่อระหว่าง containers
docker exec chuaikan_web ping -c 3 chuaikan_api

# ดู IP addresses ของ containers
docker inspect chuaikan_api | grep IPAddress
```

**Network Types:**

```
Bridge Network (default)
  - Containers บน host เดียวคุยกันได้
  - มี NAT ออก internet
  - เหมาะสำหรับ single-host deployment

Host Network
  - Container ใช้ network stack ของ host โดยตรง
  - ประสิทธิภาพสูงกว่า แต่ไม่มี isolation
  - เหมาะสำหรับ high-performance apps

Overlay Network
  - Containers บน hosts หลายเครื่องคุยกันได้
  - ใช้กับ Docker Swarm หรือ Kubernetes
  - เหมาะสำหรับ distributed systems
```

### Step 57: Volume Management

```bash
# ดู volumes ทั้งหมด
docker volume ls

# ดูรายละเอียด volume
docker volume inspect chuaikan_postgres_data

# สร้าง named volume
docker volume create chuaikan_postgres_backup

# Backup postgres data
docker run --rm \
    -v chuaikan_postgres_data:/source:ro \
    -v $(pwd)/backups:/backup \
    alpine tar czf /backup/postgres-$(date +%Y%m%d).tar.gz -C /source .

# Restore postgres data
docker run --rm \
    -v chuaikan_postgres_data:/target \
    -v $(pwd)/backups:/backup \
    alpine tar xzf /backup/postgres-20241201.tar.gz -C /target

# ลบ unused volumes
docker volume prune

# ดูขนาดของ volumes
docker system df -v
```

### Step 58: Docker Build Optimization

```bash
# Build ด้วย BuildKit และ cache
DOCKER_BUILDKIT=1 docker build \
    --cache-from type=local,src=/tmp/.buildx-cache \
    --cache-to type=local,dest=/tmp/.buildx-cache-new,mode=max \
    -t chuaikan/web:latest \
    -f apps/web/Dockerfile \
    .

# ใช้ buildx สำหรับ multi-platform build
docker buildx create --use --name chuaikan-builder
docker buildx build \
    --platform linux/amd64,linux/arm64 \
    -t chuaikan/web:latest \
    --push \
    -f apps/web/Dockerfile \
    .

# ดูขนาด image
docker images --format "{{.Repository}}:{{.Tag}}\t{{.Size}}"

# Analyze layers
docker history chuaikan/web:latest --no-trunc
```

**Tips สำหรับ Layer Caching:**

```dockerfile
# ❌ แบบนี้ cache จะ invalid ทุกครั้งที่ code เปลี่ยน
COPY . .
RUN npm install

# ✅ แบบนี้ cache dependencies ได้ถ้า package.json ไม่เปลี่ยน
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
```

### Step 59: Docker Commands Cheatsheet

```bash
# ===== Images =====
docker images                          # ดู images ทั้งหมด
docker pull node:22-alpine             # ดึง image
docker build -t myapp:1.0 .            # build image
docker tag myapp:1.0 myapp:latest      # tag image
docker push myapp:latest               # push to registry
docker rmi myapp:1.0                   # ลบ image
docker image prune                     # ลบ dangling images

# ===== Containers =====
docker ps                              # ดู running containers
docker ps -a                           # ดูทั้งหมดรวม stopped
docker run -d -p 3000:3000 myapp       # run container
docker run -it node:22-alpine sh       # run แบบ interactive
docker stop container_name             # หยุด container
docker start container_name            # เริ่ม container
docker restart container_name          # restart
docker rm container_name               # ลบ container
docker rm -f $(docker ps -aq)          # ลบทั้งหมด

# ===== Debugging =====
docker logs container_name            # ดู logs
docker logs -f container_name         # follow logs
docker exec -it container_name sh     # เข้าไปใน container
docker inspect container_name         # ดูรายละเอียด
docker stats                          # ดู resource usage
docker top container_name             # ดู processes

# ===== Docker Compose =====
docker compose up -d                   # start all services
docker compose up -d --build           # build และ start
docker compose down                    # stop และ remove containers
docker compose down -v                 # ลบ volumes ด้วย
docker compose logs -f                 # follow all logs
docker compose ps                      # ดู status
docker compose exec web sh             # เข้าไปใน service
docker compose restart api             # restart service
docker compose build --no-cache web    # rebuild without cache

# ===== System =====
docker system df                       # ดู disk usage
docker system prune                    # ลบ unused resources
docker system prune -af                # ลบทั้งหมด (ระวัง!)
```

### Step 60: Production Docker Compose

```yaml
# docker-compose.prod.yml
version: '3.9'

services:
  web:
    image: ghcr.io/chuaikan/web:${TAG:-latest}
    restart: always
    environment:
      NODE_ENV: production
    deploy:
      replicas: 2
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
      rollback_config:
        parallelism: 0
        order: stop-first
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "5"
```

---

## 🔧 Configuration Files

### สร้าง init-db.sql

```sql
-- scripts/init-db.sql
-- สร้าง database structure สำหรับ chuaikan.com

-- Enable extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- สร้าง read-only user สำหรับ monitoring
CREATE USER monitoring_user WITH PASSWORD 'mon_password_change_me';
GRANT CONNECT ON DATABASE chuaikan_dev TO monitoring_user;
GRANT USAGE ON SCHEMA public TO monitoring_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO monitoring_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO monitoring_user;
```

### .env.example file

```bash
# .env.example
# Copy ไฟล์นี้เป็น .env และแก้ไขค่า

# Application
NODE_ENV=development
APP_URL=http://localhost:3000
API_URL=http://localhost:4000

# Database
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DB=chuaikan_dev
POSTGRES_USER=chuaikan_user
# POSTGRES_PASSWORD= # ใช้ secrets แทน
DATABASE_URL=postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@${POSTGRES_HOST}:${POSTGRES_PORT}/${POSTGRES_DB}

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
# REDIS_PASSWORD= # ใช้ secrets แทน

# JWT
# JWT_SECRET= # ใช้ secrets แทน
JWT_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d
```

---

## 🧪 Testing

```bash
# Test 1: ตรวจสอบ Dockerfile syntax
docker build --no-cache --target builder -t test-build -f apps/web/Dockerfile . 2>&1
echo "Exit code: $?"

# Test 2: run containers และ check health
docker compose up -d
sleep 30

# ตรวจสอบ health status
docker compose ps
# Expected: all containers showing "healthy" or "running"

# Test 3: ทดสอบ network connectivity
docker compose exec web wget -qO- http://api:4000/health
# Expected: {"status":"ok","timestamp":"..."}

# Test 4: ทดสอบ database connection
docker compose exec api node -e "
const { Pool } = require('pg');
const pool = new Pool({ connectionString: process.env.DATABASE_URL });
pool.query('SELECT NOW()', (err, res) => {
    console.log(err ? 'ERROR: ' + err.message : 'OK: ' + res.rows[0].now);
    pool.end();
});
"

# Test 5: ทดสอบ Redis connection
docker compose exec redis redis-cli -a \$REDIS_PASSWORD ping
# Expected: PONG

# Test 6: ดู resource usage
docker stats --no-stream

# Test 7: ทดสอบ volume persistence
# หยุด postgres, เริ่มใหม่ และตรวจสอบว่าข้อมูลยังอยู่
docker compose stop postgres
docker compose start postgres
sleep 10
docker compose exec postgres psql -U chuaikan_user -d chuaikan_dev -c "\l"

# Test 8: ทดสอบ image size
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
# Target: web image < 200MB, api image < 150MB
```

---

## ❌ Common Errors & Solutions

### Error 1: Permission Denied

```
Error: Got permission denied while trying to connect to the Docker daemon socket
```

**Solution:**
```bash
# เพิ่ม user เข้า docker group
sudo usermod -aG docker $USER

# ต้อง logout และ login ใหม่ หรือใช้:
newgrp docker

# ตรวจสอบ group
groups $USER
# ควรเห็น: docker
```

### Error 2: Port Already in Use

```
Error response from daemon: driver failed programming external connectivity on endpoint
... Bind for 0.0.0.0:5432 failed: port is already allocated
```

**Solution:**
```bash
# หาว่า process ไหนใช้ port นั้นอยู่
sudo lsof -i :5432
# หรือ
sudo ss -tlnp | grep 5432

# หยุด process นั้น (เช่น local postgres)
sudo systemctl stop postgresql

# หรือ เปลี่ยน port ใน docker-compose.yml
ports:
  - "5433:5432"  # host:container
```

### Error 3: Out of Memory

```
Container exited with code 137
# (137 = 128 + 9, killed by OOM killer)
```

**Solution:**
```bash
# ตรวจสอบ memory usage
docker stats --no-stream

# ดู OOM events
dmesg | grep -i "oom\|killed"

# เพิ่ม memory limit ใน docker-compose.yml
deploy:
  resources:
    limits:
      memory: 1G  # เพิ่มขึ้น

# หรือตั้งค่า Node.js heap
environment:
  NODE_OPTIONS: "--max-old-space-size=512"
```

### Error 4: Container Unhealthy

```
Status: unhealthy
```

**Solution:**
```bash
# ดู health check logs
docker inspect --format='{{json .State.Health}}' container_name | jq

# ดู output ของ health check
docker inspect container_name | jq '.[0].State.Health.Log'

# ทดสอบ health check command ด้วยตัวเอง
docker exec container_name curl -f http://localhost:3000/api/health

# ปัญหาที่พบบ่อย: curl ไม่มีใน alpine
# Solution: ใช้ wget แทน
HEALTHCHECK CMD wget --spider -q http://localhost:3000/api/health
```

### Error 5: Build Cache Issues

```
# Build ช้ามากเพราะ cache ไม่ทำงาน
```

**Solution:**
```bash
# ตรวจสอบว่า BuildKit เปิดอยู่
echo $DOCKER_BUILDKIT
# ควรได้ 1

# ลองใช้ --progress=plain เพื่อดูรายละเอียด
docker build --progress=plain -t test .

# Clear build cache
docker builder prune

# ตรวจสอบ .dockerignore ว่าไม่ได้ ignore files ที่ใช้ใน COPY
```

### Error 6: Database Connection Failed

```
Error: connect ECONNREFUSED 127.0.0.1:5432
```

**Solution:**
```bash
# ปัญหา: ใช้ localhost แทน service name
# ✅ ใช้ service name ใน docker-compose
DATABASE_URL=postgresql://user:pass@postgres:5432/db

# ตรวจสอบว่า postgres healthy ก่อน connect
docker compose ps postgres
# ควรเป็น: healthy

# ตรวจสอบ network
docker network inspect chuaikan_database
```

---

## ✅ Checklist

### Docker Installation
- [ ] Docker CE ติดตั้งสำเร็จ (version 27+)
- [ ] Docker Compose plugin ติดตั้งสำเร็จ (version 2+)
- [ ] BuildKit เปิดใช้งานแล้ว
- [ ] User อยู่ใน docker group (ไม่ต้องใช้ sudo)

### Dockerfile
- [ ] Next.js Dockerfile มี multi-stage build (deps → builder → runner)
- [ ] Node.js API Dockerfile มี multi-stage build
- [ ] ทั้งสองใช้ non-root user
- [ ] มี HEALTHCHECK ใน Dockerfile
- [ ] มี .dockerignore สำหรับทั้ง web และ api
- [ ] Image sizes อยู่ในเกณฑ์ที่ยอมรับได้

### Docker Compose
- [ ] postgres service มี named volume
- [ ] redis service มี named volume
- [ ] ทุก service มี health check
- [ ] Services ใช้ depends_on กับ condition: service_healthy
- [ ] Secrets ถูกสร้างและใช้งานอย่างถูกต้อง
- [ ] Networks แยกตาม tier (frontend, backend, database)

### Testing
- [ ] `docker compose up -d` สำเร็จโดยไม่มี errors
- [ ] ทุก containers แสดงสถานะ "healthy"
- [ ] Web app เข้าถึงได้ที่ http://localhost:3000
- [ ] API เข้าถึงได้ที่ http://localhost:4000/health
- [ ] Database backup และ restore ทำงานได้

### Security
- [ ] ไม่มี secrets ใน Dockerfile หรือ compose file
- [ ] secrets/ directory อยู่ใน .gitignore
- [ ] ไม่ run container ด้วย root user
- [ ] Production image ไม่มี dev tools

---

## 🔗 References

- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Reference](https://docs.docker.com/compose/compose-file/)
- [Next.js Docker Examples](https://github.com/vercel/next.js/tree/canary/examples/with-docker)
- [Node.js Docker Best Practices](https://github.com/nodejs/docker-node/blob/main/docs/BestPractices.md)
- [BuildKit Documentation](https://docs.docker.com/build/buildkit/)
- [Docker Security Best Practices](https://docs.docker.com/develop/security-best-practices/)

---
*Part 006 | Road to 1,000,000 Users/Day | chuaikan.com*
