# Part 083: A/B Testing Infrastructure
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Professional
> **Steps:** 821–830
> **เวลาโดยประมาณ:** 8 ชั่วโมง
> **Prerequisites:** Part 082 (Feature Flags), Part 041 (DB)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- A/B testing ใน feature flags vs URL parameters
- Experiment design: hypothesis, metrics, sample size
- Bucketing users อย่าง consistent (hash-based)
- Statistical significance calculation
- ClickHouse สำหรับ A/B test analysis
- chuaikan.com experiments: feed ranking, SOS alert UI, notification timing
- Experiment monitoring: guardrail metrics
- Decision framework: ship, rollback, iterate

---

## 📖 ทฤษฎีและแนวคิด

### Statistical Significance

ก่อน ship ผล experiment ต้องมั่นใจว่า "ผล real ไม่ใช่ luck"

- **Confidence Level**: 95% (p-value < 0.05)
- **Statistical Power**: 80% (ตรวจหา effect ได้ 80% ของเวลา)
- **Minimum Detectable Effect (MDE)**: การเปลี่ยนแปลงที่เล็กที่สุดที่คุณอยากเห็น

### Sample Size Formula
```
n = (Z_α/2 + Z_β)² × (p1(1-p1) + p2(1-p2)) / (p1 - p2)²

โดยที่:
- Z_α/2 = 1.96 (สำหรับ 95% confidence)
- Z_β = 0.84 (สำหรับ 80% power)
- p1 = baseline conversion rate
- p2 = expected new rate
```

---

## 🛠️ Step-by-Step Implementation

### Step 821: Experiment Design Framework

```python
# scripts/experiment_design.py
"""
คำนวณ sample size สำหรับ A/B test
"""

import math
from scipy import stats
from dataclasses import dataclass
from typing import Optional

@dataclass
class ExperimentDesign:
    name: str
    hypothesis: str
    primary_metric: str
    baseline_rate: float
    minimum_detectable_effect: float  # absolute change
    confidence_level: float = 0.95
    statistical_power: float = 0.80
    traffic_percentage: float = 1.0   # % of users in experiment

    def calculate_sample_size(self) -> int:
        """คำนวณ sample size ที่ต้องการต่อ variant"""
        p1 = self.baseline_rate
        p2 = self.baseline_rate + self.minimum_detectable_effect

        alpha = 1 - self.confidence_level
        beta = 1 - self.statistical_power

        z_alpha = stats.norm.ppf(1 - alpha / 2)
        z_beta = stats.norm.ppf(self.statistical_power)

        n = ((z_alpha + z_beta) ** 2 * (p1 * (1-p1) + p2 * (1-p2))) / (p1 - p2) ** 2
        return math.ceil(n)

    def calculate_duration_days(self, daily_traffic: int) -> float:
        """คำนวณจำนวนวันที่ต้องรันการทดสอบ"""
        n_per_variant = self.calculate_sample_size()
        n_total = n_per_variant * 2  # 2 variants
        eligible_users = daily_traffic * self.traffic_percentage
        return n_total / eligible_users

    def print_summary(self, daily_traffic: int):
        n = self.calculate_sample_size()
        days = self.calculate_duration_days(daily_traffic)

        print(f"\n{'='*50}")
        print(f"Experiment: {self.name}")
        print(f"{'='*50}")
        print(f"Hypothesis: {self.hypothesis}")
        print(f"Primary Metric: {self.primary_metric}")
        print(f"Baseline Rate: {self.baseline_rate:.1%}")
        print(f"Target Rate: {self.baseline_rate + self.minimum_detectable_effect:.1%}")
        print(f"MDE: +{self.minimum_detectable_effect:.1%}")
        print(f"\nSample Size per Variant: {n:,}")
        print(f"Total Users Needed: {n*2:,}")
        print(f"Expected Duration: {days:.1f} days")
        print(f"(at {daily_traffic:,} users/day, {self.traffic_percentage:.0%} traffic)")


# chuaikan.com Experiments
experiments = [
    ExperimentDesign(
        name="feed-algorithm-v2",
        hypothesis="ML-based ranking จะเพิ่ม post engagement rate",
        primary_metric="posts_engaged_per_session",
        baseline_rate=0.35,         # 35% ของ posts ที่ user เห็น จะ engage
        minimum_detectable_effect=0.02,  # ต้องการเห็น +2% เป็นอย่างน้อย
        traffic_percentage=0.10     # ทดสอบกับ 10% ของ users
    ),

    ExperimentDesign(
        name="sos-button-color",
        hypothesis="SOS button สีแดงจะเพิ่ม click-through rate",
        primary_metric="sos_button_ctr",
        baseline_rate=0.05,
        minimum_detectable_effect=0.01,
        traffic_percentage=1.0
    ),

    ExperimentDesign(
        name="notification-timing",
        hypothesis="ส่ง notification ตอนเย็น (18:00-20:00) จะเพิ่ม open rate",
        primary_metric="notification_open_rate",
        baseline_rate=0.22,
        minimum_detectable_effect=0.03,
        traffic_percentage=0.50
    )
]

for exp in experiments:
    exp.print_summary(daily_traffic=100_000)
```

