# Part 025: Media Service (Upload/Resize/CDN)
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 241-250
> **เวลาโดยประมาณ:** 8 ชั่วโมง
> **Prerequisites:** Part 021 (Microservices), Part 022 (API Gateway), Cloudflare R2 account, Node.js 22

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- ออกแบบ Media Service architecture แบบ Upload → Process → Store → Serve
- Direct upload ไปยัง Cloudflare R2 ด้วย Presigned URLs
- Sharp image processing pipeline: thumbnail, medium, large (WebP)
- Video processing ด้วย FFmpeg: compress, thumbnail, HLS streaming
- BullMQ สำหรับ async media processing
- Blurhash สำหรับ image placeholder
- Storage cost optimization
- Content moderation placeholder
- Media service metrics

---

## 📖 ทฤษฎีและแนวคิด

### Step 241: Media Service Architecture

```
Media Service Flow:
┌─────────────────────────────────────────────────────────────────┐
│                     UPLOAD FLOW                                 │
│                                                                 │
│  1. Client → Media Service: "I want to upload a 5MB image"     │
│  2. Media Service → R2: Generate presigned URL (15 min TTL)    │
│  3. Media Service → Client: Return presigned URL + mediaId     │
│  4. Client → R2 Directly: Upload file to presigned URL         │
│     (ไม่ผ่าน server! ลด bandwidth cost 100%)                  │
│  5. Client → Media Service: "Upload done, process it"          │
│  6. Media Service → BullMQ: Queue processing job               │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                     PROCESSING FLOW                             │
│                                                                 │
│  Worker picks up job:                                           │
│  1. Download original from R2                                  │
│  2. Process with Sharp/FFmpeg:                                 │
│     - Image: resize to 3 variants, convert to WebP             │
│     - Video: compress, extract thumbnail, create HLS           │
│  3. Upload processed variants to R2                            │
│  4. Update database with URLs, dimensions, blurhash            │
│  5. Emit "media.ready" event                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                     SERVE FLOW                                  │
│                                                                 │
│  Client → CDN URL → Cloudflare Cache → R2                      │
│  URL: https://media.chuaikan.com/images/{mediaId}/{variant}    │
│  Cloudflare caches at edge → ultra-low latency worldwide       │
└─────────────────────────────────────────────────────────────────┘

Storage Layout in R2:
  originals/{year}/{month}/{mediaId}.{ext}       ← raw upload
  images/{mediaId}/thumbnail.webp                ← 150px
  images/{mediaId}/medium.webp                   ← 800px
  images/{mediaId}/large.webp                    ← 1600px
  videos/{mediaId}/original.mp4                  ← compressed
  videos/{mediaId}/thumbnail.jpg                 ← video thumb
  videos/{mediaId}/hls/master.m3u8               ← HLS playlist
  videos/{mediaId}/hls/360p.m3u8
  videos/{mediaId}/hls/720p.m3u8
```

### Step 242: Presigned URL Flow Diagram

```
Direct Upload Architecture:

Client                Media Service              Cloudflare R2
  │                       │                           │
  │─── POST /media/upload─►│                           │
  │    (filename, size,    │                           │
  │     mimeType)          │─── generatePresignedURL ──►│
  │                        │◄── presignedUrl (15min) ──│
  │◄── presignedUrl ───────│                           │
  │    + mediaId           │                           │
  │                        │                           │
  │─── PUT presignedUrl ──────────────────────────────►│
  │    (direct upload,     │                           │
  │     no server)         │                           │
  │◄── 200 OK ─────────────────────────────────────────│
  │                        │                           │
  │─── POST /media/confirm─►│                           │
  │    (mediaId)            │─── queue processing job ─►[BullMQ]
  │◄── 202 Accepted ───────│                           │
  │                        │                           │

ข้อดี: ไม่มี bandwidth ผ่าน server เลย!
ลด cost: หาก upload 100GB/วัน ประหยัด ~$9/วัน (vs $0.09/GB egress)
```

---

## ⚙️ Environment Setup

### Step 243: Project Setup

```bash
# สร้าง media service
mkdir -p ~/chuaikan-platform/apps/media-service
cd ~/chuaikan-platform/apps/media-service

# ติดตั้ง FFmpeg บน Ubuntu 24.04
sudo apt-get update
sudo apt-get install -y ffmpeg
ffmpeg -version  # ffmpeg version 6.x

# ตรวจสอบ Sharp รองรับ Ubuntu 24.04
node -e "require('sharp').versions" 2>/dev/null && echo "Sharp OK"

# สร้าง package.json
cat > package.json << 'EOF'
{
  "name": "@chuaikan/media-service",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "test": "vitest run",
    "test:coverage": "vitest run --coverage",
    "db:migrate": "prisma migrate dev",
    "db:generate": "prisma generate"
  },
  "dependencies": {
    "@aws-sdk/client-s3": "^3.620.0",
    "@aws-sdk/s3-request-presigner": "^3.620.0",
    "@chuaikan/shared-types": "workspace:*",
    "@chuaikan/utils": "workspace:*",
    "@prisma/client": "^5.17.0",
    "blurhash": "^2.0.5",
    "bullmq": "^5.12.0",
    "express": "^4.19.0",
    "fluent-ffmpeg": "^2.1.3",
    "ioredis": "^5.4.1",
    "mime-types": "^2.1.35",
    "prom-client": "^15.1.3",
    "sharp": "^0.33.4",
    "uuid": "^10.0.0",
    "winston": "^3.14.0",
    "zod": "^3.23.0"
  },
  "devDependencies": {
    "@types/express": "^4.17.21",
    "@types/fluent-ffmpeg": "^2.1.24",
    "@types/mime-types": "^2.1.4",
    "@types/node": "^22.0.0",
    "@types/uuid": "^10.0.0",
    "@vitest/coverage-v8": "^2.0.0",
    "prisma": "^5.17.0",
    "tsx": "^4.16.0",
    "typescript": "^5.5.0",
    "vitest": "^2.0.0"
  }
}
EOF

mkdir -p src/{config,controllers,queues,workers,services,routes,middleware,__tests__}
mkdir -p /tmp/media-processing  # temporary directory for processing

pnpm install
```

### Step 244: Cloudflare R2 Setup

