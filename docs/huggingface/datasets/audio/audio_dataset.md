# Create an audio dataset

您可以通過在 Hugging Face Hub 上創建數據集存儲庫來與您的團隊或社區中的任何人共享數據集：

```python
from datasets import load_dataset

dataset = load_dataset("<username>/my_dataset")
```

創建和共享音頻數據集的方法有多種：

- 使用 `Dataset.push_to_hub()` 從 python 中的本地文件創建音頻數據集。這是一個簡單的方法，只需要在 python 中執行幾個步驟。
- 使用 `AudioFolder` 構建器創建音頻數據集存儲庫。這是一個 no-code 解決方案，用於快速創建包含數千個音頻文件的音頻數據集。
- 通過編寫 `loading script` 創建音頻數據集。此方法適用於進階用戶，需要額外撰寫一些程式碼，但您在如何定義、下載和生成數據集方面具有更大的靈活性，這對於更複雜或大規模的音頻數據集非常有用。

## Local files

您可以使用音頻文件的路徑加載自己的數據集。使用 `cast_column()` 函數獲取一列音頻文件路徑，並將其投射到 Audio feature：

```python
audio_dataset = Dataset.from_dict({"audio": ["path/to/audio_1", "path/to/audio_2", ..., "path/to/audio_n"]}).cast_column("audio", Audio())

print(audio_dataset[0]["audio"])
```

結果:

```bash
{'array': array([ 0.        ,  0.00024414, -0.00024414, ..., -0.00024414,
         0.        ,  0.        ], dtype=float32),
 'path': 'path/to/audio_1',
 'sampling_rate': 16000}
```

然後使用 `Dataset.push_to_hub()` 將數據集上傳到 Hugging Face Hub：

```python
audio_dataset.push_to_hub("<username>/my_dataset")
```

這將創建一個包含您的音頻數據集的數據集存儲庫：

```bash
my_dataset/
├── README.md
└── data/
    └── train-00000-of-00001.parquet
```

## AudioFolder

`AudioFolder` 是一個數據集生成器，旨在快速加載包含數千個音頻文件的音頻數據集，而無需您編寫任何代碼。只要您在元數據文件 (`metadata.csv`/`metadata.jsonl`) 中包含有關數據集的其他額外信息（例如轉錄、說話者口音或說話者意圖），`AudioFolder` 就會自動加載這些信息。

