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
