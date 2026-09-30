# Part 084: Service Mesh ด้วย Istio
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Professional
> **Steps:** 831–840
> **เวลาโดยประมาณ:** 8 ชั่วโมง
> **Prerequisites:** Part 031 (Kubernetes), Part 079 (Latency)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- Service mesh concepts: data plane (Envoy) + control plane (Istiod)
- Istio installation บน Kubernetes 1.22+
- Traffic management: VirtualService, DestinationRule
- mTLS ระหว่าง services (automatic)
- Observability: distributed tracing ด้วย Jaeger, metrics ด้วย Prometheus
- Circuit breaker ด้วย Istio
- Retry policy configuration
- Traffic mirroring สำหรับ dark launching
- Istio resource overhead

---

## 📖 ทฤษฎีและแนวคิด

### Service Mesh Architecture

```
┌─────────────────────────────────────────────────┐
│                 Control Plane                   │
│  ┌──────────────────────────────────────────┐  │
│  │              Istiod                      │  │
│  │  (Pilot + Citadel + Galley combined)     │  │
│  └──────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
              ↕ xDS (config distribution)
┌──────────────────────────────────────────────────┐
│                  Data Plane                      │
│                                                  │
│  ┌──────────────────┐    ┌──────────────────┐   │
│  │   App Pod A      │    │   App Pod B      │   │
│  │ ┌────────────┐   │    │ ┌────────────┐   │   │
│  │ │ App        │   │    │ │ App        │   │   │
│  │ ├────────────┤   │    │ ├────────────┤   │   │
│  │ │ Envoy      │◄──┼mTLS┼►│ Envoy      │   │   │
│  │ │ (sidecar)  │   │    │ │ (sidecar)  │   │   │
│  │ └────────────┘   │    │ └────────────┘   │   │
│  └──────────────────┘    └──────────────────┘   │
└──────────────────────────────────────────────────┘
```

---

## ⚙️ Environment Setup

### Step 831: ติดตั้ง Istio

```bash
# ดาวน์โหลด Istio
ISTIO_VERSION="1.22.0"
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=${ISTIO_VERSION} sh -
export PATH="$PATH:$PWD/istio-${ISTIO_VERSION}/bin"

# ตรวจสอบ prerequisites
istioctl x precheck

# Install Istio ด้วย demo profile (development)
istioctl install --set profile=demo -y

# Install ด้วย production profile (แนะนำสำหรับ prod)
istioctl install \
  --set profile=default \
  --set values.gateways.istio-ingressgateway.type=LoadBalancer \
  --set values.pilot.resources.requests.memory=512Mi \
  --set values.pilot.resources.requests.cpu=200m \
  -y

# Enable sidecar injection สำหรับ namespace
kubectl label namespace chuaikan istio-injection=enabled

# ตรวจสอบ
kubectl get pods -n istio-system
```

---

## 🛠️ Step-by-Step Implementation

### Step 832: VirtualService และ DestinationRule

```yaml
# k8s/istio/api-virtual-service.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: chuaikan-api
  namespace: chuaikan
spec:
  hosts:
    - chuaikan-api
    - api.chuaikan.com
  gateways:
    - istio-system/chuaikan-gateway
    - mesh  # ใช้กับ internal traffic ด้วย
  http:
    # Route ตาม header (canary)
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: chuaikan-api
            subset: v2
          weight: 100

    # Default route: 95% v1, 5% v2
    - route:
        - destination:
            host: chuaikan-api
            subset: v1
          weight: 95
        - destination:
            host: chuaikan-api
            subset: v2
          weight: 5

      # Retry policy
      retries:
        attempts: 3
        perTryTimeout: 3s
        retryOn: "5xx,reset,connect-failure,retriable-4xx"

      # Timeout
      timeout: 10s

      # Fault injection สำหรับ testing (ปิดใน production)
      # fault:
      #   delay:
      #     percentage: {value: 1}
      #     fixedDelay: 5s

---
# k8s/istio/api-destination-rule.yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: chuaikan-api
  namespace: chuaikan
spec:
  host: chuaikan-api
  trafficPolicy:
    # Connection pool settings
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 5s
      http:
        h2UpgradePolicy: UPGRADE  # Upgrade ไป HTTP/2
        http2MaxRequests: 1000
        maxRequestsPerConnection: 100
        maxRetries: 3

    # Circuit breaker
    outlierDetection:
      consecutiveGatewayErrors: 5
      consecutiveLocalOriginFailures: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 50

    # mTLS: STRICT = บังคับ mTLS ทุก connection
    tls:
      mode: ISTIO_MUTUAL

  subsets:
    - name: v1
      labels:
        version: v1
      trafficPolicy:
        connectionPool:
          http:
            h2UpgradePolicy: UPGRADE

    - name: v2
      labels:
        version: v2
```

### Step 833: mTLS Policy

```yaml
# k8s/istio/mtls-policy.yaml
# กำหนดให้ namespace ต้องใช้ mTLS เท่านั้น

apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: chuaikan
spec:
  mtls:
    mode: STRICT  # ปฏิเสธ plaintext ทั้งหมด

---
# อนุญาตให้ health checks จาก K8s ผ่านได้
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: allow-health-checks
  namespace: chuaikan
spec:
  selector:
    matchLabels:
      app: chuaikan-api
  mtls:
    mode: STRICT
  portLevelMtls:
    3000:
      mode: PERMISSIVE  # health check port ไม่ต้อง mTLS
```

### Step 834: Circuit Breaker Testing

