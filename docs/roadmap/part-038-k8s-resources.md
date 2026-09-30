# Part 038: Resource Limits & Requests

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 371-380
> **เวลาโดยประมาณ:** 2.5 ชั่วโมง
> **Prerequisites:** Part 037 (HPA), Part 033 (Workloads)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- เข้าใจ CPU และ Memory requests vs limits
- กำหนด resource sizing ที่เหมาะสมสำหรับ Next.js, Node.js API, PostgreSQL
- ใช้ LimitRange สำหรับ namespace defaults
- ใช้ ResourceQuota สำหรับ namespace limits
- เข้าใจ Pod Quality of Service (QoS) classes
- แก้ปัญหา OOMKilled

---

## 📖 ทฤษฎีและแนวคิด

### Step 371 — Requests vs Limits

```
Requests (ขั้นต่ำที่รับประกัน):
- ใช้สำหรับ scheduling: K8s จะ schedule pod ลง node ที่มี resource พอ
- Pod รับประกันได้ทรัพยากรเท่า request เสมอ
- HPA ใช้ requests เป็น denominator ในการคำนวณ %

Limits (สูงสุดที่ใช้ได้):
- CPU limit: ถ้าเกิน → throttle (ช้าลง ไม่ตาย)
- Memory limit: ถ้าเกิน → OOMKilled (ตาย! ถูก restart)
```

### Step 372 — CPU Units

```bash
# CPU units
1 CPU core = 1000m (millicores)

# ตัวอย่าง:
100m  = 0.1 CPU = 10% ของ 1 core
500m  = 0.5 CPU = 50% ของ 1 core
1000m = 1 CPU   = 1 full core
2     = 2 CPUs  = 2 full cores

# คำแนะนำ:
# Request ต่ำ ให้ scheduler มีความยืดหยุ่น
# Limit สูงกว่า Request ให้ใช้ burst ได้
```

### Step 373 — Memory Units

```bash
# Memory units
Mi = Mebibyte (1024 * 1024 bytes)
Gi = Gibibyte (1024 * 1024 * 1024 bytes)
MB = Megabyte (1000 * 1000 bytes) — ใช้น้อยกว่า

# ตัวอย่าง:
128Mi  = 128 Mebibytes
256Mi  = 256 Mebibytes
1Gi    = 1024 Mebibytes = ~1 GB
2Gi    = 2048 Mebibytes = ~2 GB

# IMPORTANT: Memory limit = hard limit
# ถ้าเกิน → OOMKilled ทันที!
```

### Step 374 — QoS Classes

```
Guaranteed (ดีที่สุด ไม่ค่อยถูก evict):
- requests == limits สำหรับทุก container
- ทั้ง CPU และ Memory
- ใช้กับ: PostgreSQL, Redis, critical services

Burstable (กลาง):
- requests < limits
- หรือมีแค่ requests หรือแค่ limits
- ใช้กับ: Next.js, API ทั่วไป

BestEffort (แย่ที่สุด ถูก evict ก่อน):
- ไม่มี requests และ limits เลย
- อย่าใช้ใน production!
```

---

## ⚙️ Environment Setup

```bash
# ดู resource usage ปัจจุบัน
kubectl top nodes
kubectl top pods -n chuaikan-production --containers

# ดู resource requests/limits ของ pods
kubectl get pods -n chuaikan-production \
  -o custom-columns=\
"NAME:.metadata.name,\
CPU_REQ:.spec.containers[0].resources.requests.cpu,\
MEM_REQ:.spec.containers[0].resources.requests.memory,\
CPU_LIM:.spec.containers[0].resources.limits.cpu,\
MEM_LIM:.spec.containers[0].resources.limits.memory"
```

---

## 🛠️ Step-by-Step Implementation

### Step 375 — Resource Sizing สำหรับ chuaikan.com Services

```yaml
# deployment-nextjs-resources.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextjs-app
  namespace: chuaikan-production
spec:
  replicas: 6
  selector:
    matchLabels:
      app: nextjs
  template:
    metadata:
      labels:
        app: nextjs
    spec:
      containers:
      - name: nextjs
        image: registry.chuaikan.com/nextjs-app:v1.0.0
        resources:
          # Next.js: SSR app ใช้ CPU moderate, Memory ปานกลาง
          requests:
            cpu: "100m"       # guarantee 0.1 CPU
            memory: "256Mi"   # guarantee 256MB RAM
          limits:
            cpu: "500m"       # max 0.5 CPU
            memory: "512Mi"   # max 512MB RAM (OOMKill ถ้าเกิน)
```

