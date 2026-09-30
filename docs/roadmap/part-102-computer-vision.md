# Part 102: Computer Vision สำหรับ Flood Detection

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** World Class
> **Steps:** 1011-1020
> **เวลาโดยประมาณ:** 5 ชั่วโมง
> **Prerequisites:** Part 101 (AI/ML Integration)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

Computer Vision ช่วย chuaikan.com วิเคราะห์ภาพน้ำท่วมโดยอัตโนมัติ ลดภาระ manual review และตรวจจับ SOS ระดับรุนแรงได้เร็วขึ้น ใน Part นี้เราจะ:

- ตรวจจับน้ำท่วมจากภาพด้วย YOLOv8
- Image Classification: flood/non-flood ด้วย ResNet
- ใช้ AWS Rekognition สำหรับ managed CV
- สร้าง Processing Pipeline: upload → analyze → auto-tag SOS
- Deploy Model บน Lambda สำหรับ lightweight inference

---

## 📖 ทฤษฎีและแนวคิด

### 1. Computer Vision Use Cases สำหรับ chuaikan.com

```
SOS Image Analysis Pipeline:

User uploads photo → [CV Pipeline] → Auto-annotated SOS

CV Tasks:
1. Flood Detection:     ภาพนี้มีน้ำท่วมหรือเปล่า?
2. Severity Estimation: ระดับน้ำสูงแค่ไหน? (knee/waist/chest/roof)
3. Object Detection:    มีคนติดอยู่ไหม? รถหรือเปล่า?
4. Text Recognition:    มีป้ายระบุสถานที่ไหม? (OCR)
5. Moderation:          ภาพ inappropriate หรือเปล่า?
```

### 2. YOLOv8 vs ResNet

```
YOLOv8 (You Only Look Once):
- Object Detection และ Segmentation
- เร็วมาก: 45ms/image (GPU)
- ดีสำหรับ: หา "น้ำท่วม" ใน specific parts ของภาพ
- ดูว่า: ระดับน้ำถึงไหน, มีคนในภาพไหม

ResNet (Residual Network):
- Image Classification
- เหมาะสำหรับ: flood/non-flood binary classification
- Accuracy สูงกว่าสำหรับ whole-image classification
- ช้ากว่าเล็กน้อย แต่ accurate กว่า
```

### 3. Processing Pipeline

```
User Upload Flow:

1. User uploads image → S3 (presigned URL)
2. S3 triggers Lambda (or SQS → Lambda)
3. Lambda: Quick moderation check (AWS Rekognition)
4. Lambda: Flood detection (custom model หรือ Rekognition Custom Labels)
5. Result → SQS → Notification Service
6. Update SOS severity automatically
7. Flag for human review ถ้า confidence < 80%
```

---

## 🛠️ Step-by-Step Implementation

### Step 1011: AWS Rekognition สำหรับ Quick Moderation

```python
# moderation-lambda.py
# Lambda ที่รัน เมื่อ image upload ไป S3

import boto3
import json
import os
from dataclasses import dataclass

rekognition = boto3.client('rekognition', region_name='ap-southeast-1')
sqs = boto3.client('sqs', region_name='ap-southeast-1')

@dataclass
class ModerationResult:
    is_safe: bool
    confidence: float
    detected_labels: list
    moderation_labels: list

def lambda_handler(event, context):
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        key = record['s3']['object']['key']
        
        # 1. Content Moderation
        moderation = check_moderation(bucket, key)
        
        if not moderation.is_safe:
            # ระงับ image ที่ไม่เหมาะสม
            handle_inappropriate_content(bucket, key, moderation)
            return
        
        # 2. ส่งไป flood detection pipeline
        send_to_flood_detection(bucket, key)

def check_moderation(bucket, key):
    """ตรวจ inappropriate content ด้วย Rekognition"""
    response = rekognition.detect_moderation_labels(
        Image={
            'S3Object': {
                'Bucket': bucket,
                'Name': key
            }
        },
        MinConfidence=70  # ≥70% confidence ถึง flag
    )
    
    dangerous_labels = [
        'Explicit Nudity', 'Violence', 'Visually Disturbing',
        'Hate Symbols', 'Drugs & Tobacco'
    ]
    
    moderation_labels = response.get('ModerationLabels', [])
    flagged = [l for l in moderation_labels if l['Name'] in dangerous_labels]
    
    return ModerationResult(
        is_safe=len(flagged) == 0,
        confidence=max([l['Confidence'] for l in moderation_labels], default=0),
        detected_labels=[],
        moderation_labels=moderation_labels,
    )

def send_to_flood_detection(bucket, key):
    """ส่ง image ไป analyze flood"""
    sqs.send_message(
        QueueUrl=os.environ['FLOOD_DETECTION_QUEUE_URL'],
        MessageBody=json.dumps({
            'bucket': bucket,
            'key': key,
            'timestamp': context.aws_request_id,
        })
    )
```

