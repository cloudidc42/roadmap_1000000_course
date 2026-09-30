# Part 097: Platform Engineering (IDP)

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** World Class
> **Steps:** 961-970
> **เวลาโดยประมาณ:** 5 ชั่วโมง
> **Prerequisites:** Part 031 (Kubernetes), Part 094 (Distributed Tracing)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

เมื่อ Engineering Team โตถึง 20+ คน การสร้าง service ใหม่กลายเป็นงานหนัก Platform Engineering แก้ปัญหานี้ด้วย Internal Developer Platform (IDP) ใน Part นี้เราจะ:

- เข้าใจ IDP และทำไมถึงสำคัญ
- ติดตั้ง Backstage.io Software Catalog
- สร้าง Self-Service Templates
- วัด Developer Experience ด้วย DORA Metrics

---

## 📖 ทฤษฎีและแนวคิด

### 1. Internal Developer Platform (IDP)

**IDP คือ:**
> Platform ที่ Platform Team สร้างขึ้นเพื่อให้ Product Teams สามารถ build, deploy, และ operate services ได้โดยอิสระ

**ปัญหาก่อนมี IDP:**
```
Product Developer: "ฉันต้องการสร้าง service ใหม่"
→ ถาม Platform Team: "ต้องทำอะไรบ้าง?"
→ Platform Team: สร้าง Terraform, Kubernetes configs, CI/CD pipeline, monitoring...
→ รอ 1-2 weeks
→ สุดท้าย: Developer ยังไม่รู้จะ debug ยังไง

ผลลัพธ์: Platform Team overwhelmed, Product Team frustrated
```

**ด้วย IDP:**
```
Product Developer: "ฉันต้องการสร้าง service ใหม่"
→ เปิด Backstage, เลือก template "Node.js Microservice"
→ กรอกข้อมูล: service name, team, language
→ กด "Create"
→ 5 นาทีต่อมา: GitHub repo + CI/CD + Kubernetes configs + monitoring พร้อมหมด!
```

### 2. Backstage.io

Backstage.io พัฒนาโดย Spotify และ open-source ในปี 2020 เป็น framework สำหรับสร้าง IDP

**Core Features:**
- **Software Catalog:** รายการทุก service, API, library
- **Software Templates:** Self-service สร้าง service ใหม่
- **Tech Docs:** Documentation as Code
- **Plugins:** ต่อกับ PagerDuty, GitHub, Kubernetes, etc.

### 3. Golden Paths

Golden Path คือ "วิธีที่แนะนำ" ในการทำงานใน chuaikan.com:

```
Golden Path: Node.js Microservice

1. Language: TypeScript
2. Framework: Express + Fastify
3. Database: PostgreSQL (via Prisma ORM)
4. Cache: Redis (via ioredis)
5. Testing: Jest + Supertest
6. CI/CD: GitHub Actions → ECR → ArgoCD
7. Monitoring: Prometheus metrics + Grafana dashboard (auto-created)
8. Tracing: OpenTelemetry (pre-configured)
9. Logging: Winston → Loki
10. Secrets: AWS Secrets Manager
```

---

## ⚙️ Environment Setup

### ติดตั้ง Backstage

```bash
# สร้าง Backstage app ใหม่
npx @backstage/create-app@latest --name chuaikan-backstage

cd chuaikan-backstage

# ติดตั้ง dependencies
yarn install

# รัน locally
yarn dev
# เปิด http://localhost:3000
```

```bash
# Build Docker image สำหรับ production
yarn build:backend

docker build -t chuaikan-backstage:latest \
  --file packages/backend/Dockerfile .

docker push YOUR_REGISTRY/chuaikan-backstage:latest
```

---

## 🛠️ Step-by-Step Implementation

### Step 961: Software Catalog

