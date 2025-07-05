# Add memory

聊天機器人現在可以使用 `tool` 來回答使用者的問題，但它無法記住先前互動的上下文。這限制了它進行連貫、多輪對話的能力。

LangGraph 透過 **持久化檢查點 (persistent checkpointing)** 解決了這個問題。如果您在編譯 graph 時提供檢查點，並在呼叫 graph 時提供 `thread_id`，LangGraph 會在每一步之後自動儲存狀態。當您使用相同的 `thread_id` 再次呼叫 graph 時，graph 會載入其已儲存的狀態，從而允許聊天機器人從上次中斷的地方繼續執行。

我們稍後會看到，檢查點比簡單的聊天記憶功能強大得多——它允許您隨時保存和恢復複雜的狀態，以進行錯誤恢復、人機互動、時間旅行互動等等。但首先，讓我們加入檢查點來實現多輪對話。

## 1. Create a MemorySaver checkpointer

建立 `MemorySaver` 檢查點：

```python
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()
```

這是 in-memory checkpointer，這對於本教程來說很方便。但是，在生產應用程式中，您可能會將其變更為使用 `SqliteSaver` 或 `PostgresSaver` 並連接資料庫。

## 2. Compile the graph

使用提供的檢查點編譯 graph 物件，當 graph 通過每個 node 時，它將檢查狀態：

```python
graph = graph_builder.compile(checkpointer=memory)
```

```python
from IPython.display import Image, display

try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    # This requires some extra dependencies and is optional
    pass
```

## 3. Interact with your chatbot

現在您可以與您的機器人互動！

1. 選擇一個 `thread` 作為本次對話的 key。

    ```python
    config = {"configurable": {"thread_id": "1"}}
    ```

2. 呼叫您的聊天機器人：
   
    ```python
    user_input = "Hi there! My name is Erhwen."

    # The config is the **second positional argument** to stream() or invoke()!
    events = graph.stream(
        {"messages": [{"role": "user", "content": user_input}]},
        config,
        stream_mode="values",
    )

    for event in events:
        event["messages"][-1].pretty_print()
    ```

```bash
================================ Human Message =================================

Hi there! My name is Erhwen.
================================== Ai Message ==================================

Hello Erhwen! It's nice to meet you. How can I assist you today? Is there anything specific you'd like to know or discuss?
```

!!! info
    呼叫我們的 graph 時，配置(config)會作為第二個位置參數提供。重要的是，它沒有嵌套在 graph 的輸入中 `({'messages': []})`。

## 4. Ask a follow up question

提出後續問題：

```python
user_input = "Remember my name?"

# The config is the **second positional argument** to stream() or invoke()!
events = graph.stream(
    {"messages": [{"role": "user", "content": user_input}]},
    config,
    stream_mode="values",
)

for event in events:
    event["messages"][-1].pretty_print()
```

```bash
================================ Human Message =================================

Remember my name?
================================== Ai Message ==================================

Of course, I remember your name, Erhwen. I always try to pay attention to important details that users share with me. Is there anything else you'd like to talk about or any questions you have? I'm here to help with a wide range of topics or tasks.
```

請注意，我們沒有使用外部列表來管理記憶體：所有操作都由檢查點(checkpointer)處理！您可以查看 LangSmith 追蹤中的完整執行過程，以了解具體發生了什麼。

讓我們再試試其他配置。

```python
# 唯一的區別是我們將這裡的 `thread_id` 改為 “2” 而不是 “1”
events = graph.stream(
    {"messages": [{"role": "user", "content": user_input}]},
    {"configurable": {"thread_id": "2"}},
    stream_mode="values",
)

for event in events:
    event["messages"][-1].pretty_print()
```

```bash
================================ Human Message =================================

Remember my name?
================================== Ai Message ==================================

I apologize, but I don't have any previous context or memory of your name. As an AI assistant, I don't retain information from past conversations. Each interaction starts fresh. Could you please tell me your name so I can address you properly in this conversation?
```

!!! info
    請注意，我們所做的唯一更改是修改了配置中的 `thread_id`。請參閱此調用的 LangSmith 跟踪以進行比較。

## 5. Inspect the state

到目前為止，我們已經在兩個不同的執行緒上創建了一些檢查點(checkpoints)。但是檢查點包含哪些內容呢？若要隨時檢查給定配置的 graph 狀態，請呼叫 `get_state(config)`。

```python
snapshot = graph.get_state(config)

print(snapshot)
```

