# Part 032: การติดตั้ง K8s ด้วย kubeadm

## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Intermediate
> **Steps:** 311-320
> **เวลาโดยประมาณ:** 4 ชั่วโมง
> **Prerequisites:** Part 031 (K8s Concepts), Ubuntu 24.04 LTS servers (3 nodes)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- เตรียม system prerequisites สำหรับ K8s บน Ubuntu 24.04
- ติดตั้ง containerd v1.7.x เป็น container runtime
- ติดตั้ง kubeadm, kubelet, kubectl v1.29
- สร้าง K8s cluster ด้วย `kubeadm init`
- เพิ่ม worker nodes เข้า cluster
- ติดตั้ง Calico CNI
- ติดตั้ง MetalLB สำหรับ bare metal load balancer
- ตรวจสอบ cluster health

---

## 📖 ทฤษฎีและแนวคิด

### Step 311 — ทำความเข้าใจ kubeadm

**kubeadm** คือ tool ที่ช่วย bootstrap K8s cluster โดยทำ:
- `kubeadm init`: สร้าง control plane
- `kubeadm join`: เพิ่ม node เข้า cluster
- `kubeadm upgrade`: upgrade cluster

```
Node Requirements สำหรับ chuaikan.com production:
- Control Plane: 4 CPU, 8GB RAM, 50GB disk
- Worker Nodes: 8 CPU, 16GB RAM, 100GB disk
- OS: Ubuntu 24.04 LTS
- Network: ทุก node คุยกันได้บน port 6443, 2379-2380, 10250-10252
```

**เราจะใช้ configuration นี้:**
```
control-plane-01: 192.168.1.10
worker-01:        192.168.1.11
worker-02:        192.168.1.12
```

---

## ⚙️ Environment Setup

### Step 312 — System Prerequisites (ทำทุก Node)

```bash
# ===== ทำบน ทุก NODE =====

# อัพเดท system
sudo apt-get update && sudo apt-get upgrade -y

# ตั้ง hostname (เปลี่ยนตาม node)
sudo hostnamectl set-hostname control-plane-01
# สำหรับ worker: sudo hostnamectl set-hostname worker-01

# เพิ่ม hosts file
sudo tee -a /etc/hosts <<EOF
192.168.1.10 control-plane-01
192.168.1.11 worker-01
192.168.1.12 worker-02
EOF

# Disable swap (K8s ต้องการ swap ปิด)
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# ตรวจสอบว่า swap ปิดแล้ว
free -h
# Output: Swap: 0B 0B 0B

# Load kernel modules
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# ตั้งค่า kernel parameters
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# ตรวจสอบ
lsmod | grep br_netfilter
lsmod | grep overlay
sysctl net.bridge.bridge-nf-call-iptables net.ipv4.ip_forward
```

### Step 313 — ติดตั้ง containerd v1.7.x (ทำทุก Node)

```bash
# ติดตั้ง dependencies
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# เพิ่ม Docker repository (containerd อยู่ใน Docker repo)
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y containerd.io

# ตรวจสอบ version
containerd --version
# containerd containerd.io 1.7.x ...

# ตั้งค่า containerd ให้ใช้ systemd cgroup driver
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null

# แก้ SystemdCgroup = true
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# ตรวจสอบการแก้ไข
grep SystemdCgroup /etc/containerd/config.toml
# output: SystemdCgroup = true

# restart และ enable containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd
```

---

## 🛠️ Step-by-Step Implementation

### Step 314 — ติดตั้ง kubeadm, kubelet, kubectl v1.29 (ทำทุก Node)

```bash
# เพิ่ม Kubernetes apt repository
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update

# ติดตั้ง kubeadm, kubelet, kubectl
sudo apt-get install -y kubelet=1.29.0-1.1 kubeadm=1.29.0-1.1 kubectl=1.29.0-1.1

# Lock version ไม่ให้ upgrade อัตโนมัติ
sudo apt-mark hold kubelet kubeadm kubectl

# เปิด kubelet ให้ start อัตโนมัติ
sudo systemctl enable --now kubelet

# ตรวจสอบ versions
kubeadm version
kubectl version --client
kubelet --version
```

### Step 315 — Init Control Plane ด้วย kubeadm (Control Plane Node เท่านั้น)

```bash
# ===== ทำบน control-plane-01 เท่านั้น =====

# สร้าง kubeadm config file
sudo tee /etc/kubernetes/kubeadm-config.yaml <<EOF
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "192.168.1.10"
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  name: control-plane-01
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: "1.29.0"
clusterName: "chuaikan-cluster"
controlPlaneEndpoint: "192.168.1.10:6443"
networking:
  podSubnet: "192.168.0.0/16"    # สำหรับ Calico
  serviceSubnet: "10.96.0.0/12"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
EOF

# Init control plane
sudo kubeadm init --config /etc/kubernetes/kubeadm-config.yaml \
  --upload-certs \
  2>&1 | tee /tmp/kubeadm-init.log

# ตั้งค่า kubectl สำหรับ current user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# ตรวจสอบ
kubectl get nodes
# NAME               STATUS     ROLES           AGE   VERSION
# control-plane-01   NotReady   control-plane   1m    v1.29.0
# (NotReady เป็นปกติ ต้องติดตั้ง CNI ก่อน)

# เก็บ join command ไว้ใช้กับ workers
# (จาก output ของ kubeadm init หรือใช้คำสั่งนี้)
kubeadm token create --print-join-command
```

