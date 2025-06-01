# External data and RAG

> 增加情境和採取外部行動的最佳實踐。

## Overview

您的代理可以連接到外部資料來源來檢索資訊、儲存資料或採取其他操作。一般來說，您可以安裝任何 Python 套件或向代理程式添加自訂程式碼以使用您需要的任何資料庫或 API。

例如，您的代理可能需要：

- 在開始對話之前從資料庫載入使用者的個人資料資訊。
- 在私人知識庫中搜尋訊息，以準確回答使用者的查詢。
- 對資料庫或服務（例如日曆）執行讀取/寫入/更新操作。
- 將對話歷史或其他資料儲存到遠端伺服器。

本指南涵蓋作業初始化、檢索增強生成 (RAG)、工具呼叫以及將您的代理連接到外部資料來源和其他系統的其他技術的最佳實踐和技術。

## Initial context

預設情況下，每個 `AgentSession` 都以空白的聊天上下文開始。您可以在連接到房間並開始會話之前將使用者或任務特定的資料載入到代理的上下文中。例如，該代理程式根據 [job metadata](https://docs.livekit.io/agents/worker/job.md#metadata) 以姓名問候使用者。

```python
from livekit import agents
from livekit.agents import Agent, ChatContext, AgentSession

class Assistant(Agent):
    def __init__(self, chat_ctx: ChatContext) -> None:
        super().__init__(chat_ctx=chat_ctx, instructions="You are a helpful voice AI assistant.")


async def entrypoint(ctx: agents.JobContext):
    # Simple lookup, but you could use a database or API here if needed
    metadata = json.loads(ctx.job.metadata)
    user_name = metadata["user_name"]

    await ctx.connect()

    session = AgentSession(
        # ... stt, llm, tts, vad, turn_detection, etc.
    )
    
    initial_ctx = ChatContext()
    initial_ctx.add_message(role="assistant", content=f"The user's name is {user_name}.")

    await session.start(
        room=ctx.room,
        agent=Assistant(chat_ctx=initial_ctx),
        # ... room_input_options, etc.
    )

    await session.generate_reply(
        instructions="Greet the user by name and offer your assistance."
    )

```

!!! tip
    💡 **Load time optimizations**

    如果您的代理需要外部資料才能啟動，以下提示可以幫助最大限度地減少對使用者體驗的影響：

    1. 對於靜態資料（非使用者特定），請在 [prewarm 函數](https://docs.livekit.io/agents/worker/options.md#prewarm) 中載入。

    2. 在 [job metadata](https://docs.livekit.io/agents/worker/job.md#metadata)、[room metadata](https://docs.livekit.io/home/client/state/room-metadata.md) 或 [參與者屬性](https://docs.livekit.io/state/client-metadata.md) 或 [participant attributes](https://docs.livekit.io/home/client/state/participant-attributes/)中發送用戶特定數據，而不是在入口點加載它。

    3. 如果必須在入口點載入網路調用，請在 `ctx.connect()`之前執行此操作。這可確保您的前端在收聽傳入音訊之前不會顯示代理參與者。


## Tool calls

為了達到最高精度或採取外部行動，您可以為 LLM 提供一些 [工具](https://docs.livekit.io/agents/build/tools.md) 供其在回復中使用。這些工具可以是通用的，也可以是特定的，以滿足您的用例的需要。

例如，定義`search_calendar`, `create_event`, `update_event` 和 `delete_event` 工具，讓 LLM 完全存取使用者的日曆。使用 [participant attributes](https://docs.livekit.io/home/client/state/participant-attributes.md) 或 [job metadata](https://docs.livekit.io/agents/worker/job.md#metadata) 將使用者的日曆 ID 和存取權令牌傳遞給代理。

- **[Tool definition and use](https://docs.livekit.io/agents/build/tools.md)**: 在 LiveKit Agents 中定義和使用自訂工具的指南。

## Add context during conversation

您可以使用 [on_user_turn_completed node](https://docs.livekit.io/agents/build/nodes.md#on_user_turn_completed) 根據使用者最近的回合執行 RAG 查找，然後 LLM 產生回應。這種方法效能很高，因為它避免了工具呼叫中涉及的額外往返，但它僅適用於可以以文字形式存取使用者輪次的 STT-LLM-TTS 管道。此外，結果的好壞取決於您實施的搜尋功能的準確性。

例如，您可以使用向量搜尋來檢索與使用者查詢相關的附加上下文，並將其注入到下一代 LLM 的聊天上下文中。這是一個簡單的例子：

```python
from livekit.agents import ChatContext, ChatMessage

async def on_user_turn_completed(
    self, turn_ctx: ChatContext, new_message: ChatMessage,
) -> None:
    # RAG function definition omitted for brevity
    rag_content = await my_rag_lookup(new_message.text_content())
    turn_ctx.add_message(
        role="assistant", 
        content=f"Additional information relevant to the user's next message: {rag_content}"
    )

```

## User feedback

向使用者提供有關狀態更新的直接回饋非常重要 - 例如，解釋延遲或失敗。以下是一些範例用例：

- 當某個操作需要幾百毫秒以上時。
- 執行寫入操作（例如傳送電子郵件或排程會議）時。
- 當代理無法執行操作時。

以下部分描述了向使用者提供此回饋的各種技術。

### Verbal status updates

使用 [Agent speech](https://docs.livekit.io/agents/build/speech.md) 在長時間運行的工具呼叫或其他操作期間向使用者提供口頭回饋。

在以下範例中，只有當通話時間超過指定的逾時時間時，代理才會說出狀態更新。更新是根據查詢動態生成的，並且可以擴展以包括剩餘時間或其他資訊的估計值。

```python
import asyncio
from livekit.agents import function_tool, RunContext

@function_tool()
async def search_knowledge_base(
    self,
    context: RunContext,
    query: str,
) -> str:
    # Send a verbal status update to the user after a short delay
    async def _speak_status_update(delay: float = 0.5):
        await asyncio.sleep(delay)
        await context.session.generate_reply(instructions=f"""
            You are searching the knowledge base for \"{query}\" but it is taking a little while.
            Update the user on your progress, but be very brief.
        """)
    
    status_update_task = asyncio.create_task(_speak_status_update(0.5))

    # Perform search (function definition omitted for brevity)
    result = await _perform_search(query)
    
    # Cancel status update if search completed before timeout
    status_update_task.cancel()
    
    return result

```

有關詳細信息，請參閱以下文章：

- **[Agent speech](https://docs.livekit.io/agents/build/speech.md)**: 探索 LiveKit Agents 的語音功能和特性。

### "Thinking" sounds

新增 [background audio](https://docs.livekit.io/agents/build/audio.md#background-audio)，在工具呼叫過程中自動播放 "thinking" 的聲音。這有助於讓代理的回應感覺更自然。

```python
from livekit.agents import BackgroundAudioPlayer, AudioConfig, BuiltinAudioClip

async def entrypoint(ctx: agents.JobContext):
    await ctx.connect()

    session = AgentSession(
        # ... stt, llm, tts, vad, turn_detection, etc.
    )

    await session.start(
        room=ctx.room,
        # ... agent, etc.
    )

    background_audio = BackgroundAudioPlayer(
        thinking_sound=[
            AudioConfig(BuiltinAudioClip.KEYBOARD_TYPING, volume=0.8),
            AudioConfig(BuiltinAudioClip.KEYBOARD_TYPING2, volume=0.7),
        ],
    )
    await background_audio.start(room=ctx.room, agent_session=session)

```

### Frontend UI

如果您的應用程式包含前端，您可以新增自訂 UI 來表示代理程式操作的狀態。例如，顯示一個長時間運行的操作的彈出窗口，使用者可以選擇取消：

```python
from livekit.agents import get_job_context
import json
import asyncio

@function_tool()
async def perform_deep_search(
    self,
    context: RunContext,
    summary: str,
    query: str,
) -> str:
    """
    Initiate a deep internet search that will reference many external sources to answer the given query. This may take 1-5 minutes to complete.

    Summary: A user-friendly summary of the query
    Query: the full query to be answered
    """
    async def _notify_frontend(query: str):
        room = get_job_context().room
        response = await room.local_participant.perform_rpc(
            destination_identity=next(iter(room.remote_participants)),
            # frontend method that shows a cancellable popup
            # (method definition omitted for brevity, see RPC docs)
            method='start_deep_search',
            payload=json.dumps({
                "summary": summary,
                "estimated_completion_time": 300,
            }),
            # Allow the frontend a long time to return a response
            response_timeout=500,
        )
        # In this example the frontend has a Cancel button that returns "cancelled"
        # to stop the task
        if response == "cancelled":
            deep_search_task.cancel()

    notify_frontend_task = asyncio.create_task(_notify_frontend(query))

    # Perform deep search (function definition omitted for brevity)
    deep_search_task = asyncio.create_task(_perform_deep_search(query))

    try:
        result = await deep_search_task
    except asyncio.CancelledError:
        result = "Search cancelled by user"
    finally:
        notify_frontend_task.cancel()
        return result

```

有關更多資訊和範例，請參閱以下文章：

- **[Web and mobile frontends](https://docs.livekit.io/agents/start/frontend.md)**: 為您的代理程式建立自訂 Web 或行動前端的指南。

- **[RPC](https://docs.livekit.io/home/client/data/rpc.md)**: 了解如何使用 RPC 從前端與您的代理進行通訊。

## Fine-tuned models

有時，獲得最相關結果的最佳方法是針對您的特定用例微調模型。您可以探索可用的 [LLM 整合](https://docs.livekit.io/agents/integrations/llm.md) 來尋找支援微調的提供程序，或使用 [Ollama](https://docs.livekit.io/agents/integrations/llm/ollama.md) 來整合自訂模型。

## RAG providers and services

您可以與您選擇的任何 RAG 提供者或工具集成，以透過附加上下文增強您的代理。建議的提供者和工具包括：

- [LlamaIndex](https://www.llamaindex.ai/) - 將自訂資料連接到 LLM 的框架。
- [Mem0](https://mem0.ai) - 人工智慧助理的記憶層。
- [TurboPuffer](https://turbopuffer.com/) - 基於物件儲存構建的快速無伺服器向量搜尋。
- [Pinecone](https://pinecone.io/) - 用於 AI 應用的管理向量資料庫。
- [Annoy](https://github.com/spotify/annoy) - Spotify 的開源 Python 庫，用於最近鄰搜尋。

## Additional examples

以下範例展示如何實現 RAG 和其他技術：

- **[LlamaIndex RAG](https://github.com/livekit/agents/tree/main/examples/voice_agents/llamaindex-rag)**: 使用 LlamaIndex 為 RAG 提供語音 AI 代理來回答知識庫中的問題。

- **[LiveKit Docs RAG](https://github.com/livekit-examples/python-agents-examples/tree/main/rag)**: 可以透過查閱文件網站來回答有關 LiveKit 的問題的代理。