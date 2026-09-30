# Part 013: File Upload & Media Processing

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 121-130
> **เวลาโดยประมาณ:** 8 ชั่วโมง
> **Prerequisites:** Part 012 (WebSocket), Part 001-010 (Infrastructure)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

1. ใช้ Multer สำหรับ Multipart Upload พร้อม file type validation
2. ประมวลผลภาพด้วย Sharp: resize, WebP conversion, thumbnails
3. Upload วิดีโอและ transcode ด้วย FFmpeg
4. Upload ไปยัง Cloudflare R2 (S3-compatible) ด้วย AWS SDK v3
5. สร้าง Signed URLs สำหรับ secure access
6. Optimize รูปภาพ: Progressive JPEG และ WebP
7. Setup Image CDN ด้วย Cloudflare Images หรือ imgproxy
8. Chunked upload สำหรับไฟล์ขนาดใหญ่
9. ตรวจสอบไฟล์: MIME type check และ virus scan ด้วย ClamAV
10. Frontend: Drag-and-drop upload พร้อม progress bar

---

## 📖 ทฤษฎีและแนวคิด

### File Upload Flow สำหรับ chuaikan.com

```
User Browser                 API Server              Cloudflare R2
     │                           │                        │
     │  POST /upload (multipart) │                        │
     │──────────────────────────►│                        │
     │                           │                        │
     │                           │ 1. Validate file type  │
     │                           │ 2. Scan for viruses    │
     │                           │ 3. Resize/Convert      │
     │                           │    (Sharp/FFmpeg)      │
     │                           │                        │
     │                           │  PUT /{key}            │
     │                           │───────────────────────►│
     │                           │                        │
     │                           │  Store URL in DB       │
     │  { url, thumbnailUrl }    │                        │
     │◄──────────────────────────│                        │
     │                           │                        │

สำหรับไฟล์ขนาดใหญ่ (Chunked Upload):
     │                           │
     │  POST /upload/init        │  → ได้ uploadId
     │──────────────────────────►│
     │  PUT /upload/{id}/chunk/1 │  → ส่ง chunk 1
     │  PUT /upload/{id}/chunk/2 │  → ส่ง chunk 2
     │  POST /upload/{id}/complete│ → รวม chunks
     │◄──────────────────────────│
```

### Image Processing Pipeline

```
Original Image
(JPEG 5MB, 4000x3000px)
         │
         ▼
    ┌─────────────────────────────────────┐
    │          Sharp Processing           │
    │                                     │
    │  Original → WebP (full size)        │
    │  Original → WebP (1200px wide)      │
    │  Original → WebP (600px wide)       │
    │  Original → WebP (300px thumb)      │
    │  Original → WebP (150px avatar)     │
    └─────────────────────────────────────┘
         │
         ▼
    Cloudflare R2
    media/posts/{userId}/{uuid}/
    ├── original.webp    (full, ~800KB)
    ├── large.webp       (1200px, ~300KB)
    ├── medium.webp      (600px, ~100KB)
    ├── thumbnail.webp   (300px, ~30KB)
    └── avatar.webp      (150px, ~10KB)
```

---

## ⚙️ Environment Setup

### Step 121: ติดตั้ง Dependencies

```bash
# ไปที่ project directory
cd /home/user/chuaikan-api

# ติดตั้ง packages หลัก
npm install multer@1.4.5-lts.1 \
  sharp@0.33.4 \
  @aws-sdk/client-s3@3.590.0 \
  @aws-sdk/s3-request-presigner@3.590.0 \
  uuid@10.0.0 \
  mime-types@2.1.35 \
  file-type@19.0.0

# Type definitions
npm install -D \
  @types/multer@1.4.11 \
  @types/uuid@10.0.0 \
  @types/mime-types@2.1.4

# ติดตั้ง FFmpeg สำหรับ video processing
sudo apt-get update
sudo apt-get install -y ffmpeg

# ตรวจสอบ FFmpeg
ffmpeg -version
# ffmpeg version 6.1.x

# ติดตั้ง ClamAV สำหรับ virus scanning
sudo apt-get install -y clamav clamav-daemon
sudo systemctl start clamav-freshclam
sudo systemctl enable clamav-freshclam
sudo systemctl start clamav-daemon
sudo systemctl enable clamav-daemon

# อัปเดต virus signatures (รอประมาณ 5-10 นาที)
sudo freshclam

# ตรวจสอบ ClamAV
clamscan --version
# ClamAV 1.x.x

# ติดตั้ง fluent-ffmpeg สำหรับ Node.js
npm install fluent-ffmpeg@2.1.3
npm install -D @types/fluent-ffmpeg@2.1.24
```

### Step 122: ตั้งค่า Cloudflare R2

```bash
# ใน Cloudflare Dashboard:
# 1. ไปที่ R2 → Create bucket
# 2. ตั้งชื่อ bucket: chuaikan-media
# 3. ใน bucket settings → CORS Policy

# สร้าง R2 API Token:
# Account Home → R2 → Manage R2 API Tokens
# สร้าง token ด้วย permission: Object Read & Write

# เพิ่มไปใน .env
cat >> /home/user/chuaikan-api/.env << 'EOF'
CLOUDFLARE_ACCOUNT_ID=your_account_id
CLOUDFLARE_R2_ACCESS_KEY_ID=your_r2_access_key
CLOUDFLARE_R2_SECRET_ACCESS_KEY=your_r2_secret_key
CLOUDFLARE_R2_BUCKET_NAME=chuaikan-media
CLOUDFLARE_R2_ENDPOINT=https://your_account_id.r2.cloudflarestorage.com
CLOUDFLARE_R2_PUBLIC_URL=https://media.chuaikan.com
MAX_FILE_SIZE_IMAGE=10485760
MAX_FILE_SIZE_VIDEO=104857600
UPLOAD_DIR=/tmp/uploads
EOF
```

