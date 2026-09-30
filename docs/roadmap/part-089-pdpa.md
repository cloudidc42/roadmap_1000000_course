# Part 089: Compliance (PDPA สำหรับไทย)
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Professional
> **Steps:** 881–890
> **เวลาโดยประมาณ:** 8 ชั่วโมง
> **Prerequisites:** Part 086 (Zero Trust), Part 087 (Vault)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- PDPA (พ.ร.บ. คุ้มครองข้อมูลส่วนบุคคล) overview
- Personal data categories ใน chuaikan.com
- Lawful basis for processing
- User consent management
- Right to access (ส่งออกข้อมูล user)
- Right to erasure (ลบ user และข้อมูลทั้งหมด)
- Data breach response: 72-hour notification
- Privacy policy และ terms of service requirements
- Data processing records (ROPA)

---

## 📖 ทฤษฎีและแนวคิด

### PDPA คืออะไร?

พ.ร.บ. คุ้มครองข้อมูลส่วนบุคคล พ.ศ. 2562 (PDPA) บังคับใช้กับธุรกิจที่เก็บข้อมูลส่วนบุคคลของคนไทย

**บทลงโทษ:**
- ทางแพ่ง: ค่าเสียหายสูงสุด 2 เท่าของความเสียหายจริง
- ทางปกครอง: ปรับสูงสุด 5 ล้านบาท
- ทางอาญา: จำคุก + ปรับ (กรณีเจตนา)

### Personal Data ใน chuaikan.com

| ประเภท | ข้อมูล | ความอ่อนไหว |
|--------|--------|-------------|
| Identity | ชื่อ, อีเมล, เบอร์โทร | ปกติ |
| Location | ที่อยู่, GPS coordinates | ปกติ |
| Content | โพสต์, comments | ปกติ |
| SOS | สถานที่ฉุกเฉิน, เหตุการณ์ | สูง |
| Financial | ประวัติการชำระเงิน | สูง |

---

## 🛠️ Step-by-Step Implementation

### Step 881: Consent Management

```typescript
// src/models/consent.model.ts
import { Model, DataTypes } from 'sequelize';

interface ConsentRecord {
  id: string;
  userId: string;
  purpose: ConsentPurpose;
  granted: boolean;
  grantedAt?: Date;
  withdrawnAt?: Date;
  ipAddress: string;
  userAgent: string;
  version: string;  // version ของ privacy policy
}

type ConsentPurpose =
  | 'marketing_email'
  | 'analytics'
  | 'personalization'
  | 'third_party_sharing'
  | 'essential';  // ไม่ต้องขอ consent

export class UserConsent extends Model<ConsentRecord> {
  // ...
}

UserConsent.init({
  id: { type: DataTypes.UUID, defaultValue: DataTypes.UUIDV4, primaryKey: true },
  userId: { type: DataTypes.UUID, allowNull: false },
  purpose: {
    type: DataTypes.ENUM(
      'marketing_email', 'analytics', 'personalization',
      'third_party_sharing', 'essential'
    ),
    allowNull: false
  },
  granted: { type: DataTypes.BOOLEAN, allowNull: false },
  grantedAt: { type: DataTypes.DATE },
  withdrawnAt: { type: DataTypes.DATE },
  ipAddress: { type: DataTypes.INET, allowNull: false },
  userAgent: { type: DataTypes.TEXT, allowNull: false },
  version: { type: DataTypes.STRING, allowNull: false }
}, { sequelize, tableName: 'user_consents' });
```

