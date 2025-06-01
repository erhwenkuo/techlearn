
# Turn detection and interruptions

> 語音 AI 對話輪次管理指南。

## Overview

對話輪次 (turn detection ) 偵測是確定使用者何時開始或結束對話「輪流(turn)」的過程。這讓代理知道何時開始傾聽以及何時做出回應。

大多數對話輪次偵測技術依靠語音活動偵測 (VAD) 來偵測使用者輸入中的靜默期。代理將啟發式方法應用於 VAD 資料來執行短語端點檢測，從而確定句子或想法的結束。代理可以單獨使用端點或應用更多上下文分析來確定對話輪次何時完成。

有效的對話輪次偵測和中斷管理對於出色的語音 AI 體驗至關重要。

## Turn detection

除了始終可用的手動對話輪次控制外，`AgentSession` 還支援以下對話輪次偵測模式。

- **Turn detector model**: 用於在 VAD 或 STT 端點資料之上進行情境感知對話輪次偵測的自訂開放權重模型。
- **Realtime models**: 支援即時模型（如 OpenAI Realtime API）中的內建對話輪次偵測或 VAD。
- **VAD only**: 僅從語音和靜音資料偵測對話輪次的結束。
- **STT endpointing**: 使用您選擇的 STT 提供者傳回的即時 STT 資料中的短語端點來取代 VAD。
- **Manual turn control**: 完全停用自動對話輪次偵測。

### Turn detector model

為了實現代理的推薦行為，即在用戶說話時傾聽並在用戶完成思考後回覆，請在 STT-LLM-TTS 管道中使用以下插件：

