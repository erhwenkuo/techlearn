# Codecs and more

> 進階音訊和視訊主題。

## Video codec support

LiveKit支援多種視訊編解碼器，以滿足不同的應用需求：

- H.264
- VP8
- VP9 (including SVC)
- AV1 (including SVC)

可伸縮視訊編碼 (SVC) 是 VP9 和 AV1 等較新編解碼器的功能，具有以下優勢：

- 透過讓較高品質的層利用來自較低品質層的資訊來提高位元率效率。
- 無需等待關鍵影格即可實現即時圖層切換。
- 在單一串流中合併多個空間（解析度）和時間（幀速率）圖層。

當使用 VP9 或 AV1 時，SVC 會自動透過 L3T3_KEY `scalabilityMode`（三個空間和時間層）啟動。

您可以指定連接到房間時要使用的編解碼器。要了解更多信息，請參閱以下部分中的範例。

## Video quality presets

LiveKit 在建立視訊軌道時提供預設解析度。這些預設包括常見的解析度和寬高比：

- h720 (1280x720)
- h540 (960x540)
- h360 (640x360)
- h180 (320x180)

預設還包括為獲得最佳品質而建議的位元率和幀率。您可以根據需要使用這些預設或定義自訂參數。

=== "React"

    ```js
    const localParticipant = useLocalParticipant();

    const audioTrack = await createLocalAudioTrack();
    const audioPublication = await localParticipant.publishTrack(audioTrack, {
        red: false,
    });

    ```

=== "JavaScript"

    ```js
    const audioTrack = await createLocalAudioTrack();
    const audioPublication = await room.localParticipant.publishTrack(audioTrack, {
        red: false,
    });

    ```

## Video track configuration

LiveKit 透過兩個類別對影片軌道設定提供廣泛的控制：

- Capture settings: 設備選擇和功能（解析度、幀速率、面向模式）。
- Publish settings: 編碼參數（位元率、幀率、聯播層）。

配置這些設定的方法如下：

=== "JavaScript"

    ```typescript
    // Room defaults
    const room = new Room({
    videoCaptureDefaults: {
        deviceId: '',
        facingMode: 'user',
        resolution: {
        width: 1280,
        height: 720,
        frameRate: 30,
        },
    },
    publishDefaults: {
        videoEncoding: {
        maxBitrate: 1_500_000,
        maxFramerate: 30,
        },
        videoSimulcastLayers: [
        {
            width: 640,
            height: 360,
            encoding: {
            maxBitrate: 500_000,
            maxFramerate: 20,
            },
        },
        {
            width: 320,
            height: 180,
            encoding: {
            maxBitrate: 150_000,
            maxFramerate: 15,
            },
        },
        ],
    },
    });

    // Individual track settings
    const videoTrack = await createLocalVideoTrack({
    facingMode: 'user',
    resolution: VideoPresets.h720,
    });
    const publication = await room.localParticipant.publishTrack(videoTrack);

    ```

=== "Swift"

    ```swift
    // Room defaults
    var room = Room(
    delegate: self,
    roomOptions: RoomOptions(
            defaultCameraCaptureOptions: CameraCaptureOptions(
                position: .front,
                dimensions: .h720_169,
                fps: 30,
            ),
            defaultVideoPublishOptions: VideoPublishOptions(
                encoding: VideoEncoding(
                    maxBitrate: 1_500_000,
                    maxFps: 30,
                ),
                simulcastLayers: [
                    VideoParameters.presetH180_169,
                    VideoParameters.presetH360_169,
                ]
            ),
        )
    )

    // Individual track
    let videoTrack = try LocalVideoTrack.createCameraTrack(options: CameraCaptureOptions(
        position: .front,
        dimensions: .h720_169,
        fps: 30,
        ))

    let publication = localParticipant.publishVideoTrack(track: videoTrack)

    ```

## Video simulcast

同步廣播(Simulcast )支援發布具有不同位元速率設定檔的相同視訊軌道的多個版本。這使得 LiveKit 能夠根據每個接收者的頻寬和首選解析度動態轉送最合適的串流。

當偵測到頻寬限制時，LiveKit 會自動選擇適當的圖層，並隨著條件改善而升級到更高的解析度。

所有 LiveKit SDK 中都預設啟用同步廣播(Simulcast)，如有需要，可以在發佈設定中停用。

## Dynacast

當訂閱者沒有觀看視訊層時，動態廣播（Dynacast）會自動暫停視訊層發布。對於同步播放的視頻，如果訂閱者只使用中、低解析度層，則暫停高解析度發布。