---

## 🛠️ Step-by-Step Implementation

### Step 123: Multer Configuration และ File Validation

สร้างไฟล์ `/home/user/chuaikan-api/src/upload/multerConfig.ts`:

```typescript
import multer from 'multer';
import path from 'path';
import fs from 'fs';
import { Request } from 'express';
import { fileTypeFromBuffer } from 'file-type';

// สร้าง temp directory
const UPLOAD_DIR = process.env.UPLOAD_DIR || '/tmp/uploads';
if (!fs.existsSync(UPLOAD_DIR)) {
  fs.mkdirSync(UPLOAD_DIR, { recursive: true });
}

// MIME types ที่อนุญาต
const ALLOWED_IMAGE_TYPES = new Set([
  'image/jpeg',
  'image/png',
  'image/webp',
  'image/gif',
  'image/heic',
  'image/heif',
]);

const ALLOWED_VIDEO_TYPES = new Set([
  'video/mp4',
  'video/quicktime',
  'video/x-msvideo',
  'video/webm',
]);

const ALLOWED_DOCUMENT_TYPES = new Set([
  'application/pdf',
]);

// Multer storage (ใช้ disk storage เพื่อ process ก่อน upload)
const storage = multer.diskStorage({
  destination: (req, file, cb) => {
    cb(null, UPLOAD_DIR);
  },
  filename: (req, file, cb) => {
    // ใช้ timestamp + random เพื่อป้องกัน collision
    const uniqueSuffix = `${Date.now()}-${Math.round(Math.random() * 1e9)}`;
    const ext = path.extname(file.originalname).toLowerCase();
    cb(null, `upload-${uniqueSuffix}${ext}`);
  },
});

// File filter function
const fileFilter = async (
  req: Request,
  file: Express.Multer.File,
  cb: multer.FileFilterCallback
): Promise<void> => {
  const contentType = file.mimetype.toLowerCase();

  const allAllowed = new Set([
    ...ALLOWED_IMAGE_TYPES,
    ...ALLOWED_VIDEO_TYPES,
    ...ALLOWED_DOCUMENT_TYPES,
  ]);

  if (!allAllowed.has(contentType)) {
    cb(new Error(`File type not allowed: ${contentType}`));
    return;
  }

  cb(null, true);
};

// Export multer instances สำหรับ use cases ต่าง ๆ
export const uploadImage = multer({
  storage,
  fileFilter,
  limits: {
    fileSize: parseInt(process.env.MAX_FILE_SIZE_IMAGE || '10485760'), // 10MB
    files: 10,        // max 10 files ต่อ request
    fields: 20,       // max 20 form fields
    fieldNameSize: 100,
    fieldSize: 1024,  // 1KB per field
  },
});

export const uploadVideo = multer({
  storage,
  fileFilter,
  limits: {
    fileSize: parseInt(process.env.MAX_FILE_SIZE_VIDEO || '104857600'), // 100MB
    files: 1,
  },
});

export const uploadSingle = multer({
  storage,
  fileFilter,
  limits: { fileSize: 10485760 },
});

// Validate file magic bytes (ป้องกัน MIME type spoofing)
export async function validateFileMagicBytes(
  filePath: string,
  declaredMimeType: string
): Promise<boolean> {
  try {
    const buffer = fs.readFileSync(filePath);
    const fileType = await fileTypeFromBuffer(buffer);

    if (!fileType) {
      // ไม่สามารถระบุ type ได้ → reject
      return false;
    }

    // ตรวจสอบว่า actual type ตรงกับ declared type
    const actualMime = fileType.mime;
    
    // อนุญาต HEIC/HEIF ซึ่ง file-type อาจ detect เป็น image/heic
    if (declaredMimeType === 'image/heic' && actualMime === 'image/heic') {
      return true;
    }

    return actualMime === declaredMimeType;
  } catch (error) {
    console.error('Error validating file magic bytes:', error);
    return false;
  }
}
```

### Step 124: Sharp Image Processing

สร้างไฟล์ `/home/user/chuaikan-api/src/upload/imageProcessor.ts`:

