# Load image data

Image 數據集從 `image` 列(column)加載，其中包含 PIL object。

!!! info
    要使用 Image 數據集，您需要安裝 vision 依賴項。查看[安裝指南](https://huggingface.co/docs/datasets/v2.14.1/en/installation#vision)以了解如何安裝。

當您加載 Image 數據集並調用 image 列時，Image feature 會自動將 PIL object 解碼為 image：

```python
from datasets import load_dataset, Image

dataset = load_dataset("beans", split="train")

dataset[0]["image"]
```

!!! tip
    首先使用行 index，然後使用 image 列(column)對圖像數據集進行索引 - `dataset[0]["image"]` - 以避免對數據集中的所有圖像物件進行解碼和重新採樣。否則，當您有大量數據集，這可能是一個緩慢且耗時的過程。

##　Local files

您可以從圖像路徑加載數據集。使用 `cast_column()` 函數接受一列 image 文件路徑，並使用 Image feature 將其解碼為 PIL image：

```python
from datasets import load_dataset, Image

dataset = Dataset.from_dict({"image": ["path/to/image_1", "path/to/image_2", ..., "path/to/image_n"]}).cast_column("image", Image())

dataset[0]["image"]
```

結果:

```bash
<PIL.PngImagePlugin.PngImageFile image mode=RGBA size=1200x215 at 0x15E6D7160>]
```

如果您只想加載圖像數據集的底層路徑而不解碼 image object，請在 Image feature 中設置 `decode=False`：

```python
dataset = load_dataset("beans", split="train").cast_column("image", Image(decode=False))

dataset[0]["image"]
```


結果:

```bash
{'bytes': None,
 'path': '/root/.cache/huggingface/datasets/downloads/extracted/b0a21163f78769a2cf11f58dfc767fb458fc7cea5c05dccc0144a2c0f0bc1292/train/bean_rust/bean_rust_train.29.jpg'}
```

## ImageFolder

您還可以使用 `ImageFolder` 數據集生成器來加載數據集，這個方法不需要編寫自定義數據加載器。這使得 `ImageFolder` 非常適合快速創建和加載包含數千個圖像的圖像數據集，用於不同的視覺任務。您的圖像數據集結構應如下所示：

```bash
folder/train/dog/golden_retriever.png
folder/train/dog/german_shepherd.png
folder/train/dog/chihuahua.png

folder/train/cat/maine_coon.png
folder/train/cat/bengal.png
folder/train/cat/birman.png
```

通過在 `data_dir` 中指定 `imagefolder` 和數據集目錄來加載數據集：

```python
from datasets import load_dataset

dataset = load_dataset("imagefolder", data_dir="/path/to/folder")

dataset["train"][0]

dataset["train"][-1]
```

結果:

```bash
{"image": <PIL.PngImagePlugin.PngImageFile image mode=RGBA size=1200x215 at 0x15E6D7160>, "label": 0}

{"image": <PIL.PngImagePlugin.PngImageFile image mode=RGBA size=1200x215 at 0x15E8DAD30>, "label": 1}
```

使用 `data_files` 參數從 URL 加載遠程數據集：

```python
dataset = load_dataset("imagefolder", data_files="https://download.microsoft.com/download/3/E/1/3E1C3F21-ECDB-4869-8368-6DEBA77B919F/kagglecatsanddogs_3367a.zip", split="train")
```

某些數據集具有與其關聯的 `metadata` 文件 (`metadata.csv`/`metadata.jsonl`)，其中包含有關數據的其他信息，例如邊界框、文本標題和標籤。當您調用 `load_dataset()` 並指定圖像文件夾時，元數據會自動加載。

要忽略元數據文件中的信息，請在 `load_dataset()` 中設置 `drop_labels=False`，並允許 `ImageFolder` 自動從目錄名稱推斷標籤名稱：

```python
from datasets import load_dataset

dataset = load_dataset("imagefolder", data_dir="/path/to/folder", drop_labels=False)
```

!!! tip
    有關創建您自己的 `ImageFolder` 數據集的更多信息，請查看 [Create an image dataset](https://huggingface.co/docs/datasets/v2.14.1/en/image_dataset) 指南。