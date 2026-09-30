# Part 014: Email & SMS Notification System

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 131-140
> **เวลาโดยประมาณ:** 8 ชั่วโมง
> **Prerequisites:** Part 013 (File Upload), Part 012 (WebSocket)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

1. ตั้งค่า Email Service ด้วย Resend.com (แนะนำ) และ Nodemailer
2. สร้าง Email Templates ด้วย React Email
3. สร้าง Transactional Emails: Welcome, Password Reset, SOS Alert
4. ส่ง SMS ด้วย Twilio (รองรับ Thailand +66)
5. ส่ง SMS สำหรับ SOS Emergency Alerts
6. ตั้งค่า Push Notifications ด้วย Firebase Cloud Messaging (FCM)
7. สร้าง Web Push Notifications (Service Worker + Push API)
8. ออกแบบ Notification Preferences Table ใน PostgreSQL
9. ประมวลผล Notifications ด้วย Queue (BullMQ + Redis)
10. Rate Limiting, Batching, Unsubscribe, และ Localization

---

## 📖 ทฤษฎีและแนวคิด

### Notification System Architecture

```
                        ┌────────────────────┐
                        │  chuaikan.com API  │
                        │  (Event Publisher) │
                        └────────┬───────────┘
                                 │
                    emit('notification.send')
                                 │
                        ┌────────▼───────────┐
                        │   BullMQ Queue     │
                        │   (Redis-backed)   │
                        └────────┬───────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
     ┌────────▼──────┐  ┌────────▼──────┐  ┌───────▼───────┐
     │  Email Worker │  │  SMS Worker   │  │  Push Worker  │
     │  (Resend)     │  │  (Twilio)     │  │  (FCM)        │
     └───────────────┘  └───────────────┘  └───────────────┘

Notification Channels:
┌─────────────────────────────────────────────────┐
│  Channel    │ Use Case              │ Speed      │
├─────────────┼───────────────────────┼────────────│
│ Email       │ Welcome, Receipt      │ Slow (>5s) │
│ SMS         │ SOS Alert, OTP        │ Fast (~1s) │
│ Push (FCM)  │ New post, Like, Chat  │ Instant    │
│ Web Push    │ Browser notifications │ Instant    │
│ In-app      │ Real-time (WebSocket) │ Instant    │
└─────────────────────────────────────────────────┘
```

### Notification Priority Matrix

```
                HIGH URGENCY        LOW URGENCY
                ┌────────────────────────────────┐
HIGH            │ SOS Alert: SMS+Push+Email+WS   │
IMPORTANCE      ├────────────────────────────────┤
                │ OTP: SMS only                  │
                ├────────────────────────────────┤
LOW             │ New follower: Push+In-app      │
IMPORTANCE      ├────────────────────────────────┤
                │ Weekly digest: Email only      │
                └────────────────────────────────┘
```

---

## ⚙️ Environment Setup

### Step 131: ติดตั้ง Dependencies

```bash
# ไปที่ project directory
cd /home/user/chuaikan-api

# Email packages
npm install resend@3.3.0 \
  nodemailer@6.9.14 \
  @react-email/components@0.0.22 \
  react@18.3.1 \
  react-dom@18.3.1

# SMS
npm install twilio@5.1.0

# Push notifications
npm install firebase-admin@12.2.0 \
  web-push@3.6.7

# Queue
npm install bullmq@5.8.1 \
  ioredis@5.4.1

# Templates
npm install handlebars@4.7.8 \
  nodemailer-express-handlebars@7.0.0

# Type definitions
npm install -D \
  @types/nodemailer@6.4.15 \
  @types/web-push@3.6.3

# React Email rendering
npm install @react-email/render@0.0.17
```

### Step 132: ตั้งค่า Environment Variables

```bash
cat >> /home/user/chuaikan-api/.env << 'EOF'
# Email - Resend (Primary)
RESEND_API_KEY=re_xxxxxxxxxxxxxxxxxxxx
EMAIL_FROM=noreply@chuaikan.com
EMAIL_FROM_NAME=chuaikan

# Email - Nodemailer (Fallback)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your@gmail.com
SMTP_PASS=your_app_password

# SMS - Twilio
TWILIO_ACCOUNT_SID=ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
TWILIO_AUTH_TOKEN=your_auth_token
TWILIO_PHONE_NUMBER=+15551234567
TWILIO_MESSAGING_SERVICE_SID=MGxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Firebase FCM
FIREBASE_PROJECT_ID=chuaikan-app
FIREBASE_CLIENT_EMAIL=firebase-adminsdk@chuaikan-app.iam.gserviceaccount.com
FIREBASE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n"

# Web Push VAPID
VAPID_PUBLIC_KEY=your_vapid_public_key
VAPID_PRIVATE_KEY=your_vapid_private_key
VAPID_SUBJECT=mailto:admin@chuaikan.com

# Notification limits
MAX_SMS_PER_USER_PER_DAY=5
MAX_EMAIL_PER_USER_PER_DAY=20
MAX_PUSH_PER_USER_PER_DAY=50
EOF
```

### Generate VAPID Keys

```bash
# สร้าง VAPID keys สำหรับ Web Push
node -e "
const webpush = require('web-push');
const vapidKeys = webpush.generateVAPIDKeys();
console.log('Public Key:', vapidKeys.publicKey);
console.log('Private Key:', vapidKeys.privateKey);
"
```

---

## 🛠️ Step-by-Step Implementation

### Step 133: React Email Templates

สร้างไฟล์ `/home/user/chuaikan-api/src/notifications/templates/WelcomeEmail.tsx`:

