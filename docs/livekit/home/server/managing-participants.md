# Managing participants

> 從後端伺服器列出、刪除和靜音參與者(participant)。

## Initialize RoomServiceClient

參與者管理由 RoomServiceClient 完成，建立方式如下：

=== "Go"

    ```go
    import (
        lksdk "github.com/livekit/server-sdk-go"
        livekit "github.com/livekit/protocol/livekit"
    )

    // ...

    host := "https://my.livekit.host"
    roomClient := lksdk.NewRoomServiceClient(host, "api-key", "secret-key")

    ```


=== "Python"

    ```shell
    pip install livekit-api

    ```

    ```python
    from livekit.api import LiveKitAPI

    # Will read LIVEKIT_URL, LIVEKIT_API_KEY, and LIVEKIT_API_SECRET from environment variables
    async with api.LiveKitAPI() as lkapi:
    # ... use your client with `lkapi.room` ...

    ```

=== "Node.js"

    ```js
    import { Room, RoomServiceClient } from 'livekit-server-sdk';

    const livekitHost = 'https://my.livekit.host';
    const roomService = new RoomServiceClient(livekitHost, 'api-key', 'secret-key');

    ```

## List Participants

=== "Go"

    ```go
    res, err := roomClient.ListParticipants(context.Background(), &livekit.ListParticipantsRequest{
        Room: roomName,
    })

    ```

=== "Python"

    ```python
    from livekit.api import ListParticipantsRequest

    res = await lkapi.room.list_participants(ListParticipantsRequest(
        room=room_name
    ))

    ```

=== "Node.js"

    ```js
    const res = await roomService.listParticipants(roomName);

    ```

=== "LiveKit CLI"

    ```shell
    lk room participants list <ROOM_NAME>

    ```

## Get details on a Participant

=== "Go"

    ```go
    res, err := roomClient.GetParticipant(context.Background(), &livekit.RoomParticipantIdentity{
        Room:     roomName,
        Identity: identity,
    })

    ```

=== "Python"

    ```python
    from livekit.api import RoomParticipantIdentity

    res = await lkapi.room.get_participant(RoomParticipantIdentity(
        room=room_name,
        identity=identity,
    ))

    ```

=== "Node.js"

    ```js
    const res = await roomService.getParticipant(roomName, identity);

    ```

=== "LiveKit CLI"

    ```shell
    lk room participants get --room <ROOM_NAME> <ID>

    ```

## Updating permissions

您可以使用 `UpdateParticipant` 動態修改參與者的權限。當權限發生變化時，連線的用戶端將透過 `ParticipantPermissionChanged` 事件收到通知。

例如，當房間內的觀眾角色轉變為演講者角色時，這非常有用。

請注意，如果您撤銷參與者的 `CanPublish` 權限，他們發布的所有軌道將自動取消發布。

=== "Go"

    ```go
    // Promotes an audience member to a speaker
    res, err := c.UpdateParticipant(context.Background(), &livekit.UpdateParticipantRequest{
        Room: roomName,
        Identity: identity,
        Permission: &livekit.ParticipantPermission{
            CanSubscribe: true,
            CanPublish: true,
            CanPublishData: true,
        },
    })

    // ...and later move them back to audience
    res, err := c.UpdateParticipant(context.Background(), &livekit.UpdateParticipantRequest{
        Room: roomName,
        Identity: identity,
        Permission: &livekit.ParticipantPermission{
            CanSubscribe: true,
            CanPublish: false,
            CanPublishData: true,
        },
    })

    ```


=== "Python"

    ```python
    from livekit.api import UpdateParticipantRequest, ParticipantPermission

    # Promotes an audience member to a speaker
    await lkapi.room.update_participant(UpdateParticipantRequest(
        room=room_name,
        identity=identity,
        permission=ParticipantPermission(
            can_subscribe=True,
            can_publish=True,
            can_publish_data=True,
        ),
    ))

    # ...and later move them back to audience
    await lkapi.room.update_participant(UpdateParticipantRequest(
        room=room_name,
        identity=identity,
        permission=ParticipantPermission(
            can_subscribe=True,
            can_publish=False,
            can_publish_data=True,
        ),
    ))

    ```