```bash
# สร้าง R2 bucket ผ่าน Cloudflare Dashboard:
# 1. เข้า https://dash.cloudflare.com
# 2. R2 Object Storage → Create Bucket
# 3. ตั้งชื่อ: chuaikan-media
# 4. Location: Auto (หรือ APAC สำหรับ Thailand)

# สร้าง R2 API Token:
# 1. R2 → Manage R2 API Tokens
# 2. Create API Token: Object Read & Write สำหรับ chuaikan-media

# ตั้งค่า Custom Domain สำหรับ CDN:
# R2 bucket → Settings → Custom Domains → Add Domain
# media.chuaikan.com → enable CDN

# ทดสอบ connection ด้วย AWS CLI (R2 compatible)
sudo apt-get install -y awscli

aws configure set aws_access_key_id "YOUR_R2_ACCESS_KEY_ID"
aws configure set aws_secret_access_key "YOUR_R2_SECRET_ACCESS_KEY"
aws configure set region "auto"

# ทดสอบ upload
echo "test" | aws s3 cp - s3://chuaikan-media/test.txt \
  --endpoint-url "https://YOUR_ACCOUNT_ID.r2.cloudflarestorage.com"

# ตรวจสอบ
aws s3 ls s3://chuaikan-media \
  --endpoint-url "https://YOUR_ACCOUNT_ID.r2.cloudflarestorage.com"
```

---

## 🛠️ Step-by-Step Implementation

### Step 245: R2 Storage Service

```typescript
// apps/media-service/src/config/r2.ts

import {
  S3Client,
  PutObjectCommand,
  GetObjectCommand,
  DeleteObjectCommand,
  HeadObjectCommand,
} from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const r2Client = new S3Client({
  region: 'auto',
  endpoint: `https://${process.env.R2_ACCOUNT_ID}.r2.cloudflarestorage.com`,
  credentials: {
    accessKeyId: process.env.R2_ACCESS_KEY_ID!,
    secretAccessKey: process.env.R2_SECRET_ACCESS_KEY!,
  },
});

const BUCKET = process.env.R2_BUCKET_NAME!;
const CDN_URL = process.env.CDN_URL || `https://media.chuaikan.com`;

export async function generatePresignedUploadUrl(
  key: string,
  contentType: string,
  maxSizeMB: number = 50
): Promise<{ uploadUrl: string; expiresIn: number }> {
  const command = new PutObjectCommand({
    Bucket: BUCKET,
    Key: key,
    ContentType: contentType,
    Metadata: {
      'uploaded-by': 'chuaikan-media-service',
    },
  });

  const uploadUrl = await getSignedUrl(r2Client, command, {
    expiresIn: 900, // 15 minutes
    // Content-Length restriction (ต้อง set ใน client)
  });

  return { uploadUrl, expiresIn: 900 };
}

export async function downloadFromR2(key: string): Promise<Buffer> {
  const command = new GetObjectCommand({ Bucket: BUCKET, Key: key });
  const response = await r2Client.send(command);

  const chunks: Buffer[] = [];
  for await (const chunk of response.Body as any) {
    chunks.push(Buffer.isBuffer(chunk) ? chunk : Buffer.from(chunk));
  }
  return Buffer.concat(chunks);
}

export async function uploadToR2(
  key: string,
  buffer: Buffer,
  contentType: string,
  metadata?: Record<string, string>
): Promise<void> {
  await r2Client.send(
    new PutObjectCommand({
      Bucket: BUCKET,
      Key: key,
      Body: buffer,
      ContentType: contentType,
      Metadata: metadata,
      CacheControl: 'public, max-age=31536000, immutable', // 1 year cache
    })
  );
}

export async function deleteFromR2(key: string): Promise<void> {
  await r2Client.send(new DeleteObjectCommand({ Bucket: BUCKET, Key: key }));
}

export async function checkR2ObjectExists(key: string): Promise<boolean> {
  try {
    await r2Client.send(new HeadObjectCommand({ Bucket: BUCKET, Key: key }));
    return true;
  } catch {
    return false;
  }
}

export function getCdnUrl(key: string): string {
  return `${CDN_URL}/${key}`;
}
```

### Step 246: Sharp Image Processing Pipeline

```typescript
// apps/media-service/src/workers/image.processor.ts

import sharp from 'sharp';
import { encode as blurhashEncode } from 'blurhash';
import { uploadToR2, downloadFromR2 } from '../config/r2';

export interface ImageVariant {
  name: string;
  width: number;
  height?: number;
  quality: number;
}

const IMAGE_VARIANTS: ImageVariant[] = [
  { name: 'thumbnail', width: 150, quality: 80 },
  { name: 'medium', width: 800, quality: 85 },
  { name: 'large', width: 1600, quality: 90 },
];

export interface ProcessedImageResult {
  variants: Array<{
    name: string;
    key: string;
    width: number;
    height: number;
    fileSize: number;
    format: string;
  }>;
  blurHash: string;
  originalWidth: number;
  originalHeight: number;
  dominantColor: string;
}

export async function processImage(
  mediaId: string,
  originalKey: string
): Promise<ProcessedImageResult> {
  // Download original
  const originalBuffer = await downloadFromR2(originalKey);

  // Get metadata
  const metadata = await sharp(originalBuffer).metadata();
  const originalWidth = metadata.width || 0;
  const originalHeight = metadata.height || 0;

  if (originalWidth === 0 || originalHeight === 0) {
    throw new Error('Invalid image: cannot determine dimensions');
  }

  // Process variants in parallel
  const variantResults = await Promise.all(
    IMAGE_VARIANTS.map(async (variant) => {
      const processor = sharp(originalBuffer)
        .rotate() // Auto-rotate based on EXIF
        .resize(variant.width, variant.height, {
          fit: 'inside',            // Maintain aspect ratio
          withoutEnlargement: true, // Don't upscale small images
          kernel: sharp.kernel.lanczos3,
        })
        .webp({
          quality: variant.quality,
          effort: 4,  // 0-6, balance speed/compression
          smartSubsample: true,
        });

      const outputBuffer = await processor.toBuffer({ resolveWithObject: true });
      const { width, height, size } = outputBuffer.info;

      const key = `images/${mediaId}/${variant.name}.webp`;
      await uploadToR2(key, outputBuffer.data, 'image/webp', {
        'media-id': mediaId,
        variant: variant.name,
        'original-format': metadata.format || 'unknown',
      });

      return {
        name: variant.name,
        key,
        width: width || 0,
        height: height || 0,
        fileSize: size,
        format: 'webp',
      };
    })
  );

  // Generate blurhash ใช้ thumbnail (เร็วกว่า process จาก original)
  const blurHash = await generateBlurHash(originalBuffer);

  // Get dominant color
  const dominantColor = await getDominantColor(originalBuffer);

  return {
    variants: variantResults,
    blurHash,
    originalWidth,
    originalHeight,
    dominantColor,
  };
}

