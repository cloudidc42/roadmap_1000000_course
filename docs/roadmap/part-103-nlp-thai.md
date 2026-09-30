# Part 103: NLP สำหรับ Thai Language Processing
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** World Class
> **Steps:** 1021-1030
> **เวลาโดยประมาณ:** 12 ชั่วโมง
> **Prerequisites:** Part 072 (Kafka), Part 075 (Push Notifications), Part 016 (API Design)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ความท้าทายของภาษาไทย: ไม่มีช่องว่างระหว่างคำ
- PyThaiNLP 5.x: word tokenization, POS tagging, NER
- Thai NLP ใน Node.js ผ่าน Python subprocess
- Sentiment analysis สำหรับ SOS content moderation
- Thai keyword extraction และ text classification
- Named Entity Recognition: ดึงชื่อสถานที่จาก SOS
- Multilingual embeddings สำหรับ semantic search
- FastAPI microservice สำหรับ NLP
- Integration กับ SOS service

---

## 📖 ทฤษฎีและแนวคิด

### ทำไมภาษาไทยถึงยาก?

```
ภาษาอังกฤษ: "flood in Ayutthaya very serious"
                ↕ ง่าย: แบ่งด้วย space
ภาษาไทย:    "น้ำท่วมที่อยุธยาหนักมาก"
                ↕ ยาก: ไม่มีช่องว่าง!
ต้องแบ่งเป็น: น้ำท่วม / ที่ / อยุธยา / หนัก / มาก
```

### ขั้นตอน NLP Pipeline สำหรับ SOS

```
SOS Text Input (Thai)
    ↓ Word Tokenization (newmm engine)
    ↓ POS Tagging
    ↓ Named Entity Recognition (location)
    ↓ Sentiment Analysis (urgent/normal)
    ↓ Classification (flood/fire/accident/other)
    ↓ Keyword Extraction
Output → structured JSON สำหรับ SOS system
```

---

## ⚙️ Environment Setup

### Step 1021: ติดตั้ง Python Environment

```bash
# ติดตั้ง Python 3.11 (บน Ubuntu 24.04)
sudo apt update
sudo apt install -y python3.11 python3.11-venv python3-pip

# สร้าง virtual environment สำหรับ NLP service
python3.11 -m venv /opt/chuaikan-nlp/venv
source /opt/chuaikan-nlp/venv/bin/activate

# ติดตั้ง PyThaiNLP และ dependencies
pip install \
  pythainlp==5.0.0 \
  fastapi==0.115.0 \
  uvicorn==0.30.0 \
  scikit-learn==1.5.0 \
  sentence-transformers==3.1.0 \
  numpy==1.26.0 \
  pandas==2.2.0 \
  transformers==4.44.0 \
  torch==2.4.0 \
  anthropic==0.34.0

# Download Thai dictionaries สำหรับ PyThaiNLP
python -c "from pythainlp.corpus import download; download('best')"
python -c "from pythainlp.corpus import download; download('tha-wikitext-20210620-newmm')"
```

### Step 1022: ทดสอบ PyThaiNLP พื้นฐาน

```python
# test_pythainlp.py
from pythainlp.tokenize import word_tokenize
from pythainlp.tag import pos_tag
from pythainlp.corpus.common import thai_words

# Word tokenization
text = "น้ำท่วมหนักมากที่อยุธยา ต้องการความช่วยเหลือด่วน"
tokens = word_tokenize(text, engine="newmm")
print("Tokens:", tokens)
# Output: ['น้ำท่วม', 'หนัก', 'มาก', 'ที่', 'อยุธยา', ' ', 'ต้องการ', 'ความ', 'ช่วยเหลือ', 'ด่วน']

# POS Tagging
pos = pos_tag(tokens, corpus="orchid_ud")
print("POS:", pos)
# Output: [('น้ำท่วม', 'NOUN'), ('หนัก', 'ADJ'), ...]

# Named Entity Recognition
from pythainlp.tag import ner
entities = ner(text, pos_tag="perceptron", corpus="thainer")
print("Entities:", entities)
# Output: [('น้ำท่วม', 'O'), ('อยุธยา', 'B-LOC'), ...]
```

