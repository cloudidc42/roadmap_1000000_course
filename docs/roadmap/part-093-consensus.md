# Part 093: Consensus Algorithms

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** World Class
> **Steps:** 921-930
> **เวลาโดยประมาณ:** 4 ชั่วโมง
> **Prerequisites:** Part 092 (Distributed Systems), Part 031 (Kubernetes Concepts)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

Consensus Algorithm คือ "สมอง" ของ Distributed System ที่ทำให้ nodes ทั้งหมด "เห็นด้วย" กับ state เดียวกัน ใน Part นี้เราจะเข้าใจ:

- ทำไม Consensus ถึงยากใน Distributed System
- Paxos: algorithm แรกที่แก้ปัญหา Consensus
- Raft: algorithm ที่เข้าใจง่ายกว่าและใช้งานจริง
- etcd ใช้ Raft อย่างไรใน Kubernetes
- PostgreSQL High Availability ด้วย Raft-like consensus

---

## 📖 ทฤษฎีและแนวคิด

### 1. Consensus Problem

> "ทำให้ nodes หลายตัวใน distributed system ตกลงกันได้ว่า value อะไรคือ 'correct'"

**ตัวอย่างที่เข้าใจง่าย:**
```
มี 3 server ใน cluster:
Server A: "Leader คือ Node 1"
Server B: "Leader คือ Node 2"  
Server C: "Leader คือ Node 1"

โดยไม่มี consensus algorithm → เกิด Split Brain
ด้วย consensus algorithm → ทุกคน agree ว่า Leader คือ Node ไหน
```

**ความยากของ Consensus (FLP Impossibility Theorem):**
Fischer, Lynch, Paterson พิสูจน์ในปี 1985 ว่า:
> ในระบบ async บน network ที่อาจมี message loss ไม่มี algorithm ที่รับประกัน consensus ได้เสมอ

แต่ในทางปฏิบัติ เราทำงานกับ network timeout และ probabilistic guarantees ได้

### 2. Paxos Algorithm

Paxos เสนอโดย Leslie Lamport ในปี 1989 เป็น algorithm แรกที่แก้ Consensus ได้ แต่เข้าใจยากมาก

**Roles ใน Paxos:**
- **Proposer:** เสนอ value ที่ต้องการให้ accept
- **Acceptor:** vote ว่า accept หรือ reject
- **Learner:** เรียนรู้ว่า consensus ตกลงที่ value อะไร

**2 Phases:**
```
Phase 1 — Prepare:
  Proposer → ส่ง PREPARE(n) ไปทุก Acceptors
  Acceptors → ถ้า n > เคยเห็นมา → ตอบ PROMISE(n)

Phase 2 — Accept:
  Proposer → รับ PROMISE จาก majority → ส่ง ACCEPT(n, value)
  Acceptors → ถ้า n ยังเป็น highest → ตอบ ACCEPTED
  
COMMIT เมื่อ majority ตอบ ACCEPTED
```

### 3. Raft Algorithm

Raft ออกแบบมาให้ "understandable" กว่า Paxos โดย Diego Ongaro ใน 2014

**3 States ของ Node:**
```
FOLLOWER  →  CANDIDATE  →  LEADER
    ↑              |            |
    └──────────────┴────────────┘
         (term หมด หรือ เลือกตั้งใหม่)
```

**Leader Election:**
```
1. ทุก node เริ่มเป็น FOLLOWER
2. ถ้าไม่ได้รับ heartbeat จาก leader ใน election timeout (150-300ms):
   - FOLLOWER → CANDIDATE
   - เพิ่ม term ของตัวเอง
   - Vote ให้ตัวเอง
   - ส่ง RequestVote ไปทุก node

3. node อื่นตอบ vote ถ้า:
   - ยังไม่ vote ใน term นี้
   - Log ของ candidate ไม่เก่ากว่าของตัวเอง

4. ถ้าได้ majority votes → เป็น LEADER
5. LEADER ส่ง heartbeat ทุก 50ms เพื่อ maintain leadership
```

**Log Replication:**
```
CLIENT → LEADER:  "เพิ่ม entry X"
LEADER:           บันทึก entry X ลง local log (uncommitted)
LEADER → FOLLOWERS: AppendEntries(entry X)
FOLLOWERS:        บันทึก entry X ลง local log
FOLLOWERS → LEADER: success

ถ้า majority ตอบ success:
LEADER:           COMMIT entry X (apply to state machine)
LEADER → FOLLOWERS: commit notification
FOLLOWERS:        COMMIT entry X
```

