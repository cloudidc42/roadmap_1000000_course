# Part 008: CI/CD ด้วย GitHub Actions
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Beginner
> **Steps:** 71-80
> **เวลาโดยประมาณ:** 5 ชั่วโมง
> **Prerequisites:** Part 001-007 (Linux, Git, Node.js, PostgreSQL, Redis, Docker, Cloudflare)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

1. เข้าใจแนวคิด GitHub Actions (workflows, jobs, steps, runners)
2. สร้าง CI workflow สำหรับ lint, test, build บน Pull Request
3. สร้าง CD workflow สำหรับ deploy ไป staging
4. สร้าง CD workflow สำหรับ deploy ไป production
5. จัดการ Environment Variables และ GitHub Secrets
6. ใช้งาน Matrix testing สำหรับหลาย Node.js versions
7. Cache npm dependencies เพื่อเร่ง build
8. Build และ push Docker image ไปยัง GitHub Container Registry
9. Deploy ไปยัง VPS ผ่าน SSH
10. ส่ง Discord notifications และทำ rollback

---

## 📖 ทฤษฎีและแนวคิด

### GitHub Actions Architecture

```
Repository
└── .github/
    └── workflows/
        ├── ci.yml           ← Run on PR
        ├── deploy-staging.yml  ← Run on push to develop
        └── deploy-production.yml  ← Run on tag v*

Workflow
└── Job (runs on a runner)
    └── Step 1: checkout code
    └── Step 2: setup node
    └── Step 3: install deps
    └── Step 4: run tests
    └── Step 5: build
    └── Step 6: deploy
```

### CI/CD Flow สำหรับ chuaikan.com

```
Developer pushes code
        │
        ▼
┌───────────────┐
│  Pull Request  │ ─── triggers ──→ CI Workflow
│  (feature →   │                  (lint + test + build)
│   develop)    │
└───────────────┘
        │
        ▼ merge
┌───────────────┐
│  develop branch│ ─── triggers ──→ Deploy Staging
│               │                  (auto-deploy)
└───────────────┘
        │
        ▼ tag release
┌───────────────┐
│  main branch  │ ─── triggers ──→ Deploy Production
│  + tag v1.x.x │                  (with approval)
└───────────────┘
```

---

## ⚙️ Environment Setup

### Step 71: ตั้งค่า GitHub Repository Secrets

ไปที่ Repository → Settings → Secrets and variables → Actions

```
Secrets ที่ต้องตั้งค่า:

# VPS Connection
VPS_HOST=203.0.113.10
VPS_USER=deployer
VPS_SSH_KEY=-----BEGIN OPENSSH PRIVATE KEY-----...

# Staging VPS
STAGING_HOST=203.0.113.20
STAGING_USER=deployer
STAGING_SSH_KEY=-----BEGIN OPENSSH PRIVATE KEY-----...

# Container Registry
GHCR_TOKEN=ghp_xxxxxxxxxxxx  (GitHub Personal Access Token with packages:write)

# Application
JWT_SECRET=your_jwt_secret_here
DATABASE_URL=postgresql://...
REDIS_URL=redis://...

# Cloudflare (for cache purge)
CF_API_TOKEN=your_cloudflare_api_token
CF_ZONE_ID=your_zone_id

# Discord Notifications
DISCORD_WEBHOOK_URL=https://discord.com/api/webhooks/xxx/yyy
```

สร้าง SSH key สำหรับ deployment:

```bash
# สร้าง SSH key pair บน local machine
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/chuaikan_deploy_key

# ดู public key (เอาไปใส่บน VPS)
cat ~/.ssh/chuaikan_deploy_key.pub

# ดู private key (เอาไปใส่ใน GitHub Secrets ชื่อ VPS_SSH_KEY)
cat ~/.ssh/chuaikan_deploy_key

# บน VPS: เพิ่ม public key
# ssh into VPS as root หรือ admin user
mkdir -p /home/deployer/.ssh
echo "ssh-ed25519 AAAAC3... github-actions-deploy" >> /home/deployer/.ssh/authorized_keys
chmod 700 /home/deployer/.ssh
chmod 600 /home/deployer/.ssh/authorized_keys

# สร้าง deployer user บน VPS
sudo adduser --disabled-password --gecos "" deployer
sudo usermod -aG docker deployer
sudo usermod -aG sudo deployer
```

