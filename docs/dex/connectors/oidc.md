# 通過 OpenID Connect 提供商進行身份驗證

## 概述

Dex 能夠使用另一個 OpenID Connect 提供程序作為身份驗證源。登錄時，dex 將重定向到上游提供程序並執行必要的 OAuth2 流程以確定最終用戶的電子郵件、用戶名等。有關 OpenID Connect 協議的更多詳細信息，請參閱 [OpenID Connect 概述](https://dexidp.io/docs/openid-connect/)。

OpenID Connect 提供商的示例包括 Google Accounts、Salesforce 和 Azure AD v2。

## 配置

```yaml
connectors:
- type: oidc
  id: google
  name: Google
  config:
    # Canonical URL of the provider, also used for configuration discovery.
    # This value MUST match the value returned in the provider config discovery.
    #
    # See: https://openid.net/specs/openid-connect-discovery-1_0.html#ProviderConfig
    issuer: https://accounts.google.com

    # Connector config values starting with a "$" will read from the environment.
    clientID: $GOOGLE_CLIENT_ID
    clientSecret: $GOOGLE_CLIENT_SECRET

    # Dex's issuer URL + "/callback"
    redirectURI: http://127.0.0.1:5556/callback


    # Some providers require passing client_secret via POST parameters instead
    # of basic auth, despite the OAuth2 RFC discouraging it. Many of these
    # cases are caught internally, but some may need to uncomment the
    # following field.
    #
    # basicAuthUnsupported: true
    
    # List of additional scopes to request in token response
    # Default is profile and email
    # Full list at https://dexidp.io/docs/custom-scopes-claims-clients/
    # scopes:
    #  - profile
    #  - email
    #  - groups

    # Some providers return claims without "email_verified", when they had no usage of emails verification in enrollment process
    # or if they are acting as a proxy for another IDP etc AWS Cognito with an upstream SAML IDP
    # This can be overridden with the below option
    # insecureSkipEmailVerified: true 

    # Groups claims (like the rest of oidc claims through dex) only refresh when the id token is refreshed
    # meaning the regular refresh flow doesn't update the groups claim. As such by default the oidc connector
    # doesn't allow groups claims. If you are okay with having potentially stale group claims you can use
    # this option to enable groups claims through the oidc connector on a per-connector basis.
    # This can be overridden with the below option
    # insecureEnableGroups: true

    # When enabled, the OpenID Connector will query the UserInfo endpoint for additional claims. UserInfo claims
    # take priority over claims returned by the IDToken. This option should be used when the IDToken doesn't contain
    # all the claims requested.
    # https://openid.net/specs/openid-connect-core-1_0.html#UserInfo
    # getUserInfo: true

    # The set claim is used as user id.
    # Claims list at https://openid.net/specs/openid-connect-core-1_0.html#Claims
    # Default: sub
    # userIDKey: nickname

    # The set claim is used as user name.
    # Default: name
    # userNameKey: nickname

    # The acr_values variable specifies the Authentication Context Class Values within
    # the Authentication Request that the Authorization Server is being requested to process
    # from this Client.
    # acrValues: 
    #  - <value>
    #  - <value>

    # For offline_access, the prompt parameter is set by default to "prompt=consent". 
    # However this is not supported by all OIDC providers, some of them support different
    # value for prompt, like "prompt=login" or "prompt=none"
    # promptType: consent

    # Some providers return non-standard claims (eg. mail).
    # Use claimMapping to map those claims to standard claims:
    # https://openid.net/specs/openid-connect-core-1_0.html#Claims
    # claimMapping can only map a non-standard claim to a standard one if it's not returned in the id_token.
    claimMapping:
      # The set claim is used as preferred username.
      # Default: preferred_username
      # preferred_username: other_user_name

      # The set claim is used as email.
      # Default: email
      # email: mail

      # The set claim is used as groups.
      # Default: groups
      # groups: "cognito:groups"
```
