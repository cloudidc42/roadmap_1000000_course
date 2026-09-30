# Part 077: Global Load Balancing
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Professional
> **Steps:** 761–770
> **เวลาโดยประมาณ:** 6 ชั่วโมง
> **Prerequisites:** Part 076 (Multi-region), Part 007 (Cloudflare)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- Anycast routing ด้วย Cloudflare
- AWS Global Accelerator setup
- GeoDNS สำหรับ routing user ในไทยไปยัง Singapore
- Health-check based failover
- Weighted routing สำหรับ canary deployment
- CORS configuration ที่ Load Balancer
- Monitoring เต็มรูปแบบ

---

## 📖 ทฤษฎีและแนวคิด

### Anycast Routing
Anycast คือวิธีที่ IP address เดียวกันสามารถ route ไปยัง server หลายตัวที่กระจายอยู่ทั่วโลก ตามหลัก "ใกล้ที่สุด"

Cloudflare มี 300+ PoP (Points of Presence) ทั่วโลก รวมถึง Bangkok → ทำให้ Thai users ได้ edge ที่ใกล้ที่สุด

### GeoDNS
Route 53 สามารถ route ตาม:
1. **Geolocation**: ไทย → Singapore, ญี่ปุ่น → Tokyo
2. **Latency**: วัด latency จริงแล้วเลือก region ที่เร็วที่สุด
3. **Weighted**: 90% Primary, 10% Canary

---

## ⚙️ Environment Setup

### Step 761: ติดตั้ง AWS CLI และ Cloudflare CLI

```bash
# AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install
aws --version

# Cloudflare Wrangler CLI
npm install -g wrangler
wrangler --version

# cloudflared tunnel CLI
wget -q https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb
cloudflared --version
```

---

## 🛠️ Step-by-Step Implementation

### Step 762: AWS Global Accelerator

```hcl
# infra/terraform/global/accelerator.tf

resource "aws_globalaccelerator_accelerator" "chuaikan" {
  name            = "chuaikan-global"
  ip_address_type = "IPV4"
  enabled         = true

  attributes {
    flow_logs_enabled   = true
    flow_logs_s3_bucket = aws_s3_bucket.accelerator_logs.bucket
    flow_logs_s3_prefix = "flow-logs/"
  }

  tags = {
    Name    = "chuaikan-global-accelerator"
    Project = "chuaikan"
  }
}

# Listener สำหรับ HTTPS
resource "aws_globalaccelerator_listener" "https" {
  accelerator_arn = aws_globalaccelerator_accelerator.chuaikan.id
  protocol        = "TCP"

  port_range {
    from_port = 443
    to_port   = 443
  }

  client_affinity = "SOURCE_IP"
}

# Endpoint Group สำหรับ Singapore (Primary)
resource "aws_globalaccelerator_endpoint_group" "singapore" {
  listener_arn                  = aws_globalaccelerator_listener.https.id
  endpoint_group_region         = "ap-southeast-1"
  traffic_dial_percentage       = 100  # Primary รับ 100%
  health_check_path             = "/health"
  health_check_protocol         = "HTTPS"
  health_check_interval_seconds = 30
  threshold_count               = 3

  endpoint_configuration {
    endpoint_id                    = aws_lb.singapore.arn
    weight                         = 100
    client_ip_preservation_enabled = true
  }
}

# Endpoint Group สำหรับ Tokyo (DR - standby)
resource "aws_globalaccelerator_endpoint_group" "tokyo" {
  listener_arn                  = aws_globalaccelerator_listener.https.id
  endpoint_group_region         = "ap-northeast-1"
  traffic_dial_percentage       = 0   # DR: standby mode
  health_check_path             = "/health"
  health_check_protocol         = "HTTPS"
  health_check_interval_seconds = 30
  threshold_count               = 3

  endpoint_configuration {
    endpoint_id                    = aws_lb.tokyo.arn
    weight                         = 100
    client_ip_preservation_enabled = true
  }
}
```

