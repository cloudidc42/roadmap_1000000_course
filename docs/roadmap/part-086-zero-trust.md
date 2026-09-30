# Part 086: Zero Trust Security Model
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Professional
> **Steps:** 851–860
> **เวลาโดยประมาณ:** 8 ชั่วโมง
> **Prerequisites:** Part 084 (Service Mesh), Part 007 (Cloudflare)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- Zero trust principles: never trust, always verify
- Identity-aware proxy ด้วย Cloudflare Access
- Service-to-service authentication (mTLS + SPIFFE/SPIRE)
- Network micro-segmentation ด้วย K8s NetworkPolicy
- Device trust สำหรับ admin access
- Zero trust สำหรับ remote work (Cloudflare Access + WARP)
- Privileged Access Management (PAM)
- Audit logging สำหรับ privileged actions

---

## 📖 ทฤษฎีและแนวคิด

### Zero Trust คืออะไร?

"Never trust, always verify" — ไม่เชื่อถือใครเพียงเพราะอยู่ใน network เดียวกัน

```
Traditional Security (Perimeter-based):
  Internet → Firewall → [Trusted Network] → Services
  ถ้าเข้ามาใน perimeter ได้ = เชื่อถือ

Zero Trust:
  Internet → [Identity + Device + Context] → Services
  ทุก request ต้องพิสูจน์ตัวเอง ไม่ว่าจะมาจากไหน
```

### Zero Trust Components สำหรับ chuaikan.com

1. **Identity**: SSO ด้วย Google/Okta
2. **Device**: Device must be managed/compliant
3. **Network**: mTLS ระหว่าง services
4. **Application**: Per-application access policy
5. **Data**: Encryption at rest + in transit

---

## 🛠️ Step-by-Step Implementation

### Step 851: Cloudflare Access Setup

```bash
# ติดตั้ง cloudflared
wget -q https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb

# Login ไป Cloudflare
cloudflared tunnel login

# สร้าง tunnel สำหรับ internal tools
cloudflared tunnel create chuaikan-internal

# ดู tunnel credentials
ls ~/.cloudflared/
```

```yaml
# /etc/cloudflared/config.yml
# Cloudflare Tunnel config สำหรับ internal services

tunnel: chuaikan-internal-tunnel-id
credentials-file: /etc/cloudflared/.cloudflared/chuaikan-internal.json

ingress:
  # Grafana (monitoring)
  - hostname: grafana.internal.chuaikan.com
    service: http://grafana.monitoring.svc.cluster.local:3000

  # ArgoCD (deployment)
  - hostname: argocd.internal.chuaikan.com
    service: http://argocd-server.argocd.svc.cluster.local:80

  # Kubernetes Dashboard
  - hostname: k8s.internal.chuaikan.com
    service: https://kubernetes-dashboard.kubernetes-dashboard.svc.cluster.local:443
    originRequest:
      noTLSVerify: true

  # Vault (secrets)
  - hostname: vault.internal.chuaikan.com
    service: http://vault.vault.svc.cluster.local:8200

  # Default: 404
  - service: http_status:404
```

```bash
# Deploy cloudflared ใน Kubernetes
kubectl create secret generic cloudflared-credentials \
  --from-file=credentials.json=/etc/cloudflared/chuaikan-internal.json \
  -n cloudflare
```

```yaml
# k8s/cloudflared/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
  namespace: cloudflare
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cloudflared
  template:
    metadata:
      labels:
        app: cloudflared
    spec:
      containers:
        - name: cloudflared
          image: cloudflare/cloudflared:latest
          args:
            - tunnel
            - --config
            - /etc/cloudflared/config.yml
            - run
          volumeMounts:
            - name: config
              mountPath: /etc/cloudflared
              readOnly: true
            - name: credentials
              mountPath: /etc/cloudflared/.cloudflared
              readOnly: true
          resources:
            requests:
              memory: "64Mi"
              cpu: "50m"
            limits:
              memory: "256Mi"
              cpu: "200m"
      volumes:
        - name: config
          configMap:
            name: cloudflared-config
        - name: credentials
          secret:
            secretName: cloudflared-credentials
```

### Step 852: Cloudflare Access Policy

