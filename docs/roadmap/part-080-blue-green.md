# Part 080: Blue-Green Deployment
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Professional
> **Steps:** 791–800
> **เวลาโดยประมาณ:** 6 ชั่วโมง
> **Prerequisites:** Part 031 (Kubernetes), Part 079 (Latency Opt)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- Blue-green deployment บน Kubernetes ด้วย Argo Rollouts
- Traffic shifting: 100% blue → 100% green
- Database migration ที่ compatible กับทั้งสอง version
- Smoke tests ก่อน switch traffic
- Instant rollback
- Blue-green ด้วย Cloudflare Workers routing
- Monitoring ระหว่าง switchover
- Runbook สำหรับ blue-green deploy

---

## 📖 ทฤษฎีและแนวคิด

### Blue-Green Deployment คืออะไร?

```
Blue (current):  [v1.2.0] ← รับ 100% traffic
Green (new):     [v1.3.0] ← ยังไม่รับ traffic

After switch:
Blue (old):      [v1.2.0] ← standby (สำหรับ rollback)
Green (current): [v1.3.0] ← รับ 100% traffic
```

ข้อดีของ Blue-Green:
- Zero-downtime deployment
- Instant rollback (แค่ switch traffic กลับ)
- ทดสอบ environment ใหม่ก่อน production จริง

ข้อเสีย:
- ต้องใช้ทรัพยากร 2x ชั่วคราว
- Database ต้องรองรับทั้ง v1 และ v2 ในช่วง transition

---

## ⚙️ Environment Setup

### Step 791: ติดตั้ง Argo Rollouts

```bash
# ติดตั้ง Argo Rollouts controller
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# ติดตั้ง kubectl plugin
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64
sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts

# ตรวจสอบ
kubectl argo rollouts version
```

---

## 🛠️ Step-by-Step Implementation

### Step 792: Rollout Definition

```yaml
# k8s/rollouts/api-service.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: chuaikan-api
  namespace: chuaikan
spec:
  replicas: 10
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: chuaikan-api

  template:
    metadata:
      labels:
        app: chuaikan-api
    spec:
      containers:
        - name: api
          image: registry.chuaikan.com/api:latest
          ports:
            - containerPort: 3000
          env:
            - name: NODE_ENV
              value: production
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: host
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "1000m"
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10

  strategy:
    blueGreen:
      # Active service: รับ production traffic
      activeService: chuaikan-api-active

      # Preview service: ทดสอบก่อน promote
      previewService: chuaikan-api-preview

      # รอให้ manual approve ก่อน switch
      autoPromotionEnabled: false

      # เวลาที่รอหลัง promote ก่อนลบ old version
      scaleDownDelaySeconds: 300

      # Anti-affinity: spread pods ระหว่าง nodes
      antiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution: {}

      # Pre-promotion analysis: ตรวจสอบ metrics ก่อน promote
      prePromotionAnalysis:
        templates:
          - templateName: smoke-test-analysis
        args:
          - name: service-name
            value: chuaikan-api-preview

      # Post-promotion analysis: ตรวจสอบ metrics หลัง promote
      postPromotionAnalysis:
        templates:
          - templateName: error-rate-analysis
        args:
          - name: service-name
            value: chuaikan-api-active
```

```yaml
# k8s/rollouts/services.yaml

# Active Service: production traffic
apiVersion: v1
kind: Service
metadata:
  name: chuaikan-api-active
  namespace: chuaikan
spec:
  selector:
    app: chuaikan-api
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP

---
# Preview Service: ทดสอบ new version
apiVersion: v1
kind: Service
metadata:
  name: chuaikan-api-preview
  namespace: chuaikan
  annotations:
    # ใช้สำหรับ internal testing เท่านั้น
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
spec:
  selector:
    app: chuaikan-api
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP
```

### Step 793: AnalysisTemplate สำหรับ Smoke Tests

