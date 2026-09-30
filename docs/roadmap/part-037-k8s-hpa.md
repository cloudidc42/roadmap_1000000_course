# Part 037: Horizontal Pod Autoscaler (HPA)

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 361-370
> **เวลาโดยประมาณ:** 3 ชั่วโมง
> **Prerequisites:** Part 036 (Ingress), Part 038 (Resource Limits) — ต้องตั้งค่า resource requests ก่อน

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- สร้าง HPA สำหรับ CPU และ Memory metrics
- ใช้ Custom Metrics HPA ด้วย KEDA (RPS per pod)
- ติดตั้งและตั้งค่า KEDA สำหรับ chuaikan.com
- Scale-to-zero สำหรับ non-critical services
- ใช้ VPA (Vertical Pod Autoscaler) สำหรับหาขนาดที่เหมาะสม
- วางแผน min/max replicas สำหรับ production

---

## 📖 ทฤษฎีและแนวคิด

### Step 361 — HPA คืออะไรและทำงานอย่างไร

**Horizontal Pod Autoscaler (HPA)**:
- ปรับจำนวน Pod replicas อัตโนมัติตาม metrics
- ทำงานทุก 15 วินาที (default)
- scale up เร็ว, scale down ช้า (cooldown 5 นาที)

```
HPA Control Loop:
┌─────────────────────────────────────────────┐
│                                             │
│  Metrics API  ──► HPA Controller ──► Scale  │
│  (CPU/Memory)     (every 15s)        Pods   │
│                                             │
│  DesiredReplicas = ceil(currentReplicas *   │
│                   currentMetricValue /      │
│                   desiredMetricValue)       │
└─────────────────────────────────────────────┘
```

**ตัวอย่าง:**
```
Current: 3 pods, CPU usage: 80% (target: 50%)
DesiredReplicas = ceil(3 * 80/50) = ceil(4.8) = 5 pods
→ scale up เป็น 5 pods
```

### Step 362 — HPA vs KEDA

```
HPA (built-in K8s):
+ ไม่ต้องติดตั้งเพิ่ม
+ รองรับ CPU, Memory
- ไม่รองรับ custom metrics โดยตรง

KEDA (Kubernetes Event-Driven Autoscaling):
+ scale based on: HTTP requests, queue length, Prometheus metrics
+ scale-to-zero (0 replicas เมื่อไม่มี traffic)
+ รองรับ 50+ scalers
- ต้องติดตั้งเพิ่ม
```

---

## ⚙️ Environment Setup

```bash
# ติดตั้ง metrics-server (จำเป็นสำหรับ HPA)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# แก้ไข metrics-server ให้ใช้ insecure TLS (สำหรับ bare metal)
kubectl patch deployment metrics-server \
  -n kube-system \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

# ตรวจสอบ
kubectl get pods -n kube-system | grep metrics-server
kubectl top nodes
kubectl top pods -n chuaikan-production
```

---

## 🛠️ Step-by-Step Implementation

### Step 363 — HPA สำหรับ CPU และ Memory

```yaml
# hpa-nextjs-basic.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nextjs-hpa
  namespace: chuaikan-production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nextjs-app
  minReplicas: 3
  maxReplicas: 20
  metrics:
  # Scale based on CPU
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70    # target 70% CPU utilization

  # Scale based on Memory
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80    # target 80% Memory utilization

  behavior:
    # Scale up: เพิ่มเร็ว
    scaleUp:
      stabilizationWindowSeconds: 0     # scale up ทันที
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15               # เพิ่มได้ 100% ทุก 15 วินาที
      - type: Pods
        value: 4
        periodSeconds: 15               # หรือเพิ่ม 4 pods ทุก 15 วินาที
      selectPolicy: Max                 # เลือก policy ที่ scale ได้มากกว่า

    # Scale down: ลดช้า (ป้องกัน flapping)
    scaleDown:
      stabilizationWindowSeconds: 300   # รอ 5 นาที ก่อน scale down
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60               # ลดได้ 10% ทุก 1 นาที
```