```typescript
// src/routes/consent.ts
// API endpoints สำหรับจัดการ consent

router.get('/consent', authenticate, async (req, res) => {
  const consents = await UserConsent.findAll({
    where: { userId: req.user.id },
    order: [['createdAt', 'DESC']]
  });

  const latest = consents.reduce((acc, c) => {
    if (!acc[c.purpose] || c.createdAt > acc[c.purpose].createdAt) {
      acc[c.purpose] = c;
    }
    return acc;
  }, {} as Record<string, UserConsent>);

  return res.json({
    consents: Object.values(latest).map(c => ({
      purpose: c.purpose,
      granted: c.granted,
      lastUpdated: c.grantedAt || c.withdrawnAt
    }))
  });
});

router.post('/consent', authenticate, async (req, res) => {
  const { purpose, granted } = req.body;

  await UserConsent.create({
    userId: req.user.id,
    purpose,
    granted,
    grantedAt: granted ? new Date() : undefined,
    withdrawnAt: !granted ? new Date() : undefined,
    ipAddress: req.ip,
    userAgent: req.headers['user-agent'] || 'unknown',
    version: '2024-01'  // Privacy policy version
  });

  return res.json({ message: 'Consent updated' });
});

// ถอน consent ทั้งหมด
router.delete('/consent', authenticate, async (req, res) => {
  const purposes = ['marketing_email', 'analytics', 'personalization', 'third_party_sharing'];

  await Promise.all(purposes.map(purpose =>
    UserConsent.create({
      userId: req.user.id,
      purpose: purpose as ConsentPurpose,
      granted: false,
      withdrawnAt: new Date(),
      ipAddress: req.ip,
      userAgent: req.headers['user-agent'] || 'unknown',
      version: '2024-01'
    })
  ));

  return res.json({ message: 'All consents withdrawn' });
});
```

### Step 882: Right to Access (Data Export)

```typescript
// src/services/data-export.service.ts
import archiver from 'archiver';
import { createWriteStream } from 'fs';
import { S3 } from '@aws-sdk/client-s3';

const s3 = new S3({ region: 'ap-southeast-1' });

export async function exportUserData(userId: string): Promise<string> {
  const exportId = `export_${userId}_${Date.now()}`;

  // รวบรวมข้อมูลทั้งหมดของ user
  const [user, posts, comments, likes, follows, sosAlerts, consents] = await Promise.all([
    getUserProfile(userId),
    getUserPosts(userId),
    getUserComments(userId),
    getUserLikes(userId),
    getUserFollows(userId),
    getUserSosAlerts(userId),
    getUserConsents(userId)
  ]);

  // สร้าง JSON export
  const exportData = {
    exportedAt: new Date().toISOString(),
    requestedBy: userId,
    user: {
      id: user.id,
      name: user.name,
      email: user.email,
      phone: user.phone,
      createdAt: user.createdAt,
      bio: user.bio
    },
    content: {
      posts: posts.map(p => ({
        id: p.id,
        content: p.content,
        createdAt: p.createdAt,
        images: p.images
      })),
      comments: comments.map(c => ({
        id: c.id,
        content: c.content,
        postId: c.postId,
        createdAt: c.createdAt
      })),
      likes: likes.map(l => ({ postId: l.postId, createdAt: l.createdAt }))
    },
    social: {
      following: follows.following.map(f => ({ userId: f.id, name: f.name })),
      followers: follows.followers.map(f => ({ userId: f.id, name: f.name }))
    },
    sosHistory: sosAlerts.map(s => ({
      id: s.id,
      description: s.description,
      location: s.location,
      status: s.status,
      createdAt: s.createdAt
    })),
    consentHistory: consents.map(c => ({
      purpose: c.purpose,
      granted: c.granted,
      timestamp: c.grantedAt || c.withdrawnAt
    }))
  };

  // Upload ไป S3 (ใช้ presigned URL ให้ user download)
  const key = `exports/${userId}/${exportId}.json`;
  await s3.putObject({
    Bucket: 'chuaikan-data-exports',
    Key: key,
    Body: JSON.stringify(exportData, null, 2),
    ContentType: 'application/json',
    ServerSideEncryption: 'AES256'
  });

  // สร้าง presigned URL (expire ใน 24 ชั่วโมง)
  const { getSignedUrl } = await import('@aws-sdk/s3-request-presigner');
  const { GetObjectCommand } = await import('@aws-sdk/client-s3');

  const url = await getSignedUrl(s3, new GetObjectCommand({
    Bucket: 'chuaikan-data-exports',
    Key: key
  }), { expiresIn: 86400 });

  return url;
}
```

