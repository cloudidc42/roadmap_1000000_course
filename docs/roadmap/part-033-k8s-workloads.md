# Part 033: Pods, Deployments, Services

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 321-330
> **เวลาโดยประมาณ:** 3 ชั่วโมง
> **Prerequisites:** Part 032 (K8s Installation), K8s cluster พร้อมใช้งาน

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- เขียน Pod manifest สำหรับ chuaikan.com Next.js app
- สร้าง Deployment manifest พร้อม replicas และ update strategy
- เข้าใจ Service types: ClusterIP, NodePort, LoadBalancer
- สร้าง Services สำหรับ Next.js, Node.js API, PostgreSQL, Redis
- ตั้งค่า Rolling Update
- ใช้ Init Containers
- เข้าใจ Labels และ Selectors
- ใช้ kubectl commands: get, describe, logs, exec

---

## 📖 ทฤษฎีและแนวคิด

### Step 321 — Pod Fundamentals

**Pod** คือ unit เล็กที่สุดที่ K8s จัดการ ไม่ใช่ container โดยตรง

```
Pod = wrapper ของ 1 หรือมากกว่า container ที่:
- แชร์ network namespace (same IP)
- แชร์ storage volumes
- ถูก schedule ไปอยู่บน Node เดียวกัน
```

ประเภท Pod สำหรับ chuaikan.com:
- **Next.js frontend**: stateless, หลาย replicas
- **Node.js API**: stateless, หลาย replicas
- **PostgreSQL**: stateful, ต้องมี persistent storage
- **Redis**: stateful, ต้องมี persistent storage

### Step 322 — Service Types

**ClusterIP** (default)
- IP ภายใน cluster เท่านั้น
- ใช้สำหรับ internal communication ระหว่าง services
- เช่น Next.js -> API, API -> Database

**NodePort**
- expose service ผ่าน port บน Node (30000-32767)
- ใช้สำหรับ development/testing
- Production: ไม่แนะนำ ใช้ LoadBalancer แทน

**LoadBalancer**
- ขอ external IP จาก cloud provider หรือ MetalLB
- ใช้สำหรับ expose services ออก internet

---

## ⚙️ Environment Setup

```bash
# ตั้ง working namespace
kubectl config set-context --current --namespace=chuaikan-production

# สร้าง directory สำหรับ manifests
mkdir -p ~/k8s-manifests/chuaikan/{deployments,services,configmaps}
cd ~/k8s-manifests/chuaikan
```

---

## 🛠️ Step-by-Step Implementation

### Step 323 — Pod Manifest สำหรับ Next.js App

```yaml
# pod-nextjs.yaml (ใช้เพื่อ test เท่านั้น, production ใช้ Deployment)
apiVersion: v1
kind: Pod
metadata:
  name: nextjs-test-pod
  namespace: chuaikan-production
  labels:
    app: nextjs
    version: "1.0.0"
    env: production
spec:
  containers:
  - name: nextjs
    image: registry.chuaikan.com/nextjs-app:v1.0.0
    ports:
    - containerPort: 3000
      protocol: TCP
    env:
    - name: NODE_ENV
      value: "production"
    - name: PORT
      value: "3000"
    resources:
      requests:
        cpu: "100m"
        memory: "256Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
    readinessProbe:
      httpGet:
        path: /api/health
        port: 3000
      initialDelaySeconds: 10
      periodSeconds: 5
    livenessProbe:
      httpGet:
        path: /api/health
        port: 3000
      initialDelaySeconds: 30
      periodSeconds: 10
  restartPolicy: Always
```

```bash
# Apply pod
kubectl apply -f pod-nextjs.yaml

# ดู pod status
kubectl get pod nextjs-test-pod -n chuaikan-production
kubectl describe pod nextjs-test-pod -n chuaikan-production
```

### Step 324 — Deployment Manifest สำหรับ Next.js

```yaml
# deployment-nextjs.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextjs-app
  namespace: chuaikan-production
  labels:
    app: nextjs
    component: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nextjs
      component: frontend
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: nextjs
        component: frontend
        version: "1.0.0"
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: nextjs
        image: registry.chuaikan.com/nextjs-app:v1.0.0
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: PORT
          value: "3000"
        envFrom:
        - configMapRef:
            name: nextjs-config
        - secretRef:
            name: nextjs-secrets
        resources:
          requests:
            cpu: "100m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        readinessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
          successThreshold: 1
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 5"]
```

