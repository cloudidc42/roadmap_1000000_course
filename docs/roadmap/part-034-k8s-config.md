# Part 034: ConfigMaps & Secrets

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 331-340
> **เวลาโดยประมาณ:** 2.5 ชั่วโมง
> **Prerequisites:** Part 033 (Pods, Deployments, Services)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- สร้าง ConfigMap สำหรับ non-sensitive configuration ของ chuaikan.com
- สร้าง Secret สำหรับ sensitive data และเข้าใจ base64 encoding
- ใช้ Sealed Secrets และ External Secrets Operator สำหรับ production
- Mount ConfigMap เป็น environment variables และ files
- ทำ Secret rotation

---

## 📖 ทฤษฎีและแนวคิด

### Step 331 — ConfigMap vs Secret

```
ConfigMap:
- เก็บข้อมูล non-sensitive: NODE_ENV, PORT, API_URL
- ข้อมูลเป็น plain text
- ใช้ได้กับ env vars, command args, volume files

Secret:
- เก็บข้อมูล sensitive: passwords, API keys, TLS certificates
- ข้อมูล encode เป็น base64 (ไม่ใช่ encryption!)
- ต้องระวัง: base64 แค่ encode ไม่ใช่ปกป้อง
- production ควรใช้ Sealed Secrets หรือ External Secrets
```

### Step 332 — Base64 Encoding ใน Secrets

```bash
# encode ค่า
echo -n "my-password-123" | base64
# bXktcGFzc3dvcmQtMTIz

# decode ค่า
echo "bXktcGFzc3dvcmQtMTIz" | base64 --decode
# my-password-123

# ข้อควรระวัง: -n flag สำคัญมาก!
echo "password" | base64   # ผิด! มี newline ต่อท้าย
echo -n "password" | base64  # ถูก!
```

---

## ⚙️ Environment Setup

```bash
# namespace ที่ใช้ในตัวอย่าง
export NAMESPACE=chuaikan-production

# ตรวจสอบ namespace
kubectl get namespace $NAMESPACE
```

---

## 🛠️ Step-by-Step Implementation

### Step 333 — ConfigMap สำหรับ Next.js App

```yaml
# configmap-nextjs.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nextjs-config
  namespace: chuaikan-production
  labels:
    app: nextjs
    component: config
data:
  # Application settings
  NODE_ENV: "production"
  PORT: "3000"
  NEXT_PUBLIC_APP_NAME: "chuaikan.com"
  NEXT_PUBLIC_API_URL: "https://api.chuaikan.com"
  NEXT_PUBLIC_CDN_URL: "https://cdn.chuaikan.com"

  # Feature flags
  NEXT_PUBLIC_ENABLE_ANALYTICS: "true"
  NEXT_PUBLIC_MAINTENANCE_MODE: "false"

  # Rate limiting
  RATE_LIMIT_WINDOW_MS: "60000"
  RATE_LIMIT_MAX_REQUESTS: "100"

  # Cache settings
  CACHE_TTL_SECONDS: "300"
  REDIS_HOST: "redis-service"
  REDIS_PORT: "6379"
```

```yaml
# configmap-api.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: chuaikan-production
data:
  NODE_ENV: "production"
  PORT: "4000"
  LOG_LEVEL: "info"
  LOG_FORMAT: "json"

  # Database connection (without password)
  DB_HOST: "postgres-service"
  DB_PORT: "5432"
  DB_NAME: "chuaikan_db"
  DB_USER: "chuaikan_user"
  DB_SSL: "true"
  DB_POOL_MIN: "2"
  DB_POOL_MAX: "10"

  # Redis
  REDIS_HOST: "redis-service"
  REDIS_PORT: "6379"

  # File as ConfigMap
  nginx.conf: |
    worker_processes auto;
    events {
        worker_connections 1024;
    }
    http {
        upstream nextjs {
            server nextjs-service:3000;
        }
        server {
            listen 80;
            location / {
                proxy_pass http://nextjs;
            }
        }
    }
```

```bash
# Apply ConfigMaps
kubectl apply -f configmap-nextjs.yaml
kubectl apply -f configmap-api.yaml

# ดู ConfigMap
kubectl get configmap -n chuaikan-production
kubectl describe configmap nextjs-config -n chuaikan-production

# ดูข้อมูลทั้งหมด
kubectl get configmap nextjs-config -o yaml -n chuaikan-production
```

