# Add human-in-the-loop controls

Agents 可能不可靠，可能需要 human input 才能成功完成任務。同樣，對於某些操作，您可能需要在運行前獲得人工批准，以確保一切按預期運行。

LangGraph 的持久層支援 `human-in-the-loop` 工作流程，允許根據使用者回饋暫停和恢復執行。此功能的主要介面是 `interrupt` 函數。在節點內部呼叫 `interrupt` 將暫停執行。可以透過傳入 [Command](https://langchain-ai.github.io/langgraph/concepts/low_level/#command) 來恢復執行，並接收新的手動輸入。 `interrupt` 在人體工程學上類似於 Python 的內建 `input()`，但也有一些注意事項。

## 1. Add the `human_assistance` tool

從 [為聊天機器人添加記憶體](./3-add-memory.md) 教學中的現有程式碼開始，將 `human_assistance` 工具新增至聊天機器人。此工具使用中斷來接收來自人類的信息。

我們先構建一個聊天模型：

```bash
pip install -U "langchain[openai]"
```

```python
import os
from langchain.chat_models import init_chat_model

os.environ["OPENAI_API_KEY"] = "sk-..."

llm = init_chat_model("openai:gpt-4.1-mini")
```

我們現在可以使用附加工具將其合併到我們的 `StateGraph` 中：

```python hl_lines="12 21-28"
from typing import Annotated

from langchain_tavily import TavilySearch
from langchain_core.tools import tool
from typing_extensions import TypedDict

from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition

from langgraph.types import Command, interrupt

# 定義一個用來記錄并保存 Agent 在 node 中共同分享的 State 物件
class State(TypedDict):
    messages: Annotated[list, add_messages]

# 構建一個 graph builder 物件
graph_builder = StateGraph(State)

# 定義一個 tool (human-in-the-loop)
@tool
def human_assistance(query: str) -> str:
    """Request assistance from a human."""

    # 向人類請求幫助。
    human_response = interrupt({"query": query})
    return human_response["data"]

# 定義一個 tool (internet searching)
tool = TavilySearch(max_results=2)

# 將 tools 與 llm 建行綁定
tools = [tool, human_assistance]
llm_with_tools = llm.bind_tools(tools)

# 構建 chatbot node
def chatbot(state: State):
    message = llm_with_tools.invoke(state["messages"])
    # Because we will be interrupting during tool execution,
    # we disable parallel tool calling to avoid repeating any
    # tool invocations when we resume.

    # 由於我們將在工具執行期間中斷，
    # 我們禁用 parallel tool calling ，以避免在恢復時重複任何工具呼叫。
    assert len(message.tool_calls) <= 1
    return {"messages": [message]}

graph_builder.add_node("chatbot", chatbot)

tool_node = ToolNode(tools=tools)

graph_builder.add_node("tools", tool_node)

graph_builder.add_conditional_edges(
    "chatbot",
    tools_condition,
)

graph_builder.add_edge("tools", "chatbot")

graph_builder.add_edge(START, "chatbot")
```

!!! tip
    有關人機互動工作流程的更多資訊和範例，請參閱[Human-in-the-loop](https://langchain-ai.github.io/langgraph/concepts/human_in_the_loop/)。

## 2. Compile the graph

我們使用 checkpointer 來編譯 graph，就像之前一樣：

```python
memory = MemorySaver()

graph = graph_builder.compile(checkpointer=memory)
```

## 3. Visualize the graph (optional)

透過視覺化圖表，您可以獲得與以前相同的佈局 (因只有添加額外的工具)!

```python
from IPython.display import Image, display

try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    # This requires some extra dependencies and is optional
    pass
```

![](https://langchain-ai.github.io/langgraph/tutorials/get-started/chatbot-with-tools.png)

## 4. Prompt the chatbot

現在，向聊天機器人提出一個問題，該問題將使用新的 `human_assistance` 工具：

```python
user_input = "I need some expert guidance for building an AI agent. Could you request assistance for me?"
config = {"configurable": {"thread_id": "1"}}

events = graph.stream(
    {"messages": [{"role": "user", "content": user_input}]},
    config,
    stream_mode="values",
)

for event in events:
    if "messages" in event:
        event["messages"][-1].pretty_print()
```

```bash
================================ Human Message =================================

I need some expert guidance for building an AI agent. Could you request assistance for me?
================================== Ai Message ==================================

[{'text': "Certainly! I'd be happy to request expert assistance for you regarding building an AI agent. To do this, I'll use the human_assistance function to relay your request. Let me do that for you now.", 'type': 'text'}, {'id': 'toolu_01ABUqneqnuHNuo1vhfDFQCW', 'input': {'query': 'A user is requesting expert guidance for building an AI agent. Could you please provide some expert advice or resources on this topic?'}, 'name': 'human_assistance', 'type': 'tool_use'}]
Tool Calls:
  human_assistance (toolu_01ABUqneqnuHNuo1vhfDFQCW)
 Call ID: toolu_01ABUqneqnuHNuo1vhfDFQCW
  Args:
    query: A user is requesting expert guidance for building an AI agent. Could you please provide some expert advice or resources on this topic?
```

聊天機器人產生了一個 `tool call`，但隨後執行被中斷。如果您檢查 graph 狀態，您會發現它在工具節點處停止了：

```python
snapshot = graph.get_state(config)

print(snapshot.next)
```

```bash
('tools',)
```

!!! info
    仔細看看 `human_assistance` 工具：

    ```python
    @tool
    def human_assistance(query: str) -> str:
        """Request assistance from a human."""
        human_response = interrupt({"query": query})
        return human_response["data"]
    ```

    與 Python 內建的 `input()` 函數類似，在工具內部呼叫 `interrupt` 會暫停執行。進度會根據檢查點進行持久化；因此，如果使用 Postgres 進行持久化，只要資料庫仍處於活動狀態，就可以隨時復原。在本例中，它使用記憶體中的檢查點進行持久化，只要 Python 核心正在運行，就可以隨時恢復。

## 5. Resume execution

若要復原執行，請傳遞一個包含工具所需資料的 `Command` 物件。此資料的格式可根據需要自訂。在本例中，使用一個帶有鍵 `data` 的字典：

```python
human_response = (
    "We, the experts are here to help! We'd recommend you check out LangGraph to build your agent."
    " It's much more reliable and extensible than simple autonomous agents."
)

# 傳遞一個包含工具所需資料的 `Command` 物件
human_command = Command(resume={"data": human_response})

events = graph.stream(human_command, config, stream_mode="values")

for event in events:
    if "messages" in event:
        event["messages"][-1].pretty_print()
```

```bash
================================== Ai Message ==================================

[{'text': "Certainly! I'd be happy to request expert assistance for you regarding building an AI agent. To do this, I'll use the human_assistance function to relay your request. Let me do that for you now.", 'type': 'text'}, {'id': 'toolu_01ABUqneqnuHNuo1vhfDFQCW', 'input': {'query': 'A user is requesting expert guidance for building an AI agent. Could you please provide some expert advice or resources on this topic?'}, 'name': 'human_assistance', 'type': 'tool_use'}]
Tool Calls:
  human_assistance (toolu_01ABUqneqnuHNuo1vhfDFQCW)
 Call ID: toolu_01ABUqneqnuHNuo1vhfDFQCW
  Args:
    query: A user is requesting expert guidance for building an AI agent. Could you please provide some expert advice or resources on this topic?
================================= Tool Message =================================
Name: human_assistance

We, the experts are here to help! We'd recommend you check out LangGraph to build your agent. It's much more reliable and extensible than simple autonomous agents.
================================== Ai Message ==================================

Thank you for your patience. I've received some expert advice regarding your request for guidance on building an AI agent. Here's what the experts have suggested:

The experts recommend that you look into LangGraph for building your AI agent. They mention that LangGraph is a more reliable and extensible option compared to simple autonomous agents.

LangGraph is likely a framework or library designed specifically for creating AI agents with advanced capabilities. Here are a few points to consider based on this recommendation:

1. Reliability: The experts emphasize that LangGraph is more reliable than simpler autonomous agent approaches. This could mean it has better stability, error handling, or consistent performance.

2. Extensibility: LangGraph is described as more extensible, which suggests that it probably offers a flexible architecture that allows you to easily add new features or modify existing ones as your agent's requirements evolve.

3. Advanced capabilities: Given that it's recommended over "simple autonomous agents," LangGraph likely provides more sophisticated tools and techniques for building complex AI agents.
...
2. Look for tutorials or guides specifically focused on building AI agents with LangGraph.
3. Check if there are any community forums or discussion groups where you can ask questions and get support from other developers using LangGraph.

If you'd like more specific information about LangGraph or have any questions about this recommendation, please feel free to ask, and I can request further assistance from the experts.
Output is truncated. View as a scrollable element or open in a text editor. Adjust cell output settings...
```

輸入已接收並處理為工具訊息。查看此調用的 LangSmith 跟踪，以了解上述調用中完成的具體工作。請注意，狀態已在第一步加載，以便我們的聊天機器人可以從中斷處繼續執行。

恭喜！您已使用中斷為聊天機器人添加了人機互動執行功能，以便在需要時進行人工監督和介入。這開啟了您使用 AI 系統建立潛在 UI 的大門。由於您已新增檢查點，只要底層持久層正在運行，graph 就可以無限期暫停，並隨時恢復，如同一切正常。

請參閱下面的程式碼片段以查看本教學中的 graph:

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

from langchain_tavily import TavilySearch
from langchain_core.tools import tool
from typing_extensions import TypedDict

from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition
from langgraph.types import Command, interrupt

class State(TypedDict):
    messages: Annotated[list, add_messages]

graph_builder = StateGraph(State)

@tool
def human_assistance(query: str) -> str:
    """Request assistance from a human."""
    human_response = interrupt({"query": query})
    return human_response["data"]

tool = TavilySearch(max_results=2)
tools = [tool, human_assistance]
llm_with_tools = llm.bind_tools(tools)

def chatbot(state: State):
    message = llm_with_tools.invoke(state["messages"])
    assert(len(message.tool_calls) <= 1)
    return {"messages": [message]}

graph_builder.add_node("chatbot", chatbot)

tool_node = ToolNode(tools=tools)

graph_builder.add_node("tools", tool_node)

graph_builder.add_conditional_edges(
    "chatbot",
    tools_condition,
)

graph_builder.add_edge("tools", "chatbot")
graph_builder.add_edge(START, "chatbot")

memory = MemorySaver()
graph = graph_builder.compile(checkpointer=memory)
```

## Next steps¶

到目前為止，本教學的範例都依賴一個包含一個條目的簡單狀態： `messages`。您可以使用這個簡單狀態進行更深入的開發，但如果您想定義更複雜的行為而不依賴訊息列表，則可以向狀態添加其他欄位。