```tsx
import {
  Html,
  Head,
  Body,
  Container,
  Heading,
  Text,
  Button,
  Img,
  Hr,
  Preview,
  Tailwind,
  Section,
  Row,
  Column,
} from '@react-email/components';
import * as React from 'react';

interface WelcomeEmailProps {
  username: string;
  verificationUrl: string;
  language?: 'th' | 'en';
}

const content = {
  th: {
    preview: 'ยินดีต้อนรับสู่ chuaikan!',
    heading: 'ยินดีต้อนรับสู่ chuaikan! 🎉',
    greeting: (name: string) => `สวัสดี ${name}`,
    body: 'ขอบคุณที่สมัครสมาชิก chuaikan — แพลตฟอร์มชุมชนที่เชื่อมต่อคุณกับผู้คนในพื้นที่',
    ctaText: 'ยืนยันอีเมล',
    expiry: 'ลิงก์นี้จะหมดอายุภายใน 24 ชั่วโมง',
    footer: 'หากคุณไม่ได้สมัครสมาชิก กรุณาละเว้นอีเมลนี้',
  },
  en: {
    preview: 'Welcome to chuaikan!',
    heading: 'Welcome to chuaikan! 🎉',
    greeting: (name: string) => `Hello ${name}`,
    body: 'Thank you for joining chuaikan — the community platform connecting you with people nearby.',
    ctaText: 'Verify Email',
    expiry: 'This link expires in 24 hours.',
    footer: 'If you did not create an account, please ignore this email.',
  },
};

export const WelcomeEmail: React.FC<WelcomeEmailProps> = ({
  username,
  verificationUrl,
  language = 'th',
}) => {
  const t = content[language];

  return (
    <Html lang={language}>
      <Head />
      <Preview>{t.preview}</Preview>
      <Tailwind>
        <Body className="bg-gray-50 font-sans">
          <Container className="mx-auto my-10 max-w-[600px] rounded-lg bg-white p-8 shadow-sm">
            {/* Logo */}
            <Section className="text-center mb-6">
              <Img
                src="https://chuaikan.com/images/logo.png"
                width={120}
                height={40}
                alt="chuaikan"
                className="mx-auto"
              />
            </Section>

            <Hr className="border-gray-200 my-6" />

            {/* Greeting */}
            <Heading className="text-2xl font-bold text-gray-900 text-center">
              {t.heading}
            </Heading>

            <Text className="text-gray-600 text-base leading-relaxed mt-4">
              {t.greeting(username)},
            </Text>

            <Text className="text-gray-600 text-base leading-relaxed">
              {t.body}
            </Text>

            {/* CTA Button */}
            <Section className="text-center my-8">
              <Button
                href={verificationUrl}
                className="rounded-lg bg-indigo-600 px-8 py-4 text-white font-semibold text-base no-underline inline-block"
              >
                {t.ctaText}
              </Button>
            </Section>

            <Text className="text-gray-400 text-sm text-center">
              {t.expiry}
            </Text>

            <Hr className="border-gray-200 my-6" />

            {/* Footer */}
            <Text className="text-gray-400 text-xs text-center">
              {t.footer}
            </Text>
            <Text className="text-gray-400 text-xs text-center">
              © 2025 chuaikan.com | Bangkok, Thailand
            </Text>
          </Container>
        </Body>
      </Tailwind>
    </Html>
  );
};

export default WelcomeEmail;
```

สร้างไฟล์ `/home/user/chuaikan-api/src/notifications/templates/SOSAlertEmail.tsx`:

```tsx
import {
  Html, Head, Body, Container, Heading, Text,
  Button, Hr, Preview, Section,
} from '@react-email/components';
import * as React from 'react';

interface SOSAlertEmailProps {
  recipientName: string;
  sosType: string;
  sosAddress: string;
  sosSeverity: 'low' | 'medium' | 'high' | 'critical';
  reporterName: string;
  sosUrl: string;
  mapImageUrl?: string;
}

const severityColors: Record<string, string> = {
  low: '#22c55e',
  medium: '#f59e0b',
  high: '#ef4444',
  critical: '#7c3aed',
};

const severityLabels: Record<string, string> = {
  low: 'ความเสี่ยงต่ำ',
  medium: 'ความเสี่ยงปานกลาง',
  high: 'ความเสี่ยงสูง',
  critical: 'วิกฤต - ต้องการความช่วยเหลือด่วน',
};

export const SOSAlertEmail: React.FC<SOSAlertEmailProps> = ({
  recipientName,
  sosType,
  sosAddress,
  sosSeverity,
  reporterName,
  sosUrl,
}) => {
  const color = severityColors[sosSeverity];
  const label = severityLabels[sosSeverity];

  return (
    <Html lang="th">
      <Head />
      <Preview>🆘 แจ้งเหตุ SOS: {sosType} บริเวณ {sosAddress}</Preview>
      <Body style={{ backgroundColor: '#f3f4f6', fontFamily: 'Arial, sans-serif' }}>
        <Container style={{ maxWidth: '600px', margin: '0 auto', padding: '20px' }}>
          {/* Alert Banner */}
          <Section
            style={{
              backgroundColor: color,
              borderRadius: '8px 8px 0 0',
              padding: '20px',
              textAlign: 'center',
            }}
          >
            <Text style={{ color: '#fff', fontSize: '32px', margin: 0 }}>🆘</Text>
            <Heading style={{ color: '#fff', margin: '8px 0 0' }}>
              แจ้งเหตุฉุกเฉิน SOS
            </Heading>
            <Text
              style={{
                color: '#fff',
                backgroundColor: 'rgba(0,0,0,0.2)',
                padding: '4px 12px',
                borderRadius: '20px',
                display: 'inline-block',
                margin: '8px 0 0',
              }}
            >
              {label}
            </Text>
          </Section>

          {/* Content */}
          <Section
            style={{
              backgroundColor: '#fff',
              padding: '24px',
              borderRadius: '0 0 8px 8px',
            }}
          >
            <Text>สวัสดี {recipientName},</Text>
            <Text>
              มีการแจ้งเหตุฉุกเฉิน SOS ในพื้นที่ใกล้เคียงที่คุณสนใจ:
            </Text>

            {/* SOS Details */}
            <Section
              style={{
                backgroundColor: '#f9fafb',
                borderRadius: '8px',
                padding: '16px',
                marginTop: '16px',
              }}
            >
              <Text style={{ margin: '4px 0' }}>
                <strong>ประเภท:</strong> {sosType}
              </Text>
              <Text style={{ margin: '4px 0' }}>
                <strong>สถานที่:</strong> {sosAddress}
              </Text>
              <Text style={{ margin: '4px 0' }}>
                <strong>รายงานโดย:</strong> {reporterName}
              </Text>
              <Text style={{ margin: '4px 0' }}>
                <strong>เวลา:</strong> {new Date().toLocaleString('th-TH')}
              </Text>
            </Section>

            <Section style={{ textAlign: 'center', marginTop: '24px' }}>
              <Button
                href={sosUrl}
                style={{
                  backgroundColor: color,
                  color: '#fff',
                  padding: '12px 32px',
                  borderRadius: '8px',
                  fontWeight: 'bold',
                  textDecoration: 'none',
                  display: 'inline-block',
                }}
              >
                ดูรายละเอียดและเสนอความช่วยเหลือ
              </Button>
            </Section>

            <Hr />
            <Text style={{ color: '#9ca3af', fontSize: '12px' }}>
              คุณได้รับอีเมลนี้เพราะสมัครรับการแจ้งเตือน SOS ในพื้นที่ของคุณ
              <br />
              จัดการการแจ้งเตือน:{' '}
              <a href="https://chuaikan.com/settings/notifications">
                chuaikan.com/settings/notifications
              </a>
            </Text>
          </Section>
        </Container>
      </Body>
    </Html>
  );
};
```