---

## 🛠️ Step-by-Step Implementation

### Step 72: CI Workflow (PR Check)

สร้างไฟล์ `.github/workflows/ci.yml`:

```yaml
name: CI — Lint, Test, Build

on:
  pull_request:
    branches: [main, develop]
    types: [opened, synchronize, reopened]
  push:
    branches: [main, develop]

# ยกเลิก run ก่อนหน้าถ้ามี push ใหม่
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  NODE_VERSION: '22'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ===================================
  # Job 1: Lint
  # ===================================
  lint:
    name: Lint Code
    runs-on: ubuntu-24.04
    timeout-minutes: 10

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run ESLint
        run: npm run lint

      - name: Check TypeScript
        run: npm run type-check

      - name: Check formatting (Prettier)
        run: npm run format:check

  # ===================================
  # Job 2: Test (Matrix)
  # ===================================
  test:
    name: Test (Node ${{ matrix.node-version }})
    runs-on: ubuntu-24.04
    timeout-minutes: 15

    strategy:
      matrix:
        node-version: ['20', '22']
      fail-fast: false

    services:
      postgres:
        image: postgres:17-alpine
        env:
          POSTGRES_DB: chuaikan_test
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_password
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run database migrations
        env:
          DATABASE_URL: postgresql://test_user:test_password@localhost:5432/chuaikan_test
        run: npm run db:migrate

      - name: Run unit tests
        env:
          DATABASE_URL: postgresql://test_user:test_password@localhost:5432/chuaikan_test
          REDIS_URL: redis://localhost:6379
          JWT_SECRET: test_jwt_secret_for_ci
          NODE_ENV: test
        run: npm run test:unit -- --coverage

      - name: Run integration tests
        env:
          DATABASE_URL: postgresql://test_user:test_password@localhost:5432/chuaikan_test
          REDIS_URL: redis://localhost:6379
          JWT_SECRET: test_jwt_secret_for_ci
          NODE_ENV: test
        run: npm run test:integration

      - name: Upload coverage report
        if: matrix.node-version == '22'
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: ./coverage/lcov.info
          fail_ci_if_error: false

  # ===================================
  # Job 3: Build
  # ===================================
  build:
    name: Build Docker Images
    runs-on: ubuntu-24.04
    timeout-minutes: 20
    needs: [lint, test]

    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        with:
          driver-opts: image=moby/buildkit:v0.13.0

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata for web image
        id: meta-web
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}/web
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=sha,prefix=sha-
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      - name: Build and push web image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: apps/web/Dockerfile
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta-web.outputs.tags }}
          labels: ${{ steps.meta-web.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            BUILDKIT_INLINE_CACHE=1

      - name: Extract metadata for api image
        id: meta-api
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}/api
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=sha,prefix=sha-
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      - name: Build and push api image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: apps/api/Dockerfile
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta-api.outputs.tags }}
          labels: ${{ steps.meta-api.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Report image sizes
        if: github.event_name == 'pull_request'
        run: |
          docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
```

### Step 73: Deploy to Staging Workflow

สร้างไฟล์ `.github/workflows/deploy-staging.yml`:

