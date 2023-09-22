# HTML

> 超文本標記語言或 HTML 是設計用於在 Web 瀏覽器中顯示的文件的標準標記語言。

這涵蓋瞭解如何將 HTML 文件載入為我們可以在下游應用程式可使用的文件格式。

```python
from langchain.document_loaders import UnstructuredHTMLLoader

loader = UnstructuredHTMLLoader("example_data/fake-content.html")

data = loader.load()

print(data)
```

結果:

```python
[Document(page_content='My First Heading\n\nMy first paragraph.', lookup_str='', metadata={'source': 'example_data/fake-content.html'}, lookup_index=0)]
```

## 使用 BeautifulSoup4 載入 HTML

我們也可以使用 `BeautifulSoup4` 使用 `BSHTMLLoader` 載入 HTML 文件。這會將 HTML 中的文字提取到 `page_content` 中，並將頁面標題提取到元資料中。

```python
from langchain.document_loaders import BSHTMLLoader

loader = BSHTMLLoader("example_data/fake-content.html")

data = loader.load()

print(data)
```

結果:

```python
[Document(page_content='\n\nTest Title\n\n\nMy First Heading\nMy first paragraph.\n\n\n', metadata={'source': 'example_data/fake-content.html', 'title': 'Test Title'})]
```
