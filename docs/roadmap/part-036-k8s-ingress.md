# Part 036: Ingress Controller (Nginx)

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 351-360
> **เวลาโดยประมาณ:** 3 ชั่วโมง
> **Prerequisites:** Part 035 (Persistent Volumes), Part 032 (K8s Installation with MetalLB)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ติดตั้ง Nginx Ingress Controller ด้วย Helm
- สร้าง Ingress manifest สำหรับ www.chuaikan.com และ api.chuaikan.com
- ตั้งค่า SSL/TLS ด้วย cert-manager และ Let's Encrypt
- ใช้ Ingress annotations สำหรับ rate limiting และ real client IP
- ตั้งค่า path-based routing
- รองรับ WebSocket ใน Ingress

---

## 📖 ทฤษฎีและแนวคิด

### Step 351 — Ingress Architecture

```
Internet
    │
    ▼
[MetalLB LoadBalancer IP: 192.168.1.200]
    │
    ▼
[Nginx Ingress Controller Pod]
    │
    ├── www.chuaikan.com ──────► nextjs-service:80
    ├── api.chuaikan.com ──────► api-service:4000
    └── api.chuaikan.com/ws ───► websocket-service:4001
```

**Ingress Controller** คือ reverse proxy ที่:
- รับ traffic จาก internet
- route ไปยัง service ที่ถูกต้องตาม hostname/path
- handle TLS termination
- ทำ rate limiting, auth, logging

### Step 352 — ทำไมต้องใช้ Ingress แทน NodePort/LoadBalancer

```
ปัญหาของ LoadBalancer service:
- แต่ละ service ต้องมี external IP ของตัวเอง
- ถ้ามี 10 services = 10 external IPs = แพง!

Ingress แก้ปัญหา:
- external IP แค่ 1 IP สำหรับ Ingress Controller
- route ต่อด้วย hostname/path rules
- จัดการ TLS ที่ Ingress layer เดียว
```

---

## ⚙️ Environment Setup

### ติดตั้ง Helm

```bash
# ติดตั้ง Helm
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | \
  sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm -y

# ตรวจสอบ
helm version
```

---

## 🛠️ Step-by-Step Implementation

### Step 353 — ติดตั้ง Nginx Ingress Controller ด้วย Helm

```bash
# เพิ่ม Helm repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# สร้าง namespace
kubectl create namespace ingress-nginx

# สร้าง values file สำหรับ production
cat > nginx-ingress-values.yaml <<EOF
controller:
  replicaCount: 2
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1

  service:
    type: LoadBalancer
    annotations:
      metallb.universe.tf/address-pool: chuaikan-pool

  config:
    use-real-ip: "true"
    real-ip-header: "X-Forwarded-For"
    forwarded-for-header: "X-Forwarded-For"
    proxy-body-size: "50m"
    proxy-connect-timeout: "15"
    proxy-send-timeout: "600"
    proxy-read-timeout: "600"
    keepalive: "200"
    worker-processes: "auto"
    log-format-upstream: >
      {"time": "$time_iso8601",
       "status": $status,
       "method": "$request_method",
       "uri": "$uri",
       "host": "$host",
       "remote_addr": "$remote_addr",
       "http_referrer": "$http_referer",
       "upstream_addr": "$upstream_addr",
       "request_time": "$request_time"}

  resources:
    requests:
      cpu: 100m
      memory: 90Mi
    limits:
      cpu: 1000m
      memory: 512Mi

  metrics:
    enabled: true
    serviceMonitor:
      enabled: true

defaultBackend:
  enabled: true
  image:
    repository: registry.k8s.io/defaultbackend-amd64
    tag: "1.5"
EOF

# ติดตั้ง
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --values nginx-ingress-values.yaml

# ตรวจสอบ
kubectl get pods -n ingress-nginx
kubectl get service -n ingress-nginx

# รอ LoadBalancer IP
kubectl get service ingress-nginx-controller -n ingress-nginx -w
# NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)
# ingress-nginx-controller   LoadBalancer   10.107.58.23   192.168.1.200   80:31080/TCP,443:31443/TCP
```

### Step 354 — ติดตั้ง cert-manager สำหรับ Let's Encrypt

```bash
# เพิ่ม Helm repo
helm repo add jetstack https://charts.jetstack.io
helm repo update

# ติดตั้ง cert-manager
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.14.4 \
  --set installCRDs=true

# ตรวจสอบ
kubectl get pods -n cert-manager

# สร้าง ClusterIssuer สำหรับ Let's Encrypt (production)
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: admin@chuaikan.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - http01:
        ingress:
          ingressClassName: nginx
EOF

# ClusterIssuer สำหรับ staging (ทดสอบก่อนใช้ production)
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: admin@chuaikan.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging-key
    solvers:
    - http01:
        ingress:
          ingressClassName: nginx
EOF

# ตรวจสอบ
kubectl get clusterissuer
```

### Step 355 — Ingress Manifest สำหรับ chuaikan.com

