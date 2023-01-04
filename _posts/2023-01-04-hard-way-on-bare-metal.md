---
permalink: ls/meta.html
---

## **Setup Network**

> copy all , lb, controller, woker to /etc/hosts 
```bash
10.3.52.52 k8s-controllers-lb
10.3.54.219 k8s-controller-0
10.3.55.13 k8s-controller-1
10.3.52.68 k8s-controller-2
10.3.52.116 k8s-worker-0
10.3.53.167 k8s-worker-1
10.3.52.156 k8s-worker-2
```

HA Proxy: sudo apt install haproxy -y

> sudo vi /etc/haproxy/haproxy.cfg

```bash
frontend k8s
  bind 10.3.52.52:6443
  mode tcp
  default_backend k8s

backend k8s
  balance roundrobin
  mode tcp
  option tcplog
  option tcp-check
  server k8s-controller-0 10.3.54.219:6443 check
  server k8s-controller-1 10.3.55.13:6443 check
  server k8s-controller-2 10.3.52.68:6443 check
```

## **Local Machine**

> **Install cfssl**
```bash
wget -q --show-progress --https-only --timestamping \
https://github.com/cloudflare/cfssl/releases/download/v1.6.1\
/cfssl_1.6.1_linux_amd64 https://github.com/cloudflare/cfssl\
/releases/download/v1.6.1/cfssljson_1.6.1_linux_amd64
```
> chmod +x cfssl_1.6.1_linux_amd64 cfssljson_1.6.1_linux_amd64
		
> sudo mv cfssl_1.6.1_linux_amd64 /usr/local/bin/cfssl
		
> sudo mv cfssljson_1.6.1_linux_amd64 /usr/local/bin/cfssljson

- Provisioning CA
```bash
mkdir -p pki/{admin,api,ca,clients,controller,front-proxy,proxy,scheduler,service-account,users}
```

- Generate the CA (Certificate Authority) config files & certificates
```bash
TLS_C="VN"
TLS_L="Hanoi"
TLS_OU="VCCORP"
TLS_ST="ThanhXuan"
```
Certificate Authority
```bash
cat > pki/ca/ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF
cat > pki/ca/ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "${TLS_C}",
      "L": "${TLS_L}",
      "O": "Kubernetes",
      "OU": "${TLS_OU}",
      "ST": "${TLS_ST}"
    }
  ]
}
EOF
cfssl gencert -initca pki/ca/ca-csr.json | cfssljson -bare pki/ca/ca
```