### Step 134: Email Service (Resend + Nodemailer fallback)

สร้างไฟล์ `/home/user/chuaikan-api/src/notifications/emailService.ts`:

```typescript
import { Resend } from 'resend';
import nodemailer from 'nodemailer';
import { render } from '@react-email/render';
import * as React from 'react';
import { WelcomeEmail } from './templates/WelcomeEmail';
import { SOSAlertEmail } from './templates/SOSAlertEmail';

const resend = new Resend(process.env.RESEND_API_KEY!);

// Nodemailer transporter (fallback)
const fallbackTransporter = nodemailer.createTransport({
  host: process.env.SMTP_HOST,
  port: parseInt(process.env.SMTP_PORT || '587'),
  secure: false,
  auth: {
    user: process.env.SMTP_USER,
    pass: process.env.SMTP_PASS,
  },
});

interface SendEmailOptions {
  to: string | string[];
  subject: string;
  html: string;
  text?: string;
  replyTo?: string;
  tags?: Array<{ name: string; value: string }>;
}

// ส่ง email ผ่าน Resend (ด้วย fallback ไปยัง Nodemailer)
export async function sendEmail(options: SendEmailOptions): Promise<boolean> {
  const from = `${process.env.EMAIL_FROM_NAME} <${process.env.EMAIL_FROM}>`;

  try {
    // ลองส่งผ่าน Resend ก่อน
    const result = await resend.emails.send({
      from,
      to: Array.isArray(options.to) ? options.to : [options.to],
      subject: options.subject,
      html: options.html,
      text: options.text,
      replyTo: options.replyTo,
      tags: options.tags,
    });

    if (result.error) {
      throw new Error(result.error.message);
    }

    console.log(`Email sent via Resend: ${result.data?.id}`);
    return true;

  } catch (primaryError) {
    console.error('Resend failed, trying Nodemailer:', primaryError);

    try {
      // Fallback ไปยัง Nodemailer
      const info = await fallbackTransporter.sendMail({
        from,
        to: Array.isArray(options.to) ? options.to.join(',') : options.to,
        subject: options.subject,
        html: options.html,
        text: options.text,
        replyTo: options.replyTo,
      });

      console.log(`Email sent via Nodemailer: ${info.messageId}`);
      return true;

    } catch (fallbackError) {
      console.error('Both email providers failed:', fallbackError);
      return false;
    }
  }
}

// ============================================
// Transactional Email Functions
// ============================================

// Welcome Email
export async function sendWelcomeEmail(
  to: string,
  username: string,
  verificationToken: string,
  language: 'th' | 'en' = 'th'
): Promise<boolean> {
  const verificationUrl =
    `https://chuaikan.com/verify-email?token=${verificationToken}`;

  const html = await render(
    React.createElement(WelcomeEmail, { username, verificationUrl, language })
  );

  return sendEmail({
    to,
    subject: language === 'th'
      ? 'ยืนยันอีเมลของคุณ — chuaikan'
      : 'Verify your email — chuaikan',
    html,
    tags: [
      { name: 'category', value: 'transactional' },
      { name: 'type', value: 'welcome' },
    ],
  });
}

