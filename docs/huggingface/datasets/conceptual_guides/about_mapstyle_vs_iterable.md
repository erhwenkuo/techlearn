# Dataset 與 IterableDataset 的差異

Huggingface 有兩種類型的數據集物件：[`Dataset`](https://huggingface.co/docs/datasets/v2.14.1/en/package_reference/main_classes#datasets.Dataset) 和 [`IterableDataset`](https://huggingface.co/docs/datasets/v2.14.1/en/package_reference/main_classes#datasets.IterableDataset)。您選擇使用或創建哪種類型的數據集取決於數據集的大小。一般來說，由於其惰性行為(lazy)和速度優勢，`IterableDataset` 非常適合大型數據集（想想數百 GB！），而 `Dataset` 則適合其他所有數據集應用。本文將比較 `Dataset` 和 `IterableDataset` 之間的差異，以幫助您選擇適合您的數據集物件。

## 下載與串流

當您有常規 `Dataset` 時，可以使用 `my_dataset[0]` 訪問它。這提供了對行的隨機訪問。此類數據集也稱為 “map-style” 數據集。例如，您可以像這樣下載 [imagenet-1k](https://huggingface.co/datasets/imagenet-1k) 並訪問任意行(row)：

```python
from datasets import load_dataset

imagenet = load_dataset("imagenet-1k", split="train")  # downloads the full dataset

print(imagenet[0])
```


但需要注意的是，您必須將整個數據集存儲在磁盤或內存中，這會阻止您訪問大於磁盤的數據集。由於對於大型數據集可能會變得不方便，因此存在另一種類型的數據集：[`IterableDataset`](https://huggingface.co/docs/datasets/v2.14.1/en/package_reference/main_classes#datasets.IterableDataset)。當您擁有 `IterableDataset` 時，您可以使用 `for loop` 訪問它，以便在迭代數據集時逐步加載數據。這樣，只有一小部分樣本數據被加載到內存中，並且您不會在磁盤上寫入任何內容。

例如，您可以流式(stream)傳輸 ImageNet-1k 數據集，而無需將其下載到磁盤上：

```python
from datasets import load_dataset

imagenet = load_dataset("imagenet-1k", split="train", streaming=True)  # will start loading the data when iterated over

for example in imagenet:
    print(example)
    break
```

流式傳輸可以讀取 online 數據，而無需將任何文件寫入磁盤。例如，您可以流式傳輸由多個分片組成的數據集，每個分片都有數百 GB，例如 [C4](https://huggingface.co/datasets/c4)、[OSCAR](https://huggingface.co/datasets/oscar) 或 [LAION-2B](https://huggingface.co/datasets/laion/laion2B-en)。在 [Dataset Streaming Guide](https://huggingface.co/docs/datasets/stream) 中了解有關如何流式傳輸數據集的更多信息。


## 創建 map-style 與 iterable datasets

您可以使用 list 或 dictionary 創建 `Dataset`，並且將數據格式轉換為 Arrow，以便您可以輕鬆訪問任何行(row)：

```python
my_dataset = Dataset.from_dict({"col_1": [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]})

print(my_dataset[0])
```

另一方面，要創建 `IterableDataset`，您必須提供一種 “惰性(lazy)” 方式來加載數據。在 Python 中，我們一般使用 generator functions。這些函數一次(yield)生成一個樣本(example)，這意味著您無法像常規 `Dataset` 那樣通過切片來訪問：

```python
def my_generator(n):
    for i in range(n):
        yield {"col_1": i}

my_iterable_dataset = IterableDataset.from_generator(my_generator, gen_kwargs={"n": 10})

for example in my_iterable_dataset:
    print(example)
    break
```

## 完整且漸進地加載本地文件

可以使用 [`load_dataset()`](https://huggingface.co/docs/datasets/v2.14.1/en/package_reference/loading_methods#datasets.load_dataset) 將本地或遠程數據文件轉換為 Arrow 數據集：

```python
data_files = {"train": ["path/to/data.csv"]}

my_dataset = load_dataset("csv", data_files=data_files, split="train")

print(my_dataset[0])
```

但是，這需要從 `CSV` 到 `Arrow` 格式的轉換步驟，如果數據集很大，則需要時間和磁盤空間。

為了節省磁盤空間並跳過轉換步驟，您可以通過直接從本地文件流式傳輸來定義 `IterableDataset`。這樣，當您迭代數據集時，會逐漸從本地文件中讀取數據：

```python
data_files = {"train": ["path/to/data.csv"]}

my_iterable_dataset = load_dataset("csv", data_files=data_files, split="train", streaming=True)

for example in my_iterable_dataset:  # this reads the CSV file progressively as you iterate over the dataset
    print(example)
    break
```

Huggingfacre 支持多種文件格式的轉換，例如 CSV、JSONL 和 Parquet，以及　image　和　audio 檔案。您可以在相應的指南中找到有關加載表格、文本、視覺和音頻數據集的更多信息。

##　立即性與惰性 data processing

當您使用 `Dataset.map()` 處理 `Dataset` 物件時，會立即處理並返回整個數據集。例如，這類似於 pandas 的工作方式。

```python
my_dataset = my_dataset.map(process_fn)  # process_fn is applied on all the examples of the dataset

print(my_dataset[0])
```

另一方面，由於 `IterableDataset` 的“惰性(lazy)”本質，調用 `IterableDataset.map()` 不會將映射函數應用於整個數據集。相反，您的 map function 是即時(on-the-fly)被應用的。

因此，您可以鏈接多個處理步驟，並且當您開始迭代數據集時，它們將同時串接運行：

```python
my_iterable_dataset = my_iterable_dataset.map(process_fn_1)
my_iterable_dataset = my_iterable_dataset.filter(filter_fn)
my_iterable_dataset = my_iterable_dataset.map(process_fn_2)

# process_fn_1, filter_fn and process_fn_2 are applied on-the-fly when iterating over the dataset
for example in my_iterable_dataset:  
    print(example)
    break
```

## 精確洗牌與快速近似洗牌

當您使用 `Dataset.shuffle()` 進行數據集洗牌時，您將精確地 shuffling 數據集。它的工作原理是獲取索引列表 [0, 1, 2, ... len(my_dataset) - 1] 並打亂該列表。然後，訪問 my_dataset[0] 返回由已打亂的索引映射的第一個元素定義的行和索引：

```python
my_dataset = my_dataset.shuffle(seed=42)

print(my_dataset[0])
```

由於在 `IterableDataset` 的設計我們無法隨機訪問行(row)，因此我們無法使用打亂的索引列表並訪問任意位置的行。這阻止了精確洗牌的使用。相反，`IterableDataset.shuffle()` 中使用快速近似混洗。它使用隨機緩衝區從數據集中迭代地採樣隨機樣本。由於數據集仍然是迭代讀取的，因此它提供了出色的速度性能：

```python
my_iterable_dataset = my_iterable_dataset.shuffle(seed=42, buffer_size=100)

for example in my_iterable_dataset:
    print(example)
    break
```

如果使用 shuffle buffer 的功能若還不足以為機器學習模型訓練提供令人滿意的混洗時。因此，如果您的數據集由多個文件或 shard 組成，`IterableDataset.shuffle()` 也可用數據集分片進行洗牌：

```python
# Stream from the internet
my_iterable_dataset = load_dataset("c4", "en", split="train", streaming=True)
my_iterable_dataset.n_shards  # 1024

# Stream from local files
data_files = {"train": [f"path/to/data_{i}.csv" for i in range(1024)]}
my_iterable_dataset = load_dataset("csv", data_files=data_files, split="train", streaming=True)
my_iterable_dataset.n_shards  # 1024

# From a generator function
def my_generator(n, sources):
    for source in sources:
        for example_id_for_current_source in range(n):
            yield {"example_id": f"{source}_{example_id_for_current_source}"}

gen_kwargs = {"n": 10, "sources": [f"path/to/data_{i}" for i in range(1024)]}

my_iterable_dataset = IterableDataset.from_generator(my_generator, gen_kwargs=gen_kwargs)

my_iterable_dataset.n_shards  # 1024
```

## 速度差異

常規 `Dataset` 物件基於 Arrow 格式，它提供對行(row)的快速隨機訪問。由於內存映射以及 Arrow 是一種內存格式，從磁盤讀取數據不會執行昂貴的系統調用和反序列化。當使用 for 循環迭代連續的 Arrow 記錄批次時，它可以提供更快的數據加載。

然而，一旦您的數據集使用了索引映射（例如通過 `Dataset.shuffle()`），速度就會變慢 10 倍。這是因為 `Dataset` 需要一個額外的步驟來使用索引映射來讀取要讀取的行索引，最重要的是，{==您不再讀取連續的數據塊==}。要恢復速度，您需要使用 `Dataset.flatten_indices()` 再次重寫磁盤上的整個數據集，這會刪除索引映射。不過，這可能需要很長時間，具體取決於數據集的大小：

```python
my_dataset[0]  # fast

my_dataset = my_dataset.shuffle(seed=42)

my_dataset[0]  # up to 10x slower

my_dataset = my_dataset.flatten_indices()  # rewrite the shuffled dataset on disk as contiguous chunks of data

my_dataset[0]  # fast again
```

在這種情況下，我們建議切換到 `IterableDataset` 並利用其快速近似混洗方法 `IterableDataset.shuffle()`。它僅打亂分片順序並向數據集添加打亂緩衝區，從而使數據集的速度保持最佳。您還可以輕鬆重新整理數據集：

```python
for example in enumerate(my_iterable_dataset):  # fast
    pass

shuffled_iterable_dataset = my_iterable_dataset.shuffle(seed=42, buffer_size=100)

for example in enumerate(shuffled_iterable_dataset):  # as fast as before
    pass

shuffled_iterable_dataset = my_iterable_dataset.shuffle(seed=1337, buffer_size=100)  # reshuffling using another seed is instantaneous

for example in enumerate(shuffled_iterable_dataset):  # still as fast as before
    pass
```

如果您在多個 epoch 上使用數據集，則在洗牌緩衝區中打亂分片順序的有效種子是 `seed + epoch`。它可以輕鬆地在 epoch 之間重新整理數據集：

```python
for epoch in range(n_epochs):
    my_iterable_dataset.set_epoch(epoch)
    for example in my_iterable_dataset:  # fast + reshuffled at each epoch using `effective_seed = seed + epoch`
        pass
```

## 從 map-style 切換成 iterable

如果您想受益於 `IterableDataset` 的“惰性(lazy)”行為或其速度優勢，您可以將地 map-style 的 `Dataset` 切換為 `IterableDataset`：

```python
my_iterable_dataset = my_dataset.to_iterable_dataset()
```

如果您想打亂數據集或將其與 [PyTorch DataLoader](https://huggingface.co/docs/datasets/use_with_pytorch#stream-data) 一起使用，我們建議生成一個共享的 `IterableDataset`：

```python
my_iterable_dataset = my_dataset.to_iterable_dataset(num_shards=1024)

my_iterable_dataset.n_shards  # 1024
```