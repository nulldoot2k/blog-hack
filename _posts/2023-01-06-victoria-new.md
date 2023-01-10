---
permalink: ls/victoria-new.html
---

## NameSpace

> kubectl create namespace victoriametrics

## Add Repo

> helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

> helm repo add grafana https://grafana.github.io/helm-charts

> helm repo add vm https://victoriametrics.github.io/helm-charts/

## Update

> helm repo update

## Install 

> helm install node-exporter prometheus-community/prometheus-node-exporter -n victoriametrics

> helm install grafana grafana/grafana -n victoriametrics

> helm show values vm/victoria-metrics-agent > values.yaml
  - helm install vmagent vm/victoria-metrics-agent -f values.yaml -n victoriametrics

## Uninstall

> helm delete grafana

> helm uninstall node-exporter

> helm uninstall vmagent -n victoriametrics

## Account

- Grafana

> kubectl -n monitor get secret grafana -o jsonpath="{.data.admin-password}" `|` base64 --decode ; echo

> admin : Sa32SI8ZRREUqeyLi8PXjyatvriiVWkoHmIUHgXc

> kubectl -n monitor patch svc grafana -p '{"spec": {"type": "NodePort"}}'

---

Viet file
```yaml
apiVersion: v1 --> version của Kubernetes API
kind: Pod --> Chỉ định thành phần cần tạo (ở đây là Pod)
metadata: --> Tên của Pod
 name: kubia-manual --> Tên của Pod là kubia-manual
spec: --> Mô tả về các thông số kỹ thuật của Pod
 containers: --> chưa tên image của container cần chạy trong pod
 - image: luksa/kubia --> tên image của container cần chạy trong pod
 name: kubia --> Tên container
 ports: --> cổng của container
 - containerPort: 8080 --> Cổng đầu ra ứng dụng của container
 protocol: TCP --> giao thức tcp
```

- The main components include the following.

> vmstorage : data storage and query result return, the default port is 8482

> vminsert : data entry, similar to slicing and replica functions, default port 8480

> vmselect : data query, aggregation and data de-duplication, default port 8481

> vmagent : data metrics crawling, supports multiple back-end storage, will occupy the local disk cache, default port 8429

> vmalert : alarm-related components, not if you do not need the alarm function can not use the component, the default port is 8880

---

> helm install prometheus -n monitor prometheus-community/prometheus

NAME: prometheus
LAST DEPLOYED: Mon Jan  9 10:17:31 2023
NAMESPACE: pro
STATUS: deployed
REVISION: 1
NOTES:
The Prometheus server can be accessed via port 80 on the following DNS name from within your cluster:
prometheus-server.pro.svc.cluster.local


Get the Prometheus server URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace pro -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace pro port-forward $POD_NAME 9090


The Prometheus alertmanager can be accessed via port  on the following DNS name from within your cluster:
prometheus-%!s(<nil>).pro.svc.cluster.local


Get the Alertmanager URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace pro -l "app=prometheus,component=" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace pro port-forward $POD_NAME 9093

#################################################################################
######   WARNING: Pod Security Policy has been disabled by default since    #####
######            it deprecated after k8s 1.25+. use                        #####
######            (index .Values "prometheus-node-exporter" "rbac"          #####
###### .          "pspEnabled") with (index .Values                         #####
######            "prometheus-node-exporter" "rbac" "pspAnnotations")       #####
######            in case you still need it.                                #####
#################################################################################

The Prometheus PushGateway can be accessed via port 9091 on the following DNS name from within your cluster:
prometheus-prometheus-pushgateway.pro.svc.cluster.local


Get the PushGateway URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace pro -l "app=prometheus-pushgateway,component=pushgateway" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace pro port-forward $POD_NAME 9091

For more information on running Prometheus, visit:
https://prometheus.io/


---

> helm install node-exporter prometheus-community/prometheus-node-exporter --namespace monitoring —set fullnameOverride=node-exporter

> helm install kube-state-metrics prometheus-community/kube-state-metrics --namespace monitoring

