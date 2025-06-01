# End-to-end encryption

> 使用 E2EE 保護您的即時媒體軌道。

## Overview

LiveKit 內建了對即時音訊和視訊軌道的端對端加密 (E2EE) 的支援。啟用 E2EE 後，媒體資料從發送方到接收方都保持完全加密，確保沒有中介（包括 LiveKit 伺服器）可以存取或修改內容。這個功能是：

- 自架客戶和 LiveKit Cloud 客戶均可免費使用。
- 非常適合受監管的產業和安全關鍵型應用。
- 旨在提供超出標準傳輸加密的額外保護層。


## How E2EE works

E2EE 在房間層級啟用，並自動套用於該房間內所有參與者的所有媒體軌道。您必須在 LiveKit SDK 中為每位參與者啟用它。在許多情況下，您可以使用內建金鑰提供程序，為整個房間使用單一共用金鑰。如果您需要為每位參與者提供唯一的金鑰，或在單一房間的生命週期內進行金鑰輪換，您可以實現自己的金鑰提供者。

## Key distribution

您有責任在運行時安全地產生、儲存和分發加密金鑰給您的應用程式。 LiveKit 不會（也不能）為您儲存或傳輸加密金鑰。

如果使用共用金鑰，您通常會在建立房間的同時在伺服器上產生它，並將其與房間的存取權杖一起安全地分發給參與者。當每個參與者使用唯一的金鑰時，您可能需要一種更複雜的方法來在新參與者加入房間時分發金鑰。請記住，加密和解密都需要金鑰，因此即使使用每個參與者的金鑰，也必須確保所有參與者都擁有所有金鑰。

## Limitations

所有 LiveKit 網路流量都使用 TLS 加密，但完整的端對端加密僅適用於媒體軌道，而不適用於即時資料、文字、API 呼叫或其他訊號。

## Implementation guide

這些範例展示如何將內建金鑰提供者與共用金鑰一起使用。如果您需要使用自訂金鑰提供程序，請參閱下面的部分。

