# Road to 1,000,000 Users/Day
## คู่มือฉบับสมบูรณ์สำหรับ chuaikan.com

> **Stack:** Next.js 15 | Node.js 22 | PostgreSQL 17 | Redis 7  
> **Target:** 1,000,000 users/day  
> **Type:** Social Media + SOS + Real-time Platform  
> **อัปเดตล่าสุด:** 2026-09-30

---

## 📖 สารบัญ

### 🔴 BEGINNER LEVEL (Part 001-020)

| Part | ชื่อ | Steps | เวลา |
|------|------|-------|------|
| [001](./part-001-linux-basics.md) | พื้นฐาน Linux & Server | 1-10 | 4 ชม. |
| [002](./part-002-git-workflow.md) | Git & Version Control สำหรับ Team | 11-20 | 3 ชม. |
| [003](./part-003-nodejs-nextjs.md) | Node.js & Next.js Performance Basics | 21-30 | 5 ชม. |
| [004](./part-004-postgresql.md) | PostgreSQL พื้นฐานสู่ขั้นสูง | 31-40 | 6 ชม. |
| [005](./part-005-redis.md) | Redis — Cache Layer | 41-50 | 4 ชม. |
| [006](./part-006-docker.md) | Docker — Containerization | 51-60 | 5 ชม. |
| [007](./part-007-cloudflare.md) | Cloudflare — CDN & DDoS Protection | 61-70 | 4 ชม. |
| [008](./part-008-cicd.md) | CI/CD ด้วย GitHub Actions | 71-80 | 5 ชม. |
| [009](./part-009-monitoring.md) | Monitoring พื้นฐาน | 81-90 | 4 ชม. |
| [010](./part-010-security.md) | Security พื้นฐาน | 91-100 | 5 ชม. |
| [011](./part-011-load-testing.md) | Load Testing ด้วย k6 / Artillery | 101-110 | 4 ชม. |
| [012](./part-012-websocket.md) | WebSocket & Real-time Architecture | 111-120 | 5 ชม. |
| [013](./part-013-file-upload.md) | File Upload & Media Processing | 121-130 | 4 ชม. |
| [014](./part-014-notifications.md) | Email & SMS Notification System | 131-140 | 4 ชม. |
| [015](./part-015-search.md) | Search ด้วย Elasticsearch / MeiliSearch | 141-150 | 5 ชม. |
| [016](./part-016-api-design.md) | API Design & REST Best Practices | 151-160 | 4 ชม. |
| [017](./part-017-auth.md) | Authentication & Authorization (OAuth, RBAC) | 161-170 | 5 ชม. |
| [018](./part-018-logging.md) | Logging ด้วย ELK Stack | 171-180 | 4 ชม. |
| [019](./part-019-migration.md) | Database Migration & Schema Management | 181-190 | 3 ชม. |
| [020](./part-020-testing.md) | Testing (Unit, Integration, E2E) | 191-200 | 6 ชม. |

### 🟡 INTERMEDIATE LEVEL (Part 021-050)

| Part | ชื่อ | Steps | เวลา |
|------|------|-------|------|
| [021](./part-021-microservices-overview.md) | Microservices Architecture Overview | 201-210 | 5 ชม. |
| [022](./part-022-api-gateway.md) | Service Discovery & API Gateway | 211-220 | 5 ชม. |
| [023](./part-023-auth-service.md) | Auth Service (แยก service) | 221-230 | 5 ชม. |
| [024](./part-024-notification-service.md) | Notification Service (Push/SMS/Email) | 231-240 | 5 ชม. |
| [025](./part-025-media-service.md) | Media Service (Upload/Resize/CDN) | 241-250 | 5 ชม. |
| [026](./part-026-feed-service.md) | Feed Service (Algorithm) | 251-260 | 6 ชม. |
| [027](./part-027-sos-service.md) | SOS Service (Real-time Emergency) | 261-270 | 5 ชม. |
| [028](./part-028-location-service.md) | Location Service (GeoSpatial) | 271-280 | 5 ชม. |
| [029](./part-029-search-service.md) | Search Service | 281-290 | 4 ชม. |
| [030](./part-030-analytics-service.md) | Analytics Service | 291-300 | 5 ชม. |
| [031](./part-031-k8s-concepts.md) | Kubernetes Architecture & Concepts | 301-310 | 5 ชม. |
| [032](./part-032-k8s-install.md) | การติดตั้ง K8s ด้วย kubeadm | 311-320 | 5 ชม. |
| [033](./part-033-k8s-workloads.md) | Pods, Deployments, Services | 321-330 | 5 ชม. |
| [034](./part-034-k8s-config.md) | ConfigMaps & Secrets | 331-340 | 4 ชม. |
| [035](./part-035-k8s-storage.md) | Persistent Volumes | 341-350 | 4 ชม. |
| [036](./part-036-k8s-ingress.md) | Ingress Controller (Nginx) | 351-360 | 4 ชม. |
| [037](./part-037-k8s-hpa.md) | Horizontal Pod Autoscaler (HPA) | 361-370 | 4 ชม. |
| [038](./part-038-k8s-resources.md) | Resource Limits & Requests | 371-380 | 3 ชม. |
| [039](./part-039-k8s-updates.md) | Rolling Updates & Rollbacks | 381-390 | 3 ชม. |
| [040](./part-040-k8s-monitoring.md) | K8s Monitoring ด้วย Prometheus Stack | 391-400 | 5 ชม. |
| [041](./part-041-db-sharding.md) | Database Sharding Strategy | 401-410 | 6 ชม. |
| [042](./part-042-read-replica.md) | Read Replica Configuration | 411-420 | 4 ชม. |
| [043](./part-043-timescaledb.md) | TimescaleDB สำหรับ flood sensor data | 421-430 | 4 ชม. |
| [044](./part-044-postgis.md) | PostGIS สำหรับ geospatial data | 431-440 | 5 ชม. |
| [045](./part-045-query-optimization.md) | Database Query Optimization Advanced | 441-450 | 5 ชม. |
| [046](./part-046-caching-advanced.md) | Caching Strategies Advanced | 451-460 | 5 ชม. |
| [047](./part-047-event-sourcing.md) | Event Sourcing & CQRS | 461-470 | 6 ชม. |
| [048](./part-048-db-per-service.md) | Database per Service Pattern | 471-480 | 4 ชม. |
| [049](./part-049-data-migration.md) | Data Migration at Scale | 481-490 | 5 ชม. |
| [050](./part-050-backup-dr.md) | Backup & Disaster Recovery | 491-500 | 5 ชม. |

