# Part 081: Canary Releases
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Professional
> **Steps:** 801–810
> **เวลาโดยประมาณ:** 6 ชั่วโมง
> **Prerequisites:** Part 080 (Blue-Green), Part 031 (Kubernetes)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- Canary deployment strategy: 5% → 25% → 100%
- Argo Rollouts canary configuration
- Automated canary analysis ด้วย Prometheus metrics
- Feature flags as code canary
- Canary สำหรับ database changes
- Rollback triggers (error rate > 1%, latency increase > 20%)
- Canary monitoring dashboard

---

## 📖 ทฤษฎีและแนวคิด

### Canary Release คืออะไร?

Canary release คือการ deploy version ใหม่ให้ user กลุ่มเล็กๆ ก่อน (เหมือน canary ในเหมืองถ่านหิน ที่ถ้าตายก่อนแสดงว่ามีอันตราย)

```
Step 1: 5% traffic → v2  (monitor 10 นาที)
Step 2: 25% traffic → v2 (monitor 10 นาที)
Step 3: 50% traffic → v2 (monitor 10 นาที)
Step 4: 100% traffic → v2 (stable)
```

ต่างจาก Blue-Green:
- Blue-Green: 0% → 100% ทันที (rollback ได้เร็ว)
- Canary: 5% → 25% → 100% ค่อยๆ (ลด blast radius)

---

## 🛠️ Step-by-Step Implementation

### Step 801: Argo Rollouts Canary Configuration

```yaml
# k8s/rollouts/api-canary.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: chuaikan-api
  namespace: chuaikan
spec:
  replicas: 20
  revisionHistoryLimit: 5
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

  strategy:
    canary:
      # Canary service สำหรับ monitoring แยก metrics
      canaryService: chuaikan-api-canary
      stableService: chuaikan-api-stable

      # Traffic management ด้วย Ingress
      trafficRouting:
        nginx:
          stableIngress: chuaikan-api-ingress
          additionalIngressAnnotations:
            canary-by-header: X-Canary

      # Analysis ก่อน promote แต่ละ step
      analysis:
        templates:
          - templateName: canary-metrics-check
        startingStep: 1
        args:
          - name: service-name
            value: chuaikan-api-canary

      # Steps: 5% → 25% → 50% → 100%
      steps:
        - setWeight: 5
        - pause: {duration: 10m}
        - analysis:
            templates:
              - templateName: canary-metrics-check
            args:
              - name: service-name
                value: chuaikan-api-canary

        - setWeight: 25
        - pause: {duration: 10m}
        - analysis:
            templates:
              - templateName: canary-metrics-check
            args:
              - name: service-name
                value: chuaikan-api-canary

        - setWeight: 50
        - pause: {duration: 10m}
        - analysis:
            templates:
              - templateName: canary-metrics-check
            args:
              - name: service-name
                value: chuaikan-api-canary

        - setWeight: 100
```

### Step 802: Canary Analysis Template

```yaml
# k8s/rollouts/canary-analysis.yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: canary-metrics-check
  namespace: chuaikan
spec:
  args:
    - name: service-name
  metrics:
    # Metric 1: Error rate < 1%
    - name: error-rate
      interval: 2m
      count: 5
      successCondition: result < 0.01
      failureCondition: result > 0.05
      inconclusiveCondition: result >= 0.01 && result <= 0.05
      provider:
        prometheus:
          address: http://prometheus.monitoring.svc:9090
          query: |
            sum(rate(http_requests_total{
              service="{{ args.service-name }}",
              status=~"5.."
            }[2m]))
            /
            sum(rate(http_requests_total{
              service="{{ args.service-name }}"
            }[2m]))

    # Metric 2: P99 latency < 120ms (ไม่เพิ่มจาก baseline > 20%)
    - name: latency-p99
      interval: 2m
      count: 5
      successCondition: result < 0.12
      failureCondition: result > 0.2
      provider:
        prometheus:
          address: http://prometheus.monitoring.svc:9090
          query: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket{
                service="{{ args.service-name }}"
              }[2m])) by (le)
            )

    # Metric 3: Success rate > 99%
    - name: success-rate
      interval: 2m
      count: 5
      successCondition: result > 0.99
      failureCondition: result < 0.95
      provider:
        prometheus:
          address: http://prometheus.monitoring.svc:9090
          query: |
            sum(rate(http_requests_total{
              service="{{ args.service-name }}",
              status=~"2..|3.."
            }[2m]))
            /
            sum(rate(http_requests_total{
              service="{{ args.service-name }}"
            }[2m]))
```

