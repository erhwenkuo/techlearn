# Customize state

在本教程中，您將向 graph state 添加其他 property，以定義複雜的行為，而無需依賴訊息列表。聊天機器人將使用其搜尋工具來尋找特定資訊並將其轉發給人工審核。

## 1. Add keys to the state

透過向狀態添加名稱和生日鍵來更新聊天機器人以研究實體的生日：

```python hl_lines="10 11"
from typing import Annotated

from typing_extensions import TypedDict

from langgraph.graph.message import add_messages


class State(TypedDict):
    messages: Annotated[list, add_messages]
    name: str
    birthday: str
```

將此資訊加入 state 中，使得其他 graph 節點（如儲存或處理資訊的下游節點）以及圖形的持久層可以輕鬆存取它。

## 2. Update the state inside the tool

現在，在 `human_assistance` 工具中填入狀態鍵。這允許人類在資訊儲存到狀態之前對其進行審核。使用指令從工具內部發出狀態更新。

```python
from langchain_core.messages import ToolMessage
from langchain_core.tools import InjectedToolCallId, tool

from langgraph.types import Command, interrupt

@tool
# Note that because we are generating a ToolMessage for a state update, we
# generally require the ID of the corresponding tool call. We can use
# LangChain's InjectedToolCallId to signal that this argument should not
# be revealed to the model in the tool's schema.
# 請注意，由於我們正在為狀態更新產生 ToolMessage，因此
# 通常需要對應工具呼叫的 ID。我們可以使用
# LangChain 的 InjectedToolCallId 來指示此參數不應
# 在工具的架構中透露給模型。
def human_assistance(name: str, birthday: str, tool_call_id: Annotated[str, InjectedToolCallId]) -> str:
    """Request assistance from a human."""
    human_response = interrupt(
        {
            "question": "Is this correct?",
            "name": name,
            "birthday": birthday,
        },
    )

    # If the information is correct, update the state as-is.
    if human_response.get("correct", "").lower().startswith("y"):
        verified_name = name
        verified_birthday = birthday
        response = "Correct"
    # Otherwise, receive information from the human reviewer.
    else:
        verified_name = human_response.get("name", name)
        verified_birthday = human_response.get("birthday", birthday)
        response = f"Made a correction: {human_response}"

    # This time we explicitly update the state with a ToolMessage inside
    # the tool.
    state_update = {
        "name": verified_name,
        "birthday": verified_birthday,
        "messages": [ToolMessage(response, tool_call_id=tool_call_id)],
    }

    # We return a Command object in the tool to update our state.
    return Command(update=state_update)
```

Graph 的其餘部分保持不變。

## 3. Prompt the chatbot

提示聊天機器人尋找 LangGraph 的 birthday，並在獲得所需資訊後指示聊天機器人聯繫 `human_assistance` 工具。透過在該工具的參數中設定姓名和生日，您可以強制聊天機器人為這些欄位產生提案。

```python
user_input = (
    "Can you look up when LangGraph was released? "
    "When you have the answer, use the human_assistance tool for review."
)

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

Can you look up when LangGraph was released? When you have the answer, use the human_assistance tool for review.
================================== Ai Message ==================================

[{'text': "Certainly! I'll start by searching for information about LangGraph's release date using the Tavily search function. Then, I'll use the human_assistance tool for review.", 'type': 'text'}, {'id': 'toolu_01JoXQPgTVJXiuma8xMVwqAi', 'input': {'query': 'LangGraph release date'}, 'name': 'tavily_search_results_json', 'type': 'tool_use'}]
Tool Calls:
  tavily_search_results_json (toolu_01JoXQPgTVJXiuma8xMVwqAi)
 Call ID: toolu_01JoXQPgTVJXiuma8xMVwqAi
  Args:
    query: LangGraph release date
================================= Tool Message =================================
Name: tavily_search_results_json

[{"url": "https://blog.langchain.dev/langgraph-cloud/", "content": "We also have a new stable release of LangGraph. By LangChain 6 min read Jun 27, 2024 (Oct '24) Edit: Since the launch of LangGraph Platform, we now have multiple deployment options alongside LangGraph Studio - which now fall under LangGraph Platform. LangGraph Platform is synonymous with our Cloud SaaS deployment option."}, {"url": "https://changelog.langchain.com/announcements/langgraph-cloud-deploy-at-scale-monitor-carefully-iterate-boldly", "content": "LangChain - Changelog | ☁ 🚀 LangGraph Platform: Deploy at scale, monitor LangChain LangSmith LangGraph LangChain LangSmith LangGraph LangChain LangSmith LangGraph LangChain Changelog Sign up for our newsletter to stay up to date DATE: The LangChain Team LangGraph LangGraph Platform ☁ 🚀 LangGraph Platform: Deploy at scale, monitor carefully, iterate boldly DATE: June 27, 2024 AUTHOR: The LangChain Team LangGraph Platform is now in closed beta, offering scalable, fault-tolerant deployment for LangGraph agents. LangGraph Platform also includes a new playground-like studio for debugging agent failure modes and quick iteration: Join the waitlist today for LangGraph Platform. And to learn more, read our blog post announcement or check out our docs. Subscribe By clicking subscribe, you accept our privacy policy and terms and conditions."}]
================================== Ai Message ==================================

[{'text': "Based on the search results, it appears that LangGraph was already in existence before June 27, 2024, when LangGraph Platform was announced. However, the search results don't provide a specific release date for the original LangGraph. \n\nGiven this information, I'll use the human_assistance tool to review and potentially provide more accurate information about LangGraph's initial release date.", 'type': 'text'}, {'id': 'toolu_01JDQAV7nPqMkHHhNs3j3XoN', 'input': {'name': 'Assistant', 'birthday': '2023-01-01'}, 'name': 'human_assistance', 'type': 'tool_use'}]
Tool Calls:
  human_assistance (toolu_01JDQAV7nPqMkHHhNs3j3XoN)
 Call ID: toolu_01JDQAV7nPqMkHHhNs3j3XoN
  Args:
    name: Assistant
    birthday: 2023-01-01
```