---

## 🛠️ Step-by-Step Implementation

### Step 921: ดู etcd ทำงานยังไงใน Kubernetes

```bash
# etcd คือ "source of truth" ของ Kubernetes
# ทุก configuration, state ของ cluster เก็บใน etcd

# ดู etcd cluster members
kubectl exec -n kube-system etcd-master-1 -- \
  etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# OUTPUT:
# 2c4d3a1b5e6f7890, started, master-1, https://10.0.0.1:2380, https://10.0.0.1:2379, false
# 3d5e4b2c6f7a8901, started, master-2, https://10.0.0.2:2380, https://10.0.0.2:2379, false  
# 4e6f5c3d7a8b9012, started, master-3, https://10.0.0.3:2380, https://10.0.0.3:2379, false

# ดู leader ปัจจุบัน
kubectl exec -n kube-system etcd-master-1 -- \
  etcdctl endpoint status \
  --endpoints=https://10.0.0.1:2379,https://10.0.0.2:2379,https://10.0.0.3:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --write-out=table
```

```bash
# Simulate leader failure — ดู election process
# (ทำบน test cluster เท่านั้น!)

# ดู current leader
LEADER_POD=$(kubectl exec -n kube-system etcd-master-1 -- \
  etcdctl endpoint status \
  --endpoints=https://10.0.0.1:2379 \
  --write-out=json | jq -r '.[].Status.leader')

echo "Current leader: $LEADER_POD"

# Kill leader pod
kubectl delete pod etcd-$LEADER_NODE -n kube-system

# รอ election ใหม่ (~150-300ms)
sleep 2

# ดู leader ใหม่
kubectl exec -n kube-system etcd-master-2 -- \
  etcdctl endpoint status \
  --endpoints=https://10.0.0.2:2379 \
  --write-out=json | jq '.[].Status'
```

### Step 922: PostgreSQL High Availability ด้วย Patroni

Patroni ใช้ etcd/Consul/ZooKeeper เป็น Distributed Lock ในการ elect PostgreSQL primary

```yaml
# patroni.yaml
scope: chuaikan-postgres
namespace: /db/
name: postgres-1

restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.0.1.10:8008

etcd3:
  hosts: 10.0.0.1:2379,10.0.0.2:2379,10.0.0.3:2379

bootstrap:
  dcs:
    ttl: 30              # Leader lease timeout
    loop_wait: 10        # Patroni loop interval  
    retry_timeout: 10    # Retry on DCS operation
    maximum_lag_on_failover: 1048576  # Max 1MB lag
    
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        wal_keep_size: 128MB
        max_wal_senders: 10
        max_replication_slots: 10
        
  initdb:
    - encoding: UTF8
    - data-checksums
    
postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.0.1.10:5432
  data_dir: /data/patroni
  
  authentication:
    replication:
      username: replicator
      password: "{{ REPLICATION_PASSWORD }}"
    superuser:
      username: postgres
      password: "{{ POSTGRES_PASSWORD }}"
```

```bash
# ติดตั้ง Patroni บน Kubernetes
helm repo add patroni https://patroni.github.io/patroni-chart
helm install patroni patroni/patroni \
  --values patroni.yaml \
  --namespace postgres \
  --create-namespace

# ดู cluster status
kubectl exec -n postgres patroni-0 -- \
  patronictl -c /etc/patroni/patroni.yml list

# OUTPUT:
# + Cluster: chuaikan-postgres +---------+----+-----------+
# | Member     | Host        | Role    | State   | TL | Lag in MB |
# +------------+-------------+---------+---------+----+-----------+
# | postgres-0 | 10.0.1.10:5432 | Leader | running |  1 |           |
# | postgres-1 | 10.0.1.11:5432 | Replica | running |  1 |       0.0 |
# | postgres-2 | 10.0.1.12:5432 | Replica | running |  1 |       0.0 |
```

### Step 923: Quorum Reads/Writes

