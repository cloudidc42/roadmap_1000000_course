# Part 018: Logging ด้วย ELK Stack
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Beginner
> **Steps:** 171-180
> **เวลาโดยประมาณ:** 10 ชั่วโมง
> **Prerequisites:** Part 017 (Authentication & Authorization)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

1. Logging philosophy: structured logging (JSON format) และทำไมถึงสำคัญ
2. Log levels: ERROR, WARN, INFO, DEBUG, TRACE - เมื่อไหร่ใช้อะไร
3. Winston logger setup สำหรับ Node.js 22
4. Pino logger: ทางเลือกที่เร็วกว่าสำหรับ high-throughput
5. ELK Stack installation: Elasticsearch 8, Logstash, Kibana บน Ubuntu 24.04
6. Filebeat สำหรับ log shipping จาก Node.js log files
7. Logstash pipeline config สำหรับ parsing Node.js logs
8. Kibana dashboard สำหรับ chuaikan.com
9. Log retention policy (30 วันแบบ hot, 90 วันแบบ cold)
10. Log masking สำหรับ sensitive data
11. Correlation IDs สำหรับ tracing requests ข้าม services
12. Alert เมื่อ error spike

---

## 📖 ทฤษฎีและแนวคิด

### ทำไม Structured Logging ถึงสำคัญ

```
Plain text logging (ไม่ดี):
[2024-01-15 10:30:45] ERROR: User login failed for email test@example.com

Problems:
- Parse ยาก
- Search ยาก (ต้อง grep)
- Filter ไม่ได้
- Aggregate ไม่ได้

Structured logging (ดี):
{
  "timestamp": "2024-01-15T10:30:45.123Z",
  "level": "error",
  "message": "User login failed",
  "service": "auth",
  "userId": null,
  "email": "test@example.com",
  "errorCode": "INVALID_CREDENTIALS",
  "requestId": "req_abc123",
  "ip": "1.2.3.4",
  "userAgent": "Mozilla/5.0..."
}

Benefits:
✓ Query ด้วย field ได้ทันที
✓ Count errors by type, user, service
✓ Set alerts เมื่อ error rate สูง
✓ Correlate logs ข้าม services
```

### Log Levels Guide

```
TRACE  - ละเอียดสุด, ใช้เมื่อ debug หาสาเหตุ, ไม่ใช้ใน production
DEBUG  - ข้อมูล debug, ใช้ใน development, ปิดใน production
INFO   - ข้อมูล normal operation (request received, user login, alert created)
WARN   - สิ่งที่น่ากังวล แต่ไม่ crash (retry attempt, deprecated API used)
ERROR  - เกิดข้อผิดพลาด ต้องแก้ไข (DB connection failed, payment failed)
FATAL  - ระบบ crash ต้องหยุดทำงาน (config missing, port already in use)

Level Priority: FATAL > ERROR > WARN > INFO > DEBUG > TRACE

Production config:
  service: INFO (log INFO, WARN, ERROR, FATAL)
  cron jobs: WARN (log WARN, ERROR, FATAL only)
  
Development config:
  All services: DEBUG
```

### ELK Stack Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     ELK Stack Flow                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐    ┌──────────┐    ┌───────────────────────┐ │
│  │ Node.js App  │───>│ Filebeat │───>│ Logstash              │ │
│  │ (Winston/    │    │ (Agent)  │    │ (Parse, Transform,    │ │
│  │  Pino)       │    │          │    │  Enrich)              │ │
│  └──────────────┘    └──────────┘    └──────────┬────────────┘ │
│                                                  │              │
│  ┌──────────────┐    ┌──────────┐                │              │
│  │ Kibana       │<───│ Elastic  │<───────────────┘              │
│  │ (Visualize,  │    │ search   │                               │
│  │  Alert)      │    │ (Store,  │                               │
│  └──────────────┘    │  Index)  │                               │
│                      └──────────┘                               │
└─────────────────────────────────────────────────────────────────┘

Data Flow:
1. Node.js app writes JSON logs to files
2. Filebeat watches log files and ships to Logstash
3. Logstash parses, enriches and sends to Elasticsearch
4. Kibana reads from Elasticsearch and visualizes
```

---

## ⚙️ Environment Setup

### Step 171: ติดตั้ง Java (Elasticsearch ต้องการ)

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install OpenJDK 21 (required by Elasticsearch 8)
sudo apt install -y openjdk-21-jdk

# Verify
java -version
# openjdk version "21.0.x" ...
```

