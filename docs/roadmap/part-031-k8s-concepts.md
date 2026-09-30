# Part 031: Kubernetes Architecture & Concepts

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 301-310
> **เวลาโดยประมาณ:** 3 ชั่วโมง
> **Prerequisites:** Part 001-030 (Docker, CI/CD, Infrastructure basics)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- เข้าใจสถาปัตยกรรม Kubernetes ทั้ง Control Plane และ Worker Nodes
- รู้จัก K8s Objects ทุกตัวที่จำเป็นสำหรับ chuaikan.com
- ตั้งค่า Namespace สำหรับแยก environment (dev/staging/production)
- ติดตั้ง kubectl และใช้คำสั่งพื้นฐาน
- ตั้งค่า kubeconfig สำหรับจัดการหลาย cluster
- เข้าใจ Kubernetes Networking Model (CNI)

---

## 📖 ทฤษฎีและแนวคิด

### Step 301 — ทำไม chuaikan.com ต้องใช้ Kubernetes?

เมื่อ chuaikan.com เติบโตถึง 1,000,000 users/day เราต้องการ:
- **High Availability**: ไม่มี single point of failure
- **Auto-scaling**: scale up/down อัตโนมัติตาม traffic
- **Zero-downtime deployment**: deploy ใหม่โดยไม่ต้อง downtime
- **Resource efficiency**: จัดสรร CPU/Memory อย่างมีประสิทธิภาพ
- **Self-healing**: restart containers ที่พัง, schedule workload ใหม่

### Step 302 — Control Plane Components

```
Control Plane คือ "สมองหลัก" ของ K8s cluster
```

**API Server (`kube-apiserver`)**
- จุดเชื่อมต่อหลักของ cluster ทุกอย่าง
- รับ request จาก kubectl, controllers, kubelet
- validate และ process REST API requests
- port: 6443 (HTTPS)

**etcd**
- Distributed key-value store
- เก็บ state ทั้งหมดของ cluster
- เป็น single source of truth
- ต้อง backup อย่างสม่ำเสมอ!

**Scheduler (`kube-scheduler`)**
- ตัดสินใจว่า Pod จะรันบน Node ไหน
- พิจารณา: resource requests, affinity rules, taints/tolerations
- ยังไม่ได้รัน Pod จริง แค่ "assign" ให้ Node

**Controller Manager (`kube-controller-manager`)**
- รัน control loops หลายตัวพร้อมกัน:
  - Node controller: ตรวจสอบ node health
  - Replication controller: รักษาจำนวน Pod
  - Endpoints controller: จัดการ Service endpoints
  - Service Account controller

### Step 303 — Worker Node Components

**kubelet**
- Agent รันบนทุก worker node
- รับ PodSpec จาก API server
- สั่ง container runtime ให้รัน/หยุด containers
- รายงานสถานะ node และ pod กลับไปยัง API server

**kube-proxy**
- รันบนทุก node
- จัดการ network rules (iptables/ipvs)
- ทำให้ Service สามารถเข้าถึง Pods ได้
- Load balance traffic ระหว่าง Pod endpoints

**Container Runtime**
- รัน containers จริง: containerd, CRI-O
- คุยกับ kubelet ผ่าน CRI (Container Runtime Interface)

### Step 304 — ASCII Diagram ของ chuaikan.com K8s Architecture