### Step 763: Cloudflare DNS และ Load Balancing

```bash
# ตั้งค่าผ่าน Cloudflare API
export CF_API_TOKEN="your_cloudflare_api_token"
export CF_ZONE_ID="your_zone_id"

# สร้าง Origin Pool สำหรับ Singapore
curl -X POST "https://api.cloudflare.com/client/v4/user/load_balancers/pools" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" \
  -H "Content-Type: application/json" \
  --data '{
    "name": "singapore-pool",
    "description": "Singapore Primary Region",
    "origins": [
      {
        "name": "sg-alb-1",
        "address": "sg-alb.chuaikan.com",
        "enabled": true,
        "weight": 1,
        "header": {
          "Host": ["sg-alb.chuaikan.com"]
        }
      }
    ],
    "minimum_origins": 1,
    "monitor": "MONITOR_ID",
    "notification_email": "ops@chuaikan.com"
  }'

# สร้าง Origin Pool สำหรับ Tokyo
curl -X POST "https://api.cloudflare.com/client/v4/user/load_balancers/pools" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" \
  -H "Content-Type: application/json" \
  --data '{
    "name": "tokyo-pool",
    "description": "Tokyo DR Region",
    "origins": [
      {
        "name": "jp-alb-1",
        "address": "jp-alb.chuaikan.com",
        "enabled": true,
        "weight": 1
      }
    ],
    "minimum_origins": 1,
    "monitor": "MONITOR_ID"
  }'
```

```bash
# สร้าง Health Monitor
curl -X POST "https://api.cloudflare.com/client/v4/user/load_balancers/monitors" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" \
  -H "Content-Type: application/json" \
  --data '{
    "type": "https",
    "description": "chuaikan health check",
    "method": "GET",
    "path": "/health",
    "header": {
      "Host": ["api.chuaikan.com"],
      "X-Health-Check": ["cloudflare"]
    },
    "interval": 60,
    "timeout": 5,
    "retries": 2,
    "expected_codes": "200",
    "expected_body": "\"status\":\"ok\"",
    "follow_redirects": false,
    "allow_insecure": false
  }'
```

```bash
# สร้าง Load Balancer
curl -X POST "https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/load_balancers" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" \
  -H "Content-Type: application/json" \
  --data '{
    "name": "api.chuaikan.com",
    "description": "Global API Load Balancer",
    "fallback_pool": "TOKYO_POOL_ID",
    "default_pools": ["SINGAPORE_POOL_ID"],
    "proxied": true,
    "session_affinity": "ip_cookie",
    "session_affinity_ttl": 1800,
    "steering_policy": "geo",
    "region_pools": {
      "SEAS": ["SINGAPORE_POOL_ID"],
      "EASA": ["TOKYO_POOL_ID"]
    },
    "country_pools": {
      "TH": ["SINGAPORE_POOL_ID"],
      "JP": ["TOKYO_POOL_ID"],
      "SG": ["SINGAPORE_POOL_ID"],
      "MY": ["SINGAPORE_POOL_ID"]
    },
    "rules": [
      {
        "name": "canary-5-percent",
        "condition": "http.request.headers[\"x-canary\"][0] == \"true\"",
        "overrides": {
          "default_pools": ["CANARY_POOL_ID"]
        },
        "disabled": false
      }
    ]
  }'
```

### Step 764: GeoDNS สำหรับ Route 53

