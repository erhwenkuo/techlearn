# 批量 mapping

將 `Dataset.map()` 的實用程序與批處理模式結合起來非常強大。它允許您加快處理速度，並自由控制生成的數據集的大小。

## 需要速度

批量 mapping 的主要目標是加快處理速度。很多時候，處理批量數據比處理單個樣本更快。當然，批量 mapping 適合 tokenization。例如，🤗 Tokenizers 庫在批處理方面工作得更快，因為它並行化了批處理中所有樣本的 tokenization。

## Input size != output size

控制生成的數據集大小的能力可用於許多有趣的用例。在“[如何映射](https://huggingface.co/docs/datasets/about_map_batch#map)”部分中，提供了使用批量 mapping 來執行以下操作的範例：

- 將長句子分成較短的塊。
- 使用額外的 token 來擴充數據集。

了解其工作原理很有幫助，因此您可以想出自己的方法來使用批量 mapping。此時，您可能想知道如何控制生成的數據集的大小。答案是：`map()` 函數不必返回相同大小的輸出批次。

換句話說，您的 `map()` 函數輸入可以是一批大小為 N 的批次，並返回一批大小為 M 的批次。輸出 M 可以大於或小於 N。這意味著您可以連接示例、將其分割，甚至添加更多例子！

但是，請記住，輸出字典中的所有值必須包含與輸出字典中其他字段相同數量的元素。否則，無法定義映射函數返回的輸出中的示例數量。映射函數處理的連續批次之間的數量可能會有所不同。但對於單個批次，輸出字典的所有值應具有相同的長度（即元素數量）。

例如，對於 1 列 3 行的數據集，如果使用 map 返回行數是兩倍的新列，則會出現錯誤。在本例中，您最終會得到一列 3 行，以及一列 6 行。如您所見，該表無效：

```python
from datasets import Dataset

dataset = Dataset.from_dict({"a": [0, 1, 2]})

dataset.map(lambda batch: {"b": batch["a"] * 2}, batched=True)  # new column with 6 elements: [0, 1, 2, 0, 1, 2]
```

要使其有效，您必須刪除其中一列：

```python
from datasets import Dataset

dataset = Dataset.from_dict({"a": [0, 1, 2]})

dataset_with_duplicates = dataset.map(lambda batch: {"b": batch["a"] * 2}, remove_columns=["a"], batched=True)

len(dataset_with_duplicates)
```