### Step 822: Hash-based User Bucketing

```typescript
// src/lib/experiment-bucketing.ts
import crypto from 'crypto';

interface ExperimentConfig {
  id: string;
  name: string;
  variants: Array<{
    name: string;
    weight: number;  // 0-100
  }>;
  startDate: Date;
  endDate?: Date;
  trafficPercentage: number; // 0-100
}

/**
 * ใส่ user ลงใน experiment variant
 * Deterministic: user เดิม → variant เดิมเสมอ
 */
export function assignVariant(
  userId: string,
  experiment: ExperimentConfig
): string | null {
  // ตรวจสอบว่า experiment ยังอยู่ในช่วง valid
  const now = new Date();
  if (now < experiment.startDate) return null;
  if (experiment.endDate && now > experiment.endDate) return null;

  // ตรวจสอบว่า user อยู่ใน traffic bucket
  const trafficHash = murmurHash(`${userId}:${experiment.id}:traffic`);
  const trafficBucket = trafficHash % 100;
  if (trafficBucket >= experiment.trafficPercentage) return null;

  // กำหนด variant
  const variantHash = murmurHash(`${userId}:${experiment.id}:variant`);
  const totalWeight = experiment.variants.reduce((sum, v) => sum + v.weight, 0);
  let bucket = variantHash % totalWeight;

  for (const variant of experiment.variants) {
    if (bucket < variant.weight) {
      return variant.name;
    }
    bucket -= variant.weight;
  }

  return experiment.variants[0].name;
}

/**
 * MurmurHash สำหรับ fast, deterministic hashing
 */
function murmurHash(key: string): number {
  const seed = 42;
  let h = seed;

  for (let i = 0; i < key.length; i++) {
    const char = key.charCodeAt(i);
    h = Math.imul(h ^ char, 0x9e3779b1);
    h = (h << 13) | (h >>> 19);
  }

  h = Math.imul(h ^ (h >>> 16), 0x85ebca6b);
  h = Math.imul(h ^ (h >>> 13), 0xc2b2ae35);
  h = h ^ (h >>> 16);

  return Math.abs(h);
}
```

### Step 823: Experiment Tracking