```yaml
# hpa-api-basic.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: chuaikan-production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nodejs-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60    # API scale ที่ 60% CPU
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
    scaleDown:
      stabilizationWindowSeconds: 300
```

```bash
kubectl apply -f hpa-nextjs-basic.yaml
kubectl apply -f hpa-api-basic.yaml

# ดู HPA status
kubectl get hpa -n chuaikan-production
# NAME          REFERENCE              TARGETS         MINPODS   MAXPODS   REPLICAS
# nextjs-hpa    Deployment/nextjs-app  45%/70%         3         20        3
# api-hpa       Deployment/nodejs-api  30%/60%         2         10        2

# ดูรายละเอียด
kubectl describe hpa nextjs-hpa -n chuaikan-production
```

### Step 364 — ติดตั้ง KEDA

```bash
# เพิ่ม Helm repo
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

# ติดตั้ง KEDA
helm install keda kedacore/keda \
  --namespace keda \
  --create-namespace \
  --version 2.13.1

# ตรวจสอบ
kubectl get pods -n keda
# NAME                                      READY   STATUS
# keda-operator-xxxx                        1/1     Running
# keda-operator-metrics-apiserver-xxxx      1/1     Running
```

### Step 365 — KEDA ScaledObject สำหรับ HTTP Traffic

```yaml
# keda-http-scaler.yaml
# ต้องติดตั้ง http-add-on ก่อน
# helm install http-add-on kedacore/keda-add-ons-http --namespace keda

apiVersion: http.keda.sh/v1alpha1
kind: HTTPScaledObject
metadata:
  name: nextjs-http-scaler
  namespace: chuaikan-production
spec:
  host: www.chuaikan.com
  targetPendingRequests: 100    # scale เมื่อมี pending requests > 100 ต่อ pod
  scaledownPeriod: 300          # scale down หลัง 5 นาทีไม่มี traffic
  scaleTargetRef:
    name: nextjs-app
    apiVersion: apps/v1
    kind: Deployment
    service: nextjs-service
    port: 3000
  replicas:
    min: 3
    max: 20
```

### Step 366 — KEDA ScaledObject สำหรับ Prometheus Metrics

```yaml
# keda-prometheus-scaler.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: nextjs-prometheus-scaler
  namespace: chuaikan-production
spec:
  scaleTargetRef:
    name: nextjs-app
  minReplicaCount: 3
  maxReplicaCount: 20
  cooldownPeriod: 300         # รอ 5 นาทีก่อน scale down
  pollingInterval: 15         # check metrics ทุก 15 วินาที
  triggers:
  # Scale based on HTTP RPS per pod
  - type: prometheus
    metadata:
      serverAddress: http://prometheus-operated.monitoring.svc.cluster.local:9090
      metricName: http_requests_per_second
      query: |
        sum(rate(nginx_ingress_controller_requests{
          exported_namespace="chuaikan-production",
          ingress="chuaikan-ingress"
        }[1m])) / count(kube_pod_info{
          namespace="chuaikan-production",
          pod=~"nextjs-app-.*"
        })
      threshold: "50"          # scale เมื่อ RPS ต่อ pod > 50

  # Scale based on queue length (เช่น Redis queue)
  - type: redis
    metadata:
      address: redis-service.chuaikan-production.svc.cluster.local:6379
      listName: job-queue
      listLength: "100"        # scale เมื่อ queue มากกว่า 100 items
    authenticationRef:
      name: redis-trigger-auth
```

```yaml
# TriggerAuthentication สำหรับ Redis
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: redis-trigger-auth
  namespace: chuaikan-production
spec:
  secretTargetRef:
  - parameter: password
    name: redis-secrets
    key: REDIS_PASSWORD
```

