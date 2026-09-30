# Part 101: AI/ML Integration สำหรับ Feed Algorithm

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** World Class
> **Steps:** 1001-1010
> **เวลาโดยประมาณ:** 6 ชั่วโมง
> **Prerequisites:** Part 100 (Case Study), Part 026 (Feed Service)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

ML-powered Feed Ranking เป็นสิ่งที่ทำให้ Platform อย่าง TikTok, Instagram ประสบความสำเร็จ ใน Part นี้เราจะ:

- เปรียบเทียบ Rule-based vs ML Feed Ranking
- ออกแบบ Feature Engineering สำหรับ Feed
- Deploy ML Model ด้วย TensorFlow Serving
- Monitor Model Drift
- A/B Test ML vs Rule-based

---

## 📖 ทฤษฎีและแนวคิด

### 1. ML vs Rule-Based Feed Ranking

**Rule-Based (ที่ใช้อยู่ปัจจุบัน):**
```javascript
function rankPosts(posts, userId) {
  return posts.sort((a, b) => {
    const scoreA = a.createdAt * 0.5 + a.likesCount * 0.3 + a.commentsCount * 0.2;
    const scoreB = b.createdAt * 0.5 + b.likesCount * 0.3 + b.commentsCount * 0.2;
    return scoreB - scoreA;
  });
}
```

ข้อจำกัด:
- Weights เหมือนกันทุก user (ไม่ personalized)
- ต้อง tune manually
- ไม่เรียนรู้จาก user behavior

**ML-Based:**
```
Input Features → ML Model → Probability of Engagement

Features:
- User history (likes, shares, comments)
- Content features (type, length, hashtags)
- Context features (time of day, device, location)
- Social features (poster relationship strength)

Output:
- P(like) = 0.73
- P(comment) = 0.12
- P(share) = 0.08
- P(skip) = 0.07

Score = P(like)*w1 + P(comment)*w2 + P(share)*w3
```

### 2. Feature Engineering

```python
# feature-engineering.py

FEATURES = {
    # User Context Features
    'user_avg_session_length': 'float',
    'user_daily_active_ratio': 'float',  # % days active in last 30 days
    'user_sos_engagement_rate': 'float',  # ชอบ SOS content แค่ไหน
    'user_time_of_day_preference': 'int',  # 0-23 peak usage hour
    'user_content_type_preference': 'categorical',  # text/image/video/sos
    
    # Content Features
    'post_type': 'categorical',  # sos/news/community/update
    'post_age_hours': 'float',   # อายุของ post
    'post_like_velocity': 'float',  # likes per hour (trending?)
    'post_engagement_rate': 'float',  # engagement/views
    'post_has_images': 'bool',
    'post_has_location': 'bool',
    'post_location_distance_km': 'float',  # ระยะห่างจาก user
    
    # Social Features
    'poster_follow_strength': 'float',  # 0-1: แค่ follow หรือ interact บ่อย
    'mutual_friends_count': 'int',      # จำนวน mutual friends
    'poster_credibility_score': 'float',  # poster น่าเชื่อถือแค่ไหน (SOS)
    
    # Context Features
    'request_hour': 'int',       # 0-23
    'request_day_of_week': 'int', # 0-6
    'is_disaster_event': 'bool',   # มี active disaster ในพื้นที่?
    'nearby_sos_count': 'int',     # SOS ใกล้ user กี่ alert
}
```

---

## 🛠️ Step-by-Step Implementation

### Step 1001: Feature Store

