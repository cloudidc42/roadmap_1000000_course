# Part 103: NLP สำหรับ Thai Language Processing

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** World Class
> **Steps:** 1021-1030
> **เวลาโดยประมาณ:** 5 ชั่วโมง
> **Prerequisites:** Part 101 (AI/ML Integration), Part 102 (Computer Vision)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

ภาษาไทยมีความท้าทายพิเศษสำหรับ NLP เพราะไม่มีช่องว่างระหว่างคำ ใน Part นี้เราจะ:

- ตัดคำภาษาไทยด้วย PyThaiNLP
- วิเคราะห์ความรู้สึก (Sentiment) ของ SOS content
- แยกหมวดหมู่ emergency: น้ำท่วม ไฟไหม้ อุบัติเหตุ
- ดึงชื่อสถานที่จากข้อความ SOS (Named Entity Recognition)
- ใช้ Claude AI API สำหรับ complex content moderation

---

## 📖 ทฤษฎีและแนวคิด

### 1. ความท้าทายของภาษาไทย

```
ภาษาอังกฤษ: "flood in my area"
ตัดคำ:       "flood" | "in" | "my" | "area"

ภาษาไทย:     "น้ำท่วมที่บ้าน"
ปัญหา:       ไม่มี space! ต้องรู้จักคำก่อนถึงตัดได้
ตัดคำที่ถูก: "น้ำ" | "ท่วม" | "ที่" | "บ้าน"
หรือ:        "น้ำท่วม" | "ที่" | "บ้าน"  (better!)

ภาษาไทยยังมี:
- Polysemy: คำเดียวกันความหมายต่างกัน ("ขัน" = bucket หรือ funny?)
- Abbreviations: "น.ท." = น้ำท่วม หรือ นาวาโท
- Informal language: "ขอความช่วยเหลือด้วยย" (คำลงท้ายที่ยืด)
```

### 2. NLP Use Cases สำหรับ chuaikan.com

```
SOS Text: "น้ำท่วมสูงมากแล้วถนนพหลโยธิน กม.50 ใกล้วัดมีชัยภิกขุ 
          ขอความช่วยเหลือด้วยครับ มีผู้สูงอายุติดอยู่ 3 คน"

NLP Tasks:
1. Emergency Category: floods ✓
2. Location NER: ถนนพหลโยธิน กม.50, วัดมีชัยภิกขุ
3. Severity: High (ผู้สูงอายุ, ติดอยู่)
4. Keyword Extraction: น้ำท่วม, ถนน, ขอความช่วยเหลือ, ผู้สูงอายุ
5. Sentiment: Urgent/Distress
```

---

## ⚙️ Environment Setup

```bash
# ติดตั้ง PyThaiNLP
pip install pythainlp[full]

# ดาวน์โหลด Thai language models
python -c "from pythainlp.corpus import download; download('best')"
python -c "from pythainlp.corpus import download; download('newmm-wordseg-20210101')"

# ติดตั้ง transformers สำหรับ BERT-based Thai models
pip install transformers torch sentencepiece

# ติดตั้ง aiforthai packages
pip install wangchanberta  # Thai BERT by NECTEC
```

---

## 🛠️ Step-by-Step Implementation

### Step 1021: Thai Word Segmentation