### Step 367 — Scale-to-Zero สำหรับ Non-Critical Services

```yaml
# keda-scale-to-zero.yaml
# ใช้สำหรับ background job processors ที่ทำงานตามคิว
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: email-processor-scaler
  namespace: chuaikan-production
spec:
  scaleTargetRef:
    name: email-processor
  minReplicaCount: 0          # scale ลง 0 เมื่อไม่มีงาน!
  maxReplicaCount: 5
  cooldownPeriod: 60
  triggers:
  - type: redis
    metadata:
      address: redis-service.chuaikan-production.svc.cluster.local:6379
      listName: email-queue
      listLength: "1"          # scale up เมื่อมีแม้แต่ 1 item
    authenticationRef:
      name: redis-trigger-auth
```

```bash
# ดู ScaledObject
kubectl get scaledobject -n chuaikan-production
kubectl describe scaledobject nextjs-prometheus-scaler -n chuaikan-production

# ดู HPA ที่ KEDA สร้างให้
kubectl get hpa -n chuaikan-production
```

### Step 368 — VPA (Vertical Pod Autoscaler) สำหรับ Initial Sizing

```bash
# ติดตั้ง VPA
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler/
./hack/vpa-up.sh

# ตรวจสอบ
kubectl get pods -n kube-system | grep vpa
```

```yaml
# vpa-nextjs.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: nextjs-vpa
  namespace: chuaikan-production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nextjs-app
  updatePolicy:
    updateMode: "Off"     # Off = แนะนำเท่านั้น ไม่ต้อง apply อัตโนมัติ
  resourcePolicy:
    containerPolicies:
    - containerName: nextjs
      minAllowed:
        cpu: 50m
        memory: 128Mi
      maxAllowed:
        cpu: 2000m
        memory: 2Gi
      controlledResources: ["cpu", "memory"]
```

```bash
kubectl apply -f vpa-nextjs.yaml

# รอสักพักแล้วดู recommendations
kubectl describe vpa nextjs-vpa -n chuaikan-production
# Recommendation:
#   Container Recommendations:
#     Container Name: nextjs
#     Lower Bound:
#       Cpu: 80m
#       Memory: 180Mi
#     Target:              <-- ค่าที่แนะนำ!
#       Cpu: 150m
#       Memory: 300Mi
#     Upper Bound:
#       Cpu: 500m
#       Memory: 800Mi
```

### Step 369 — HPA Min/Max Replicas Planning

การวางแผน replicas สำหรับ chuaikan.com ที่ 1M users/day:

```bash
# คำนวณ replicas
# 1M users/day = ~11,574 req/min = ~193 req/sec
# Next.js รับได้ ~50 RPS ต่อ pod (conservative)
# Min replicas = 193 / 50 = ~4 (เพิ่ม buffer 50% = 6)
# Max replicas = peak traffic 5x = 193 * 5 / 50 = 20

# Peak hours: สมมติ traffic 10x เวลากลางวัน
# 193 RPS * 10 = 1930 RPS
# Max replicas = 1930 / 50 = 39 → เอา 40

cat <<EOF > hpa-production-plan.md
## HPA Planning for chuaikan.com

| Service    | Min | Max | CPU Target | Memory Target |
|------------|-----|-----|------------|---------------|
| Next.js    | 6   | 40  | 70%        | 80%           |
| Node.js API| 3   | 20  | 60%        | 75%           |
| Workers    | 0   | 10  | N/A        | N/A (KEDA)    |
EOF
```

```yaml
# hpa-nextjs-production.yaml (ค่า production จริง)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nextjs-hpa
  namespace: chuaikan-production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nextjs-app
  minReplicas: 6
  maxReplicas: 40
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 6
        periodSeconds: 15
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60
```

### Step 370 — ทดสอบ HPA ด้วย Load Test