async function generateBlurHash(imageBuffer: Buffer): Promise<string> {
  // Resize ลงเป็น 32x32 ก่อน encode blurhash (เร็วมาก)
  const { data, info } = await sharp(imageBuffer)
    .resize(32, 32, { fit: 'fill' })
    .ensureAlpha()
    .raw()
    .toBuffer({ resolveWithObject: true });

  // Convert Buffer to Uint8ClampedArray
  const pixels = new Uint8ClampedArray(data);
  
  // BlurHash encode: componentX=4, componentY=3 (balance detail/size)
  return blurhashEncode(pixels, info.width, info.height, 4, 3);
}

async function getDominantColor(imageBuffer: Buffer): Promise<string> {
  const { dominant } = await sharp(imageBuffer)
    .resize(1, 1, { kernel: sharp.kernel.cubic })
    .raw()
    .toBuffer({ resolveWithObject: true })
    .then(({ data }) => ({
      dominant: `#${data[0].toString(16).padStart(2, '0')}${data[1].toString(16).padStart(2, '0')}${data[2].toString(16).padStart(2, '0')}`,
    }));
  return dominant;
}

// Validate uploaded image
export async function validateImage(buffer: Buffer): Promise<{
  valid: boolean;
  error?: string;
  metadata?: sharp.Metadata;
}> {
  try {
    const metadata = await sharp(buffer).metadata();

    // Check format
    const allowedFormats = ['jpeg', 'png', 'gif', 'webp', 'avif', 'heic'];
    if (!metadata.format || !allowedFormats.includes(metadata.format)) {
      return { valid: false, error: `Unsupported format: ${metadata.format}` };
    }

    // Check dimensions
    if ((metadata.width || 0) > 8000 || (metadata.height || 0) > 8000) {
      return { valid: false, error: 'Image too large: max 8000x8000 pixels' };
    }

    return { valid: true, metadata };
  } catch (error) {
    return { valid: false, error: 'Invalid or corrupt image file' };
  }
}
```

### Step 247: FFmpeg Video Processing

```typescript
// apps/media-service/src/workers/video.processor.ts

import ffmpeg from 'fluent-ffmpeg';
import fs from 'fs';
import path from 'path';
import os from 'os';
import { v4 as uuidv4 } from 'uuid';
import { uploadToR2, downloadFromR2 } from '../config/r2';

const TEMP_DIR = process.env.TEMP_PROCESSING_DIR || os.tmpdir();

export interface VideoProcessingResult {
  compressedKey: string;
  thumbnailKey: string;
  hlsMasterKey?: string;
  duration: number;
  width: number;
  height: number;
  compressedSize: number;
  thumbnailSize: number;
}

export interface VideoQualityLevel {
  name: string;
  height: number;
  bitrate: string;
  audioBitrate: string;
}

const VIDEO_QUALITY_LEVELS: VideoQualityLevel[] = [
  { name: '360p', height: 360, bitrate: '800k', audioBitrate: '64k' },
  { name: '720p', height: 720, bitrate: '2500k', audioBitrate: '128k' },
];

export async function processVideo(
  mediaId: string,
  originalKey: string,
  options: {
    generateHLS?: boolean;
    maxDurationSeconds?: number;
  } = {}
): Promise<VideoProcessingResult> {
  const workDir = path.join(TEMP_DIR, `video-${uuidv4()}`);
  fs.mkdirSync(workDir, { recursive: true });

  try {
    // Download original
    const originalBuffer = await downloadFromR2(originalKey);
    const originalPath = path.join(workDir, 'original');
    fs.writeFileSync(originalPath, originalBuffer);

    // Get video metadata
    const metadata = await getVideoMetadata(originalPath);

    // Check duration limit
    if (options.maxDurationSeconds && metadata.duration > options.maxDurationSeconds) {
      throw new Error(
        `Video too long: ${metadata.duration}s (max: ${options.maxDurationSeconds}s)`
      );
    }

    // Process in parallel
    const [compressedResult, thumbnailResult, hlsResult] = await Promise.all([
      compressVideo(originalPath, workDir, mediaId, metadata),
      extractThumbnail(originalPath, workDir, mediaId, metadata.duration),
      options.generateHLS
        ? generateHLS(originalPath, workDir, mediaId)
        : Promise.resolve(null),
    ]);

    return {
      compressedKey: compressedResult.key,
      thumbnailKey: thumbnailResult.key,
      hlsMasterKey: hlsResult?.masterKey,
      duration: metadata.duration,
      width: metadata.width,
      height: metadata.height,
      compressedSize: compressedResult.size,
      thumbnailSize: thumbnailResult.size,
    };
  } finally {
    // Cleanup temp directory
    fs.rmSync(workDir, { recursive: true, force: true });
  }
}

function getVideoMetadata(filePath: string): Promise<{
  duration: number;
  width: number;
  height: number;
  codec: string;
  bitrate: number;
}> {
  return new Promise((resolve, reject) => {
    ffmpeg.ffprobe(filePath, (err, data) => {
      if (err) return reject(err);

      const videoStream = data.streams.find((s) => s.codec_type === 'video');
      if (!videoStream) return reject(new Error('No video stream found'));

      resolve({
        duration: data.format.duration || 0,
        width: videoStream.width || 0,
        height: videoStream.height || 0,
        codec: videoStream.codec_name || 'unknown',
        bitrate: data.format.bit_rate ? parseInt(data.format.bit_rate) : 0,
      });
    });
  });
}

