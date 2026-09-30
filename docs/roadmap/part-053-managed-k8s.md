# Part 053: EKS / GKE (Managed Kubernetes)
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Advanced
> **Steps:** 521-530
> **เวลาโดยประมาณ:** 6 ชั่วโมง
> **Prerequisites:** Part 052 (VPC Network), Part 030-040 (Kubernetes basics)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- สร้าง EKS cluster ด้วย eksctl
- ตั้งค่า Node Groups: general purpose, memory-optimized, spot instances
- เข้าใจความแตกต่าง EKS Managed Node Groups vs Fargate
- เขียน eksctl cluster.yaml สำหรับ chuaikan.com
- ตั้งค่า IRSA (IAM Roles for Service Accounts) สำหรับ pod-level permissions
- ติดตั้ง EKS Add-ons: CoreDNS, kube-proxy, VPC CNI, EBS CSI Driver
- ตั้งค่า Cluster Autoscaler

---

## 📖 ทฤษฎีและแนวคิด

### Step 521: EKS Architecture

**Amazon EKS (Elastic Kubernetes Service)** คือ Managed Kubernetes ที่ AWS ดูแล Control Plane ให้

```
EKS Architecture:
┌─────────────────────────────────────────────────────────┐
│  EKS Control Plane (AWS Managed)                        │
│  ├── kube-apiserver                                      │
│  ├── etcd                                                │
│  ├── kube-scheduler                                      │
│  └── kube-controller-manager                             │
└──────────────────────────────┬──────────────────────────┘
                               │ API calls
┌──────────────────────────────▼──────────────────────────┐
│  Worker Nodes (VPC ของเรา)                               │
│  ├── Node Group: general (t3.large)                      │
│  │   └── Pods: Web, API, Background jobs                 │
│  ├── Node Group: memory (r6g.xlarge)                     │
│  │   └── Pods: Redis-dependent services                  │
│  └── Node Group: spot (t3.large-spot)                    │
│      └── Pods: Batch jobs, non-critical workloads        │
└─────────────────────────────────────────────────────────┘
```

### Step 522: Node Type Selection

| Instance | vCPU | RAM | On-Demand Price/hr | Use Case |
|----------|------|-----|-------------------|----------|
| t3.large | 2 | 8GB | $0.0928 | General purpose |
| t3.xlarge | 4 | 16GB | $0.1856 | Moderate workloads |
| m6g.large | 2 | 8GB | $0.0770 | ARM, better price/perf |
| r6g.large | 2 | 16GB | $0.1008 | Memory-intensive |
| r6g.xlarge | 4 | 32GB | $0.2016 | Redis-heavy workloads |
| c6g.xlarge | 4 | 8GB | $0.1360 | CPU-intensive |

**สำหรับ chuaikan.com:**
- **General nodes**: m6g.large (ARM, better price/perf)
- **Memory nodes**: r6g.xlarge (สำหรับ services ที่ใช้ Redis มาก)
- **Spot nodes**: t3.large Spot (ประหยัด 70% สำหรับ batch jobs)

### Step 523: EKS vs Fargate

| Feature | EKS Managed Node Groups | EKS Fargate |
|---------|------------------------|-------------|
| Infrastructure | ต้องดูแล EC2 | Serverless, AWS ดูแล |
| Cost | On-demand/Spot pricing | Per vCPU + memory |
| Startup time | เร็ว (node พร้อมอยู่) | ช้า (cold start ~30-60s) |
| Customization | ได้ (ติดตั้ง tools ได้) | จำกัด |
| DaemonSets | รองรับ | ไม่รองรับ |
| Persistent Volumes | รองรับ | จำกัด (EFS only) |
| **Best for** | Production workloads | Dev/test, microservices |

**สรุป: chuaikan.com ใช้ Managed Node Groups เป็นหลัก + Fargate สำหรับ jobs ชั่วคราว**

---

## ⚙️ Environment Setup

### ติดตั้ง eksctl

```bash
# Linux/WSL
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# macOS
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl

# ตรวจสอบ version
eksctl version
# 0.167.0
```

### ติดตั้ง kubectl

```bash
# Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# macOS
brew install kubectl

kubectl version --client
# Client Version: v1.29.0
```

### ติดตั้ง Helm