我們再次遇到了 `human_assistance` 工具中的中斷。

## 4. Add human assistance

聊天機器人無法識別正確的日期，因此請向其提供以下資訊：

```python
human_command = Command(
    resume={
        "name": "LangGraph",
        "birthday": "Jan 17, 2024",
    },
)

events = graph.stream(human_command, config, stream_mode="values")

for event in events:
    if "messages" in event:
        event["messages"][-1].pretty_print()
```

```bash
================================== Ai Message ==================================

[{'text': "Based on the search results, it appears that LangGraph was already in existence before June 27, 2024, when LangGraph Platform was announced. However, the search results don't provide a specific release date for the original LangGraph. \n\nGiven this information, I'll use the human_assistance tool to review and potentially provide more accurate information about LangGraph's initial release date.", 'type': 'text'}, {'id': 'toolu_01JDQAV7nPqMkHHhNs3j3XoN', 'input': {'name': 'Assistant', 'birthday': '2023-01-01'}, 'name': 'human_assistance', 'type': 'tool_use'}]
Tool Calls:
  human_assistance (toolu_01JDQAV7nPqMkHHhNs3j3XoN)
 Call ID: toolu_01JDQAV7nPqMkHHhNs3j3XoN
  Args:
    name: Assistant
    birthday: 2023-01-01
================================= Tool Message =================================
Name: human_assistance

Made a correction: {'name': 'LangGraph', 'birthday': 'Jan 17, 2024'}
================================== Ai Message ==================================

Thank you for the human assistance. I can now provide you with the correct information about LangGraph's release date.

LangGraph was initially released on January 17, 2024. This information comes from the human assistance correction, which is more accurate than the search results I initially found.

To summarize:
1. LangGraph's original release date: January 17, 2024
2. LangGraph Platform announcement: June 27, 2024

It's worth noting that LangGraph had been in development and use for some time before the LangGraph Platform announcement, but the official initial release of LangGraph itself was on January 17, 2024.
```

請注意，這些欄位現在反映在狀態中：

```python
snapshot = graph.get_state(config)

{k: v for k, v in snapshot.values.items() if k in ("name", "birthday")}
```

```bash
{'name': 'LangGraph', 'birthday': 'Jan 17, 2024'}
```

這使得下游節點（例如，進一步處理或儲存資訊的節點）可以輕鬆存取它們。

## 5. Manually update the state

LangGraph 對應用程式狀態提供了高度的控制。例如，在任何時候（包括中斷時），你都可以使用 `graph.update_state` 手動覆蓋某個鍵：

```python
graph.update_state(config, {"name": "LangGraph (library)"})
```

```bash
{
    'configurable': {
        'thread_id': '1',
        'checkpoint_ns': '',
        'checkpoint_id': '1efd4ec5-cf69-6352-8006-9278f1730162'
    }
}
```

## 6. View the new value

如果你呼叫 `graph.get_state` ，你可以看到反映出的新值：

```python
snapshot = graph.get_state(config)

{k: v for k, v in snapshot.values.items() if k in ("name", "birthday")}
```

```bash
{'name': 'LangGraph (library)', 'birthday': 'Jan 17, 2024'}
```

手動狀態更新會在 LangSmith 中產生追蹤。如果需要，它們也可以用來控制人機互動的工作流程。通常建議使用中斷函數，因為它允許在人機互動中傳輸數據，而不受狀態更新的影響。

恭喜！您已向狀態新增了自訂鍵，以方便更複雜的工作流程，並學習如何從工具內部產生狀態更新。

請參閱下面的程式碼片段，並查看本教學中的 graph：

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
from langchain_core.messages import ToolMessage
from langchain_core.tools import InjectedToolCallId, tool
from typing_extensions import TypedDict

from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition
from langgraph.types import Command, interrupt

class State(TypedDict):
    messages: Annotated[list, add_messages]
    name: str
    birthday: str

@tool
def human_assistance(
    name: str, birthday: str, tool_call_id: Annotated[str, InjectedToolCallId]
) -> str:
    """Request assistance from a human."""
    human_response = interrupt(
        {
            "question": "Is this correct?",
            "name": name,
            "birthday": birthday,
        },
    )

    if human_response.get("correct", "").lower().startswith("y"):
        verified_name = name
        verified_birthday = birthday
        response = "Correct"
    else:
        verified_name = human_response.get("name", name)
        verified_birthday = human_response.get("birthday", birthday)
        response = f"Made a correction: {human_response}"

    state_update = {
        "name": verified_name,
        "birthday": verified_birthday,
        "messages": [ToolMessage(response, tool_call_id=tool_call_id)],
    }

    return Command(update=state_update)


tool = TavilySearch(max_results=2)
tools = [tool, human_assistance]
llm_with_tools = llm.bind_tools(tools)

def chatbot(state: State):
    message = llm_with_tools.invoke(state["messages"])
    assert(len(message.tool_calls) <= 1)
    return {"messages": [message]}

graph_builder = StateGraph(State)
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

## Next steps

在完成 LangGraph 基礎教學之前，還有一個概念需要複習：將檢查點和狀態更新與時間旅行連結起來。