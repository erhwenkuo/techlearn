
# Session recording and transcripts

> 匯出視訊、音訊或文字格式的會話資料。

## Overview

從品質監控到法規遵循，記錄或保存應用中發生的會話有許多原因。 LiveKit 可讓您錄製客服人員會話的視訊和音頻，或儲存文字記錄。

## Video or audio recording

使用 [Egress 功能](https://docs.livekit.io/home/egress/overview.md) 錄製音訊和/或影片。最簡單的方法是在代理程式的入口點啟動一個 [room composite recorder](https://docs.livekit.io/home/egress/composite-recording.md)。當代理進入房間時，它會開始錄製，並自動捕捉房間內共享的所有音訊和視訊。當所有參與者離開時，錄製結束。錄製內容將儲存在您選擇的雲端儲存提供者中。

### Example

此範例示範如何修改 [Voice AI 快速入門](https://docs.livekit.io/agents/start/voice-ai.md) 來錄製會話。它使用 Amazon S3，但您也可以將檔案儲存到任何與 Amazon S3 相容的儲存供應商、Google Cloud Storage 或 Azure Blob Storage。

有關使用 Google 和 Azure 的更多 egress 範例，請參閱 [Egress examples](https://docs.livekit.io/home/egress/examples.md)。

若要修改 [Voice AI 快速入門](https://docs.livekit.io/agents/start/voice-ai.md)以記錄會話，請新增下列程式碼：

** Filename: `agent.py`**

```python
from livekit import api

async def entrypoint(ctx: JobContext):
    # Add the following code to the top, before calling ctx.connect()

    # Set up recording
    req = api.RoomCompositeEgressRequest(
        room_name=ctx.room.name,
        audio_only=True,
        file_outputs=[api.EncodedFileOutput(
            file_type=api.EncodedFileType.OGG,
            filepath="livekit/my-room-test.ogg",
            s3=api.S3Upload(
                endpoint="<S3 Service endpoint>",
                bucket="<S3 bucket name>",
                region="<S3 region>",
                access_key="<S3 access key>",
                secret="<S3 secret key>",
            ),
        )],
    )

    lkapi = api.LiveKitAPI()
    res = await lkapi.egress.start_room_composite_egress(req)

    await lkapi.aclose()

    # .. The rest of your entrypoint code follows ...

```

## Text transcripts

文字轉錄可透過 `llm_node` 或 `transcription_node` 即時取得，詳情請參閱 [Pipeline nodes](https://docs.livekit.io/agents/build/nodes.md)文件。您可以將其與其他事件和回調一起使用，以記錄您的會話以及任何其他所需的資料。

此外，您可以隨時存取 `session.history` 屬性來取得完整的對話歷史記錄。使用 `add_shutdown_callback` 方法，您可以在使用者離開並關閉房間後將對話歷史記錄儲存到檔案中。

為了更快速地存取正在進行的對話，您可以監聽相關 [events](https://docs.livekit.io/agents/build/events.md)。每當有新內容加入聊天歷史記錄時，都會觸發 `conversation_item_added` 事件。每當使用者輸入被轉錄時，都會觸發 `user_input_transcribed` 事件。這些結果可能與最終轉錄結果不同。

### Example

此範例顯示如何修改 [Voice AI 快速入門](https://docs.livekit.io/agents/start/voice-ai.md) 以將對話歷史記錄儲存到 JSON 檔案。

** Filename: `agent.py`**

```python
from datetime import datetime
import json

def entrypoint(ctx: JobContext):
    # Add the following code to the top, before calling ctx.connect()
    
    async def write_transcript():
        current_date = datetime.now().strftime("%Y%m%d_%H%M%S")

        # This example writes to the temporary directory, but you can save to any location
        filename = f"/tmp/transcript_{ctx.room.name}_{current_date}.json"
        
        with open(filename, 'w') as f:
            json.dump(session.history.to_dict(), f, indent=2)
            
        print(f"Transcript for {ctx.room.name} saved to {filename}")

    ctx.add_shutdown_callback(write_transcript)

    # .. The rest of your entrypoint code follows ...

```