```typescript
// src/routes/data-rights.ts
// Data Subject Rights API

router.post('/privacy/export', authenticate, async (req, res) => {
  const userId = req.user.id;

  // Check ว่าไม่ได้ request ถี่เกิน (1 ครั้งต่อ 30 วัน)
  const lastExport = await DataExportRequest.findOne({
    where: {
      userId,
      createdAt: { [Op.gte]: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000) }
    }
  });

  if (lastExport) {
    return res.status(429).json({
      error: 'You can only request data export once per 30 days',
      nextAvailableAt: new Date(lastExport.createdAt.getTime() + 30 * 24 * 60 * 60 * 1000)
    });
  }

  // สร้าง export request
  const request = await DataExportRequest.create({ userId, status: 'pending' });

  // Process แบบ async (อาจใช้เวลา)
  processDataExport(userId, request.id)
    .then(async (downloadUrl) => {
      await request.update({ status: 'completed', downloadUrl });

      // ส่ง email ให้ user
      await sendEmail(req.user.email, 'Your data export is ready', {
        downloadUrl,
        expiresAt: new Date(Date.now() + 86400000)
      });
    })
    .catch(async (err) => {
      await request.update({ status: 'failed', error: err.message });
    });

  return res.json({
    message: 'Data export request received. You will receive an email when ready (usually within 24 hours).',
    requestId: request.id
  });
});
```

### Step 883: Right to Erasure (ลบ User)

```typescript
// src/services/user-erasure.service.ts
// Right to Erasure (Right to be Forgotten)

import { writePool } from '../db/pool';

interface ErasureResult {
  userId: string;
  erasedAt: Date;
  recordsDeleted: Record<string, number>;
}

export async function eraseUserData(userId: string, reason: string): Promise<ErasureResult> {
  const client = await writePool.connect();

  try {
    await client.query('BEGIN');

    const counts: Record<string, number> = {};

    // 1. ลบ content ของ user
    const postResult = await client.query(
      'DELETE FROM posts WHERE user_id = $1 RETURNING id',
      [userId]
    );
    counts.posts = postResult.rowCount || 0;

    const commentResult = await client.query(
      'DELETE FROM comments WHERE user_id = $1',
      [userId]
    );
    counts.comments = commentResult.rowCount || 0;

    // 2. ลบ interactions
    await client.query('DELETE FROM likes WHERE user_id = $1', [userId]);
    await client.query('DELETE FROM follows WHERE follower_id = $1 OR following_id = $1', [userId]);
    await client.query('DELETE FROM notifications WHERE user_id = $1 OR actor_id = $1', [userId]);

    // 3. ลบ SOS history
    await client.query('DELETE FROM sos_alerts WHERE user_id = $1', [userId]);

    // 4. ลบ consent records
    await client.query('DELETE FROM user_consents WHERE user_id = $1', [userId]);

    // 5. Anonymize แทนที่จะลบ (เพื่อ audit trail)
    // ไม่ลบแถว users เพื่อรักษา referential integrity
    await client.query(`
      UPDATE users SET
        name = 'Deleted User',
        email = $2,
        phone = NULL,
        bio = NULL,
        avatar_url = NULL,
        deleted_at = NOW(),
        deletion_reason = $3,
        is_active = false
      WHERE id = $1
    `, [userId, `deleted_${userId}@deleted.invalid`, reason]);

    // 6. บันทึก erasure request
    await client.query(`
      INSERT INTO data_erasure_log (user_id, erased_at, reason, records_deleted)
      VALUES ($1, NOW(), $2, $3)
    `, [userId, reason, JSON.stringify(counts)]);

    await client.query('COMMIT');

    return {
      userId,
      erasedAt: new Date(),
      recordsDeleted: counts
    };
  } catch (e) {
    await client.query('ROLLBACK');
    throw e;
  } finally {
    client.release();
  }
}
```

### Step 884: Data Breach Response

```typescript
// src/services/breach-response.service.ts
// Data Breach Response (72-hour notification requirement)

interface DataBreach {
  id: string;
  discoveredAt: Date;
  description: string;
  affectedUsers: string[];
  dataCategories: string[];
  severity: 'low' | 'medium' | 'high' | 'critical';
}

export async function reportDataBreach(breach: DataBreach): Promise<void> {
  const notificationDeadline = new Date(breach.discoveredAt.getTime() + 72 * 60 * 60 * 1000);

  console.log(`DATA BREACH DETECTED!`);
  console.log(`ID: ${breach.id}`);
  console.log(`Discovered: ${breach.discoveredAt.toISOString()}`);
  console.log(`PDPA Notification Deadline: ${notificationDeadline.toISOString()}`);
  console.log(`Affected Users: ${breach.affectedUsers.length}`);

  // 1. บันทึกใน database
  await writePool.query(`
    INSERT INTO data_breach_log
      (id, discovered_at, description, affected_user_count, data_categories, severity, notification_deadline)
    VALUES ($1, $2, $3, $4, $5, $6, $7)
  `, [
    breach.id,
    breach.discoveredAt,
    breach.description,
    breach.affectedUsers.length,
    JSON.stringify(breach.dataCategories),
    breach.severity,
    notificationDeadline
  ]);

  // 2. แจ้งทีม (immediate)
  await notifySecurityTeam(breach);

  // 3. แจ้ง users ที่ได้รับผลกระทบ (ถ้า high/critical)
  if (breach.severity === 'high' || breach.severity === 'critical') {
    await notifyAffectedUsers(breach);
  }

  // 4. สร้าง PDPA notification draft (สำหรับแจ้ง PDPC ภายใน 72 ชั่วโมง)
  await createPDPCNotificationDraft(breach, notificationDeadline);
}

async function createPDPCNotificationDraft(breach: DataBreach, deadline: Date): Promise<void> {
  const notification = `
