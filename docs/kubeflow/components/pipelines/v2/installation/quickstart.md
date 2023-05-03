# Quickstart

本教程幫助您開始使用 KFP 部署和使用 KFP SDK 創建的管道。

在開始之前，您需要滿足以下先決條件：

- Kubernetes 集群：如果您沒有 Kubernetes 集群。
- `kubectl` 命令行工具：安裝和配置您的 kubectl 上下文以連接您的集群。
- 運行以下腳本安裝 KFP SDK：

    ```bash
    pip install kfp --pre
    ```

    !!! info
        注意：此命令安裝 KFP v2，它處於預發布階段並且還不穩定。 v2 文檔正在不斷開發中，某些指向 v2 文檔的鏈接可能不可用。

完成先決條件後，執行下列的步驟：

## Step 1: 將 KFP 獨立實例部署到您的集群中

此步驟演示如何將 KFP 獨立實例部署到現有 Kubernetes 集群中。

將 `PIPELINE_VERSION` 替換為所需版本的 KFP 後運行以下腳本（此處列出了發布）：

```bash
export PIPELINE_VERSION="2.0.0b15"

kubectl apply -k "github.com/kubeflow/pipelines/manifests/kustomize/cluster-scoped-resources?ref=$PIPELINE_VERSION"
kubectl wait --for condition=established --timeout=60s crd/applications.app.k8s.io
kubectl apply -k "github.com/kubeflow/pipelines/manifests/kustomize/env/dev?ref=$PIPELINE_VERSION"
```

部署 Kubernetes 後，[按照這些說明](https://www.kubeflow.org/docs/components/pipelines/v2/installation/)獲取您的 KFP 端點。

## Step 2: 使用 KFP SDK 創建並運行簡單的管道

此步驟顯示如何使用 KFP SDK 編寫管道並將其提交以供 KFP 執行。

下面的簡單管道將兩個整數相加，然後將另一個整數與結果相加以得出最終總和。

```python
from kfp import dsl
from kfp import client


@dsl.component
def addition_component(num1: int, num2: int) -> int:
    return num1 + num2


@dsl.pipeline(name='addition-pipeline')
def my_pipeline(a: int, b: int, c: int = 10):
    add_task_1 = addition_component(num1=a, num2=b)
    add_task_2 = addition_component(num1=add_task_1.output, num2=c)


endpoint = '<KFP_ENDPOINT>'
kfp_client = client.Client(host=endpoint)
run = kfp_client.create_run_from_pipeline_func(
    my_pipeline,
    arguments={
        'a': 1,
        'b': 2
    },
)
url = f'{endpoint}/#/runs/details/{run.run_id}'
print(url)
```

上面的代碼由以下幾部分組成：

1. 在第一部分中，以下幾行使用 `@dsl.component` 裝飾器創建了一個輕量級 Python 組件：

    ```python
    @dsl.component
    def addition_component(num1: int, num2: int) -> int:
        return num1 + num2
    ```

    `@dsl.component` 裝飾器將 Python 函數轉換為`組件`，可以在`管道`中使用。您需要在`參數`和`返回值`上指定類型註釋，因為這些會告知 KFP 執行程序如何序列化和反序列化在組件之間傳遞的數據。類型註釋和返回值還使 KFP 編譯器能夠對流水線任務之間傳遞的任何數據進行類型檢查。

2. 在第二部分中，以下幾行使用 `@dsl.pipeline` 裝飾器創建了一個管道：

    ```python
    @dsl.pipeline(name='addition-pipeline')
    def my_pipeline(a: int, b: int, c: int = 10):
    ...

    ```

    與組件裝飾器一樣，`@dsl.pipeline` 裝飾器將 Python 函數轉換為可由 KFP 後端執行的`管道`。管道可以有參數。這些參數還需要類型註釋。在此示例中，參數 `c` 的默認值為 10。

3. 在第三部分中，以下行將組件連接在一起以在管道函數主體內形成計算有向無環圖 (DAG)：

    ```python
    add_task_1 = addition_component(num1=a, num2=b)
    add_task_2 = addition_component(num1=add_task_1.output, num2=c)
    ```

    此示例通過將不同的參數傳遞給每個任務的組件函數，從名為 `addition_component` 的同一組件實例化兩個不同的加法任務，如下所示：

    - 第一個任務接受管道參數 `a` 和 `b` 作為輸入參數。
    - 第二個任務接受 `add_task_1.output` 作為第一個輸入參數，它是 `add_task_1` 的輸出。管道參數 `c` 是第二個輸入參數。

    您必須始終將組件參數作為關鍵字參數傳遞。

4. 在第四部分中，以下幾行使用在 **步驟#1** 中獲得的端點實例化一個 KFP 客戶端，並使用所需的管道參數將管道提交給 KFP 後端：

    ```python
    endpoint = '<KFP_ENDPOINT>'
    kfp_client = client.Client(host=endpoint)
    run = kfp_client.create_run_from_pipeline_func(
        my_pipeline,
        arguments={
            'a': 1,
            'b': 2
        },
    )
    url = f'{endpoint}/#/runs/details/{run.run_id}'
    print(url)
    ```

    在此示例中，將端點替換為您在 **步驟#1** 中獲得的 KFP 端點 URL。

    或者，您可以將管道編譯為 IR YAML，以便在其他時間使用：

    ```python
    from kfp import compiler

    compiler.Compiler().compile(pipeline_func=my_pipeline, package_path='pipeline.yaml')
    ```

## Step 3: 在 KFP 儀表板中查看管道

此步驟演示如何查看在 KFP 儀表板上運行的管道。為此，請轉到從第2步打印的 URL。

要查看每個任務的詳細信息，包括輸入和輸出，請單擊相應的任務節點。