```javascript
// quorum-client.js
// ใช้ Quorum เพื่อให้ได้ Strong Consistency โดยไม่ต้องรอทุก node

class QuorumDatabase {
  constructor(nodes, writeQuorum, readQuorum) {
    this.nodes = nodes;
    // W + R > N  →  Strong Consistency
    // Example: N=3, W=2, R=2 → 2+2 > 3 ✓
    this.writeQuorum = writeQuorum || Math.ceil(nodes.length / 2) + 1;
    this.readQuorum = readQuorum || Math.ceil(nodes.length / 2) + 1;
  }
  
  async write(key, value) {
    const timestamp = Date.now();
    const results = await Promise.allSettled(
      this.nodes.map(node => node.set(key, value, timestamp))
    );
    
    const successes = results.filter(r => r.status === 'fulfilled').length;
    
    if (successes < this.writeQuorum) {
      throw new Error(`Write quorum not met: ${successes}/${this.writeQuorum}`);
    }
    
    return { success: true, writtenTo: successes };
  }
  
  async read(key) {
    const results = await Promise.allSettled(
      this.nodes.map(node => node.get(key))
    );
    
    const successes = results
      .filter(r => r.status === 'fulfilled')
      .map(r => r.value);
    
    if (successes.length < this.readQuorum) {
      throw new Error(`Read quorum not met: ${successes.length}/${this.readQuorum}`);
    }
    
    // Return value with highest timestamp (latest)
    return successes.reduce((latest, current) => 
      current.timestamp > latest.timestamp ? current : latest
    );
  }
}

// ใช้งาน
const db = new QuorumDatabase(
  [node1, node2, node3],  // N=3
  2,                      // W=2
  2                       // R=2
);
// W(2) + R(2) > N(3) = Strong Consistency
```

### Step 924: Practical Consensus สำหรับ chuaikan.com

**เมื่อไรที่ chuaikan.com ต้องการ Consensus:**

```javascript
// 1. SOS Alert Status ต้องสอดคล้องกันทุก region
// ใช้ etcd เพื่อ coordinate SOS status

const { Etcd3 } = require('etcd3');
const etcd = new Etcd3({
  hosts: ['etcd-1:2379', 'etcd-2:2379', 'etcd-3:2379'],
});

async function updateSOSStatus(sosId, newStatus) {
  // ใช้ etcd transaction เพื่อ atomically update
  const result = await etcd
    .if(`sos/${sosId}/status`, 'Value', '==', 'active')
    .then(etcd.put(`sos/${sosId}/status`).value(newStatus))
    .else(etcd.get(`sos/${sosId}/status`))
    .commit();
  
  if (result.succeeded) {
    console.log(`SOS ${sosId} status updated to ${newStatus}`);
    return true;
  } else {
    const currentStatus = result.responses[0].kvs[0]?.value.toString();
    console.log(`SOS ${sosId} status is already ${currentStatus}, not active`);
    return false;
  }
}

// 2. Feature Flags ต้องสอดคล้องกัน (ไม่งั้น inconsistent UX)
async function setFeatureFlag(featureName, enabled) {
  // etcd watch — ทุก service instance จะได้รับ update ทันที
  await etcd
    .put(`features/${featureName}`)
    .value(enabled ? 'true' : 'false')
    .exec();
}

// Service instances watch for changes
etcd.watch()
  .prefix('features/')
  .create()
  .then(watcher => {
    watcher.on('put', event => {
      const featureName = event.key.toString().replace('features/', '');
      const enabled = event.value.toString() === 'true';
      featureFlags.set(featureName, enabled);
      console.log(`Feature ${featureName} is now ${enabled ? 'enabled' : 'disabled'}`);
    });
  });
```

---

## 🔧 Configuration Files

### etcd Backup สำหรับ Production

```bash
#!/bin/bash
# etcd-backup.sh — รัน daily

BACKUP_DIR="/backup/etcd/$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

ETCDCTL_API=3 etcdctl snapshot save $BACKUP_DIR/etcd-snapshot.db \
  --endpoints=https://etcd-0.etcd:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key

# Verify backup
ETCDCTL_API=3 etcdctl snapshot status $BACKUP_DIR/etcd-snapshot.db \
  --write-out=table

# Upload to S3
aws s3 cp $BACKUP_DIR/etcd-snapshot.db \
  s3://chuaikan-backups/etcd/$(date +%Y%m%d)/etcd-snapshot.db \
  --sse aws:kms

echo "Backup completed: $BACKUP_DIR/etcd-snapshot.db"
```

### Patroni Monitoring

```yaml
# prometheus-rules-patroni.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: patroni-alerts
spec:
  groups:
  - name: patroni
    rules:
    - alert: PatroniClusterHasNoLeader
      expr: max(patroni_master) == 0
      for: 30s
      labels:
        severity: critical
      annotations:
        summary: "PostgreSQL cluster has no leader!"
        description: "No Patroni master for more than 30 seconds"
    
    - alert: PatroniReplicaLagging
      expr: patroni_lag_in_mb > 50
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "PostgreSQL replica is lagging behind"
        description: "Replica lag is {{ $value }}MB"
```