---

## 🛠️ Step-by-Step Implementation

### Step 1023: FastAPI NLP Microservice

```python
# /opt/chuaikan-nlp/app/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional, List
import logging

from .nlp_processor import ThaiNLPProcessor
from .sentiment import ThaiSentimentAnalyzer
from .classifier import SOSClassifier

app = FastAPI(title="Thai NLP Service", version="1.0.0")
logger = logging.getLogger(__name__)

# Initialize processors (โหลดครั้งเดียวตอน startup)
nlp = ThaiNLPProcessor()
sentiment = ThaiSentimentAnalyzer()
classifier = SOSClassifier()

class AnalyzeRequest(BaseModel):
    text: str
    include_entities: bool = True
    include_sentiment: bool = True
    include_classification: bool = True
    include_keywords: bool = True

class AnalyzeResponse(BaseModel):
    tokens: List[str]
    entities: List[dict]
    sentiment: Optional[dict]
    classification: Optional[dict]
    keywords: List[str]
    locations: List[str]
    processingMs: float

@app.post("/analyze", response_model=AnalyzeResponse)
async def analyze_text(request: AnalyzeRequest):
    import time
    start = time.time()

    try:
        result = nlp.analyze(request.text)
        
        if request.include_sentiment:
            result['sentiment'] = sentiment.analyze(request.text)
        
        if request.include_classification:
            result['classification'] = classifier.classify(request.text)
        
        result['processingMs'] = (time.time() - start) * 1000
        return result

    except Exception as e:
        logger.error(f"NLP Error: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health():
    return {"status": "ok", "pythainlp_version": "5.0.0"}
```

### Step 1024: Thai NLP Processor