```bash
# Linux/macOS
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm version
# version.BuildInfo{Version:"v3.14.0"}
```

---

## 🛠️ Step-by-Step Implementation

### Step 524: สร้าง EKS Cluster ด้วย eksctl

```bash
# สร้าง cluster จาก config file
eksctl create cluster -f cluster.yaml

# หรือ quick create (สำหรับทดสอบ)
eksctl create cluster \
  --name chuaikan-prod \
  --region ap-southeast-1 \
  --nodegroup-name general \
  --node-type m6g.large \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 10 \
  --managed \
  --amd64
```

### Step 525: ตรวจสอบ Cluster

```bash
# ดู cluster info
eksctl get cluster --region ap-southeast-1

# Update kubeconfig
aws eks update-kubeconfig --region ap-southeast-1 --name chuaikan-prod

# ทดสอบ connection
kubectl get nodes
# NAME                STATUS   ROLES    AGE   VERSION
# ip-10-0-10-123...   Ready    <none>   5m    v1.29.0-eks-xxxxxx

# ดู node details
kubectl describe node ip-10-0-10-123...
```

### Step 526: ตั้งค่า IRSA (IAM Roles for Service Accounts)

```bash
# เปิด OIDC Provider สำหรับ cluster
eksctl utils associate-iam-oidc-provider \
  --region ap-southeast-1 \
  --cluster chuaikan-prod \
  --approve

# ตรวจสอบ OIDC provider
aws iam list-open-id-connect-providers | grep $(aws eks describe-cluster \
  --name chuaikan-prod \
  --query "cluster.identity.oidc.issuer" \
  --output text | cut -d '/' -f 5)

# สร้าง IAM Service Account สำหรับ S3 access
eksctl create iamserviceaccount \
  --name s3-access-sa \
  --namespace production \
  --cluster chuaikan-prod \
  --region ap-southeast-1 \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess \
  --approve \
  --override-existing-serviceaccounts

# ใช้ใน Pod
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: s3-test-pod
  namespace: production
spec:
  serviceAccountName: s3-access-sa
  containers:
  - name: aws-cli
    image: amazon/aws-cli
    command: ["aws", "s3", "ls"]
EOF
```

### Step 527: ติดตั้ง EKS Add-ons

```bash
# ติดตั้ง EBS CSI Driver (สำหรับ Persistent Volumes)
eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster chuaikan-prod \
  --region ap-southeast-1 \
  --service-account-role-arn arn:aws:iam::123456789012:role/AmazonEBSCSIDriverRole \
  --force

# ติดตั้ง VPC CNI (สำหรับ network)
eksctl create addon \
  --name vpc-cni \
  --cluster chuaikan-prod \
  --region ap-southeast-1 \
  --force

# ติดตั้ง CoreDNS
eksctl create addon \
  --name coredns \
  --cluster chuaikan-prod \
  --region ap-southeast-1

# ตรวจสอบ add-ons
eksctl get addon --cluster chuaikan-prod --region ap-southeast-1
# NAME                    VERSION        STATUS
# coredns                 v1.11.1-eksbuild.4  ACTIVE
# kube-proxy              v1.29.0-eksbuild.2  ACTIVE  
# vpc-cni                 v1.16.4-eksbuild.2  ACTIVE
# aws-ebs-csi-driver      v1.26.1-eksbuild.1  ACTIVE
```

### Step 528: ติดตั้ง Cluster Autoscaler

```bash
# สร้าง IAM Policy สำหรับ Autoscaler
cat > cluster-autoscaler-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeScalingActivities",
        "autoscaling:DescribeTags",
        "ec2:DescribeInstanceTypes",
        "ec2:DescribeLaunchTemplateVersions"
      ],
      "Resource": ["*"]
    },
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup",
        "ec2:DescribeImages",
        "ec2:GetInstanceTypesFromInstanceRequirements",
        "eks:DescribeNodegroup"
      ],
      "Resource": ["*"]
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name ChuaikanClusterAutoscalerPolicy \
  --policy-document file://cluster-autoscaler-policy.json

# สร้าง Service Account
eksctl create iamserviceaccount \
  --name cluster-autoscaler \
  --namespace kube-system \
  --cluster chuaikan-prod \
  --region ap-southeast-1 \
  --attach-policy-arn arn:aws:iam::123456789012:policy/ChuaikanClusterAutoscalerPolicy \
  --approve \
  --override-existing-serviceaccounts

# ติดตั้ง Cluster Autoscaler ด้วย Helm
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm repo update

helm upgrade --install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=chuaikan-prod \
  --set awsRegion=ap-southeast-1 \
  --set rbac.serviceAccount.create=false \
  --set rbac.serviceAccount.name=cluster-autoscaler \
  --set extraArgs.balance-similar-node-groups=true \
  --set extraArgs.skip-nodes-with-system-pods=false
```

