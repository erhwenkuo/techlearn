# Load

您的數據可以存儲在不同的地方；它們可以位於本地計算機的磁盤上、Github 存儲庫中以及內存中的數據結構（例如 Python 字典和 Pandas DataFrames）中。無論數據集存儲在何處，🤗 數據集都可以幫助您加載它。

本指南將向您展示如何從以下位置加載數據集：

- The Hub without a dataset loading script
- Local loading script
- Local files
- In-memory data
- Offline
- A specific slice of a split

有關加載其他數據集模式的更多詳細信息，請查看 [load audio dataset guide](https://huggingface.co/docs/datasets/audio_load)、[load image dataset guide](https://huggingface.co/docs/datasets/image_load) 或 [load text dataset guide](https://huggingface.co/docs/datasets/nlp_load)。

## Hugging Face Hub

數據集是從下載的 `dataset loading script` 所生成的。但是，您也可以從 Hub 上的任何數據集存儲庫加載數據集，而無需加載腳本！

首先在 Huggingface 上創建 dataset repo 數據集存儲庫並上傳數據文件。接著您就可以使用 `load_dataset()` 函數加載數據集。

例如，嘗試通過提供存儲庫命名空間和數據集名稱([demo repository](https://huggingface.co/datasets/lhoestq/demo1))來加載數據。該數據集存儲庫包含 CSV 文件，下面的程式碼從 CSV 文件加載數據集：

```python
from datasets import load_dataset

dataset = load_dataset("lhoestq/demo1")

print(dataset)
```

某些數據集可能有多個基於 Git 標籤、分支或提交的版本。使用 `revision` 參數指定要加載的數據集版本：

```python
dataset = load_dataset(
  "lhoestq/custom_squad",
  revision="main"  # tag name, or branch name, or commit hash
)
```

默認情況下，沒有 `dataset loading script` 的數據集會將所有數據加載到 `train` 的　`split` 中。使用 `data_files` 參數將數據文件映射到 `train`、`validation` 和 `test`　的 splits：

```python
data_files = {"train": "train.csv", "test": "test.csv"}

dataset = load_dataset("namespace/your_dataset_name", data_files=data_files)
```

!!! tip
    如果不指定使用哪些數據文件，`load_dataset()` 將返回所有數據文件。如果加載像 C4 這樣的大型數據集（大約有 13TB 的數據），這可能需要很長時間。

您還可以使用 `data_files` 或 `data_dir` 參數加載特定的文件子集。這些參數可以接受相對路徑，該相對路徑解析為與數據集加載位置相對應的基本路徑。

```python
from datasets import load_dataset

# load files that match the grep pattern
c4_subset = load_dataset("allenai/c4", data_files="en/c4-train.0000*-of-01024.json.gz")

# load dataset from the en directory on the Hub
c4_subset = load_dataset("allenai/c4", data_dir="en")
```

`split` 參數還可以將數據文件映射到特定的 `split`：

```python
data_files = {"validation": "en/c4-validation.*.json.gz"}

c4_validation = load_dataset("allenai/c4", data_files=data_files, split="validation")
```

## Local loading script

您的本地計算機上可能有一個本地 `datasets loading script`。在這種情況下，通過將以下路徑之一傳遞給 `load_dataset()` 來加載數據集：

- `datasets loading script` 的本地路徑。
- 包含 `datasets loading script` 的目錄的本地路徑（僅當腳本文件與目錄同名時）。

```python
dataset = load_dataset("path/to/local/loading_script/loading_script.py", split="train")

dataset = load_dataset("path/to/local/loading_script", split="train")  # equivalent because the file has the same name as the directory
```

### Edit loading script

您還可以從 Hub 編輯加載腳本以添加您自己的修改。將數據集存儲庫下載到本地，以便可以加載加載腳本中相對路徑引用的任何數據文件：

```python
git clone https://huggingface.co/datasets/eli5
```

對 `datasets loading script` 進行編輯，然後通過將其本地路徑傳遞給 `load_dataset()` 來加載它：

```python
from datasets import load_dataset

eli5 = load_dataset("path/to/local/eli5")
```

## Local and remote files

可以從計算機上存儲的本地文件和遠程文件加載數據集。數據集很可能存儲為 `csv`、`json`、`txt` 或 `parquet` 文件。 `load_dataset()` 函數可以加載每種文件類型。

### CSV

🤗 數據集可以讀取由一個或多個 CSV 文件組成的數據集（在本例中，將 CSV 文件作為列表傳遞）：

```python
from datasets import load_dataset

dataset = load_dataset("csv", data_files="my_file.csv")
```

要通過 `S3(AWS/Minio)` 加載遠程 CSV 文件，請設定 `storage_options` 物件：

```python
storage_options = {
        "anon": False,  # for anonymous connection
        "key": "***********",  # access_key
        "secret": "***********",  # secret_key
        "use_ssl": False,  # use https or http
        "client_kwargs": {
            "endpoint_url": "http://localhost:9000"  # minio endpoint
        }
    }


data_files="s3://my_bucket/imdb/titanic/train.csv"

dataset = load_dataset("csv", data_files=data_files, storage_options=storage_options)
```

!!! tip
    有關更多詳細信息，請查看 [how to load tabular datasets from CSV files ](https://huggingface.co/docs/datasets/tabular_load#csv-files) 指南。

### JSON

JSON 文件直接使用 `load_dataset()` 加載，如下所示：

```python
from datasets import load_dataset

dataset = load_dataset("json", data_files="my_file.json")
```

JSON 文件有多種格式，但我們認為最有效的格式是擁有多個 JSON 對象；每行代表一行數據。例如：

```json
{"a": 1, "b": 2.0, "c": "foo", "d": false}
{"a": 4, "b": -5.5, "c": null, "d": true}
```

您可能遇到的另一種 JSON 格式是嵌套字段，在這種情況下，您需要指定欄位參數，如下所示：

```json
{"version": "0.1.0",
 "data": [{"a": 1, "b": 2.0, "c": "foo", "d": false},
          {"a": 4, "b": -5.5, "c": null, "d": true}]
}
```

```python
from datasets import load_dataset

dataset = load_dataset("json", data_files="my_file.json", field="data")
```

要通過 HTTP 加載遠程 JSON 文件，請傳遞 URL：

```python
base_url = "https://rajpurkar.github.io/SQuAD-explorer/dataset/"

dataset = load_dataset("json", data_files={"train": base_url + "train-v1.1.json", "validation": base_url + "dev-v1.1.json"}, field="data")
```

要通過 `S3(AWS/Minio)` 加載遠程 JSON 文件，請設定 `storage_options` 物件：

```python
storage_options = {
        "anon": False,  # for anonymous connection
        "key": "***********",  # access_key
        "secret": "***********",  # secret_key
        "use_ssl": False,  # use https or http
        "client_kwargs": {
            "endpoint_url": "http://localhost:9000"  # minio endpoint
        }
    }


base_url="s3://my_bucket/squad/"

dataset = load_dataset("json", 
  data_files={"train": base_url + "train-v1.1.json", "validation": base_url + "dev-v1.1.json"}, 
  field="data", 
  storage_options=storage_options)
```

雖然這些是最常見的 JSON 格式，但您會看到其他格式不同的數據集。 🤗 數據集可識別這些其他格式，並將相應地回退到 Python JSON 加載方法來處理它們。

### Parquet

`Parquet` 文件以列格式存儲，與 `CSV` 等基於行的文件不同。大型數據集可以存儲在 `Parquet` 文件中，因為它在返回查詢時更高效、更快速。

要加載 `Parquet` 文件：

```python
from datasets import load_dataset

dataset = load_dataset("parquet", data_files={'train': 'train.parquet', 'test': 'test.parquet'})
```

要通過 HTTP 加載遠程 Parquet 文件，請傳遞 URL：

```python
base_url = "https://storage.googleapis.com/huggingface-nlp/cache/datasets/wikipedia/20200501.en/1.0.0/"

data_files = {"train": base_url + "wikipedia-train.parquet"}

wiki = load_dataset("parquet", data_files=data_files, split="train")
```

要通過 `S3(AWS/Minio)` 加載遠程 JSON 文件，請設定 `storage_options` 物件：

```python
storage_options = {
        "anon": False,  # for anonymous connection
        "key": "***********",  # access_key
        "secret": "***********",  # secret_key
        "use_ssl": False,  # use https or http
        "client_kwargs": {
            "endpoint_url": "http://localhost:9000"  # minio endpoint
        }
    }


base_url="s3://my_bucket/datasets/wikipedia/"

data_files = {"train": base_url + "wikipedia-train.parquet"}

wiki = load_dataset(
  "parquet", 
  data_files={"train": base_url + "wikipedia-train.parquet"},
  storage_options=storage_options)
```


### Arrow

與 `CSV` 等基於行的格式和 `Parquet` 等未壓縮格式不同，`Arrow` 文件以內存中的列格式存儲。

要加載 `Arrow` 文件：

```python
from datasets import load_dataset

dataset = load_dataset("arrow", data_files={'train': 'train.arrow', 'test': 'test.arrow'})
```

要通過 HTTP 加載遠程 Arrow 文件，請傳遞 URL：

```python
base_url = "https://storage.googleapis.com/huggingface-nlp/cache/datasets/wikipedia/20200501.en/1.0.0/"

data_files = {"train": base_url + "wikipedia-train.arrow"}

wiki = load_dataset("arrow", data_files=data_files, split="train")
```

要通過 `S3(AWS/Minio)` 加載遠程 JSON 文件，請設定 `storage_options` 物件：

```python
storage_options = {
        "anon": False,  # for anonymous connection
        "key": "***********",  # access_key
        "secret": "***********",  # secret_key
        "use_ssl": False,  # use https or http
        "client_kwargs": {
            "endpoint_url": "http://localhost:9000"  # minio endpoint
        }
    }


base_url="s3://my_bucket/datasets/wikipedia/"

data_files = {"train": base_url + "wikipedia-train.arrow"}

wiki = load_dataset(
  "arrow", 
  data_files={"train": base_url + "wikipedia-train.parquet"},
  storage_options=storage_options)
```

`Arrow` 是 🤗 Datasets 在底層使用的文件格式，因此您可以直接使用 `Dataset.from_file()` 加載本地 `Arrow` 文件：

```python
from datasets import Dataset

dataset = Dataset.from_file("data.arrow")
```

與 `load_dataset()` 不同，`Dataset.from_file()` 內存映射 `Arrow` 文件，無需在緩存中準備數據集，從而節省磁盤空間。在這種情況下，存儲中間處理結果的緩存目錄將是 Arrow 文件目錄。

目前僅支持 Arrow streaming 格式。不支持 Arrow IPC 文件格式（也稱為 Feather V2）。

### SQL

通過指定連接到數據庫的 URI，使用 `from_sql()` 讀取數據庫內容。您可以讀取表名和查詢：

```python
from datasets import Dataset

dataset = Dataset.from_sql("data_table_name", con="sqlite:///sqlite_file.db")

dataset = Dataset.from_sql("SELECT text FROM table WHERE length(text) > 100 LIMIT 10", con="sqlite:///sqlite_file.db")
```

!!! tip
    有關更多詳細信息，請查看 [how to load tabular datasets from SQL databases](https://huggingface.co/docs/datasets/tabular_load#databases) 指南。

## Multiprocessing

當數據集由多個文件（我們稱為“分片”）組成時，可以顯著加快數據集下載和準備步驟。

您可以使用 `num_proc` 選擇要使用多少個進程來並行準備數據集。在這種情況下，每個進程都會獲得一個要準備的分片子集：

```python
from datasets import load_dataset

oscar_afrikaans = load_dataset("oscar-corpus/OSCAR-2201", "af", num_proc=8)

imagenet = load_dataset("imagenet-1k", num_proc=8)

ml_librispeech_spanish = load_dataset("facebook/multilingual_librispeech", "spanish", num_proc=8)
```

## In-memory data

🤗 數據集還允許您直接從內存數據結構（如 Python 字典和 Pandas DataFrames）創建數據集。

### Python dictionary

使用 `from_dict()` 加載 Python 字典：

```python
from datasets import Dataset

my_dict = {"a": [1, 2, 3]}

dataset = Dataset.from_dict(my_dict)
```

### Python list of dictionaries

使用 `from_list()` 加載 Python 字典列表：

```python
from datasets import Dataset

my_list = [{"a": 1}, {"a": 2}, {"a": 3}]

dataset = Dataset.from_list(my_list)
```

### Python generator

使用 `from_generator()` 從 Python 生成器創建數據集：

```python
from datasets import Dataset

def my_gen():
    for i in range(1, 4):
        yield {"a": i}

dataset = Dataset.from_generator(my_gen)
```

此方法支持加載大於可用內存的數據。

您還可以通過將列表傳遞給 `gen_kwargs 來定義分片數據集：

```python
def gen(shards):
    for shard in shards:
        with open(shard) as f:
            for line in f:
                yield {"line": line}

shards = [f"data{i}.txt" for i in range(32)]
ds = IterableDataset.from_generator(gen, gen_kwargs={"shards": shards})
ds = ds.shuffle(seed=42, buffer_size=10_000)  # shuffles the shards order + uses a shuffle buffer

from torch.utils.data import DataLoader

dataloader = DataLoader(ds.with_format("torch"), num_workers=4)  # give each worker a subset of 32/4=8 shards
```

### Pandas DataFrame

使用 `from_pandas()` 加載 Pandas DataFrame：

```python
from datasets import Dataset
import pandas as pd

df = pd.DataFrame({"a": [1, 2, 3]})

dataset = Dataset.from_pandas(df)
```

!!! tip
    有關更多詳細信息，請查看 [how to load tabular datasets from Pandas DataFrames ](https://huggingface.co/docs/datasets/tabular_load#pandas-dataframes) 指南。

## Offline

即使您沒有互聯網連接，仍然可以加載數據集。只要您之前從 Hub 存儲庫下載過數據集，就應該對其進行緩存。這意味著您可以從緩存中重新加載數據集並離線使用。

如果您知道自己無法訪問互聯網，則可以在完全離線模式下運行 🤗 數據集。這可以節省時間，因為數據集將直接在緩存中查找，而不是等待數據集生成器下載超時。將環境變量 `HF_DATASETS_OFFLINE` 設置為 `1` 以啟用完全離線模式。

## Slice splits

您還可以選擇僅加載拆分的特定切片。有兩種用於分割的選項：使用字符串或 [`ReadInstruction API`](https://huggingface.co/docs/datasets/v2.14.0/en/package_reference/builder_classes#datasets.ReadInstruction)。對於簡單情況，字符串更加緊湊和可讀，而 `ReadInstruction` 更易於與可變切片參數一起使用。

串接 `train` 和 `test` split：

=== "String API"

    ```python
    train_test_ds = datasets.load_dataset("bookcorpus", split="train+test")
    ```

=== "ReadInstruction"

    ```python
    ri = datasets.ReadInstruction("train") + datasets.ReadInstruction("test")

    train_test_ds = datasets.load_dataset("bookcorpus", split=ri)
    ```

選擇 `train` split 的特定 rows 數據：

=== "String API"

    ```python
    train_10_20_ds = datasets.load_dataset("bookcorpus", split="train[10:20]")
    ```

=== "ReadInstruction"

    ```python
    train_10_20_ds = datasets.load_dataset("bookcorpu", split=datasets.ReadInstruction("train", from_=10, to=20, unit="abs"))
    ```

或者選擇某個 split 特定百分比的數據：

=== "String API"

    ```python
    train_10pct_ds = datasets.load_dataset("bookcorpus", split="train[:10%]")
    ```

=== "ReadInstruction"

    ```python
    train_10_20_ds = datasets.load_dataset("bookcorpus", split=datasets.ReadInstruction("train", to=10, unit="%"))
    ```

從每個 split 中選擇特定百分比的組合數據：

=== "String API"

    ```python
    train_10_80pct_ds = datasets.load_dataset("bookcorpus", split="train[:10%]+train[-80%:]")
    ```

=== "ReadInstruction"

    ```python
    ri = (datasets.ReadInstruction("train", to=10, unit="%") + datasets.ReadInstruction("train", from_=-80, unit="%"))

    train_10_80pct_ds = datasets.load_dataset("bookcorpus", split=ri)
    ```

最後，您甚至可以創建交叉驗證的分割。下面的範例創建 10-fold cross-validated splits。每個驗證數據集都是 10% 的量，訓練數據集構成剩餘的 90% 的量：

=== "String API"

    ```python
    val_ds = datasets.load_dataset("bookcorpus", split=[f"train[{k}%:{k+10}%]" for k in range(0, 100, 10)])

    train_ds = datasets.load_dataset("bookcorpus", split=[f"train[:{k}%]+train[{k+10}%:]" for k in range(0, 100, 10)])
    ```

=== "ReadInstruction"

    ```python
    val_ds = datasets.load_dataset("bookcorpus", [datasets.ReadInstruction("train", from_=k, to=k+10, unit="%") for k in range(0, 100, 10)])

    train_ds = datasets.load_dataset("bookcorpus", [(datasets.ReadInstruction("train", to=k, unit="%") + datasets.ReadInstruction("train", from_=k+10, unit="%")) for k in range(0, 100, 10)])
    ```

###　Percent slicing and rounding

對於請求的切片邊界無法被 `100` 整除的數據集，默認行為是將邊界舍入為最接近的整數。如下所示，某些切片可能比其他切片包含更多 examples 數據。例如，如果以下 `train` 分割包含 999 條記錄：

```python
# 19 records, from 500 (included) to 519 (excluded).
train_50_52_ds = datasets.load_dataset("bookcorpus", split="train[50%:52%]")

# 20 records, from 519 (included) to 539 (excluded).
train_52_54_ds = datasets.load_dataset("bookcorpus", split="train[52%:54%]")
```

如果您想要相同大小的分割，請使用 `pct1_dropremainder` 舍入代替。這將指定的百分比邊界視為 1% 的倍數。

```python
# 18 records, from 450 (included) to 468 (excluded).
train_50_52pct1_ds = datasets.load_dataset("bookcorpus", split=datasets.ReadInstruction("train", from_=50, to=52, unit="%", rounding="pct1_dropremainder"))

# 18 records, from 468 (included) to 486 (excluded).
train_52_54pct1_ds = datasets.load_dataset("bookcorpus", split=datasets.ReadInstruction("train",from_=52, to=54, unit="%", rounding="pct1_dropremainder"))

# Or equivalently:
train_50_52pct1_ds = datasets.load_dataset("bookcorpus", split="train[50%:52%](pct1_dropremainder)")
train_52_54pct1_ds = datasets.load_dataset("bookcorpus", split="train[52%:54%](pct1_dropremainder)")
```

!!! info
    如果數據集中的示例數量不能被 100 整除，則 `pct1_dropremainder` 舍入可能會截斷數據集中的最後一個示例。

## Troubleshooting

有時，加載數據集時可能會得到意想不到的結果。您可能遇到的兩個最常見的問題是手動下載數據集和指定數據集的特徵。

### Manual download

由於許可不兼容或者文件隱藏在登錄頁面後面，某些數據集要求您手動下載數據集文件。這會導致 `load_dataset()` 拋出錯誤。但是🤗數據集提供了下載丟失文件的詳細說明。下載文件後，使用 `data_dir` 參數指定剛剛下載的文件的路徑。

例如，如果您嘗試從 [MATINF 數據集](https://huggingface.co/datasets/matinf)下載某一個配置：

```python
dataset = load_dataset("matinf", "summarization")
```

結果:

```bash
Downloading and preparing dataset matinf/summarization (download: Unknown size, generated: 246.89 MiB, post-processed: Unknown size, total: 246.89 MiB) to /root/.cache/huggingface/datasets/matinf/summarization/1.0.0/82eee5e71c3ceaf20d909bca36ff237452b4e4ab195d3be7ee1c78b53e6f540e...

AssertionError: The dataset matinf with config summarization requires manual data. 

Please follow the manual download instructions: To use MATINF you have to download it manually. Please fill this google form (https://forms.gle/nkH4LVE4iNQeDzsc9). You will receive a download link and a password once you complete the form. Please extract all files in one folder and load the dataset with: *datasets.load_dataset('matinf', data_dir='path/to/folder/folder_name')*. 

Manual data can be loaded with `datasets.load_dataset(matinf, data_dir='<path/to/manual/data>') 
```

如果您已經使用 `loading script` 從 Hub 下載了數據集到您的計算機，那麼您需要將絕對路徑傳遞給 `data_dir` 或 `data_files` 參數來加載該數據集。否則，如果傳遞相對路徑，`load_dataset()` 將從 Hub 上的存儲庫加載目錄，而不是本地目錄。

### Specify features

當您從本地文件創建數據集時，Apache Arrow 會自動推斷特徵。然而，數據集的特徵可能並不總是符合您的期望，或者您可能想自己定義這些特徵。以下示例顯示如何使用 [`ClassLabel`](https://huggingface.co/docs/datasets/v2.14.0/en/package_reference/main_classes#datasets.ClassLabel) 功能添加自定義標籤。

首先使用 [Features](https://huggingface.co/docs/datasets/v2.14.0/en/package_reference/main_classes#datasets.Features) class 定義您自己的標籤：

```python
class_names = ["sadness", "joy", "love", "anger", "fear", "surprise"]

emotion_features = Features({'text': Value('string'), 'label': ClassLabel(names=class_names)})
```

接下來，使用您剛剛創建的特徵指定 `load_dataset()` 中的 `features` 參數：

```python
dataset = load_dataset('csv', data_files=file_dict, delimiter=';', column_names=['text', 'label'], features=emotion_features)
```

現在，當您查看數據集功能時，您可以看到它使用您定義的自定義標籤：

```python
print(dataset['train'].features)
```

結果:

```bash
{'text': Value(dtype='string', id=None),
'label': ClassLabel(num_classes=6, names=['sadness', 'joy', 'love', 'anger', 'fear', 'surprise'], names_file=None, id=None)}
```