# Sending files & bytes

> 使用位元組流在參與者之間發送文件、圖像或任何其他類型的資料。

## Overview

位元組流(Byte streams)提供了一種在參與者之間即時發送文件、圖像或其他二進位資料的簡單方法。每個單獨的串流都與一個主題相關聯，並且您必須註冊一個處理程序來接收該主題的傳入流。流可以針對特定參與者或整個房間。

若要傳送文字數據，請改用 [text streams](./text-streams.md)

## Sending files

若要傳送檔案或映像，請使用 `sendFile` 方法。精確支援因 SDK 而異，因為它與平台自己的文件 API 整合。


=== "Python"

    ```python
    # Send a file from disk by specifying its path
    info = await room.local_participant.send_file(
    file_path="path/to/file.jpg",
    topic="my-topic",
    )
    print(f"Sent file with stream ID: {info.stream_id}")

    ```

=== "JavaScript"

    ```typescript
    // Send a `File` object
    const file = ($('file') as HTMLInputElement).files?.[0]!; 
    const info = await room.localParticipant.sendFile(file, {
        mimeType: file.type,
        topic: 'my-topic',

        // Optional, allows progress to be shown to the user
        onProgress: (progress) => console.log('sending file, progress', Math.ceil(progress * 100)), 
    });
    console.log(`Sent file with stream ID: ${info.id}`);

    ```

=== "Node.js"

    ```typescript
    // Send a file from disk by specifying its path
    const info = await room.localParticipant.sendFile("path/to/file.jpg", {
    topic: "my-topic",
    });
    console.log(`Sent file with stream ID: ${info.id}`);

    ```

=== "Swift"

    ```swift
    // Send a file from disk by specifying its path
    let fileURL = URL(filePath: "path/to/file.jpg")
    let info = try await room.localParticipant
        .sendFile(fileURL, for: "my-topic")

    print("Sent file with stream ID: \(info.id)")

    ```

=== "Rust"

    ```rust
    let options = StreamByteOptions {
        topic: "my-topic".to_string(),
        ..Default::default()
    };
    let info = room.local_participant()
        .send_file("path/to/file.jpg", options).await?;

    println!("Sent file with stream ID: {}", info.id);

    ```


=== "Go"

    ```go
    filePath := "path/to/file.jpg"
    info, err := room.LocalParticipant.SendFile(filePath, livekit.StreamBytesOptions{
        Topic: "my-topic",
        FileName: &filePath,
    })
    if err != nil {
        fmt.Printf("failed to send file: %v\n", err)
    }
    fmt.Printf("Sent file with stream ID: %s\n", info.ID)

    ```

## Streaming bytes

若要傳輸任何類型的二進位數據，請使用 `streamBytes` 方法開啟流寫入器。發送完資料後，必須明確關閉流。


=== "Python"

    ```python
    writer = await self.stream_bytes(
        # All byte streams must have a name, which is like a filename
        name="my-byte-stream",

        # The topic must match the topic used in the receiver's `register_byte_stream_handler`
        topic="my-topic",
    )

    print(f"Opened byte stream with ID: {writer.stream_id}")

    chunk_size = 15000 # 15KB, a recommended max chunk size

    # This an example to send a file, but you can send any kind of binary data
    async with aiofiles.open(file_path, "rb") as f:
        while bytes := await f.read(chunk_size):
            await writer.write(bytes)

    await writer.aclose()

    ```


=== "Node.js"

    ```typescript
    const writer = await room.localParticipant.streamBytes({
    // All byte streams must have a name, which is like a filename
    name: "my-byte-stream",

    // The topic must match the topic used in the receiver's `registerByteStreamHandler`
    topic: "my-topic",
    });

    console.log(`Opened byte stream with ID: ${writer.info.id}`);

    const chunkSize = 15000; // 15KB, a recommended max chunk size

    // This is an example to send a file, but you can send any kind of binary data
    const fileStream = fs.createReadStream(filePath, { highWaterMark: chunkSize });

    for await (const chunk of fileStream) {
    await writer.write(chunk);
    }

    await writer.close();

    ```

=== "Swift"

    ```swift
    let writer = try await room.localParticipant
        .streamBytes(for: "my-topic")

    print("Opened byte stream with ID: \(writer.info.id)")

    // Example sending arbitrary binary data
    // For sending files, use `sendFile` instead
    let dataChunks = [Data([0x00, 0x01]), Data([0x03, 0x04])]
    for chunk in dataChunks {
        try await writer.write(chunk)
    }

    // The stream must be explicitly closed when done
    try await writer.close()

    print("Closed byte stream with ID: \(writer.info.id)")

    ```