- **[Turn detection model](https://docs.livekit.io/agents/build/turns/turn-detector.md)**: 用於情境感知轉彎偵測的開放權重模型。

- **[Silero VAD](https://docs.livekit.io/agents/build/turns/vad.md)**: 用於語音活動偵測的 Silero VAD 模型。

```python
from livekit.plugins.turn_detector.multilingual import MultilingualModel
from livekit.plugins import silero

session = AgentSession(
    turn_detection=MultilingualModel(), # or EnglishModel()
    vad=silero.VAD.load(),
    # ... stt, tts, llm, etc.
)

```

有關完整範例，請參閱 [Voice AI 快速入門](https://docs.livekit.io/agents/start/voice-ai.md)。

!!! info
    **Realtime model turn detection**

    對於即時模型，LiveKit 建議使用所選模型提供者的 [chosen model provider](https://docs.livekit.io/agents/integrations/realtime.md)。這是最具成本效益的選擇，因為自訂轉彎偵測模型需要單獨運行的即時語音轉文字 (STT)。

### Realtime models

即時模型包括基於 VAD 和其他技術的內建轉彎偵測選項。保持 `turn_detection` 參數未設定並直接配置即時模型的轉彎偵測選項。

要將 LiveKit 對話輪次模型與即時模型一起使用，您還必須提供 STT 插件。轉彎檢測器模型在 STT 輸出上運行。

- **[OpenAI Realtime API turn detection](https://docs.livekit.io/agents/integrations/realtime/openai.md#turn-detection)**: OpenAI Realtime API 的對話輪次偵測選項。

- **[Gemini Live API turn detection](https://docs.livekit.io/agents/integrations/realtime/gemini.md#turn-detection)**: 轉向 Gemini Live API 的對話輪次偵測選項。

### VAD only

在某些情況下，VAD 是對話輪次檢測的最佳選擇。例如，VAD 適用於任何口語。若要單獨使用 VAD，請使用 Silero VAD 外掛程式並設定`turn_detection="vad"`。

```python
session = AgentSession(
    turn_detection="vad",
    vad=silero.VAD.load(),
    # ... stt, tts, llm, etc.
)

```

### STT endpointing

您也可以使用 STT 模型進行轉彎檢測，因為它們處理音訊並執行短語端點來建立語音片段。在這種模式下，`AgentSession` 將最終的 STT 記錄視為轉彎邊界。

請注意，STT 端點對中斷的回應不如 VAD 快。

```python
session = AgentSession(
    turn_detection="stt",
    stt=deepgram.STT(),
    # ... tts, llm, etc.
)

```

### Manual turn control

透過在 `AgentSession` 建構函式中設定 `turn_detection="manual"` 來完全停用自動對話輪次偵測。

現在您可以使用`session.interrupt()`,`session.clear_user_turn()` 和 `session.commit_user_turn()` 方法來控制使用者的對話輪次。

例如，您可以使用它來實作一鍵通介面。以下是一個使用前端可以呼叫的 [RPC](https://docs.livekit.io/home/client/data/rpc.md) 方法的簡單範例：

```python
session = AgentSession(
    turn_detection="manual",
    # ... stt, tts, llm, etc.
)

# Disable audio input at the start
session.input.set_audio_enabled(False)

# When user starts speaking
@ctx.room.local_participant.register_rpc_method("start_turn")
async def start_turn(data: rtc.RpcInvocationData):
    session.interrupt()  # Stop any current agent speech
    session.clear_user_turn()  # Clear any previous input
    session.input.set_audio_enabled(True)  # Start listening

# When user finishes speaking
@ctx.room.local_participant.register_rpc_method("end_turn")
async def end_turn(data: rtc.RpcInvocationData):
    session.input.set_audio_enabled(False)  # Stop listening
    session.commit_user_turn()  # Process the input and generate response

# When user cancels their turn
@ctx.room.local_participant.register_rpc_method("cancel_turn")
async def cancel_turn(data: rtc.RpcInvocationData):
    session.input.set_audio_enabled(False)  # Stop listening
    session.clear_user_turn()  # Discard the input

```

這裡有一個更完整的範例：

- **[Push-to-Talk Agent](https://github.com/livekit/agents/blob/main/examples/voice_agents/push_to_talk.py)**: 一種語音 AI 代理，使用一鍵通功能進行受控的多參與者對話，僅在明確觸發時才啟用音訊輸入。

### Reducing background noise

[增強型噪音消除功能](https://docs.livekit.io/home/cloud/noise-cancellation.md) 可在 LiveKit Cloud 中使用，並可提高語音 AI 應用的轉彎偵測和語音轉文字 (STT) 的品質。您可以在啟動代理會話時將其新增至 `room_input_options`，從而為您的代理程式新增背景雜訊和語音消除功能。要了解如何啟用它，請參閱 [Voice AI 快速入門](https://docs.livekit.io/agents/start/voice-ai.md)。

## Interruptions

使用者可以隨時打斷代理，可以透過自動轉彎檢測或透過 `session.interrupt()` 方法進行說話。當中斷發生時，代理會停止講話並自動截斷其對話歷史記錄，以僅反映用戶在中斷之前實際聽到的講話。

## Session configuration

`AgentSession` 建構函式中提供了以下與轉彎偵測和中斷相關的參數：

- **`allow_interruptions`** _(bool)_ (optional) - Default: `True`: 是否允許用戶在中間打斷代理。當使用內建轉彎檢測的即時模型時被忽略。

- **`min_interruption_duration`** _(float)_ (optional) - Default: `0.5`: 觸發中斷之前偵測到的最短語音持續時間。

- **`min_endpointing_delay`** _(float)_ (optional) - Default: `0.5`: 視為轉彎完成前等待的秒數。當不存在轉彎偵測器模型或模型指示可能的轉彎邊界時，會話將使用此延遲。

- **`max_endpointing_delay`** _(float)_ (optional) - Default: `6.0`: 轉彎偵測器模型指示使用者可能繼續說話後等待使用者說話的最長時間。如果沒有轉彎偵測器模型，此參數則不起作用。

## Turn-taking events

`AgentSession` 公開使用者和代理狀態事件來監視對話流程：

```python
from livekit.agents import UserStateChangedEvent, AgentStateChangedEvent

@session.on("user_state_changed")
def on_user_state_changed(ev: UserStateChangedEvent):
    if ev.new_state == "speaking":
        print("User started speaking")
    elif ev.new_state == "listening":
        print("User stopped speaking")
    elif ev.new_state == "away":
        print("User is not present (e.g. disconnected)")

@session.on("agent_state_changed")
def on_agent_state_changed(ev: AgentStateChangedEvent):
    if ev.new_state == "initializing":
        print("Agent is starting up")
    elif ev.new_state == "idle":
        print("Agent is ready but not processing")
    elif ev.new_state == "listening":
        print("Agent is listening for user input")
    elif ev.new_state == "thinking":
        print("Agent is processing user input and generating a response")
    elif ev.new_state == "speaking":
        print("Agent started speaking")

```

## Further reading

- **[Agent speech](https://docs.livekit.io/agents/build/audio.md)**: 代理講話及相關方法的指南。

- **[Pipeline nodes](https://docs.livekit.io/agents/build/nodes.md)**: 監控流經語音管道的輸入和輸出。