async function compressVideo(
  inputPath: string,
  workDir: string,
  mediaId: string,
  metadata: { width: number; height: number }
): Promise<{ key: string; size: number }> {
  const outputPath = path.join(workDir, 'compressed.mp4');

  // Target height: max 1080p, maintain aspect ratio
  const targetHeight = Math.min(metadata.height, 1080);
  const targetWidth = Math.round((metadata.width / metadata.height) * targetHeight);
  // Ensure even dimensions (required by H.264)
  const evenWidth = targetWidth % 2 === 0 ? targetWidth : targetWidth - 1;
  const evenHeight = targetHeight % 2 === 0 ? targetHeight : targetHeight - 1;

  await new Promise<void>((resolve, reject) => {
    ffmpeg(inputPath)
      .videoCodec('libx264')
      .size(`${evenWidth}x${evenHeight}`)
      .videoBitrate('2000k')
      .fps(30)
      .audioCodec('aac')
      .audioBitrate('128k')
      .outputOptions([
        '-preset fast',         // Balance speed/quality
        '-crf 23',             // Constant Rate Factor: 18-28 (lower=better)
        '-movflags +faststart', // Enable streaming (moov atom at start)
        '-pix_fmt yuv420p',    // Wide compatibility
        '-profile:v baseline', // Maximum device compatibility
      ])
      .output(outputPath)
      .on('start', (cmd) => console.log('FFmpeg start:', cmd))
      .on('progress', (p) => console.log(`Compression: ${Math.round(p.percent || 0)}%`))
      .on('end', () => resolve())
      .on('error', reject)
      .run();
  });

  const buffer = fs.readFileSync(outputPath);
  const key = `videos/${mediaId}/original.mp4`;
  await uploadToR2(key, buffer, 'video/mp4', { 'media-id': mediaId });

  return { key, size: buffer.length };
}

async function extractThumbnail(
  inputPath: string,
  workDir: string,
  mediaId: string,
  videoDuration: number
): Promise<{ key: string; size: number }> {
  const outputPath = path.join(workDir, 'thumbnail.jpg');
  // Extract frame at 10% of duration (avoid black frames at start)
  const timestamp = Math.min(videoDuration * 0.1, 5);

  await new Promise<void>((resolve, reject) => {
    ffmpeg(inputPath)
      .seekInput(timestamp)
      .frames(1)
      .size('800x?')          // 800px wide, maintain aspect ratio
      .outputOptions([
        '-vf scale=800:-2',   // Scale width to 800, maintain aspect ratio (even height)
        '-q:v 3',             // JPEG quality: 2-31 (lower=better)
      ])
      .output(outputPath)
      .on('end', () => resolve())
      .on('error', reject)
      .run();
  });

  const buffer = fs.readFileSync(outputPath);
  const key = `videos/${mediaId}/thumbnail.jpg`;
  await uploadToR2(key, buffer, 'image/jpeg', { 'media-id': mediaId });

  return { key, size: buffer.length };
}

async function generateHLS(
  inputPath: string,
  workDir: string,
  mediaId: string
): Promise<{ masterKey: string }> {
  const hlsDir = path.join(workDir, 'hls');
  fs.mkdirSync(hlsDir, { recursive: true });

  // Generate HLS for each quality level
  await Promise.all(
    VIDEO_QUALITY_LEVELS.map(
      (level) =>
        new Promise<void>((resolve, reject) => {
          const outputPath = path.join(hlsDir, `${level.name}.m3u8`);

          ffmpeg(inputPath)
            .videoCodec('libx264')
            .size(`?x${level.height}`)
            .videoBitrate(level.bitrate)
            .audioCodec('aac')
            .audioBitrate(level.audioBitrate)
            .outputOptions([
              '-hls_time 6',              // 6-second segments
              '-hls_list_size 0',         // Include all segments
              `-hls_segment_filename ${path.join(hlsDir, `${level.name}_%03d.ts`)}`,
              '-preset fast',
              '-profile:v baseline',
              '-pix_fmt yuv420p',
            ])
            .output(outputPath)
            .on('end', () => resolve())
            .on('error', reject)
            .run();
        })
    )
  );

  // Create master playlist
  const masterPlaylist = VIDEO_QUALITY_LEVELS.map(
    (level) =>
      `#EXT-X-STREAM-INF:BANDWIDTH=${parseInt(level.bitrate) * 1000},RESOLUTION=?x${level.height}\n${level.name}.m3u8`
  ).join('\n');

  const masterContent = `#EXTM3U\n#EXT-X-VERSION:3\n${masterPlaylist}`;
  const masterPath = path.join(hlsDir, 'master.m3u8');
  fs.writeFileSync(masterPath, masterContent);

  // Upload all HLS files to R2
  const hlsFiles = fs.readdirSync(hlsDir);
  await Promise.all(
    hlsFiles.map(async (filename) => {
      const filePath = path.join(hlsDir, filename);
      const buffer = fs.readFileSync(filePath);
      const contentType = filename.endsWith('.m3u8')
        ? 'application/vnd.apple.mpegurl'
        : 'video/mp2t';
      await uploadToR2(`videos/${mediaId}/hls/${filename}`, buffer, contentType);
    })
  );

  return { masterKey: `videos/${mediaId}/hls/master.m3u8` };
}
```

### Step 248: BullMQ Media Processing Queue

```typescript
// apps/media-service/src/queues/media.queue.ts

import { Queue, Worker, Job } from 'bullmq';
import { Redis } from 'ioredis';
import { PrismaClient } from '@prisma/client';
import { processImage } from '../workers/image.processor';
import { processVideo } from '../workers/video.processor';
import { getCdnUrl } from '../config/r2';
import { Histogram, Counter, Gauge } from 'prom-client';

export interface MediaProcessingJob {
  mediaId: string;
  originalKey: string;
  mediaType: 'image' | 'video';
  userId: string;
  options?: {
    generateHLS?: boolean;
    maxDurationSeconds?: number;
    isProfilePicture?: boolean;
  };
}

const redis = new Redis(process.env.REDIS_URL!, { maxRetriesPerRequest: null });

export const mediaQueue = new Queue<MediaProcessingJob>('media:processing', {
  connection: redis,
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: 'exponential', delay: 30000 },  // 30s, 60s, 120s
    removeOnComplete: { count: 1000 },
    removeOnFail: { count: 5000 },
  },
});

