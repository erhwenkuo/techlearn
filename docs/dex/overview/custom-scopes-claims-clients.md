# Custom Scopes, Claims 與 Client Features

本文檔描述了 dex 實現的 OAuth2 和 OpenID Connect 功能。

## Scopes

以下是 dex 支持的 scopes 的詳盡列表：

| Name | Description |
| ---- | ------------|
| `openid` | 所有登錄請求的必需要有的資訊。 |
| `email` | ID 令牌聲明應包括最終用戶的電子郵件以及該電子郵件是否已由上游提供商驗證。|
| `profile` | ID 令牌聲明應包括最終用戶的用戶名。 |
| `groups` | ID 令牌聲明應包括最終用戶所屬群組的列表。 |
| `federated:id` | ID 令牌聲明應包括來自 ID 提供者的信息。令牌將包含連接器 ID 和在提供商處分配的用戶 ID。|
| `offline_access` | 令牌響應應包含 refresh 令牌。不能與某些連接器結合使用，值得注意的是 [SAML 連接器](/docs/connectors/saml/) 會忽略此資訊。 |
| `audience:server:client_id:( client-id )` | 動態範圍指示應代表另一個客戶端頒發 ID 令牌。請參閱下面的 “跨客戶端信任和授權方” 部分。|

## Custom claims

除了所需的 OpenID Connect 聲明和一些標準聲明之外，dex 還實現了以下非標準聲明。

| Name | Description |
| ---- | ------------|
| `groups` | 表示用戶所屬群組的字符串列表。 |
| `federated_claims` | 在提供商處分配給用戶的連接器 ID 和用戶 ID。 |
| `email` | 用戶的電子郵件。 |
| `email_verified` | 如果上游提供商已驗證電子郵件。 |
| `name` | 用戶的顯示名稱。 |

`federated_claims` 聲明具有以下格式：

```json
"federated_claims": {
  "connector_id": "github",
  "user_id": "110272483197731336751"
}
```

## Cross-client trust 與 authorized party

Dex 有能力代表其他客戶向客戶發放 ID 令牌。在 OpenID Connect 術語中，這意味著 ID 令牌的 “aud”（受眾）聲明是與執行登錄的客戶端不同的客戶端 ID。

例如，此功能可用於允許 Web 應用代表命令行工具來生成 ID 令牌：

```yaml    {
        "aud": "cli-app",
        "azp": "web-app",
        "email": "foo@bar.com",
        // other claims...
    }
    ``` 
staticClients:
- id: web-app
  redirectURIs:
  - 'https://web-app.example.com/callback'
  name: 'Web app'
  secret: web-app-secret

- id: cli-app
  redirectURIs:
  - 'https://cli-app.example.com/callback'
  name: 'Command line tool'
  secret: cli-app-secret
  # The command line tool lets the web app issue ID tokens on its behalf.
  trustedPeers:
  - web-app
```

!!! tip
    請注意，命令行工具必須使用 “trustedPeers” 欄位來宣告明確信任 `web-app`。然後，web-app 可以使用以下範圍來請求為命令行工具頒發的 ID 令牌。

    ```
    audience:server:client_id:cli-app
    ```

    ID 令牌聲明將包括以下受眾和授權方：

    ```json
    {
        "aud": "cli-app",
        "azp": "web-app",
        "email": "foo@bar.com",
        // other claims...
    }
    ```

## Public clients

公共客戶端受到谷歌“[已安裝應用程序](https://developers.google.com/api-client-library/python/auth/installed-app)”的啟發，旨在對不打算將其客戶端機密保密的應用程序施加限制。可以使用 `public` 配置選項將客戶端聲明為公共的。

```yaml
staticClients:
- id: cli-app
  public: true
  name: 'CLI app'
  redirectURIs:
  - ...
```

如果未指定 `redirectURI`，公共客戶端僅支持以 “http://localhost” 或特殊 “out-of-browser” URL “urn:ietf:wg:oauth:2.0:oob” 開頭的重定向。後者觸發 dex 在瀏覽器中顯示 OAuth2 代碼，提示最終用戶手動將其複製到他們的應用程序中。客戶有責任創建屏幕或提示以接收代碼，然後執行代碼交換以獲得令牌響應。

使用“out-of-browser” 流程時，強烈建議使用 ID 令牌隨機數。