=== "Rust"

    ```rust
    let options = StreamByteOptions {
        topic: "my-topic".to_string(),
        ..Default::default()
    };
    let stream_writer = room.local_participant()
        .stream_bytes(options).await?;

    let id = stream_writer.info().id.clone();
    println!("Opened text stream with ID: {}", id);

    // Example sending arbitrary binary data
    // For sending files, use `send_file` instead
    let data_chunks = [[0x00, 0x01], [0x03, 0x04]];
    for chunk in data_chunks {
        stream_writer.write(&chunk).await?;
    }
    // The stream can be closed explicitly or will be closed implicitly
    // when the last writer is dropped
    stream_writer.close().await?;

    println!("Closed text stream with ID: {}", id);

    ```


=== "Go"

    ```go
    writer := room.LocalParticipant.StreamBytes(livekit.StreamBytesOptions{
    Topic: "my-topic",
    })

    // Use the writer to send data
    // onDone is called when a chunk is sent
    // writer can be closed in onDone of the last chunk
    writer.Write(data, onDone)

    // Close the writer when done, if you haven't already
    writer.Close()

    ```

## Handling incoming streams

無論資料是作為文件(file)還是流(stream)發送，它總是作為流接收。您必須註冊一個處理程序(handler)來接收它。


=== "Python"

    ```python
    import asyncio

    # Store active tasks to prevent garbage collection
    _active_tasks = []

    async def async_handle_byte_stream(reader, participant_identity):
        info = reader.info

        # Read the stream to a file
        with open(reader.info["name"], mode="wb") as f:
            async for chunk in reader:
                f.write(chunk)

            f.close()

        print(
            f'File "{info.name}" received from {participant_identity}\n'
            f'  Topic: {info.topic}\n'
            f'  Timestamp: {info.timestamp}\n'
            f'  ID: {info.id}\n'
            f'  Size: {info.size}'  # Optional, only available if the stream was sent with `send_file`
        )

    def handle_byte_stream(reader, participant_identity):
        task = asyncio.create_task(async_handle_byte_stream(reader, participant_identity))
        _active_tasks.append(task)
        task.add_done_callback(lambda t: _active_tasks.remove(t))

    room.register_byte_stream_handler(
        "my-topic",
        handle_byte_stream
    )

    ```