```python
# thai-tokenizer.py
from pythainlp.tokenize import word_tokenize, sent_tokenize
from pythainlp.corpus.common import thai_stopwords
import re

class ThaiTextProcessor:
    def __init__(self):
        self.stopwords = set(thai_stopwords())
        # เพิ่ม stopwords เฉพาะ chuaikan
        self.stopwords.update(['ครับ', 'ค่ะ', 'นะ', 'นะครับ', 'ด้วย', 'หน่อย'])
    
    def tokenize(self, text, engine='newmm'):
        """
        ตัดคำภาษาไทย
        engine options:
        - 'newmm': fastest, most accurate (แนะนำ)
        - 'attacut': ดีสำหรับ informal text
        - 'deepcut': deep learning based (ช้าแต่ accurate กว่า)
        """
        # clean text ก่อน
        text = self._preprocess(text)
        
        tokens = word_tokenize(text, engine=engine, keep_whitespace=False)
        
        # Remove stopwords และ tokens สั้นเกินไป
        tokens = [t for t in tokens if t not in self.stopwords and len(t) > 1]
        
        return tokens
    
    def _preprocess(self, text):
        """Clean text ก่อน processing"""
        # Remove URLs
        text = re.sub(r'http\S+', '', text)
        
        # Remove phone numbers
        text = re.sub(r'0[0-9]{8,9}', '[PHONE]', text)
        
        # Normalize repeated characters (ด้วยยยย → ด้วย)
        text = re.sub(r'(.)\1{2,}', r'\1\1', text)
        
        # Remove special chars ยกเว้น Thai, English, digits
        text = re.sub(r'[^฀-๿a-zA-Z0-9\s.,?!]', '', text)
        
        return text.strip()
    
    def extract_keywords(self, text, top_k=10):
        """ดึง keywords สำคัญจากข้อความ"""
        from pythainlp.util.trie import Trie
        
        tokens = self.tokenize(text)
        
        # Word frequency (simple approach)
        from collections import Counter
        freq = Counter(tokens)
        
        # TF-IDF แบบง่าย (ใน production ควรใช้ sklearn)
        keywords = [(word, count) for word, count in freq.most_common(top_k)]
        
        return keywords

# ทดสอบ
processor = ThaiTextProcessor()

sos_text = "น้ำท่วมสูงมากแล้วถนนพหลโยธิน กม.50 ขอความช่วยเหลือด้วยครับ มีผู้สูงอายุติดอยู่ 3 คน"

tokens = processor.tokenize(sos_text)
print("Tokens:", tokens)
# ['น้ำท่วม', 'สูง', 'มาก', 'แล้ว', 'ถนน', 'พหลโยธิน', 'กม', '50', 'ขอความช่วยเหลือ', 'ผู้สูงอายุ', 'ติด', '3', 'คน']

keywords = processor.extract_keywords(sos_text)
print("Keywords:", keywords)
```

### Step 1022: Thai Sentiment Analysis สำหรับ SOS

```python
# thai-sentiment.py
# วิเคราะห์ความเร่งด่วนของ SOS content

from transformers import pipeline, AutoTokenizer, AutoModelForSequenceClassification
import torch

class ThaiSentimentAnalyzer:
    def __init__(self):
        # ใช้ WangchanBERTa (Thai BERT จาก NECTEC/PyThaiNLP)
        model_name = "airesearch/wangchanberta-base-att-spm-uncased"
        
        self.tokenizer = AutoTokenizer.from_pretrained(model_name)
        self.model = AutoModelForSequenceClassification.from_pretrained(
            model_name,
            num_labels=3  # ฉุกเฉิน / เร่งด่วน / ทั่วไป
        )
        
        # โหลด fine-tuned weights สำหรับ SOS classification
        # (ต้อง fine-tune ก่อนด้วย SOS data)
        checkpoint = torch.load('models/sos-urgency-classifier.pt')
        self.model.load_state_dict(checkpoint)
        self.model.eval()
    
    def classify_urgency(self, text):
        """จำแนกระดับความเร่งด่วนของ SOS"""
        inputs = self.tokenizer(
            text,
            return_tensors='pt',
            max_length=512,
            truncation=True,
            padding=True
        )
        
        with torch.no_grad():
            outputs = self.model(**inputs)
            logits = outputs.logits
            probs = torch.softmax(logits, dim=1).squeeze()
        
        labels = ['general', 'urgent', 'critical']
        predicted_label = labels[torch.argmax(probs).item()]
        confidence = probs[torch.argmax(probs)].item()
        
        return {
            'urgency': predicted_label,
            'confidence': confidence,
            'probabilities': {
                label: float(prob)
                for label, prob in zip(labels, probs)
            }
        }

# Rule-based urgency keywords (fallback)
CRITICAL_KEYWORDS = [
    'ขอความช่วยเหลือ', 'ช่วยด้วย', 'ฉุกเฉิน', 'เร่งด่วน',
    'ติดอยู่', 'ออกไม่ได้', 'อันตราย', 'บาดเจ็บ', 'หัวใจ',
    'หมดสติ', 'หายใจไม่ออก', 'ไฟไหม้', 'คนจมน้ำ'
]

URGENT_KEYWORDS = [
    'น้ำท่วม', 'น้ำสูง', 'ต้องการความช่วยเหลือ', 'รถติด',
    'ถนนปิด', 'ไฟฟ้าดับ', 'แก๊สรั่ว'
]

def rule_based_urgency(text):
    """Rule-based fallback ถ้า ML model ไม่พร้อม"""
    text_lower = text.lower()
    
    for keyword in CRITICAL_KEYWORDS:
        if keyword in text_lower:
            return 'critical', 0.85
    
    for keyword in URGENT_KEYWORDS:
        if keyword in text_lower:
            return 'urgent', 0.80
    
    return 'general', 0.90
```

