# Tool definition and use

> 讓您的代理程式呼叫外部工具等等。

## Overview

LiveKit Agents 完全支援 LLM 工具的使用。此功能可讓您建立自訂工具庫來擴展代理程式的上下文、建立互動式體驗並克服 LLM 限制。在工具中，您可以：

- 使用 `session.say()` 或 `session.generate_reply()` 產生 [代理語音](https://docs.livekit.io/agents/build/audio.md)。
- 使用 [RPC](https://docs.livekit.io/home/client/data/rpc.md) 呼叫前端的方法。
- 作為[工作流程](https://docs.livekit.io/agents/build/workflows.md)的一部分，將控制權移交給另一個代理程式。
- 從 `context` 儲存和檢索會話資料。
- Python 函數可以做的任何事情。
- [呼叫外部 API 或尋找 RAG 的資料](https://docs.livekit.io/agents/build/external-data.md)。
## Tool definition

使用 `@function_tool` 裝飾器為您的代理類別新增工具。 LLM 可以自動存取它們。

```python
from livekit.agents import function_tool, Agent, RunContext


class MyAgent(Agent):
    @function_tool()
    async def lookup_weather(
        self,
        context: RunContext,
        location: str,
    ) -> dict[str, Any]:
        """Look up weather information for a given location.
        
        Args:
            location: The location to look up weather information for.
        """

        return {"weather": "sunny", "temperature_f": 70}

```

!!! tip
    **Best practices**

    良好的工具定義 (tool definition) 是可靠使用 LLM 工具的關鍵。具體說明該工具的功能、何時應該使用或不應該使用、參數的用途以及預期的回傳值類型。


### Name and description

預設情況下，工具名稱是函數的名稱，描述是其 docstring。使用 `@function_tool` 裝飾器的 `name` 和 `description` 參數覆寫此行為。

### Arguments and return value

工具參數會根據名稱從函數參數中自動複製。如果存在，則包含參數和傳回值的類型提示。

如果需要，請在工具描述中放置有關工具參數和傳回值的附加資訊。

### RunContext

工具包括對特殊 `context` 參數的支援。這包含對目前`context`, `function_call`, `speech_handle` 和 `userdata` 的存取。有關如何使用這些功能的更多信息，請參閱有關 [speech]（https://docs.livekit.io/agents/build/audio.md）和 [state within workflows](https://docs.livekit.io/agents/build/workflows.md) 的文檔。

### Adding tools dynamically

您可以透過直接設定 `tools` 參數來對可用的工具進行更多的控制。

若要在多個代理程式之間共用工具，請在類別之外定義它，然後將其提供給每個代理程式。`RunContext` 對於存取目前會話、代理程式和狀態特別有用。

`tools` 值中設定的工具可與使用 `@function_tool` 裝飾器在類別中註冊的任何工具一起使用。

```python
from livekit.agents import function_tool, Agent, RunContext

@function_tool()
async def lookup_user(
    context: RunContext,
    user_id: str,
) -> dict:
    """Look up a user's information by ID."""

    return {"name": "John Doe", "email": "john.doe@example.com"}


class AgentA(Agent):
    def __init__(self):
        super().__init__(
            tools=[lookup_user],
            # ...
        )


class AgentB(Agent):
    def __init__(self):
        super().__init__(
            tools=[lookup_user],
            # ...
        )

```

建立代理程式後，使用 `agent.update_tools()` 更新可用工具。這將取代所有工具，包括代理類別中自動註冊的工具。若要在替換之前引用現有工具，請造訪 `agent.tools` 屬性：

```python
# add a tool
agent.update_tools(agent.tools + [tool_a]) 

# remove a tool
agent.update_tools(agent.tools - [tool_a]) 

# replace all tools
agent.update_tools([tool_a, tool_b]) 

```

### Creating tools programmatically

若要動態建立工具，請使用 `function_tool` 作為函數而不是裝飾器。您必須提供名稱、描述和可呼叫函數。這對於基於相同底層程式碼編寫特定工具或從外部來源（例如資料庫或模型上下文協定 (MCP) 伺服器）載入它們很有用。

在以下範例中，該應用程式具有設定任何使用者設定檔欄位的單一功能，但為每個欄位向代理程式提供一個工具，以提高可靠性：


```python
from livekit.agents import function_tool, RunContext

class Assistant(Agent):
    def _set_profile_field_func_for(self, field: str):
        async def set_value(context: RunContext, value: str):
            # custom logic to set input
            return f"field {field} was set to {value}"

        return set_value

    def __init__(self):
        super().__init__(
            tools=[
                function_tool(self._set_profile_field_func_for("phone"),
                              name="set_phone_number",
                              description="Call this function when user has provided their phone number."),
                function_tool(self._set_profile_field_func_for("email"),
                              name="set_email",
                              description="Call this function when user has provided their email."),
                # ... other tools ...
            ],
            # instructions, etc ...
        )

```

## Error handling

引發 `ToolError` 異常以將錯誤傳回給 LLM 而不是回應。您可以包含自訂訊息來描述錯誤和/或復原選項。

```python
@function_tool()
async def lookup_weather(
    self,
    context: RunContext,
    location: str,
) -> dict[str, Any]:
    if location == "mars":
        raise ToolError("This location is coming soon. Please join our mailing list to stay updated.")
    else:
        return {"weather": "sunny", "temperature_f": 70}

```

## Model Context Protocol (MCP)

LiveKit Agents 完全支援 [MCP](https://modelcontextprotocol.io/) 伺服器從外部來源載入工具。

要使用它，首先安裝 `mcp` 可選依賴項：

```shell
pip install livekit-agents[mcp]~=1.0

```

然後將 MCP 伺服器 URL 傳遞給 `AgentSession` 或 `Agent` 建構子。該工具將像任何其他工具一樣自動加載。

```python
from livekit.agents import mcp

session = AgentSession(
    #... other arguments ...
    mcp_servers=[
        mcp.MCPServerHTTP(
            "https://your-mcp-server.com"
        )       
    ]       
)

```

```python
from livekit.agents import mcp

agent = Agent(
    #... other arguments ...
    mcp_servers=[
        mcp.MCPServerHTTP(
            "https://your-mcp-server.com"
        )       
    ]       
)

```

## Forwarding to the frontend

使用 [RPC](https://docs.livekit.io/home/client/data/rpc.md) 將工具呼叫轉送到前端應用程式。當完成函數呼叫所需的資料僅在前端可用時，這很有用。您也可以使用 RPC 以結構化的方式觸發操作或 UI 更新。

例如，這裡有一個從網頁瀏覽器存取用戶即時位置的功能：

### Agent implementation

```python
from livekit.agents import function_tool, get_job_context, RunContext

@function_tool()
async def get_user_location(
    context: RunContext,
    high_accuracy: bool
):
    """Retrieve the user's current geolocation as lat/lng.
    
    Args:
        high_accuracy: Whether to use high accuracy mode, which is slower but more precise
    
    Returns:
        A dictionary containing latitude and longitude coordinates
    """
    try:
        room = get_job_context().room
        participant_identity = next(iter(room.remote_participants))
        response = await room.local_participant.perform_rpc(
            destination_identity=participant_identity,
            method="getUserLocation",
            payload=json.dumps({
                "highAccuracy": high_accuracy
            }),
            response_timeout=10.0 if high_accuracy else 5.0,
        )
        return response
    except Exception:
        raise ToolError("Unable to retrieve user location")

```

### Frontend implementation

以下範例使用 JavaScript SDK。相同模式也適用於其他 SDK。如需更多範例，請參閱 [RPC 文件](https://docs.livekit.io/home/client/data/rpc.md)。

```typescript
import { RpcError, RpcInvocationData } from 'livekit-client';

localParticipant.registerRpcMethod(
    'getUserLocation',
    async (data: RpcInvocationData) => {
        try {
            let params = JSON.parse(data.payload);
            const position: GeolocationPosition = await new Promise((resolve, reject) => {
                navigator.geolocation.getCurrentPosition(resolve, reject, {
                    enableHighAccuracy: params.highAccuracy ?? false,
                    timeout: data.responseTimeout,
                });
            });

            return JSON.stringify({
                latitude: position.coords.latitude,
                longitude: position.coords.longitude,
            });
        } catch (error) {
            throw new RpcError(1, "Could not retrieve user location");
        }
    }
);

```

## Slow tool calls

有關在長時間運行的工具呼叫期間向使用者提供回饋的最佳實踐，請參閱外部資料和 RAG 指南中的 [user feedback](https://docs.livekit.io/agents/build/external-data.md#user-feedback) 部分。

## External tools and MCP

若要從外部來源載入工具作為模型上下文協定 (MCP) 伺服器，請使用 `function_tool` 函數並使用 `tools` 屬性或 `update_tools()` 方法註冊工具。有關完整的 MCP 實現，請參閱以下範例：

- **[MCP Agent](https://github.com/livekit-examples/python-agents-examples/tree/main/mcp)**: 具有用於 LiveKit API 的整合模型上下文協定 (MCP) 伺服器的語音 AI 代理程式。

## Examples

以下附加範例展示如何以不同的方式使用工具：

- **[Use of enum](https://github.com/livekit/agents/blob/main/examples/voice_agents/annotated_tool_args.py)**: 展示如何使用枚舉註解參數的範例。

- **[Dynamic tool creation](https://github.com/livekit/agents/blob/main/examples/voice_agents/dynamic_tool_creation.py)**: 帶有動態工具清單的完整範例。

- **[MCP Agent](https://github.com/livekit-examples/python-agents-examples/tree/main/mcp)**: 具有用於 LiveKit API 的整合模型上下文協定 (MCP) 伺服器的語音 AI 代理程式。

- **[LiveKit Docs RAG](https://github.com/livekit-examples/python-agents-examples/tree/main/rag)**: 可以透過查閱文件網站來回答有關 LiveKit 的問題的代理。

## Further reading

以下文章提供了有關本指南中討論的主題的更多資訊：

- **[RPC](https://docs.livekit.io/home/client/data/rpc.md)**: 完成有關 LiveKit 參與者之間的函數呼叫的文件。

- **[Agent speech](https://docs.livekit.io/agents/build/audio.md)**: 有關精確控制代理語音輸出的詳細資訊。

- **[Workflows](https://docs.livekit.io/agents/build/workflows.md)**: 閱讀有關將控制權移交給其他代理商的更多資訊。

- **[External data and RAG](https://docs.livekit.io/agents/build/external-data.md)**: 添加情境和採取外部行動的最佳實踐。