```yaml
name: Deploy — Staging

on:
  push:
    branches: [develop]

env:
  REGISTRY: ghcr.io
  WEB_IMAGE: ghcr.io/${{ github.repository }}/web
  API_IMAGE: ghcr.io/${{ github.repository }}/api

jobs:
  deploy-staging:
    name: Deploy to Staging VPS
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    environment: staging

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Set image tag
        id: vars
        run: echo "sha_short=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT

      - name: Deploy to Staging VPS
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            set -e
            
            # Login to registry
            echo "${{ secrets.GITHUB_TOKEN }}" | \
              docker login ghcr.io -u "${{ github.actor }}" --password-stdin
            
            # Pull latest images
            docker pull ${{ env.WEB_IMAGE }}:develop
            docker pull ${{ env.API_IMAGE }}:develop
            
            # Create .env file
            cat > /opt/chuaikan-staging/.env <<'ENVEOF'
            NODE_ENV=staging
            WEB_IMAGE=${{ env.WEB_IMAGE }}:develop
            API_IMAGE=${{ env.API_IMAGE }}:develop
            DATABASE_URL=${{ secrets.STAGING_DATABASE_URL }}
            REDIS_URL=${{ secrets.STAGING_REDIS_URL }}
            JWT_SECRET=${{ secrets.STAGING_JWT_SECRET }}
            APP_URL=https://staging.chuaikan.com
            ENVEOF
            
            # Deploy with zero-downtime
            cd /opt/chuaikan-staging
            docker compose pull
            docker compose up -d --remove-orphans
            
            # Wait for health check
            sleep 15
            
            # Verify deployment
            MAX_RETRIES=10
            RETRY_COUNT=0
            until curl -sf http://localhost:3000/api/health; do
              RETRY_COUNT=$((RETRY_COUNT + 1))
              if [ $RETRY_COUNT -ge $MAX_RETRIES ]; then
                echo "Health check failed after $MAX_RETRIES attempts"
                docker compose logs --tail=50
                exit 1
              fi
              sleep 5
            done
            
            echo "Staging deployment successful!"
            
            # Clean up old images
            docker image prune -f

      - name: Notify Discord - Success
        if: success()
        run: |
          curl -H "Content-Type: application/json" \
            -X POST \
            -d '{
              "embeds": [{
                "title": "✅ Staging Deployed",
                "description": "Branch `develop` deployed to staging successfully",
                "color": 3066993,
                "fields": [
                  {"name": "Commit", "value": "${{ steps.vars.outputs.sha_short }}", "inline": true},
                  {"name": "Author", "value": "${{ github.actor }}", "inline": true},
                  {"name": "URL", "value": "https://staging.chuaikan.com"}
                ]
              }]
            }' \
            ${{ secrets.DISCORD_WEBHOOK_URL }}

      - name: Notify Discord - Failure
        if: failure()
        run: |
          curl -H "Content-Type: application/json" \
            -X POST \
            -d '{
              "embeds": [{
                "title": "❌ Staging Deploy Failed",
                "description": "Deployment to staging failed!",
                "color": 15158332,
                "fields": [
                  {"name": "Commit", "value": "${{ steps.vars.outputs.sha_short }}", "inline": true},
                  {"name": "Author", "value": "${{ github.actor }}", "inline": true},
                  {"name": "Run URL", "value": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"}
                ]
              }]
            }' \
            ${{ secrets.DISCORD_WEBHOOK_URL }}
```

### Step 74: Deploy to Production Workflow

สร้างไฟล์ `.github/workflows/deploy-production.yml`:

```yaml
name: Deploy — Production

on:
  push:
    tags:
      - 'v*.*.*'  # trigger เฉพาะ semver tags เช่น v1.2.3

env:
  REGISTRY: ghcr.io
  WEB_IMAGE: ghcr.io/${{ github.repository }}/web
  API_IMAGE: ghcr.io/${{ github.repository }}/api

jobs:
  # ===================================
  # Job 1: Build Production Images
  # ===================================
  build:
    name: Build Production Images
    runs-on: ubuntu-24.04
    timeout-minutes: 20
    outputs:
      web_tag: ${{ steps.vars.outputs.web_tag }}
      api_tag: ${{ steps.vars.outputs.api_tag }}

    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set variables
        id: vars
        run: |
          VERSION=${GITHUB_REF#refs/tags/}
          echo "version=${VERSION}" >> $GITHUB_OUTPUT
          echo "web_tag=${{ env.WEB_IMAGE }}:${VERSION}" >> $GITHUB_OUTPUT
          echo "api_tag=${{ env.API_IMAGE }}:${VERSION}" >> $GITHUB_OUTPUT

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push web image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: apps/web/Dockerfile
          push: true
          tags: |
            ${{ steps.vars.outputs.web_tag }}
            ${{ env.WEB_IMAGE }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Build and push api image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: apps/api/Dockerfile
          push: true
          tags: |
            ${{ steps.vars.outputs.api_tag }}
            ${{ env.API_IMAGE }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ===================================
  # Job 2: Deploy Production
  # ===================================
  deploy:
    name: Deploy to Production
    runs-on: ubuntu-24.04
    timeout-minutes: 20
    needs: [build]
    environment:
      name: production
      url: https://chuaikan.com

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Get version
        id: vars
        run: echo "version=${GITHUB_REF#refs/tags/}" >> $GITHUB_OUTPUT

      - name: Deploy to Production VPS
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            set -e
            
            DEPLOY_DIR=/opt/chuaikan
            VERSION="${{ steps.vars.outputs.version }}"
            
            echo "=== Deploying version: $VERSION ==="
            
            # Login to registry
            echo "${{ secrets.GITHUB_TOKEN }}" | \
              docker login ghcr.io -u "${{ github.actor }}" --password-stdin
            
            # Pull new images
            echo "--- Pulling images ---"
            docker pull ${{ needs.build.outputs.web_tag }}
            docker pull ${{ needs.build.outputs.api_tag }}
            
            # Backup current deployment info
            echo "--- Backing up current state ---"
            if [ -f "$DEPLOY_DIR/.env" ]; then
              cp "$DEPLOY_DIR/.env" "$DEPLOY_DIR/.env.backup.$(date +%Y%m%d%H%M%S)"
            fi
            
            # Update environment
            cat > $DEPLOY_DIR/.env <<ENVEOF
            NODE_ENV=production
            VERSION=${VERSION}
            WEB_IMAGE=${{ needs.build.outputs.web_tag }}
            API_IMAGE=${{ needs.build.outputs.api_tag }}
            DATABASE_URL=${{ secrets.DATABASE_URL }}
            REDIS_URL=${{ secrets.REDIS_URL }}
            JWT_SECRET=${{ secrets.JWT_SECRET }}
            APP_URL=https://chuaikan.com
            ENVEOF
            
            # Zero-downtime deploy
            echo "--- Starting zero-downtime deploy ---"
            cd $DEPLOY_DIR
            
            # Scale up new containers
            docker compose up -d --scale web=2 --no-recreate
            sleep 20
            
            # Health check new instances
            MAX_RETRIES=15
            RETRY_COUNT=0
            until curl -sf http://localhost:3000/api/health; do
              RETRY_COUNT=$((RETRY_COUNT + 1))
              if [ $RETRY_COUNT -ge $MAX_RETRIES ]; then
                echo "Health check failed, rolling back..."
                docker compose up -d --scale web=1 --no-recreate
                exit 1
              fi
              echo "Health check attempt $RETRY_COUNT/$MAX_RETRIES..."
              sleep 5
            done
            
            echo "Health check passed!"
            
            # Scale back to normal
            docker compose up -d --scale web=1 --remove-orphans
            
            echo "=== Deploy $VERSION successful! ==="
            docker compose ps

      - name: Purge Cloudflare Cache
        run: |
          curl -s -X POST \
            "https://api.cloudflare.com/client/v4/zones/${{ secrets.CF_ZONE_ID }}/purge_cache" \
            -H "Authorization: Bearer ${{ secrets.CF_API_TOKEN }}" \
            -H "Content-Type: application/json" \
            --data '{"purge_everything": true}' | jq '.success'

      - name: Create GitHub Release
        uses: ncipollo/release-action@v1
        with:
          tag: ${{ steps.vars.outputs.version }}
          name: Release ${{ steps.vars.outputs.version }}
          body: |
            ## Changes in ${{ steps.vars.outputs.version }}
            
            Web Image: `${{ needs.build.outputs.web_tag }}`
            API Image: `${{ needs.build.outputs.api_tag }}`
          generateReleaseNotes: true
          makeLatest: true

      - name: Notify Discord - Success
        if: success()
        run: |
          curl -H "Content-Type: application/json" \
            -X POST \
            -d '{
              "content": "@here",
              "embeds": [{
                "title": "🚀 Production Deployed!",
                "description": "Version `${{ steps.vars.outputs.version }}` is now live!",
                "color": 3066993,
                "fields": [
                  {"name": "Version", "value": "${{ steps.vars.outputs.version }}", "inline": true},
                  {"name": "Author", "value": "${{ github.actor }}", "inline": true},
                  {"name": "URL", "value": "https://chuaikan.com"}
                ]
              }]
            }' \
            ${{ secrets.DISCORD_WEBHOOK_URL }}

      - name: Notify Discord - Failure
        if: failure()
        run: |
          curl -H "Content-Type: application/json" \
            -X POST \
            -d '{
              "content": "@here PRODUCTION DEPLOY FAILED!",
              "embeds": [{
                "title": "🔥 Production Deploy FAILED",
                "description": "IMMEDIATE ACTION REQUIRED",
                "color": 15158332,
                "fields": [
                  {"name": "Version", "value": "${{ steps.vars.outputs.version }}", "inline": true},
                  {"name": "Run URL", "value": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"}
                ]
              }]
            }' \
            ${{ secrets.DISCORD_WEBHOOK_URL }}

  # ===================================
  # Job 3: Smoke Tests
  # ===================================
  smoke-test:
    name: Smoke Tests
    runs-on: ubuntu-24.04
    needs: [deploy]
    timeout-minutes: 10

    steps:
      - name: Wait for deployment to stabilize
        run: sleep 30

      - name: Run smoke tests
        run: |
          # ทดสอบ endpoints หลัก
          check_endpoint() {
            local url=$1
            local expected_status=$2
            local status=$(curl -s -o /dev/null -w "%{http_code}" "$url")
            if [ "$status" != "$expected_status" ]; then
              echo "FAIL: $url returned $status (expected $expected_status)"
              exit 1
            fi
            echo "PASS: $url returned $status"
          }
          
          check_endpoint "https://chuaikan.com" "200"
          check_endpoint "https://chuaikan.com/api/health" "200"
          check_endpoint "https://api.chuaikan.com/health" "200"
          
          echo "All smoke tests passed!"
```