```bash
# ตั้งค่า Access Application ผ่าน Cloudflare API

export CF_API_TOKEN="your_api_token"
export CF_ACCOUNT_ID="your_account_id"

# สร้าง Access Application สำหรับ Grafana
curl -X POST "https://api.cloudflare.com/client/v4/accounts/${CF_ACCOUNT_ID}/access/apps" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Grafana - chuaikan monitoring",
    "domain": "grafana.internal.chuaikan.com",
    "type": "self_hosted",
    "session_duration": "8h",
    "auto_redirect_to_identity": true,
    "http_only_cookie_attribute": true,
    "same_site_cookie_attribute": "strict"
  }'

# สร้าง Access Policy: เฉพาะ @chuaikan.com email
curl -X POST "https://api.cloudflare.com/client/v4/accounts/${CF_ACCOUNT_ID}/access/apps/APP_ID/policies" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "chuaikan team",
    "decision": "allow",
    "include": [
      {
        "email_domain": {"domain": "chuaikan.com"}
      }
    ],
    "require": [
      {
        "email_domain": {"domain": "chuaikan.com"}
      }
    ],
    "exclude": []
  }'
```

### Step 853: Kubernetes NetworkPolicy

```yaml
# k8s/network-policies/default-deny.yaml
# Deny all ingress/egress by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: chuaikan
spec:
  podSelector: {}  # Apply ทุก pod
  policyTypes:
    - Ingress
    - Egress

---
# อนุญาต DNS egress (ต้องการสำหรับ service discovery)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: chuaikan
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53

---
# อนุญาต API → Database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
  namespace: chuaikan
spec:
  podSelector:
    matchLabels:
      role: database
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: chuaikan-api
      ports:
        - protocol: TCP
          port: 5432

---
# อนุญาต API → Redis
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-redis
  namespace: chuaikan
spec:
  podSelector:
    matchLabels:
      role: cache
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: chuaikan-api
      ports:
        - protocol: TCP
          port: 6379

---
# อนุญาต Ingress Controller → API
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-api
  namespace: chuaikan
spec:
  podSelector:
    matchLabels:
      app: chuaikan-api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 3000
```

### Step 854: SPIFFE/SPIRE สำหรับ Service Identity

```bash
# ติดตั้ง SPIRE ใน Kubernetes
kubectl apply -f https://raw.githubusercontent.com/spiffe/spire/main/support/k8s/quickstart/spire-namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/spiffe/spire/main/support/k8s/quickstart/server-account.yaml
kubectl apply -f https://raw.githubusercontent.com/spiffe/spire/main/support/k8s/quickstart/spire-bundle-configmap.yaml
kubectl apply -f https://raw.githubusercontent.com/spiffe/spire/main/support/k8s/quickstart/server-cluster-role.yaml
kubectl apply -f https://raw.githubusercontent.com/spiffe/spire/main/support/k8s/quickstart/server-configmap.yaml
kubectl apply -f https://raw.githubusercontent.com/spiffe/spire/main/support/k8s/quickstart/server-statefulset.yaml
kubectl apply -f https://raw.githubusercontent.com/spiffe/spire/main/support/k8s/quickstart/server-service.yaml
kubectl apply -f https://raw.githubusercontent.com/spiffe/spire/main/support/k8s/quickstart/agent-account.yaml
kubectl apply -f https://raw.githubusercontent.com/spiffe/spire/main/support/k8s/quickstart/agent-cluster-role.yaml
kubectl apply -f https://raw.githubusercontent.com/spiffe/spire/main/support/k8s/quickstart/agent-configmap.yaml
kubectl apply -f https://raw.githubusercontent.com/spiffe/spire/main/support/k8s/quickstart/agent-daemonset.yaml

# ลงทะเบียน service entries
kubectl exec -n spire spire-server-0 -- \
  /opt/spire/bin/spire-server entry create \
  -spiffeID spiffe://chuaikan.com/api \
  -parentID spiffe://chuaikan.com/ns/chuaikan/sa/default \
  -selector k8s:ns:chuaikan \
  -selector k8s:sa:default \
  -selector k8s:pod-label:app:chuaikan-api
```

### Step 855: Privileged Access Management

```bash
# scripts/pam-setup.sh
# Privileged Access Management สำหรับ Production access

# 1. Just-in-time (JIT) access ด้วย AWS IAM
# สร้าง IAM Role สำหรับ temporary admin access

aws iam create-role \
  --role-name ChuaikanTempAdmin \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "TEMP_ACCESS_TOKEN"
        },
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    }]
  }'

# 2. ตั้ง policy สำหรับ temp admin (read-only + specific write)
aws iam put-role-policy \
  --role-name ChuaikanTempAdmin \
  --policy-name TempAdminPolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "ReadOnly",
        "Effect": "Allow",
        "Action": ["ec2:Describe*", "eks:Describe*", "rds:Describe*"],
        "Resource": "*"
      },
      {
        "Sid": "LimitedWrite",
        "Effect": "Allow",
        "Action": [
          "eks:ListNodegroups",
          "eks:UpdateNodegroupConfig"
        ],
        "Resource": "arn:aws:eks:ap-southeast-1:123456789012:cluster/chuaikan-*"
      }
    ]
  }'
```

