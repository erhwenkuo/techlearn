# 數據型別 (Data Types)

**組件和管道 I/O 類型**

KFP 組件和管道可以接受輸入並創建輸出。為此，他們必須通過函數簽名和註釋來聲明類型化接口。

KFP 中有兩組類型：parameters 和 artifacts。Parameters 對於在組件之間傳遞少量數據很有用。Artifacts 類型是 KFP 為 ML 工件輸出（例如數據集、模型、指標等）提供一流支持的機制。

到目前為止，[Hello World 管道](https://www.kubeflow.org/docs/components/pipelines/v2/hello-world)和[組件](https://www.kubeflow.org/docs/components/pipelines/v2/components)中的示例已經演示瞭如何使用輸入和輸出參數。

KFP 自動跟踪參數和工件在組件之間傳遞的方式，並將此數據傳遞歷史記錄存儲在[ML 元數據](https://github.com/google/ml-metadata)中。這支持開箱即用的 ML 工件沿襲跟踪和易於重現的管道執行。此外，KFP 的強類型組件在管道中的任務之間提供數據契約。