```yaml
# k8s/rollouts/analysis-templates.yaml

# Smoke test analysis
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: smoke-test-analysis
  namespace: chuaikan
spec:
  args:
    - name: service-name
  metrics:
    - name: smoke-test
      provider:
        job:
          spec:
            template:
              spec:
                restartPolicy: Never
                containers:
                  - name: smoke-test
                    image: registry.chuaikan.com/smoke-tests:latest
                    env:
                      - name: TARGET_URL
                        value: "http://{{ args.service-name }}.chuaikan.svc.cluster.local"
                    command:
                      - /bin/sh
                      - -c
                      - |
                        echo "Running smoke tests against {{ args.service-name }}..."

                        # Test 1: Health check
                        curl -sf "$TARGET_URL/health" || exit 1
                        echo "Health check: PASS"

                        # Test 2: API response format
                        RESPONSE=$(curl -sf "$TARGET_URL/v1/posts?limit=1")
                        echo $RESPONSE | jq '.data | length > 0' | grep -q true || exit 1
                        echo "API format: PASS"

                        # Test 3: Auth endpoint
                        STATUS=$(curl -o /dev/null -sw "%{http_code}" "$TARGET_URL/v1/me")
                        [ "$STATUS" = "401" ] || exit 1
                        echo "Auth check: PASS"

                        echo "All smoke tests passed!"
                        exit 0

---
# Error rate analysis หลัง promote
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate-analysis
  namespace: chuaikan
spec:
  args:
    - name: service-name
  metrics:
    - name: error-rate
      interval: 1m
      count: 5
      successCondition: result < 0.01   # error rate < 1%
      failureCondition: result > 0.05   # error rate > 5% = fail immediately
      provider:
        prometheus:
          address: http://prometheus.monitoring.svc:9090
          query: |
            sum(rate(http_requests_total{
              service="{{ args.service-name }}",
              status=~"5.."
            }[1m]))
            /
            sum(rate(http_requests_total{
              service="{{ args.service-name }}"
            }[1m]))

    - name: latency-p99
      interval: 1m
      count: 5
      successCondition: result < 0.1    # P99 < 100ms
      failureCondition: result > 0.5    # P99 > 500ms = fail
      provider:
        prometheus:
          address: http://prometheus.monitoring.svc:9090
          query: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket{
                service="{{ args.service-name }}"
              }[1m])) by (le)
            )
```

### Step 794: Database Migration ที่ Compatible

```sql
-- ตัวอย่าง: เพิ่ม column ใหม่ใน posts table
-- Strategy: Expand-Contract (ทำหลาย step)

-- STEP 1: Expand (ทำก่อน deploy new version)
-- เพิ่ม column ที่ nullable → ทั้ง v1 และ v2 ทำงานได้
ALTER TABLE posts ADD COLUMN view_count INTEGER DEFAULT 0;
-- v1 ไม่รู้จัก column นี้ แต่ก็ไม่พัง
-- v2 เริ่มอ่าน/เขียน view_count ได้

-- STEP 2: Deploy v2 (blue-green)
-- v2 เริ่ม populate view_count

-- STEP 3: Contract (หลัง v2 stable แล้ว)
-- เพิ่ม NOT NULL constraint หลังจาก backfill เสร็จ
UPDATE posts SET view_count = 0 WHERE view_count IS NULL;
ALTER TABLE posts ALTER COLUMN view_count SET NOT NULL;
```

```python
# scripts/db_migration_check.py
"""
ตรวจสอบว่า database migration compatible กับทั้งสอง version
"""

import psycopg2
import sys

def check_migration_compatibility(dsn: str, migration_file: str) -> bool:
    """
    อ่าน migration file และตรวจสอบว่า:
    1. ไม่มี DROP COLUMN
    2. ไม่มี NOT NULL ที่ไม่มี DEFAULT
    3. ไม่มี RENAME COLUMN/TABLE
    """
    unsafe_patterns = [
        'DROP COLUMN',
        'DROP TABLE',
        'RENAME COLUMN',
        'RENAME TABLE',
        'ALTER COLUMN.*NOT NULL',  # ไม่ ok ถ้าไม่มี DEFAULT
    ]

    with open(migration_file, 'r') as f:
        migration_sql = f.read().upper()

    issues = []
    for pattern in unsafe_patterns:
        import re
        if re.search(pattern, migration_sql):
            issues.append(f"Found potentially unsafe pattern: {pattern}")

    if issues:
        print("Migration compatibility issues found:")
        for issue in issues:
            print(f"  - {issue}")
        print("\nFix: Use expand-contract pattern")
        return False

    print("Migration compatibility check: PASSED")
    return True

if __name__ == "__main__":
    ok = check_migration_compatibility(
        dsn="host=sg.db.chuaikan.com dbname=chuaikan",
        migration_file=sys.argv[1]
    )
    sys.exit(0 if ok else 1)
```

### Step 795: Blue-Green Deploy Script

