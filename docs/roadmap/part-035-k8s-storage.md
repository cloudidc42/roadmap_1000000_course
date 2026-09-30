# Part 035: Persistent Volumes

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 341-350
> **เวลาโดยประมาณ:** 3 ชั่วโมง
> **Prerequisites:** Part 034 (ConfigMaps & Secrets), Part 033 (Workloads)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- เข้าใจ PersistentVolume (PV) และ PersistentVolumeClaim (PVC)
- ใช้ StorageClass สำหรับ dynamic provisioning
- สร้าง PostgreSQL StatefulSet พร้อม PVC
- สร้าง Redis StatefulSet พร้อม PVC
- เข้าใจ ReadWriteOnce vs ReadWriteMany
- Backup PersistentVolumes ด้วย Velero

---

## 📖 ทฤษฎีและแนวคิด

### Step 341 — PV และ PVC คืออะไร

```
PersistentVolume (PV):
- Storage resource ใน cluster (จัดการโดย admin)
- เป็นเหมือน "storage ที่มีอยู่ให้ใช้"
- lifecycle เป็นอิสระจาก Pod

PersistentVolumeClaim (PVC):
- Request ขอใช้ storage จาก user/developer
- เหมือนการ "จอง storage"
- Pod ใช้ PVC เพื่อ mount storage

StorageClass:
- กำหนดประเภทของ storage
- ใช้สำหรับ dynamic provisioning
- สร้าง PV ให้อัตโนมัติเมื่อมี PVC
```

```
User creates PVC --> StorageClass creates PV automatically --> Pod mounts PVC
```

### Step 342 — Access Modes

```
ReadWriteOnce (RWO):
- mount ได้ด้วย 1 node พร้อมกัน (อ่าน/เขียน)
- ใช้กับ: PostgreSQL, MySQL (single instance)
- ส่วนใหญ่ cloud disk ใช้แบบนี้

ReadWriteMany (RWX):
- mount ได้หลาย nodes พร้อมกัน (อ่าน/เขียน)
- ใช้กับ: shared file storage, NFS
- ต้องใช้ NFS หรือ cloud file storage

ReadOnlyMany (ROX):
- mount ได้หลาย nodes แต่ read-only
```

---

## ⚙️ Environment Setup

```bash
# ติดตั้ง local-path-provisioner สำหรับ development/VPS
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.26/deploy/local-path-storage.yaml

# ตรวจสอบ
kubectl get storageclass
# NAME                   PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE
# local-path (default)   rancher.io/local-path          Delete          WaitForFirstConsumer

# ตรวจสอบ pods
kubectl get pods -n local-path-storage
```

---

## 🛠️ Step-by-Step Implementation

### Step 343 — StorageClass สำหรับ chuaikan.com

```yaml
# storageclass-local.yaml (สำหรับ VPS/bare metal)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
reclaimPolicy: Retain       # Retain เมื่อ PVC ถูกลบ (ปลอดภัยกว่า Delete)
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

```yaml
# storageclass-aws-ebs.yaml (สำหรับ AWS)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-encrypted
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

```bash
kubectl apply -f storageclass-local.yaml
kubectl get storageclass
```

### Step 344 — PostgreSQL StatefulSet พร้อม PVC

```yaml
# statefulset-postgres.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: chuaikan-production
  labels:
    app: postgres
    component: database
spec:
  serviceName: postgres-headless
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
        component: database
    spec:
      terminationGracePeriodSeconds: 60
      securityContext:
        fsGroup: 999
        runAsUser: 999
      containers:
      - name: postgres
        image: postgres:16-alpine
        ports:
        - containerPort: 5432
          name: postgres
        env:
        - name: POSTGRES_DB
          value: "chuaikan_db"
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: POSTGRES_USER
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: POSTGRES_PASSWORD
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        resources:
          requests:
            cpu: "250m"
            memory: "512Mi"
          limits:
            cpu: "2000m"
            memory: "4Gi"
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
        - name: postgres-config
          mountPath: /etc/postgresql/conf.d
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
            - -d
            - chuaikan_db
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 5
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
      volumes:
      - name: postgres-config
        configMap:
          name: postgres-config
  # VolumeClaimTemplates สร้าง PVC ใหม่สำหรับแต่ละ replica
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: local-path
      resources:
        requests:
          storage: 50Gi
```

```yaml
# configmap-postgres.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: chuaikan-production
data:
  custom.conf: |
    # Performance tuning
    max_connections = 100
    shared_buffers = 256MB
    effective_cache_size = 768MB
    maintenance_work_mem = 64MB
    checkpoint_completion_target = 0.7
    wal_buffers = 16MB
    default_statistics_target = 100
    random_page_cost = 1.1
    effective_io_concurrency = 200
    min_wal_size = 1GB
    max_wal_size = 4GB
    max_worker_processes = 2
    max_parallel_workers_per_gather = 1
    max_parallel_workers = 2
    max_parallel_maintenance_workers = 1

    # Logging
    log_min_duration_statement = 1000
    log_checkpoints = on
    log_connections = on
    log_disconnections = on
    log_lock_waits = on
```

