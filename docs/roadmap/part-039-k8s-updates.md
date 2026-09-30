# Part 039: Rolling Updates & Rollbacks

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 381-390
> **เวลาโดยประมาณ:** 3 ชั่วโมง
> **Prerequisites:** Part 038 (Resource Limits), Part 033 (Workloads), Helm ติดตั้งแล้ว

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ตั้งค่า Rolling Update strategy ที่ถูกต้อง
- ใช้ Readiness และ Liveness probes
- ใช้ kubectl rollout commands
- ทำ Blue-Green Deployment บน K8s
- Upgrade และ rollback Helm charts
- เข้าใจ GitOps ด้วย Argo CD

---

## 📖 ทฤษฎีและแนวคิด

### Step 381 — Deployment Strategies เปรียบเทียบ

```
Rolling Update (default):
- ค่อยๆ แทนที่ old pods ด้วย new pods
- ไม่มี downtime
- ใช้ทรัพยากรปกติ
- rollback ได้ง่าย

Blue-Green:
- deploy ทั้ง environment ใหม่คู่กัน
- switch traffic ทีเดียว
- ใช้ทรัพยากร 2x ชั่วคราว
- rollback เร็วมาก (switch กลับ)

Canary:
- ส่ง traffic บางส่วนไป version ใหม่
- ทดสอบกับ production traffic จริง
- ลด risk
- ซับซ้อนกว่า

สำหรับ chuaikan.com:
- ทั่วไป: Rolling Update
- Feature ใหม่: Canary
- Infrastructure change: Blue-Green
```

### Step 382 — Probe Types

```
Readiness Probe:
- ตรวจว่า pod พร้อมรับ traffic หรือยัง
- ถ้า fail → ถูกถอดออกจาก Service endpoints
- Service จะไม่ส่ง traffic ไปถ้า pod ยังไม่ ready

Liveness Probe:
- ตรวจว่า pod ยังมีชีวิตอยู่หรือไม่
- ถ้า fail → restart container
- ใช้สำหรับ detect deadlock, hung state

Startup Probe (สำหรับ slow start apps):
- ตรวจว่า pod start เสร็จแล้วหรือยัง
- Liveness probe จะไม่ทำงานจนกว่า Startup probe จะผ่าน
```

---

## ⚙️ Environment Setup

```bash
# ตรวจสอบ deployment ปัจจุบัน
kubectl get deployments -n chuaikan-production
kubectl rollout history deployment/nextjs-app -n chuaikan-production
```

---

## 🛠️ Step-by-Step Implementation

### Step 383 — Rolling Update Configuration

```yaml
# deployment-nextjs-rolling.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextjs-app
  namespace: chuaikan-production
  annotations:
    # บันทึก deployment cause
    kubernetes.io/change-cause: "Deploy version 1.1.0 - Add user profile page"
spec:
  replicas: 6
  selector:
    matchLabels:
      app: nextjs
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1   # ลด 1 pod ต่อครั้ง (มีอย่างน้อย 5 pods ตลอด)
      maxSurge: 2         # เพิ่ม 2 pods ชั่วคราว (สูงสุด 8 pods)
  template:
    metadata:
      labels:
        app: nextjs
        version: "1.1.0"
    spec:
      terminationGracePeriodSeconds: 60   # รอ 60 วิ ก่อน force kill
      containers:
      - name: nextjs
        image: registry.chuaikan.com/nextjs-app:v1.1.0
        ports:
        - containerPort: 3000
        resources:
          requests:
            cpu: "100m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        # Readiness Probe
        readinessProbe:
          httpGet:
            path: /api/health
            port: 3000
            httpHeaders:
            - name: Accept
              value: application/json
          initialDelaySeconds: 10   # รอ 10 วิหลัง start
          periodSeconds: 5          # check ทุก 5 วิ
          timeoutSeconds: 3         # timeout 3 วิ
          successThreshold: 1       # ผ่าน 1 ครั้งถือว่า ready
          failureThreshold: 3       # fail 3 ครั้งถือว่า not ready

        # Liveness Probe
        livenessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 30   # รอ 30 วิหลัง start
          periodSeconds: 10         # check ทุก 10 วิ
          timeoutSeconds: 5
          failureThreshold: 3       # fail 3 ครั้ง → restart

        # Startup Probe (สำหรับ Next.js ที่ build นาน)
        startupProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 0
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 30      # รอได้นานสุด 30 * 5 = 150 วิ

        # Graceful shutdown
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 5 && kill -SIGTERM 1"]
```