### Step 334 — Secret สำหรับ Sensitive Data

```bash
# สร้าง Secret แบบ imperative (command line)
kubectl create secret generic api-secrets \
  --from-literal=DATABASE_PASSWORD='Str0ng!Pass#2024' \
  --from-literal=JWT_SECRET='super-secret-jwt-key-change-in-production' \
  --from-literal=STRIPE_API_KEY='sk_live_xxxxxxxxxxxx' \
  --from-literal=SENDGRID_API_KEY='SG.xxxxxxxxxxxx' \
  --namespace=chuaikan-production

# ดู secret (ค่าจะถูก mask)
kubectl get secret api-secrets -n chuaikan-production
kubectl describe secret api-secrets -n chuaikan-production
```

ไฟล์ `secret-api.yaml` (สำหรับ version control — encode เอง):

```bash
# encode ค่าก่อน
echo -n "Str0ng!Pass#2024" | base64
# U3RyMGchUGFzcyMyMDI0

echo -n "super-secret-jwt-key-change-in-production" | base64
# c3VwZXItc2VjcmV0LWp3dC1rZXktY2hhbmdlLWluLXByb2R1Y3Rpb24=
```

```yaml
# secret-api.yaml (อย่า commit ลง Git จริง!)
apiVersion: v1
kind: Secret
metadata:
  name: api-secrets
  namespace: chuaikan-production
  labels:
    app: nodejs-api
type: Opaque
data:
  DATABASE_PASSWORD: U3RyMGchUGFzcyMyMDI0
  JWT_SECRET: c3VwZXItc2VjcmV0LWp3dC1rZXktY2hhbmdlLWluLXByb2R1Y3Rpb24=
  STRIPE_API_KEY: c2tfbGl2ZV94eHh4eHh4eHh4eHg=
  SENDGRID_API_KEY: U0cueHh4eHh4eHh4eHh4eA==
---
# Secret สำหรับ PostgreSQL
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secrets
  namespace: chuaikan-production
type: Opaque
data:
  POSTGRES_USER: Y2h1YWlrYW5fdXNlcg==
  POSTGRES_PASSWORD: U3RyMGchUGFzcyMyMDI0
  POSTGRES_DB: Y2h1YWlrYW5fZGI=
---
# TLS Secret สำหรับ HTTPS
apiVersion: v1
kind: Secret
metadata:
  name: chuaikan-tls
  namespace: chuaikan-production
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi... # base64 encoded certificate
  tls.key: LS0tLS1CRUdJTi... # base64 encoded private key
```

### Step 335 — Mount ConfigMap เป็น Environment Variables

```yaml
# deployment-nextjs-with-config.yaml (snippet)
spec:
  containers:
  - name: nextjs
    image: registry.chuaikan.com/nextjs-app:v1.0.0
    # วิธีที่ 1: envFrom - import ทั้ง ConfigMap
    envFrom:
    - configMapRef:
        name: nextjs-config
    - secretRef:
        name: api-secrets

    # วิธีที่ 2: env - เลือก key เฉพาะที่ต้องการ
    env:
    - name: MY_NODE_ENV
      valueFrom:
        configMapKeyRef:
          name: nextjs-config
          key: NODE_ENV
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: api-secrets
          key: DATABASE_PASSWORD
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
```

### Step 336 — Mount ConfigMap เป็น Files (Volume)

```yaml
# deployment-with-configmap-volume.yaml (snippet)
spec:
  volumes:
  - name: nginx-config-volume
    configMap:
      name: api-config
      items:
      - key: nginx.conf
        path: nginx.conf

  containers:
  - name: nginx-sidecar
    image: nginx:alpine
    volumeMounts:
    - name: nginx-config-volume
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf
      readOnly: true

  - name: app-config-volume
    configMap:
      name: nextjs-config
      # mount ทั้ง ConfigMap เป็น files
```

```bash
# ตรวจสอบว่า config ถูก mount
kubectl exec -it nextjs-app-xxx -n chuaikan-production -- \
  env | grep NODE_ENV

kubectl exec -it nextjs-app-xxx -n chuaikan-production -- \
  cat /etc/nginx/nginx.conf
```

