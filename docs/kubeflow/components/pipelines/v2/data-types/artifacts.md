# Artifacts

**創建、使用、傳遞和追踪 ML 工件**

大多數機器學習管道旨在創建一個或多個機器學習的工件，例如模型、數據集、評估指標等。

KFP 通過 `dsl.Artifact` 類和其他工件子類為創建機器學習工件提供先進的支持。 KFP 將這些工件映射到它們的底層 [ML 元數據](https://github.com/google/ml-metadata) schema，即 artifact type 的規範名稱。

通常，工件及其關聯的註釋有多種用途：

- 提供組件/管道輸入/輸出類型的邏輯分組
- 提供一種方便的機制，通過任務的本地文件系統寫入對象存儲
- 啟用創建 ML 工件的管道的類型檢查
- 通過特殊的 UI 渲染使某些工件類型的內容易於觀察

以下 `training_component` 演示了輸入和輸出工件的標準用法：

```python
from kfp.dsl import Input, Output, Dataset, Model

@dsl.component
def training_component(dataset: Input[Dataset], model: Output[Model]):
    """Trains an output Model on an input Dataset."""
    with open(dataset.path) as f:
        contents = f.read()

    # ... train tf_model model on contents of dataset ...

    tf_model.save(model.path)
    tf_model.metadata['framework'] = 'tensorflow'
```

此 `training_component` 執行以下操作：

1. 接受輸入數據集
2. 從本地文件系統讀取輸入數據集的內容
3. 訓練模型（略）
4. 將模型保存為組件輸出
5. 設置有關已保存模型的一些元數據

如 `training_component` 所示，工件只是一些工件屬性的包裝，包括可以從中讀取/寫入工件的 `.path` 和工件的 `.metadata`。以下部分詳細描述了工件的這些屬性和其他方面。

## Artifact 型別

工件註釋指示工件的類型。 KFP 在 DSL 中提供了多種工件類型：

|DSL 物件	|Artifact schema title|
|-----------|---------------------|
|`[Artifact](https://kubeflow-pipelines.readthedocs.io/en/latest/source/dsl.html#kfp.dsl.Artifact)`	|system.Artifact|
|`[Dataset](https://kubeflow-pipelines.readthedocs.io/en/latest/source/dsl.html#kfp.dsl.Dataset)`	|system.Dataset|
|`[Model](https://kubeflow-pipelines.readthedocs.io/en/latest/source/dsl.html#kfp.dsl.Model)`	|system.Model|
|`[Metrics](https://kubeflow-pipelines.readthedocs.io/en/latest/source/dsl.html#kfp.dsl.Metrics)`	|system.Metrics|
|`[ClassificationMetrics](https://kubeflow-pipelines.readthedocs.io/en/latest/source/dsl.html#kfp.dsl.ClassificationMetrics)`	|system.ClassificationMetrics|
|`[SlicedClassificationMetrics](https://kubeflow-pipelines.readthedocs.io/en/latest/source/dsl.html#kfp.dsl.SlicedClassificationMetrics)`	|system.SlicedClassificationMetrics|
|`[HTML](https://kubeflow-pipelines.readthedocs.io/en/latest/source/dsl.html#kfp.dsl.HTML)`	|system.HTML|
|`[Markdown](https://kubeflow-pipelines.readthedocs.io/en/latest/source/dsl.html#kfp.dsl.Markdown)`	|system.Markdown|


`Artifact`, `Dataset`, `Model` 和 `Metrics` 是最通用和最常用的工件類型。 `Artifact` 是默認的工件基本類型，應該在工件類型不能完全適合另一個工件類別的情況下使用。 `Artifact` 還與所有其他工件類型兼容。從這個意義上說，`Artifact` 類型也是一個 “any” 類型的工件。

在 KFP 開源 UI 上，`ClassificationMetrics`、`SlicedClassificationMetrics`、`HTML` 和 `Markdown` 提供了特殊的 UI 渲染，使工件的內容易於觀察。

## 宣告輸入/輸出 artifacts

在組件中，工件註釋必須始終包含在輸入或輸出類型標記中以指示工件的 I/O 類型。這是必需的，否則工件是輸入還是輸出會變得不明確，因為輸入和輸出工件都是通過 Python 函數參數聲明的。

在管道中，輸入工件註釋應包裝在輸入類型標記中，並且與組件不同，輸出工件應作為返回註釋提供，如 `concat_pipeline` 的 `dataset` 輸出所示：

```python
from kfp import dsl
from kfp.dsl import Dataset, Input, Output

@dsl.component
def concat_component(
    dataset1: Input[Dataset],
    dataset2: Input[Dataset],
    out_dataset: Output[Dataset],
):
    with open(dataset1.path) as f:
        contents1 = f.read()
    with open(dataset2.path) as f:
        contents2 = f.read()
    with open(out_dataset.path, 'w') as f:
        f.write(contents1 + contents2)

@dsl.pipeline
def concat_pipeline(
    d1: Input[Dataset],
    d2: Input[Dataset],
) -> Dataset:
    return concat_component(
        dataset1=d1,
        dataset2=d2
    ).output['out_dataset']
```

您可以指定多個管道工件輸出，就像您指定參數一樣。這由 `concat_pipeline2` 的輸出 `intermediate_dataset` 和 `final_dataset` 顯示：

```python
from typing import NamedTuple
from kfp import dsl
from kfp.dsl import Dataset, Input

@dsl.pipeline
def concat_pipeline2(
    d1: Input[Dataset],
    d2: Input[Dataset],
    d3: Input[Dataset],
) -> NamedTuple('Outputs',
                intermediate_dataset=Dataset,
                final_dataset=Dataset):
    Outputs = NamedTuple('Outputs',
                         intermediate_dataset=Dataset,
                         final_dataset=Dataset)
    concat1 = concat_component(
        dataset1=d1,
        dataset2=d2
    )
    concat2 = concat_component(
        dataset1=concat1.outputs['out_dataset'],
        dataset2=d3
    )
    return Outputs(intermediate_dataset=concat1.outputs['out_dataset'],
                   final_dataset=concat2.outputs['out_dataset'])
```

[KFP SDK 編譯器](https://kubeflow-pipelines.readthedocs.io/en/latest/source/compiler.html#kfp.compiler.Compiler)將根據類型檢查中描述的規則對工件使用進行類型檢查。

## 使用 output artifacts

當您在組件中使用輸入或輸出註釋時，您的組件會在運行時有效地請求工件的 `URI path`。

對於 output artifacts，正在創建的工件尚不存在（您的組件將要創建它！）。為了讓組件更容易創建工件，KFP 後端提供了一個唯一的系統生成的 URI，組件應該在其中寫入輸出工件。對於輸入和輸出工件，URI 是指定為管道根的雲對象存儲桶內的路徑。 URI 通過名稱、生產者任務和管道唯一標識輸出。系統生成的 URI 可以作為在運行時自動傳遞給組件的工件實例的 `.uri` 屬性的一個屬性來訪問：

```python
from kfp import dsl
from kfp.dsl import Model
from kfp.dsl import Output

@dsl.component
def print_artifact(model: Output[Model]):
    print('URI:', model.uri)
```

請注意，在編寫管道時，您永遠不會將輸出工件直接傳遞給組件。例如，在上面的 `concat_pipeline2` 中，我們沒有將 `out_dataset` 傳遞給 `concat_component`。輸出工件將在運行時使用正確的系統生成的 `URI` 自動傳遞給組件。

雖然您可以將輸出工件直接寫入 `URI`，但 KFP 通過工件的 `.path` 屬性提供了一種更簡單的機制：

```python
from kfp import dsl
from kfp.dsl import Model
from kfp.dsl import Output

@dsl.component
def print_and_create_artifact(model: Output[Model]):
    print('path:', model.path)
    with open(model.path, 'w') as f:
        f.write('my model!')
```

任務執行後，KFP 會自動處理將 `.path` 中的文件複製到 `.uri` 中的 `URI`，從而允許您僅通過與本地文件系統交互來創建工件文件。當輸出工件存儲為文件或目錄時，此方法有效。

對於輸出工件不容易用文件表示的情況（例如，輸出是包含模型的容器鏡像），您應該通過直接在工件上設置來覆蓋系統生成的 `.uri`，然後將輸出寫入那個位置。 KFP 會將更新後的 `URI` 存儲在 ML 元數據中。工件的 `.path` 屬性將沒有用。

## 使用 input artifacts

對於 input artifacts，`artifact URI` 已存在，因為工件已經創建好了。 KFP 根據管道中建立的數據交換處理(組件中的input/output串鏈)將正確的 `URI` 傳遞給您的組件。至於 output artifacts，KFP 處理將 `.uri` 中的現有文件複製到 `.path` 中的路徑，以便您的組件可以從本地文件系統讀取它。

input artifacts 應該被視為不可變的。您不應嘗試修改 `.path` 文件的內容，對 `.metadata` 的任何更改都不會影響 [ML 元數據](https://github.com/google/ml-metadata)中的工件元數據。

## Artifact 名稱和元數據

除了 `.uri` 和 `.path` 之外，工件還有 `.name` 和 `.metadata`。

```python
from kfp import dsl
from kfp.dsl import Dataset
from kfp.dsl import Input

@dsl.component
def count_rows(dataset: Input[Dataset]) -> int:
    with open(dataset.path) as f:
        lines = f.readlines()
    
    print('Information about the artifact:')
    print('Name:', dataset.name)
    print('URI:', dataset.uri)
    print('Path:', dataset.path)
    print('Metadata:', dataset.metadata)
    
    return len(lines)
```

在 KFP 中，工件可以有元數據，可以通過工件的 `.metadata` 屬性在組件中訪問元數據。元數據對於記錄有關工件的信息很有用，例如哪個 ML 框架生成了工件，其下游用途是什麼等。對於輸出工件，可以直接在 .`metadata` 字典上設置元數據，如前面 `training_component` 中的模型所示。