```python
# /opt/chuaikan-nlp/app/nlp_processor.py
from pythainlp.tokenize import word_tokenize
from pythainlp.tag import pos_tag, ner
from pythainlp.util.trie import Trie
from pythainlp.corpus import thai_stopwords
from collections import Counter
import re

# ชื่อจังหวัดไทยทั้งหมด สำหรับ location detection
THAI_PROVINCES = {
    "กรุงเทพ", "กรุงเทพมหานคร", "กทม", "เชียงใหม่", "เชียงราย",
    "อยุธยา", "พระนครศรีอยุธยา", "นนทบุรี", "ปทุมธานี", "สมุทรปราการ",
    "นครราชสีมา", "โคราช", "อุบลราชธานี", "ขอนแก่น", "อุดรธานี",
    "สุราษฎร์ธานี", "นครศรีธรรมราช", "ภูเก็ต", "สงขลา", "หาดใหญ่",
    "ชลบุรี", "พัทยา", "ระยอง", "นครปฐม", "กาญจนบุรี",
}

class ThaiNLPProcessor:
    def __init__(self):
        self.stopwords = thai_stopwords()
        # Custom dictionary สำหรับ SOS terms
        sos_words = ["น้ำท่วม", "ไฟไหม้", "อุบัติเหตุ", "ดินถล่ม", "วาตภัย", "แผ่นดินไหว"]
        self.custom_trie = Trie(sos_words)
        
    def tokenize(self, text: str) -> list[str]:
        """Word tokenization ด้วย newmm engine"""
        # ทำความสะอาด text
        text = re.sub(r'\s+', ' ', text.strip())
        tokens = word_tokenize(text, engine="newmm", custom_dict=self.custom_trie)
        return [t for t in tokens if t.strip()]  # ลบ whitespace tokens

    def extract_entities(self, text: str) -> list[dict]:
        """Named Entity Recognition"""
        try:
            tokens = self.tokenize(text)
            tagged = ner(text, corpus="thainer")
            
            entities = []
            current_entity = None
            
            for word, tag in tagged:
                if tag.startswith('B-'):  # Beginning of entity
                    if current_entity:
                        entities.append(current_entity)
                    current_entity = {'text': word, 'type': tag[2:], 'words': [word]}
                elif tag.startswith('I-') and current_entity:  # Inside entity
                    current_entity['text'] += word
                    current_entity['words'].append(word)
                else:
                    if current_entity:
                        entities.append(current_entity)
                        current_entity = None
            
            if current_entity:
                entities.append(current_entity)
                
            return entities
        except Exception:
            return []

    def extract_locations(self, text: str) -> list[str]:
        """ดึงชื่อสถานที่จากข้อความ"""
        locations = []
        
        # ตรวจสอบชื่อจังหวัดที่รู้จัก
        for province in THAI_PROVINCES:
            if province in text:
                locations.append(province)
        
        # ดึงจาก NER entities ด้วย
        entities = self.extract_entities(text)
        for entity in entities:
            if entity['type'] in ('LOC', 'GPE'):
                locations.append(entity['text'])
        
        return list(set(locations))  # ลบ duplicates

    def extract_keywords(self, text: str, top_n: int = 10) -> list[str]:
        """Extract keywords โดยใช้ TF-IDF หลักการ"""
        tokens = self.tokenize(text)
        
        # ลบ stopwords และ tokens สั้นๆ
        meaningful_tokens = [
            t for t in tokens
            if t not in self.stopwords and len(t) > 1 and not t.isdigit()
        ]
        
        # Count frequency
        freq = Counter(meaningful_tokens)
        return [word for word, _ in freq.most_common(top_n)]

    def analyze(self, text: str) -> dict:
        """Full NLP analysis"""
        tokens = self.tokenize(text)
        entities = self.extract_entities(text)
        locations = self.extract_locations(text)
        keywords = self.extract_keywords(text)
        
        return {
            'tokens': tokens,
            'entities': entities,
            'locations': locations,
            'keywords': keywords,
        }
```

### Step 1025: Sentiment Analysis

```python
# /opt/chuaikan-nlp/app/sentiment.py
from pythainlp.sentiment import sentiment as thai_sentiment
from pythainlp.tokenize import word_tokenize

# Keywords ที่บ่งบอกความด่วน
URGENCY_KEYWORDS = {
    "ด่วน", "ฉุกเฉิน", "เร่งด่วน", "ช่วยด้วย", "ขอความช่วยเหลือ",
    "อันตราย", "อันตรายมาก", "ติดอยู่", "จมน้ำ", "บาดเจ็บ", "เสียชีวิต",
    "ไฟไหม้", "ระเบิด", "ดินถล่ม", "น้ำท่วมสูง",
}

SEVERITY_KEYWORDS = {
    "critical": {"เสียชีวิต", "จมน้ำ", "ระเบิด", "ไฟไหม้ลาม"},
    "high": {"บาดเจ็บสาหัส", "ติดอยู่", "น้ำท่วมสูง", "ต้องการความช่วยเหลือด่วน"},
    "medium": {"น้ำท่วม", "ไฟไหม้", "อุบัติเหตุ", "บาดเจ็บ"},
    "low": {"น้ำขัง", "ถนนลื่น", "ต้นไม้ล้ม"},
}

class ThaiSentimentAnalyzer:
    def analyze(self, text: str) -> dict:
        """
        Analyze sentiment ของ SOS text
        Returns: { sentiment, urgency_score, severity, is_urgent }
        """
        # Basic sentiment
        try:
            base_sentiment = thai_sentiment(text)  # positive/negative/neutral
        except Exception:
            base_sentiment = "neutral"
        
        tokens = set(word_tokenize(text, engine="newmm"))
        
        # Urgency score (0.0 - 1.0)
        urgency_count = len(tokens & URGENCY_KEYWORDS)
        urgency_score = min(urgency_count / 3.0, 1.0)
        
        # Severity level
        severity = "low"
        for level in ("critical", "high", "medium", "low"):
            if tokens & SEVERITY_KEYWORDS[level]:
                severity = level
                break
        
        return {
            "sentiment": base_sentiment,
            "urgency_score": round(urgency_score, 2),
            "severity": severity,
            "is_urgent": urgency_score > 0.3 or severity in ("critical", "high"),
        }
```