```bash
#!/bin/bash
# scripts/blue-green-deploy.sh
set -euo pipefail

IMAGE_TAG="${1:-latest}"
NAMESPACE="chuaikan"
ROLLOUT_NAME="chuaikan-api"

echo "=== Blue-Green Deploy: ${IMAGE_TAG} ==="

# Step 1: ตรวจสอบ current status
echo "Current rollout status:"
kubectl argo rollouts get rollout ${ROLLOUT_NAME} -n ${NAMESPACE}

# Step 2: Check migration compatibility
if [ -f "migrations/latest.sql" ]; then
    echo "Checking migration compatibility..."
    python3 scripts/db_migration_check.py migrations/latest.sql || {
        echo "ERROR: Migration not compatible! Aborting."
        exit 1
    }

    # Apply migration
    echo "Applying database migration..."
    psql "${DATABASE_URL}" < migrations/latest.sql
fi

# Step 3: Update image (triggers rollout)
echo "Updating image to ${IMAGE_TAG}..."
kubectl argo rollouts set image ${ROLLOUT_NAME} \
  api=registry.chuaikan.com/api:${IMAGE_TAG} \
  -n ${NAMESPACE}

# Step 4: รอ preview stable
echo "Waiting for preview to be stable..."
kubectl argo rollouts status ${ROLLOUT_NAME} \
  -n ${NAMESPACE} \
  --timeout=300s \
  --watch || true

# Step 5: ตรวจสอบ preview health
PREVIEW_URL="http://chuaikan-api-preview.${NAMESPACE}.svc.cluster.local"
echo "Testing preview service..."
kubectl run smoke-test-$(date +%s) \
  --image=curlimages/curl:latest \
  --restart=Never \
  --rm \
  --command -- curl -sf "${PREVIEW_URL}/health"

# Step 6: Manual approval
echo ""
echo "=== PREVIEW IS READY FOR PROMOTION ==="
echo "Preview URL: https://preview.chuaikan.com"
echo ""
read -p "Promote to production? (yes/no): " CONFIRM

if [ "${CONFIRM}" != "yes" ]; then
    echo "Aborting. Rolling back..."
    kubectl argo rollouts abort ${ROLLOUT_NAME} -n ${NAMESPACE}
    exit 0
fi

# Step 7: Promote
echo "Promoting to production..."
kubectl argo rollouts promote ${ROLLOUT_NAME} -n ${NAMESPACE}

# Step 8: Monitor post-promotion
echo "Monitoring post-promotion for 5 minutes..."
sleep 300

# ตรวจสอบ error rate
ERROR_RATE=$(kubectl exec -n monitoring deploy/prometheus -- \
  promtool query instant \
  'sum(rate(http_requests_total{status=~"5..",service="chuaikan-api-active"}[5m])) / sum(rate(http_requests_total{service="chuaikan-api-active"}[5m]))' \
  | grep -o '[0-9.]*$')

if (( $(echo "${ERROR_RATE} > 0.01" | bc -l) )); then
    echo "ERROR: Error rate ${ERROR_RATE} > 1%! Rolling back..."
    kubectl argo rollouts undo ${ROLLOUT_NAME} -n ${NAMESPACE}
    exit 1
fi

echo "=== DEPLOY SUCCESSFUL ==="
echo "New version ${IMAGE_TAG} is now live."
```

### Step 796: Cloudflare Workers Blue-Green Routing

```javascript
// cloudflare-workers/blue-green-router.js
// ควบคุม traffic routing ระหว่าง blue/green

export default {
  async fetch(request, env, ctx) {
    const activeVersion = await env.KV.get('active_version') || 'blue';

    // Route ไปยัง active version
    const targetOrigin = activeVersion === 'blue'
      ? 'https://blue.api.chuaikan.com'
      : 'https://green.api.chuaikan.com';

    const response = await fetch(new Request(targetOrigin + new URL(request.url).pathname, request));

    // เพิ่ม version header สำหรับ debugging
    const headers = new Headers(response.headers);
    headers.set('X-Active-Version', activeVersion);

    return new Response(response.body, {
      status: response.status,
      headers
    });
  }
};

// สลับ traffic ด้วย KV store
// wrangler kv:key put --binding KV active_version green
```

---

## 📋 Runbook: Blue-Green Deploy

```bash
#!/bin/bash
# Runbook: Blue-Green Deploy สำหรับ chuaikan.com

# Pre-deployment checklist
echo "=== Pre-deployment Checklist ==="
echo "1. [ ] Migration script ผ่าน compatibility check"
echo "2. [ ] Feature flags เปิดสำหรับ internal testers"
echo "3. [ ] Monitoring dashboards พร้อม"
echo "4. [ ] Rollback plan พร้อม"
echo "5. [ ] On-call engineer พร้อม"
read -p "Checklist complete? (yes): "

# Deploy
bash scripts/blue-green-deploy.sh v1.3.0

# Post-deployment
echo "=== Post-deployment Monitoring (30 minutes) ==="
echo "ดู Dashboard: https://grafana.chuaikan.com/d/deployments"
echo "Error rate target: < 0.1%"
echo "P99 latency target: < 100ms"
```

---

## ✅ Checklist

- [ ] Argo Rollouts ติดตั้งและ config แล้ว
- [ ] Active service และ Preview service สร้างแล้ว
- [ ] Smoke test analysis template ทำงานได้
- [ ] Error rate analysis template ทำงานได้
- [ ] Migration compatibility checker ใช้งานได้
- [ ] Blue-green deploy script ทดสอบใน staging แล้ว
- [ ] Cloudflare Workers routing พร้อม
- [ ] Rollback ทดสอบแล้ว (< 2 นาที)
- [ ] Monitoring dashboard สำหรับ switchover พร้อม
- [ ] Team รู้ขั้นตอน deploy (runbook)

---

## 🔗 References

- [Argo Rollouts Blue-Green](https://argoproj.github.io/argo-rollouts/features/bluegreen/)
- [Expand-Contract Pattern](https://openpracticelibrary.com/practice/expand-and-contract-pattern/)

---
*Part 080 | Road to 1,000,000 Users/Day | chuaikan.com*
