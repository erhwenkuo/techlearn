# Hello World Pipeline

創建您的第一個 Kubeflow 的管道。

要開始使用教程，請先安裝 `kfp v2`：

```bash
pip install kfp==2.0.0b15
```

這是一個打印問候語的簡單管道：

```python
from kfp import dsl

@dsl.component
def say_hello(name: str) -> str:
    hello_text = f'Hello, {name}!'
    print(hello_text)
    return hello_text

@dsl.pipeline
def hello_pipeline(recipient: str) -> str:
    hello_task = say_hello(name=recipient)
    return hello_task.output
```

您可以使用 KFP SDK DSL [編譯器](https://kubeflow-pipelines.readthedocs.io/en/master/source/compiler.html#kfp.compiler.Compiler)將管道[編譯](https://www.kubeflow.org/docs/components/pipelines/v2/compile-a-pipeline/)為 YAML：

```python
from kfp import compiler

compiler.Compiler().compile(hello_pipeline, 'pipeline.yaml')
```

[dsl.component](https://kubeflow-pipelines.readthedocs.io/en/master/source/dsl.html#kfp.dsl.component) 和 [dsl.pipeline](https://kubeflow-pipelines.readthedocs.io/en/master/source/dsl.html#kfp.dsl.pipeline) 裝飾器分別將帶類型註釋的 Python 函數轉換為**組件**和**管道**。 KFP SDK 編譯器將域特定語言 (DSL) 對象編譯為獨立的[管道 YAML 文件](https://www.kubeflow.org/docs/components/pipelines/v2/compile-a-pipeline#ir-yaml)。

您可以將 YAML 文件提交到符合 KFP 標準的後端以供執行。如果您已經部署了 KFP 開源後端實例並獲得了部署的端點，則可以使用 [KFP SDK 客戶端](https://kubeflow-pipelines.readthedocs.io/en/master/source/client.html#kfp.client.Client)提交管道以供執行。以下使用參數 `recipient='World'` 提交要執行的管道：

```python
from kfp.client import Client

client = Client(host='<MY-KFP-ENDPOINT>')
run = client.create_run_from_pipeline_package(
    'pipeline.yaml',
    arguments={
        'recipient': 'World',
    },
)
```

客戶端將打印一個鏈接以在 UI 中查看管道執行圖和日誌。在這種情況下，管道有一個任務打印並返回 “Hello, World!”。

在接下來的幾節中，您將了解有關創作管道的核心概念以及如何創建更具表現力、更有用的管道的更多信息。