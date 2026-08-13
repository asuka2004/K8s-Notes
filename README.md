# 二進制安裝 K8s

## 各個節點角色

| 節點   | IP            | 角色                                                                       |
| ---- | ------------- | ------------------------------------------------------------------------ |
| k8-1 | 192.168.88.91 | etcd + apiserver + controller-manager + scheduler + kubelet + kube-proxy |
| k8-2 | 192.168.88.92 | etcd + apiserver + controller-manager + scheduler + kubelet + kube-proxy |
| k8-3 | 192.168.88.93 | etcd + apiserver + controller-manager + scheduler + kubelet + kube-proxy |
| k8-4 | 192.168.88.94 | kubelet + kube-proxy                                                     |
| k8-5 | 192.168.88.95 | kubelet + kube-proxy                                                     |
| Lab  | 192.168.88.60 | NGINX 反向代理                                                               |

## 網段規劃

| 用途         | 網段              |
| ---------- | --------------- |
| Linux 主機網段 | 192.168.88.0/24 |
| Service 網段 | 10.96.0.0/16    |
| Pod 網段     | 10.244.0.0/16   |

## 軟體安裝

### 系統

```text
CentOS Stream release 9
```

### 容器

```bash
dnf install -y yum-utils

yum-config-manager --add-repo \
https://download.docker.com/linux/centos/docker-ce.repo

dnf install -y containerd.io

mkdir -p /etc/containerd

containerd config default > /etc/containerd/config.toml

sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' \
/etc/containerd/config.toml

systemctl enable --now containerd
systemctl start containerd
```

### PKI / TLS 工具包

```bash
curl -sLo cfssl \
https://github.com/cloudflare/cfssl/releases/download/v1.6.5/cfssl_1.6.5_linux_amd64

curl -sLo cfssljson \
https://github.com/cloudflare/cfssl/releases/download/v1.6.5/cfssljson_1.6.5_linux_amd64

curl -sLo cfssl-certinfo \
https://github.com/cloudflare/cfssl/releases/download/v1.6.5/cfssl-certinfo_1.6.5_linux_amd64

chmod +x cfssl cfssljson cfssl-certinfo

mv cfssl cfssljson cfssl-certinfo /usr/local/bin/
```

## 環境設定

### K8-1 ~ K8-5

以下設定需要在 **K8-1 ~ K8-5 五台主機全部執行**。

### 關閉 Swap

```bash
swapoff -a

sed -i '/swap/s/^/#/' /etc/fstab
```

### 關閉 SELinux

```bash
setenforce 0

sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' \
/etc/selinux/config
```

### 關閉 Firewalld

```bash
systemctl disable --now firewalld
systemctl stop firewalld
```

### 載入 Kubernetes Kernel Module

```bash
cat > /etc/modules-load.d/k8s.conf << 'EOF'
overlay
br_netfilter
EOF
```

```bash
modprobe overlay
modprobe br_netfilter
```

### 設定 Kernel Parameters