### Step 75: Rollback Workflow

สร้างไฟล์ `.github/workflows/rollback.yml`:

```yaml
name: Rollback Production

on:
  workflow_dispatch:  # manual trigger
    inputs:
      version:
        description: 'Version to rollback to (e.g., v1.2.3)'
        required: true
        type: string
      reason:
        description: 'Reason for rollback'
        required: true
        type: string

jobs:
  rollback:
    name: Rollback to ${{ inputs.version }}
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    environment: production

    steps:
      - name: Validate version format
        run: |
          if ! echo "${{ inputs.version }}" | grep -qE '^v[0-9]+\.[0-9]+\.[0-9]+$'; then
            echo "Invalid version format. Must be v1.2.3"
            exit 1
          fi

      - name: Notify Discord - Starting Rollback
        run: |
          curl -H "Content-Type: application/json" \
            -X POST \
            -d '{
              "content": "@here",
              "embeds": [{
                "title": "⚠️ Rolling Back Production",
                "description": "Rolling back to `${{ inputs.version }}`\nReason: ${{ inputs.reason }}",
                "color": 16776960
              }]
            }' \
            ${{ secrets.DISCORD_WEBHOOK_URL }}

      - name: Rollback Production
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            set -e
            
            VERSION="${{ inputs.version }}"
            DEPLOY_DIR=/opt/chuaikan
            
            echo "Rolling back to $VERSION..."
            
            # Login to registry
            echo "${{ secrets.GITHUB_TOKEN }}" | \
              docker login ghcr.io -u "${{ github.actor }}" --password-stdin
            
            # Pull version-specific images
            WEB_IMAGE="ghcr.io/${{ github.repository }}/web:${VERSION}"
            API_IMAGE="ghcr.io/${{ github.repository }}/api:${VERSION}"
            
            docker pull $WEB_IMAGE
            docker pull $API_IMAGE
            
            # Update .env to use old version
            sed -i "s|WEB_IMAGE=.*|WEB_IMAGE=${WEB_IMAGE}|g" $DEPLOY_DIR/.env
            sed -i "s|API_IMAGE=.*|API_IMAGE=${API_IMAGE}|g" $DEPLOY_DIR/.env
            sed -i "s|VERSION=.*|VERSION=${VERSION}|g" $DEPLOY_DIR/.env
            
            # Deploy
            cd $DEPLOY_DIR
            docker compose up -d --remove-orphans
            
            sleep 15
            
            # Health check
            if curl -sf http://localhost:3000/api/health; then
              echo "Rollback to $VERSION successful!"
            else
              echo "Health check failed after rollback!"
              exit 1
            fi

      - name: Notify Discord - Rollback Success
        if: success()
        run: |
          curl -H "Content-Type: application/json" \
            -X POST \
            -d '{
              "embeds": [{
                "title": "✅ Rollback Successful",
                "description": "Successfully rolled back to `${{ inputs.version }}`",
                "color": 3066993
              }]
            }' \
            ${{ secrets.DISCORD_WEBHOOK_URL }}
```

