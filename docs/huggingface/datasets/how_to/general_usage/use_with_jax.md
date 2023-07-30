# Use with JAX

本文檔快速介紹瞭如何將數據集與 JAX 結合使用，特別關注如何從數據集中獲取 `jax.Array` 物件，以及如何使用它們來訓練 JAX 模型。

!!! info
    需要 `jax` 和 `jaxlib` 才能運行本教程的相關範例程式碼，因此請確保相關套件的安裝與執行 `pip install datasets[jax]`。

##　Dataset format

默認情況下，數據集返回常規 Python 物件：integers, floats, strings, lists　等，string 和 binary 物件不變，因為 JAX 僅支持 numbers。

要獲取 JAX 陣列（類似 numpy），您可以將數據集的格式設置為 `jax`：

```python
from datasets import Dataset

data = [[1, 2], [3, 4]]

ds = Dataset.from_dict({"data": data})

ds = ds.with_format("jax")

print(ds[0])

print(ds[:2])
```

結果:

```bash
{'data': DeviceArray([1, 2], dtype=int32)}

{'data': DeviceArray([
    [1, 2],
    [3, 4]], dtype=int32)}
```

!!! info
    `Dataset` 物件是 Apache Arrow table 的 wrapper，它允許從數據集中的 array 轉換成 JAX arrays。

請注意，完全相同的過程適用於 `DatasetDict` 物件，因此當將 `DatasetDict` 的格式設置為 `jax` 時，所有數據集都將被格式化為 `jax`：

```python
from datasets import DatasetDict

data = {"train": {"data": [[1, 2], [3, 4]]}, "test": {"data": [[5, 6], [7, 8]]}}

dds = DatasetDict.from_dict(data)

dds = dds.with_format("jax")

print(dds["train"][:2])
```

結果:

```bash
{'data': DeviceArray([
    [1, 2],
    [3, 4]], dtype=int32)}
```

您需要考慮的另一件事是，在您實際訪問數據之前，格式的轉換不會被執行。因此，如果您想從數據集中獲取 JAX 陣列，則需要首先訪問數據，否則格式將保持不變。

最後，要將數據加載到您選擇的設備中，您可以指定 `device` 參數，但請注意，不支持 `jaxlib.xla_extension.Device`，因為它不能用 `pickle` 或 `dill` 進行序列化，因此您需要使用它的字符串標識符改為：

```python
import jax
from datasets import Dataset

data = [[1, 2], [3, 4]]

ds = Dataset.from_dict({"data": data})

device = str(jax.devices()[0])  # Not casting to `str` before passing it to `with_format` will raise a `ValueError`

ds = ds.with_format("jax", device=device)

print(ds[0])

print(ds[0]["data"].device())
```

結果:

```bash
{'data': DeviceArray([1, 2], dtype=int32)}

TFRT_CPU_0
```

請注意，如果沒有向 `with_format` 提供設備參數，那麼它將使用默認設備 `jax.devices()[0]`。

## N-dimensional arrays

如果您的數據集由 N 維數組組成，您將看到默認情況下它們被視為嵌套列表。特別是，JAX 格式的數據集輸出一個 `DeviceArray` 物件，它是一個類似 `numpy` 的陣列，因此與 PyTorch 或 TensorFlow 格式化程序不同，它不需要指定數組特徵類型。

```python
from datasets import Dataset

data = [[[1, 2],[3, 4]], [[5, 6],[7, 8]]]

ds = Dataset.from_dict({"data": data})

ds = ds.with_format("jax")

print(ds[0])
```

結果:

```bash
{'data': DeviceArray([[1, 2],
             [3, 4]], dtype=int32)}
```

## Other feature types

