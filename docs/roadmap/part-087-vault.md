# Part 087: Secrets Management ด้วย Vault
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Professional
> **Steps:** 861–870
> **เวลาโดยประมาณ:** 8 ชั่วโมง
> **Prerequisites:** Part 086 (Zero Trust), Part 031 (Kubernetes)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- HashiCorp Vault installation บน Ubuntu 24.04
- Vault initialization และ unsealing
- Secret engines: KV v2 สำหรับ static secrets, dynamic PostgreSQL credentials
- Dynamic database credentials (auto-rotate ทุก 1 ชั่วโมง)
- Vault Agent สำหรับ Kubernetes (sidecar pattern)
- External Secrets Operator สำหรับ K8s
- Secret rotation runbook
- Vault monitoring และ audit log

---

## 📖 ทฤษฎีและแนวคิด

### ทำไมต้องใช้ Vault?

ปัญหากับ secrets ใน environment variables:
- Hard-coded ใน `.env` files → รั่วผ่าน git
- Rotate ยาก (ต้อง redeploy)
- ไม่มี audit trail ว่าใครดู secret
- ไม่มี expiry

Vault แก้ปัญหาเหล่านี้:
- **Dynamic credentials**: DB credentials ใหม่ทุก request
- **Automatic rotation**: credentials expire อัตโนมัติ
- **Audit log**: ทุก secret access บันทึก
- **Access control**: policy-based access

---

## ⚙️ Environment Setup

### Step 861: ติดตั้ง Vault บน Ubuntu 24.04

```bash
# เพิ่ม HashiCorp repository
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update
sudo apt install vault -y
vault version
# Vault v1.17.x
```

```hcl
# /etc/vault.d/vault.hcl
ui            = true
disable_mlock = true

# Storage backend: Raft (built-in HA)
storage "raft" {
  path    = "/opt/vault/data"
  node_id = "vault-1"

  retry_join {
    auto_join = "provider=aws region=ap-southeast-1 tag_key=vault-cluster tag_value=chuaikan"
  }
}

# Listener
listener "tcp" {
  address            = "0.0.0.0:8200"
  cluster_address    = "0.0.0.0:8201"
  tls_cert_file      = "/etc/vault.d/tls/vault.crt"
  tls_key_file       = "/etc/vault.d/tls/vault.key"
  tls_min_version    = "tls12"
}

# API address
api_addr     = "https://vault.chuaikan.internal:8200"
cluster_addr = "https://vault.chuaikan.internal:8201"

# Telemetry
telemetry {
  prometheus_retention_time = "30s"
  disable_hostname          = false
}

# Auto-unseal ด้วย AWS KMS
seal "awskms" {
  region     = "ap-southeast-1"
  kms_key_id = "alias/vault-unseal"
}

# Audit logging
audit {
  enabled = true
  path    = "file/"
  description = "File audit device"
  options = {
    file_path = "/var/log/vault/audit.log"
  }
}
```

---

## 🛠️ Step-by-Step Implementation

### Step 862: Vault Initialization

```bash
# Start Vault service
sudo systemctl enable vault
sudo systemctl start vault

# Initialize Vault (ครั้งแรกเท่านั้น)
vault operator init \
  -key-shares=5 \
  -key-threshold=3 \
  -format=json > /tmp/vault-init.json

# บันทึก keys อย่างปลอดภัย!
cat /tmp/vault-init.json | jq '.unseal_keys_b64'
# ได้ 5 keys → เก็บแต่ละอันไว้ที่ต่างกัน

ROOT_TOKEN=$(cat /tmp/vault-init.json | jq -r '.root_token')

# Unseal ด้วย 3 ใน 5 keys
vault operator unseal $(cat /tmp/vault-init.json | jq -r '.unseal_keys_b64[0]')
vault operator unseal $(cat /tmp/vault-init.json | jq -r '.unseal_keys_b64[1]')
vault operator unseal $(cat /tmp/vault-init.json | jq -r '.unseal_keys_b64[2]')

# Login ด้วย root token
vault login ${ROOT_TOKEN}

# ตรวจสอบ status
vault status
```

### Step 863: Enable Secret Engines

```bash
# Export Vault address
export VAULT_ADDR="https://vault.chuaikan.internal:8200"

# Enable KV v2 secret engine
vault secrets enable -path=chuaikan kv-v2

# เพิ่ม static secrets
vault kv put chuaikan/api \
  jwt_secret="$(openssl rand -base64 64)" \
  stripe_key="sk_live_xxxxx" \
  cloudflare_api_token="cf_xxxxx"

vault kv put chuaikan/email \
  smtp_host="smtp.sendgrid.net" \
  smtp_user="apikey" \
  smtp_pass="SG.xxxxx"

vault kv put chuaikan/redis \
  url="rediss://chuaikan-redis.cluster.sg.cache.amazonaws.com:6379" \
  auth_token="redis_auth_token_here"

# ดู secrets
vault kv get chuaikan/api
vault kv list chuaikan/
```

