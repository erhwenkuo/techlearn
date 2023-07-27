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

## Map

🤗 數據集的一些更強大的應用來自於使用 [`map()`](https://huggingface.co/docs/datasets/v2.14.0/en/package_reference/main_classes#datasets.Dataset.map) 函數。 `map()` 的主要目的是加速數據處理轉換的函數。它允許您獨立或批量地將處理函數應用於數據集中的每個示例。該函數甚至可以創建新的行和列。

在以下示例中，在數據集中的每個 Sentence1 值前面添加 `My Sentence:` 前綴。

首先創建一個函數，將 'My sentence: ' 添加到每個 `sentence1` 的開頭。該函數需要接受並輸出一個 `dict`：

```python
def add_prefix(example):
    example["sentence1"] = 'My sentence: ' + example["sentence1"]
    return example
```

現在使用 `map()` 將 `add_prefix` 函數應用到整個數據集：

```python
updated_dataset = small_dataset.map(add_prefix)

print(updated_dataset["sentence1"][:5])
```

結果:

```bash
['My sentence: Amrozi accused his brother , whom he called " the witness " , of deliberately distorting his evidence .',
"My sentence: Yucaipa owned Dominick 's before selling the chain to Safeway in 1998 for $ 2.5 billion .",
'My sentence: They had published an advertisement on the Internet on June 10 , offering the cargo for sale , he added .',
'My sentence: Around 0335 GMT , Tab shares were up 19 cents , or 4.4 % , at A $ 4.56 , having earlier set a record high of A $ 4.57 .',
]
```

讓我們看另一個示例，只不過這次您將使用 `map()` 刪除一列(column)。當您刪除列時，只有在將示例提供給映射函數後才會刪除該列。這允許映射函數在刪除列之前使用它們的內容。

使用 `map()` 中的 `remove_columns` 參數指定要刪除的列：

```python
updated_dataset = dataset.map(lambda example: {"new_sentence": example["sentence1"]}, remove_columns=["sentence1"])

print(updated_dataset.column_names)
```

結果:

```bash
['sentence2', 'label', 'idx', 'new_sentence']
```

!!! tip
    🤗 數據集還有一個 `remove_columns()` 函數，該函數速度更快，因為它不會復制剩餘列的數據。

如果設置 `with_indices=True`，您還可以將 `map()` 與索引一起使用。下面的示例將索引添加到每個句子的開頭：

```python
updated_dataset = dataset.map(lambda example, idx: {"sentence2": f"{idx}: " + example["sentence2"]}, with_indices=True)

updated_dataset["sentence2"][:5]
```

結果:

```bash
['0: Referring to him as only " the witness " , Amrozi accused his brother of deliberately distorting his evidence .',
 "1: Yucaipa bought Dominick 's in 1995 for $ 693 million and sold it to Safeway for $ 1.8 billion in 1998 .",
 "2: On June 10 , the ship 's owners had published an advertisement on the Internet , offering the explosives for sale .",
 '3: Tab shares jumped 20 cents , or 4.6 % , to set a record closing high at A $ 4.57 .',
 '4: PG & E Corp. shares jumped $ 1.63 or 8 percent to $ 21.03 on the New York Stock Exchange on Friday .'
]
```

如果設置 `with_rank=True`，`map()` 還可以處理進程的等級。這類似於 `with_indices` 參數。如果映射函數中的 `with_rank` 參數已經存在，則該參數位於索引 one 之後。

```python
from multiprocess import set_start_method
import torch
import os

set_start_method("spawn")

def gpu_computation(example, rank):
    os.environ["CUDA_VISIBLE_DEVICES"] = str(rank % torch.cuda.device_count())
    # Your big GPU call goes here
    return examples

updated_dataset = dataset.map(gpu_computation, with_rank=True)
```

Rank 的主要用例是跨多個 GPU 並行計算。這需要設置 `multiprocess.set_start_method("spawn")`。如果不這樣做，您將收到以下 CUDA 錯誤：

```bash
RuntimeError: Cannot re-initialize CUDA in forked subprocess. To use CUDA with multiprocessing, you must use the 'spawn' start method.
```

### Multiprocessing

Multiprocessing 通過 CPU 上的並行進程顯著加快處理速度。在 `map()` 中設置 `num_proc` 參數來設置要使用的進程數：

```python
updated_dataset = dataset.map(lambda example, idx: {"sentence2": f"{idx}: " + example["sentence2"]}, num_proc=4)
```

### Batch processing

`map()` 函數支持處理 {==批量==} 示例。通過設置 `batched=True` 來進行批量操作。默認批量大小為 `1000`，但您可以使用 `batch_size` 參數進行調整。批量處理可以實現有趣的應用，例如將長句子分割成較短的塊和數據增強。

####　Split long examples

當 example 數據太長時，您可能需要將它們分成幾個較小的 chunk。首先創建一個函數：

1. 將 Sentence1 欄位拆分為 50 個字符的塊。
2. 將所有 chunks 堆疊在一起以創建新的數據集。

```python
def chunk_examples(examples):
    chunks = []
    for sentence in examples["sentence1"]:
        chunks += [sentence[i:i + 50] for i in range(0, len(sentence), 50)]
    return {"chunks": chunks}
```

使用 `map()` 應用該函數：

```python
chunked_dataset = dataset.map(chunk_examples, batched=True, remove_columns=dataset.column_names)

chunked_dataset[:10]
```

結果:

```bash
{'chunks': ['Amrozi accused his brother , whom he called " the ',
            'witness " , of deliberately distorting his evidenc',
            'e .',
            "Yucaipa owned Dominick 's before selling the chain",
            ' to Safeway in 1998 for $ 2.5 billion .',
            'They had published an advertisement on the Interne',
            't on June 10 , offering the cargo for sale , he ad',
            'ded .',
            'Around 0335 GMT , Tab shares were up 19 cents , or',
            ' 4.4 % , at A $ 4.56 , having earlier set a record']}
```

請注意現在句子是如何分割成較短的塊的，並且數據集中有更多的行。

```python
print(dataset)
```

結果:

```bash
Dataset({
 features: ['sentence1', 'sentence2', 'label', 'idx'],
 num_rows: 3668
})
```

```python
print(chunked_dataset)
```

結果:

```bash
Dataset(schema: {'chunks': 'string'}, num_rows: 10470)
```

#### Data augmentation

`map()` 函數也可用於數據增強。以下示例為句子中的屏蔽標記生成附加單詞。

在 🤗 Transformers 的 [FillMaskPipeline](https://huggingface.co/transformers/main_classes/pipelines#transformers.FillMaskPipeline) 中加載並使用 [RoBERTA](https://huggingface.co/roberta-base) 模型：

```python
from random import randint
from transformers import pipeline

fillmask = pipeline("fill-mask", model="roberta-base")

mask_token = fillmask.tokenizer.mask_token

smaller_dataset = dataset.filter(lambda e, i: i<100, with_indices=True)
```

創建一個函數來隨機選擇要在句子中屏蔽的單詞。該函數還應返回原始句子和 RoBERTA 生成的前兩個替換句子。

```python
def augment_data(examples):
    outputs = []
    for sentence in examples["sentence1"]:
        words = sentence.split(' ')
        K = randint(1, len(words)-1)
        masked_sentence = " ".join(words[:K]  + [mask_token] + words[K+1:])
        predictions = fillmask(masked_sentence)
        augmented_sequences = [predictions[i]["sequence"] for i in range(3)]
        outputs += [sentence] + augmented_sequences
    return {"data": outputs}
```

使用 `map()` 將函數應用於整個數據集：

```python
augmented_dataset = smaller_dataset.map(augment_data, batched=True, remove_columns=dataset.column_names, batch_size=8)

augmented_dataset[:9]["data"]
```

結果:

```bash
['Amrozi accused his brother , whom he called " the witness " , of deliberately distorting his evidence .',
 'Amrozi accused his brother, whom he called " the witness ", of deliberately withholding his evidence.',
 'Amrozi accused his brother, whom he called " the witness ", of deliberately suppressing his evidence.',
 'Amrozi accused his brother, whom he called " the witness ", of deliberately destroying his evidence.',
 "Yucaipa owned Dominick 's before selling the chain to Safeway in 1998 for $ 2.5 billion .",
 'Yucaipa owned Dominick Stores before selling the chain to Safeway in 1998 for $ 2.5 billion.',
 "Yucaipa owned Dominick's before selling the chain to Safeway in 1998 for $ 2.5 billion.",
 'Yucaipa owned Dominick Pizza before selling the chain to Safeway in 1998 for $ 2.5 billion.'
]
```

### Process multiple splits

許多數據集都有可以使用 `DatasetDict.map()` 同時處理的分割(split)。例如，將 `train` 和 `test` 中的 Sentence1 欄位進行 tokenizing：

```python
from datasets import load_dataset

# load all the splits
dataset = load_dataset('glue', 'mrpc')

encoded_dataset = dataset.map(lambda examples: tokenizer(examples["sentence1"]), batched=True)

encoded_dataset["train"][0]
```

結果:

```bash
{'sentence1': 'Amrozi accused his brother , whom he called " the witness " , of deliberately distorting his evidence .',
'sentence2': 'Referring to him as only " the witness " , Amrozi accused his brother of deliberately distorting his evidence .',
'label': 1,
'idx': 0,
'input_ids': [  101,  7277,  2180,  5303,  4806,  1117,  1711,   117,  2292, 1119,  1270,   107,  1103,  7737,   107,   117,  1104,  9938, 4267, 12223, 21811,  1117,  2554,   119,   102],
'token_type_ids': [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0],
'attention_mask': [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
}
```

### Distributed usage

當你在分佈式環境中使用 `map()` 時，你還應該使用 `torch.distributed.barrier`。這確保主進程執行映射，而其他進程加載結果，從而避免重複工作。

以下示例展示瞭如何使用 `torch.distributed.barrier` 來同步進程：

```python
from datasets import Dataset
import torch.distributed

dataset1 = Dataset.from_dict({"a": [0, 1, 2]})

if training_args.local_rank > 0:
    print("Waiting for main process to perform the mapping")
    torch.distributed.barrier()

dataset2 = dataset1.map(lambda x: {"a": x["a"] + 1})

if training_args.local_rank == 0:
    print("Loading results from main process")
    torch.distributed.barrier()
```

## Concatenate

如果單獨的數據集共享相同的列類型(column type)，則可以將它們連接起來。使用 `concatenate_datasets()` 連接數據集：

```python
from datasets import concatenate_datasets, load_dataset

bookcorpus = load_dataset("bookcorpus", split="train")

wiki = load_dataset("wikipedia", "20220301.en", split="train")

wiki = wiki.remove_columns([col for col in wiki.column_names if col != "text"])  # only keep the 'text' column

assert bookcorpus.features.type == wiki.features.type

bert_dataset = concatenate_datasets([bookcorpus, wiki])
```

您還可以通過設置 `axis=1` 水平連接兩個數據集，只要數據集具有相同的行數即可：

```python
from datasets import Dataset

bookcorpus_ids = Dataset.from_dict({"ids": list(range(len(bookcorpus)))})

bookcorpus_with_ids = concatenate_datasets([bookcorpus, bookcorpus_ids], axis=1)
```

### Interleave

您還可以通過交替使用每個數據集的示例來將多個數據集混合在一起，以創建一個新的數據集。這稱為交錯(interleaving)，由 `interleave_datasets()` 函數來驅動。 `interleave_datasets()` 和 `concatenate_datasets()` 都適用於常規 `Dataset` 和 `IterableDataset` 物件。有關如何交錯 `IterableDataset` 物件的示例，請參閱 [Stream](https://huggingface.co/docs/datasets/stream#interleave) 指南。

您可以為每個原始數據集定義採樣概率，以指定如何交錯數據集。在這種情況下，新數據集是通過從隨機數據集中逐一獲取示例來構建的，直到其中一個數據集用完樣本為止。

```python
seed = 42
probabilities = [0.3, 0.5, 0.2]

d1 = Dataset.from_dict({"a": [0, 1, 2]})
d2 = Dataset.from_dict({"a": [10, 11, 12, 13]})
d3 = Dataset.from_dict({"a": [20, 21, 22]})

dataset = interleave_datasets([d1, d2, d3], probabilities=probabilities, seed=seed)

print(dataset["a"])
```

結果:

```bash
[10, 11, 20, 12, 0, 21, 13]
```

您還可以指定 `stopping_strategy`。默認策略 `first_exhausted` 是二次採樣策略，即一旦其中一個數據集用完樣本，數據集構建就會停止。

您可以指定 `stopping_strategy=all_exhausted` 來執行過採樣策略。在這種情況下，一旦每個數據集中的每個樣本被添加至少一次，數據集構建就會停止。

實際上，這意味著如果數據集耗盡，它將返回到該數據集的開頭，直到達到停止標準。請注意，如果未指定採樣概率，則新數據集將具有 `max_length_datasets*nb_dataset` 樣本。

```python
d1 = Dataset.from_dict({"a": [0, 1, 2]})
d2 = Dataset.from_dict({"a": [10, 11, 12, 13]})
d3 = Dataset.from_dict({"a": [20, 21, 22]})

dataset = interleave_datasets([d1, d2, d3], stopping_strategy="all_exhausted")

print(dataset["a"])
```

結果:

```bash
[0, 10, 20, 1, 11, 21, 2, 12, 22, 0, 13, 20]
```

## Format

`set_format()` 函數更改列(column)的格式以與一些常見的數據格式兼容。在類型參數中指定您想要的輸出以及要設置格式的列。格式化是即時發生的。

例如，通過設置 `type="torch"` 創建 `PyTorch tensor`：

```python
import torch

dataset.set_format(type="torch", columns=["input_ids", "token_type_ids", "attention_mask", "label"])
```

`with_format()` 函數還可以更改列的格式，但它返回一個新的 Dataset 物件：

```python
dataset = dataset.with_format(type="torch", columns=["input_ids", "token_type_ids", "attention_mask", "label"])
```

!!! tip
    🤗 Datasets 還提供對其他常見數據格式的支持，例如 `NumPy`、`Pandas` 和 `JAX`。查看 [Using Datasets with TensorFlow](https://huggingface.co/docs/datasets/master/en/use_with_tensorflow#using-totfdataset) 指南，了解有關如何高效創建 TensorFlow 數據集的更多詳細信息。

如果需要將數據集重置為其原始格式，請使用 `reset_format()` 函數：

```python
print(dataset.format)
```

結果:

```bash
{'type': 'torch', 'format_kwargs': {}, 'columns': ['label'], 'output_all_columns': False}
```

```python
dataset.reset_format()
```

```python
dataset.format
```

結果:

```bash
{'type': 'python', 'format_kwargs': {}, 'columns': ['idx', 'label', 'sentence1', 'sentence2'], 'output_all_columns': False}
```

### Format transform

`set_transform()` 函數即時應用自定義格式轉換。此函數替換任何先前指定的格式。例如，您可以使用此函數去 tokenize 和 pad tokens。僅在訪問示例時才應用 Tokenization：

```python
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")

def encode(batch):
    return tokenizer(batch["sentence1"], padding="longest", truncation=True, max_length=512, return_tensors="pt")

dataset.set_transform(encode)

print(dataset.format)
```

結果:

```bash
{'type': 'custom', 'format_kwargs': {'transform': <function __main__.encode(batch)>}, 'columns': ['idx', 'label', 'sentence1', 'sentence2'], 'output_all_columns': False}
```

您還可以使用 `set_transform()` 函數來解碼功能不支持的格式。例如，音頻功能使用聲音文件（一個安裝快速且簡單的庫），但它不提供對不太常見的音頻格式的支持。您可以在此處使用 `set_transform()`` 動態應用自定義解碼轉換。您可以隨意使用任何您喜歡的庫來解碼音頻文件。

下面的示例使用 [pydub](http://pydub.com/) 套件打開 `soundfile` 不支持的音頻格式：

```python
import numpy as np
from pydub import AudioSegment

audio_dataset_amr = Dataset.from_dict({"audio": ["audio_samples/audio.amr"]})

def decode_audio_with_pydub(batch, sampling_rate=16_000):
    def pydub_decode_file(audio_path):
        sound = AudioSegment.from_file(audio_path)
        if sound.frame_rate != sampling_rate:
            sound = sound.set_frame_rate(sampling_rate)
        channel_sounds = sound.split_to_mono()
        samples = [s.get_array_of_samples() for s in channel_sounds]
        fp_arr = np.array(samples).T.astype(np.float32)
        fp_arr /= np.iinfo(samples[0].typecode).max
        return fp_arr
    batch["audio"] = [pydub_decode_file(audio_path) for audio_path in batch["audio"]]
    return batch

audio_dataset_amr.set_transform(decode_audio_with_pydub)
```

## Save

處理完數據集後，您可以使用 [`save_to_disk()`](https://huggingface.co/docs/datasets/v2.14.0/en/package_reference/main_classes#datasets.Dataset.save_to_disk) 保存並稍後重複使用它。

通過提供您希望將其保存到的目錄的路徑來保存數據集：

```python
encoded_dataset.save_to_disk("path/of/my/dataset/directory")
```

使用 `load_from_disk()` 函數重新加載數據集：

```python
from datasets import load_from_disk

reloaded_dataset = load_from_disk("path/of/my/dataset/directory")
```

!!! tip
    想要將數據集保存到雲存儲空間嗎？閱讀我們的 [Cloud Storage](https://huggingface.co/docs/datasets/filesystems) 指南，了解如何將數據集保存到 AWS(S3) 或 Google 雲存儲。

## Export

🤗 數據集也支持導出，因此您可以在其他應用程序中使用您的數據集。下表顯示了當前支持的導出文件格式：

|File type	|Export method|
|-----------|-------------|
|`CSV`	|[`Dataset.to_csv()`](https://huggingface.co/docs/datasets/v2.14.0/en/package_reference/main_classes#datasets.Dataset.to_csv)|
|`JSON`	|[`Dataset.to_json()`](https://huggingface.co/docs/datasets/v2.14.0/en/package_reference/main_classes#datasets.Dataset.to_json)|
|`Parquet`	|[`Dataset.to_parquet()`](https://huggingface.co/docs/datasets/v2.14.0/en/package_reference/main_classes#datasets.Dataset.to_parquet)|
|`SQL`	|[`Dataset.to_sql()`](https://huggingface.co/docs/datasets/v2.14.0/en/package_reference/main_classes#datasets.Dataset.to_sql)|
|`In-memory` |Python object	[`Dataset.to_pandas()`](https://huggingface.co/docs/datasets/v2.14.0/en/package_reference/main_classes#datasets.Dataset.to_pandas) or [`Dataset.to_dict()`](https://huggingface.co/docs/datasets/v2.14.0/en/package_reference/main_classes#datasets.Dataset.to_dict)|

例如，將數據集導出到 CSV 文件，如下所示：

```python
encoded_dataset.to_csv("path/of/my/dataset.csv")
```