### Step 384 — Probe สำหรับแต่ละ Service

```yaml
# probes สำหรับ Node.js API
readinessProbe:
  httpGet:
    path: /health
    port: 4000
  initialDelaySeconds: 15
  periodSeconds: 5
  failureThreshold: 3

livenessProbe:
  httpGet:
    path: /health
    port: 4000
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
```

```yaml
# probes สำหรับ PostgreSQL
readinessProbe:
  exec:
    command:
    - /bin/bash
    - -c
    - pg_isready -U $POSTGRES_USER -d $POSTGRES_DB
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 6

livenessProbe:
  exec:
    command:
    - /bin/bash
    - -c
    - pg_isready -U $POSTGRES_USER
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
```

```yaml
# probes สำหรับ Redis
readinessProbe:
  exec:
    command:
    - redis-cli
    - ping
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3

livenessProbe:
  exec:
    command:
    - redis-cli
    - ping
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 3
```

### Step 385 — kubectl rollout Commands

```bash
# ===== Update Deployment =====

# วิธีที่ 1: แก้ไข image ตรงๆ
kubectl set image deployment/nextjs-app \
  nextjs=registry.chuaikan.com/nextjs-app:v1.1.0 \
  -n chuaikan-production \
  --record

# วิธีที่ 2: apply manifest ใหม่
kubectl apply -f deployment-nextjs-rolling.yaml

# วิธีที่ 3: edit แบบ interactive
kubectl edit deployment/nextjs-app -n chuaikan-production

# ===== Monitor Rollout =====

# ดู progress แบบ real-time
kubectl rollout status deployment/nextjs-app -n chuaikan-production
# Waiting for deployment "nextjs-app" rollout to finish: 1 out of 6 new replicas have been updated...
# Waiting for deployment "nextjs-app" rollout to finish: 2 out of 6 new replicas have been updated...
# ...
# deployment "nextjs-app" successfully rolled out

# ดู pods ขณะ rolling update
kubectl get pods -l app=nextjs -n chuaikan-production -w

# ===== History =====
kubectl rollout history deployment/nextjs-app -n chuaikan-production
# REVISION  CHANGE-CAUSE
# 1         Deploy version 1.0.0 - Initial deployment
# 2         Deploy version 1.1.0 - Add user profile page

# ดูรายละเอียด revision
kubectl rollout history deployment/nextjs-app \
  --revision=2 \
  -n chuaikan-production

# ===== Rollback =====

# Rollback ไป version ก่อนหน้า
kubectl rollout undo deployment/nextjs-app -n chuaikan-production

# Rollback ไป revision ที่ระบุ
kubectl rollout undo deployment/nextjs-app \
  --to-revision=1 \
  -n chuaikan-production

# ===== Pause / Resume =====

# หยุด rolling update ชั่วคราว (ทดสอบ partial rollout)
kubectl rollout pause deployment/nextjs-app -n chuaikan-production

# ดู status (จะเห็น "Deployment is paused")
kubectl rollout status deployment/nextjs-app -n chuaikan-production

# resume rollout
kubectl rollout resume deployment/nextjs-app -n chuaikan-production

# ===== Restart Deployment (force re-pull image) =====
kubectl rollout restart deployment/nextjs-app -n chuaikan-production
```

### Step 386 — Blue-Green Deployment บน K8s