!!! tip
    💡 查看 [Split pattern hierarchy](https://huggingface.co/docs/datasets/v2.14.1/en/repository_structure#split-pattern-hierarchy) 結構，了解有關 `AudioFolder` 如何根據數據集存儲庫結構創建數據集分割的更多信息。

在 Hugging Face Hub 上創建數據集存儲庫，並按照 `AudioFolder` 結構上傳數據集目錄：

```bash
my_dataset/
├── README.md
├── metadata.csv
└── data/
```

數據文件夾可以是您想要的任何名稱。

!!! tip
    如果數據列包含更複雜的格式（如浮點數列表），則將元數據存儲為 `jsonl` 文件會很有幫助，以避免解析錯誤或將復雜值讀取為字符串。

元數據文件應包含 `file_name` 列，以將音頻文件鏈接到其元數據：

```csv title="metadata.csv"
file_name,transcription
data/first_audio_file.mp3,znowu się duch z ciałem zrośnie w młodocianej wstaniesz wiosnie i możesz skutkiem tych leków umierać wstawać wiek wieków dalej tam były przestrogi jak siekać głowę jak nogi
data/second_audio_file.mp3,już u źwierzyńca podwojów król zasiada przy nim książęta i panowie rada a gdzie wzniosły krążył ganek rycerze obok kochanek król skinął palcem zaczęto igrzysko
data/third_audio_file.mp3,pewnie kędyś w obłędzie ubite minęły szlaki zaczekajmy dzień jaki poślemy szukać wszędzie dziś jutro pewnie będzie posłali wszędzie sługi czekali dzień i drugi gdy nic nie doczekali z płaczem chcą jechać dali
```

然後您可以將數據集存儲在如下目錄結構中：

```bash
metadata.csv
data/first_audio_file.mp3
data/second_audio_file.mp3
data/third_audio_file.mp3
```

用戶現在可以通過在 `load_dataset()` 中指定音頻文件夾和在 `data_dir` 中指定數據集目錄來加載數據集和關聯的元數據：

```python
from datasets import load_dataset

dataset = load_dataset("audiofolder", data_dir="/path/to/data")

print(dataset["train"][0])
```

結果:

```bash
{'audio':
    {'path': '/path/to/extracted/audio/first_audio_file.mp3',
    'array': array([ 0.00088501,  0.0012207 ,  0.00131226, ..., -0.00045776, -0.00054932, -0.00054932], dtype=float32),
    'sampling_rate': 16000},
 'transcription': 'znowu się duch z ciałem zrośnie w młodocianej wstaniesz wiosnie i możesz skutkiem tych leków umierać wstawać wiek wieków dalej tam były przestrogi jak siekać głowę jak nogi'
}
```

您還可以使用 `audiofolder` 加載涉及多個分割的數據集。為此，您的數據集目錄可能具有以下結構：

```bash
data/train/first_train_audio_file.mp3
data/train/second_train_audio_file.mp3

data/test/first_test_audio_file.mp3
data/test/second_test_audio_file.mp3
```

!!! warning
    請注意，如果音頻文件不位於元數據文件旁邊，則 `file_name` 列應該是音頻文件的完整相對路徑，而不僅僅是其文件名。

對於沒有任何關聯元數據的音頻數據集，`AudioFolder` 會根據目錄名稱自動推斷數據集的類標籤。它可能對音頻分類任務有用。您的數據集目錄可能如下所示：

```python
data/train/electronic/01.mp3
data/train/punk/01.mp3

data/test/electronic/09.mp3
data/test/punk/09.mp3
```

使用 `AudioFolder` 加載數據集，它將根據目錄名稱（語言 ID）創建一個標籤列：

```python
from datasets import load_dataset

dataset = load_dataset("audiofolder", data_dir="/path/to/data")

print(dataset["train"][0])

print(dataset["train"][-1])
```

結果:

```bash
{'audio':
    {'path': '/path/to/electronic/01.mp3',
     'array': array([ 3.9714024e-07,  7.3031038e-07,  7.5640685e-07, ...,
         -1.1963668e-01, -1.1681189e-01, -1.1244172e-01], dtype=float32),
     'sampling_rate': 44100},
 'label': 0  # "electronic"
}

{'audio':
    {'path': '/path/to/punk/01.mp3',
     'array': array([0.15237972, 0.13222949, 0.10627693, ..., 0.41940814, 0.37578005,
         0.33717662], dtype=float32),
     'sampling_rate': 44100},
 'label': 1  # "punk"
}
```

!!! warning
    如果所有音頻文件都包含在一個目錄中或者它們不在同一級別的目錄結構中，則不會自動添加標籤列。如果需要，請明確設置 `drop_labels=False`。

!!! tip
    一些音頻數據集（例如 Kaggle 競賽中的數據集）對於每個分割都有單獨的元數據文件。如果每個分割的元數據功能相同，則可以使用音頻文件夾一次性加載所有分割。如果每個拆分的元數據功能不同，您應該使用單獨的 `load_dataset()` 調用來加載它們。


## Loading script

編寫數據集加載腳本來手動創建數據集。它定義數據集的分割和配置，並處理下載和生成數據集示例。該腳本應與您的數據集文件夾或存儲庫具有相同的名稱：

```bash
my_dataset/
├── README.md
├── my_dataset.py
└── data/
```

數據文件夾 `data` 可以是您想要的任何名稱，不一定要是 `data`。該文件夾是可選的，除非您在 Hub 上託管數據集。

此目錄結構允許您的數據集在一行中加載：

```python
from datasets import load_dataset

dataset = load_dataset("path/to/my_dataset")
```

本指南將向您展示如何為音頻數據集創建數據集加載腳本，這與為文本數據集創建加載腳本有點不同。音頻數據集通常存儲在 `tar.gz` 檔案中，這需要特定的方法來支持流模式。雖然不需要流式傳輸，但我們強烈鼓勵在音頻數據集中實現流式傳輸支持，因為：

1. 沒有大量磁盤空間的用戶無需下載即可使用您的數據集。在 [Stream 指南](https://huggingface.co/docs/datasets/v2.14.1/en/stream) 中了解有關 `streaming` 的更多信息！
2. 用戶可以在數據集查看器中預覽數據集。

以下是使用 TAR 存檔的範例：

```bash
my_dataset/
├── README.md
├── my_dataset.py
└── data/
    ├── train.tar.gz
    ├── test.tar.gz
    └── metadata.csv
```

除了學習如何創建 streamable dataset 之外，您還將學習如何：

- 構建一個 dataset builder class
- 構建 dataset configurations
- 增加 dataset metadata
- 下載並定義 dataset splits
- 構建 dataset.
- 上傳 dataset 到 Hub

最好的學習方法是打開現有的音頻數據集加載腳本，例如 [Vivos](https://huggingface.co/datasets/vivos/blob/main/vivos.py)，然後學習相關的程式撰寫！

!!! tip
    本指南介紹如何處理存儲在 TAR 存檔中的音頻數據 - 這是音頻數據集最常見的情況。查看 [Minds14](https://huggingface.co/datasets/PolyAI/minds14/blob/main/minds14.py) 數據集，獲取使用 ZIP 存檔的音頻腳本示例。

!!! info
    為了幫助您入門，我們創建了一個 [loading script template](https://github.com/huggingface/datasets/blob/main/templates/new_dataset_script.py)，您可以復制並用作起點！

### Create a dataset builder class

[GeneratorBasedBuilder](https://huggingface.co/docs/datasets/v2.14.1/en/package_reference/builder_classes#datasets.GeneratorBasedBuilder) 是字典生成器生成的數據集的基類。在此類中，提供了三種方法來幫助創建數據集：

- `_info` 存儲有關數據集的信息，例如其描述、許可證和功能。
- `_split_generators` 下載數據集並定義其分割。
- `_generate_examples` 生成數據集的樣本，其中包含每個分割的音頻數據和 info 中指定的其他特徵。

首先創建數據集類作為 `GeneratorBasedBuilder` 的子類並添加三個方法。不用擔心填寫這些方法，您將在接下來的幾節中開發這些方法：

```python
class VivosDataset(datasets.GeneratorBasedBuilder):
    """VIVOS is a free Vietnamese speech corpus consisting of 15 hours of recording speech prepared for
    Vietnamese Automatic Speech Recognition task."""

    def _info(self):

    def _split_generators(self, dl_manager):

    def _generate_examples(self, prompts_path, path_to_clips, audio_files):
```

#### Multiple configurations

在某些情況下，數據集可能具有多個配置。例如，[LibriVox Indonesia](https://huggingface.co/datasets/indonesian-nlp/librivox-indonesia) 數據集有幾種對應不同語言的配置。

要創建不同的配置，請使用 `BuilderConfig` 類別創建數據集的子類。唯一需要的參數是配置的名稱，它必須傳遞給配置的超類 `__init__()`。否則，您可以在配置類中指定所需的任何自定義參數。

```python
class LibriVoxIndonesiaConfig(datasets.BuilderConfig):
    """BuilderConfig for LibriVoxIndonesia."""

    def __init__(self, name, version, **kwargs):
        self.language = kwargs.pop("language", None)
        self.release_date = kwargs.pop("release_date", None)
        self.num_clips = kwargs.pop("num_clips", None)
        self.num_speakers = kwargs.pop("num_speakers", None)
        self.validated_hr = kwargs.pop("validated_hr", None)
        self.total_hr = kwargs.pop("total_hr", None)
        self.size_bytes = kwargs.pop("size_bytes", None)
        self.size_human = size_str(self.size_bytes)

        description = (
            f"LibriVox-Indonesia speech to text dataset in {self.language} released on {self.release_date}. "
            f"The dataset comprises {self.validated_hr} hours of transcribed speech data"
        )

        super(LibriVoxIndonesiaConfig, self).__init__(
            name=name,
            version=datasets.Version(version),
            description=description,
            **kwargs,
        )
```

在 `GeneratorBasedBuilder` 內的 `BUILDER_CONFIGS` 類別變量中定義配置。在此示例中，作者從其存儲庫中的單獨的 `release_stats.py` 文件導入語言，然後循環遍歷每種語言以創建配置：

```python
class LibriVoxIndonesiaConfig(datasets.GeneratorBasedBuilder):
    DEFAULT_CONFIG_NAME = "all"

    BUILDER_CONFIGS = [
        LibriVoxIndonesiaConfig(
            name=lang,
            version=STATS["version"],
            language=LANGUAGES[lang],
            release_date=STATS["date"],
            num_clips=lang_stats["clips"],
            num_speakers=lang_stats["users"],
            total_hr=float(lang_stats["totalHrs"]) if lang_stats["totalHrs"] else None,
            size_bytes=int(lang_stats["size"]) if lang_stats["size"] else None,
        )
        for lang, lang_stats in STATS["locales"].items()
    ]
```

!!! info
    通常，用戶需要指定要在 `load_dataset()` 中加載的配置，否則會引發 `ValueError`。您可以通過設置要在 `DEFAULT_CONFIG_NAME` 中加載的默認數據集配置來避免這種情況。

現在，如果用戶想要加載 Balinese（bal）配置，他們可以使用配置名稱：

```python
from datasets import load_dataset

dataset = load_dataset("indonesian-nlp/librivox-indonesia", "bal", split="train")
```

### Add dataset metadata

添加有關數據集的信息可以幫助用戶了解更多信息。此信息存儲在由 `info` 方法返回的 `DatasetInfo` 類別中。用戶可以通過以下方式訪問此信息：

```python
from datasets import load_dataset_builder

ds_builder = load_dataset_builder("vivos")

print(ds_builder.info)
```

您可以包含很多有關數據集的信息，但其中一些重要信息是：

1. `description` 提供數據集的簡潔描述。
2. `features` 指定數據集列類型。由於您正在創建音頻加載腳本，因此您需要包含音頻功能和數據集的 `sampling_rate`。
3. `homepage` 提供數據集主頁的鏈接。
4. `license` 指定使用許可證類型定義的數據集的權限。
5. `citation` 是數據集的 BibTeX 引用。

!!! info
    您會注意到許多數據集信息是在 `loading script` 裡面定義的，這可以使 dataset 其更易於被人了解歟使用。您還可以輸入其他 `Dataset.Features`，因此請務必查看完整列表和 [features guide](https://huggingface.co/docs/datasets/v2.14.1/en/about_dataset_features) 以了解更多詳細信息。

```python
def _info(self):
    return datasets.DatasetInfo(
        description=_DESCRIPTION,
        features=datasets.Features(
            {
                "speaker_id": datasets.Value("string"),
                "path": datasets.Value("string"),
                "audio": datasets.Audio(sampling_rate=16_000),
                "sentence": datasets.Value("string"),
            }
        ),
        supervised_keys=None,
        homepage=_HOMEPAGE,
        license=_LICENSE,
        citation=_CITATION,
    )
```

### Download and define the dataset splits

現在您已經添加了有關數據集的一些信息，下一步是下載數據集並定義分割。

1. 使用 `download()` 方法下載 `_PROMPTS_URLS` 處的元數據文件和 `_DATA_URL` 處的音頻 TAR 存檔。此方法返回本地文件/存檔的路徑。在流模式下，它不會下載文件，只是返回一個 URL 以從中流式傳輸數據。該方法接受：

    - Hub 數據集存儲庫內文件的相對路徑（例如，在 `data/` 文件夾中）
    - 託管在其他地方的文件的 URL
    - 文件名或 URL 的（嵌套）列表或字典

2. 下載數據集後，使用 `SplitGenerator` 來組織每個分割中的音頻文件和句子提示。使用標準名稱命名每個 split，例如：`Split.TRAIN`、`Split.TEST` 和 `SPLIT.Validation`。

    在 `gen_kwargs` 參數中，指定 `prompts_path` 和 `path_to_clips` 的文件路徑。對於 `audio_files`，您需要使用 `iter_archive()` 來迭代 TAR 存檔中的音頻文件。這將為您的數據集啟用流式傳輸。所有這些文件路徑都將傳遞到下一步，您將在其中實際生成數據集。

    ```python
    def _split_generators(self, dl_manager):
        """Returns SplitGenerators."""
        prompts_paths = dl_manager.download(_PROMPTS_URLS)
        archive = dl_manager.download(_DATA_URL)
        train_dir = "vivos/train"
        test_dir = "vivos/test"

        return [
            datasets.SplitGenerator(
                name=datasets.Split.TRAIN,
                gen_kwargs={
                    "prompts_path": prompts_paths["train"],
                    "path_to_clips": train_dir + "/waves",
                    "audio_files": dl_manager.iter_archive(archive),
                },
            ),
            datasets.SplitGenerator(
                name=datasets.Split.TEST,
                gen_kwargs={
                    "prompts_path": prompts_paths["test"],
                    "path_to_clips": test_dir + "/waves",
                    "audio_files": dl_manager.iter_archive(archive),
                },
            ),
        ]
    ```

!!! tip
    此實現不會提取下載的檔案。如果您想在下載後解壓文件，則需要另外使用 `extract()`，請 [(Advanced) Extract TAR archives](https://huggingface.co/docs/datasets/v2.14.1/en/audio_dataset#advanced-extract-tar-archives-locally)。


### Generate the dataset

`GeneratorBasedBuilder` 類別中的最後一個方法實際上生成數據集中的樣本。它根據 `info` 方法中的特徵中指定的結構生成數據集。如您所見，`generate_examples` 接受上一個方法中的`prompts_path`、`path_to_clips` 和 `audio_files` 作為參數。

TAR 存檔內的文件是按順序訪問和生成的。這意味著您需要首先擁有與 TAR 文件中的音頻文件關聯的元數據，以便您可以生成它及其相應的音頻文件。

```python
examples = {}

with open(prompts_path, encoding="utf-8") as f:
    for row in f:
        data = row.strip().split(" ", 1)
        speaker_id = data[0].split("_")[0]
        audio_path = "/".join([path_to_clips, speaker_id, data[0] + ".wav"])
        examples[audio_path] = {
            "speaker_id": speaker_id,
            "path": audio_path,
            "sentence": data[1],
        }
```

最後，迭代 `audio_files` 中的文件並生成它們及其相應的元數據。 `iter_archive()` 生成一個 `(path, f)` 元組，其中 `path` 是 TAR 存檔內文件的相對路徑，`f` 是文件對象本身。

```python
inside_clips_dir = False

id_ = 0

for path, f in audio_files:
    if path.startswith(path_to_clips):
        inside_clips_dir = True
        if path in examples:
            audio = {"path": path, "bytes": f.read()}
            yield id_, {**examples[path], "audio": audio}
            id_ += 1
    elif inside_clips_dir:
        break
```

將這兩個步驟放在一起，整個 `_generate_examples` 方法如下所示：

```python
def _generate_examples(self, prompts_path, path_to_clips, audio_files):
    """Yields examples as (key, example) tuples."""
    examples = {}
    with open(prompts_path, encoding="utf-8") as f:
        for row in f:
            data = row.strip().split(" ", 1)
            speaker_id = data[0].split("_")[0]
            audio_path = "/".join([path_to_clips, speaker_id, data[0] + ".wav"])
            examples[audio_path] = {
                "speaker_id": speaker_id,
                "path": audio_path,
                "sentence": data[1],
            }

    inside_clips_dir = False
    id_ = 0

    for path, f in audio_files:
        if path.startswith(path_to_clips):
            inside_clips_dir = True
            if path in examples:
                audio = {"path": path, "bytes": f.read()}
                yield id_, {**examples[path], "audio": audio}
                id_ += 1
        elif inside_clips_dir:
            break
```

### Upload the dataset to the Hub

腳本準備就緒後，[create a dataset card](https://huggingface.co/docs/datasets/v2.14.1/en/dataset_card) 並將其 [upload it to the Hub](https://huggingface.co/docs/datasets/v2.14.1/en/share)。

恭喜，您現在可以從 Hub 加載數據集！

```python
from datasets import load_dataset

load_dataset("<username>/my_dataset")
```

### (Advanced) Extract TAR archives locally

在上面的示例中，未提取下載的存檔，因此示例不包含有關它們本地存儲位置的信息。為了解釋如何以支持流的方式進行提取，我們將簡要介紹 [LibriVox Indonesia](https://huggingface.co/datasets/indonesian-nlp/librivox-indonesia/blob/main/librivox-indonesia.py) 加載腳本。

#### Download and define the dataset splits

1. 使用 [`download()`](https://huggingface.co/docs/datasets/v2.14.1/en/package_reference/builder_classes#datasets.DownloadManager.download) 方法下載 `_AUDIO_URL` 的音頻數據。

2. 要在本地提取音頻 TAR 存檔，請使用 `extract()`。您只能在非流模式下使用此方法（當 `dl_manager.is_streaming=False` 時）。這將返回提取的存檔目錄的本地路徑：

    ```python
    local_extracted_archive = dl_manager.extract(audio_path) if not dl_manager.is_streaming else None
    ```

3. 使用 `iter_archive()` 方法迭代 `audio_path` 處的存檔，就像上面的 Vivos 示例一樣。`iter_archive()` 不提供有關存檔中文件的完整路徑的任何信息，即使它已被提取。因此，您需要將 `local_extracted_archive` 路徑傳遞到 `gen_kwargs` 中的下一步，以保留有關存檔提取到的位置的信息。這是在生成示例時構造本地文件的正確路徑所必需的。

    !!! warning
        需要結合使用 `download()` 和 `iter_archive()` 的原因是 TAR 存檔中的文件無法通過其路徑直接訪問。
        
        相反，您需要迭代存檔中的文件！您只能在非流模式下對 TAR 存檔使用 `download_and_extract()` 和 `extract()` ，否則會引發錯誤。

4. 使用 `download_and_extract()` 方法下載 `_METADATA_URL` 中指定的元數據文件。該方法返回非流模式下本地文件的路徑。在流模式下，它不會在本地下載文件並返回相同的 URL。

5. 現在使用 `SplitGenerator` 來組織每個分割中的音頻文件和元數據。使用標準名稱命名每個 split，例如：`Split.TRAIN`、`Split.TEST` 和 `SPLIT.Validation`。

    在 `gen_kwargs` 參數中，指定 `local_extracted_archive`、`audio_files`、`metadata_path` 和 `path_to_clips` 的文件路徑。請記住，對於 `audio_files`，您需要使用 `iter_archive()` 來迭代 TAR 存檔中的音頻文件。這將為您的數據集啟用流式傳輸！所有這些文件路徑都將傳遞到生成數據集樣本的下一步。

```python
def _split_generators(self, dl_manager):
    """Returns SplitGenerators."""
    dl_manager.download_config.ignore_url_params = True

    audio_path = dl_manager.download(_AUDIO_URL)
    local_extracted_archive = dl_manager.extract(audio_path) if not dl_manager.is_streaming else None
    path_to_clips = "librivox-indonesia"

    return [
        datasets.SplitGenerator(
            name=datasets.Split.TRAIN,
            gen_kwargs={
                "local_extracted_archive": local_extracted_archive,
                "audio_files": dl_manager.iter_archive(audio_path),
                "metadata_path": dl_manager.download_and_extract(_METADATA_URL + "/metadata_train.csv.gz"),
                "path_to_clips": path_to_clips,
            },
        ),
        datasets.SplitGenerator(
            name=datasets.Split.TEST,
            gen_kwargs={
                "local_extracted_archive": local_extracted_archive,
                "audio_files": dl_manager.iter_archive(audio_path),
                "metadata_path": dl_manager.download_and_extract(_METADATA_URL + "/metadata_test.csv.gz"),
                "path_to_clips": path_to_clips,
            },
        ),
    ]
```

#### Generate the dataset

這裡 `_generate_examples` 接受之前方法中的 `local_extracted_archive`、`audio_files`、`metadata_path` 和 `path_to_clips` 作為參數。

1. TAR 文件按順序訪問和生成。這意味著您需要首先擁有與 TAR 文件中的音頻文件關聯的 `metadata_path` 中的元數據，以便您可以進一步生成它及其相應的音頻文件：

    ```python
    with open(metadata_path, "r", encoding="utf-8") as f:
        reader = csv.DictReader(f)
        for row in reader:
            if self.config.name == "all" or self.config.name == row["language"]:
                row["path"] = os.path.join(path_to_clips, row["path"])
                # if data is incomplete, fill with empty values
                for field in data_fields:
                    if field not in row:
                        row[field] = ""
                metadata[row["path"]] = row
    ```

2. 現在您可以生成 `audio_files` 存檔中的文件。當您使用 `iter_archive()` 時，它會生成一個 `(path, f)` 元組，其中 `path` 是存檔內文件的相對路徑，`f` 是文件對象本身。要獲取本地提取文件的完整路徑，請將存檔提取到的目錄路徑 `(local_extracted_pa​​th)` 與相對音頻文件路徑 `(path)` 相連接：

    ```python
    for path, f in audio_files:
        if path in metadata:
            result = dict(metadata[path])
            # set the audio feature and the path to the extracted file
            path = os.path.join(local_extracted_archive, path) if local_extracted_archive else path
            result["audio"] = {"path": path, "bytes": f.read()}
            result["path"] = path
            yield id_, result
            id_ += 1    
    ```

將這兩個步驟放在一起，整個 `_generate_examples` 方法應如下所示：

```python
def _generate_examples(
        self,
        local_extracted_archive,
        audio_files,
        metadata_path,
        path_to_clips,
    ):
        """Yields examples."""
        data_fields = list(self._info().features.keys())
        metadata = {}

        with open(metadata_path, "r", encoding="utf-8") as f:
            reader = csv.DictReader(f)
            for row in reader:
                if self.config.name == "all" or self.config.name == row["language"]:
                    row["path"] = os.path.join(path_to_clips, row["path"])
                    # if data is incomplete, fill with empty values
                    for field in data_fields:
                        if field not in row:
                            row[field] = ""
                    metadata[row["path"]] = row
        id_ = 0

        for path, f in audio_files:
            if path in metadata:
                result = dict(metadata[path])
                # set the audio feature and the path to the extracted file
                path = os.path.join(local_extracted_archive, path) if local_extracted_archive else path
                result["audio"] = {"path": path, "bytes": f.read()}
                result["path"] = path
                yield id_, result
                id_ += 1
```