### Step 1012: Flood Detection Model ด้วย YOLOv8

```python
# train-flood-detector.py
# Train YOLOv8 model สำหรับ flood detection

from ultralytics import YOLO
import yaml
import os

# 1. Dataset Structure
# data/
#   images/
#     train/  (800 images)
#     val/    (100 images)
#     test/   (100 images)
#   labels/
#     train/  (YOLO format: class x_center y_center width height)
#     val/
#     test/

# dataset.yaml
dataset_config = {
    'path': '/data/flood-dataset',
    'train': 'images/train',
    'val': 'images/val',
    'test': 'images/test',
    
    'nc': 4,  # number of classes
    'names': {
        0: 'water_surface',     # พื้นที่น้ำท่วม
        1: 'person_in_water',   # คนในน้ำ
        2: 'vehicle',           # รถในน้ำ
        3: 'water_level_marker', # เสาหรือสิ่งอ้างอิงระดับน้ำ
    }
}

with open('dataset.yaml', 'w') as f:
    yaml.dump(dataset_config, f)

# 2. Fine-tune YOLOv8
model = YOLO('yolov8m.pt')  # Medium size — balance ระหว่าง speed และ accuracy

results = model.train(
    data='dataset.yaml',
    epochs=100,
    imgsz=640,
    batch=16,
    device='cuda:0',  # หรือ 'cpu' สำหรับ test
    project='flood-detection',
    name='yolov8m-flood-v1',
    
    # Augmentation สำหรับ flood images
    flipud=0.5,      # flip vertical (น้ำท่วมมาจากล่าง)
    degrees=10,      # slight rotation
    translate=0.1,
    scale=0.5,
    hsv_h=0.015,     # color variation
    hsv_s=0.7,
    hsv_v=0.4,
)

# 3. Evaluate
metrics = model.val()
print(f"mAP50: {metrics.box.map50:.3f}")
print(f"mAP50-95: {metrics.box.map:.3f}")

# 4. Export สำหรับ deployment
model.export(format='onnx', dynamic=True)  # ONNX สำหรับ Lambda
```