```yaml
# 1. Deploy Blue (current production)
# deployment-nextjs-blue.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextjs-blue
  namespace: chuaikan-production
  labels:
    app: nextjs
    slot: blue
spec:
  replicas: 6
  selector:
    matchLabels:
      app: nextjs
      slot: blue
  template:
    metadata:
      labels:
        app: nextjs
        slot: blue
        version: "1.0.0"
    spec:
      containers:
      - name: nextjs
        image: registry.chuaikan.com/nextjs-app:v1.0.0
        # ... resources, probes
```

```yaml
# 2. Deploy Green (new version)
# deployment-nextjs-green.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextjs-green
  namespace: chuaikan-production
  labels:
    app: nextjs
    slot: green
spec:
  replicas: 6
  selector:
    matchLabels:
      app: nextjs
      slot: green
  template:
    metadata:
      labels:
        app: nextjs
        slot: green
        version: "1.1.0"
    spec:
      containers:
      - name: nextjs
        image: registry.chuaikan.com/nextjs-app:v1.1.0
        # ... resources, probes
```

```yaml
# 3. Service ที่ชี้ไป Blue (ปัจจุบัน)
# service-nextjs-production.yaml
apiVersion: v1
kind: Service
metadata:
  name: nextjs-service
  namespace: chuaikan-production
spec:
  selector:
    app: nextjs
    slot: blue    # ชี้ไป blue ก่อน
  ports:
  - port: 3000
    targetPort: 3000
```

```bash
# Deploy Blue และ Green
kubectl apply -f deployment-nextjs-blue.yaml
kubectl apply -f deployment-nextjs-green.yaml

# รอ Green พร้อม
kubectl rollout status deployment/nextjs-green -n chuaikan-production

# ทดสอบ Green โดยตรง (ผ่าน port-forward)
kubectl port-forward deployment/nextjs-green 3001:3000 -n chuaikan-production
curl http://localhost:3001/api/health

# Switch traffic ไป Green (atomic switch!)
kubectl patch service nextjs-service \
  -n chuaikan-production \
  -p '{"spec":{"selector":{"app":"nextjs","slot":"green"}}}'

# ตรวจสอบว่า traffic ไป Green แล้ว
kubectl get service nextjs-service -n chuaikan-production -o yaml | grep slot

# ถ้ามีปัญหา: switch กลับ Blue ทันที
kubectl patch service nextjs-service \
  -n chuaikan-production \
  -p '{"spec":{"selector":{"app":"nextjs","slot":"blue"}}}'

# หลัง Green stable: ลบ Blue
kubectl delete deployment nextjs-blue -n chuaikan-production
```

### Step 387 — Helm Charts สำหรับ chuaikan.com

```bash
# สร้าง Helm chart สำหรับ chuaikan.com
helm create chuaikan-nextjs
cd chuaikan-nextjs

# โครงสร้าง chart
# chuaikan-nextjs/
# ├── Chart.yaml
# ├── values.yaml
# ├── templates/
# │   ├── deployment.yaml
# │   ├── service.yaml
# │   ├── ingress.yaml
# │   ├── hpa.yaml
# │   └── configmap.yaml
```

```yaml
# values.yaml
image:
  repository: registry.chuaikan.com/nextjs-app
  tag: "v1.0.0"
  pullPolicy: Always

replicas: 6
namespace: chuaikan-production

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

ingress:
  enabled: true
  host: www.chuaikan.com
  tls: true

hpa:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70
```

```bash
# Deploy ด้วย Helm
helm install chuaikan-nextjs ./chuaikan-nextjs \
  --namespace chuaikan-production \
  --values values.production.yaml

# ดู releases
helm list -n chuaikan-production

# Upgrade (deploy version ใหม่)
helm upgrade chuaikan-nextjs ./chuaikan-nextjs \
  --namespace chuaikan-production \
  --set image.tag=v1.1.0 \
  --atomic \            # rollback อัตโนมัติถ้า fail
  --timeout 5m

# ดู upgrade history
helm history chuaikan-nextjs -n chuaikan-production

# Rollback Helm release
helm rollback chuaikan-nextjs 1 -n chuaikan-production
# 1 = revision number

# ดูความแตกต่างระหว่าง revision
helm diff revision chuaikan-nextjs 1 2 -n chuaikan-production
```

