# Events and error handling

> LiveKit Agents 中的事件和錯誤處理的指南和參考。

## Events

`AgentSession` 發出事件來通知您狀態變更。每個事件都以事件物件作為其唯一參數發出。

### user_input_transcribed

當使用者語音轉錄成文字可用時，會發出 `UserInputTranscribedEvent`。

#### Properties

- `transcript`: str
- `is_final`: bool

#### Example

```python
from livekit.agents import UserInputTranscribedEvent

@session.on("user_input_transcribed")
def on_user_input_transcribed(event: UserInputTranscribedEvent):
    print(f"User input transcribed: {event.transcript}, final: {event.is_final}")

```

### conversation_item_added

當某項內容被提交到聊天歷史時發出。`ConversationItemAddedEvent` 事件針對使用者和代理發出。

#### Properties

- `item`: [ChatMessage](https://github.com/livekit/agents/blob/3ee369e7783a2588cffecc0725e582cac10efa39/livekit-agents/livekit/agents/llm/chat_context.py#L105)

#### Example

```python
from livekit.agents import ConversationItemAddedEvent
from livekit.agents.llm import ImageContent, AudioContent

...

@session.on("conversation_item_added")
def on_conversation_item_added(event: ConversationItemAddedEvent):
    print(f"Conversation item added from {event.item.role}: {event.item.text_content}. interrupted: {event.item.interrupted}")
    # to iterate over all types of content:
    for content in event.item.content:
        if isinstance(content, str):
            print(f" - text: {content}")
        elif isinstance(content, ImageContent):
            # image is either a rtc.VideoFrame or URL to the image
            print(f" - image: {content.image}")
        elif isinstance(content, AudioContent):
            # frame is a list[rtc.AudioFrame]
            print(f" - audio: {content.frame}, transcript: {content.transcript}")

```

### function_tools_executed

在針對給定的使用者輸入執行完所有功能工具後 `FunctionToolsExecutedEvent` 事件發出。

#### Methods

- `zipped()` 傳回函數呼叫及其輸出的元組列表。

#### Properties

- `function_calls`: list[[FunctionCall](https://github.com/livekit/agents/blob/3ee369e7783a2588cffecc0725e582cac10efa39/livekit-agents/livekit/agents/llm/chat_context.py#L129)]
- `function_call_outputs`: list[[FunctionCallOutput](https://github.com/livekit/agents/blob/3ee369e7783a2588cffecc0725e582cac10efa39/livekit-agents/livekit/agents/llm/chat_context.py#L137)]

### metrics_collected

當有新的指標可供報告時，會發出 `MetricsCollectedEvent`。有關指標的更多信息，請參閱[捕獲指標](./metrics.md)。

#### Properties

- `metrics`: Union[STTMetrics, LLMMetrics, TTSMetrics, VADMetrics, EOUMetrics]

### speech_created

當創建新代理語音時，會發出 `SpeechCreatedEvent`。Speech 可能因以下任一原因而產生：

- 使用者已提供輸入
- `session.say` 用於創建代理語音
- 呼叫 `session.generate_reply` 來建立回覆

#### Properties

- `user_initiated`: str - True 如果語音是使用 `say` 或 `generate_reply` 等公共方法創建的
- `source`: str - `say`, `generate_reply`, 或 `tool_response`
- `speech_handle`: [SpeechHandle](https://docs.livekit.io/agents/build/audio.md#speechhandle) - 處理追蹤語音播放。

### agent_state_changed

當代理程式的狀態改變時，會發出 `AgentStateChangedEvent`。代理參與者上的 `lk.agent.state` 屬性已更新以反映新狀態，從而允許前端程式碼輕鬆回應變更。

#### Properties

- `old_state`: AgentState
- `new_state`: AgentState

#### AgentState

代理可能處於以下狀態之一：

- `initializing` - 代理正在啟動。這應該很簡短。
- `listening` - 代理正在等待使用者輸入
- `thinking` - 代理正在處理使用者輸入
- `speaking` - 代理人正在講話

### user_state_changed

當使用者狀態改變時，會發出 `UserStateChangedEvent`。此變化是由在用戶音訊輸入上運行的 VAD 模組驅動的。

#### Properties

- `old_state`: UserState
- `new_state`: UserState

#### UserState

使用者的狀態可以是以下之一：

- `speaking` - VAD 偵測到使用者已開始講話
- `listening` - VAD 偵測到使用者已停止說話

### close

當 AgentSession 關閉且代理程式不再運作時，會發出 `CloseEvent`。出現這種情況的原因可能有多種：

- 用戶結束了對話
- 呼叫了 `session.aclose()`
- 房間被刪除，代理斷開連接
- 會話期間發生不可恢復的錯誤

#### Properties

- `error`: LLMError | STTError | TTSError | RealtimeModelError |None - 導致會話關閉的錯誤（如果適用）

## Handling errors

除了狀態變化之外，處理會話期間可能發生的錯誤也很重要。在即時對話中，推理 API 故障可能會中斷流程，這可能導致代理無法繼續。

### FallbackAdapter

對於 STT, LLM 和 TTS，Agents 框架包含一個 `FallbackAdapter`，如果主提供者發生故障，它可以回退到輔助提供者。


使用時，`FallbackAdapter` 將：

- 當主提供者發生故障時，自動將失敗的請求重新提交給備份供應商
- 將失敗的提供者標記為不健康並停止向其發送請求
- 繼續使用備用提供者，直到主提供者恢復
- 定期在背景檢查主要提供者的狀態

```python
from livekit.agents import llm, stt, tts
from livekit.plugins import assemblyai, deepgram, elevenlabs, openai, groq

session = AgentSession(
    stt=stt.FallbackAdapter(
        [
            assemblyai.STT(),
            deepgram.STT(),
        ]
    ),
    llm=llm.FallbackAdapter(
        [
            openai.LLM(model="gpt-4o"),
            openai.LLM.with_azure(model="gpt-4o", ...),
        ]
    ),
    tts=tts.FallbackAdapter(
        [
            elevenlabs.TTS(...),
            groq.TTS(...),
        ]
    ),
)

```

### Error event

當會話期間發生錯誤時，`AgentSession` 會發出 `ErrorEvent`。它包含一個 `error` 對象，其中有一個 `recoverable` 字段，指示會話是否將重試失敗的操作。


- 如果 `recoverable` 為 `True`，則該事件為資訊性事件，會話將按預期繼續。
- 如果 `recoverable` 為 `False` (例如，在用盡重試之後)，則會話需要介入。您可以處理錯誤 - 例如，透過使用 `.say（）` 通知使用者問題。

#### Properties

- `model_config`: dict - 表示目前模型配置的字典
- `error`: [LLMError | STTError | TTSError | RealtimeModelError](https://github.com/livekit/agents/blob/db551d2/livekit-agents/livekit/agents/voice/events.py#L138) - 發生的錯誤。 `recoverable` 是 `error` 中的一個欄位。
- `source`: LLM | STT | TTS | RealtimeModel - 導致錯誤的來源對象

### Example

- **[Error handling](https://github.com/livekit/agents/blob/main/examples/voice_agents/error_callback.py)**: 使用預先合成的訊息處理不可恢復的錯誤。