// Password Reset Email
export async function sendPasswordResetEmail(
  to: string,
  username: string,
  resetToken: string
): Promise<boolean> {
  const resetUrl = `https://chuaikan.com/reset-password?token=${resetToken}`;

  const html = `
    <!DOCTYPE html>
    <html lang="th">
    <head><meta charset="UTF-8"><meta name="viewport" content="width=device-width, initial-scale=1.0"></head>
    <body style="font-family: Arial, sans-serif; background: #f3f4f6; padding: 20px;">
      <div style="max-width: 600px; margin: 0 auto; background: #fff; border-radius: 8px; padding: 32px;">
        <h2 style="color: #1f2937;">รีเซ็ตรหัสผ่าน</h2>
        <p>สวัสดี ${username},</p>
        <p>คุณได้ขอรีเซ็ตรหัสผ่านสำหรับ chuaikan กรุณาคลิกปุ่มด้านล่างเพื่อตั้งรหัสผ่านใหม่:</p>
        <div style="text-align: center; margin: 32px 0;">
          <a href="${resetUrl}"
             style="background: #4f46e5; color: #fff; padding: 12px 32px; border-radius: 8px;
                    text-decoration: none; font-weight: bold; display: inline-block;">
            รีเซ็ตรหัสผ่าน
          </a>
        </div>
        <p style="color: #6b7280; font-size: 14px;">ลิงก์นี้จะหมดอายุภายใน 1 ชั่วโมง</p>
        <p style="color: #6b7280; font-size: 14px;">หากคุณไม่ได้ขอรีเซ็ตรหัสผ่าน กรุณาละเว้นอีเมลนี้</p>
      </div>
    </body>
    </html>
  `;

  return sendEmail({
    to,
    subject: 'รีเซ็ตรหัสผ่าน — chuaikan',
    html,
    tags: [{ name: 'type', value: 'password_reset' }],
  });
}

// SOS Alert Email
export async function sendSOSAlertEmail(params: {
  to: string;
  recipientName: string;
  sosType: string;
  sosAddress: string;
  sosSeverity: 'low' | 'medium' | 'high' | 'critical';
  reporterName: string;
  sosId: string;
}): Promise<boolean> {
  const html = await render(
    React.createElement(SOSAlertEmail, {
      recipientName: params.recipientName,
      sosType: params.sosType,
      sosAddress: params.sosAddress,
      sosSeverity: params.sosSeverity,
      reporterName: params.reporterName,
      sosUrl: `https://chuaikan.com/sos/${params.sosId}`,
    })
  );

  return sendEmail({
    to: params.to,
    subject: `🆘 แจ้งเหตุ SOS: ${params.sosType} บริเวณ ${params.sosAddress}`,
    html,
    tags: [
      { name: 'type', value: 'sos_alert' },
      { name: 'severity', value: params.sosSeverity },
    ],
  });
}
```

### Step 135: SMS Service ด้วย Twilio

สร้างไฟล์ `/home/user/chuaikan-api/src/notifications/smsService.ts`:

```typescript
import twilio from 'twilio';
import { createClient } from 'redis';

const twilioClient = twilio(
  process.env.TWILIO_ACCOUNT_SID!,
  process.env.TWILIO_AUTH_TOKEN!
);

const redisClient = createClient({
  url: process.env.REDIS_URL || 'redis://localhost:6379',
});
redisClient.connect();

const MAX_SMS_PER_DAY = parseInt(process.env.MAX_SMS_PER_USER_PER_DAY || '5');

// ตรวจสอบ rate limit
async function checkSMSRateLimit(userId: string): Promise<boolean> {
  const key = `sms_count:${userId}:${new Date().toISOString().split('T')[0]}`;
  
  const current = await redisClient.get(key);
  const count = current ? parseInt(current) : 0;

  if (count >= MAX_SMS_PER_DAY) {
    console.warn(`SMS rate limit reached for user ${userId}: ${count}/${MAX_SMS_PER_DAY}`);
    return false;
  }

  // Increment counter (expire at midnight)
  const now = new Date();
  const midnight = new Date(now);
  midnight.setHours(24, 0, 0, 0);
  const secondsUntilMidnight = Math.floor((midnight.getTime() - now.getTime()) / 1000);

  await redisClient.set(key, (count + 1).toString(), { EX: secondsUntilMidnight });
  return true;
}

interface SMSOptions {
  to: string;        // Phone number เช่น +66812345678
  message: string;
  userId?: string;   // สำหรับ rate limiting
}

// ส่ง SMS ทั่วไป
export async function sendSMS(options: SMSOptions): Promise<boolean> {
  const { to, message, userId } = options;

  // Validate Thailand phone number
  const normalizedPhone = normalizeThaiPhone(to);
  if (!normalizedPhone) {
    console.error(`Invalid phone number: ${to}`);
    return false;
  }

  // Rate limit check
  if (userId) {
    const allowed = await checkSMSRateLimit(userId);
    if (!allowed) return false;
  }

  try {
    const result = await twilioClient.messages.create({
      body: message,
      from: process.env.TWILIO_PHONE_NUMBER!,
      to: normalizedPhone,
      // ใช้ Messaging Service SID สำหรับ better deliverability
      // messagingServiceSid: process.env.TWILIO_MESSAGING_SERVICE_SID,
    });

    console.log(`SMS sent: ${result.sid} → ${normalizedPhone}`);
    return true;

  } catch (error: any) {
    console.error(`SMS failed to ${normalizedPhone}:`, error.message);
    return false;
  }
}

// Normalize phone number สำหรับ Thailand
function normalizeThaiPhone(phone: string): string | null {
  // Remove spaces, dashes, etc.
  const cleaned = phone.replace(/[\s\-\(\)]/g, '');

  // +66812345678 → ถูกต้องแล้ว
  if (/^\+66\d{9}$/.test(cleaned)) return cleaned;

  // 0812345678 → +66812345678
  if (/^0[6-9]\d{8}$/.test(cleaned)) {
    return '+66' + cleaned.slice(1);
  }

  // 66812345678 → +66812345678
  if (/^66[6-9]\d{8}$/.test(cleaned)) {
    return '+' + cleaned;
  }

  return null;
}