// Prometheus metrics
const processingDuration = new Histogram({
  name: 'media_processing_duration_seconds',
  help: 'Media processing time',
  labelNames: ['type', 'variant'],
  buckets: [1, 5, 15, 30, 60, 120, 300],
});

const processedTotal = new Counter({
  name: 'media_processed_total',
  help: 'Total media files processed',
  labelNames: ['type', 'status'],
});

const queueDepthGauge = new Gauge({
  name: 'media_queue_depth',
  help: 'Current media processing queue depth',
});

export function createMediaWorker(prisma: PrismaClient): Worker {
  const worker = new Worker<MediaProcessingJob>(
    'media:processing',
    async (job: Job<MediaProcessingJob>) => {
      const { mediaId, originalKey, mediaType, options } = job.data;
      const startTime = Date.now();

      try {
        // Update status to processing
        await prisma.mediaFile.update({
          where: { id: mediaId },
          data: { status: 'processing', processingStartedAt: new Date() },
        });

        if (mediaType === 'image') {
          const result = await processImage(mediaId, originalKey);

          // Update database with processed variants
          await prisma.mediaFile.update({
            where: { id: mediaId },
            data: {
              status: 'ready',
              width: result.originalWidth,
              height: result.originalHeight,
              blurHash: result.blurHash,
              dominantColor: result.dominantColor,
              processingCompletedAt: new Date(),
              variants: {
                create: result.variants.map((v) => ({
                  name: v.name,
                  url: getCdnUrl(v.key),
                  width: v.width,
                  height: v.height,
                  fileSize: v.fileSize,
                  format: v.format,
                })),
              },
            },
          });

          const duration = (Date.now() - startTime) / 1000;
          processingDuration.labels('image', 'all').observe(duration);
          processedTotal.labels('image', 'success').inc();

        } else if (mediaType === 'video') {
          const result = await processVideo(mediaId, originalKey, {
            generateHLS: options?.generateHLS,
            maxDurationSeconds: options?.maxDurationSeconds || 300,  // 5 min max
          });

          await prisma.mediaFile.update({
            where: { id: mediaId },
            data: {
              status: 'ready',
              width: result.width,
              height: result.height,
              duration: result.duration,
              processingCompletedAt: new Date(),
              variants: {
                create: [
                  {
                    name: 'original',
                    url: getCdnUrl(result.compressedKey),
                    width: result.width,
                    height: result.height,
                    fileSize: result.compressedSize,
                    format: 'mp4',
                  },
                  {
                    name: 'thumbnail',
                    url: getCdnUrl(result.thumbnailKey),
                    width: 800,
                    height: -1,  // calculated
                    fileSize: result.thumbnailSize,
                    format: 'jpeg',
                  },
                  ...(result.hlsMasterKey
                    ? [
                        {
                          name: 'hls_master',
                          url: getCdnUrl(result.hlsMasterKey),
                          width: 0,
                          height: 0,
                          fileSize: 0,
                          format: 'm3u8',
                        },
                      ]
                    : []),
                ],
              },
            },
          });

          const duration = (Date.now() - startTime) / 1000;
          processingDuration.labels('video', 'all').observe(duration);
          processedTotal.labels('video', 'success').inc();
        }

        return { success: true, mediaId };
      } catch (error) {
        await prisma.mediaFile.update({
          where: { id: mediaId },
          data: {
            status: 'failed',
            processingError: (error as Error).message,
          },
        });

        processedTotal.labels(mediaType, 'failed').inc();
        throw error;
      }
    },
    {
      connection: new Redis(process.env.REDIS_URL!, { maxRetriesPerRequest: null }),
      concurrency: 3,  // Process 3 files concurrently (CPU bound)
      // ไม่ตั้ง limiter เพราะ limited by CPU, not API
    }
  );

  // Update queue depth metric periodically
  setInterval(async () => {
    const counts = await mediaQueue.getJobCounts();
    queueDepthGauge.set(counts.waiting + counts.active);
  }, 5000);

  worker.on('completed', (job) => {
    console.log(`Media processed: ${job.data.mediaId}`);
  });

  worker.on('failed', (job, error) => {
    console.error(`Media processing failed: ${job?.data.mediaId}`, error.message);
  });

  return worker;
}
```

### Step 249: Media Service API Controllers

```typescript
// apps/media-service/src/controllers/media.controller.ts

import { Request, Response, NextFunction } from 'express';
import { z } from 'zod';
import { v4 as uuidv4 } from 'uuid';
import mime from 'mime-types';
import { generatePresignedUploadUrl, getCdnUrl } from '../config/r2';
import { mediaQueue } from '../queues/media.queue';
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

const RequestUploadSchema = z.object({
  filename: z.string().min(1).max(255),
  mimeType: z.enum([
    'image/jpeg', 'image/png', 'image/gif', 'image/webp', 'image/avif',
    'video/mp4', 'video/mov', 'video/avi', 'video/webm',
  ]),
  fileSize: z.number().min(1).max(50 * 1024 * 1024),  // max 50MB
  generateHLS: z.boolean().optional(),
});

