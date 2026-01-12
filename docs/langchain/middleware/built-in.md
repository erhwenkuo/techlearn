# Built-in middleware

> Prebuilt middleware for common agent use cases

LangChain 提供常見用例的預先建置中間件。每個中間件都已準備好投入生產環境，並且可以根據您的特定需求進行配置。

## Provider-agnostic middleware

以下中間件可與任何 LLM provider 配合使用：

| Middleware | Description  |
| ---------- | ------------ |
| [Summarization](#summarization)         | 當接近會話次數上限時，自動匯總對話歷史記錄。|
| [Human-in-the-loop](#human-in-the-loop) | 暫停執行，等待人工審核工具呼叫。          |
| [Model call limit](#model-call-limit)   | 限制模型呼叫次數，以防止成本過高。        |
| [Tool call limit](#tool-call-limit)     | 透過限制呼叫次數來控制工具執行。          |
| [Model fallback](#model-fallback)       | 當主模型發生故障時，自動回退到備用模型。   |
| [PII detection](#pii-detection)         | 偵測和處理個人識別資訊 (PII)。           |
| [To-do list](#to-do-list)               | 為 agent 配備任務規劃和追蹤功能。        |
| [LLM tool selector](#llm-tool-selector) | 使用 LLM 在呼叫主模型之前選擇相關工具。   |
| [Tool retry](#tool-retry)               | 使用指數退避演算法自動重試失敗的工具呼叫。  |
| [Model retry](#model-retry)             | 使用指數退避演算法自動重試失敗的模型呼叫。  |
| [LLM tool emulator](#llm-tool-emulator) | 使用 LLM 模擬工具執行以進行測試。         |
| [Context editing](#context-editing)     | 透過精簡或清除工具使用情況來管理對話上下文。 |
| [Shell tool](#shell-tool)               | 向 agent 公開持久 shell 會話以執行命令。  |
| [File search](#file-search)             | 提供檔案系統檔案的 Glob 和 Grep 搜尋工具。 |


### Summarization

當接近 token limits 上限時，會自動產生對話歷史記錄摘要，保留最近的訊息，同時壓縮較早的上下文。Summarization 適用於以下情況：

- 持續時間過長的對話，超出上下文視窗的限制。
- 豐富歷史記錄的多輪對話。
- 需要完整保留對話上下文的應用場景。

API reference: [SummarizationMiddleware](https://reference.langchain.com/python/langchain/middleware/?_gl=1*g03z6c*_gcl_au*MTA0NTc2NjA2OC4xNzYzNzI5OTg2*_ga*MTM5NTM1ODE0Mi4xNzQ3MDkyMTE0*_ga_47WX3HKKY2*czE3NjcwNTA0NzckbzkyJGcxJHQxNzY3MDUxNTQyJGo2MCRsMCRoMA..#langchain.agents.middleware.SummarizationMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware

agent = create_agent(
    model="gpt-4o",
    tools=[your_weather_tool, your_calculator_tool],
    middleware=[
        SummarizationMiddleware(
            model="gpt-4o-mini",
            trigger=("tokens", 4000),
            keep=("messages", 20),
        ),
    ],
)
```

#### Configuration options

!!! tip
    `trigger` 和 `keep` 的 `fraction` 條件（如下所示）依賴聊天模型的個人資料資料（如果使用 langchain>=1.1）。如果資料不可用，請使用其他條件或手動指定：

    ```python
    from langchain.chat_models import init_chat_model

    custom_profile = {
        "max_input_tokens": 100_000,
        # ...
    }

    model = init_chat_model("gpt-4o", profile=custom_profile)
    ```

- `model`: string | BaseChatModel `required`

    用於產生摘要的模型。可以是模型識別字串（例如，`openai:gpt-4o-mini`），也可以是 `BaseChatModel` 實例。有關更多信息，請參閱 `init_chat_model`。

- `trigger`: ContextSize | list[ContextSize] | None

    觸發摘要生成的條件。可以是：

    - 單一 `ContextSize` 元組（必須滿足指定條件）
    - `ContextSize` 元組清單（必須滿足任意條件 - 邏輯 OR 運算）

    條件應為以下之一：

    - `fraction`（float）：模型上下文大小的比例（0-1）
    - `tokens`（int）：詞元總數
    - `messages`（int）：訊息總數

    必須至少指定一個條件。如果未提供條件，則不會自動觸發摘要。

    有關更多信息，請參閱 [ContextSize](https://reference.langchain.com/python/langchain/middleware/?_gl=1*152nfr0*_gcl_au*MTA0NTc2NjA2OC4xNzYzNzI5OTg2*_ga*MTM5NTM1ODE0Mi4xNzQ3MDkyMTE0*_ga_47WX3HKKY2*czE3NjcwOTY3MjkkbzkzJGcxJHQxNzY3MDk2NzU0JGozNSRsMCRoMA..#langchain.agents.middleware.summarization.ContextSize) 的 API 參考文檔。


### To-do list

為 agent 配備任務規劃和追蹤功能，以處理複雜的多步驟任務。待辦事項清單在以下方面非常有用：

- 需要協調多種工具的複雜多步驟任務。
- 需要密切注意進度可見性的長期運行任務。

!!! info
    此中間件會自動為 agent 提供 `write_todos` 工具和 system prompts，以指導有效的任務規劃。

API reference: [TodoListMiddleware](https://reference.langchain.com/python/langchain/middleware/?_gl=1*e0i3zx*_gcl_au*MTA0NTc2NjA2OC4xNzYzNzI5OTg2*_ga*MTM5NTM1ODE0Mi4xNzQ3MDkyMTE0*_ga_47WX3HKKY2*czE3NjcyMjkwMTEkbzk1JGcwJHQxNzY3MjI5MDExJGo2MCRsMCRoMA..#langchain.agents.middleware.TodoListMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import TodoListMiddleware

agent = create_agent(
    model="gpt-4o",
    tools=[read_file, write_file, run_tests],
    middleware=[TodoListMiddleware()],
)
```

請觀看此[影片指南](https://www.youtube.com/watch?v=yTWocbVKQxw)，了解待辦事項清單中間件的行為。

![type:video](https://www.youtube.com/watch?v=yTWocbVKQxw)

??? example

    ```python
    import os
    import subprocess
    import tempfile

    from langchain.agents import create_agent
    from langchain.agents.middleware import TodoListMiddleware
    from langchain_core.tools import tool


    @tool(parse_docstring=True)
    def create_file(filename: str, content: str) -> str:
        """Create a new file with the given content.

        Args:
            filename: Name of the file to create.
            content: Content to write to the file.

        Returns:
            Confirmation message.
        """
        # Create in a temp directory for demo purposes
        temp_dir = tempfile.gettempdir()
        file_path = os.path.join(temp_dir, filename)
        with open(file_path, "w") as f:
            f.write(content)
        return f"Created {file_path} successfully"


    @tool(parse_docstring=True)
    def run_command(command: str) -> str:
        """Run a shell command and return the output.

        Args:
            command: Shell command to execute.

        Returns:
            Command output.
        """
        try:
            result = subprocess.run(
                command,
                shell=True,  # noqa: S602
                capture_output=True,
                text=True,
                timeout=10,
                cwd=tempfile.gettempdir(),
            )
            if result.returncode == 0:
                return f"Command succeeded:\n{result.stdout}"
            return f"Command failed (exit code {result.returncode}):\n{result.stderr}"
        except subprocess.TimeoutExpired:
            return "Command timed out after 10 seconds"
        except Exception as e:
            return f"Error running command: {e}"


    agent = create_agent(
        model="openai:gpt-4o",
        tools=[create_file, run_command],
        system_prompt="You are a software development assistant.",
        middleware=[TodoListMiddleware()],
    )
    ```


#### Configuration options

- **system_prompt** `string`
  
    自訂系統提示，用於指導待辦事項的使用。如果未指定，則使用內建提示。

- **tool_description** `string`
  
    為 `write_todos` 工具自訂描述。如果未指定，則使用內建描述。