```bash
kubectl apply -f deployment-nextjs.yaml
kubectl get deployment nextjs-app -n chuaikan-production
kubectl get pods -l app=nextjs -n chuaikan-production
```

### Step 325 — Deployment สำหรับ Node.js API

```yaml
# deployment-api.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-api
  namespace: chuaikan-production
  labels:
    app: nodejs-api
    component: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nodejs-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: nodejs-api
        component: backend
    spec:
      initContainers:
      - name: wait-for-postgres
        image: busybox:1.28
        command: ['sh', '-c',
          'until nc -z postgres-service 5432; do echo "Waiting for PostgreSQL..."; sleep 2; done']
      containers:
      - name: api
        image: registry.chuaikan.com/nodejs-api:v1.0.0
        ports:
        - containerPort: 4000
        env:
        - name: NODE_ENV
          value: "production"
        - name: PORT
          value: "4000"
        envFrom:
        - configMapRef:
            name: api-config
        - secretRef:
            name: api-secrets
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "1000m"
            memory: "512Mi"
        readinessProbe:
          httpGet:
            path: /health
            port: 4000
          initialDelaySeconds: 15
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 4000
          initialDelaySeconds: 30
          periodSeconds: 10
```

### Step 326 — Services สำหรับทุก Components

```yaml
# services.yaml
---
# Service สำหรับ Next.js (LoadBalancer - external access)
apiVersion: v1
kind: Service
metadata:
  name: nextjs-service
  namespace: chuaikan-production
  labels:
    app: nextjs
spec:
  type: LoadBalancer
  selector:
    app: nextjs
    component: frontend
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 3000
  - name: https
    protocol: TCP
    port: 443
    targetPort: 3000
---
# Service สำหรับ Node.js API (ClusterIP - internal only)
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: chuaikan-production
  labels:
    app: nodejs-api
spec:
  type: ClusterIP
  selector:
    app: nodejs-api
    component: backend
  ports:
  - name: http
    protocol: TCP
    port: 4000
    targetPort: 4000
---
# Service สำหรับ PostgreSQL (ClusterIP - internal only)
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: chuaikan-production
  labels:
    app: postgres
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
  - name: postgres
    protocol: TCP
    port: 5432
    targetPort: 5432
---
# Headless Service สำหรับ PostgreSQL StatefulSet
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: chuaikan-production
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
---
# Service สำหรับ Redis (ClusterIP - internal only)
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: chuaikan-production
  labels:
    app: redis
spec:
  type: ClusterIP
  selector:
    app: redis
  ports:
  - name: redis
    protocol: TCP
    port: 6379
    targetPort: 6379
```

```bash
kubectl apply -f services.yaml
kubectl get services -n chuaikan-production
```

### Step 327 — Init Containers

Init Containers รันก่อน main container และต้อง complete ก่อน:

```yaml
# deployment-api-with-init.yaml (snippet)
spec:
  initContainers:
  # รอ PostgreSQL พร้อม
  - name: wait-for-postgres
    image: busybox:1.28
    command:
    - sh
    - -c
    - |
      echo "Waiting for PostgreSQL..."
      until nc -z postgres-service 5432; do
        echo "PostgreSQL not ready. Sleeping..."
        sleep 2
      done
      echo "PostgreSQL is ready!"

  # Run database migrations
  - name: run-migrations
    image: registry.chuaikan.com/nodejs-api:v1.0.0
    command: ["node", "scripts/migrate.js"]
    env:
    - name: DATABASE_URL
      valueFrom:
        secretKeyRef:
          name: api-secrets
          key: DATABASE_URL

  containers:
  - name: api
    image: registry.chuaikan.com/nodejs-api:v1.0.0
    # ... rest of config
```

### Step 328 — Labels และ Selectors

Labels คือ key-value pairs ที่ attach กับ K8s objects