### Step 388 — GitOps ด้วย Argo CD (พื้นฐาน)

```bash
# ติดตั้ง Argo CD
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# รอ Argo CD พร้อม
kubectl wait --for=condition=available deployment \
  -l app.kubernetes.io/name=argocd-server \
  -n argocd \
  --timeout=120s

# Get initial password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
echo ""

# Port-forward Argo CD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# เปิด https://localhost:8080
# username: admin, password: (จากขั้นตอนก่อน)

# ติดตั้ง argocd CLI
curl -sSL -o argocd-linux-amd64 \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

# Login
argocd login localhost:8080 \
  --username admin \
  --password <password> \
  --insecure
```

```yaml
# argocd-app-chuaikan.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: chuaikan-nextjs
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/chuaikan/k8s-manifests.git
    targetRevision: main
    path: production/nextjs
  destination:
    server: https://kubernetes.default.svc
    namespace: chuaikan-production
  syncPolicy:
    automated:
      prune: true        # ลบ resource ที่ไม่อยู่ใน Git
      selfHeal: true     # แก้ไข manual change กลับให้ตรง Git
    syncOptions:
    - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

```bash
kubectl apply -f argocd-app-chuaikan.yaml

# ดู application status
argocd app get chuaikan-nextjs
argocd app sync chuaikan-nextjs   # force sync

# GitOps workflow:
# 1. แก้ไข image tag ใน Git repo
# 2. Argo CD detect change (ทุก 3 นาที หรือ webhook)
# 3. Argo CD sync deployment โดยอัตโนมัติ
```

### Step 389 — Deploy Pipeline ที่สมบูรณ์

```bash
# สร้าง deploy script ที่ใช้ใน CI/CD
cat > deploy.sh <<'EOF'
#!/bin/bash
set -e

NAMESPACE="chuaikan-production"
DEPLOYMENT="nextjs-app"
IMAGE="registry.chuaikan.com/nextjs-app"
TAG="${1:-latest}"

echo "=== Deploying $IMAGE:$TAG ==="

# 1. Update image
kubectl set image deployment/$DEPLOYMENT \
  nextjs=$IMAGE:$TAG \
  -n $NAMESPACE \
  --record

# 2. Wait for rollout
echo "Waiting for rollout to complete..."
kubectl rollout status deployment/$DEPLOYMENT \
  -n $NAMESPACE \
  --timeout=300s

# 3. Verify pods are healthy
READY=$(kubectl get deployment $DEPLOYMENT -n $NAMESPACE \
  -o jsonpath='{.status.readyReplicas}')
DESIRED=$(kubectl get deployment $DEPLOYMENT -n $NAMESPACE \
  -o jsonpath='{.spec.replicas}')

if [ "$READY" != "$DESIRED" ]; then
  echo "ERROR: Only $READY/$DESIRED pods are ready"
  echo "Rolling back..."
  kubectl rollout undo deployment/$DEPLOYMENT -n $NAMESPACE
  exit 1
fi

echo "✓ Deployment successful: $READY/$DESIRED pods ready"
EOF

chmod +x deploy.sh
./deploy.sh v1.1.0
```

### Step 390 — Zero-Downtime Deploy Verification

```bash
# สร้าง script ทดสอบ zero-downtime
cat > test-zero-downtime.sh <<'EOF'
#!/bin/bash
URL="https://www.chuaikan.com/api/health"
SUCCESS=0
FAIL=0
TOTAL=60

echo "Testing zero-downtime deploy for $TOTAL seconds..."