### Step 1023: Thai Named Entity Recognition

```python
# thai-ner.py
# ดึงชื่อสถานที่จาก SOS text

from pythainlp import word_tokenize
from pythainlp.tag import pos_tag
import re

class ThaiLocationExtractor:
    """ดึงชื่อสถานที่จากข้อความ SOS"""
    
    # Thai location indicators
    LOCATION_PREFIXES = [
        'ถนน', 'ซอย', 'ตำบล', 'อำเภอ', 'จังหวัด', 'เขต', 'แขวง',
        'หมู่บ้าน', 'หมู่', 'คลอง', 'บึง', 'สะพาน', 'วัด', 'โรงเรียน',
        'โรงพยาบาล', 'ตลาด', 'กม.', 'กิโลเมตรที่'
    ]
    
    PROVINCE_NAMES = [
        'กรุงเทพ', 'เชียงใหม่', 'เชียงราย', 'ขอนแก่น', 'อุดรธานี',
        'นครราชสีมา', 'สุราษฎร์ธานี', 'ภูเก็ต', 'สงขลา', 'นนทบุรี',
        'ปทุมธานี', 'สมุทรปราการ', 'นครปฐม', 'ราชบุรี', 'พระนครศรีอยุธยา',
        # ... (76 จังหวัด)
    ]
    
    def extract_locations(self, text):
        locations = []
        
        # Method 1: Rule-based — หา location prefixes
        for prefix in self.LOCATION_PREFIXES:
            pattern = f'{prefix}[ก-๙a-zA-Z0-9\\s]+(?=[\\s,.]|$)'
            matches = re.findall(pattern, text)
            locations.extend(matches)
        
        # Method 2: Province names
        for province in self.PROVINCE_NAMES:
            if province in text:
                locations.append(province)
        
        # Method 3: POS tagging — ดึง NE (Named Entities)
        tokens_with_pos = pos_tag(word_tokenize(text), corpus='orchid_ud')
        
        # ใน orchid tagset: NNP = proper noun (ชื่อสถานที่ มักจะเป็น NNP)
        current_ne = []
        for token, pos in tokens_with_pos:
            if pos in ['NNP', 'NE']:
                current_ne.append(token)
            else:
                if current_ne:
                    locations.append(''.join(current_ne))
                    current_ne = []
        
        # Remove duplicates และ clean
        locations = list(set([l.strip() for l in locations if len(l.strip()) > 2]))
        
        return locations
    
    def geocode_location(self, location_text):
        """แปลงชื่อสถานที่เป็น coordinates (Google Maps API)"""
        import googlemaps
        
        gmaps = googlemaps.Client(key=os.environ['GOOGLE_MAPS_API_KEY'])
        
        # เพิ่ม "ประเทศไทย" เพื่อเพิ่ม accuracy
        query = f"{location_text} ประเทศไทย"
        
        result = gmaps.geocode(query)
        
        if result:
            location = result[0]['geometry']['location']
            return {
                'lat': location['lat'],
                'lng': location['lng'],
                'formatted_address': result[0]['formatted_address'],
                'place_id': result[0]['place_id'],
            }
        
        return None

# ทดสอบ
extractor = ThaiLocationExtractor()

sos_text = "น้ำท่วมสูงมากที่ถนนพหลโยธิน กม.50 ใกล้วัดมีชัยภิกขุ อำเภอเมือง จังหวัดเชียงใหม่"

locations = extractor.extract_locations(sos_text)
print("Extracted locations:", locations)
# ['ถนนพหลโยธิน กม.50', 'วัดมีชัยภิกขุ', 'อำเภอเมือง', 'จังหวัดเชียงใหม่']

# Geocode ทั้งหมด
for loc in locations:
    coords = extractor.geocode_location(loc)
    if coords:
        print(f"{loc}: {coords['lat']}, {coords['lng']}")
```

