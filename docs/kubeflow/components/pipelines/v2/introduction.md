# 介紹

Kubeflow Pipelines (KFP) 是一個平台，用於使用 Docker 容器構建和部署可移植且可擴展的機器學習 (ML) 工作流。

借助 KFP，您可以使用 KFP Python SDK 編寫[組件](https://www.kubeflow.org/docs/components/pipelines/v2/components)和[管道](https://www.kubeflow.org/docs/components/pipelines/v2/pipelines)，將管道編譯為[中間表示 YAML](https://www.kubeflow.org/docs/components/pipelines/v2/compile-a-pipeline#ir-yaml)，並提交管道以在符合 KFP 的後端（例如[開源 KFP 後端](https://www.kubeflow.org/docs/components/pipelines/v2/installation)或 [Google Cloud Vertex AI 管道](https://cloud.google.com/vertex-ai/docs/pipelines/introduction)）上運行。

[開源 KFP 後端](https://www.kubeflow.org/docs/components/pipelines/v2/installation)可作為 Kubeflow 的核心組件或作為獨立安裝使用。按照[安裝說明](https://www.kubeflow.org/docs/components/pipelines/v2/installation)和 [Hello World Pipeline 示例](./hello-world.md)快速開始使用 KFP。

## 為什麼選擇 Kubeflow 管道？

KFP 使數據科學家和機器學習工程師能夠：

- 在 Python 中原生編寫端到端機器學習工作流程
- 創建完全自定義的 ML 組件或利用現有組件的生態系統
- 輕鬆管理、跟踪和可視化管道定義、運行、實驗和 ML 工件
- 通過並行任務執行和通過緩存消除冗餘執行來有效地使用計算資源
- 通過平台中立的 [IR YAML 管道定義](https://www.kubeflow.org/docs/components/pipelines/v2/compile-a-pipeline#ir-yaml)維護跨平台管道的可移植性

## 什麼是管道？

[管道](https://www.kubeflow.org/docs/components/pipelines/v2/pipelines)是工作流的定義，它將一個或多個[組件](https://www.kubeflow.org/docs/components/pipelines/v2/components)組合在一起以形成計算有向無環圖 (DAG)。在運行時，每個組件執行對應於一個容器執行，這可能會創建 ML 工件。管道也可能具有[控制流](https://www.kubeflow.org/docs/components/pipelines/v2/pipelines/control-flow)的特徵。