### 🟠 ADVANCED LEVEL (Part 051-075)

| Part | ชื่อ | Steps | เวลา |
|------|------|-------|------|
| [051](./part-051-aws-gcp-overview.md) | AWS/GCP Architecture Overview | 501-510 | 5 ชม. |
| [052](./part-052-vpc-network.md) | VPC & Network Design | 511-520 | 5 ชม. |
| [053](./part-053-managed-k8s.md) | EKS / GKE (Managed Kubernetes) | 521-530 | 5 ชม. |
| [054](./part-054-managed-db.md) | RDS / Cloud SQL (Managed Database) | 531-540 | 4 ชม. |
| [055](./part-055-managed-redis.md) | ElastiCache / Memorystore | 541-550 | 4 ชม. |
| [056](./part-056-object-storage.md) | S3 / GCS (Object Storage) | 551-560 | 4 ชม. |
| [057](./part-057-cloud-cdn.md) | CloudFront / Cloud CDN | 561-570 | 4 ชม. |
| [058](./part-058-serverless.md) | Lambda / Cloud Functions (Serverless) | 571-580 | 5 ชม. |
| [059](./part-059-message-queue.md) | SQS / Pub/Sub (Message Queue) | 581-590 | 5 ชม. |
| [060](./part-060-terraform.md) | Infrastructure as Code ด้วย Terraform | 591-600 | 6 ชม. |
| [061](./part-061-nodejs-profiling.md) | Profiling Node.js Applications | 601-610 | 5 ชม. |
| [062](./part-062-memory-leak.md) | Memory Leak Detection & Fix | 611-620 | 5 ชม. |
| [063](./part-063-cpu-optimization.md) | CPU Optimization | 621-630 | 4 ชม. |
| [064](./part-064-db-profiling.md) | Database Query Profiling | 631-640 | 5 ชม. |
| [065](./part-065-network-opt.md) | Network Optimization | 641-650 | 4 ชม. |
| [066](./part-066-frontend-perf.md) | Frontend Performance (Core Web Vitals) | 651-660 | 5 ชม. |
| [067](./part-067-load-testing-scale.md) | Load Testing at Scale (100k concurrent) | 661-670 | 5 ชม. |
| [068](./part-068-chaos-engineering.md) | Chaos Engineering | 671-680 | 5 ชม. |
| [069](./part-069-performance-budget.md) | Performance Budget | 681-690 | 3 ชม. |
| [070](./part-070-sla-slo-sli.md) | SLA/SLO/SLI Definition | 691-700 | 4 ชม. |
| [071](./part-071-websocket-scaling.md) | WebSocket Scaling ด้วย Redis Pub/Sub | 701-710 | 5 ชม. |
| [072](./part-072-kafka.md) | Apache Kafka สำหรับ Event Streaming | 711-720 | 6 ชม. |
| [073](./part-073-feed-algorithm.md) | Real-time Feed Algorithm | 721-730 | 6 ชม. |
| [074](./part-074-live-streaming.md) | Live Streaming Architecture | 731-740 | 5 ชม. |
| [075](./part-075-push-notifications.md) | Push Notification at Scale (10M devices) | 741-750 | 5 ชม. |

