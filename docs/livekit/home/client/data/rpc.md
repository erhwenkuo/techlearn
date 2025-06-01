# Remote method calls (RPC)

> 使用 RPC 對房間中的其他參與者執行自訂方法並等待回應。

## Overview

使用 RPC，您的應用程式可以在一個參與者上定義方法，該方法可以被房間內的其他參與者遠端調用，並可能返回回應。此功能可用於請求資料、協調特定於應用程式的狀態等。當用於從 AI 代理程式 [forward tool calls](https://docs.livekit.io/agents/build/tools.md#forwarding) 時，您的 LLM 可以直接存取資料或操作應用程式前端的 UI。


## Method registration

首先使用 `localParticipant.registerRpcMethod` 在目標參與者處註冊該方法，並提供方法的名稱和處理程序函數。單一參與者可以註冊任意數量的方法。


=== "Python"

    ```python
    @room.local_participant.register_rpc_method("greet")
    async def handle_greet(data: RpcInvocationData):
        print(f"Received greeting from {data.caller_identity}: {data.payload}")
        return f"Hello, {data.caller_identity}!"

    ```

=== "JavaScript"

    ```typescript
    localParticipant.registerRpcMethod(
    'greet',
    async (data: RpcInvocationData) => {
        console.log(`Received greeting from ${data.callerIdentity}: ${data.payload}`);
        return `Hello, ${data.callerIdentity}!`;
    }
    );

    ```

=== "Node.js"

    ```typescript
    localParticipant.registerRpcMethod(
    'greet',
    async (data: RpcInvocationData) => {
        console.log(`Received greeting from ${data.callerIdentity}: ${data.payload}`);
        return `Hello, ${data.callerIdentity}!`;
    }
    );

    ```

=== "Rust"

    ```rust
    room.local_participant().register_rpc_method(
        "greet".to_string(),
        |data| {
            Box::pin(async move {
                println!(
                    "Received greeting from {}: {}",
                    data.caller_identity,
                    data.payload
                );
                return Ok("Hello, ".to_string() + &data.caller_identity);
            })
        },
    );

    ```

=== "Android"

    ```kotlin
    localParticipant.registerRpcMethod(
        "greet"
    ) { data ->
        println("Received greeting from ${data.callerIdentity}: ${data.payload}")
        "Hello, ${data.callerIdentity}!"
    }

    ```

=== "Swift"

    ```swift
    localParticipant.registerRpcMethod("greet") { data in
        print("Received greeting from \(data.callerIdentity): \(data.payload)")
        return "Hello, \(data.callerIdentity)!"
    }

    ```

=== "Go"

    ```go
    greetHandler := func(data livekit.RpcInvocationData) (string, error) {
        fmt.Printf("Received greeting from %s: %s\n", data.CallerIdentity, data.Payload)
        return "Hello, " + data.CallerIdentity + "!", nil
    }
    room.LocalParticipant.RegisterRpcMethod("greet", greetHandler)

    ```

## Method invocation

使用 `localParticipant.performRpc` 透過提供目標參與者的身分、方法名稱和有效負載來呼叫遠端參與者上已註冊的 RPC 方法。這是一個傳回字串的非同步操作，並且可能會引發錯誤。


=== "Python"

    ```python
    try:
        response = await room.local_participant.perform_rpc(
            destination_identity='recipient-identity',
            method='greet',
            payload='Hello from RPC!'
        )
        print(f"RPC response: {response}")
    except Exception as e:
        print(f"RPC call failed: {e}")

    ```

=== "JavaScript"

    ```typescript
    try {
        const response = await localParticipant.performRpc({
            destinationIdentity: 'recipient-identity',
            method: 'greet',
            payload: 'Hello from RPC!',
        });
        console.log('RPC response:', response);
    } catch (error) {
        console.error('RPC call failed:', error);
    }

    ```

=== "Node.js"

    ```typescript
    try {
        const response = await localParticipant.performRpc({
            destinationIdentity: 'recipient-identity',
            method: 'greet',
            payload: 'Hello from RPC!',
        });
        console.log('RPC response:', response);
    } catch (error) {
        console.error('RPC call failed:', error);
    }

    ```

=== "Rust"

    ```rust
    match room
        .local_participant()
        .perform_rpc(PerformRpcParams {
            destination_identity: "recipient-identity".to_string(),
            method: "greet".to_string(),
            payload: "Hello from RPC!".to_string(),
            ..Default::default()
        })
        .await
    {
        Ok(response) => {
            println!("RPC response: {}", response);
        }
        Err(e) => log::error!("RPC call failed: {:?}", e),
    }

    ```

=== "Android"

    ```kotlin
    try {
        val response = localParticipant.performRpc(
            destinationIdentity = "recipient-identity",
            method = "greet",
            payload = "Hello from RPC!"
        ).await()
        println("RPC response: $response")
    } catch (e: RpcError) {
        println("RPC call failed: $e")
    }

    ```

=== "Swift"

    ```swift
    do {
        let response = try await localParticipant.performRpc(
        destinationIdentity: "recipient-identity",
        method: "greet",
        payload: "Hello from RPC!"
        )
        print("RPC response: \(response)")
    } catch let error as RpcError {
        print("RPC call failed: \(error)")
    }

    ```

=== "Go"

    ```go
    res, err := room.LocalParticipant.PerformRpc(livekit.PerformRpcParams{
    DestinationIdentity: "recipient-identity",
    Method: "greet",
    Payload: "Hello from RPC!",
    })
    if err != nil {
    fmt.Printf("RPC call failed: %v\n", err)
    }
    fmt.Printf("RPC response: %s\n", res)

    ```

## Method names

方法名稱可以是任何字串，最長為 64 個位元組（UTF-8）。

## Payload format

RPC 請求和回應都支援字串有效負載，最大大小為 15KiB（UTF-8）。您可以使用任何有意義的格式，例如 JSON 或 base64 編碼資料。

## Response timeout

如果回應時間過長，`performRpc` 會使用逾時自動掛斷。預設逾時時間為 10 秒，但您可以在 `performRpc` 呼叫中根據需要隨意更改它。一般來說，您應該設定盡可能短的超時時間，同時仍然滿足您的用例。

您設定的逾時將用於整個請求持續時間，包括網路延遲。這意味著處理程序提供的超時時間將比總體超時時間短。

## Errors

`performRpc` 將傳回某些內建錯誤（詳見下文），或您在遠端方法處理程序中產生的自訂錯誤。

若要向呼叫者傳回自訂錯誤，處理程序應拋出具有下列屬性的 `RpcError` 類型的錯誤：

- `code`: 指示錯誤類型的數字。代碼 1001-1999 保留用於 LiveKit 內部錯誤。
- `message`: 提供錯誤可讀描述的字串。
- `data`: 可選字串，提供有關錯誤的更多上下文，具有與請求/回應有效負載相同的格式和限制。

處理程序中拋出的任何其他錯誤都會被捕獲，並且呼叫者將收到通用的“1500 應用程式錯誤 `1500 Application Error`。

#### Built-in error types

| Code | Name | Description |
|------|------|-------------|
| 1400 | UNSUPPORTED_METHOD | 目標不支援此方法 |
| 1401 | RECIPIENT_NOT_FOUND | 未找到收件人 |
| 1402 | REQUEST_PAYLOAD_TOO_LARGE | 請求負載(payload)太大 |
| 1403 | UNSUPPORTED_SERVER | 伺服器不支援 RPC |
| 1404 | UNSUPPORTED_VERSION | 不受支援的 RPC 版本 |
| 1500 | APPLICATION_ERROR | 方法處理程序中的應用程式錯誤 |
| 1501 | CONNECTION_TIMEOUT | 連線逾時 |
| 1502 | RESPONSE_TIMEOUT | 回應超時 |
| 1503 | RECIPIENT_DISCONNECTED | 收件人已斷開連接 |
| 1504 | RESPONSE_PAYLOAD_TOO_LARGE | 響應負載過大 |
| 1505 | SEND_FAILED | 發送失敗 |