// ============================================
// SMS Templates
// ============================================

// OTP SMS
export async function sendOTPSMS(
  to: string,
  otp: string,
  userId: string
): Promise<boolean> {
  const message = `รหัส OTP ของคุณสำหรับ chuaikan คือ: ${otp}\nหมดอายุใน 5 นาที\nอย่าแชร์รหัสนี้กับผู้อื่น`;

  return sendSMS({ to, message, userId });
}

// SOS Emergency SMS
export async function sendSOSEmergencySMS(
  to: string,
  sosType: string,
  sosAddress: string,
  sosId: string
): Promise<boolean> {
  // SOS alerts ไม่มี rate limit (emergency)
  const message =
    `🆘 chuaikan SOS Alert!\n` +
    `ประเภท: ${sosType}\n` +
    `สถานที่: ${sosAddress}\n` +
    `ดูรายละเอียด: https://chuaikan.com/sos/${sosId}`;

  try {
    const result = await twilioClient.messages.create({
      body: message,
      from: process.env.TWILIO_PHONE_NUMBER!,
      to: normalizeThaiPhone(to) || to,
    });

    console.log(`SOS SMS sent: ${result.sid}`);
    return true;
  } catch (error: any) {
    console.error('SOS SMS failed:', error.message);
    return false;
  }
}

// Digest SMS (สรุปรายสัปดาห์)
export async function sendWeeklyDigestSMS(
  to: string,
  userId: string,
  stats: { posts: number; likes: number; followers: number }
): Promise<boolean> {
  const message =
    `📊 chuaikan สัปดาห์นี้:\n` +
    `โพสต์: ${stats.posts} | ถูกใจ: ${stats.likes} | ผู้ติดตามใหม่: ${stats.followers}\n` +
    `chuaikan.com`;

  return sendSMS({ to, message, userId });
}
```

### Step 136: Firebase Cloud Messaging (FCM)

สร้างไฟล์ `/home/user/chuaikan-api/src/notifications/fcmService.ts`:

```typescript
import admin from 'firebase-admin';
import { Message, MulticastMessage } from 'firebase-admin/messaging';

// Initialize Firebase Admin
if (!admin.apps.length) {
  admin.initializeApp({
    credential: admin.credential.cert({
      projectId: process.env.FIREBASE_PROJECT_ID,
      clientEmail: process.env.FIREBASE_CLIENT_EMAIL,
      privateKey: process.env.FIREBASE_PRIVATE_KEY?.replace(/\\n/g, '\n'),
    }),
  });
}

const messaging = admin.messaging();

interface FCMPayload {
  title: string;
  body: string;
  imageUrl?: string;
  data?: Record<string, string>;
  clickAction?: string;
}

// ส่ง push notification ไปยัง token เดียว
export async function sendPushToToken(
  fcmToken: string,
  payload: FCMPayload
): Promise<boolean> {
  const message: Message = {
    token: fcmToken,
    notification: {
      title: payload.title,
      body: payload.body,
      imageUrl: payload.imageUrl,
    },
    data: {
      ...payload.data,
      clickAction: payload.clickAction || 'OPEN_APP',
    },
    android: {
      notification: {
        sound: 'default',
        priority: 'high',
        channelId: 'chuaikan_notifications',
        clickAction: 'OPEN_APP',
      },
    },
    apns: {
      payload: {
        aps: {
          sound: 'default',
          badge: 1,
        },
      },
    },
    webpush: {
      notification: {
        icon: '/icons/icon-192x192.png',
        badge: '/icons/badge-72x72.png',
        vibrate: [200, 100, 200],
        requireInteraction: payload.data?.type === 'sos_alert',
      },
      fcmOptions: {
        link: payload.clickAction || 'https://chuaikan.com',
      },
    },
  };

  try {
    const response = await messaging.send(message);
    console.log(`FCM sent: ${response}`);
    return true;
  } catch (error: any) {
    if (error.code === 'messaging/registration-token-not-registered') {
      // Token ไม่ valid → ลบออกจาก database
      console.log(`Invalid FCM token: ${fcmToken}`);
      await removeInvalidFCMToken(fcmToken);
    }
    console.error('FCM error:', error.message);
    return false;
  }
}

// ส่ง push notification ไปยังหลาย tokens พร้อมกัน (max 500 tokens)
export async function sendPushToMultipleTokens(
  tokens: string[],
  payload: FCMPayload
): Promise<{ success: number; failure: number }> {
  if (tokens.length === 0) return { success: 0, failure: 0 };

  // FCM รองรับ max 500 tokens ต่อ request
  const batchSize = 500;
  let totalSuccess = 0;
  let totalFailure = 0;

  for (let i = 0; i < tokens.length; i += batchSize) {
    const batch = tokens.slice(i, i + batchSize);

    const message: MulticastMessage = {
      tokens: batch,
      notification: {
        title: payload.title,
        body: payload.body,
        imageUrl: payload.imageUrl,
      },
      data: {
        ...payload.data,
        clickAction: payload.clickAction || 'OPEN_APP',
      },
    };

    const response = await messaging.sendEachForMulticast(message);

    totalSuccess += response.successCount;
    totalFailure += response.failureCount;

    // ลบ invalid tokens
    response.responses.forEach((resp, index) => {
      if (!resp.success && resp.error?.code === 'messaging/registration-token-not-registered') {
        removeInvalidFCMToken(batch[index]);
      }
    });
  }

  return { success: totalSuccess, failure: totalFailure };
}

