# Use with PyTorch

本文檔快速介紹瞭如何通過 PyTorch 使用數據集，特別關注如何從數據集中獲取 `torch.Tensor` 物件，以及如何使用 `PyTorch DataLoader` 和 Hugging Face 數據集以實現最佳性能。

## Dataset format

默認情況下，數據集返回常規 Python 物件：integers, floats, strings, lists 等。

要獲取 PyTorch tensors，您可以使用 [`Dataset.with_format()`](https://huggingface.co/docs/datasets/v2.14.1/en/package_reference/main_classes#datasets.Dataset.with_format) 將數據集的格式設置為 `pytorch`：

```python
from datasets import Dataset

data = [[1, 2],[3, 4]]

ds = Dataset.from_dict({"data": data})

ds = ds.with_format("torch")

print(ds[0])

print(ds[:2])
```

結果:

```bash
{'data': tensor([1, 2])}

{'data': tensor([[1, 2],
         [3, 4]])}
```

!!! info
    `Dataset` 物件是 Apache Arrow table 的 wrapper，它允許從數據集中的 array 轉換成 PyTorch tensor的快速零複製讀取。

要將數據作為 tensor 加載到 GPU 上，請指定 `device` 參數：

```python
import torch

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

ds = ds.with_format("torch", device=device)

print(ds[0])
```

結果:

```bash
{'data': tensor([1, 2], device='cuda:0')}
```

## N-dimensional arrays

如果您的數據集由 N 維陣例組成，您將看到默認情況下它們被視為嵌套列表。特別是，PyTorch 格式的數據集輸出嵌套列表而不是單個張量：

```python
from datasets import Dataset

data = [[[1, 2],[3, 4]],[[5, 6],[7, 8]]]

ds = Dataset.from_dict({"data": data})

ds = ds.with_format("torch")

print(ds[0])
```

結果:

```bash
{'data': [tensor([1, 2]), tensor([3, 4])]}
```

要獲取單個張量，您必須顯式使用 `Array` 特徵類型並指定張量的 shape：

```python
from datasets import Dataset, Features, Array2D

data = [[[1, 2],[3, 4]],[[5, 6],[7, 8]]]

features = Features({"data": Array2D(shape=(2, 2), dtype='int32')})

ds = Dataset.from_dict({"data": data}, features=features)

ds = ds.with_format("torch")

print(ds[0])

print(ds[:2])
```

結果:

```bash
{'data': tensor([[1, 2],
         [3, 4]])}

{'data': tensor([[[1, 2],
          [3, 4]],
 
         [[5, 6],
          [7, 8]]])}
```


## Other feature types

[ClassLabel](https://huggingface.co/docs/datasets/v2.14.1/en/package_reference/main_classes#datasets.ClassLabel) 數據已正確轉換為張量：

```python
from datasets import Dataset, Features, ClassLabel

labels = [0, 0, 1]

features = Features({"label": ClassLabel(names=["negative", "positive"])})

ds = Dataset.from_dict({"label": labels}, features=features)

ds = ds.with_format("torch")

print(ds[:3])
```

結果:

```bash
{'label': tensor([0, 0, 1])}
```

String 和 binary 物件保持不變，因為 PyTorch 僅支持 numbers。

還支持 [Image](https://huggingface.co/docs/datasets/v2.14.0/en/package_reference/main_classes#datasets.Image) 和 [Audio](https://huggingface.co/docs/datasets/v2.14.0/en/package_reference/main_classes#datasets.Audio) feature types。

!!! tip
    要使用[Image](https://huggingface.co/docs/datasets/v2.14.0/en/package_reference/main_classes#datasets.Image) feature type，您需要安裝額外的 `vision` 套件。
    
    ```python
    pip install datasets[vision]
    ```

```python
from datasets import Dataset, Features, Audio, Image

images = ["path/to/image.png"] * 10

features = Features({"image": Image()})

ds = Dataset.from_dict({"image": images}, features=features) 

ds = ds.with_format("torch")

print(ds[0]["image"].shape)

print(ds[0])

print(ds[:2]["image"].shape)

print(ds[:2])
```

結果:

```bash
torch.Size([512, 512, 4])

{'image': tensor([[[255, 215, 106, 255],
         [255, 215, 106, 255],
         ...,
         [255, 255, 255, 255],
         [255, 255, 255, 255]]], dtype=torch.uint8)}

torch.Size([2, 512, 512, 4])

{'image': tensor([[[[255, 215, 106, 255],
          [255, 215, 106, 255],
          ...,
          [255, 255, 255, 255],
          [255, 255, 255, 255]]]], dtype=torch.uint8)}
```


!!! tip
    要使用[Audio](https://huggingface.co/docs/datasets/v2.14.0/en/package_reference/main_classes#datasets.Audio) feature type，您需要安裝額外的 `audio` 套件。
    
    ```python
    pip install datasets[audio]
    ```

```python
from datasets import Dataset, Features, Audio, Image

audio = ["path/to/audio.wav"] * 10

features = Features({"audio": Audio()})

ds = Dataset.from_dict({"audio": audio}, features=features)

ds = ds.with_format("torch")

print(ds[0]["audio"]["array"])

print(ds[0]["audio"]["sampling_rate"])
```

結果:

```bash
tensor([ 6.1035e-05,  1.5259e-05,  1.6785e-04,  ..., -1.5259e-05,
        -1.5259e-05,  1.5259e-05])

tensor(44100)
```

## Data loading

與 `torch.utils.data.Dataset` 物件，數據集可以直接傳遞給 PyTorch `DataLoader`：

```python
import numpy as np
from datasets import Dataset 
from torch.utils.data import DataLoader

data = np.random.rand(16)

label = np.random.randint(0, 2, size=16)

ds = Dataset.from_dict({"data": data, "label": label}).with_format("torch")

dataloader = DataLoader(ds, batch_size=4)

for batch in dataloader:
    print(batch)                                                              
```

結果:

```bash
{'data': tensor([0.0047, 0.4979, 0.6726, 0.8105]), 'label': tensor([0, 1, 0, 1])}
{'data': tensor([0.4832, 0.2723, 0.4259, 0.2224]), 'label': tensor([0, 0, 0, 0])}
{'data': tensor([0.5837, 0.3444, 0.4658, 0.6417]), 'label': tensor([0, 1, 0, 0])}
{'data': tensor([0.7022, 0.1225, 0.7228, 0.8259]), 'label': tensor([1, 1, 1, 1])}
```

### Optimize data loading

有多種方法可以提高數據加載速度，從而節省時間，特別是在處理大型數據集時。 PyTorch 提供並行數據加載、批量檢索索引而不是單獨檢索索引，以及流式傳輸以迭代數據集而無需將其下載到磁盤上。

#### Use multiple Workers

您可以使用 PyTorch `DataLoader` 的 `num_workers` 參數並行化數據加載，並獲得更高的吞吐量。

在背後，`DataLoader` 啟動 `num_workers` 進程。每個進程都會重新加載傳遞給 `DataLoader` 的數據集並用於查詢 examples。在工作進程中重新加載數據集不會填滿您的 RAM，因為它只是從磁盤再次對數據集進行內存映射。

```python
import numpy as np
from datasets import Dataset, load_from_disk
from torch.utils.data import DataLoader

data = np.random.rand(10_000)

Dataset.from_dict({"data": data}).save_to_disk("my_dataset")

ds = load_from_disk("my_dataset").with_format("torch")

dataloader = DataLoader(ds, batch_size=32, num_workers=4)
```

### Stream data

通過將數據集加載為 `IterableDataset` 來流式傳輸數據集。這允許您逐步迭代遠程數據集，而無需將其下載到磁盤和/或本地數據文件上。[在常規數據集或可迭代數據集之間進行選擇時](https://huggingface.co/docs/datasets/about_mapstyle_vs_iterable)，詳細了解哪種類型的數據集最適合您的用例。

datasets 中的可迭代數據集繼承自 `torch.utils.data.IterableDataset`，因此您可以將其直接傳遞給 `torch.utils.data.DataLoader`　來使用：

```python
import numpy as np
from datasets import Dataset, load_dataset
from torch.utils.data import DataLoader

data = np.random.rand(10_000)

Dataset.from_dict({"data": data}).push_to_hub("<username>/my_dataset")  # Upload to the Hugging Face Hub

my_iterable_dataset = load_dataset("<username>/my_dataset", streaming=True, split="train")

dataloader = DataLoader(my_iterable_dataset, batch_size=32)
```

如果數據集分為多個分片（即，如果數據集由多個數據文件組成），則可以使用 `num_workers` 並行流式傳輸：

```python
my_iterable_dataset = load_dataset("c4", "en", streaming=True, split="train")

print(my_iterable_dataset.n_shards)

dataloader = DataLoader(my_iterable_dataset, batch_size=32, num_workers=4)
```

結果:

```bash
1024
```

在這種情況下，每個 worker 進程都會獲得要從中進行流式傳輸的分片列表的子集。

###　Distributed

要將數據集拆分到訓練節點上，您可以使用 [`datasets.distributed.split_dataset_by_node()`](https://huggingface.co/docs/datasets/v2.14.1/en/package_reference/main_classes#datasets.distributed.split_dataset_by_node)：

```python
import os
from datasets.distributed import split_dataset_by_node

ds = split_dataset_by_node(ds, rank=int(os.environ["RANK"]), world_size=int(os.environ["WORLD_SIZE"]))
```

這適用於 map-style 數據集和 iterable datasets。數據集被分割為大小為 `world_size` 的節點池中排名為 `Rank` 的節點。

**map-style 數據集**:

每個節點都被分配了一塊數據，例如 rank 0 被賦予數據集的第一個塊。

**iterable datasets**:

如果數據集的分片數量是 `world_size` 的一個 factor（即，如果 `dataset.n_shards % world_size == 0`），則分片在節點之間均勻分配，這是最優化的。否則，每個節點在 `world_size` 中保留 1 個 example，並跳過其他 examples。


如果您希望每個節點使用多個工作線程來加載數據，也可以將其與 `torch.utils.data.DataLoader` 結合使用。