# Document loaders

!!! info
    查看 [Integrations](https://python.langchain.com/docs/integrations/document_loaders) 以取得 langchain 內建的文件載入器與第三方工具整合的文件。

![](../assets/data_connection-c42d68c3d092b85f50d08d4cc171fc25.jpg)

使用文件載入器從文件檔案來源載入資料。[Document](https://api.python.langchain.com/en/latest/schema/langchain.schema.document.Document.html?highlight=document#langchain.schema.document.Document) 是一段文本和相關的元資料。例如，有一些文件載入器可以載入簡單的 `.txt` 檔案、載入任何網頁的文字內容，甚至載入 YouTube 影片的 transcript。

文件載入器提供了一種 "load" 方法，用於從配置的來源將資料載入為文件。它們還可以選擇實現 "延遲載入(lazy load)"，以便將資料延遲載入到記憶體中。

## 開始使用

最簡單的 loader 將文件作為文字讀入，並將其全部放入一個文件中。

```python
from langchain.document_loaders import TextLoader

loader = TextLoader("./index.md")
loader.load()
```

結果:

```python
[
    Document(page_content='---\nsidebar_position: 0\n---\n# Document loaders\n\nUse document loaders to load data from a source as `Document`\'s. A `Document` is a piece of text\nand associated metadata. For example, there are document loaders for loading a simple `.txt` file, for loading the text\ncontents of any web page, or even for loading a transcript of a YouTube video.\n\nEvery document loader exposes two methods:\n1. "Load": load documents from the configured source\n2. "Load and split": load documents from the configured source and split them using the passed in text splitter\n\nThey optionally implement:\n\n3. "Lazy load": load documents into memory lazily\n', metadata={'source': '../docs/docs_skeleton/docs/modules/data_connection/document_loaders/index.md'})
]
```