### Step 337 — Sealed Secrets (Production Recommended)

Sealed Secrets ช่วยให้เราเก็บ encrypted secrets ใน Git ได้อย่างปลอดภัย:

```bash
# ติดตั้ง Sealed Secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.4/controller.yaml

# ติดตั้ง kubeseal CLI
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.4/kubeseal-0.24.4-linux-amd64.tar.gz
tar -xzf kubeseal-0.24.4-linux-amd64.tar.gz
sudo install -m 755 kubeseal /usr/local/bin/kubeseal

# ตรวจสอบ
kubeseal --version

# สร้าง Secret ปกติก่อน (ยังไม่ apply ลง cluster)
kubectl create secret generic api-secrets \
  --from-literal=DATABASE_PASSWORD='Str0ng!Pass#2024' \
  --from-literal=JWT_SECRET='super-secret-jwt-key' \
  --namespace=chuaikan-production \
  --dry-run=client -o yaml > /tmp/api-secrets.yaml

# Seal ด้วย kubeseal
kubeseal --format yaml < /tmp/api-secrets.yaml > sealed-api-secrets.yaml

# ดู SealedSecret (สามารถ commit ลง Git ได้!)
cat sealed-api-secrets.yaml

# Apply SealedSecret ลง cluster
kubectl apply -f sealed-api-secrets.yaml

# Controller จะ decrypt และสร้าง Secret ให้อัตโนมัติ
kubectl get secret api-secrets -n chuaikan-production
```

### Step 338 — External Secrets Operator

สำหรับ enterprise ที่ใช้ Vault, AWS Secrets Manager, GCP Secret Manager:

```bash
# ติดตั้ง External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets \
  external-secrets/external-secrets \
  -n external-secrets-system \
  --create-namespace

# ตรวจสอบ
kubectl get pods -n external-secrets-system
```

ตัวอย่าง ExternalSecret กับ AWS Secrets Manager:

```yaml
# external-secret-aws.yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secretsmanager
  namespace: chuaikan-production
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-southeast-1
      auth:
        secretRef:
          accessKeyIDSecretRef:
            name: aws-credentials
            key: access-key
          secretAccessKeySecretRef:
            name: aws-credentials
            key: secret-access-key
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: api-secrets
  namespace: chuaikan-production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: SecretStore
  target:
    name: api-secrets
    creationPolicy: Owner
  data:
  - secretKey: DATABASE_PASSWORD
    remoteRef:
      key: chuaikan/production/database
      property: password
  - secretKey: JWT_SECRET
    remoteRef:
      key: chuaikan/production/jwt
      property: secret
```

### Step 339 — Secret Rotation

```bash
# Update Secret (จะ trigger pod restart ถ้า set envFrom)
kubectl create secret generic api-secrets \
  --from-literal=DATABASE_PASSWORD='NewStr0ng!Pass#2024' \
  --from-literal=JWT_SECRET='new-super-secret-jwt-key' \
  --namespace=chuaikan-production \
  --dry-run=client -o yaml | kubectl apply -f -

# Force pod restart เพื่อใช้ secret ใหม่
kubectl rollout restart deployment/nextjs-app -n chuaikan-production
kubectl rollout restart deployment/nodejs-api -n chuaikan-production

# ตรวจสอบ rollout
kubectl rollout status deployment/nodejs-api -n chuaikan-production
```

### Step 340 — Best Practices สำหรับ Secrets

```bash
# ไม่ควรทำ: เก็บ secret ใน plain text ใน Git
# ควรทำ: ใช้ Sealed Secrets หรือ External Secrets

# ตรวจสอบว่า RBAC จำกัดการอ่าน secret
kubectl create role secret-reader \
  --verb=get,list \
  --resource=secrets \
  --resource-name=api-secrets \
  -n chuaikan-production

kubectl create rolebinding api-secret-reader \
  --role=secret-reader \
  --serviceaccount=chuaikan-production:api-service-account \
  -n chuaikan-production

# ดู secrets ทั้งหมดใน namespace
kubectl get secrets -n chuaikan-production

# ลบ secret ที่ไม่ใช้แล้ว
kubectl delete secret old-api-secrets -n chuaikan-production

# Encrypt etcd at rest (ใน kubeadm config)
# apiVersion: apiserver.config.k8s.io/v1
# kind: EncryptionConfiguration
# resources:
#   - resources:
#     - secrets
#     providers:
#     - aescbc:
#         keys:
#         - name: key1
#           secret: <base64-encoded-32-byte-key>
```

