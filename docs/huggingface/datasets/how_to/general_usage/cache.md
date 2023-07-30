# Cache management

下載數據集時，processing scripts 和 data 都會存儲在您的本地計算機上。Cache 允許🤗數據集避免在每次使用時重新下載或處理整個數據集。

本指南將向您展示如何：

- 更改緩存目錄。
- 控制如何從緩存加載數據集。
- 清理目錄中的緩存文件。
- 啟用或禁用緩存

## Cache directory

默認緩存目錄是 `~/.cache/huggingface/datasets`。通過將 shell 環境變量 `HF_DATASETS_CACHE` 設置為另一個目錄來更改緩存位置：

```bash
$ export HF_DATASETS_CACHE="/path/to/another/directory"
```

加載數據集時，您還可以選擇更改數據的緩存位置。將 `cache_dir` 參數更改為您想要的路徑：

```python
from datasets import load_dataset

dataset = load_dataset('LOADING_SCRIPT', cache_dir="PATH/TO/MY/CACHE/DIR")
```

同樣，您可以使用 `cache_dir` 參數更改指標的緩存位置：

```python
from datasets import load_metric

metric = load_metric('glue', 'mrpc', cache_dir="MY/CACHE/DIRECTORY")
```

## Download mode

下載數據集後，使用 `download_mode` 參數控制 `load_dataset()` 加載數據的方式。默認情況下，🤗 數據集將重用存在的數據集。但如果您需要未應用任何處理函數的原始數據集，請重新下載文件，如下所示：

```python
from datasets import load_dataset

dataset = load_dataset('squad', download_mode='force_redownload')
```

有關下載模式的完整列表，請參閱 [DownloadMode](https://huggingface.co/docs/datasets/v2.14.1/en/package_reference/builder_classes#datasets.DownloadMode)。

## Cache files

使用 `Dataset.cleanup_cache_files()` 清理目錄中的緩存文件：

```python
# Returns the number of removed cache files
dataset.cleanup_cache_files()
```

## Enable or disable caching

如果您在本地使用緩存文件，它將自動重新加載數據集以及您之前應用於數據集的任何轉換。通過在 `Dataset.map()` 中設置參數 `load_from_cache_file=False` 禁用此行為：

```python
updated_dataset = small_dataset.map(add_prefix, load_from_cache_file=False)
```

在上面的範例中，🤗 Datasets 將再次對整個數據集執行 `add_prefix` 函數，而不是從之前的狀態加載數據集。

使用 `disable_caching()` 在全局範圍內禁用緩存：

```python
from datasets import disable_caching

disable_caching()
```

當您禁用緩存時，🤗 在將轉換應用於數據集時，數據集將不再重新加載緩存的文件。您對數據集應用的任何轉換都需要重新應用。

!!! tip
    如果您想從頭開始重用數據集，請嘗試在 `load_dataset()` 中設置 `download_mode` 參數。

您還可以完全避免緩存指標，並將其保存在 CPU 內存中：

```python
from datasets import load_metric

metric = load_metric('glue', 'mrpc', keep_in_memory=True)
```

## Improve performance

禁用緩存並在內存中復制數據集將加快數據集操作的速度。有兩種用於在內存中復制數據集的選項：

1. 將 `datasets.config.IN_MEMORY_MAX_SIZE` 設置為適合 RAM 內存的非零值（以 byte 為單位）。
2. 將環境變量 `HF_DATASETS_IN_MEMORY_MAX_SIZE` 設置為非零值。請注意，第一種方法具有更高的優先級。