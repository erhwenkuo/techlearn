# Add tools

為了處理聊天機器人無法 "憑記憶" 回答的問題，請整合一個網頁搜尋工具。聊天機器人可以使用此工具找到相關訊息，並提供更準確的回應。

## Prerequisites

在開始本教學之前，請確保您已具備以下條件：

- [Tavily Search Engine](https://python.langchain.com/docs/integrations/tools/tavily_search/?_gl=1*10vh02w*_gcl_au*MTMzOTIyOTUzNS4xNzUwNTUxODU5*_ga*MTM5NTM1ODE0Mi4xNzQ3MDkyMTE0*_ga_47WX3HKKY2*czE3NTEwODQ2MzYkbzExJGcwJHQxNzUxMDg0NjM2JGo2MCRsMCRoMA..)的 API 金鑰。

## 1. Install the search engine

安裝使用 Tavily 搜尋引擎的必備條件：

```bash
pip install -U langchain-tavily
```

## 2. Configure your environment

使用您的搜尋引擎 API 金鑰配置您的環境：

```bash
_set_env("TAVILY_API_KEY")
```

## 3. Define the tool

定義網路搜尋工具：

```python
from langchain_tavily import TavilySearch

tool = TavilySearch(max_results=2)
tools = [tool]
tool.invoke("What's a 'node' in LangGraph?")
```

結果就是我們的聊天機器人可以用來回答問題的頁面摘要：

```bash
{
    'query': "What's a 'node' in LangGraph?",
    'follow_up_questions': None,
    'answer': None,
    'images': [],
    'results': [
        {
            'title': "Introduction to LangGraph: A Beginner's Guide - Medium",
            'url': 'https://medium.com/@cplog/introduction-to-langgraph-a-beginners-guide-14f9be027141',
            'content': 'Stateful Graph: LangGraph revolves around the concept of a stateful graph, where each node in the graph represents a step in your computation, and the graph maintains a state that is passed around and updated as the computation progresses. LangGraph supports conditional edges, allowing you to dynamically determine the next node to execute based on the current state of the graph. We define nodes for classifying the input, handling greetings, and handling search queries. def classify_input_node(state): LangGraph is a versatile tool for building complex, stateful applications with LLMs. By understanding its core concepts and working through simple examples, beginners can start to leverage its power for their projects. Remember to pay attention to state management, conditional edges, and ensuring there are no dead-end nodes in your graph.',
            'score': 0.7065353,
            'raw_content': None
        },
        {
            'title': 'LangGraph Tutorial: What Is LangGraph and How to Use It?',
            'url': 'https://www.datacamp.com/tutorial/langgraph-tutorial',
            'content': 'LangGraph is a library within the LangChain ecosystem that provides a framework for defining, coordinating, and executing multiple LLM agents (or chains) in a structured and efficient manner. By managing the flow of data and the sequence of operations, LangGraph allows developers to focus on the high-level logic of their applications rather than the intricacies of agent coordination. Whether you need a chatbot that can handle various types of user requests or a multi-agent system that performs complex tasks, LangGraph provides the tools to build exactly what you need. LangGraph significantly simplifies the development of complex LLM applications by providing a structured framework for managing state and coordinating agent interactions.',
            'score': 0.5008063,
            'raw_content': None
        }
    ],
    'response_time': 1.38
}
```

## 4. Define the graph

對於您在第一個教學課程中建立的 `StateGraph`，請在 LLM 上新增 `bind_tools`。這可以讓 LLM 知道如果要使用搜尋引擎應該使用正確的 JSON 格式。

我們首先選擇我們的 LLM：

```bash
pip install -U "langchain[openai]"
```

```python
import os
from langchain.chat_models import init_chat_model

os.environ["OPENAI_API_KEY"] = "sk-..."

llm = init_chat_model("openai:gpt-4.1")
```

我們現在可以將其合併到 `StateGraph` 中：

```python
from typing import Annotated

from typing_extensions import TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]

graph_builder = StateGraph(State)

# 告訴 LLM 可以呼叫哪些工具
llm_with_tools = llm.bind_tools(tools)

def chatbot(state: State):
    return {"messages": [llm_with_tools.invoke(state["messages"])]}

graph_builder.add_node("chatbot", chatbot)
```

## 5. Create a function to run the tools

現在，建立一個函數，用於在工具被呼叫時運行它們。方法是將工具新增至名為 `BasicToolNode` 的新節點，該節點檢查狀態中的最新訊息，並在訊息包含 `tool_calls` 時呼叫工具。它依賴 LLM 的 `tool_calling` 支持，該支持在 Anthropic, OpenAI, Google Gemini 和其他一些 LLM 提供者中均可用。

```python
import json

from langchain_core.messages import ToolMessage


class BasicToolNode:
    """A node that runs the tools requested in the last AIMessage."""
    """一個用來執行最後一個 AIMessage 裡請求 tool request 的 node"""

    def __init__(self, tools: list) -> None:
        # 從所有的 tool 構建一個 tool_name -> tool 的 dictionary
        self.tools_by_name = {tool.name: tool for tool in tools}

    def __call__(self, inputs: dict):
        # 取回 messages 的列表
        if messages := inputs.get("messages", []):
            # 取回最後一個 message 物件
            message = messages[-1]
        else:
            raise ValueError("No message found in input")
        outputs = []
        for tool_call in message.tool_calls:
            tool_result = self.tools_by_name[tool_call["name"]].invoke(
                tool_call["args"]
            )
            outputs.append(
                ToolMessage(
                    content=json.dumps(tool_result),
                    name=tool_call["name"],
                    tool_call_id=tool_call["id"],
                )
            )
        return {"messages": outputs}

# 構建一個 tool node
tool_node = BasicToolNode(tools=[tool])

# 在 graph 中增加 tool node
graph_builder.add_node("tools", tool_node)
```

!!! info
    如果您將來不想自己建置它，您可以使用 LangGraph 的預先建置 `ToolNode`。

## 6. Define the conditional_edges

新增工具節點後，現在您可以定義 `conditional_edges`。

**Edges** 將控制流從一個節點路由到下一個節點。**Conditional edges** 從單一節點開始，通常包含 `if` 語句，根據目前 graph state 路由到不同的節點。這些函數接收目前 graph state，並傳回字串或字串列表，指示接下來要呼叫哪個節點。

接下來，定義一個名為 `route_tools` 的路由器函數，用於檢查聊天機器人輸出中是否存在 `tool_calls`。透過呼叫 `add_conditional_edges` 將此函數提供給 graph，這將告訴 graph，每當聊天機器人節點完成時，請檢查此函數以確定下一步要去哪裡。

如果存在工具調用，則條件將路由至 `tool` 節點；如果不存在 `tool calls`，則條件將路由至 `END`。由於條件可以傳回 `END`，因此您這次無需明確設定 `finish_point`。

```python
def route_tools(state: State):
    """
    Use in the conditional_edge to route to the ToolNode if the last message
    has tool calls. Otherwise, route to the end.
    """
    if isinstance(state, list):
        ai_message = state[-1]
    elif messages := state.get("messages", []):
        ai_message = messages[-1]
    else:
        raise ValueError(f"No messages found in input state to tool_edge: {state}")
    if hasattr(ai_message, "tool_calls") and len(ai_message.tool_calls) > 0:
        return "tools"
    return END


# The `tools_condition` function returns "tools" if the chatbot asks to use a tool, and "END" if
# it is fine directly responding. This conditional routing defines the main agent loop.
graph_builder.add_conditional_edges(
    "chatbot",
    route_tools,
    # The following dictionary lets you tell the graph to interpret the condition's outputs as a specific node
    # It defaults to the identity function, but if you
    # want to use a node named something else apart from "tools",
    # You can update the value of the dictionary to something else
    # e.g., "tools": "my_tools"
    {"tools": "tools", END: END},
)

# Any time a tool is called, we return to the chatbot to decide the next step
graph_builder.add_edge("tools", "chatbot")

graph_builder.add_edge(START, "chatbot")

graph = graph_builder.compile()
```

!!! info
    您可以用預先建立的 [tools_condition](https://langchain-ai.github.io/langgraph/reference/prebuilt/#tools_condition) 替換它，使其更加簡潔。

## 7. Visualize the graph (optional)

您可以使用 `get_graph` 方法和某個 `draw` 方法（例如 `draw_ascii` 或 `draw_png`）來視覺化 graph。每個 draw 方法都需要額外的依賴項。

```python
from IPython.display import Image, display

try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    # This requires some extra dependencies and is optional
    pass
```

![](https://langchain-ai.github.io/langgraph/tutorials/get-started/chatbot-with-tools.png)

## 8. Ask the bot questions

現在您可以向聊天機器人詢問其訓練資料以外的問題：

```python
def stream_graph_updates(user_input: str):
    for event in graph.stream({"messages": [{"role": "user", "content": user_input}]}):
        for value in event.values():
            print("Assistant:", value["messages"][-1].content)

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

## 9. Use prebuilts

為了方便使用，請調整您的程式碼，將以下內容替換為 LangGraph 預先建置元件。這些元件內建了並行 API 執行等功能。

- `BasicToolNode` 被預先建置的 `ToolNode` 取代
- `route_tools` 被預先建造的 `tools_condition` 替換

```bash
pip install -U "langchain[openai]"
```

```python
from typing import Annotated

from langchain_tavily import TavilySearch
from langchain_core.messages import BaseMessage
from typing_extensions import TypedDict

from langgraph.graph import StateGraph, START, END
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
# Any time a tool is called, we return to the chatbot to decide the next step
graph_builder.add_edge("tools", "chatbot")
graph_builder.add_edge(START, "chatbot")
graph = graph_builder.compile()
```

恭喜！您已在 LangGraph 中建立了一個對話代理，它可以使用搜尋引擎在需要時檢索更新的資訊。現在，它可以處理更廣泛的用戶查詢。要檢查您的代理剛剛執行的所有步驟，請查看此 LangSmith 追蹤。

## Next steps

聊天機器人無法自行記住過去的互動，這限制了它進行連貫、多輪對話的能力。在下一部分中，你將會加入記憶功能來解決這個問題。