```yaml
# deployment-api-resources.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-api
  namespace: chuaikan-production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nodejs-api
  template:
    metadata:
      labels:
        app: nodejs-api
    spec:
      containers:
      - name: api
        image: registry.chuaikan.com/nodejs-api:v1.0.0
        resources:
          # Node.js API: CPU spike เวลา process, Memory ปานกลาง
          requests:
            cpu: "200m"       # guarantee 0.2 CPU
            memory: "256Mi"   # guarantee 256MB
          limits:
            cpu: "1000m"      # max 1 CPU (spike ได้)
            memory: "512Mi"   # max 512MB
```

```yaml
# statefulset-postgres-resources.yaml
# (snippet - เฉพาะ resources section)
resources:
  # PostgreSQL: ต้องการ Memory สูง, CPU ปานกลาง-สูง
  requests:
    cpu: "500m"       # guarantee 0.5 CPU
    memory: "1Gi"     # guarantee 1GB RAM
  limits:
    cpu: "2000m"      # max 2 CPU
    memory: "4Gi"     # max 4GB RAM
    # Note: PostgreSQL ควรเป็น Burstable หรือ Guaranteed
```

```yaml
# statefulset-redis-resources.yaml
# (snippet - เฉพาะ resources section)
resources:
  # Redis: ใช้ Memory สูง, CPU ต่ำ
  requests:
    cpu: "100m"       # guarantee 0.1 CPU
    memory: "512Mi"   # guarantee 512MB
  limits:
    cpu: "500m"       # max 0.5 CPU
    memory: "1Gi"     # max 1GB (ต้องตรง maxmemory ใน redis.conf)
```

### Step 376 — LimitRange สำหรับ Namespace Defaults

LimitRange ช่วยกำหนด default resources สำหรับ pods ที่ไม่ได้ตั้งค่า:

```yaml
# limitrange-production.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: chuaikan-limitrange
  namespace: chuaikan-production
spec:
  limits:
  # Default สำหรับ Container
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "256Mi"
    max:
      cpu: "4000m"     # max CPU per container
      memory: "8Gi"    # max Memory per container
    min:
      cpu: "50m"       # min CPU per container
      memory: "64Mi"   # min Memory per container

  # Default สำหรับ Pod (sum ของทุก containers)
  - type: Pod
    max:
      cpu: "8000m"
      memory: "16Gi"

  # Default สำหรับ PVC
  - type: PersistentVolumeClaim
    max:
      storage: "200Gi"
    min:
      storage: "1Gi"
```

```bash
kubectl apply -f limitrange-production.yaml

# ตรวจสอบ
kubectl describe limitrange chuaikan-limitrange -n chuaikan-production

# ทดสอบ: สร้าง pod โดยไม่ระบุ resources
# K8s จะใช้ default จาก LimitRange
kubectl run test-default \
  --image=nginx:alpine \
  --namespace=chuaikan-production

kubectl describe pod test-default -n chuaikan-production | grep -A 10 "Resources:"
# Resources:
#   Limits:
#     cpu:     500m
#     memory:  512Mi
#   Requests:
#     cpu:     100m
#     memory:  256Mi

kubectl delete pod test-default -n chuaikan-production
```

### Step 377 — ResourceQuota สำหรับ Namespace Limits

```yaml
# resourcequota-production.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: chuaikan-quota
  namespace: chuaikan-production
spec:
  hard:
    # Compute resources
    requests.cpu: "20"         # total CPU requests ใน namespace
    requests.memory: "40Gi"   # total Memory requests ใน namespace
    limits.cpu: "40"           # total CPU limits
    limits.memory: "80Gi"     # total Memory limits

    # Object counts
    pods: "100"                # max pods
    services: "30"             # max services
    persistentvolumeclaims: "20"
    secrets: "50"
    configmaps: "50"

    # Services ประเภท LoadBalancer
    services.loadbalancers: "3"
    services.nodeports: "0"    # ไม่อนุญาต NodePort ใน production
```

```yaml
# resourcequota-dev.yaml (สำหรับ dev namespace - จำกัดน้อยกว่า)
apiVersion: v1
kind: ResourceQuota
metadata:
  name: chuaikan-dev-quota
  namespace: chuaikan-dev
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "20"
    services: "10"
    services.loadbalancers: "0"   # ไม่มี LoadBalancer ใน dev
    services.nodeports: "5"       # ใช้ NodePort แทน
```