=== "JavaScript"

    ```typescript
    room.registerByteStreamHandler('my-topic', (reader, participantInfo) => {
    const info = reader.info;

    // Optional, allows you to display progress information if the stream was sent with `sendFile`
    reader.onProgress = (progress) => {
        console.log(`"progress ${progress ? (progress * 100).toFixed(0) : 'undefined'}%`);
    };

    // Option 1: Process the stream incrementally using a for-await loop.
    for await (const chunk of reader) {
        // Collect these however you want. 
        console.log(`Next chunk: ${chunk}`); 
    }

    // Option 2: Get the entire file after the stream completes.
    const result = new Blob(await reader.readAll(), { type: info.mimeType });

    console.log(
        `File "${info.name}" received from ${participantInfo.identity}\n` +
        `  Topic: ${info.topic}\n` +
        `  Timestamp: ${info.timestamp}\n` +
        `  ID: ${info.id}\n` +
        `  Size: ${info.size}` // Optional, only available if the stream was sent with `sendFile`
    );
    });

    ```

=== "Swift"

    ```swift
    try await room.localParticipant
        .registerByteStreamHandler(for: "my-topic") { reader, participantIdentity in
            let info = reader.info

            // Option 1: Process the stream incrementally using a for-await loop
            for try await chunk in reader {
                // Collect these however you want
                print("Next chunk received: \(chunk.count) bytes")
            }

            // Option 2: Get the entire file after the stream completes
            let data = try await reader.readAll()

            // Option 3: Write the stream to a local file on disk as it arrives
            let fileURL = try await reader.writeToFile()
            print("Wrote file to: \(fileURL)")

            print("""
                File "\(info.name ?? "unnamed")" received from \(participantIdentity)
                Topic: \(info.topic)
                Timestamp: \(info.timestamp)
                ID: \(info.id)
                Size: \(info.size) (only available if the stream was sent with `sendFile`)
                """)
        }

    ```

=== "Rust"

Rust API 與其他 SDK 略有不同。如果您希望處理流程，則無需註冊主題處理程序，而是處理 `ByteStreamOpened` 房間事件並從事件中取得讀取器。

    ```rust
    while let Some(event) = room.subscribe().recv().await {
        match event {
            RoomEvent::ByteStreamOpened { reader, topic, participant_identity } => {
                if topic != "my-topic" { continue };
                let Some(mut reader) = reader.take() else { continue };
                let info = reader.info();

                // Option 1: Process the stream incrementally as a Stream
                //           using `TryStreamExt` from the `futures_util` crate
                while let Some(chunk) = reader.try_next().await? {
                    println!("Next chunk: {:?}", chunk);
                }

                // Option 2: Get the entire file after the stream completes
                let data = reader.read_all().await?;

                // Option 3: Write the stream to a local file on disk as it arrives
                let file_path = reader.write_to_file().await?;
                println!("Wrote file to: {}", file_path.display());

                println!("File '{}' received from {}", info.name, participant_identity);
                println!("  Topic: {}", info.topic);
                println!("  Timestamp: {}", info.timestamp);
                println!("  ID: {}", info.id);
                println!("  Size: {:?}", info.total_length); // Only available when sent with `send_file`
            }
            _ => {}
        }
    }

    ```

=== "Node.js"

    ```typescript
    room.registerByteStreamHandler('my-topic', (reader, participantInfo) => {
    const info = reader.info;

    // Option 1: Process the stream incrementally using a for-await loop.
    for await (const chunk of reader) {
        // Collect these however you want. 
        console.log(`Next chunk: ${chunk}`); 
    }

    // Option 2: Get the entire file after the stream completes.
    const result = new Blob(await reader.readAll(), { type: info.mimeType });

    console.log(
        `File "${info.name}" received from ${participantInfo.identity}\n` +
        `  Topic: ${info.topic}\n` +
        `  Timestamp: ${info.timestamp}\n` +
        `  ID: ${info.id}\n` +
        `  Size: ${info.size}` // Optional, only available if the stream was sent with `sendFile`
    );
    });

    ```

=== "Go"

    ```go
    room.RegisterByteStreamHandler(
    "my-topic",
    func(reader livekit.ByteStreamReader, participantIdentity livekit.ParticipantIdentity) {
        fmt.Printf("Byte stream received from %s\n", participantIdentity)

        // Option 1: Process the stream incrementally
        res := []byte{}
        for {
        chunk := make([]byte, 1024)
        n, err := reader.Read(chunk)
        res = append(res, chunk[:n]...)
        if err != nil {
            if err == io.EOF {
            break
            } else {
            fmt.Printf("failed to read byte stream: %v\n", err)
            break
            }
        }
        }
        // Similar to Read, there is ReadByte(), ReadBytes(delim byte)

        // Option 2: Get the entire stream after it completes
        data := reader.ReadAll()
        fmt.Printf("received data: %v\n", data)
    },
    )

    ```

## Stream properties

這些是文字流 (text stream) 上可用的所有屬性，可以從發送/流 (send/stream) 方法中設定或從處理程序中讀取。

| Property | Description | Type |
|----------|-------------|------|
| `id` | 此流的唯一識別碼。 | string |
| `topic` | 用於將流路由到適當處理程序的主題名稱。 | string |
| `timestamp` | 流的創建時間。 | number |
| `mimeType` | 串流資料的 MIME 類型。自動偵測文件，否則預設為 `application/octet-stream`。 | string |
| `name` | 正在傳送的文件的名稱。| string |
| `size` | 如果知道的話，預計的總大小（以位元組為單位）。 | number |
| `attributes` | 您的應用程式所需的附加屬性。 | string dict |
| `destinationIdentities` | 發送流的參與者的身分。如果為空，將發送給所有人。 | array |

## Concurrency

可以同時寫入或讀取多個流。如果您在同一主題上多次呼叫 `sendFile` 或 `streamBytes`，則接收方的處理程序將被呼叫多次，每個流調用一次。這些呼叫將按照發送方打開流的順序發生，並且流讀取器將按照發送方關閉流的順序關閉。

## Joining mid-stream

在直播開始後加入房間的參與者將不會收到任何訊息。只有在串流開啟時連接的參與者才有資格接收它。

## Chunk sizes

寫入流和讀取流的過程分別進行了最佳化。這意味著發送的區塊的數量和大小可能與接收的區塊的數量和大小不符。但保證收到的完整數據是完整且有序的。塊通常小於 15kB。

!!! info
    串流是一種簡單且強大的資料傳送方式，但如果您需要精確控制單一資料包的行為，則較低層級的 [data packets](https://docs.livekit.io/home/client/data/packets.md) API 可能更合適。
