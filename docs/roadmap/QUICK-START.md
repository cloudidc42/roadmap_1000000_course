# Quick Start Guide
## Road to 1,000,000 Users/Day — chuaikan.com

---

## 🎯 คู่มือนี้สำหรับใคร

คู่มือนี้เหมาะสำหรับทีม chuaikan.com ที่ต้องการ:
- Scale จาก 0 ไปถึง 1,000,000 users/day
- เรียนรู้ infrastructure ระดับ production จริง
- มี runbook และ checklist ที่ใช้งานได้จริง

---

## 🛠️ สิ่งที่ต้องเตรียมก่อนเริ่ม

### Hardware/VPS Requirements

```
Development:
  - CPU: 2 cores
  - RAM: 4 GB
  - Disk: 50 GB SSD

Staging:
  - CPU: 4 cores
  - RAM: 8 GB
  - Disk: 100 GB SSD

Production (เริ่มต้น):
  - CPU: 8 cores
  - RAM: 16 GB
  - Disk: 500 GB NVMe SSD
  - Network: 1 Gbps
```

### Software Requirements

```bash
# Ubuntu 24.04 LTS (OS หลักที่ใช้ในคู่มือนี้ทุก Part)
lsb_release -a
# Distributor ID: Ubuntu
# Release: 24.04
# Codename: noble

# Node.js 22 LTS
node --version   # v22.13.0 หรือใหม่กว่า
npm --version    # 10.x.x

# PostgreSQL 17
psql --version   # psql (PostgreSQL) 17.x

# Redis 7
redis-cli --version  # Redis server v=7.x.x

# Docker 27
docker --version  # Docker version 27.x.x

# Git 2.x
git --version  # git version 2.x.x
```

---

## 🗺️ Learning Path

### สำหรับ Junior Developer (0-1 ปี)
```
เริ่มจาก Part 001 → Part 020 เรียงตามลำดับ
ใช้เวลา: 3-4 เดือน
```

### สำหรับ Mid-level Developer (1-3 ปี)
```
เริ่มจาก Part 003 → Part 050
ใช้เวลา: 2-3 เดือน
```

### สำหรับ Senior Developer (3+ ปี)
```
เริ่มจาก Part 021 → Part 100
ใช้เวลา: 2-4 เดือน
```

### สำหรับ DevOps/SRE
```
Part 006, 008, 009, 031-040, 051-060, 076-090
ใช้เวลา: 2-3 เดือน
```

---

## 📋 Setup Environment

### 1. Clone Repository chuaikan.com

```bash
# สมมติว่า repo อยู่ที่ GitHub
git clone https://github.com/chuaikan/chuaikan-app.git
cd chuaikan-app

# ตรวจสอบ branch
git branch -a
```

### 2. ติดตั้ง Dependencies

```bash
# Node.js dependencies
npm install

# Copy environment variables
cp .env.example .env.local

# แก้ไขค่าใน .env.local
nano .env.local
```

### 3. Start Development Environment

```bash
# ใช้ Docker Compose สำหรับ local development
docker compose -f docker-compose.dev.yml up -d

# ตรวจสอบ services
docker compose ps

# Run migrations
npm run db:migrate

# Start development server
npm run dev
```

---

## 🔄 การใช้คู่มือแต่ละ Part

แต่ละ Part มีโครงสร้าง:

1. **ทฤษฎีและแนวคิด** — เข้าใจก่อนลงมือ
2. **Environment Setup** — เตรียมสภาพแวดล้อม
3. **Step-by-Step Implementation** — ทำตามทีละ step
4. **Configuration Files** — ไฟล์ config ที่ใช้งานจริง
5. **Testing** — ทดสอบว่าใช้งานได้
6. **Common Errors** — แก้ปัญหาที่พบบ่อย
7. **Checklist** — ตรวจสอบก่อนไป Part ถัดไป

---

## ⚠️ หมายเหตุสำคัญ

- ทุก command ใช้ **Ubuntu 24.04 LTS** เป็นหลัก
- Version ที่ระบุในคู่มือคือ version ที่ทดสอบแล้ว ห้ามใช้ "latest"
- ทำ Checklist ให้ครบทุก Part ก่อนไป Part ถัดไป
- เก็บ `.env` ไว้ใน secret manager อย่า commit ลง git

---

*Road to 1,000,000 Users/Day | chuaikan.com*
