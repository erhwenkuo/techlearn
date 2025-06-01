# Camera & microphone

> 從任何裝置發佈即時音訊和視訊。

## Overview

LiveKit 包含一種簡單且一致的方法來發布使用者的攝影機和麥克風，無論他們使用什麼裝置或瀏覽器。在所有情況下，LiveKit 在錄製處於活動狀態時都會顯示正確的指示並從使用者那裡獲取必要的權限。

```typescript
// Enables the camera and publishes it to a new video track
room.localParticipant.setCameraEnabled(true);

// Enables the microphone and publishes it to a new audio track
room.localParticipant.setMicrophoneEnabled(true);

```

## Device permissions

在原生和行動應用程式中，您通常需要獲得用戶的同意才能存取麥克風或攝影機。 LiveKit 與系統隱私設定集成，以記錄權限並在音訊或視訊擷取處於活動狀態時顯示正確的指示器。

對於網路瀏覽器，當您的應用程式第一次嘗試存取攝影機和麥克風權限時，系統會自動提示使用者授予權限，無需進行其他配置。

=== "Swift"

    將這些條目新增至您的 `Info.plist`：

    ```xml
    <key>NSCameraUsageDescription</key>
    <string>$(PRODUCT_NAME) uses your camera</string>
    <key>NSMicrophoneUsageDescription</key>
    <string>$(PRODUCT_NAME) uses your microphone</string>

    ```

    要啟用背景音頻，您還必須添加 `Background Modes` 功能並選擇 "Audio, AirPlay, and Picture in Picture"。

    你的 `Info.plist` 應該包含：

    ```xml
    <key>UIBackgroundModes</key>
    <array>
    <string>audio</string>
    </array>

    ```

=== "Android"

    將這些權限加入您的 `AndroidManifest.xml`：

    ```xml
    <uses-feature android:name="android.hardware.camera" />
    <uses-feature android:name="android.hardware.camera.autofocus" />
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-permission android:name="android.permission.RECORD_AUDIO" />
    <uses-permission android:name="android.permission.MODIFY_AUDIO_SETTINGS" />

    ```

    Runtime 請求權限：

    ```kotlin
    private fun requestPermissions() {
        val requestPermissionLauncher =
            registerForActivityResult(
                ActivityResultContracts.RequestMultiplePermissions()
            ) { grants ->
                for (grant in grants.entries) {
                    if (!grant.value) {
                        Toast.makeText(
                            this,
                            "Missing permission: ${grant.key}",
                            Toast.LENGTH_SHORT
                        ).show()
                    }
                }
            }

        val neededPermissions = listOf(
            Manifest.permission.RECORD_AUDIO,
            Manifest.permission.CAMERA
        ).filter {
            ContextCompat.checkSelfPermission(
                this,
                it
            ) == PackageManager.PERMISSION_DENIED
        }.toTypedArray()

        if (neededPermissions.isNotEmpty()) {
            requestPermissionLauncher.launch(neededPermissions)
        }
    }

    ```

=== "React Native"

    對於 iOS，添加到 `Info.plist`：

    ```xml
    <key>NSCameraUsageDescription</key>
    <string>$(PRODUCT_NAME) uses your camera</string>
    <key>NSMicrophoneUsageDescription</key>
    <string>$(PRODUCT_NAME) uses your microphone</string>

    ```

    對於 Android，添加到 `AndroidManifest.xml`：

    ```xml
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-permission android:name="android.permission.RECORD_AUDIO" />
    <uses-permission android:name="android.permission.MODIFY_AUDIO_SETTINGS" />

    ```

    您需要在執行時間使用權限庫（如 `react-native-permissions` ）請求權限。


=== "Flutter"

    對於 iOS，添加到 `Info.plist`：

    ```xml
    <key>NSCameraUsageDescription</key>
    <string>$(PRODUCT_NAME) uses your camera</string>
    <key>NSMicrophoneUsageDescription</key>
    <string>$(PRODUCT_NAME) uses your microphone</string>

    ```

    對於 Android，添加到 `AndroidManifest.xml`：

    ```xml
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-permission android:name="android.permission.RECORD_AUDIO" />
    <uses-permission android:name="android.permission.MODIFY_AUDIO_SETTINGS" />

    ```

    使用 `permission_handler` 套件請求權限：

    ```dart
    import 'package:permission_handler/permission_handler.dart';

    // Request permissions
    await Permission.camera.request();
    await Permission.microphone.request();

    ```

## Mute and unmute

您可以使任何軌道靜音以阻止其向伺服器發送資料。當某個軌道靜音時，LiveKit 將在房間內的所有參與者上觸發 `TrackMuted` 事件。您可以使用此事件來更新應用程式的 UI，並向房間中的所有使用者反映正確的狀態。

使用其對應的 `LocalTrackPublication` 物件將軌道靜音/取消靜音。