```
                    ┌─────────────────────────────────────────────────┐
                    │              CONTROL PLANE                       │
                    │  ┌──────────┐ ┌──────┐ ┌───────────────────┐   │
                    │  │  API     │ │ etcd │ │ Controller Manager│   │
                    │  │ Server   │ │      │ │ + Scheduler        │   │
                    │  └────┬─────┘ └──────┘ └───────────────────┘   │
                    └───────┼─────────────────────────────────────────┘
                            │ HTTPS :6443
               ┌────────────┼────────────┐
               │            │            │
    ┌──────────▼──┐  ┌───────▼──┐  ┌────▼──────────┐
    │  Worker-1   │  │ Worker-2 │  │   Worker-3    │
    │             │  │          │  │               │
    │ ┌─────────┐ │  │ ┌──────┐ │  │ ┌───────────┐ │
    │ │ Next.js │ │  │ │ API  │ │  │ │PostgreSQL │ │
    │ │  Pod x2 │ │  │ │ Pod  │ │  │ │   Pod     │ │
    │ └─────────┘ │  │ └──────┘ │  │ └───────────┘ │
    │ ┌─────────┐ │  │ ┌──────┐ │  │ ┌───────────┐ │
    │ │  Redis  │ │  │ │ Next │ │  │ │  kubelet  │ │
    │ │   Pod   │ │  │ │  .js │ │  │ │kube-proxy │ │
    │ └─────────┘ │  │ └──────┘ │  │ └───────────┘ │
    │  kubelet    │  │ kubelet  │  │               │
    │  kube-proxy │  │kube-proxy│  │               │
    └─────────────┘  └──────────┘  └───────────────┘
           │                │               │
    ┌──────▼────────────────▼───────────────▼──────┐
    │                  CNI Network                   │
    │              (Calico / Flannel)                │
    └───────────────────────────────────────────────┘
```

### Step 305 — Kubernetes Objects ที่สำคัญ

**Pod**
- Unit เล็กที่สุดของ K8s
- รัน 1+ containers ที่แชร์ network และ storage
- มี IP address เป็นของตัวเอง
- ephemeral — ถ้าตายแล้วไม่ฟื้นขึ้นมาเอง

**ReplicaSet**
- รักษาจำนวน Pod replicas ที่กำหนด
- ถ้า Pod ตาย จะสร้างใหม่
- ไม่ควรใช้ตรงๆ ใช้ Deployment แทน

**Deployment**
- จัดการ ReplicaSet
- รองรับ rolling update, rollback
- ใช้สำหรับ stateless applications เช่น Next.js, Node.js API

**Service**
- Stable endpoint สำหรับเข้าถึง Pods
- Load balance ระหว่าง Pod replicas
- Types: ClusterIP, NodePort, LoadBalancer, ExternalName

**Ingress**
- จัดการ HTTP/HTTPS traffic จากภายนอกเข้า cluster
- Route based on hostname/path
- ต้องมี Ingress Controller (nginx-ingress)

**ConfigMap**
- เก็บ non-sensitive configuration data
- mount เป็น env vars หรือ files

**Secret**
- เก็บ sensitive data (passwords, API keys)
- เข้ารหัส base64 (ไม่ใช่ encryption จริง!)
- ควรใช้ Sealed Secrets หรือ External Secrets

**PersistentVolume (PV) & PersistentVolumeClaim (PVC)**
- จัดการ persistent storage สำหรับ stateful apps
- PV คือ storage จริง, PVC คือ request ขอใช้ storage

**HorizontalPodAutoscaler (HPA)**
- Auto-scale Deployment based on metrics
- CPU, Memory, Custom metrics (RPS)

### Step 306 — Namespace สำหรับ chuaikan.com

```
Namespace = virtual cluster ภายใน cluster จริง
ใช้แยก environment ไม่ให้กระทบกัน
```

---

## ⚙️ Environment Setup

### Step 307 — ติดตั้ง kubectl บน Ubuntu 24.04

```bash
# ดาวน์โหลด kubectl binary
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# ตรวจสอบ checksum
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check

# ติดตั้ง
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# ตรวจสอบ version
kubectl version --client
```

---

## 🛠️ Step-by-Step Implementation

### Step 308 — สร้าง Namespaces สำหรับ chuaikan.com

```bash
# สร้าง namespaces
kubectl create namespace chuaikan-dev
kubectl create namespace chuaikan-staging
kubectl create namespace chuaikan-production

# ดู namespaces ทั้งหมด
kubectl get namespaces

# ตั้ง default namespace (เพื่อไม่ต้องพิมพ์ -n ทุกครั้ง)
kubectl config set-context --current --namespace=chuaikan-production
```

