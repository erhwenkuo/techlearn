# Job lifecycle

> 了解有關 entrypoint 功能以及如何結束和清理 LiveKit 會話的更多資訊。

## Lifecycle

當 [worker](https://docs.livekit.io/agents/worker.md) 接受來自 LiveKit 伺服器的工作請求時，它會啟動一個新進程並在其中執行代理程式碼。每個作業都在單獨的進程中運行，以將代理彼此隔離。如果會話實例崩潰，它不會影響在同一工作器上執行的其他代理程式。該作業一直運行，直到所有標準和 SIP 參與者離開房間，或您明確將其關閉。

## Entrypoint

對於 worker 執行的每個新作業(new job)，`entrypoint` 作為流程的主要功能執行，有效地將控制權移交給您的程式碼。您應該加載任何必要的特定於應用程式的數據，調用 `ctx.connect()` 加入房間，然後執行代理的邏輯。

此範例展示了一個簡單的 entrypoint，它處理傳入的音軌並向房間發布文字訊息。

=== "Python"

    ```python
    async def do_something(track: rtc.RemoteAudioTrack):
        audio_stream = rtc.AudioStream(track)
        async for event in audio_stream:
            # Do something here to process event.frame
            pass
        await audio_stream.aclose()

    async def entrypoint(ctx: JobContext):
        # an rtc.Room instance from the LiveKit Python SDK
        room = ctx.room

        # set up listeners on the room before connecting
        @room.on("track_subscribed")
        def on_track_subscribed(track: rtc.Track, *_):
            if track.kind == rtc.TrackKind.KIND_AUDIO:
                asyncio.create_task(do_something(track))

        # connect to room
        await ctx.connect(auto_subscribe=AutoSubscribe.AUDIO_ONLY)

        # when connected, room.local_participant represents the agent
        await room.local_participant.send_text('hello world', topic='hello-world')
    )
        # iterate through currently connected remote participants
        for rp in room.remote_participants.values():
            print(rp.identity)

    ```

更多 LiveKit Agents 範例，請參閱 [GitHub 儲存庫](https://github.com/livekit/agents/tree/main/examples)。要了解有關發布和接收軌蹟的更多信息，請參閱以下主題：

- **[Media tracks](https://docs.livekit.io/home/client/tracks.md)**: 使用麥克風、揚聲器、相機並與您的代理共享螢幕。

- **[Realtime text and data](https://docs.livekit.io/home/client/data.md)**: 使用文字和數據管道與您的代理溝通。

## Passing data to a job

您可以使用作業元資料、房間元資料或參與者屬性，透過使用者或作業特定的資料來定製作業。

### Job metadata

作業元資料(job metadata)是在 [dispatch request](https://docs.livekit.io/agents/worker/agent-dispatch.md#via-api) 中定義的自由格式字串字段，並在 `entrypoint` 中使用。使用 JSON 或類似的結構化資料來傳遞複雜的訊息。

例如，您可以傳遞使用者的 ID、姓名和電話號碼：

```python
import json

async def entrypoint(ctx: JobContext):
    metadata = json.loads(ctx.job.metadata)
    user_id = metadata["user_id"]
    user_name = metadata["user_name"]
    user_phone = metadata["user_phone"]
    # ...

```

有關調度的更多信息，請參閱以下文章：

- **[Agent dispatch](https://docs.livekit.io/agents/worker/agent-dispatch.md#via-api)**: 了解如何使用自訂元資料調度代理程式。

### Room metadata and participant attributes

您還可以使用房間名稱、元資料和參與者屬性等屬性來自訂代理行為。

以下是一個展示如何存取各種屬性的範例：

```python
async def entrypoint(ctx: JobContext):
  # connect to the room
  await ctx.connect(auto_subscribe=AutoSubscribe.AUDIO_ONLY)

  # wait for the first participant to arrive
  participant = await ctx.wait_for_participant()

  # customize behavior based on the participant
  print(f"connected to room {ctx.room.name} with participant {participant.identity}")

  # inspect the current value of the attribute
  language = participant.attributes.get("user.language")

  # listen to when the attribute is changed
  @ctx.room.on("participant_attributes_changed")
  def on_participant_attributes_changed(changed_attrs: dict[str, str], p: rtc.Participant):
      if p == participant:
        language = p.attributes.get("user.language")
        print(f"participant {p.identity} changed language to {language}")

```

有關詳細信息，請參閱以下文章：

- **[Room metadata](https://docs.livekit.io/home/client/state/room-metadata.md)**: 了解如何設定和使用房間元資料。

- **[Participant attributes & metadata](https://docs.livekit.io/home/client/data.md#participant-attributes)**: 了解如何設定和使用參與者屬性和元資料。

## Ending the session

### Disconnecting the agent

當代理完成其任務且房間內不再需要該代理程式時，您可以中斷該代理程式的連線。這使得 LiveKit 會話中的其他參與者可以繼續。您的 [shutdown hooks](#post-processing-and-cleanup) 在 `shutdown` 函數之後執行。

=== "Python"

    ```python
    async def entrypoint(ctx: JobContext):
        # do some work
        ...

        # disconnect from the room
        ctx.shutdown(reason="Session ended")

    ```

### Disconnecting everyone

如果需要結束所有人的會話，請使用伺服器 API [deleteRoom](https://docs.livekit.io/home/server/managing-rooms.md#delete-a-room) 結束會話。

將發送 `Disconnected` [oom event](https://docs.livekit.io/home/client/events.md)，並且該房間將從伺服器中刪除。

=== "Python"

    ```python
    from livekit import api

    async def entrypoint(ctx: JobContext):
        # do some work
        ...

        api_client = api.LiveKitAPI(
            os.getenv("LIVEKIT_URL"),
            os.getenv("LIVEKIT_API_KEY"),
            os.getenv("LIVEKIT_API_SECRET"),
        )
        await api_client.room.delete_room(api.DeleteRoomRequest(
            room=ctx.job.room.name,
        ))

    ```

## Post-processing and cleanup

會話結束後，您可以使用 shutdown hooks 執行後處理或清理任務。例如，您可能希望將使用者狀態保存在資料庫中。

=== "Python"

    ```python
    async def entrypoint(ctx: JobContext):
        async def my_shutdown_hook():
            # save user state
            ...
        ctx.add_shutdown_callback(my_shutdown_hook)

    ```

!!! info
    關閉掛鉤(shutdown hooks)應在短時間內完成。預設情況下，框架會等待 60 秒才強制終止該進程。您可以使用 [WorkerOptions](https://docs.livekit.io/agents/worker/options.md) 中的 `shutdown_process_timeout` 參數調整此逾時時間。