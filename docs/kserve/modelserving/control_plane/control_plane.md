# Control Plane

**KServe Control Plane**: 負責協調 `InferenceService` 客制資源定義(CRD)。它為預測器、轉換器、解釋器建立 Knative 無伺服器部署，以根據傳入請求工作負載實現自動縮放，包括在未收到流量時縮小到零。啟用 raw deployment mode 時，控制平面會建立 Kubernetes `deployment`、`service`、`ingress`、`HPA`。

![](./assets/controlplane.png)

## Control Plane Components

- **KServe Controller**: 負責建立 service、ingress 資源、model server 和 model agent container，用於請求/回應日誌記錄、批次和模型拉取。

- **Ingress Gateway**: 用於路由外部或內部請求的網關。

Serverless Mode:

- **Knative Serving Controller**: 負責服務修訂管理、建立網路路由資源、具有佇列代理程式的無伺服器容器以公開流量指標並強制執行並發限制。

- **Knative Activator**: 返回縮放至零的 Pod 並轉發請求。

- **Knative Autoscaler(KPA)**: 監視流向應用程式的流量，並根據配置的指標向上或向下擴展副本。