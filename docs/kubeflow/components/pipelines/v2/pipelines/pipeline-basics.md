# Pipeline 基礎

**將組件組合成管道**

雖然組件 (Component) 具有三種主要的創作方法，但管道 (pipeline) 只有一種創作方法：它們是用一個用 `@dsl.pipeline` 裝飾器裝飾的管道函數定義的。

舉下列一個管道 `pythagorean` 作範例，它通過簡單的算術組件將 Pythagorean 定理實現為管道:

```python
from kfp import dsl

@dsl.component
def square(x: float) -> float:
    return x ** 2

@dsl.component
def add(x: float, y: float) -> float:
    return x + y

@dsl.component
def square_root(x: float) -> float:
    return x ** .5

@dsl.pipeline
def pythagorean(a: float, b: float) -> float:
    a_sq_task = square(x=a)
    b_sq_task = square(x=b)
    sum_task = add(x=a_sq_task.output, y=b_sq_task.output)
    return square_root(x=sum_task.output).output
```

雖然用 `@dsl.pipeline` 裝飾器裝飾的 KFP 管道看起來像一個普通的 Python 函數，但它實際上是管道拓撲和控制流語義的表達，使用 KFP 領域特定語言 (DSL) 構建。

管道定義有四個部分：

1. Pipeline 裝飾器
2. 函數簽名中聲明的輸入和輸出
3. 數據傳遞和任務依賴
4. Task 配置
5. Pipeline [控制流](https://www.kubeflow.org/docs/components/pipelines/v2/pipelines/control-flow)

本節涵蓋前四個部分。下一節將介紹控制流。


## Pipeline 裝飾器

KFP 管道用 `@dsl.pipeline` 裝飾器裝飾的函數中定義。裝飾器採用三個可選參數：

- `name` 是您的管道的名稱。如果未提供，則名稱默認為管道函數名稱。
- `description` 是管道的描述。
- `pipeline_root` 是遠程存儲目標的根路徑，管道中的任務將在其中創建輸出。 `pipeline_root` 也可以由管道提交客戶端設置或覆蓋。
- `display_name` 是您的管道的人類可讀名稱。

您可以修改 pythagorean 的定義以使用這些參數：

```python
@dsl.pipeline(name='pythagorean-theorem-pipeline',
              description='Solve for the length of a hypotenuse of a triangle with sides length `a` and `b`.',
              pipeline_root='gs://my-pipelines-bucket',
              display_name='Pythagorean pipeline.')
def pythagorean(a: float, b: float) -> float:
    ...
```

另請參閱附加功能：組件文檔字符串格式，了解有關如何通過 `docstrings` 提供管道元數據的信息。

## 函數簽名中聲明的輸入和輸出

與組件一樣，管道`輸入`和`輸出`由管道函數簽名中的`參數`和`註釋`定義。

在前面的示例中，`pythagorean` 接受輸入 `a` 和 `b`，每個類型都是 `float`，並創建一個 `float` 輸出。

`管道輸入`通過`函數輸入參數/註釋`聲明，`管道輸出`通過`函數輸出註釋`聲明。{==`管道輸出`永遠不會通過`管道函數輸入參數`聲明，這與使用輸出工件的組件或使用 `dsl.OutputPath` 的容器組件不同==}。

有關如何聲明管道函數輸入和輸出的更多信息，請參閱[Data Types](https://www.kubeflow.org/docs/components/pipelines/v2/data-types)。

## 數據傳遞和任務依賴

當您調用管道定義中的組件時，它會構造一個 `PipelineTask` 實例。您可以使用 `PipelineTask` 的 `.output` 和 `.outputs` 屬性在任務之間傳遞數據。

對於具有由單個返回註釋指示的單個未命名輸出的任務，使用 `PipelineTask.output` 訪問輸出。組件 `square`、`add` 和 `square_root` 就是這種情況，它們每個都有一個未命名的輸出。

對於具有多個輸出或命名輸出的任務，使用 `PipelineTask.outputs['<output-key>']` 訪問輸出。[Data Types: Parameters](https://www.kubeflow.org/docs/components/pipelines/v2/data-types/parameters#multiple-output-parameters) 中更詳細地描述了使用命名輸出參數。

在沒有數據交換的情況下，任務將併行運行以實現高效的管道執行。這是任務之間沒有交換數據的 `a_sq_task` 和 `b_sq_task` 的情況。

當任務之間有交換數據的時候，Kubeflow 將在這些任務之間建立執行順序。這是為了確保上游任務在下游任務嘗試使用這些輸出之前創建它們的輸出。例如，在 `pythagorean` 中，後端會先執行 `a_sq_task` 和`b_sq_task`，然後再執行 `sum_task`。同樣，它會在執行從 `square_root` 組件創建的最終任務之前執行 `sum_task`。

在某些情況下，您可能希望在沒有數據交換的情況下建立執行順序。在這些情況下，您可以在另一個任務上調用一個任務的 `.after()` 方法。例如，雖然 `a_sq_task` 和 `b_sq_task` 不交換數據，但我們可以指定 `a_sq_task` 在 `b_sq_task` 之前運行：

```python
@dsl.pipeline
def pythagorean(a: float, b: float) -> float:
    a_sq_task = square(x=a)
    b_sq_task = square(x=b)
    b_sq_task.after(a_sq_task)
    ...
```

## 特殊輸入類型

您可以將一些特殊的輸入值傳遞給管道定義中的組件，以使組件能夠訪問有關其自身的一些元數據。這些值可以傳遞給類型為 `str` 的輸入參數。

例如，以下 `print_op` 組件使用 `dsl.PIPELINE_JOB_NAME_PLACEHOLDER` 在組件運行時打印管道作業名稱：

```python
from kfp import dsl

@dsl.pipeline
def my_pipeline():
    print_op(text=dsl.PIPELINE_JOB_NAME_PLACEHOLDER)
```

此樣式中可能會使用幾個特殊值，包括：

- `dsl.PIPELINE_JOB_NAME_PLACEHOLDER`
- `dsl.PIPELINE_JOB_RESOURCE_NAME_PLACEHOLDER`
- `dsl.PIPELINE_JOB_ID_PLACEHOLDER`
- `dsl.PIPELINE_TASK_NAME_PLACEHOLDER`
- `dsl.PIPELINE_TASK_ID_PLACEHOLDER`
- `dsl.PIPELINE_JOB_CREATE_TIME_UTC_PLACEHOLDER`
- `dsl.PIPELINE_JOB_SCHEDULE_TIME_UTC_PLACEHOLDER`
- `dsl.PIPELINE_ROOT_PLACEHOLDER`

有關每個特殊輸入提供的數據的更多信息，請參閱 [KFP SDK DSL 參考文檔](https://kubeflow-pipelines.readthedocs.io/en/master/source/dsl.html)。

## 任務配置

KFP SDK 通過任務方法公開了幾個與平台無關的任務級配置。與平台無關的配置是指那些預計會在所有符合 KFP 的後端（例如開源 KFP 後端或 Google Cloud Vertex AI Pipelines）上表現出類似執行行為的配置。

所有與平台無關的任務級配置都是使用 `PipelineTask` 方法設置的。以下面的環境變量為例:

```python
from kfp import dsl

@dsl.component
def print_env_var():
    import os
    print(os.environ.get('MY_ENV_VAR'))

@dsl.pipeline()
def my_pipeline():
    task = print_env_var()
    task.set_env_variable('MY_ENV_VAR', 'hello')
```

執行時，`print_env_var` 組件應打印 “hello”。

任務級配置方法也可以鏈接起來：

```python
print_env_var().set_env_variable('MY_ENV_VAR', 'hello').set_env_variable('OTHER_VAR', 'world')
```

KFP SDK 提供了以下任務方法來設置任務級別的配置：

- `.add_accelerator_type`
- `.set_accelerator_limit`
- `.set_cpu_limit`
- `.set_memory_limit`
- `.set_env_variable`
- `.set_caching_options`
- `.set_display_name`
- `.set_retry`
- `.ignore_upstream_failure`

有關這些方法的更多信息，請參閱 [`PipelineTask` 參考文檔](https://kubeflow-pipelines.readthedocs.io/en/master/source/dsl.html#kfp.dsl.PipelineTask)。

## 管道作為組件

管道本身可以用作其他管道中的組件，就像您在管道中使用任何其他單步組件一樣。例如，我們可以輕鬆地重組前面的畢達哥拉斯管道以使用內部輔助管道 `square_and_sum`:

```python
@dsl.pipeline
def square_and_sum(a: float, b: float) -> float:
    a_sq_task = square(x=a)
    b_sq_task = square(x=b)
    return add(x=a_sq_task.output, y=b_sq_task.output).output


@dsl.pipeline
def pythagorean(a: float = 1.2, b: float = 1.2) -> float:
    sq_and_sum_task = square_and_sum(a=a, b=b)
    return square_root(x=sq_and_sum_task.output).output
```