### Step 864: Dynamic Database Credentials

```bash
# Enable database secret engine
vault secrets enable database

# Configure PostgreSQL connection
vault write database/config/chuaikan-db \
  plugin_name=postgresql-database-plugin \
  allowed_roles="api-role,readonly-role" \
  connection_url="postgresql://{{username}}:{{password}}@sg.db.chuaikan.com:5432/chuaikan?sslmode=require" \
  username="vault_admin" \
  password="vault_admin_password" \
  verify_connection=true

# สร้าง role สำหรับ API (read/write, expire 1 ชั่วโมง)
vault write database/roles/api-role \
  db_name=chuaikan-db \
  creation_statements="
    CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';
    GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";
    GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO \"{{name}}\";
  " \
  revocation_statements="
    REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA public FROM \"{{name}}\";
    DROP ROLE IF EXISTS \"{{name}}\";
  " \
  default_ttl="1h" \
  max_ttl="24h"

# สร้าง role สำหรับ read-only (analytics)
vault write database/roles/readonly-role \
  db_name=chuaikan-db \
  creation_statements="
    CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";
  " \
  revocation_statements="
    REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA public FROM \"{{name}}\";
    DROP ROLE IF EXISTS \"{{name}}\";
  " \
  default_ttl="4h" \
  max_ttl="24h"

# ทดสอบ: ขอ dynamic credentials
vault read database/creds/api-role
# username: v-api-role-xxxxxxxx
# password: A1B2C3D4E5F6...
# lease_duration: 1h0m0s
```

### Step 865: Vault Policies

```bash
# สร้าง policy สำหรับ API service
cat > /tmp/api-policy.hcl << 'EOF'
# อ่าน static secrets
path "chuaikan/data/api" {
  capabilities = ["read"]
}

path "chuaikan/data/redis" {
  capabilities = ["read"]
}

# ขอ dynamic DB credentials
path "database/creds/api-role" {
  capabilities = ["read"]
}

# Renew token และ leases ตัวเอง
path "auth/token/renew-self" {
  capabilities = ["update"]
}

path "sys/leases/renew" {
  capabilities = ["update"]
}
EOF

vault policy write api-service /tmp/api-policy.hcl

# สร้าง policy สำหรับ analytics
cat > /tmp/analytics-policy.hcl << 'EOF'
path "database/creds/readonly-role" {
  capabilities = ["read"]
}

path "auth/token/renew-self" {
  capabilities = ["update"]
}
EOF

vault policy write analytics-service /tmp/analytics-policy.hcl
```

### Step 866: Vault Agent สำหรับ Kubernetes

```bash
# Enable Kubernetes Auth Method
vault auth enable kubernetes

# Configure Kubernetes auth
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc:443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  issuer="https://kubernetes.default.svc.cluster.local"

# สร้าง role สำหรับ API pods
vault write auth/kubernetes/role/chuaikan-api \
  bound_service_account_names=chuaikan-api \
  bound_service_account_namespaces=chuaikan \
  policies=api-service \
  ttl=1h
```

```yaml
# k8s/vault/api-deployment.yaml
# API deployment ด้วย Vault Agent sidecar

apiVersion: apps/v1
kind: Deployment
metadata:
  name: chuaikan-api
  namespace: chuaikan
spec:
  replicas: 3
  selector:
    matchLabels:
      app: chuaikan-api
  template:
    metadata:
      labels:
        app: chuaikan-api
      annotations:
        # Vault Agent injection annotations
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "chuaikan-api"
        vault.hashicorp.com/agent-inject-secret-db: "database/creds/api-role"
        vault.hashicorp.com/agent-inject-template-db: |
          {{- with secret "database/creds/api-role" -}}
          DATABASE_URL=postgresql://{{ .Data.username }}:{{ .Data.password }}@sg.db.chuaikan.com:5432/chuaikan?sslmode=require
          {{- end }}
        vault.hashicorp.com/agent-inject-secret-api: "chuaikan/data/api"
        vault.hashicorp.com/agent-inject-template-api: |
          {{- with secret "chuaikan/data/api" -}}
          JWT_SECRET={{ .Data.data.jwt_secret }}
          STRIPE_KEY={{ .Data.data.stripe_key }}
          {{- end }}
        vault.hashicorp.com/agent-pre-populate-only: "false"
        vault.hashicorp.com/agent-cache-enable: "true"
    spec:
      serviceAccountName: chuaikan-api
      containers:
        - name: api
          image: registry.chuaikan.com/api:latest
          command: ["/bin/sh", "-c"]
          args:
            - |
              # Load secrets จาก Vault Agent
              export $(cat /vault/secrets/db)
              export $(cat /vault/secrets/api)
              exec node dist/index.js
          ports:
            - containerPort: 3000
```

### Step 867: External Secrets Operator

```bash
# ติดตั้ง External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets \
  external-secrets/external-secrets \
  -n external-secrets-system \
  --create-namespace \
  --set installCRDs=true
```