### Step 803: Canary Ingress Configuration

```yaml
# k8s/ingress/canary-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: chuaikan-api-ingress
  namespace: chuaikan
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
spec:
  tls:
    - hosts:
        - api.chuaikan.com
      secretName: chuaikan-tls
  rules:
    - host: api.chuaikan.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: chuaikan-api-stable
                port:
                  number: 80
```

### Step 804: Feature Flags as Code Canary

```typescript
// src/config/feature-flags.ts
// ใช้ feature flags ควบคู่กับ canary release

interface FeatureConfig {
  newFeedAlgorithm: {
    enabled: boolean;
    rolloutPercentage: number;
  };
  sosAlertV2: {
    enabled: boolean;
    userSegments: string[];
  };
}

// อ่านจาก environment variable (inject ตอน deploy)
export const features: FeatureConfig = {
  newFeedAlgorithm: {
    enabled: process.env.FEATURE_NEW_FEED === 'true',
    rolloutPercentage: parseInt(process.env.FEATURE_NEW_FEED_ROLLOUT || '0', 10)
  },
  sosAlertV2: {
    enabled: process.env.FEATURE_SOS_V2 === 'true',
    userSegments: (process.env.FEATURE_SOS_V2_SEGMENTS || '').split(',').filter(Boolean)
  }
};

// ตรวจสอบว่า user ควรได้ feature ใหม่หรือไม่
export function isFeatureEnabled(
  featureName: keyof FeatureConfig,
  userId: string,
  userSegment?: string
): boolean {
  const config = features[featureName];
  if (!config || !config.enabled) return false;

  // Percentage-based rollout
  if ('rolloutPercentage' in config) {
    const hash = hashUserId(userId);
    const userPercentile = hash % 100;
    return userPercentile < config.rolloutPercentage;
  }

  // Segment-based rollout
  if ('userSegments' in config && userSegment) {
    return config.userSegments.includes(userSegment);
  }

  return config.enabled;
}

// Hash ที่ deterministic สำหรับ user เดิม
function hashUserId(userId: string): number {
  let hash = 5381;
  for (const char of userId) {
    hash = ((hash << 5) + hash) + char.charCodeAt(0);
    hash = hash & hash; // Convert to 32bit integer
  }
  return Math.abs(hash);
}
```

```typescript
// src/routes/feed.ts
import { isFeatureEnabled } from '../config/feature-flags';
import { getFeedV1, getFeedV2 } from '../services/feed';

router.get('/feed', authenticate, async (req, res) => {
  const userId = req.user.id;
  const useNewAlgorithm = isFeatureEnabled('newFeedAlgorithm', userId);

  const startTime = Date.now();
  let feedPosts;

  if (useNewAlgorithm) {
    feedPosts = await getFeedV2(userId);
    res.set('X-Feed-Algorithm', 'v2');
  } else {
    feedPosts = await getFeedV1(userId);
    res.set('X-Feed-Algorithm', 'v1');
  }

  const duration = Date.now() - startTime;

  // Track metrics แยกตาม algorithm
  metrics.histogram('feed_load_duration_ms', duration, {
    algorithm: useNewAlgorithm ? 'v2' : 'v1',
    userId
  });

  return res.json({ data: feedPosts });
});
```

### Step 805: Canary สำหรับ Database Changes

```bash
#!/bin/bash
# scripts/canary-db-migration.sh
# Deploy database migration แบบ canary

set -euo pipefail

MIGRATION_FILE="$1"
CANARY_PERCENTAGE="${2:-5}"

echo "=== Canary Database Migration ==="
echo "File: ${MIGRATION_FILE}"
echo "Initial traffic: ${CANARY_PERCENTAGE}%"

# Step 1: ตรวจสอบ migration เป็น additive only
python3 scripts/db_migration_check.py "${MIGRATION_FILE}"

# Step 2: Apply migration ไป Read Replica ก่อน (ทดสอบ)
echo "Testing migration on read replica..."
psql "${DATABASE_REPLICA_URL}" < "${MIGRATION_FILE}"
echo "Migration on replica: SUCCESS"

# Step 3: Apply migration บน Primary
echo "Applying migration on primary..."
psql "${DATABASE_URL}" < "${MIGRATION_FILE}"
echo "Migration on primary: SUCCESS"

# Step 4: Deploy canary pods ด้วย new feature flag
echo "Deploying ${CANARY_PERCENTAGE}% canary pods..."
kubectl argo rollouts set image chuaikan-api \
  api=registry.chuaikan.com/api:${NEW_VERSION} \
  -n chuaikan

# Step 5: Monitor 30 นาที
echo "Monitoring for 30 minutes..."
for i in $(seq 1 6); do
    sleep 300
    ERROR_RATE=$(kubectl exec -n monitoring deploy/prometheus -- \
      promtool query instant \
      'sum(rate(http_requests_total{status=~"5..",service="chuaikan-api-canary"}[5m])) / sum(rate(http_requests_total{service="chuaikan-api-canary"}[5m]))' \
      2>/dev/null | grep -o '[0-9.]*$' || echo "0")

    echo "Check ${i}/6 - Error rate: ${ERROR_RATE}"

    if (( $(echo "${ERROR_RATE} > 0.01" | bc -l 2>/dev/null || echo 0) )); then
        echo "ERROR: Error rate too high! Rolling back..."
        kubectl argo rollouts undo chuaikan-api -n chuaikan
        exit 1
    fi
done

echo "Canary migration: SUCCESS"
```

