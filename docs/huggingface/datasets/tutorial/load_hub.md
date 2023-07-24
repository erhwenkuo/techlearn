# Load a dataset from the Hub

尋找可重複且可訪問的高質量數據集可能很困難。 🤗 數據集的主要目標之一是提供一種簡單的方法來加載任何格式或類型的數據集。最簡單的入門方法是在 [Hugging Face Hub](https://huggingface.co/datasets) 上發現現有數據集（一個社區驅動的用於 NLP、計算機視覺和音頻任務的數據集集合），然後使用 🤗 數據集下載和生成數據集。

本教程使用 [rotten_tomatoes](https://huggingface.co/datasets/rotten_tomatoes) 和 [MInDS-14](https://huggingface.co/datasets/PolyAI/minds14) 數據集，但您可以隨意加載您想要的任何數據集並繼續操作。現在前往 Hub 並找到適合您任務的數據集！

## Load a dataset

在花時間下載數據集之前，快速獲取有關數據集的一些常用信息通常會很有幫助。數據集的信息存儲在 [DatasetInfo](https://huggingface.co/docs/datasets/v2.13.1/en/package_reference/main_classes#datasets.DatasetInfo) 中，可以包括數據集描述、特徵和數據集大小等信息。

使用 `load_dataset_builder()` 函數加載數據集構建器並檢查數據集的屬性，而無需真實地下載它：

```python
from datasets import load_dataset_builder

ds_builder = load_dataset_builder("rotten_tomatoes")

# Inspect dataset description
print(ds_builder.info.description)

# Inspect dataset features
print(ds_builder.info.features)
```

結果:

```bash
Movie Review Dataset. This is a dataset of containing 5,331 positive and 5,331 negative processed sentences from Rotten Tomatoes movie reviews. This data was first used in Bo Pang and Lillian Lee, ``Seeing stars: Exploiting class relationships for sentiment categorization with respect to rating scales.'', Proceedings of the ACL, 2005.
```

```bash
{'label': ClassLabel(num_classes=2, names=['neg', 'pos'], id=None),
 'text': Value(dtype='string', id=None)}
```

如果您對數據集感到滿意，請使用 `load_dataset()` 加載它：

```python
from datasets import load_dataset

dataset = load_dataset("rotten_tomatoes", split="train")
```

## Splits

Split 是數據集的特定子集，例如 `train` 和 `test`。使用 `get_dataset_split_names()` 函數列出數據集的擁有的 split 名稱：

```python
from datasets import get_dataset_split_names

get_dataset_split_names("rotten_tomatoes")
```

結果:

```bash
['train', 'validation', 'test']
```

然後您可以使用 `split` 參數加載特定的分割。加載數據集分割會返回一個 `Dataset` 物件：

```python
from datasets import load_dataset

dataset = load_dataset("rotten_tomatoes", split="train")

print(dataset)
```

結果:

```bash
Dataset({
    features: ['text', 'label'],
    num_rows: 8530
})
```

如果您不指定分割，🤗 Datasets 將返回一個 `DatasetDict` 物件：

```python
from datasets import load_dataset

dataset = load_dataset("rotten_tomatoes")

print(dataset)
```

結果:

```bash
DatasetDict({
    train: Dataset({
        features: ['text', 'label'],
        num_rows: 8530
    })
    validation: Dataset({
        features: ['text', 'label'],
        num_rows: 1066
    })
    test: Dataset({
        features: ['text', 'label'],
        num_rows: 1066
    })
})
```

## Configurations

一些數據集包含多個子數據集。例如，[MINDS-14](https://huggingface.co/datasets/PolyAI/minds14) 數據集有多個子數據集，每個子數據集都包含不同語言的音頻數據。這些子數據集稱為 `configurations`，您在加載數據集時必須明確選擇一個。如果您不提供配置名稱，🤗 Datasets 將引發 ValueError 並提醒您選擇配置。

使用 `get_dataset_config_names()` 函數檢索數據集可用的所有可能配置的列表：

```python
from datasets import get_dataset_config_names

configs = get_dataset_config_names("PolyAI/minds14")

print(configs)
```

結果:

```bash
['cs-CZ', 'de-DE', 'en-AU', 'en-GB', 'en-US', 'es-ES', 'fr-FR', 'it-IT', 'ko-KR', 'nl-NL', 'pl-PL', 'pt-PT', 'ru-RU', 'zh-CN', 'all']
```

然後加載你想要的配置：

```python
from datasets import load_dataset

mindsFR = load_dataset("PolyAI/minds14", "fr-FR", split="train")
```
