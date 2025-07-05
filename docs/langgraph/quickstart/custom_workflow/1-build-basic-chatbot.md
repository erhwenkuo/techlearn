# Build a basic chatbot

在本教程中，您將建立一個基本的聊天機器人。這個聊天機器人是後續系列教學的基礎，您將逐步添加更複雜的功能，並在過程中學習 LangGraph 的關鍵概念。讓我們開始吧！ 🌟

## Prerequisites

在開始本教學之前，請確保您可以存取支援工具呼叫功能的 LLM，例如 OpenAI, Anthropic 或 Google Gemini。

## 1. Install packages

安裝所需的軟體包：

```bash
pip install -U langgraph langchain-openai langsmith 
```

## 2. Create a `StateGraph`

現在，您可以使用 LangGraph 建立一個基本的聊天機器人。該聊天機器人將直接回覆用戶訊息。

首先建立一個 `StateGraph`。 `StateGraph` 物件將聊天機器人的結構定義為 "state machine"。我們將新增 `nodes` 來表示 llm 和聊天機器人可以呼叫的函數，以及 `edges` 來指定機器人如何在這些函數之間轉換。

```python
from typing import Annotated

from typing_extensions import TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages

# 定義一個用來記錄并保存 Agent 在 node 中共同分享的 State 物件
class State(TypedDict):
    # Messages have the type "list". The `add_messages` function
    # in the annotation defines how this state key should be updated
    # (in this case, it appends messages to the list, rather than overwriting them)
    messages: Annotated[list, add_messages]


graph_builder = StateGraph(State)
```

我們的 graph 現在可以處理兩個 key tasks：

- 每個 `node` 可以接收目前 `State` 作為輸入，並輸出更新的 `State`。
- 由於預先建置的 `add_messages` 函數與 `Annotated` 語法配合使用，訊息更新將附加到現有清單(`messages`)中，而不是覆寫它。

!!! tip
    Python 中的 TypedDict 是一種類型提示，用於定義具有固定字串鍵和其對應值的特定類型的字典。它為字典帶來了一定程度的結構化類型，允許像 MyPy 這樣的靜態類型檢查器在開發過程中驗證字典資料的結構和類型。

    例如:

    ```python
    from typing import TypedDict

    class UserProfile(TypedDict):
        name: str
        age: int
        email: str
    ```

    預設來說, 所有被定義在 TypedDict 的 properties 都是 "Must", 如果要設定 "Optional":

    ```python
    from typing import TypedDict, NotRequired

    class UserProfileOptional(TypedDict):
        name: str
        age: int
        email: NotRequired[str] # email is optional

    class UserProfileTotalFalse(TypedDict, total=False):
        name: str # name is optional
        age: int # age is optional
        email: str # email is optional
    ```

!!! tip
    Annotated 是 Python 3.9 中引入的一種特殊類型提示，允許將任意元資料附加到類型註解。

    Annotated 則允許添加可供其他庫使用或用於運行時驗證的額外詳細信息。

    工作原理：
        - Annotated 至少接受兩個參數：
            - 實際類型：這是類型檢查器將識別的基底類型。
            - 元資料：可以是任意數量的表示元資料的附加參數。

    ```python
    from typing import Annotated

    # Define a type hint with metadata for validation
    Age = Annotated[int, "value should be between 0 and 120"]

    def process_age(age: Age):
        # In a real application, a library like Pydantic or a custom validator
        # would use the metadata to enforce the constraint.
        print(f"Processing age: {age}")

    process_age(30)
    # process_age(150) # This would likely raise a validation error if a validator is in place
    ```

    靜態類型檢查器會忽略這些參數，但庫或自訂程式碼可以在運行時存取它們。


## 3. Add a node

接下來，新增一個 `chatbot` 節點。**Nodes` 代表工作單元，通常是常規的 Python 函數。

```bash
pip install -U "langchain[openai]"
```

我們先選擇一個 chat model:

```python
import os
from langchain.chat_models import init_chat_model

os.environ["OPENAI_API_KEY"] = "sk-..."

