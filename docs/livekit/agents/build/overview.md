# Building voice agents

> 使用 LiveKit Agents 進行構建語音 AI 代理的深入指南。

## Overview

建立出色的語音 AI 代理應用程式需要精心協調多個組件。 LiveKit Agents 建立在 [Realtime SDK](https://github.com/livekit/python-sdks) 之上，提供專用的抽象，簡化開發，同時讓您完全控制底層程式碼。


## Agent sessions

`AgentSession` 是語音 AI 代理應用的主要協調器。`AgentSession` 負責收集使用者輸入、管理語音管道、呼叫 LLM 並將輸出發送回使用者。

每個會話 (`session`) 至少需要一個 `Agent` 來協調。`Agent` 負責定義應用程式的核心 AI 邏輯 - 指令、工具等。此框架支援自訂工作流程的設計，以協調多個代理之間的交接和委派。

以下範例顯示如何開始簡單的單一代理會話：

```python
from livekit.agents import AgentSession, Agent, RoomInputOptions
from livekit.plugins import openai, cartesia, deepgram, noise_cancellation, silero
from livekit.plugins.turn_detector.multilingual import MultilingualModel

# 構建一個 AgentSession
session = AgentSession(
    stt=deepgram.STT(), # 定義 Sound to Text
    llm=openai.LLM(),   # 定義 LLM
    tts=cartesia.TTS(), # 定義 Text to Sound
    vad=silero.VAD.load(), # Voice Activity Detection (VAD)
    turn_detection=turn_detector.MultilingualModel(),
)

# 啟動 AgentSession 當有一個 participant 加入到 Livekit 的 room 時, 
# LK 會發出一個 dispatch request 來啟動一個 "Job" (worker 的 sub-process)
await session.start(
    room=ctx.room,
    agent=Agent(instructions="You are a helpful voice AI assistant."),
    room_input_options=RoomInputOptions(
        noise_cancellation=noise_cancellation.BVC(),
    ),
)

```

### RoomIO

代理和使用者參與者之間的溝通使用媒體串流（也稱為 `track`）進行。對於語音 AI 應用，這主要是音頻，但可以包括視覺。預設情況下，軌道管理由 `RoomIO` 處理，這是一個 utility 類別，可充當 agent session 和 LiveKit room 之間的橋樑。當啟動 `AgentSession` 時，它會自動建立一個 `RoomIO` 對象，使所有Livekit room 的參與者 (participants) 能夠訂閱可用的音軌。

要了解有關發布音訊和視訊的更多信息，請參閱以下主題:

- **[Agent speech and audio](https://docs.livekit.io/agents/build/audio.md)**: 為您的 AI 代理添加語音、音訊和背景音訊。

- **[Vision](https://docs.livekit.io/agents/build/vision.md)**: 讓您的代理能夠查看圖像和即時視訊。

- **[Text and transcription](https://docs.livekit.io/agents/build/text.md)**: 向您的代理發送和接收簡訊和轉錄。

- **[Realtime media](https://docs.livekit.io/home/client/tracks.md)**: Tracks 是 LiveKit 的核心概念。了解有關發布和訂閱媒體的更多資訊。

- **[Camera and microphone](https://docs.livekit.io/home/client/tracks/publish.md)**: 使用 LiveKit SDK 從用戶的裝置發布音訊和視訊軌道。

#### Custom RoomIO

為了更好地控制房間內的媒體共享，您可以建立自訂的 `RoomIO` 物件。例如，您可能希望手動控制使用哪些輸入和輸出設備，或控制代理監聽或回應哪些參與者。

若要替換在 `AgentSession` 中建立的預設對象，請在 entrypoint function 中建立一個 `RoomIO` 對象，並在建構函式中向其傳遞 `AgentSession` 的實例。有關範例，請參閱 GitHub 儲存庫中的以下內容：

- **[Toggling audio](https://github.com/livekit/agents/blob/main/examples/voice_agents/push_to_talk.py)**: 建立 push-to-talk 介面來切換音訊輸入和輸出。

- **[Toggling input and output](https://github.com/livekit/agents/blob/main/examples/voice_agents/toggle_io.py)**: 切換音訊和文字的輸入和輸出。

## Voice AI providers

您可以從語音管道(voice pipeline)每個部分的多個提供者中進行選擇，以滿足您的需求。該框架支援高性能 `STT-LLM-TTS` 管道和語音到語音模型。無論哪種情況，它都會自動管理中斷、轉錄轉發、語氣轉折檢測等。

您可以將這些元件新增至 `AgentSession` 中，它們在其中充當應用程式內的全域預設值，或根據需要新增至每個單獨的 `Agent`。

- **[TTS](https://docs.livekit.io/agents/integrations/tts.md)**: Text-to-speech 整合

- **[STT](https://docs.livekit.io/agents/integrations/stt.md)**: Speech-to-text 整合

- **[LLM](https://docs.livekit.io/agents/integrations/llm.md)**: Language model 整合

- **[Multimodal](https://docs.livekit.io/agents/integrations/realtime.md)**: Realtime multimodal APIs

## Capabilities

除了本節中的其他指南外，以下指南還介紹了 `AgentSession` 的核心功能以及如何在您的應用程式中利用它們。

- **[Workflows](https://docs.livekit.io/agents/build/workflows.md)**: 在多個代理之間協調複雜的任務。

- **[Tool definition & use](https://docs.livekit.io/agents/build/tools.md)**: 使用工具呼叫外部服務、注入自訂邏輯等。

- **[Pipeline nodes](https://docs.livekit.io/agents/build/nodes.md)**: 在語音管道的任何組件中添加自訂行為。