[ClassLabel](https://huggingface.co/docs/datasets/v2.14.1/en/package_reference/main_classes#datasets.ClassLabel) 數據已正確轉換為 `arrays`：

```python
from datasets import Dataset, Features, ClassLabel

labels = [0, 0, 1]

features = Features({"label": ClassLabel(names=["negative", "positive"])})

ds = Dataset.from_dict({"label": labels}, features=features)

ds = ds.with_format("jax")

print(ds[:3])
```

結果:

```bash
{'label': DeviceArray([0, 0, 1], dtype=int32)}
```

String 和 binary 物件未更改，因為 JAX 僅支持　numbers。

還支持 [Image](https://huggingface.co/docs/datasets/v2.14.0/en/package_reference/main_classes#datasets.Image) 和 [Audio](https://huggingface.co/docs/datasets/v2.14.0/en/package_reference/main_classes#datasets.Audio) feature types。

!!! tip
    要使用[Image](https://huggingface.co/docs/datasets/v2.14.0/en/package_reference/main_classes#datasets.Image) feature type，您需要安裝額外的 `vision` 套件。
    
    ```python
    pip install datasets[vision]
    ```

```python
from datasets import Dataset, Features, Image

images = ["path/to/image.png"] * 10

features = Features({"image": Image()})

ds = Dataset.from_dict({"image": images}, features=features)

ds = ds.with_format("jax")

print(ds[0]["image"].shape)

print(ds[0])

print(ds[:2]["image"].shape)

print(ds[:2])
```

結果:

```bash
(512, 512, 3)

{'image': DeviceArray([[[ 255, 255, 255],
              [ 255, 255, 255],
              ...,
              [ 255, 255, 255],
              [ 255, 255, 255]]], dtype=uint8)}

(2, 512, 512, 3)

{'image': DeviceArray([[[[ 255, 255, 255],
              [ 255, 255, 255],
              ...,
              [ 255, 255, 255],
              [ 255, 255, 255]]]], dtype=uint8)}
```

!!! tip
    要使用[Audio](https://huggingface.co/docs/datasets/v2.14.0/en/package_reference/main_classes#datasets.Audio) feature type，您需要安裝額外的 `audio` 套件。
    
    ```python
    pip install datasets[audio]
    ```

```python
from datasets import Dataset, Features, Audio

audio = ["path/to/audio.wav"] * 10

features = Features({"audio": Audio()})

ds = Dataset.from_dict({"audio": audio}, features=features)

ds = ds.with_format("jax")

print(ds[0]["audio"]["array"])

print(ds[0]["audio"]["sampling_rate"])
```

結果:

```bash
DeviceArray([-0.059021  , -0.03894043, -0.00735474, ...,  0.0133667 ,
              0.01809692,  0.00268555], dtype=float32)

DeviceArray(44100, dtype=int32, weak_type=True)
```

## Data loading

JAX 沒有任何內置數據加載功能，因此您需要使用 PyTorch 等套件來使用 `DataLoader` 或 TensorFlow 使用 `tf.data.Dataset` 來加載數據。

引用有關此主題的 [JAX 文檔](https://jax.readthedocs.io/en/latest/notebooks/Neural_Network_and_Data_Loading.html#data-loading-with-pytorch)：“JAX 專注於程序轉換和加速器支持的 NumPy，因此我們不在 JAX 庫中包含數據加載或修改。已經有很多很棒的數據加載器了，所以讓我們使用它們而不是重新發明任何東西。我們將獲取 PyTorch 的數據加載器，並製作一個小墊片以使其與 NumPy 數組一起使用。”

這就是數據集中的 JAX 格式如此有用的原因，因為它允許您通過 JAX 使用 HuggingFace Hub 中的任何模型，而不必擔心數據加載部分。

###　Using `with_format('jax')`

從數據集中獲取 JAX 數組的最簡單方法是使用 `with_format('jax')` 方法。假設我們想要在 HuggingFace Hub 上提供的 [MNIST 數據集](https://huggingface.co/datasets/mnist) 上訓練神經網絡。

```python
from datasets import load_dataset

ds = load_dataset("mnist")

ds = ds.with_format("jax")

print(ds["train"][0])
```

結果:

```bash
{'image': DeviceArray([[  0,   0,   0, ...],
                       [  0,   0,   0, ...],
                       ...,
                       [  0,   0,   0, ...],
                       [  0,   0,   0, ...]], dtype=uint8),
 'label': DeviceArray(5, dtype=int32)}
```

設置格式後，我們可以使用 `Dataset.iter()` 方法將數據集批量提供給 JAX 模型：

```python
for epoch in range(epochs):
    for batch in ds["train"].iter(batch_size=32):
        x, y = batch["image"], batch["label"]
        ...
```