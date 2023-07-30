# Using Datasets with TensorFlow

本文檔快速介紹瞭如何在 TensorFlow 中使用數據集，特別關注如何從數據集中獲取 `tf.Tensor` 物件，以及如何將數據從 Hugging Face `Dataset` 物件將數據流式傳輸到 Keras method（如 `model.fit()`）。

## Dataset format

默認情況下，數據集返回常規 Python 物件: `integers`, `floats`, `strings`, `lists` 等。

要獲取 TensorFlow `tensor`，您可以將數據集的 format 設置為 `tf`：

```python
from datasets import Dataset

data = [[1, 2],[3, 4]]

ds = Dataset.from_dict({"data": data})

ds = ds.with_format("tf")

print(ds[0])

print(ds[:2])
```

```bash
{'data': <tf.Tensor: shape=(2,), dtype=int64, numpy=array([1, 2])>}

{'data': <tf.Tensor: shape=(2, 2), dtype=int64, numpy=
array([[1, 2],
       [3, 4]])>}
```

!!! info
    Dataset 物件是 Apache Arrow table 的 wrapper，它允許從數據集中的 array 快速轉換成 TensorFlow tensor。

這對於將數據集轉換為 Tensor 物件的字典，或者編寫 generator 以從中加載 `TF samples` 非常有用。如果您希望將整個數據集轉換為 Tensor，只需查詢完整數據集：

```python
print(ds[:])
```

```bash
{'data': <tf.Tensor: shape=(2, 2), dtype=int64, numpy=
array([[1, 2],
       [3, 4]])>}
```

## N-dimensional arrays

如果您的數據集由 N 維陣列組成，您將看到默認情況下它們被視為嵌套列表。特別是，TensorFlow 格式的數據集輸出 `RaggedTensor`，而不是單個 tensor：

```python
from datasets import Dataset

data = [[[1, 2],[3, 4]],[[5, 6],[7, 8]]]

ds = Dataset.from_dict({"data": data})

ds = ds.with_format("tf")

print(ds[0])
```

```bash
{'data': <tf.RaggedTensor [[1, 2], [3, 4]]>}
```

要獲取單個 tensor，您必須顯式使用 `Array` feature 類型並指定張量的 shape：

```python
from datasets import Dataset, Features, Array2D

data = [[[1, 2],[3, 4]],[[5, 6],[7, 8]]]

features = Features({"data": Array2D(shape=(2, 2), dtype='int32')})

ds = Dataset.from_dict({"data": data}, features=features)

ds = ds.with_format("tf")

print(ds[0])

print(ds[:2])
```

結果:

```bash
{'data': <tf.Tensor: shape=(2, 2), dtype=int64, numpy=
 array([[1, 2],
        [3, 4]])>}

{'data': <tf.Tensor: shape=(2, 2, 2), dtype=int64, numpy=
 array([[[1, 2],
         [3, 4]],
 
        [[5, 6],
         [7, 8]]])>}
```

## Other feature types

