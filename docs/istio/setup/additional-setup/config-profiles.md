# 安裝配置文件

原文:[Installation Configuration Profiles](https://istio.io/latest/docs/setup/additional-setup/config-profiles/)

本文描述了在安裝 Istio 時所有能夠使用的內部配置文件。這些配置文件提供了對 Istio 控制面和 Istio 數據面 Sidecar 的設定。

您可以從Istio 內部配置文件的其中一個開始，然後根據您的特定需要一步一步自定義配置文件。當前提供以下幾種：

1. **default**：根據 IstioOperator API 的默認設置啟動組件。建議用於生產部署和 Multicluster Mesh 中的 Primary Cluster。您可以運行 `istioctl profile dump` 命令來查看默認設置。

2. **demo**：這一配置具有適度的資源需求，旨在展示 Istio 的功能。它適合運行 Bookinfo 應用程序和相關任務。這是通過快速開始指導安裝的配置。

    !!! warning
            此配置文件啟用了高級別的鏈路追踪和訪問日誌，因此不適合進行性能測試。

3. **minimal**：與 `default` 配置文件相同，但只安裝了控制面組件。它允許您使用 Separate Profile 配置控制面和數據面組件(例如 Gateway)。

4. **remote**：配置 Multicluster Mesh 的 Remote Cluster。

5. **empty**：不部署任何東西。可以作為自定義配置的基本配置文件。

6. **preview**：預覽文件包含的功能都是實驗性。這是為了探索 Istio 的新功能。不確保穩定性、安全性和性能（使用風險需自負）。

標註 ✔ 的組件安裝在每個配置文件中：

|核心组件              |default|demo   |minimal	    |remote|empty  |preview|
|---------------------|-------|-------|-----------|-------|-------|-------|
|istiod               |✔	  |✔      |✔   	      |       |       |✔      |
|istio-ingressgateway |✔      |✔      |   	      |       |       |✔      |
|istio-egressgateway  |       |✔      |           |       |       |       |