```bash
cat > /etc/sysctl.d/k8s.conf << 'EOF'
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

```bash
sysctl --system
```

### 設定 Hosts

```bash
cat >> /etc/hosts << 'EOF'
192.168.88.91 k8-1
192.168.88.92 k8-2
192.168.88.93 k8-3
192.168.88.94 k8-4
192.168.88.95 k8-5
192.168.88.60 Lab
EOF
```

### 安裝 IPVS

```bash
dnf install -y ipvsadm ipset conntrack-tools
```

```bash
cat > /etc/modules-load.d/ipvs.conf << 'EOF'
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
EOF
```

```bash
modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe nf_conntrack
```

## 憑證列表

先使用 CFSSL 產生自簽 CA 憑證，再使用 `ca.pem`、`ca-key.pem` 產生各 Kubernetes 元件的憑證。

### 1. etcd 憑證

**K8-1 ~ K8-3**

```bash
cfssl gencert \
-ca=ca.pem \
-ca-key=ca-key.pem \
-config=ca-config.json \
-hostname=127.0.0.1,192.168.88.91,192.168.88.92,192.168.88.93,k8-1,k8-2,k8-3 \
-profile=kubernetes \
etcd-csr.json | cfssljson -bare etcd
```

### 2. kube-apiserver 憑證

**K8-1 ~ K8-3**

```bash
cfssl gencert \
-ca=ca.pem \
-ca-key=ca-key.pem \
-config=ca-config.json \
-hostname=10.96.0.1,127.0.0.1,192.168.88.60,192.168.88.91,192.168.88.92,192.168.88.93,192.168.88.94,192.168.88.95,Lab,k8-1,k8-2,k8-3,k8-4,k8-5,kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster.local \
-profile=kubernetes \
apiserver-csr.json | cfssljson -bare apiserver
```

### 3. front-proxy CA / 憑證

**K8-1 ~ K8-3**

```bash
cfssl gencert \
-initca front-proxy-ca-csr.json | cfssljson -bare front-proxy-ca
```

```bash
cfssl gencert \
-ca=front-proxy-ca.pem \
-ca-key=front-proxy-ca-key.pem \
-config=ca-config.json \
-profile=kubernetes \
front-proxy-client-csr.json | cfssljson -bare front-proxy-client
```

### 4. controller-manager 憑證

**K8-1 ~ K8-3**

```bash
cfssl gencert \
-ca=ca.pem \
-ca-key=ca-key.pem \
-config=ca-config.json \
-profile=kubernetes \
controller-manager-csr.json | cfssljson -bare controller-manager
```

### 5. scheduler 憑證

**K8-1 ~ K8-3**

```bash
cfssl gencert \
-ca=ca.pem \
-ca-key=ca-key.pem \
-config=ca-config.json \
-profile=kubernetes \
scheduler-csr.json | cfssljson -bare scheduler
```

### 6. admin 憑證

**K8-1 ~ K8-3**

```bash
cfssl gencert \
-ca=ca.pem \
-ca-key=ca-key.pem \
-config=ca-config.json \
-profile=kubernetes \
admin-csr.json | cfssljson -bare admin
```

### 7. kubelet 憑證

**K8-1 ~ K8-5**

五台 kubelet 憑證都不同。

以 K8-1 為例：

```bash
cfssl gencert \
-ca=ca.pem \
-ca-key=ca-key.pem \
-config=ca-config.json \
-hostname=192.168.88.91,k8-1 \
-profile=kubernetes \
kubelet-k8-1-csr.json | cfssljson -bare kubelet-k8-1
```

### 驗證憑證

```bash
openssl x509 -in xxx.pem -noout -text | \
grep -A2 "Subject Alternative Name"
```

## Kubeconfig 部署

### 1. controller-manager

**K8-1 ~ K8-3**

```bash
kubectl config set-cluster kubernetes \
--certificate-authority=ca.pem \
--embed-certs=true \
--server=https://192.168.88.60:6443 \
--kubeconfig=controller-manager.kubeconfig
```

```bash
kubectl config set-credentials system:kube-controller-manager \
--client-certificate=controller-manager.pem \
--client-key=controller-manager-key.pem \
--embed-certs=true \
--kubeconfig=controller-manager.kubeconfig
```

```bash
kubectl config set-context default \
--cluster=kubernetes \
--user=system:kube-controller-manager \
--kubeconfig=controller-manager.kubeconfig
```

```bash
kubectl config use-context default \
--kubeconfig=controller-manager.kubeconfig
```

### 2. scheduler kubeconfig

**K8-1 ~ K8-3**

```bash
kubectl config set-cluster kubernetes \
--certificate-authority=ca.pem \
--embed-certs=true \
--server=https://192.168.88.60:6443 \
--kubeconfig=scheduler.kubeconfig
```

```bash
kubectl config set-credentials system:kube-scheduler \
--client-certificate=scheduler.pem \
--client-key=scheduler-key.pem \
--embed-certs=true \
--kubeconfig=scheduler.kubeconfig
```

```bash
kubectl config set-context default \
--cluster=kubernetes \
--user=system:kube-scheduler \
--kubeconfig=scheduler.kubeconfig
```

```bash
kubectl config use-context default \
--kubeconfig=scheduler.kubeconfig
```

### 3. admin kubeconfig

**K8-1 ~ K8-3**

```bash
kubectl config set-cluster kubernetes \
--certificate-authority=ca.pem \
--embed-certs=true \
--server=https://192.168.88.60:6443 \
--kubeconfig=admin.kubeconfig
```

```bash
kubectl config set-credentials admin \
--client-certificate=admin.pem \
--client-key=admin-key.pem \
--embed-certs=true \
--kubeconfig=admin.kubeconfig
```

```bash
kubectl config set-context default \
--cluster=kubernetes \
--user=admin \
--kubeconfig=admin.kubeconfig
```

```bash
kubectl config use-context default \
--kubeconfig=admin.kubeconfig
```

### 驗證

```bash
kubectl --kubeconfig=/etc/kubernetes/admin.kubeconfig \
get componentstatuses
```

```bash
kubectl --kubeconfig=/etc/kubernetes/admin.kubeconfig \
cluster-info
```

### 4. kubelet kubeconfig

**K8-1 ~ K8-5**

因五台主機的 kubelet 憑證不同，因此每台的 Kubeconfig 設定檔也不同。

以 K8-1 為例：

```bash
kubectl config set-cluster kubernetes \
--certificate-authority=ca.pem \
--embed-certs=true \
--server=https://192.168.88.60:6443 \
--kubeconfig=kubelet-k8-1.kubeconfig
```

```bash
kubectl config set-credentials system:node:k8-1 \
--client-certificate=kubelet-k8-1.pem \
--client-key=kubelet-k8-1-key.pem \
--embed-certs=true \
--kubeconfig=kubelet-k8-1.kubeconfig
```

```bash
kubectl config set-context default \
--cluster=kubernetes \
--user=system:node:k8-1 \
--kubeconfig=kubelet-k8-1.kubeconfig
```

```bash
kubectl config use-context default \
--kubeconfig=kubelet-k8-1.kubeconfig
```

## K8s 部署

### 部署 etcd 集群

**K8-1 ~ K8-3**

```bash
curl -L -o etcd.tar.gz \
https://github.com/etcd-io/etcd/releases/download/v3.5.17/etcd-${ETCD_VER}-linux-amd64.tar.gz

tar xzf etcd.tar.gz

cp etcd etcdctl etcdutl /usr/local/bin/
```

### 部署 kube-apiserver / controller-manager / scheduler

**K8-1 ~ K8-3**

```bash
curl -L -o kubernetes-server.tar.gz \
https://dl.k8s.io/v1.36.3/kubernetes-server-linux-amd64.tar.gz

tar xzf kubernetes-server.tar.gz

cp kube-apiserver kube-controller-manager kube-scheduler kubectl \
/usr/local/bin/
```

### 部署 kubelet / kube-proxy

**K8-1 ~ K8-5**

```bash
curl -L -o kubernetes-server.tar.gz \
https://dl.k8s.io/v1.36.3/kubernetes-server-linux-amd64.tar.gz

tar xzf kubernetes-server.tar.gz

cp kubelet kube-proxy /usr/local/bin/
```

## 常駐服務

可以參照 systemd Service 文件。

1. `etcd.service`
2. `kube-apiserver.service`（每台的 `--advertise-address` 不同）
3. `kube-controller-manager.service`（三台內容完全一樣）
4. `kube-scheduler.service`（三台內容完全一樣）
5. `kubelet.service`（五台內容完全一樣）
6. `kube-proxy.service`（五台內容完全一樣）