// ส่ง SOS alert push notification (priority: critical)
export async function sendSOSPushNotification(
  tokens: string[],
  sosData: {
    sosId: string;
    type: string;
    severity: string;
    address: string;
  }
): Promise<void> {
  const payload: FCMPayload = {
    title: `🆘 SOS Alert: ${sosData.type}`,
    body: `เหตุการณ์: ${sosData.severity} ที่ ${sosData.address}`,
    data: {
      type: 'sos_alert',
      sosId: sosData.sosId,
      severity: sosData.severity,
    },
    clickAction: `https://chuaikan.com/sos/${sosData.sosId}`,
  };

  await sendPushToMultipleTokens(tokens, payload);
}

async function removeInvalidFCMToken(token: string): Promise<void> {
  // TODO: ลบ FCM token ออกจาก database
  console.log(`TODO: Remove invalid FCM token: ${token.substring(0, 20)}...`);
}
```

### Step 137: BullMQ Queue สำหรับ Notification Processing

สร้างไฟล์ `/home/user/chuaikan-api/src/notifications/notificationQueue.ts`:

```typescript
import { Queue, Worker, QueueEvents, Job } from 'bullmq';
import Redis from 'ioredis';
import { sendEmail, sendWelcomeEmail, sendSOSAlertEmail } from './emailService';
import { sendSMS, sendSOSEmergencySMS } from './smsService';
import { sendPushToToken, sendSOSPushNotification } from './fcmService';

// Redis connection
const redisConnection = new Redis(process.env.REDIS_URL || 'redis://localhost:6379', {
  maxRetriesPerRequest: null,  // ต้องใช้ null สำหรับ BullMQ
});

// Queue definitions
export const notificationQueue = new Queue('notifications', {
  connection: redisConnection,
  defaultJobOptions: {
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 2000,  // เริ่มที่ 2 วินาที, ลองซ้ำที่ 2s, 4s, 8s
    },
    removeOnComplete: { age: 3600, count: 1000 },  // เก็บ completed jobs 1 ชั่วโมง
    removeOnFail: { age: 86400, count: 500 },       // เก็บ failed jobs 24 ชั่วโมง
  },
});

// Job types
export type NotificationJobType =
  | 'send_email'
  | 'send_sms'
  | 'send_push'
  | 'send_sos_alert'
  | 'send_welcome'
  | 'send_password_reset'
  | 'send_digest';

export interface NotificationJob {
  type: NotificationJobType;
  userId?: string;
  email?: string;
  phone?: string;
  fcmToken?: string;
  fcmTokens?: string[];
  payload: Record<string, unknown>;
}

// ============================================
// Queue เพิ่ม jobs
// ============================================

export async function queueWelcomeEmail(
  userId: string,
  email: string,
  username: string,
  verificationToken: string,
  language: 'th' | 'en' = 'th'
): Promise<void> {
  await notificationQueue.add(
    'send_welcome',
    {
      type: 'send_welcome',
      userId,
      email,
      payload: { username, verificationToken, language },
    } as NotificationJob,
    { priority: 2 }
  );
}

export async function queueSOSAlert(params: {
  sosId: string;
  recipientUserIds: string[];
  sosType: string;
  sosSeverity: 'low' | 'medium' | 'high' | 'critical';
  sosAddress: string;
  reporterName: string;
  recipientEmails: string[];
  recipientPhones: string[];
  recipientFCMTokens: string[];
}): Promise<void> {
  // SOS alerts มี priority สูงสุด
  await notificationQueue.add(
    'send_sos_alert',
    {
      type: 'send_sos_alert',
      payload: params,
    } as NotificationJob,
    { priority: 1 }  // Priority 1 = highest
  );
}

export async function queueDigestEmail(
  userId: string,
  email: string,
  digestData: Record<string, unknown>
): Promise<void> {
  await notificationQueue.add(
    'send_digest',
    {
      type: 'send_digest',
      userId,
      email,
      payload: digestData,
    } as NotificationJob,
    {
      priority: 5,  // Low priority
      delay: 0,
    }
  );
}

// ============================================
// Worker สำหรับประมวลผล notifications
// ============================================

export function startNotificationWorker(): Worker {
  const worker = new Worker(
    'notifications',
    async (job: Job<NotificationJob>) => {
      const { type, userId, email, phone, fcmToken, fcmTokens, payload } = job.data;

      console.log(`Processing notification job: ${job.id} (${type})`);

      switch (type) {
        case 'send_welcome':
          await sendWelcomeEmail(
            email!,
            payload.username as string,
            payload.verificationToken as string,
            (payload.language as 'th' | 'en') || 'th'
          );
          break;

        case 'send_sos_alert': {
          const sos = payload as any;

          // ส่ง email ไปทุกคนพร้อมกัน
          const emailPromises = sos.recipientEmails.map((recipientEmail: string, i: number) =>
            sendSOSAlertEmail({
              to: recipientEmail,
              recipientName: `ผู้ใช้งาน`,
              sosType: sos.sosType,
              sosAddress: sos.sosAddress,
              sosSeverity: sos.sosSeverity,
              reporterName: sos.reporterName,
              sosId: sos.sosId,
            })
          );

          // ส่ง SMS ฉุกเฉิน
          const smsPromises = sos.recipientPhones.map((recipientPhone: string) =>
            sendSOSEmergencySMS(
              recipientPhone,
              sos.sosType,
              sos.sosAddress,
              sos.sosId
            )
          );

          // ส่ง Push notification
          const pushPromise = sendSOSPushNotification(
            sos.recipientFCMTokens,
            {
              sosId: sos.sosId,
              type: sos.sosType,
              severity: sos.sosSeverity,
              address: sos.sosAddress,
            }
          );

          await Promise.allSettled([...emailPromises, ...smsPromises, pushPromise]);
          break;
        }

        case 'send_sms':
          if (phone) {
            await sendSMS({
              to: phone,
              message: payload.message as string,
              userId,
            });
          }
          break;

        case 'send_push':
          if (fcmToken) {
            await sendPushToToken(fcmToken, payload as any);
          }
          break;

        default:
          console.warn(`Unknown notification type: ${type}`);
      }
    },
    {
      connection: redisConnection,
      concurrency: 20,  // ประมวลผล 20 jobs พร้อมกัน
      limiter: {
        max: 100,        // max 100 jobs ต่อ
        duration: 1000,  // 1 วินาที
      },
    }
  );

  worker.on('completed', (job) => {
    console.log(`Notification job ${job.id} completed`);
  });

  worker.on('failed', (job, err) => {
    console.error(`Notification job ${job?.id} failed:`, err.message);
  });

  return worker;
}
```

---

## 🔧 Configuration Files

### Step 138: Notification Preferences Database Schema

```sql
-- PostgreSQL 17 schema