```python
# flood-detector-lambda.py
# Lambda function สำหรับ run flood detection

import boto3
import json
import numpy as np
import onnxruntime as ort
from PIL import Image
import io
import os

# Load model เมื่อ Lambda init (cold start)
MODEL_PATH = '/opt/ml/models/flood-detector.onnx'
session = ort.InferenceSession(MODEL_PATH, providers=['CPUExecutionProvider'])

s3 = boto3.client('s3')
db = boto3.client('dynamodb')

def preprocess_image(image_bytes):
    """Preprocess image สำหรับ YOLO inference"""
    img = Image.open(io.BytesIO(image_bytes)).convert('RGB')
    img = img.resize((640, 640))
    
    img_array = np.array(img, dtype=np.float32)
    img_array = img_array / 255.0  # Normalize 0-1
    img_array = np.transpose(img_array, (2, 0, 1))  # HWC → CHW
    img_array = np.expand_dims(img_array, axis=0)   # Add batch dim
    
    return img_array

def analyze_flood_severity(detections, image_size):
    """วิเคราะห์ระดับความรุนแรงจาก detections"""
    
    water_area = 0
    has_person_in_water = False
    has_vehicle = False
    
    for det in detections:
        class_id, confidence, x, y, w, h = det
        area = w * h * image_size[0] * image_size[1]
        
        if class_id == 0:  # water_surface
            water_area += area
        elif class_id == 1:  # person_in_water
            has_person_in_water = True
        elif class_id == 2:  # vehicle
            has_vehicle = True
    
    # คำนวณ % พื้นที่น้ำท่วม
    water_ratio = water_area / (image_size[0] * image_size[1])
    
    if has_person_in_water:
        return 'critical', 0.95  # คนติดในน้ำ = critical เสมอ
    elif water_ratio > 0.6:
        return 'severe', water_ratio
    elif water_ratio > 0.3:
        return 'moderate', water_ratio
    elif water_ratio > 0.1:
        return 'minor', water_ratio
    else:
        return 'none', 0.95

def lambda_handler(event, context):
    for record in event['Records']:
        body = json.loads(record['body'])
        bucket = body['bucket']
        key = body['key']
        
        # Download image จาก S3
        response = s3.get_object(Bucket=bucket, Key=key)
        image_bytes = response['Body'].read()
        
        # Preprocess
        input_tensor = preprocess_image(image_bytes)
        
        # Inference
        outputs = session.run(None, {'images': input_tensor})
        
        # Parse YOLO output
        detections = parse_yolo_output(outputs[0], confidence_threshold=0.5)
        
        # วิเคราะห์ severity
        severity, confidence = analyze_flood_severity(detections, (640, 640))
        
        # อัพเดท SOS record
        sos_id = key.split('/')[1]  # จาก path: sos/{sos_id}/image.jpg
        
        update_sos_severity(sos_id, severity, confidence, detections)
        
        # ถ้า confidence ต่ำ → ส่งไป human review
        if confidence < 0.8:
            queue_for_human_review(sos_id, severity, confidence)
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'sos_id': sos_id,
                'severity': severity,
                'confidence': confidence,
                'detections_count': len(detections),
            })
        }

def parse_yolo_output(output, confidence_threshold=0.5, iou_threshold=0.45):
    """Parse YOLO v8 output tensor"""
    # YOLOv8 output shape: [1, 8, 8400] (4 bbox + 4 class scores × 8400 anchors)
    output = output[0]  # Remove batch dim
    
    # Transpose: [8, 8400] → [8400, 8]
    output = output.T
    
    # Filter by confidence
    # score = max class confidence
    class_scores = output[:, 4:]  # [8400, 4]
    max_class_score = np.max(class_scores, axis=1)
    class_id = np.argmax(class_scores, axis=1)
    
    mask = max_class_score >= confidence_threshold
    filtered = output[mask]
    class_ids = class_id[mask]
    scores = max_class_score[mask]
    
    # Get bbox coordinates (center format)
    boxes_cx = filtered[:, 0]
    boxes_cy = filtered[:, 1]
    boxes_w = filtered[:, 2]
    boxes_h = filtered[:, 3]
    
    detections = list(zip(class_ids, scores, boxes_cx, boxes_cy, boxes_w, boxes_h))
    return detections
```

### Step 1013: Image Classification ด้วย ResNet

