# 用於 Kubernetes 的 OpenTelemetry Operator

原文: [OpenTelemetry Operator for Kubernetes](https://opentelemetry.io/docs/k8s-operator/)

Kubernetes Operator 的實現，它使用 OpenTelemetry 檢測庫管理收集器和工作負載的自動檢測。

## 介紹

OpenTelemetry Operator 是 [Kubernetes Operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) 的實現。

Operator 管理：

- OpenTelemetry Collector
- 使用 OpenTelemetry 檢測庫自動檢測(auto-instrumentation)工作負載

## 入門

要在現有集群中安裝 operator，請確保已安裝並運行 cert-manager。

執行如下命令，安裝cert-manager:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.10.0/cert-manager.yaml
```

默認情況下，cert-manager 將安裝到 `cert-manager` 命名空間中。可以在不同的命名空間中運行 cert-manager，儘管您需要對部署清單進行修改。

執行如下命令，安裝 OpenTelemetry Operator:

```bash
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml
```

執行如下命令，安裝 OpenTelemetry Collector:

```bash
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: simplest
spec:
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
          http:
    processors:

    exporters:
      logging:
        loglevel: debug

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: []
          exporters: [logging]
EOF
```

執行如下命令，安裝 OpenTelemetry Auto-Instrumentation:

```bash
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: my-java-instrumentation
spec:
  exporter:
    endpoint: http://simplest-collector.default:4317
  propagators:
    - tracecontext
    - baggage
    - b3
  java:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:latest
  nodejs:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-nodejs:latest
  python:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:latest
  dotnet:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-dotnet:latest
  env:
    - name: OTEL_RESOURCE_ATTRIBUTES
      value: service.name=your_service,service.namespace=your_service_namespace
EOF
```

針對服務名、服務空間等參數，您可以通過 Auto-Instrumentation 環境變量進行傳遞。存在多個應用時，建議創建多個 Auto-Instrumentation 以區分服務名。

!!! info
    多個 Auto-Instrumentation 中的 metadata.name 參數不可重複。

針對不同開發語言的應用（java、nodejs、python、dotnet等） ，請根據實際部署的情況，配置對應語言的 Auto-Instrumentation。

## 在配置文件中添加自動注入探針配置

根據實際情況，在具體應用的配置文件中添加腳本。目前只支持 Python、Node.js、Java 和 dotNET 應用。

=== "dotNET"
        ```yaml
        instrumentation.opentelemetry.io/inject-dotnet: "my-dotnet-instrumentation"
        instrumentation.opentelemetry.io/inject-sdk: "my-dotnet-instrumentation"
        ```

=== "Java"
        ```yaml
        instrumentation.opentelemetry.io/inject-java: "my-java-instrumentation"
        instrumentation.opentelemetry.io/inject-sdk: "my-java-instrumentation"
        ```

=== "Node.js"
        ```yaml
        instrumentation.opentelemetry.io/inject-nodejs: "my-nodejs-instrumentation"
        instrumentation.opentelemetry.io/inject-sdk: "my-nodejs-instrumentation"
        ```

=== "Python"
        ```yaml
        instrumentation.opentelemetry.io/inject-python: "my-python-instrumentation"
        instrumentation.opentelemetry.io/inject-sdk: "my-python-instrumentation"
        ```

## 測試

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  annotations:
    sidecar.opentelemetry.io/inject: "true"
spec:
  containers:
  - name: myapp
    image: jaegertracing/vertx-create-span:operator-e2e-tests
    ports:
      - containerPort: 8080
        protocol: TCP
EOF
```

## OpenTelemetry 儀器化自動注入

Operator 可以注入和配置 OpenTelemetry 自動儀器化庫。目前支持 DotNet、Java、NodeJS 和 Python。

要使用自動檢測，請使用 SDK 和檢測的配置配置檢測資源。

```bash
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: my-instrumentation
spec:
  exporter:
    endpoint: http://simplest-collector.default:4317
  propagators:
    - tracecontext
    - baggage
    - b3
  sampler:
    type: parentbased_traceidratio
    argument: "0.25"
  java:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:latest
  nodejs:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-nodejs:latest
  python:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:latest
  dotnet:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-dotnet:latest
  env:
    - name: OTEL_TRACES_EXPORTER
      value: "otlp"
    - name: OTEL_METRICS_EXPORTER
      value: "prometheus"
    - name: OTEL_EXPORTER_PROMETHEUS_PORT
      value: "9464"
EOF
```

上面的 CR 可以通過 `kubectl get otelinst` 查詢。

然後向 pod 添加 annotation 以啟用注入。可以將 annotation 添加到命名空間，以便該命名空間內的所有 pod 都將獲得檢測，或者通過將註釋添加到單個 PodSpec 對象，這些對象可作為 Deployment、Statefulset 和其他資源的一部分使用。

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: dotnet-todoapi
  annotations:
    instrumentation.opentelemetry.io/inject-dotnet: "true"
spec:
  containers:
  - name: dotnet-todoapi
    image: witlab/dotnet-todoapi:base
    ports:
      - containerPort: 8080
        protocol: TCP
EOF
```

### Java

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: springboot-petclinic
  annotations:
    instrumentation.opentelemetry.io/inject-java: "true"
spec:
  containers:
  - name: springboot-petclinic
    image: ikolaxis/petclinic:java11-openj9
    ports:
      - containerPort: 8080
        protocol: TCP
EOF
```

### Python

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: flask-restapi
  annotations:
    instrumentation.opentelemetry.io/inject-python: "true"
spec:
  containers:
  - name: flask-restapi
    image: bbachin1/flask-restapi:latest
    ports:
      - containerPort: 5000
        protocol: TCP
EOF
```