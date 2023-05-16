# 控制 Flow

**創建具有控制流的管道**

雖然用 `@dsl.pipeline` 裝飾器裝飾的 KFP 管道看起來像一個普通的 Python 函數，但它實際上是管道拓撲和控制流語義的表達，使用 KFP 領域特定語言 (DSL) 構建。[Pipeline 基礎知識](./pipeline-basics.md)介紹了數據傳遞如何通過任務依賴關係表示管道拓撲。本節介紹如何使用 KFP DSL 在管道中使用控制流。 DSL 具有三種類型的控制流，每種都由 Python 上下文管理器實現：

1. Conditions
2. Looping
3. Exit handling

## Conditions (`dsl.Condition`)

`dsl.Condition` 上下文管理器可以根據上游任務或管道輸入參數的輸出在其範圍內有條件地執行任務。上下文管理器有兩個參數：**一個必需的條件**和**一個可選的名稱**。`condition` 是一個比較表達式，其中兩個操作數中至少有一個是上游任務的輸出或管道輸入參數。

在以下管道中，`conditional_task` 僅在 `coin_flip_task` 具有輸出 “heads” 時執行。

```python
from kfp import dsl

@dsl.pipeline
def my_pipeline():
    coin_flip_task = flip_coin()
    with dsl.Condition(coin_flip_task.output == 'heads'):
        conditional_task = my_comp()
```

## Parallel looping (dsl.ParallelFor)

`dsl.ParallelFor` 上下文管理器允許在一組靜態項目上併行執行任務。上下文管理器採用三個參數：必需 `items`、可選 `parallelism` 和可選 `name`。 `items` 是要循環的靜態項目集，`parallelism` 是執行 `dsl.ParallelFor` 組時允許的最大併發迭代數。 `parallelism=0` 表示不受約束的併行性。

在以下管道中，`train_model` 將訓練一個模型 `1`、`5`、`10` 和 `25` 個 epoch，一次運行的訓練任務不超過兩個：

```python
from kfp import dsl

@dsl.pipeline
def my_pipeline():
    with dsl.ParallelFor(
        items=[1, 5, 10, 25],
        parallelism=2
    ) as epochs:
        train_model(epochs=epochs)
```

使用 `dsl.Collected` 和 `dsl.ParallelFor` 從併行任務循環中收集輸出：

```python
from kfp import dsl

@dsl.pipeline
def my_pipeline():
    with dsl.ParallelFor(
        items=[1, 5, 10, 25],
    ) as epochs:
        train_model_task = train_model(epochs=epochs)
    max_accuracy(models=dsl.Collected(train_model_task.outputs['model']))
```

下游任務可能會通過使用參數列表或工件列表註釋的輸入來使用 `dsl.Collected` 輸出。例如，前面示例中的 `max_accuracy` 具有類型為 `Input[List[Model]]` 的輸入模型，如以下組件定義所示：

```python
from kfp import dsl
from kfp.dsl import Model, Input

@dsl.component
def select_best(models: Input[List[Model]]) -> float:
    return max(score_model(model) for model in models)
```

您可以使用 `dsl.Collected` 從嵌套參數列表中的嵌套循環收集輸出。例如，來自兩個嵌套 `dsl.ParallelFor` 組的輸出參數被收集在多級嵌套參數列表中，其中每個嵌套列表包含來自 `dsl.ParallelFor` 組之一的輸出參數。嵌套級別的數量基於嵌套的 `dsl.ParallelFor` 上下文的數量。

相比之下，在嵌套循環中創建的工件被收集在一個平面列表中。

您還可以從管道返回 `dsl.Collected`。在返回註釋中使用參數列表或工件列表，如以下示例所示：

```python
from kfp import dsl
from kfp.dsl import Model

@dsl.pipeline
def my_pipeline() -> List[Model]:
    with dsl.ParallelFor(
        items=[1, 5, 10, 25],
    ) as epochs:
        train_model_task = train_model(epochs=epochs)
    return dsl.Collected(train_model_task.outputs['model'])
```

## Exit handling (dsl.ExitHandler)

`dsl.ExitHandler` 上下文管理器允許管道作者指定一個退出任務，該任務將在上下文管理器範圍內的任務完成執行後運行，即使其中一個任務失敗。這類似於在普通 Python 中使用 `try:` 塊後跟 `finally:` 塊，其中退出任務位於 `finally:` 塊中。上下文管理器有兩個參數：一個必需的 `exit_task` 和一個可選的 `name`。 `exit_task` 接受一個實例化的 `PipelineTask`。

在以下管道中，`clean_up_task` 將在 `create_dataset` 和 `train_and_save_models` 完成或其中一個失敗後執行：

```python
from kfp import dsl

@dsl.pipeline
def my_pipeline():
    clean_up_task = clean_up_resources()
    with dsl.ExitHandler(exit_task=clean_up_task):
        dataset_task = create_datasets()
        train_task = train_and_save_models(dataset=dataset_task.output)
```

您用作退出任務的任務可能會使用特殊輸入，該輸入提供對管道和任務狀態元數據的訪問，包括管道失敗或成功狀態。您可以通過使用 `dsl.PipelineTaskFinalStatus` 註釋註釋您的退出任務來使用此特殊輸入。該參數的參數將在運行時由後端自動提供。實例化退出任務時，不應向此註釋提供任何輸入。

以下管道使用 `dsl.PipelineTaskFinalStatus` 獲取有關管道和任務失敗的信息，即使在 `fail_op` 失敗後也是如此：

```python
from kfp import dsl
from kfp.dsl import PipelineTaskFinalStatus


@dsl.component
def exit_op(user_input: str, status: PipelineTaskFinalStatus):
    """Prints pipeline run status."""
    print(user_input)
    print('Pipeline status: ', status.state)
    print('Job resource name: ', status.pipeline_job_resource_name)
    print('Pipeline task name: ', status.pipeline_task_name)
    print('Error code: ', status.error_code)
    print('Error message: ', status.error_message)

@dsl.component
def fail_op():
    import sys
    sys.exit(1)

@dsl.pipeline
def my_pipeline():
    print_op()
    print_status_task = exit_op(user_input='Task execution status:')
    with dsl.ExitHandler(exit_task=print_status_task):
        fail_op()
```

## Ignore upstream failure

`PipelineTask` 上的 `.ignore_upstream_failure()` 任務方法啟用了另一種方法來編寫具有退出處理行為的管道。在任務上調用此方法會導致任務忽略任何指定上游任務的失敗（通過數據交換或使用 `.after()` 建立）。如果任務沒有上游任務，則此方法無效。

在下面的流水線定義中，不管 `fail_op` 是否成功，`clean_up_task` 都會在 `fail_op` 之後執行：

```python
from kfp import dsl

@dsl.pipeline()
def my_pipeline(text: str = 'message'):
    task = fail_op(message=text)
    clean_up_task = print_op(
        message=task.output).ignore_upstream_failure()
```

請注意，用於調用者任務的組件（上例中的 `print_op`）需要它從上游任務消耗的所有輸入的默認值。如果上游任務無法生成傳遞給調用者任務的輸出，則應用默認值。指定默認值可確保調用者任務始終成功，而不管上游任務的狀態如何。