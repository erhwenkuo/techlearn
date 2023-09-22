# 索引

在這裡，我們將了解使用 LangChain 索引 API 的基本索引工作流程。

索引 API 允許您將任何來源的文件載入到向量儲存中並保持同步。具體來說，它有助於：

- 避免將重複的內容寫入向量存儲
- 避免重寫未更改的內容
- 避免在未更改的內容上重新計算嵌入

所有這些都可以節省您的時間和金錢，並改善您的向量搜尋結果。

至關重要的是，索引 API 甚至可以處理相對於原始來源文件經歷了多個轉換步驟(例如，透過 text chunking)的文檔。

## 運作機制

LangChain 索引利用記錄管理器 (`RecordManager`) 來追蹤文件寫入向量儲存的情況。

索引內容時，會計算每個文件的雜湊值，並將以下資訊儲存在記錄管理器中：

- document hash (page content 和 metadata 的 hash)
- write time
- source id - 每個文件應在其元資料中包含此信息，以便我們確定該文件的最終來源

## 刪除模式

將文件索引到向量儲存時，可能會刪除向量儲存中的某些現有文件。在某些情況下，您可能想要刪除與正在編制索引的新文件源自相同來源的任何現有文件。在有些情況下，您可能希望大量刪除所有現有文件。

索引 API 刪除模式可讓您選擇所需的行為：

|刪除模式|刪除重複內容|可併行化|清理已刪除的來源文檔|清理來源文檔和/或派生文檔|清理時間|
|-------|---------|-------|-------------------------------|-----|------|
|`None`|✅|✅|❌|❌|-|
|`incremental` - 增量|✅|✅|❌|✅|Continuously|
|`full` - 完整|✅|❌|✅|✅|At end of indexing|

`None` 不會執行任何自動清理，允許使用者手動清理舊內容。

`incremental` 和 `full` 提供以下自動清理功能：

- 如果來源文件或衍生文件的內容發生更改，則增量(incremental)或完整(full)模式都會清理（刪除）內容的先前版本。
- 如果來源文件已被刪除（表示它不包含在目前正在索引的文件中），則完全清理(full)模式將正確地將其從向量儲存中刪除，但增量(incremental)模式不會。

當內容變更（例如，來源 PDF 檔案被修改）時，在索引期間將有一段時間新舊版本都可能傳回給使用者。這種情況發生在新內容寫入之後、舊版被刪除之前。

- 增量索引可以最大限度地減少新舊交替的時間，因為它能夠在寫入時連續清理。
- 完整模式在所有批次寫入後才開始進行清理。

## 要求

1. 不要與已預先填入了獨立於索引 API 的內容的儲存空間一起使用，因為 `record manager` 不會知道先前已插入記錄。
2. 僅適用於支援的 LangChain 向量存儲
    - 按 `id` 新增文件（帶有 `ids` 參數的 `add_documents` 方法）
    - 按 `id` 刪除（刪除方法有 `id` 參數）

## 注意事項

`Record manager` 依靠基於時間的機制來確定可以清理哪些內容（當使用 `full` 或 `incremental` 清理模式時）。

如果兩個任務連續運行，且第一個任務在時鐘時間變更之前完成，則第二個任務可能無法清理內容。

這在實際設定中不太可能成為問題，原因如下：

1. RecordManager 使用更高解析度的時間戳記。
2. 資料需要在第一個和第二個
任務運行之間發生變化，如果任務之間的時間間隔很小時，這種情況就不太可能發生。
3. 索引任務通常只需要幾毫秒的時間。

## 快速開始

```python
from langchain.embeddings import OpenAIEmbeddings
from langchain.indexes import SQLRecordManager, index
from langchain.schema import Document
from langchain.vectorstores import ElasticsearchStore
```

**API Reference:**