### Step 806: Canary Monitoring Dashboard

```yaml
# k8s/grafana/canary-dashboard.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: canary-monitoring-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "true"
data:
  canary-dashboard.json: |
    {
      "title": "Canary Release Monitor - chuaikan.com",
      "refresh": "30s",
      "panels": [
        {
          "title": "Traffic Split (Stable vs Canary)",
          "type": "piechart",
          "targets": [
            {
              "expr": "sum(rate(http_requests_total{service='chuaikan-api-stable'}[1m]))",
              "legendFormat": "Stable"
            },
            {
              "expr": "sum(rate(http_requests_total{service='chuaikan-api-canary'}[1m]))",
              "legendFormat": "Canary"
            }
          ]
        },
        {
          "title": "Error Rate Comparison",
          "type": "timeseries",
          "targets": [
            {
              "expr": "sum(rate(http_requests_total{service='chuaikan-api-stable',status=~'5..'}[2m])) / sum(rate(http_requests_total{service='chuaikan-api-stable'}[2m]))",
              "legendFormat": "Stable Error Rate"
            },
            {
              "expr": "sum(rate(http_requests_total{service='chuaikan-api-canary',status=~'5..'}[2m])) / sum(rate(http_requests_total{service='chuaikan-api-canary'}[2m]))",
              "legendFormat": "Canary Error Rate"
            }
          ],
          "thresholds": [
            {"color": "green", "value": null},
            {"color": "red", "value": 0.01}
          ]
        },
        {
          "title": "P99 Latency Comparison",
          "type": "timeseries",
          "targets": [
            {
              "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{service='chuaikan-api-stable'}[2m])) by (le))",
              "legendFormat": "Stable P99"
            },
            {
              "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{service='chuaikan-api-canary'}[2m])) by (le))",
              "legendFormat": "Canary P99"
            }
          ]
        }
      ]
    }
```

---

## 🧪 Testing

```bash
# ดู status ของ canary rollout
kubectl argo rollouts get rollout chuaikan-api -n chuaikan --watch

# ตรวจสอบว่า canary pods มี label ถูกต้อง
kubectl get pods -n chuaikan -l app=chuaikan-api \
  --show-labels | grep rollouts-pod-template-hash

# Promote canary ไปต่อ step ถัดไป
kubectl argo rollouts promote chuaikan-api -n chuaikan

# Abort canary (rollback ทันที)
kubectl argo rollouts abort chuaikan-api -n chuaikan

# ดู analysis run
kubectl get analysisrun -n chuaikan
kubectl describe analysisrun -n chuaikan <analysis-run-name>
```

---

## ✅ Checklist

- [ ] Argo Rollouts canary steps: 5% → 25% → 50% → 100%
- [ ] Canary analysis: error rate < 1%
- [ ] Canary analysis: P99 latency ไม่เพิ่ม > 20%
- [ ] Auto-rollback เมื่อ analysis fail
- [ ] Canary service แยกจาก stable service
- [ ] Metrics scrape แยกสำหรับ canary vs stable
- [ ] Feature flags ใช้งานในโค้ดได้
- [ ] DB migration canary script ทำงานได้
- [ ] Grafana dashboard แสดง traffic split และ error rate เปรียบเทียบ
- [ ] Runbook สำหรับ manual promote/rollback

---

## 🔗 References

- [Argo Rollouts Canary](https://argoproj.github.io/argo-rollouts/features/canary/)
- [Automated Canary Analysis](https://argoproj.github.io/argo-rollouts/features/analysis/)

---
*Part 081 | Road to 1,000,000 Users/Day | chuaikan.com*