- Generating TLS Certificates<br>
Generate the admin user config files and certificates
```bash
cat > pki/admin/admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "${TLS_C}",
      "L": "${TLS_L}",
      "O": "system:masters",
      "OU": "${TLS_OU}",
      "ST": "${TLS_ST}"
    }
  ]
}
EOF
cfssl gencert \
  -ca=pki/ca/ca.pem \
  -ca-key=pki/ca/ca-key.pem \
  -config=pki/ca/ca-config.json \
  -profile=kubernetes \
  pki/admin/admin-csr.json | cfssljson -bare pki/admin/admin
```
Generate the worker(s) certs and keys
```bash
export EXTERNAL_IP=$(curl -s -4 https://ifconfig.co/)
```
```bash
ip addr show eth0 | grep -Po 'inet \K[\d.]+'
```
```bash
for instance in k8s-worker-0 k8s-worker-1 k8s-worker-2; do
cat > pki/clients/${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "${TLS_C}",
      "L": "${TLS_L}",
      "O": "system:nodes",
      "OU": "${TLS_OU}",
      "ST": "${TLS_ST}"
    }
  ]
}
EOF
INTERNAL_IP=$(ip addr show eth0 | grep -Po 'inet \K[\d.]+')
cfssl gencert \
  -ca=pki/ca/ca.pem \
  -ca-key=pki/ca/ca-key.pem \
  -config=pki/ca/ca-config.json \
  -hostname=${instance},${INTERNAL_IP},${EXTERNAL_IP} \
  -profile=kubernetes \
  pki/clients/${instance}-csr.json | cfssljson -bare pki/clients/${instance}
done
```
Generate the kube-controller-manager cert and key
```bash
cat > pki/controller/kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "${TLS_C}",
      "L": "${TLS_L}",
      "O": "system:kube-controller-manager",
      "OU": "${TLS_OU}",
      "ST": "${TLS_ST}"
    }
  ]
}
EOF
cfssl gencert \
  -ca=pki/ca/ca.pem \
  -ca-key=pki/ca/ca-key.pem \
  -config=pki/ca/ca-config.json \
  -profile=kubernetes \
  pki/controller/kube-controller-manager-csr.json | cfssljson -bare pki/controller/kube-controller-manager
```
Generate the kube-proxy cert and key
```bash
cat > pki/proxy/kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "${TLS_C}",
      "L": "${TLS_L}",
      "O": "system:node-proxier",
      "OU": "${TLS_OU}",
      "ST": "${TLS_ST}"
    }
  ]
}
EOF
cfssl gencert \
  -ca=pki/ca/ca.pem \
  -ca-key=pki/ca/ca-key.pem \
  -config=pki/ca/ca-config.json \
  -profile=kubernetes \
  pki/proxy/kube-proxy-csr.json | cfssljson -bare pki/proxy/kube-proxy
```
Generate the scheduler cert and key
```bash
cat > pki/scheduler/kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "${TLS_C}",
      "L": "${TLS_L}",
      "O": "system:kube-scheduler",
      "OU": "${TLS_OU}",
      "ST": "${TLS_ST}"
    }
  ]
}
EOF
cfssl gencert \
  -ca=pki/ca/ca.pem \
  -ca-key=pki/ca/ca-key.pem \
  -config=pki/ca/ca-config.json \
  -profile=kubernetes \
  pki/scheduler/kube-scheduler-csr.json | cfssljson -bare pki/scheduler/kube-scheduler
```
Generate the front-proxy cert and key
```bash
cat > pki/front-proxy/front-proxy-csr.json <<EOF
{
  "CN": "front-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "${TLS_C}",
      "L": "${TLS_L}",
      "O": "front-proxy",
      "OU": "${TLS_OU}",
      "ST": "${TLS_ST}"
    }
  ]
}
EOF
cfssl gencert \
  -ca=pki/ca/ca.pem \
  -ca-key=pki/ca/ca-key.pem \
  -config=pki/ca/ca-config.json \
  -profile=kubernetes \
  pki/front-proxy/front-proxy-csr.json | cfssljson -bare pki/front-proxy/front-proxy
```
Generate the api-server cert and key
```bash
KUBERNETES_PUBLIC_ADDRESS=10.3.52.52
cat > pki/api/kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "${TLS_C}",
      "L": "${TLS_L}",
      "O": "Kubernetes",
      "OU": "${TLS_OU}",
      "ST": "${TLS_ST}"
    }
  ]
}
EOF
cfssl gencert \
  -ca=pki/ca/ca.pem \
  -ca-key=pki/ca/ca-key.pem \
  -config=pki/ca/ca-config.json \
  -hostname=10.3.54.219,10.3.55.13,10.3.55.13,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,kubernetes.default \
  -profile=kubernetes \
  pki/api/kubernetes-csr.json | cfssljson -bare pki/api/kubernetes
```
Generate the service-account cert and key
```bash
cat > pki/service-account/service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "${TLS_C}",
      "L": "${TLS_L}",
      "O": "Kubernetes",
      "OU": "${TLS_OU}",
      "ST": "${TLS_ST}"
    }
  ]
}
EOF
cfssl gencert \
  -ca=pki/ca/ca.pem \
  -ca-key=pki/ca/ca-key.pem \
  -config=pki/ca/ca-config.json \
  -profile=kubernetes \
  pki/service-account/service-account-csr.json | cfssljson -bare pki/service-account/service-account
```
Finally, lets move the certs
```bash
for instance in k8s-worker-0 k8s-worker-1 k8s-worker-2; do
  scp pki/ca/ca.pem pki/clients/${instance}-key.pem pki/clients/${instance}.pem root@${instance}:~/
done
for instance in k8s-controller-0 k8s-controller-1 k8s-controller-2; do
  scp pki/ca/ca.pem pki/ca/ca-key.pem pki/api/kubernetes-key.pem pki/api/kubernetes.pem pki/service-account/service-account-key.pem pki/service-account/service-account.pem pki/front-proxy/front-proxy-key.pem pki/front-proxy/front-proxy.pem root@${instance}:~/
done
scp pki/ca/ca.pem pki/api/kubernetes-key.pem pki/api/kubernetes.pem root@k8s-controllers-lb:~/
```
  


## Generate Worker kubeconfigs