```hcl
# infra/terraform/global/geodns.tf

# Geolocation record: ไทย → Singapore
resource "aws_route53_record" "api_thailand" {
  zone_id        = var.hosted_zone_id
  name           = "api.chuaikan.com"
  type           = "A"
  set_identifier = "thailand"

  geolocation_routing_policy {
    country = "TH"
  }

  alias {
    name                   = aws_lb.singapore.dns_name
    zone_id                = aws_lb.singapore.zone_id
    evaluate_target_health = true
  }
}

# Geolocation record: อาเซียน → Singapore
resource "aws_route53_record" "api_asean" {
  zone_id        = var.hosted_zone_id
  name           = "api.chuaikan.com"
  type           = "A"
  set_identifier = "asean"

  geolocation_routing_policy {
    continent = "AS"
  }

  alias {
    name                   = aws_lb.singapore.dns_name
    zone_id                = aws_lb.singapore.zone_id
    evaluate_target_health = true
  }
}

# Default record: ทุกที่อื่น → Global Accelerator
resource "aws_route53_record" "api_default" {
  zone_id        = var.hosted_zone_id
  name           = "api.chuaikan.com"
  type           = "A"
  set_identifier = "default"

  geolocation_routing_policy {
    country = "*"
  }

  alias {
    name                   = aws_globalaccelerator_accelerator.chuaikan.dns_name
    zone_id                = "Z2BJ6XQ5FK7U4H"  # Global Accelerator hosted zone
    evaluate_target_health = false
  }
}
```

### Step 765: CORS Configuration ที่ ALB

```hcl
# infra/terraform/alb/cors.tf
# ALB Listener Rule สำหรับ CORS preflight

resource "aws_lb_listener_rule" "cors_preflight" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 1

  action {
    type = "fixed-response"
    fixed_response {
      content_type = "text/plain"
      message_body = ""
      status_code  = "204"
    }
  }

  condition {
    http_request_method {
      values = ["OPTIONS"]
    }
  }
}
```

```nginx
# k8s/nginx-configmap.yaml - CORS headers ใน NGINX
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-cors-config
  namespace: chuaikan
data:
  cors.conf: |
    map $http_origin $cors_origin {
        default "";
        "~^https://(www\.)?chuaikan\.com$" $http_origin;
        "~^https://app\.chuaikan\.com$" $http_origin;
        "~^http://localhost(:[0-9]+)?$" $http_origin;
    }

    add_header 'Access-Control-Allow-Origin' $cors_origin always;
    add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
    add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type, X-Request-ID, X-App-Version' always;
    add_header 'Access-Control-Allow-Credentials' 'true' always;
    add_header 'Access-Control-Max-Age' '86400' always;

    if ($request_method = 'OPTIONS') {
        return 204;
    }
```

### Step 766: Weighted Routing สำหรับ Canary

```bash
# สร้าง Weighted records สำหรับ canary deployment
# 95% → Stable, 5% → Canary

aws route53 change-resource-record-sets \
  --hosted-zone-id ${HOSTED_ZONE_ID} \
  --change-batch '{
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "api.chuaikan.com",
          "Type": "A",
          "SetIdentifier": "stable",
          "Weight": 95,
          "HealthCheckId": "STABLE_HC_ID",
          "AliasTarget": {
            "HostedZoneId": "ALB_ZONE_ID",
            "DNSName": "stable-alb.chuaikan.com",
            "EvaluateTargetHealth": true
          }
        }
      },
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "api.chuaikan.com",
          "Type": "A",
          "SetIdentifier": "canary",
          "Weight": 5,
          "HealthCheckId": "CANARY_HC_ID",
          "AliasTarget": {
            "HostedZoneId": "ALB_ZONE_ID",
            "DNSName": "canary-alb.chuaikan.com",
            "EvaluateTargetHealth": true
          }
        }
      }
    ]
  }'

echo "Canary deployment: 5% traffic shifted to new version"
```

### Step 767: Load Balancer Monitoring

```yaml
# k8s/monitoring/lb-dashboard.yaml
# Prometheus ServiceMonitor สำหรับ ALB metrics

apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: aws-alb-metrics
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: aws-load-balancer-controller
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

```python
# scripts/lb_health_check.py
# ตรวจสอบสุขภาพของ Load Balancer targets