### Step 172: ติดตั้ง Elasticsearch 8

```bash
# Import Elasticsearch GPG key
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | \
  sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

# Add Elasticsearch repository
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] \
  https://artifacts.elastic.co/packages/8.x/apt stable main" | \
  sudo tee /etc/apt/sources.list.d/elastic-8.x.list

# Install Elasticsearch
sudo apt update && sudo apt install elasticsearch -y

# Start and enable service
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch

# ดู initial password (สำคัญมาก! เก็บไว้)
sudo cat /etc/elasticsearch/elasticsearch.yml
# หรือดูจาก setup output
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

```bash
# Configure Elasticsearch for production
sudo nano /etc/elasticsearch/elasticsearch.yml
```

```yaml
# /etc/elasticsearch/elasticsearch.yml
cluster.name: chuaikan-cluster
node.name: chuaikan-node-1

# Storage path
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch

# Network
network.host: 127.0.0.1
http.port: 9200

# Discovery
discovery.type: single-node

# Security (enable for production)
xpack.security.enabled: true
xpack.security.enrollment.enabled: true
xpack.security.http.ssl:
  enabled: false  # Enable in production with real cert

# Memory lock (prevent swapping)
bootstrap.memory_lock: true

# Index settings
indices.query.bool.max_clause_count: 8096
```

```bash
# Set JVM heap size (เป็น 50% ของ RAM, max 32GB)
sudo nano /etc/elasticsearch/jvm.options.d/heap.options
```

```
-Xms2g
-Xmx2g
```

```bash
# Test Elasticsearch
curl -u elastic:YOUR_PASSWORD http://localhost:9200

# Expected response:
# {
#   "name": "chuaikan-node-1",
#   "cluster_name": "chuaikan-cluster",
#   "version": { "number": "8.x.x" },
#   "tagline": "You Know, for Search"
# }
```

### Step 173: ติดตั้ง Logstash

```bash
# Logstash อยู่ใน repo เดียวกับ Elasticsearch แล้ว
sudo apt install logstash -y

sudo systemctl enable logstash
sudo systemctl start logstash
```

### Step 174: ติดตั้ง Kibana

```bash
sudo apt install kibana -y

# Configure Kibana
sudo nano /etc/kibana/kibana.yml
```

```yaml
# /etc/kibana/kibana.yml
server.port: 5601
server.host: "0.0.0.0"
server.name: "chuaikan-kibana"