```bash
for instance in k8s-worker-0 k8s-worker-1 k8s-worker-2; do
kubectl config set-cluster Drewbernetes\
    --certificate-authority=pki/ca/ca.pem \
    --embed-certs=true \
    --server=https://10.3.52.52:6443 \
    --kubeconfig=configs/clients/${instance}.kubeconfig
kubectl config set-credentials system:node:${instance} \
    --client-certificate=pki/clients/${instance}.pem \
    --client-key=pki/clients/${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=configs/clients/${instance}.kubeconfig
kubectl config set-context default \
    --cluster=Drewbernetes \
    --user=system:node:${instance} \
    --kubeconfig=configs/clients/${instance}.kubeconfig
kubectl config use-context default --kubeconfig=configs/clients/${instance}.kubeconfig
done
```


## **Setting up ETCD** - **Controller**

- Get the files required and move them into place
```bash
wget -q --show-progress --https-only --timestamping "https://github.com/etcd-io/etcd/releases/download/v3.5.1/etcd-v3.5.1-linux-amd64.tar.gz"
```

> tar -xvf etcd-v3.5.1-linux-amd64.tar.gz

> sudo mv etcd-v3.5.1-linux-amd64/etcd* /usr/local/bin/

- Configuring the ETCD server

```bash
sudo mkdir -p /etc/etcd /var/lib/etcd

sudo cp ~/ca.pem ~/kubernetes-key.pem ~/kubernetes.pem /etc/etcd/

INTERNAL_IP=$(ip addr show eth0 | grep -Po 'inet \K[\d.]+')

ETCD_NAME=$(hostname -s)
```

- Create the service file

k8s-controllers-lb
  
```bash
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \
  --name ubuntu-vyhamii-test1 \
  --data-dir=/var/lib/etcd \
  --listen-peer-urls https://10.3.52.52:2380 \
  --listen-client-urls https://10.3.52.52:2379,https://127.0.0.1:2379 \
  --initial-advertise-peer-urls https://10.3.52.52:2380 \
  --initial-cluster k8s-controller-0=https://10.3.54.219:2380,k8s-controller-1=https://10.3.55.13:2380,k8s-controller-2=https://10.3.52.68:2380 \
  --initial-cluster-state new \
  --initial-cluster-token etcd-cluster-0 \
  --advertise-client-urls https://10.3.52.52:2379 \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --client-cert-auth \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --peer-client-cert-auth \
  --peer-trusted-ca-file=/etc/etcd/ca.pem
  
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

k8s-controller-0

```bash
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \
  --name ubuntu-vyhamii-test2 \
  --data-dir=/var/lib/etcd \
  --listen-peer-urls https://10.3.54.219:2380 \
  --listen-client-urls https://10.3.54.219:2379,https://127.0.0.1:2379 \
  --initial-advertise-peer-urls https://10.3.54.219:2380 \
  --initial-cluster k8s-controller-0=https://10.3.54.219:2380,k8s-controller-1=https://10.3.55.13:2380,k8s-controller-2=https://10.3.52.68:2380 \
  --initial-cluster-state new \
  --initial-cluster-token etcd-cluster-0 \
  --advertise-client-urls https://10.3.54.219:2379 \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --client-cert-auth \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --peer-client-cert-auth \
  --peer-trusted-ca-file=/etc/etcd/ca.pem
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

k8s-controller-1

```bash
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \
  --name ubuntu-vyhamii-test3 \
  --data-dir=/var/lib/etcd \
  --listen-peer-urls https://10.3.55.13:2380 \
  --listen-client-urls https://10.3.55.13:2379,https://127.0.0.1:2379 \
  --initial-advertise-peer-urls https://10.3.55.13:2380 \
  --initial-cluster k8s-controller-0=https://10.3.54.219:2380,k8s-controller-1=https://10.3.55.13:2380,k8s-controller-2=https://10.3.52.68:2380 \
  --initial-cluster-state new \
  --initial-cluster-token etcd-cluster-0 \
  --advertise-client-urls https://10.3.55.13:2379 \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --client-cert-auth \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --peer-client-cert-auth \
  --peer-trusted-ca-file=/etc/etcd/ca.pem
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

k8s-controller-2

```bash
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \
  --name ubuntu-vyhamii-test4 \
  --data-dir=/var/lib/etcd \
  --listen-peer-urls https://10.3.52.68:2380 \
  --listen-client-urls https://10.3.52.68:2379,https://127.0.0.1:2379 \
  --initial-advertise-peer-urls https://10.3.52.68:2380 \
  --initial-cluster k8s-controller-0=https://10.3.54.219:2380,k8s-controller-1=https://10.3.55.13:2380,k8s-controller-2=https://10.3.52.68:2380 \
  --initial-cluster-state new \
  --initial-cluster-token etcd-cluster-0 \
  --advertise-client-urls https://10.3.52.68:2379 \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --client-cert-auth \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --peer-client-cert-auth \
  --peer-trusted-ca-file=/etc/etcd/ca.pem
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