```bash
# ทดสอบ circuit breaker ด้วย fault injection
cat << 'EOF' > /tmp/test-fault-injection.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: chuaikan-api-fault-test
  namespace: chuaikan
spec:
  hosts:
    - chuaikan-api
  http:
    - fault:
        abort:
          percentage:
            value: 20  # 20% ของ requests return 503
          httpStatus: 503
    - route:
        - destination:
            host: chuaikan-api
            subset: v1
EOF

kubectl apply -f /tmp/test-fault-injection.yaml

# ทดสอบ
for i in $(seq 1 20); do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" "http://api.chuaikan.com/health")
  echo "Request $i: $STATUS"
done

# ลบ fault injection หลังทดสอบ
kubectl delete -f /tmp/test-fault-injection.yaml
```

### Step 835: Distributed Tracing ด้วย Jaeger

```bash
# ติดตั้ง Jaeger
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.22/samples/addons/jaeger.yaml

# ตรวจสอบ
kubectl get pods -n istio-system | grep jaeger

# Forward port สำหรับดู UI
kubectl port-forward -n istio-system svc/tracing 16686:80
# เปิด http://localhost:16686
```

```typescript
// src/middleware/tracing.ts
// Propagate trace headers เพื่อให้ Istio track ได้

const TRACE_HEADERS = [
  'x-request-id',
  'x-b3-traceid',
  'x-b3-spanid',
  'x-b3-parentspanid',
  'x-b3-sampled',
  'x-b3-flags',
  'x-ot-span-context'
];

export function propagateTraceHeaders(
  req: Request,
  outgoingHeaders: Record<string, string>
): Record<string, string> {
  const headers = { ...outgoingHeaders };

  for (const header of TRACE_HEADERS) {
    const value = req.headers[header] as string;
    if (value) {
      headers[header] = value;
    }
  }

  return headers;
}

// Middleware สำหรับ propagate trace headers
export function traceMiddleware(req: Request, res: Response, next: NextFunction) {
  // เก็บ trace headers ไว้ใน request context
  req.traceHeaders = {};
  for (const header of TRACE_HEADERS) {
    const value = req.headers[header] as string;
    if (value) {
      req.traceHeaders[header] = value;
    }
  }
  next();
}
```

### Step 836: Traffic Mirroring (Dark Launch)

```yaml
# k8s/istio/traffic-mirror.yaml
# Mirror 10% ของ traffic ไปยัง v2 โดยไม่ส่ง response กลับ user

apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: chuaikan-api-mirror
  namespace: chuaikan
spec:
  hosts:
    - chuaikan-api
  http:
    - route:
        - destination:
            host: chuaikan-api
            subset: v1
          weight: 100
      # Mirror 10% ไป v2 (async, ไม่กระทบ user)
      mirror:
        host: chuaikan-api
        subset: v2
      mirrorPercentage:
        value: 10.0
```

### Step 837: Istio Resource Overhead

```yaml
# k8s/istio/resource-limits.yaml
# ตั้ง resource limits สำหรับ Envoy sidecar

apiVersion: v1
kind: ConfigMap
metadata:
  name: istio-sidecar-injector
  namespace: istio-system
data:
  values: |
    {
      "global": {
        "proxy": {
          "resources": {
            "requests": {
              "cpu": "10m",
              "memory": "40Mi"
            },
            "limits": {
              "cpu": "200m",
              "memory": "256Mi"
            }
          }
        }
      }
    }
```

```bash
# ตรวจสอบ resource usage ของ Envoy sidecars
kubectl top pods -n chuaikan --containers | grep istio-proxy

# Example output:
# chuaikan-api-xxx    app          80m   150Mi
# chuaikan-api-xxx    istio-proxy  15m    45Mi  ← overhead ~15-20% CPU, 45MB RAM
```

---

## 🧪 Testing

```bash
# ตรวจสอบ mTLS ทำงาน
istioctl authn tls-check chuaikan-api.chuaikan.svc.cluster.local

# ตรวจสอบ circuit breaker
istioctl proxy-config cluster chuaikan-api-xxxx.chuaikan | grep chuaikan-api

# ดู Envoy metrics
kubectl exec -n chuaikan chuaikan-api-xxxx -c istio-proxy -- \
  curl -s localhost:15090/stats | grep "upstream_cx_overflow"

# ดู distributed traces
kubectl port-forward -n istio-system svc/tracing 16686:80
# จากนั้นเปิด http://localhost:16686 และเลือก Service: chuaikan-api

# ตรวจสอบ retry behavior
kubectl exec -n chuaikan chuaikan-api-xxxx -c istio-proxy -- \
  curl -s localhost:15090/stats | grep "retry"
```

---

## ✅ Checklist

- [ ] Istio ติดตั้งแล้ว (istiod running)
- [ ] Namespace chuaikan มี istio-injection=enabled
- [ ] ทุก pod มี envoy sidecar (2 containers per pod)
- [ ] mTLS STRICT mode เปิดใน namespace chuaikan
- [ ] VirtualService กำหนด retry policy
- [ ] DestinationRule กำหนด circuit breaker
- [ ] Jaeger ติดตั้งและ trace ทำงานได้
- [ ] Traffic mirroring ทดสอบแล้ว
- [ ] Prometheus scrape Istio metrics ได้
- [ ] Resource overhead ≤ 20% CPU, ≤ 100MB RAM ต่อ pod

---

## 🔗 References

- [Istio Documentation](https://istio.io/latest/docs/)
- [Istio Traffic Management](https://istio.io/latest/docs/concepts/traffic-management/)
- [Envoy Proxy](https://www.envoyproxy.io/docs/)

---
*Part 084 | Road to 1,000,000 Users/Day | chuaikan.com*