---

## 🔧 Configuration Files

### eksctl cluster.yaml

```yaml
# cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: chuaikan-prod
  region: ap-southeast-1
  version: "1.29"
  tags:
    Environment: production
    Project: chuaikan

iam:
  withOIDC: true
  serviceAccounts:
  - metadata:
      name: aws-load-balancer-controller
      namespace: kube-system
    wellKnownPolicies:
      awsLoadBalancerController: true
  - metadata:
      name: cluster-autoscaler
      namespace: kube-system
    wellKnownPolicies:
      autoScaler: true
  - metadata:
      name: ebs-csi-controller-sa
      namespace: kube-system
    wellKnownPolicies:
      ebsCSIController: true

vpc:
  id: "vpc-xxxxxxxxxxxxxxxxx"
  subnets:
    private:
      ap-southeast-1a:
        id: "subnet-xxxxxxxxxxxxxxxxx"
      ap-southeast-1b:
        id: "subnet-xxxxxxxxxxxxxxxxy"
    public:
      ap-southeast-1a:
        id: "subnet-xxxxxxxxxxxxxxxxa"
      ap-southeast-1b:
        id: "subnet-xxxxxxxxxxxxxxxxb"

managedNodeGroups:
  # General Purpose Node Group
  - name: general
    instanceType: m6g.large
    amiFamily: AmazonLinux2023
    minSize: 2
    maxSize: 20
    desiredCapacity: 3
    volumeSize: 50
    volumeType: gp3
    privateNetworking: true
    availabilityZones: ["ap-southeast-1a", "ap-southeast-1b"]
    labels:
      role: general
      workload: standard
    taints: []
    tags:
      NodeGroup: general
      k8s.io/cluster-autoscaler/enabled: "true"
      k8s.io/cluster-autoscaler/chuaikan-prod: "owned"
    iam:
      withAddonPolicies:
        imageBuilder: true
        cloudWatch: true
        albIngress: true
    updateConfig:
      maxUnavailablePercentage: 25

  # Memory-Optimized Node Group (สำหรับ services ที่ใช้ memory เยอะ)
  - name: memory
    instanceType: r6g.xlarge
    amiFamily: AmazonLinux2023
    minSize: 1
    maxSize: 5
    desiredCapacity: 2
    volumeSize: 50
    volumeType: gp3
    privateNetworking: true
    availabilityZones: ["ap-southeast-1a", "ap-southeast-1b"]
    labels:
      role: memory
      workload: memory-intensive
    taints:
    - key: workload
      value: memory-intensive
      effect: NoSchedule
    tags:
      NodeGroup: memory
      k8s.io/cluster-autoscaler/enabled: "true"
      k8s.io/cluster-autoscaler/chuaikan-prod: "owned"

  # Spot Instance Node Group (สำหรับ batch jobs, ประหยัดค่าใช้จ่าย)
  - name: spot
    instanceTypes:
    - t3.large
    - t3.xlarge
    - t3a.large
    spot: true
    minSize: 0
    maxSize: 10
    desiredCapacity: 0
    privateNetworking: true
    availabilityZones: ["ap-southeast-1a", "ap-southeast-1b"]
    labels:
      role: spot
      workload: batch
    taints:
    - key: workload
      value: batch
      effect: NoSchedule
    tags:
      NodeGroup: spot
      k8s.io/cluster-autoscaler/enabled: "true"
      k8s.io/cluster-autoscaler/chuaikan-prod: "owned"

addons:
- name: vpc-cni
  version: latest
  resolveConflicts: overwrite
- name: coredns
  version: latest
  resolveConflicts: overwrite
- name: kube-proxy
  version: latest
  resolveConflicts: overwrite
- name: aws-ebs-csi-driver
  version: latest
  resolveConflicts: overwrite

cloudWatch:
  clusterLogging:
    enableTypes:
    - api
    - audit
    - authenticator
    - controllerManager
    - scheduler
    logRetentionInDays: 30
```