### Step 316 — Join Worker Nodes (Worker Node เท่านั้น)

```bash
# ===== ทำบน worker-01 และ worker-02 =====

# คำสั่ง join (ได้จาก output ของ kubeadm init ใน step 315)
sudo kubeadm join 192.168.1.10:6443 \
  --token <token-from-init> \
  --discovery-token-ca-cert-hash sha256:<hash-from-init> \
  --cri-socket unix:///var/run/containerd/containerd.sock

# ===== กลับมาตรวจสอบบน control-plane-01 =====
kubectl get nodes
# NAME               STATUS     ROLES           AGE   VERSION
# control-plane-01   NotReady   control-plane   5m    v1.29.0
# worker-01          NotReady   <none>          1m    v1.29.0
# worker-02          NotReady   <none>          30s   v1.29.0
# (ยังเป็น NotReady อยู่ จนกว่าจะติดตั้ง Calico)
```

### Step 317 — ติดตั้ง Calico CNI

```bash
# ===== ทำบน control-plane-01 =====

# ดาวน์โหลด Calico manifest
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

# ติดตั้ง Calico
kubectl apply -f calico.yaml

# ดู Calico pods ขึ้นมา
kubectl get pods -n kube-system -l k8s-app=calico-node -w

# รอจนทุก pod เป็น Running (ประมาณ 2-3 นาที)
# NAME                READY   STATUS    RESTARTS   AGE
# calico-node-4kjl9   1/1     Running   0          2m
# calico-node-7bkqj   1/1     Running   0          2m
# calico-node-8xnpl   1/1     Running   0          2m

# ตรวจสอบ nodes (ควรเป็น Ready แล้ว)
kubectl get nodes
# NAME               STATUS   ROLES           AGE   VERSION
# control-plane-01   Ready    control-plane   10m   v1.29.0
# worker-01          Ready    <none>          5m    v1.29.0
# worker-02          Ready    <none>          4m    v1.29.0
```

### Step 318 — ติดตั้ง MetalLB สำหรับ Bare Metal Load Balancer

```bash
# MetalLB จะให้ LoadBalancer type Service ทำงานบน bare metal ได้

# ติดตั้ง MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml

# รอ MetalLB pods พร้อม
kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s

# ตั้งค่า IP address pool สำหรับ Load Balancers
# (ใช้ IP range ใน network เดียวกับ nodes)
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: chuaikan-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.200-192.168.1.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: chuaikan-l2-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - chuaikan-pool
EOF

# ตรวจสอบ
kubectl get ipaddresspool -n metallb-system
kubectl get l2advertisement -n metallb-system
```

### Step 319 — ตั้งค่า kubectl bash completion และ tools

```bash
# Bash completion สำหรับ kubectl
sudo apt-get install -y bash-completion

echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
source ~/.bashrc

# ติดตั้ง kubectx / kubens
sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -s /opt/kubectx/kubectx /usr/local/bin/kubectx
sudo ln -s /opt/kubectx/kubens /usr/local/bin/kubens

# ติดตั้ง k9s (terminal UI สำหรับ K8s)
curl -sS https://webi.sh/k9s | sh
# หรือ
wget https://github.com/derailed/k9s/releases/download/v0.31.7/k9s_Linux_amd64.tar.gz
tar -xzf k9s_Linux_amd64.tar.gz
sudo mv k9s /usr/local/bin/

# เปิด k9s
k9s
```

### Step 320 — Verify Cluster Health

```bash
# ตรวจสอบ component status
kubectl get componentstatuses
# หรือ
kubectl get cs

# ตรวจสอบ nodes และ roles
kubectl get nodes -o wide
# NAME               STATUS   ROLES           AGE   VERSION   INTERNAL-IP    OS-IMAGE             KERNEL-VERSION
# control-plane-01   Ready    control-plane   30m   v1.29.0   192.168.1.10   Ubuntu 24.04 LTS     6.8.0-38-generic
# worker-01          Ready    <none>          25m   v1.29.0   192.168.1.11   Ubuntu 24.04 LTS     6.8.0-38-generic
# worker-02          Ready    <none>          24m   v1.29.0   192.168.1.12   Ubuntu 24.04 LTS     6.8.0-38-generic

# ตรวจสอบ system pods ทั้งหมด
kubectl get pods -n kube-system

# ตรวจสอบ cluster info
kubectl cluster-info

# ตรวจสอบ etcd health
kubectl exec -n kube-system etcd-control-plane-01 -- \
  etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key

# Test deploy งาน application
kubectl create deployment hello-k8s \
  --image=nginx:alpine \
  --replicas=3 \
  --namespace=default

kubectl get pods -n default
kubectl expose deployment hello-k8s --port=80 --type=LoadBalancer
kubectl get service hello-k8s
# External IP จาก MetalLB pool

# ลบ test deployment
kubectl delete deployment hello-k8s
kubectl delete service hello-k8s
```