```python
# feature-store.py
# เก็บ features ที่ compute แล้วเพื่อ reuse

import redis
import json
import numpy as np
from datetime import datetime, timedelta

class FeatureStore:
    def __init__(self, redis_client, db_pool):
        self.redis = redis_client
        self.db = db_pool
    
    async def get_user_features(self, user_id):
        """ดึง user features พร้อม cache"""
        cache_key = f"features:user:{user_id}"
        cached = await self.redis.get(cache_key)
        
        if cached:
            return json.loads(cached)
        
        features = await self._compute_user_features(user_id)
        
        # Cache 1 ชั่วโมง (user behavior ไม่เปลี่ยนบ่อย)
        await self.redis.setex(cache_key, 3600, json.dumps(features))
        
        return features
    
    async def _compute_user_features(self, user_id):
        # Query user behavior ใน 30 วันที่ผ่านมา
        query = """
        SELECT
            AVG(session_duration_minutes) as avg_session_length,
            COUNT(DISTINCT DATE(created_at)) / 30.0 as daily_active_ratio,
            SUM(CASE WHEN post_type = 'sos' AND action = 'like' THEN 1 ELSE 0 END)::float /
                NULLIF(SUM(CASE WHEN post_type = 'sos' THEN 1 ELSE 0 END), 0) as sos_engagement_rate,
            MODE() WITHIN GROUP (ORDER BY EXTRACT(HOUR FROM created_at)) as peak_hour,
            MODE() WITHIN GROUP (ORDER BY preferred_content_type) as content_preference
        FROM user_interactions ui
        JOIN posts p ON ui.post_id = p.id
        WHERE ui.user_id = $1
          AND ui.created_at > NOW() - INTERVAL '30 days'
        """
        
        result = await self.db.fetchrow(query, user_id)
        
        return {
            'user_avg_session_length': float(result['avg_session_length'] or 15),
            'user_daily_active_ratio': float(result['daily_active_ratio'] or 0.5),
            'user_sos_engagement_rate': float(result['sos_engagement_rate'] or 0.3),
            'user_time_of_day_preference': int(result['peak_hour'] or 12),
            'user_content_type_preference': result['content_preference'] or 'mixed',
        }
    
    async def get_post_features(self, post_id, user_location=None):
        """ดึง post features"""
        cache_key = f"features:post:{post_id}"
        cached = await self.redis.get(cache_key)
        
        if cached:
            base_features = json.loads(cached)
        else:
            base_features = await self._compute_post_features(post_id)
            # Cache 5 นาที (post engagement changes quickly)
            await self.redis.setex(cache_key, 300, json.dumps(base_features))
        
        # เพิ่ม user-specific features ที่ cache ไม่ได้
        if user_location:
            base_features['post_location_distance_km'] = self._calculate_distance(
                user_location,
                base_features.get('post_location')
            )
        
        return base_features
```

### Step 1002: Model Training

```python
# train-feed-model.py

import pandas as pd
import numpy as np
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import roc_auc_score, classification_report
import mlflow
import mlflow.sklearn
import joblib

# 1. Load training data
print("Loading training data...")
df = pd.read_parquet('s3://chuaikan-ml/training-data/feed-interactions-2024.parquet')

print(f"Training samples: {len(df):,}")
print(f"Positive rate: {df['engaged'].mean():.2%}")

# Features ที่ใช้
FEATURE_COLUMNS = [
    'user_avg_session_length',
    'user_daily_active_ratio',
    'user_sos_engagement_rate',
    'user_time_of_day_preference',
    'post_age_hours',
    'post_like_velocity',
    'post_engagement_rate',
    'post_has_images',
    'post_has_location',
    'post_location_distance_km',
    'poster_follow_strength',
    'mutual_friends_count',
    'request_hour',
    'request_day_of_week',
    'is_disaster_event',
    'nearby_sos_count',
]

X = df[FEATURE_COLUMNS].fillna(0)
y = df['engaged']  # 1 = liked/commented/shared, 0 = skipped

# 2. Train/test split (time-based — ไม่ random!)
# ใช้ data ล่าสุดเป็น test set
cutoff_date = df['date'].quantile(0.8)
train_mask = df['date'] <= cutoff_date

X_train, X_test = X[train_mask], X[~train_mask]
y_train, y_test = y[train_mask], y[~train_mask]

# 3. Train model ด้วย MLflow tracking
with mlflow.start_run(run_name="feed-ranking-v3"):
    mlflow.log_params({
        'model_type': 'GradientBoosting',
        'n_estimators': 200,
        'max_depth': 6,
        'learning_rate': 0.1,
        'features': FEATURE_COLUMNS,
    })
    
    model = GradientBoostingClassifier(
        n_estimators=200,
        max_depth=6,
        learning_rate=0.1,
        subsample=0.8,
        random_state=42,
    )
    
    model.fit(X_train, y_train)
    
    # Evaluate
    y_pred_proba = model.predict_proba(X_test)[:, 1]
    auc = roc_auc_score(y_test, y_pred_proba)
    
    mlflow.log_metric('auc_roc', auc)
    mlflow.sklearn.log_model(model, 'feed-ranking-model')
    
    print(f"AUC-ROC: {auc:.4f}")
    print(classification_report(y_test, (y_pred_proba > 0.5).astype(int)))
    
    # Feature importance
    importance = pd.Series(model.feature_importances_, index=FEATURE_COLUMNS)
    print("\nTop Features:")
    print(importance.sort_values(ascending=False).head(10))

print(f"\nModel saved: Run ID = {mlflow.active_run().info.run_id}")
```