### Step 1024: Emergency Category Classification

```python
# emergency-classifier.py
# จำแนกประเภทภัยพิบัติ

from pythainlp.tokenize import word_tokenize

EMERGENCY_CATEGORIES = {
    'flood': {
        'keywords': ['น้ำท่วม', 'น้ำสูง', 'น้ำป่า', 'น้ำเอ่อ', 'น้ำขัง', 'ดินโคลน'],
        'severity_modifiers': {
            'high': ['สูงมาก', 'ท่วมหลังคา', 'ท่วมชั้น2', 'ไม่สามารถออก'],
            'medium': ['ท่วมเอว', 'ท่วมอก', 'ท่วมหัวเข่า'],
            'low': ['ท่วมเล็กน้อย', 'น้ำขังนิดหน่อย'],
        }
    },
    'fire': {
        'keywords': ['ไฟไหม้', 'เพลิงไหม้', 'ไฟลุก', 'ควันไฟ', 'เพลิง'],
        'severity_modifiers': {
            'high': ['ควบคุมไม่ได้', 'ลุกลาม', 'อาคารไหม้'],
            'medium': ['ไฟลุก', 'ต้องการดับเพลิง'],
            'low': ['ควันไฟ', 'ไฟเล็กน้อย'],
        }
    },
    'accident': {
        'keywords': ['อุบัติเหตุ', 'รถชน', 'รถคว่ำ', 'ชนกัน', 'สิ่งกีดขวาง'],
        'severity_modifiers': {
            'high': ['บาดเจ็บสาหัส', 'หมดสติ', 'เลือดออกมาก'],
            'medium': ['บาดเจ็บ', 'ต้องการรถพยาบาล'],
            'low': ['รถเสีย', 'ถนนปิด'],
        }
    },
    'medical': {
        'keywords': ['เจ็บป่วย', 'หัวใจ', 'หายใจไม่ออก', 'หมดสติ', 'ล้มหมดสติ'],
        'severity_modifiers': {
            'high': ['หัวใจวาย', 'หมดสติ', 'ชัก'],
            'medium': ['เจ็บปวดมาก', 'ต้องการแพทย์'],
            'low': ['ไม่สบาย', 'ต้องการความช่วยเหลือ'],
        }
    },
}

def classify_emergency(text):
    """จำแนกประเภทและระดับความรุนแรงของ emergency"""
    
    tokens = word_tokenize(text, engine='newmm')
    text_joined = ' '.join(tokens)
    
    detected_categories = []
    
    for category, data in EMERGENCY_CATEGORIES.items():
        # ตรวจ keywords
        keyword_matches = [k for k in data['keywords'] if k in text_joined]
        
        if keyword_matches:
            # วิเคราะห์ severity
            severity = 'medium'
            for level, modifiers in data['severity_modifiers'].items():
                if any(m in text_joined for m in modifiers):
                    severity = level
                    break
            
            detected_categories.append({
                'category': category,
                'severity': severity,
                'matched_keywords': keyword_matches,
            })
    
    if not detected_categories:
        return {
            'category': 'unknown',
            'severity': 'unknown',
            'requires_human_review': True
        }
    
    # Return primary category (ที่มี keyword matches มากที่สุด)
    primary = max(detected_categories, key=lambda x: len(x['matched_keywords']))
    
    return {
        'primary_category': primary['category'],
        'severity': primary['severity'],
        'all_categories': detected_categories,
        'requires_human_review': primary['severity'] == 'high',
    }
```

