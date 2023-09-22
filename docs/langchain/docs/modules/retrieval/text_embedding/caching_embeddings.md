# Caching

嵌入可以被儲存或臨時快取以避免需要重新計算它們。

Caching embeddings 可以使用 `CacheBackedEmbeddings` 來完成。支援快取的嵌入器是嵌入器的包裝器，它將嵌入緩存在鍵值儲存中。對文字進行哈希處理，並將哈希值用作快取中的鍵值(key)。

初始化 `CacheBackedEmbeddings` 的主要支援方式是 `from_bytes_store`。這需要以下參數：

- `underlying_embedder`: 用於嵌入的嵌入器。
- `document_embedding_cache`: 用於儲存文件嵌入的快取。
- `namespace`: (optional，預設為 `""`）用於文件快取的命名空間。此命名空間用於避免與其他快取發生衝突。例如，將其設定為所使用的嵌入模型的名稱。

!!! tip
    注意：請務必設定 `namespace` 參數，以避免使用不同嵌入模型嵌入的相同文字發生衝突。

```python
from langchain.storage import InMemoryStore, LocalFileStore, RedisStore

from langchain.embeddings import OpenAIEmbeddings, CacheBackedEmbeddings
```

**API Reference:**

- [InMemoryStore](https://api.python.langchain.com/en/latest/storage/langchain.storage.in_memory.InMemoryStore.html)
- [LocalFileStore](https://api.python.langchain.com/en/latest/storage/langchain.storage.file_system.LocalFileStore.html)
- [RedisStore](https://api.python.langchain.com/en/latest/storage/langchain.storage.redis.RedisStore.html)
- [OpenAIEmbeddings](https://api.python.langchain.com/en/latest/embeddings/langchain.embeddings.openai.OpenAIEmbeddings.html)
- [CacheBackedEmbeddings](https://api.python.langchain.com/en/latest/embeddings/langchain.embeddings.cache.CacheBackedEmbeddings.html)

## 與 vector store 一起使用

---

首先，讓我們看一個使用本機檔案系統儲存嵌入並使用 FAISS 向量儲存進行檢索的範例。

```python
from langchain.document_loaders import TextLoader
from langchain.embeddings.openai import OpenAIEmbeddings
from langchain.text_splitter import CharacterTextSplitter
from langchain.vectorstores import FAISS
```

**API Reference:**

- [TextLoader](https://api.python.langchain.com/en/latest/document_loaders/langchain.document_loaders.text.TextLoader.html)
- [OpenAIEmbeddings](https://api.python.langchain.com/en/latest/embeddings/langchain.embeddings.openai.OpenAIEmbeddings.html)
- [CharacterTextSplitter](https://api.python.langchain.com/en/latest/text_splitter/langchain.text_splitter.CharacterTextSplitter.html)
- [FAISS](https://api.python.langchain.com/en/latest/vectorstores/langchain.vectorstores.faiss.FAISS.html)

```python
underlying_embeddings = OpenAIEmbeddings()

fs = LocalFileStore("./cache/")

cached_embedder = CacheBackedEmbeddings.from_bytes_store(
    underlying_embeddings, fs, namespace=underlying_embeddings.model
)
```

嵌入之前緩存為空：

```python
list(fs.yield_keys())
```

結果:

```
[]
```

載入文檔，將其分割成區塊，嵌入每個區塊並將其載入到向量儲存中。

```python
raw_documents = TextLoader("../state_of_the_union.txt").load()

text_splitter = CharacterTextSplitter(chunk_size=1000, chunk_overlap=0)

documents = text_splitter.split_documents(raw_documents)
```

建立向量儲存：

```python
db = FAISS.from_documents(documents, cached_embedder)
```

結果:

```
    CPU times: user 608 ms, sys: 58.9 ms, total: 667 ms
    Wall time: 1.3 s
```

如果我們嘗試再次建立向量存儲，它會快得多，因為它不需要重新計算任何嵌入。

```python
db2 = FAISS.from_documents(documents, cached_embedder)
```

結果:

```
    CPU times: user 33.6 ms, sys: 3.96 ms, total: 37.6 ms
    Wall time: 36.8 ms
```

以下是創建的一些嵌入(embeddings)：

```python
list(fs.yield_keys())[:5]
```

結果:

```
    ['text-embedding-ada-002614d7cf6-46f1-52fa-9d3a-740c39e7a20e',
     'text-embedding-ada-0020fc1ede2-407a-5e14-8f8f-5642214263f5',
     'text-embedding-ada-002e4ad20ef-dfaa-5916-9459-f90c6d8e8159',
     'text-embedding-ada-002a5ef11e4-0474-5725-8d80-81c91943b37f',
     'text-embedding-ada-00281426526-23fe-58be-9e84-6c7c72c8ca9a']
```

## 記憶體快取

---

本節介紹如何為嵌入設定記憶體快取。這種類型的快取主要用於單元測試或原型設計。如果您需要實際儲存嵌入，請勿使用此快取。

```python
store = InMemoryStore()

underlying_embeddings = OpenAIEmbeddings()

embedder = CacheBackedEmbeddings.from_bytes_store(
    underlying_embeddings, store, namespace=underlying_embeddings.model
)

embeddings = embedder.embed_documents(["hello", "goodbye"])
```

結果:

```
    CPU times: user 10.9 ms, sys: 916 µs, total: 11.8 ms
    Wall time: 159 ms
```

第二次我們嘗試嵌入時，嵌入時間僅為 2 毫秒，因為嵌入是在快取中尋找的。

```python
embeddings_from_cache = embedder.embed_documents(["hello", "goodbye"])
```

結果:

```
    CPU times: user 1.67 ms, sys: 342 µs, total: 2.01 ms
    Wall time: 2.01 ms
```

```python
embeddings == embeddings_from_cache
```

結果:

```
    True
```

## 檔案系統

---

本節介紹如何使用檔案系統儲存。

```python
fs = LocalFileStore("./test_cache/")

embedder2 = CacheBackedEmbeddings.from_bytes_store(
    underlying_embeddings, fs, namespace=underlying_embeddings.model
)

embeddings = embedder2.embed_documents(["hello", "goodbye"])
```

結果:

```
    CPU times: user 6.89 ms, sys: 4.89 ms, total: 11.8 ms
    Wall time: 184 ms
```

以下是已儲存到目錄 `./test_cache` 的嵌入。

請注意，嵌入器採用 `namespace` 參數。

```python
list(fs.yield_keys())
```

結果:

```
    ['text-embedding-ada-002e885db5b-c0bd-5fbc-88b1-4d1da6020aa5',
     'text-embedding-ada-0026ba52e44-59c9-5cc9-a084-284061b13c80']
```

## Redis Store

---

```python
from langchain.storage import RedisStore
```

**API Reference:**

- [RedisStore](https://api.python.langchain.com/en/latest/storage/langchain.storage.redis.RedisStore.html)

```python
# For cache isolation can use a separate DB
# Or additional namepace
store = RedisStore(redis_url="redis://localhost:6379", client_kwargs={'db': 2}, namespace='embedding_caches')

underlying_embeddings = OpenAIEmbeddings()

embedder = CacheBackedEmbeddings.from_bytes_store(
    underlying_embeddings, store, namespace=underlying_embeddings.model
)

embeddings = embedder.embed_documents(["hello", "goodbye"])
```

結果:

```
    CPU times: user 3.99 ms, sys: 0 ns, total: 3.99 ms
    Wall time: 3.5 ms
```

```python
embeddings = embedder.embed_documents(["hello", "goodbye"])
```

結果:

```
    CPU times: user 2.47 ms, sys: 767 µs, total: 3.24 ms
    Wall time: 2.75 ms
```

```python
list(store.yield_keys())
```

結果:

```
    ['text-embedding-ada-002e885db5b-c0bd-5fbc-88b1-4d1da6020aa5',
     'text-embedding-ada-0026ba52e44-59c9-5cc9-a084-284061b13c80']
```

```python
list(store.client.scan_iter())
```

結果:

```
    [b'embedding_caches/text-embedding-ada-002e885db5b-c0bd-5fbc-88b1-4d1da6020aa5',
     b'embedding_caches/text-embedding-ada-0026ba52e44-59c9-5cc9-a084-284061b13c80']
```