[ClassLabel](https://huggingface.co/docs/datasets/v2.14.0/en/package_reference/main_classes#datasets.ClassLabel) 數據已正確轉換為張量：

```python
from datasets import Dataset, Features, ClassLabel

labels = [0, 0, 1]

features = Features({"label": ClassLabel(names=["negative", "positive"])})

ds = Dataset.from_dict({"label": labels}, features=features) 

ds = ds.with_format("tf")  

print(ds[:3])
```

結果:

```bash
{'label': <tf.Tensor: shape=(3,), dtype=int64, numpy=array([0, 0, 1])>}
```

還支持 Strings 和 binary 物件：

```python
from datasets import Dataset, Features

text = ["foo", "bar"]
data = [0, 1] 

ds = Dataset.from_dict({"text": text, "data": data})

ds = ds.with_format("tf")

print(ds[:2])
```

結果:

```bash
{'text': <tf.Tensor: shape=(2,), dtype=string, numpy=array([b'foo', b'bar'], dtype=object)>,
 'data': <tf.Tensor: shape=(2,), dtype=int64, numpy=array([0, 1])>}
```

您還可以顯式格式化某些列並保留其他列未格式化：

```python
ds = ds.with_format("tf", columns=["data"], output_all_columns=True)

print(ds[:2])
```

結果:

```bash
{'data': <tf.Tensor: shape=(2,), dtype=int64, numpy=array([0, 1])>,
 'text': ['foo', 'bar']}
```

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

ds = ds.with_format("tf")

print(ds[0])

print(ds[:2])
```


結果:

```bash
{'image': <tf.Tensor: shape=(512, 512, 4), dtype=uint8, numpy=
 array([[[255, 215, 106, 255],
         [255, 215, 106, 255],
         ...,
         [255, 255, 255, 255],
         [255, 255, 255, 255]]], dtype=uint8)>}

{'image': <tf.Tensor: shape=(2, 512, 512, 4), dtype=uint8, numpy=
 array([[[[255, 215, 106, 255],
          [255, 215, 106, 255],
          ...,
          [255, 255, 255, 255],
          [255, 255, 255, 255]]]], dtype=uint8)>}
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

ds = ds.with_format("tf")

print(ds[0]["audio"]["array"])

print(ds[0]["audio"]["sampling_rate"])
```


結果:

```bash
<tf.Tensor: shape=(202311,), dtype=float32, numpy=
array([ 6.1035156e-05,  1.5258789e-05,  1.6784668e-04, ...,
       -1.5258789e-05, -1.5258789e-05,  1.5258789e-05], dtype=float32)>

<tf.Tensor: shape=(), dtype=int32, numpy=44100>
```

## Data loading

儘管您可以通過索引到數據集來加載單個 samples 和 batches，但如果您想使用 `fit()` 和 `predict()` 等 Keras methods，這將不起作用。

您可以編寫一個 generator 函數，從數據集中重新排序和加載批次，並對其進行 `fit()` ，但這聽起來像是很多不必要的工作。

相反，如果您想實時從數據集中流式傳輸數據，我們建議使用 `to_tf_dataset()` 方法將數據集轉換為 `tf.data.Dataset`。

`tf.data.Dataset` 類別涵蓋了廣泛的用例 - 它通常是從內存中的張量創建的，或者使用加載函數讀取磁盤或外部存儲上的文件。可以使用 `map()` 方法任意轉換數據集，或者可以使用 `batch()` 和 `shuffle()` 等方法來創建準備訓練的數據集。

這些方法不會以任何方式修改存儲的數據 - 相反，這些方法構建一個 data pipeline graph，該 data pipeline graph 將在迭代數據集時執行，通常是在模型訓練或推理期間。這與 Hugging Face Dataset 對象的 `map()` 方法不同，後者立即運行 map 函數並保存新的或更改的列。

由於整個數據預處理管道可以在 `tf.data.Dataset` 中編譯，因此這種方法允許大規模並行、異步數據加載和訓練。

然而，graph 編譯的要求可能是一個限制，特別是對於 Hugging Face 分詞器(tokenizer)來說，它們還不能作為 TF graph 的一部分進行編譯。

因此，我們通常建議將數據集預處理為 Hugging Face dataset，其中可以使用任意 Python 函數，然後使用 `to_tf_dataset()` 轉換為 `tf.data.Dataset`，以獲得準備用於訓練的批處理數據集。

要查看此方法的示例，請參閱 transformers 的 [examples](https://github.com/huggingface/transformers/tree/main/examples) 或 [notebooks](https://huggingface.co/docs/transformers/notebooks)。

### Using to_tf_dataset()

使用 `to_tf_dataset()` 很簡單。一旦您的數據集經過預處理並準備就緒，只需像這樣調用它：

```python
from datasets import Dataset

data = {"inputs": [[1, 2],[3, 4]], "labels": [0, 1]}

ds = Dataset.from_dict(data)

tf_ds = ds.to_tf_dataset(
            columns=["inputs"],
            label_cols=["labels"],
            batch_size=2,
            shuffle=True
            )
```

這裡返回的 `tf_ds` 物件現在已完全準備好來進行訓練，並且可以直接傳遞給 `model.fit()`。請注意，您在創建數據集時設置了批量大小，因此在調用 `fit()` 時無需再指定它：

```python
model.fit(tf_ds, epochs=2)
```

有關參數的完整說明，請參閱 [`to_tf_dataset()`](https://huggingface.co/docs/datasets/v2.14.0/en/package_reference/main_classes#datasets.Dataset.to_tf_dataset) 文檔。

在許多情況下，您還需要在調用中添加 `collat​​e_fn`。該函數獲取數據集的多個元素並將它們組合成一個批次。當所有元素具有相同的長度時，內置的默認排序器就足夠了，但對於更複雜的任務，可能需要自定義排序器。

特別是，許多任務具有不同序列長度的樣本，這將需要一個可以正確填充批次的 [data collator](https://huggingface.co/docs/transformers/main/en/main_classes/data_collator)。您可以在 Transformer NLP [examples](https://github.com/huggingface/transformers/tree/main/examples) 或 [notebooks](https://huggingface.co/docs/transformers/notebooks) 中看到這方面的範例，其中可變序列長度非常常見。

如果您發現使用 `to_tf_dataset` 加載速度很慢，您還可以使用 `num_workers` 參數。這會啟動多個子進程以並行加載數據。此功能是最近才推出的，仍然處於實驗階段 - 如果您在使用它時遇到任何錯誤，請提出問題！

### When to use to_tf_dataset

精明的讀者可能已經註意到，我們提供了兩種方法來實現相同的目標 - 如果您想將數據集傳遞到 TensorFlow 模型，您可以使用 `.with_format('tf')` 來將數據集轉換為張量或張量字典，或者您可以使用 `to_tf_dataset()` 將數據集轉換為 `tf.data.Dataset`。

其中任何一個都可以傳遞給 `model.fit()`，那麼您應該選擇哪一種方法呢?

需要認識到的關鍵一點是，當您將整個數據集轉換為張量時，它是靜態的並完全加載到 RAM 中。這既簡單又方便，但如果適用以下任何一種情況，您可能應該使用 `to_tf_dataset()`：

- 您的數據集太大，RAM 無法容納。 `to_tf_dataset()` 一次僅傳輸一批數據，因此即使非常大的數據集也可以使用此方法處理。
- 您想要使用 `dataset.with_transform()` 或 `collat​​e_fn` 應用隨機轉換。這在多種模式中很常見，例如訓練視覺模型時的圖像增強，或訓練屏蔽語言模型時的隨機屏蔽。使用 `to_tf_dataset()` 將在加載批次時應用這些轉換，這意味著相同的樣本每次加載時都會獲得不同的增強。這通常就是您想要的。
- 您的數據具有可變維度，例如 NLP 中由不同數量的 tokens 組成的輸入文本。當您創建包含可變尺寸樣本的批次時，標準解決方案是將較短樣本填充到最長樣本的長度。當您使用 `to_tf_dataset` 從數據集中流式傳輸樣本時，您可以通過 `collat​​e_fn` 將此填充應用於每個批次。但是，如果您想將這樣的數據集轉換為密集張量，那麼您將必須將樣本填充到整個數據集中最長樣本的長度！這可能會導致大量的填充，從而浪費內存並降低模型的速度。

### Caveats and limitations

現在， `to_tf_dataset()` 始終返回批處理數據集 - 我們將很快添加對非批處理數據集的支持！