# Data packets

> 適用於高頻或進階用例的低階 API。

## Overview

使用 `LocalParticipant.publishData` 或 [RoomService.SendData](https://docs.livekit.io/reference/server/server-apis.md#senddata) 將單獨的資料包傳送給房間中的一個或多個參與者。

!!! info
    這是一個低階 API，用於對單一資料包行為進行進階控制。對於大多數用例，請考慮使用更高層級的 [text streams](https://docs.livekit.io/home/client/data/text-streams.md)、[byte streams](https://docs.livekit.io/home/client/data/byte-streams.md) 或 [RPC](https://docs.livekit/client/data/byte-streams。


### Delivery options

LiveKit 提供兩種資料包傳送形式：

- **Reliable**: 資料包依序傳送，如果資料包遺失則自動重新傳輸。這對於優先考慮交付而非延遲的場景（例如室內聊天）來說是更可取的。
- **Lossy**: 每個資料包發送一次，不保證順序。這對於優先考慮交付速度的即時更新是理想的。

!!! info
    可靠傳送表示 "best-effort" delivery。它不能完全保證資料包在所有情況下都能被送達。例如，在發送資料包時暫時斷開連接的接收器將無法接收該資料包。資料包不會在伺服器上緩衝，只會嘗試有限次數的重新傳輸。

### Size limits

在 **reliable** 傳送模式下，每個資料包的大小最大可達 15KiB。整個資料包的協定限制為 16KiB，但 LiveKit 新增了各種標頭以正確路由資料包，從而減少了可用於使用者資料的空間。

雖然某些平台可能支援更大的資料包大小而不會返回錯誤，但 LiveKit 建議此 16KiB 限制，以最大限度地提高跨平台相容性並解決串流控制傳輸協定 (SCTP) 的限制。  要了解更多信息，請參閱[了解訊息大小限制](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Using_data_channels#understanding_message_size_limits)。

在 **lossy** 傳送模式下，LiveKit 建議使用更小的資料包（最大僅 1300 位元組），以保持在 1400 位元組的網路最大傳輸單元 (MTU) 內。較大的資料包會分割成多個資料包，如果任何一個資料包遺失，整個資料包也會隨之遺失。

### Selective delivery

可以使用 `publishData` 呼叫中的 `destinationIdentities` 參數將封包傳送到整個房間或參與者的子集。要發送到整個房間，請將 `destinationIdentities` 留空。

### Topic

您可能有不同類型和用途的資料包。為了輕鬆區分，請將 `topic` 欄位設定為對您的應用程式有意義的任何字串。

例如，在即時多人遊戲中，您可能會使用不同的主題來發送聊天訊息、更新角色位置和更新環境。

## Usage


=== "Python"

    ```python
    @room.on("data_received")
    def on_data_received(data: rtc.DataPacket):
    logging.info("received data from %s: %s", data.participant.identity, data.data)

    # string payload will be encoded to bytes with UTF-8
    await room.local_participant \
    .publish_data("my payload",
                    reliable=True,
                    destination_identities=["identity1", "identity2"],
                    topic="topic1")

    ```

=== "JavaScript"

    ```typescript
    const strData = JSON.stringify({some: "data"})
    const encoder = new TextEncoder()
    const decoder = new TextDecoder()

    // publishData takes in a Uint8Array, so we need to convert it
    const data = encoder.encode(strData);

    // Publish lossy data to the entire room
    room.localParticipant.publishData(data, {reliable: false})

    // Publish reliable data to a set of participants
    room.localParticipant.publishData(data, {reliable: true, destinationIdentities: ['my-participant-identity']})

    // Receive data from other participants
    room.on(RoomEvent.DataReceived, (payload: Uint8Array, participant: Participant, kind: DataPacket_Kind) => {
    const strData = decoder.decode(payload)
    ...
    })

    ```

=== "Swift"

    ```swift
    import LiveKit

    public class DataExample {
        func publishData(localParticipant: LocalParticipant, destinationIdentities: [Participant.Identity]) async throws {
            let someVal = "your value"

            // Publish lossy data to the entire room
            let options1 = DataPublishOptions(reliable: false)
            try await localParticipant.publish(data: someVal.data(using: .utf8), options: options1)

            // Publish reliable data to a set of participants
            let options2 = DataPublishOptions(reliable: true, destinationIdentities: destinationIdentities)
            try await localParticipant.publish(data: someVal.data(using: .utf8), options: options2)
        }
    }

    extension DataExample: RoomDelegate {
        func room(_ room: Room, participant: RemoteParticipant?, didReceiveData data: Data, forTopic topic: String) {
            // Received data
        }
    }

    ```

=== "Kotlin"

    ```kotlin
    // Publishing data
    coroutineScope.launch {
    val data: ByteArray = //...

    // Publish lossy data to the entire room
    room.localParticipant.publishData(data, DataPublishReliability.LOSSY)

    // Publish reliable data to a set of participants
    val identities = listOf(
        Participant.Identity("alice"),
        Participant.Identity("bob"),
    )
    room.localParticipant.publishData(data, DataPublishReliability.RELIABLE, identities)
    }

    // Processing received data
    coroutineScope.launch {
    room.events.collect { event ->
        if(event is RoomEvent.DataReceived) {
            // Process data
        }
    }
    }

    ```

=== "Flutter"

    ```dart
    class DataExample {
    Room room;
    late final _listener = room.createListener();

    DataExample() {
        _listener.on<DataReceivedEvent>((e) {
        // Process received data: e.data
        })
    }

    void publishData() {
        // publish lossy data to the entire room
        room.localParticipant.publishData(data, reliable: false);

        // publish reliable data to a set of participants with a specific topic
        room.localParticipant.publishData(data,
                reliable: true,
                destinationIdentities: ["identity1", "identity2"],
                topic: "topic1");
    }

    void dispose() {
        _listener.dispose();
    }
    }

    ```


=== "Go"

    ```go
    room := lksdk.ConnectToRoom(
        url,
        info,
        &lksdk.RoomCallback{
            OnDataReceived: func(data []byte, rp *lksdk.RemoteParticipant) {
                // Process received data
            },
        },
    )

    // Publish lossy data to the entire room
    room.LocalParticipant.PublishDataPacket(lksdk.UserData(data))

    // Publish reliable data to a set of participants
    room.LocalParticipant.PublishDataPacket(
        lksdk.UserData(data),
        lksdk.WithDataPublishReliable(true),
        lksdk.WithDataPublishDestination([]string{"alice", "bob"}),
    )

    ```

=== "Unity"

    ```csharp
    yield return room.LocalParticipant.PublishData(data, DataPacketKind.RELIABLE, participant1, participant2);

    room.DataReceived += (data, participant, kind) =>
    {
        // Process received data
    };

    ```
