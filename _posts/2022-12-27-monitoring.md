---
permalink: ls/prometheus-monitoring-stack-k8s.html
---

### Prometheus

- git clone https://github.com/techiescamp/kubernetes-prometheus

> kubectl create namespace monitoring

```bash
kubectl create -f clusterRole.yaml
kubectl create -f config-map.yaml
kubectl create -f prometheus-deployment.yaml
```

kubectl get deployments --namespace=monitoring

```logs
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
alertmanager            1/1     1            1           56m
grafana                 1/1     1            1           23m
kube-state-metrics      1/1     1            1           5h1m
prometheus-deployment   1/1     1            1           156m
```

> kubectl get pods --namespace=monitoring

```logs
NAME                                           READY   STATUS             RESTARTS         AGE
alertmanager-6d77489f7b-nbwjd                  1/1     Running            0                54m
grafana-68b7b49968-t89zv                       1/1     Running            0                22m
kube-state-metrics-69cb6bf49c-q989j            1/1     Running            0                5h
node-exporter-prometheus-node-exporter-blksv   1/1     Running            0                5h5m
node-exporter-prometheus-node-exporter-l5c4d   1/1     Running            0                5h5m
node-exporter-prometheus-node-exporter-n4cxs   1/1     Running            0                5h5m
node-exporter-prometheus-node-exporter-nw7ms   0/1     CrashLoopBackOff   64 (3m1s ago)    5h5m
node-exporter-prometheus-node-exporter-zwpwt   0/1     CrashLoopBackOff   64 (3m33s ago)   5h5m
prometheus-deployment-954488b65-2chgx          1/1     Running            0                155m
```


### kube state metric

helm install kube-state-metrics prometheus-community/kube-state-metrics --namespace monitoring

	NAME: kube-state-metrics
	LAST DEPLOYED: Tue Dec 27 10:14:50 2022
	NAMESPACE: monitoring
	STATUS: deployed
	REVISION: 1
	TEST SUITE: None
	NOTES:
	kube-state-metrics is a simple service that listens to the Kubernetes API server and generates metrics about the state of the objects.
	The exposed metrics can be found here:
	https://github.com/kubernetes/kube-state-metrics/blob/master/docs/README.md#exposed-metrics

	The metrics are exported on the HTTP endpoint /metrics on the listening port.
	In your case, kube-state-metrics.monitoring.svc.cluster.local:8080/metrics

	They are served either as plaintext or protobuf depending on the Accept header.
	They are designed to be consumed either by Prometheus itself or by a scraper that is compatible with scraping a Prometheus client endpoint.


### AlertManager

```yaml
alerting:
  alertmanagers:
    - scheme: http
      static_configs:
      - targets:
        - "alertmanager.monitoring.svc:9093"
```

- config.yaml

```yaml
rule_files:
  - /etc/prometheus/prometheus.rules
```

- git clone https://github.com/bibinwilson/kubernetes-alert-manager.git

```bash
kubectl create -f AlertManagerConfigmap.yaml
kubectl create -f AlertTemplateConfigMap.yaml
kubectl create -f Deployment.yaml
```

> kubectl create -f Service.yaml

### Node_Exporter

helm install node-exporter prometheus-community/prometheus-node-exporter --namespace monitoring

	NAME: node-exporter
	LAST DEPLOYED: Tue Dec 27 10:09:49 2022
	NAMESPACE: monitoring
	STATUS: deployed
	REVISION: 1
	TEST SUITE: None
	NOTES:
	1. Get the application URL by running these commands:
		export POD_NAME=$(kubectl get pods --namespace monitoring -l "app.kubernetes.io/name=prometheus-node-exporter,app.kubernetes.io/instance=node-exporter" -o jsonpath="{.items[0].metadata.name}")
		echo "Visit http://127.0.0.1:9100 to use your application"
		kubectl port-forward --namespace monitoring $POD_NAME 9100
		
	curl 14.225.44.60

### Grafana

helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

    kubectl get deployments --namespace=monitoring
      NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
      kube-state-metrics      1/1     1            1           20m
      prometheus-deployment   1/1     1            1           94s
      traefik                 1/1     1            1           49m

> kubectl get pods --namespace=monitoring

	prometheus-deployment-67cf879cc4-r98k9         1/1     Running            0              3m43s
	

- git clone https://github.com/bibinwilson/kubernetes-grafana.git

```bash
kubectl create -f grafana-datasource-config.yaml
kubectl create -f deployment.yaml
kubectl create -f service.yaml
```

> Account

```bash
Pass: admin
User: admin
```

### vmAgent


