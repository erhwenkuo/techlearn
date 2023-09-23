# Parent Document Retriever

在拆分文件進行檢索時，經常會出現相互衝突的需求：

1. 您可能想要小文檔，以便它們的嵌入能夠最準確地反映它們的含義。如果太長，那麼嵌入可能會失去意義。
2. 您希望擁有足夠長的文檔以保留每個區塊的上下文。

`ParentDocumentRetriever` 透過分割和儲存小塊資料來實現這種平衡。在檢索過程中，它首先取得 small chunk，然後尋找這些區塊的父 ID，並傳回那些較大的文件。

請注意，"parent document" 是指 small chunk 來源的文檔。這個 "parent document" 可以是整個原始文件或更大的 chunk。

```python
from langchain.retrievers import ParentDocumentRetriever
```

**API Reference:**

- [ParentDocumentRetriever](https://api.python.langchain.com/en/latest/retrievers/langchain.retrievers.parent_document_retriever.ParentDocumentRetriever.html)

```python
from langchain.vectorstores import Chroma
from langchain.embeddings import OpenAIEmbeddings
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.storage import InMemoryStore
from langchain.document_loaders import TextLoader
```

**API Reference:**

- [Chroma](https://api.python.langchain.com/en/latest/vectorstores/langchain.vectorstores.chroma.Chroma.html)
- [OpenAIEmbeddings](https://api.python.langchain.com/en/latest/embeddings/langchain.embeddings.openai.OpenAIEmbeddings.html)
- [RecursiveCharacterTextSplitter](https://api.python.langchain.com/en/latest/text_splitter/langchain.text_splitter.RecursiveCharacterTextSplitter.html)
- [InMemoryStore](https://api.python.langchain.com/en/latest/storage/langchain.storage.in_memory.InMemoryStore.html)
- [TextLoader](https://api.python.langchain.com/en/latest/document_loaders/langchain.document_loaders.text.TextLoader.html)

```python
loaders = [
    TextLoader('../../paul_graham_essay.txt'),
    TextLoader('../../state_of_the_union.txt'),
]

docs = []

for l in loaders:
    docs.extend(l.load())
```

## 檢索完整文檔

---

在這種模式下，我們想要檢索完整的文件。因此，我們只指定一個子分割器(child splitter)。

```python
# This text splitter is used to create the child documents
child_splitter = RecursiveCharacterTextSplitter(chunk_size=400)

# The vectorstore to use to index the child chunks
vectorstore = Chroma(
    collection_name="full_documents",
    embedding_function=OpenAIEmbeddings()
)

# The storage layer for the parent documents
store = InMemoryStore()

retriever = ParentDocumentRetriever(
    vectorstore=vectorstore, 
    docstore=store, 
    child_splitter=child_splitter,
)

retriever.add_documents(docs, ids=None)
```

這應該會產生兩個鍵，因為我們添加了兩個文件。

```python
list(store.yield_keys())
```

結果:

```
    ['05fe8d8a-bf60-4f87-b576-4351b23df266',
     '571cc9e5-9ef7-4f6c-b800-835c83a1858b']
```

現在讓我們調用向量存儲搜尋功能 - 我們應該看到它返回小塊（因為我們正在存儲小塊）。

```python
sub_docs = vectorstore.similarity_search("justice breyer")

print(sub_docs[0].page_content)
```

結果:

```
    Tonight, I’d like to honor someone who has dedicated his life to serve this country: Justice Stephen Breyer—an Army veteran, Constitutional scholar, and retiring Justice of the United States Supreme Court. Justice Breyer, thank you for your service. 
    
    One of the most serious constitutional responsibilities a President has is nominating someone to serve on the United States Supreme Court.
```

現在讓我們從整體檢索器中檢索。這應該會傳回大文件 - 因為它會傳回較小區塊所在的文件。

```python
retrieved_docs = retriever.get_relevant_documents("justice breyer")

len(retrieved_docs[0].page_content)
```

結果:

```
    38539
```

## 檢索更大的區塊

有時，完整文件可能太大而無法按原樣檢索它們。在這種情況下，我們真正想做的是先將原始文件分割成更大的區塊，然後將其分割成更小的區塊。然後我們索引較小的區塊，但在檢索時我們檢索較大的區塊（但仍然不是完整的文檔）。

```python
# This text splitter is used to create the parent documents
parent_splitter = RecursiveCharacterTextSplitter(chunk_size=2000)

# This text splitter is used to create the child documents
# It should create documents smaller than the parent
child_splitter = RecursiveCharacterTextSplitter(chunk_size=400)

# The vectorstore to use to index the child chunks
vectorstore = Chroma(collection_name="split_parents", embedding_function=OpenAIEmbeddings())

# The storage layer for the parent documents
store = InMemoryStore()

retriever = ParentDocumentRetriever(
    vectorstore=vectorstore, 
    docstore=store, 
    child_splitter=child_splitter,
    parent_splitter=parent_splitter,
)

retriever.add_documents(docs)
```

我們可以看到現在有兩個以上的文檔 - 這些是較大的區塊。

```python
len(list(store.yield_keys()))
```

結果:

```
    66
```

讓我們確保底層向量儲存仍然檢索小塊。

```python
sub_docs = vectorstore.similarity_search("justice breyer")

print(sub_docs[0].page_content)
```

結果:

```
    Tonight, I’d like to honor someone who has dedicated his life to serve this country: Justice Stephen Breyer—an Army veteran, Constitutional scholar, and retiring Justice of the United States Supreme Court. Justice Breyer, thank you for your service. 
    
    One of the most serious constitutional responsibilities a President has is nominating someone to serve on the United States Supreme Court.
```

```python
retrieved_docs = retriever.get_relevant_documents("justice breyer")

len(retrieved_docs[0].page_content)
```

結果:

```
    1849
```

```python
print(retrieved_docs[0].page_content)
```

結果:

```
    In state after state, new laws have been passed, not only to suppress the vote, but to subvert entire elections. 
    
    We cannot let this happen. 
    
    Tonight. I call on the Senate to: Pass the Freedom to Vote Act. Pass the John Lewis Voting Rights Act. And while you’re at it, pass the Disclose Act so Americans can know who is funding our elections. 
    
    Tonight, I’d like to honor someone who has dedicated his life to serve this country: Justice Stephen Breyer—an Army veteran, Constitutional scholar, and retiring Justice of the United States Supreme Court. Justice Breyer, thank you for your service. 
    
    One of the most serious constitutional responsibilities a President has is nominating someone to serve on the United States Supreme Court. 
    
    And I did that 4 days ago, when I nominated Circuit Court of Appeals Judge Ketanji Brown Jackson. One of our nation’s top legal minds, who will continue Justice Breyer’s legacy of excellence. 
    
    A former top litigator in private practice. A former federal public defender. And from a family of public school educators and police officers. A consensus builder. Since she’s been nominated, she’s received a broad range of support—from the Fraternal Order of Police to former judges appointed by Democrats and Republicans. 
    
    And if we are to advance liberty and justice, we need to secure the Border and fix the immigration system. 
    
    We can do both. At our border, we’ve installed new technology like cutting-edge scanners to better detect drug smuggling.  
    
    We’ve set up joint patrols with Mexico and Guatemala to catch more human traffickers.  
    
    We’re putting in place dedicated immigration judges so families fleeing persecution and violence can have their cases heard faster. 
    
    We’re securing commitments and supporting partners in South and Central America to host more refugees and secure their own borders.
```