```yaml
# service-postgres.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: chuaikan-production
  labels:
    app: postgres
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: chuaikan-production
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

```bash
kubectl apply -f configmap-postgres.yaml
kubectl apply -f service-postgres.yaml
kubectl apply -f statefulset-postgres.yaml

# ดู StatefulSet
kubectl get statefulset -n chuaikan-production
kubectl get pods -l app=postgres -n chuaikan-production
kubectl get pvc -n chuaikan-production
kubectl get pv
```

### Step 345 — Redis StatefulSet พร้อม PVC

```yaml
# statefulset-redis.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: chuaikan-production
  labels:
    app: redis
    component: cache
spec:
  serviceName: redis-headless
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
        component: cache
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        command:
        - redis-server
        - /etc/redis/redis.conf
        ports:
        - containerPort: 6379
          name: redis
        resources:
          requests:
            cpu: "100m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "1Gi"
        volumeMounts:
        - name: redis-data
          mountPath: /data
        - name: redis-config
          mountPath: /etc/redis
        readinessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 10
          periodSeconds: 10
      volumes:
      - name: redis-config
        configMap:
          name: redis-config
  volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: local-path
      resources:
        requests:
          storage: 10Gi
```

```yaml
# configmap-redis.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
  namespace: chuaikan-production
data:
  redis.conf: |
    # Redis configuration
    maxmemory 512mb
    maxmemory-policy allkeys-lru
    save 900 1
    save 300 10
    save 60 10000
    appendonly yes
    appendfsync everysec
    hz 10
    dynamic-hz yes
    loglevel notice
    databases 16
    tcp-keepalive 300
    timeout 0
```

```bash
kubectl apply -f configmap-redis.yaml
kubectl apply -f statefulset-redis.yaml

# ตรวจสอบ
kubectl get statefulset redis -n chuaikan-production
kubectl get pvc -l app=redis -n chuaikan-production
```

### Step 346 — สร้าง PVC แบบ manual (สำหรับ Shared Storage)

```yaml
# pvc-shared-uploads.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-uploads
  namespace: chuaikan-production
  labels:
    app: chuaikan
    type: uploads
spec:
  accessModes:
  - ReadWriteMany         # ต้องใช้กับ NFS
  storageClassName: nfs-client
  resources:
    requests:
      storage: 100Gi
```

```yaml
# ติดตั้ง NFS Provisioner (ถ้าต้องการ RWX)
# storageclass-nfs.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
provisioner: nfs.csi.k8s.io
parameters:
  server: 192.168.1.100    # NFS server IP
  share: /exports/k8s
reclaimPolicy: Retain
volumeBindingMode: Immediate
mountOptions:
- nfsvers=4.1
```

### Step 347 — ดู PV และ PVC Status

```bash
# ดู PersistentVolumes ทั้งหมด
kubectl get pv
# NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM
# pvc-abc123-xxx                             50Gi       RWO            Retain           Bound    chuaikan-production/postgres-data-postgres-0

# ดู PersistentVolumeClaims
kubectl get pvc -n chuaikan-production
# NAME                          STATUS   VOLUME                CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# postgres-data-postgres-0      Bound    pvc-abc123-xxx        50Gi       RWO            local-path     5m

# ดูรายละเอียด PVC
kubectl describe pvc postgres-data-postgres-0 -n chuaikan-production

# ดู storage ที่ใช้จริงใน pod
kubectl exec -it postgres-0 -n chuaikan-production -- df -h /var/lib/postgresql/data
```

### Step 348 — Resize PVC (Expand Storage)

```bash
# ตรวจสอบ StorageClass รองรับ expansion
kubectl get storageclass local-path -o yaml | grep allowVolumeExpansion
# allowVolumeExpansion: true

# แก้ PVC ให้ใหญ่ขึ้น
kubectl patch pvc postgres-data-postgres-0 \
  -n chuaikan-production \
  -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'

# ตรวจสอบ (อาจต้อง restart pod)
kubectl get pvc postgres-data-postgres-0 -n chuaikan-production -w
```

### Step 349 — Backup PV ด้วย Velero

```bash
# ติดตั้ง Velero CLI
wget https://github.com/vmware-tanzu/velero/releases/download/v1.13.0/velero-v1.13.0-linux-amd64.tar.gz
tar -xzf velero-v1.13.0-linux-amd64.tar.gz
sudo mv velero-v1.13.0-linux-amd64/velero /usr/local/bin/

# ติดตั้ง Velero ใน cluster (ตัวอย่างกับ MinIO/S3)
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.9.0 \
  --bucket chuaikan-velero-backup \
  --secret-file ./credentials-velero \
  --use-volume-snapshots=false \
  --backup-location-config region=ap-southeast-1,s3ForcePathStyle="true",s3Url=https://minio.chuaikan.com \
  --use-node-agent \
  --default-volumes-to-fs-backup

# credentials-velero file:
# [default]
# aws_access_key_id=<minio-access-key>
# aws_secret_access_key=<minio-secret-key>

# Backup namespace
velero backup create chuaikan-production-backup \
  --include-namespaces=chuaikan-production \
  --wait