- [OpenAIEmbeddings](https://api.python.langchain.com/en/latest/embeddings/langchain.embeddings.openai.OpenAIEmbeddings.html)
- [SQLRecordManager](https://api.python.langchain.com/en/latest/indexes/langchain.indexes._sql_record_manager.SQLRecordManager.html)
- [index](https://api.python.langchain.com/en/latest/indexes/langchain.indexes._api.index.html)
- [Document](https://api.python.langchain.com/en/latest/schema/langchain.schema.document.Document.html)
- [ElasticsearchStore](https://api.python.langchain.com/en/latest/vectorstores/langchain.vectorstores.elasticsearch.ElasticsearchStore.html)

初始化向量儲存並設定嵌入：

```python
collection_name = "test_index"

embedding = OpenAIEmbeddings()

vectorstore = ElasticsearchStore(
    es_url="http://localhost:9200", index_name="test_index", embedding=embedding
)
```

使用適當的命名空間初始化 `record manager`。

**Suggestion:** 同時考慮向量儲存和向量儲存中的集合名稱的命名空間；例如，`redis/my_docs`、`chromadb/my_docs` 或 `postgres/my_docs`。

```python
namespace = f"elasticsearch/{collection_name}"

# 構建一個 record manager 實例來監控 Document (CUD)
record_manager = SQLRecordManager(
    namespace, db_url="sqlite:///record_manager_cache.sql"
)
```

在使用 record manager 之前先建立 schema。

```python
record_manager.create_schema()
```

讓我們索引一些測試文件:

```python
doc1 = Document(page_content="kitty", metadata={"source": "kitty.txt"})
doc2 = Document(page_content="doggy", metadata={"source": "doggy.txt"})
```

索引到一個空的向量存儲中：

```python
def _clear():
    """Hacky helper method to clear content. See the `full` mode section to to understand why it works."""
    index([], record_manager, vectorstore, cleanup="full", source_id_key="source")
```

### `None` 刪除模式

此模式不會自動清理舊版的內容；但是，它仍然負責內容重複刪除。

清除/重置:

```python
_clear()
```

索引文件:

```python
index(
    [doc1, doc1, doc1, doc1, doc1],
    record_manager,
    vectorstore,
    cleanup=None,
    source_id_key="source",
)
```

結果:

```bash
{'num_added': 1, 'num_updated': 0, 'num_skipped': 0, 'num_deleted': 0}
```

清除/重置:

```python
_clear()
```

索引文件:

```python
index(
    [doc1, doc2], record_manager, vectorstore, cleanup=None, source_id_key="source"
)
```

結果:

```bash
{'num_added': 2, 'num_updated': 0, 'num_skipped': 0, 'num_deleted': 0}
```

第二次將跳過所有內容：

索引文件:

```python
index(
    [doc1, doc2], record_manager, vectorstore, cleanup=None, source_id_key="source"
)
```

結果:

```bash
{'num_added': 0, 'num_updated': 0, 'num_skipped': 2, 'num_deleted': 0}
```

### `incremental` 刪除模式

清除/重置:

```python
_clear()
```

索引文件:

```python
index(
    [doc1, doc2],
    record_manager,
    vectorstore,
    cleanup="incremental",
    source_id_key="source",
)
```

結果:

```bash
{'num_added': 2, 'num_updated': 0, 'num_skipped': 0, 'num_deleted': 0}
```

再次索引應該會導致兩個文件都被 **skipped** — 同時也會跳過 embedding 的作業！

索引文件:

```python
index(
    [doc1, doc2],
    record_manager,
    vectorstore,
    cleanup="incremental",
    source_id_key="source",
)
```

結果:

```bash
{'num_added': 0, 'num_updated': 0, 'num_skipped': 2, 'num_deleted': 0}
```

如果我們不提供增量索引模式的文檔，則什麼都不會改變。

索引文件:

```python
index(
    [], record_manager, vectorstore, cleanup="incremental", source_id_key="source"
)
```

結果:

```bash
{'num_added': 0, 'num_updated': 0, 'num_skipped': 0, 'num_deleted': 0}
```

如果我們改變一個文檔，新版本將被寫入，所有共享相同來源的舊版本將被刪除。

```python
changed_doc_2 = Document(page_content="puppy", metadata={"source": "doggy.txt"})
```

索引文件:

```python
index(
    [changed_doc_2],
    record_manager,
    vectorstore,
    cleanup="incremental",
    source_id_key="source",
)
```

結果:

```bash
{'num_added': 1, 'num_updated': 0, 'num_skipped': 0, 'num_deleted': 1}
```

### `full` 刪除模式

在完整模式下，使用者應該傳遞應索引到索引功能中的完整內容。

任何未傳遞到索引功能且存在於向量儲存中的文件都將被刪除！

此行為對於處理來源文件的刪除很有用。

清除/重置:

```python
_clear()
```

索引文件:

```python
all_docs = [doc1, doc2]
```

```python
index(all_docs, record_manager, vectorstore, cleanup="full", source_id_key="source")
```

結果:

```bash
{'num_added': 2, 'num_updated': 0, 'num_skipped': 0, 'num_deleted': 0}
```

假設有人刪除了第一個文檔：

```python
del all_docs[0]
```

顯示現在在 Vector store 的所有文件:

```python
all_docs
```

結果:

```bash
[Document(page_content='doggy', metadata={'source': 'doggy.txt'})]
```

使用完整模式也會清除已刪除的內容。

索引文件:

```python
index(all_docs, record_manager, vectorstore, cleanup="full", source_id_key="source")
```

結果:

```bash
{'num_added': 0, 'num_updated': 0, 'num_skipped': 1, 'num_deleted': 1}
```

## Source 元數據

元資料屬性包含一個稱為 `source` 的欄位。該 `source` 應指出與給定文檔相關的最終出處。

例如，如果這些文檔代表某個父文檔的區塊，則兩個文檔的來源應該相同並引用父文檔。

一般來說，應始終指定來源。如果您從不打算使用增量模式，並且由於某種原因無法正確指定來源字段，則僅使用  `None`。

```python
from langchain.text_splitter import CharacterTextSplitter
```

**API Reference:**

- [CharacterTextSplitter](https://api.python.langchain.com/en/latest/text_splitter/langchain.text_splitter.CharacterTextSplitter.html)

```python
doc1 = Document(
    page_content="kitty kitty kitty kitty kitty", metadata={"source": "kitty.txt"}
)

doc2 = Document(
    page_content="doggy doggy the doggy", metadata={"source": "doggy.txt"}
)
```

```python
new_docs = CharacterTextSplitter(
    separator="t", keep_separator=True, chunk_size=12, chunk_overlap=2
).split_documents([doc1, doc2])

new_docs
```

結果:

```python
[
    Document(page_content='kitty kit', metadata={'source': 'kitty.txt'}),
    Document(page_content='tty kitty ki', metadata={'source': 'kitty.txt'}),
    Document(page_content='tty kitty', metadata={'source': 'kitty.txt'}),
    Document(page_content='doggy doggy', metadata={'source]': 'doggy.txt'}),
    Document(page_content='the doggy', metadata={'source': 'doggy.txt'})
]
```

清除/重置:

```python
_clear()
```

索引文件:

```python
index(
    new_docs,
    record_manager,
    vectorstore,
    cleanup="incremental",
    source_id_key="source",
)
```

結果:

```python
{'num_added': 5, 'num_updated': 0, 'num_skipped': 0, 'num_deleted': 0}
```

文件準備:

```python
changed_doggy_docs = [
    Document(page_content="woof woof", metadata={"source": "doggy.txt"}),
    Document(page_content="woof woof woof", metadata={"source": "doggy.txt"}),
]
```

這應該刪除與 `doggy.txt` 來源關聯的文件的舊版本，並將其替換為新版本。

索引文件:

```python
index(
    changed_doggy_docs,
    record_manager,
    vectorstore,
    cleanup="incremental",
    source_id_key="source",
)
```

結果:

```python
{'num_added': 0, 'num_updated': 0, 'num_skipped': 2, 'num_deleted': 2}
```

檢索文件:

```python
vectorstore.similarity_search("dog", k=30)
```

結果:

```python
[Document(page_content='tty kitty', metadata={'source': 'kitty.txt'}),
    Document(page_content='tty kitty ki', metadata={'source': 'kitty.txt'}),
    Document(page_content='kitty kit', metadata={'source': 'kitty.txt'})]
```

## 與 Loader 一起使用

索引可以接受 iterable of document 或任何 loader。

!!! warning
    Loader 必須正確設定 source 密鑰。

```python
from langchain.document_loaders.base import BaseLoader


class MyCustomLoader(BaseLoader):

    def lazy_load(self):
        text_splitter = CharacterTextSplitter(
            separator="t", keep_separator=True, chunk_size=12, chunk_overlap=2
        )
        docs = [
            Document(page_content="woof woof", metadata={"source": "doggy.txt"}),
            Document(page_content="woof woof woof", metadata={"source": "doggy.txt"}),
        ]
        yield from text_splitter.split_documents(docs)

    def load(self):
        return list(self.lazy_load())
```

**API Reference:**

- [BaseLoader](https://api.python.langchain.com/en/latest/document_loaders/langchain.document_loaders.base.BaseLoader.html)


清除/重置:

```python
_clear()
```

構建 loader 實例:

```python
loader = MyCustomLoader()
```

啟動 lazzy load:

```python
loader.load()
```

索引文件:

```python
index(loader, record_manager, vectorstore, cleanup="full", source_id_key="source")
```

結果:

```python
{'num_added': 2, 'num_updated': 0, 'num_skipped': 0, 'num_deleted': 0}
```

檢索文件:

```python
vectorstore.similarity_search("dog", k=30)
```

結果:

```python
[Document(page_content='tty kitty', metadata={'source': 'kitty.txt'}),
    Document(page_content='tty kitty ki', metadata={'source': 'kitty.txt'}),
    Document(page_content='kitty kit', metadata={'source': 'kitty.txt'})]
```

檢索文件:

```python
vectorstore.similarity_search("dog", k=30)
```

結果:

```python
[Document(page_content='woof woof', metadata={'source': 'doggy.txt'}),
    Document(page_content='woof woof woof', metadata={'source': 'doggy.txt'})]
```