import boto3
import sys
from datetime import datetime

def check_target_health(alb_arn_suffix, region):
    """ตรวจสอบ health ของ targets ทั้งหมด"""
    elbv2 = boto3.client('elbv2', region_name=region)
    
    # หา Target Groups
    response = elbv2.describe_target_groups(
        LoadBalancerArn=alb_arn_suffix
    )
    
    unhealthy_targets = []
    
    for tg in response['TargetGroups']:
        health_response = elbv2.describe_target_health(
            TargetGroupArn=tg['TargetGroupArn']
        )
        
        for target in health_response['TargetHealthDescriptions']:
            state = target['TargetHealth']['State']
            target_id = target['Target']['Id']
            
            if state != 'healthy':
                unhealthy_targets.append({
                    'tg': tg['TargetGroupName'],
                    'target': target_id,
                    'state': state,
                    'reason': target['TargetHealth'].get('Reason', 'Unknown')
                })
                print(f"UNHEALTHY: {tg['TargetGroupName']} - {target_id} ({state})")
            else:
                print(f"HEALTHY: {tg['TargetGroupName']} - {target_id}")
    
    return len(unhealthy_targets) == 0

def main():
    # ตรวจ Singapore
    sg_healthy = check_target_health(
        "arn:aws:elasticloadbalancing:ap-southeast-1:123456789:loadbalancer/app/chuaikan-sg/abc123",
        "ap-southeast-1"
    )
    
    # ตรวจ Tokyo
    jp_healthy = check_target_health(
        "arn:aws:elasticloadbalancing:ap-northeast-1:123456789:loadbalancer/app/chuaikan-jp/def456",
        "ap-northeast-1"
    )
    
    if not sg_healthy:
        print("ALERT: Singapore ALB has unhealthy targets!")
        sys.exit(1)
    
    print("All ALB targets are healthy.")

if __name__ == "__main__":
    main()
```

---

## 🧪 Testing

```bash
# ทดสอบ GeoDNS
# จาก Thailand
dig api.chuaikan.com @8.8.8.8
# ควรได้ Singapore IP

# ทดสอบ CORS
curl -v -X OPTIONS "https://api.chuaikan.com/v1/posts" \
  -H "Origin: https://chuaikan.com" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: Authorization,Content-Type"
# ควรเห็น Access-Control-Allow-Origin: https://chuaikan.com

# ทดสอบ Health Check
curl -sf "https://api.chuaikan.com/health" | jq .
# {"status":"ok","region":"ap-southeast-1","version":"1.2.3"}

# ทดสอบ Global Accelerator IPs
nslookup chuaikan.awsglobalaccelerator.com
# ควรได้ Anycast IPs 2 ตัว
```

---

## ✅ Checklist

- [ ] AWS Global Accelerator สร้างแล้วมี 2 static Anycast IPs
- [ ] Singapore endpoint group รับ 100% traffic
- [ ] Tokyo endpoint group อยู่ใน standby mode
- [ ] Cloudflare GeoDNS: TH/SG/MY → Singapore pool
- [ ] Health Monitor ทุก 60 วินาที บน `/health` endpoint
- [ ] CORS headers ถูกส่งไปอย่างถูกต้อง (OPTIONS return 204)
- [ ] Weighted routing สำหรับ canary พร้อมใช้งาน
- [ ] ALB access logs เปิดอยู่ (ส่งไป S3)
- [ ] CloudWatch dashboards แสดง request count, latency, error rate
- [ ] Alarm: error rate > 1% notify ทีม

---

## 🔗 References

- [AWS Global Accelerator](https://docs.aws.amazon.com/global-accelerator/)
- [Cloudflare Load Balancing](https://developers.cloudflare.com/load-balancing/)
- [Route 53 Geolocation Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-geo.html)

---
*Part 077 | Road to 1,000,000 Users/Day | chuaikan.com*
