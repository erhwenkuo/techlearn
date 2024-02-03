# 5 層級文本分割

原文: [5 Levels Of Text Splitting](https://github.com/FullStackRetrieval-com/RetrievalTutorials/blob/main/5_Levels_Of_Text_Splitting.ipynb)

在本文中，我們將回顧文字拆分的 5 個層級。這是出於教育目的而整理的非官方清單。

您是否曾經嘗試將一段很長的文字放入 ChatGPT 但它告訴您它太長了？或者您試圖為您的應用程式提供更好的長期記憶，但它仍然不太有效。

提高語言模型應用程式效能的最有效策略之一是將大數據分割成更小的部分。這稱為拆分或分塊（我們將互換使用這些術語）。在多模態世界中，分割也適用於影像。

我們將涵蓋很多內容，但如果您堅持到最後，我保證您將牢牢掌握組塊理論、策略和資源以了解更多資訊。

## 文本分割的層級

- **Level 1: Character Splitting** -簡單的靜態字元文本分割
- **Level 2: Recursive Character Text Splitting** - 基於分隔符號清單的遞歸文本分割
- **Level 3: Document Specific Splitting** - 針對不同文件類型（PDF、Python、Markdown）的多種文本分割方法
- **Level 4: Semantic Splitting** - 基於 embedding 的分塊
- **Level 5: Agentic Splitting** - 使用類似代理的系統分割文字的實驗方法

## Level 1: Character Splitting

字元拆分是拆分文字最基本的形式。這是將文字簡單地劃分為 N 個字元大小的區塊的過程，無論其內容或形式如何。

不建議任何應用程式使用此方法 - 但它是我們了解基礎知識的一個很好的起點。

- **Pros**: 容易又簡單
- **Cons**: 非常死板，沒有考慮文本的結構

需要了解的概念：

- **Chunk Size** - 您希望在區塊中包含的字元數。 例如 50、100、100,000 等
- **Chunk Overlap** - 您希望連續區塊重疊的數量。這是為了盡量避免將單一上下文切割成多個部分。這將在區塊之間建立重複資料。

首先讓我們取得一些範例文本

```python
text = "This is the text I would like to chunk up. It is the example text for this exercise"
```

然後我們手動分割這段文本

```python
# Create a list that will hold your chunks
chunks = []

chunk_size = 35 # Characters

# Run through the a range with the length of your text and iterate every chunk_size you want
for i in range(0, len(text), chunk_size):
    chunk = text[i:i + chunk_size]
    chunks.append(chunk)

print(chunks)
```

結果:

```
['This is the text I would like to ch',
 'unk up. It is the example text for ',
 'this exercise']
```

在語言模型世界中處理文字時，我們不處理原始字串。處理 `document` 更為常見。`document` 是保存您關心的文本的對象，同時也是附加元數據，使以後的過濾和操作變得更加容易。

我們可以將字串列表轉換為 `document`，但我寧願從頭開始並建立 `document`。

```python
from langchain.text_splitter import CharacterTextSplitter
```

