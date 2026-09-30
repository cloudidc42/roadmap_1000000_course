# Part 055: ElastiCache / Memorystore (Managed Redis)
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Advanced
> **Steps:** 541-550
> **เวลาโดยประมาณ:** 4 ชั่วโมง
> **Prerequisites:** Part 052 (VPC Network), Part 054 (RDS)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ตั้งค่า ElastiCache Redis 7 แบบ Cluster Mode Enabled
- เลือก Node Type: cache.r6g.large (6.38 GB)
- ตั้งค่า Cluster: 3 shards × 2 replicas = 6 nodes
- ตั้งค่า Multi-AZ with Automatic Failover
- เปิด Encryption at rest และ in transit
- เขียน Terraform code สำหรับ ElastiCache
- เขียน Node.js Cluster Mode Client
- ตั้งค่า Backup และ Restore

---

## 📖 ทฤษฎีและแนวคิด

### Step 541: ElastiCache Redis Architecture

```
ElastiCache Redis Cluster Mode Enabled:

Shard 1 (Slot 0-5460):
├── Primary: cache.r6g.large (AZ: 1a)
└── Replica: cache.r6g.large (AZ: 1b)

Shard 2 (Slot 5461-10922):
├── Primary: cache.r6g.large (AZ: 1b)
└── Replica: cache.r6g.large (AZ: 1a)

Shard 3 (Slot 10923-16383):
├── Primary: cache.r6g.large (AZ: 1a)
└── Replica: cache.r6g.large (AZ: 1b)

Total: 3 shards × 2 nodes = 6 nodes
Memory per shard: 6.38 GB
Total memory: 3 × 6.38 = 19.14 GB
```

### Step 542: Cluster Mode vs Non-Cluster Mode

| Feature | Non-Cluster Mode | Cluster Mode |
|---------|-----------------|--------------|
| Sharding | ไม่มี (single shard) | มี (หลาย shards) |
| Max Memory | 6.38 GB (r6g.large) | ไม่จำกัด (add shards) |
| Multi-key ops | ได้ | จำกัด (ต้องอยู่ shard เดียวกัน) |
| Read scaling | Replica endpoint | แต่ละ shard มี replica |
| Failover | ได้ | ได้ (per shard) |
| **Best for** | Simple, <6GB | Large cache, high availability |

**สำหรับ chuaikan.com**: เลือก Cluster Mode เพราะ data >6GB และต้องการ HA

### Step 543: Redis Use Cases ใน chuaikan.com

```
1. Session Store (Hash):
   key: session:{session_id}
   value: {user_id, role, expires_at, ...}
   TTL: 24 hours

2. API Rate Limiting (Sorted Set + String):
   key: rate_limit:{user_id}:{minute}
   value: request_count
   TTL: 60 seconds

3. Cache API Responses (String):
   key: cache:api:/dogs/popular:v1
   value: JSON response
   TTL: 5 minutes

4. Real-time counters (String):
   key: counter:daily_active_users:2024-01-15
   value: number
   TTL: 2 days

5. Job Queue (List):
   key: queue:email_notifications
   value: List of job objects

6. Pub/Sub for real-time features:
   channel: notifications:{user_id}
   message: notification events
```

---

## ⚙️ Environment Setup

### ตรวจสอบ Security Group

```bash
# ตรวจสอบ DB Security Group อนุญาต port 6379 จาก app tier
aws ec2 describe-security-groups \
  --group-ids sg-xxxxxxxxx \
  --query 'SecurityGroups[0].IpPermissions'
```

---

## 🛠️ Step-by-Step Implementation

### Step 544: สร้าง ElastiCache Subnet Group

```bash
# สร้าง Cache Subnet Group
aws elasticache create-cache-subnet-group \
  --cache-subnet-group-name chuaikan-cache-subnet-group \
  --cache-subnet-group-description "Subnet group for chuaikan ElastiCache" \
  --subnet-ids subnet-db-1a subnet-db-1b subnet-db-1c
```

### Step 545: สร้าง ElastiCache Parameter Group

```bash
# สร้าง Parameter Group สำหรับ Redis 7
aws elasticache create-cache-parameter-group \
  --cache-parameter-group-name chuaikan-redis7-params \
  --cache-parameter-group-family redis7 \
  --description "Redis 7 parameters for chuaikan"

# ปรับแต่ง Parameters
aws elasticache modify-cache-parameter-group \
  --cache-parameter-group-name chuaikan-redis7-params \
  --parameter-name-values \
    ParameterName=maxmemory-policy,ParameterValue=allkeys-lru \
    ParameterName=activerehashing,ParameterValue=yes \
    ParameterName=lazyfree-lazy-eviction,ParameterValue=yes \
    ParameterName=lazyfree-lazy-expire,ParameterValue=yes \
    ParameterName=latency-tracking,ParameterValue=yes \
    ParameterName=slowlog-log-slower-than,ParameterValue=10000 \
    ParameterName=slowlog-max-len,ParameterValue=128
```