llm = init_chat_model("openai:gpt-4.1-mini")
```

我們現在可以將 chat model 合併到一個簡單的 node：

```python
# 新增一個 `chatbot` 節點
def chatbot(state: State):
    return {"messages": [llm.invoke(state["messages"])]}


# 第一個參數是唯一的節點名稱
# 第二個參數是每當節點被使用時將被呼叫的函數或物件。
graph_builder.add_node("chatbot", chatbot)
```

!!! info
    請注意，`chatbot` 節點函數如何將當前狀態作為輸入，並傳回 `dict`，其中包含鍵 **messages** 下的更新訊息清單。這是所有 LangGraph 節點函數的基本模式。

    我們狀態中的 `add_messages` 函數會將 LLM 的回應訊息附加到狀態中已有的任何訊息之後。

## 4. Add an `entry` point

增加一個 `entry` 來告訴 graph 物件每次運行時從那個 `node` 開始工作：

```python
graph_builder.add_edge(START, "chatbot")
```

## 5. Add an `exit` point

新增 `exit` 節點，指示 graph 物件應在何處結束執行。這對於更複雜的流程很有幫助，但即使在像這樣的簡單 graph 中，添加結束節點也能提高清晰度。

```python
graph_builder.add_edge("chatbot", END)
```

這告訴 graph 在運行 chatbot節點後就轉換到 `END`。


## 6. Compile the graph

在運行 graph 之前，我們需要對其進行編譯。我們可以透過在 graph 建構器上呼叫 `compile()` 來實現。這將建立一個 `CompiledGraph`，我們可以在狀態上呼叫它。

```python
graph = graph_builder.compile()
```

## 7. Visualize the graph (optional)

您可以使用 `get_graph` 方法和某個 `draw` 方法（例如 `draw_ascii` 或 `draw_png`）來視覺化圖表。每個 `draw` 方法都需要額外的依賴項。

```python
from IPython.display import Image, display

try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    # This requires some extra dependencies and is optional
    pass
```

## 8. Run the chatbot

現在來運行聊天機器人！

!!! tip
    您可以隨時輸入 `quit`, `exit` 或 `q` 退出聊天循環。

```python
def stream_graph_updates(user_input: str):
    # 執行 graph 並傳入使用者的 input
    for event in graph.stream({"messages": [{"role": "user", "content": user_input}]}):
        # 串流地取回回應
        for value in event.values():
            # 打印最後的 message
            print("Assistant:", value["messages"][-1].content)

# 構建 loop 來持績與 chatbot 進行對話
while True:
    try:
        user_input = input("User: ")
        if user_input.lower() in ["quit", "exit", "q"]:
            print("Goodbye!")
            break
        stream_graph_updates(user_input)
    except:
        # fallback if input() is not available
        user_input = "What do you know about LangGraph?"
        print("User: " + user_input)
        stream_graph_updates(user_input)
        break
```

恭喜！您已經使用 LangGraph 建立了第一個聊天機器人。該機器人可以透過接收用戶輸入並使用 LLM 生成回應來進行基本對話。您可以檢查上述調用的 LangSmith Trace。

以下是本教學的完整程式碼：

```python
from typing import Annotated

from langchain.chat_models import init_chat_model
from typing_extensions import TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages


class State(TypedDict):
    messages: Annotated[list, add_messages]


graph_builder = StateGraph(State)


llm = init_chat_model("openai:gpt-4.1-mini")


def chatbot(state: State):
    return {"messages": [llm.invoke(state["messages"])]}


# The first argument is the unique node name
# The second argument is the function or object that will be called whenever
# the node is used.
graph_builder.add_node("chatbot", chatbot)
graph_builder.add_edge(START, "chatbot")
graph_builder.add_edge("chatbot", END)
graph = graph_builder.compile()
```

## Next steps

你可能已經注意到，chatbot 的知識僅限於其訓練資料。在下一部分中，我們將添加一個網頁搜尋工具，以擴展 chatbot 的知識，使其更加強大。

