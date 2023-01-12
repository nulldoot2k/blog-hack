---
permalink: ls/install-tools.html
---

## Linux
```bash
Runtime					Path to Unix domain socket
containerd				unix:///var/run/containerd/containerd.sock
CRI-O					unix:///var/run/crio/crio.sock
Docker Engine (using cri-dockerd)	unix:///var/run/cri-dockerd.sock
```

## Update

> sudo apt-get update 

## Install Docker

- Watch: [Docker](docker.html#install-docker)

## Install GoLang

```bash
git clone https://github.com/Mirantis/cri-dockerd.git
wget https://go.dev/dl/go1.19.4.linux-amd64.tar.gz
sudo apt-get remove golang-go
sudo apt-get remove --auto-remove golang-go
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.19.4.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
go version
```

## Install DockerCE

```bash
cd cri-dockerd
mkdir bin
go build -o bin/cri-dockerd
mkdir -p /usr/local/bin
install -o root -g root -m 0755 bin/cri-dockerd /usr/local/bin/cri-dockerd
cp -a packaging/systemd/* /etc/systemd/system
sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
systemctl daemon-reload
systemctl enable cri-docker.service
systemctl enable --now cri-docker.socket
```

- [Cài đặt thời gian chạy vùng chứa container](install-tools.html#cài-đặt-thời-gian-chạy-vùng-chứa-container)

## Install k8s

Update the apt package index and install packages needed to use the Kubernetes apt repository:

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

Download the Google Cloud public signing key:

```bash
sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```
		
Add the Kubernetes apt repository:

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Update apt package index, install kubelet, kubeadm and kubectl, and pin their version:

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

tắt swap

```bash
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

```bash
$ sudo vim /etc/fstab
#/swap.img	none	swap	sw	0	0
```

Xác nhận cài đặt là chính xác

```bash
sudo mount -a
free -h
```

Cài đặt và cấu hình điều kiện tiên quyết, Kích hoạt các mô-đun hạt nhân và định cấu hình sysctl.

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

```bash
# Enable kernel modules
sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

Xác minh rằng các mô-đun br_netfilter, lớp phủ được tải bằng cách chạy các hướng dẫn bên dưới:
```bash
lsmod | grep br_netfilter
lsmod | grep overlay
```

> sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward

---

## Cài đặt thời gian chạy vùng chứa container

> Docker

> CRI-O

> Containerd

- Installing Docker CE runtime:

```bash
# Add repo and Install packages
sudo apt update
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update
sudo apt install -y containerd.io docker-ce docker-ce-cli

# Create required directories
sudo mkdir -p /etc/systemd/system/docker.service.d

# Create daemon json config file
sudo tee /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

# Start and enable Services
sudo systemctl daemon-reload 
sudo systemctl restart docker
sudo systemctl enable docker
```

- Installing CRI-O:

```bash
# Ensure you load modules
sudo modprobe overlay
sudo modprobe br_netfilter

# Set up required sysctl params
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Reload sysctl
sudo sysctl --system

# Add Cri-o repo Ubuntu 18.04:
sudo su -
OS=xUbuntu_18.04
CRIO_VERSION=1.23
echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /"|sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
echo "deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$CRIO_VERSION/$OS/ /"|sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$CRIO_VERSION.list
# import GPG key:
curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$CRIO_VERSION/$OS/Release.key | sudo apt-key add -
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | sudo apt-key add -

# Install CRI-O
sudo apt update
sudo apt -y install cri-o cri-o-runc

# Checking the version of CRI-O installed on Ubuntu:
apt show cri-o

# Update CRI-O CIDR subnet
sudo sed -i 's/10.85.0.0/192.168.0.0/g' /etc/cni/net.d/100-crio-bridge.conf
# chú ý: nhìn vào ip a --> để xem ip cần set bridge
	# docker0: 172.17.0.1/16 --> 172.17.0.0/16 
	# cni0: 10.85.0.1/16 --> 10.85.0.0/16 

# Start and enable Service
sudo systemctl daemon-reload
sudo systemctl restart crio
sudo systemctl enable crio
sudo systemctl status crio
```

- Installing Containerd:

```bash
# Configure persistent loading of modules
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
ebtable_broute
EOF

# Load at runtime
sudo modprobe overlay
sudo modprobe br_netfilter
sudo modprobe ebtable_broute

# Ensure sysctl params are set
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Reload configs
sudo sysctl --system

# Install required packages
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates

# Add Docker repo
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

# Install containerd
sudo apt update
sudo apt install -y containerd.io

# Configure containerd and start service
sudo su -
mkdir -p /etc/containerd
containerd config default>/etc/containerd/config.toml

# restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
systemctl status  containerd
```

Để sử dụng trình điều khiển cgroup systemd, hãy đặt `plugins.cri.systemd_cgroup = true`

> /etc/containerd/config.toml

Đăng nhập vào máy chủ sẽ được sử dụng làm máy chủ và đảm bảo rằng mô-đun br_netfilter đã được tải:

```bash
$ lsmod | grep br_netfilter
br_netfilter           22256  0 
bridge                151336  2 br_netfilter,ebtable_broute
```

Enable kubelet service.
```bash
sudo systemctl enable kubelet
```

Bây giờ khởi tạo máy sẽ chạy các thành phần của mặt phẳng điều khiển bao gồm etcd (cơ sở dữ liệu cụm) và Máy chủ API.

```bash
$ sudo kubeadm config images pull
[config/images] Pulled k8s.gcr.io/kube-apiserver:v1.24.3
[config/images] Pulled k8s.gcr.io/kube-controller-manager:v1.24.3
[config/images] Pulled k8s.gcr.io/kube-scheduler:v1.24.3
[config/images] Pulled k8s.gcr.io/kube-proxy:v1.24.3
[config/images] Pulled k8s.gcr.io/pause:3.7
[config/images] Pulled k8s.gcr.io/etcd:3.5.3-0
[config/images] Pulled k8s.gcr.io/coredns/coredns:v1.8.6
```

Nếu có nhiều ổ cắm CRI, vui lòng sử dụng --cri-socket để chọn một ổ cắm:

```bash
# CRI-O
sudo kubeadm config images pull --cri-socket unix:///var/run/crio/crio.sock

# Containerd
sudo kubeadm config images pull --cri-socket unix:///run/containerd/containerd.sock

# Docker
sudo kubeadm config images pull --cri-socket unix:///run/cri-dockerd.sock
```

Đây là các tùy chọn khởi tạo kubeadm cơ bản được sử dụng để khởi động cụm.

```bash
--control-plane-endpoint :  đặt điểm endpoint chung cho tất cả control-plane nodes. Có thể là DNS/IP
--pod-network-cidr : Được sử dụng để đặt CIDR tiện ích bổ sung cho mạng Pod
--cri-socket : Sử dụng nếu có nhiều container runtime để đặt đường dẫn runtime socket.
--apiserver-advertise-address : Đặt địa chỉ quảng cáo cho Server API của control-plane node cụ thể này
```

Để khởi động một cụm mà không sử dụng điểm cuối DNS, hãy chạy

```bash
### With Docker CE ###
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket unix:///run/cri-dockerd.sock 

### With CRI-O###
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket unix:///var/run/crio/crio.sock

### With Containerd ###
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket unix:///run/containerd/containerd.sock
```

Đặt tên DNS của điểm cuối cụm hoặc thêm bản ghi vào tệp /etc/hosts.

```bash
$ sudo vim /etc/hosts
172.29.20.5 k8s-cluster.nulldoot2k.com
```

## Create cluster:

```bash
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket --cri-socket unix:///run/cri-dockerd.sock \
  --apiserver-advertise-address=172.29.20.5
```

Bạn có thể tùy chọn chuyển tệp Ổ cắm cho thời gian chạy và địa chỉ quảng cáo tùy thuộc vào thiết lập của bạn.

```bash
# CRI-O
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket unix:///var/run/crio/crio.sock \
  --upload-certs \
  --control-plane-endpoint=k8s-cluster.computingforgeeks.com

# Containerd
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket unix:///run/containerd/containerd.sock \
  --upload-certs \
  --control-plane-endpoint=k8s-cluster.computingforgeeks.com

# Docker
# Must do https://computingforgeeks.com/install-mirantis-cri-dockerd-as-docker-engine-shim-for-kubernetes/
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket unix:///run/cri-dockerd.sock  \
  --upload-certs \
  --control-plane-endpoint=k8s-cluster.computingforgeeks.com
```

Định cấu hình kubectl bằng các lệnh trong đầu ra:

```bash
mkdir -p $HOME/.kube
sudo cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

> Đối với người dùng root:

```bash
export KUBECONFIG=/etc/kubenetes/admin.conf
```

Kiểm tra trạng thái cụm:

```bash
$ kubectl cluster-info
Kubernetes master is running at https://k8s-cluster.computingforgeeks.com:6443
KubeDNS is running at https://k8s-cluster.computingforgeeks.com:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

Có thể thêm các nút Master bổ sung bằng cách sử dụng lệnh trong đầu ra cài đặt:

```bash
kubeadm join k8s-cluster.computingforgeeks.com:6443 --token sr4l2l.2kvot0pfalh5o4ik \
    --discovery-token-ca-cert-hash sha256:c692fb047e15883b575bd6710779dc2c5af8073f7cab460abd181fd3ddb29a18 \
    --control-plane
```

Cài đặt plugin mạng trên Master

> sử dụng Calico

```bash
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml 
kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
```

Xác nhận rằng tất cả các nhóm đang chạy:

```bash
$ watch kubectl get pods --all-namespaces
NAMESPACE          NAME                                      READY   STATUS    RESTARTS   AGE
calico-apiserver   calico-apiserver-966cf7f85-b27s2          1/1     Running   0          2m17s
calico-apiserver   calico-apiserver-966cf7f85-xck9d          1/1     Running   0          2m17s
calico-system      calico-kube-controllers-657d56796-pw2f2   1/1     Running   0          8m25s
calico-system      calico-node-vz6fv                         1/1     Running   0          8m25s
calico-system      calico-typha-849bd7dc87-c7w6j             1/1     Running   0          8m26s
kube-system        coredns-6d4b75cb6d-4ctp4                  1/1     Running   0          9m57s
kube-system        coredns-6d4b75cb6d-52qbz                  1/1     Running   0          9m57s
kube-system        etcd-ubuntu-01                            1/1     Running   0          10m
kube-system        kube-apiserver-ubuntu-01                  1/1     Running   0          10m
kube-system        kube-controller-manager-ubuntu-01         1/1     Running   0          10m
kube-system        kube-proxy-652dl                          1/1     Running   0          9m57s
kube-system        kube-scheduler-ubuntu-01                  1/1     Running   0          10m
tigera-operator    tigera-operator-6995cc5df5-6mzw8          1/1     Running   0          8m40s
```

Đối với cụm nút đơn cho phép Pod chạy trên các nút chính:

```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
kubectl taint nodes --all  node-role.kubernetes.io/control-plane-
```

Xác nhận nút master đã sẵn sàng:

```bash
# CRI-O
$ kubectl get nodes -o wide
NAME     STATUS   ROLES                  AGE   VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
ubuntu   Ready    control-plane,master   38s   v1.24.3   143.198.114.46   <none>        Ubuntu 20.04.3 LTS   5.4.0-88-generic   cri-o://1.23.2

# Containerd
$ kubectl get nodes -o wide
NAME     STATUS   ROLES                  AGE   VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
ubuntu   Ready    control-plane,master   15m   v1.24.3   143.198.114.46   <none>        Ubuntu 20.04.3 LTS   5.4.0-88-generic   containerd://1.4.11

# Docker
$ kubectl get nodes -o wide
NAME           STATUS   ROLES    AGE   VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE           KERNEL-VERSION     CONTAINER-RUNTIME
k8s-master01   Ready    master   64m   v1.24.3   135.181.28.113   <none>        Ubuntu 20.04 LTS   5.4.0-37-generic   docker://20.10.8
```

## Add worker nodes

Với mặt phẳng điều khiển đã sẵn sàng, bạn có thể thêm các nút worker vào cụm để chạy khối lượng công việc theo lịch trình.

Nếu địa chỉ điểm cuối không có trong DNS, hãy thêm bản ghi vào /etc/hosts.

```bash
$ sudo vim /etc/hosts
172.29.20.5 k8s-cluster.computingforgeeks.com
```

Lệnh tham gia đã đưa ra được sử dụng để thêm một nút công nhân vào cụm.

```bash
kubeadm join k8s-cluster.computingforgeeks.com:6443 \
  --token sr4l2l.2kvot0pfalh5o4ik \
  --discovery-token-ca-cert-hash sha256:c692fb047e15883b575bd6710779dc2c5af8073f7cab460abd181fd3ddb29a18
```

Chạy lệnh bên dưới trên control-plane để xem nút có tham gia cụm không.

```bash
$ kubectl get nodes
NAME                                 STATUS   ROLES    AGE   VERSION
k8s-master01.computingforgeeks.com   Ready    master   10m   v1.24.3
k8s-worker01.computingforgeeks.com   Ready    <none>   50s   v1.24.3
k8s-worker02.computingforgeeks.com   Ready    <none>   12s   v1.24.3
```

---

> kubectl describe node vy-nulldoot2k | grep -i tain
```bash
--> Taints:             node-role.kubernetes.io/control-plane:NoSchedule
```

> kubectl taint node vy-nulldoot2k node-role.kubernetes.io/control-plane:NoSchedule-
```bash
node/vy-nulldoot2k untainted
```

> kubectl describe node vy-nulldoot2k | grep -i tain
```bash
Taints:             <none>
```

## Nginx

> kubectl run --image=nginx webapp-nginx -n default

> kubectl get pods -o wide

> kubectl get svc -o wide

> kubectl expose pod webapp-nginx --port=80 --name="webapp-svc" --type='NodePort'
