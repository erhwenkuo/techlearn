# Running agents

代理程式支援同步和非同步執行，可以使用 `.invoke()` / `await .ainvoke()` 實現完整響應，或使用 `.stream()` / `.astream()` 實現增量串流輸出。本節介紹如何提供輸入、解釋輸出、啟用串流以及控制執行限制。

## Basic usage

代理可以透過兩種主要模式執行：

- **Synchronous** 使用 `.invoke()` 或 `.stream()`
- **Asynchronous** 使用 `await .ainvoke()` 或 `async for` 與 `.astream()`

=== "Syn invocation"

    ```python
    from langgraph.prebuilt import create_react_agent

    agent = create_react_agent(...)

    response = agent.invoke({"messages": [{"role": "user", "content": "what is the weather in sf"}]})
    ```

=== "Async invocation"

    ```python
    from langgraph.prebuilt import create_react_agent

    agent = create_react_agent(...)

    response = await agent.ainvoke({"messages": [{"role": "user", "content": "what is the weather in sf"}]})
    ```

## Inputs and outputs

代理程式使用的語言模型需要以 `messages` 列表作為輸入。因此，代理程式的輸入和輸出以訊息清單的形式儲存在代理狀態的 `messages` 的鍵。


## Input format

代理輸入必須是包含 `messages` 鍵的 dictionary 。支援的格式如下：

| Format              | Example                                                                 |
|---------------------|-------------------------------------------------------------------------|
| String              | `{"messages": "Hello"}` — 解譯為 `HumanMessage`                 |
| Message dictionary  | `{"messages": {"role": "user", "content": "Hello"}}`                    |
| List of messages    | `{"messages": [{"role": "user", "content": "Hello"}]}`                  |
| With custom state   | `{"messages": [{"role": "user", "content": "Hello"}], "user_name": "Alice"}` — 如果使用自訂 state_schema |

Messages 自動轉換為 LangChain 的內部 `message` 格式。您可以在 LangChain 文件中了解更多關於 [LangChain `messages`](https://python.langchain.com/docs/concepts/messages/?_gl=1*xkt5yx*_gcl_au*MTMzOTIyOTUzNS4xNzUwNTUxODU5*_ga*MTM5NTM1ODE0Mi4xNzQ3MDkyMTE0*_ga_47WX3HKKY2*czE3NTE2ODMyODIkbzE4JGcxJHQxNzUxNjg1MzI1JGo2MCRsMCRoMA..#langchain-messages) 的資訊。

!!! note
    訊息的字串輸入會被轉換為 `HumanMessage`。此行為與 `create_react_agent` 中的 prompt 參數不同，後者以字串形式傳遞時會被解釋為 `SystemMessage`。


## Output format

Agent output 是一個包含以下內容的 dictionary：
- `messages`: Agents 執行期間交換的所有訊息的清單（使用者輸入、助手回覆、工具呼叫）。
- `structured_response` (optional): 如果配置了 [structured output](https://langchain-ai.github.io/langgraph/agents/agents/#6-configure-structured-output)。
- `state_schema`: 如果使用自訂 state_schema，則輸出中也可能存在與您定義的欄位對應的附加鍵。這些鍵可以保存工具執行或提示邏輯中更新的狀態值。

有關使用自訂狀態模式和存取上下文的更多詳細信息，請參閱 [context guide](https://langchain-ai.github.io/langgraph/agents/context/)。

## Streaming output

Agents 程式支援串流響應，以實現更靈敏的應用程式。其中包括：

- **Progress updates** after each step
- **LLM tokens** as they're generated
- **Custom tool messages** during execution

Streaming 可在同步和非同步模式下使用：

=== "Sync streaming"

    ```python
    for chunk in agent.stream(
        {"messages": [{"role": "user", "content": "what is the weather in sf"}]},
        stream_mode="updates"
    ):
        print(chunk)
    ```

=== "Async streaming"

    ```python
    async for chunk in agent.astream(
        {"messages": [{"role": "user", "content": "what is the weather in sf"}]},
        stream_mode="updates"
    ):
        print(chunk)
    ```

## Max iterations

為了控制代理程式的執行並避免無限循環，請設定 `recursion limit`。這定義了代理程式在引發 `GraphRecursionError` 之前可以執行的最大步數。您可以在執行時期或透過 `.with_config()` 定義代理程式時設定 `recursion_limit`：

=== "Runtime"

    ```python hl_lines="5 14"
    from langgraph.errors import GraphRecursionError
    from langgraph.prebuilt import create_react_agent

    max_iterations = 3
    recursion_limit = 2 * max_iterations + 1
    agent = create_react_agent(
        model="anthropic:claude-3-5-haiku-latest",
        tools=[get_weather]
    )

    try:
        response = agent.invoke(
            {"messages": [{"role": "user", "content": "what's the weather in sf"}]},
            {"recursion_limit": recursion_limit},
        )
    except GraphRecursionError:
        print("Agent stopped due to max iterations.")
    ```

=== ".with_config()"

    ```python hl_lines="5 10"
    from langgraph.errors import GraphRecursionError
    from langgraph.prebuilt import create_react_agent

    max_iterations = 3
    recursion_limit = 2 * max_iterations + 1
    agent = create_react_agent(
        model="anthropic:claude-3-5-haiku-latest",
        tools=[get_weather]
    )
    agent_with_recursion_limit = agent.with_config(recursion_limit=recursion_limit)

    try:
        response = agent_with_recursion_limit.invoke(
            {"messages": [{"role": "user", "content": "what's the weather in sf"}]},
        )
    except GraphRecursionError:
        print("Agent stopped due to max iterations.")
    ```
