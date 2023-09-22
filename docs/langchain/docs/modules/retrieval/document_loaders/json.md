# JSON

> [JSON (JavaScript 物件表示法)](https://en.wikipedia.org/wiki/JSON)是一種開放標準檔案格式和資料交換格式，它使用人類可讀的文字來儲存和傳輸由屬性值對和陣列（或其他可序列化值）組成的資料物件。

> [JSON Lines](https://jsonlines.org/) 是一種檔案格式，每一行都是有效的 JSON 值。

> `JSONLoader` 使用指定的 [`jq schema`](https://en.wikipedia.org/wiki/Jq_(programming_language)) 來解析 JSON 檔案。它使用 `jq` python 套件。請查看本[手冊](https://stedolan.github.io/jq/manual/#Basicfilters)以取得 `jq` 語法的詳細文件。

```bash
pip install jq
```

```python
from langchain.document_loaders import JSONLoader

import json
from pathlib import Path
from pprint import pprint


file_path='./example_data/facebook_chat.json'

data = json.loads(Path(file_path).read_text())

pprint(data)
```

結果:

```python
{'image': {'creation_timestamp': 1675549016, 'uri': 'image_of_the_chat.jpg'},
'is_still_participant': True,
'joinable_mode': {'link': '', 'mode': 1},
'magic_words': [],
'messages': [{'content': 'Bye!',
            'sender_name': 'User 2',
            'timestamp_ms': 1675597571851},
            {'content': 'Oh no worries! Bye',
            'sender_name': 'User 1',
            'timestamp_ms': 1675597435669},
            {'content': 'No Im sorry it was my mistake, the blue one is not '
                        'for sale',
            'sender_name': 'User 2',
            'timestamp_ms': 1675596277579},
            {'content': 'I thought you were selling the blue one!',
            'sender_name': 'User 1',
            'timestamp_ms': 1675595140251},
            {'content': 'Im not interested in this bag. Im interested in the '
                        'blue one!',
            'sender_name': 'User 1',
            'timestamp_ms': 1675595109305},
            {'content': 'Here is $129',
            'sender_name': 'User 2',
            'timestamp_ms': 1675595068468},
            {'photos': [{'creation_timestamp': 1675595059,
                        'uri': 'url_of_some_picture.jpg'}],
            'sender_name': 'User 2',
            'timestamp_ms': 1675595060730},
            {'content': 'Online is at least $100',
            'sender_name': 'User 2',
            'timestamp_ms': 1675595045152},
            {'content': 'How much do you want?',
            'sender_name': 'User 1',
            'timestamp_ms': 1675594799696},
            {'content': 'Goodmorning! $50 is too low.',
            'sender_name': 'User 2',
            'timestamp_ms': 1675577876645},
            {'content': 'Hi! Im interested in your bag. Im offering $50. Let '
                        'me know if you are interested. Thanks!',
            'sender_name': 'User 1',
            'timestamp_ms': 1675549022673}],
'participants': [{'name': 'User 1'}, {'name': 'User 2'}],
'thread_path': 'inbox/User 1 and User 2 chat',
'title': 'User 1 and User 2 chat'}
```

## 使用 `JSONLoader`

假設我們有興趣提取 JSON 資料的 `messages` 鍵中的 `context` 欄位下的值。這可以透過 `JSONLoader` 輕鬆完成，如下所示。

### JSON 格式檔案

```python
loader = JSONLoader(
    file_path='./example_data/facebook_chat.json',
    jq_schema='.messages[].content')

data = loader.load()

pprint(data)
```

結果:

```python
[Document(page_content='Bye!', metadata={'source': '/Users/avsolatorio/WBG/langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 1}),
Document(page_content='Oh no worries! Bye', metadata={'source': '/Users/avsolatorio/WBG/langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 2}),
Document(page_content='No Im sorry it was my mistake, the blue one is not for sale', metadata={'source': '/Users/avsolatorio/WBG/langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 3}),
Document(page_content='I thought you were selling the blue one!', metadata={'source': '/Users/avsolatorio/WBG/langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 4}),
Document(page_content='Im not interested in this bag. Im interested in the blue one!', metadata={'source': '/Users/avsolatorio/WBG/langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 5}),
Document(page_content='Here is $129', metadata={'source': '/Users/avsolatorio/WBG/langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 6}),
Document(page_content='', metadata={'source': '/Users/avsolatorio/WBG/langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 7}),
Document(page_content='Online is at least $100', metadata={'source': '/Users/avsolatorio/WBG/langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 8}),
Document(page_content='How much do you want?', metadata={'source': '/Users/avsolatorio/WBG/langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 9}),
Document(page_content='Goodmorning! $50 is too low.', metadata={'source': '/Users/avsolatorio/WBG/langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 10}),
Document(page_content='Hi! Im interested in your bag. Im offering $50. Let me know if you are interested. Thanks!', metadata={'source': '/Users/avsolatorio/WBG/langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 11})]
```

### JSON Lines 格式檔案

如果要從 JSON Lines 格式的文件載入文本，請傳遞 `json_lines=True` 並指定 `jq_schema` 以從單一 JSON 物件中提取 `page_content`。

```python
file_path = './example_data/facebook_chat_messages.jsonl'

pprint(Path(file_path).read_text())
```

結果:

```console
('{"sender_name": "User 2", "timestamp_ms": 1675597571851, "content": "Bye!"}\n'
    '{"sender_name": "User 1", "timestamp_ms": 1675597435669, "content": "Oh no '
    'worries! Bye"}\n'
    '{"sender_name": "User 2", "timestamp_ms": 1675596277579, "content": "No Im '
    'sorry it was my mistake, the blue one is not for sale"}\n')
```

```python
loader = JSONLoader(
    file_path='./example_data/facebook_chat_messages.jsonl',
    jq_schema='.content',
    json_lines=True)

data = loader.load()

pprint(data)
```

結果:

```python
[Document(page_content='User 2', metadata={'source': 'langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat_messages.jsonl', 'seq_num': 1}),
    Document(page_content='User 1', metadata={'source': 'langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat_messages.jsonl', 'seq_num': 2}),
    Document(page_content='User 2', metadata={'source': 'langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat_messages.jsonl', 'seq_num': 3})]
```

另一個選項是設定 `jq_schema='.'` 並提供 `content_key`：

```python
loader = JSONLoader(
    file_path='./example_data/facebook_chat_messages.jsonl',
    jq_schema='.',
    content_key='sender_name',
    json_lines=True)

data = loader.load()

pprint(data)
```

結果:

```python
[Document(page_content='User 2', metadata={'source': 'langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat_messages.jsonl', 'seq_num': 1}),
Document(page_content='User 1', metadata={'source': 'langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat_messages.jsonl', 'seq_num': 2}),
Document(page_content='User 2', metadata={'source': 'langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat_messages.jsonl', 'seq_num': 3})]
```

## 擷取元數據

通常，我們希望將 JSON 檔案中可用的元資料包含到我們根據內容建立的 `document` 物件實例中。

下面示範如何使用 [JSONLoader](https://api.python.langchain.com/en/latest/document_loaders/langchain.document_loaders.json_loader.JSONLoader.html?highlight=jsonloader#langchain.document_loaders.json_loader.JSONLoader) 提取元資料。

有一些關鍵變化需要注意。在前面的範例中，我們沒有收集元數據，我們設法直接在架構中指定可以從中提取 `page_content` 值的位置。

```python
.messages[].content
```

在目前範例中，我們必須告訴 loader 來迭代 `messages` 欄位中的記錄。 `jq_schema` 必須是：

```python
.messages[]
```

這允許我們將記錄（dict）傳遞到必須實現的 `metadata_func`。 `metadata_func` 負責識別記錄中的哪些資訊應包含在最終 `Document` 物件中儲存的元資料中。

此外，我們現在必須在載入器中透過 `content_key` 參數明確指定需要從中提取 `page_content` 值的記錄中的鍵。

```python
# Define the metadata extraction function.
def metadata_func(record: dict, metadata: dict) -> dict:

    metadata["sender_name"] = record.get("sender_name")
    metadata["timestamp_ms"] = record.get("timestamp_ms")

    return metadata


loader = JSONLoader(
    file_path='./example_data/facebook_chat.json',
    jq_schema='.messages[]',
    content_key="content",
    metadata_func=metadata_func
)

data = loader.load()

pprint(data)
```

結果:

```python
[Document(page_content='Bye!', metadata={'source': '/Users/avsolatorio/WBG/langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 1, 'sender_name': 'User 2', 'timestamp_ms': 1675597571851}),
Document(page_content='Oh no worries! Bye', metadata={'source': '/Users/avsolatorio/WBG/langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 2, 'sender_name': 'User 1', 'timestamp_ms': 1675597435669}),
Document(page_content='No Im sorry it was my mistake, the blue one is not for sale', metadata={'source': '/Users/avsolatorio/WBG/langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 3, 'sender_name': 'User 2', 'timestamp_ms': 1675596277579}),
Document(page_content='I thought you were selling the blue one!', metadata={'source': '/Users/avsolatorio/WBG/langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 4, 'sender_name': 'User 1', 'timestamp_ms': 1675595140251}),
Document(page_content='Im not interested in this bag. Im interested in the blue one!', metadata={'source': '/Users/avsolatorio/WBG/langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 5, 'sender_name': 'User 1', 'timestamp_ms': 1675595109305}),
Document(page_content='Here is $129', metadata={'source': '/Users/avsolatorio/WBG/langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 6, 'sender_name': 'User 2', 'timestamp_ms': 1675595068468}),
Document(page_content='', metadata={'source': '/Users/avsolatorio/WBG/langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 7, 'sender_name': 'User 2', 'timestamp_ms': 1675595060730}),
Document(page_content='Online is at least $100', metadata={'source': '/Users/avsolatorio/WBG/langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 8, 'sender_name': 'User 2', 'timestamp_ms': 1675595045152}),
Document(page_content='How much do you want?', metadata={'source': '/Users/avsolatorio/WBG/langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 9, 'sender_name': 'User 1', 'timestamp_ms': 1675594799696}),
Document(page_content='Goodmorning! $50 is too low.', metadata={'source': '/Users/avsolatorio/WBG/langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 10, 'sender_name': 'User 2', 'timestamp_ms': 1675577876645}),
Document(page_content='Hi! Im interested in your bag. Im offering $50. Let me know if you are interested. Thanks!', metadata={'source': '/Users/avsolatorio/WBG/langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 11, 'sender_name': 'User 1', 'timestamp_ms': 1675549022673})]
```

現在，您將看到文件包含與我們提取的內容關聯的元資料。

## 定義 `metadata_func`

如上所示，`metadata_func` 接受 `JSONLoader` 產生的預設元資料。這允許使用者完全控制元資料的格式。

例如，預設元資料包含 `source` 和 `seq_num` 鍵。但是，JSON 資料也可能包含這些鍵。然後，使用者可以利用 `metadata_func` 重命名預設鍵並使用 JSON 資料中的鍵。

下面的範例展示如何修改 `source` 以僅包含相對於 langchain 目錄的檔案來源資訊。

```python
# Define the metadata extraction function.
def metadata_func(record: dict, metadata: dict) -> dict:

    metadata["sender_name"] = record.get("sender_name")
    metadata["timestamp_ms"] = record.get("timestamp_ms")

    if "source" in metadata:
        source = metadata["source"].split("/")
        source = source[source.index("langchain"):]
        metadata["source"] = "/".join(source)

    return metadata


loader = JSONLoader(
    file_path='./example_data/facebook_chat.json',
    jq_schema='.messages[]',
    content_key="content",
    metadata_func=metadata_func
)

data = loader.load()

pprint(data)
```

結果:

```python
[Document(page_content='Bye!', metadata={'source': 'langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 1, 'sender_name': 'User 2', 'timestamp_ms': 1675597571851}),
Document(page_content='Oh no worries! Bye', metadata={'source': 'langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 2, 'sender_name': 'User 1', 'timestamp_ms': 1675597435669}),
Document(page_content='No Im sorry it was my mistake, the blue one is not for sale', metadata={'source': 'langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 3, 'sender_name': 'User 2', 'timestamp_ms': 1675596277579}),
Document(page_content='I thought you were selling the blue one!', metadata={'source': 'langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 4, 'sender_name': 'User 1', 'timestamp_ms': 1675595140251}),
Document(page_content='Im not interested in this bag. Im interested in the blue one!', metadata={'source': 'langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 5, 'sender_name': 'User 1', 'timestamp_ms': 1675595109305}),
Document(page_content='Here is $129', metadata={'source': 'langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 6, 'sender_name': 'User 2', 'timestamp_ms': 1675595068468}),
Document(page_content='', metadata={'source': 'langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 7, 'sender_name': 'User 2', 'timestamp_ms': 1675595060730}),
Document(page_content='Online is at least $100', metadata={'source': 'langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 8, 'sender_name': 'User 2', 'timestamp_ms': 1675595045152}),
Document(page_content='How much do you want?', metadata={'source': 'langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 9, 'sender_name': 'User 1', 'timestamp_ms': 1675594799696}),
Document(page_content='Goodmorning! $50 is too low.', metadata={'source': 'langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 10, 'sender_name': 'User 2', 'timestamp_ms': 1675577876645}),
Document(page_content='Hi! Im interested in your bag. Im offering $50. Let me know if you are interested. Thanks!', metadata={'source': 'langchain/docs/modules/indexes/document_loaders/examples/example_data/facebook_chat.json', 'seq_num': 11, 'sender_name': 'User 1', 'timestamp_ms': 1675549022673})]
```

## 常見 JSON 結構的 `jq schema`

下面的列表提供了對可能的 `jq_schema` 的引用，使用者可以使用它根據結構從 JSON 資料中提取內容。

```
JSON        -> [{"text": ...}, {"text": ...}, {"text": ...}]
jq_schema   -> ".[].text"

JSON        -> {"key": [{"text": ...}, {"text": ...}, {"text": ...}]}
jq_schema   -> ".key[].text"

JSON        -> ["...", "...", "..."]
jq_schema   -> ".[]"
```