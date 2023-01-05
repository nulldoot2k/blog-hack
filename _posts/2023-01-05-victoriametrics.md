---
permalink: ls/victoria-metrics.html
---

## Link

> https://medium0.com/@dotdc/a-set-of-modern-grafana-dashboards-for-kubernetes-4b989c72a4b2

```bash
helm install node-exporter prometheus-community/prometheus-node-exporter --namespace monitoring —set fullnameOverride=node-exporter

helm install kube-state-metrics prometheus-community/kube-state-metrics --namespace monitoring
```


Step 

> git clone https://github.com/dotdc/grafana-dashboards-kubernetes.git
cd grafana-dashboards-kubernetes


