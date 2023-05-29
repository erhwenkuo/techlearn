# 控制平面 (Control Plane)

**KServe Control Plane**: 負責協調 `InferenceService` 自定義資源。它為 `predictor`、`transformer`、`explainer` 創建 Knative serverless 部署，並根據傳入的請求工作負載啟用自動縮放，包括在未收到流量時縮小到零。當啟用 `raw deployment` 模式時，控制平面創建 Kubernetes deployment、service、ingress、HPA。

![](./assets/controlplane.png)

## 控制平面組件

* **KServe Controller** : 負責創建 service、ingress 資源、model server 容器和模型代理容器，用於請求/響應日誌記錄、批次處理和模型拉取。

* **Ingress Gateway** : 用於路由外部或內部請求的網關。

### 在 Serverless 模式下：

- **Knative Serving Controller** ：負責服務修訂管理、創建網絡路由資源、帶有隊列代理的無服務器容器以公開流量指標並強制執行併發限制。

- **Knative Activator**：將縮減為零的 pod 恢復並轉發請求。

- **Knative Autoscaler (KPA)** ：觀察流向應用程序的流量，並根據配置的指標向上或向下擴展副本。