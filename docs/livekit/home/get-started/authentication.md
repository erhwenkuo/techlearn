# Authentication

> 了解如何在 LiveKit 會話中驗證使用者身分。

## Overview

為了使 LiveKit SDK 成功連接到伺服器，它必須透過請求傳遞存取令牌。

此令牌對參與者的身分、房間名稱、能力和權限進行編碼。存取令牌基於 JWT，並使用您的 API 金鑰進行簽名，以防止偽造。

存取令牌還帶有過期時間，過期後伺服器將拒絕使用該令牌的連線。注意：到期時間僅影響初始連接，而不會影響後續重新連接。

## Creating a token

=== "LiveKit CLI"

    ```bash
    lk token create \
    --api-key <KEY> \
    --api-secret <SECRET> \
    --identity <NAME> \
    --room <ROOM_NAME> \
    --join \
    --valid-for 1h

    ```

=== "Node.js"

    ```typescript
    import { AccessToken, VideoGrant } from 'livekit-server-sdk';

    const roomName = 'name-of-room';
    const participantName = 'user-name';

    const at = new AccessToken('api-key', 'secret-key', {
    identity: participantName,
    });

    const videoGrant: VideoGrant = {
    room: roomName,
    roomJoin: true,
    canPublish: true,
    canSubscribe: true,
    };

    at.addGrant(videoGrant);

    const token = await at.toJwt();
    console.log('access token', token);

    ```

=== "Go"

    ```go
    import (
    "time"

    "github.com/livekit/protocol/auth"
    )

    func getJoinToken(apiKey, apiSecret, room, identity string) (string, error) {
    canPublish := true
    canSubscribe := true

    at := auth.NewAccessToken(apiKey, apiSecret)
    grant := &auth.VideoGrant{
        RoomJoin:     true,
        Room:         room,
        CanPublish:   &canPublish,
        CanSubscribe: &canSubscribe,
    }
    at.SetVideoGrant(grant).
        SetIdentity(identity).
        SetValidFor(time.Hour)

    return at.ToJWT()
    }

    ```

=== "Ruby"

    ```ruby
    require 'livekit'

    token = LiveKit::AccessToken.new(api_key: 'yourkey', api_secret: 'yoursecret')
    token.identity = 'participant-identity'
    token.name = 'participant-name'
    token.video_grant=(LiveKit::VideoGrant.from_hash(roomJoin: true,
                                                    room: 'room-name'))

    puts token.to_jwt

    ```

=== "Java"

    ```java
    import io.livekit.server.*;

    public String createToken() {
    AccessToken token = new AccessToken("apiKey", "secret");
    token.setName("participant-name");
    token.setIdentity("participant-identity");
    token.setMetadata("metadata");
    token.addGrants(new RoomJoin(true), new Room("room-name"));

    return token.toJwt();
    }

    ```

=== "Python"

    ```python
    from livekit import api
    import os

    token = api.AccessToken(os.getenv('LIVEKIT_API_KEY'), os.getenv('LIVEKIT_API_SECRET')) \
        .with_identity("identity") \
        .with_name("name") \
        .with_grants(api.VideoGrants(
            room_join=True,
            room="my-room",
        )).to_jwt()

    ```


=== "Rust"

    ```rust
    use livekit_api::access_token;
    use std::env;

    fn create_token() -> Result<String, AccessTokenError> {
    let api_key = env::var("LIVEKIT_API_KEY").expect("LIVEKIT_API_KEY is not set");
    let api_secret = env::var("LIVEKIT_API_SECRET").expect("LIVEKIT_API_SECRET is not set");

    let token = access_token::AccessToken::with_api_key(&api_key, &api_secret)
        .with_identity("identity")
        .with_name("name")
        .with_grants(access_token::VideoGrants {
            room_join: true,
            room: "my-room".to_string(),
            ..Default::default()
        })
        .to_jwt();
    return token
    }

    ```