for i in $(seq 1 $TOTAL); do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" --max-time 3 $URL)
  if [ "$STATUS" = "200" ]; then
    SUCCESS=$((SUCCESS + 1))
    echo -n "."
  else
    FAIL=$((FAIL + 1))
    echo -n "X($STATUS)"
  fi
  sleep 1
done

echo ""
echo "Results: $SUCCESS/$TOTAL successful, $FAIL failures"

if [ $FAIL -gt 0 ]; then
  echo "FAIL: $FAIL requests failed during deployment"
  exit 1
else
  echo "PASS: Zero downtime achieved!"
fi
EOF

chmod +x test-zero-downtime.sh

# รัน test ขณะ deploy
./test-zero-downtime.sh &
./deploy.sh v1.1.0
wait
```

---

## 🔧 Configuration Files

### Deployment Notification ด้วย Slack

```yaml
# ใน CI/CD pipeline (GitLab CI ตัวอย่าง)
# .gitlab-ci.yml snippet
deploy-production:
  stage: deploy
  script:
    - |
      # Deploy
      ./deploy.sh $CI_COMMIT_TAG

      # Notify Slack
      curl -X POST -H 'Content-type: application/json' \
        --data "{
          \"text\": \"Deployed *chuaikan.com* v$CI_COMMIT_TAG to production\",
          \"attachments\": [{
            \"color\": \"good\",
            \"fields\": [
              {\"title\": \"Version\", \"value\": \"$CI_COMMIT_TAG\"},
              {\"title\": \"Deployed by\", \"value\": \"$GITLAB_USER_NAME\"}
            ]
          }]
        }" \
        $SLACK_WEBHOOK_URL
```

---

## ❌ Common Errors & Solutions

### Error: `Deployment does not have minimum availability`
```bash
# สาเหตุ: pods ไม่ผ่าน readiness probe
kubectl get pods -l app=nextjs -n chuaikan-production
kubectl describe pod <pod-name> -n chuaikan-production | grep -A 20 "Events:"

# ตรวจสอบ health endpoint
kubectl port-forward <pod-name> 3000:3000 -n chuaikan-production
curl http://localhost:3000/api/health
```

### Error: Helm upgrade fails - `UPGRADE FAILED`
```bash
# ดู history
helm history chuaikan-nextjs -n chuaikan-production

# Rollback ไป revision ก่อนหน้า
helm rollback chuaikan-nextjs 0 -n chuaikan-production
# 0 = rollback to previous
```

---

## ✅ Checklist

- [ ] **Step 381**: เข้าใจ deployment strategies: Rolling Update, Blue-Green, Canary
- [ ] **Step 382**: อธิบาย probe types ได้: Readiness, Liveness, Startup
- [ ] **Step 383**: ตั้งค่า Rolling Update strategy พร้อม maxUnavailable, maxSurge
- [ ] **Step 384**: ตั้งค่า probes ที่เหมาะสมสำหรับทุก services
- [ ] **Step 385**: ใช้ kubectl rollout commands: status, history, undo, pause, resume, restart
- [ ] **Step 386**: ทำ Blue-Green deployment และ switch traffic ด้วย Service selector
- [ ] **Step 387**: สร้าง Helm chart, upgrade, และ rollback ได้
- [ ] **Step 388**: ติดตั้ง Argo CD และสร้าง Application ที่ sync จาก Git ได้
- [ ] **Step 389**: สร้าง deploy script ที่มี rollback อัตโนมัติเมื่อ fail
- [ ] **Step 390**: ยืนยัน zero-downtime deployment ด้วย test script

---

## 🔗 References

- [Kubernetes Deployment Strategies](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy)
- [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Helm Documentation](https://helm.sh/docs/)
- [Argo CD Documentation](https://argo-cd.readthedocs.io/en/stable/)

---

*Part 039 | Road to 1,000,000 Users/Day | chuaikan.com*