```yaml
# catalog-info.yaml — ทุก service ต้องมีไฟล์นี้ใน root ของ repo

apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: feed-service
  description: Personalized feed generation for chuaikan.com users
  annotations:
    github.com/project-slug: chuaikan/feed-service
    backstage.io/kubernetes-id: feed-service
    backstage.io/kubernetes-namespace: production
    prometheus.io/rule: "namespace=production,container=feed-service"
    pagerduty.com/integration-key: "PAGERDUTY_KEY"
    grafana/dashboard-selector: service=feed-service
  tags:
    - nodejs
    - typescript
    - microservice
    - feed
  links:
    - url: https://grafana.internal/d/feed-service
      title: Grafana Dashboard
      icon: dashboard
    - url: https://jaeger.internal/search?service=feed-service
      title: Jaeger Traces
      icon: search
    - url: https://runbook.chuaikan.com/feed-service
      title: Runbook
      icon: book

spec:
  type: service
  lifecycle: production
  owner: group:feed-team
  system: chuaikan-platform
  providesApis:
    - feed-api
  consumesApis:
    - user-api
    - post-api
  dependsOn:
    - resource:default/postgres-main
    - resource:default/redis-cluster
    - component:default/post-service
```

```yaml
# api-definition.yaml — ระบุ API ของ service
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: feed-api
  description: Feed Service REST API
spec:
  type: openapi
  lifecycle: production
  owner: group:feed-team
  definition:
    $text: ./openapi.yaml
```

### Step 962: Software Templates (Self-Service)

```yaml
# template-nodejs-service.yaml

apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: nodejs-microservice
  title: Node.js Microservice
  description: Create a production-ready Node.js microservice for chuaikan.com
  tags:
    - nodejs
    - typescript
    - recommended

spec:
  owner: group:platform-team
  type: service

  parameters:
    - title: Service Information
      required:
        - name
        - description
        - owner
      properties:
        name:
          title: Service Name
          type: string
          description: Name of the service (lowercase, hyphens allowed)
          pattern: '^[a-z][a-z0-9-]{2,30}[a-z0-9]$'
        description:
          title: Description
          type: string
          description: What does this service do?
        owner:
          title: Team Owner
          type: string
          description: Which team owns this service?
          ui:field: OwnerPicker
          ui:options:
            allowedKinds:
              - Group
    
    - title: Database Configuration
      properties:
        needsDatabase:
          title: Does this service need a database?
          type: boolean
          default: false
        databaseType:
          title: Database Type
          type: string
          enum: ['postgresql', 'mongodb', 'none']
          default: 'postgresql'
    
    - title: Infrastructure
      properties:
        minReplicas:
          title: Minimum Replicas
          type: integer
          default: 2
          minimum: 1
          maximum: 20
        maxReplicas:
          title: Maximum Replicas
          type: integer
          default: 10
          minimum: 2
          maximum: 100
        cpuRequest:
          title: CPU Request
          type: string
          default: "100m"
        memoryRequest:
          title: Memory Request
          type: string
          default: "128Mi"

  steps:
    - id: fetch-base
      name: Fetch Base Template
      action: fetch:template
      input:
        url: ./skeleton
        values:
          name: ${{ parameters.name }}
          description: ${{ parameters.description }}
          owner: ${{ parameters.owner }}
          needsDatabase: ${{ parameters.needsDatabase }}
          minReplicas: ${{ parameters.minReplicas }}
          maxReplicas: ${{ parameters.maxReplicas }}
    
    - id: create-github-repo
      name: Create GitHub Repository
      action: github:repo:create
      input:
        repoUrl: github.com/chuaikan/${{ parameters.name }}
        description: ${{ parameters.description }}
        repoVisibility: private
        defaultBranch: main
        topics:
          - microservice
          - chuaikan
    
    - id: push-to-github
      name: Push to GitHub
      action: github:repo:push
      input:
        repoUrl: github.com/chuaikan/${{ parameters.name }}
        defaultBranch: main
    
    - id: create-kubernetes-namespace
      name: Create Kubernetes Namespace
      action: kubernetes:apply
      input:
        manifest: |
          apiVersion: v1
          kind: Namespace
          metadata:
            name: ${{ parameters.name }}
            labels:
              team: ${{ parameters.owner }}
    
    - id: register-catalog
      name: Register in Software Catalog
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps['create-github-repo'].output.repoContentsUrl }}
        catalogInfoPath: /catalog-info.yaml
    
    - id: create-slack-channel
      name: Create Slack Channel
      action: slack:channel:create
      input:
        channelName: "svc-${{ parameters.name }}"
        description: "Alerts and discussions for ${{ parameters.name }}"
  
  output:
    links:
      - title: View in GitHub
        url: ${{ steps['create-github-repo'].output.repoUrl }}
      - title: View in Catalog
        url: ${{ steps['register-catalog'].output.entityRef }}
      - title: Open in IDE
        icon: web
        url: ${{ steps['create-github-repo'].output.repoUrl }}
```

