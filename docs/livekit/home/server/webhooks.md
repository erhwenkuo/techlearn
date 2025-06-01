# Webhooks

> 設定 LiveKit 以在房間事件發生時通知您的伺服器。

## Overview

可以設定 LiveKit 在房間事件發生時通知您的伺服器。這有助於您的後端了解房間何時結束或參與者何時離開。

使用 Cloud，可以在專案儀表板的「Settings section」部分配置 webhook。

當自架(self-hosting) LiveKit 實例時，可以透過設定配置中的 `webhook` 部分來啟用 webhook。

對於 Egress，也可以[configured inside Egress requests](https://docs.livekit.io/home/egress/api.md#WebhookConfig)額外的 webhook。


```yaml
webhook:
  # The API key to use in order to sign the message
  # This must match one of the keys LiveKit is configured with
  api_key: 'api-key-to-sign-with'
  urls:
    - 'https://yourhost'

```

## Receiving webhooks

Webhook 要求是傳送到您在設定或雲端儀表板中指定的 URL 的 HTTP POST 要求。 `WebhookEvent` 被編碼為 JSON 並在請求正文中發送。

請求的 `Content-Type` 標頭設定為 `application/webhook+json`。請確保您的網路伺服器已配置為接收具有此內容類型的有效負載。

為了確保 webhook 請求來自 LiveKit，這些請求有一個包含簽署 JWT 令牌的 `Authorization` 標頭。該令牌包含有效負載的 sha256 哈希。

LiveKit 的伺服器 SDK 提供了 webhook receiver 函式庫，這有助於驗證和解碼有效負載。

=== "Node.js"

    ```typescript
    import { WebhookReceiver } from 'livekit-server-sdk';

    const receiver = new WebhookReceiver('apikey', 'apisecret');

    // In order to use the validator, WebhookReceiver must have access to the raw
    // POSTed string (instead of a parsed JSON object). If you are using express
    // middleware, ensure that `express.raw` is used for the webhook endpoint
    // app.use(express.raw({type: 'application/webhook+json'}));

    app.post('/webhook-endpoint', async (req, res) => {
    // Event is a WebhookEvent object
        const event = await receiver.receive(req.body, req.get('Authorization'));
    });

    ```


=== "Go"

    ```go
    import (
    "github.com/livekit/protocol/auth"
    "github.com/livekit/protocol/livekit"
    "github.com/livekit/protocol/webhook"
    )

    func ServeHTTP(w http.ResponseWriter, r *http.Request) {
        authProvider := auth.NewSimpleKeyProvider(
            apiKey, apiSecret,
        )
        // Event is a livekit.WebhookEvent{} object
        event, err := webhook.ReceiveWebhookEvent(r, authProvider)
        if err != nil {
            // Could not validate, handle error
            return
        }
        // Consume WebhookEvent
    }

    ```

=== "Java"

    ```java
    import io.livekit.server.*;

    WebhookReceiver webhookReceiver = new WebhookReceiver("apiKey", "secret");

    // postBody is the raw POSTed string.
    // authHeader is the value of the "Authorization" header in the request.
    LivekitWebhook.WebhookEvent event = webhookReceiver.receive(postBody, authHeader);

    // Consume WebhookEvent

    ```

## Delivery and retries

Webhook 是由 LiveKit 發起並傳送到您的後端的 HTTP 請求。由於該協議基於推送的性質，因此無法保證交付。

LiveKit 旨在透過多次重試 webhook 請求來緩解瞬態故障。每個訊息在被放棄之前都會經過多次傳遞嘗試。如果有多個事件排隊等待傳遞，LiveKit 將對它們進行正確的排序；僅在舊事件已傳送或放棄後才傳送新事件。

## Events

除了以下欄位之外，所有 webhook 事件還將包含以下欄位：

- `id` - a UUID identifying the event
- `createdAt` - UNIX timestamp in seconds

### Room Started

```typescript
interface WebhookEvent {
  event: 'room_started';
  room: Room;
}

```

### Room Finished

```typescript
interface WebhookEvent {
  event: 'room_finished';
  room: Room;
}

```

### Participant Joined

```typescript
interface WebhookEvent {
  event: 'participant_joined';
  room: Room;
  participant: ParticipantInfo;
}

```

### Participant Left

```typescript
interface WebhookEvent {
  event: 'participant_left';
  room: Room;
  participant: ParticipantInfo;
}

```

### Track Published

在 Room 和 Participant 物件中，僅傳送 sid、identity 和 name。

```typescript
interface WebhookEvent {
  event: 'track_published';
  room: Room;
  participant: ParticipantInfo;
  track: TrackInfo;
}

```

### Track Unpublished

在 Room 和 Participant 物件中，僅傳送 sid、identity 和 name。

```typescript
interface WebhookEvent {
  event: 'track_unpublished';
  room: Room;
  participant: ParticipantInfo;
  track: TrackInfo;
}

```

### Egress Started

```typescript
interface WebhookEvent {
  event: 'egress_started';
  egressInfo: EgressInfo;
}

```

### Egress Updated

```typescript
interface WebhookEvent {
  event: 'egress_updated';
  egressInfo: EgressInfo;
}

```

### Egress Ended

```typescript
interface WebhookEvent {
  event: 'egress_ended';
  egressInfo: EgressInfo;
}

```

### Ingress Started

```typescript
interface WebhookEvent {
  event: 'ingress_started';
  ingressInfo: IngressInfo;
}

```

### Ingress Ended

```typescript
interface WebhookEvent {
  event: 'ingress_ended';
  ingressInfo: IngressInfo;
}

```