```typescript
import sharp from 'sharp';
import path from 'path';
import fs from 'fs';
import { v4 as uuidv4 } from 'uuid';

export interface ImageVariant {
  name: string;
  width?: number;
  height?: number;
  quality: number;
  suffix: string;
}

export interface ProcessedImage {
  variant: string;
  width: number;
  height: number;
  size: number;
  format: string;
  localPath: string;
  key: string;   // S3/R2 key
}

// Image variants ที่ต้องสร้างสำหรับ chuaikan.com
const IMAGE_VARIANTS: ImageVariant[] = [
  { name: 'original', quality: 85, suffix: 'original' },
  { name: 'large',    width: 1200, quality: 80, suffix: 'large' },
  { name: 'medium',   width: 600,  quality: 75, suffix: 'medium' },
  { name: 'thumbnail', width: 300, height: 300, quality: 70, suffix: 'thumb' },
  { name: 'avatar',   width: 150,  height: 150,  quality: 70, suffix: 'avatar' },
];

export async function processImage(
  inputPath: string,
  userId: string,
  purpose: 'post' | 'avatar' | 'sos' = 'post'
): Promise<ProcessedImage[]> {
  const outputDir = `/tmp/processed/${uuidv4()}`;
  fs.mkdirSync(outputDir, { recursive: true });

  const baseKey = `media/${purpose}/${userId}/${uuidv4()}`;
  const results: ProcessedImage[] = [];

  // เลือก variants ตาม purpose
  const variants =
    purpose === 'avatar'
      ? IMAGE_VARIANTS.filter((v) => ['thumbnail', 'avatar'].includes(v.name))
      : IMAGE_VARIANTS;

  try {
    // โหลด original image metadata
    const metadata = await sharp(inputPath).metadata();
    console.log(`Processing image: ${metadata.width}x${metadata.height} ${metadata.format}`);

    for (const variant of variants) {
      const outputPath = path.join(outputDir, `${variant.suffix}.webp`);
      const key = `${baseKey}/${variant.suffix}.webp`;

      let pipeline = sharp(inputPath)
        .rotate()  // Auto-rotate based on EXIF
        .webp({ quality: variant.quality, effort: 4 });

      // Resize ถ้ากำหนด dimensions
      if (variant.width || variant.height) {
        if (variant.width && variant.height) {
          // Crop แบบ cover (สำหรับ thumbnail และ avatar)
          pipeline = sharp(inputPath)
            .rotate()
            .resize(variant.width, variant.height, {
              fit: 'cover',
              position: 'centre',
            })
            .webp({ quality: variant.quality, effort: 4 });
        } else {
          // Resize แบบ contain (คง aspect ratio)
          pipeline = sharp(inputPath)
            .rotate()
            .resize(variant.width, variant.height, {
              fit: 'inside',
              withoutEnlargement: true,
            })
            .webp({ quality: variant.quality, effort: 4 });
        }
      }

      // บันทึกไฟล์
      const outputInfo = await pipeline.toFile(outputPath);

      results.push({
        variant: variant.name,
        width: outputInfo.width,
        height: outputInfo.height,
        size: outputInfo.size,
        format: 'webp',
        localPath: outputPath,
        key,
      });

      console.log(
        `Created ${variant.name}: ${outputInfo.width}x${outputInfo.height} (${(outputInfo.size / 1024).toFixed(1)}KB)`
      );
    }

    return results;
  } finally {
    // Cleanup input file
    if (fs.existsSync(inputPath)) {
      fs.unlinkSync(inputPath);
    }
  }
}

// สร้าง Progressive JPEG (fallback สำหรับ browsers ที่ไม่รองรับ WebP)
export async function createProgressiveJpeg(
  inputPath: string,
  outputPath: string,
  width: number
): Promise<void> {
  await sharp(inputPath)
    .rotate()
    .resize(width, undefined, { fit: 'inside', withoutEnlargement: true })
    .jpeg({
      quality: 80,
      progressive: true,  // Progressive JPEG
      mozjpeg: true,       // ใช้ mozjpeg encoder (ไฟล์เล็กกว่า ~10-20%)
    })
    .toFile(outputPath);
}

// ดึง dominant color จากรูปภาพ (สำหรับ placeholder)
export async function getDominantColor(inputPath: string): Promise<string> {
  const { dominant } = await sharp(inputPath)
    .resize(1, 1)
    .raw()
    .toBuffer({ resolveWithObject: true })
    .then(({ data }) => ({
      dominant: {
        r: data[0],
        g: data[1],
        b: data[2],
      },
    }));

  return `#${dominant.r.toString(16).padStart(2, '0')}${dominant.g.toString(16).padStart(2, '0')}${dominant.b.toString(16).padStart(2, '0')}`;
}
```

### Step 125: Video Processing ด้วย FFmpeg

สร้างไฟล์ `/home/user/chuaikan-api/src/upload/videoProcessor.ts`:

```typescript
import ffmpeg from 'fluent-ffmpeg';
import path from 'path';
import fs from 'fs';
import { v4 as uuidv4 } from 'uuid';

export interface VideoOutput {
  variant: string;
  width: number;
  height: number;
  duration: number;
  size: number;
  localPath: string;
  key: string;
  thumbnailPath?: string;
  thumbnailKey?: string;
}

// ตรวจสอบ video metadata
export function getVideoMetadata(inputPath: string): Promise<ffmpeg.FfprobeData> {
  return new Promise((resolve, reject) => {
    ffmpeg.ffprobe(inputPath, (err, metadata) => {
      if (err) reject(err);
      else resolve(metadata);
    });
  });
}