```python
# train-flood-classifier.py
# Binary classification: flood vs non-flood

import torch
import torchvision.transforms as transforms
from torchvision import models
from torch.utils.data import DataLoader, Dataset
from PIL import Image
import os

# Custom Dataset
class FloodDataset(Dataset):
    def __init__(self, data_dir, transform=None):
        self.data_dir = data_dir
        self.transform = transform
        self.samples = []
        
        # Load flood images (label=1)
        flood_dir = os.path.join(data_dir, 'flood')
        for img_file in os.listdir(flood_dir):
            self.samples.append((os.path.join(flood_dir, img_file), 1))
        
        # Load non-flood images (label=0)
        normal_dir = os.path.join(data_dir, 'normal')
        for img_file in os.listdir(normal_dir):
            self.samples.append((os.path.join(normal_dir, img_file), 0))
    
    def __len__(self):
        return len(self.samples)
    
    def __getitem__(self, idx):
        img_path, label = self.samples[idx]
        image = Image.open(img_path).convert('RGB')
        
        if self.transform:
            image = self.transform(image)
        
        return image, label

# Data transforms
train_transform = transforms.Compose([
    transforms.RandomResizedCrop(224),
    transforms.RandomHorizontalFlip(),
    transforms.ColorJitter(brightness=0.2, contrast=0.2),
    transforms.ToTensor(),
    transforms.Normalize([0.485, 0.456, 0.406], [0.229, 0.224, 0.225]),
])

val_transform = transforms.Compose([
    transforms.Resize(256),
    transforms.CenterCrop(224),
    transforms.ToTensor(),
    transforms.Normalize([0.485, 0.456, 0.406], [0.229, 0.224, 0.225]),
])

# Load pre-trained ResNet50 และ Fine-tune
model = models.resnet50(pretrained=True)

# Freeze early layers
for param in list(model.parameters())[:-20]:
    param.requires_grad = False

# Replace final layer
model.fc = torch.nn.Sequential(
    torch.nn.Dropout(0.5),
    torch.nn.Linear(2048, 1),
    torch.nn.Sigmoid()
)

# Train
criterion = torch.nn.BCELoss()
optimizer = torch.optim.Adam(model.parameters(), lr=0.001)
scheduler = torch.optim.lr_scheduler.StepLR(optimizer, step_size=7, gamma=0.1)

device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
model = model.to(device)

for epoch in range(25):
    # Training loop...
    pass

# Export ไป ONNX
dummy_input = torch.randn(1, 3, 224, 224).to(device)
torch.onnx.export(
    model, dummy_input,
    'flood-classifier.onnx',
    opset_version=12,
    input_names=['image'],
    output_names=['flood_probability'],
    dynamic_axes={'image': {0: 'batch_size'}}
)
```

---

## 🔧 Configuration Files

### Lambda Deployment สำหรับ CV Model

```yaml
# lambda-cv-config.yaml (AWS SAM)
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Timeout: 30
    MemorySize: 3008  # Max memory สำหรับ ONNX inference
    Runtime: python3.11
    Layers:
      - !Sub arn:aws:lambda:ap-southeast-1:${AWS::AccountId}:layer:onnxruntime:5

Resources:
  FloodDetectionFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: flood-detection/
      Handler: lambda_handler.handler
      Events:
        SQSEvent:
          Type: SQS
          Properties:
            Queue: !GetAtt FloodDetectionQueue.Arn
            BatchSize: 5
            FunctionResponseTypes:
              - ReportBatchItemFailures
      Environment:
        Variables:
          MODEL_PATH: /opt/ml/models/flood-detector.onnx
          CONFIDENCE_THRESHOLD: '0.7'
          HUMAN_REVIEW_QUEUE: !Ref HumanReviewQueue
      Policies:
        - S3ReadPolicy:
            BucketName: chuaikan-media
        - SQSSendMessagePolicy:
            QueueName: !GetAtt HumanReviewQueue.QueueName
```

---

## 🧪 Testing

### Model Accuracy Testing

