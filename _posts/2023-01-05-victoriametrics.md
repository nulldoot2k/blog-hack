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

### ============

Node-Exporter
> helm install node-exporter prometheus-community/prometheus-node-exporter -n stack

```bash
NAME: node-exporter
LAST DEPLOYED: Thu Jan  5 14:44:12 2023
NAMESPACE: stack
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace stack -l "app.kubernetes.io/name=prometheus-node-exporter,app.kubernetes.io/instance=node-exporter" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:9100 to use your application"
  kubectl port-forward --namespace stack $POD_NAME 9100
```

Grafana
> helm install my-release grafana/grafana -n stack

```bash
NAME: my-release
LAST DEPLOYED: Thu Jan  5 14:46:28 2023
NAMESPACE: stack
STATUS: deployed
REVISION: 1
NOTES:
1. Get your 'admin' user password by running:

   kubectl get secret --namespace stack my-release-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
   --> CNPhVbTM9T7kXtEEJq3jG3tZoORpW4eTK0RECjER

2. The Grafana server can be accessed via port 80 on the following DNS name from within your cluster:

   my-release-grafana.stack.svc.cluster.local

   Get the Grafana URL to visit by running these commands in the same shell:
     export POD_NAME=$(kubectl get pods --namespace stack -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=my-release" -o jsonpath="{.items[0].metadata.name}")
     kubectl --namespace stack port-forward $POD_NAME 3000

3. Login with the password from step 1 and the username: admin
#################################################################################
######   WARNING: Persistence is disabled!!! You will lose your data when   #####
######            the Grafana pod is terminated.                            #####
#################################################################################
```

Metrics
> helm install metrics prometheus-community/kube-state-metrics -n stack

```bash
NAME: metrics
LAST DEPLOYED: Thu Jan  5 14:45:42 2023
NAMESPACE: stack
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
kube-state-metrics is a simple service that listens to the Kubernetes API server and generates metrics about the state of the objects.
The exposed metrics can be found here:
https://github.com/kubernetes/kube-state-metrics/blob/master/docs/README.md#exposed-metrics

The metrics are exported on the HTTP endpoint /metrics on the listening port.
In your case, metrics-kube-state-metrics.stack.svc.cluster.local:8080/metrics

They are served either as plaintext or protobuf depending on the Accept header.
They are designed to be consumed either by Prometheus itself or by a scraper that is compatible with scraping a Prometheus client endpoint.
```

Prometheus

> helm install prometheus prometheus-community/kube-prometheus-stack -n stack

