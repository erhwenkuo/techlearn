# Time-weighted vector store retriever

此檢索器使用語義相似性和時間衰減的組合。

對它們進行評分的演算法是：

```python
semantic_similarity + (1.0 - decay_rate) ^ hours_passed
```

值得注意的是，`hours_passed` 指的是自上次存取檢索器中的物件以來經過的時數，而不是自創建以來經過的時數。這意味著經常訪問的對象保持“新鮮”。

```python
import faiss

from datetime import datetime, timedelta
from langchain.docstore import InMemoryDocstore
from langchain.embeddings import OpenAIEmbeddings
from langchain.retrievers import TimeWeightedVectorStoreRetriever
from langchain.schema import Document
from langchain.vectorstores import FAISS
```

## 低衰減率

---

低衰減率（極端情況下，我們將其設定為接近 `0`）意味著記憶將被「記住」更長時間。衰減率為 `0` 表示記憶永遠不會被遺忘，使得該檢索器相當於向量查找。

```python
# Define your embedding model
embeddings_model = OpenAIEmbeddings()

# Initialize the vectorstore as empty
embedding_size = 1536

index = faiss.IndexFlatL2(embedding_size)

vectorstore = FAISS(embeddings_model.embed_query, index, InMemoryDocstore({}), {})

retriever = TimeWeightedVectorStoreRetriever(vectorstore=vectorstore, decay_rate=.0000000000000000000000001, k=1)

yesterday = datetime.now() - timedelta(days=1)

retriever.add_documents([Document(page_content="hello world", metadata={"last_accessed_at": yesterday})])

retriever.add_documents([Document(page_content="hello foo")])
```

結果:

```
    ['d7f85756-2371-4bdf-9140-052780a0f9b3']
```

```python
# "Hello World" is returned first because it is most salient, and the decay rate is close to 0., meaning it's still recent enough
retriever.get_relevant_documents("hello world")
```

結果:

```
    [Document(page_content='hello world', metadata={'last_accessed_at': datetime.datetime(2023, 5, 13, 21, 0, 27, 678341), 'created_at': datetime.datetime(2023, 5, 13, 21, 0, 27, 279596), 'buffer_idx': 0})]
```

## 高衰減率

---

如果衰減率較高（例如，幾個 `9`），`recency score` 很快就會變成 `0`！如果將其一直設為 `1`，則所有物件的新近度均為 `0`，這再次相當於向量查找。

```python
# Define your embedding model
embeddings_model = OpenAIEmbeddings()

# Initialize the vectorstore as empty
embedding_size = 1536

index = faiss.IndexFlatL2(embedding_size)

vectorstore = FAISS(embeddings_model.embed_query, index, InMemoryDocstore({}), {})

retriever = TimeWeightedVectorStoreRetriever(vectorstore=vectorstore, decay_rate=.999, k=1)

yesterday = datetime.now() - timedelta(days=1)

retriever.add_documents([Document(page_content="hello world", metadata={"last_accessed_at": yesterday})])

retriever.add_documents([Document(page_content="hello foo")])
```

結果:

```
    ['40011466-5bbe-4101-bfd1-e22e7f505de2']
```

```python
# "Hello Foo" is returned first because "hello world" is mostly forgotten
retriever.get_relevant_documents("hello world")
```

結果:

```
    [Document(page_content='hello foo', metadata={'last_accessed_at': datetime.datetime(2023, 4, 16, 22, 9, 2, 494798), 'created_at': datetime.datetime(2023, 4, 16, 22, 9, 2, 178722), 'buffer_idx': 1})]
```

## 虛擬時間

---

使用 LangChain 中的一些實用程序，您可以模擬時間組件。

```python
from langchain.utils import mock_now
import datetime

# Notice the last access time is that date time
with mock_now(datetime.datetime(2011, 2, 3, 10, 11)):
    print(retriever.get_relevant_documents("hello world"))
```

結果:

```
    [Document(page_content='hello world', metadata={'last_accessed_at': MockDateTime(2011, 2, 3, 10, 11), 'created_at': datetime.datetime(2023, 5, 13, 21, 0, 27, 279596), 'buffer_idx': 0})]
```

