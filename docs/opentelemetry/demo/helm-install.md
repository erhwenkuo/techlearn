# OpenTelemetry Demo

![](./assets/istio-ecosystem.png)

請跟隨本教程一起，使用 Helm 安裝、配置、並深入評估 Istio 網格系統。本教程會安裝下列的元件:

- Ubuntu 20.04 (O.S)
- Docker
- Kubernetes (K3D)
- Metallb
- Nginx Ingress Controller
- Istio
- Kiali
- OpenTelemetry
- Prometheus
- Grafana
- Tempo

同時也會利用不同類型的範例應用來驗證不同元件的功能與彼此的整合，其中會特別值得關注的是:

1. 啟動 Istio tracing 的功能與設定拋轉 envoy 的 tracing 資訊到 Tempo
2. 啟動 OpenTelemetryCollector 並設定基本的 tracing receiver 與 exporter
3. 啟動 Grafana Tempo (v2) 來接收由 OTel Collector 傳送進來的 tracing 資料
4. 設定 Grafana TracingQL 的 UI plugin, 並用來查找 tracing 相關數據

## 步驟 01 - 環境安裝

![](./assets/lab-env.png)

### 先決條件

- 安裝 Helm 客戶端，版本 3.6 或更高版本。
- 配置 Helm 存儲庫：

```bash title="執行下列命令  >_"
helm repo add istio https://istio-release.storage.googleapis.com/charts

helm repo update
```

### 創建本機 Docker Network

使用 docker 創建一個虛擬的網路來做為本次教程的網路架構。

|   |   |
|--- |---|
|CIDR|172.22.0.0/24|
|CIDR IP Range|172.20.0.0 - 172.20.0.255|
|IPs|256|
|Subnet Mask|255.255.255.0|
|Gateay|172.20.0.1|

```bash
docker network create \
  --driver=bridge \
  --subnet=172.20.0.0/24 \
  --gateway=172.20.0.1 \
  lab-network
```

檢查 Docker 虛擬網絡 `lab-network` 的設定。

```bash
docker network inspect lab-network
```

結果:

```json hl_lines="14-15"
[
    {
        "Name": "lab-network",
        "Id": "2e2ca22fbb712cbc19d93acb16fc4e1715488c4c18b82d12dba4c1634ac5b1b6",
        "Created": "2023-02-09T23:07:33.186003336+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.20.0.0/24",
                    "Gateway": "172.20.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
```

讓我們從這個虛擬網段裡的 CIDR IP Range 中保留 5 個 IP (`172.20.0.10-172.20.0.15`) 來做本次的練習。

### 創建 K8S 集群

執行下列命令來創建實驗 Kubernetes 集群:

```bash title="執行下列命令  >_"
k3d cluster create  --api-port 6443 \
  --port 8080:80@loadbalancer --port 8443:443@loadbalancer \
  --k3s-arg "--disable=traefik@server:0" \
  --k3s-arg "--disable=servicelb@server:0" \
  --network lab-network
```

參數說明:

- `--k3s-arg "--disable=servicelb@server:0"` 不安裝 K3D 預設的 traefik (IngressController), 我們將使用　nginx ingress controller
- `--k3s-arg "--disable=traefik@server:0"` 不安裝 K3D 預設的 servicelb (klipper-lb), 我們將使用 metallb
- `--network lab-network` 使用預先創建的 docker 虛擬網段

### 安裝/設定 MetalLB

在本次的結構裡, 有兩個負責南北向 traffic 的亓件, 主要原因是 Nginx Ingress Controller 是很多團隊己經使用的 ingress 的元件, 雖說 Istio Ingress Gateway 也可負責對外開放在 Kubernetes 裡頭的服務, 但是對大多數的團隊來說這樣的轉換需要熟悉與過渡。因此在本次的 lab 架構會同時並存這兩個元件並且使用 metallb 來配置固定的 IP。

![](./assets/dual-north-south-traffic.png)

#### Helm 安裝

使用 Helm 的手法來進行 Ｍetallb 安裝:

```bash
#　setup helm repo
helm repo add metallb https://metallb.github.io/metallb

# install metallb to specific namespace
helm upgrade --install --create-namespace --namespace metallb-system \
  metallb metallb/metallb
```