=== "JavaScript"

    ```typescript
    // 1. Initialize the external key provider
    const keyProvider = new ExternalE2EEKeyProvider();

    // 2. Configure room options
    const roomOptions: RoomOptions = {
    e2ee: {
        keyProvider: keyProvider,
        // Required for web implementations
        worker: new Worker(new URL('livekit-client/e2ee-worker', import.meta.url)),
    },
    };

    // 3. Create and configure the room
    const room = new Room(roomOptions);

    // 4. Set your externally distributed encryption key
    await keyProvider.setKey(yourSecureKey);

    // 5. Enable E2EE for all local tracks
    await room.setE2EEEnabled(true);

    // 6. Connect to the room
    await room.connect(url, token);

    ```

    #### Example implementation

    對於可用於生產的實現，請參閱我們的 [Meet 範例應用程式](https://github.com/livekit-examples/meet)，該應用程式使用 `ExternalE2EEKeyProvider` 在生產級應用程式中示範了 E2EE。


    === "iOS"

    ```swift
    // 1. Initialize the key provider with options
    let keyProvider = BaseKeyProvider(isSharedKey: true, sharedKey: "yourSecureKey")

    // 2. Configure room options with E2EE
    let roomOptions = RoomOptions(e2eeOptions: E2EEOptions(keyProvider: keyProvider))

    // 3. Create the room
    let room = Room(roomOptions: roomOptions)

    // 4. Connect to the room
    try await room.connect(url: url, token: token)

    ```

=== "Android"

    ```kotlin
    // 1. Initialize the key provider
    val keyProvider = BaseKeyProvider()

    // 2. Configure room options
    val roomOptions = RoomOptions(
        e2eeOptions = E2EEOptions(
            keyProvider = keyProvider
        )
    )
    // 3. Create and configure the room
    val room = LiveKit.create(context, options = roomOptions)

    // 4. Set your externally distributed encryption key
    keyProvider.setSharedKey(yourSecureKey)

    // 5. Connect to the room
    room.connect(url, token)

    ```

    #### Example implementation

    我們的主要 [Android 範例應用程式](https://github.com/livekit/client-sdk-android/tree/main/sample-app) 包含使用內建金鑰提供者的 E2EE 範例實作。


=== "Flutter"

    ```dart
    // 1. Initialize the key provider
    final keyProvider = await BaseKeyProvider.create();

    // 2. Configure room options
    final roomOptions = RoomOptions(
    e2eeOptions: E2EEOptions(
        keyProvider: keyProvider,
    ),
    );

    // 3. Create and configure the room
    final room = Room(options: roomOptions);

    // 4. Set your externally distributed encryption key
    await keyProvider.setSharedKey(yourSecureKey);

    // 5. Connect to the room
    await room.connect(url, token);

    ```

    #### Example implementation

    Flutter SDK 包含一個完整的多平台範例實現，使用共用金鑰支援 E2EE。

=== "React Native"

    ```jsx
    // 1. Use the hook to create an RNE2EEManager 
    //    with your externally distributed shared key
    // (Note: if you need a custom key provider, then you'll need 
    //        to create the key provider and `RNE2EEManager` directly)
    const { e2eeManager } = useRNE2EEManager({
    sharedKey: yourSecureKey,
    });

    // 2. Provide the e2eeManager in your room options
    const roomOptions = {
    e2ee: {
        e2eeManager,
    },
    };

    // 3. Pass the room options when creating your room
    <LiveKitRoom
    serverUrl={url}
    token={token}
    connect={true}
    options={roomOptions}
    audio={true}
    video={true}
    >
    </LiveKitRoom>

    ```

    #### Example implementation

    React Native SDK 包含一個完整的範例應用，該應用程式示範如何使用 `useRNE2EEManager` 鉤子和共用金鑰。


=== "Python"

    ```python
    # 1. Initialize key provider options with a shared key
    e2ee_options = rtc.E2EEOptions()
    e2ee_options.key_provider_options.shared_key = YOUR_SHARED_KEY

    # 2. Configure room options with E2EE
    room_options = RoomOptions(
        auto_subscribe=True,
        e2ee=e2ee_options
    )

    # 3. Create and connect to the room
    room = Room()
    await room.connect(url, token, options=room_options)

    ```

    #### Example implementation

    Python SDK 包含一個使用共用金鑰的 E2EE 的[簡單範例應用程](https://github.com/livekit/python-sdks/blob/main/examples/e2ee.py)。


=== "Node.js"

    ```javascript
    // 1. Initialize the key provider with options
    const keyProviderOptions = {
    sharedKey: yourSecureKey, // Your externally distributed encryption key
    };

    // 2. Configure E2EE options
    const e2eeOptions = {
    keyProviderOptions,
    };

    // 3. Create and configure the room
    const room = new Room();

    // 4. Connect to the room with E2EE enabled
    await room.connect(url, token, {
        e2ee: e2eeOptions,
    }
    );

    ```

## Using a custom key provider

如果您的應用程式需要在單一房間的生命週期內進行金鑰輪換，或每位參與者使用唯一的金鑰（例如在實作 [MEGOLM](https://gitlab.matrix.org/matrix-org/olm/blob/master/docs/megolm.html) 或 [MLS](https://messaginglayersecurity.rocks/ftm.html) 或 [MLS](https://messaginglayersecurity.rocks/ft.html.協定時），則需要實作自己的金鑰提供者。  其中的完整細節超出了本指南的範圍，但下面提供了 JS SDK 的簡要概述（其他 SDK 中的流程也類似）：

1. Extend the `BaseKeyProvider` class.
2. Call `onSetEncryptionKey` with each key/identity pair
3. Set appropriate ratcheting options (`ratchetSalt`, `ratchetWindowSize`, `failureTolerance`, `keyringSize`).
4. Implement the `onKeyRatcheted` method to handle key updates.
5. Call `ratchetKey()` when key rotation is needed.
6. Pass your custom key provider in the room options, in place of the built-in key provider.
