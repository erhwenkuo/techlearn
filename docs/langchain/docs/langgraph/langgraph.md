# LangGraph

⚡ 將語言 agents 建構成 graph ⚡

## 概述

LangGraph 是一個用於使用 LLM 建立 {==stateful、multi-actor==} 應用程式的函式庫，它建構在 LangChain 之上。它擴展了 LangChain 表達式語言，能夠以循環方式跨多個計算步驟協調多個 chains（或參與者）。它的靈感來自 Pregel 和 Apache Beam。目前公開的介面是受 NetworkX 啟發的。

主要用途是為您的 LLM 申請添加 cycles 的循環呼叫設計。至關重要的是，這不是 DAG 框架。如果你想建立一個 DAG，你應該使用 LangChain 表達式語言。

循環(cycles) 對於 agent-like 的行為非常重要，您可以在循環中呼叫 LLM，詢問它下一步要採取什麼操作。

## 安裝

```bash
pip install langgraph
```

## Quick Start

在這裡，我們將回顧一個創建使用 chat model 和 function calling 的簡單代理程式的範例。該代理將其所有狀態表示為訊息清單。

我們需要安裝一些 LangChain 軟體包以及 Tavily 來用作範例工具。

```bash
pip install -U langchain langchain_openai tavily-python
```

我們還需要導出一些環境變數以供 OpenAI 和 Tavily API 存取。

```bash
export OPENAI_API_KEY=sk-...
export TAVILY_API_KEY=tvly-...
```

或者，我們可以設定 LangSmith 以獲得一流的可觀察性。

```bash
export LANGCHAIN_TRACING_V2="true"
export LANGCHAIN_API_KEY=ls__...
```

### 設定 tool