---

## 🧪 Testing

### Test Leader Election

```bash
#!/bin/bash
# test-leader-election.sh

echo "=== Consensus Leader Election Test ==="

# 1. ดู leader ปัจจุบัน
LEADER=$(kubectl exec -n postgres patroni-0 -- \
  patronictl -c /etc/patroni/patroni.yml list -f json | \
  jq -r '.[] | select(.Role=="Leader") | .Member')
echo "Current leader: $LEADER"

# 2. Simulate leader failure
echo "Simulating leader failure..."
kubectl delete pod $LEADER -n postgres --force

# 3. วัดเวลา election
START=$(date +%s%3N)

# รอจนกว่าจะมี leader ใหม่
while true; do
  NEW_LEADER=$(kubectl exec -n postgres patroni-0 -- \
    patronictl -c /etc/patroni/patroni.yml list -f json 2>/dev/null | \
    jq -r '.[] | select(.Role=="Leader") | .Member')
  
  if [ -n "$NEW_LEADER" ] && [ "$NEW_LEADER" != "$LEADER" ]; then
    END=$(date +%s%3N)
    ELECTION_TIME=$((END - START))
    echo "New leader elected: $NEW_LEADER"
    echo "Election time: ${ELECTION_TIME}ms"
    break
  fi
  
  sleep 0.1
done

# 4. ตรวจว่า writes ยังทำงานได้
echo "Testing writes after election..."
kubectl exec -n postgres patroni-0 -- \
  psql -U postgres -c "INSERT INTO health_check (ts) VALUES (NOW())"

echo "✅ Leader election test passed"
```

---

## ❌ Common Errors & Solutions

### Error 1: etcd "no leader" error

```
Error: etcdserver: no leader
```

**สาเหตุ:** Network partition ทำให้ไม่มี majority → ไม่สามารถ elect leader ได้

```bash
# ตรวจ etcd member status
ETCDCTL_API=3 etcdctl \
  --endpoints=https://etcd-0:2379,https://etcd-1:2379,https://etcd-2:2379 \
  endpoint health

# ถ้า 2 ใน 3 nodes healthy → ควร elect leader ได้
# ถ้าน้อยกว่า majority → ต้องแก้ network ก่อน

# Force new election (ระวัง! ใช้เฉพาะ emergency)
ETCDCTL_API=3 etcdctl \
  --endpoints=https://etcd-0:2379 \
  move-leader MEMBER_ID
```

### Error 2: Patroni "failover" ไม่สำเร็จ

```
2024/01/01 12:00:00 UTC Could not failover: quorum check failed
```

```bash
# ตรวจ DCS (etcd) connectivity จาก patroni
kubectl exec -n postgres patroni-0 -- \
  patronictl -c /etc/patroni/patroni.yml failover \
  --master postgres-0 \
  --candidate postgres-1

# ถ้า etcd ไม่ connect: ตรวจ network policy
kubectl get networkpolicy -n postgres
kubectl describe networkpolicy -n postgres

# แก้: อนุญาต traffic จาก postgres namespace ไป etcd
```

---

## ✅ Checklist

- [ ] etcd cluster มี 3 nodes (odd number)
- [ ] Patroni ตั้งค่าสำหรับ PostgreSQL HA แล้ว
- [ ] Leader election time < 10 seconds (ทดสอบแล้ว)
- [ ] etcd backup daily ไป S3
- [ ] etcd backup restore ทดสอบแล้ว (quarterly)
- [ ] Patroni Prometheus metrics expose แล้ว
- [ ] Alert สำหรับ "no leader" และ "high replica lag"
- [ ] Runbook สำหรับ manual failover เขียนแล้ว
- [ ] Network policy อนุญาต etcd traffic แล้ว
- [ ] Split brain scenario ทดสอบแล้ว (บน staging)

---

## 🔗 References

- [Raft Consensus Algorithm](https://raft.github.io/)
- [The Raft Paper — Ongaro & Ousterhout 2014](https://raft.github.io/raft.pdf)
- [etcd Documentation](https://etcd.io/docs/)
- [Patroni Documentation](https://patroni.readthedocs.io/)
- [Paxos Made Simple — Leslie Lamport](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf)
- [FLP Impossibility Theorem](https://dl.acm.org/doi/10.1145/3149.214121)

---

*Part 093 | Road to 1,000,000 Users/Day | chuaikan.com*
