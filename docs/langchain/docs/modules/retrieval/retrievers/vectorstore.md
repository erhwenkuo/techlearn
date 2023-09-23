# Vector store-backed retriever

向量儲存檢索器(vector store retriever)是使用向量儲存來檢索文件的檢索器。它是向量儲存類別的輕量級包裝器，使其符合檢索器介面。它使用向量儲存實現的搜尋方法（例如相似性搜尋和 `MMR`）來查詢向量儲存中的文字。

一旦建立了向量存儲，建立檢索器就變得非常容易。讓我們來看一個例子。

```python
from langchain.document_loaders import TextLoader

loader = TextLoader('../../../state_of_the_union.txt')

from langchain.text_splitter import CharacterTextSplitter
from langchain.vectorstores import FAISS
from langchain.embeddings import OpenAIEmbeddings

documents = loader.load()

text_splitter = CharacterTextSplitter(chunk_size=1000, chunk_overlap=0)

texts = text_splitter.split_documents(documents)

embeddings = OpenAIEmbeddings()

db = FAISS.from_documents(texts, embeddings)
```

結果:

```
    Exiting: Cleaning up .chroma directory
```

```python
retriever = db.as_retriever()

docs = retriever.get_relevant_documents("what did he say about ketanji brown jackson")
```

## 最大邊際相關檢索 (MMR)

---

預設情況下，向量儲存檢索器使用相似性搜尋。如果底層向量儲存支援最大邊際相關性搜索，您可以將其指定為搜尋類型。

```python
retriever = db.as_retriever(search_type="mmr")

docs = retriever.get_relevant_documents("what did he say about ketanji brown jackson")
```

## 相似度閾值檢索

---

您也可以使用一種檢索方法來設定相似性分數閾值，並且僅傳回分數高於該閾值的文件。

```python
retriever = db.as_retriever(search_type="similarity_score_threshold", search_kwargs={"score_threshold": .5})

docs = retriever.get_relevant_documents("what did he say about ketanji brown jackson")
```

## 指定 top k

---

您也可以指定在檢索時使用的搜尋 `kwargs`，例如 `k`。

```python
retriever = db.as_retriever(search_kwargs={"k": 1})

docs = retriever.get_relevant_documents("what did he say about ketanji brown jackson")

len(docs)
```

結果:

```
    1
```