### 🔴 PROFESSIONAL LEVEL (Part 076-090)

| Part | ชื่อ | Steps | เวลา |
|------|------|-------|------|
| [076](./part-076-multi-region.md) | Multi-region Deployment | 751-760 | 6 ชม. |
| [077](./part-077-global-lb.md) | Global Load Balancing | 761-770 | 5 ชม. |
| [078](./part-078-data-replication.md) | Data Replication across regions | 771-780 | 5 ชม. |
| [079](./part-079-latency-opt.md) | Latency Optimization (< 100ms globally) | 781-790 | 5 ชม. |
| [080](./part-080-blue-green.md) | Blue-Green Deployment | 791-800 | 4 ชม. |
| [081](./part-081-canary.md) | Canary Releases | 801-810 | 4 ชม. |
| [082](./part-082-feature-flags.md) | Feature Flags | 811-820 | 4 ชม. |
| [083](./part-083-ab-testing.md) | A/B Testing Infrastructure | 821-830 | 5 ชม. |
| [084](./part-084-service-mesh.md) | Service Mesh ด้วย Istio | 831-840 | 6 ชม. |
| [085](./part-085-zero-downtime.md) | Zero-downtime Migration | 841-850 | 5 ชม. |
| [086](./part-086-zero-trust.md) | Zero Trust Security Model | 851-860 | 5 ชม. |
| [087](./part-087-vault.md) | Secrets Management ด้วย Vault | 861-870 | 5 ชม. |
| [088](./part-088-pentest.md) | Penetration Testing Guide | 871-880 | 5 ชม. |
| [089](./part-089-pdpa.md) | Compliance (PDPA สำหรับไทย) | 881-890 | 4 ชม. |
| [090](./part-090-incident-response.md) | Security Incident Response | 891-900 | 4 ชม. |

### 🌍 WORLD CLASS LEVEL (Part 091-105)

| Part | ชื่อ | Steps | เวลา |
|------|------|-------|------|
| [091](./part-091-100m-architecture.md) | Architecture ที่รองรับ 100M users | 901-910 | 8 ชม. |
| [092](./part-092-distributed-systems.md) | Distributed Systems Fundamentals (CAP) | 911-920 | 6 ชม. |
| [093](./part-093-consensus.md) | Consensus Algorithms (Raft/Paxos) | 921-930 | 6 ชม. |
| [094](./part-094-distributed-tracing.md) | Distributed Tracing ด้วย Jaeger | 931-940 | 5 ชม. |
| [095](./part-095-cost-optimization.md) | Cost Optimization at Scale | 941-950 | 5 ชม. |
| [096](./part-096-finops.md) | FinOps — Cloud Cost Management | 951-960 | 5 ชม. |
| [097](./part-097-platform-engineering.md) | Platform Engineering (IDP) | 961-970 | 6 ชม. |
| [098](./part-098-sre.md) | SRE (Site Reliability Engineering) | 971-980 | 6 ชม. |
| [099](./part-099-disaster-recovery.md) | Disaster Recovery & Business Continuity | 981-990 | 5 ชม. |
| [100](./part-100-case-study.md) | Case Study — chuaikan.com จาก 0 ถึง 1M | 991-1000 | 8 ชม. |
| [101](./part-101-ai-ml.md) | AI/ML Integration สำหรับ Feed Algorithm | 1001-1010 | 6 ชม. |
| [102](./part-102-computer-vision.md) | Computer Vision สำหรับ Flood Detection | 1011-1020 | 6 ชม. |
| [103](./part-103-nlp-thai.md) | NLP สำหรับ Thai Language Processing | 1021-1030 | 6 ชม. |
| [104](./part-104-realtime-analytics.md) | Real-time Analytics Dashboard | 1031-1040 | 5 ชม. |
| [105](./part-105-advanced-topics.md) | Advanced Topics & Future Roadmap | 1041-1050 | 4 ชม. |

---

## 🚀 เริ่มต้นใช้คู่มือนี้

1. อ่าน [QUICK-START.md](./QUICK-START.md) ก่อน
2. ทำความเข้าใจ [GLOSSARY.md](./GLOSSARY.md) สำหรับคำศัพท์
3. เริ่มจาก Part ที่ตรงกับ level ของคุณ
4. ทำ Checklist ท้ายทุก Part ก่อนไป Part ถัดไป

---

## 📊 ภาพรวม Curriculum

```
Beginner    ████████████████████ 200 steps (Part 001-020)
Intermediate ██████████████████████████████ 300 steps (Part 021-050)
Advanced    ████████████████████████ 250 steps (Part 051-075)
Professional ████████████████ 150 steps (Part 076-090)
World Class ██████████████████████ 150+ steps (Part 091-105)
                                      Total: 1050+ steps
```

---

*Road to 1,000,000 Users/Day | chuaikan.com*