ประกาศการรั่วไหลของข้อมูลส่วนบุคคล
บริษัท ช่วยกัน จำกัด (chuaikan.com)

วันที่แจ้ง: ${new Date().toLocaleDateString('th-TH')}
กำหนดส่งถึง PDPC: ${deadline.toLocaleDateString('th-TH')} เวลา ${deadline.toLocaleTimeString('th-TH')}

1. ลักษณะของเหตุการณ์:
${breach.description}

2. ข้อมูลส่วนบุคคลที่เกี่ยวข้อง:
${breach.dataCategories.join(', ')}

3. จำนวนผู้ได้รับผลกระทบโดยประมาณ:
${breach.affectedUsers.length} ราย

4. มาตรการที่ดำเนินการแล้ว:
- แยก systems ที่ได้รับผลกระทบออก
- รีเซ็ต credentials ที่เกี่ยวข้อง
- ตรวจสอบ access logs

5. มาตรการป้องกันในอนาคต:
[กำหนดเพิ่มเติม]

ผู้ประสานงาน: DPO@chuaikan.com
  `;

  console.log('PDPC Notification Draft:');
  console.log(notification);
}
```

### Step 885: ROPA (Records of Processing Activities)

```typescript
// src/admin/ropa.ts
// Records of Processing Activities

interface ProcessingActivity {
  id: string;
  name: string;
  purpose: string;
  lawfulBasis: 'consent' | 'contract' | 'legal_obligation' | 'legitimate_interest';
  dataCategories: string[];
  dataSubjects: string[];
  recipients: string[];
  retentionPeriod: string;
  securityMeasures: string[];
  crossBorderTransfer?: {
    country: string;
    safeguards: string;
  };
}

const CHUAIKAN_PROCESSING_ACTIVITIES: ProcessingActivity[] = [
  {
    id: 'PA-001',
    name: 'User Registration and Authentication',
    purpose: 'สร้างและจัดการ user accounts',
    lawfulBasis: 'contract',
    dataCategories: ['ชื่อ', 'อีเมล', 'เบอร์โทรศัพท์', 'รหัสผ่าน (hashed)'],
    dataSubjects: ['users', 'ผู้ใช้บริการ'],
    recipients: ['Internal IT', 'AWS (processor)'],
    retentionPeriod: 'ตลอดระยะเวลาที่ account ยังใช้งาน + 1 ปี',
    securityMeasures: ['Encryption at rest', 'TLS in transit', 'MFA available']
  },
  {
    id: 'PA-002',
    name: 'SOS Alert Service',
    purpose: 'ช่วยเหลือผู้ใช้ในกรณีฉุกเฉิน',
    lawfulBasis: 'legitimate_interest',
    dataCategories: ['ตำแหน่งที่ตั้ง GPS', 'คำอธิบายเหตุการณ์', 'รูปภาพ/วิดีโอ'],
    dataSubjects: ['users', 'ผู้แจ้งเหตุ'],
    recipients: ['ผู้รับแจ้งใน network', 'Internal operations'],
    retentionPeriod: '3 ปี',
    securityMeasures: ['Access control', 'Audit log']
  },
  {
    id: 'PA-003',
    name: 'Analytics and Product Improvement',
    purpose: 'วิเคราะห์การใช้งานเพื่อปรับปรุงบริการ',
    lawfulBasis: 'consent',
    dataCategories: ['Usage patterns', 'Device info', 'IP address (anonymized)'],
    dataSubjects: ['users ที่ให้ consent'],
    recipients: ['ClickHouse (internal analytics)'],
    retentionPeriod: '2 ปี',
    securityMeasures: ['Anonymization', 'Aggregation before retention']
  },
  {
    id: 'PA-004',
    name: 'Marketing Communications',
    purpose: 'ส่ง newsletter, promotions',
    lawfulBasis: 'consent',
    dataCategories: ['อีเมล', 'ชื่อ', 'ประวัติการใช้บริการ'],
    dataSubjects: ['users ที่ opt-in'],
    recipients: ['SendGrid (email processor)'],
    retentionPeriod: 'จนกว่าจะถอน consent',
    securityMeasures: ['Unsubscribe mechanism', 'Consent log']
  }
];

// Export ROPA เป็น PDF
async function generateROPA(): Promise<Buffer> {
  // ใช้ library เช่น pdfkit
  // return PDF buffer
  console.log('ROPA generated with', CHUAIKAN_PROCESSING_ACTIVITIES.length, 'activities');
  return Buffer.from('PDF content');
}
```