```typescript
// src/lib/experiment-tracker.ts
import { Kafka, Producer } from 'kafkajs';
import { readPool } from './db';

const kafka = new Kafka({
  clientId: 'chuaikan-experiment-tracker',
  brokers: [process.env.KAFKA_BROKER!]
});

const producer: Producer = kafka.producer();
await producer.connect();

interface ExposureEvent {
  userId: string;
  experimentId: string;
  variant: string;
  timestamp: Date;
  sessionId?: string;
  metadata?: Record<string, unknown>;
}

interface ConversionEvent {
  userId: string;
  experimentId: string;
  metricName: string;
  value: number;
  timestamp: Date;
}

export async function trackExposure(event: ExposureEvent): Promise<void> {
  await producer.send({
    topic: 'experiment-exposures',
    messages: [{
      key: event.userId,
      value: JSON.stringify({
        ...event,
        timestamp: event.timestamp.toISOString()
      })
    }]
  });
}

export async function trackConversion(event: ConversionEvent): Promise<void> {
  await producer.send({
    topic: 'experiment-conversions',
    messages: [{
      key: event.userId,
      value: JSON.stringify({
        ...event,
        timestamp: event.timestamp.toISOString()
      })
    }]
  });
}

// Middleware ที่ track exposure อัตโนมัติ
export function withExperiment(experimentConfig: ExperimentConfig) {
  return async (req: Request, res: Response, next: NextFunction) => {
    if (req.user) {
      const variant = assignVariant(req.user.id, experimentConfig);

      if (variant) {
        req.experimentVariant = variant;
        req.experimentId = experimentConfig.id;

        // Track exposure (fire-and-forget)
        trackExposure({
          userId: req.user.id,
          experimentId: experimentConfig.id,
          variant,
          timestamp: new Date(),
          sessionId: req.session?.id
        }).catch(err => console.error('Exposure tracking error:', err));
      }
    }

    next();
  };
}
```

### Step 824: ClickHouse สำหรับ Analysis

```sql
-- ClickHouse schema สำหรับ experiment data

-- Exposures table
CREATE TABLE experiment_exposures (
    user_id      String,
    experiment_id String,
    variant      String,
    timestamp    DateTime64(3),
    session_id   Nullable(String),
    date         Date MATERIALIZED toDate(timestamp)
)
ENGINE = MergeTree()
PARTITION BY date
ORDER BY (experiment_id, user_id, timestamp);

-- Conversions table
CREATE TABLE experiment_conversions (
    user_id       String,
    experiment_id String,
    metric_name   String,
    value         Float64,
    timestamp     DateTime64(3),
    date          Date MATERIALIZED toDate(timestamp)
)
ENGINE = MergeTree()
PARTITION BY date
ORDER BY (experiment_id, user_id, metric_name, timestamp);

-- Analysis query: เปรียบเทียบ conversion rate ระหว่าง variants
SELECT
    e.experiment_id,
    e.variant,
    count(DISTINCT e.user_id) AS exposed_users,
    count(DISTINCT c.user_id) AS converted_users,
    count(DISTINCT c.user_id) / count(DISTINCT e.user_id) AS conversion_rate,
    avg(c.value) AS avg_metric_value,
    stddevPop(c.value) AS std_metric_value
FROM experiment_exposures e
LEFT JOIN experiment_conversions c
    ON e.user_id = c.user_id
    AND e.experiment_id = c.experiment_id
    AND c.timestamp >= e.timestamp  -- conversion หลัง exposure
    AND c.metric_name = 'post_engagement'
WHERE e.experiment_id = 'feed-algorithm-v2'
  AND e.date >= today() - 7
GROUP BY e.experiment_id, e.variant
ORDER BY conversion_rate DESC;
```