// Transcode video เป็น MP4 H.264
export async function transcodeVideo(
  inputPath: string,
  userId: string
): Promise<VideoOutput[]> {
  const outputDir = `/tmp/video-processed/${uuidv4()}`;
  fs.mkdirSync(outputDir, { recursive: true });

  const baseKey = `media/video/${userId}/${uuidv4()}`;
  const results: VideoOutput[] = [];

  // ดึง metadata ก่อน
  const metadata = await getVideoMetadata(inputPath);
  const videoStream = metadata.streams.find((s) => s.codec_type === 'video');
  const duration = metadata.format.duration || 0;

  if (!videoStream) {
    throw new Error('No video stream found');
  }

  const originalWidth = videoStream.width || 1920;
  const originalHeight = videoStream.height || 1080;

  // กำหนด output variants
  const variants = [
    { name: 'hd', width: 1280, height: 720, crf: 23, preset: 'medium' },
    { name: 'sd', width: 854, height: 480, crf: 25, preset: 'fast' },
    { name: 'mobile', width: 640, height: 360, crf: 27, preset: 'fast' },
  ].filter(
    (v) => v.width <= originalWidth || v.height <= originalHeight
  );

  for (const variant of variants) {
    const outputPath = path.join(outputDir, `${variant.name}.mp4`);
    const key = `${baseKey}/${variant.name}.mp4`;

    await new Promise<void>((resolve, reject) => {
      ffmpeg(inputPath)
        .videoCodec('libx264')
        .audioCodec('aac')
        .audioBitrate('128k')
        .outputOptions([
          `-crf ${variant.crf}`,
          `-preset ${variant.preset}`,
          '-movflags +faststart',     // ย้าย moov atom ไปต้นไฟล์ (streaming)
          '-pix_fmt yuv420p',          // compat สำหรับ player ต่าง ๆ
          `-vf scale=${variant.width}:${variant.height}:force_original_aspect_ratio=decrease,pad=${variant.width}:${variant.height}:(ow-iw)/2:(oh-ih)/2`,
          '-threads 0',               // ใช้ทุก CPU threads
        ])
        .output(outputPath)
        .on('start', (cmd) => console.log('FFmpeg command:', cmd))
        .on('progress', (progress) => {
          console.log(`Processing ${variant.name}: ${progress.percent?.toFixed(1)}%`);
        })
        .on('end', resolve)
        .on('error', reject)
        .run();
    });

    const stats = fs.statSync(outputPath);
    results.push({
      variant: variant.name,
      width: variant.width,
      height: variant.height,
      duration,
      size: stats.size,
      localPath: outputPath,
      key,
    });
  }

  // สร้าง Thumbnail จาก video
  const thumbnailPath = path.join(outputDir, 'thumbnail.jpg');
  const thumbnailKey = `${baseKey}/thumbnail.jpg`;

  await new Promise<void>((resolve, reject) => {
    ffmpeg(inputPath)
      .screenshots({
        count: 1,
        folder: outputDir,
        filename: 'thumbnail.jpg',
        timemarks: ['00:00:01'],   // ดึงภาพที่ 1 วินาที
        size: '640x360',
      })
      .on('end', resolve)
      .on('error', reject);
  });

  if (results.length > 0) {
    results[0].thumbnailPath = thumbnailPath;
    results[0].thumbnailKey = thumbnailKey;
  }

  return results;
}
```

### Step 126: Cloudflare R2 Upload Service

สร้างไฟล์ `/home/user/chuaikan-api/src/upload/r2Service.ts`:

```typescript
import {
  S3Client,
  PutObjectCommand,
  DeleteObjectCommand,
  GetObjectCommand,
  HeadObjectCommand,
  CreateMultipartUploadCommand,
  UploadPartCommand,
  CompleteMultipartUploadCommand,
  AbortMultipartUploadCommand,
} from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import fs from 'fs';
import mime from 'mime-types';

const r2Client = new S3Client({
  region: 'auto',
  endpoint: process.env.CLOUDFLARE_R2_ENDPOINT,
  credentials: {
    accessKeyId: process.env.CLOUDFLARE_R2_ACCESS_KEY_ID!,
    secretAccessKey: process.env.CLOUDFLARE_R2_SECRET_ACCESS_KEY!,
  },
});

const BUCKET = process.env.CLOUDFLARE_R2_BUCKET_NAME!;
const PUBLIC_URL = process.env.CLOUDFLARE_R2_PUBLIC_URL!;

// Upload ไฟล์เดียว
export async function uploadFileToR2(
  localPath: string,
  key: string,
  options: { contentType?: string; isPublic?: boolean; metadata?: Record<string, string> } = {}
): Promise<string> {
  const fileBuffer = fs.readFileSync(localPath);
  const contentType =
    options.contentType || mime.lookup(key) || 'application/octet-stream';

  const command = new PutObjectCommand({
    Bucket: BUCKET,
    Key: key,
    Body: fileBuffer,
    ContentType: contentType,
    CacheControl: 'public, max-age=31536000, immutable', // 1 ปี
    Metadata: options.metadata,
  });

  await r2Client.send(command);

  // คืน public URL
  return `${PUBLIC_URL}/${key}`;
}

// Upload หลายไฟล์พร้อมกัน
export async function uploadMultipleFiles(
  files: Array<{ localPath: string; key: string; contentType?: string }>
): Promise<string[]> {
  const uploads = files.map(({ localPath, key, contentType }) =>
    uploadFileToR2(localPath, key, { contentType })
  );
  return Promise.all(uploads);
}

// สร้าง Signed URL สำหรับ private files (หมดอายุใน N วินาที)
export async function createSignedUrl(
  key: string,
  expiresInSeconds: number = 3600
): Promise<string> {
  const command = new GetObjectCommand({
    Bucket: BUCKET,
    Key: key,
  });

  return getSignedUrl(r2Client, command, { expiresIn: expiresInSeconds });
}