### Step 1003: Model Serving

```python
# model-server.py
# Serve ML model ด้วย FastAPI

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
import joblib
import asyncio
import logging

app = FastAPI(title="Feed Ranking Model Server")

# Load model เมื่อ start
model = None
feature_store = None

@app.on_event("startup")
async def load_model():
    global model, feature_store
    
    # Load model จาก S3 (ใน production)
    model = joblib.load('/models/feed-ranking-v3.pkl')
    feature_store = FeatureStore(redis_client, db_pool)
    
    logging.info("Model loaded successfully")

class RankRequest(BaseModel):
    user_id: int
    post_ids: List[int]
    context: Optional[dict] = {}

class RankedPost(BaseModel):
    post_id: int
    score: float
    features: Optional[dict]

@app.post("/rank", response_model=List[RankedPost])
async def rank_posts(request: RankRequest):
    if len(request.post_ids) == 0:
        return []
    
    if len(request.post_ids) > 100:
        raise HTTPException(400, "Too many posts (max 100)")
    
    # ดึง features แบบ parallel
    user_features_task = feature_store.get_user_features(request.user_id)
    post_features_tasks = [
        feature_store.get_post_features(post_id, request.context.get('location'))
        for post_id in request.post_ids
    ]
    
    user_features, *post_features_list = await asyncio.gather(
        user_features_task, *post_features_tasks
    )
    
    # Build feature matrix
    feature_rows = []
    for post_features in post_features_list:
        row = {**user_features, **post_features}
        feature_rows.append([row.get(f, 0) for f in FEATURE_COLUMNS])
    
    X = np.array(feature_rows)
    
    # Predict
    scores = model.predict_proba(X)[:, 1]  # P(engage)
    
    # Return ranked posts
    results = [
        RankedPost(post_id=post_id, score=float(score))
        for post_id, score in zip(request.post_ids, scores)
    ]
    
    return sorted(results, key=lambda x: x.score, reverse=True)

@app.get("/health")
def health():
    return {"status": "ok", "model_loaded": model is not None}
```

### Step 1004: Model Monitoring

```python
# model-monitoring.py
# ตรวจ Data Drift และ Concept Drift

import scipy.stats as stats
import numpy as np
from datetime import datetime, timedelta

class ModelMonitor:
    def __init__(self, reference_data, model):
        # Reference distribution จาก training data
        self.reference_stats = self._compute_stats(reference_data)
        self.model = model
        self.alert_threshold = 0.05  # p-value threshold
    
    def _compute_stats(self, data):
        stats = {}
        for feature in FEATURE_COLUMNS:
            stats[feature] = {
                'mean': np.mean(data[feature]),
                'std': np.std(data[feature]),
                'percentiles': np.percentile(data[feature], [25, 50, 75]),
            }
        return stats
    
    def detect_data_drift(self, current_data):
        """ใช้ KS test ตรวจ distribution shift"""
        drifted_features = []
        
        for feature in FEATURE_COLUMNS:
            ref_values = self._get_reference_values(feature)
            cur_values = current_data[feature].values
            
            ks_stat, p_value = stats.ks_2samp(ref_values, cur_values)
            
            if p_value < self.alert_threshold:
                drifted_features.append({
                    'feature': feature,
                    'ks_statistic': ks_stat,
                    'p_value': p_value,
                    'current_mean': np.mean(cur_values),
                    'reference_mean': self.reference_stats[feature]['mean'],
                })
        
        return drifted_features
    
    def detect_concept_drift(self, predictions, actuals):
        """ตรวจว่า model accuracy ลดลงหรือเปล่า"""
        from sklearn.metrics import roc_auc_score
        
        current_auc = roc_auc_score(actuals, predictions)
        
        # Alert ถ้า AUC ลดลงเกิน 5% จาก baseline
        baseline_auc = 0.78  # AUC ตอน deploy
        
        if current_auc < baseline_auc * 0.95:
            return {
                'drift_detected': True,
                'current_auc': current_auc,
                'baseline_auc': baseline_auc,
                'degradation': (baseline_auc - current_auc) / baseline_auc,
                'action': 'retrain_required',
            }
        
        return {'drift_detected': False, 'current_auc': current_auc}
```