```yaml
# k8s/external-secrets/vault-store.yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: chuaikan
spec:
  provider:
    vault:
      server: "https://vault.chuaikan.internal:8200"
      path: "chuaikan"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "chuaikan-api"
          serviceAccountRef:
            name: chuaikan-api

---
# สร้าง K8s Secret จาก Vault
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: chuaikan-api-secrets
  namespace: chuaikan
spec:
  refreshInterval: "5m"  # sync ทุก 5 นาที
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: chuaikan-api-env  # K8s secret name
    creationPolicy: Owner
  data:
    - secretKey: JWT_SECRET
      remoteRef:
        key: api
        property: jwt_secret
    - secretKey: STRIPE_KEY
      remoteRef:
        key: api
        property: stripe_key
    - secretKey: REDIS_URL
      remoteRef:
        key: redis
        property: url
```

### Step 868: Secret Rotation Runbook

```bash
#!/bin/bash
# scripts/rotate-secrets.sh
# Rotate static secrets ใน Vault

set -euo pipefail
export VAULT_ADDR="https://vault.chuaikan.internal:8200"

echo "=== Secret Rotation Runbook ==="

# 1. Rotate JWT Secret
rotate_jwt_secret() {
    echo "Rotating JWT secret..."
    NEW_SECRET=$(openssl rand -base64 64)

    # บันทึก secret เก่าไว้ก่อน (dual-key period)
    OLD_SECRET=$(vault kv get -field=jwt_secret chuaikan/api)

    # อัพเดท Vault พร้อมทั้ง secret ใหม่และเก่า
    vault kv patch chuaikan/api \
      jwt_secret="${NEW_SECRET}" \
      jwt_secret_old="${OLD_SECRET}"

    echo "JWT secret rotated. Old secret saved as jwt_secret_old."
    echo "IMPORTANT: Deploy new version that accepts BOTH old and new secrets."
    echo "After deployment stable for 2h: remove jwt_secret_old"
}

# 2. Rotate Database Password (vault admin user)
rotate_vault_admin_password() {
    echo "Rotating Vault admin DB password..."

    vault write -force database/config/chuaikan-db \
      rotate-root=true

    echo "Vault admin DB password rotated."
    echo "Vault will now use new password for dynamic credential creation."
}

# 3. Revoke all existing dynamic credentials
revoke_all_db_leases() {
    echo "Revoking all database leases..."
    vault lease revoke -prefix database/creds/api-role
    vault lease revoke -prefix database/creds/readonly-role
    echo "All leases revoked. Apps will automatically request new credentials."
}

# Main
echo "Which rotation to perform?"
select action in "jwt_secret" "vault_admin_db" "all_db_leases" "exit"; do
    case $action in
        jwt_secret) rotate_jwt_secret ;;
        vault_admin_db) rotate_vault_admin_password ;;
        all_db_leases) revoke_all_db_leases ;;
        exit) break ;;
    esac
    break
done
```

---

## 🧪 Testing

```bash
# ทดสอบ dynamic credentials
vault read database/creds/api-role
# รันอีกครั้ง → ได้ credentials ต่างกัน

# ทดสอบ credentials หมดอายุ
CREDS=$(vault read -format=json database/creds/api-role)
USERNAME=$(echo $CREDS | jq -r .data.username)
PASSWORD=$(echo $CREDS | jq -r .data.password)
LEASE=$(echo $CREDS | jq -r .lease_id)

psql "host=sg.db.chuaikan.com user=${USERNAME} password=${PASSWORD} dbname=chuaikan" -c "SELECT 1;"
# PASS

# Revoke lease
vault lease revoke ${LEASE}

# ลอง connect อีกครั้ง
psql "host=sg.db.chuaikan.com user=${USERNAME} password=${PASSWORD} dbname=chuaikan" -c "SELECT 1;"
# FAIL: role does not exist (ถูก delete แล้ว)
```

---

## ✅ Checklist

- [ ] Vault ติดตั้งและ unsealed อัตโนมัติด้วย AWS KMS
- [ ] KV v2 secret engine เปิดที่ path "chuaikan"
- [ ] Database secret engine: PostgreSQL dynamic credentials
- [ ] Dynamic credentials expire ภายใน 1 ชั่วโมง
- [ ] Kubernetes auth method: pods ขอ token ได้
- [ ] Vault Agent sidecar inject secrets ให้ API pods
- [ ] External Secrets Operator sync K8s secrets จาก Vault
- [ ] Vault audit log เปิดและส่งไป centralized logging
- [ ] Secret rotation runbook ทดสอบแล้ว
- [ ] ไม่มี hardcoded secrets ใน codebase (git secret scan)

---

## 🔗 References

- [HashiCorp Vault Documentation](https://developer.hashicorp.com/vault/docs)
- [Vault Agent Kubernetes](https://developer.hashicorp.com/vault/docs/platform/k8s/injector)
- [External Secrets Operator](https://external-secrets.io/latest/)

---
*Part 087 | Road to 1,000,000 Users/Day | chuaikan.com*