// สร้าง Presigned URL สำหรับ direct upload จาก client (ไม่ผ่าน server)
export async function createPresignedUploadUrl(
  key: string,
  contentType: string,
  expiresInSeconds: number = 3600
): Promise<{ url: string; fields: Record<string, string> }> {
  const command = new PutObjectCommand({
    Bucket: BUCKET,
    Key: key,
    ContentType: contentType,
  });

  const url = await getSignedUrl(r2Client, command, {
    expiresIn: expiresInSeconds,
  });

  return { url, fields: {} };
}

// ลบไฟล์
export async function deleteFileFromR2(key: string): Promise<void> {
  const command = new DeleteObjectCommand({
    Bucket: BUCKET,
    Key: key,
  });
  await r2Client.send(command);
}

// ============================================
// Step 127: Chunked Upload สำหรับไฟล์ขนาดใหญ่
// ============================================

interface MultipartUploadState {
  uploadId: string;
  key: string;
  parts: Array<{ ETag: string; PartNumber: number }>;
}

// เริ่ม multipart upload
export async function initializeMultipartUpload(
  key: string,
  contentType: string
): Promise<string> {
  const command = new CreateMultipartUploadCommand({
    Bucket: BUCKET,
    Key: key,
    ContentType: contentType,
    CacheControl: 'public, max-age=31536000',
  });

  const response = await r2Client.send(command);
  return response.UploadId!;
}

// Upload แต่ละ part
export async function uploadPart(
  key: string,
  uploadId: string,
  partNumber: number,
  chunk: Buffer
): Promise<string> {
  const command = new UploadPartCommand({
    Bucket: BUCKET,
    Key: key,
    UploadId: uploadId,
    PartNumber: partNumber,
    Body: chunk,
  });

  const response = await r2Client.send(command);
  return response.ETag!;
}

// Complete multipart upload
export async function completeMultipartUpload(
  key: string,
  uploadId: string,
  parts: Array<{ ETag: string; PartNumber: number }>
): Promise<string> {
  const command = new CompleteMultipartUploadCommand({
    Bucket: BUCKET,
    Key: key,
    UploadId: uploadId,
    MultipartUpload: { Parts: parts },
  });

  await r2Client.send(command);
  return `${PUBLIC_URL}/${key}`;
}

// Abort multipart upload (กรณี error)
export async function abortMultipartUpload(
  key: string,
  uploadId: string
): Promise<void> {
  const command = new AbortMultipartUploadCommand({
    Bucket: BUCKET,
    Key: key,
    UploadId: uploadId,
  });
  await r2Client.send(command);
}
```

### Step 128: Virus Scan ด้วย ClamAV

สร้างไฟล์ `/home/user/chuaikan-api/src/upload/virusScanner.ts`:

```typescript
import { exec } from 'child_process';
import { promisify } from 'util';
import path from 'path';

const execAsync = promisify(exec);

export interface ScanResult {
  infected: boolean;
  virusName?: string;
  error?: string;
}

export async function scanFileForViruses(filePath: string): Promise<ScanResult> {
  try {
    // ใช้ clamdscan (daemon mode - เร็วกว่า clamscan)
    const { stdout, stderr } = await execAsync(
      `clamdscan --no-summary --fdpass "${filePath}"`,
      { timeout: 30000 }
    );

    // Parse result
    if (stdout.includes('OK')) {
      return { infected: false };
    }

    if (stdout.includes('FOUND')) {
      // Extract virus name จาก output
      const match = stdout.match(/: (.+) FOUND/);
      const virusName = match ? match[1] : 'Unknown virus';
      return { infected: true, virusName };
    }

    return { infected: false };

  } catch (error: unknown) {
    const err = error as { code?: number; message?: string };
    
    if (err.code === 1) {
      // Exit code 1 = infected file found
      return { infected: true, virusName: 'Detected by ClamAV' };
    }

    // Exit code 2 = error
    console.error('ClamAV scan error:', err.message);
    
    // ใน production อาจ reject ถ้า scan ไม่ได้
    // ใน development/staging อาจ allow ผ่านไปก่อน
    if (process.env.NODE_ENV === 'production') {
      return { infected: true, error: 'Scan failed - rejecting for safety' };
    }
    
    return { infected: false, error: err.message };
  }
}
```

### Step 129: Upload Route (Express)

สร้างไฟล์ `/home/user/chuaikan-api/src/routes/upload.ts`:

```typescript
import { Router, Request, Response } from 'express';
import { uploadImage, uploadVideo, uploadSingle, validateFileMagicBytes } from '../upload/multerConfig';
import { processImage } from '../upload/imageProcessor';
import { transcodeVideo } from '../upload/videoProcessor';
import { uploadFileToR2, uploadMultipleFiles, initializeMultipartUpload, uploadPart, completeMultipartUpload, abortMultipartUpload } from '../upload/r2Service';
import { scanFileForViruses } from '../upload/virusScanner';
import { authMiddleware } from '../middleware/auth';
import { v4 as uuidv4 } from 'uuid';
import fs from 'fs';
import path from 'path';

const router = Router();