### Step 963: DORA Metrics Dashboard

```javascript
// dora-metrics.js
// ดึงและแสดง DORA Metrics จาก GitHub + deployment data

const { Octokit } = require('@octokit/rest');
const octokit = new Octokit({ auth: process.env.GITHUB_TOKEN });

async function getDeploymentFrequency(repo, days = 30) {
  const since = new Date(Date.now() - days * 24 * 60 * 60 * 1000).toISOString();
  
  const deployments = await octokit.repos.listDeployments({
    owner: 'chuaikan',
    repo,
    environment: 'production',
    per_page: 100,
  });
  
  // กรอง deployments ใน time window
  const recentDeployments = deployments.data.filter(
    d => new Date(d.created_at) > new Date(since)
  );
  
  return {
    total: recentDeployments.length,
    perDay: recentDeployments.length / days,
    // Elite: multiple times/day, High: once/day-week
    rating: recentDeployments.length / days > 1 ? 'elite' 
          : recentDeployments.length / days > 0.14 ? 'high'  // once/week
          : recentDeployments.length / days > 0.03 ? 'medium' // once/month
          : 'low',
  };
}

async function getLeadTimeForChanges(repo, days = 30) {
  // ดู PRs ที่ merged ใน 30 วัน
  const prs = await octokit.pulls.list({
    owner: 'chuaikan',
    repo,
    state: 'closed',
    sort: 'updated',
    direction: 'desc',
    per_page: 100,
  });
  
  const mergedPRs = prs.data.filter(pr => 
    pr.merged_at && 
    new Date(pr.merged_at) > new Date(Date.now() - days * 24 * 60 * 60 * 1000)
  );
  
  const leadTimes = mergedPRs.map(pr => {
    const firstCommit = new Date(pr.created_at);
    const deployed = new Date(pr.merged_at);
    return (deployed - firstCommit) / (1000 * 60 * 60); // hours
  });
  
  const avgLeadTime = leadTimes.reduce((a, b) => a + b, 0) / leadTimes.length;
  
  return {
    avgHours: avgLeadTime,
    // Elite: < 1 hour, High: 1 day-1 week
    rating: avgLeadTime < 1 ? 'elite'
          : avgLeadTime < 24 ? 'high'
          : avgLeadTime < 168 ? 'medium'  // 1 week
          : 'low',
  };
}

async function getChangeFailureRate(repo, days = 30) {
  // ดู deployments ที่ fail หรือ rollback
  const deployments = await octokit.repos.listDeployments({
    owner: 'chuaikan', repo,
    environment: 'production',
  });
  
  const recentDeployments = deployments.data.filter(
    d => new Date(d.created_at) > new Date(Date.now() - days * 24 * 60 * 60 * 1000)
  );
  
  let failures = 0;
  for (const deployment of recentDeployments) {
    const statuses = await octokit.repos.listDeploymentStatuses({
      owner: 'chuaikan', repo,
      deployment_id: deployment.id,
    });
    const latestStatus = statuses.data[0]?.state;
    if (latestStatus === 'failure' || latestStatus === 'error') {
      failures++;
    }
  }
  
  const failureRate = failures / recentDeployments.length;
  
  return {
    failures,
    total: recentDeployments.length,
    rate: failureRate,
    ratePercent: (failureRate * 100).toFixed(1),
    // Elite: < 5%, High: 5-10%
    rating: failureRate < 0.05 ? 'elite'
          : failureRate < 0.10 ? 'high'
          : failureRate < 0.15 ? 'medium'
          : 'low',
  };
}

// แสดง DORA Metrics สำหรับทุก service
async function printDoraMetrics() {
  const services = ['feed-service', 'sos-service', 'user-service', 'notification-service'];
  
  console.log('\n=== DORA Metrics Dashboard ===\n');
  console.log('Service'.padEnd(25) + 'Deploy Freq'.padEnd(15) + 'Lead Time'.padEnd(15) + 'Failure Rate');
  console.log('-'.repeat(70));
  
  for (const service of services) {
    const [deployFreq, leadTime, failureRate] = await Promise.all([
      getDeploymentFrequency(service),
      getLeadTimeForChanges(service),
      getChangeFailureRate(service),
    ]);
    
    console.log(
      service.padEnd(25) +
      `${deployFreq.perDay.toFixed(1)}/day`.padEnd(15) +
      `${leadTime.avgHours.toFixed(1)}h`.padEnd(15) +
      `${failureRate.ratePercent}%`
    );
  }
}

printDoraMetrics();
```