若要啟用此頻寬最佳化：

=== "JavaScript"

    ```typescript
    const room = new Room({
        dynacast: true
    });

    ```

=== "Swift"

    ```swift
    let room = Room(
        delegate: self,
        roomOptions: RoomOptions(
                dynacast: true
            )
    )

    ```

=== "Android"

    ```kotlin
    val options = RoomOptions(
        dynacast = true
    )
    var room = LiveKit.create(
        options = options
    )

    ```

=== "Flutter"

    ```dart
    var room = Room(
        roomOptions: RoomOptions(
            dynacast: true
        ),
    )


使用 SVC 編解碼器（VP9 和 AV1）時，由於 SVC 編碼特性，Dynacast 只能暫停整個串流，而無法暫停單一層。

## Hi-fi audio

對於高品質的音訊串流，LiveKit 提供了多種配置選項來優化音訊品質。

#### Recommended hi-fi settings

為了獲得高品質的音頻，我們提供了具有建議設定的預設：

=== "React"

    ```js
    const localParticipant = useLocalParticipant();

    const audioTrack = await createLocalAudioTrack({
        channelCount: 2,
        echoCancellation: false,
        noiseSuppression: false,
        });
    const audioPublication = await localParticipant.publishTrack(audioTrack, {
        audioPreset: AudioPresets.musicHighQualityStereo,
        dtx: false,
        red: false,
        });

    ```

=== "JavaScript"

    ```js
    const audioTrack = await createLocalAudioTrack({
        channelCount: 2,
        echoCancellation: false,
        noiseSuppression: false,
    });

    const audioPublication = await room.localParticipant.publishTrack(audioTrack, {
        audioPreset: AudioPresets.musicHighQualityStereo,
        dtx: false,
        red: false,
    });

    ```

#### Maximum quality settings

LiveKit 支援高達 510kbps 立體聲的音軌 - 這是理論上的最高品質。請注意，聽眾的播放堆疊可能會對音訊重新取樣，因此實際播放品質可能低於發布品質。相較之下，對於 Spotify 等音樂串流服務來說，256kbps AAC 編碼音訊被認為是高品質的。

=== "React"

    ```js
    const localParticipant = useLocalParticipant();

    const audioTrack = await createLocalAudioTrack({
        channelCount: 2,
        echoCancellation: false,
        noiseSuppression: false,
    });

    const audioPublication = await localParticipant.publishTrack(audioTrack, {
        audioBitrate: 510000,
        dtx: false,
        red: false,
    });

    ```

=== "JavaScript"

    ```js
    const audioTrack = await createLocalAudioTrack({
        channelCount: 2,
        echoCancellation: false,
        noiseSuppression: false,
    });

    const audioPublication = await room.localParticipant.publishTrack(audioTrack, {
        audioBitrate: 510000,
        dtx: false,
        red: false,
    });

    ```

如果您配置了高位元率，我們建議在真實條件下進行測試，以找到最適合您的用例的設定。

## Audio RED

冗餘編碼是一種透過在不同的資料包中發送相同音訊資料的多個副本來提高音訊品質的技術。這在可能遺失資料包的有損網路中很有用。然後接收器可以使用冗餘資料包來重建原始音訊資料包。

冗餘編碼會增加頻寬使用量，以實現更高的音訊品質。 LiveKit 建議啟用此功能，因為音訊故障非常令人分心，因此這種權衡幾乎總是值得的。如果您的用例優先考慮頻寬並且可以容忍音訊故障，則可以停用 RED。

#### Disabling Audio RED when publishing

您可以在發布新音軌時停用 Audio RED：

=== "React"

    ```js
    const localParticipant = useLocalParticipant();

    const audioTrack = await createLocalAudioTrack();
    const audioPublication = await localParticipant.publishTrack(audioTrack, {
        red: false,
    });

    ```

=== "JavaScript"

    ```js
    const audioTrack = await createLocalAudioTrack();
    const audioPublication = await room.localParticipant.publishTrack(audioTrack, {
        red: false,
    });

    ```

=== "Swift"

    ```swift
    let audioTrack = LocalAudioTrack.createTrack()
    let audioPublication = room.localParticipant.publish(audioTrack: audioTrack, options: AudioPublishOptions(red: false))

    ```

=== "Android"

    ```kotlin
    val audioTrack = localParticipant.createAudioTrack()
    coroutineScope.launch {
        val publication = localParticipant.publishAudioTrack(
            track = localAudioTrack,
            red = false
        )
    }

    ```