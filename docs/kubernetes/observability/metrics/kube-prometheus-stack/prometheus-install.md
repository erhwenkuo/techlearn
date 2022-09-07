# 安裝 Prometheus

監控的解決方案有很多種，我這裡選擇的是 Prometheus。實際上只有 Prometheus 還不夠，真正其實會安裝以下項目：

1. Grafana - 視覺化工具，可以用圖形顯示監控的資料。（其實還可以做 distribute tracing, log 查詢）
2. Prometheus。
3. Prometheus Operator - 提供 Kubernetes CRD，讓我們透過 CRD 設定 Prometheus 的設定。
4. Prometheus Node Exporter - 蒐集 Kubernetes Worker Node 的資訊（Linux 提供的資訊）。
5. kube-state-metrics - 蒐集 Kubernetes 的資訊（Kubernetes API Server 提供的資訊）。
6. Prometheus Adapter for Kubernetes Metrics APIs.

以上這些安裝項目都可以用 [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) 這個專案提供的 helm chart 安裝。

!!! info
    註 - [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus) 專案其實僅提供 YAML 的方式安裝。而 [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) 專案將 kube-prometheus 包裝成 helm chart。

## 安裝 kube-prometheus-stack

用 helm 安裝：

```bash
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
$ helm repo update
$ helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
```

參考: https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack

## 檢視 Web UI

當安裝完畢後，有 3 個 Web UI 可以使用，參考如下：

**Prometheus-UI**

```bash
kubectl port-forward service/prometheus-kube-prometheus-prometheus 9090
```

**Alert Manager UI**

```bash
kubectl port-forward svc/prometheus-kube-prometheus-alertmanager 9093
```

**Grafana**

```bash
kubectl port-forward deployment/prometheus-grafana 3000
```

Grafana 預設的帳號、密碼如下：

```
user: admin
pwd: prom-operator
```

## PodMonitor, ServiceMonitor

安裝 helm chart 的時候，我認為建議將以下值設定為 false，讓 Prometheus 抓取全部的 PodMonitor, ServiceMonitor，不透過 label 篩選（避免學習的時候會發生 PodMonitor, ServiceMonitor 找不到的問題）：

```bash
prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues
prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues
```