export class MediaController {
  // Step 1: Request presigned URL
  requestUpload = async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    try {
      const userId = (req as any).user.sub;
      const data = RequestUploadSchema.parse(req.body);
      
      const mediaType = data.mimeType.startsWith('image/') ? 'image' : 'video';
      const mediaId = uuidv4();
      const ext = mime.extension(data.mimeType) || 'bin';
      const year = new Date().getFullYear();
      const month = String(new Date().getMonth() + 1).padStart(2, '0');
      const originalKey = `originals/${year}/${month}/${mediaId}.${ext}`;

      // Generate presigned URL
      const { uploadUrl, expiresIn } = await generatePresignedUploadUrl(
        originalKey,
        data.mimeType,
        data.fileSize / (1024 * 1024)
      );

      // Create media record (status: pending)
      await prisma.mediaFile.create({
        data: {
          id: mediaId,
          userId,
          type: mediaType,
          originalKey,
          mimeType: data.mimeType,
          fileSize: data.fileSize,
          status: 'pending',
        },
      });

      res.json({
        success: true,
        data: {
          mediaId,
          uploadUrl,
          expiresIn,
          method: 'PUT',
          headers: {
            'Content-Type': data.mimeType,
          },
        },
      });
    } catch (error) {
      next(error);
    }
  };

  // Step 2: Confirm upload and start processing
  confirmUpload = async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    try {
      const userId = (req as any).user.sub;
      const { mediaId } = req.params;
      const { generateHLS } = z.object({ generateHLS: z.boolean().optional() }).parse(req.body);

      const media = await prisma.mediaFile.findFirst({
        where: { id: mediaId, userId, status: 'pending' },
      });

      if (!media) {
        res.status(404).json({ success: false, error: { code: 'MEDIA_NOT_FOUND' } });
        return;
      }

      // Queue processing job
      await mediaQueue.add(
        `process-${media.type}`,
        {
          mediaId,
          originalKey: media.originalKey,
          mediaType: media.type as 'image' | 'video',
          userId,
          options: {
            generateHLS: generateHLS && media.type === 'video',
            maxDurationSeconds: 300,
          },
        },
        {
          priority: 3, // Normal priority
          delay: 0,
        }
      );

      // Update status to queued
      await prisma.mediaFile.update({
        where: { id: mediaId },
        data: { status: 'queued' },
      });

      res.status(202).json({
        success: true,
        data: {
          mediaId,
          status: 'queued',
          message: 'Media processing started',
        },
      });
    } catch (error) {
      next(error);
    }
  };

  // Get media status and URLs
  getMedia = async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    try {
      const { mediaId } = req.params;

      const media = await prisma.mediaFile.findUnique({
        where: { id: mediaId },
        include: { variants: true },
      });

      if (!media) {
        res.status(404).json({ success: false, error: { code: 'MEDIA_NOT_FOUND' } });
        return;
      }

      res.json({
        success: true,
        data: {
          id: media.id,
          type: media.type,
          status: media.status,
          mimeType: media.mimeType,
          fileSize: media.fileSize,
          width: media.width,
          height: media.height,
          duration: media.duration,
          blurHash: media.blurHash,
          dominantColor: media.dominantColor,
          variants: media.variants.map((v) => ({
            name: v.name,
            url: v.url,
            width: v.width,
            height: v.height,
            fileSize: v.fileSize,
            format: v.format,
          })),
          createdAt: media.createdAt,
        },
      });
    } catch (error) {
      next(error);
    }
  };

  // Delete media
  deleteMedia = async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    try {
      const userId = (req as any).user.sub;
      const { mediaId } = req.params;

      const media = await prisma.mediaFile.findFirst({
        where: { id: mediaId, userId },
        include: { variants: true },
      });

      if (!media) {
        res.status(404).json({ success: false, error: { code: 'MEDIA_NOT_FOUND' } });
        return;
      }

      // Queue deletion job (async - ลบ R2 objects)
      await mediaQueue.add('delete-media', {
        mediaId,
        originalKey: media.originalKey,
        mediaType: media.type as 'image' | 'video',
        userId,
      });

      // Soft delete in database
      await prisma.mediaFile.update({
        where: { id: mediaId },
        data: { status: 'deleted', deletedAt: new Date() },
      });

      res.json({ success: true });
    } catch (error) {
      next(error);
    }
  };
}
```

---

## 🔧 Configuration Files

### Prisma Schema

```prisma
// apps/media-service/prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model MediaFile {
  id        String   @id @default(uuid())
  userId    String
  type      String   // 'image' | 'video'
  
  originalKey  String  // R2 key of original upload
  mimeType     String
  fileSize     Int     // bytes
  
  // Dimensions
  width   Int?
  height  Int?
  duration Float?  // seconds (video)
  
  // Visual
  blurHash      String?
  dominantColor String?
  
  // Status
  status              String   @default("pending")
  processingError     String?
  processingStartedAt DateTime?
  processingCompletedAt DateTime?
  
  deletedAt DateTime?
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
  
  variants MediaVariant[]
  
  @@index([userId])
  @@index([status])
  @@index([createdAt])
}

model MediaVariant {
  id       String @id @default(cuid())
  mediaId  String
  name     String   // 'thumbnail' | 'medium' | 'large' | 'original' | 'hls_master'
  url      String   // CDN URL
  width    Int
  height   Int
  fileSize Int
  format   String   // 'webp' | 'jpeg' | 'mp4' | 'm3u8'
  
  createdAt DateTime @default(now())
  
  mediaFile MediaFile @relation(fields: [mediaId], references: [id], onDelete: Cascade)
  
  @@unique([mediaId, name])
  @@index([mediaId])
}
```

### Storage Cost Optimization

```typescript
// apps/media-service/src/services/storage-optimizer.ts

interface StorageStats {
  totalOriginals: number;
  totalVariants: number;
  originalSizeGB: number;
  variantSizeGB: number;
  monthlyR2CostUSD: number;
  monthlySavingsFromWebP: number;
}

// Storage Cost Analysis สำหรับ chuaikan.com

/**
 * Cloudflare R2 Pricing (2024):
 * Storage: $0.015/GB/month
 * Class A (PUT): $4.50/million operations
 * Class B (GET): $0.36/million operations
 * Egress: FREE (ข้อดีหลักของ R2)
 *
 * เปรียบเทียบกับ AWS S3:
 * Storage: $0.023/GB/month (+53%)
 * Egress: $0.09/GB (R2 ประหยัดได้มหาศาล)
 */

export function estimateStorageCost(params: {
  dailyUploads: number;
  avgImageSizeMB: number;
  avgVideoSizeMB: number;
  imagePercent: number;  // 0-1
  retentionDays: number;
}): StorageStats {
  const {
    dailyUploads,
    avgImageSizeMB,
    avgVideoSizeMB,
    imagePercent,
    retentionDays,
  } = params;

  const imageUploads = dailyUploads * imagePercent;
  const videoUploads = dailyUploads * (1 - imagePercent);

  // Storage calculation
  const dailyImageStorageGB = (imageUploads * avgImageSizeMB) / 1024;
  const dailyVideoStorageGB = (videoUploads * avgVideoSizeMB) / 1024;

  const totalOriginalStorageGB = (dailyImageStorageGB + dailyVideoStorageGB) * retentionDays;

  // Variants: 3 WebP variants (avg ~60% smaller than original JPEG)
  // thumbnail(10%) + medium(40%) + large(80%) = ~130% of original
  // With WebP: ~130% * 0.4 = 52% of original
  const totalVariantStorageGB = totalOriginalStorageGB * 0.52;

  const totalStorageGB = totalOriginalStorageGB + totalVariantStorageGB;
  const monthlyR2CostUSD = totalStorageGB * 0.015;

  // Savings from WebP vs JPEG
  const savingsPercent = 0.60; // WebP ~60% smaller than JPEG
  const monthlySavingsFromWebP = totalVariantStorageGB * savingsPercent * 0.015;

  return {
    totalOriginals: dailyUploads * retentionDays,
    totalVariants: dailyUploads * retentionDays * 3,  // 3 variants per image
    originalSizeGB: totalOriginalStorageGB,
    variantSizeGB: totalVariantStorageGB,
    monthlyR2CostUSD,
    monthlySavingsFromWebP,
  };
}

