---
permalink: ls/meta-2.html
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

Certificate Authority
```bash
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
```

```bash
ca-key.pem
ca.pem
```

Client and Server Certificates
```bash
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
```

```bash
admin.csr
admin-csr.json
admin-key.pem
admin.pem
```

The Kubelet Client Certificates
```bash
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
```
```bash
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
```

result
```bash
ls -l worker*pem
worker-0-key.pem
worker-0.pem
worker-1-key.pem
worker-1.pem
worker-2-key.pem
worker-2.pem
```

The Controller Manager Client Certificate
```bash
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
```

The Kube Proxy Client Certificate
```bash
kube-controller-manager.csr
kube-controller-manager-csr.json
kube-controller-manager-key.pem
kube-controller-manager.pem
```

```bash
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
```

```bash
kube-proxy.csr
kube-proxy-csr.json
kube-proxy-key.pem
kube-proxy.pem
```

The Scheduler Client Certificate
```bash
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

The Kubernetes API Server Certificate
```bash
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
```

```bash
kubernetes.csr
kubernetes-csr.json
kubernetes-key.pem
kubernetes.pem
```

The Service Account Key Pair
```bash
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
```

```bash
service-account.csr
service-account-csr.json
service-account-key.pem
service-account.pem
```

Distribute the Client and Server Certificates
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

KUBERNETES_PUBLIC_ADDRESS=10.3.52.52
```bash
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
```

```bash
k8s-worker-0.kubeconfig
k8s-worker-1.kubeconfig
k8s-worker-2.kubeconfig
```

The kube-proxy Kubernetes Configuration File
```bash
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
```

```bash
kube-proxy.kubeconfig
```

The kube-controller-manager Kubernetes Configuration File
```bash
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
```

```bash
kube-controller-manager.kubeconfig
```

The kube-scheduler Kubernetes Configuration File
```bash
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
```

```bash
kube-scheduler.kubeconfig
```

The admin Kubernetes Configuration File
```bash
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
```

```bash
admin.kubeconfig
```

Distribute the Kubernetes Configuration Files
```bash
for instance in k8s-worker-0 k8s-worker-1 k8s-worker-2; do
  scp ${instance}.kubeconfig kube-proxy.kubeconfig root@${instance}:~/
done


for instance in k8s-controller-0 k8s-controller-1 k8s-controller-2; do
  scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig root@${instance}:~/
done
```

### The Encryption Key

```bash
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
```

### controller 0




