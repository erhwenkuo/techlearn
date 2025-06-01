# Text and transcriptions

> 將即時文字功能整合到您的代理程式中。

## Overview

LiveKit Agents 除了支援音訊之外，還支援基於 LiveKit SDK 的 [text streams](https://docs.livekit.io/home/client/data/text-streams.md) 功能的文字輸入和輸出。本指南解釋了什麼是可能的以及如何在您的應用程式中使用它。

## Transcriptions

當代理將 STT 作為其處理管道的一部分執行時，轉錄也會即時發佈到前端。此外，當代理講話時，代理語音的文字表示也會與音訊播放同步發布。使用 `AgentSession` 時，這些功能都是預設啟用的。

轉錄使用 `lk.transcription` 文字流主題 (text stream topic)。它們包括一個 `lk.transcribed_track_id` 屬性，並且發送者身分是被轉錄的參與者。

若要停用轉錄輸出，請在 `transcription_enabled=False` 中設定 `RoomOutputOptions`。

### Synchronized transcription forwarding

當同時啟用語音和轉錄時，代理的語音與其轉錄同步，並在說話時逐字顯示文字。如果代理被打斷，轉錄就會停止並被截斷以匹配口頭輸出。

### Accessing from AgentSession

透過監聽 [conversation_item_added](https://docs.livekit.io/agents/build/events.md#conversation_item_added) 事件，每當文字輸入或輸出提交至聊天歷史記錄時，您都會在代理程式中收到通知。

## Text input

您的代理也會監控 `lk.chat` 文字流主題 (text stream topic)，以取得來自其連結參與者的傳入文字訊息。代理程式會中斷其當前語音（如果有）來處理訊息並產生新的回應。

若要停用文字輸入，請在 `RoomInputOptions` 中設定 `text_enabled=False`。

## Text-only output

若要完全停用音訊輸出並僅傳送文本，請在 `RoomOutputOptions` 中設定 `audio_enabled=False`。代理程式將向 `lk.transcription` 文字流主題 (text stream topic) 發佈文字回應，不含 `lk.transcribed_track_id` 屬性，也不帶語音同步。

## Usage examples

本節包含一些小程式碼範例，示範如何使用文字功能。

有關詳細信息，請參閱 [text streams](https://docs.livekit.io/home/client/data/text-streams.md) 文檔。有關更完整的範例，請參閱 [recipes](https://docs.livekit.io/recipes.md) 集合。

### Frontend integration

使用 `registerTextStreamHandler` 方法接收傳入的轉錄或文字：

=== "JavaScript"

    ```typescript
    room.registerTextStreamHandler('lk.transcription', async (reader, participantInfo) => {
        const message = await reader.readAll();
        if (reader.info.attributes['lk.transcribed_track_id']) {
            console.log(`New transcription from ${participantInfo.identity}: ${message}`);
        } else {
            console.log(`New message from ${participantInfo.identity}: ${message}`);
        }
    });

```

=== "Swift"

    ```swift
    try await room.localParticipant.registerTextStreamHandler(for: "lk.transcription") { reader, participantIdentity in
        let message = try await reader.readAll()
        if let transcribedTrackId = reader.info.attributes["lk.transcribed_track_id"] {
            print("New transcription from \(participantIdentity): \(message)")
        } else {
            print("New message from \(participantIdentity): \(message)")
        }
    }

    ```

    使用 `sendText` 方法發送訊息：

=== "JavaScript"

    ```typescript
    const text = 'Hello how are you today?';
    const info = await room.localParticipant.sendText(text, {
        topic: 'lk.chat',
    });

    ```

=== "Swift"

    ```swift
    let text = "Hello how are you today?"
    let info = try await room.localParticipant.sendText(text, for: "lk.chat")

    ```

### Configuring input/output options

AgentSession 建構函式接受輸入和輸出選項的配置：

```python
session = AgentSession(
    ..., # STT, LLM, etc.
    room_input_options=RoomInputOptions(
        text_enabled=False # disable text input
    ), 
    room_output_options=RoomOutputOptions(
        audio_enabled=False # disable audio output
    )
)

```

## Manual text input

若要插入文字輸入並產生回應，請使用 AgentSession 的 `generate_reply` 方法：`session.generate_reply(user_input="...")`。

## Transcription events

前端 SDK 還可以透過 `RoomEvent.TranscriptionReceived` 接收轉錄事件。

!!! tip
    🔥 **Deprecated feature**

    轉錄事件將在未來的版本中被刪除。請改用 `lk.chat` 主題上的 [text streams](https://docs.livekit.io/home/client/data/text-streams.md)。

=== "JavaScript"

    ```typescript
    room.on(RoomEvent.TranscriptionReceived, (segments) => {
        for (const segment of segments) {
            console.log(`New transcription from ${segment.senderIdentity}: ${segment.text}`);
        }
    });

    ```

=== "Swift"

    ```swift
    func room(_ room: Room, didReceiveTranscriptionSegments segments: [TranscriptionSegment]) {
        for segment in segments {
            print("New transcription from \(segment.senderIdentity): \(segment.text)")
        }
    }

    ```

=== "Android"

    ```kotlin
    room.events.collect { event ->
        if (event is RoomEvent.TranscriptionReceived) {
            event.transcriptionSegments.forEach { segment ->
            println("New transcription from ${segment.senderIdentity}: ${segment.text}")
            }
        }
    }

    ```

=== "Flutter"

    ```dart
    room.createListener().on<TranscriptionEvent>((event) {
        for (final segment in event.segments) {
            print("New transcription from ${segment.senderIdentity}: ${segment.text}");
        }
    });

    ```
