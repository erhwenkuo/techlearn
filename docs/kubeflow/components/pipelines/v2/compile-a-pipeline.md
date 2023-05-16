# 編譯 Pipeline

**將管道和組件編譯為 YAML**

要提交執行管道，您必須使用 KFP SDK 編譯器將其編譯為 YAML：

```python
from kfp import dsl
from kfp import compiler

@dsl.component
def comp(message: str) -> str:
    print(message)
    return message

@dsl.pipeline
def my_pipeline(message: str) -> str:
    """My ML pipeline."""
    return comp(message=message).output

compiler.Compiler().compile(my_pipeline, package_path='pipeline.yaml')
```

在此示例中，編譯器創建一個名為 `pipeline.yaml` 的文件，其中包含管道的密封型式的表示。編譯的結果稱為 `intermediate representation (IR) YAML`。您可以在 [two_step_pipeline.yaml](https://github.com/kubeflow/pipelines/blob/984d8a039d2ff105ca6b21ab26be057b9552b51d/sdk/python/test_data/pipelines/two_step_pipeline.yaml) 上查看一個 IR YAML 示例。該文件的內容是序列化的 `PipelineSpec` protocol buffer message，其主要目的並非 human-readable。

您可以在已編譯 YAML 頂部的註釋中找到有關管道的人類可讀信息：

```yaml
# PIPELINE DEFINITION
# Name: my-pipeline
# Description: My ML pipeline.
# Inputs:
#    message: str
# Outputs:
#    Output: str
...
```

您還可以將組件（而不是管道）編譯為 IR YAML：

```python
@dsl.component
def comp(message: str) -> str:
    print(message)
    return message

compiler.Compiler().compile(comp, package_path='component.yaml')
```

## 編譯器參數

`Compiler.compile` 方法接受以下參數：

|Name	|Type	|Rquired? |Description|
|-----|-----|---------|-----------|
|`pipeline_func`	|`function`	|Required |使用 `@dsl.pipeline` 構造的管道函數或使用 `@dsl.component` 裝飾器構造的組件。|
|`package_path`	|`string`	|Required |輸出 YAML 文件路徑。例如，`~/my_pipeline.yaml` 或 `~/my_component.yaml`。|
|`pipeline_name`	|`string`	|Optional |如果指定，則在已編譯的 IR YAML 輸出的 `pipelineInfo.name` 欄位中設置管道模板的名稱。覆蓋由 `@dsl.pipeline decorator` 中的 `name` 參數指定的管道或組件的名稱。|
|`pipeline_parameters`	|`Dict[str, Any]`	|Optional |參數名稱到參數值的映射。這使您可以為管道或組件參數提供默認值。您可以在管道提交期間覆蓋這些默認值。|
|`type_check`	|`bool`	|Optional |指示在編譯期間是否啟用靜態類型檢查。|

## Type 檢查

默認情況下，DSL 編譯器靜態類型檢查您的管道，以確保在彼此之間傳遞數據的組件之間的類型一致性。靜態類型檢查有助於識別組件 I/O 不一致，而無需運行管道，從而縮短開發迭代。

具體來說，類型檢查器檢查組件輸入期望的數據類型與提供的數據類型之間的類型是否相等。有關 KFP 數據類型的更多信息，請參閱 [Data Types](./data-types/index.md)。

例如，對於參數，列表輸入只能傳遞給帶有 `typing.List` 註釋的參數。同樣，浮點數只能傳遞給帶有浮點註釋的參數。

輸入數據類型和註釋也必須匹配工件，但有一個例外：工件類型與所有其他工件類型兼容。從這個意義上說，工件類型既是默認工件類型，也是工件“任何”類型。

如下一節所述，您可以禁用類型檢查。

## IR YAML

IR YAML 是已編譯管道或組件的中間表示。它是 `PipelineSpec` protocol buffer 消息類型的一個實例，它是一種與平台無關的管道表示協議。它被認為是一種中間表示，因為 KFP 後端將 `PipelineSpec` 編譯為 `Argo Workflow YAML` 作為執行的最終管道定義。

與 v1 組件 YAML 不同，IR YAML 並不是設計用來讓開發者來直接編寫。

雖然 IR YAML 並非旨在易於人類閱讀，但如果您對它的內容有所了解，您仍然可以檢查它：

|Section	|Description	|Example|
|---------|-------------|-------|
|`components`	|此部分是管道中使用的所有組件的名稱到 `[ComponentSpec](https://github.com/kubeflow/pipelines/blob/41b69fd90da812005965f2209b64fd1278f1cdc9/api/v2alpha1/pipeline_spec.proto#L85-L96)` 的映射。 `ComponentSpec` 定義了組件的接口，包括輸入和輸出。<br/>對於原始組件，`ComponentSpec` 包含對包含組件實現的執行器的引用。<br/>對於用作組件的管道，`ComponentSpec` 包含一個 `DagSpec` 實例，其中包括對底層原始組件的引用。|[View on Github](https://github.com/kubeflow/pipelines/blob/984d8a039d2ff105ca6b21ab26be057b9552b51d/sdk/python/test_data/pipelines/two_step_pipeline.yaml#L1-L21)|
|`deployment_spec`	|此部分包含執行程序名稱到 `[ExecutorSpec](https://github.com/kubeflow/pipelines/blob/41b69fd90da812005965f2209b64fd1278f1cdc9/api/v2alpha1/pipeline_spec.proto#L788-L803)` 的映射。 `ExecutorSpec` 包含原始組件的實現。|[View on Github](https://github.com/kubeflow/pipelines/blob/984d8a039d2ff105ca6b21ab26be057b9552b51d/sdk/python/test_data/pipelines/two_step_pipeline.yaml#L23-L49)|
|`root`	|本節定義了最外層管道定義的步驟，也稱為管道根定義。根定義是您提交 IR YAML 時執行的工作流。它是 `ComponentSpec` 的一個實例。|[View on Github](https://github.com/kubeflow/pipelines/blob/984d8a039d2ff105ca6b21ab26be057b9552b51d/sdk/python/test_data/pipelines/two_step_pipeline.yaml#L52-L85)|
|`pipeline_info`	|此部分包含管道元數據，包括 pipelineInfo.name 字段。此字段包含您的管道模板的名稱。上傳管道時，將根據此模板名稱創建管道上下文名稱。管道上下文讓後端和儀表板使用管道模板關聯來自管道運行的工件和執行。您可以使用管道上下文通過比較基於同一訓練管道的多個管道運行的指標和工件來確定最佳模型。|[View on Github](https://github.com/kubeflow/pipelines/blob/984d8a039d2ff105ca6b21ab26be057b9552b51d/sdk/python/test_data/pipelines/two_step_pipeline.yaml#L50-L51)|
|`sdk_version`	|本節記錄用於編譯管道的 KFP SDK 的版本。|[View on Github](https://github.com/kubeflow/pipelines/blob/984d8a039d2ff105ca6b21ab26be057b9552b51d/sdk/python/test_data/pipelines/two_step_pipeline.yaml#L50-L51)|
|`schema_version`	|此部分記錄用於 IR YAML 的 `PipelineSpec` 架構的版本。|[View on Github](https://github.com/kubeflow/pipelines/blob/984d8a039d2ff105ca6b21ab26be057b9552b51d/sdk/python/test_data/pipelines/two_step_pipeline.yaml#L86)|
|`default_pipeline_root`	|此部分記錄遠程存儲根路徑，例如 MinIO URI 或 Google Cloud Storage URI，其中寫入管道輸出。|[View on Github](https://github.com/kubeflow/pipelines/blob/984d8a039d2ff105ca6b21ab26be057b9552b51d/sdk/python/test_data/pipelines/two_step_pipeline.yaml#L22)|

