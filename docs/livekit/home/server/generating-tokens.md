# Generating tokens

> 為你的前端產生 AccessToken 令牌

為了讓前端應用程式連接到 LiveKit 房間，它們需要由後端伺服器產生的令牌(access token)。在本指南中，我們將介紹如何設定伺服器來為您的前端產生令牌。

## 1. Install LiveKit Server SDK


=== "Python"
    ```bash
    pip install livekit-api\
    
    ```

=== "Node.js"
    ```bash
    # yarn:
    yarn add livekit-server-sdk

    # npm:
    npm install livekit-server-sdk --save

    ```

=== "Go"
    ```bash
    go get github.com/livekit/server-sdk-go/v2

    ```

=== "Ruby"
    ```ruby
    # Add to your Gemfile
    gem 'livekit-server-sdk'

    ```

=== "Rust"
    ```toml
    # Cargo.toml
    [package]
    name = "example_server"
    version = "0.1.0"
    edition = "2021"

    [dependencies]
    livekit-api = "0.2.0"
    # Remaining deps are for the example server
    warp = "0.3"
    serde = { version = "1.0", features = ["derive"] }
    serde_json = "1.0"
    tokio = { version = "1", features = ["full"] }

    ```

=== "PHP"
    ```bash
    composer require agence104/livekit-server-sdk

    ```

## 2. Keys and Configuration

在 `development.env` 中建立一個新文件，並使用您的 API Key 和 Secret：

```bash
export LIVEKIT_API_KEY=%{apiKey}%
export LIVEKIT_API_SECRET=%{apiSecret}%

```

## 3. Make an endpoint that returns a token

建立伺服器：


=== "Python"
    ```python
    # server.py
    import os
    from livekit import api
    from flask import Flask

    app = Flask(__name__)

    @app.route('/getToken')
    def getToken():
      token = api.AccessToken(os.getenv('LIVEKIT_API_KEY'), os.getenv('LIVEKIT_API_SECRET')) \
        .with_identity("identity") \
        .with_name("my name") \
        .with_grants(api.VideoGrants(
            room_join=True,
            room="my-room",
        ))
      return token.to_jwt()

    ```


=== "Node.js"
    ```js
    // server.js
    import express from 'express';
    import { AccessToken } from 'livekit-server-sdk';

    const createToken = async () => {
      // If this room doesn't exist, it'll be automatically created when the first
      // participant joins
      const roomName = 'quickstart-room';
      // Identifier to be used for participant.
      // It's available as LocalParticipant.identity with livekit-client SDK
      const participantName = 'quickstart-username';

      const at = new AccessToken(process.env.LIVEKIT_API_KEY, process.env.LIVEKIT_API_SECRET, {
        identity: participantName,
        // Token to expire after 10 minutes
        ttl: '10m',
      });
      at.addGrant({ roomJoin: true, room: roomName });

      return await at.toJwt();
    };

    const app = express();
    const port = 3000;

    app.get('/getToken', async (req, res) => {
      res.send(await createToken());
    });

    app.listen(port, () => {
      console.log(`Server listening on port ${port}`);
    });

    ```

=== "Go"
    ```go
    // server.go
    import (
      "net/http"
      "log"
      "time"
      "os"

      "github.com/livekit/protocol/auth"
    )

    func getJoinToken(room, identity string) string {
      at := auth.NewAccessToken(os.Getenv("LIVEKIT_API_KEY"), os.Getenv("LIVEKIT_API_SECRET"))
      grant := &auth.VideoGrant{
        RoomJoin: true,
        Room: room,
      }
      at.AddGrant(grant).
        SetIdentity(identity).
        SetValidFor(time.Hour)

      token, _ := at.ToJWT()
      return token
    }

    func main() {
      http.HandleFunc("/getToken", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte(getJoinToken("my-room", "identity")))
      })

      log.Fatal(http.ListenAndServe(":8080", nil))
    }

    ```

=== "Ruby"
    ```ruby
    # server.rb
    require 'livekit'
    require 'sinatra'

    def createToken()
      token = LiveKit::AccessToken.new(api_key: ENV['LIVEKIT_API_KEY'], api_secret: ENV['LIVEKIT_API_SECRET'])
      token.identity = 'quickstart-identity'
      token.name = 'quickstart-name'
      token.add_grant(roomJoin: true, room: 'room-name')

      token.to_jwt
    end

    get '/getToken' do
      createToken
    end

    ```

=== "Rust"
    ```rust
    // src/main.rs

    use livekit_api::access_token;
    use warp::Filter;
    use serde::{Serialize, Deserialize};
    use std::env;

    #[tokio::main]
    async fn main() {
        // Define the route
        let create_token_route = warp::path("create-token")
            .map(|| {
                let token = create_token().unwrap();
                warp::reply::json(&TokenResponse { token })
            });

        // Start the server
        warp::serve(create_token_route).run(([127, 0, 0, 1], 3030)).await;
    }

    // Token creation function
    fn create_token() -> Result<String, access_token::AccessTokenError> {
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

    // Response structure
    #[derive(Serialize, Deserialize)]
    struct TokenResponse {
        token: String,
    }

    ```

=== "PHP"
    ```php
    // If this room doesn't exist, it'll be automatically created when the first
    // participant joins.
    $roomName = 'name-of-room';
    // The identifier to be used for participant.
    $participantName = 'user-name';

    // Define the token options.
    $tokenOptions = (new AccessTokenOptions())
      ->setIdentity($participantName);

    // Define the video grants.
    $videoGrant = (new VideoGrant())
      ->setRoomJoin()
      ->setRoomName($roomName);

    // Initialize and fetch the JWT Token.
    $token = (new AccessToken(getenv('LIVEKIT_API_KEY'), getenv('LIVEKIT_API_SECRET')))
      ->init($tokenOptions)
      ->setGrant($videoGrant)
      ->toJwt();

    ```

載入環境變數並運行伺服器：

=== "Python"
    ```bash
    $ source development.env
    $ python server.py

    ```

=== "Node.js"
    ```bash
    $ source development.env
    $ node server.js

    ```

=== "Go"
    ```bash
    $ source development.env
    $ go run server.go

    ```

=== "Ruby"
    ```bash
    $ source development.env
    $ ruby server.rb

    ```

=== "Rust"
    ```bash
    $ source development.env
    $ cargo r src/main.rs

    ```

=== "PHP"
    ```bash
    $ source development.env
    $ php server.php

    ```

!!! info
    有關如何使用自訂權限產生令牌的更多信息，請參閱 [Authentication](https://docs.livekit.io/home/get-started/authentication.md)。

## 4. Create a frontend app to connect

創建一個前端應用程序，從我們剛剛創建的伺服器獲取令牌，然後使用它連接到 LiveKit 房間：

- [iOS](https://docs.livekit.io/home/quickstarts/swift.md)
- [Android](https://docs.livekit.io/home/quickstarts/android.md)
- [Flutter](https://docs.livekit.io/home/quickstarts/flutter.md)
- [React Native](https://docs.livekit.io/home/quickstarts/react-native.md)
- [React](https://docs.livekit.io/home/quickstarts/react.md)
- [Unity (web)](https://docs.livekit.io/home/quickstarts/unity-web.md)
- [JavaScript](https://docs.livekit.io/home/quickstarts/javascript.md)
