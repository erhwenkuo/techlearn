# ML Metadata

**Kubeflow 管道中元數據的概念概述**

注意：Kubeflow Pipelines 已經從使用 [kubeflow/metadata](https://github.com/kubeflow/metadata) 轉變為使用 [google/ml-metadata](https://github.com/google/ml-metadata) 作為元數據依賴。

Kubeflow Pipelines 後端存儲在元數據存儲中運行的管道的運行時信息。運行時信息包括任務的狀態、工件的可用性、與執行或工件關聯的自定義屬性等。在 ML 元數據入門中了解更多信息。

如果一個工件被不同運行中的多個執行使用，您可以查看跨流水線運行的工件和執行之間的連接。這種連接可視化稱為 Lineage Graph。

