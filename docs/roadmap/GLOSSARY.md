# Glossary — คำศัพท์ที่ใช้ในคู่มือนี้
## Road to 1,000,000 Users/Day — chuaikan.com

---

## A

**API Gateway**  
จุดเดียวที่ client ติดต่อ เพื่อ route request ไปยัง microservices ต่างๆ (เช่น Kong, AWS API Gateway, Nginx)

**Autoscaling**  
การเพิ่ม/ลด server instances อัตโนมัติตาม load (Horizontal: เพิ่ม nodes, Vertical: เพิ่ม CPU/RAM)

**APM (Application Performance Monitoring)**  
การ monitor performance ของ application ในระดับ code (เช่น New Relic, Datadog, Elastic APM)

---

## B

**Blue-Green Deployment**  
การ deploy โดยมี 2 environment (Blue=production เดิม, Green=version ใหม่) สลับ traffic เมื่อ ready

**Bottleneck**  
จุดที่ทำให้ system ช้าลง เช่น slow database query, N+1 query problem

**BFF (Backend for Frontend)**  
Pattern ที่สร้าง API layer เฉพาะสำหรับแต่ละ frontend (web, mobile, etc.)

---

## C

**CAP Theorem**  
ทฤษฎีที่บอกว่า distributed system สามารถรับประกันได้แค่ 2 ใน 3: Consistency, Availability, Partition tolerance

**CDN (Content Delivery Network)**  
เครือข่าย server ที่กระจายอยู่ทั่วโลก ให้ static assets โหลดเร็วขึ้นจาก server ที่ใกล้ user

**Circuit Breaker**  
Pattern ที่หยุด call service ที่ fail อยู่ชั่วคราว เพื่อป้องกัน cascade failure

**CQRS (Command Query Responsibility Segregation)**  
Pattern แยก read (Query) และ write (Command) ออกจากกัน ใช้คนละ model/database

**Connection Pooling**  
การ reuse database connections แทนการสร้าง connection ใหม่ทุกครั้ง (ลด overhead มาก)

---

## D

**DDoS (Distributed Denial of Service)**  
การโจมตีโดยส่ง traffic มหาศาลจากหลาย source เพื่อทำให้ service ล่ม

**Dead Letter Queue (DLQ)**  
Queue พิเศษที่เก็บ message ที่ process ไม่สำเร็จ เพื่อ debug หรือ retry ภายหลัง

**Docker**  
Platform สำหรับ containerize application ให้รันได้เหมือนกันทุก environment

**DNS (Domain Name System)**  
ระบบแปลง domain name (chuaikan.com) ไปเป็น IP address

---

## E

**Elasticsearch**  
Search engine ที่ใช้ Lucene เป็น basis รองรับ full-text search และ analytics ที่รวดเร็ว

**Event Sourcing**  
Pattern เก็บ state ของ system เป็น sequence of events แทนการเก็บ current state อย่างเดียว

**Eviction Policy**  
นโยบายที่ Redis ใช้ตัดสินใจว่าจะลบ key ไหนออกเมื่อ memory เต็ม (LRU, LFU, TTL-based)

---

## F

**Feature Flag**  
Switch ที่ใช้เปิด/ปิด feature โดยไม่ต้อง deploy code ใหม่ ใช้ทำ A/B testing, canary release

**Failover**  
การสลับไปใช้ backup system อัตโนมัติเมื่อ primary system ล้มเหลว

**Feed Algorithm**  
อัลกอริทึมที่กำหนดว่า user จะเห็น post อะไรใน feed (เช่น chronological, ranking-based)

---

## G

**GeoSpatial**  
ข้อมูลที่เกี่ยวข้องกับตำแหน่งทางภูมิศาสตร์ (lat/lng, polygon, distance calculation)

**gRPC**  
Protocol สำหรับ communication ระหว่าง microservices โดยใช้ Protocol Buffers (เร็วกว่า REST)

**Grafana**  
เครื่องมือ visualization สำหรับ metrics จาก Prometheus, InfluxDB และ data source อื่นๆ

---

## H

**HPA (Horizontal Pod Autoscaler)**  
Kubernetes resource ที่ auto scale จำนวน pods ตาม CPU/memory usage

**Health Check**  
Endpoint หรือ script ที่ตรวจสอบว่า service ทำงานปกติ (HTTP 200 = healthy)

**Helm**  
Package manager สำหรับ Kubernetes ใช้ deploy application ด้วย chart templates

---

## I

**Idempotent**  
Operation ที่ทำซ้ำกี่ครั้งก็ได้ผลลัพธ์เดิม (สำคัญมากสำหรับ retry logic)

**Index (Database)**  
โครงสร้างข้อมูลที่ทำให้ query เร็วขึ้น แลกกับ disk space และ write performance

**Ingress (Kubernetes)**  
Resource ที่จัดการ HTTP traffic เข้า cluster จาก outside world

---

## J

**JWT (JSON Web Token)**  
Token format ที่ encode user info ไว้ข้างใน ไม่ต้องเก็บ session ใน server

**Jaeger**  
Distributed tracing system สำหรับ monitor request flow ข้าม microservices

