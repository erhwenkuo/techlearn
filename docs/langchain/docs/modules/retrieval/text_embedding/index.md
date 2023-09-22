# Text embedding models

!!! info
    查看 [Integrations](https://python.langchain.com/docs/integrations/text_embedding/) 以取得 langchain 的 text embedding 與第三方工具整合的文件。

![](../assets/data_connection-c42d68c3d092b85f50d08d4cc171fc25.jpg)

Embeddings 類別是設計用於與文字嵌入模型互動的類別。有許多嵌入模型提供者（OpenAI、Cohere、Hugging Face 等）- 這個類別旨在為所有這些服務提供者提供標準介面。

嵌入創建一段文字的向量表示。這很有用，因為它意味著我們可以在向量空間中思考文本，並執行語義搜尋之類的操作，在向量空間中找到最相似的文本片段。

LangChain 中的 Embeddings 基底類別提供了兩種方法：一種用於嵌入文檔，另一種用於嵌入查詢。前者採用多個文本作為輸入，而後者採用單一文本。將它們作為兩種單獨方法的原因是，某些嵌入提供者對文本（要搜尋的）與查詢（搜尋查詢本身）有不同的嵌入方法。

## 開始使用

---

### 設定

首先，我們需要安裝 OpenAI Python 套件：

```bash
pip install openai
```

存取 API 需要 API 金鑰，您可以透過建立帳戶並前往此處取得該金鑰。一旦我們有了密鑰，我們就需要透過執行以下命令將其設定為環境變數：

```bash
export OPENAI_API_KEY="..."
```

如果您不想設定環境變量，可以在啟動 `OpenAI LLM` 類別時直接透過 `openai_api_key` 命名參數傳遞金鑰：

```python
from langchain.embeddings import OpenAIEmbeddings

embeddings_model = OpenAIEmbeddings(openai_api_key="...")
```

當然您也可以在沒有使用任何參數的情況下進行初始化：

```python
from langchain.embeddings import OpenAIEmbeddings

embeddings_model = OpenAIEmbeddings()
```

### `embed_documents`

**Embed list of texts**

```python
embeddings = embeddings_model.embed_documents(
    [
        "Hi there!",
        "Oh, hello!",
        "What's your name?",
        "My friends call me World",
        "Hello World!"
    ]
)

len(embeddings), len(embeddings[0])
```

結果:

```bash
(5, 1536)
```

### `embed_query`

**Embed single query**

嵌入一​​段文字是為了與其他嵌入的文本進行比較。

```python
embedded_query = embeddings_model.embed_query("What was the name mentioned in the conversation?")

embedded_query[:5]
```

結果:

```bash
[0.0053587136790156364,
 -0.0004999046213924885,
 0.038883671164512634,
 -0.003001077566295862,
 -0.00900818221271038]
```