### StorageClass for EBS

```yaml
# storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: true
```

### Namespace Setup

```yaml
# namespaces.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    environment: staging
---
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    purpose: monitoring
```

---

## 🧪 Testing

```bash
# ทดสอบ cluster health
kubectl get componentstatus
kubectl get nodes -o wide

# ทดสอบ Pod scheduling
kubectl run test-pod --image=nginx --restart=Never -n production
kubectl get pod test-pod -n production
kubectl delete pod test-pod -n production

# ทดสอบ autoscaler
kubectl -n kube-system logs -l app.kubernetes.io/name=cluster-autoscaler | grep -i scale

# ทดสอบ EBS PVC
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
  namespace: production
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: gp3
  resources:
    requests:
      storage: 1Gi
EOF

kubectl get pvc test-pvc -n production
# NAME       STATUS   VOLUME         CAPACITY   ACCESS MODES   STORAGECLASS
# test-pvc   Bound    pvc-xxxxxxxx   1Gi        RWO            gp3

kubectl delete pvc test-pvc -n production

# ทดสอบ IRSA
kubectl -n production create serviceaccount test-irsa-sa
kubectl -n production get serviceaccount test-irsa-sa -o yaml

# ทดสอบ node affinity
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-general
  namespace: production
spec:
  nodeSelector:
    role: general
  containers:
  - name: test
    image: nginx
EOF
kubectl get pod test-general -n production -o wide
kubectl delete pod test-general -n production
```

---

## ❌ Common Errors & Solutions

### Error 1: "nodes are not ready"

```
The connection to the server was refused
```

**แก้ไข:**
```bash
# ตรวจสอบ cluster status
eksctl get cluster --region ap-southeast-1

# Update kubeconfig
aws eks update-kubeconfig --region ap-southeast-1 --name chuaikan-prod

# ตรวจสอบ node status
kubectl get nodes
kubectl describe node <node-name>
```

### Error 2: Pods stuck in Pending

```
0/3 nodes are available: 3 Insufficient memory
```

**แก้ไข:**
```bash
# ดู resource usage
kubectl top nodes

# ดู pending reason
kubectl describe pod <pod-name> -n production

# Check autoscaler logs
kubectl logs -n kube-system -l app.kubernetes.io/name=cluster-autoscaler --tail=50
```

### Error 3: EBS volume เก็บ sts ไม่ได้

```
AttachVolume.Attach failed for volume: timed out waiting for volume to attach
```

**แก้ไข:**
```bash
# ตรวจสอบ EBS CSI driver
kubectl get pods -n kube-system | grep ebs

# ตรวจสอบ IRSA annotation
kubectl describe serviceaccount ebs-csi-controller-sa -n kube-system
# ต้องมี: eks.amazonaws.com/role-arn: arn:aws:iam::...

# ตรวจสอบ IAM permissions
aws iam get-role --role-name AmazonEBSCSIDriverRole
```

---

## ✅ Checklist

- [ ] Step 521: เข้าใจ EKS Architecture
- [ ] Step 522: เลือก Node Type ที่เหมาะสม
- [ ] Step 523: เปรียบเทียบ Managed Node Groups vs Fargate
- [ ] Step 524: ติดตั้ง eksctl และสร้าง EKS cluster
- [ ] Step 525: ตั้งค่า kubeconfig และทดสอบ connection
- [ ] Step 526: ตั้งค่า IRSA สำหรับ pod-level permissions
- [ ] Step 527: ติดตั้ง EKS Add-ons
- [ ] Step 528: ติดตั้ง Cluster Autoscaler
- [ ] Step 529: สร้าง StorageClass และ Namespaces
- [ ] Step 530: ทดสอบ autoscaling และ PVC

---

## 🔗 References

- [EKS Getting Started](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html)
- [eksctl Documentation](https://eksctl.io/)
- [IRSA Setup](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)

---
*Part 053 | Road to 1,000,000 Users/Day | chuaikan.com*
