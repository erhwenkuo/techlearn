# Connecting to LiveKit

> 了解如何連接 realtime SDK。

## Overview

您的應用程式將使用 `Room` 物件連接到 LiveKit，該物件是 LiveKit 中的基本建構。可以將其想像成電話會議——多個參與者可以加入一個房間並相互共享即時音訊、視訊和數據。

根據您的應用程序，每個參與者 (participant) 可能代表一個用戶、一個人工智慧代理、一個連接的設備或您創建的其他程式。房間內的參與者數量沒有限制，每個參與者都可以向房間發布音訊、視訊和數據。

## Installing the LiveKit SDK

LiveKit 包含適用於每個主要平台的開源 SDK，包括 JavaScript, Swift, Android, React Native, Flutter 和 Unity。

LiveKit 也為 Python, Node.js, Go 和 Rust 中的即時後端應用程式提供 SDK。它們旨在與 [Agents framework](https://docs.livekit.io/agents.md) 一起用於 realtime AI 應用程式。

=== "JavaScript"

    安裝 LiveKit SDK 和 optional 的 React Components 函式庫：

    ```shell
    npm install livekit-client @livekit/components-react @livekit/components-styles --save

    ```

    該 SDK 也可使用 `yarn` 或 `pnpm`。

    如果您使用其中一個平台，請查看 [React](https://docs.livekit.io/home/quickstarts/react.md) 或 [Next.js](https://docs.livekit.io/home/quickstarts/nextjs.md) 的專用快速入門。


=== "Swift"

    使用 Swift 套件管理器將 Swift SDK 和可選的 Swift 元件庫新增到您的專案中。軟體包 URL 為：

    - [https://github.com/livekit/client-sdk-swift](https://github.com/livekit/client-sdk-swift)
    - [https://github.com/livekit/components-swift](https://github.com/livekit/components-swift)

    有關更多詳細信息，請參閱[Adding package dependencies to your app](https://developer.apple.com/documentation/xcode/adding-package-dependencies-to-your-app)。

    如果需要，您也必須在 `Info.plist` 檔案中聲明攝影機和麥克風權限：

    ```xml
    <dict>
    ...
    <key>NSCameraUsageDescription</key>
    <string>$(PRODUCT_NAME) uses your camera</string>
    <key>NSMicrophoneUsageDescription</key>
    <string>$(PRODUCT_NAME) uses your microphone</string>
    ...
    </dict>

    ```

    更多詳細信息，請參閱 [Swift 快速入門](https://docs.livekit.io/home/quickstarts/swift.md)。

=== "Android"

    LiveKit SDK 和元件庫以 Maven 套件的形式提供。

    ```groovy
    dependencies {
    implementation "io.livekit:livekit-android:2.+"
    implementation "io.livekit:livekit-android-compose-components:1.+"
    }

    ```

    有關 SDK 最新版本的信息，請參閱[releases page](https://github.com/livekit/client-sdk-android/releases)。

    您還需要 JitPack 作為您的存儲庫之一。在你的 `settings.gradle` 檔案中：

    ```groovy
    dependencyResolutionManagement {
        repositories {
            //...
            maven { url 'https://jitpack.io' }
        }
    }

    ```


=== "React Native"

    使用 NPM 安裝 React Native SDK：

    ```shell
    npm install @livekit/react-native @livekit/react-native-webrtc livekit-client

    ```

    請參閱 [Expo](https://docs.livekit.io/home/quickstarts/expo.md) 或 [React Native](https://docs.livekit.io/home/quickstarts/react-native.md) 的專用快速入門以了解更多詳細資訊。


=== "Flutter"

    安裝最新版本的 Flutter SDK 和元件庫。

    ```shell
    flutter pub add livekit_client livekit_components

    ```

    您還需要聲明攝影機和麥克風權限。更多詳細信息，請參閱 [Flutter 快速入門](https://docs.livekit.io/home/quickstarts/flutter.md)。

    如果您的 SDK 未列在上面，請查看 [平台特定的快速入門](https://docs.livekit.io/home/quickstarts.md) 和 [SDK 參考文件](https://docs.livekit.io/reference.md) 的完整清單以獲取更多詳細資訊。

## Connecting to a room

房間透過其名稱(name)來識別，名稱可以是任何唯一的字串。當第一個參與者加入時，房間本身會自動創建，而當最後一個參與者離開時，房間就會關閉。

連接到房間時必須使用參與者身分。該身分可以是任何字串，但對於每個參與者來說必須是唯一的。

連接到房間始終需要兩個參數：

- `wsUrl`: 您的 LiveKit 伺服器的 WebSocket URL。 - LiveKit Cloud 使用者可以在 [project](https://cloud.livekit.io/projects/p_/settings/project) 上找到他們的 URL。
- 遵循 [self-hosting](https://docs.livekit.io/home/self-hosting/local.md) 的自架用戶可以在開發時使用 `ws://localhost:7880`。
- `token`: 每位參與者必須使用一個唯一的 [access token](https://docs.livekit.io/concepts/authentication.md) 進行連接。此令牌對房間名稱、參與者的身分及其權限進行編碼。
- 有關產生令牌的協助，請參閱[generating-tokens](https://docs.livekit.io/home/server/generating-tokens.md)。

=== "JavaScript"

    ```js
    const room = new Room();
    await room.connect(wsUrl, token);

    ```



=== "React"

    ```js
    <LiveKitRoom audio={true} video={true} token={token} serverUrl={wsUrl}>
    <!-- your components here -->
    </LiveKitRoom>

    ```



=== "Swift"

    ```swift
    RoomScope(url: wsURL, token: token, connect: true, enableCamera: true) {
    // your components here
    }

    ```

=== "Android"

    ```kotlin
    RoomScope(
    url = wsURL,
    token = token,
    audio = true,
    video = true,
    connect = true,
    ) {
    // your components here
    }

    ```

=== "React Native"

    ```js
    <LiveKitRoom audio={true} video={true} token={token} serverUrl={wsUrl}>
    <!-- your components here -->
    </LiveKitRoom>

    ```

=== "Flutter"

    ```dart
    final room = Room();
    await room.connect(wsUrl, token);

    ```

連線成功後，`Room` 物件將包含兩個關鍵屬性： `localParticipant` 物件（代表目前使用者）和 `remoteParticipants` 物件（房間中其他參與者的陣列）。

連線後，您可以 [publish](https://docs.livekit.io/home/client/tracks/publish.md) 和 [subscribe](https://docs.livekit.io/home/client/tracks/subscribe.md) 即時媒體軌道或與其他參與者[交換資料](https://docs.livekit.io/home/client/data.md)。

LiveKit 也會在 `Room` 物件上發出許多事件，例如新參與者加入或發布媒體軌道時。有關詳細信息，請參閱[處理事件](https://docs.livekit.io/home/client/events.md)。

## Disconnection

呼叫 `Room.disconnect()` 離開房間。如果您在未呼叫 `disconnect()` 的情況下終止應用程序，則參與者將在 15 秒後消失。

!!! info
    在某些平台上，包括 JavaScript 和 Swift，當應用程式退出時會自動呼叫 `Room.disconnect`。

### Automatic disconnection

由於伺服器發起的操作，參與者可能會與房間斷開連接。如果使用 [DeleteRoom](https://docs.livekit.io/home/server/managing-rooms.md#Delete-a-room) API 關閉房間，或使用 [RemoveParticipant](https://docs.livekit.io/home/server/managing-participants.md#remove-a-mdicant)

在這種情況下，會發出 `Disconnected` 事件，提供斷開連線的原因。常見的[斷線原因](https://github.com/livekit/protocol/blob/main/protobufs/livekit_models.proto#L333)包含：

- **DUPLICATE_IDENTITY**: 由於另一位具有相同身份的參與者加入了房間，因此連接已斷開。
- **ROOM_DELETED**: 該房間已透過 `DeleteRoom` API關閉。
- **PARTICIPANT_REMOVED**: 使用 `RemoveParticipant` API 從房間中移除。
- **JOIN_FAILURE**: 無法連接到房間，可能是由於網路問題。
- **ROOM_CLOSED**: 由於所有 [Standard and Ingress participants](https://docs.livekit.io/home/get-started/api-primitives.md#types-of-participants) 均已離開，因此房間已關閉。

## Connection reliability

LiveKit 可在各種網路條件下實現可靠的連線。它按降序嘗試以下 WebRTC 連線類型：

1. **ICE** over **UDP**: 理想的連接類型，適用於大多數情況
2. **TURN** with **UDP** (3478): 當 ICE/UDP 不可達時使用
3. **ICE** over **TCP**: 當網路不允許 UDP 時使用（即透過 VPN 或公司防火牆）
4. **TURN** with **TLS**: 當防火牆僅允許出站 TLS 連線時使用

=== "Cloud"
    LiveKit Cloud 支援以上所有連線類型。帶有 TLS 的 TURN 伺服器由 LiveKit Cloud 提供和維護。

=== "Self-hosted"

    ICE 透過 UDP 和 TCP 開箱即用，而 TURN 需要額外的設定和您自己的 SSL 憑證。

### Network changes and reconnection

使用 WiFi 和手機網路時，使用者有時可能會遇到網路變化，導致與伺服器的連線中斷。這可能包括從 WiFi 切換到蜂窩網路或經過連接較差的地方。

當發生這種情況時，LiveKit 將嘗試自動恢復連線。它重新連線到訊號 WebSocket 並為 WebRTC 連線發起 [ICE 重新啟動](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Session_lifetime#ice_restart)。這個過程通常不會對使用者造成任何干擾，甚至對使用者造成極小的干擾。但是，如果透過先前的連線傳輸媒體失敗，用戶可能會注意到影片暫時暫停幾秒鐘，直到建立新的連線。

在 ICE 重新啟動不可行或不成功的情況下，LiveKit 將執行完全重新連線。由於完全重新連接需要更多時間並且可能造成更大的破壞，因此會觸發 `Reconnecting` 事件。這允許您的應用程式在重新連接過程中做出回應（可能透過顯示 UI 元素）。

該序列如下：

1. `ParticipantDisconnected` 當房間內其他參與者已斷開連接時觸發
2. 如果有未發布的軋道，您將收到 `LocalTrackUnpublished`
3. 觸發 `Reconnecting`
4. 執行完全重新連接
5. 觸發 `Reconnected`
6. 對於目前在房間裡的每個人，您將收到 `ParticipantConnected`
7. 本地軌道重新發布，發出 `LocalTrackPublished` 事件

本質上，完整的重新連接序列與其他人離開房間並返回的序列相同。