=== "Other"

  對於其他平台，您可以自行實現令牌生成，也可以使用 `lk` 命令。

  令牌簽名相當簡單，請參閱[JS 實作](https://github.com/livekit/node-sdks/blob/main/packages/livekit-server-sdk/src/AccessToken.ts)作為參考。

  LiveKit CLI 可從 [https://github.com/livekit/livekit-cli](https://github.com/livekit/livekit-cli) 取得

### Token example

以下是連接令牌解碼主體的範例：

```json
{
  "exp": 1621657263,
  "iss": "APIMmxiL8rquKztZEoZJV9Fb",
  "sub": "myidentity",
  "nbf": 1619065263,
  "video": {
    "room": "myroom",
    "roomJoin": true
  },
  "metadata": ""
}

```

| field | description |
|-------|-------------|
| exp | Token 的過期時間 |
| nbf | Token 生效的開始時間 |
| iss | 用於頒發此令牌的 API 金鑰 |
| sub | 參與者的唯一身份 |
| metadata | 參與者元數據 |
| attributes | 參與者屬性（字串的鍵/值對）|
| video | Video grant，包括房間權限（見下文） |
| sip | SIP 授權 |

## Video grant

房間權限在解碼的連線令牌的 `video` 欄位中指定。它可能包含以下一個或多個屬性：


| field | type | description |
|-------|------|-------------|
| roomCreate | bool | 建立或刪除房間的權限 |
| roomList | bool | 允許列出可用房間 |
| roomJoin | bool | 加入房間的權限 |
| roomAdmin | bool | 管理房間的權限 |
| roomRecord | bool | 使用 Egress 服務的權限 |
| ingressAdmin | bool | 使用 Ingress 服務的權限 |
| room | string | 房間名稱，如果設定了加入或使用 Egress 服務的權限 admin，則為必填項 |
| canPublish | bool | 允許參與者發布軌道 |
| canPublishData | bool | 允許參與者向房間發布數據 |
| canPublishSources | string[] | 需要 `canPublish` 被設成 true. 設定後，只有列出的來源可以發布。(camera, microphone, screen_share, screen_share_audio) |
| canSubscribe | bool | 允許參與者訂閱軌道 |
| canUpdateOwnMetadata | bool | 允許參與者更新自己的元數據 |
| hidden | bool | 向房間中的其他人隱藏參與者 |
| kind | string | 參與者的類型(standard, ingress, egress, sip, or agent)。此欄位通常由 LiveKit 內部設定。 |

### Example: subscribe-only token

要建立參與者只能訂閱而不能發佈到房間的令牌，您可以使用以下授權：

```json
{
  ...
  "video": {
    "room": "myroom",
    "roomJoin": true,
    "canSubscribe": true,
    "canPublish": false,
    "canPublishData": false
  }
}

```

### Example: camera-only

允許參與者發布攝像頭，但禁止其他來源：

```json
{
  ...
  "video": {
    "room": "myroom",
    "roomJoin": true,
    "canSubscribe": true,
    "canPublish": true,
    "canPublishSources": ["camera"]
  }
}

```

## SIP grant

為了與 SIP 服務交互，必須在 JWT 的 `sip` 欄位中授予權限。它可能包含以下屬性：

| field | type | description |
|-------|------|-------------|
| admin | bool | 管理 SIP 中繼和調度規則的權限。|
| call | bool | 透過 `CreateSIPParticipant` 進行 SIP 通話的權限。|

### Creating a token with SIP grants

=== "Node.js"

  ```typescript
  import { AccessToken, SIPGrant, VideoGrant } from 'livekit-server-sdk';

  const roomName = 'name-of-room';
  const participantName = 'user-name';

  const at = new AccessToken('api-key', 'secret-key', {
    identity: participantName,
  });

  const sipGrant: SIPGrant = { 
    admin: true,
    call: true,
  };  

  const videoGrant: VideoGrant = { 
    room: roomName,
    roomJoin: true,
  };  

  at.addGrant(sipGrant);
  at.addGrant(videoGrant);

  const token = await at.toJwt();
  console.log('access token', token);

  ```

=== "Go"

  ```go
  import (
    "time"

    "github.com/livekit/protocol/auth"
  )

  func getJoinToken(apiKey, apiSecret, room, identity string) (string, error) {

    at := auth.NewAccessToken(apiKey, apiSecret)

    videoGrant := &auth.VideoGrant{
      RoomJoin:     true,
      Room:         room,
    }

    sipGrant := &auth.SIPGrant{
      Admin:     true,
      Call:      true,
    }

    at.SetSIPGrant(sipGrant).
      SetVideoGrant(videoGrant).
      SetIdentity(identity).
      SetValidFor(time.Hour)

    return at.ToJWT()
  }

  ```

=== "Ruby"

  ```ruby
  require 'livekit'

  token = LiveKit::AccessToken.new(api_key: 'yourkey', api_secret: 'yoursecret')
  token.identity = 'participant-identity'
  token.name = 'participant-name'

  token.video_grant=(LiveKit::VideoGrant.from_hash(roomJoin: true,
                                                  room: 'room-name'))
  token.sip_grant=(LiveKit::SIPGrant.from_hash(admin: true, call: true))

  puts token.to_jwt

  ```

=== "Java"

  ```java
  import io.livekit.server.*;

  public String createToken() {
    AccessToken token = new AccessToken("apiKey", "secret");

    // Fill in token information.
    token.setName("participant-name");
    token.setIdentity("participant-identity");
    token.setMetadata("metadata");
  
    // Add room and SIP privileges.
    token.addGrants(new RoomJoin(true), new RoomName("room-name"));
    token.addSIPGrants(new SIPAdmin(true), new SIPCall(true));
    
    return token.toJwt();
  }

  ```

=== "Python"

  ```python
  from livekit import api
  import os

  token = api.AccessToken(os.getenv('LIVEKIT_API_KEY'),
                          os.getenv('LIVEKIT_API_SECRET')) \
      .with_identity("identity") \
      .with_name("name") \
      .with_grants(api.VideoGrants(
          room_join=True,
          room="my-room")) \
      .with_sip_grants(api.SIPGrants(
          admin=True,
          call=True)).to_jwt()

  ```

=== "Rust"

  ```rust
  use livekit_api::access_token;
  use std::env;

  fn create_token() -> Result<String, access_token::AccessTokenError> {
      let api_key = env::var("LIVEKIT_API_KEY").expect("LIVEKIT_API_KEY is not set");
      let api_secret = env::var("LIVEKIT_API_SECRET").expect("LIVEKIT_API_SECRET is not set");

      let token = access_token::AccessToken::with_api_key(&api_key, &api_secret)
          .with_identity("rust-bot")
          .with_name("Rust Bot")
          .with_grants(access_token::VideoGrants {
              room_join: true,
              room: "my-room".to_string(),
              ..Default::default()
          })  
          .with_sip_grants(access_token::SIPGrants {
              admin: true,
              call: true
          })  
          .to_jwt();
      return token
  }

  ```

## Room configuration

您可以為使用者建立包含房間配置選項的存取權杖(token)。當為使用者建立房間時，將使用儲存在令牌中的配置來建立房間。例如，當使用者加入房間時，您可以使用它來 [explicitly dispatch an agent](https://docs.livekit.io/agents/worker/agent-dispatch.md)。

有關 `RoomConfiguration` 欄位的完整列表，請參閱 [RoomConfiguration](https://docs.livekit.io/reference/server/server-apis.md#roomconfiguration)。

### Creating a token with room configuration

=== "Node.js"

  有關顯式代理程式排程的完整範例，請參閱 GitHub 中的[範例](https://github.com/livekit/node-sdks/blob/main/examples/agent-dispatch/index.ts)。

  ```typescript
  import { AccessToken, SIPGrant, VideoGrant } from 'livekit-server-sdk';
  import { RoomAgentDispatch, RoomConfiguration } from '@livekit/protocol';

  const roomName = 'name-of-room';
  const participantName = 'user-name';
  const agentName = 'my-agent';

  const at = new AccessToken('api-key', 'secret-key', {
    identity: participantName,
  });

  const videoGrant: VideoGrant = { 
    room: roomName,
    roomJoin: true,
  };  

  at.addGrant(videoGrant);
  at.roomConfig = new RoomConfiguration (
    agents: [
      new RoomAgentDispatch({
        agentName: "test-agent",
        metadata: "test-metadata"
      })
    ]
  );

  const token = await at.toJwt();
  console.log('access token', token);

  ```

=== "Go"

  ```go
  import (
    "time"

    "github.com/livekit/protocol/auth"
    "github.com/livekit/protocol/livekit"
  )

  func getJoinToken(apiKey, apiSecret, room, identity string) (string, error) {

    at := auth.NewAccessToken(apiKey, apiSecret)

    videoGrant := &auth.VideoGrant{
      RoomJoin:     true,
      Room:         room,
    }

    roomConfig := &livekit.RoomConfiguration{
      Agents: []*livekit.RoomAgentDispatch{{
        AgentName: "test-agent",
        Metadata:  "test-metadata",
      }}, 
    }

    at.SetVideoGrant(videoGrant).
      SetRoomConfig(roomConfig).
      SetIdentity(identity).
      SetValidFor(time.Hour)

    return at.ToJWT()
  }

  ```

=== "Ruby"

  ```ruby
  require 'livekit'

  token = LiveKit::AccessToken.new(api_key: 'yourkey', api_secret: 'yoursecret')
  token.identity = 'participant-identity'
  token.name = 'participant-name'

  token.video_grant=(LiveKit::VideoGrant.new(roomJoin: true,
                                            room: 'room-name'))
  token.room_config=(LiveKit::Proto::RoomConfiguration.new(
      max_participants: 10
      agents: [LiveKit::Proto::RoomAgentDispatch.new(
        agent_name: "test-agent",
        metadata: "test-metadata",
      )]
    )
  )

  puts token.to_jwt

  ```

=== "Python"

  有關顯式代理程式排程的完整範例，請參閱 GitHub 中的[範例](https://github.com/livekit/python-sdks/blob/main/examples/agent_dispatch.py)。

  ```python
  from livekit import api
  import os

  token = api.AccessToken(os.getenv('LIVEKIT_API_KEY'),
                          os.getenv('LIVEKIT_API_SECRET')) \
      .with_identity("identity") \
      .with_name("name") \
      .with_grants(api.VideoGrants(
          room_join=True,
          room="my-room")) \
          .with_room_config(
              api.RoomConfiguration(
                  agents=[
                      api.RoomAgentDispatch(
                          agent_name="test-agent", metadata="test-metadata"
                      )
                  ],
              ),
          ).to_jwt()

  ```

=== "Rust"

  ```rust
  use livekit_api::access_token;
  use std::env;

  fn create_token() -> Result<String, access_token::AccessTokenError> {
      let api_key = env::var("LIVEKIT_API_KEY").expect("LIVEKIT_API_KEY is not set");
      let api_secret = env::var("LIVEKIT_API_SECRET").expect("LIVEKIT_API_SECRET is not set");

      let token = access_token::AccessToken::with_api_key(&api_key, &api_secret)
          .with_identity("rust-bot")
          .with_name("Rust Bot")
          .with_grants(access_token::VideoGrants {
              room_join: true,
              room: "my-room".to_string(),
              ..Default::default()
          })
          .with_room_config(livekit::RoomConfiguration {
              agents: [livekit::AgentDispatch{
                name: "my-agent"
              }]  
          })  
          .to_jwt();
      return token
  }

  ```

## Token refresh

LiveKit 伺服器主動向已連線的用戶端發出刷新的令牌，確保它們在斷開連線時可以重新連線。這些刷新的存取令牌的有效期為 10 分鐘。

此外，當參與者的姓名、權限或元資料變更時，token 也會刷新。

## Updating permissions

參與者的權限可以隨時更新，即使他們已經連線。這在參與者的角色可能在會話期間發生變化的應用程式中很有用，例如在參與式直播中。

可以先發出一個帶有 `canPublish：false` 的令牌，然後在會話期間將其更新為 `canPublish：true`。可以使用 [UpdateParticipant](https://docs.livekit.io/home/server/managing-participants.md#updating-permissions) 伺服器 API 變更權限。