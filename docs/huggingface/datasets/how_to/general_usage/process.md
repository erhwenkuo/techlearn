# Process

🤗 Datasets 提供了許多用於修改數據集的結構和內容的工具。這些工具對於整理數據集、創建附加列、在功能和格式之間進行轉換等非常重要。

本指南將向您展示如何：

- 對行(rows)重新排序並拆分數據集。
- 重命名和刪除列(columns)以及其他常見的列操作。
- 將 processing function 應用於數據集中的每個示例行(row)。
- 串接數據集。
- 應用自定義格式轉換。
- 保存並導出處理後的數據集。

有關處理其他數據集模態的更多詳細信息，請查看 [process audio dataset](https://huggingface.co/docs/datasets/audio_process) 指南、[process image dataset](https://huggingface.co/docs/datasets/image_process) 指南或 [process text dataset](https://huggingface.co/docs/datasets/nlp_process) 指南。

本指南中的示例使用 [MRPC]() 數據集，但請隨意加載您選擇的任何數據集並繼續操作！

```python
from datasets import load_dataset

dataset = load_dataset("glue", "mrpc", split="train")
```

!!! tip
    本指南中的所有處理方法都會返回一個新的 [Dataset](https://huggingface.co/docs/datasets/v2.14.0/en/package_reference/main_classes#datasets.Dataset) 物件。

## Sort, shuffle, select, split, and shard

有幾個函數可用於重新排列數據集的結構。這些函數對於僅選擇所需的行、創建訓練和測試分割以及將非常大的數據集分割成較小的分塊非常有用。

###　Sort

使用 `sort()` 根據列值(column)的數值進行排序。提供的列的欄位名(column name)必須與 NumPy 兼容。

```python
dataset["label"][:10]
sorted_dataset = dataset.sort("label")
sorted_dataset["label"][:10]
sorted_dataset["label"][-10:]
```

結果:

```bash
[1, 0, 1, 0, 1, 1, 0, 1, 0, 0]
```

```python
sorted_dataset = dataset.sort("label")

sorted_dataset["label"][:10]
```

結果:

```bash
[0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
```

```python
sorted_dataset["label"][-10:]
```

結果:

```bash
[1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
```

在底層，這會創建一個根據列值排序的索引列表。然後，該索引映射用於訪問基礎 Arrow 表中的正確行。

### Shuffle

`shuffle()` 函數隨機重新排列列值(column values)。如果您想更好地控制用於洗牌數據集的算法，您可以在此函數中指定 `generator` 參數來使用不同的 `numpy.random.Generator`。

```python
shuffled_dataset = sorted_dataset.shuffle(seed=42)

shuffled_dataset["label"][:10]
```

結果:

```bash
[1, 1, 1, 0, 1, 1, 1, 1, 1, 0]
```

Shuffling 獲取索引列表 [0:len(my_dataset)] 並創建一個新的 shuffling 的索引映射。然而，一旦您的數據集具有索引映射，速度就會變慢 10 倍。

這是因為需要一個額外的步驟來使用索引映射來讀取要讀取的行索引，最重要的是，您不再讀取連續的數據塊。

要恢復速度，您需要使用 [`Dataset.flatten_indices()`](https://huggingface.co/docs/datasets/v2.14.0/en/package_reference/main_classes#datasets.Dataset.flatten_indices) 再次重寫磁盤上的整個數據集，這會刪除索引映射。或者，您可以切換到 [IterableDataset](https://huggingface.co/docs/datasets/v2.14.0/en/package_reference/main_classes#datasets.IterableDataset) 並利用其快速近似的方法 [IterableDataset.shuffle()](https://huggingface.co/docs/datasets/v2.14.0/en/package_reference/main_classes#datasets.IterableDataset.shuffle)：

```python
iterable_dataset = dataset.to_iterable_dataset(num_shards=128)

shuffled_iterable_dataset = iterable_dataset.shuffle(seed=42, buffer_size=1000)
```

### Select and Filter

有兩個選項可用於過濾數據集中的行：`select()` 和 `filter()`。

- `select()` 根據索引列表返回行(rows)：

    ```python
    small_dataset = dataset.select([0, 10, 20, 30, 40, 50])

    len(small_dataset)
    ```

    結果:

    ```bash
    6
    ```

- `filter()` 返回符合指定條件的行(rows)：

    ```python
    start_with_ar = dataset.filter(lambda example: example["sentence1"].startswith("Ar"))

    print(len(start_with_ar))

    print(start_with_ar["sentence1"])
    ```

    結果:

    ```bash
    6

    ['Around 0335 GMT , Tab shares were up 19 cents , or 4.4 % , at A $ 4.56 , having earlier set a record high of A $ 4.57 .',
    'Arison said Mann may have been one of the pioneers of the world music movement and he had a deep love of Brazilian music .',
    'Arts helped coach the youth on an eighth-grade football team at Lombardi Middle School in Green Bay .',
    'Around 9 : 00 a.m. EDT ( 1300 GMT ) , the euro was at $ 1.1566 against the dollar , up 0.07 percent on the day .',
    "Arguing that the case was an isolated example , Canada has threatened a trade backlash if Tokyo 's ban is not justified on scientific grounds .",
    'Artists are worried the plan would harm those who need help most - performers who have a difficult time lining up shows .'
    ]    
    ```

    如果設定 `with_indices=True`，`filter()` 還可以按索引來進行過濾：

    ```python
    even_dataset = dataset.filter(lambda example, idx: idx % 2 == 0, with_indices=True)

    print(len(even_dataset))

    print(len(dataset) / 2)
    ```

    ```bash
    1834
    
    1834.0
    ```

    除非要保留​​的索引列表是連續的，否則這些方法還會在幕後創建索引映射。


### Split

如果您的數據集還沒有區隔 `train` 和 `test` 的 splits，[`train_test_split()`](https://huggingface.co/docs/datasets/v2.14.0/en/package_reference/main_classes#datasets.Dataset.train_test_split) 函數會創建它們。這允許您調整每個分割中樣本的相對比例或絕對數量。在下面的示例中，使用 `test_size` 參數創建一個佔原始數據集 10% 的測試分割：

```python
new_dataset = dataset.train_test_split(test_size=0.1)

print(new_dataset)
```

結果:

```bash
{'train': Dataset(schema: {'sentence1': 'string', 'sentence2': 'string', 'label': 'int64', 'idx': 'int32'}, num_rows: 3301),
'test': Dataset(schema: {'sentence1': 'string', 'sentence2': 'string', 'label': 'int64', 'idx': 'int32'}, num_rows: 367)}
```

比對一下原有的 dataset:

```python
print(0.1 * len(dataset))
```

結果:

```bash
366.8
```

默認情況下，分割是隨機排列的，但您可以設置 `shuffle=False` 來防止隨機排列。

### Shard

🤗 數據集支持分片(sharding)，將非常大的數據集劃分為預定義數量的塊。在 `shard()` 中指定 `num_shards` 參數來確定將數據集拆分為的分片數量。您還需要提供要使用索引參數返回的分片。

例如，[imdb](https://huggingface.co/datasets/imdb) 數據集有 `25000` 個示例：

```python
from datasets import load_dataset

datasets = load_dataset("imdb", split="train")

print(dataset)
```

結果:

```bash
Dataset({
    features: ['text', 'label'],
    num_rows: 25000
})
```

將數據集分片為四個塊(chunks)後，第一個分片(shard)將只有 `6250` 個示例：

```python
first_shard_dataset = dataset.shard(num_shards=4, index=0)

print(first_shard_dataset)

print(25000/4)
```

結果:

```bash
Dataset({
    features: ['text', 'label'],
    num_rows: 6250
})

6250.0
```

## Rename, remove, cast, and flatten

以下函數允許您修改數據集的列(column)。這些函數對於重命名或刪除列、將列更改為一組新功能以及展平嵌套列結構非常有用。

### Rename

當您需要重命名數據集中的列(column)時，請使用 `rename_column()`。與原始列關聯的特徵實際上被移動到新列名稱下，而不是僅僅就地替換原始列。

為 `rename_column()` 提供原始列的名稱和新列的名稱：

```python
print(dataset)

dataset = dataset.rename_column("sentence1", "sentenceA")
dataset = dataset.rename_column("sentence2", "sentenceB")

print(dataset)
```

結果:

```bash
# before
Dataset({
    features: ['sentence1', 'sentence2', 'label', 'idx'],
    num_rows: 3668
})

# after
Dataset({
    features: ['sentenceA', 'sentenceB', 'label', 'idx'],
    num_rows: 3668
})
```

### Remove

當您需要刪除一列或多列時，請向 `remove_columns()` 函數提供要刪除的列名稱。通過提供列名稱列表來刪除多個列：

```python
dataset = dataset.remove_columns("label")

print(dataset)
```

結果:

```bash
Dataset({
    features: ['sentence1', 'sentence2', 'idx'],
    num_rows: 3668
})
```

```python
dataset = dataset.remove_columns(["sentence1", "sentence2"])

print(dataset)
```

結果:

```bash
Dataset({
    features: ['idx'],
    num_rows: 3668
})
```

### Cast

`cast()` 函數轉換一列或多列的特徵類型。該函數接受您的新功能作為其參數。下面的示例演示瞭如何更改 `ClassLabel` 和 `Value` 這兩個在 dataset 裡的 feature 數據：

```python
print(dataset.features)
```

結果:

```bash
{'sentence1': Value(dtype='string', id=None),
'sentence2': Value(dtype='string', id=None),
'label': ClassLabel(num_classes=2, names=['not_equivalent', 'equivalent'], names_file=None, id=None),
'idx': Value(dtype='int32', id=None)}
```

```python
from datasets import ClassLabel, Value

new_features = dataset.features.copy()

new_features["label"] = ClassLabel(names=["negative", "positive"])

new_features["idx"] = Value("int64")

dataset = dataset.cast(new_features)

print(dataset.features)
```

結果:

```bash
{'sentence1': Value(dtype='string', id=None),
'sentence2': Value(dtype='string', id=None),
'label': ClassLabel(num_classes=2, names=['negative', 'positive'], names_file=None, id=None),
'idx': Value(dtype='int64', id=None)}
```

!!! tip
    僅當原始 feature type 和新 feature type 兼容時，`cast()` 才有效。例如，如果原始列僅包含 1 和 0，則可以將要素類型 `Value("int32")` 的列轉換為 `Value("bool")`。

使用 [`cast_column()`](https://huggingface.co/docs/datasets/v2.14.0/en/package_reference/main_classes#datasets.Dataset.cast_column) 函數更改單個列的要素類型。將 `column name` 及 `new feature type` 作為參數傳遞：

```python
print(dataset.features)
```

結果:

```bash
{'audio': Audio(sampling_rate=44100, mono=True, id=None)}
```

```python
dataset = dataset.cast_column("audio", Audio(sampling_rate=16000))

print(dataset.features)
```

結果:

```bash
{'audio': Audio(sampling_rate=16000, mono=True, id=None)}
```

### Flatten

有時，列可以是多種類型的嵌套結構。看一下 [SQuAD](https://huggingface.co/datasets/squad) 數據集中的嵌套結構：

```python
from datasets import load_dataset

dataset = load_dataset("squad", split="train")

print(dataset.features)
```

結果:

```bash hl_lines="1"
{'answers': Sequence(feature={'text': Value(dtype='string', id=None), 'answer_start': Value(dtype='int32', id=None)}, length=-1, id=None),
'context': Value(dtype='string', id=None),
'id': Value(dtype='string', id=None),
'question': Value(dtype='string', id=None),
'title': Value(dtype='string', id=None)}
```

`answers` 欄位包含兩個子欄位： `text` 和 `answer_start`。使用 `flatten()` 函數將子欄位提取到它們自己的單獨列中：

```python
flat_dataset = dataset.flatten()

print(flat_dataset)
```

結果:

```bash hl_lines="1"
Dataset({
    features: ['id', 'title', 'context', 'question', 'answers.text', 'answers.answer_start'],
 num_rows: 87599
})
```

請注意，子欄位現在是它們自己的獨立欄位了：`answers.text` 和 `answer.answer_start`。

