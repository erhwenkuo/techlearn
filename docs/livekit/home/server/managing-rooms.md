
# Managing rooms

> 從後端伺服器建立、列出和刪除房間。

## Initialize RoomServiceClient

房間管理由 RoomServiceClient 完成，建立如下：

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

## Create a room

=== "Python"

    ```python
    from livekit.api import CreateRoomRequest

    room = await lkapi.room.create_room(CreateRoomRequest(
        name="myroom",
        empty_timeout=10 * 60,
        max_participants=20,
    ))

    ```

=== "Node.js"

    ```js
    const opts = {
        name: 'myroom',
        emptyTimeout: 10 * 60, // 10 minutes
        maxParticipants: 20,
    };
    roomService.createRoom(opts).then((room: Room) => {
        console.log('room created', room);
    });

    ```

=== "Go"

    ```go
    room, _ := roomClient.CreateRoom(context.Background(), &livekit.CreateRoomRequest{
        Name:            "myroom",
        EmptyTimeout:    10 * 60, // 10 minutes
        MaxParticipants: 20,
    })

    ```

=== "LiveKit CLI"

    ```shell
    lk room create --empty-timeout 600 myroom

    ```

## List rooms


=== "Python"

    ```python
    from livekit.api import ListRoomsRequest

    rooms = await lkapi.room.list_rooms(ListRoomsRequest())

    ```

=== "Node.js"

    ```js
    roomService.listRooms().then((rooms: Room[]) => {
        console.log('existing rooms', rooms);
    });

    ```

=== "Go"

    ```go
    rooms, _ := roomClient.ListRooms(context.Background(), &livekit.ListRoomsRequest{})

    ```


=== "LiveKit CLI"

    ```shell
    lk room list

    ```

## Delete a room

刪除房間會導致所有參與者斷開連接。


=== "Python"

    ```python
    from livekit.api import DeleteRoomRequest

    await lkapi.room.delete_room(DeleteRoomRequest(
        room="myroom",
    ))

    ```

=== "Node.js"

    ```js
    // Delete a room
    roomService.deleteRoom('myroom').then(() => {
        console.log('room deleted');
    });

    ```

=== "Go"

    ```go
    _, _ = roomClient.DeleteRoom(context.Background(), &livekit.DeleteRoomRequest{
        Room: "myroom",
    })

    ```

=== "LiveKit CLI"

    ```shell
    lk room delete myroom

    ```