---

## 🔧 Configuration Files

### .gitignore สำหรับ Kubernetes secrets

```bash
# k8s/.gitignore
*-secrets.yaml
!sealed-*-secrets.yaml
*.key
*.pem
.env
.env.local
kubeconfig
```

---

## 🧪 Testing

### ทดสอบ ConfigMap และ Secret ถูก inject เข้า Pod

```bash
# ดู env vars ใน pod
kubectl exec -it $(kubectl get pod -l app=nextjs -n chuaikan-production -o name | head -1) \
  -n chuaikan-production -- env | sort

# ตรวจสอบ config ถูก load
kubectl exec -it $(kubectl get pod -l app=nodejs-api -n chuaikan-production -o name | head -1) \
  -n chuaikan-production -- node -e "console.log(process.env.NODE_ENV)"

# ทดสอบ Secret ถูกส่งผ่าน env var (จะเห็นค่า แต่ masked ใน kubectl describe)
kubectl exec -it $(kubectl get pod -l app=nodejs-api -n chuaikan-production -o name | head -1) \
  -n chuaikan-production -- node -e "console.log('DB:', process.env.DB_HOST)"
```

### ทดสอบ Volume mount

```bash
kubectl exec -it $(kubectl get pod -l app=nginx-sidecar -n chuaikan-production -o name | head -1) \
  -n chuaikan-production -- cat /etc/nginx/nginx.conf
```

---

## ❌ Common Errors & Solutions

### Error: `Error from server (Forbidden): secrets is forbidden`
```bash
# สาเหตุ: ServiceAccount ไม่มีสิทธิ์อ่าน secret
# แก้: ตรวจสอบ RBAC
kubectl auth can-i get secrets \
  --as=system:serviceaccount:chuaikan-production:default \
  -n chuaikan-production
```

### Error: `secret "api-secrets" not found`
```bash
# สาเหตุ: Secret ไม่ได้อยู่ใน namespace เดียวกับ Pod
kubectl get secret api-secrets -n chuaikan-production
# ถ้าไม่มี ต้องสร้างใน namespace ที่ถูก
kubectl apply -f secret-api.yaml -n chuaikan-production
```

### Error: `could not find expected key in ConfigMap`
```bash
# สาเหตุ: key ใน ConfigMap reference ผิด
kubectl describe configmap nextjs-config -n chuaikan-production
# ตรวจสอบ key names ให้ตรงกับที่ใช้ใน deployment
```

---

## ✅ Checklist

- [ ] **Step 331**: เข้าใจความแตกต่าง ConfigMap vs Secret
- [ ] **Step 332**: ใช้ base64 encode/decode สำหรับ Secret values ได้
- [ ] **Step 333**: สร้าง ConfigMap สำหรับ Next.js และ API ได้
- [ ] **Step 334**: สร้าง Secret สำหรับ DATABASE_PASSWORD, JWT_SECRET, API_KEYS ได้
- [ ] **Step 335**: Mount ConfigMap เป็น env vars ด้วย envFrom และ env ได้
- [ ] **Step 336**: Mount ConfigMap เป็น files ผ่าน Volume ได้
- [ ] **Step 337**: ติดตั้งและใช้ Sealed Secrets สำหรับเก็บ encrypted secrets ใน Git ได้
- [ ] **Step 338**: เข้าใจ External Secrets Operator และตั้งค่ากับ AWS Secrets Manager ได้
- [ ] **Step 339**: ทำ Secret rotation และ restart pods ได้
- [ ] **Step 340**: ทำตาม best practices สำหรับ Secrets management

---

## 🔗 References

- [ConfigMaps Documentation](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Secrets Documentation](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- [External Secrets Operator](https://external-secrets.io/latest/)

---

*Part 034 | Road to 1,000,000 Users/Day | chuaikan.com*