### Step 546: สร้าง ElastiCache Replication Group

```bash
# สร้าง Redis Cluster
aws elasticache create-replication-group \
  --replication-group-id chuaikan-redis-cluster \
  --replication-group-description "chuaikan Redis cluster" \
  --cache-node-type cache.r6g.large \
  --engine redis \
  --engine-version 7.1 \
  --num-node-groups 3 \
  --replicas-per-node-group 1 \
  --cache-parameter-group-name chuaikan-redis7-params \
  --cache-subnet-group-name chuaikan-cache-subnet-group \
  --security-group-ids sg-xxxxxxxxx \
  --automatic-failover-enabled \
  --multi-az-enabled \
  --at-rest-encryption-enabled \
  --transit-encryption-enabled \
  --auth-token "SuperSecureRedisToken123!" \
  --snapshot-retention-limit 5 \
  --snapshot-window "15:00-16:00" \
  --preferred-maintenance-window "sun:16:00-sun:17:00" \
  --tags Key=Environment,Value=production Key=Project,Value=chuaikan

# รอ cluster พร้อม
echo "Waiting for ElastiCache cluster (10-15 minutes)..."
aws elasticache wait replication-group-available \
  --replication-group-id chuaikan-redis-cluster
echo "Redis cluster is ready!"
```

---

## 🔧 Configuration Files

### Terraform ElastiCache Module

```hcl
# terraform/modules/elasticache/main.tf

resource "aws_elasticache_subnet_group" "main" {
  name        = "${var.project}-${var.environment}-cache-subnet-group"
  description = "Cache subnet group for ${var.project}"
  subnet_ids  = var.database_subnet_ids

  tags = var.tags
}

resource "aws_elasticache_parameter_group" "main" {
  name        = "${var.project}-${var.environment}-redis7-params"
  family      = "redis7"
  description = "Redis 7 parameters for ${var.project}"

  parameter {
    name  = "maxmemory-policy"
    value = "allkeys-lru"
  }

  parameter {
    name  = "activerehashing"
    value = "yes"
  }

  parameter {
    name  = "lazyfree-lazy-eviction"
    value = "yes"
  }

  parameter {
    name  = "lazyfree-lazy-expire"
    value = "yes"
  }

  parameter {
    name  = "slowlog-log-slower-than"
    value = "10000"
  }

  tags = var.tags
}

# Store Redis auth token in Secrets Manager
resource "random_password" "redis_auth_token" {
  length  = 32
  special = false  # Redis auth token ไม่รองรับ special characters บางตัว
}

resource "aws_secretsmanager_secret" "redis_auth_token" {
  name                    = "${var.project}/${var.environment}/redis/auth-token"
  description             = "Redis auth token for ${var.project}"
  recovery_window_in_days = 7

  tags = var.tags
}

resource "aws_secretsmanager_secret_version" "redis_auth_token" {
  secret_id     = aws_secretsmanager_secret.redis_auth_token.id
  secret_string = random_password.redis_auth_token.result
}

resource "aws_elasticache_replication_group" "main" {
  replication_group_id = "${var.project}-${var.environment}-redis"
  description          = "${var.project} Redis cluster"

  node_type            = var.cache_node_type
  engine               = "redis"
  engine_version       = "7.1"
  port                 = 6379

  # Cluster mode configuration
  num_node_groups         = var.num_shards
  replicas_per_node_group = var.replicas_per_shard

  parameter_group_name = aws_elasticache_parameter_group.main.name
  subnet_group_name    = aws_elasticache_subnet_group.main.name
  security_group_ids   = [var.database_security_group_id]

  # High Availability
  automatic_failover_enabled = true
  multi_az_enabled          = true

  # Security
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token                 = random_password.redis_auth_token.result
  auth_token_update_strategy = "ROTATE"

  # Backup
  snapshot_retention_limit = 5
  snapshot_window          = "15:00-16:00"

  # Maintenance
  maintenance_window = "sun:16:00-sun:17:00"

  # Auto minor version upgrade
  auto_minor_version_upgrade = true

  apply_immediately = false

  tags = merge(var.tags, {
    Name = "${var.project}-${var.environment}-redis"
  })

  lifecycle {
    prevent_destroy = true
  }
}
```

### Terraform Variables