---

## 🔧 Configuration Files

### ตรวจสอบ Certificate Expiry

```bash
# ตรวจสอบ certificates
sudo kubeadm certs check-expiration

# Output:
# CERTIFICATE                EXPIRES                  RESIDUAL TIME   ...
# admin.conf                 Jan 01, 2026 10:00 UTC   364d            ...
# apiserver                  Jan 01, 2026 10:00 UTC   364d            ...
# ...

# Renew certificates (ก่อนหมดอายุ)
sudo kubeadm certs renew all
```

### ตั้งค่า Node Labels สำหรับ chuaikan.com

```bash
# Label worker nodes ตาม role
kubectl label node worker-01 node-role.kubernetes.io/worker=worker
kubectl label node worker-02 node-role.kubernetes.io/worker=worker

# Label สำหรับ workload placement
kubectl label node worker-01 workload=application
kubectl label node worker-02 workload=database

# ดู labels
kubectl get nodes --show-labels
```

---

## 🧪 Testing

### ทดสอบ DNS ภายใน Cluster

```bash
# สร้าง pod สำหรับ test
kubectl run dns-test \
  --image=busybox:1.28 \
  --restart=Never \
  -- sleep 300

# ทดสอบ DNS resolution
kubectl exec -it dns-test -- nslookup kubernetes.default.svc.cluster.local
# Output: Server: 10.96.0.10
#         Name: kubernetes.default.svc.cluster.local
#         Address: 10.96.0.1

# ทดสอบ internet access จาก pod
kubectl exec -it dns-test -- wget -qO- http://example.com

# ลบ test pod
kubectl delete pod dns-test
```

### ทดสอบ LoadBalancer Service

```bash
# Deploy test application
kubectl create deployment test-lb \
  --image=nginx:alpine \
  --replicas=2

kubectl expose deployment test-lb \
  --port=80 \
  --type=LoadBalancer

# รอ external IP (จาก MetalLB)
kubectl get service test-lb -w
# NAME      TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
# test-lb   LoadBalancer   10.100.58.23   192.168.1.200   80:31200/TCP   30s

# ทดสอบ
curl http://192.168.1.200

# ลบ
kubectl delete deployment test-lb
kubectl delete service test-lb
```

---

## ❌ Common Errors & Solutions

### Error: `[WARNING Swap]: swap is enabled`
```bash
# แก้: ปิด swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### Error: `[ERROR CRI]: container runtime is not running`
```bash
# แก้: restart containerd
sudo systemctl restart containerd
sudo systemctl status containerd

# ตรวจสอบ config
cat /etc/containerd/config.toml | grep SystemdCgroup
```

### Error: `node not found` หลัง join
```bash
# ตรวจสอบ firewall
sudo ufw status
# ถ้าเปิดอยู่ ต้อง allow ports K8s
sudo ufw allow 6443/tcp     # API server
sudo ufw allow 2379:2380/tcp # etcd
sudo ufw allow 10250/tcp    # kubelet
sudo ufw allow 10257/tcp    # controller manager
sudo ufw allow 10259/tcp    # scheduler
```

### Error: `Nodes NotReady` หลังติดตั้ง CNI
```bash
# ตรวจสอบ Calico pods
kubectl get pods -n kube-system -l k8s-app=calico-node
kubectl describe pod calico-node-xxxx -n kube-system

# ตรวจสอบ logs
kubectl logs -n kube-system -l k8s-app=calico-node
```

---

## ✅ Checklist

- [ ] **Step 311**: เข้าใจ kubeadm และ node requirements สำหรับ production
- [ ] **Step 312**: ปิด swap, load kernel modules, ตั้งค่า sysctl บนทุก node
- [ ] **Step 313**: ติดตั้ง containerd v1.7.x พร้อมตั้งค่า SystemdCgroup=true
- [ ] **Step 314**: ติดตั้ง kubeadm, kubelet, kubectl v1.29 และ lock version
- [ ] **Step 315**: init control plane ด้วย `kubeadm init` สำเร็จ
- [ ] **Step 316**: join worker nodes เข้า cluster สำเร็จ
- [ ] **Step 317**: ติดตั้ง Calico CNI, nodes status เป็น Ready
- [ ] **Step 318**: ติดตั้ง MetalLB, ตั้งค่า IP pool สำเร็จ
- [ ] **Step 319**: ตั้งค่า bash completion, kubectx, k9s
- [ ] **Step 320**: ตรวจสอบ cluster health, ทดสอบ DNS และ LoadBalancer

---

## 🔗 References

- [kubeadm Installation Guide](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [containerd Installation](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)
- [Calico Installation](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises)
- [MetalLB Documentation](https://metallb.universe.tf/installation/)

---

*Part 032 | Road to 1,000,000 Users/Day | chuaikan.com*
