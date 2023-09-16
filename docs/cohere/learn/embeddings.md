# Embeddings

嵌入(embedding)是將文本含義表示為數字列表的一種方法。這很有用，因為一旦文本採用這種形式，就可以將其與其他文本進行比較，以進行相似性、聚類、分類和其他用例。使用簡單的比較函數，我們可以計算兩個嵌入的相似度得分，以確定兩個文本是否在談論相似的事物。

在下面的示例中，兩個相似短語的嵌入具有較高的相似性得分，而兩個不相關短語的嵌入具有較低的相似性得分：

```python
import cohere
import numpy as np

co = cohere.Client("YOUR_API_KEY")

# get the embeddings
phrases = ["i love soup", "soup is my favorite", "london is far away"]
(soup1, soup2, london) = co.embed(phrases).embeddings

# compare them
def calculate_similarity(a, b):
  return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

calculate_similarity(soup1, soup2) # 0.9 - very similar!
calculate_similarity(soup1, london) # 0.3 - not similar!
```

<figure markdown>
  ![](./assets/e8ea586-Embeddings_Visual_1_1.png)
  <figcaption>Turning text into embeddings.</figcaption>
</figure>

## 應用領域

- 構建一個 FAQ 機器人，將客戶問題與現有常見問題集合進行相似性比較。
- 可視化大量非結構化文本時，embedding 這會很有幫助。例如，使用 k 均值聚類對大量文本進行有效聚類。還可以使用 PCA、UMAP 或 t-SNE 等投影技術來可視化嵌入。
- 對數據庫中的文本執行語義搜索。
- 與隨機森林或 SVM 等下游分類器配對，以執行二元或多類分類或情感分類或毒性檢測等任務。

## 如何獲得 Embedding

對於短文本(少於 `512` 個 token)，我們遵循 Reimers 和 Gurevych，返回通過平均文本中每個標記的上下文嵌入而獲得的嵌入。因此，最終的嵌入捕獲了整個文本的語義信息。對於超過 `512` 個 token 的文本，我們會將您的輸入截斷為最大上下文長度。