```yaml
# ingress-chuaikan.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: chuaikan-ingress
  namespace: chuaikan-production
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: "letsencrypt-prod"

    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "20"

    # Real IP
    nginx.ingress.kubernetes.io/use-real-ip: "true"
    nginx.ingress.kubernetes.io/real-ip-header: "X-Forwarded-For"

    # Proxy settings
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "15"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"

    # HTTPS redirect
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # HSTS
    nginx.ingress.kubernetes.io/hsts: "true"
    nginx.ingress.kubernetes.io/hsts-max-age: "31536000"
    nginx.ingress.kubernetes.io/hsts-include-subdomains: "true"

spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - www.chuaikan.com
    - chuaikan.com
    secretName: chuaikan-tls-cert
  rules:
  # www.chuaikan.com → Next.js
  - host: www.chuaikan.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nextjs-service
            port:
              number: 3000

  # chuaikan.com → redirect to www.chuaikan.com
  - host: chuaikan.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nextjs-service
            port:
              number: 3000
```

```yaml
# ingress-api.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: chuaikan-production
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: "letsencrypt-prod"

    # API rate limiting (เข้มงวดกว่า frontend)
    nginx.ingress.kubernetes.io/limit-rps: "50"
    nginx.ingress.kubernetes.io/limit-connections: "10"
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "5"

    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://www.chuaikan.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-headers: "Authorization, Content-Type"

    # WebSocket support
    nginx.ingress.kubernetes.io/proxy-http-version: "1.1"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";

    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"

spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.chuaikan.com
    secretName: api-chuaikan-tls-cert
  rules:
  - host: api.chuaikan.com
    http:
      paths:
      # REST API
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 4000

      # WebSocket endpoint
      - path: /ws
        pathType: Prefix
        backend:
          service:
            name: websocket-service
            port:
              number: 4001

      # Health check (no rate limit needed)
      - path: /health
        pathType: Exact
        backend:
          service:
            name: api-service
            port:
              number: 4000
```

```bash
kubectl apply -f ingress-chuaikan.yaml
kubectl apply -f ingress-api.yaml

# ตรวจสอบ Ingress
kubectl get ingress -n chuaikan-production
kubectl describe ingress chuaikan-ingress -n chuaikan-production
```

### Step 356 — ตรวจสอบ SSL Certificate

```bash
# ดู Certificate ที่ cert-manager สร้างให้
kubectl get certificate -n chuaikan-production
# NAME                  READY   SECRET                AGE
# chuaikan-tls-cert     True    chuaikan-tls-cert     2m

# ดูรายละเอียด Certificate
kubectl describe certificate chuaikan-tls-cert -n chuaikan-production

# ดู CertificateRequest
kubectl get certificaterequest -n chuaikan-production

# ดู Challenge (ACME challenge สำหรับ Let's Encrypt)
kubectl get challenge -n chuaikan-production
```

### Step 357 — Path-Based Routing

```yaml
# ingress-path-routing.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-routing-ingress
  namespace: chuaikan-production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - host: www.chuaikan.com
    http:
      paths:
      # Next.js main app
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nextjs-service
            port:
              number: 3000

      # API proxy (path rewrite: /api/v1/users → /v1/users)
      - path: /api(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: api-service
            port:
              number: 4000

      # Static files (ถ้าใช้ CDN ไม่ต้อง)
      - path: /uploads
        pathType: Prefix
        backend:
          service:
            name: minio-service
            port:
              number: 9000
```

### Step 358 — WebSocket Support

```yaml
# ingress-websocket.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: websocket-ingress
  namespace: chuaikan-production
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
      proxy_cache_bypass $http_upgrade;
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - ws.chuaikan.com
    secretName: ws-chuaikan-tls
  rules:
  - host: ws.chuaikan.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: websocket-service
            port:
              number: 4001
```

### Step 359 — Rate Limiting แบบ Advanced

```yaml
# ingress-rate-limit.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-rate-limited
  namespace: chuaikan-production
  annotations:
    # Rate limit: 10 requests per second
    nginx.ingress.kubernetes.io/limit-rps: "10"

    # Rate limit: max 5 concurrent connections
    nginx.ingress.kubernetes.io/limit-connections: "5"

    # Burst multiplier (อนุญาต burst 3x)
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "3"

    # Rate limit by IP (whitelist IPs ไม่ต้อง rate limit)
    nginx.ingress.kubernetes.io/limit-whitelist: "10.0.0.0/8,192.168.0.0/16"

    # Custom error page เมื่อ rate limit
    nginx.ingress.kubernetes.io/custom-http-errors: "429,503"
    nginx.ingress.kubernetes.io/default-backend: rate-limit-backend
spec:
  ingressClassName: nginx
  rules:
  - host: api.chuaikan.com
    http:
      paths:
      - path: /v1/auth
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 4000
```

### Step 360 — ทดสอบ Ingress

