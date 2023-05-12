# 附加功能

**有關創作 KFP 組件的更多信息**

## 組件 docstring format

KFP 允許您使用 Python docstrings 記錄您的組件和管道。當您編譯組件和管道時，KFP SDK 會自動解析您的文檔字符串並在 [IR YAML](https://www.kubeflow.org/docs/components/pipelines/v2/compile-a-pipeline#ir-yaml) 中包含某些字段。

對於組件，KFP 可以提取您的組件 input descriptions 和 output descriptions。

對於管道，KFP 可以提取您的管道 input descriptions 和 output descriptions,，以及您的完整管道的描述。

為了讓 KFP SDK 正確解析您的文檔字符串，您應該以 KFP 文檔字符串樣式編寫docstring。 KFP 文檔 docstring 樣式是 [Google docstring](https://sphinxcontrib-napoleon.readthedocs.io/en/latest/example_google.html) 樣式的一個特殊變體，具有以下更改：

- `Returns`: 採用與 `Args` 部分相同的結構，其中 `Returns:` 的每個返回值應採用 `<name>: <description>` 的形式。這與典型的 Google 文檔字符串 Returns: 部分不同，後者採用 <type>: <description> 的形式，沒有返回值的名稱。
- 組件輸出應包含在 `Returns:` 部分中，即使它們是通過組件函數輸入參數聲明的。這適用於使用 `dsl.OutputPath` 和用於聲明輸出工件的 `Output[<Artifact>]` 類型標記註釋的函數參數。
- 建議：類型信息，包括哪些輸入是可選的/必需的，應該從輸入/輸出描述中省略。此信息與註釋重複。

例如，KFP SDK 可以從以下使用 KFP 文檔字符串樣式的組件文檔字符串中提取輸入和輸出描述：

```python
@dsl.component
def join_datasets(
    dataset_a: Input[Dataset],
    dataset_b: Input[Dataset],
    out_dataset: Output[Dataset],
) -> str:
    """Concatenates two datasets.

    Args:
        dataset_a: First dataset.
        dataset_b: Second dataset.

    Returns:
        out_dataset: The concatenated dataset.
        Output: The concatenated string.
    """
    ...
```

類似地，KFP 可以從以下管道文檔字符串中提取組件輸入描述、組件輸出描述和管道描述：

```python
@dsl.pipeline(display_name='Concatenation pipeline')
def dataset_concatenator(
    string: str,
    in_dataset: Input[Dataset],
) -> Dataset:
    """Pipeline to convert string to a Dataset, then concatenate with
    in_dataset.

    Args:
        string: String to concatenate to in_artifact.
        in_dataset: Dataset to which to concatenate string.

    Returns:
        Output: The final concatenated dataset.
    """
    ...
```

!!! info
    請注意，如果您向 `@dsl.pipeline` 裝飾器提供描述參數，KFP 將使用此描述而不是文檔字符串描述。