### Step 1026: SOS Text Classifier

```python
# /opt/chuaikan-nlp/app/classifier.py
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
from pythainlp.tokenize import word_tokenize
import pickle
import os

# Training data (ตัวอย่างขั้นต่ำ - ใน production ใช้ dataset จริง)
TRAINING_DATA = [
    # flood
    ("น้ำท่วมถนน บ้านจมน้ำ ต้องการเรือ", "flood"),
    ("น้ำท่วมสูงมาก ระดับน้ำขึ้นเร็ว อพยพไม่ทัน", "flood"),
    ("น้ำท่วมหนักที่อยุธยา ขอความช่วยเหลือ", "flood"),
    # fire
    ("ไฟไหม้บ้านข้างๆ ควันลาม ขอรถดับเพลิง", "fire"),
    ("เพลิงไหม้โรงงาน เปลวไฟสูง ไม่มีใครช่วย", "fire"),
    # accident
    ("รถชนกันบนทางหลวง มีคนบาดเจ็บ ต้องการรถพยาบาล", "accident"),
    ("อุบัติเหตุรถมอเตอร์ไซค์ บาดเจ็บสาหัส", "accident"),
    # other
    ("ต้นไม้ล้มทับรถ ต้องการความช่วยเหลือ", "other"),
    ("ไฟฟ้าดับทั้งหมู่บ้าน ลมแรงมาก", "other"),
]

class SOSClassifier:
    def __init__(self):
        self.vectorizer = TfidfVectorizer(
            analyzer=self._thai_tokenizer,
            ngram_range=(1, 2),
            max_features=5000,
        )
        self.model = MultinomialNB(alpha=0.1)
        self._train()

    def _thai_tokenizer(self, text: str) -> list[str]:
        return word_tokenize(text, engine="newmm")

    def _train(self):
        texts, labels = zip(*TRAINING_DATA)
        X = self.vectorizer.fit_transform(texts)
        self.model.fit(X, labels)

    def classify(self, text: str) -> dict:
        X = self.vectorizer.transform([text])
        prediction = self.model.predict(X)[0]
        probabilities = self.model.predict_proba(X)[0]
        classes = self.model.classes_

        confidence = dict(zip(classes, [round(p, 3) for p in probabilities]))

        return {
            "category": prediction,
            "confidence": confidence,
            "top_category": prediction,
            "confidence_score": round(max(probabilities), 3),
        }
```

### Step 1027: Semantic Search ด้วย Embeddings

```python
# /opt/chuaikan-nlp/app/embeddings.py
from sentence_transformers import SentenceTransformer
import numpy as np
from fastapi import APIRouter

router = APIRouter(prefix="/embeddings")

# Multilingual model รองรับภาษาไทย
model = SentenceTransformer('paraphrase-multilingual-MiniLM-L12-v2')

@router.post("/encode")
async def encode_text(request: dict):
    """
    แปลง text เป็น vector embedding สำหรับ semantic search
    Model: paraphrase-multilingual-MiniLM-L12-v2 (384 dimensions)
    """
    text = request.get('text', '')
    embedding = model.encode(text, normalize_embeddings=True)
    return {"embedding": embedding.tolist(), "dimensions": len(embedding)}

@router.post("/similar")
async def find_similar(request: dict):
    """
    หา similarity ระหว่างข้อความ 2 ข้อความ
    """
    text1 = request.get('text1', '')
    text2 = request.get('text2', '')

    embeddings = model.encode([text1, text2], normalize_embeddings=True)
    similarity = float(np.dot(embeddings[0], embeddings[1]))

    return {
        "similarity": round(similarity, 4),
        "are_similar": similarity > 0.7,
    }
```

