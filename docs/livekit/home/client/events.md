# Handling events

> 觀察並回應 LiveKit SDK 中的事件。

## Overview

LiveKit SDK 使用事件與房間內正在發生的應用程式變更進行通訊。

有兩種類型的事件，**room events** 和 **participant events**。房間事件由主 `Room` 物件發出，反映房間內的任何變化。當特定參與者發生變化時，每個 `Participant` 都會發出參與者事件。

房間事件通常是參與者事件的超集。如您所見，一些事件在 `Room` 和 `Participant` 上都會觸發；這是故意的。這種重複的目的是為了使您的應用程式更容易組件化。例如，如果您有一個呈現參與者的 UI 元件，它應該只監聽該參與者範圍內的事件。

## Declarative UI

在即時、多用戶系統中，事件處理可能相當複雜。參與者可以加入或離開，各自發布軌道或將其靜音。為了簡化這一點，LiveKit 為大多數平台提供了對 [聲明式 UI](https://alexsidorenko.com/blog/react-is-declarative-what-does-it-mean/) 的內建支援。

使用聲明性 UI，您可以指定 UI 在特定狀態下的外觀，而不必擔心要套用的轉換順序。現代框架在檢測變化和僅呈現變化的內容方面非常有效率。

=== "React"

    我們提供了一些 hooks 和元件，使使用 React 變得更加簡單。

    - [useParticipant](https://docs.livekit.io/reference/components/react/hook/useparticipants.md) - 將參與者事件映射到狀態
    - [useTracks](https://docs.livekit.io/reference/components/react/hook/usetracks.md) - 傳回指定音訊或視訊軌道的目前狀態
    - [VideoTrack](https://docs.livekit.io/reference/components/react/component/videotrack.md) - 渲染視訊軌道的 React 元件
    - [RoomAudioRenderer](https://docs.livekit.io/reference/components/react/component/roomaudiorenderer.md) - 渲染所有音軌聲音的 React 元件

    ```tsx
    const Stage = () => {
    const tracks = useTracks([Track.Source.Camera, Track.Source.ScreenShare]);
    return (
        <LiveKitRoom
        {/* ... */}
        >
        // Render all video
        {tracks.map((track) => {
            <VideoTrack trackRef={track} />;
        })}
        // ...and all audio tracks.
        <RoomAudioRenderer />
        </LiveKitRoom>
    );
    };

    function ParticipantList() {
    // Render a list of all participants in the room.
    const participants = useParticipants();
    <ParticipantLoop participants={participants}>
        <ParticipantName />
    </ParticipantLoop>;
    }

    ```

=== "SwiftUI"

    Swift SDK 中的大多數核心物件（包括 `Room`, `Participant` 和 `TrackReference`）都實作了 `ObservableObject` 協議，因此它們可以立即用於 SwiftUI。

    為了實現最簡單的集成，[Swift Components SDK](https://github.com/livekit/components-swift) 包含基於 `.environmentObject` 構建的適用於現代 SwiftUI 應用的現成實用程式：

    - `RoomScope` - 創建並（optional）連接到 `Room`，解僱後離開
    - `ForEachParticipant` - 迭代當前房間中的每個 `Participant`，自動更新
    - `ForEachTrack` - 迭代目前參與者的每個 `TrackReference`，自動更新

    ```swift
    struct MyChatView: View {
        var body: some View {
            RoomScope(url: /* URL */,
                    token: /* Token */,
                    connect: true,
                    enableCamera: true,
                    enableMicrophone: true) {
                VStack {
                    ForEachParticipant { _ in
                        VStack {
                            ForEachTrack(filter: .video) { _ in
                                MyVideoView()
                                    .frame(width: 100, height: 100)
                            }
                        }
                    }
                }
            }
        }
    }

    struct MyVideoView: View {
    @EnvironmentObject private var trackReference: TrackReference

    var body: some View {
        VideoTrackView(trackReference: trackReference)
            .frame(width: 100, height: 100)
    }
    }

    ```

=== "Android Compose"

    `Room` 和 `Participant` 物件具有內建的 `Flow` 支援。任何標示 `@FlowObservable` 註解的屬性都可以用 `flow` 實用程式方法觀察。可以這樣使用：

    ```kotlin
    @Composable
    fun Content(
    room: Room
    ) {
    val remoteParticipants by room::remoteParticipants.flow.collectAsState(emptyMap())
    val remoteParticipantsList = remoteParticipants.values.toList()
    LazyRow {
        items(
            count = remoteParticipantsList.size,
            key = { index -> remoteParticipantsList[index].sid }
        ) { index ->
            ParticipantItem(room = room, participant = remoteParticipantsList[index])
        }
    }
    }

    @Composable
    fun ParticipantItem(
        room: Room,
        participant: Participant,
    ) {
    val videoTracks by participant::videoTracks.flow.collectAsState(emptyList())
    val subscribedTrack = videoTracks.firstOrNull { (pub) -> pub.subscribed } ?: return
    val videoTrack = subscribedTrack.second as? VideoTrack ?: return

    VideoTrackView(
        room = room,
        videoTrack = videoTrack,
    )
    }

    ```


=== "Flutter"

    Flutter 預設支援 [宣告式 UI](https://docs.flutter.dev/get-started/flutter-for/declarative)。 LiveKit SDK 以兩種方式通知變更：

    - ChangeNotifier - 通用變更通知。當您建立反應式 UI 並且只關心可能影響渲染的變更時，這很有用
    - EventsListener<Event> - 監聽器模式來監聽特定事件 (參見 [events.dart](https://github.com/livekit/client-sdk-flutter/blob/main/lib/src/events.dart))

    ```dart
    class RoomWidget extends StatefulWidget {
    final Room room;

    RoomWidget(this.room);

    @override
    State<StatefulWidget> createState() {
        return _RoomState();
    }
    }

    class _RoomState extends State<RoomWidget> {
    late final EventsListener<RoomEvent> _listener = widget.room.createListener();

    @override
    void initState() {
        super.initState();
        // used for generic change updates
        widget.room.addListener(_onChange);

        // Used for specific events
        _listener
        ..on<RoomDisconnectedEvent>((_) {
            // handle disconnect
        })
        ..on<ParticipantConnectedEvent>((e) {
            print("participant joined: ${e.participant.identity}");
        })
    }

    @override
    void dispose() {
        // Be sure to dispose listener to stop listening to further updates
        _listener.dispose();
        widget.room.removeListener(_onChange);
        super.dispose();
    }

    void _onChange() {
        // Perform computations and then call setState
        // setState will trigger a build
        setState(() {
        // your updates here
        });
    }

    @override
    Widget build(BuildContext context) => Scaffold(
        // Builds a room layout with a main participant in the center, and a row of
        // participants at the bottom.
        // ParticipantWidget is located here: https://github.com/livekit/client-sdk-flutter/blob/main/example/lib/widgets/participant.dart
        body: Column(
        children: [
            Expanded(
                child: participants.isNotEmpty
                    ? ParticipantWidget.widgetFor(participants.first)
                    : Container()),
            SizedBox(
            height: 100,
            child: ListView.builder(
                scrollDirection: Axis.horizontal,
                itemCount: math.max(0, participants.length - 1),
                itemBuilder: (BuildContext context, int index) => SizedBox(
                width: 100,
                height: 100,
                child: ParticipantWidget.widgetFor(participants[index + 1]),
                ),
            ),
            ),
        ],
        ),
    );
    }

    ```

## Events

下表捕獲了跨平台 SDK 可用的一組一致事件。除了這裡列出的之外，某些平台上可能還存在特定於平台的事件。

| Event | Description | Room Event | Participant Event |
|-------|-------------|------------|-------------------|
| `ParticipantConnected` | 遠端參與者(RemoteParticipant)在本地參與者(local participant)之後加入。| ✔️ |  |
| `ParticipantDisconnected` | 遠端參與者離開 | ✔️ |  |
| `Reconnecting` | 與伺服器的連線已中斷，正在嘗試重新連線。| ✔️ |  |
| `Reconnected` | 重新連線已成功 | ✔️ |  |
| `Disconnected` | 由於房間關閉或不可恢復的故障而與房間斷開連接 | ✔️ |  |
| `TrackPublished` | 本地參與者加入後，新的軌跡將發佈到房間 | ✔️ | ✔️ |
| `TrackUnpublished` | RemoteParticipant 已取消發布軌道 | ✔️ | ✔️ |
| `TrackSubscribed` | LocalParticipant 已訂閱一個軌道 | ✔️ | ✔️ |
| `TrackUnsubscribed` | 之前訂閱的軌道已取消訂閱 | ✔️ | ✔️ |
| `TrackMuted` | 軌道被靜音，本地軌道和遠端軌道都會觸發 | ✔️ | ✔️ |
| `TrackUnmuted` | 軌道取消靜音，本地軌道和遠端軌道均觸發 | ✔️ | ✔️ |
| `LocalTrackPublished` | 本地軌道已成功發布 | ✔️ | ✔️ |
| `LocalTrackUnpublished` | 本地軌道未發布 | ✔️ | ✔️ |
| `ActiveSpeakersChanged` | 當前活躍發言者已改變 | ✔️ |  |
| `IsSpeakingChanged` | 現任參與者已改變發言狀態 |  | ✔️ |
| `ConnectionQualityChanged` | 參與者的連接品質已更改 | ✔️ | ✔️ |
| `ParticipantAttributesChanged` | 參與者的屬性已更新 | ✔️ | ✔️ |
| `ParticipantMetadataChanged` | 參與者的元資料已更新 | ✔️ | ✔️ |
| `RoomMetadataChanged` | 與房間相關的元資料已更改 | ✔️ |  |
| `DataReceived` | 從另一個參與者或伺服器接收的數據 | ✔️ | ✔️ |
| `TrackStreamStateChanged` | 指示已訂閱的軌道是否因頻寬而暫停 | ✔️ | ✔️ |
| `TrackSubscriptionPermissionChanged` | 其中一個已訂閱的軌道已更改目前參與者的軌道級權限 | ✔️ | ✔️ |
| `ParticipantPermissionsChanged` | 當前參與者的權限發生變化時 | ✔️ | ✔️ |
