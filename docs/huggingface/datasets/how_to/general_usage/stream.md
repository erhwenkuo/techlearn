# Stream

Dataset streaming 處理讓您無需下載數據集即可使用它。當您迭代數據集時，數據將被流式傳輸。這在以下情況下特別有用：

- 您不想等待下載非常大的數據集。
- 數據集大小超出了計算機上的可用磁盤空間量。
- 您只想快速探索數據集的幾個樣本。

![](./assets/streaming.gif)

例如，[oscar-corpus/OSCAR-2201](https://huggingface.co/datasets/oscar-corpus/OSCAR-2201) 數據集的英文分割為 `1.2 TB`，但您可以通過流式傳輸立即使用它。通過在`load_dataset()` 中設置 `streaming=True` 來流式傳輸數據集，如下所示：

```python
from datasets import load_dataset

dataset = load_dataset('oscar-corpus/OSCAR-2201', 'en', split='train', streaming=True)

print(next(iter(dataset)))
```

結果:

```bash
{'id': 0, 'text': 'Founded in 2015, Golden Bees is a leading programmatic recruitment platform dedicated to employers, HR agencies and job boards. The company has developed unique HR-custom technologies and predictive algorithms to identify and attract the best candidates for a job opportunity.', ...
```

Dataset streaming 還允許您使用由本地文件組成的數據集，而無需進行任何轉換。在這種情況下，當您迭代數據集時，數據將從本地文件流式傳輸。這在以下情況下特別有用：

- 您不想等待非常大的本地數據集轉換為 Arrow。
- 轉換後的文件大小將超出計算機上的可用磁盤空間量。
- 您只想快速探索數據集的幾個樣本。

例如，您可以流式傳輸數百個壓縮 JSONL 文件（例如 [oscar-corpus/OSCAR-2201](https://huggingface.co/datasets/oscar-corpus/OSCAR-2201)）的本地數據集以立即使用它：

```python
from datasets import load_dataset

data_files = {'train': 'path/to/OSCAR-2201/compressed/en_meta/*.jsonl.gz'}

dataset = load_dataset('json', data_files=data_files, split='train', streaming=True)

print(next(iter(dataset)))
```

結果:

```bash
{'id': 0, 'text': 'Founded in 2015, Golden Bees is a leading programmatic recruitment platform dedicated to employers, HR agencies and job boards. The company has developed unique HR-custom technologies and predictive algorithms to identify and attract the best candidates for a job opportunity.', ...
```

以流模式加載數據集會創建一個新的數據集類型實例（而不是經典的 `Dataset` 物件），稱為 `IterableDataset`。這種特殊類型的數據集有自己的一套 processing data 的方法，如下所示。

!!! tip
    `IterableDataset` 對於迭代作業（例如訓練模型）非常有用。您不應該將 `IterableDataset` 用於需要隨機訪問示例的作業，因為您必須使用 `for loop` 對其進行迭代。獲取可迭代數據集中的最後一個示例需要迭代所有前面的示例。您可以在 [Dataset vs. IterableDataset](https://huggingface.co/docs/datasets/about_mapstyle_vs_iterable) 指南中找到更多詳細信息。

## Shuffle

與常規 `Dataset` 物件一樣，您也可以使用 `IterableDataset.shuffle()` 對 `IterableDataset` 進行數據洗牌。

`buffer_size` 參數控制從中隨機採樣示例的緩衝區的大小。假設您的數據集有一百萬個示例，並且您將 `buffer_size` 設置為一萬。 `IterableDataset.shuffle()` 將從緩衝區中的前一萬個示例中隨機選擇示例。緩衝區中選定的示例將被新示例替換。默認情況下，緩衝區大小為 `1,000`。

```python
from datasets import load_dataset

dataset = load_dataset('oscar', "unshuffled_deduplicated_en", split='train', streaming=True)

shuffled_dataset = dataset.shuffle(seed=42, buffer_size=10_000)
```

## Reshuffle

有時您可能想在每個 epoch 後重新排列數據集。這將要求您為每個時期設置不同的種子。在 epoch 之間使用 `IterableDataset.set_epoch()`` 來告訴數據集您所處的 epoch。

你的種子實際上變成：`initial seed + current epoch`

```python
for epoch in range(epochs):
    shuffled_dataset.set_epoch(epoch)
    for example in shuffled_dataset:
        ...

```

## Split dataset

您可以通過以下兩種方式之一分割數據集：

- `IterableDataset.take()` 返回數據集中的前 n 個示例：

    ```python
    dataset = load_dataset('oscar', "unshuffled_deduplicated_en", split='train', streaming=True)
    dataset_head = dataset.take(2)
    
    print(list(dataset_head))
    ```

    結果:

    ```bash
    [{'id': 0, 'text': 'Mtendere Village was...'}, {'id': 1, 'text': 'Lily James cannot fight the music...'}]
    ```

- `IterableDataset.skip()` 忽略數據集中的前 n 個示例並返回剩餘的示例：

    ```python
    train_dataset = shuffled_dataset.skip(1000)
    ```

## Interleave

`interleave_datasets()` 可以將 `IterableDataset` 與其他數據集組合起來。組合數據集返回每個原始數據集的交替示例。

```python
from datasets import interleave_datasets

en_dataset = load_dataset('oscar', "unshuffled_deduplicated_en", split='train', streaming=True)
fr_dataset = load_dataset('oscar', "unshuffled_deduplicated_fr", split='train', streaming=True)

multilingual_dataset = interleave_datasets([en_dataset, fr_dataset])

print(list(multilingual_dataset.take(2)))
```

結果:

```bash
[{'text': 'Mtendere Village was inspired by the vision...'}, {'text': "Média de débat d'idées, de culture et de littérature..."}]
```

定義每個原始數據集的採樣概率，以便更好地控制每個數據集的採樣和組合方式。使用所需的採樣概率設置概率參數：

```python
multilingual_dataset_with_oversampling = interleave_datasets([en_dataset, fr_dataset], probabilities=[0.8, 0.2], seed=42)

print(list(multilingual_dataset_with_oversampling.take(2)))
```

結果:

```bash
[{'text': 'Mtendere Village was inspired by the vision...'}, {'text': 'Lily James cannot fight the music...'}]
```

最終數據集的大約 `80%` 由 `en_dataset` 組成，`20%` 由 `fr_dataset` 組成。

您還可以指定 `stopping_strategy`。默認策略 `first_exhausted` 是二次採樣策略，即一旦其中一個數據集用完樣本，數據集構建就會停止。

您可以指定 `stopping_strategy=all_exhausted` 來執行過採樣策略。在這種情況下，一旦每個數據集中的每個樣本被添加至少一次，數據集構建就會停止。

實際上，這意味著如果數據集耗盡，它將返回到該數據集的開頭，直到達到停止標準。請注意，如果未指定採樣概率，則新數據集將具有 `max_length_datasets * nb_dataset` 樣本。

## Rename, remove, and cast

以下方法允許您修改數據集的列(column)。這些方法對於重命名或刪除列以及將列更改為一組新 feature 非常有用。

### Rename

當您需要重命名數據集中的列(column)時，請使用 `IterableDataset.rename_column()`。與原始列關聯的特徵實際上被移動到新列名稱下，而不是僅僅就地替換原始列。

為 `IterableDataset.rename_column()`` 提供原始列的名稱和新列的名稱：

```python
from datasets import load_dataset

dataset = load_dataset('mc4', 'en', streaming=True, split='train')

dataset = dataset.rename_column("text", "content")
```

### Remove

當您需要刪除一列或多列時，請為 `IterableDataset.remove_columns() 指定要刪除的列的名稱。通過提供列名稱列表來刪除多個列：

```python
from datasets import load_dataset

dataset = load_dataset('mc4', 'en', streaming=True, split='train')

dataset = dataset.remove_columns('timestamp')
```

### Cast

`IterableDataset.cast()` 更改一列或多列的特徵類型。此方法將您的新功能作為其參數。以下示例代碼展示瞭如何更改 `ClassLabel` 和 `Value` 的要素類型：

```python
from datasets import load_dataset

dataset = load_dataset('glue', 'mrpc', split='train', streaming=True)

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

new_features["label"] = ClassLabel(names=['negative', 'positive'])

new_features["idx"] = Value('int64')

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

!!! warn
    僅當原始要素類型和新要素類型兼容時，`cast` 才有效。例如，如果原始列僅包含 1 和 0，則可以將要素類型 `Value('int32')` 的列轉換為 `Value('bool')`。

使用 `IterableDataset.cast_column()` 更改僅一列的特徵類型。將列名稱及其新特徵類型作為參數傳遞：

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

## Map

與常規數據集的 `Dataset.map()` 函數類似，🤗 `Datasets` 具有用於處理 `IterableDataset` 的 `IterableDataset.map()` 功能。 `IterableDataset.map()` 在示例流式傳輸時應用即時處理。

它允許您獨立或批量地將處理函數應用於數據集中的每個示例。該函數甚至可以創建新的行和列。

以下示例演示瞭如何對 `IterableDataset` 進行 tokenize。該函數需要接受並輸出一個 dict：

```python
def add_prefix(example):
    example['text'] = 'My text: ' + example['text']
    return example
```

接下來，使用 `IterableDataset.map()` 將此函數應用於數據集：

```python
from datasets import load_dataset

dataset = load_dataset('oscar', 'unshuffled_deduplicated_en', streaming=True, split='train')

updated_dataset = dataset.map(add_prefix)

print(list(updated_dataset.take(3)))
```

結果:

```bash
[{'id': 0, 'text': 'My text: Mtendere Village was inspired by...'},
 {'id': 1, 'text': 'My text: Lily James cannot fight the music...'},
 {'id': 2, 'text': 'My text: "I\'d love to help kickstart...'}]
```

讓我們看另一個示例，只不過這次，您將使用 `IterableDataset.map()` 刪除一列。當您刪除列時，只有在將示例提供給映射函數後才會刪除該列。這允許映射函數在刪除列之前使用它們的內容。

使用 `IterableDataset.map()`` 中的 `remove_columns` 參數指定要刪除的列：

```python
updated_dataset = dataset.map(add_prefix, remove_columns=["id"])

print(list(updated_dataset.take(3)))
```

結果:

```bash
[{'text': 'My text: Mtendere Village was inspired by...'},
 {'text': 'My text: Lily James cannot fight the music...'},
 {'text': 'My text: "I\'d love to help kickstart...'}]
```

### Batch processing

`IterableDataset.map()` 還支持處理批量示例。通過設置 `batched=True` 來進行批量操作。默認批量大小為 `1000`，但您可以使用 `batch_size` 參數進行調整。這為許多有趣的應用打開了大門，例如 tokenization、將長句子分割成較短的塊以及數據增強。

#### Tokenization

```python
from datasets import load_dataset
from transformers import AutoTokenizer

dataset = load_dataset("mc4", "en", streaming=True, split="train")

tokenizer = AutoTokenizer.from_pretrained('distilbert-base-uncased')

def encode(examples):
    return tokenizer(examples['text'], truncation=True, padding='max_length')

dataset = dataset.map(encode, batched=True, remove_columns=["text", "timestamp", "url"])

print(next(iter(dataset)))
```

結果:

```bash
{'input_ids': 101, 8466, 1018, 1010, 4029, 2475, 2062, 18558, 3100, 2061, ...,1106, 3739, 102],
'attention_mask': [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, ..., 1, 1]}
```

!!! tip
    請參閱 [batched map processing](https://huggingface.co/docs/datasets/process#batch-processing) 文檔中的其他批處理示例。它們對於 `iterable datasets` 的工作原理相同。

### Filter

您可以使用 `Dataset.filter()` 根據謂詞函數過濾數據集中的行。它返回與指定條件匹配的行：

```python
from datasets import load_dataset

dataset = load_dataset('oscar', 'unshuffled_deduplicated_en', streaming=True, split='train')

start_with_ar = dataset.filter(lambda example: example['text'].startswith('Ar'))

print(next(iter(start_with_ar)))
```

結果:

```bash
{'id': 4, 'text': 'Are you looking for Number the Stars (Essential Modern Classics)?...'}
```

如果設置 `with_indices=True`，`Dataset.filter()` 還可以按索引過濾：

```python
even_dataset = dataset.filter(lambda example, idx: idx % 2 == 0, with_indices=True)

print(list(even_dataset.take(3)))
```

結果:

```bash
[{'id': 0, 'text': 'Mtendere Village was inspired by the vision of Chief Napoleon Dzombe, ...'},
 {'id': 2, 'text': '"I\'d love to help kickstart continued development! And 0 EUR/month...'},
 {'id': 4, 'text': 'Are you looking for Number the Stars (Essential Modern Classics)? Normally, ...'}]
```

## Stream in a training loop

`IterableDataset` 可以集成到訓練循環中。首先，對數據集進行洗牌：

!!! tip "Pytorch"

    ```python
    seed, buffer_size = 42, 10_000

    dataset = dataset.shuffle(seed, buffer_size=buffer_size)
    ```

    最後，創建一個簡單的訓練循環並開始訓練：

    ```python
    import torch
    from torch.utils.data import DataLoader
    from transformers import AutoModelForMaskedLM, DataCollatorForLanguageModeling
    from tqdm import tqdm

    dataset = dataset.with_format("torch")

    dataloader = DataLoader(dataset, collate_fn=DataCollatorForLanguageModeling(tokenizer))

    device = 'cuda' if torch.cuda.is_available() else 'cpu' 

    model = AutoModelForMaskedLM.from_pretrained("distilbert-base-uncased")

    model.train().to(device)

    optimizer = torch.optim.AdamW(params=model.parameters(), lr=1e-5)
    
    for epoch in range(3):
        dataset.set_epoch(epoch)
        for i, batch in enumerate(tqdm(dataloader, total=5)):
            if i == 5:
                break
            batch = {k: v.to(device) for k, v in batch.items()}
            outputs = model(**batch)
            loss = outputs[0]
            loss.backward()
            optimizer.step()
            optimizer.zero_grad()
            if i % 10 == 0:
                print(f"loss: {loss}")
    ```