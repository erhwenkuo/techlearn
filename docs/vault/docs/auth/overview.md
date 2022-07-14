# Auth Methods

原文: https://www.vaultproject.io/docs/auth

身份驗證方法是 Vault 中執行身份驗證的組件，負責為用戶分配身份和一組策略。在所有情況下，Vault 都會在請求處理過程中強制執行身份驗證。在大多數情況下，Vault 會將身份驗證管理和決策委託給相關配置的外部身份驗證方法（例如，Amazon Web Services、GitHub、Google Cloud Platform、Kubernetes、Microsoft Azure、Okta ...）。

擁有多個身份驗證方法使您能夠使用對您的 Vault 和您的組織的用例最有意義的身份驗證方法。

例如，在程式開發者的電腦上，`GitHub auth` 方法最容易使用。但是對於服務器，`AppRole` 方法是推薦的選擇。

要了解有關身份驗證的更多信息，請參閱[身份驗證概念](https://www.vaultproject.io/docs/concepts/auth)頁面。

## Enabling/Disabling Auth Methods

可以使用 CLI 或 API 啟用/禁用身份驗證方法。

```bash
$ vault auth enable userpass
```

啟用後，auth 方法類似於 secrets engines：它們被掛載在 Vault mount table 中，並且可以使用標準讀/寫 API 訪問和配置。所有 auth 方法都安裝在 `auth/` 前綴下。

默認情況下，auth 方法掛載到 `auth/<type>`。例如，如果您啟用“github”，那麼您可以在 `auth/github` 與它進行交互。但是，此路徑是可自定義的，允許具有高級用例的用戶多次掛載單個身份驗證方法。

```bash
$ vault auth enable -path=my-login userpass
```

禁用身份驗證方法時，通過該方法進行身份驗證的所有用戶都將自動註銷。

## External Auth Method Considerations

當使用外部身份驗證方法（例如 GitHub）時，Vault 將在身份驗證時調用外部服務以及任何後續令牌更新。這意味著已發行的令牌在其整個有效期內都有效，並且在更新或用戶重新身份驗證發生之前不會失效。運營商應確保在使用這些身份驗證方法時設置適當的`令牌 TTL`。