// ============================================
// POST /upload/image - Upload รูปภาพ
// ============================================
router.post(
  '/image',
  authMiddleware,
  uploadImage.array('images', 10),
  async (req: Request, res: Response) => {
    const files = req.files as Express.Multer.File[];
    const userId = (req as any).user.sub;

    if (!files || files.length === 0) {
      return res.status(400).json({ error: 'No files uploaded' });
    }

    const results = [];

    try {
      for (const file of files) {
        // 1. Validate magic bytes
        const isValidMagic = await validateFileMagicBytes(file.path, file.mimetype);
        if (!isValidMagic) {
          fs.unlinkSync(file.path);
          return res.status(400).json({ error: `Invalid file type: ${file.originalname}` });
        }

        // 2. Scan for viruses
        const scanResult = await scanFileForViruses(file.path);
        if (scanResult.infected) {
          fs.unlinkSync(file.path);
          return res.status(400).json({
            error: `File rejected: ${scanResult.virusName || 'Virus detected'}`,
          });
        }

        // 3. Process image (resize, convert to WebP)
        const processedImages = await processImage(file.path, userId, 'post');

        // 4. Upload ทุก variants ไป R2
        const uploadResults = await uploadMultipleFiles(
          processedImages.map((img) => ({
            localPath: img.localPath,
            key: img.key,
            contentType: 'image/webp',
          }))
        );

        // 5. Cleanup temp files
        processedImages.forEach((img) => {
          if (fs.existsSync(img.localPath)) fs.unlinkSync(img.localPath);
        });

        // สร้าง URL map
        const urlMap: Record<string, string> = {};
        processedImages.forEach((img, index) => {
          urlMap[img.variant] = uploadResults[index];
        });

        results.push({
          originalName: file.originalname,
          urls: urlMap,
          // TODO: บันทึกลง database และคืน mediaId
          mediaId: uuidv4(),
        });
      }

      res.status(201).json({ files: results });
    } catch (error) {
      console.error('Upload error:', error);
      // Cleanup ไฟล์ที่เหลือ
      files.forEach((file) => {
        if (fs.existsSync(file.path)) fs.unlinkSync(file.path);
      });
      res.status(500).json({ error: 'Upload failed' });
    }
  }
);

// ============================================
// POST /upload/video - Upload วิดีโอ
// ============================================
router.post(
  '/video',
  authMiddleware,
  uploadVideo.single('video'),
  async (req: Request, res: Response) => {
    const file = req.file;
    const userId = (req as any).user.sub;

    if (!file) {
      return res.status(400).json({ error: 'No video uploaded' });
    }

    try {
      // Validate MIME type
      const isValid = await validateFileMagicBytes(file.path, file.mimetype);
      if (!isValid) {
        fs.unlinkSync(file.path);
        return res.status(400).json({ error: 'Invalid video file' });
      }

      // Scan viruses
      const scan = await scanFileForViruses(file.path);
      if (scan.infected) {
        fs.unlinkSync(file.path);
        return res.status(400).json({ error: 'Video rejected by virus scanner' });
      }

      // Transcode video (อาจใช้เวลานาน → ใช้ background job)
      res.status(202).json({
        message: 'Video upload accepted. Processing...',
        jobId: uuidv4(),
      });

      // Process ใน background
      setImmediate(async () => {
        try {
          const videoOutputs = await transcodeVideo(file.path, userId);

          for (const output of videoOutputs) {
            await uploadFileToR2(output.localPath, output.key, {
              contentType: 'video/mp4',
            });

            if (output.thumbnailPath) {
              await uploadFileToR2(output.thumbnailPath, output.thumbnailKey!, {
                contentType: 'image/jpeg',
              });
            }

            // Cleanup
            if (fs.existsSync(output.localPath)) fs.unlinkSync(output.localPath);
          }
        } catch (bgError) {
          console.error('Background video processing error:', bgError);
        }
      });

    } catch (error) {
      console.error('Video upload error:', error);
      if (fs.existsSync(file.path)) fs.unlinkSync(file.path);
      res.status(500).json({ error: 'Video upload failed' });
    }
  }
);

// ============================================
// Chunked Upload Endpoints
// ============================================

// เริ่ม chunked upload
router.post('/chunked/init', authMiddleware, async (req, res) => {
  const { filename, contentType, fileSize } = req.body;

  if (fileSize > 500 * 1024 * 1024) { // max 500MB
    return res.status(400).json({ error: 'File too large (max 500MB)' });
  }

  const userId = (req as any).user.sub;
  const key = `media/uploads/${userId}/${uuidv4()}/${filename}`;
  const uploadId = await initializeMultipartUpload(key, contentType);

  res.json({ uploadId, key });
});

// Upload แต่ละ chunk
router.put('/chunked/:uploadId/chunk/:partNumber', authMiddleware, uploadSingle.single('chunk'), async (req, res) => {
  const { uploadId } = req.params;
  const partNumber = parseInt(req.params.partNumber);
  const { key } = req.body;
  const file = req.file;

  if (!file) return res.status(400).json({ error: 'No chunk data' });

  try {
    const chunk = fs.readFileSync(file.path);
    const etag = await uploadPart(key, uploadId, partNumber, chunk);
    fs.unlinkSync(file.path);
    res.json({ etag, partNumber });
  } catch (error) {
    if (fs.existsSync(file!.path)) fs.unlinkSync(file!.path);
    res.status(500).json({ error: 'Chunk upload failed' });
  }
});

