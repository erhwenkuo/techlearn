# Pipeline

**Kubeflow Pipelines 中管道的概念概述**

管道是對機器學習 (ML) 工作流的描述，包括工作流中的所有組件以及組件如何以圖形的形式相互關聯。

管道配置包括運行管道所需的輸入（參數）的定義以及每個組件的輸入和輸出。

當您運行管道時，系統會啟動一個或多個與您的工作流（管道）中的步驟（組件）相對應的 Kubernetes Pod。 Pod 啟動容器，容器依次啟動您的程序。

開發管道後，您可以使用 Kubeflow Pipelines UI 或 Kubeflow Pipelines SDK 上傳您的管道。