!!! tips
    Ｍetallb 在[Version 0.13.2](https://metallb.universe.tf/release-notes/#version-0-13-2) 版本有一個很重大的修改:

    - 新功能:　支持CRD！期待已久的功能 MetalLB 現在可通過 CR 進行配置。
    - 行為變化:　最大的變化是引入了 CRD 並刪除了對通過 ConfigMap 進行配置的支持。

#### 設定 IP Adress Pool

我們將使用 MetalLB 的 Layer 2 模式是最簡單的配置：在大多數的情況下，你不需要任何特定於協議的配置，只需要 IP 地址範圍。

Layer 2 模式模式不需要將 IP 綁定到工作程序節點的網絡接口。它通過直接響應本地網絡上的 ARP 請求來工作，將機器的 MAC 地址提供給客戶端。

讓我們使用 CRD 來設定 Metallb:

```bash hl_lines="9"
cat <<EOF | kubectl apply -n metallb-system -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: ip-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.20.0.10-172.20.0.15
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2advertise
  namespace: metallb-system
spec:
  ipAddressPools:
  - ip-pool
EOF
```

結果:

```bash
ipaddresspool.metallb.io/ip-pool created
l2advertisement.metallb.io/l2advertise created
```

!!! tip
    如果只有一個 IP 要讓 Metallb 來給予，那麼 CIDR 的設定可設成 172.20.0.5/32 (也就是只有一個 IP: `172.20.0.5` 可被指派使用)

### 安裝/設定 Nginx Ingress Controller

#### Helm 安裝

使用以下命令添加 Nginx Ingress Controller 的 chart 存儲庫：

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

helm repo update
```

設定 `ingress-nginx` 要從 metallb 取得特定的預設 IP (`172.20.0.13`):

```yaml title="ingress-nginx-values.yaml"
controller:
  # add annotations to get ip from metallb
  service:
    annotations:
      metallb.universe.tf/address-pool: ip-pool
    loadBalancerIP: "172.20.0.13"
  # set ingressclass as default
  ingressClassResource:
    default: true
```

將 Nginx Ingress Controller 安裝到 kube-system 命名空間中：

```bash
helm upgrade --install \
     --create-namespace --namespace kube-system \
     ingress-nginx ingress-nginx/ingress-nginx \
     --values ingress-nginx-values.yaml
```

檢查:

```bash
kubectl get svc -n kube-system
```

結果:

```
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.43.160.250   172.20.0.13    80:30672/TCP,443:30990/TCP   91s
```

!!! tip
    特別注意 `ingress-nginx-controller` 的 EXTERNAL-IP 是否從 metallb 取得 `172.20.0.13`

#### 驗證 Ingress 設定

創建一個 Nginx 的 Deployment 與 Service:

```bash
kubectl create deployment nginx --image=nginx

kubectl create service clusterip nginx --tcp=80:80
```

創建 Ingress 來曝露這個測試的 Nginx 網站:

```bash
kubectl apply -f -<<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-nginx-svc
spec:
  rules:
  - host: "nginx.example.it"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: nginx
            port:
              number: 80
EOF
```

檢查看這個 ingress 是否有取得 IP ADDRESS:

```bash
kubectl get ing/ingress-nginx-svc
```

結果:

```
NAME                CLASS    HOSTS              ADDRESS       PORTS   AGE
ingress-nginx-svc   <none>   nginx.example.it   172.20.0.13   80      21s
```

修改 `/etc/hosts` 來增加一筆 entry 來模擬 DNS 解析:

``` title="/etc/hosts"
...
172.20.0.13  nginx.example.it
...
```

使用瀏覽器瀏覽 `http://nginx.example.it`:

![](./assets/ingress-test-nginx.png)

### 安裝/設定 OTel Demo

參考: https://github.com/open-telemetry/opentelemetry-helm-charts/tree/main/charts/opentelemetry-demo

#### Helm chart 參數

Helm chart 參數分為 4 個區塊:

- Default - 用於指定應用於所有演示組件的默認值
- Components - 用於為演示配置各個組件（微服務）
- Observability - 用於啟用/禁用依賴項
- Sub-charts - 所有子 chart 的配置

**Default**:

| Property | Description | Default |
|----------|-------------|---------|
|default.env	|Environment variables added to all components	|Array of several OpenTelemetry environment variables|
|default.envOverrides	|Used to override individual environment variables without re-specifying the entire array.	|[]|
|default.image.repository	|Demo components image name	|otel/demo|
|default.image.tag	|Demo components image tag (leave blank to use app version)	|nil|
|default.image.pullPolicy	|Demo components image pull policy	|IfNotPresent|
|default.image.pullSecrets	|Demo components image pull secrets	|[]|
|default.schedulingRules.nodeSelector	|Node labels for pod assignment	|{}|
|default.schedulingRules.affinity	|Man of node/pod affinities	|{}|
|default.schedulingRules.tolerations	|Tolerations for pod assignment	|[]|
|default.securityContext	|Demo components container security context	|{}|
|serviceAccount.annotations	|Annotations for the serviceAccount	|{}|
|serviceAccount.create	|Whether to create a serviceAccount or use an existing one	|true|
|serviceAccount.name	|The name of the ServiceAccount to use for demo components	|""|

**Component**:

OpenTelemetry 演示包含多個組件（微服務）。每個組件都配置了一組通用參數。所有組件都將在 components.[NAME] 中定義，其中 [NAME] 是演示組件的名稱。

!!! info
    以下參數需要 components.[NAME]。前綴，其中 [NAME] 是演示組件的名稱

會被套用於所有演示組件

| Property | Description | Default |
|----------|-------------|---------|
|enabled	|Is this component enabled	|true|
|useDefault.env	|Use the default environment variables in this component	|true|
|imageOverride.repository	|Name of image for this component	|Defaults to the overall default image repository|
|imageOverride.tag	|Tag of the image for this component	|Defaults to the overall default image tag|
|imageOverride.pullPolicy	|Image pull policy for this component	|IfNotPresent|
|imageOverride.pullSecrets	|Image pull secrets for this component	|[]|
|service.type	|Service type used for this component	|ClusterIP|
|service.port	|Service port used for this component	|nil|
|service.nodePort	|Service node port used for this component	|nil|
|service.annotations	|Annotations to add to the component's service	|{}|
|ports	|Array of ports to open for deployment and service of this component	|[]|
|env	|Array of environment variables added to this component	|Each component will have its own set of environment variables|
|envOverrides	|Used to override individual environment variables without re-specifying the entire array	|[]|
|resources	|CPU/Memory resource requests/limits	|Each component will have a default memory limit set|
|schedulingRules.nodeSelector	|Node labels for pod assignment	|{}|
|schedulingRules.affinity	|Man of node/pod affinities	|{}|
|schedulingRules.tolerations	|Tolerations for pod assignment	|[]|
|securityContext	|Container security context to define user ID (UID), group ID (GID) and other security policies	|{}|
|podAnnotations	|Pod annotations for this component	|{}|
|ingress.enabled	|Enable the creation of Ingress rules	|false|
|ingress.annotations	|Annotations to add to the ingress rule	|{}|
|ingress.ingressClassName	|Ingress class to use. If not specified default Ingress class will be used.	|nil|
|ingress.hosts	|Array of Hosts to use for the ingress rule.	|[]|
|ingress.hosts[].paths	|Array of paths / routes to use for the ingress rule host.	|[]|
|ingress.hosts[].paths[].path	|Actual path route to use	|nil|
|ingress.hosts[].paths[].pathType	|Path type to use for the given path. Typically this is Prefix.	|nil|
|ingress.hosts[].paths[].port	|Port to use for the given path	|nil|
|ingress.additionalIngresses	|Array of additional ingress rules to add. This is handy if you need to differently annotated ingress rules	|[]|
|ingress.additionalIngresses[].name	|Each additional ingress rule needs to have a unique name	|nil|
|command	|Command & arguments to pass to the container being spun up for this service	|[]|
|configuration	|Configuration for the container being spun up; will create a ConfigMap, Volume and VolumeMount	|{}|
|initContainers	|Array of init containers to add to the pod	|[]|
|initContainers[].name	|Name of the init container	|nil|
|initContainers[].image	|Image to use for the init container	|nil|
|initContainers[].command	|Command to run for the init container	|nil|

**Sub-chart**:

OpenTelemetry Demo Helm chart 依賴於 4 個子 chart：

- OpenTelemetry Collector
- Jaeger
- Prometheus
- Grafana

每個子 chart 的參數可以在該子圖各自的頂層中指定。默認情況下，此圖表將覆蓋一些相關的子圖表參數。覆蓋的參數在下面指定。

**OpenTelemetry Collector**

以下參數有一個 `opentelemetry-collector.` 前綴。

| Property | Description | Default |
|----------|-------------|---------|
|enabled	|Install the OpenTelemetry collector	|true|
|nameOverride	|Name that will be used by the sub-chart release	|otelcol|
|mode	The |Deployment or Daemonset mode	|deployment|
|resources	|CPU/Memory resource requests/limits	|100Mi memory limit|
|service.type	|Service Type to use	|ClusterIP|
|ports	|Ports to enabled for the collector pod and service	|metrics is enabled and prometheus is defined/enabled|
|podAnnotations	|Pod annotations	|Annotations leveraged by Prometheus scrape|
|config	|OpenTelemetry Collector configuration	|Configuration required for demo|

**Jaeger**

以下參數有一個 `jaeger.` 前綴。

| Property | Description | Default |
|----------|-------------|---------|
|enabled	|Install the Jaeger sub-chart	|true|
|provisionDataStore.cassandra	|Provision a cassandra data store	|false (required for AllInOne mode)|
|allInOne.enabled	|Enable All in One In-Memory Configuration	|true|
|allInOne.args	|Command arguments to pass to All in One deployment	|["--memory.max-traces", "10000", "--query.base-path", "/jaeger/ui"]|
|allInOne.resources	|CPU/Memory resource requests/limits for All in One	|275Mi memory limit|
|storage.type	|Storage type to use	|none (required for AllInOne mode)|
|agent.enabled	|Enable Jaeger agent	|false (required for AllInOne mode)|
|collector.enabled	|Enable Jaeger Collector	|false (required for AllInOne mode)|
|query.enabled	|Enable Jaeger Query	|false (required for AllInOne mode)|


**Prometheus**

以下參數有一個 `prometheus.` 前綴。

| Property | Description | Default |
|----------|-------------|---------|
|enabled	|Install the Prometheus sub-chart	|true|
|alertmanager.enabled	|Install the alertmanager	|false|
|configmapReload.prometheus.enabled	|Install the configmap-reload container	|false|
|kube-state-metrics.enabled	|Install the kube-state-metrics sub-chart	|false|
|prometheus-node-exporter.enabled	|Install the Prometheus Node Exporter sub-chart	|false|
|prometheus-pushgateway.enabled	|Install the Prometheus Push Gateway sub-chart	|false|
|server.extraFlags	|Additional flags to add to Prometheus server	|["enable-feature=exemplar-storage"]|
|server.persistentVolume.enabled	|Enable persistent storage for Prometheus data	|false|
|server.global.scrape_interval	|How frequently to scrape targets by default	|5s|
|server.global.scrap_timeout	|How long until a scrape request times out	|3s|
|server.global.evaluation_interval	|How frequently to evaluate rules	|30s|
|service.servicePort	|Service port used	|9090|
|serverFiles.prometheus.yml	|Prometheus configuration file	|Scrape config to get metrics from OpenTelemetry collector|


**Grafana**

以下參數有一個 `grafana.` 前綴。

| Property | Description | Default |
|----------|-------------|---------|
|enabled	|Install the Grafana sub-chart	|true|
|grafana.ini	|Grafana's primary configuration	|Enables anonymous login, and proxy through the frontendProxy service|
|adminPassword	|Password used by admin user	|admin|
|rbac.pspEnabled	|Enable PodSecurityPolicy resources	|false|
|datasources	|Configure grafana datasources (passed through tpl)	|Prometheus and Jaeger data sources|
|dashboardProviders	|Configure grafana dashboard providers	|Defines a default provider based on a file path|
|dashboardConfigMaps	|ConfigMaps reference that contains dashboards	|Dashboard config map deployed with this Helm chart|

#### Helm 安裝

使用以下命令添加 OpenTelemetry 的 chart 存儲庫：

```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts

helm repo update
```

要安裝版本名稱為 `otel-demo` 的 chart，請運行以下命令：

```bash title="執行下列命令  >_"
helm upgrade --install --create-namespace --namespace default \
  otel-demo open-telemetry/opentelemetry-demo 
```

結果:

```bash
Release "otel-demo" does not exist. Installing it now.
NAME: otel-demo
LAST DEPLOYED: Thu May 11 00:45:34 2023
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
=======================================================================================


 ██████╗ ████████╗███████╗██╗         ██████╗ ███████╗███╗   ███╗ ██████╗
██╔═══██╗╚══██╔══╝██╔════╝██║         ██╔══██╗██╔════╝████╗ ████║██╔═══██╗
██║   ██║   ██║   █████╗  ██║         ██║  ██║█████╗  ██╔████╔██║██║   ██║
██║   ██║   ██║   ██╔══╝  ██║         ██║  ██║██╔══╝  ██║╚██╔╝██║██║   ██║
╚██████╔╝   ██║   ███████╗███████╗    ██████╔╝███████╗██║ ╚═╝ ██║╚██████╔╝
 ╚═════╝    ╚═╝   ╚══════╝╚══════╝    ╚═════╝ ╚══════╝╚═╝     ╚═╝ ╚═════╝


- All services are available via the Frontend proxy: http://localhost:8080
  by running these commands:
     kubectl port-forward svc/otel-demo-frontendproxy 8080:8080

  The following services are available at these paths once the proxy is exposed:
  Webstore             http://localhost:8080/
  Grafana              http://localhost:8080/grafana/
  Feature Flags UI     http://localhost:8080/feature/
  Load Generator UI    http://localhost:8080/loadgen/
  Jaeger UI            http://localhost:8080/jaeger/ui/

- OpenTelemetry Collector OTLP/HTTP receiver (required for browser spans to be emitted):
  by running these commands:
     kubectl port-forward svc/otel-demo-otelcol 4318:4318

The OpenTelemetry Collector OTLP/HTTP receiver is bound to 0.0.0.0 in order to allow the "kubectl port-forward" command.
This may be susceptible to denial of service attacks.
See: CWE-1327 https://cwe.mitre.org/data/definitions/1327.html
```

## 步驟 02 - 使用範例應用

演示應用程序需要在 Kubernetes 集群外部公開的服務才能使用它們。您可以使用 kubectl port-forward 命令或通過使用可選部署的入口資源配置服務類型（即：LoadBalancer）來將服務暴露給本地系統。

### 使用 kubectl port-forward 公開服務

要公開 frontendproxy 服務，請使用以下命令（將 otel-demo 相應地替換為您的 Helm chart 版本名稱）：

```bash
kubectl port-forward svc/otel-demo-frontendproxy 7080:8080
```

為了正確收集來自瀏覽器的 span，您還需要公開 OpenTelemetry collector 的 OTLP/HTTP 端口（相應地用您的 Helm chart 版本名稱替換 otel-demo）：


```bash
kubectl port-forward svc/otel-demo-otelcol 4318:4318
```

通過 `frontendproxy` 和 `Collector` port-forward 設置，您可以訪問：

- Webstore: http://localhost:8080/
- Grafana: http://localhost:8080/grafana/
- Feature Flags UI: http://localhost:8080/feature/
- Load Generator UI: http://localhost:8080/loadgen/
- Jaeger UI: http://localhost:8080/jaeger/ui/


### 使用其它服務類型公開服務

### 部署範例應用程序

每個演示服務（即：frontendproxy）都提供了一種配置其 Kubernetes 服務類型的方法。默認情況下，這些將是 `ClusterIP`，但您可以使用每個服務的 `serviceType` 屬性進行更改。

要將 `frontendproxy` 服務配置為使用 `LoadBalancer` 服務類型，您可以在值文件中指定以下內容：

```yaml
components:
  frontendProxy:
    service:
      type: LoadBalancer
```

!!! info
    建議在安裝 Helm chart時使用 values 文件以指定其他配置選項。

Helm chart 不提供創建 ingress 資源。如果需要，則需要在安裝 Helm chart 後手動創建。

為了正確收集來自瀏覽器的 span，您還需要公開 OpenTelemetry collector 的 OTLP/HTTP 端口，以便用戶 Web 瀏覽器可以訪問。公開 OpenTelemetry collector 的位置也必須使用 `PUBLIC_OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` 環境變量傳遞到前端服務。您可以在 values 文件中使用以下內容來執行此操作：

```yaml
components:
  frontend:
    env:
      - name: PUBLIC_OTEL_EXPORTER_OTLP_TRACES_ENDPOINT
        value: http://otel-demo-collector.mydomain.com:4318/v1/traces
```

要使用自定義 my-values-file.yaml 值文件安裝 Helm chart，請使用：

```bash
helm install otel-demo open-telemetry/opentelemetry-demo --values my-values-file.yaml
```

暴露 frontendproxy 和 OpenTelemetry collector 後，您可以在 frontendproxy 的基本路徑中訪問演示 UI。可以在以下子路徑訪問其他演示組件：

- Webstore: `/` (base)
- Grafana: `/grafana`
- Feature Flags UI: `/feature`
- Load Generator UI: `/loadgen/` (must include trailing slash)
- Jaeger UI: `/jaeger/ui`

### 自帶 Tracing 後台

您可能希望將 Webstore 用作您已有的可觀察性後端的演示應用程序（例如，現有的 Jaeger、Zipkin 實例或您選擇的供應商之一)。

OpenTelemetry collector 的配置在 Helm chart 中公開。您所做的任何添加都將合併到默認配置中。您可以使用它來添加自己的 exporter，並將它們添加到所需的管道中:

```yaml
opentelemetry-collector:
  config:
    exporters:
      otlphttp/example:
        endpoint: <your-endpoint-url>

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [otlphttp/example]
```

其它的 Tracking 後端可能需要您添加額外的身份驗證參數，請查看他們的文檔。某些後端需要不同的導出器，您可以在 opentelemetry-collector-contrib/exporter 找到它們及其文檔。

要使用自定義 my-values-file.yaml 值文件安裝 Helm 圖表，請使用：