-- Table: user notification preferences
CREATE TABLE notification_preferences (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  
  -- Email preferences
  email_enabled       BOOLEAN DEFAULT TRUE,
  email_welcome       BOOLEAN DEFAULT TRUE,
  email_password_reset BOOLEAN DEFAULT TRUE,
  email_sos_alert     BOOLEAN DEFAULT TRUE,
  email_new_follower  BOOLEAN DEFAULT FALSE,
  email_weekly_digest BOOLEAN DEFAULT TRUE,
  
  -- SMS preferences
  sms_enabled         BOOLEAN DEFAULT FALSE,
  sms_sos_alert       BOOLEAN DEFAULT TRUE,  -- SOS default ON เสมอ
  sms_otp             BOOLEAN DEFAULT TRUE,
  
  -- Push notification preferences
  push_enabled        BOOLEAN DEFAULT TRUE,
  push_new_post       BOOLEAN DEFAULT TRUE,
  push_new_like       BOOLEAN DEFAULT TRUE,
  push_new_comment    BOOLEAN DEFAULT TRUE,
  push_new_follower   BOOLEAN DEFAULT TRUE,
  push_sos_alert      BOOLEAN DEFAULT TRUE,
  push_mention        BOOLEAN DEFAULT TRUE,
  
  -- Language preference
  language            VARCHAR(10) DEFAULT 'th',
  
  -- Quiet hours (ไม่ส่ง push notification)
  quiet_hours_enabled BOOLEAN DEFAULT FALSE,
  quiet_hours_start   TIME DEFAULT '22:00',
  quiet_hours_end     TIME DEFAULT '07:00',
  quiet_hours_timezone VARCHAR(50) DEFAULT 'Asia/Bangkok',
  
  created_at   TIMESTAMPTZ DEFAULT NOW(),
  updated_at   TIMESTAMPTZ DEFAULT NOW(),
  
  UNIQUE(user_id)
);

-- Table: FCM device tokens
CREATE TABLE fcm_tokens (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token       TEXT NOT NULL UNIQUE,
  platform    VARCHAR(20),  -- 'web', 'android', 'ios'
  device_info JSONB,
  is_active   BOOLEAN DEFAULT TRUE,
  last_used   TIMESTAMPTZ DEFAULT NOW(),
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_fcm_tokens_user_id ON fcm_tokens(user_id) WHERE is_active = TRUE;

-- Table: notification log (tracking ว่าส่งอะไรไปแล้ว)
CREATE TABLE notification_logs (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id),
  channel         VARCHAR(20) NOT NULL,  -- 'email', 'sms', 'push'
  type            VARCHAR(50) NOT NULL,  -- 'welcome', 'sos_alert', etc.
  status          VARCHAR(20) DEFAULT 'sent',  -- 'sent', 'failed', 'bounced'
  external_id     VARCHAR(255),  -- Twilio SID, Resend ID, etc.
  metadata        JSONB,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_notif_logs_user_channel ON notification_logs(user_id, channel, created_at);

-- Function: นับจำนวน SMS ที่ส่งวันนี้
CREATE OR REPLACE FUNCTION get_sms_count_today(p_user_id UUID)
RETURNS INTEGER AS $$
  SELECT COUNT(*)::INTEGER
  FROM notification_logs
  WHERE user_id = p_user_id
    AND channel = 'sms'
    AND status = 'sent'
    AND created_at >= CURRENT_DATE
    AND created_at < CURRENT_DATE + INTERVAL '1 day';
$$ LANGUAGE sql STABLE;
```

### Step 139: Web Push (Service Worker)

สร้างไฟล์ `/home/user/chuaikan-web/public/sw.js`:

```javascript
// Service Worker สำหรับ Web Push Notifications

const CACHE_NAME = 'chuaikan-v1';
const URLS_TO_CACHE = ['/', '/offline'];

// Install event
self.addEventListener('install', (event) => {
  console.log('Service Worker installing...');
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => cache.addAll(URLS_TO_CACHE))
  );
  self.skipWaiting();
});

// Activate event
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((cacheNames) => {
      return Promise.all(
        cacheNames
          .filter((name) => name !== CACHE_NAME)
          .map((name) => caches.delete(name))
      );
    })
  );
  self.clients.claim();
});