Start it up
```bash
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```

Rejoice (and test)!
```bash
sudo etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```

# ===================


{

cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "VN",
      "L": "Hanoi",
      "O": "Kubernetes",
      "OU": "VCCORP",
      "ST": "ThanhXuan"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

}

# ==============

{

cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "VN",
      "L": "Hanoi",
      "O": "system:masters",
      "OU": "VCCORP",
      "ST": "ThanhXuan"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin

}

# ==============

for instance in worker-0 worker-1 worker-2; do
cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF
done

EXTERNAL_IP=10.3.52.116

for instance in worker-0; do
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance},${EXTERNAL_IP} \
  -profile=kubernetes \
  ${instance}-csr.json | cfssljson -bare ${instance}
done

EXTERNAL_IP=10.3.53.167

for instance in worker-1; do
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance},${EXTERNAL_IP} \
  -profile=kubernetes \
  ${instance}-csr.json | cfssljson -bare ${instance}
done

EXTERNAL_IP=10.3.52.156

for instance in worker-2; do
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance},${EXTERNAL_IP} \
  -profile=kubernetes \
  ${instance}-csr.json | cfssljson -bare ${instance}
done



```bash
ls -l worker*pem
worker-0-key.pem
worker-0.pem
worker-1-key.pem
worker-1.pem
worker-2-key.pem
worker-2.pem
```

{

cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "VN",
      "L": "Hanoi",
      "O": "system:kube-controller-manager",
      "OU": "VCCORP",
      "ST": "ThanhXuan"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

}

```bash
kube-controller-manager.csr
kube-controller-manager-csr.json
kube-controller-manager-key.pem
kube-controller-manager.pem
```

{

cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "VN",
      "L": "Hanoi",
      "O": "system:kube-controller-manager",
      "OU": "VCCORP",
      "ST": "ThanhXuan"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy

}

```bash
kube-proxy.csr
kube-proxy-csr.json
kube-proxy-key.pem
kube-proxy.pem
```


{

cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "VN",
      "L": "Hanoi",
      "O": "system:kube-controller-manager",
      "OU": "VCCORP",
      "ST": "ThanhXuan"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler

}


```bash
kube-scheduler.csr
kube-scheduler-csr.json
kube-scheduler-key.pem
kube-scheduler.pem
```

{

KUBERNETES_PUBLIC_ADDRESS=10.3.52.52

cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "VN",
      "L": "Hanoi",
      "O": "system:kube-controller-manager",
      "OU": "VCCORP",
      "ST": "ThanhXuan"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.3.54.219,10.3.55.13,10.3.55.13,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,kubernetes.default \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes

}

```bash
kubernetes.csr
kubernetes-csr.json
kubernetes-key.pem
kubernetes.pem
```

{

cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "VN",
      "L": "Hanoi",
      "O": "system:kube-controller-manager",
      "OU": "VCCORP",
      "ST": "ThanhXuan"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account

}


```bash
service-account.csr
service-account-csr.json
service-account-key.pem
service-account.pem
```

```bash
for instance in k8s-worker-0 k8s-worker-1 k8s-worker-2; do
  scp ca.pem ${instance}-key.pem ${instance}.pem root@${instance}:~/
done
```
```bash
for instance in k8s-controller-0 k8s-controller-1 k8s-controller-2; do
  scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem ${instance}:~/
done
```

for instance in k8s-worker-0 k8s-worker-1 k8s-worker-2; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done

```bash
k8s-worker-0.kubeconfig
k8s-worker-1.kubeconfig
k8s-worker-2.kubeconfig
```

{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.pem \
    --client-key=kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
}

```bash
kube-proxy.kubeconfig
```

{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
}

```bash
kube-controller-manager.kubeconfig
```

{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
}

```bash
kube-scheduler.kubeconfig
```

{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default --kubeconfig=admin.kubeconfig
}

```bash
admin.kubeconfig
```


for instance in k8s-worker-0 k8s-worker-1 k8s-worker-2; do
  scp ${instance}.kubeconfig kube-proxy.kubeconfig root@${instance}:~/
done


for instance in k8s-controller-0 k8s-controller-1 k8s-controller-2; do
  scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig root@${instance}:~/
done

# ===========

ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)

cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF

for instance in k8s-controller-0 k8s-controller-1 k8s-controller-2; do
  scp encryption-config.yaml root@${instance}:~/
done


### controller 0




