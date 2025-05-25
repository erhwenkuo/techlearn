# Workflows

> 如何使用多個代理來建模可重複、準確的任務。

## Overview

代理可以在單一會話內組合，以高度可靠的方式模擬複雜任務。一些有用的特定場景包括：

- 通話開始時取得錄音同意。
- 收集特定的結構化訊息，例如地址或信用卡號。
- 逐一回答一連串問題。
- 當用戶不在時留下語音郵件訊息。
- 在單一會話中包含具有獨特特徵的多個角色。

## Defining an agent

擴展 `Agent` 類別來定義自訂代理程式。

```python
from livekit.agents import Agent

class HelpfulAssistant(Agent):
    def __init__(self):
        super().__init__(instructions="You are a helpful voice AI assistant.")

    async def on_enter(self) -> None:
        await self.session.say("Hello, how can I help you today?")

```

您也可以直接建立 `Agent` 類別的實例：

```python
agent = Agent(instructions="You are a helpful voice AI assistant.")

```

## Handing off control to another agent

從工具呼叫 (tool call) 中傳回不同的代理來移交控制。這使得 LLM 可以決定何時進行交接。

```python
from livekit.agents import Agent, function_tool, get_job_context

class ConsentCollector(Agent):
    def __init__(self):
        super().__init__(
            instructions="""Your are a voice AI agent with the singular task to collect positive 
            recording consent from the user. If consent is not given, you must end the call."""
        )

    async def on_enter(self) -> None:
        await self.session.say("May I record this call for quality assurance purposes?")

    @function_tool()
    async def on_consent_given(self):
        """Use this tool to indicate that consent has been given and the call may proceed."""

        # Perform a handoff, immediately transfering control to the new agent
        return HelpfulAssistant()

    @function_tool()
    async def end_call(self) -> None:
        """Use this tool to indicate that consent has not been given and the call should end."""
        await self.session.say("Thank you for your time, have a wonderful day.")
        job_ctx = get_job_context()
        await job_ctx.api.room.delete_room(api.DeleteRoomRequest(room=job_ctx.room.name))

```

## Context preservation

預設情況下，每個新代理程式都會以全新的對話歷史記錄開始其 LLM 提示。若要包含先前的對話，請在 `Agent` 建構函式中設定 `chat_ctx` 參數。您可以複製先前代理的 `chat_ctx`，也可以基於自訂業務邏輯建立新的代理來提供適當的上下文。

```python
from livekit.agents import ChatContext, function_tool, Agent

class HelpfulAssistant(Agent):
    def __init__(self, chat_ctx: ChatContext):
        super().__init__(
            instructions="You are a helpful voice AI assistant.",
            chat_ctx=chat_ctx
        )

class ConsentCollector(Agent):
    # ...

    @function_tool()
    async def on_consent_given(self):
        """Use this tool to indicate that consent has been given and the call may proceed."""

        # Pass the chat context during handoff
        return HelpfulAssistant(chat_ctx=self.session.chat_ctx)

```

會話的完整對話歷史記錄始終在 `session.history` 中可用。

## Passing state

若要在會話中儲存自訂狀態，請使用 `userdata` 屬性。使用者資料的類型由您決定，但建議的方法是使用 `dataclass`。

```python
from livekit.agents import AgentSession
from dataclasses import dataclass

@dataclass
class MySessionInfo:
    user_name: str | None = None
    age: int | None = None

```

若要將使用者資料新增至會話，請將其傳遞到建構函式中。您也必須在 `AgentSession` 本身上指定使用者資料類型。

```python
session = AgentSession[MySessionInfo](
    userdata=MySessionInfo(),
    # ... tts, stt, llm, etc.
)

```

Userdata 可以作為 `session.userdata` 使用，也可以在 `RunContext` 上的功能工具中使用。以下範例顯示如何在以 `IntakeAgent` 開始的代理程式工作流程中使用 userdata。

```python
class IntakeAgent(Agent):
    def __init__(self):
        super().__init__(
            instructions="""Your are an intake agent. Learn the user's name and age."""
        )
        
    @function_tool()
    async def record_name(self, context: RunContext[MySessionInfo], name: str):
        """Use this tool to record the user's name."""
        context.userdata.user_name = name
        return self._handoff_if_done()
    
    @function_tool()
    async def record_age(self, context: RunContext[MySessionInfo], age: int):
        """Use this tool to record the user's age."""
        context.userdata.age = age
        return self._handoff_if_done()
    
    def _handoff_if_done(self):
        if self.session.userdata.user_name and self.session.userdata.age:
            return HelpfulAssistant()
        else:
            return None

class HelpfulAssistant(Agent):
    def __init__(self):
        super().__init__(instructions="You are a helpful voice AI assistant.")

    async def on_enter(self) -> None:
        userdata: MySessionInfo = self.session.userdata
        await self.session.generate_reply(
            instructions=f"Greet {userdata.user_name} and tell them a joke about being {userdata.age} years old."
        )

```

## Overriding plugins

您可以透過在 `Agent` 建構函式中設定對應的屬性來覆寫會話中使用的任何外掛程式。例如，您可以透過覆寫 `tts` 屬性來變更特定代理程式的聲音：

```python
from livekit.agents import Agent
from livekit.plugins import cartesia

class AssistantManager(Agent):
    def __init__(self):
        super().__init__(
            instructions="You are manager behind a team of helpful voice assistants.",
            tts=cartesia.TTS(voice="6f84f4b8-58a2-430c-8c79-688dad597532")
        )

```

## Examples

這些範例展示如何使用多個代理程式來建立更複雜的工作流程:

- **[Medical Office Triage](https://github.com/livekit-examples/python-agents-examples/tree/main/complex-agents/medical_office_triage)**: 根據症狀和病史對患者進行分類的代理。

- **[Restaurant Agent](https://github.com/livekit/agents/blob/main/examples/voice_agents/restaurant_agent.py)**: 餐廳前台代理可以接受訂單、將商品添加到共享購物車並結帳。

## Further reading

有關本文中涉及的概念的更多信息，請參閱以下相關文章:

- **[Tool definition and use](https://docs.livekit.io/agents/build/tools.md)**: 有關在代理程式中定義和使用工具的完整指南。

- **[Nodes](https://docs.livekit.io/agents/build/nodes.md)**: 在語音管道的任何組件中添加自訂行為。

- **[Agent speech](https://docs.livekit.io/agents/build/audio.md)**: 自訂代理的語音輸出。