```bash
kubectl apply -f resourcequota-production.yaml
kubectl apply -f resourcequota-dev.yaml

# ดู quota usage
kubectl get resourcequota -n chuaikan-production
kubectl describe resourcequota chuaikan-quota -n chuaikan-production
# Name:                    chuaikan-quota
# Namespace:               chuaikan-production
# Resource                 Used    Hard
# --------                 ---     ---
# limits.cpu               3       40
# limits.memory            6Gi     80Gi
# pods                     12      100
# requests.cpu             1500m   20
# requests.memory          3Gi     40Gi
```

### Step 378 — QoS Class ที่เหมาะสม

```yaml
# Guaranteed QoS สำหรับ PostgreSQL (requests = limits)
resources:
  requests:
    cpu: "1000m"
    memory: "2Gi"
  limits:
    cpu: "1000m"    # เท่ากับ requests
    memory: "2Gi"   # เท่ากับ requests
# QoS: Guaranteed → จะไม่ถูก evict ก่อน

# Burstable QoS สำหรับ Next.js (requests < limits)
resources:
  requests:
    cpu: "100m"
    memory: "256Mi"
  limits:
    cpu: "500m"     # มากกว่า requests
    memory: "512Mi" # มากกว่า requests
# QoS: Burstable → ถูก evict หลัง BestEffort

# ดู QoS ของ pod
kubectl get pod postgres-0 -n chuaikan-production \
  -o jsonpath='{.status.qosClass}'
# Guaranteed
```

### Step 379 — OOMKilled Troubleshooting

```bash
# ตรวจสอบว่า pod ถูก OOMKilled
kubectl get pods -n chuaikan-production
# NAME              READY   STATUS      RESTARTS   AGE
# nextjs-app-xxx    0/1     OOMKilled   3          5m

# ดู event
kubectl describe pod nextjs-app-xxx -n chuaikan-production | grep -A 10 "Last State:"
# Last State: Terminated
#   Reason: OOMKilled
#   Exit Code: 137
#   Started: Mon, 15 Jan 2024 10:00:00 +0700
#   Finished: Mon, 15 Jan 2024 10:01:30 +0700

# ดู memory usage ก่อน crash
kubectl logs nextjs-app-xxx --previous -n chuaikan-production | tail -20

# ดู memory usage ปัจจุบัน
kubectl top pod -n chuaikan-production | grep nextjs

# แก้ไข: เพิ่ม memory limit
kubectl set resources deployment/nextjs-app \
  -c=nextjs \
  --limits=memory=1Gi \
  --requests=memory=512Mi \
  -n chuaikan-production

# ตรวจสอบหลังแก้
kubectl get pods -n chuaikan-production -l app=nextjs -w
```

### Step 380 — Resource Monitoring Dashboard

```bash
# ดู resource usage แบบ real-time
watch -n 5 kubectl top pods -n chuaikan-production

# script เพื่อดู resource summary
cat > check-resources.sh <<'EOF'
#!/bin/bash
NAMESPACE=${1:-chuaikan-production}
echo "=== Resource Usage in $NAMESPACE ==="
echo ""
echo "--- Nodes ---"
kubectl top nodes
echo ""
echo "--- Pods ---"
kubectl top pods -n $NAMESPACE --sort-by=memory
echo ""
echo "--- Quota ---"
kubectl describe resourcequota -n $NAMESPACE
echo ""
echo "--- OOMKilled Pods ---"
kubectl get events -n $NAMESPACE \
  --field-selector reason=OOMKilling \
  --sort-by='.lastTimestamp' | tail -10
EOF

chmod +x check-resources.sh
./check-resources.sh

# ตั้งค่า Prometheus alert สำหรับ high memory
# (จะทำในรายละเอียดใน Part 040)
cat > alert-memory.yaml <<EOF
groups:
- name: memory-alerts
  rules:
  - alert: PodMemoryUsageHigh
    expr: |
      (container_memory_working_set_bytes{namespace="chuaikan-production"}
       / on(pod, namespace) kube_pod_container_resource_limits{resource="memory",namespace="chuaikan-production"}
      ) > 0.85
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Pod {{ \$labels.pod }} memory usage > 85%"
      description: "Pod {{ \$labels.pod }} is using {{ \$value | humanizePercentage }} of its memory limit"
EOF
```

---

## 🔧 Configuration Files

### Resource Sizing Reference Table

