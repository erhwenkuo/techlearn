# Load audio data

您可以使用 [Audio](https://huggingface.co/docs/datasets/v2.14.1/en/package_reference/main_classes#datasets.Audio) feature 加載音頻數據集，該功能在您訪問樣本時會自動解碼音頻文件並重新採樣。音頻解碼基於 [soundfile](https://github.com/bastibe/python-soundfile) python 套件，它在底層使用 [libsndfile](https://github.com/libsndfile/libsndfile) C 函式庫。

##　Installation

要使用音頻數據集，您需要安裝音頻依賴項。查看[安裝指南](https://huggingface.co/docs/datasets/v2.14.1/en/installation#audio)以了解如何安裝。

```bash
pip install datasets[audio]
```

## Local files

您可以使用音頻文件的路徑加載自己的數據集。使用 `cast_column()` 函數獲取一列音頻文件路徑，並將其投射到音頻功能：

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

## AudioFolder

您還可以使用 `AudioFolder` 數據集生成器加載數據集。它不需要編寫自定義數據加載器，因此可用於快速創建和加載包含數千個音頻文件的音頻數據集。

## AudioFolder with metadata

要將音頻文件與元數據信息鏈接，請確保您的數據集具有 `metadata.csv` 文件。您的數據集結構可能如下所示：

```bash
folder/train/metadata.csv
folder/train/first_audio_file.mp3
folder/train/second_audio_file.mp3
folder/train/third_audio_file.mp3
```

您的 `metadata.csv` 文件必須有一個 `file_name` 列，用於鏈接音頻文件及其元數據。範例 `metadata.csv` 文件可能如下所示：

```csv "metadata.csv"
file_name,transcription
first_audio_file.mp3,znowu się duch z ciałem zrośnie w młodocianej wstaniesz wiosnie i możesz skutkiem tych leków umierać wstawać wiek wieków dalej tam były przestrogi jak siekać głowę jak nogi
second_audio_file.mp3,już u źwierzyńca podwojów król zasiada przy nim książęta i panowie rada a gdzie wzniosły krążył ganek rycerze obok kochanek król skinął palcem zaczęto igrzysko
third_audio_file.mp3,pewnie kędyś w obłędzie ubite minęły szlaki zaczekajmy dzień jaki poślemy szukać wszędzie dziś jutro pewnie będzie posłali wszędzie sługi czekali dzień i drugi gdy nic nie doczekali z płaczem chcą jechać dali
```

AudioFolder 將加載音頻數據並創建一個包含來自 `metadata.csv` 的文本的轉錄列：

```bash
from datasets import load_dataset

dataset = load_dataset("audiofolder", data_dir="/path/to/folder")

# OR by specifying the list of files
dataset = load_dataset("audiofolder", data_files=["path/to/audio_1", "path/to/audio_2", ..., "path/to/audio_n"])
```

您可以使用 `data_files` 參數從 URL 加載遠程數據集：

```python
dataset = load_dataset("audiofolder", data_files=["https://foo.bar/audio_1", "https://foo.bar/audio_2", ..., "https://foo.bar/audio_n"]

# for example, pass SpeechCommands archive:
dataset = load_dataset("audiofolder", data_files="https://s3.amazonaws.com/datasets.huggingface.co/SpeechCommands/v0.01/v0.01_test.tar.gz")
```

Metadata 還可以指定為 JSON 行，在這種情況下，使用 `metadata.jsonl` 作為元數據文件的名稱。這種格式在其中一列很複雜的情況下很有用，例如浮點數列表，以避免解析錯誤或將復雜值讀取為字符串。

要忽略元數據文件中的信息，請在 `load_dataset()` 中設置 `drop_metadata=True`：

```python
from datasets import load_dataset

dataset = load_dataset("audiofolder", data_dir="/path/to/folder", drop_metadata=True)
```

如果您沒有元數據文件，AudioFolder 會自動從目錄名稱推斷標籤名稱。如果要刪除自動創建的標籤，請設置 `drop_labels=True`。在這種情況下，您的數據集將僅包含音頻列：

```python
from datasets import load_dataset

dataset = load_dataset("audiofolder", data_dir="/path/to/folder_without_metadata", drop_labels=True)
```

!!! tip
    有關創建您自己的 AudioFolder 數據集的更多信息，請查看 [Create an audio dataset](https://huggingface.co/docs/datasets/v2.14.1/en/audio_dataset) 指南。