// Example: chuaikan.com ที่ 1M users/day
const estimate = estimateStorageCost({
  dailyUploads: 100_000,    // 10% of users upload
  avgImageSizeMB: 3,        // 3MB average JPEG
  avgVideoSizeMB: 50,       // 50MB average video
  imagePercent: 0.85,       // 85% images, 15% videos
  retentionDays: 365,
});

console.log('Storage estimate for 1M users/day:');
console.log(`Total storage: ${(estimate.originalSizeGB + estimate.variantSizeGB).toFixed(1)} GB`);
console.log(`Monthly cost: $${estimate.monthlyR2CostUSD.toFixed(2)}`);
console.log(`WebP savings: $${estimate.monthlySavingsFromWebP.toFixed(2)}/month`);

// Tiered storage: ลบ originals หลัง processing เสร็จ (เหลือแค่ variants)
export function shouldDeleteOriginal(media: any): boolean {
  // ลบ original เมื่อ:
  // 1. Processing เสร็จแล้ว (status = ready)
  // 2. ไม่ใช่ video (video original ต้องเก็บสำหรับ re-processing)
  // 3. ผ่านไปแล้ว 24 ชั่วโมง (safety buffer)
  return (
    media.status === 'ready' &&
    media.type === 'image' &&
    media.processingCompletedAt &&
    Date.now() - media.processingCompletedAt.getTime() > 24 * 60 * 60 * 1000
  );
}
```

### Content Moderation Placeholder

```typescript
// apps/media-service/src/services/moderation.service.ts

export interface ModerationResult {
  approved: boolean;
  confidence: number;
  flags: string[];
  provider: string;
}

/**
 * NSFW Detection Placeholder
 *
 * สำหรับ production ให้ integrate กับ:
 * 1. AWS Rekognition - $1/1000 images
 * 2. Google Vision AI SafeSearch - $1.50/1000 images
 * 3. Clarifai Moderation - $3/1000 images
 * 4. Self-hosted NSFW.js (free, less accurate)
 *
 * chuaikan.com recommendation:
 * - ใช้ AWS Rekognition ที่ scale ได้
 * - ตรวจสอบเฉพาะ public posts (ไม่ตรวจ private)
 * - Manual review queue สำหรับ confidence 40-70%
 */
export async function checkNSFWContent(
  imageBuffer: Buffer
): Promise<ModerationResult> {
  // TODO: Integrate with actual NSFW detection service
  // For now: return approved for all content
  console.warn('NSFW detection not configured - approving all content');

  return {
    approved: true,
    confidence: 0,
    flags: [],
    provider: 'placeholder',
  };
}

// Simple file type validation (security)
export function validateMimeType(buffer: Buffer, declaredMimeType: string): boolean {
  // Check magic bytes to verify actual file type
  const magicBytes = buffer.subarray(0, 12);

  const signatures: Record<string, Buffer | RegExp> = {
    'image/jpeg': Buffer.from([0xff, 0xd8, 0xff]),
    'image/png': Buffer.from([0x89, 0x50, 0x4e, 0x47]),
    'image/gif': Buffer.from([0x47, 0x49, 0x46, 0x38]),
    'image/webp': Buffer.from([0x52, 0x49, 0x46, 0x46]),
    'video/mp4': Buffer.from([0x00, 0x00, 0x00]),  // simplified
  };

  const expectedSignature = signatures[declaredMimeType];
  if (!expectedSignature) return false;

  if (Buffer.isBuffer(expectedSignature)) {
    return magicBytes.subarray(0, expectedSignature.length).equals(expectedSignature);
  }

  return true; // fallback
}
```

---

## 🧪 Testing

```typescript
// apps/media-service/src/__tests__/image.processor.test.ts

import { describe, it, expect, vi, beforeAll } from 'vitest';
import sharp from 'sharp';
import { processImage, validateImage } from '../workers/image.processor';

// Mock R2 functions
vi.mock('../config/r2', () => ({
  downloadFromR2: vi.fn(),
  uploadToR2: vi.fn().mockResolvedValue(undefined),
  getCdnUrl: (key: string) => `https://media.chuaikan.com/${key}`,
}));

async function createTestImage(width: number, height: number): Promise<Buffer> {
  return sharp({
    create: {
      width,
      height,
      channels: 3,
      background: { r: 100, g: 150, b: 200 },
    },
  })
    .jpeg()
    .toBuffer();
}

describe('Image Processor', () => {
  describe('validateImage', () => {
    it('should validate valid JPEG', async () => {
      const buffer = await createTestImage(800, 600);
      const result = await validateImage(buffer);
      
      expect(result.valid).toBe(true);
      expect(result.metadata?.format).toBe('jpeg');
      expect(result.metadata?.width).toBe(800);
    });

    it('should reject oversized image', async () => {
      // Create fake large image
      const fakeBuffer = await sharp({
        create: { width: 9000, height: 9000, channels: 3, background: '#fff' },
      }).jpeg().toBuffer();
      
      const result = await validateImage(fakeBuffer);
      expect(result.valid).toBe(false);
      expect(result.error).toContain('too large');
    });
  });

  describe('processImage', () => {
    it('should create 3 variants with correct sizes', async () => {
      const testImage = await createTestImage(2000, 1500);
      const { downloadFromR2 } = await import('../config/r2');
      (downloadFromR2 as any).mockResolvedValue(testImage);

      const result = await processImage('test-media-id', 'originals/test.jpg');

      expect(result.variants).toHaveLength(3);
      
      const thumbnail = result.variants.find(v => v.name === 'thumbnail');
      const medium = result.variants.find(v => v.name === 'medium');
      
      expect(thumbnail?.width).toBe(150);
      expect(medium?.width).toBe(800);
      
      // All variants should be WebP
      expect(result.variants.every(v => v.format === 'webp')).toBe(true);
    });

    it('should generate valid blurhash', async () => {
      const testImage = await createTestImage(400, 300);
      const { downloadFromR2 } = await import('../config/r2');
      (downloadFromR2 as any).mockResolvedValue(testImage);

      const result = await processImage('test-media-id', 'originals/test.jpg');

      // BlurHash format: {components}{hash}
      expect(result.blurHash).toBeTruthy();
      expect(result.blurHash.length).toBeGreaterThan(10);
    });
  });
});
```

```bash
# Run tests
cd apps/media-service
pnpm test

