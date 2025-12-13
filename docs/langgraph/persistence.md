# Persistence

LangGraph 內建了持久層(persistence layer)，透過 checkpointers 實現。當您使用 checkpointer 編譯 graph 時，檢查點會在每個 super-step 中儲存 graph 狀態的 `checkpoint`。這些 checkpoint 保存在一個 `thread` 中，可以在 graph 執行完畢後存取。由於 `thread` 允許在執行完畢後存取 graph 的狀態，因此可以實現許多強大的功能，包括人機互動、記憶體存取、時間旅行和容錯。下面，我們將更詳細地討論這些概念。

![](./assets/checkpoints.avif)

!!! info
    LangGraph API 會自動處理 checkpointing。使用 LangGraph API 時，您無需手動實作或設定 checkpointers。 API 會在背景為您處理所有持久化基礎架構。

## Threads

Thread 是分配給每個由 checkpointing 保存的 checkpointer的唯一 ID 或執行緒標識符。它包含一系列運行的累積狀態。當執行一次 run，助手底層 graph 的狀態將會持久化到該執行緒。

使用檢查點呼叫 graph 時，必須在配置的可設定部分中指定 `thread_id`:

```json
{"configurable": {"thread_id": "1"}}
```

透過 `thread_id` 可以檢索一個 thread 當前狀態和歷史狀態。要持久化狀態，必須在執行運行之前建立 thread。 LangSmith API 提供了多個用於建立和管理 thread 及 thread state 的端點。更多詳情請參閱 [API 參考文件](https://reference.langchain.com/python/langsmith/?_gl=1*4i8rk5*_gcl_au*MTA0NTc2NjA2OC4xNzYzNzI5OTg2*_ga*MTM5NTM1ODE0Mi4xNzQ3MDkyMTE0*_ga_47WX3HKKY2*czE3NjQ5ODM4ODMkbzUwJGcxJHQxNzY0OTg4MjA5JGoyOSRsMCRoMA..)。


檢查點器使用 `thread_id` 作為儲存和檢索檢查點的主鍵。如果沒有 `thread_id`，檢查點器將無法儲存狀態或在中斷後恢復執行，因為檢查點器使用 `thread_id` 來載入已儲存的狀態。

## Checkpoints

Thread 在特定時間點的狀態稱為檢查點(checkpoint)。檢查點是每個超級步驟保存的 graph 狀態快照，由具有以下關鍵屬性的 `StateSnapshot` 物件表示:

- `config`: 與此檢查點關聯的配置。
- `metadata`: 與此檢查點相關的元資料。
- `values`: 此時各狀態通道的值。
- `next`: Graph 中接下來要執行的節點名稱元組。
- `tasks`: 一個包含 `PregelTask`​​ 物件元組的集合，其中包含有關要執行的後續任務的資訊。如果該步驟之前已嘗試過，則會包含錯誤訊息。如果 graph 在節點內部被動態中斷，則任務將包含與中斷相關的附加資料。

Checkpoints 會被持久化，並可用於在稍後恢復執行緒的狀態。

讓我們看看當呼叫如下所示的簡單 graph 時，會儲存哪些檢查點：

```python
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver
from langchain_core.runnables import RunnableConfig
from typing import Annotated
from typing_extensions import TypedDict
from operator import add

class State(TypedDict):
    foo: str
    bar: Annotated[list[str], add]

def node_a(state: State):
    return {"foo": "a", "bar": ["a"]}

def node_b(state: State):
    return {"foo": "b", "bar": ["b"]}


workflow = StateGraph(State)
workflow.add_node(node_a)
workflow.add_node(node_b)
workflow.add_edge(START, "node_a")
workflow.add_edge("node_a", "node_b")
workflow.add_edge("node_b", END)

checkpointer = InMemorySaver()
graph = workflow.compile(checkpointer=checkpointer)

config: RunnableConfig = {"configurable": {"thread_id": "1"}}

graph.invoke({"foo": "", "bar":[]}, config)
```

運行 graph 後，我們預期會看到 4 個檢查點：

- 空檢查點，下一個要執行的節點為 `START`
- 檢查點，使用者輸入為 `{'foo': '', 'bar': []}`，下一個要執行的節點為 `node_a`
- 檢查點，`node_a` 的輸出為 `{'foo': 'a', 'bar': ['a']}`，下一個要執行的節點為 `node_b`
- 檢查點，`node_b` 的輸出為 `{'foo': 'b', 'bar': ['a', 'b']}`，沒有下一個要執行的節點

請注意，由於我們有 `bar` 通道的 reducer，因此 `bar` 通道值包含來自兩個節點的輸出。

### Get state

與已儲存的 graph 狀態互動時，必須指定 **thread identifier**。您可以透過呼叫 `graph.get_state(config)` 查看 graph 的最新狀態。這將傳回一個 `StateSnapshot` 物件，該物件對應於配置中提供的 Thread ID 關聯的最新檢查點，或者如果提供了 thread 的檢查點 ID，則傳回與該檢查點關聯的檢查點。

```python
# 取得最新的狀態快照
config = {"configurable": {"thread_id": "1"}}

graph.get_state(config)

# 取得特定 checkpoint_id 的狀態快照
config = {"configurable": {"thread_id": "1", "checkpoint_id": "1ef663ba-28fe-6528-8002-5a559208592c"}}

graph.get_state(config)
```

在我們的範例中，`get_state` 的輸出將如下所示：

```python
StateSnapshot(
    values={'foo': 'b', 'bar': ['a', 'b']},
    next=(),
    config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28fe-6528-8002-5a559208592c'}},
    metadata={'source': 'loop', 'writes': {'node_b': {'foo': 'b', 'bar': ['b']}}, 'step': 2},
    created_at='2024-08-29T19:19:38.821749+00:00',
    parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f9-6ec4-8001-31981c2c39f8'}}, tasks=()
)
```

### Get state history

您可以透過呼叫 `graph.get_state_history(config)` 來取得給定執行緒的完整 graph 來取得 graph 執行歷史記錄。這將傳回一個與配置中提供的執行緒 ID 關聯的 `StateSnapshot` 物件清單。重要的是，檢查點將按時間順序排列，最新的檢查點/`StateSnapshot` 位於清單頂部。

```python
config = {"configurable": {"thread_id": "1"}}

list(graph.get_state_history(config))
```

在我們的範例中，`get_state_history` 的輸出將如下所示：

```python
[
    StateSnapshot(
        values={'foo': 'b', 'bar': ['a', 'b']},
        next=(),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28fe-6528-8002-5a559208592c'}},
        metadata={'source': 'loop', 'writes': {'node_b': {'foo': 'b', 'bar': ['b']}}, 'step': 2},
        created_at='2024-08-29T19:19:38.821749+00:00',
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f9-6ec4-8001-31981c2c39f8'}},
        tasks=(),
    ),
    StateSnapshot(
        values={'foo': 'a', 'bar': ['a']},
        next=('node_b',),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f9-6ec4-8001-31981c2c39f8'}},
        metadata={'source': 'loop', 'writes': {'node_a': {'foo': 'a', 'bar': ['a']}}, 'step': 1},
        created_at='2024-08-29T19:19:38.819946+00:00',
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f4-6b4a-8000-ca575a13d36a'}},
        tasks=(PregelTask(id='6fb7314f-f114-5413-a1f3-d37dfe98ff44', name='node_b', error=None, interrupts=()),),
    ),
    StateSnapshot(
        values={'foo': '', 'bar': []},
        next=('node_a',),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f4-6b4a-8000-ca575a13d36a'}},
        metadata={'source': 'loop', 'writes': None, 'step': 0},
        created_at='2024-08-29T19:19:38.817813+00:00',
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f0-6c66-bfff-6723431e8481'}},
        tasks=(PregelTask(id='f1b14528-5ee5-579c-949b-23ef9bfbed58', name='node_a', error=None, interrupts=()),),
    ),
    StateSnapshot(
        values={'bar': []},
        next=('__start__',),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f0-6c66-bfff-6723431e8481'}},
        metadata={'source': 'input', 'writes': {'foo': ''}, 'step': -1},
        created_at='2024-08-29T19:19:38.816205+00:00',
        parent_config=None,
        tasks=(PregelTask(id='6d27aa2e-d72b-5504-a36f-8620e54a76dd', name='__start__', error=None, interrupts=()),),
    )
]
```

![](./assets/get_state.avif)

### Replay

透過 checkpoints 的機制就可回放之前的 graph 執行過程。如果我們使用 `thread_id` 和 `checkpoint_id` 呼叫一個 graph，那麼我們將重播與 `checkpoint_id` 對應的檢查點之前執行的步驟，並且只執行檢查點之後的步驟。

- `thread_id` 是 thread 的 ID。
- `checkpoint_id` 是指向 thread 中特定檢查點的識別碼。

在配置的可配置部分中呼叫該 graph 時，必須傳遞這些參數：

```python
config = {"configurable": {"thread_id": "1", "checkpoint_id": "0c62ca34-ac19-445d-bbb0-5b4984975b2a"}}

graph.invoke(None, config=config)
```

重要的是，LangGraph 能夠判斷某個步驟是否曾經執行過。如果已被執行過，LangGraph 只會重播 graph 中的該步驟，而不會重新執行該步驟本身，但僅限於指定的 `checkpoint_id` 之前的步驟。 `checkpoint_id` 之後的所有步驟都會被執行（即建立一個新的分支），即使它們之前已被執行過。請參閱這篇[關於 time-travel 的操作指南，以了解更多關於 Replay 的資訊](https://docs.langchain.com/oss/python/langgraph/use-time-travel)。

![](./assets/re_play.avif)

### Update state

除了從特定檢查點重播 graph 之外，我們還可以編輯 graph 的狀態。我們使用 `update_state` 方法來實現這一點。此方法接受三個不同的參數：

#### config

配置中應包含 `thread_id`，用於指定要更新的執行緒。如果僅傳遞 `thread_id`，則更新（或 fork）目前狀態。此外，如果包含 `checkpoint_id` 字段，則 fork 選定的檢查點。

#### values

這些值將用於更新狀態。請注意，此更新的處理方式與節點中的任何更新完全相同。這意味著，如果 graph 狀態中的某些通道定義了 `reducer` 函數，則這些值將傳遞給 `reducer` 函數。也就是說，`update_state` 不會自動覆寫每個通道的值，而只會覆寫沒有 `reducer` 的通道的值。讓我們來看一個例子。

假設您已使用以下模式定義了 graph 的狀態（請參閱上面的完整範例）：

```python
from typing import Annotated
from typing_extensions import TypedDict
from operator import add

class State(TypedDict):
    foo: int
    bar: Annotated[list[str], add]
```

現在假設 graph 的目前狀態為:

```python
{"foo": 1, "bar": ["a"]}
```

如果您如下更新狀態:

```python
graph.update_state(config, {"foo": 2, "bar": ["b"]})
```

那麼，graph 的新狀態將是:

```python
{"foo": 2, "bar": ["a", "b"]}
```

`foo` 鍵（channel）的狀態被完全改變了（因為沒有為該通道指定 reducer，所以 `update_state` 會覆寫它）。但是，`bar` 鍵指定了 reducer，因此它會將 `"b"` 新增至 `bar` 的狀態。

#### as_node

當呼叫 `update_state` 時，最後一個可選參數是 `as_node`。如果指定了 `as_node`，則更新將被視為來自 `as_node` 節點。如果未指定 `as_node`，則預設為上次更新狀態的節點（除非存在歧義）。這一點很重要，因為後續步驟的執行取決於上次更新的節點，因此可以使用 `as_node` 來控制接下來執行哪個節點的操作。

![](./assets/checkpoints_full_story.avif)

## Memory Store

![](./assets/shared_state.avif)

[state schema](https://docs.langchain.com/oss/python/langgraph/graph-api#schema) 指定了一組鍵，這些鍵會在執行 graph 時填入。如上所述，狀態可以透過檢查點在每個圖步驟中寫入線程，從而實現狀態持久化。

但是，如果我們想跨線程保留某些資訊該怎麼辦? 考慮一下聊天機器人的情況，我們希望在與該用戶的所有聊天對話（例如，threads）中保留有關該用戶的特定資訊！

僅靠檢查點，我們無法在 threads 間共享資訊。這就需要引入 [Store 介面](https://reference.langchain.com/python/langgraph/store/?_gl=1*13ug90c*_gcl_au*MTA0NTc2NjA2OC4xNzYzNzI5OTg2*_ga*MTM5NTM1ODE0Mi4xNzQ3MDkyMTE0*_ga_47WX3HKKY2*czE3NjQ5ODM4ODMkbzUwJGcxJHQxNzY0OTg5MzgzJGoxJGwwJGgw)。例如，我們可以定義一個 `InMemoryStore` 來跨 threads 儲存使用者資訊。我們只需像之前一樣，使用檢查點編譯我們的 graph，並新增新的 `in_memory_store` 變數即可。

!!! info
    LangGraph API 可自動處理 Stores。使用 LangGraph API 時，您無需手動實作或設定 Stores。 API 會在背景為您處理所有儲存基礎架構。

### Basic Usage

首先，讓我們在不使用 LangGraph 的情況下單獨展示一下。

```python
from langgraph.store.memory import InMemoryStore
in_memory_store = InMemoryStore()
```

Memory 內容透過 `tuple` 進行命名空間劃分，在本例中為 `(<user_id>, "memories")`。命名空間長度不限，可以代表任何內容，不必與使用者相關。

```python
user_id = "1"
namespace_for_memory = (user_id, "memories")
```

我們使用 `store.put` 方法將記憶體儲存到 store 中的命名空間。執行此操作時，我們需要指定命名空間（如上所述）以及 memory 的 key-value 鍵值對:

- key 是 memory 的唯一識別碼（`memory_id`）
- value 是（a dictionary）是 memory 本身。

```python
memory_id = str(uuid.uuid4())

memory = {"food_preference" : "I like pizza"}

in_memory_store.put(namespace_for_memory, memory_id, memory)
```

我們可以使用 `store.search` 方法讀取命名空間中的 memory，該方法會傳回給定用戶的所有內存列表。列表末尾的 memory 是最近的 memory。

```python
memories = in_memory_store.search(namespace_for_memory)

memories[-1].dict()
{'value': {'food_preference': 'I like pizza'},
 'key': '07e0caf4-1631-47b7-b15f-65515d4c1843',
 'namespace': ['1', 'memories'],
 'created_at': '2024-10-02T17:22:31.590602+00:00',
 'updated_at': '2024-10-02T17:22:31.590605+00:00'}
```

每種 memory 類型都是具有特定屬性的 Python 類別（Item）。我們可以透過上述的 `.dict` 方法將其轉換為字典來存取。

它具有以下屬性:

- `value`: The value (itself a dictionary) of this memory
- `key`: A unique key for this memory in this namespace
- `namespace`: A list of strings, the namespace of this memory type
- `created_at`: Timestamp for when this memory was created
- `updated_at`: Timestamp for when this memory was updated

### Semantic Search

除了簡單的檢索功能外，該存儲還支援語義搜索，可讓您根據含義而非精確匹配來查找 memory。若要啟用此功能，請使用 embedding 模型配置儲存:

```python
from langchain.embeddings import init_embeddings

store = InMemoryStore(
    index={
        "embed": init_embeddings("openai:text-embedding-3-small"),  # Embedding provider
        "dims": 1536,                              # Embedding dimensions
        "fields": ["food_preference", "$"]              # Fields to embed
    }
)
```

現在，在搜尋時，您可以使用自然語言查詢來尋找相關記憶：

```python
# Find memories about food preferences
# (This can be done after putting memories into the store)
memories = store.search(
    namespace_for_memory,
    query="What does the user like to eat?",
    limit=3  # Return top 3 matches
)
```

您可以透過配置欄位參數或在儲存 memory 時指定索引參數來控制 memory 的哪些部分已嵌入:

```python
# Store with specific fields to embed
store.put(
    namespace_for_memory,
    str(uuid.uuid4()),
    {
        "food_preference": "I love Italian cuisine",
        "context": "Discussing dinner plans"
    },
    index=["food_preference"]  # Only embed "food_preferences" field
)

# Store without embedding (still retrievable, but not searchable)
store.put(
    namespace_for_memory,
    str(uuid.uuid4()),
    {"system_info": "Last updated: 2024-01-01"},
    index=False
)
```

### Using in LangGraph

一切就緒後，我們在 LangGraph 中使用 `in_memory_store`。 `in_memory_store` 與 checkpointer 協同工作：如上所述，checkpointer 將狀態保存到線程中，而 `in_memory_store` 允許我們儲存任意資訊以供跨 thread 存取。我們以以下方式編譯包含 checkpointer 和 `in_memory_store` 的 graph。

```python
from langgraph.checkpoint.memory import InMemorySaver

# We need this because we want to enable threads (conversations)
checkpointer = InMemorySaver()

# ... Define the graph ...

# Compile the graph with the checkpointer and store
graph = graph.compile(checkpointer=checkpointer, store=in_memory_store)
```

我們像以前一樣使用 `thread_id` 呼叫 graph，同時也使用 `user_id`，我們將使用 `user_id` 將我們的記憶體命名空間分配給這個特定的用戶，如上所示。

```python
# Invoke the graph
user_id = "1"
config = {"configurable": {"thread_id": "1", "user_id": user_id}}

# First let's just say hi to the AI
for update in graph.stream(
    {"messages": [{"role": "user", "content": "hi"}]}, config, stream_mode="updates"
):
    print(update)
```

我們可以透過傳遞 `store: BaseStore` 和 `config: RunnableConfig` 作為 node 參數，存取任何節點中的 `in_memory_store` 和 `user_id`。以下是如何在 node 中使用語義搜尋來尋找相關 memory：

```python
def update_memory(state: MessagesState, config: RunnableConfig, *, store: BaseStore):

    # Get the user id from the config
    user_id = config["configurable"]["user_id"]

    # Namespace the memory
    namespace = (user_id, "memories")

    # ... Analyze conversation and create a new memory

    # Create a new memory ID
    memory_id = str(uuid.uuid4())

    # We create a new memory
    store.put(namespace, memory_id, {"memory": memory})
```

如上所示，我們也可以存取任意節點中的 store，並使用 `store.search` 方法來取得記憶。需要注意的是，記憶是以物件列表的形式傳回的，該列表可以轉換為字典。

```python
memories[-1].dict()
{'value': {'food_preference': 'I like pizza'},
 'key': '07e0caf4-1631-47b7-b15f-65515d4c1843',
 'namespace': ['1', 'memories'],
 'created_at': '2024-10-02T17:22:31.590602+00:00',
 'updated_at': '2024-10-02T17:22:31.590605+00:00'}
```

我們可以存取這些 memory，並在我們的模型呼叫中使用它們。

```python
def call_model(state: MessagesState, config: RunnableConfig, *, store: BaseStore):
    # Get the user id from the config
    user_id = config["configurable"]["user_id"]

    # Namespace the memory
    namespace = (user_id, "memories")

    # Search based on the most recent message
    memories = store.search(
        namespace,
        query=state["messages"][-1].content,
        limit=3
    )
    info = "\n".join([d.value["memory"] for d in memories])

    # ... Use memories in the model call
```

如果我們建立一個新 thread，只要 `user_id` 相同，我們仍然可以存取相同的 memory。

```python
# Invoke the graph
config = {"configurable": {"thread_id": "2", "user_id": "1"}}

# Let's say hi again
for update in graph.stream(
    {"messages": [{"role": "user", "content": "hi, tell me about my memories"}]}, config, stream_mode="updates"
):
    print(update)
```

當我們使用 LangSmith 時，無論是在本地（例如在 Studio 中）還是託管在 LangSmith 伺服器上，預設都可以使用基礎存儲，無需在 graph 編譯期間指定。但是，要啟用語義搜索，您需要在 `langgraph.json` 檔案中配置索引設定。例如：

```python
{
    ...
    "store": {
        "index": {
            "embed": "openai:text-embeddings-3-small",
            "dims": 1536,
            "fields": ["$"]
        }
    }
}
```

請參閱[部署指南](https://docs.langchain.com/langsmith/semantic-search)以了解更多詳情和配置選項。

## Checkpointer libraries

在底層，檢查點機制由符合 [BaseCheckpointSaver](https://reference.langchain.com/python/langgraph/checkpoints/?_gl=1*3qaxeh*_gcl_au*MTA0NTc2NjA2OC4xNzYzNzI5OTg2*_ga*MTM5NTM1ODE0Mi4xNzQ3MDkyMTE0*_ga_47WX3HKKY2*czE3NjQ5ODM4ODMkbzUwJGcxJHQxNzY0OTg5MzgzJGoxJGwwJGgw#langgraph.checkpoint.base.BaseCheckpointSaver) 介面的檢查點物件驅動。 LangGraph 提供了多種檢查點實現，所有這些實現都是透過獨立的、可安裝的程式庫實現的：

- `langgraph-checkpoint`: 檢查點保存器 (`BaseCheckpointSaver`) 和序列化/反序列化介面 (`SerializerProtocol`) 的基礎介面。包含用於實驗的記憶體檢查點實作 (`InMemorySaver`)。 LangGraph 內建了 `langgraph-checkpoint`。
- `langgraph-checkpoint-sqlite`: 這是 LangGraph 檢查點實作的一個版本，它使用 SQLite 資料庫（`SqliteSaver` / `AsyncSqliteSaver`）。非常適合實驗和本地工作流程。需要單獨安裝。
- `langgraph-checkpoint-postgres`: LangSmith 中使用的 Postgres 資料庫（`PostgresSaver` / `AsyncPostgresSaver`）的高階檢查點工具。非常適合生產環境使用。需要單獨安裝。

### Checkpointer interface

每個檢查點都符合 [BaseCheckpointSaver](https://reference.langchain.com/python/langgraph/checkpoints/?_gl=1*aasuie*_gcl_au*MTA0NTc2NjA2OC4xNzYzNzI5OTg2*_ga*MTM5NTM1ODE0Mi4xNzQ3MDkyMTE0*_ga_47WX3HKKY2*czE3NjQ5ODM4ODMkbzUwJGcxJHQxNzY0OTg5MzgzJGoxJGwwJGgw#langgraph.checkpoint.base.BaseCheckpointSaver) 接口，並實作以下方法：
- `.put` - 儲存包含其配置和元資料的檢查點。
- `.put_writes ` - 儲存與檢查點關聯的中間寫入（即待處理的寫入）。
- `.get_tuple` - 根據給定的配置（`thread_id` 和 `checkpoint_id`）取得檢查點元組。這將用於在 `graph.get_state()` 中填充 `StateSnapshot`。
- `.list` - 列出符合給定配置和篩選條件的檢查點。這用於填入 `graph.get_state_history()` 中的狀態歷史記錄。

如果 checkpointer 與 asynchronous graph 執行一起使用（即透過 `.ainvoke`, `.astream`, `.abatch` 執行 graph），則會使用上述方法的非同步版本（`.aput`, `.aput_writes`, `.aget_tuple`, `.alist`）。

!!! info
    要非同步運行您的 graph，您可以使用 `InMemorySaver`，或 `Sqlite/Postgres checkpointers 的非同步版本- [AsyncSqliteSaver](https://reference.langchain.com/python/langgraph/checkpoints/?_gl=1*iokj8*_gcl_au*MTA0NTc2NjA2OC4xNzYzNzI5OTg2*_ga*MTM5NTM1ODE0Mi4xNzQ3MDkyMTE0*_ga_47WX3HKKY2*czE3NjQ5ODM4ODMkbzUwJGcxJHQxNzY0OTg5MzgzJGoxJGwwJGgw#langgraph.checkpoint.sqlite.aio.AsyncSqliteSaver) / [AsyncPostgresSaver](https://reference.langchain.com/python/langgraph/checkpoints/?_gl=1*iokj8*_gcl_au*MTA0NTc2NjA2OC4xNzYzNzI5OTg2*_ga*MTM5NTM1ODE0Mi4xNzQ3MDkyMTE0*_ga_47WX3HKKY2*czE3NjQ5ODM4ODMkbzUwJGcxJHQxNzY0OTg5MzgzJGoxJGwwJGgw#langgraph.checkpoint.postgres.aio.AsyncPostgresSaver) 檢查點。


### Serializer

當 checkpointers 保存 graph 狀態時，需要序列化狀態中的通道值。這是透過序列化器物件實現的。

`langgraph_checkpoint` 定義了用於實現序列化的協議，並提供了一個預設實作（`JsonPlusSerializer`），該實作可以處理各種類型，包括 LangChain 和 LangGraph 原語、日期時間、枚舉等等。

#### Serialization with `pickle`

預設序列化器 `JsonPlusSerializer` 底層使用 ormsgpack 和 JSON，但這並不適用於所有類型的物件。

如果您希望對目前 msgpack 編碼器不支援的物件（例如 Pandas dataframe）回退到 pickle，可以使用 `JsonPlusSerializer` 的 pickle_fallback 參數:

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.checkpoint.serde.jsonplus import JsonPlusSerializer

# ... Define the graph ...
graph.compile(
    checkpointer=InMemorySaver(serde=JsonPlusSerializer(pickle_fallback=True))
)
```

#### Encryption

檢查點可以選擇性地加密所有持久化狀態。若要啟用此功能，請將 `EncryptedSerializer` 實例傳遞給任何 `BaseCheckpointSaver` 實作的 serde 參數。建立加密序列化器最簡單的方法是透過 `from_pycryptodome_aes`，它會從 LANGGRAPH_AES_KEY 環境變數中讀取 AES 金鑰（或接受一個金鑰參數）:

```python
import sqlite3

from langgraph.checkpoint.serde.encrypted import EncryptedSerializer
from langgraph.checkpoint.sqlite import SqliteSaver

serde = EncryptedSerializer.from_pycryptodome_aes()  # reads LANGGRAPH_AES_KEY
checkpointer = SqliteSaver(sqlite3.connect("checkpoint.db"), serde=serde)
```

```python
from langgraph.checkpoint.serde.encrypted import EncryptedSerializer
from langgraph.checkpoint.postgres import PostgresSaver

serde = EncryptedSerializer.from_pycryptodome_aes()
checkpointer = PostgresSaver.from_conn_string("postgresql://...", serde=serde)
checkpointer.setup()
```

在 LangSmith 上運行時，只要存在 LANGGRAPH_AES_KEY，加密就會自動啟用，因此您只需提供該環境變數即可。若要使用其他加密方案，請實作 `CipherProtocol` 並提供給 `EncryptedSerializer`。

## Capabilities

### Human-in-the-loop

首先，檢查點透過允許人檢查、中斷和批准 graph 步驟，簡化了 human-in-the-loop workflows。這些工作流程需要檢查點，因為人們必須能夠隨時查看 graph 的狀態，並且 graph 必須在人更新狀態後恢復執行。有關範例，請參閱操作指南。

### Memory

其次，檢查點允許互動之間保留 "memory"。對於重複的人際互動（例如對話），任何後續訊息都可以傳送到同一對話線程，該線程會保留對先前訊息的記憶。有關如何使用檢查點添加和管理對話記憶的信息，請參閱 [Add memory](https://docs.langchain.com/oss/python/langgraph/add-memory)。

### Time Travel

第三，檢查點允許 "time travel"，使用戶能夠重播先前的 graph 執行過程，以查看和/或調試特定的 graph 步驟。此外，檢查點還允許在任意檢查點對 graph 狀態進行分支，以探索其他可能的路徑。

### Fault-tolerance

最後，檢查點機制也提供了容錯和錯誤復原功能：如果一個或多個節點在某個超級步驟中發生故障，您可以從上一個成功步驟重新啟動圖表。此外，當圖節點在某個超級步驟執行過程中發生故障時，LangGraph 會儲存來自該超級步驟中其他成功完成的節點的待處理檢查點寫入，這樣，每當我們從該超級步驟恢復圖執行時，就不會重新執行那些已經成功完成的節點。

### Pending writes

此外，當 graph 節點在給定的超級步驟執行過程中失敗時，LangGraph 會儲存來自在該超級步驟成功完成的任何其他節點的待處理檢查點寫入，以便無論何時我們從該超級步驟恢復 graph 執行，我們都不會重新運行成功的節點。

