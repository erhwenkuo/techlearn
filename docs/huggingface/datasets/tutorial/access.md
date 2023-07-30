# Know your dataset

有兩種類型的 dataset objects，一種是常規 Dataset，另一種是 ✨ IterableDataset ✨。 Dataset 提供對數據集內的每一行(row) data 的快速隨機訪問和內存映射，因此即使加載大型數據集也僅會使用相對少量的設備記憶體。但對於非常非常大的數據集，甚至無法容納在磁盤或內存中，IterableDataset 允許您訪問和使用數據集，而無需等待它完全下載！

本教程將向您展示如何加載和訪問 Dataset 和 IterableDataset。

## Dataset

當您加載數據集拆分時，您將獲得一個 Dataset 物件。您可以使用 Dataset 物件執行許多操作，這就是為什麼學習如何操作存儲在其中的數據並與之交互非常重要。

本教程使用 [rotten_tomatoes](https://huggingface.co/datasets/rotten_tomatoes) 數據集，但請隨意加載您想要的任何數據集並繼續操作！

```python
from datasets import load_dataset

dataset = load_dataset("rotten_tomatoes", split="train")
```

### Indexing

數據集包​​含數據列，每列可以是不同類型的數據。索引或軸標籤用於訪問數據集中的示例。例如，按行索引會返回數據集中示例的字典：

```python
# Get the first row in the dataset
dataset[0]
```

結果:

```bash
{'label': 1,
 'text': 'the rock is destined to be the 21st century\'s new " conan " and that he\'s going to make a splash even greater than arnold schwarzenegger , jean-claud van damme or steven segal .'}
```

使用 `-` 運算符從數據集末尾開始：

```python
# Get the last row in the dataset
dataset[-1]
```

結果:

```bash
{'label': 0,
 'text': 'things really get weird , though not particularly scary : the movie is all portent and no content .'}
```

按列名索引會返回該列中所有值的列表：

```python
dataset["text"]
```

結果:

```bash
['the rock is destined to be the 21st century\'s new " conan " and that he\'s going to make a splash even greater than arnold schwarzenegger , jean-claud van damme or steven segal .',
 'the gorgeously elaborate continuation of " the lord of the rings " trilogy is so huge that a column of words cannot adequately describe co-writer/director peter jackson\'s expanded vision of j . r . r . tolkien\'s middle-earth .',
 'effective but too-tepid biopic',
 ...,
 'things really get weird , though not particularly scary : the movie is all portent and no content .']
```

您可以組合行名和列名索引以返回某個位置的特定值：

```python
dataset[0]["text"]
```

結果:

```bash
'the rock is destined to be the 21st century\'s new " conan " and that he\'s going to make a splash even greater than arnold schwarzenegger , jean-claud van damme or steven segal .'
```

但重要的是要記住，索引順序很重要，尤其是在處理大型音頻和圖像數據集時。按列名索引首先返回該列中的所有值，然後加載該位置的值。對於大型數據集，首先按列名建立索引可能會比較慢。

```python
with Timer():
   dataset[0]['text']
```

結果:

```bash
Elapsed time: 0.0031 seconds
```

```python
with Timer():
  dataset["text"][0]
```

結果:

```bash
Elapsed time: 0.0094 seconds
```

### Slicing

切片返回數據集的切片或子集，這對於一次查看多行很有用。要對數據集進行切片，請使用 `:` 運算符指定位置範圍。

```python
# Get the first three rows
dataset[:3]
```

結果:

```bash
{'label': [1, 1, 1],
 'text': ['the rock is destined to be the 21st century\'s new " conan " and that he\'s going to make a splash even greater than arnold schwarzenegger , jean-claud van damme or steven segal .',
  'the gorgeously elaborate continuation of " the lord of the rings " trilogy is so huge that a column of words cannot adequately describe co-writer/director peter jackson\'s expanded vision of j . r . r . tolkien\'s middle-earth .',
  'effective but too-tepid biopic']}
```

```python
# Get rows between three and six
dataset[3:6]
```

結果:

```bash
{'label': [1, 1, 1],
 'text': ['if you sometimes like to go to the movies to have fun , wasabi is a good place to start .',
  "emerges as something rare , an issue movie that's so honest and keenly observed that it doesn't feel like one .",
  'the film provides some great insight into the neurotic mindset of all comics -- even those who have reached the absolute top of the game .']}
```

## IterableDataset

當您在 `load_dataset()` 中將 `streaming` 參數設置為 `True` 時，將加載 IterableDataset：

```python
from datasets import load_dataset

iterable_dataset = load_dataset("food101", split="train", streaming=True)

for example in iterable_dataset:
    print(example)
    break
```

結果:

```
{'image': <PIL.JpegImagePlugin.JpegImageFile image mode=RGB size=384x512 at 0x7F0681F5C520>, 'label': 6}
```

`IterableDataset` 一次逐步迭代一個數據集的一個示例，因此您無需等待整個數據集下載完畢即可使用它。正如您可以想像的，這對於您想要立即使用的大型數據集非常有用！

然而，這意味著 `IterableDataset` 的行為與常規數據集不同。您無法隨機訪問 `IterableDataset` 中的示例。相反，您應該迭代其元素，例如，通過調用 `next(iter())` 或使用 `for loop` 從 `IterableDataset` 返回下一筆數據：

```pythonfor example in iterable_dataset:
    print(example)
    break
```

結果:

```bash
{'image': <PIL.JpegImagePlugin.JpegImageFile image mode=RGB size=384x512 at 0x7F0681F59B50>,
 'label': 6}
```

```python
for example in iterable_dataset:
    print(example)
    break
```

結果:

```bash
{'image': <PIL.JpegImagePlugin.JpegImageFile image mode=RGB size=384x512 at 0x7F7479DE82B0>, 'label': 6}
```

您可以使用 `IterableDataset.take()` 返回數據集的子集，其中包含特定數量的示例：

```python
# Get first three examples
list(iterable_dataset.take(3))
```

結果:

```bash
[{'image': <PIL.JpegImagePlugin.JpegImageFile image mode=RGB size=384x512 at 0x7F7479DEE9D0>,
  'label': 6},
 {'image': <PIL.JpegImagePlugin.JpegImageFile image mode=RGB size=512x512 at 0x7F7479DE8190>,
  'label': 6},
 {'image': <PIL.JpegImagePlugin.JpegImageFile image mode=RGB size=512x383 at 0x7F7479DE8310>,
  'label': 6}]
```

但與切片不同的是，`IterableDataset.take() 創建一個新的 `IterableDataset`。

## Next steps

有興趣了解更多關於這兩類數據集之間的差異嗎？[在 Dataset 和 IterableDataset 之間的差異概念指南](https://huggingface.co/docs/datasets/about_mapstyle_vs_iterable)中了解有關它們的更多信息。

要更多地實踐這些數據集類型，請查看[處理指南](https://huggingface.co/docs/datasets/process)以了解如何預處理數據集，或查看 [Stream 指南](https://huggingface.co/docs/datasets/stream)以了解如何預處理 `IterableDataset`。