ไฟล์ `namespaces.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: chuaikan-dev
  labels:
    env: development
    team: chuaikan
---
apiVersion: v1
kind: Namespace
metadata:
  name: chuaikan-staging
  labels:
    env: staging
    team: chuaikan
---
apiVersion: v1
kind: Namespace
metadata:
  name: chuaikan-production
  labels:
    env: production
    team: chuaikan
```

```bash
kubectl apply -f namespaces.yaml
```

### Step 309 — kubeconfig Setup สำหรับหลาย Clusters

```bash
# ดู kubeconfig ปัจจุบัน
kubectl config view

# ดู contexts ทั้งหมด
kubectl config get-contexts

# เพิ่ม cluster ใหม่ (staging)
kubectl config set-cluster chuaikan-staging \
  --server=https://staging.k8s.chuaikan.com:6443 \
  --certificate-authority=/path/to/ca.crt

# เพิ่ม credentials
kubectl config set-credentials chuaikan-admin \
  --client-certificate=/path/to/client.crt \
  --client-key=/path/to/client.key

# สร้าง context
kubectl config set-context chuaikan-staging \
  --cluster=chuaikan-staging \
  --user=chuaikan-admin \
  --namespace=chuaikan-staging

# switch context
kubectl config use-context chuaikan-staging

# ดู context ปัจจุบัน
kubectl config current-context
```

ไฟล์ `~/.kube/config` ตัวอย่าง:

```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    server: https://prod.k8s.chuaikan.com:6443
    certificate-authority-data: LS0tLS1CRUdJT...
  name: chuaikan-production
- cluster:
    server: https://staging.k8s.chuaikan.com:6443
    certificate-authority-data: LS0tLS1CRUdJT...
  name: chuaikan-staging
contexts:
- context:
    cluster: chuaikan-production
    namespace: chuaikan-production
    user: admin-prod
  name: prod
- context:
    cluster: chuaikan-staging
    namespace: chuaikan-staging
    user: admin-staging
  name: staging
current-context: prod
users:
- name: admin-prod
  user:
    client-certificate-data: LS0tLS1CRUdJT...
    client-key-data: LS0tLS1CRUdJT...
```

### Step 310 — K8s Networking Model (CNI)

**CNI (Container Network Interface)** คือ standard ที่กำหนดว่า network plugins ทำงานอย่างไรใน K8s

K8s Network Model มีกฎ 3 ข้อ:
1. Pods สามารถคุยกันได้โดยตรง (ไม่ผ่าน NAT) แม้อยู่คนละ Node
2. Nodes สามารถคุยกับ Pods ได้โดยตรง
3. Pod เห็น IP ของตัวเองตรงกับที่ Pods อื่นเห็น

```bash
# ดู pod network CIDR
kubectl cluster-info dump | grep -m 1 cluster-cidr

# ดู IP ของ pods ทั้งหมด
kubectl get pods -o wide -n chuaikan-production

# ตรวจสอบ CNI ที่ใช้
ls /etc/cni/net.d/
cat /etc/cni/net.d/10-calico.conflist
```

---

## 🔧 Configuration Files

### kubectl bash completion

```bash
# เพิ่ม bash completion
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
source ~/.bashrc

# ทดสอบ
k get pods
k get nodes
```

### ติดตั้ง kubectx และ kubens (ช่วย switch context/namespace)

```bash
# ติดตั้ง kubectx + kubens
sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -s /opt/kubectx/kubectx /usr/local/bin/kubectx
sudo ln -s /opt/kubectx/kubens /usr/local/bin/kubens

# ใช้งาน
kubectx                    # ดู contexts ทั้งหมด
kubectx prod               # switch ไป production
kubens chuaikan-production # switch namespace
```

### คำสั่ง kubectl ที่ใช้บ่อย

```bash
# --- Cluster Info ---
kubectl cluster-info
kubectl get nodes -o wide
kubectl describe node worker-1

# --- Pods ---
kubectl get pods -n chuaikan-production
kubectl get pods --all-namespaces
kubectl describe pod <pod-name> -n chuaikan-production
kubectl logs <pod-name> -n chuaikan-production
kubectl logs <pod-name> -c <container-name> --previous
kubectl exec -it <pod-name> -- /bin/sh

# --- Deployments ---
kubectl get deployments -n chuaikan-production
kubectl describe deployment nextjs-app
kubectl rollout status deployment/nextjs-app

# --- Services ---
kubectl get services -n chuaikan-production
kubectl describe service nextjs-service

# --- Events (ดู error) ---
kubectl get events -n chuaikan-production --sort-by='.lastTimestamp'

# --- Resource usage ---
kubectl top nodes
kubectl top pods -n chuaikan-production
```

