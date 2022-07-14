# Userpass Auth Method

原文: https://www.vaultproject.io/docs/auth/userpass

`userpass` auth 方法允許用戶使用 `username` 和 `password` 組合向 Vault 進行身份驗證。

用戶名/密碼組合直接配置為使用 `users/` 路徑的 auth 方法。此方法無法從外部源讀取用戶名和密碼。

該方法將所有提交的用戶名小寫，例如 Mary 和 mary 是同一個使用者。

## Authentication

### Via the CLI

---

```bash
$ vault login -method=userpass \
    username=mitchellh \
    password=foo
```

### Via the API

---

```bash
$ curl \
    --request POST \
    --data '{"password": "foo"}' \
    http://127.0.0.1:8200/v1/auth/userpass/login/mitchellh
```

響應將包含 `auth.client_token` 中的令牌：

```json hl_lines="7"
{
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": null,
  "auth": {
    "client_token": "c4f280f6-fdb2-18eb-89d3-589e2e834cdb",
    "policies": ["admins"],
    "metadata": {
      "username": "mitchellh"
    },
    "lease_duration": 0,
    "renewable": false
  }
}
```

## Configuration

在用戶或機器可以進行身份驗證之前，必須提前配置身份驗證方法。這些步驟通常由Vault 管理員或某種配置管理工具完成。

1. 啟用 userpass auth 方法：

```bash
$ vault auth enable userpass
```

2. 使用允許進行身份驗證的用戶對其進行配置：

```bash
$ vault write auth/userpass/users/mitchellh \
    password=foo \
    policies=admins
```

這將創建一個密碼為 `foo` 的新用戶 `mitchellh`，該用戶將與 `admins` 策略相關聯。這是唯一需要的配置。

## API

Userpass 身份驗證方法具有完整的 HTTP API。有關更多詳細信息，請參閱 [Userpass 身份驗證方法 API](https://www.vaultproject.io/api-docs/auth/userpass)。