```bash
# ดู labels ของ pods
kubectl get pods --show-labels -n chuaikan-production

# filter pods ด้วย label selector
kubectl get pods -l app=nextjs -n chuaikan-production
kubectl get pods -l app=nextjs,component=frontend -n chuaikan-production
kubectl get pods -l 'env in (production,staging)' -n chuaikan-production

# เพิ่ม label ให้ pod ที่มีอยู่
kubectl label pod nextjs-app-xxx version=1.1.0 -n chuaikan-production

# ลบ label
kubectl label pod nextjs-app-xxx version- -n chuaikan-production
```

Label conventions สำหรับ chuaikan.com:

```yaml
labels:
  app: nextjs              # ชื่อ application
  component: frontend      # ส่วนของ architecture (frontend/backend/database)
  version: "1.0.0"        # version ของ app
  env: production          # environment
  team: chuaikan          # team owner
  managed-by: helm         # ถ้าใช้ Helm
```

### Step 329 — Rolling Update Configuration

```bash
# ดู update history
kubectl rollout history deployment/nextjs-app -n chuaikan-production

# Update image เป็น version ใหม่
kubectl set image deployment/nextjs-app \
  nextjs=registry.chuaikan.com/nextjs-app:v1.1.0 \
  -n chuaikan-production

# ดู progress ของ rollout
kubectl rollout status deployment/nextjs-app -n chuaikan-production

# rollback ถ้ามีปัญหา
kubectl rollout undo deployment/nextjs-app -n chuaikan-production

# rollback ไป version ที่ระบุ
kubectl rollout undo deployment/nextjs-app \
  --to-revision=2 \
  -n chuaikan-production

# หยุด rollout ชั่วคราว
kubectl rollout pause deployment/nextjs-app -n chuaikan-production

# resume rollout
kubectl rollout resume deployment/nextjs-app -n chuaikan-production
```

### Step 330 — kubectl Commands ที่ใช้บ่อย

```bash
# --- GET Commands ---
# ดู pods พร้อม IP และ Node
kubectl get pods -o wide -n chuaikan-production

# ดูในรูปแบบ YAML
kubectl get deployment nextjs-app -o yaml -n chuaikan-production

# ดูหลาย resource พร้อมกัน
kubectl get pods,services,deployments -n chuaikan-production

# Watch changes แบบ real-time
kubectl get pods -w -n chuaikan-production

# --- DESCRIBE Commands ---
# ดูรายละเอียด pod (events, conditions)
kubectl describe pod nextjs-app-xxx -n chuaikan-production

# ดูรายละเอียด node
kubectl describe node worker-01

# --- LOGS Commands ---
# ดู logs ล่าสุด
kubectl logs nextjs-app-xxx -n chuaikan-production

# Follow logs (เหมือน tail -f)
kubectl logs -f nextjs-app-xxx -n chuaikan-production

# ดู logs ของ container ที่ crash (previous)
kubectl logs nextjs-app-xxx --previous -n chuaikan-production

# ดู logs จากทุก pods ใน deployment
kubectl logs -l app=nextjs --all-containers=true -n chuaikan-production

# ดู 100 บรรทัดสุดท้าย
kubectl logs nextjs-app-xxx --tail=100 -n chuaikan-production

# --- EXEC Commands ---
# เข้าไปใน pod
kubectl exec -it nextjs-app-xxx -n chuaikan-production -- /bin/sh

# รัน command ใน pod
kubectl exec nextjs-app-xxx -n chuaikan-production -- ls -la /app

# copy file เข้า/ออก pod
kubectl cp nextjs-app-xxx:/app/logs/app.log ./app.log -n chuaikan-production

# --- PORT FORWARD ---
# forward local port ไป pod (สำหรับ debug)
kubectl port-forward pod/nextjs-app-xxx 3000:3000 -n chuaikan-production

# forward ผ่าน service
kubectl port-forward service/nextjs-service 8080:80 -n chuaikan-production
```

---

## 🔧 Configuration Files

### Deployment สมบูรณ์สำหรับ PostgreSQL

```yaml
# statefulset-postgres.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: chuaikan-production
spec:
  serviceName: postgres-headless
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
        component: database
    spec:
      containers:
      - name: postgres
        image: postgres:16-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: "chuaikan_db"
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: POSTGRES_USER
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: POSTGRES_PASSWORD
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        resources:
          requests:
            cpu: "250m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "2Gi"
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 10
          periodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "local-path"
      resources:
        requests:
          storage: 20Gi
```