### Step 964: Platform Team vs Product Team Model

```
chuaikan.com Team Topology:

┌─────────────────────────────────────────────────────────┐
│                  Platform Team                          │
│  "เราสร้าง tooling เพื่อให้ Product Teams productive"  │
│                                                         │
│  Infrastructure │ Developer Experience │ Security       │
└─────────────────────────────────────────────────────────┘
         │ Provides                    │ Provides
         ▼                             ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Feed Team   │  │  SOS Team    │  │ User/Auth    │
│              │  │              │  │ Team         │
│  "เราสร้าง  │  │  "เราสร้าง  │  │              │
│  products"   │  │  products"   │  │              │
└──────────────┘  └──────────────┘  └──────────────┘
```

**Platform Team SLA:**
- Self-service template: สร้าง service ใหม่ < 10 นาที
- Documentation: ทุก service มี runbook
- On-call: Platform incidents ตอบสนองใน 15 นาที

---

## 🔧 Configuration Files

### ArgoCD ApplicationSet สำหรับ New Services

```yaml
# applicationset.yaml — Auto-create ArgoCD app เมื่อมี catalog entry ใหม่

apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: chuaikan-services
  namespace: argocd
spec:
  generators:
    # ดู services จาก Backstage catalog
    - list:
        elements:
          - service: feed-service
            team: feed-team
            namespace: feed
          - service: sos-service
            team: sos-team
            namespace: sos
          - service: user-service
            team: user-team
            namespace: user
  
  template:
    metadata:
      name: '{{service}}'
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: https://github.com/chuaikan/helm-charts
        targetRevision: HEAD
        path: 'services/{{service}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

---

## 🧪 Testing

### Platform Self-Service Test

```bash
#!/bin/bash
# test-self-service.sh — ทดสอบว่า template สร้าง service ได้ครบ

set -e

echo "=== Platform Self-Service Test ==="

SERVICE_NAME="test-$(date +%s)"

