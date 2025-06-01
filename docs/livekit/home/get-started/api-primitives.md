# Rooms, participants, and tracks

> LiveKit 中核心 API 原生結構的指南。

## Overview

LiveKit 只有三個核心結構：`房間(room)`、`參與者(participant)` 和 `媒體資料軌(track)`。房間只是一個或多個參與者之間的即時會話。參與者可以發布一個或多個媒體資料軌和/或訂閱另一個參與者的一個或多個媒體資料軌。


## Room

`Room` 是表示 LiveKit 會話的容器物件。

房間中的每個參與者都會收到有關同一房間中其他參與者的變化的更新。例如，當參與者新增、刪除或修改軌道的狀態（例如靜音）時，其他參與者會收到此變更的通知。這是一種強大的同步狀態機制，也是建立任何即時體驗的基礎。

可以透過 [server API](https://docs.livekit.io/home/server/managing-rooms.md#create-a-room) 手動建立房間，也可以在第一個參與者加入時自動建立房間。一旦最後一位參與者離開房間，房間會在短暫的延遲後關閉。


## Participant

`Participant` 是參與即時會話的使用者或進程。它們由開發人員提供的唯一 `identity` 和伺服器生成的 `sid` 來表示。參與者物件還包含有關其狀態和已發布的媒體資料軌的元資料。

> ❗ **Important**
> 
> 每個房間的參與者身分都是唯一的。因此，如果具有相同身分的參與者加入房間，則只會保留最近加入的參與者；伺服器會自動中斷使用該身分的其他參與者的連線。

SDK 中有兩種 participant objects:

- `LocalParticipant` 代表當前用戶，預設情況下，可以在房間中發布媒體資料軌。
- `RemoteParticipant` 代表遠端使用者。本地參與者預設可以訂閱遠端參與者發布的任何媒體資料軌。

參與者也可以與一個或多個其他參與者來 [exchange data](https://docs.livekit.io/home/client/data.md)。

### Participant fields

| Field | Type | Description |
|-------|------|-------------|
| sid | string | 此特定參與者的 UID，由 LiveKit 伺服器產生。 |
| identity | string | 參與者的唯一身份，連線時指定。 |
| name | string | Optional 顯示名稱。|
| state | ParticipantInfo.State | JOINING, JOINED, ACTIVE, 或 DISCONNECTED. |
| kind | ParticipantInfo.Kind | 參與者的類型；更多內容請見下文。 |
| attributes | string | 參與者的使用者指定[屬性](https://docs.livekit.io/home/client/data.md)。 |
| permission | ParticipantInfo.Permission | 授予參與者的權限。 |

### Types of participants

在即時會話中，參與者可以代表最終用戶以及伺服器端進程。可以使用 `kind` 欄位​​來區分它們：

- `STANDARD`: 常規參與者，通常是應用程式中的最終用戶。
- `AGENT`: 使用 [Agents 框架](https://docs.livekit.io/agents.md) 產生的代理程式。
- `SIP`: 透過 [SIP](https://docs.livekit.io/sip.md) 連線的電話使用者。
- `EGRESS`: 使用 [LiveKit Egress](https://docs.livekit.io/home/egress/overview.md) 記錄會話的伺服器端進程。
- `INGRESS`: 使用 [LiveKit Ingress](https://docs.livekit.io/home/ingress/overview.md) 將媒體提取到會話中的伺服器端進程。

## Track

`Track` 代表媒體資訊流，可以是音訊、視訊或自訂資料。預設情況下，房間中的參與者可以發布軌道，例如他們的攝影機或麥克風串流，並訂閱其他參與者發布的一個或多個軌道。為了對本地參與者可能未訂閱的軌道進行建模，所有軌道物件都有一個相應的 `TrackPublication` 物件:

- `Track`: 圍繞原生 WebRTC `MediaStreamTrack` 的包裝器，代表可播放的軌道。
- `TrackPublication`: 已發佈到伺服器的軌道。如果該軌道被本地參與者訂閱並可在本地播放，它將具有一個表示相關 `Track` 物件的 `.track` 屬性。

我們現在可以列出和操作其他參與者發布的軌道（透過 track publications），即使本地參與者未訂閱它們。

### TrackPublication fields

`TrackPublication` 包含有關其關聯軌道的資訊：

| Field | Type | Description |
|-------|------|-------------|
| sid | string | 此特定軌道的 UID，由 LiveKit 伺服器產生。 |
| kind | Track.Kind | 軌道的類型，無論是音訊、視訊或任意資料。|
| source | Track.Source | 媒體來源：攝影機、麥克風、ScreenShare 或 ScreenShareAudio。|
| name | string | 該特定軌道首次發佈時所賦予的名稱。|
| subscribed | boolean | 指示該軌道是否已由本地參與者訂閱。|
| track | Track | 如果本地參與者已訂閱，則關聯的 `Track` 物件代表 WebRTC 軌道。|
| muted | boolean | 本地參與者是否將此軌道靜音。靜音時，它不會從伺服器接收新位元組。 |

### Track subscription

當參與者訂閱某個軌道（該軌道尚未被發布參與者靜音）時，他們會持續接收其資料。如果參與者取消訂閱，他們將停止接收該軌道的媒體，並可隨時重新訂閱。

當參與者建立或加入房間時，`autoSubscribe` 選項預設為 `true`。這意味著參與者會自動訂閱所有現有的已發布軌道以及將來發布的任何軌道。為了對軌道訂閱進行更細微的控制，您可以將 `autoSubscribe` 設定為 `false`，並使用 [selective subscriptions](https://docs.livekit.io/home/client/receive.md#selective-subscription)。

??? info
    對於大多數用例，通常建議在發布者端將軌道靜音或在訂閱者端取消訂閱，而不是取消發布。發佈軌道需要協商階段，因此首字節傳輸時間效能較差。