```bash
StateSnapshot(values={'messages': [HumanMessage(content='Hi there! My name is Erhwen.', additional_kwargs={}, response_metadata={}, id='8c1ca919-c553-4ebf-95d4-b59a2d61e078'), AIMessage(content="Hello Erhwen! It's nice to meet you. How can I assist you today? Is there anything specific you'd like to know or discuss?", additional_kwargs={}, response_metadata={'id': 'msg_01WTQebPhNwmMrmmWojJ9KXJ', 'model': 'claude-3-5-sonnet-20240620', 'stop_reason': 'end_turn', 'stop_sequence': None, 'usage': {'input_tokens': 405, 'output_tokens': 32}}, id='run-58587b77-8c82-41e6-8a90-d62c444a261d-0', usage_metadata={'input_tokens': 405, 'output_tokens': 32, 'total_tokens': 437}), HumanMessage(content='Remember my name?', additional_kwargs={}, response_metadata={}, id='daba7df6-ad75-4d6b-8057-745881cea1ca'), AIMessage(content="Of course, I remember your name, Erhwen. I always try to pay attention to important details that users share with me. Is there anything else you'd like to talk about or any questions you have? I'm here to help with a wide range of topics or tasks.", additional_kwargs={}, response_metadata={'id': 'msg_01E41KitY74HpENRgXx94vag', 'model': 'claude-3-5-sonnet-20240620', 'stop_reason': 'end_turn', 'stop_sequence': None, 'usage': {'input_tokens': 444, 'output_tokens': 58}}, id='run-ffeaae5c-4d2d-4ddb-bd59-5d5cbf2a5af8-0', usage_metadata={'input_tokens': 444, 'output_tokens': 58, 'total_tokens': 502})]}, next=(), config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef7d06e-93e0-6acc-8004-f2ac846575d2'}}, metadata={'source': 'loop', 'writes': {'chatbot': {'messages': [AIMessage(content="Of course, I remember your name, Erhwen. I always try to pay attention to important details that users share with me. Is there anything else you'd like to talk about or any questions you have? I'm here to help with a wide range of topics or tasks.", additional_kwargs={}, response_metadata={'id': 'msg_01E41KitY74HpENRgXx94vag', 'model': 'claude-3-5-sonnet-20240620', 'stop_reason': 'end_turn', 'stop_sequence': None, 'usage': {'input_tokens': 444, 'output_tokens': 58}}, id='run-ffeaae5c-4d2d-4ddb-bd59-5d5cbf2a5af8-0', usage_metadata={'input_tokens': 444, 'output_tokens': 58, 'total_tokens': 502})]}}, 'step': 4, 'parents': {}}, created_at='2024-09-27T19:30:10.820758+00:00', parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef7d06e-859f-6206-8003-e1bd3c264b8f'}}, tasks=())
```

```python
snapshot.next  # (since the graph ended this turn, `next` is empty. If you fetch a state from within a graph invocation, next tells which node will execute next)
```

上面的快照包含目前狀態值、對應的配置以及下一個要處理的節點。在我們的例子中，graph 已達到 END 狀態，因此下一個為空。

恭喜！借助 LangGraph 的檢查點(checkpointer)系統，您的聊天機器人現在可以跨會話保持對話狀態。這為更自然、更符合情境的互動開啟了令人興奮的可能性。 LangGraph 的檢查點甚至可以處理任意複雜的 graph 狀態，這比簡單的聊天記憶更具表現力和功能強大得多。

請參閱下面的程式碼片段以查看本教學中的 graph 定義：

```bash
pip install -U "langchain[openai]"
```

```python
import os
from langchain.chat_models import init_chat_model

os.environ["OPENAI_API_KEY"] = "sk-..."

llm = init_chat_model("openai:gpt-4.1")
```

```python
from typing import Annotated

from langchain.chat_models import init_chat_model
from langchain_tavily import TavilySearch
from langchain_core.messages import BaseMessage
from typing_extensions import TypedDict

from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import StateGraph
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition

class State(TypedDict):
    messages: Annotated[list, add_messages]

graph_builder = StateGraph(State)

tool = TavilySearch(max_results=2)

tools = [tool]

llm_with_tools = llm.bind_tools(tools)

def chatbot(state: State):
    return {"messages": [llm_with_tools.invoke(state["messages"])]}

graph_builder.add_node("chatbot", chatbot)

tool_node = ToolNode(tools=[tool])

graph_builder.add_node("tools", tool_node)

graph_builder.add_conditional_edges(
    "chatbot",
    tools_condition,
)

graph_builder.add_edge("tools", "chatbot")

graph_builder.set_entry_point("chatbot")

memory = MemorySaver()

graph = graph_builder.compile(checkpointer=memory)
```

## Next steps

在下一個教學中，您將向聊天機器人添加人機互動 (human-in-the-loop) 功能，以處理在繼續之前可能需要指導或驗證的情況。