// Complete chunked upload
router.post('/chunked/:uploadId/complete', authMiddleware, async (req, res) => {
  const { uploadId } = req.params;
  const { key, parts } = req.body;

  try {
    const finalUrl = await completeMultipartUpload(key, uploadId, parts);
    res.json({ url: finalUrl });
  } catch (error) {
    await abortMultipartUpload(key, uploadId).catch(() => {});
    res.status(500).json({ error: 'Failed to complete upload' });
  }
});

export default router;
```

---

## 🔧 Configuration Files

### Database Schema สำหรับ Media Files

```sql
-- PostgreSQL 17 schema สำหรับ media files

CREATE TABLE media_files (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id      UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  post_id      UUID REFERENCES posts(id) ON DELETE SET NULL,
  
  -- File info
  original_name   VARCHAR(255),
  file_type       VARCHAR(50) NOT NULL,  -- 'image', 'video', 'document'
  mime_type       VARCHAR(100) NOT NULL,
  file_size       BIGINT NOT NULL,  -- bytes
  
  -- Storage keys (Cloudflare R2)
  storage_key     TEXT NOT NULL,
  public_url      TEXT NOT NULL,
  
  -- Image variants (JSONB)
  variants        JSONB,
  /* Example:
  {
    "original": "https://media.chuaikan.com/media/post/user1/uuid/original.webp",
    "large":    "https://media.chuaikan.com/media/post/user1/uuid/large.webp",
    "medium":   "https://media.chuaikan.com/media/post/user1/uuid/medium.webp",
    "thumbnail":"https://media.chuaikan.com/media/post/user1/uuid/thumb.webp"
  }
  */
  
  -- Image metadata
  width        INT,
  height       INT,
  duration     FLOAT,  -- สำหรับ video (วินาที)
  
  -- Processing status
  status       VARCHAR(50) DEFAULT 'pending',
  -- 'pending', 'processing', 'ready', 'failed'
  
  -- Security
  virus_scanned   BOOLEAN DEFAULT FALSE,
  virus_scan_at   TIMESTAMPTZ,
  
  -- Dominant color สำหรับ placeholder (LQIP)
  dominant_color  VARCHAR(7),  -- HEX เช่น '#FF5733'
  
  -- Timestamps
  created_at   TIMESTAMPTZ DEFAULT NOW(),
  updated_at   TIMESTAMPTZ DEFAULT NOW(),
  deleted_at   TIMESTAMPTZ  -- soft delete
);

CREATE INDEX idx_media_user_id ON media_files(user_id);
CREATE INDEX idx_media_post_id ON media_files(post_id) WHERE post_id IS NOT NULL;
CREATE INDEX idx_media_status ON media_files(status) WHERE status != 'ready';
```

---

## 🧪 Testing

### Step 130: Frontend Drag-and-Drop Upload Component

สร้างไฟล์ `/home/user/chuaikan-web/src/components/FileUpload/ImageUploader.tsx`:

```tsx
'use client';

import { useState, useCallback, useRef } from 'react';

interface UploadedFile {
  id: string;
  name: string;
  preview: string;
  progress: number;
  status: 'pending' | 'uploading' | 'success' | 'error';
  urls?: Record<string, string>;
  error?: string;
}

interface ImageUploaderProps {
  onUploadComplete: (files: UploadedFile[]) => void;
  maxFiles?: number;
  accept?: string;
}

export function ImageUploader({
  onUploadComplete,
  maxFiles = 10,
  accept = 'image/*',
}: ImageUploaderProps) {
  const [files, setFiles] = useState<UploadedFile[]>([]);
  const [isDragging, setIsDragging] = useState(false);
  const inputRef = useRef<HTMLInputElement>(null);

  const processFiles = useCallback(
    async (newFiles: File[]) => {
      const uploadFiles: UploadedFile[] = newFiles.map((file) => ({
        id: Math.random().toString(36).substr(2, 9),
        name: file.name,
        preview: URL.createObjectURL(file),
        progress: 0,
        status: 'pending',
      }));

      setFiles((prev) => [...prev, ...uploadFiles].slice(0, maxFiles));

      // Upload แต่ละไฟล์
      for (let i = 0; i < uploadFiles.length; i++) {
        const uploadFile = uploadFiles[i];
        const file = newFiles[i];

        try {
          // อัปเดต status เป็น uploading
          setFiles((prev) =>
            prev.map((f) =>
              f.id === uploadFile.id ? { ...f, status: 'uploading' } : f
            )
          );

          // ใช้ XMLHttpRequest เพื่อดู upload progress
          const urls = await uploadWithProgress(
            file,
            (progress) => {
              setFiles((prev) =>
                prev.map((f) =>
                  f.id === uploadFile.id ? { ...f, progress } : f
                )
              );
            }
          );

          setFiles((prev) =>
            prev.map((f) =>
              f.id === uploadFile.id
                ? { ...f, status: 'success', progress: 100, urls }
                : f
            )
          );
        } catch (error) {
          setFiles((prev) =>
            prev.map((f) =>
              f.id === uploadFile.id
                ? { ...f, status: 'error', error: 'Upload failed' }
                : f
            )
          );
        }
      }
    },
    [maxFiles]
  );

  const handleDrop = useCallback(
    (e: React.DragEvent) => {
      e.preventDefault();
      setIsDragging(false);
      const droppedFiles = Array.from(e.dataTransfer.files).filter(
        (f) => f.type.startsWith('image/')
      );
      processFiles(droppedFiles);
    },
    [processFiles]
  );

  return (
    <div className="image-uploader">
      {/* Drop zone */}
      <div
        className={`drop-zone ${isDragging ? 'dragging' : ''}`}
        onDrop={handleDrop}
        onDragOver={(e) => { e.preventDefault(); setIsDragging(true); }}
        onDragLeave={() => setIsDragging(false)}
        onClick={() => inputRef.current?.click()}
      >
        <input
          ref={inputRef}
          type="file"
          accept={accept}
          multiple
          hidden
          onChange={(e) => {
            const selected = Array.from(e.target.files || []);
            processFiles(selected);
          }}
        />
        <p>ลากและวางรูปภาพที่นี่ หรือคลิกเพื่อเลือก</p>
        <p className="hint">รองรับ JPG, PNG, WebP, HEIC ขนาดสูงสุด 10MB</p>
      </div>

      {/* File list with progress */}
      <div className="file-list">
        {files.map((file) => (
          <div key={file.id} className={`file-item ${file.status}`}>
            <img src={file.preview} alt={file.name} width={60} height={60} />
            <div className="file-info">
              <span>{file.name}</span>
              {file.status === 'uploading' && (
                <div className="progress-bar">
                  <div
                    className="progress-fill"
                    style={{ width: `${file.progress}%` }}
                  />
                  <span>{file.progress}%</span>
                </div>
              )}
              {file.status === 'success' && <span className="success">✓ อัปโหลดสำเร็จ</span>}
              {file.status === 'error' && <span className="error">{file.error}</span>}
            </div>
          </div>
        ))}
      </div>
    </div>
  );
}