```bash
# ติดตั้ง k6 สำหรับ load test
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
  --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | \
  sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update && sudo apt-get install k6 -y

# สร้าง load test script
cat > load-test-hpa.js <<'EOF'
import http from 'k6/http';
import { sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 50 },    // ramp up
    { duration: '5m', target: 200 },   // high load
    { duration: '2m', target: 0 },     // ramp down
  ],
};

export default function () {
  http.get('https://www.chuaikan.com');
  sleep(1);
}
EOF

# รัน load test
k6 run load-test-hpa.js

# ดู HPA scaling ขณะทดสอบ
watch -n 5 kubectl get hpa nextjs-hpa -n chuaikan-production

# ดู pods ที่ scale
watch -n 5 kubectl get pods -l app=nextjs -n chuaikan-production | wc -l
```

---

## 🔧 Configuration Files

### Monitor HPA Events

```bash
# ดู HPA events
kubectl describe hpa nextjs-hpa -n chuaikan-production | grep -A 20 "Events:"

# ดู events ใน namespace
kubectl get events -n chuaikan-production \
  --field-selector reason=SuccessfulRescale \
  --sort-by='.lastTimestamp'

# ดู metrics ปัจจุบัน
kubectl get hpa nextjs-hpa -n chuaikan-production \
  -o jsonpath='{.status.currentMetrics}' | jq
```

---

## ❌ Common Errors & Solutions

### Error: `HPA unable to fetch metrics`
```bash
# สาเหตุ: metrics-server ไม่ทำงาน
kubectl get pods -n kube-system | grep metrics-server
kubectl logs -n kube-system -l k8s-app=metrics-server

# ถ้า TLS error:
kubectl patch deployment metrics-server -n kube-system \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

### Error: `HPA stuck at minimum replicas`
```bash
# สาเหตุ: Pods ไม่มี resource requests ตั้งค่าไว้
kubectl describe deployment nextjs-app -n chuaikan-production | grep -A 5 "Resources:"
# ถ้าไม่มี requests → HPA ไม่รู้จะ scale ยังไง
# แก้: เพิ่ม resources.requests ในทุก container
```

### Error: `KEDA ScaledObject error`
```bash
kubectl describe scaledobject nextjs-prometheus-scaler -n chuaikan-production
# ดูที่ Conditions:

# ตรวจสอบ Prometheus query
kubectl exec -n monitoring prometheus-0 -- \
  promtool query instant http://localhost:9090 \
  'sum(rate(nginx_ingress_controller_requests[1m]))'
```

---

## ✅ Checklist

- [ ] **Step 361**: เข้าใจ HPA control loop และสูตรคำนวณ desired replicas
- [ ] **Step 362**: อธิบายความแตกต่าง HPA vs KEDA ได้
- [ ] **Step 363**: สร้าง HPA สำหรับ CPU/Memory metrics พร้อม behavior (scale up/down) สำเร็จ
- [ ] **Step 364**: ติดตั้ง KEDA ใน keda namespace สำเร็จ
- [ ] **Step 365**: สร้าง KEDA ScaledObject สำหรับ HTTP traffic สำเร็จ
- [ ] **Step 366**: สร้าง KEDA ScaledObject สำหรับ Prometheus metrics สำเร็จ
- [ ] **Step 367**: ตั้งค่า scale-to-zero สำหรับ email processor สำเร็จ
- [ ] **Step 368**: ใช้ VPA รับ resource recommendations สำเร็จ
- [ ] **Step 369**: วางแผน min/max replicas สำหรับ 1M users/day
- [ ] **Step 370**: ทดสอบ HPA ด้วย k6 load test เห็น pods scale up/down

---

## 🔗 References

- [HPA Documentation](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [KEDA Documentation](https://keda.sh/docs/latest/)
- [VPA Documentation](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
- [k6 Load Testing](https://k6.io/docs/)

---

*Part 037 | Road to 1,000,000 Users/Day | chuaikan.com*