### Step 76: Deploy Script บน VPS

สร้างไฟล์ `scripts/deploy.sh` สำหรับรันบน VPS:

```bash
#!/bin/bash
# scripts/deploy.sh
# รันบน VPS โดย GitHub Actions

set -euo pipefail

DEPLOY_DIR=${DEPLOY_DIR:-/opt/chuaikan}
LOG_FILE="/var/log/chuaikan-deploy.log"
MAX_HEALTH_RETRIES=20
HEALTH_CHECK_INTERVAL=5

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

health_check() {
    local url=$1
    local retries=0
    
    while [ $retries -lt $MAX_HEALTH_RETRIES ]; do
        if curl -sf "$url" > /dev/null 2>&1; then
            log "Health check passed: $url"
            return 0
        fi
        retries=$((retries + 1))
        log "Health check attempt $retries/$MAX_HEALTH_RETRIES failed, retrying in ${HEALTH_CHECK_INTERVAL}s..."
        sleep $HEALTH_CHECK_INTERVAL
    done
    
    log "ERROR: Health check failed after $MAX_HEALTH_RETRIES attempts"
    return 1
}

# Main deploy logic
main() {
    log "=== Starting deployment ==="
    
    cd "$DEPLOY_DIR"
    
    # Pull new images
    log "Pulling Docker images..."
    docker compose pull
    
    # Rolling update
    log "Starting rolling update..."
    docker compose up -d --remove-orphans --no-build
    
    # Health check
    log "Running health checks..."
    health_check "http://localhost:3000/api/health"
    health_check "http://localhost:4000/health"
    
    # Clean up
    log "Cleaning up old images..."
    docker image prune -f --filter "until=24h"
    
    log "=== Deployment completed successfully ==="
    docker compose ps
}

main "$@"
```

---

## 🔧 Configuration Files

### package.json scripts ที่ต้องมี

```json
{
  "scripts": {
    "lint": "eslint . --ext .ts,.tsx --max-warnings 0",
    "type-check": "tsc --noEmit",
    "format:check": "prettier --check .",
    "format": "prettier --write .",
    "test:unit": "jest --testPathPattern=unit",
    "test:integration": "jest --testPathPattern=integration",
    "test:e2e": "playwright test",
    "test": "jest",
    "db:migrate": "prisma migrate deploy",
    "build": "turbo run build",
    "start": "node apps/web/server.js"
  }
}
```

### GitHub Environments Setup

```
Settings → Environments

Environment: staging
  - No protection rules (auto-deploy)
  - Secrets: STAGING_HOST, STAGING_USER, etc.

Environment: production
  - Required reviewers: 1 person (team lead)
  - Wait timer: 0 minutes
  - Deployment branches: tags matching v*.*.*
  - Secrets: VPS_HOST, VPS_USER, etc.
```

---

## 🧪 Testing

```bash
# Test 1: ตรวจสอบ workflow syntax
cd your-repo
gh workflow list

# Test 2: ทดสอบ run workflow manually
gh workflow run deploy-staging.yml

# Test 3: ดู workflow run status
gh run list --workflow=ci.yml --limit=5

# Test 4: ดู run details
gh run view <run-id> --log

# Test 5: ทดสอบ secrets ถูกตั้งค่าแล้ว
gh secret list

# Test 6: ทดสอบ SSH connection
ssh -i ~/.ssh/chuaikan_deploy_key deployer@YOUR_VPS_IP "docker ps"

# Test 7: ทดสอบ deploy script
ssh deployer@VPS "bash /opt/chuaikan/scripts/deploy.sh"

# Test 8: ดู production logs หลัง deploy
ssh deployer@VPS "cd /opt/chuaikan && docker compose logs --tail=100"
```

