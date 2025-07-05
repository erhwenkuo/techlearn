# LangGraph quickstart

本指南向您展示如何設定和使用 LangGraph 的預先建置、可重複使用的元件，這些元件旨在幫助您快速可靠地建立代理系統。

## Prerequisites

在開始本教學之前，請確保您已具備以下條件：

-  OpenAI API 金鑰


## 1. Install dependencies

如果尚未安裝，請安裝 LangGraph 和 LangChain：

```bash
pip install -U langgraph  langchain-openai
```

## 2. Create an agent

若要建立 agent，請使用 `create_react_agent`：

```python
from langgraph.prebuilt import create_react_agent

def get_weather(city: str) -> str:  
    """Get weather for a given city."""
    return f"It's always sunny in {city}!"

agent = create_react_agent(
    model="openai:gtp-4.1-mini",  
    tools=[get_weather],  
    prompt="You are a helpful assistant"  
)

# Run the agent
agent.invoke(
    {"messages": [{"role": "user", "content": "what is the weather in sf"}]}
)
```

## 3. Configure an LLM

若要使用特定參數（例如溫度）配置 LLM，請使用 `init_chat_model`：

```python
from langchain.chat_models import init_chat_model
from langgraph.prebuilt import create_react_agent

def get_weather(city: str) -> str:  
    """Get weather for a given city."""
    return f"It's always sunny in {city}!"

model = init_chat_model(
    "openai:gtp-4.1-mini",
    temperature=0
)

agent = create_react_agent(
    model=model,
    tools=[get_weather],
)

# Run the agent
agent.invoke(
    {"messages": [{"role": "user", "content": "what is the weather in sf"}]}
)
```

有關如何配置 LLM 的更多信息，請參閱[模型](https://langchain-ai.github.io/langgraph/agents/models/)。

## 4. Add a custom prompt

提示會指示 LLM 如何執行操作。請新增以下類型的提示之一：

- **Static**：字串會被解釋為 `system message`。
- **Dynamic**：根據輸入或配置在 `runtime` 產生的 message list。


=== "Static prompt"

    定義固定的提示字串或訊息清單：

    ```python
    from langgraph.prebuilt import create_react_agent

    agent = create_react_agent(
        model="openai:gtp-4.1-mini",
        tools=[get_weather],
        # A static prompt that never changes
        prompt="Never answer questions about the weather."
    )

    agent.invoke(
        {"messages": [{"role": "user", "content": "what is the weather in sf"}]}
    )
    ```

    

=== "Dynamic prompt"

    定義一個根據代理程式的狀態和配置傳回訊息清單的函數：

    ```python
    from langchain_core.messages import AnyMessage
    from langchain_core.runnables import RunnableConfig
    from langgraph.prebuilt.chat_agent_executor import AgentState
    from langgraph.prebuilt import create_react_agent

    #定義一個根據代理程式的狀態和配置傳回訊息清單的函數
    def prompt(state: AgentState, config: RunnableConfig) -> list[AnyMessage]:  
        user_name = config["configurable"].get("user_name")
        system_msg = f"You are a helpful assistant. Address the user as {user_name}."
        return [{"role": "system", "content": system_msg}] + state["messages"]

    agent = create_react_agent(
        model="openai:gtp-4.1-mini",
        tools=[get_weather],
        prompt=prompt
    )

    # Run the agent
    agent.invoke(
        {"messages": [{"role": "user", "content": "what is the weather in sf"}]},
        config={"configurable": {"user_name": "Erhwen Kuo"}}
    )
    ```

有關詳細信息，請參閱 [Context](https://langchain-ai.github.io/langgraph/agents/context/)。

## 5. Add memory

若要允許與 agent 程式進行多輪對話，您需要在建立代理程式時提供 `checkpointer` 來啟用持久性。在運行時，您需要提供一個包含 `thread_id` 的配置 — conversation（session）的唯一識別碼：

```python
from langgraph.prebuilt import create_react_agent
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()

agent = create_react_agent(
    model="openai:gpt-4.1-mini",
    tools=[get_weather],
    checkpointer=checkpointer  
)

# Run the agent
config = {"configurable": {"thread_id": "1"}} # session 的唯一識別碼

sf_response = agent.invoke(
    {"messages": [{"role": "user", "content": "what is the weather in sf"}]},
    config  
)

ny_response = agent.invoke(
    {"messages": [{"role": "user", "content": "what about new york?"}]},
    config
)
```

啟用 `checkpointer` 後，它會將代理程式每一步的狀態儲存在提供的檢查點資料庫中（如果使用 `InMemorySaver`，則儲存在記憶體中）。

!!! info
    請注意，在上面的範例中，當使用相同的 `thread_id` 第二次呼叫代理程式時，會自動包含第一次對話的原始訊息歷史記錄以及新的使用者輸入。

## 6. Configure structured output

若要產生符合特定 schema 的結構化回應，請使用 `request_format` 參數。此模式可以使用 Pydantic 模型或 TypedDict 定義。結果可透過`structured_response` 欄位存取。

```python
from pydantic import BaseModel
from langgraph.prebuilt import create_react_agent

class WeatherResponse(BaseModel):
    conditions: str

agent = create_react_agent(
    model="openai:gpt-4.1-mini",
    tools=[get_weather],
    response_format=WeatherResponse  
)

response = agent.invoke(
    {"messages": [{"role": "user", "content": "what is the weather in sf"}]}
)

response["structured_response"]
```
