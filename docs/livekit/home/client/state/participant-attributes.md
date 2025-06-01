# Participant attributes and metadata

> 每位參與者狀態的鍵值儲存。

## Overview

每個 LiveKit 參與者都有兩個用於應用程式特定狀態的欄位：

- **Participant.attributes**: 字串鍵值存儲。
- **Participant.metadata**: 可以儲存任何資料的單一字串。

這些欄位由 LiveKit 伺服器儲存和管理，並自動同步給稍後加入房間的新參與者。

可以在參與者的 [access token](https://docs.livekit.io/concepts/authentication.md) 中設定初始值，確保參與者連線時該值立即可用。

元資料欄位是單一字串，而屬性欄位是鍵值儲存。這允許對狀態的不同部分進行細粒度更新，而不會影響或傳輸其他鍵的值。

## Deleting attributes

若要刪除屬性鍵，請將其值設為空字串 (`''`)。

## Update frequency

由於伺服器上的同步開銷，屬性和元資料不適合高頻更新（每隔幾秒鐘更新一次以上）。如果您需要更頻繁地傳送更新，請考慮使用 [data packets](https://docs.livekit.io/home/client/data/packets.md)。

## Size limits

元資料和屬性各自都有 64 KiB 的限制。對於屬性，此限制包括所有鍵和值的組合大小。

## Usage from LiveKit SDKs

LiveKit SDK 接收有關本地參與者和房間內任何遠端參與者的屬性和元資料變化的事件。有關詳細信息，請參閱 [Handling events](https://docs.livekit.io/home/client/events.md#events)。

參與者必須在其存取權杖中擁有 `canUpdateOwnMetadata` 權限才能更新自己的屬性或元資料。


=== "Python"

    ```python
    @room.on("participant_attributes_changed")
    def on_attributes_changed(
        changed_attributes: dict[str, str], participant: rtc.Participant
    ):
        logging.info(
            "participant attributes changed: %s %s",
            participant.attributes,
            changed_attributes,
        )

    @room.on("participant_metadata_changed")
    def on_metadata_changed(
        participant: rtc.Participant, old_metadata: str, new_metadata: str
    ):
        logging.info(
            "metadata changed from %s to %s",
            old_metadata,
            participant.metadata,
        )

    # setting attributes & metadata are async functions
    async def myfunc():
        await room.local_participant.set_attributes({"foo": "bar"})
        await room.local_participant.set_metadata("some metadata")

    asyncio.run(myfunc())

    ```

=== "JavaScript"

    ```typescript
    // receiving changes
    room.on(
        RoomEvent.ParticipantAttributesChanged,
        (changed: Record<string, string>, participant: Participant) => {
            console.log(
                'participant attributes changed',
                changed,
                'all attributes',
                participant.attributes,
            );
        },
    );

    room.on(
        RoomEvent.ParticipantMetadataChanged,
        (oldMetadata: string | undefined, participant: Participant) => {
            console.log('metadata changed from', oldMetadata, participant.metadata);
        },
    );

    // updating local participant
    room.localParticipant.setAttributes({
        myKey: 'myValue',
        myOtherKey: 'otherValue',
        });
        room.localParticipant.setMetadata(
        JSON.stringify({
            some: 'values',
        }),
    );

    ```

=== "React"

    我們的 React 元件庫提供了一些方便的 hooks 來處理參與者屬性。

    ```jsx
    function MyComponent() {
        // getting all attributes of a participant
        const { attributes } = useParticipantAttributes({ participant: participant });

        // getting a single attribute of a participant
        const myKey = useParticipantAttribute('myKey', { participant: participant });

        // setting attributes and metadata would be the same as in JS
    }

    ```

=== "Swift"

    ```swift
    extension MyClass: RoomDelegate {
        // receiving participant attributes changes
        func room(_ room: Room, participant: Participant, didUpdateAttributes changedAttributes: [String: String]) {

        }

        // receiving room metadata changes
        func room(_ room: Room, didUpdateMetadata newMetadata: String?) {

        }
    }

    // updating participant attributes (from async function)
    try await room.localParticipant.set(attributes: ["mykey" : "myvalue"])

    // updating participant metadata
    try await room.localParticipant.set(metadata: "some metadata")

    ```

=== "Kotlin"

    ```kotlin
    room.events.collect { event ->
        when (event) {
            is RoomEvent.ParticipantAttributesChanged -> {
            }
            is RoomEvent.ParticipantMetadataChanged -> {
            }
        }
    }

    localParticipant.updateAttributes(mapOf("myKey" to "myvalue"))

    localParticipant.updateMetadata("mymetadata")

    ```


=== "Flutter"

    ```dart
    final listener = room.createListener();

    listener
    ..on<ParticipantAttributesChanged>((event) {})
    ..on<ParticipantMetadataUpdatedEvent>((event) {});

    room.localParticipant?.setAttributes({
    'myKey': 'myValue',
    });

    room.localParticipant?.setMetadata('myMetadata');

    ```


## Usage from server APIs

從伺服器端，您可以使用 [RoomService.UpdateParticipant](https://docs.livekit.io/server/room-management.md#updateparticipant) API 更新房間中任何參與者的屬性或元資料。


=== "Python"

    ```python
    import livekit.api

    lkapi = livekit.api.LiveKitAPI()
    lkapi.room.update_participant(
        UpdateParticipantRequest(
            room="roomName",
            identity="participantIdentity",
            metadata="new metadata",
            attributes={
                "myKey": "myValue",
            },
        ),
    )

    ```

=== "Node.js"

    ```typescript
    import { RoomServiceClient } from 'livekit-server-sdk';

    const roomServiceClient = new RoomServiceClient('myhost', 'api-key', 'my secret');
    roomServiceClient.updateParticipant('room', 'identity', {
    attributes: {
        myKey: 'myValue',
    },
    metadata: 'updated metadata',
    });

    ```

=== "Go"

    ```go
    import (
    "context"
    lksdk "github.com/livekit/server-sdk-go/v2"
    )

    func updateMetadata(values interface{}) {
    roomClient := lksdk.NewRoomServiceClient(host, apiKey, apiSecret)

        _, err := roomClient.UpdateParticipant(context.Background(), &livekit.UpdateParticipantRequest{
            Room:     "roomName",
            Identity: "participantIdentity",
            Metadata: "new metadata",
            Attributes: map[string]string{
                "myKey": "myvalue",
            },
        })
    }

    ```


=== "Ruby"

    ```ruby
    require "livekit"

    roomServiceClient = LiveKit::RoomServiceClient.new("https://my-livekit-url")
    roomServiceClient.update_participant(
    room: "roomName",
    identity: "participantIdentity",
    attributes: {"myKey": "myvalue"})

    ```

=== "Java/Kotlin"

    下面的範例是 Kotlin 中的，Java API 類似。

    ```kotlin
    // Update participant attributes and metadata
    val call = roomServiceClient.updateParticipant(
        roomName = "room123",
        identity = "participant456",
        metadata = "New metadata",
        attributes = mapOf("myKey" to "myValue")
    )
    val response = call.execute()

    ```