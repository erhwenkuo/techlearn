# 從 KFP SDK v1 遷移

如果您有現有的 KFP 管道，無論是編譯到 [Argo Workflow](https://argoproj.github.io/argo-workflows/)（使用 SDK v1 主命名空間）還是編譯到 [IR YAML](https://www.kubeflow.org/docs/components/pipelines/v2/compile-a-pipeline/#ir-yaml)（使用 SDK v1 v2-命名空間），您都可以在新的 [KFP v2 後端](https://www.kubeflow.org/docs/components/pipelines/v2/installation/)上運行這些管道而無需任何更改。

如果您希望創作新的管道，可以通過一些推薦和必需的步驟將您的管道創作代碼遷移到 KFP SDK v2。

## 術語

- **SDK v1**：KFP SDK 的 `1.x.x` 版本。 `1.8` 是將要發布的 v1 KFP SDK 的最高的 [minor version](https://semver.org/#:~:text=MINOR%20version%20when%20you%20add%20functionality%20in%20a%20backwards%20compatible%20manner)。

- **SDK v1 v2-namespace**：指的是 v1 KFP SDK 的 `v2 模塊`（即 from kfp.v2 import *），它允許從 v1 SDK 訪問 v2 創作語法和編譯為 [IR YAML](https://www.kubeflow.org/docs/components/pipelines/v2/compile-a-pipeline/#ir-yaml)。除非明確說明，否則您應該假設對 v1 SDK 的引用不引用 v2-namespace。在發布 [v2 KFP OSS 後端](https://www.kubeflow.org/docs/components/pipelines/v2/installation/)之前，這些管道只能在 [Google Cloud Vertex AI Pipelines](https://cloud.google.com/vertex-ai/docs/pipelines/introduction) 上執行。

- **SDK v2**：KFP SDK 的 `2.x.x` 版本。使用 `v2` 創作語法並編譯為 IR YAML。

有兩種常見的遷移路徑：

1. 如果您現有的 `kfp==1.x.x` 代碼從 v2-namespace 導入（即，`from kfp.v2 import *`），請按照 [SDK v1 v2-namespace 到 SDK v2 遷移說明](https://www.kubeflow.org/docs/components/pipelines/v2/migration/#sdk-v1-v2-namespace-to-sdk-v2)進行操作。此遷移路徑僅影響在 [Google Cloud Vertex AI Pipelines](https://cloud.google.com/vertex-ai/docs/pipelines/introduction) 上運行管道的 v1 SDK 用戶。

2. 如果您現有的 `kfp==1.x.x` 代碼從主命名空間導入（即，從 `kfp import *`），請按照 [SDK v1 到 SDK v2 遷移說明](https://www.kubeflow.org/docs/components/pipelines/v2/migration/#sdk-v1-to-sdk-v2)進行操作。

## SDK v1 v2-namespace 到 SDK v2

除了少數例外，KFP SDK v2 向後兼容使用 KFP SDK v1 v2-namespace 的用戶代碼。

### Non-breaking changes

本部分記錄了 SDK v2 中相對於 SDK v1 v2-namespace 的非破壞性更改。我們建議您將代碼遷移到“新用法”，即使“以前的用法”仍然可以使用警告。

#### Import namespace

KFP SDK v1 v2-namespace 導入（來自 `kfp.v2 import *`）應轉換為來自主命名空間的導入（`from kfp import *`）。

**更改**：從任何 KFP SDK v1 v2-namespace 導入中刪除 `.v2` 模塊。

=== "Previous usage"

    ```python
    from kfp.v2 import dsl
    from kfp.v2 import compiler

    @dsl.pipeline(name='my-pipeline')
    def pipeline():
    ...

    compiler.Compiler().compile(...)
    ```

=== "New usage"

    ```python
    from kfp import dsl
    from kfp import compiler

    @dsl.pipeline(name='my-pipeline')
    def pipeline():
    ...

    compiler.Compiler().compile(...)  
    ```

#### output_component_file 參數

在 KFP SDK v2 中，組件可以像管道一樣[編譯](https://www.kubeflow.org/docs/components/pipelines/v2/compile-a-pipeline)到 [IR YAML](https://www.kubeflow.org/docs/components/pipelines/v2/compile-a-pipeline/#ir-yaml) 並從中加載。

KFP SDK v1 v2-namespace 支持通過 [@dsl.component](https://kubeflow-pipelines.readthedocs.io/en/master/source/dsl.html#kfp.dsl.component) 裝飾器的 `output_component_file` 參數編譯組件。這在 KFP SDK v2 中已棄用。如果您選擇仍然使用此參數，您的管道將被編譯為 IR YAML 而不是 v1 組件 YAML。

**更改**：刪除對 `output_component_file` 的使用。替換為對 `Compiler().compile()` 的調用。

=== "Previous usage"

    ```python
    from kfp.v2.dsl import component

    @component(output_component_file='my_component.yaml')
    def my_component(input: str):
    ...
    ```

=== "New usage"

    ```python
    from kfp.dsl import component
    from kfp import compiler

    @component()
    def my_component(input: str):
    ...

    compiler.Compiler().compile(my_component, 'my_component.yaml')    
    ```

#### Pipeline 打包的文件擴展名

KFP 編譯器將根據提供給編譯器的擴展名（`.yaml` 或 `.json`）編譯您的管道。

在 KFP SDK v2 中，YAML 是首選的序列化格式。

**更改**：將使用 `.json` 擴展名的 `package_path` 參數轉換為使用 `.yaml` 擴展名。

=== "Previous usage"

    ```python
    from kfp.v2 import compiler
    # .json extension, deprecated format
    compiler.Compiler().compile(pipeline, package_path='my_pipeline.json')    
    ```

=== "New usage"

    ```python
    from kfp import compiler
    # .yaml extension, preferred format
    compiler.Compiler().compile(pipeline, package_path='my_pipeline.yaml')
    ```

### Breaking changes

相對於 SDK v1 v2-namespace，SDK v2 中只有一些細微的破壞性變化。

#### 放棄對 Python 3.6 的支持

KFP SDK v1 支持 Python 3.6。 KFP SDK v2 支持 Python >=3.7.0,<3.12.0。

#### CLI output change

v2 KFP CLI 更加一致、可讀和可解析。解析 v1 CLI 輸出的代碼可能無法解析 v2 CLI 輸出。

#### `.after` referencing upstream task in a `dsl.ParallelFor` loop

以下管道無法在 KFP SDK v2 中編譯：

```python hl_lines="4"
with dsl.ParallelFor(...):
    t1 = comp()

t2 = comp().after(t1)
```

此用法主要由實現自定義 `dsl.ParallelFor` 的 KFP SDK v1 用戶使用。 KFP SDK v2 本機支持使用 `dsl.Collected` 從 `dsl.ParallelFor` fan-in。有關說明，請參閱[控制流](https://www.kubeflow.org/docs/components/pipelines/v2/pipelines/control-flow/#parallel-looping-dslparallelfor)用戶文檔。

## SDK v1 到 SDK v2

KFP SDK v2 通常不向後兼容使用 KFP SDK v1 主命名空間的用戶代碼。本節介紹升級到 KFP SDK v2 的一些重要中斷更改和遷移步驟。

我們指出每個重大更改是否會影響 [KFP OSS 後端](https://www.kubeflow.org/docs/components/pipelines/v1/)用戶或 [Google Cloud Vertex AI Pipelines](https://cloud.google.com/vertex-ai/docs/pipelines/introduction) 用戶。

### Breaking changes

#### 移除 `create_component_from_func` 與 `func_to_container_op`

**影響**：KFP OSS 用戶和 Vertex AI Pipelines 用戶

`create_component_from_func` 和 `func_to_container_op` 都在 KFP SDK v1 中用於創建輕量級的基於 Python 函數的組件。

KFP SDK v2 中刪除了這兩個函數。

**更改**：使用 `@dsl.component` 裝飾器，如輕量級 Python 組件和容器化 Python 組件中所述。

=== "Previous usage"

    ```python
    from kfp.components import create_component_from_func
    from kfp.components import func_to_container_op

    @create_component_from_func
    def component1(...):
        ...

    def component2(...):
        ...

    component2 = create_component_from_func(component2)

    @func_to_container_op
    def component3(...):
        ...

    @dsl.pipeline(name='my-pipeline')
    def pipeline():
        component1(...)
        component2(...)
        component3(...) 
    ```

=== "New usage"

    ```python
    from kfp import dsl

    @dsl.component
    def component1(...):
        ...

    @dsl.component
    def component2(...):
        ...

    @dsl.component
    def component3(...):
        ...

    @dsl.pipeline(name='my-pipeline')
    def pipeline():
        component1(...)
        component2(...)
        component3(...)
    ```

#### 移除 ContainerOp

**影響**：KFP OSS 用戶

自 2020 年年中以來，`ContainerOp` 已被棄用。 `ContainerOp` 實例不包含輸入和輸出的描述，因此無法編譯為 [IR YAML](https://www.kubeflow.org/docs/components/pipelines/v2/compile-a-pipeline/#ir-yaml)。

**更改**：使用 `@dsl.container_component` 裝飾器，如容器組件中所述。

=== "Previous usage"

    ```python
    from kfp import dsl

    # v1 ContainerOp will not be supported.
    component_op = dsl.ContainerOp(...)

    # v1 ContainerOp from class will not be supported.
    class FlipCoinOp(dsl.ContainerOp):
    ```

=== "New usage"

    ```python
    from kfp import dsl

    @dsl.container_component
    def flip_coin(rand: int, result: dsl.OutputPath(str)):
    return ContainerSpec(
        image='gcr.io/flip-image'
        command=['flip'],
        arguments=['--seed', rand, '--result-file', result])
    ```

#### 對 VolumeOp 與 ResourceOp 支持

**影響**：KFP OSS 用戶

`VolumeOp` 和 `ResourceOp` 在管道定義中公開對 Kubernetes 資源的直接訪問。在非 Kubernetes 平台上不支持這些功能。

這些特性將通過特定於平台的功能得到支持，其中包括特定於 Kubernetes 的功能。此功能正在持續開發中，將在 KFP v2 GA 版本中得到支持。

#### 對 v1 component YAML 支持

**影響**：KFP OSS users 與 Vertex AI Pipelines users

KFP v1 支持通過 v1 組件 YAML 格式（[示例](https://github.com/kubeflow/pipelines/blob/01c87f8a032e70a6ca92cdbefa974a7da387f204/sdk/python/test_data/v1_component_yaml/add_component.yaml)）直接在 YAML 中編寫組件。這種創作風格使組件作者能夠直接設置其組件的圖像、命令和參數。

在 KFP v2 中，組件和管道都被編譯為相同的 [IR YAML](https://www.kubeflow.org/docs/components/pipelines/v2/compile-a-pipeline/#ir-yaml) 格式，這與 v1 組件 YAML 格式不同。

KFP v2 將繼續支持使用 `components.load_component_from_file` 函數和類似函數加載現有的 v1 組件 YAML，以實現向後兼容性。

**更改**：要通過自定義鏡像、命令和 args 編寫組件，請使用 `@dsl.container_component` 裝飾器，如容器組件中所述。請注意，與編寫 v1 組件 YAML 不同，容器組件不支持在組件本身上設置環境變量。應使用 `.set_env_variable` 任務配置方法在管道定義中從組件實例化的任務上[設置環境變量](https://www.kubeflow.org/docs/components/pipelines/v2/pipelines/pipeline-basics/#task-configurations)。

#### 對 v1 lightweight component types InputTextFile, InputBinaryFile, OutputTextFile 與 OutputBinaryFile 支持

**影響**：KFP OSS users 與 Vertex AI Pipelines users

這些類型確保文件在使用 KFP SDK v1 編寫的組件中以文本模式或二進制模式寫入。

KFP SDK v2 不支持使用這些類型進行代碼撰寫，因為用戶可以輕鬆地自己進行撰寫。

**更改**：組件開發者應該使用 KFP 的 artifact 和 parameter 類型進行輸入和輸出。

#### 對 AIPlatformClient 支持

**影響**：Vertex AI Pipelines users

KFP SDK v1 包含一個 `AIPlatformClient`，用於將管道提交到 Vertex AI Pipelines。

KFP SDK v2 不包含此客戶端。

**更改**：使用官方 Python Vertex SDK 的 PipelineJob 類。


=== "Previous usage"

    ```python
    from kfp.v2.google.client import AIPlatformClient

    api_client = AIPlatformClient(
        project_id=PROJECT_ID,
        region=REGION,
    )

    response = api_client.create_run_from_job_spec(
        job_spec_path=PACKAGE_PATH, pipeline_root=PIPELINE_ROOT,
    )
    ```

=== "New usage"

    ```python
    # pip install google-cloud-aiplatform
    from google.cloud import aiplatform

    aiplatform.init(
        project=PROJECT_ID,
        location=REGION,
    )

    job = aiplatform.PipelineJob(
        display_name=DISPLAY_NAME,
        template_path=PACKAGE_PATH,
        pipeline_root=PIPELINE_ROOT,
    )

    job.submit()
    ```

#### 對 run_as_aiplatform_custom_job 支持

**影響**：Vertex AI Pipelines users

KFP v1 的 `run_as_aiplatform_custom_job` 是一項實驗性功能，允許將任何組件轉換為 Vertex AI CustomJob。

KFP v2 不包含此功能。

**更改**：使用[谷歌云管道組件](https://cloud.google.com/vertex-ai/docs/pipelines/components-introduction)的 `create_custom_training_job_from_component` 函數。


=== "Previous usage"

    ```python
    from kfp import components
    from kfp.v2 import dsl
    from kfp.v2.google.experimental import run_as_aiplatform_custom_job

    training_op = components.load_component_from_url(...)

    @dsl.pipeline(name='my-pipeline')
    def pipeline():
    training_task = training_op(...)
    run_as_aiplatform_custom_job(
        training_task, ...)
    ```

=== "New usage"

    ```python
    # pip install google-cloud-pipeline-components
    from kfp import components
    from kfp import dsl
    from google_cloud_pipeline_components.v1.custom_job import utils

    training_op = components.load_component_from_url(...)

    @dsl.pipeline(name='my-pipeline')
    def pipeline():
        utils.create_custom_training_job_from_component(training_op, ...)
    ```


