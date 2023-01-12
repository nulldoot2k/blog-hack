---
permalink: ls/dashboard-k8s.html
---

## Install

> kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

## check

> kubectl get svc -n kubernetes-dashboard -o wide
```bash
NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE   SELECTOR
dashboard-metrics-scraper   ClusterIP   10.108.64.221   <none>        8000/TCP        24m   k8s-app=dashboard-metrics-scraper
kubernetes-dashboard        NodePort    10.104.95.243   <none>        443:32331/TCP   24m   k8s-app=kubernetes-dashboard
```

> kubectl get nodes -n kubernetes-dashboard -o wide
```bash
NAME                                  STATUS     ROLES     AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
chien                                 NotReady   worker3   9d    v1.26.0   10.20.1.73    <none>        Ubuntu 20.04.5 LTS   5.4.0-113-generic   containerd://1.6.14
dat-k8s-1                             NotReady   worker1   10d   v1.26.0   10.20.1.207   <none>        Ubuntu 20.04.5 LTS   5.4.0-113-generic   containerd://1.6.14
kubernetes-master-1.heyvaldemar.net   Ready      master    10d   v1.26.0   10.20.1.116   <none>        Ubuntu 20.04.5 LTS   5.4.0-113-generic   containerd://1.6.14
ubuntu-vyhamii-test1                  NotReady   master1   7d    v1.26.0   10.3.52.52    <none>        Ubuntu 20.04 LTS     5.4.0-40-generic    containerd://1.6.14
vy-ansible                            NotReady   worker2   10d   v1.26.0   10.20.1.173   <none>        Ubuntu 20.04.5 LTS   5.4.0-113-generic   containerd://1.6.14
vy-gr                                 NotReady   <none>    10d   v1.26.0   10.20.1.93    <none>        Ubuntu 20.04.5 LTS   5.4.0-132-generic   containerd://1.6.14
```

> kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard
```bash
kubectl --namespace kubernetes-dashboard patch svc kubernetes-dashboard -p '{"spec": {"type": "NodePort"}}'
```
```bash
change: ClusterIP --> NodePort
add: 	nodePort: 32331
:x --> EXIT AND SAVED
```


> kubectl get sa -n kubernetes-dashboard
```bash
NAME                   SECRETS   AGE
default                0         21m
kubernetes-dashboard   0         21m
```

> kubectl describe sa kubernetes-dashboard -n kubernetes-dashboard

```bash
token: ...
```
- note that: if not show token then follow step below here!

## create token

> kubectl create token default --allow-missing-template-keys=true

Add to https

> https://localhost:32331/#/login