### Step 886: Privacy Policy Requirements

```typescript
// src/routes/legal.ts

router.get('/privacy-policy', async (req, res) => {
  // Return current privacy policy version
  const policy = await LegalDocument.findOne({
    where: { type: 'privacy_policy', isActive: true },
    order: [['version', 'DESC']]
  });

  return res.json({
    version: policy.version,
    effectiveDate: policy.effectiveDate,
    content: policy.content,
    dataController: {
      name: 'บริษัท ช่วยกัน จำกัด',
      address: '...',
      email: 'privacy@chuaikan.com',
      dpo: 'dpo@chuaikan.com'
    }
  });
});

// ตรวจสอบว่า user รับรู้ privacy policy version ล่าสุด
router.get('/privacy-check', authenticate, async (req, res) => {
  const currentVersion = '2024-01';

  const acknowledged = await PolicyAcknowledgment.findOne({
    where: { userId: req.user.id, version: currentVersion }
  });

  return res.json({
    acknowledged: !!acknowledged,
    currentVersion,
    acknowledgmentRequired: !acknowledged
  });
});

router.post('/privacy-acknowledge', authenticate, async (req, res) => {
  const { version } = req.body;

  await PolicyAcknowledgment.upsert({
    userId: req.user.id,
    version,
    acknowledgedAt: new Date(),
    ipAddress: req.ip,
    userAgent: req.headers['user-agent']
  });

  return res.json({ message: 'Acknowledged' });
});
```

---

## 🧪 Testing

```bash
# ทดสอบ data export
curl -X POST "https://api.chuaikan.com/v1/privacy/export" \
  -H "Authorization: Bearer YOUR_TOKEN" | jq .

# ทดสอบ right to erasure
curl -X POST "https://api.chuaikan.com/v1/account/delete" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{"reason": "No longer want to use the service", "confirmation": "DELETE MY ACCOUNT"}' | jq .

# ทดสอบ consent withdrawal
curl -X DELETE "https://api.chuaikan.com/v1/consent" \
  -H "Authorization: Bearer YOUR_TOKEN" | jq .
```

---

## ✅ Checklist

- [ ] Privacy Policy เขียนเสร็จ ครอบคลุม data categories ทั้งหมด
- [ ] Consent checkbox ใน registration form
- [ ] Consent log บันทึก timestamp, IP, version
- [ ] Right to Access: user ขอ export data ได้ภายใน 30 วัน
- [ ] Right to Erasure: ลบ user data ได้ทั้งหมด
- [ ] Data Breach Response Plan เขียนเสร็จ
- [ ] PDPC notification template พร้อม
- [ ] ROPA document ครบ 4+ processing activities
- [ ] DPO แต่งตั้งแล้ว (Data Protection Officer)
- [ ] Privacy training ให้ทีมทุกคน

---

## 🔗 References

- [PDPA Text (ภาษาไทย)](https://www.pdpc.or.th/pdpa/)
- [PDPC Guidelines](https://www.pdpc.or.th/category/guidance/)
- [GDPR (basis for PDPA)](https://gdpr.eu/)

---
*Part 089 | Road to 1,000,000 Users/Day | chuaikan.com*
