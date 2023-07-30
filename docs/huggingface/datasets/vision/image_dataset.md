# Create an image dataset

有兩種創建和共享圖像數據集的方法。本指南將向您展示如何：

- 使用 `ImageFolder` 和一些 `metadata` 來創建 image dataset。這是一個 no-code 解決方案，用於快速創建包含數千張圖像的 image dataset。
- 通過編寫 `loading script` 創建圖像數據集。此方法稍微複雜一些，但您可以更靈活地定義、下載和生成數據集，這對於更複雜或大規模的圖像數據集非常有用。

## ImageFolder

ImageFolder 是一個數據集生成器，旨在快速加載包含數千張圖像的圖像數據集，而無需您編寫任何代碼。

!!! tip
    💡 查看 [Split pattern hierarchy](https://huggingface.co/docs/datasets/v2.14.1/en/repository_structure#split-pattern-hierarchy) 結構，了解有關 ImageFolder 如何根據數據集存儲庫結構創建數據集分割的更多信息。

ImageFolder 根據目錄名稱自動推斷數據集的類別標籤。將數據集存儲在如下目錄結構中：

```bash
folder/train/dog/golden_retriever.png
folder/train/dog/german_shepherd.png
folder/train/dog/chihuahua.png

folder/train/cat/maine_coon.png
folder/train/cat/bengal.png
folder/train/cat/birman.png
```

然後用戶可以通過在 `load_dataset()` 中指定 `imagefolder` 和 `data_dir` 中指定目錄來加載數據集：

```python
from datasets import load_dataset

dataset = load_dataset("imagefolder", data_dir="/path/to/folder")
```

您還可以使用 imagefolder 加載涉及多個 split 的數據集。為此，您的數據集目錄應具有以下結構：

```bash
folder/train/dog/golden_retriever.png
folder/train/cat/maine_coon.png
folder/test/dog/german_shepherd.png
folder/test/cat/bengal.png
```

!!! tip
    如果所有圖像文件都包含在一個目錄中或者它們不在同一級別的目錄結構中，則不會自動添加標籤列。如果需要，請明確設置 `drop_labels=False` 。

如果您想要包含有關數據集的其他信息（例如文本標題或邊界框），請將其作為 `metadata.csv` 文件添加到您的文件夾中。這使您可以快速為不同的計算機視覺任務（例如文本字幕或對象檢測）創建數據集。您還可以使用 JSONL 文件 `metadata.jsonl`。

```bash
folder/train/metadata.csv
folder/train/0001.png
folder/train/0002.png
folder/train/0003.png
```

您還可以壓縮圖像：

```bash
folder/metadata.csv
folder/train.zip
folder/test.zip
folder/valid.zip
```

您的 `metadata.csv` 文件必須有一個 `file_name` 列(column)，它將 image file 與其 metadata 鏈接起來：

```csv
file_name,additional_feature
0001.png,This is a first value of a text feature you added to your images
0002.png,This is a second value of a text feature you added to your images
0003.png,This is a third value of a text feature you added to your images
```

或使用 `metadata.jsonl`：

```bash
{"file_name": "0001.png", "additional_feature": "This is a first value of a text feature you added to your images"}
{"file_name": "0002.png", "additional_feature": "This is a second value of a text feature you added to your images"}
{"file_name": "0003.png", "additional_feature": "This is a third value of a text feature you added to your images"}
```

!!! info
    如果存在 metadata 文件，則默認情況下會刪除基於目錄名稱推斷的標籤。要包含這些標籤，請在 `load_dataset()` 中設置 `drop_labels=False`。

### Image captioning

Image captioning datasets 包含描述圖像的文本。範例 `metadata.csv` 可能如下所示：

```csv
file_name,text
0001.png,This is a golden retriever playing with a ball
0002.png,A german shepherd
0003.png,One chihuahua
```

使用 `ImageFolder` 加載數據集，它將為圖像標題創建一個文本列：

```python
dataset = load_dataset("imagefolder", data_dir="/path/to/folder", split="train")

print(dataset[0]["text"])
```

結果:

```bash
"This is a golden retriever playing with a ball"
```

### Object detection

Object detection datasets 具有識別圖像中物件的邊界框和類別。示例 `metadata.jsonl` 可能如下所示：

```jsonl
{"file_name": "0001.png", "objects": {"bbox": [[302.0, 109.0, 73.0, 52.0]], "categories": [0]}}
{"file_name": "0002.png", "objects": {"bbox": [[810.0, 100.0, 57.0, 28.0]], "categories": [1]}}
{"file_name": "0003.png", "objects": {"bbox": [[160.0, 31.0, 248.0, 616.0], [741.0, 68.0, 202.0, 401.0]], "categories": [2, 2]}}
```

使用 ImageFolder 加載數據集，它將創建一個包含邊界框和類別的物件列：

```python
dataset = load_dataset("imagefolder", data_dir="/path/to/folder", split="train")

print(dataset[0]["objects"])
```

結果:

```bash
{"bbox": [[302.0, 109.0, 73.0, 52.0]], "categories": [0]}
```

### Upload dataset to the Hub

創建數據集後，您可以使用 `push_to_hub()` 方法將其共享到 Hub。確保您已安裝 `huggingface_hub` 套件並且已登入您的 Hugging Face 帳戶（有關更多詳細信息，請參閱 [Upload with Python tutorial](https://huggingface.co/docs/datasets/v2.14.1/en/upload_dataset#upload-with-python)）。

使用 `push_to_hub()` 上傳數據集：

```python
from datasets import load_dataset

dataset = load_dataset("imagefolder", data_dir="/path/to/folder", split="train")

dataset.push_to_hub("stevhliu/my-image-captioning-dataset")
```

## Loading script

編寫數據集加載腳本來共享數據集。它定義數據集的分割和配置，並處理數據集的下載和生成。該腳本位於與數據集相同的文件夾或存儲庫中，並且應具有相同的名稱。

```bash
my_dataset/
├── README.md
├── my_dataset.py
└── data/  # optional, may contain your images or TAR archives
```

這種結構允許您的數據集使用一行程式碼中加載：

```python
from datasets import load_dataset

dataset = load_dataset("path/to/my_dataset")
```

本指南將向您展示如何為圖像數據集創建數據集加載腳本，這與 [creating a loading script for text datasets](https://huggingface.co/docs/datasets/v2.14.1/en/dataset_script)　有點不同。您將學習如何：

- 構建 dataset builder class
- 構建 dataset configurations
- 增加 dataset metadata
- 下載與定義 dataset splits
- 創建 dataset
- 創建 dataset metadata (optional)
- 上傳 dataset 到 Hub

最好的學習方法是打開現有的圖像數據集加載腳本，例如 [Food-101](https://huggingface.co/datasets/food101/blob/main/food101.py)，然後跟著操作！

!!! tip
    為了幫助您入門，Huggingface 創建了一個 [loading script template](https://github.com/huggingface/datasets/blob/main/templates/new_dataset_script.py)，您可以複製並用來作為開發的起點！

###　Create a dataset builder class

[`GeneratorBasedBuilder`](https://huggingface.co/docs/datasets/v2.14.1/en/package_reference/builder_classes#datasets.GeneratorBasedBuilder) 是字典生成器生成的數據集的基類。在此類中，提供了三種方法來幫助創建數據集：

- `info` 存儲有關數據集的信息，例如其描述、許可證和 features。
- `split_generators` 下載數據集並定義其分割。
- `generate_examples` 為每個分割生成圖像和標籤。

首先創建數據集類作為 `GeneratorBasedBuilder` 的子類並添加三個方法。不用擔心填寫這些方法，您將在接下來的章節中開發這些方法：

```python
class Food101(datasets.GeneratorBasedBuilder):
    """Food-101 Images dataset"""

    def _info(self):

    def _split_generators(self, dl_manager):

    def _generate_examples(self, images, metadata_path):
```

#### Multiple configurations

在某些情況下，數據集可能具有多個配置。例如，如果您查看 [Imagenette 數據集](https://huggingface.co/datasets/frgfm/imagenette)，您會注意到存在三個子集。

要創建不同的配置，請使用 [BuilderConfig](https://huggingface.co/docs/datasets/v2.14.1/en/package_reference/builder_classes#datasets.BuilderConfig) 類為數據集創建子類。在 `data_url` 和 `metadata_urls` 中提供下載圖像和標籤的鏈接：

```python
class Food101Config(datasets.BuilderConfig):
    """Builder Config for Food-101"""
 
    def __init__(self, data_url, metadata_urls, **kwargs):
        """BuilderConfig for Food-101.
        Args:
          data_url: `string`, url to download the zip file from.
          metadata_urls: dictionary with keys 'train' and 'validation' containing the archive metadata URLs
          **kwargs: keyword arguments forwarded to super.
        """
        super(Food101Config, self).__init__(version=datasets.Version("1.0.0"), **kwargs)
        self.data_url = data_url
        self.metadata_urls = metadata_urls
```

現在您可以在 `GeneratorBasedBuilder` 的頂部定義您的子集。想像一下，您想要根據 `Food-101` 數據集是早餐還是晚餐食物來創建兩個子集。

1. 使用 `BUILDER_CONFIGS` 列表中的 `Food101Config` 定義您的子集。
2. 對於每個配置，請提供名稱、描述以及圖像和標籤的下載位置。

```python
class Food101(datasets.GeneratorBasedBuilder):
    """Food-101 Images dataset"""
 
    BUILDER_CONFIGS = [
        Food101Config(
            name="breakfast",
            description="Food types commonly eaten during breakfast.",
            data_url="https://link-to-breakfast-foods.zip",
            metadata_urls={
                "train": "https://link-to-breakfast-foods-train.txt", 
                "validation": "https://link-to-breakfast-foods-validation.txt"
            },
        ,
        Food101Config(
            name="dinner",
            description="Food types commonly eaten during dinner.",
            data_url="https://link-to-dinner-foods.zip",
            metadata_urls={
                "train": "https://link-to-dinner-foods-train.txt", 
                "validation": "https://link-to-dinner-foods-validation.txt"
            },
        )...
    ]
```

現在，如果用戶想要加載 `breakfast` 配置，他們可以使用配置名稱：

```python
from datasets import load_dataset

ds = load_dataset("food101", "breakfast", split="train")
```

### Add dataset metadata

添加有關數據集的信息對於用戶了解更多信息很有用。此信息存儲在由 `info` 方法返回的 `DatasetInfo` 類別中。用戶可以通過以下方式訪問此信息：

```python
from datasets import load_dataset_builder

ds_builder = load_dataset_builder("food101")

print(ds_builder.info)
```

您可以指定有關數據集的大量信息，但需要包含的一些重要信息包括：

1. `description` 提供數據集的簡潔描述。
2. `features` 指定數據集列類型。由於您正在創建圖像加載腳本，因此您需要包含圖像功能。
3. `supervised_keys` 指定輸入特徵和標籤。
4. `homepage` 提供數據集主頁的鏈接。
5. `citation` 是數據集的 BibTeX 引用。
6. `license` 說明數據集的許可證。

!!! info
    您會注意到許多數據集信息是在加載腳本的前面定義的，這使得它更易於閱讀。您還可以輸入其他 Datasets.Features，因此請務必查看完整列表以了解更多詳細信息。

```python
def _info(self):
    return datasets.DatasetInfo(
        description=_DESCRIPTION,
        features=datasets.Features(
            {
                "image": datasets.Image(),
                "label": datasets.ClassLabel(names=_NAMES),
            }
        ),
        supervised_keys=("image", "label"),
        homepage=_HOMEPAGE,
        citation=_CITATION,
        license=_LICENSE,
        task_templates=[ImageClassification(image_column="image", label_column="label")],
    )
```

### Download and define the dataset splits

現在您已經添加了有關數據集的一些信息，下一步是下載數據集並生成拆分。

1. 使用 `DownloadManager.download()` 方法下載數據集以及您想要與其關聯的任何其他 metadata。該方法接受：

    - 一個在 Hub dataset repository 的資料集名稱 (`data/folder`)
    - 一個 URL 假如相關檔案被置放在其它地方
    - 一個 list 或 dictionary 來表示 file names 或 URLs

    在 Food-101 加載腳本中，您會再次注意到 URL 是在腳本的前面定義的。

2. 下載數據集後，使用 [SplitGenerator](https://huggingface.co/docs/datasets/v2.14.1/en/package_reference/builder_classes#datasets.SplitGenerator) 來組織每個分割中的圖像和標籤。使用標準名稱命名每個 split，例如：`Split.TRAIN`、`Split.TEST` 和 `SPLIT.Validation`。

    在 `gen_kwargs` 參數中，指定要迭代和加載的圖像的文件路徑。如有必要，您可以使用 `DownloadManager.iter_archive()` 迭代 TAR 存檔中的圖像。您還可以在 `metadata_path` 中指定關聯的標籤。圖像和 `metadata_path` 實際上會傳遞到下一步，您將在其中實際生成數據集。

```python
def _split_generators(self, dl_manager):
    archive_path = dl_manager.download(_BASE_URL)
    split_metadata_paths = dl_manager.download(_METADATA_URLS)
    return [
        datasets.SplitGenerator(
            name=datasets.Split.TRAIN,
            gen_kwargs={
                "images": dl_manager.iter_archive(archive_path),
                "metadata_path": split_metadata_paths["train"],
            },
        ),
        datasets.SplitGenerator(
            name=datasets.Split.VALIDATION,
            gen_kwargs={
                "images": dl_manager.iter_archive(archive_path),
                "metadata_path": split_metadata_paths["test"],
            },
        ),
    ]
```

### Generate the dataset

`GeneratorBasedBuilder` 類別中的最後一個方法實際上生成數據集中的圖像和標籤。它根據 `info` 方法中的 features 中指定的結構生成數據集。如您所見，`generate_examples` 接受上一個方法中的圖像和 `metadata_path` 作為參數。

現在您可以編寫一個函數來打開和加載數據集中的範例：

```python
def _generate_examples(self, images, metadata_path):
    """Generate images and labels for splits."""
    with open(metadata_path, encoding="utf-8") as f:
        files_to_keep = set(f.read().split("\n"))

    for file_path, file_obj in images:
        if file_path.startswith(_IMAGES_DIR):
            if file_path[len(_IMAGES_DIR) : -len(".jpg")] in files_to_keep:
                label = file_path.split("/")[2]
                yield file_path, {
                    "image": {"path": file_path, "bytes": file_obj.read()},
                    "label": label,
                }
```

###　Generate the dataset metadata (optional)

可以生成 `dataset metadata` 並將其存儲在數據集卡（`README.md` 文件）中。

運行以下命令在 `README.md` 中生成數據集元數據並確保新的加載腳本正常工作：

```bash
datasets-cli test path/to/<your-dataset-loading-script> --save_info --all_configs
```

如果您的加載腳本通過了測試，您的數據集文件夾中的 `README.md` 文件的標頭中現在應該包含 `dataset_info` YAML 欄位。

###　Upload the dataset to the Hub

腳本準備就緒後，創建數據集卡並將其上傳到 Hub。

恭喜，您現在可以從 Hub 加載數據集！

```python
from datasets import load_dataset

load_dataset("<username>/my_dataset")
```