# Elasticsearch connection
elasticsearch.hosts: ["http://localhost:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "KIBANA_PASSWORD"

# Logging
logging.appenders:
  file:
    type: file
    fileName: /var/log/kibana/kibana.log
    layout:
      type: json

# Internationalization
i18n.locale: "en"
```

```bash
# Set kibana_system password
curl -u elastic:YOUR_ELASTIC_PASSWORD \
  -X POST http://localhost:9200/_security/user/kibana_system/_password \
  -H "Content-Type: application/json" \
  -d '{"password": "KIBANA_PASSWORD"}'

sudo systemctl enable kibana
sudo systemctl start kibana
```

### Step 175: ติดตั้ง Filebeat

```bash
sudo apt install filebeat -y

# Configure Filebeat
sudo nano /etc/filebeat/filebeat.yml
```

---

## 🛠️ Step-by-Step Implementation

### Step 176: Winston Logger Setup

```typescript
// src/lib/logger/winston.ts
import winston from 'winston';
import { ElasticsearchTransport } from 'winston-elasticsearch';

// Custom log format
const jsonFormat = winston.format.combine(
  winston.format.timestamp({ format: 'ISO' }),
  winston.format.errors({ stack: true }),
  winston.format.json()
);

// Mask sensitive data
const maskSensitiveData = winston.format((info) => {
  const sensitiveKeys = ['password', 'token', 'secret', 'authorization', 'phone', 'creditCard'];
  
  function maskValue(obj: Record<string, unknown>): Record<string, unknown> {
    const masked = { ...obj };
    for (const key of Object.keys(masked)) {
      const lowerKey = key.toLowerCase();
      if (sensitiveKeys.some(s => lowerKey.includes(s))) {
        masked[key] = '[REDACTED]';
      } else if (typeof masked[key] === 'object' && masked[key] !== null) {
        masked[key] = maskValue(masked[key] as Record<string, unknown>);
      }
    }
    return masked;
  }

  // Mask phone numbers in messages
  if (typeof info.message === 'string') {
    info.message = info.message
      .replace(/\b0[6-9]\d{8}\b/g, '[PHONE_REDACTED]')
      .replace(/\b\d{13}\b/g, '[ID_REDACTED]');
  }

  return maskValue(info as Record<string, unknown>) as winston.Logform.TransformableInfo;
})();

// Elasticsearch transport for ELK
const esTransport = new ElasticsearchTransport({
  level: 'info',
  clientOpts: {
    node: process.env.ELASTICSEARCH_URL || 'http://localhost:9200',
    auth: {
      username: process.env.ELASTICSEARCH_USERNAME || 'elastic',
      password: process.env.ELASTICSEARCH_PASSWORD || '',
    },
  },
  indexPrefix: 'chuaikan-logs',
  indexSuffixPattern: 'YYYY.MM.DD',
  transformer: (logData) => ({
    '@timestamp': logData.timestamp,
    severity: logData.level,
    message: logData.message,
    service: process.env.SERVICE_NAME || 'chuaikan-api',
    environment: process.env.NODE_ENV,
    ...logData.meta,
  }),
});

// Create logger
export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || (process.env.NODE_ENV === 'production' ? 'info' : 'debug'),
  format: winston.format.combine(
    maskSensitiveData,
    jsonFormat
  ),
  defaultMeta: {
    service: process.env.SERVICE_NAME || 'chuaikan-api',
    environment: process.env.NODE_ENV || 'development',
    version: process.env.APP_VERSION || '1.0.0',
  },
  transports: [
    // Console output (development)
    new winston.transports.Console({
      format: process.env.NODE_ENV === 'production'
        ? jsonFormat
        : winston.format.combine(
            winston.format.colorize(),
            winston.format.simple()
          ),
    }),
    
    // File transport (all logs)
    new winston.transports.File({
      filename: '/var/log/chuaikan/app.log',
      maxsize: 100 * 1024 * 1024, // 100MB
      maxFiles: 10,
      tailable: true,
    }),
    
    // File transport (errors only)
    new winston.transports.File({
      filename: '/var/log/chuaikan/error.log',
      level: 'error',
      maxsize: 100 * 1024 * 1024,
      maxFiles: 5,
    }),
    
    // Elasticsearch transport (production)
    ...(process.env.NODE_ENV === 'production' ? [esTransport] : []),
  ],
  
  exceptionHandlers: [
    new winston.transports.File({ filename: '/var/log/chuaikan/exceptions.log' }),
  ],
  
  rejectionHandlers: [
    new winston.transports.File({ filename: '/var/log/chuaikan/rejections.log' }),
  ],
});
```

### Step 177: Pino Logger (High-Performance Alternative)

```typescript
// src/lib/logger/pino.ts
import pino from 'pino';

// Pino เร็วกว่า Winston ~3-5x เพราะ async logging
// แนะนำสำหรับ high-throughput endpoints

const pinoLogger = pino({
  level: process.env.LOG_LEVEL || 'info',
  
  // Custom serializers
  serializers: {
    req: pino.stdSerializers.req,
    res: pino.stdSerializers.res,
    err: pino.stdSerializers.err,
    
    // Custom user serializer (exclude sensitive data)
    user: (user: any) => ({
      id: user.id,
      role: user.role,
      // ไม่ log email หรือ personal data
    }),
  },
  
  // Redact sensitive paths
  redact: {
    paths: [
      'req.headers.authorization',
      'req.body.password',
      'req.body.token',
      'req.body.phone',
      '*.password',
      '*.token',
      '*.secret',
    ],
    censor: '[REDACTED]',
  },
  
  // Transport
  transport: process.env.NODE_ENV === 'production' ? {
    targets: [
      {
        target: 'pino/file',
        options: { destination: '/var/log/chuaikan/app.log' },
        level: 'info',
      },
      {
        target: 'pino-elasticsearch',
        options: {
          node: process.env.ELASTICSEARCH_URL,
          auth: {
            username: process.env.ELASTICSEARCH_USERNAME,
            password: process.env.ELASTICSEARCH_PASSWORD,
          },
          index: 'chuaikan-logs',
          consistency: 'one',
        },
        level: 'info',
      },
    ],
  } : {
    target: 'pino-pretty',
    options: {
      colorize: true,
      translateTime: 'SYS:standard',
      ignore: 'pid,hostname',
    },
  },
  
  // Base fields
  base: {
    service: process.env.SERVICE_NAME || 'chuaikan-api',
    environment: process.env.NODE_ENV,
    version: process.env.APP_VERSION || '1.0.0',
  },
});