=== "Node.js"

    ```js
    // Promotes an audience member to a speaker
    await roomService.updateParticipant(roomName, identity, undefined, {
        canPublish: true,
        canSubscribe: true,
        canPublishData: true,
    });

    // ...and later move them back to audience
    await roomService.updateParticipant(roomName, identity, undefined, {
        canPublish: false,
        canSubscribe: true,
        canPublishData: true,
    });

    ```

=== "LiveKit CLI"

    ```shell
    lk room participants update \
    --permissions '{"can_publish":true,"can_subscribe":true,"can_publish_data":true}' \
    --room <ROOM_NAME> \
    <ID>

    ```

## Updating metadata

您可以在必要時修改參與者的元資料。一旦發生改變，連線的客戶端將收到 `ParticipantMetadataChanged` 事件。

=== "Go"

    ```go
    data, err := json.Marshal(values)
    _, err = c.UpdateParticipant(context.Background(), &livekit.UpdateParticipantRequest{
        Room: roomName,
        Identity: identity,
        Metadata: string(data),
    })

    ```

=== "Python"

    ```python
    from livekit.api import UpdateParticipantRequest

    await lkapi.room.update_participant(UpdateParticipantRequest(
        room=room_name,
        identity=identity,
        metadata=json.dumps({"some": "values"}),
    ))

    ```

=== "Node.js"

    ```js
    const data = JSON.stringify({
    some: 'values',
    });

    await roomService.updateParticipant(roomName, identity, data);

    ```

=== "LiveKit CLI"

    ```shell
    lk room participants update \
    --metadata '{"some":"values"}' \
    --room <ROOM_NAME> \
    <ID>

    ```

## Remove a Participant

`RemoteParticipant` 將強制斷開參與者與房間的連接。但是，此操作不會使參與者的 token 無效。

為了防止參與者重新加入同一房間，請考慮以下措施：

- 產生具有短 TTL (生存時間)的存取權杖(access token)。
- 不要透過應用程式的後端向同一參與者提供新的 token。

=== "Go"

    ```go
    res, err := roomClient.RemoveParticipant(context.Background(), &livekit.RoomParticipantIdentity{
        Room:     roomName,
        Identity: identity,
    })

    ```

=== "Python"

    ```python
    from livekit.api import RoomParticipantIdentity

    await lkapi.room.remove_participant(RoomParticipantIdentity(
        room=room_name,
        identity=identity,
    ))

    ```

=== "Node.js"

    ```js
    await roomService.removeParticipant(roomName, identity);

    ```

=== "LiveKit CLI"

    ```shell
    lk room participants remove <ID>

    ```

## Mute/unmute a Participant's Track

若要將參與者的某個特定軌道靜音，請先從 `GetParticipant`（上文）取得 TrackSid，然後呼叫 `MutePublishedTrack`：

=== "Go"

    ```go
    res, err := roomClient.MutePublishedTrack(context.Background(), &livekit.MuteRoomTrackRequest{
        Room:     roomName,
        Identity: identity,
        TrackSid: "track_sid",
        Muted:    true,
    })

    ```

=== "Python"

    ```python
    from livekit.api import MuteRoomTrackRequest

    await lkapi.room.mute_published_track(MuteRoomTrackRequest(
        room=room_name,
        identity=identity,
        track_sid="track_sid",
        muted=True,
    ))

    ```


=== "Node.js"

    ```js
    await roomService.mutePublishedTrack(roomName, identity, 'track_sid', true);

    ```

=== "LiveKit CLI"

    ```shell
    lk room mute-track \
    --room <ROOM_NAME> \
    --identity <ID> \
    <TRACK_SID>

    ```

您也可以將 `muted` 設為 `false` 來取消軌道靜音。

!!! info
    遠端取消靜音可能會讓使用者感到意外，因此預設情況下它是禁用的。

    若要允許遠端取消靜音，請在[專案設定](https://cloud.livekit.io/projects/p_/settings/project)中選擇 `Admins can remotely unmute tracks` 選項。

    如果您是自託管(self-hosting)，請在設定 YAML 中設定 `room.enable_remote_unmute：true`。