async function uploadWithProgress(
  file: File,
  onProgress: (progress: number) => void
): Promise<Record<string, string>> {
  return new Promise((resolve, reject) => {
    const formData = new FormData();
    formData.append('images', file);

    const xhr = new XMLHttpRequest();

    xhr.upload.addEventListener('progress', (e) => {
      if (e.lengthComputable) {
        onProgress(Math.round((e.loaded / e.total) * 100));
      }
    });

    xhr.addEventListener('load', () => {
      if (xhr.status === 201) {
        const response = JSON.parse(xhr.responseText);
        resolve(response.files[0].urls);
      } else {
        reject(new Error(`Upload failed: ${xhr.status}`));
      }
    });

    xhr.addEventListener('error', () => reject(new Error('Network error')));

    xhr.open('POST', '/api/upload/image');
    xhr.setRequestHeader('Authorization', `Bearer ${getAccessToken()}`);
    xhr.send(formData);
  });
}

function getAccessToken(): string {
  return localStorage.getItem('accessToken') || '';
}
```

---

## ❌ Common Errors & Solutions

### Error 1: Sharp ไม่รองรับ HEIC/HEIF

```bash
# ติดตั้ง libvips ที่รองรับ HEIF
sudo apt-get install -y libvips-dev libheif-dev

# Rebuild sharp
npm rebuild sharp --build-from-source

# หรือ ติดตั้ง sharp ใหม่ด้วย flag
npm install sharp --platform=linux --arch=x64
```

### Error 2: FFmpeg out of memory สำหรับ large video

```bash
# จำกัด memory ใน FFmpeg command
ffmpeg -i input.mp4 \
  -vf "scale=1280:720:force_original_aspect_ratio=decrease" \
  -c:v libx264 \
  -crf 23 \
  -preset fast \
  -threads 2 \        # จำกัด threads
  -bufsize 2M \       # จำกัด buffer size
  output.mp4
```

### Error 3: ClamAV daemon ไม่ respond

```bash
# ตรวจสอบ ClamAV status
sudo systemctl status clamav-daemon

# Restart daemon
sudo systemctl restart clamav-daemon

# ตรวจสอบ socket
ls -la /var/run/clamav/clamd.ctl

# Test scan
echo "test" > /tmp/test.txt
clamdscan /tmp/test.txt
```

---

## ✅ Checklist

- [ ] Multer configured ด้วย file size limits ที่เหมาะสม
- [ ] MIME type validation ด้วยทั้ง content-type header และ magic bytes
- [ ] ClamAV ติดตั้งและ daemon ทำงาน
- [ ] Sharp สร้าง WebP variants ทุก size
- [ ] FFmpeg transcode video เป็น MP4 H.264
- [ ] Cloudflare R2 credentials ใน .env (ไม่ commit ลง git)
- [ ] R2 CORS policy configured สำหรับ frontend origin
- [ ] Multipart upload ทำงานสำหรับไฟล์ > 10MB
- [ ] Signed URLs สำหรับ private content
- [ ] Database schema บันทึก media metadata
- [ ] Frontend drag-and-drop component พร้อม progress bar
- [ ] Temp files cleanup หลัง upload
- [ ] Storage cost monitoring ตั้งค่า R2 billing alerts

---

## 🔗 References

- [Sharp Documentation](https://sharp.pixelplumbing.com/)
- [FFmpeg Documentation](https://ffmpeg.org/documentation.html)
- [Cloudflare R2 Documentation](https://developers.cloudflare.com/r2/)
- [AWS SDK v3 for S3](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/client/s3/)
- [ClamAV Documentation](https://docs.clamav.net/)
- [Multer Documentation](https://github.com/expressjs/multer)

---

*Part 013 | Road to 1,000,000 Users/Day | chuaikan.com*
