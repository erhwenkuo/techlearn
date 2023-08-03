# The cache

緩存是🤗數據集如此高效的原因之一。它存儲以前下載和處理的數據集，因此當您需要再次使用它們時，它們會直接從緩存中重新加載。這避免了必須重新下載數據集或重新應用處理功能。即使您關閉並啟動另一個 Python 會話後，🤗 數據集也會直接從緩存重新加載您的數據集！

## Fingerprint

緩存如何跟踪應用於數據集的轉換?　嗯，🤗 數據集為緩存文件分配指紋。指紋跟踪數據集的當前狀態。初始指紋是使用 Arrow 表中的哈希值計算的，如果數據集位於磁盤上，則使用 Arrow 文件的哈希值計算。後續指紋是通過組合先前狀態的指紋和所應用的最新變換的散列來計算的。

以下是實際指紋的樣子：

```python
from datasets import Dataset

dataset1 = Dataset.from_dict({"a": [0, 1, 2]})

dataset2 = dataset1.map(lambda x: {"a": x["a"] + 1})

print(dataset1._fingerprint, dataset2._fingerprint)
```

結果:

```bash
d19493523d95e2dc 5b86abacd4b42434
```

為了使 transform 可 hashable，它需要可通過 [dill](https://dill.readthedocs.io/en/latest/) 或 [pickle](https://docs.python.org/3/library/pickle) 進行 picklable。

當您使用 non-hashable 的 transform 時，🤗 數據集會使用隨機指紋並發出警告。non-hashable 的 transform 被認為與之前的 transform 不同。因此，🤗 數據集將重新計算所有transform。確保您的 transform 可以使用 pickle 或 dill 進行序列化！

🤗 數據集重新計算所有內容的一個例子是禁用緩存時。發生這種情況時，每次都會生成緩存文件並將它們寫入臨時目錄。Python 會話結束後，臨時目錄中的緩存文件將被刪除。為這些緩存文件分配隨機散列，而不是指紋。

## Hashing

數據集的指紋通過散列傳遞給映射的函數以及映射參數（batch_size、remove_columns 等）來更新。

您可以使用 Fingerprint.Hasher 檢查任何 Python 物件的哈希值：

```python
from datasets.fingerprint import Hasher

my_func = lambda example: {"length": len(example["text"])}

print(Hasher.hash(my_func))
```

哈希值是通過使用 `dill` 來 pickler 轉儲物件並哈希轉儲的字節來計算的。 pickler 遞歸地轉儲函數中使用的所有變量，因此您對函數中使用的對象所做的任何更改都將導致哈希值更改。

如果您的某個函數在會話之間似乎沒有相同的哈希值，則意味著它的至少一個變量包含一個不確定的 Python 物件。發生這種情況時，請隨意對您發現可疑的任何物件進行 hashing，以嘗試找到導致 hashing 更改的物件。例如，如果您使用的列表的元素順序在會話中不確定，則哈希在會話中也不會相同。

