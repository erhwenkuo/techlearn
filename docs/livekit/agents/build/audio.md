# Agent speech and audio

> LiveKit 代理程式的語音和音訊功能。

## Overview

語音功能是 LiveKit 代理的核心功能，使他們能夠透過語音與使用者互動。本指南涵蓋了代理可用的各種語音特性和功能。

LiveKit Agents 提供了一個統一的介面，用於使用 `STT-LLM-TTS` 管道和即時模型來控制代理。

如欲了解更多資訊並查看使用範例，請參閱以下主題：

- **[Text-to-speech (TTS)](https://docs.livekit.io/agents/integrations/tts.md)**: `TTS` 是一種將文字轉換為音訊的合成過程，為 AI 代理提供「聲音」。

- **[Speech-to-speech](https://docs.livekit.io/agents/integrations/realtime.md)**: 多模態即時 API 可以理解語音輸入並直接產生語音輸出。

## Initiating speech

預設情況下，代理在回應之前等待使用者輸入 - 代理框架自動處理回應產生。

但在某些情況下，代理可能需要發起對話。例如，它可能在會話開始時向用戶打招呼，或者在一段時間的沉默後啟動新的 session。

### session.say

要讓代理程式說出預先定義的訊息，請使用 `session.say()`。這會觸發配置的 `TTS` 合成語音並播放給使用者。

您也可以選擇提供預先合成的音訊以供播放。這會跳過 `TTS` 步驟並減少回應時間。

> 💡 **Realtime models and TTS**
> 
> `say` 方法需要 TTS 插件。如果您使用的是即時模型，則需要在會話中新增 TTS 外掛程式或使用 `generate_reply()` 方法。

```python
await session.say(
   "Hello. How can I help you today?",
   allow_interruptions=False,
)

```

#### Parameters

- **`text`** _(str | AsyncIterable[str])_: 要講的文字。

 - **`audio`** _(AsyncIterable[rtc.AudioFrame])_ (optional): 播放預先合成的音訊。

 - **`allow_interruptions`** _(boolean)_ (optional): 如果為 `True`，則允許使用者在說話時打斷代理。（預設為 `True`）

 - **`add_to_chat_ctx`** _(boolean)_ (optional): 如果為 `True`，則播放後將文字新增至代理程式的聊天上下文。（預設為 `True`）

#### Returns

回傳一個 [`SpeechHandle`](#speechhandle) 物件。

#### Events

此方法觸發 [`speech_created`](https://docs.livekit.io/agents/build/events.md#speech_created) 事件。

### generate_reply

為了讓對話更加動態，請使用 `session.generate_reply()` 來提示 LLM 產生回應。

有兩種使用 `generate_reply` 的方法：

1. 向代理發出指令以產生回應

    ```python
    session.generate_reply(
    instructions="greet the user and ask where they are from",
    )

    ```

2. 透過文字提供使用者的輸入

    ```python
    session.generate_reply(
    user_input="how is the weather today?",
    )

    ```

> ℹ️ **Impact to chat history**
> 
> 當使用 `generate_reply` 和 `instructions` 時，代理程式使用指令產生回應，並將其加入聊天歷史記錄中。這些指示本身並未被記錄在歷史記錄中。
>
> 相較之下，`user_input` 則直接加入聊天記錄。

#### Parameters

- **`user_input`** _(string)_ (optional): 要回應的使用者輸入。

 - **`instructions`** _(string)_ (optional): 代理人回覆時使用的說明。

 - **`allow_interruptions`** _(boolean)_ (optional): 如果為 `True`，則允許使用者在說話時打斷代理。（預設為 `True`）

#### Returns

回傳一個 [`SpeechHandle`](#speechhandle) 物件。

#### Events

此方法觸發 [`speech_created`](https://docs.livekit.io/agents/build/events.md#speech_created) 事件。

## Controlling agent speech

您可以使用 `say()` 和 `generate_reply()` 方法傳回的 `SpeechHandle` 物件控制代理語音，並允許使用者中斷。


### SpeechHandle

`say()` 和 `generate_reply()` 方法傳回一個 `SpeechHandle` 對象，該物件可讓您追蹤代理語音的狀態。這對於協調後續行動很有用 - 例如，在結束通話之前通知用戶。


```python

await session.say("Goodbye for now.", allow_interruptions=False)

# the above is a shortcut for 
# handle = session.say("Goodbye for now.", allow_interruptions=False)
# await handle.wait_for_playout()

```

您可以等待代理說完再繼續：

```python
handle = session.generate_reply(instructions="Tell the user we're about to run some slow operations.")

# perform an operation that takes time
...

await handle # finally wait for the speech

```

以下範例為使用者發出 Web 請求，並在使用者中斷時取消該請求：

```python
async with aiohttp.ClientSession() as client_session:
    web_request = client_session.get('https://api.example.com/data')
    handle = await session.generate_reply(instructions="Tell the user we're processing their request.")
    if handle.interrupted:
        # if the user interrupts, cancel the web_request too
        web_request.cancel()

```

`SpeechHandle` 具有與 `ayncio.Future` 類似的 API，可讓您新增回呼：

```python
handle = session.say("Hello world")
handle.add_done_callback(lambda _: print("speech done"))

```

### Getting the current speech handle

代理會話的活動語音 handle（如果有）可透過 `current_speech` 屬性取得。如果沒有活躍的語音，此屬性傳回 `None`。否則，它會返回活動的 `SpeechHandle`。

使用活動語音 handle 與發言狀態相協調。例如，您可以確保僅在當前講話結束後而不是講話中途掛斷電話：

```python
# to hang up the call as part of a function call
@function_tool
async def end_call(self, ctx: RunContext):
   """Use this tool when the user has signaled they wish to end the current call. The session will end automatically after invoking this tool."""
   # let the agent finish speaking
   current_speech = ctx.session.current_speech
   if current_speech:
      await current_speech.wait_for_playout()

   # call API to delete_room
   ...

```

### Interruptions

預設情況下，當代理程式偵測到使用者開始說話時，它會停止說話。可以透過在 scheduling speech 時設定 `allow_interruptions=False` 來停用此行為。

若要明確中斷代理，請隨時在 handle 或 session 上呼叫 `interrupt()` 方法。即使將 `allow_interruptions` 設為 `False`，也可以執行此操作。

```python
handle = session.say("Hello world")
handle.interrupt()

# or from the session
session.interrupt()

```

## Customizing pronunciation

大多數 `TTS` 提供者允許您使用語音合成標記語言 (SSML) 自訂單字的發音。以下範例使用 [tts_node](https://docs.livekit.io/agents/build/nodes.md#tts_node) 新增自訂發音規則：


**檔案名稱: `agent.py`**

```python
async def tts_node(
    self,
    text: AsyncIterable[str],
    model_settings: ModelSettings
) -> AsyncIterable[rtc.AudioFrame]:
    # Pronunciation replacements for common technical terms and abbreviations.
    # Support for custom pronunciations depends on the TTS provider.
    pronunciations = {
        "API": "A P I",
        "REST": "rest",
        "SQL": "sequel",
        "kubectl": "kube control",
        "AWS": "A W S",
        "UI": "U I",
        "URL": "U R L",
        "npm": "N P M",
        "LiveKit": "Live Kit",
        "async": "a sink",
        "nginx": "engine x",
    }
    
    async def adjust_pronunciation(input_text: AsyncIterable[str]) -> AsyncIterable[str]:
        async for chunk in input_text:
            modified_chunk = chunk
            
            # Apply pronunciation rules
            for term, pronunciation in pronunciations.items():
                # Use word boundaries to avoid partial replacements
                modified_chunk = re.sub(
                    rf'\b{term}\b',
                    pronunciation,
                    modified_chunk,
                    flags=re.IGNORECASE
                )
            
            yield modified_chunk
    
    # Process with modified text through base TTS implementation
    async for frame in Agent.default.tts_node(
        self,
        adjust_pronunciation(text),
        model_settings
    ):
        yield frame

```

**檔案名稱: `Required imports`**

```python
import re
from livekit import rtc
from livekit.agents.voice import ModelSettings
from livekit.agents import tts
from typing import AsyncIterable

```

下表列出了大多數 `TTS` 提供者支援的 SSML 標籤：

| SSML Tag | Description |
|----------|-------------|
| `phoneme` | 用於使用標準音標字母進行語音發音。這些標籤為所附文字提供了語音發音。 |
| `say as` | 指定如何解釋所附的文字。例如，使用 `character` 單獨說出每個角色，或使用 `date` 指定日曆日期。|
| `lexicon` | 使用音標或文字到發音映射來定義某些單字的發音的自訂字典。 |
| `emphasis` | 強調地朗讀文字。|
| `break` | 新增手動暫停。 |
| `prosody` | 控制語音輸出的音調、語速和音量。|

## Adjusting speech volume

若要調整代理語音的音量，請在 `tts_node` 或 `realtime_audio_output_node` 中新增處理器。或者，您也可以在前端 SDK 中[調整播放音量](https://docs.livekit.io/home/client/tracks/subscribe.md#volume)。

以下範例代理程式具有 0 到 100 之間的可調音量，並提供了[工具呼叫](https://docs.livekit.io/agents/build/tools.md)來更改它。

**檔案名稱: `agent.py`**

```python
class Assistant(Agent):
    def __init__(self) -> None:
        self.volume: int = 50
        super().__init__(
            instructions=f"You are a helpful voice AI assistant. Your starting volume level is {self.volume}."
        )

    @function_tool()
    async def set_volume(self, volume: int):
        """Set the volume of the audio output.

        Args:
            volume (int): The volume level to set. Must be between 0 and 100.
        """
        self.volume = volume

    # Audio node used by STT-LLM-TTS pipeline models
    async def tts_node(self, text: AsyncIterable[str], model_settings: ModelSettings):
        return self._adjust_volume_in_stream(
            Agent.default.tts_node(self, text, model_settings)
        )

    # Audio node used by realtime models
    async def realtime_audio_output_node(
        self, audio: AsyncIterable[rtc.AudioFrame], model_settings: ModelSettings
    ) -> AsyncIterable[rtc.AudioFrame]:
        return self._adjust_volume_in_stream(
            Agent.default.realtime_audio_output_node(self, audio, model_settings)
        )

    async def _adjust_volume_in_stream(
        self, audio: AsyncIterable[rtc.AudioFrame]
    ) -> AsyncIterable[rtc.AudioFrame]:
        stream: utils.audio.AudioByteStream | None = None
        async for frame in audio:
            if stream is None:
                stream = utils.audio.AudioByteStream(
                    sample_rate=frame.sample_rate,
                    num_channels=frame.num_channels,
                    samples_per_channel=frame.sample_rate // 10,  # 100ms
                )
            for f in stream.push(frame.data):
                yield self._adjust_volume_in_frame(f)

        if stream is not None:
            for f in stream.flush():
                yield self._adjust_volume_in_frame(f)

    def _adjust_volume_in_frame(self, frame: rtc.AudioFrame) -> rtc.AudioFrame:
        audio_data = np.frombuffer(frame.data, dtype=np.int16)
        audio_float = audio_data.astype(np.float32) / np.iinfo(np.int16).max
        audio_float = audio_float * max(0, min(self.volume, 100)) / 100.0
        processed = (audio_float * np.iinfo(np.int16).max).astype(np.int16)

        return rtc.AudioFrame(
            data=processed.tobytes(),
            sample_rate=frame.sample_rate,
            num_channels=frame.num_channels,
            samples_per_channel=len(processed) // frame.num_channels,
        )

```

**檔案名稱: `Required imports`**

```python
import numpy as np
from typing import AsyncIterable
from livekit.agents import Agent, function_tool, utils
from livekit.plugins import rtc

```

## Adding background audio

預設情況下，您的代理除了合成語音外不會產生任何音訊。為了增加真實感，您可以發布環境背景音頻，例如辦公室或呼叫中心的噪音。你的代理還可以在 "thinking" 時調整背景音頻，例如添加鍵盤的聲音。

`BackgroundAudioPlayer` 類別管理房間的音訊播放，可播放以下兩種類型的音訊：

- **Ambient sound**: 在背景播放的循環音訊檔案。
- **Thinking sound**: 代理思考時播放的音檔。

以下範例示範了內建音訊剪輯的簡單用法。

**檔案名稱: `agent.py`**

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
      # play office ambience sound looping in the background
      ambient_sound=AudioConfig(BuiltinAudioClip.OFFICE_AMBIENCE, volume=0.8),
      # play keyboard typing sound when the agent is thinking
      thinking_sound=[
               AudioConfig(BuiltinAudioClip.KEYBOARD_TYPING, volume=0.8),
               AudioConfig(BuiltinAudioClip.KEYBOARD_TYPING2, volume=0.7),
         ],
      )

   await background_audio.start(room=ctx.room, agent_session=session)

```

請參閱以下範例以了解更多詳細資訊：

- **[Background Audio](https://github.com/livekit/agents/blob/main/examples/voice_agents/background_audio.py)**: 帶有背景音訊的語音 AI 代理，用於表達思考狀態和氛圍。

### Reference

有關使用 `BackgroundAudioPlayer` 類別的更多詳細信息，請參閱以下部分。

#### BackgroundAudioPlayer

`BackgroundAudioPlayer` 類別管理房間的音訊播放，並具有以下參數：

- **`ambient_sound`** _(AudioSource | AudioConfig | list[AudioConfig])_ (optional): 環境聲音的音訊來源或來源清單。環境聲音在背景中循環播放。

- **`thinking_sound`** _(AudioSource | AudioConfig | list[AudioConfig])_ (optional): 思考聲音的音訊來源或來源清單。當代理思考時，會播放思考聲音。

要啟動背景音頻，請呼叫 [start](#start-the-background-audio-player) 方法。您也可以透過呼叫 `BackgroundAudioPlayer` 類別實例的 [play](#play-audio-files) 方法來隨時播放任意音訊檔案。

#### AudioConfig

`AudioConfig` 類別可讓您控制音量和播放機率。機率值決定了特定聲音被選擇播放的機會。如果所有機率值的總和小於 1，則有時可能只會出現沉默。這對於創建更自然的背景音訊效果很有用。


`AudioConfig` 具有下列屬性：

- **`source`** _(str | AsyncIterator[rtc.AudioFrame]| BuiltInAudioClip)_: 要播放的音訊來源。它可以是檔案路徑、音訊幀的非同步迭代器或[內建音訊剪輯](#builtinaudioclip)。

- **`volume`** _(float)_ (optional) - Default: `1`: 播放音訊來源的音量。

- **`probability`** _(float)_ (optional) - Default: `1`: 播放的機率。如果所有音訊來源的「機率」值總和小於 1，則有可能不會選擇音訊來源，而只會保持靜音。

#### AudioSource

`AudioSource` 可以是下列類型之一：

- `String`: 音訊檔案的路徑。
- `AsyncIterator[rtc.AudioFrame]`: 音訊幀的非同步迭代器。
- `BuiltInAudioClip`: 一段 [built-in audio clip](#builtinaudioclip).

#### BuiltinAudioClip

`BuiltinAudioClip` 枚舉提供了可與背景音訊播放器一起使用的預設音訊剪輯清單：

- `OFFICE_AMBIENCE`: 辦公室氛圍聲音。
- `KEYBOARD_TYPING`: 鍵盤打字的聲音。
- `KEYBOARD_TYPING2`: 鍵盤打字的聲音。這是 `KEYBOARD_TYPING` 聲音的較短片段。

#### Start the background audio player

`start` 方法採用下列參數。如果在 [BackgroundAudioPlayer](#backgroundaudioplayer) 參數中包含環境聲音，它會立即開始播放。如果包含思考聲音，它只會在代理「思考」時播放。

- `room`: 發布音訊的房間。
- `agent_session`: 發布音訊的代理會話。

#### Play audio files

您可以透過呼叫 `BackgroundAudioPlayer` 類別實例的 `play` 方法來隨時播放任何音訊檔案。`play` 方法採用下列參數：

- **`audio`** _(AudioSource | AudioConfig | list[AudioConfig])_: 要播放的音訊來源或來源清單。要了解更多信息，請參閱 [AudioSource](#audiosource)。

- **`loop`** _(boolean)_ (optional) - Default: `False`: 設定為 `True` 以循環播放音訊來源。

例如，如果您在上一個範例中建立了 `background_audio`，則可以像這樣播放音訊檔案：

```python
MY_AUDIO_FILE = "<PATH_TO_AUDIO_FILE>"
background_audio.play(MY_AUDIO_FILE)

```

## Additional resources

要了解更多信息，請參閱以下資源。

- **[Voice AI quickstart](https://docs.livekit.io/agents/start/voice-ai.md)**: 使用快速入門作為添加音訊程式碼的起點。

- **[Speech related event](https://docs.livekit.io/agents/build/events.md#speech_created)**: 了解有關 `speech_created` 事件的更多信息，該事件在創建新代理語音時觸發。

- **[LiveKit SDK](https://docs.livekit.io/home/client/tracks/publish.md#publishing-audio-tracks)**: 了解如何使用 LiveKit SDK 播放音軌。

- **[Background audio example](https://github.com/livekit/agents/blob/main/examples/voice_agents/background_audio.py)**: 使用 `BackgroundAudioPlayer` 類別播放辦公室環境噪音和思考聲音的範例。

- **[Text-to-speech (TTS)](https://docs.livekit.io/agents/integrations/tts.md)**: 管道代理的 TTS 使用和範例。

- **[Speech-to-speech](https://docs.livekit.io/agents/integrations/realtime.md)**: 多模態即時 API 可以理解語音輸入並直接產生語音輸出。