export { pinoLogger as logger };
```

### Step 178: Correlation IDs

```typescript
// src/lib/logger/correlation.ts
import { AsyncLocalStorage } from 'async_hooks';
import { NextRequest, NextResponse } from 'next/server';

interface CorrelationContext {
  requestId: string;
  traceId: string;
  userId?: string;
  sessionId?: string;
}

// AsyncLocalStorage for storing correlation context
const correlationStorage = new AsyncLocalStorage<CorrelationContext>();

export function getCorrelationContext(): CorrelationContext | undefined {
  return correlationStorage.getStore();
}

export function getRequestId(): string {
  return correlationStorage.getStore()?.requestId || 'unknown';
}

export function withCorrelation<T>(
  context: CorrelationContext,
  fn: () => T
): T {
  return correlationStorage.run(context, fn);
}

// Middleware to inject correlation IDs
export async function correlationMiddleware(
  req: NextRequest,
  handler: (req: NextRequest) => Promise<NextResponse>
): Promise<NextResponse> {
  const requestId = req.headers.get('X-Request-Id') || crypto.randomUUID();
  const traceId = req.headers.get('X-Trace-Id') || crypto.randomUUID();
  const userId = req.headers.get('X-User-Id') || undefined;

  const context: CorrelationContext = { requestId, traceId, userId };

  return new Promise((resolve, reject) => {
    correlationStorage.run(context, async () => {
      try {
        const response = await handler(req);
        response.headers.set('X-Request-Id', requestId);
        response.headers.set('X-Trace-Id', traceId);
        resolve(response);
      } catch (error) {
        reject(error);
      }
    });
  });
}

// Enhanced logger that includes correlation context automatically
import { logger as baseLogger } from './winston';

export const logger = {
  error: (message: string, meta?: Record<string, unknown>) => {
    baseLogger.error(message, { ...getCorrelationContext(), ...meta });
  },
  warn: (message: string, meta?: Record<string, unknown>) => {
    baseLogger.warn(message, { ...getCorrelationContext(), ...meta });
  },
  info: (message: string, meta?: Record<string, unknown>) => {
    baseLogger.info(message, { ...getCorrelationContext(), ...meta });
  },
  debug: (message: string, meta?: Record<string, unknown>) => {
    baseLogger.debug(message, { ...getCorrelationContext(), ...meta });
  },
};
```

### Step 179: Logstash Pipeline Configuration

```bash
# สร้าง Logstash pipeline config
sudo nano /etc/logstash/conf.d/chuaikan.conf
```

```ruby
# /etc/logstash/conf.d/chuaikan.conf

input {
  beats {
    port => 5044
    ssl => false
  }
  
  # Direct TCP input (optional)
  tcp {
    port => 5000
    codec => json_lines
  }
}

