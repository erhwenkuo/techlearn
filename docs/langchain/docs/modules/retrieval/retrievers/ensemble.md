# Ensemble Retriever

`EnsembleRetriever` 將檢索器清單作為輸入，並整合其 `get_relevant_documents()` 方法的結果，並根據 [Reciprocal Rank Fusion](https://plg.uwaterloo.ca/~gvcormac/cormacksigir09-rrf.pdf) 演算法對結果進行重新排名。

透過利用不同演算法的優勢，`EnsembleRetriever` 可以獲得比任何單一演算法更好的效能。

最常見的模式是將稀疏(sparse)檢索器（如 BM25）與密集檢索器（如嵌入相似性）結合，因為它們的優點是互補的。它也被稱為“混合搜尋”(`hybrid search`)。稀疏檢索器擅長根據關鍵字尋找相關文檔，而密集檢索器擅長根據語意相似度尋找相關文檔。

```python
from langchain.retrievers import BM25Retriever, EnsembleRetriever
from langchain.vectorstores import FAISS
```

**API Reference:**

- [BM25Retriever](https://api.python.langchain.com/en/latest/retrievers/langchain.retrievers.bm25.BM25Retriever.html)
- [EnsembleRetriever](https://api.python.langchain.com/en/latest/retrievers/langchain.retrievers.ensemble.EnsembleRetriever.html)
- [FAISS](https://api.python.langchain.com/en/latest/vectorstores/langchain.vectorstores.faiss.FAISS.html)


```python
doc_list = [
    "I like apples",
    "I like oranges",
    "Apples and oranges are fruits",
]

# initialize the bm25 retriever and faiss retriever
bm25_retriever = BM25Retriever.from_texts(doc_list)
bm25_retriever.k = 2

embedding = OpenAIEmbeddings()
faiss_vectorstore = FAISS.from_texts(doc_list, embedding)
faiss_retriever = faiss_vectorstore.as_retriever(search_kwargs={"k": 2})

# initialize the ensemble retriever
ensemble_retriever = EnsembleRetriever(retrievers=[bm25_retriever, faiss_retriever], weights=[0.5, 0.5])

docs = ensemble_retriever.get_relevant_documents("apples")

print(docs)
```

結果:

```
    [Document(page_content='I like apples', metadata={}),
     Document(page_content='Apples and oranges are fruits', metadata={})]
```