---

## ❌ Common Errors & Solutions

### Error 1: SSH Permission Denied

```
Error: ssh: handshake failed: ssh: unable to authenticate
```

**Solution:**
```bash
# ตรวจสอบ SSH key format
# GitHub Actions ต้องการ private key format
head -1 ~/.ssh/chuaikan_deploy_key
# ควรขึ้นต้นด้วย: -----BEGIN OPENSSH PRIVATE KEY-----

# ตรวจสอบ authorized_keys บน VPS
ssh root@VPS "cat /home/deployer/.ssh/authorized_keys"

# ตรวจสอบ permissions
ssh root@VPS "ls -la /home/deployer/.ssh/"
# authorized_keys ต้องเป็น 600
# .ssh directory ต้องเป็น 700
```

### Error 2: Docker Registry Auth Failed

```
Error: unauthorized: authentication required
```

**Solution:**
```yaml
# ต้องมี permissions ใน workflow
permissions:
  contents: read
  packages: write  # ต้องมีข้อนี้สำหรับ ghcr.io

# และ token ต้องมี packages:write scope
# Settings → Developer settings → Personal access tokens
```

### Error 3: Build Cache Miss

```
# Build ช้ามากแม้ไม่มีอะไรเปลี่ยน
```

**Solution:**
```yaml
# ตรวจสอบ cache key
- uses: docker/build-push-action@v5
  with:
    cache-from: type=gha        # GitHub Actions Cache
    cache-to: type=gha,mode=max

# หรือใช้ Registry cache
    cache-from: type=registry,ref=ghcr.io/org/image:buildcache
    cache-to: type=registry,ref=ghcr.io/org/image:buildcache,mode=max
```

### Error 4: Health Check Timeout

```
Health check failed after 20 attempts
```

**Solution:**
```bash
# เพิ่ม start_period ใน Dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=5 \
    CMD curl -f http://localhost:3000/api/health || exit 1

# หรือเพิ่ม sleep ใน deploy script
sleep 60  # รอให้ app start ก่อน health check
```

### Error 5: Concurrency Issues

```
# Multiple deploys running at same time
```

**Solution:**
```yaml
# ใช้ concurrency group
concurrency:
  group: production-deploy
  cancel-in-progress: false  # ไม่ cancel ถ้ากำลัง deploy production
```

---

## ✅ Checklist

### GitHub Repository Setup
- [ ] Secrets ทั้งหมดตั้งค่าแล้ว (VPS_HOST, VPS_SSH_KEY, etc.)
- [ ] Environments สร้างแล้ว (staging, production)
- [ ] Production environment มี required reviewers
- [ ] GHCR permissions ตั้งค่าแล้ว

### CI Workflow
- [ ] lint job ทำงานสำเร็จ
- [ ] test job ทำงานสำเร็จ (Node 20 และ 22)
- [ ] build job สร้าง Docker images สำเร็จ
- [ ] Images push ไป ghcr.io สำเร็จ

### Staging Deploy
- [ ] Push to develop trigger deploy สำเร็จ
- [ ] Health check pass หลัง deploy
- [ ] Discord notification ส่งสำเร็จ

### Production Deploy
- [ ] Create tag v1.0.0 trigger production deploy
- [ ] Requires approval ก่อน deploy
- [ ] Cloudflare cache purged หลัง deploy
- [ ] GitHub Release สร้างแล้ว
- [ ] Smoke tests pass

### Rollback
- [ ] Rollback workflow ทดสอบแล้ว
- [ ] สามารถ rollback ไป version ก่อนหน้าได้
- [ ] Discord notification ทำงาน

---

## 🔗 References

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitHub Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [appleboy/ssh-action](https://github.com/appleboy/ssh-action)
- [docker/build-push-action](https://github.com/docker/build-push-action)
- [GitHub Environments](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)

---
*Part 008 | Road to 1,000,000 Users/Day | chuaikan.com*