### Step 856: Audit Logging

```typescript
// src/middleware/audit.middleware.ts
// Audit log สำหรับ privileged actions

import { Request, Response, NextFunction } from 'express';

interface AuditEvent {
  timestamp: string;
  userId: string;
  userEmail: string;
  action: string;
  resource: string;
  resourceId?: string;
  ip: string;
  userAgent: string;
  result: 'success' | 'failure';
  details?: Record<string, unknown>;
}

const PRIVILEGED_ACTIONS = [
  { method: 'DELETE', pathPattern: /^\/admin\// },
  { method: 'POST', pathPattern: /^\/admin\/users\/.*\/ban/ },
  { method: 'PUT', pathPattern: /^\/admin\/settings/ },
  { method: 'DELETE', pathPattern: /^\/v1\/posts\// },
];

export function auditLog(req: Request, res: Response, next: NextFunction) {
  const isPrivileged = PRIVILEGED_ACTIONS.some(a =>
    req.method === a.method && a.pathPattern.test(req.path)
  );

  if (!isPrivileged) return next();

  const startTime = Date.now();

  res.on('finish', () => {
    const event: AuditEvent = {
      timestamp: new Date().toISOString(),
      userId: req.user?.id || 'anonymous',
      userEmail: req.user?.email || 'unknown',
      action: `${req.method} ${req.path}`,
      resource: req.path.split('/')[2] || 'unknown',
      resourceId: req.params.id,
      ip: req.ip || req.socket.remoteAddress || 'unknown',
      userAgent: req.headers['user-agent'] || 'unknown',
      result: res.statusCode < 400 ? 'success' : 'failure',
      details: {
        statusCode: res.statusCode,
        duration: Date.now() - startTime,
        body: req.method !== 'GET' ? sanitizeBody(req.body) : undefined
      }
    };

    // ส่ง audit event ไป centralized logging
    sendAuditEvent(event);
  });

  next();
}

function sanitizeBody(body: unknown): unknown {
  if (!body || typeof body !== 'object') return body;
  const sensitive = ['password', 'token', 'secret', 'credit_card'];
  return Object.fromEntries(
    Object.entries(body as Record<string, unknown>).map(([k, v]) =>
      sensitive.some(s => k.toLowerCase().includes(s))
        ? [k, '[REDACTED]']
        : [k, v]
    )
  );
}

async function sendAuditEvent(event: AuditEvent) {
  // ส่งไป CloudWatch Logs + S3 สำหรับ long-term retention
  console.log(JSON.stringify({ type: 'AUDIT', ...event }));
}
```

---

## 🧪 Testing

```bash
# ทดสอบ NetworkPolicy
# ควร fail (no direct access to DB from unrelated pod)
kubectl run test-pod --image=busybox -n chuaikan --rm -it -- \
  wget -T 3 chuaikan-postgres:5432
# ควรได้ "Connection refused" หรือ timeout

# ทดสอบ Cloudflare Access
curl -v "https://grafana.internal.chuaikan.com/" \
  -H "Cookie: CF_Authorization=invalid_token"
# ควรได้ 403

# ตรวจสอบ mTLS
istioctl authn tls-check chuaikan-api.chuaikan.svc.cluster.local
```

---

## ✅ Checklist

- [ ] Cloudflare Access ปกป้อง internal tools ทั้งหมด
- [ ] SSO (Google Workspace) ทำงานกับ Cloudflare Access
- [ ] NetworkPolicy: default-deny-all ใน chuaikan namespace
- [ ] NetworkPolicy: อนุญาตเฉพาะ service-to-service ที่จำเป็น
- [ ] mTLS (Istio) ระหว่าง services ทั้งหมด
- [ ] SPIFFE/SPIRE ออก service identity certificates
- [ ] JIT admin access (ไม่มี permanent admin credentials)
- [ ] Audit log สำหรับ privileged actions ทุกครั้ง
- [ ] Audit logs ส่งไป S3 + SIEM
- [ ] Regular access review ทุก 3 เดือน

---

## 🔗 References

- [Cloudflare Zero Trust](https://developers.cloudflare.com/cloudflare-one/)
- [SPIFFE/SPIRE](https://spiffe.io/docs/latest/spire-about/)
- [Kubernetes NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

---
*Part 086 | Road to 1,000,000 Users/Day | chuaikan.com*
