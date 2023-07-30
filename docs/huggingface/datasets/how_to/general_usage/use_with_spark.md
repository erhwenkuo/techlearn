# Use with Spark

本文檔快速介紹瞭如何將 🤗 數據集與 Spark 結合使用，特別關注如何將 `Spark DataFrame` 轉換成 Huggingface 的 `Dataset` 物件。

從那裡，您可以快速訪問任何元素，並且可以將其用作數據加載器來訓練模型。

## Load from Spark

Dataset 物件是 Apache Arrow table 的 wrapper，它允許從數據集中的 array 快速讀取到 PyTorch、TensorFlow 和 JAX 張量。 Arrow table 是從磁盤映射的內存，它可以加載大於可用 RAM 的數據集。

您可以使用 `Dataset.from_spark()` 從 `Spark DataFrame` 獲取數據集：

```python
from datasets import Dataset

df = spark.createDataFrame(
    data=[[1, "Elia"], [2, "Teo"], [3, "Fang"]],
    columns=["id", "name"],
)

ds = Dataset.from_spark(df)
```

Spark worker 線程將數據集作為 Arrow 文件寫入磁盤上的緩存目錄中，並從那裡加載數據集。

或者，您可以使用 `IterableDataset.from_spark()` ，它返回一個 `IterableDataset`：

```python
from datasets import IterableDataset

df = spark.createDataFrame(
    data=[[1, "Elia"], [2, "Teo"], [3, "Fang"]],
    columns=["id", "name"],
)

ds = IterableDataset.from_spark(df)

print(next(iter(ds)))
```

結果:

```bash
{"id": 1, "name": "Elia"}
```

### Caching

使用 `Dataset.from_spark()` 時，生成的 Dataset 會被緩存；如果您在同一個 DataFrame 上多次調用 `Dataset.from_spark()` ，它不會重新運行將數據集作為 Arrow 文件寫入磁盤的 Spark job。

您可以通過將 `cache_dir=` 傳遞給 `Dataset.from_spark()` 來設置緩存位置。確保使用您的 worker 和當前計算機（驅動程序）都可用的 disk。

### Feature types

如果您的數據集由 images、audio 數據或 N 維陣列組成，您可以在 `Dataset.from_spark()`（或 `IterableDataset.from_spark()`）中指定 `features=` 參數：

```python
from datasets import Dataset, Features, Image, Value

data = [(0, open("image.png", "rb").read())]

df = spark.createDataFrame(data, "idx: int, image: binary")

# Also works if you have arrays
# data = [(0, np.zeros(shape=(32, 32, 3), dtype=np.int32).tolist())]
# df = spark.createDataFrame(data, "idx: int, image: array<array<array<int>>>")
features = Features({"idx": Value("int64"), "image": Image()})

dataset = Dataset.from_spark(df, features=features)

print(dataset[0])
```

結果:

```bash
{'idx': 0, 'image': <PIL.PngImagePlugin.PngImageFile image mode=RGB size=32x32>}
```

您可以查看 [Features 文檔](https://huggingface.co/docs/datasets/v2.14.1/en/package_reference/main_classes#datasets.Features)以了解所有可用的 feature types。