我們將首先定義我們想要使用的工具。對於這個簡單的範例，我們將透過 Tavily 使用內建搜尋工具。然而，創建自己的工具確實很容易 - 請參閱此處的[文件](https://python.langchain.com/docs/modules/agents/tools/custom_tools)以了解如何做到這一點。

```python
from langchain_community.tools.tavily_search import TavilySearchResults

tools = [TavilySearchResults(max_results=1)]
```

現在我們可以將這些工具包裝在一個簡單的 LangGraph `ToolExecutor` 中。這是一個簡單的類別，它接收 `ToolInitation` 物件、呼叫該工具並傳回輸出。 `ToolIncation` 是具有 `tool` 和 `tool_input` 屬性的任何類別。

```python
from langgraph.prebuilt import ToolExecutor

tool_executor = ToolExecutor(tools)
```

### 設定模型

現在我們需要載入我們想要使用的 chat model。重要的是，這應該滿足兩個標準：

1. 它應該與訊息列表一起使用。我們將以訊息的形式表示所有代理狀態，因此它需要能夠與它們很好地配合。
2. 它應該與 OpenAI 函數呼叫介面配合使用。這意味著它應該是 OpenAI 模型或公開類似介面的模型。

!!! info
    注意：這些模型要求不是使用 LangGraph 的要求 - 它們只是這個範例的要求。

```python
from langchain_openai import ChatOpenAI

# We will set streaming=True so that we can stream tokens
# See the streaming section for more information on this.
model = ChatOpenAI(temperature=0, streaming=True)
```

完成此操作後，我們應該確保模型知道它可以呼叫這些工具。我們可以透過將 LangChain 工具轉換為 OpenAI function calling 的格式，然後將它們綁定到模型類別來實現。

```python
from langchain.tools.render import format_tool_to_openai_function

functions = [format_tool_to_openai_function(t) for t in tools]

model = model.bind_functions(functions)
```

### 定義代理狀態

langgraph 中的主要 graph 類型是 `StatefulGraph`。此 graph 由傳遞到每個節點的 state object 參數化。然後每個節點返回操作來更新該狀態。這些操作可以設定狀態的特定屬性（例如覆寫現有值）或新增至現有屬性。是設定還是添加是透過註解用於建構圖形的狀態物件來指示的。

對於這個例子，我們將追蹤的狀態只是一個訊息清單。我們希望每個節點只將訊息新增到該清單中。因此，我們將使用帶有一個鍵（messages）的 `TypedDict` 並對其進行註釋，以便始終添加 messages 屬性。

```python
from typing import TypedDict, Annotated, Sequence
import operator
from langchain_core.messages import BaseMessage


class AgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], operator.add]
```

### 定義節點

我們現在需要在 graph 中定義一些不同的 node。在 langgraph 中，節點可以是 function，也可以是 runnable 的。為此，我們需要兩個主要節點：

1. Agent：負責決定採取什麼（如果有）行動。
2. 一個呼叫 tool 的函數：如果 agent 決定採取操作，則該節點將執行該操作。

我們還需要定義一些 edges。其中一些 edge 可能是有條件的。它們是有條件的原因是，根據節點的輸出，可以採取多個路徑之一。在該節點運行之前，所採用的路徑是未知的（LLM 決定）。

1. Conditional Edge: 當代理呼叫後，我們應該：
    a. 如果代理說要採取行動，那麼應該呼叫呼叫工具的函數
    b. 如果代理說已經完成，那麼就應該完成

2. Normal Edge: 當工具呼叫後，它應該始終返回給 agent 來決定下一步要做什麼

讓我們定義節點以及決定如何採用條件邊的函數。

```python
from langgraph.prebuilt import ToolInvocation
import json
from langchain_core.messages import FunctionMessage

# Define the function that determines whether to continue or not
def should_continue(state):
    messages = state['messages']
    last_message = messages[-1]
    # If there is no function call, then we finish
    if "function_call" not in last_message.additional_kwargs:
        return "end"
    # Otherwise if there is, we continue
    else:
        return "continue"

# Define the function that calls the model
def call_model(state):
    messages = state['messages']
    response = model.invoke(messages)
    # We return a list, because this will get added to the existing list
    return {"messages": [response]}

# Define the function to execute tools
def call_tool(state):
    messages = state['messages']
    # Based on the continue condition
    # we know the last message involves a function call
    last_message = messages[-1]
    # We construct an ToolInvocation from the function_call
    action = ToolInvocation(
        tool=last_message.additional_kwargs["function_call"]["name"],
        tool_input=json.loads(last_message.additional_kwargs["function_call"]["arguments"]),
    )
    # We call the tool_executor and get back a response
    response = tool_executor.invoke(action)
    # We use the response to create a FunctionMessage
    function_message = FunctionMessage(content=str(response), name=action.tool)
    # We return a list, because this will get added to the existing list
    return {"messages": [function_message]}
```

### 定義 graph

我們現在可以將它們放在一起並定義 graph！

```python
from langgraph.graph import StateGraph, END
# Define a new graph
workflow = StateGraph(AgentState)

# Define the two nodes we will cycle between
workflow.add_node("agent", call_model)
workflow.add_node("action", call_tool)

# Set the entrypoint as `agent`
# This means that this node is the first one called
workflow.set_entry_point("agent")

# We now add a conditional edge
workflow.add_conditional_edges(
    # First, we define the start node. We use `agent`.
    # This means these are the edges taken after the `agent` node is called.
    "agent",
    # Next, we pass in the function that will determine which node is called next.
    should_continue,
    # Finally we pass in a mapping.
    # The keys are strings, and the values are other nodes.
    # END is a special node marking that the graph should finish.
    # What will happen is we will call `should_continue`, and then the output of that
    # will be matched against the keys in this mapping.
    # Based on which one it matches, that node will then be called.
    {
        # If `tools`, then we call the tool node.
        "continue": "action",
        # Otherwise we finish.
        "end": END
    }
)

# We now add a normal edge from `tools` to `agent`.
# This means that after `tools` is called, `agent` node is called next.
workflow.add_edge('action', 'agent')

# Finally, we compile it!
# This compiles it into a LangChain Runnable,
# meaning you can use it as you would any other runnable
app = workflow.compile()
```

### 執行


我們現在可以使用它了！現在，它公開了與所有其他 LangChain 相同的 runnable 介面。該可運行程式接受訊息清單。

```python
from langchain_core.messages import HumanMessage

inputs = {"messages": [HumanMessage(content="what is the weather in sf")]}
app.invoke(inputs)
```

## Streaming

LangGraph 支援多種不同類型的串流。

### 串流 Node Output

使用 LangGraph 的好處之一是可以輕鬆地傳輸由每個節點產生的輸出。

```python
inputs = {"messages": [HumanMessage(content="what is the weather in sf")]}

for output in app.stream(inputs):
    # stream() yields dictionaries with output keyed by node name
    for key, value in output.items():
        print(f"Output from node '{key}':")
        print("---")
        print(value)
    print("\n---\n")
```

結果:

```bash
Output from node 'agent':
---
{'messages': [AIMessage(content='', additional_kwargs={'function_call': {'arguments': '{\n  "query": "weather in San Francisco"\n}', 'name': 'tavily_search_results_json'}})]}

---

Output from node 'action':
---
{'messages': [FunctionMessage(content="[{'url': 'https://weatherspark.com/h/m/557/2024/1/Historical-Weather-in-January-2024-in-San-Francisco-California-United-States', 'content': 'January 2024 Weather History in San Francisco California, United States  Daily Precipitation in January 2024 in San Francisco Observed Weather in January 2024 in San Francisco  San Francisco Temperature History January 2024 Hourly Temperature in January 2024 in San Francisco  Hours of Daylight and Twilight in January 2024 in San FranciscoThis report shows the past weather for San Francisco, providing a weather history for January 2024. It features all historical weather data series we have available, including the San Francisco temperature history for January 2024. You can drill down from year to month and even day level reports by clicking on the graphs.'}]", name='tavily_search_results_json')]}

---

Output from node 'agent':
---
{'messages': [AIMessage(content="I couldn't find the current weather in San Francisco. However, you can visit [WeatherSpark](https://weatherspark.com/h/m/557/2024/1/Historical-Weather-in-January-2024-in-San-Francisco-California-United-States) to check the historical weather data for January 2024 in San Francisco.")]}

---

Output from node '__end__':
---
{'messages': [HumanMessage(content='what is the weather in sf'), AIMessage(content='', additional_kwargs={'function_call': {'arguments': '{\n  "query": "weather in San Francisco"\n}', 'name': 'tavily_search_results_json'}}), FunctionMessage(content="[{'url': 'https://weatherspark.com/h/m/557/2024/1/Historical-Weather-in-January-2024-in-San-Francisco-California-United-States', 'content': 'January 2024 Weather History in San Francisco California, United States  Daily Precipitation in January 2024 in San Francisco Observed Weather in January 2024 in San Francisco  San Francisco Temperature History January 2024 Hourly Temperature in January 2024 in San Francisco  Hours of Daylight and Twilight in January 2024 in San FranciscoThis report shows the past weather for San Francisco, providing a weather history for January 2024. It features all historical weather data series we have available, including the San Francisco temperature history for January 2024. You can drill down from year to month and even day level reports by clicking on the graphs.'}]", name='tavily_search_results_json'), AIMessage(content="I couldn't find the current weather in San Francisco. However, you can visit [WeatherSpark](https://weatherspark.com/h/m/557/2024/1/Historical-Weather-in-January-2024-in-San-Francisco-California-United-States) to check the historical weather data for January 2024 in San Francisco.")]}

---
```

### 串流 LLM Tokens

您也可以存取每個節點產生的 LLM tokens。在這種情況下，只有 agent 節點產生 LLM 令牌。為了使其正常工作，您必須使用支援串流的 LLM，並在建置 LLM 時對其進行設定（例如`ChatOpenAI(model="gpt-3.5-turbo-1106"，streaming=True)`）

```python
inputs = {"messages": [HumanMessage(content="what is the weather in sf")]}

async for output in app.astream_log(inputs, include_types=["llm"]):
    # astream_log() yields the requested logs (here LLMs) in JSONPatch format
    for op in output.ops:
        if op["path"] == "/streamed_output/-":
            # this is the output from .stream()
            ...
        elif op["path"].startswith("/logs/") and op["path"].endswith(
            "/streamed_output/-"
        ):
            # because we chose to only include LLMs, these are LLM tokens
            print(op["value"])
```

結果:

```bash
content='' additional_kwargs={'function_call': {'arguments': '', 'name': 'tavily_search_results_json'}}
content='' additional_kwargs={'function_call': {'arguments': '{\n', 'name': ''}}
content='' additional_kwargs={'function_call': {'arguments': ' ', 'name': ''}}
content='' additional_kwargs={'function_call': {'arguments': ' "', 'name': ''}}
content='' additional_kwargs={'function_call': {'arguments': 'query', 'name': ''}}
content='' additional_kwargs={'function_call': {'arguments': '":', 'name': ''}}
content='' additional_kwargs={'function_call': {'arguments': ' "', 'name': ''}}
content='' additional_kwargs={'function_call': {'arguments': 'weather', 'name': ''}}
content='' additional_kwargs={'function_call': {'arguments': ' in', 'name': ''}}
content='' additional_kwargs={'function_call': {'arguments': ' San', 'name': ''}}
content='' additional_kwargs={'function_call': {'arguments': ' Francisco', 'name': ''}}
content='' additional_kwargs={'function_call': {'arguments': '"\n', 'name': ''}}
content='' additional_kwargs={'function_call': {'arguments': '}', 'name': ''}}
content=''
content=''
content='I'
content="'m"
content=' sorry'
content=','
content=' but'
content=' I'
content=' couldn'
content="'t"
content=' find'
content=' the'
content=' current'
content=' weather'
content=' in'
content=' San'
content=' Francisco'
content='.'
content=' However'
content=','
content=' you'
content=' can'
content=' check'
content=' the'
content=' historical'
content=' weather'
content=' data'
content=' for'
content=' January'
content=' '
content='202'
content='4'
content=' in'
content=' San'
content=' Francisco'
content=' ['
content='here'
content=']('
content='https'
content='://'
content='we'
content='athers'
content='park'
content='.com'
content='/h'
content='/m'
content='/'
content='557'
content='/'
content='202'
content='4'
content='/'
content='1'
content='/H'
content='istorical'
content='-'
content='Weather'
content='-in'
content='-Jan'
content='uary'
content='-'
content='202'
content='4'
content='-in'
content='-S'
content='an'
content='-F'
content='r'
content='anc'
content='isco'
content='-Cal'
content='ifornia'
content='-'
content='United'
content='-'
content='States'
content=').'
content=''
```

## 何時使用

什麼時候應該使用 LangGraph 而不是 LangChain 表達式語言？

如果你需要 cycle。

Langchain 表達式語言可讓您輕鬆定義 chain (DAG)，但它沒有良好的添加循環的機制。 而 langgraph 新增了該語法。

## 操作指南

這些指南顯示如何以特定方式使用 LangGraph。

### Async

如果您在非同步工作流程中執行 LangGraph，您可能想要預設建立非同步節點。有關如何執行此操作的演練，請參閱此[文檔](https://github.com/langchain-ai/langgraph/blob/main/examples/async.ipynb)

### Streaming Tokens

有時，語言模型需要一段時間才能回應，您可能希望將令牌串流傳輸給最終用戶。有關如何執行此操作的指南，請參閱此[文檔](https://github.com/langchain-ai/langgraph/blob/main/examples/streaming-tokens.ipynb)

### Persistence

LangGraph 具有內建的持久性，可讓您儲存目前圖形的狀態並從那裡恢復。有關如何執行此操作的演練，請參閱此[文檔](https://github.com/langchain-ai/langgraph/blob/main/examples/persistence.ipynb)

### Human-in-the-loop

LangGraph 內建了人機互動工作流程的支援。當您希望在進入特定節點之前讓人工檢查當前狀態時，這非常有用。有關如何執行此操作的演練，請參閱此[文檔](https://github.com/langchain-ai/langgraph/blob/main/examples/human-in-the-loop.ipynb)

### Visualizing the graph

使用 LangGraph 建立的代理程式可能很複雜。為了更容易理解幕後發生的事情，我們添加了列印和視覺化圖表的方法。這可以創建 ascii 藝術和 png。有關如何執行此操作的演練，請參閱此[文檔](https://github.com/langchain-ai/langgraph/blob/main/examples/visualization.ipynb)



