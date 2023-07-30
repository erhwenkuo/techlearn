# Process image data

本指南展示了處理圖像數據集的具體方法。了解如何：

- 對　image dataset　使用 `map()`
- 使用 `set_transform()` 來對 image dataset 進行　data augmentations

##　Map

`map()` 函數可以對整個數據集應用變換。

例如，創建一個基本的 `Resize` 函數：

```python
def transforms(examples):
    examples["pixel_values"] = [image.convert("RGB").resize((100,100)) for image in examples["image"]]
    return examples
```

現在使用 `map()` 函數調整整個數據集的大小，並設置 `batched=True` 以通過接受批量示例來加速該過程。轉換返回 `pixel_values` 作為可緩存的 PIL.Image object：

```python
dataset = dataset.map(transforms, remove_columns=["image"], batched=True)

print(dataset[0])
```

結果:

```bash
{'label': 6,
 'pixel_values': <PIL.PngImagePlugin.PngImageFile image mode=RGB size=100x100 at 0x7F058237BB10>}
```

緩存文件可以節省時間，因為您不必執行相同的轉換兩次。 `map()` 函數最適合每次訓練只運行一次的操作（例如調整圖像大小），而不是將其用於每個 epoch 執行的操作（例如數據增強）。

`map()` 會佔用一些內存，但您可以通過以下參數減少其內存需求：

- `batch_size` 確定一次調用轉換函數時處理的樣本數據數量。
- `writer_batch_size` 確定在存儲之前保存在內存中的已處理樣本的數量。

這兩個參數值都默認為 1000，如果您要存儲圖像，這可能會很耗時。使用 `map()` 時，降低這些值可使用更少的內存。

## Apply transforms

🤗 數據集將任何庫或包中的數據增強應用到您的數據集。可以使用 `set_transform()` 對批量數據即時應用轉換，這會消耗更少的磁盤空間。

!!! info
    以下範例使用 `torchvision`，但您可以隨意使用其他數據增強庫，例如 Albumentations、Kornia 和 imgaug。

例如，如果您想隨機更改圖像的顏色屬性：

```python
from torchvision.transforms import Compose, ColorJitter, ToTensor

jitter = Compose(
    [
         ColorJitter(brightness=0.25, contrast=0.25, saturation=0.25, hue=0.7),
         ToTensor(),
    ]
)
```

創建一個函數來應用 ColorJitter 變換：

```python
def transforms(examples):
    examples["pixel_values"] = [jitter(image.convert("RGB")) for image in examples["image"]]
    return examples
```

使用 `set_transform()` 函數應用變換：

```python
dataset.set_transform(transforms)
```