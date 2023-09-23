# Retrievers

!!! info
    查看 [Integrations](https://python.langchain.com/docs/integrations/retrievers/) 以取得 langchain 與第三方工具整合的文件。

![](../assets/data_connection-c42d68c3d092b85f50d08d4cc171fc25.jpg)

Retriever 是一個接口，它根據非結構化查詢返回文件。它比 vector store 更通用。Retriever 并不需要儲存 document，只需返回（或檢索）它們即可。Vector store 可以用作 retriever 的骨幹，但也有其他類型的 retriever。

## 開始使用

---

LangChain 中 `BaseRetriever` 類別的公共 API 如下：

```python
from abc import ABC, abstractmethod
from typing import Any, List
from langchain.schema import Document
from langchain.callbacks.manager import Callbacks

class BaseRetriever(ABC):
    ...
    def get_relevant_documents(
        self, query: str, *, callbacks: Callbacks = None, **kwargs: Any
    ) -> List[Document]:
        """Retrieve documents relevant to a query.
        Args:
            query: string to find relevant documents for
            callbacks: Callback manager or list of callbacks
        Returns:
            List of relevant documents
        """
        ...

    async def aget_relevant_documents(
        self, query: str, *, callbacks: Callbacks = None, **kwargs: Any
    ) -> List[Document]:
        """Asynchronously get documents relevant to a query.
        Args:
            query: string to find relevant documents for
            callbacks: Callback manager or list of callbacks
        Returns:
            List of relevant documents
        """
        ...
```

就這麼簡單！您可以呼叫 `get_relevant_documents` 或 `async aget_relevant_documents` 方法來檢索與查詢相關的文檔，其中 `relevance` 由您呼叫的特定 `retriever` 物件定義。

當然，我們也幫助建立我們認為有用的 `retrievers`。我們關注的檢索器的主要類型是向量儲存檢索器(vector store retriever)。我們將在本指南的其餘部分重點討論這一點。

為了了解什麼是向量儲存檢索器，了解什麼是向量儲存非常重要。那麼讓我們看看這個。

預設情況下，LangChain 使用 [Chroma](https://python.langchain.com/docs/ecosystem/integrations/chroma.html) 作為向量儲存來索引和搜尋嵌入。要完成本教程，我們首先需要安裝 `chromadb`。

```bash
pip install chromadb
```

此範例展示了針對文件的問答。我們選擇這個作為入門範例，因為它很好地結合了許多不同的元素（文字分割器、嵌入、向量儲存），然後也展示瞭如何在鏈中使用它們。

針對文件的問答包括四個步驟：

1. 建立索引
2. 從該索引建立一個檢索器
3. 創建問答鏈
4. 問問題

每個步驟都有多個子步驟和潛在的配置。在本章節中，我們將主要關注 `(1)`。我們將首先展示這樣做的簡短說明，然後分解實際發生的情況。

首先，讓我們導入一些無論如何都會用到的通用類別。

```python
from langchain.chains import RetrievalQA
from langchain.llms import OpenAI
```

接下來在通用設定中，我們指定要使用的文檔載入器。您可以在[此處](https://github.com/hwchase17/langchain/blob/master/docs/extras/modules/state_of_the_union.txt)下載 `state_of_the_union.txt` 檔案。

```python
from langchain.document_loaders import TextLoader

loader = TextLoader('../state_of_the_union.txt', encoding='utf8')
```

## 一行索引創建

---

為了盡快開始，我們可以使用 [`VectorstoreIndexCreator`](https://api.python.langchain.com/en/latest/indexes/langchain.indexes.vectorstore.VectorstoreIndexCreator.html)。

```python
from langchain.indexes.vectorstore import VectorstoreIndexCreator

index = VectorstoreIndexCreator().from_loaders([loader])
```

現在索引已創建，我們可以使用它來詢問資料問題！請注意，在幕後，這實際上也執行了幾個步驟，我們將在本指南的後面部分介紹這些步驟。

```python
query = "What did the president say about Ketanji Brown Jackson"

index.query(query)
```

結果:

```
    " The president said that Ketanji Brown Jackson is one of the nation's top legal minds, a former top litigator in private practice, a former federal public defender, and from a family of public school educators and police officers. He also said that she is a consensus builder and has received a broad range of support from the Fraternal Order of Police to former judges appointed by Democrats and Republicans."
```

```python
query = "What did the president say about Ketanji Brown Jackson"

index.query_with_sources(query)
```

結果:

```
    {'question': 'What did the president say about Ketanji Brown Jackson',
     'answer': " The president said that he nominated Circuit Court of Appeals Judge Ketanji Brown Jackson, one of the nation's top legal minds, to continue Justice Breyer's legacy of excellence, and that she has received a broad range of support from the Fraternal Order of Police to former judges appointed by Democrats and Republicans.\n",
     'sources': '../state_of_the_union.txt'}
```

從 `VectorstoreIndexCreator` 傳回的是 `VectorStoreIndexWrapper`，它提供了這些很好的 `query` 和 `query_with_sources` 功能。如果我們只想直接存取向量存儲，我們也可以這樣做。

```python
index.vectorstore.as_retriever()
```

結果:

```
    VectorStoreRetriever(vectorstore=<langchain.vectorstores.chroma.Chroma object at 0x119aa5940>, search_kwargs={})
```

透過與文件關聯的元資料來過濾向量儲存也很方便，特別是當您的向量儲存有多個來源時。這可以使用查詢方法來完成，如下所示：

```python
index.query("Summarize the general content of this document.", retriever_kwargs={"search_kwargs": {"filter": {"source": "../state_of_the_union.txt"}}})
```

結果:

```
    " The document is a speech given by President Trump to the nation on the occasion of his 245th birthday. The speech highlights the importance of American values and the challenges facing the country, including the ongoing conflict in Ukraine, the ongoing trade war with China, and the ongoing conflict in Syria. The speech also discusses the importance of investing in emerging technologies and American manufacturing, and calls on Congress to pass the Bipartisan Innovation Act and other important legislation."
```

## Walkthrough

---

好吧，那麼到底發生了什麼事？這個索引是如何建立的？

很多神奇之處都隱藏在這個 `VectorstoreIndexCreator` 裡。它做了什麼？

載入文件後它將執行三個主要步驟：

1. 將文件分割成區塊(chunks)
2. 為每個文件建立嵌入(embeddings)
3. 在向量儲存中儲存文件和嵌入

讓我們用程式碼來示範一下:

```python
documents = loader.load()
```

接下來，我們將把文檔分成區塊。

```python
from langchain.text_splitter import CharacterTextSplitter

text_splitter = CharacterTextSplitter(chunk_size=1000, chunk_overlap=0)

texts = text_splitter.split_documents(documents)
```

然後我們將選擇要使用的嵌入(embedding)。

```python
from langchain.embeddings import OpenAIEmbeddings

embeddings = OpenAIEmbeddings()
```

我們現在創建向量存儲來用作索引。

```python
from langchain.vectorstores import Chroma

db = Chroma.from_documents(texts, embeddings)
```

結果:

```
    Running Chroma using direct local API.
    Using DuckDB in-memory for database. Data will be transient.
```

這就是創建索引。然後，我們在檢索器介面中公開該索引。

```python
retriever = db.as_retriever()
```

然後，像以前一樣，我們創建一條鏈並用它來回答問題！

```python
qa = RetrievalQA.from_chain_type(llm=OpenAI(), chain_type="stuff", retriever=retriever)
```

接著來問問題吧!

```python
query = "What did the president say about Ketanji Brown Jackson"

qa.run(query)
```

結果:

```
    " The President said that Judge Ketanji Brown Jackson is one of the nation's top legal minds, a former top litigator in private practice, a former federal public defender, and from a family of public school educators and police officers. He said she is a consensus builder and has received a broad range of support from organizations such as the Fraternal Order of Police and former judges appointed by Democrats and Republicans."
```

`VectorstoreIndexCreator` 是所有這些邏輯的包裝。它可以在它使用的文字分割器、它使用的嵌入以及它使用的向量儲存中進行配置。例如，您可以如下配置：

```python
index_creator = VectorstoreIndexCreator(
    vectorstore_cls=Chroma,
    embedding=OpenAIEmbeddings(),
    text_splitter=CharacterTextSplitter(chunk_size=1000, chunk_overlap=0)
)
```

希望這能突顯 `VectorstoreIndexCreator` 背後發生的事。雖然我們認為有一種簡單的方法來創建索引很重要，但我們也認為了解背後發生的事情也很重要。