---

## 🔧 Configuration Files

### MLflow Tracking Server

```yaml
# mlflow-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mlflow-server
  namespace: ml
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: mlflow
        image: ghcr.io/mlflow/mlflow:2.9.0
        command:
          - mlflow
          - server
          - --backend-store-uri
          - postgresql://mlflow:password@postgres-ml/mlflow
          - --default-artifact-root
          - s3://chuaikan-ml/artifacts
          - --host
          - "0.0.0.0"
          - --port
          - "5000"
        env:
        - name: AWS_DEFAULT_REGION
          value: ap-southeast-1
        ports:
        - containerPort: 5000
```

---

## 🧪 Testing

### A/B Test ML vs Rule-Based

```javascript
// ab-test-config.js
const abTestConfig = {
  experimentId: 'feed-ml-vs-rule-based',
  description: 'Compare ML ranking vs rule-based ranking',
  startDate: '2024-02-01',
  endDate: '2024-03-01',
  
  variants: [
    { id: 'control', name: 'Rule-Based', allocation: 0.5 },
    { id: 'treatment', name: 'ML Ranking', allocation: 0.5 },
  ],
  
  metrics: {
    primary: 'session_engagement_rate',
    secondary: ['time_spent', 'posts_viewed', 'sos_alert_rate'],
  },
  
  successCriteria: {
    session_engagement_rate: { minImprovement: 0.05, significance: 0.95 },
  },
};
```

---

## ❌ Common Errors & Solutions

### Error 1: Model Latency สูงเกินไป

```python
# ปัญหา: model inference ใช้เวลา 200ms
# แก้: batch prediction และ async

# ❌ ช้า: predict ทีละ post
for post_id in post_ids:
    score = model.predict([get_features(user_id, post_id)])

# ✅ เร็ว: batch predict ทีเดียว
features_batch = [get_features(user_id, post_id) for post_id in post_ids]
scores = model.predict_proba(features_batch)[:, 1]
```

### Error 2: Feature Skew (Training-Serving Skew)

```
ปัญหา: features ใน training ≠ features ตอน serve → model accuracy ลดลง

เช่น: training ใช้ post_age_minutes แต่ serving ใช้ post_age_hours
```

```python
# แก้: ใช้ shared feature computation code ทั้ง training และ serving
# จาก Feature Store เดียวกัน — ไม่ compute ซ้ำ
from feature_store import FeatureStore, FEATURE_COLUMNS

# Training
features = feature_store.get_post_features_batch(post_ids)

# Serving (ใช้ code เดียวกัน)
features = feature_store.get_post_features_batch(post_ids)
```

---

## ✅ Checklist

- [ ] Feature Engineering ออกแบบแล้ว (≥ 10 features)
- [ ] MLflow tracking server ตั้งค่าแล้ว
- [ ] Model training pipeline อัตโนมัติ (weekly retrain)
- [ ] A/B test infrastructure พร้อม
- [ ] Model serving API (< 50ms P99)
- [ ] Data drift monitoring เปิดใช้งาน
- [ ] Concept drift alerts ตั้งค่าแล้ว
- [ ] Feature Store ป้องกัน training-serving skew
- [ ] Fallback: ถ้า ML model ล้ม → rule-based ranking อัตโนมัติ
- [ ] Model performance dashboard ใน Grafana

---

## 🔗 References

- [MLflow Documentation](https://mlflow.org/docs/latest/index.html)
- [Feature Store — Feast](https://feast.dev/)
- [TensorFlow Serving](https://www.tensorflow.org/tfx/guide/serving)
- [Evidently AI — ML Monitoring](https://www.evidentlyai.com/)
- [Netflix Recommendation System](https://research.netflix.com/research-area/recommendations)

---

*Part 101 | Road to 1,000,000 Users/Day | chuaikan.com*