```bash
cat > RESOURCE-SIZING.md <<'EOF'
# Resource Sizing Guide สำหรับ chuaikan.com

## Small (dev, single user)
| Service    | CPU Req | CPU Lim | Mem Req | Mem Lim |
|------------|---------|---------|---------|---------|
| Next.js    | 50m     | 200m    | 128Mi   | 256Mi   |
| Node.js API| 100m    | 500m    | 128Mi   | 256Mi   |
| PostgreSQL | 100m    | 500m    | 256Mi   | 512Mi   |
| Redis      | 50m     | 200m    | 128Mi   | 256Mi   |

## Production (1M users/day)
| Service    | CPU Req | CPU Lim | Mem Req | Mem Lim |
|------------|---------|---------|---------|---------|
| Next.js    | 100m    | 500m    | 256Mi   | 512Mi   |
| Node.js API| 200m    | 1000m   | 256Mi   | 512Mi   |
| PostgreSQL | 500m    | 2000m   | 1Gi     | 4Gi     |
| Redis      | 100m    | 500m    | 512Mi   | 1Gi     |
EOF
```

---

## 🧪 Testing

### ทดสอบ Resource Limits ทำงาน

```bash
# ทดสอบ CPU throttling (pod จะทำงานช้าลงถ้าเกิน limit ไม่ตาย)
kubectl run cpu-test \
  --image=ubuntu:22.04 \
  --namespace=chuaikan-production \
  --restart=Never \
  --limits=cpu=100m \
  -- bash -c "while true; do echo 'burning CPU'; done"

# ดู CPU ที่ใช้จริง (จะเห็น throttling)
kubectl top pod cpu-test -n chuaikan-production
kubectl delete pod cpu-test -n chuaikan-production

# ทดสอบ Memory OOMKill
kubectl run oom-test \
  --image=ubuntu:22.04 \
  --namespace=chuaikan-production \
  --restart=Never \
  --limits=memory=64Mi \
  -- bash -c "cat <(yes | head -c 100000000) > /dev/null"

kubectl get pod oom-test -n chuaikan-production
# STATUS: OOMKilled
kubectl delete pod oom-test -n chuaikan-production
```

---

## ❌ Common Errors & Solutions

### Error: `Insufficient cpu`
```bash
kubectl describe pod <pod-name> -n chuaikan-production | grep -A 5 "Events:"
# Events: FailedScheduling: 0/3 nodes are available: 3 Insufficient cpu

# ดู node capacity
kubectl describe nodes | grep -A 5 "Allocated resources:"

# แก้: ลด CPU request หรือเพิ่ม nodes
```

### Error: `exceeded quota: pods: must be less than or equal to 100`
```bash
# สาเหตุ: ถึง quota limit
kubectl get resourcequota -n chuaikan-production
# แก้: เพิ่ม quota หรือลด pods
```

### Error: Container keeps getting `OOMKilled`
```bash
# วิเคราะห์ memory trend
kubectl top pods -n chuaikan-production | grep nextjs

# ดู VPA recommendations ถ้ามี
kubectl describe vpa nextjs-vpa -n chuaikan-production | grep -A 10 "Target:"

# เพิ่ม memory limit ชั่วคราว
kubectl set resources deployment/nextjs-app \
  --limits=memory=1Gi \
  -n chuaikan-production

# แล้วหา memory leak ในโค้ด
kubectl exec -it nextjs-app-xxx -n chuaikan-production -- \
  node --expose-gc -e "global.gc(); console.log(process.memoryUsage())"
```

---

## ✅ Checklist

- [ ] **Step 371**: อธิบายความแตกต่าง requests vs limits ได้ชัดเจน
- [ ] **Step 372**: เข้าใจ CPU units (millicores) และแปลงได้
- [ ] **Step 373**: เข้าใจ Memory units (Mi, Gi) และรู้ว่า limit = hard limit
- [ ] **Step 374**: อธิบาย QoS classes ได้: Guaranteed, Burstable, BestEffort
- [ ] **Step 375**: ตั้งค่า resources ที่เหมาะสมสำหรับทุก services ของ chuaikan.com
- [ ] **Step 376**: สร้าง LimitRange ใน production namespace สำเร็จ
- [ ] **Step 377**: สร้าง ResourceQuota ใน production และ dev namespace สำเร็จ
- [ ] **Step 378**: ตรวจสอบ QoS class ของ pods ได้
- [ ] **Step 379**: debug และแก้ปัญหา OOMKilled ได้
- [ ] **Step 380**: สร้าง resource monitoring script และ alert rules ได้

---

## 🔗 References

- [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [LimitRange](https://kubernetes.io/docs/concepts/policy/limit-range/)
- [ResourceQuota](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Pod QoS](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/)

---

*Part 038 | Road to 1,000,000 Users/Day | chuaikan.com*