```python
# test-flood-detection.py
import pytest

def test_flood_detection_accuracy():
    """Model accuracy ต้องเกิน threshold"""
    
    # Load test dataset
    test_images = load_test_dataset('test-data/flood-detection/')
    
    true_positives = 0
    false_positives = 0
    false_negatives = 0
    true_negatives = 0
    
    for image_path, true_label in test_images:
        prediction = run_flood_detection(image_path)
        
        if true_label == 'flood' and prediction == 'flood':
            true_positives += 1
        elif true_label == 'flood' and prediction != 'flood':
            false_negatives += 1
        elif true_label != 'flood' and prediction == 'flood':
            false_positives += 1
        else:
            true_negatives += 1
    
    precision = true_positives / (true_positives + false_positives)
    recall = true_positives / (true_positives + false_negatives)
    f1 = 2 * (precision * recall) / (precision + recall)
    
    print(f"Precision: {precision:.3f}")
    print(f"Recall: {recall:.3f}")
    print(f"F1 Score: {f1:.3f}")
    
    # SOS context: Recall สำคัญกว่า Precision
    # (ดีกว่าตรวจเจอเกิน ดีกว่าตรวจไม่เจอ)
    assert recall >= 0.90, f"Recall too low: {recall:.3f} (min: 0.90)"
    assert precision >= 0.80, f"Precision too low: {precision:.3f} (min: 0.80)"
    assert f1 >= 0.85, f"F1 too low: {f1:.3f} (min: 0.85)"
```

---

## ❌ Common Errors & Solutions

### Error 1: Lambda Cold Start ช้า

```python
# ปัญหา: ONNX model load ใช้เวลา 5-10 วินาที = bad cold start

# แก้ 1: Provisioned Concurrency
aws lambda put-provisioned-concurrency-config \
  --function-name flood-detection \
  --qualifier production \
  --provisioned-concurrent-executions 5  # Keep 5 instances warm

# แก้ 2: AWS Lambda Snapstart (Java) หรือ
#         Initialize model outside handler (Python)
session = None  # Global

def load_model():
    global session
    if session is None:
        session = ort.InferenceSession(MODEL_PATH)
    return session

def lambda_handler(event, context):
    model = load_model()  # ใช้ cached session
    # ...
```

### Error 2: False Positives สูงสำหรับ Swimming Pools

```python
# ปัญหา: สระว่ายน้ำถูก classify ว่า flood

# แก้: เพิ่ม training data ของ swimming pools เป็น negative examples
# และเพิ่ม context features

def analyze_with_context(image, location_context):
    flood_prob = model.predict(image)
    
    # Context: ถ้าไม่ใช่ฤดูน้ำท่วม → ลด confidence
    if not location_context.get('is_flood_season'):
        flood_prob *= 0.8
    
    # Context: ถ้าอยู่ใกล้ recreational area → ลด confidence
    if location_context.get('near_recreational_area'):
        flood_prob *= 0.6
    
    return flood_prob
```

---

## ✅ Checklist

- [ ] AWS Rekognition content moderation เปิดใช้งาน
- [ ] YOLOv8 flood detection model trained (F1 > 0.85)
- [ ] ResNet binary classifier trained (Recall > 0.90)
- [ ] Lambda processing pipeline ทำงาน end-to-end
- [ ] Human review queue สำหรับ low-confidence predictions
- [ ] Model deployed บน Lambda (cold start < 3s)
- [ ] S3 event trigger → Lambda → SQS setup
- [ ] Confidence threshold ตั้งค่าแล้ว (0.8)
- [ ] Model monitoring: accuracy tracking ทุกสัปดาห์
- [ ] Training data labeling process มีแล้ว

---

## 🔗 References

- [YOLOv8 Documentation](https://docs.ultralytics.com/)
- [AWS Rekognition Custom Labels](https://docs.aws.amazon.com/rekognition/latest/customlabels-dg/what-is.html)
- [ONNX Runtime](https://onnxruntime.ai/)
- [Flood Detection Research Papers](https://arxiv.org/search/?searchtype=all&query=flood+detection+deep+learning)
- [Roboflow — Flood Dataset](https://universe.roboflow.com/search?q=flood)

---

*Part 102 | Road to 1,000,000 Users/Day | chuaikan.com*
