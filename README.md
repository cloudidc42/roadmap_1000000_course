# Road to 1,000,000 Users/Day
## คู่มือ Scale chuaikan.com ฉบับสมบูรณ์

> **Social Media + SOS Emergency Platform**  
> Stack: Next.js 15 · Node.js 22 · PostgreSQL 17 · Redis 7 · Ubuntu 24.04 LTS

---

## ภาพรวม

คู่มือนี้สอนวิธี scale **chuaikan.com** ตั้งแต่ 0 จนถึง **1,000,000 users/day** แบบ step-by-step ครบ **105 Parts**, **1,050 Steps** — ทุก code รันได้จริง ทุก config ใช้งานได้จริง

chuaikan.com เป็นแพลตฟอร์มที่รวม Social Media และระบบแจ้งเตือนภัย SOS (น้ำท่วม, ไฟไหม้) เข้าด้วยกัน ต้องการ architecture ที่รองรับทั้ง real-time communication และ high availability พร้อมกัน

---

## สิ่งที่จะได้เรียนรู้

| ระดับ | Parts | สิ่งที่ได้ |
|-------|-------|------------|
| 🔴 Beginner | 001–020 | Linux, Git, Node.js, PostgreSQL, Redis, Docker, Cloudflare, CI/CD, Monitoring, Security |
| 🟡 Intermediate | 021–050 | Microservices, Kubernetes, Database Sharding, CQRS/Event Sourcing, Disaster Recovery |
| 🟠 Advanced | 051–075 | AWS/GCP, Terraform, Performance Profiling, Chaos Engineering, Kafka, Live Streaming |
| 🔴 Professional | 076–090 | Multi-region, Service Mesh (Istio), Zero Trust, Vault, Penetration Testing, PDPA |
| 🌍 World Class | 091–105 | 100M Architecture, Distributed Systems, SRE, FinOps, AI/ML, Real-time Analytics |

---

## โครงสร้างคู่มือ

```
docs/roadmap/
├── INDEX.md              # สารบัญทั้ง 105 Parts
├── QUICK-START.md        # เริ่มต้นเร็ว + learning paths
├── GLOSSARY.md           # คำศัพท์ English/Thai
├── part-001-linux-basics.md
├── part-002-git-workflow.md
├── ...
└── part-105-advanced-topics.md
```

แต่ละ Part มีโครงสร้าง:
1. ทฤษฎีและแนวคิด (ภาษาไทย)
2. Environment Setup
3. Step-by-Step Implementation (code จริง)
4. Configuration Files (ใช้งานได้ทันที)
5. Testing & Verification
6. Common Errors & Solutions
7. Checklist ก่อนไป Part ถัดไป

---

## เริ่มต้นใช้งาน

```bash
# อ่านสารบัญทั้งหมด
cat docs/roadmap/INDEX.md

# อ่าน quick start guide
cat docs/roadmap/QUICK-START.md

# เริ่มจาก Part 001
cat docs/roadmap/part-001-linux-basics.md
```

### Learning Path ตาม Level

| Experience | เริ่มจาก | ใช้เวลา |
|-----------|---------|---------|
| Junior (0–1 ปี) | Part 001 → 020 | 3–4 เดือน |
| Mid-level (1–3 ปี) | Part 003 → 050 | 2–3 เดือน |
| Senior (3+ ปี) | Part 021 → 100 | 2–4 เดือน |
| DevOps/SRE | Part 006, 008, 009, 031–040, 051–060, 076–090 | 2–3 เดือน |

---

## Tech Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Frontend | Next.js (App Router, PPR, Server Components) | 15.x |
| Runtime | Node.js LTS | 22.x |
| Database | PostgreSQL | 17.x |
| Cache | Redis | 7.x |
| OS | Ubuntu LTS | 24.04 |
| Container | Docker | 27.x |
| Orchestration | Kubernetes (kubeadm / EKS) | 1.31+ |
| CDN/Edge | Cloudflare Workers, R2, WAF | — |
| Monitoring | Prometheus + Grafana + Alertmanager | — |
| Tracing | Jaeger + OpenTelemetry | — |
| Message Bus | Apache Kafka (KRaft mode) | 3.7+ |
| IaC | Terraform + Terragrunt | 1.9+ |

---

## Curriculum Progress

```
Beginner     ████████████████████  200 steps  (Part 001–020)
Intermediate ██████████████████████████████  300 steps  (Part 021–050)
Advanced     ████████████████████████  250 steps  (Part 051–075)
Professional ████████████████  150 steps  (Part 076–090)
World Class  ██████████████████████  150 steps  (Part 091–105)
                                    ──────────────────────────
                                    Total: 1,050 steps
```

---

## ข้อกำหนดสำคัญ

- ทุก command ทดสอบบน **Ubuntu 24.04 LTS** เท่านั้น
- ระบุ version จริงทุกที่ — ห้ามใช้ `latest`
- ทุก code รันได้จริง ไม่มี placeholder ที่ไม่อธิบาย
- คำอธิบายเป็นภาษาไทย, code/command เป็นภาษาอังกฤษ

---

## ลิงก์ด่วน

- [สารบัญทั้งหมด](./docs/roadmap/INDEX.md)
- [Quick Start Guide](./docs/roadmap/QUICK-START.md)
- [คำศัพท์ Glossary](./docs/roadmap/GLOSSARY.md)

---

*Road to 1,000,000 Users/Day | chuaikan.com*