---

## 🧪 Testing

### ทดสอบ Service Connectivity ภายใน Cluster

```bash
# สร้าง debug pod
kubectl run debug-pod \
  --image=curlimages/curl:latest \
  --restart=Never \
  --namespace=chuaikan-production \
  -- sleep 300

# ทดสอบ ClusterIP service
kubectl exec -it debug-pod -n chuaikan-production -- \
  curl http://api-service:4000/health

# ทดสอบ PostgreSQL
kubectl exec -it debug-pod -n chuaikan-production -- \
  nc -zv postgres-service 5432

# ลบ debug pod
kubectl delete pod debug-pod -n chuaikan-production
```

### ตรวจสอบ Rolling Update

```bash
# Update image
kubectl set image deployment/nextjs-app \
  nextjs=registry.chuaikan.com/nextjs-app:v1.1.0 \
  -n chuaikan-production --record

# ดู rollout status
kubectl rollout status deployment/nextjs-app -n chuaikan-production
# Waiting for deployment "nextjs-app" rollout to finish: 1 out of 3 new replicas have been updated...
# Waiting for deployment "nextjs-app" rollout to finish: 2 out of 3 new replicas have been updated...
# deployment "nextjs-app" successfully rolled out

# ดู history
kubectl rollout history deployment/nextjs-app -n chuaikan-production
```

---

## ❌ Common Errors & Solutions

### Error: `ImagePullBackOff`
```bash
# สาเหตุ: pull image ไม่ได้
kubectl describe pod <pod-name> -n chuaikan-production | grep -A 5 "Events"

# แก้: ตรวจสอบ image name และ registry credentials
kubectl get secret regcred -n chuaikan-production
# สร้าง secret สำหรับ private registry
kubectl create secret docker-registry regcred \
  --docker-server=registry.chuaikan.com \
  --docker-username=<username> \
  --docker-password=<password> \
  -n chuaikan-production

# เพิ่ม imagePullSecrets ใน pod spec
# spec:
#   imagePullSecrets:
#   - name: regcred
```

### Error: `CrashLoopBackOff`
```bash
# สาเหตุ: container start แล้ว crash ซ้ำๆ
# ดู logs ของ previous container
kubectl logs <pod-name> --previous -n chuaikan-production

# ดู events
kubectl describe pod <pod-name> -n chuaikan-production | tail -20
```

### Error: `Pending` (Pod ไม่ถูก schedule)
```bash
# สาเหตุที่เป็นไปได้:
# 1. Insufficient resources
# 2. Node selector ไม่ match
# 3. Taints/tolerations ไม่ match

kubectl describe pod <pod-name> -n chuaikan-production
# ดูที่ Events: Failed Scheduling ...
kubectl get nodes -o wide
kubectl top nodes
```

---

## ✅ Checklist

- [ ] **Step 321**: เข้าใจ Pod fundamentals และประเภท workloads ของ chuaikan.com
- [ ] **Step 322**: อธิบาย Service types ได้: ClusterIP, NodePort, LoadBalancer
- [ ] **Step 323**: เขียน Pod manifest สำหรับ Next.js ได้
- [ ] **Step 324**: สร้าง Deployment manifest พร้อม replicas, probes, resource limits
- [ ] **Step 325**: สร้าง Deployment สำหรับ Node.js API พร้อม init container
- [ ] **Step 326**: สร้าง Services สำหรับทุก components (Next.js, API, PostgreSQL, Redis)
- [ ] **Step 327**: ใช้ Init Containers รอ dependencies และ run migrations ได้
- [ ] **Step 328**: เข้าใจ Labels/Selectors และใช้ label conventions
- [ ] **Step 329**: ทำ Rolling Update และ Rollback ได้
- [ ] **Step 330**: ใช้ kubectl commands ได้คล่อง: get, describe, logs, exec, port-forward

---

## 🔗 References

- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Init Containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
- [Rolling Update Strategy](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment)

---

*Part 033 | Road to 1,000,000 Users/Day | chuaikan.com*