### Step 1028: Node.js Integration

```typescript
// src/lib/nlp/thai-nlp-client.ts
import axios from 'axios';

const NLP_SERVICE_URL = process.env.NLP_SERVICE_URL || 'http://localhost:8001';

export interface NLPAnalysisResult {
  tokens: string[];
  entities: Array<{ text: string; type: string }>;
  locations: string[];
  keywords: string[];
  sentiment?: {
    sentiment: string;
    urgency_score: number;
    severity: 'low' | 'medium' | 'high' | 'critical';
    is_urgent: boolean;
  };
  classification?: {
    category: 'flood' | 'fire' | 'accident' | 'other';
    confidence_score: number;
  };
  processingMs: number;
}

export async function analyzeThaiText(text: string): Promise<NLPAnalysisResult> {
  const response = await axios.post<NLPAnalysisResult>(
    `${NLP_SERVICE_URL}/analyze`,
    {
      text,
      include_entities: true,
      include_sentiment: true,
      include_classification: true,
      include_keywords: true,
    },
    { timeout: 5000 }
  );
  return response.data;
}

export async function getTextEmbedding(text: string): Promise<number[]> {
  const response = await axios.post<{ embedding: number[] }>(
    `${NLP_SERVICE_URL}/embeddings/encode`,
    { text },
    { timeout: 10000 }
  );
  return response.data.embedding;
}
```

### Step 1029: Auto-categorize SOS Posts

```typescript
// src/services/sos-nlp-service.ts
import { analyzeThaiText } from '../lib/nlp/thai-nlp-client';
import { getTextEmbedding } from '../lib/nlp/thai-nlp-client';
import { db } from '../lib/db/postgres';
import Anthropic from '@anthropic-ai/sdk';

const anthropic = new Anthropic();

export async function processSOSPost(postId: string, content: string): Promise<{
  category: string;
  severity: string;
  locations: string[];
  isUrgent: boolean;
}> {
  let category = 'other';
  let severity = 'medium';
  let locations: string[] = [];
  let isUrgent = false;

  try {
    // ใช้ PyThaiNLP เป็นหลัก
    const nlpResult = await analyzeThaiText(content);

    category = nlpResult.classification?.category || 'other';
    severity = nlpResult.sentiment?.severity || 'medium';
    locations = nlpResult.locations;
    isUrgent = nlpResult.sentiment?.is_urgent || false;

    // ถ้า confidence ต่ำ ใช้ Claude AI เป็น fallback
    if ((nlpResult.classification?.confidence_score || 0) < 0.6) {
      const claudeResult = await fallbackToClaudeAI(content);
      category = claudeResult.category;
      severity = claudeResult.severity;
      if (claudeResult.locations.length > 0) {
        locations = claudeResult.locations;
      }
    }

    // บันทึก embedding สำหรับ semantic search
    const embedding = await getTextEmbedding(content);

    // อัพเดท SOS post
    await db.query(
      `UPDATE posts SET
         sos_category = $2,
         sos_severity = $3,
         location_province = COALESCE($4, location_province),
         embedding = $5,
         nlp_processed_at = NOW()
       WHERE id = $1`,
      [postId, category, severity, locations[0], JSON.stringify(embedding)]
    );

    console.log(`[NLP] SOS ${postId}: ${category}/${severity} locations=${locations.join(',')}`);

  } catch (error) {
    console.error('[NLP] Processing failed:', error);
  }

  return { category, severity, locations, isUrgent };
}

async function fallbackToClaudeAI(content: string) {
  const message = await anthropic.messages.create({
    model: 'claude-opus-4-5',
    max_tokens: 256,
    messages: [
      {
        role: 'user',
        content: `วิเคราะห์ข้อความ SOS ภาษาไทยต่อไปนี้และตอบเป็น JSON เท่านั้น:
"${content}"

Format: {"category": "flood|fire|accident|other", "severity": "low|medium|high|critical", "locations": ["province_name"]}`,
      },
    ],
  });

  const text = message.content[0].type === 'text' ? message.content[0].text : '{}';
  return JSON.parse(text);
}
```

---

## 🔧 Configuration Files

### Systemd Service สำหรับ FastAPI NLP

```bash
sudo tee /etc/systemd/system/chuaikan-nlp.service << 'EOF'
[Unit]
Description=Chuaikan Thai NLP Service
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/opt/chuaikan-nlp
Environment="PATH=/opt/chuaikan-nlp/venv/bin"
ExecStart=/opt/chuaikan-nlp/venv/bin/uvicorn app.main:app --host 127.0.0.1 --port 8001 --workers 4
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable chuaikan-nlp
sudo systemctl start chuaikan-nlp
```

---

## 🧪 Testing

### Test PyThaiNLP

```bash
# Test basic tokenization
curl -X POST http://localhost:8001/analyze \
  -H "Content-Type: application/json" \
  -d '{"text":"น้ำท่วมหนักมากที่อยุธยา ต้องการความช่วยเหลือด่วน"}'
```

### Test Semantic Search

```bash
# ทดสอบ similarity
curl -X POST http://localhost:8001/embeddings/similar \
  -H "Content-Type: application/json" \
  -d '{"text1":"น้ำท่วม","text2":"น้ำหลาก"}'
# คาด: similarity > 0.8
```

---

## ❌ Common Errors & Solutions

### Error 1: PyThaiNLP Dictionary ไม่พบ

```
FileNotFoundError: [Errno 2] No such file or directory: 'best'
```

**แก้ไข:**
```python
from pythainlp.corpus import download
download('best')
download('tha-wikitext-20210620-newmm')
```

### Error 2: Out of Memory (BERT model)

**สาเหตุ:** sentence-transformers โหลด model ใหญ่

**แก้ไข:**
```python
# ใช้ quantized model
model = SentenceTransformer('paraphrase-multilingual-MiniLM-L12-v2')
# ขนาด: ~470MB RAM (acceptable)
```

### Error 3: Thai Text Encoding

**แก้ไข:**
```python
# ตั้งค่า encoding ใน Python
import sys
sys.stdout.reconfigure(encoding='utf-8')
```

---

## ✅ Checklist

- [ ] **Step 1021:** Python 3.11 + PyThaiNLP 5.x ติดตั้งสำเร็จ
- [ ] **Step 1022:** `word_tokenize("น้ำท่วมที่อยุธยา")` ให้ผลลัพธ์ถูกต้อง
- [ ] **Step 1023:** FastAPI NLP service รันที่ port 8001
- [ ] **Step 1024:** NLP processor tokenize + NER ทำงานได้
- [ ] **Step 1025:** Sentiment analysis ระบุ urgency ได้
- [ ] **Step 1026:** Classifier จำแนก flood/fire/accident/other ได้
- [ ] **Step 1027:** Sentence embeddings encode Thai text ได้
- [ ] **Step 1028:** Node.js client เรียก NLP service สำเร็จ
- [ ] **Step 1029:** SOS posts auto-categorize และ extract locations
- [ ] **Step 1030:** Claude AI fallback ทำงานเมื่อ confidence ต่ำ

---

## 🔗 References

- [PyThaiNLP Documentation](https://pythainlp.github.io/pythainlp-doc/5.0/api/)
- [Sentence Transformers](https://www.sbert.net/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [ThaiNER Dataset](https://github.com/wannaphong/thai-ner)

---
*Part 103 | Road to 1,000,000 Users/Day | chuaikan.com*