// Push event - รับ push notification จาก server
self.addEventListener('push', (event) => {
  let data = {};
  
  try {
    data = event.data?.json() ?? {};
  } catch (e) {
    data = { title: 'chuaikan', body: event.data?.text() ?? '' };
  }

  const {
    title = 'chuaikan',
    body = '',
    icon = '/icons/icon-192x192.png',
    badge = '/icons/badge-72x72.png',
    url = '/',
    tag,
    requireInteraction = false,
    data: notificationData = {},
  } = data;

  const options = {
    body,
    icon,
    badge,
    tag: tag || 'default',
    requireInteraction,
    vibrate: [200, 100, 200],
    data: { url, ...notificationData },
    actions: notificationData.type === 'sos_alert'
      ? [
          { action: 'view', title: 'ดูรายละเอียด' },
          { action: 'dismiss', title: 'ปิด' },
        ]
      : [],
  };

  event.waitUntil(
    self.registration.showNotification(title, options)
  );
});

// Notification click event
self.addEventListener('notificationclick', (event) => {
  const notification = event.notification;
  const action = event.action;
  const url = notification.data?.url || '/';

  notification.close();

  if (action === 'dismiss') return;

  event.waitUntil(
    clients.matchAll({ type: 'window' }).then((windowClients) => {
      // ถ้ามี window เปิดอยู่แล้ว → focus และ navigate
      for (const client of windowClients) {
        if (client.url.includes(self.location.origin)) {
          client.focus();
          client.navigate(url);
          return;
        }
      }
      // ไม่มี window เปิดอยู่ → เปิด window ใหม่
      return clients.openWindow(url);
    })
  );
});
```

---

## 🧪 Testing

### Step 140: ทดสอบ Notification System

```bash
# ทดสอบ Email ด้วย Resend (ใช้ test mode)
curl -X POST https://api.resend.com/emails \
  -H "Authorization: Bearer ${RESEND_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "test@chuaikan.com",
    "to": ["test@example.com"],
    "subject": "Test Email",
    "html": "<h1>Test</h1>"
  }'

# ทดสอบ SMS ด้วย Twilio Test Credentials
node -e "
const twilio = require('twilio');
const client = twilio(
  process.env.TWILIO_ACCOUNT_SID,
  process.env.TWILIO_AUTH_TOKEN
);
client.messages.create({
  body: 'Test SMS from chuaikan',
  from: process.env.TWILIO_PHONE_NUMBER,
  to: '+66812345678'
}).then(m => console.log('SMS SID:', m.sid));
"

# ตรวจสอบ BullMQ queue status
node -e "
const { Queue } = require('bullmq');
const q = new Queue('notifications', { connection: { host: 'localhost', port: 6379 } });
q.getJobCounts().then(counts => {
  console.log('Queue counts:', counts);
  q.close();
});
"

# ดู queue ด้วย Bull Board (web UI)
# ติดตั้ง
npm install @bull-board/express @bull-board/api

# เปิด Bull Board ที่ http://localhost:3000/admin/queues
```

---

## ❌ Common Errors & Solutions

### Error 1: Resend "Domain not verified"

```bash
# ต้อง verify domain ใน Resend dashboard
# 1. ไปที่ resend.com → Domains → Add Domain
# 2. เพิ่ม DNS records ที่กำหนด:
#    TXT: resend._domainkey.chuaikan.com
#    MX: mx.resend.com
# 3. Click Verify

# ตรวจสอบ DNS propagation
dig TXT resend._domainkey.chuaikan.com
```

### Error 2: Twilio "Geo Permission" error

```bash
# Thailand ต้องเปิด geo permission ใน Twilio Console
# Console → Messaging → Settings → Geo Permissions
# เปิด Thailand (TH) flag

# หรือตรวจสอบว่า phone number ถูกต้อง
# Thailand format: +66812345678 (ไม่ใช่ 0812345678)
```

### Error 3: FCM Token expired

```typescript
// ดัก error และลบ token ที่ invalid
try {
  await messaging.send(message);
} catch (error: any) {
  if (
    error.code === 'messaging/registration-token-not-registered' ||
    error.code === 'messaging/invalid-registration-token'
  ) {
    // ลบ token ออกจาก database
    await db.query(
      'UPDATE fcm_tokens SET is_active = FALSE WHERE token = $1',
      [fcmToken]
    );
  }
}
```

---

## ✅ Checklist

- [ ] Resend.com account สร้างแล้ว, domain verified
- [ ] React Email templates: Welcome, Password Reset, SOS Alert
- [ ] Nodemailer configured เป็น fallback
- [ ] Twilio account, Thailand geo permission เปิดแล้ว
- [ ] SMS rate limiting: max 5 SMS/user/day (ยกเว้น SOS)
- [ ] FCM Admin SDK configured
- [ ] Service Worker ลงทะเบียนบน frontend
- [ ] VAPID keys สร้างแล้ว
- [ ] notification_preferences table created
- [ ] fcm_tokens table created
- [ ] BullMQ workers เริ่มทำงานแล้ว
- [ ] Queue monitoring ด้วย Bull Board หรือ Grafana
- [ ] Unsubscribe flow: ลิงก์ใน email, settings page
- [ ] Thai/English localization ทำงาน
- [ ] Quiet hours feature ทำงาน
- [ ] SOS alert ส่งได้ทุก channels (Email + SMS + Push)

---

## 🔗 References

- [Resend Documentation](https://resend.com/docs)
- [React Email Documentation](https://react.email/)
- [Twilio SMS Thailand](https://www.twilio.com/en-us/sms/pricing/th)
- [Firebase Cloud Messaging](https://firebase.google.com/docs/cloud-messaging)
- [Web Push Protocol](https://web.dev/articles/push-notifications-overview)
- [BullMQ Documentation](https://docs.bullmq.io/)

---

*Part 014 | Road to 1,000,000 Users/Day | chuaikan.com*
