# Vision

> 透過影像和即時視訊的視覺理解增強您的代理。

## Overview

如果您的 [LLM](https://docs.livekit.io/agents/integrations/llm.md) 包含對視覺的支持，您可以將圖像添加到其聊天上下文中以充分利用其功能。 LiveKit 具有從磁碟、網路或直接從前端應用程式上傳新增原始映像的工具。此外，您可以使用 `STT-LLM-TTS` 管道模型中的採樣幀作為實時視頻，或使用實時模型（例如 [Gemini Live](https://docs.livekit.io/agents/integrations/realtime/gemini.md)）的真實視頻輸入。

本指南包括每個用例的視覺功能和程式碼範例的概述。

## Images

代理的聊天上下文支援圖像和文字。您可以在聊天上下文中添加任意數量的圖像，但請記住，較大的上下文視窗會導致回應時間變慢。

若要將圖像新增至聊天上下文，請建立一個 `ImageContent` 物件並將其包含在聊天訊息中。圖像內容可以是 Base 64 資料 URL、外部 URL 或來自 [video track](https://docs.livekit.io/home/client/tracks.md) 的幀。

### Load into initial context

以下範例顯示了在啟動時使用影像初始化的代理程式。此範例使用外部 URL，但您可以修改它以使用 base 64 資料 URL 載入本機檔案：

**檔案名稱: `agent.py`**

```python
def entrypoint(ctx: JobContext):
    # ctx.connect, etc.

    session = AgentSession(
        # ... stt, tts, llm, etc.
    )
    
    initial_ctx = ChatContext()
    initial_ctx.add_message(
        role="user",
        content=[
            "Here is a picture of me", 
            ImageContent(image="https://example.com/image.jpg")
        ],
    )

    await session.start(
        room=ctx.room,
        agent=Agent(chat_ctx=initial_ctx,),
        # ... room_input_options, etc.
    )        

```

**套件載入: `Required imports`**

```python
from livekit.agents.llm import ImageContent
from livekit.agents import Agent, AgentSession, ChatContext, JobContext

```

> 🔥 **LLM provider support for external URLs**
> 
> 並非每個提供者都支援外部圖像 URL。有關詳細信息，請參閱其文件。

### Upload from frontend

若要從前端應用程式上傳圖片，請使用 LiveKit SDK 的 [sendFile 方法](https://docs.livekit.io/home/client/data/byte-streams.md#sending-files)。為您的代理程式新增一個位元組流處理程序來接收影像資料並將其新增至聊天上下文。這是一個簡單的代理，能夠從位元組流主題 `"images"` 上接收來自用戶的圖像：

**檔案名稱: `agent.py`**

```python
class Assistant(Agent):
    def __init__(self) -> None:
        self._tasks = [] # Prevent garbage collection of running tasks
        super().__init__(instructions="You are a helpful voice AI assistant.")
    
    async def on_enter(self):
        def _image_received_handler(reader, participant_identity):
            task = asyncio.create_task(
                self._image_received(reader, participant_identity)
            )
            self._tasks.append(task)
            task.add_done_callback(lambda t: self._tasks.remove(t))
        
        # Add the handler when the agent joins
        get_job_context().room.register_byte_stream_handler("images", _image_received_handler)
    
    async def _image_received(self, reader, participant_identity):
        image_bytes = bytes()
        async for chunk in reader:
            image_bytes += chunk

        chat_ctx = self.chat_ctx.copy()

        # Encode the image to base64 and add it to the chat context
        chat_ctx.add_message(
            role="user",
            content=[
                ImageContent(
                    image=f"data:image/png;base64,{base64.b64encode(image_bytes).decode('utf-8')}"
                )
            ],
        )
        await self.update_chat_ctx(chat_ctx)

```

**套件載入: `Required imports`**

```python
import asyncio
import base64
from livekit.agents import Agent, get_job_context
from livekit.agents.llm import ImageContent

```

### Sample video frames

LLM 可以處理靜止影像形式的視頻，但許多 LLM 並未針對此用例進行訓練，並且在透過視訊來源理解運動和其他變化時會產生次優結果。即時模型，例如 [Gemini Live](https://docs.livekit.io/agents/integrations/realtime/gemini.md)，是在視訊上進行訓練的，您可以啟用 [即時視訊輸入](#video) 以獲得自動支援。

如果您使用 `STT-LLM-TTS` 管道，您仍然可以透過在適當的時間對視訊軌道進行取樣來處理影片。例如，在下面的範例中，代理程式始終在使用者每次對話時包含最新的視訊幀。這為模型提供了額外的上下文，而不會使其被資料淹沒，也不會期望它一次解釋許多連續的幀：

**檔案名稱: `agent.py`**

```python
class Assistant(Agent):
    def __init__(self) -> None:
        self._latest_frame = None
        self._video_stream = None
        self._tasks = []
        super().__init__(instructions="You are a helpful voice AI assistant.")
    
    async def on_enter(self):
        room = get_job_context().room

        # Find the first video track (if any) from the remote participant
        remote_participant = list(room.remote_participants.values())[0]
        video_tracks = [publication.track for publication in list(remote_participant.track_publications.values()) if publication.track.kind == rtc.TrackKind.KIND_VIDEO]
        if video_tracks:
            self._create_video_stream(video_tracks[0])
        
        # Watch for new video tracks not yet published
        @room.on("track_subscribed")
        def on_track_subscribed(track: rtc.Track, publication: rtc.RemoteTrackPublication, participant: rtc.RemoteParticipant):
            if track.kind == rtc.TrackKind.KIND_VIDEO:
                self._create_video_stream(track)
                        
    async def on_user_turn_completed(self, turn_ctx: ChatContext, new_message: ChatMessage) -> None:
        # Add the latest video frame, if any, to the new message
        if self._latest_frame:
            new_message.content.append(ImageContent(image=self._latest_frame))
            self._latest_frame = None
    
    # Helper method to buffer the latest video frame from the user's track
    def _create_video_stream(self, track: rtc.Track):
        # Close any existing stream (we only want one at a time)
        if self._video_stream is not None:
            self._video_stream.close()

        # Create a new stream to receive frames    
        self._video_stream = rtc.VideoStream(track)
        async def read_stream():
            async for event in self._video_stream:
                # Store the latest frame for use later
                self._latest_frame = event.frame
        
        # Store the async task
        task = asyncio.create_task(read_stream())
        task.add_done_callback(lambda t: self._tasks.remove(t))
        self._tasks.append(task)

```

**套件載入: `Required imports`**

```python
import asyncio
from livekit import rtc
from livekit.agents import Agent, get_job_context
from livekit.agents.llm import ImageContent

```

#### Video frame encoding

預設情況下，`ImageContent` 將視訊畫面以其原始大小編碼為 JPEG。若要調整編碼訊框的大小，請設定 `inference_width` 和 `inference_height` 參數。每個幀的大小都會調整到適合提供的尺寸，同時保持原始的縱橫比。為了獲得更多控制，請使用 `livekit.agents.utils.images` 模組的 `encode` 方法並將結果作為資料 URL 傳遞：

**檔案名稱: `agent.py`**

```python
image_bytes = encode(
    event.frame,
    EncodeOptions(
        format="PNG",
        resize_options=ResizeOptions(
            width=512, 
            height=512, 
            strategy="scale_aspect_fit"
        )
    )
)
image_content = ImageContent(
    image=f"data:image/png;base64,{base64.b64encode(image_bytes).decode('utf-8')}"
)

```

**套件載入: `Required imports`**

```python
import base64
from livekit.agents.utils.images import encode, EncodeOptions, ResizeOptions

```

### Inference detail

如果您的 LLM 提供者支援它，您可以將 `inference_detail` 參數設為 `"high"` 或 `"low"` 來控制所應用的令牌使用情況和推理品質。預設值為 `auto`，即使用提供者的預設值。

## Live video

> ℹ️ **Limited support**
> 
> 即時視訊輸入需要具有視訊支援的即時模型。目前僅 [Gemini Live](https://docs.livekit.io/agents/integrations/realtime/gemini.md) 具備此功能。

將 `RoomInputOptions` 中的 `video_enabled` 參數設定為 `True` 以啟用即時視訊輸入。如果可用，您的代理程式會自動從使用者的 [camera](https://docs.livekit.io/home/client/tracks/publish.md) 或 [screen sharing](https://docs.livekit.io/home/client/tracks/screenshare.md) 軌道接收訊框。僅使用最近發布的單一視訊軌道。

預設情況下，當使用者說話時，代理每秒採樣一幀，否則每三秒採樣一幀。每格適合 1024x1024 並編碼為 JPEG。若要覆蓋幀速率，請使用自訂實例在 `AgentSession` 上設定 `video_sampler`。

視訊輸入是被動的，對語氣轉折檢測沒有影響。若要在非對話環境中利用即時視訊輸入，請使用[manual turn control](https://docs.livekit.io/agents/build/turns.md#manual)並根據計時器或其他時間表觸發 LLM 回應或工具呼叫。

以下範例顯示如何將 Gemini Live 視覺添加到您的 [voice AI quickstart](https://docs.livekit.io/agents/start/voice-ai.md) 代理程式中：

**檔案名稱: `agent.py`**

```python
class VideoAssistant(Agent):
    def __init__(self) -> None:
        super().__init__(
            instructions="You are a helpful voice assistant with live video input from your user.",
            llm=google.beta.realtime.RealtimeModel(
                voice="Puck",
                temperature=0.8,
            ),
        ) 

async def entrypoint(ctx: JobContext):
    await ctx.connect()

    session = AgentSession()

    await session.start(
        agent=VideoAssistant(),
        room=ctx.room,
        room_input_options=RoomInputOptions(
            video_enabled=True,
            # ... noise_cancellation, etc.
        ),
    )

```

**套件載入: `Required imports`**

```python
from livekit.agents import (
    AgentSession,
    RoomInputOptions,
)
from livekit.plugins import google

```

## Additional resources

以下文件和範例可以幫助您開始使用 LiveKit Agents 中的視覺。

- **[Voice AI quickstart](https://docs.livekit.io/agents/start/voice-ai.md)**: 使用快速入門作為添加視覺程式碼的起點。

- **[Byte streams](https://docs.livekit.io/home/client/data/byte-streams.md)**: 使用位元組流將圖像從前端發送到代理。

- **[RoomIO](https://docs.livekit.io/agents/build.md#roomio)**: 了解有關“RoomIO”及其如何管理軌道的更多資訊。

- **[Vision Assistant](https://github.com/livekit-examples/vision-demo)**: 由 Gemini Live 提供支援的具有視訊輸入的語音 AI 代理。

- **[Camera and microphone](https://docs.livekit.io/home/client/tracks/publish.md)**: 從前端發布攝影機和麥克風軌道。

- **[Screen sharing](https://docs.livekit.io/home/client/tracks/screenshare.md)**: 從您的前端發布螢幕分享軌道。