# 1. สร้าง service ผ่าน Backstage API
echo "Creating service via template..."
RESPONSE=$(curl -s -X POST \
  https://backstage.internal.chuaikan.com/api/scaffolder/v2/tasks \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $BACKSTAGE_TOKEN" \
  -d "{
    \"templateRef\": \"template:default/nodejs-microservice\",
    \"values\": {
      \"name\": \"$SERVICE_NAME\",
      \"description\": \"Test service\",
      \"owner\": \"group:platform-team\",
      \"needsDatabase\": false,
      \"minReplicas\": 1,
      \"maxReplicas\": 3
    }
  }")

TASK_ID=$(echo $RESPONSE | jq -r '.id')
echo "Task ID: $TASK_ID"

# 2. รอ task เสร็จ
echo "Waiting for task to complete..."
timeout 300 bash -c "
  while true; do
    STATUS=\$(curl -s https://backstage.internal.chuaikan.com/api/scaffolder/v2/tasks/$TASK_ID \
      -H 'Authorization: Bearer $BACKSTAGE_TOKEN' | jq -r '.status')
    if [ \"\$STATUS\" = 'completed' ]; then break; fi
    if [ \"\$STATUS\" = 'failed' ]; then echo 'Task failed!'; exit 1; fi
    sleep 5
  done
"

# 3. ตรวจว่า GitHub repo ถูกสร้าง
echo "Checking GitHub repo..."
gh repo view chuaikan/$SERVICE_NAME > /dev/null
echo "✅ GitHub repo exists"

# 4. ตรวจว่า service อยู่ใน Backstage catalog
echo "Checking Backstage catalog..."
curl -s https://backstage.internal.chuaikan.com/api/catalog/entities/by-name/component/default/$SERVICE_NAME \
  -H "Authorization: Bearer $BACKSTAGE_TOKEN" | jq '.metadata.name'
echo "✅ Service in catalog"

# Cleanup
gh repo delete chuaikan/$SERVICE_NAME --yes
echo "✅ All tests passed!"
```

---

## ❌ Common Errors & Solutions

### Error 1: Backstage Catalog ไม่ sync

```
ปัญหา: catalog-info.yaml แก้ไขแล้ว แต่ Backstage ยังแสดงข้อมูลเก่า
```

```bash
# Force refresh catalog entity
curl -X POST \
  https://backstage.internal.chuaikan.com/api/catalog/refresh \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $BACKSTAGE_TOKEN" \
  -d '{"entityRef": "component:default/feed-service"}'
```

### Error 2: Template ล้มเหลวกลางทาง

```bash
# ดู task logs
curl https://backstage.internal.chuaikan.com/api/scaffolder/v2/tasks/$TASK_ID/eventstream \
  -H "Authorization: Bearer $BACKSTAGE_TOKEN"

# Dry-run template ก่อน
backstage-cli package start --role app \
  --check template/template-nodejs-service.yaml
```

---

## ✅ Checklist

- [ ] Backstage ติดตั้งและ accessible สำหรับ engineers ทุกคน
- [ ] ทุก production service มี catalog-info.yaml
- [ ] Node.js Microservice template ทำงานได้
- [ ] Template สร้าง service ใหม่ได้ใน < 10 นาที
- [ ] Software Catalog แสดง dependencies ถูกต้อง
- [ ] DORA Metrics dashboard พร้อมใช้
- [ ] TechDocs (MkDocs) สำหรับทุก service
- [ ] Backstage ต่อกับ PagerDuty, Grafana, GitHub
- [ ] Golden Path documentation เขียนแล้ว
- [ ] Platform Team SLA ชัดเจนและวัดได้

---

## 🔗 References

- [Backstage.io Documentation](https://backstage.io/docs)
- [DORA Metrics](https://dora.dev/research/2022/dora-report/)
- [Team Topologies — Matthew Skelton](https://teamtopologies.com/)
- [Google's IDP Guide](https://cloud.google.com/blog/products/devops-sre/using-the-four-keys-to-measure-your-devops-performance)
- [Spotify Engineering Blog](https://engineering.atspotify.com/)

---

*Part 097 | Road to 1,000,000 Users/Day | chuaikan.com*