## Track permissions

預設情況下，任何已發布的軌道都可以被所有參與者訂閱。但是，發布者可以使用曲目訂閱權限來限制誰可以訂閱他們的軌道：

=== "JavaScript"

    ```typescript
    localParticipant.setTrackSubscriptionPermissions(false, [
    {
        participantIdentity: 'allowed-identity',
        allowAll: true,
    },
    ]);

    ```

=== "Swift"

    ```swift
    localParticipant.setTrackSubscriptionPermissions(
        allParticipantsAllowed: false,
        trackPermissions: [
            ParticipantTrackPermission(participantSid: "allowed-sid", allTracksAllowed: true)
        ]
    )

    ```

=== "Android"

    ```kotlin
    localParticipant.setTrackSubscriptionPermissions(false, listOf(
        ParticipantTrackPermission(participantIdentity = "allowed-identity", allTracksAllowed = true),
    ))

    ```


=== "Flutter"

    ```dart
    localParticipant.setTrackSubscriptionPermissions(
    allParticipantsAllowed: false,
    trackPermissions: [
        const ParticipantTrackPermission('allowed-identity', true, null)
    ],
    );

    ```

=== "Python"

    ```python
    from livekit import rtc

    local_participant.set_track_subscription_permissions(
        all_participants_allowed=False,
        participant_permissions=[
            rtc.ParticipantTrackPermission(
                participant_identity="allowed-identity",
                allow_all=True,
            ),
        ],
    )

    ```

## Publishing from backend

