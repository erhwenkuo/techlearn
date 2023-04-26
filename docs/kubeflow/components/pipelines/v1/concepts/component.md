# Component

**Kubeflow Pipelines 中組件的概念概述**


管道組件是一組獨立的代碼，執行 ML 工作流（管道）中的一個步驟，例如數據預處理、數據轉換、模型訓練等。組件類似於函數，因為它具有名稱、參數、返回值和主體。

## Component code

每個組件的代碼包括以下內容：

- **Client code**: 與端點通信以提交作業的代碼。例如，與 Google Dataproc API 通信以提交 Spark 作業的代碼。
- **Runtime code**: 執行實際工作並通常在集群中運行的代碼。例如，將原始數據轉換為預處理數據的 Spark 代碼。

請注意客戶端代碼和運行時代碼的命名約定--對於名為 “mytask” 的任務：

- `mytask.py` 程序包含客戶端代碼。
- `mytask` 目錄包含所有運行時代碼。

## Component 定義

YAML 格式的組件規範描述了 Kubeflow Pipelines 系統的組件。組件定義包含以下部分：

- Metadata：名稱、描述等。
- Interface：輸入/輸出規範（名稱、類型、描述、默認值等）。
- Implementation：給定組件輸入的一組參數值時如何運行組件的規範。實現部分還描述瞭如何在組件完成運行後從組件獲取輸出值。

有關組件的完整定義，請參閱[組件規範](https://www.kubeflow.org/docs/components/pipelines/reference/component-spec/)。

## 容器化組件

您必須將組件打包為容器映像。組件代表容器內的特定程序或入口點。

管道中的每個組件獨立執行。這些組件不在同一個進程中運行，不能直接共享內存數據。您必須序列化（到字符串或文件）您在組件之間傳遞的所有數據片段，以便數據可以通過分佈式網絡傳輸。然後，您必須反序列化數據以供下游組件使用。