# Sending text

> 使用文字流在參與者之間發送任意數量的文字。

## Overview

文字流提供了一種在參與者之間即時發送文字的簡單方法，支援聊天、串流 LLM 回應等用例。每個單獨的串流都與一個主題相關聯，並且您必須註冊一個處理程序 (handler) 來接收該主題的傳入流。流可以針對特定參與者或整個房間。

若要傳送其他類型的數據，請改用 [byte streams](https://docs.livekit.io/home/client/data/byte-streams.md)。

## Sending all at once

當整個字串預先可用時，使用 `sendText` 方法。輸入字串會自動分塊並串流傳輸，因此字串大小沒有限制。


=== "Python"

    ```python
    text = 'Lorem ipsum dolor sit amet...'
    info = await room.local_participant.send_text(text, 
        topic='my-topic'
    )
    print(f"Sent text with stream ID: {info.stream_id}")

    ```

=== "JavaScript"

    ```typescript
    const text = 'Lorem ipsum dolor sit amet...';
    const info = await room.localParticipant.sendText(text, {
        topic: 'my-topic',
    });

    console.log(`Sent text with stream ID: ${info.id}`);

    ```


=== "Node.js"

    ```typescript
    const text = 'Lorem ipsum dolor sit amet...';
    const info = await room.localParticipant.sendText(text, {
        topic: 'my-topic',
    });

    console.log(`Sent text with stream ID: ${info.id}`);

    ```

=== "Swift"

    ```swift
    let text = "Lorem ipsum dolor sit amet..."
    let info = try await room.localParticipant
        .sendText(text, for: "my-topic")

    print("Sent text with stream ID: \(info.id)")

    ```


=== "Rust"

    ```rust
    let text = "Lorem ipsum dolor sit amet...";
    let options = StreamTextOptions {
        topic: "my-topic".to_string(),
        ..Default::default()
    };
    let info = room.local_participant()
        .send_text(&text, options).await?;

    println!("Sent text with stream ID: {}", info.id);

    ```


=== "Go"

    ```go
    text := "Lorem ipsum dolor sit amet..."
    info := room.LocalParticipant.SendText(text, livekit.StreamTextOptions{
        Topic: "my-topic",
    })

    fmt.Printf("Sent text with stream ID: %s\n", info.ID)

    ```

## Streaming incrementally

如果您的文字是增量生成的，請使用 `streamText` 開啟流寫入器。發送完資料後，必須明確關閉流。



=== "Python"

    ```python
    writer = await room.local_participant.stream_text(
        topic="my-topic",
    )

    print(f"Opened text stream with ID: {writer.stream_id}")

    # In a real application, you might receive chunks of text from an LLM or other source
    text_chunks = ["Lorem ", "ipsum ", "dolor ", "sit ", "amet..."]
    for chunk in text_chunks:
        await writer.write(chunk)

    await writer.close()

    print(f"Closed text stream with ID: {writer.stream_id}")

    ```

=== "JavaScript"

    ```typescript
    const streamWriter = await room.localParticipant.streamText({
        topic: 'my-topic',
        });   

    console.log(`Opened text stream with ID: ${streamWriter.info.id}`);

    // In a real app, you would generate this text asynchronously / incrementally as well
    const textChunks = ["Lorem ", "ipsum ", "dolor ", "sit ", "amet..."]
    for (const chunk of textChunks) {
        await streamWriter.write(chunk)
    }

    // The stream must be explicitly closed when done
    await streamWriter.close(); 

    console.log(`Closed text stream with ID: ${streamWriter.info.id}`);

    ```


=== "Node.js"

    ```typescript
    const streamWriter = await room.localParticipant.streamText({
        topic: 'my-topic',
    });   

    console.log(`Opened text stream with ID: ${streamWriter.info.id}`);

    // In a real app, you would generate this text asynchronously / incrementally as well
    const textChunks = ["Lorem ", "ipsum ", "dolor ", "sit ", "amet..."]
    for (const chunk of textChunks) {
        await streamWriter.write(chunk)
    }

    // The stream must be explicitly closed when done
    await streamWriter.close(); 

    console.log(`Closed text stream with ID: ${streamWriter.info.id}`);

    ```

=== "Swift"

    ```swift
    let writer = try await room.localParticipant
        .streamText(for: "my-topic")

    print("Opened text stream with ID: \(writer.info.id)")

    // In a real application, you might receive chunks of text from an LLM or other source
    let textChunks = ["Lorem ", "ipsum ", "dolor ", "sit ", "amet..."]
    for chunk in textChunks {
        try await writer.write(chunk)
    }

    // The stream must be explicitly closed when done
    try await writer.close()

    print("Closed text stream with ID: \(writer.info.id)")

    ```


=== "Rust"

    ```rust
    let options = StreamTextOptions {
        topic: "my-topic".to_string(),
        ..Default::default()
    };
    let stream_writer = room.local_participant()
        .stream_text(options).await?;

    let id = stream_writer.info().id.clone();
    println!("Opened text stream with ID: {}", id);

    let text_chunks = ["Lorem ", "ipsum ", "dolor ", "sit ", "amet..."];
    for chunk in text_chunks {
        stream_writer.write(&chunk).await?;
    }
    // The stream can be closed explicitly or will be closed implicitly
    // when the last writer is dropped
    stream_writer.close().await?;

    println!("Closed text stream with ID: {}", id);

    ```


=== "Go"

    ```go
    // In a real application, you would generate this text asynchronously / incrementally as well
    textChunks := []string{"Lorem ", "ipsum ", "dolor ", "sit ", "amet..."}

    writer := room.LocalParticipant.SendText(livekit.StreamTextOptions{
        Topic: "my-topic",
    })

    for i, chunk := range textChunks {
    // Close the stream when the last chunk is sent
    onDone := func() {
        if i == len(textChunks) - 1 {
            writer.Close()
        }
    } 

    writer.Write(chunk, onDone)
    }

    fmt.Printf("Closed text stream with ID: %s\n", writer.Info.ID)

    ```

## Handling incoming streams

無論資料是透過 `sendText` 還是 `streamText` 發送的，它總是以流的形式接收。您必須註冊一個處理程序來接收它。


=== "Python"

    ```python
    import asyncio

    # Store active tasks to prevent garbage collection
    _active_tasks = set()

    async def async_handle_text_stream(reader, participant_identity):
        info = reader.info

        print(
            f'Text stream received from {participant_identity}\n'
            f'  Topic: {info.topic}\n'
            f'  Timestamp: {info.timestamp}\n'
            f'  ID: {info.id}\n'
            f'  Size: {info.size}'  # Optional, only available if the stream was sent with `send_text`
        )

        # Option 1: Process the stream incrementally using an async for loop.
        async for chunk in reader:
            print(f"Next chunk: {chunk}")

        # Option 2: Get the entire text after the stream completes.
        text = await reader.read_all()
        print(f"Received text: {text}")
    
    def handle_text_stream(reader, participant_identity):
        task = asyncio.create_task(async_handle_text_stream(reader, participant_identity))
        _active_tasks.add(task)
        task.add_done_callback(lambda t: _active_tasks.remove(t))

    room.register_text_stream_handler(
        "my-topic",
        handle_text_stream
    )

    ```

=== "JavaScript"

    ```typescript
    room.registerTextStreamHandler('my-topic', (reader, participantInfo) => {
    const info = reader.info;
    console.log(
        `Received text stream from ${participantInfo.identity}\n` +
        `  Topic: ${info.topic}\n` +
        `  Timestamp: ${info.timestamp}\n` +
        `  ID: ${info.id}\n` +
        `  Size: ${info.size}` // Optional, only available if the stream was sent with `sendText`
    );  

    // Option 1: Process the stream incrementally using a for-await loop.
    for await (const chunk of reader) {
        console.log(`Next chunk: ${chunk}`);
    }

    // Option 2: Get the entire text after the stream completes.
    const text = await reader.readAll();
    console.log(`Received text: ${text}`);
    });

    ```


=== "Node.js"

    ```typescript
    room.registerTextStreamHandler('my-topic', (reader, participantInfo) => {
    const info = reader.info;
    console.log(
        `Received text stream from ${participantInfo.identity}\n` +
        `  Topic: ${info.topic}\n` +
        `  Timestamp: ${info.timestamp}\n` +
        `  ID: ${info.id}\n` +
        `  Size: ${info.size}` // Optional, only available if the stream was sent with `sendText`
    );  

    // Option 1: Process the stream incrementally using a for-await loop.
    for await (const chunk of reader) {
        console.log(`Next chunk: ${chunk}`);
    }

    // Option 2: Get the entire text after the stream completes.
    const text = await reader.readAll();
    console.log(`Received text: ${text}`);
    });

    ```

=== "Swift"

    ```swift
    try await room.localParticipant
        .registerTextStreamHandler(for: "my-topic") { reader, participantIdentity in
            let info = reader.info

            print("""
                Text stream received from \(participantIdentity)
                Topic: \(info.topic)
                Timestamp: \(info.timestamp)
                ID: \(info.id)
                Size: \(info.size) (only available if the stream was sent with `sendText`)
                """)

            // Option 1: Process the stream incrementally using a for-await loop
            for try await chunk in reader {
                print("Next chunk: \(chunk)")
            }

            // Option 2: Get the entire text after the stream completes
            let text = try await reader.readAll()
            print("Received text: \(text)")
        }

    ```

=== "Rust"

    Rust API 與其他 SDK 略有不同。如果您希望處理流程，則無需註冊主題處理程序，而是處理 `TextStreamOpened` 房間事件並從事件中取得讀取器。

    ```rust
    while let Some(event) = room.subscribe().recv().await {
        match event {
            RoomEvent::TextStreamOpened { reader, topic, participant_identity } => {
                if topic != "my-topic" { continue };
                let Some(mut reader) = reader.take() else { continue };
                let info = reader.info();

                println!("Text stream received from {participant_identity}");
                println!("  Topic: {}", info.topic);
                println!("  Timestamp: {}", info.timestamp);
                println!("  ID: {}", info.id);
                println!("  Size: {:?}", info.total_length);

                // Option 1: Process the stream incrementally as a Stream
                //           using `TryStreamExt` from the `futures_util` crate
                while let Some(chunk) = reader.try_next().await? {
                    println!("Next chunk: {chunk}");
                }

                // Option 2: Get the entire text after the stream completes
                let text = reader.read_all().await?;
                println!("Received text: {text}");
            }
            _ => {}
        }
    }

    ```


=== "Go"

    ```go
    room.RegisterTextStreamHandler(
    "my-topic",
    func(reader livekit.TextStreamReader, participantIdentity livekit.ParticipantIdentity) {
        fmt.Printf("Text stream received from %s\n", participantIdentity)

        // Option 1: Process the stream incrementally
        res := ""
            for {
        // ReadString takes a delimiter
                word, err := reader.ReadString(' ')
                fmt.Printf("read word: %s\n", word)
                res += word
                if err != nil {
                    // EOF represents the end of the stream
                    if err == io.EOF {
                        break
                    } else {
                        fmt.Printf("failed to read text stream: %v\n", err)
                        break
                    }
                }
            }
        // Similar to ReadString, there is Read(p []bytes), ReadByte(), ReadBytes(delim byte) and ReadRune() as well
        // All of these methods return io.EOF when the stream is closed
        // If the stream has no data, it will block until there is data or the stream is closed
        // If the stream has data, but not as much as requested, it will return what is available without any error

        // Option 2: Get the entire text after the stream completes
        text := reader.ReadAll()
        fmt.Printf("received text: %s\n", text)
    },
    )

    ```

## Stream properties

這些是文字流上可用的所有屬性，可以從 `send/stream` 方法中設定或從處理程序(handler)中讀取。

| Property | Description | Type |
|----------|-------------|------|
| `id` | 此流的唯一識別碼。 | string |
| `topic` | 用於將流路由到適當處理程序的主題名稱。 | string |
| `timestamp` | 流的創建時間。 | number |
| `size` | 預計總大小（以位元組為單位，UTF-8 編碼）（如果知道）。 | number |
| `attributes` | 您的應用程式所需的附加屬性。 | string dict |
| `destinationIdentities` | 發送流的參與者的身分。如果為空，則發送給所有人。 | array |

## Concurrency

可以同時寫入或讀取多個流。如果對相同主題多次呼叫 `sendText` 或 `streamText`，則接收方的處理程序將被呼叫多次，每個流調用一次。這些呼叫將按照發送方打開流的順序發生，並且流讀取器將按照發送方關閉流的順序關閉。

## Joining mid-stream

在直播開始後加入房間的參與者將不會收到任何訊息。只有在串流開啟時連接的參與者才有資格接收它。

## No message persistence

LiveKit 不包含文字流的長期持久性。所有數據僅在連接的參與者之間即時傳輸。如果您需要訊息歷史記錄，則需要使用資料庫或其他持久層自行實現儲存。

## Chat components

LiveKit 為聊天等常見的文字流用例提供了預先建立的 React 元件。有關詳細信息，請參閱 [Chat component](https://docs.livekit.io/reference/components/react/component/chat.md) 和 [useChat hook](https://docs.livekit.io/reference/components/react/hook/usechat.md)。

!!! info
    流是一種簡單而強大的發送文字的方式，但如果您需要精確控制單個資料包的行為，則較低級別的 [data packets](https://docs.livekit.io/home/client/data/packets.md) API 可能更合適。