---

## 🧪 Testing

### ทดสอบ kubectl เชื่อม cluster ได้

```bash
# ตรวจสอบการเชื่อมต่อ
kubectl cluster-info

# ตรวจสอบ nodes
kubectl get nodes

# คาดหวัง output:
# NAME       STATUS   ROLES           AGE   VERSION
# master-1   Ready    control-plane   1d    v1.29.0
# worker-1   Ready    <none>          1d    v1.29.0
# worker-2   Ready    <none>          1d    v1.29.0
```

### ทดสอบสร้าง Pod แรก (ทดสอบ cluster ทำงาน)

```bash
# สร้าง test pod
kubectl run test-nginx \
  --image=nginx:alpine \
  --namespace=chuaikan-dev \
  --restart=Never

# ตรวจสอบ status
kubectl get pod test-nginx -n chuaikan-dev

# ดู logs
kubectl logs test-nginx -n chuaikan-dev

# เข้าไปใน pod
kubectl exec -it test-nginx -n chuaikan-dev -- sh

# ลบ pod
kubectl delete pod test-nginx -n chuaikan-dev
```

---

## ❌ Common Errors & Solutions

### Error: `The connection to the server was refused`
```bash
# สาเหตุ: kubectl ไม่สามารถเชื่อมต่อ API server
# แก้: ตรวจสอบ kubeconfig
kubectl config view
echo $KUBECONFIG

# ถ้าไม่ได้ set ตรวจสอบ default path
ls ~/.kube/config
```

### Error: `No resources found in default namespace`
```bash
# สาเหตุ: ทำงานอยู่ใน namespace ผิด
# แก้: ระบุ namespace ให้ถูก
kubectl get pods -n chuaikan-production

# หรือ switch default namespace
kubectl config set-context --current --namespace=chuaikan-production
```

### Error: `Unable to connect to the server: x509 certificate`
```bash
# สาเหตุ: certificate หมดอายุหรือ mismatch
# ตรวจสอบ certificate
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates

# renew certificate (ถ้าใช้ kubeadm)
sudo kubeadm certs renew all
sudo systemctl restart kube-apiserver
```

---

## ✅ Checklist

- [ ] **Step 301**: เข้าใจว่าทำไม chuaikan.com ต้องใช้ K8s
- [ ] **Step 302**: อธิบาย Control Plane components ได้: API Server, etcd, Scheduler, Controller Manager
- [ ] **Step 303**: อธิบาย Worker Node components ได้: kubelet, kube-proxy, container runtime
- [ ] **Step 304**: วาด ASCII diagram ของ K8s architecture ได้
- [ ] **Step 305**: รู้จัก K8s objects: Pod, ReplicaSet, Deployment, Service, Ingress, ConfigMap, Secret, PV/PVC, HPA
- [ ] **Step 306**: เข้าใจ Namespace และประโยชน์ในการแยก environment
- [ ] **Step 307**: ติดตั้ง kubectl บน Ubuntu 24.04 สำเร็จ
- [ ] **Step 308**: สร้าง namespaces: chuaikan-dev, chuaikan-staging, chuaikan-production
- [ ] **Step 309**: ตั้งค่า kubeconfig สำหรับหลาย clusters และ switch contexts ได้
- [ ] **Step 310**: เข้าใจ K8s Networking Model และ CNI

---

## 🔗 References

- [Kubernetes Official Documentation](https://kubernetes.io/docs/concepts/)
- [K8s Architecture Overview](https://kubernetes.io/docs/concepts/overview/components/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Kubernetes Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)

---

*Part 031 | Road to 1,000,000 Users/Day | chuaikan.com*