# Test video processing manually
cat > /tmp/test-video.sh << 'EOF'
#!/bin/bash
# สร้าง test video ด้วย FFmpeg
ffmpeg -f lavfi -i testsrc=duration=5:size=1280x720:rate=30 \
  -f lavfi -i sine=frequency=440:duration=5 \
  -c:v libx264 -c:a aac \
  /tmp/test-video.mp4

# ตรวจสอบ output
ffprobe -v quiet -print_format json -show_format /tmp/test-video.mp4
EOF
bash /tmp/test-video.sh
```

---

## ❌ Common Errors & Solutions

### Error 1: Sharp "Input file is missing" หรือ "Input buffer contains unsupported image format"

```bash
# ตรวจสอบว่า sharp ติดตั้งถูก platform
node -e "console.log(require('sharp').versions)"

# Ubuntu 24.04: ต้องใช้ libvips
sudo apt-get install -y libvips-dev

# Rebuild sharp
cd apps/media-service
npm rebuild sharp --platform=linux --arch=x64

# ตรวจสอบ
node -e "require('sharp')({ create: { width: 10, height: 10, channels: 3, background: '#fff' } }).jpeg().toBuffer().then(() => console.log('Sharp OK'))"
```

### Error 2: FFmpeg "No such file or directory" สำหรับ HLS segments

```bash
# สาเหตุ: output directory ไม่มี
# แก้ไข: สร้าง directory ก่อนเสมอ

mkdir -p /tmp/media-processing/hls-$(uuid)
# ใน code: fs.mkdirSync(hlsDir, { recursive: true })

# ตรวจสอบ FFmpeg path
which ffmpeg
# ถ้าไม่เจอ: sudo apt-get install -y ffmpeg

# Set FFmpeg path ใน code
import ffmpeg from 'fluent-ffmpeg';
ffmpeg.setFfmpegPath('/usr/bin/ffmpeg');
ffmpeg.setFfprobePath('/usr/bin/ffprobe');
```

### Error 3: R2 "SignatureDoesNotMatch" เมื่อ upload

```bash
# สาเหตุ: Clock skew ระหว่าง server กับ R2
# ตรวจสอบ server time:
timedatectl
# ต้อง UTC และ synchronized

# Sync time:
sudo apt-get install -y ntp
sudo systemctl enable ntp
sudo systemctl start ntp

# ตรวจสอบ R2 credentials
aws s3 ls s3://chuaikan-media \
  --endpoint-url https://YOUR_ACCOUNT.r2.cloudflarestorage.com
```

### Error 4: "Worker OOM" เมื่อ process video ขนาดใหญ่

```bash
# สาเหตุ: Video processing ใช้ RAM สูงมาก
# แก้ไข:

# 1. จำกัด video size ก่อน accept
const MAX_VIDEO_SIZE = 100 * 1024 * 1024; // 100MB

# 2. ลด concurrency สำหรับ video worker
concurrency: 1  // process ทีละ 1 video

# 3. เพิ่ม Node.js heap size
NODE_OPTIONS="--max-old-space-size=2048" node dist/index.js

# 4. Monitor memory
docker stats chuaikan-media-service
```

---

## ✅ Checklist

### Step 241-242: Architecture Design
- [ ] เข้าใจ upload → process → store → serve flow
- [ ] วาง Presigned URL flow
- [ ] กำหนด storage layout ใน R2

### Step 243-244: Setup
- [ ] FFmpeg ติดตั้งสำเร็จ (`ffmpeg -version`)
- [ ] Sharp ติดตั้งสำเร็จ
- [ ] R2 bucket สร้างแล้ว
- [ ] Presigned URL generate สำเร็จ

### Step 245-246: Image Processing
- [ ] Sharp resize thumbnail (150px) สำเร็จ
- [ ] Sharp resize medium (800px) สำเร็จ
- [ ] Sharp resize large (1600px) สำเร็จ
- [ ] WebP conversion ทำงาน (file size ลด >40%)
- [ ] Blurhash generate ถูกต้อง

### Step 247: Video Processing
- [ ] FFmpeg compress video สำเร็จ
- [ ] Extract thumbnail สำเร็จ
- [ ] HLS generate (optional)

### Step 248-249: Queue & API
- [ ] BullMQ queue สร้างสำเร็จ
- [ ] Worker process jobs
- [ ] POST /media/upload-request สร้าง presigned URL
- [ ] POST /media/:id/confirm queue processing job
- [ ] GET /media/:id return status และ URLs

### Step 250: Monitoring & Optimization
- [ ] Prometheus metrics endpoint ทำงาน
- [ ] Queue depth metric แสดงใน Grafana
- [ ] Processing time metric แสดงใน Grafana
- [ ] Storage cost calculation สมเหตุสมผล
- [ ] WebP compression ratio >40% verified
- [ ] Original file deletion (tiered storage) ทำงาน

---

## 🔗 References

- [Sharp Documentation](https://sharp.pixelplumbing.com/)
- [FFmpeg Documentation](https://ffmpeg.org/documentation.html)
- [Cloudflare R2](https://developers.cloudflare.com/r2/)
- [Blurhash](https://blurha.sh/)
- [BullMQ](https://docs.bullmq.io/)
- [HLS Streaming](https://developer.apple.com/streaming/)
- [WebP Compression](https://developers.google.com/speed/webp)

---
*Part 025 | Road to 1,000,000 Users/Day | chuaikan.com*
