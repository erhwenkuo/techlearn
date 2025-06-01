# Room metadata

> 與所有參與者共享特定於應用程式的狀態。

## Overview

與 [Participant metadata](./participant-attributes.md) 類似，Rooms 也具有一個元資料字段，用於儲存所有參與者可見的應用程式特定資料。

房間元資料只能使用伺服器 API 進行設置，但房間內的所有參與者都可以使用 LiveKit SDK 進行存取。

若要設定房間元數據，請使用 [CreateRoom](https://docs.livekit.io/home/server/managing-rooms.md#create-a-room) 和 [UpdateRoomMetadata](https://docs.livekit.io/server/room-management.#updateroommetadata) API。

若要訂閱更新，您必須 [handle](https://docs.livekit.io/home/client/events.md#events) `RoomMetadataChanged` 事件。