```python
# scripts/ab_test_analysis.py
"""
Statistical analysis สำหรับ A/B test results
"""

from scipy import stats
import clickhouse_driver
import pandas as pd
import numpy as np

def run_ab_analysis(experiment_id: str, metric_name: str, days_back: int = 7):
    """
    วิเคราะห์ผล A/B test และตัดสินใจ
    Returns: dict with results and recommendation
    """
    client = clickhouse_driver.Client(host='clickhouse.chuaikan.internal')

    # ดึงข้อมูลจาก ClickHouse
    query = f"""
        SELECT
            e.variant,
            count(DISTINCT e.user_id) AS n,
            sum(CASE WHEN c.user_id IS NOT NULL THEN 1 ELSE 0 END) AS conversions,
            avg(CASE WHEN c.value IS NOT NULL THEN c.value ELSE 0 END) AS avg_value
        FROM experiment_exposures e
        LEFT JOIN experiment_conversions c
            ON e.user_id = c.user_id
            AND e.experiment_id = c.experiment_id
            AND c.metric_name = '{metric_name}'
        WHERE e.experiment_id = '{experiment_id}'
          AND e.date >= today() - {days_back}
        GROUP BY e.variant
    """

    results = client.execute(query, with_column_types=True)
    df = pd.DataFrame(results[0], columns=[c[0] for c in results[1]])

    if len(df) < 2:
        return {"error": "Not enough variants"}

    control = df[df['variant'] == 'control'].iloc[0]
    treatment = df[df['variant'] != 'control'].iloc[0]

    # Chi-square test สำหรับ binary metric (conversion rate)
    ctrl_conv = int(control['conversions'])
    ctrl_n = int(control['n'])
    treat_conv = int(treatment['conversions'])
    treat_n = int(treatment['n'])

    contingency = [[ctrl_conv, ctrl_n - ctrl_conv],
                   [treat_conv, treat_n - treat_conv]]

    chi2, p_value, _, _ = stats.chi2_contingency(contingency)

    ctrl_rate = ctrl_conv / ctrl_n
    treat_rate = treat_conv / treat_n
    relative_lift = (treat_rate - ctrl_rate) / ctrl_rate

    significant = p_value < 0.05

    recommendation = "SHIP" if significant and relative_lift > 0 else \
                    "ROLLBACK" if significant and relative_lift < 0 else \
                    "ITERATE"  # ไม่ significant → ทดสอบต่อหรือ iterate

    return {
        "experiment_id": experiment_id,
        "metric": metric_name,
        "control": {
            "n": ctrl_n,
            "conversions": ctrl_conv,
            "rate": ctrl_rate
        },
        "treatment": {
            "n": treat_n,
            "conversions": treat_conv,
            "rate": treat_rate
        },
        "relative_lift": f"{relative_lift:.2%}",
        "p_value": f"{p_value:.4f}",
        "significant": significant,
        "recommendation": recommendation
    }

if __name__ == "__main__":
    import json
    result = run_ab_analysis("sos-button-color", "sos_button_ctr")
    print(json.dumps(result, indent=2))
```

### Step 825: Guardrail Metrics

```yaml
# k8s/monitoring/experiment-alerts.yaml
# Guardrail metrics: ตรวจสอบว่า experiment ไม่ทำให้ metrics สำคัญแย่ลง

apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: experiment-guardrails
  namespace: monitoring
spec:
  groups:
    - name: experiment.guardrails
      rules:
        # Guardrail: Error rate ไม่เพิ่ม
        - alert: ExperimentHighErrorRate
          expr: |
            sum(rate(http_requests_total{experiment_id!="",status=~"5.."}[5m]))
            by (experiment_id, variant)
            /
            sum(rate(http_requests_total{experiment_id!=""}[5m]))
            by (experiment_id, variant) > 0.02
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Experiment {{ $labels.experiment_id }} variant {{ $labels.variant }} has high error rate"

        # Guardrail: Latency ไม่เพิ่ม > 20%
        - alert: ExperimentHighLatency
          expr: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket{experiment_id!=""}[5m]))
              by (le, experiment_id, variant)
            ) > 0.2
          for: 5m
          labels:
            severity: warning
```

---

## ✅ Checklist

- [ ] Experiment design ชัดเจน: hypothesis, metric, MDE
- [ ] Sample size คำนวณก่อนเริ่ม experiment
- [ ] Hash-based bucketing deterministic
- [ ] Exposure events track ได้ครบ
- [ ] Conversion events connect กับ exposure ได้
- [ ] ClickHouse รับ experiment events ได้
- [ ] Analysis script รัน statistical significance test
- [ ] Guardrail metrics alert ทำงาน
- [ ] Decision framework ชัดเจน: SHIP/ROLLBACK/ITERATE
- [ ] Experiment archive หลังสิ้นสุด

---

## 🔗 References

- [Trustworthy A/B Testing - Airbnb](https://medium.com/airbnb-engineering/trustworthy-online-controlled-experiments-a-b-testing-e2f9b8a30f73)
- [ClickHouse Documentation](https://clickhouse.com/docs/)
- [Evan Miller Sample Size Calculator](https://www.evanmiller.org/ab-testing/sample-size.html)

---
*Part 083 | Road to 1,000,000 Users/Day | chuaikan.com*