```hcl
# terraform/modules/elasticache/variables.tf

variable "project" {
  type = string
}

variable "environment" {
  type = string
}

variable "database_subnet_ids" {
  type = list(string)
}

variable "database_security_group_id" {
  type = string
}

variable "cache_node_type" {
  type    = string
  default = "cache.r6g.large"
}

variable "num_shards" {
  type    = number
  default = 3
}

variable "replicas_per_shard" {
  type    = number
  default = 1
}

variable "tags" {
  type    = map(string)
  default = {}
}
```

### Node.js Redis Client (Cluster Mode)

```typescript
// src/redis/client.ts
import { Cluster, ClusterOptions } from 'ioredis';
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';

let redisCluster: Cluster | null = null;

async function getRedisAuthToken(): Promise<string> {
  const client = new SecretsManagerClient({ region: 'ap-southeast-1' });
  const command = new GetSecretValueCommand({
    SecretId: process.env.REDIS_SECRET_ARN || 'chuaikan/production/redis/auth-token'
  });
  const response = await client.send(command);
  return response.SecretString!;
}

export async function getRedisClient(): Promise<Cluster> {
  if (redisCluster) return redisCluster;

  const authToken = await getRedisAuthToken();
  
  const clusterNodes = process.env.REDIS_CLUSTER_ENDPOINT!.split(',').map(endpoint => {
    const [host, port] = endpoint.split(':');
    return { host, port: parseInt(port, 10) };
  });

  const options: ClusterOptions = {
    slotsRefreshTimeout: 2000,
    enableOfflineQueue: false,
    retryDelayOnFailover: 500,
    retryDelayOnClusterDown: 300,
    maxRedirections: 16,
    redisOptions: {
      password: authToken,
      tls: {},  // Enable TLS
      connectTimeout: 5000,
      commandTimeout: 3000,
      keepAlive: 30000,
      noDelay: true,
    },
    natMap: {},  // สำหรับ EKS ภายใน VPC
  };

  redisCluster = new Cluster(clusterNodes, options);

  redisCluster.on('error', (err) => {
    console.error('Redis cluster error:', err);
  });

  redisCluster.on('connect', () => {
    console.log('Redis cluster connected');
  });

  redisCluster.on('+node', (node) => {
    console.log('Redis node added:', node.options.host);
  });

  return redisCluster;
}

// Session management
export async function setSession(sessionId: string, data: object, ttl = 86400): Promise<void> {
  const redis = await getRedisClient();
  await redis.setex(`session:${sessionId}`, ttl, JSON.stringify(data));
}

export async function getSession(sessionId: string): Promise<object | null> {
  const redis = await getRedisClient();
  const data = await redis.get(`session:${sessionId}`);
  return data ? JSON.parse(data) : null;
}

// Rate limiting
export async function checkRateLimit(userId: string, limit = 100, windowSeconds = 60): Promise<boolean> {
  const redis = await getRedisClient();
  const key = `rate_limit:${userId}:${Math.floor(Date.now() / 1000 / windowSeconds)}`;
  
  const pipeline = redis.pipeline();
  pipeline.incr(key);
  pipeline.expire(key, windowSeconds * 2);
  const results = await pipeline.exec();
  
  const count = results?.[0]?.[1] as number;
  return count <= limit;
}

// Cache with tags (สำหรับ invalidation)
export async function cacheSet(key: string, value: any, ttl: number, tags: string[] = []): Promise<void> {
  const redis = await getRedisClient();
  const pipeline = redis.pipeline();
  
  pipeline.setex(`cache:${key}`, ttl, JSON.stringify(value));
  
  // Store key in tag sets for easy invalidation
  for (const tag of tags) {
    pipeline.sadd(`tag:${tag}`, `cache:${key}`);
    pipeline.expire(`tag:${tag}`, ttl);
  }
  
  await pipeline.exec();
}

export async function cacheGet(key: string): Promise<any | null> {
  const redis = await getRedisClient();
  const data = await redis.get(`cache:${key}`);
  return data ? JSON.parse(data) : null;
}

export async function invalidateTag(tag: string): Promise<void> {
  const redis = await getRedisClient();
  const keys = await redis.smembers(`tag:${tag}`);
  
  if (keys.length > 0) {
    const pipeline = redis.pipeline();
    for (const key of keys) {
      pipeline.del(key);
    }
    pipeline.del(`tag:${tag}`);
    await pipeline.exec();
  }
}
```

### Kubernetes Secret สำหรับ Redis

```yaml
# k8s/secrets/redis-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: redis-config
  namespace: production
type: Opaque
stringData:
  REDIS_CLUSTER_ENDPOINT: "clustercfg.chuaikan-production-redis.xxxxxx.apse1.cache.amazonaws.com:6379"
  REDIS_SECRET_ARN: "arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:chuaikan/production/redis/auth-token"
```