# ดู backup
velero backup describe chuaikan-production-backup
velero backup get

# Restore
velero restore create --from-backup chuaikan-production-backup \
  --namespace-mappings chuaikan-production:chuaikan-restored

# Schedule backup ทุกวัน
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces=chuaikan-production \
  --ttl 168h    # เก็บ 7 วัน
```

### Step 350 — Database Backup ด้วย CronJob

```yaml
# cronjob-postgres-backup.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: chuaikan-production
spec:
  schedule: "0 2 * * *"    # ทุกวัน 02:00 UTC
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: postgres-backup
            image: postgres:16-alpine
            command:
            - /bin/sh
            - -c
            - |
              BACKUP_FILE="backup-$(date +%Y%m%d-%H%M%S).sql.gz"
              pg_dump -h postgres-service -U $POSTGRES_USER $POSTGRES_DB | \
                gzip > /backup/$BACKUP_FILE
              echo "Backup completed: $BACKUP_FILE"
              # ลบ backup เก่ากว่า 7 วัน
              find /backup -name "*.sql.gz" -mtime +7 -delete
            env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secrets
                  key: POSTGRES_USER
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secrets
                  key: POSTGRES_PASSWORD
            - name: POSTGRES_DB
              valueFrom:
                secretKeyRef:
                  name: postgres-secrets
                  key: POSTGRES_DB
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: postgres-backup-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-backup-pvc
  namespace: chuaikan-production
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 50Gi
```

```bash
kubectl apply -f cronjob-postgres-backup.yaml

# ทดสอบ run backup ทันที
kubectl create job --from=cronjob/postgres-backup manual-backup -n chuaikan-production

# ดู job
kubectl get jobs -n chuaikan-production
kubectl logs -l job-name=manual-backup -n chuaikan-production
```

---

## 🧪 Testing

### ทดสอบ PostgreSQL Persistence

```bash
# เชื่อมต่อ PostgreSQL
kubectl exec -it postgres-0 -n chuaikan-production -- \
  psql -U chuaikan_user -d chuaikan_db

# สร้าง test data
CREATE TABLE test_persistence (id SERIAL, value TEXT);
INSERT INTO test_persistence VALUES (1, 'hello chuaikan');
SELECT * FROM test_persistence;
\q

# ลบ pod (StatefulSet จะสร้างใหม่)
kubectl delete pod postgres-0 -n chuaikan-production

# รอ pod กลับมา
kubectl get pod postgres-0 -n chuaikan-production -w

# ตรวจสอบ data ยังอยู่
kubectl exec -it postgres-0 -n chuaikan-production -- \
  psql -U chuaikan_user -d chuaikan_db -c "SELECT * FROM test_persistence"
# ควรยังเห็น "hello chuaikan"
```

---

## ❌ Common Errors & Solutions

### Error: `PVC stuck in Pending state`
```bash
kubectl describe pvc postgres-data-postgres-0 -n chuaikan-production
# ดูที่ Events: 
# "waiting for first consumer to be created before binding" = normal สำหรับ WaitForFirstConsumer
# ถ้า error อื่น: ตรวจสอบ StorageClass

kubectl get storageclass
kubectl get pods -n local-path-storage
```

### Error: `pod has unbound immediate PersistentVolumeClaims`
```bash
# สาเหตุ: PVC ยังไม่ Bound
kubectl get pvc -n chuaikan-production
# ถ้า STATUS = Pending ดู events ของ PVC
kubectl describe pvc <pvc-name> -n chuaikan-production
```

### Error: `OOMKilled` ใน PostgreSQL
```bash
# เพิ่ม memory limit
kubectl patch statefulset postgres -n chuaikan-production \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/resources/limits/memory", "value": "8Gi"}]'
```

---

## ✅ Checklist

- [ ] **Step 341**: เข้าใจ PV, PVC, StorageClass และความสัมพันธ์กัน
- [ ] **Step 342**: อธิบาย Access Modes ได้: RWO, RWX, ROX
- [ ] **Step 343**: ติดตั้ง StorageClass: local-path สำหรับ VPS และเข้าใจ EBS สำหรับ AWS
- [ ] **Step 344**: สร้าง PostgreSQL StatefulSet พร้อม PVC ขนาด 50Gi สำเร็จ
- [ ] **Step 345**: สร้าง Redis StatefulSet พร้อม PVC สำเร็จ
- [ ] **Step 346**: เข้าใจ PVC สำหรับ Shared Storage (RWX + NFS)
- [ ] **Step 347**: ดู PV/PVC status และตีความ Bound/Released/Pending ได้
- [ ] **Step 348**: Resize PVC ให้ใหญ่ขึ้นได้
- [ ] **Step 349**: ติดตั้งและใช้ Velero backup namespace สำเร็จ
- [ ] **Step 350**: สร้าง CronJob backup PostgreSQL ทุกวันสำเร็จ

---

## 🔗 References

- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Velero Documentation](https://velero.io/docs/latest/)
- [local-path-provisioner](https://github.com/rancher/local-path-provisioner)

---

*Part 035 | Road to 1,000,000 Users/Day | chuaikan.com*
