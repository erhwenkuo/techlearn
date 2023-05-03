# 通過 OAuth 2.0 提供商進行身份驗證

## 概述

Dex 用戶可以利用此連接器與符合標準的 OAuth 2.0 授權提供程序一起使用，以防這些授權提供程序不在 Dex 連接器列表中。

## 配置

以下是將 OAuth 連接器與 Reddit 結合使用的配置示例。

```yaml
connectors:
- type: oauth
  # ID of OAuth 2.0 provider
  id: reddit 
  # Name of OAuth 2.0 provider
  name: reddit
  config:
    # Connector config values starting with a "$" will read from the environment.
    clientID: $REDDIT_CLIENT_ID
    clientSecret: $REDDIT_CLIENT_SECRET
    redirectURI: http://127.0.0.1:5556/callback

    tokenURL: https://www.reddit.com/api/v1/access_token
    authorizationURL: https://www.reddit.com/api/v1/authorize
    userInfoURL: https://www.reddit.com/api/v1/me
 
    # Optional: Specify whether to communicate to Auth provider without
    # validating SSL certificates
    # insecureSkipVerify: false

    # Optional: The location of file containing SSL certificates to communicate
    # to Auth provider
    # rootCAs: /etc/ssl/reddit.pem

    # Optional: List of scopes to request Auth provider for access user account
    # scopes:
    #  - identity

    # Optional: Configurable keys for user ID look up
    # Default: id
    # userIDKey:

    # Auth providers return non-standard user identity profile
    # Use claimMapping to map those user informations to standard claims:
    claimMapping:
      # Optional: Configurable keys for user name look up
      # Default: user_name
      # userNameKey:

      # Optional: Configurable keys for preferred username look up
      # Default: preferred_username
      # preferredUsernameKey:

      # Optional: Configurable keys for user groups look up
      # Default: groups
      # groupsKey:

      # Optional: Configurable keys for email look up
      # Default: email
      # emailKey:

      # Optional: Configurable keys for email verified look up
      # Default: email_verified
      # emailVerifiedKey:
```