---

## K

**Kafka**  
Distributed event streaming platform รองรับ throughput สูงมาก (millions of events/sec)

**Kubernetes (K8s)**  
ระบบ orchestrate containers ระดับ production จัดการ deployment, scaling, healing อัตโนมัติ

---

## L

**Load Balancer**  
ระบบกระจาย traffic ไปยัง server หลายตัว เพื่อป้องกัน single point of failure

**Latency**  
เวลาที่ใช้ในการ respond ต่อ request (วัดเป็น ms) P50, P95, P99 คือ percentile

**LRU (Least Recently Used)**  
Algorithm ที่ลบ item ที่ไม่ได้ใช้นานที่สุดออกก่อน ใช้ใน Redis eviction และ CPU cache

---

## M

**Microservices**  
Architecture ที่แบ่ง application ออกเป็น services เล็กๆ แต่ละ service รับผิดชอบ 1 domain

**Message Queue**  
ระบบ buffer สำหรับ async communication ระหว่าง services (RabbitMQ, SQS, Kafka)

**Metrics**  
ตัวเลขที่วัด performance ของ system เช่น RPS, latency, error rate, CPU usage

---

## N

**N+1 Query Problem**  
Bug pattern ที่ query database N+1 ครั้งแทนที่จะ 1 ครั้ง (ทำให้ช้ามาก)

**Node Exporter**  
Prometheus exporter สำหรับเก็บ metrics ของ Linux system (CPU, memory, disk, network)

---

## O

**ORM (Object-Relational Mapping)**  
Library ที่แปลง database records เป็น objects ใน code (Prisma, TypeORM, Sequelize)

**OWASP Top 10**  
รายการ security vulnerabilities ที่พบบ่อยที่สุดใน web applications

---

## P

**P99 Latency**  
99th percentile latency หมายความว่า 99% ของ requests เร็วกว่าค่านี้

**PgBouncer**  
Connection pooler สำหรับ PostgreSQL ลด overhead จาก connection management

**Pod (Kubernetes)**  
หน่วยเล็กที่สุดใน Kubernetes ที่รัน container หนึ่งหรือหลาย containers

**Prometheus**  
Open-source monitoring system ที่ scrape metrics จาก services และ alert เมื่อผิดปกติ

**Pub/Sub (Publish/Subscribe)**  
Messaging pattern ที่ publisher ส่ง event ไปยัง channel และ subscriber รับ event นั้น

---

## R

**Race Condition**  
Bug ที่เกิดเมื่อ 2 operations แข่งกัน ทำให้ได้ผลลัพธ์ที่ไม่คาดหวัง

**Read Replica**  
Database copy ที่ sync จาก primary แบบ real-time ใช้รับ read queries เพื่อลด load

**Redis**  
In-memory data structure store ใช้เป็น cache, session store, pub/sub, rate limiter

**RPS (Requests Per Second)**  
จำนวน HTTP requests ที่ server รับได้ต่อวินาที

---

## S

**Service Mesh**  
Infrastructure layer ที่จัดการ communication ระหว่าง microservices (Istio, Linkerd)

**Sharding**  
การแบ่ง database ออกเป็นหลาย partitions เพื่อ scale horizontally

**SLA (Service Level Agreement)**  
ข้อตกลง uptime ที่ commit กับ user เช่น 99.9% = ล่มได้ไม่เกิน 8.7 ชม./ปี

**SLO (Service Level Objective)**  
เป้าหมาย internal ที่ทีมตั้งไว้ เช่น P99 latency < 200ms

**SLI (Service Level Indicator)**  
ตัวชี้วัดจริงที่วัดได้ เช่น actual P99 latency

**SOS Platform**  
ระบบฉุกเฉินใน chuaikan.com สำหรับแจ้งเตือนภัยพิบัติ (น้ำท่วม, ไฟไหม้)

---

## T

**Terraform**  
Infrastructure as Code tool ที่ใช้ provision cloud resources ด้วย declarative config

**TimescaleDB**  
PostgreSQL extension สำหรับ time-series data เหมาะสำหรับ sensor data, metrics

**TTL (Time To Live)**  
อายุของ cache entry หรือ DNS record หลังจากนี้จะ expire

---

## V

**Vault (HashiCorp)**  
Secrets management tool สำหรับเก็บและจัดการ API keys, passwords, certificates

**VPC (Virtual Private Cloud)**  
Network ส่วนตัวใน cloud ที่แยกจาก public internet

---

## W

**WebSocket**  
Protocol สำหรับ bidirectional real-time communication ระหว่าง browser และ server

**Worker Process**  
Process ที่ทำงาน background เช่น process queue, send email, resize image

---

## Z

**Zero Downtime Deployment**  
การ deploy version ใหม่โดยไม่ทำให้ service หยุดทำงานแม้แต่วินาทีเดียว

**Zero Trust**  
Security model ที่ไม่ไว้วางใจ traffic ใดๆ ทั้งภายในและภายนอก network ต้องยืนยัน identity ทุกครั้ง

---

*Road to 1,000,000 Users/Day | chuaikan.com*