filter {
  # Parse JSON logs from Node.js
  if [fields][app] == "chuaikan" {
    json {
      source => "message"
      target => "parsed"
    }
    
    # Extract fields from parsed JSON
    mutate {
      rename => {
        "[parsed][timestamp]" => "@timestamp"
        "[parsed][level]" => "log_level"
        "[parsed][message]" => "log_message"
        "[parsed][service]" => "service"
        "[parsed][requestId]" => "request_id"
        "[parsed][traceId]" => "trace_id"
        "[parsed][userId]" => "user_id"
        "[parsed][method]" => "http_method"
        "[parsed][url]" => "request_url"
        "[parsed][statusCode]" => "http_status"
        "[parsed][duration]" => "response_time_ms"
        "[parsed][errorCode]" => "error_code"
      }
      
      # Remove original message field
      remove_field => ["message", "parsed"]
    }
    
    # Convert log level to numeric for easier filtering
    translate {
      field => "log_level"
      destination => "log_level_num"
      dictionary => {
        "fatal" => "60"
        "error" => "50"
        "warn"  => "40"
        "info"  => "30"
        "debug" => "20"
        "trace" => "10"
      }
    }
    
    # Add GeoIP for IP addresses
    if [client_ip] {
      geoip {
        source => "client_ip"
        target => "geoip"
        fields => ["country_code2", "country_name", "city_name", "location"]
      }
    }
    
    # Tag slow requests
    if [response_time_ms] {
      ruby {
        code => "
          rt = event.get('response_time_ms').to_i
          if rt > 5000
            event.tag('very_slow')
          elsif rt > 1000
            event.tag('slow')
          end
        "
      }
    }
    
    # Tag errors
    if [log_level] in ["error", "fatal"] {
      mutate {
        add_tag => ["error"]
      }
    }
  }
  
  # Drop DEBUG logs in production (save storage)
  if [log_level] == "debug" and [environment] == "production" {
    drop {}
  }
}

output {
  # Send to Elasticsearch
  elasticsearch {
    hosts => ["http://localhost:9200"]
    user => "elastic"
    password => "${ES_PASSWORD}"
    
    # Dynamic index naming
    index => "chuaikan-%{service}-%{+YYYY.MM.dd}"
    
    # ILM for index lifecycle management
    ilm_enabled => true
    ilm_rollover_alias => "chuaikan-logs"
    ilm_pattern => "{now/d}-000001"
    ilm_policy => "chuaikan-log-policy"
    
    template_name => "chuaikan-template"
    template_pattern => "chuaikan-*"
  }
  
  # Debug output (development only)
  # stdout { codec => rubydebug }
}
```

### Step 180: Filebeat Configuration

```yaml
# /etc/filebeat/filebeat.yml

filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/chuaikan/*.log
    
    # Only ship JSON logs
    json.message_key: message
    json.keys_under_root: false
    json.add_error_key: true
    
    # Add metadata
    fields:
      app: chuaikan
      environment: production
    fields_under_root: false
    
    # Multiline support (for stack traces)
    multiline.pattern: '^\s'
    multiline.negate: false
    multiline.match: after
    multiline.max_lines: 50

  # Next.js access logs
  - type: log
    enabled: true
    paths:
      - /var/log/nginx/chuaikan-access.log
    fields:
      app: chuaikan-nginx
      log_type: access

# Output to Logstash
output.logstash:
  hosts: ["localhost:5044"]
  
  # Load balancing (if multiple Logstash)
  # hosts: ["logstash1:5044", "logstash2:5044"]
  # loadbalance: true
  
  # SSL (enable in production)
  # ssl.certificate_authorities: ["/etc/filebeat/ca.crt"]

# Processors
processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_docker_metadata: ~

# Logging
logging.level: warning
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7

# Registry (track shipped files)
filebeat.registry.path: /var/lib/filebeat/registry
```

```bash
# สร้าง log directory
sudo mkdir -p /var/log/chuaikan
sudo chown -R www-data:www-data /var/log/chuaikan

# Test Filebeat config
sudo filebeat test config
sudo filebeat test output

# Start Filebeat
sudo systemctl enable filebeat
sudo systemctl start filebeat
```

---

## 🔧 Configuration Files

### Elasticsearch ILM Policy

```bash
# สร้าง Index Lifecycle Management Policy
curl -u elastic:YOUR_PASSWORD \
  -X PUT http://localhost:9200/_ilm/policy/chuaikan-log-policy \
  -H "Content-Type: application/json" \
  -d '{
    "policy": {
      "phases": {
        "hot": {
          "min_age": "0ms",
          "actions": {
            "rollover": {
              "max_primary_shard_size": "50GB",
              "max_age": "1d"
            },
            "set_priority": {
              "priority": 100
            }
          }
        },
        "warm": {
          "min_age": "7d",
          "actions": {
            "shrink": {
              "number_of_shards": 1
            },
            "forcemerge": {
              "max_num_segments": 1
            },
            "set_priority": {
              "priority": 50
            }
          }
        },
        "cold": {
          "min_age": "30d",
          "actions": {
            "set_priority": {
              "priority": 0
            },
            "freeze": {}
          }
        },
        "delete": {
          "min_age": "90d",
          "actions": {
            "delete": {}
          }
        }
      }
    }
  }'
```

### Kibana Dashboard Queries

```bash
# สร้าง Index Pattern ใน Kibana
curl -u elastic:YOUR_PASSWORD \
  -X POST http://localhost:5601/api/index_patterns/index_pattern \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{
    "index_pattern": {
      "title": "chuaikan-*",
      "timeFieldName": "@timestamp"
    }
  }'
```

```json
// Kibana Saved Queries สำหรับ chuaikan.com

// Error Rate (Kibana Lens Query)
// {
//   "query": "log_level: error OR log_level: fatal",
//   "aggregation": "count",
//   "interval": "1m",
//   "visualization": "line_chart"
// }

// Slow Requests (> 1 second)
// {
//   "query": "response_time_ms > 1000",
//   "aggregation": "avg(response_time_ms)",
//   "groupBy": "request_url",
//   "visualization": "bar_chart"
// }

// SOS Alert Creation Rate
// {
//   "query": "message: \"SOS alert created\"",
//   "aggregation": "count",
//   "interval": "5m",
//   "visualization": "area_chart"
// }
```

### Alert Configuration

```yaml
# Elasticsearch Watcher Alert: Error Spike
# สร้างผ่าน Kibana Stack Management > Watcher

watch:
  trigger:
    schedule:
      interval: "1m"
  
  input:
    search:
      request:
        indices: ["chuaikan-*"]
        body:
          query:
            bool:
              filter:
                - range:
                    "@timestamp":
                      gte: "now-5m"
                - terms:
                    log_level: ["error", "fatal"]
          aggs:
            error_count:
              value_count:
                field: "@timestamp"
  
  condition:
    compare:
      ctx.payload.aggregations.error_count.value:
        gte: 10  # Alert ถ้ามี error มากกว่า 10 ครั้งใน 5 นาที
  
  actions:
    send_slack:
      webhook:
        scheme: https
        host: hooks.slack.com
        port: 443
        method: post
        path: "/services/YOUR_WEBHOOK_PATH"
        params: {}
        headers:
          Content-Type: application/json
        body: >
          {
            "text": "⚠️ Error Spike Alert",
            "blocks": [
              {
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "*Error Count:* {{ctx.payload.aggregations.error_count.value}} errors in 5 minutes\n*Service:* chuaikan-api\n*Time:* {{ctx.execution_time}}"
                }
              }
            ]
          }
```

---

## 🧪 Testing

### Test Logging Setup

```typescript
// __tests__/lib/logger.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { logger } from '@/lib/logger/winston';

describe('Logger', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it('should mask sensitive data', () => {
    const logSpy = vi.spyOn(logger, 'info');
    
    logger.info('User login', {
      email: 'test@example.com',
      password: 'secret123',  // Should be masked
      token: 'jwt_token',     // Should be masked
    });

    expect(logSpy).toHaveBeenCalledWith(
      'User login',
      expect.objectContaining({
        email: 'test@example.com',
        password: '[REDACTED]',
        token: '[REDACTED]',
      })
    );
  });

  it('should mask phone numbers in messages', () => {
    const logSpy = vi.spyOn(logger, 'info');
    
    logger.info('User phone: 0891234567 called for help');
    
    expect(logSpy).toHaveBeenCalledWith(
      expect.stringContaining('[PHONE_REDACTED]'),
      expect.any(Object)
    );
  });

  it('should include correlation context', () => {
    const { withCorrelation, logger: correlatedLogger } = require('@/lib/logger/correlation');
    const logSpy = vi.spyOn(require('@/lib/logger/winston'), 'logger', 'get')
      .mockReturnValue({ info: vi.fn() });
    
    const mockRequestId = 'test-request-id';
    
    withCorrelation({ requestId: mockRequestId, traceId: 'trace-id' }, () => {
      correlatedLogger.info('Test message');
    });
    
    // Verify requestId is included
    expect(logSpy.mock.results[0].value.info).toHaveBeenCalledWith(
      'Test message',
      expect.objectContaining({ requestId: mockRequestId })
    );
  });
});
```

### Test Log Format

```bash
# ทดสอบ log format ถูกต้อง
node -e "
const { logger } = require('./src/lib/logger/winston');
logger.info('Test SOS alert created', {
  alertId: 'alert-123',
  userId: 'user-456',
  type: 'MEDICAL',
  severity: 'HIGH',
  latitude: 13.7563,
  longitude: 100.5018,
});
"

# ตรวจสอบว่า output เป็น JSON format
cat /var/log/chuaikan/app.log | tail -1 | jq .
```

```bash
# ทดสอบ Elasticsearch connection
curl -u elastic:YOUR_PASSWORD \
  -X POST http://localhost:9200/chuaikan-test/_doc \
  -H "Content-Type: application/json" \
  -d '{
    "@timestamp": "2024-01-15T10:30:00Z",
    "level": "info",
    "message": "Test log entry",
    "service": "chuaikan-api"
  }'

# Query logs
curl -u elastic:YOUR_PASSWORD \
  "http://localhost:9200/chuaikan-*/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "bool": {
        "filter": [
          { "term": { "level": "error" } },
          { "range": { "@timestamp": { "gte": "now-1h" } } }
        ]
      }
    },
    "sort": [{ "@timestamp": "desc" }],
    "size": 10
  }'
