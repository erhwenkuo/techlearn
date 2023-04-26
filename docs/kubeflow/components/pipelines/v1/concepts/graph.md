# Graph

**Kubeflow Pipelines 中 graph 的概念概述**

Graph 是管道運行時執行的 Kubeflow Pipelines UI 中的圖形表示。該圖顯示了管道運行已執行或正在執行的步驟，箭頭指示每個步驟表示的管道組件之間的父/子關係。運行一開始就可以查看 graph。Graph 中的每個節點都對應於管道中的一個步驟，並相應地進行了標記。

下面的屏幕截圖顯示了管道圖的示例：

![](./assets/pipelines-xgboost-graph.png)

每個節點的右上角都有一個圖標，指示其狀態：正在運行、成功、失敗或已跳過。（當其父節點包含條件子句時，可以跳過該節點。）