### Step 1025: Claude AI สำหรับ Content Moderation

```python
# claude-moderation.py
# ใช้ Claude AI สำหรับ complex cases ที่ rule-based ไม่เพียงพอ

import anthropic
import json

client = anthropic.Anthropic(api_key=os.environ['ANTHROPIC_API_KEY'])

def moderate_sos_content(text, image_analysis=None):
    """
    ใช้ Claude สำหรับ complex moderation decisions
    Call นี้มี cost ดังนั้นใช้เฉพาะกรณีที่ rule-based ไม่แน่ใจ
    """
    
    context_parts = [f"SOS Text: {text}"]
    if image_analysis:
        context_parts.append(f"Image Analysis: {json.dumps(image_analysis)}")
    
    prompt = f"""You are a content moderator for chuaikan.com, a disaster response platform in Thailand.

{chr(10).join(context_parts)}

Please analyze this SOS post and respond with a JSON object:
{{
  "is_genuine_emergency": true/false,
  "emergency_type": "flood/fire/accident/medical/other/unknown",
  "severity": "critical/high/medium/low",
  "is_inappropriate": true/false,
  "requires_immediate_response": true/false,
  "location_mentioned": "location or null",
  "reason": "brief explanation in Thai",
  "confidence": 0.0-1.0
}}

Consider:
- Thai language nuances and informal expressions
- Context of Thailand's disaster response
- Whether this seems like a genuine call for help vs spam/test
- Severity based on described situation"""

    message = client.messages.create(
        model="claude-3-5-haiku-20241022",  # ใช้ Haiku เพราะเร็วและถูกกว่า
        max_tokens=500,
        messages=[
            {"role": "user", "content": prompt}
        ]
    )
    
    response_text = message.content[0].text
    
    try:
        # Parse JSON response
        result = json.loads(response_text)
        return result
    except json.JSONDecodeError:
        # ถ้า parse ไม่ได้ → ส่ง human review
        return {
            'requires_human_review': True,
            'parse_error': True,
            'raw_response': response_text
        }

# Caching สำหรับ similar content (ประหยัด API cost)
from functools import lru_cache
import hashlib

@lru_cache(maxsize=1000)
def cached_moderate(text_hash, text):
    return moderate_sos_content(text)

def moderate_with_cache(text, image_analysis=None):
    # Create hash สำหรับ cache key
    text_hash = hashlib.md5(text.encode()).hexdigest()
    
    if image_analysis:
        # มี image → ต้อง analyze ใหม่ (ไม่ cache)
        return moderate_sos_content(text, image_analysis)
    
    return cached_moderate(text_hash, text)
```

---

## 🔧 Configuration Files

### Kubernetes Deployment สำหรับ NLP Service

```yaml
# nlp-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nlp-service
  namespace: production
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: nlp-service
        image: chuaikan/nlp-service:latest
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "2000m"
            memory: "4Gi"  # NLP models ใช้ memory เยอะ
        env:
        - name: ANTHROPIC_API_KEY
          valueFrom:
            secretKeyRef:
              name: ai-credentials
              key: anthropic-api-key
        - name: GOOGLE_MAPS_API_KEY
          valueFrom:
            secretKeyRef:
              name: maps-credentials
              key: google-maps-key
        - name: MODEL_CACHE_DIR
          value: "/models"
        volumeMounts:
        - name: model-cache
          mountPath: /models
      volumes:
      - name: model-cache
        persistentVolumeClaim:
          claimName: nlp-model-cache
```

---

## 🧪 Testing

### NLP Pipeline Test