```

---

## ❌ Common Errors & Solutions

### Error 1: Elasticsearch fails to start - max virtual memory areas

```
max virtual memory areas vm.max_map_count [65530] is too low
```

**แก้ไข:**
```bash
# เพิ่ม vm.max_map_count
sudo sysctl -w vm.max_map_count=262144

# ทำให้ permanent
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### Error 2: Elasticsearch Out of Memory

```
java.lang.OutOfMemoryError: Java heap space
```

**แก้ไข:**
```bash
# ลด heap size
sudo nano /etc/elasticsearch/jvm.options.d/heap.options
# -Xms1g
# -Xmx1g

# หรือเพิ่ม RAM ให้ server
free -h  # ดู RAM ที่มี
```

### Error 3: Filebeat permission denied

```
Harvester could not be started: Permission denied
```

**แก้ไข:**
```bash
# เพิ่ม filebeat user เข้า group
sudo usermod -a -G adm filebeat

# หรือเปลี่ยน owner ของ log files
sudo chown -R filebeat:filebeat /var/log/chuaikan/
```

### Error 4: Logstash กิน CPU สูง

**แก้ไข:**
```yaml
# ลด worker threads
# /etc/logstash/logstash.yml
pipeline.workers: 2  # Default คือ CPU cores
pipeline.batch.size: 125
pipeline.batch.delay: 50
```

---

## ✅ Checklist

### Logging Setup
- [ ] Winston logger configured กับ JSON format
- [ ] Log masking: password, token, phone numbers ถูก redact
- [ ] Log levels ถูกกำหนดถูกต้อง (INFO ใน production)
- [ ] Correlation IDs ทุก request
- [ ] Error และ exception handlers configured
- [ ] Log rotation configured (max size 100MB, keep 10 files)

### ELK Stack
- [ ] Elasticsearch 8 ทำงานบน port 9200
- [ ] Kibana ทำงานบน port 5601
- [ ] Logstash pipeline configured สำหรับ Node.js logs
- [ ] Filebeat shipping logs ไปยัง Logstash
- [ ] Authentication เปิดใช้งานบน Elasticsearch

### Data Management
- [ ] ILM Policy: hot (30 วัน), cold (90 วัน), delete
- [ ] Index template configured
- [ ] Kibana index pattern สร้างแล้ว

### Monitoring & Alerts
- [ ] Kibana dashboard: error rate, slow requests
- [ ] Alert rule: error spike (>10 errors/5min)
- [ ] Slack/LINE notification configured

### Security
- [ ] Elasticsearch password secured
- [ ] Log files ไม่เก็บ personal data โดยตรง
- [ ] Access to Kibana ต้อง authenticate

---

## 🔗 References

- [Winston Logger Documentation](https://github.com/winstonjs/winston)
- [Pino Logger](https://getpino.io/)
- [Elasticsearch 8 Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/8.x/index.html)
- [Logstash Pipeline](https://www.elastic.co/guide/en/logstash/current/configuration.html)
- [Filebeat Documentation](https://www.elastic.co/guide/en/beats/filebeat/current/index.html)
- [Kibana Alerting](https://www.elastic.co/guide/en/kibana/current/alerting-getting-started.html)

---
*Part 018 | Road to 1,000,000 Users/Day | chuaikan.com*
