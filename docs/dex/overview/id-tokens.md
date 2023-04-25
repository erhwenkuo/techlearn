# ID 令牌

## 概述

ID 令牌是由 OpenID Connect 和 Dex 的主要功能引入的 OAuth2 擴展。 ID 令牌是由 Dex 簽名的 [JSON Web 令牌 (JWT)](https://jwt.io/)，並作為證明最終用戶身份的 OAuth2 響應的一部分返回。示例 JWT 可能如下所示：

```console
eyJhbGciOiJSUzI1NiIsImtpZCI6IjlkNDQ3NDFmNzczYjkzOGNmNjVkZDMyNjY4NWI4NjE4MGMzMjRkOTkifQ.eyJpc3MiOiJodHRwOi8vMTI3LjAuMC4xOjU1NTYvZGV4Iiwic3ViIjoiQ2djeU16UXlOelE1RWdabmFYUm9kV0kiLCJhdWQiOiJleGFtcGxlLWFwcCIsImV4cCI6MTQ5Mjg4MjA0MiwiaWF0IjoxNDkyNzk1NjQyLCJhdF9oYXNoIjoiYmk5NmdPWFpTaHZsV1l0YWw5RXFpdyIsImVtYWlsIjoiZXJpYy5jaGlhbmdAY29yZW9zLmNvbSIsImVtYWlsX3ZlcmlmaWVkIjp0cnVlLCJncm91cHMiOlsiYWRtaW5zIiwiZGV2ZWxvcGVycyJdLCJuYW1lIjoiRXJpYyBDaGlhbmcifQ.OhROPq_0eP-zsQRjg87KZ4wGkjiQGnTi5QuG877AdJDb3R2ZCOk2Vkf5SdP8cPyb3VMqL32G4hLDayniiv8f1_ZXAde0sKrayfQ10XAXFgZl_P1yilkLdknxn6nbhDRVllpWcB12ki9vmAxklAr0B1C4kr5nI3-BZLrFcUR5sQbxwJj4oW1OuG6jJCNGHXGNTBTNEaM28eD-9nhfBeuBTzzO7BKwPsojjj4C9ogU4JQhGvm_l4yfVi0boSx8c0FX3JsiB0yLa1ZdJVWVl9m90XmbWRSD85pNDQHcWZP9hR6CMgbvGkZsgjG32qeRwUL_eNkNowSBNWLrGNPoON1gMg
```

!!! info
    有關 JWT 的詳細說明, 請見: [是誰在敲打我窗？什麼是 JWT ？](https://5xruby.tw/posts/what-is-jwt)

![](./assets/jwt-token.png)

ID 令牌包含標準聲明 (standard claims) 斷言用戶登錄的客戶端應用程序、令牌何時過期以及用戶的身份。

```json
{
  "iss": "http://127.0.0.1:5556/dex",
  "sub": "CgcyMzQyNzQ5EgZnaXRodWI",
  "aud": "example-app",
  "exp": 1492882042,
  "iat": 1492795642,
  "at_hash": "bi96gOXZShvlWYtal9Eqiw",
  "email": "jane.doe@coreos.com",
  "email_verified": true,
  "groups": [
    "admins",
    "developers"
  ],
  "name": "Jane Doe"
}
```

因為這些令牌由 Dex 簽名並包含基於標準的聲明，所以其他服務可以將它們用作服務到服務的憑證。已經可以使用 Dex 發行的 OpenID Connect ID 令牌的系統包括：

- [Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens) 
- [AWS STS](https://docs.aws.amazon.com/STS/latest/APIReference/Welcome.html)

有關如何請求或驗證 ID 令牌的詳細信息，請參閱“[編寫使用 Dex 的應用程序](https://dexidp.io/docs/using-dex/)”。

## Refresh 令牌

`Refresh 令牌`是用於獲取 access token 的憑據。`Refresh 令牌`由授權服務器頒發給客戶端，用於在當前 `id 令牌`失效或過期時獲取新的 `id 令牌`。發布 `refresh 令牌` 是可選的功能，通過將 `offline_access` scope 傳遞給 Dex 服務器來提供。

!!! info
    某些 IdP 連接器不支持 `offline_access` scope。您可以通過查看[連接器列表](https://dexidp.io/docs/connectors/)來了解哪些連接器支持 `refresh 令牌`。

帶有 `refresh 令牌`的服務器響應示例：

```json
{
 "access_token": "eyJhbGciOiJSUzI1N...",
 "token_type": "Bearer",
 "refresh_token": "lxzzsvasxho5exvwkfa5zhefl",
 "expires_in": 3600,
 "id_token": "eyJhbGciO..."
}
```

!!! info
    對於 id 令牌的每次刷新，Dex 都會發出一個新的 `refresh 令牌`。這種安全措施稱為 refresh token rotation，可防止有人竊取它。這個想法在相應的 [RFC](https://tools.ietf.org/html/rfc6819#section-5.2.2.3) 中有詳細描述。

## 過期和輪換設置

Dex 在配置文件中有一個部分，您可以在其中為 ID 令牌和刷新令牌指定到期和輪換設置。

```yaml title="dex/examples/config-dev.yaml"
...
...
    # 可以僅使用 s、m 和 h 後綴指定單位。
    expiry:
      deviceRequests: "5m"
      signingKeys: "6h"
      idTokens: "24h"
      refreshTokens:
        reuseInterval: "3s"
        validIfNotUsedFor: "2160h" # 90 days
        absoluteLifetime: "3960h" # 165 days
...
...
```

!!! info
    所有持續時間選項應以以下格式設置：數字 + 時間單位（s、m、h），例如 10m。

- `expiry` - 各種到期設置的部分，包括令牌設置：
    - `idTokens` - id_token 的生命週期。最好使用短期 id 令牌。
    - `refreshTokens` - 各種刷新令牌設置的部分：
        - `validIfNotUsedFor` - 如果在指定的時間內未使用，則使刷新令牌無效。
        - `absoluteLifetime` - 前一個選項的更嚴格變體，刷新令牌的絕對生命週期。它強制用戶重新驗證並獲取新的刷新令牌。
        - `disableRotation` - 完全禁用每個請求的旋轉。用戶還必須指定先前的刷新令牌選項之一，以在切換時保持刷新令牌的安全。
        - `reuseInterval` - 允許在指定的時間間隔內從刷新端點獲取相同的刷新令牌，但前提是用戶的請求包含先前的刷新令牌。

!!! info
    `disableRotation` 和 `reuseInterval` 選項有助於有效地處理網絡延遲、並發請求等問題，以換取安全性。謹慎使用它們。