```python
# test-thai-nlp.py
import pytest

class TestThaiNLP:
    def test_word_tokenization(self):
        processor = ThaiTextProcessor()
        
        result = processor.tokenize("น้ำท่วมที่บ้านฉัน")
        
        # ต้องตัดได้ถูกต้อง
        assert 'น้ำท่วม' in result or ('น้ำ' in result and 'ท่วม' in result)
        assert 'บ้าน' in result
    
    def test_emergency_classification_flood(self):
        result = classify_emergency("น้ำท่วมสูงมากที่บ้านผม ขอความช่วยเหลือ")
        
        assert result['primary_category'] == 'flood'
        assert result['severity'] in ['high', 'critical']
    
    def test_location_extraction(self):
        extractor = ThaiLocationExtractor()
        text = "เกิดอุบัติเหตุที่ถนนพหลโยธิน กม.50 จังหวัดเชียงใหม่"
        
        locations = extractor.extract_locations(text)
        
        assert any('พหลโยธิน' in loc for loc in locations)
        assert any('เชียงใหม่' in loc for loc in locations)
    
    def test_stopword_removal(self):
        processor = ThaiTextProcessor()
        
        text = "ขอความช่วยเหลือด้วยนะครับ"
        tokens = processor.tokenize(text)
        
        # stopwords ต้องถูกลบออก
        assert 'ครับ' not in tokens
        assert 'นะ' not in tokens
        
        # keyword สำคัญต้องอยู่
        assert 'ขอความช่วยเหลือ' in tokens or 'ความช่วยเหลือ' in tokens
```

---

## ❌ Common Errors & Solutions

### Error 1: Thai Tokenization ผิดพลาด

```python
# ปัญหา: "น้ำท่วม" ถูกตัดเป็น "น้ำ" + "ท่วม" แทนที่จะเป็น "น้ำท่วม"

# แก้: เพิ่มคำ custom ใน dictionary
from pythainlp.corpus import add_to_custom_dict

# เพิ่ม SOS-specific vocabulary
emergency_terms = [
    'น้ำท่วม', 'ไฟไหม้', 'ดินโคลน', 'น้ำป่า', 'แผ่นดินไหว',
    'ขอความช่วยเหลือ', 'ติดอยู่', 'ออกไม่ได้', 'ต้องการความช่วยเหลือ',
]

for term in emergency_terms:
    add_to_custom_dict(term)
```

### Error 2: Claude API Rate Limit

```python
# แก้: ใช้ exponential backoff + queue

import time
import anthropic

def call_claude_with_retry(prompt, max_retries=3):
    for attempt in range(max_retries):
        try:
            return client.messages.create(...)
        except anthropic.RateLimitError:
            if attempt < max_retries - 1:
                wait_time = (2 ** attempt) + random.random()
                time.sleep(wait_time)
            else:
                raise
        except anthropic.APIStatusError as e:
            if e.status_code == 529:  # Overloaded
                time.sleep(5)
            else:
                raise
```

---

## ✅ Checklist

- [ ] PyThaiNLP ติดตั้งและ models download แล้ว
- [ ] Emergency category classifier ทำงาน (accuracy > 90%)
- [ ] Named Entity Recognition ดึงสถานที่ได้ (Recall > 80%)
- [ ] Sentiment/Urgency classifier แม่นยำ
- [ ] Geocoding สำหรับสถานที่ที่ดึงได้
- [ ] Claude AI integration สำหรับ complex cases
- [ ] Cost monitoring สำหรับ Claude API calls
- [ ] Rate limiting และ retry logic
- [ ] Custom dictionary สำหรับ emergency terms
- [ ] Thai NLP unit tests ผ่าน

---

## 🔗 References

- [PyThaiNLP Documentation](https://pythainlp.github.io/docs/)
- [WangchanBERTa — Thai BERT](https://huggingface.co/airesearch/wangchanberta-base-att-spm-uncased)
- [Claude API Documentation](https://docs.anthropic.com/)
- [Thai NLP Resources](https://github.com/PyThaiNLP/pythainlp)
- [Google Maps Geocoding API](https://developers.google.com/maps/documentation/geocoding)

---

*Part 103 | Road to 1,000,000 Users/Day | chuaikan.com*