```bash
# ทดสอบ HTTP redirect ไป HTTPS
curl -I http://www.chuaikan.com
# HTTP/1.1 308 Permanent Redirect
# Location: https://www.chuaikan.com/

# ทดสอบ HTTPS
curl -I https://www.chuaikan.com
# HTTP/2 200

# ทดสอบ API
curl https://api.chuaikan.com/health
# {"status":"ok","timestamp":"2024-01-15T10:00:00Z"}

# ตรวจสอบ SSL certificate
echo | openssl s_client -connect www.chuaikan.com:443 -servername www.chuaikan.com 2>/dev/null | \
  openssl x509 -noout -dates
# notBefore=Jan 15 00:00:00 2024 GMT
# notAfter=Apr 15 00:00:00 2024 GMT

# ทดสอบ Rate Limiting
for i in {1..20}; do
  curl -s -o /dev/null -w "%{http_code}\n" https://api.chuaikan.com/v1/test
done
# 200 200 200 ... 429 429 (เมื่อเกิน limit)

# ดู Nginx logs
kubectl logs -l app.kubernetes.io/name=ingress-nginx \
  -n ingress-nginx \
  --tail=50 \
  -f
```

---

## 🔧 Configuration Files

### Nginx ConfigMap สำหรับ Custom Settings

```yaml
# nginx-custom-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
data:
  # Gzip compression
  use-gzip: "true"
  gzip-level: "5"
  gzip-types: "text/plain application/json application/javascript text/css"

  # Security headers
  add-headers: "ingress-nginx/custom-headers"

  # Timeout
  proxy-connect-timeout: "15"
  proxy-send-timeout: "600"
  proxy-read-timeout: "600"

  # Buffer sizes
  proxy-buffer-size: "16k"
  proxy-buffers-number: "4"

  # Log format
  log-format-upstream: >
    $remote_addr - $request_id [$time_local] "$request"
    $status $body_bytes_sent "$http_referer" "$http_user_agent"
    $request_length $request_time [$proxy_upstream_name]
    [$proxy_alternative_upstream_name] $upstream_addr
    $upstream_response_length $upstream_response_time $upstream_status
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: custom-headers
  namespace: ingress-nginx
data:
  X-Frame-Options: SAMEORIGIN
  X-Content-Type-Options: nosniff
  X-XSS-Protection: "1; mode=block"
  Referrer-Policy: strict-origin-when-cross-origin
  Permissions-Policy: camera=(), microphone=(), geolocation=()
```

---

## ❌ Common Errors & Solutions

### Error: `503 Service Temporarily Unavailable`
```bash
# สาเหตุ: Ingress ชี้ไป service ที่ไม่มี pod พร้อม
kubectl get endpoints nextjs-service -n chuaikan-production
# ถ้าไม่มี ENDPOINTS ต้องตรวจสอบ pods
kubectl get pods -l app=nextjs -n chuaikan-production
```

### Error: `Certificate not ready`
```bash
# ดู cert-manager events
kubectl describe certificate chuaikan-tls-cert -n chuaikan-production
kubectl describe certificaterequest -n chuaikan-production
kubectl describe challenge -n chuaikan-production

# ตรวจสอบว่า DNS ชี้มาที่ Ingress IP ถูกต้อง
dig www.chuaikan.com
# ควรได้ IP ของ MetalLB LoadBalancer
```

### Error: `413 Request Entity Too Large`
```bash
# แก้: เพิ่ม proxy-body-size
kubectl annotate ingress chuaikan-ingress \
  nginx.ingress.kubernetes.io/proxy-body-size=100m \
  -n chuaikan-production
```

---

## ✅ Checklist

- [ ] **Step 351**: เข้าใจ Ingress architecture และทำไมถึงดีกว่า multiple LoadBalancers
- [ ] **Step 352**: อธิบายความแตกต่าง Ingress vs LoadBalancer vs NodePort ได้
- [ ] **Step 353**: ติดตั้ง Nginx Ingress Controller ด้วย Helm สำเร็จ, มี external IP
- [ ] **Step 354**: ติดตั้ง cert-manager และสร้าง ClusterIssuer สำหรับ Let's Encrypt สำเร็จ
- [ ] **Step 355**: สร้าง Ingress สำหรับ www.chuaikan.com และ api.chuaikan.com สำเร็จ
- [ ] **Step 356**: SSL certificate จาก Let's Encrypt ถูกออกให้ (READY=True)
- [ ] **Step 357**: ตั้งค่า path-based routing สำเร็จ
- [ ] **Step 358**: WebSocket ทำงานผ่าน Ingress ได้
- [ ] **Step 359**: Rate limiting ทำงาน ส่ง 429 เมื่อ exceed limit
- [ ] **Step 360**: ทดสอบ HTTPS, rate limiting, Nginx logs

---

## 🔗 References

- [Nginx Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
- [cert-manager Documentation](https://cert-manager.io/docs/)
- [Ingress Annotations](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/)
- [WebSocket Support](https://kubernetes.github.io/ingress-nginx/user-guide/miscellaneous/#websockets)

---

*Part 036 | Road to 1,000,000 Users/Day | chuaikan.com*