您還可以從後端進程發布音訊和視訊軌道，這些軌道可以像任何攝影機或麥克風軌道一樣使用。[LiveKit Agents](https://docs.livekit.io/agents.md) 框架可以輕鬆地將可程式參與者新增至任何房間，並發布合成語音或視訊等媒體。

LiveKit 也包含適用於 [Go](https://github.com/livekit/server-sdk-go), [Rust](https://github.com/livekit/rust-sdks), [Python](https://github.com/livekit/python-sdks) 和 [Node.Kjs](https://github.com/livekit/python-sdks) 和 [live.jjs](https://github.com/sdkS7de/sd3)伺服器的完整環境。

您也可以使用 [LiveKit CLI](https://github.com/livekit/livekit-cli?tab=readme-ov-file#publishing-to-a-room) 發佈媒體。

### Publishing audio tracks

您可以透過建立 `AudioSource` 並將其發佈為曲目來發布音訊。

音訊流以指定的取樣率和通道數承載原始 PCM 資料。發布音訊涉及將串流分割成可配置長度的音訊幀。內部緩衝區保存 50 毫秒的排隊音訊以傳送到即時堆疊。用於傳送新訊框的 `capture_frame` 方法處於阻塞狀態，並且直到緩衝區接收整個訊框後才會傳回控制。這使得中斷處理變得更加容易。

為了發布音軌，您需要預先確定取樣率和通道數，以及每個畫面的長度（取樣數）。在以下範例中，代理程式以 10 毫秒長的幀傳輸 48kHz 的恆定 16 位元正弦波：

=== "Python"

    ```python
    import numpy as np

    from livekit import agents,rtc

    SAMPLE_RATE = 48000
    NUM_CHANNELS = 1 # mono audio
    AMPLITUDE = 2 ** 8 - 1
    SAMPLES_PER_CHANNEL = 480 # 10 ms at 48kHz

    async def entrypoint(ctx: agents.JobContext):
        await ctx.connect()

        source = rtc.AudioSource(SAMPLE_RATE, NUM_CHANNELS)
        track = rtc.LocalAudioTrack.create_audio_track("example-track", source)
        # since the agent is a participant, our audio I/O is its "microphone"
        options = rtc.TrackPublishOptions(source=rtc.TrackSource.SOURCE_MICROPHONE)
        # ctx.agent is an alias for ctx.room.local_participant
        publication = await ctx.agent.publish_track(track, options)

        frequency = 440
        async def _sinewave():
            audio_frame = rtc.AudioFrame.create(SAMPLE_RATE, NUM_CHANNELS, SAMPLES_PER_CHANNEL)
            audio_data = np.frombuffer(audio_frame.data, dtype=np.int16)

            time = np.arange(SAMPLES_PER_CHANNEL) / SAMPLE_RATE
            total_samples = 0
            while True:
                time = (total_samples + np.arange(SAMPLES_PER_CHANNEL)) / SAMPLE_RATE
                sinewave = (AMPLITUDE * np.sin(2 * np.pi * frequency * time)).astype(np.int16)
                np.copyto(audio_data, sinewave)

                # send this frame to the track
                await source.capture_frame(audio_frame)
                total_samples += SAMPLES_PER_CHANNEL

        await _sinewave()

    ```

!!! warning
    當串流傳輸有限音訊（例如，來自檔案）時，請確保幀長度不長於剩餘要串流的樣本數，否則緩衝區的末端將包含雜訊。

#### Audio examples

有關使用 LiveKit SDK 的音訊範例，請參閱 GitHub 儲存庫中的以下內容：

- **[Manipulate TTS audio](https://github.com/livekit/agents/blob/main/examples/voice_agents/speedup_output_audio.py)**: 使用 [TTS node](https://docs.livekit.io/agents/build/nodes.md#tts-node) 來加快音訊輸出速度。

- **[Echo agent](https://github.com/livekit/agents/blob/main/examples/primitives/echo-agent/echo-agent.py)**: 將用戶音訊回傳給他們。

- **[Sync TTS transcription](https://github.com/livekit/agents/blob/main/examples/other/text-to-speech/sync_tts_transcription.py)**: 採用手動訂閱、轉錄轉發，並手動發布音訊輸出。

### Publishing video tracks

代理將資料以連續的即時回饋形式發佈到他們的軌道上。視訊串流可以採用 [11 種緩衝區編碼](https://github.com/livekit/python-sdks/blob/main/livekit-rtc/livekit/rtc/_proto/video_frame_pb2.pyi#L93) 中的任何一種來傳輸資料。發布影片軌道時，需要預先設定影片的幀率和緩衝編碼。

在此範例中，代理連接到房間並開始以每秒 10 幀 (FPS) 的速度發布 solid color frame。將以下程式碼複製到您的 `entrypoint` 函數中：

=== "Python"

    ```python
    from livekit import rtc
    from livekit.agents import JobContext

    WIDTH = 640
    HEIGHT = 480

    source = rtc.VideoSource(WIDTH, HEIGHT)
    track = rtc.LocalVideoTrack.create_video_track("example-track", source)
    options = rtc.TrackPublishOptions(
        # since the agent is a participant, our video I/O is its "camera"
        source=rtc.TrackSource.SOURCE_CAMERA,
        simulcast=True,
        # when modifying encoding options, max_framerate and max_bitrate must both be set
        video_encoding=rtc.VideoEncoding(
            max_framerate=30,
            max_bitrate=3_000_000,
        ),
        video_codec=rtc.VideoCodec.H264,
    )
    publication = await ctx.agent.publish_track(track, options)

    # this color is encoded as ARGB. when passed to VideoFrame it gets re-encoded.
    COLOR = [255, 255, 0, 0]; # FFFF0000 RED

    async def _draw_color():
        argb_frame = bytearray(WIDTH * HEIGHT * 4)
        while True:
            await asyncio.sleep(0.1) # 10 fps
            argb_frame[:] = COLOR * WIDTH * HEIGHT
            frame = rtc.VideoFrame(WIDTH, HEIGHT, rtc.VideoBufferType.RGBA, argb_frame)

            # send this frame to the track
            source.capture_frame(frame)

    asyncio.create_task(_draw_color())

    ```

!!! info
    - 儘管發布的幀是靜態的，但為了在發送初始幀後加入房間的參與者的利益，仍然需要連續地流式傳輸它。
    - 與音訊不同，視訊 `capture_frame` 不保留內部緩衝區。

LiveKit 可以自動在視訊緩衝區編碼之間進行轉換。 `VideoFrame` 提供目前視訊緩衝區類型以及將其轉換為任何其他編碼的方法：

=== "Python"

    ```python

    async def handle_video(track: rtc.Track):
        video_stream = rtc.VideoStream(track)
        async for event in video_stream:
            video_frame = event.frame
            current_type = video_frame.type
            frame_as_bgra = video_frame.convert(rtc.VideoBufferType.BGRA)
            # [...]
        await video_stream.aclose()

    @ctx.room.on("track_subscribed")
    def on_track_subscribed(
        track: rtc.Track,
        publication: rtc.TrackPublication,
        participant: rtc.RemoteParticipant,
    ):
        if track.kind == rtc.TrackKind.KIND_VIDEO:
            asyncio.create_task(handle_video(track))

    ```

### Audio and video synchronization

!!! info
    `AVSynchronizer` 目前僅在 Python 中可用。

雖然 WebRTC 本身可以處理 A/V 同步，但某些場景需要手動同步 - 例如，將產生的視訊與語音輸出同步時。

[`AVSynchronizer`](https://docs.livekit.io/reference/python/v1/livekit/rtc/index.html.md#livekit.rtc.AVSynchronizer) 實用程式透過對齊第一個音訊和視訊幀來幫助保持同步。後續影格根據配置的視訊 FPS 和音訊取樣率自動同步。

- **[Audio and video synchronization](https://github.com/livekit/python-sdks/tree/main/examples/video-stream)**: 示範如何使用 `AVSynchronizer` 實用程式同步視訊和音訊串流的範例。