---

## 🧪 Testing

```bash
# ตรวจสอบ cluster status
aws elasticache describe-replication-groups \
  --replication-group-id chuaikan-production-redis \
  --query 'ReplicationGroups[0].{Status:Status,NodeGroups:NodeGroups[*].{ID:NodeGroupId,Status:Status}}'

# ทดสอบ connection จาก EKS pod
kubectl run redis-test --image=redis:7 -n production --rm -it -- \
  redis-cli -h clustercfg.chuaikan-production-redis.xxxxxx.apse1.cache.amazonaws.com \
            -p 6379 \
            --tls \
            -a "your-auth-token" \
            -c \
            CLUSTER INFO

# ทดสอบ key distribution
kubectl run redis-test --image=redis:7 -n production --rm -it -- \
  redis-cli -h clustercfg.chuaikan-production-redis.xxxxxx.apse1.cache.amazonaws.com \
            -p 6379 --tls -a "your-auth-token" -c \
            SET test:key "hello world"

# ดู metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/ElastiCache \
  --metric-name CacheHits \
  --dimensions Name=ReplicationGroupId,Value=chuaikan-production-redis \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Sum

# ทดสอบ failover
aws elasticache test-failover \
  --replication-group-id chuaikan-production-redis \
  --node-group-id 0001
```

---

## ❌ Common Errors & Solutions

### Error 1: MOVED error (Cluster Mode)

```
ReplyError: MOVED 7638 10.0.20.50:6379
```

**แก้ไข:**
```typescript
// ต้องใช้ cluster client ที่รองรับ MOVED redirects
// ioredis Cluster client จัดการให้อัตโนมัติ
// ตรวจสอบว่า option enableReadyCheck = true
const redis = new Cluster(nodes, {
  enableReadyCheck: true,
  slotsRefreshTimeout: 2000,
  maxRedirections: 16  // เพิ่มค่านี้ถ้า MOVED เกิดบ่อย
});
```

### Error 2: Auth failed

```
ReplyError: WRONGPASS invalid username-password pair
```

**แก้ไข:**
```bash
# ตรวจสอบ auth token ใน Secrets Manager
aws secretsmanager get-secret-value \
  --secret-id chuaikan/production/redis/auth-token \
  --query SecretString --output text

# ตรวจสอบ TLS cert
openssl s_client -connect clustercfg.xxxx.cache.amazonaws.com:6379
```

### Error 3: Cross-slot command error

```
ReplyError: CROSSSLOT Keys in request don't hash to the same slot
```

**แก้ไข:**
```typescript
// ใช้ hash tags เพื่อให้ keys ไปอยู่ shard เดียวกัน
// keys ที่มี {} จะใช้ {} content เป็น hash key
await redis.mset(
  '{user:123}:name', 'John',
  '{user:123}:email', 'john@example.com'
);
// ทั้งสอง keys จะอยู่ shard เดียวกันเพราะ hash tag {user:123} เหมือนกัน
```

---

## 📊 Cost Estimation

```
ElastiCache Cluster Mode (3 shards × 2 nodes):
├── cache.r6g.large × 6 nodes: $0.166/hr × 6 × 730h = $727/month
├── 1-year Reserved: $0.098/hr × 6 × 730h = $429/month (ประหยัด 41%)
└── Backup Storage (5 snapshots × 20GB): ~$1/month

TOTAL: ~$430/month (reserved)
```

---

## ✅ Checklist

- [ ] Step 541: เข้าใจ ElastiCache Cluster Mode Architecture
- [ ] Step 542: เปรียบเทียบ Cluster Mode vs Non-Cluster Mode
- [ ] Step 543: วางแผน Redis use cases ใน chuaikan.com
- [ ] Step 544: สร้าง ElastiCache Subnet Group
- [ ] Step 545: สร้างและปรับแต่ง Parameter Group
- [ ] Step 546: สร้าง Redis Cluster (3 shards)
- [ ] Step 547: ตั้งค่า TLS และ Auth Token
- [ ] Step 548: เขียน Node.js Cluster Client
- [ ] Step 549: ทดสอบ connection และ failover
- [ ] Step 550: ตั้งค่า backup และ monitoring

---

## 🔗 References

- [ElastiCache Redis Cluster Mode](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Clusters.Create.html)
- [ioredis Cluster Mode](https://github.com/redis/ioredis#cluster)
- [Redis Commands Reference](https://redis.io/commands/)
- [ElastiCache Best Practices](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/BestPractices.html)

---
*Part 055 | Road to 1,000,000 Users/Day | chuaikan.com*
