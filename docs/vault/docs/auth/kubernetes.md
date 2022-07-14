# Kubernetes Auth Method

原文: https://www.vaultproject.io/docs/auth/kubernetes

`kubernetes` auth 方法可用於使用 Kubernetes 服務帳戶(Service account)令牌向 Vault 進行身份驗證。這種身份驗證方法可以輕鬆地將 Vault 令牌引入 Kubernetes Pod。

您還可以使用 Kubernetes 服務帳戶令牌通過 [JWT 身份驗證登錄](https://www.vaultproject.io/docs/auth/jwt/oidc_providers#kubernetes)。請參閱[如何使用短期 Kubernetes 令牌](https://www.vaultproject.io/docs/auth/kubernetes#how-to-work-with-short-lived-kubernetes-tokens)部分，了解您可能希望使用 JWT 身份驗證的原因以及它與 Kubernetes 身份驗證的比較。

!!! info
    注意：如果您要升級到 Kubernetes v1.21+，請確保配置選項 `disable_iss_validation` 設置為 `true`。假設默認掛載路徑，您可以使用 `vault read -field disable_iss_validation auth/kubernetes/config` 檢查。有關更多詳細信息，請參閱下面的 [Kubernetes 1.21](https://www.vaultproject.io/docs/auth/kubernetes#kubernetes-1-21)。

## Authentication

### Via the CLI

默認路徑是 `/kubernetes`。如果在不同的路徑上啟用了此身份驗證方法，請在 CLI 中指定 `-path=/my-path`。

```bash
$ vault write auth/kubernetes/login role=demo jwt=...
```

### Via the API

默認端點是 `auth/kubernetes/login`。如果在不同的路徑上啟用了此身份驗證方法，請使用該值而不是 kubernetes。

```bash
$ curl \
    --request POST \
    --data '{"jwt": "<your service account jwt>", "role": "demo"}' \
    http://127.0.0.1:8200/v1/auth/kubernetes/login
```

響應將在 auth.client_token 包含一個令牌：

```json hl_lines="3"
{
  "auth": {
    "client_token": "38fe9691-e623-7238-f618-c94d4e7bc674",
    "accessor": "78e87a38-84ed-2692-538f-ca8b9f400ab3",
    "policies": ["default"],
    "metadata": {
      "role": "demo",
      "service_account_name": "vault-auth",
      "service_account_namespace": "default",
      "service_account_secret_name": "vault-auth-token-pd21c",
      "service_account_uid": "aa9aa8ff-98d0-11e7-9bb7-0800276d99bf"
    },
    "lease_duration": 2764800,
    "renewable": true
  }
}
```

## Configuration

在用戶或機器可以進行身份驗證之前，必須提前在 Vault　配置身份驗證方法。這些步驟通常由 Vault 管理員或配置管理工具完成。

1. 啟用 Kubernetes 身份驗證方法：

```bash
$ vault auth enable kubernetes
```

使用 `/config` 端點配置 Vault 以與 Kubernetes 通信。使用 `kubectl cluster-info` 驗證 Kubernetes 主機地址和 TCP 端口。有關可用配置選項的列表，請參閱 API 文檔。

```bash
$ vault write auth/kubernetes/config \
    token_reviewer_jwt="<your reviewer service account JWT>" \
    kubernetes_host=https://192.168.99.100:<your TCP port or blank for 443> \
    kubernetes_ca_cert=@ca.crt
```

創建 name role：

```bash
$ vault write auth/kubernetes/role/demo \
    bound_service_account_names=vault-auth \
    bound_service_account_namespaces=default \
    policies=default \
    ttl=1h
```

此角色授權默認命名空間中的 `vault-auth` 服務帳戶，並為其提供默認策略。

有關配置選項的完整列表，請參閱 [API 文檔](https://www.vaultproject.io/api-docs/auth/kubernetes)。

## Kubernetes 1.21

從版本 1.21 開始，Kubernetes `BoundServiceAccountTokenVolume` 功能默認為啟用。默認情況下，這會以兩種對 Kubernetes 身份驗證很重要的方式更改掛載到容器中的 JWT 令牌：

- 它有一個過期時間，並且綁定到 pod 和服務帳戶的生命週期。
- JWT 的 `iss` 聲明的值取決於集群的配置。

配置 token_reviewer_jwt 選項時，對令牌生命週期的更改很重要。如果使用短期令牌，Kubernetes 將在 pod 或服務帳戶被刪除或過期時間過後立即撤銷它，並且 Vault 將無法再使用 `TokenReview API`。

為了響應 `iss` 的更改，Kubernetes auth 已在 Vault 1.9.0 中更新，默認情況下不驗證頒發者。 Kubernetes API 在審查令牌時執行相同的驗證，因此在 Vault 端啟用頒發者驗證是重複的工作。

在不禁用 Vault 的頒發者驗證的情況下，單個 Kubernetes 身份驗證配置無法同時用於 Kubernetes 1.20 和 1.21 的默認掛載 pod 令牌。請注意，在 Vault 1.9 之前創建的身份驗證掛載將保持舊的默認值，您需要在將 Kubernetes 升級到 1.21 之前顯式設置 `disable_iss_validation=true`。如果您希望在 Vault 中啟用頒發者驗證，請參閱下面的發現服務帳戶頒發者以獲取指導。

### 如何克服 short-lived Kubernetes tokens 帶來的問題

當默認掛載的 pod 令牌是短暫的時，有幾種不同的方法可以為 Kubernetes pod 配置身份驗證，每種方法都有自己的權衡。此表總結了選項，下面將更詳細地解釋每個選項。

|Option	|All tokens are short-lived	|Can revoke tokens early	|Other considerations|
| ----- | ------------------------- | ------------------------- | ------------------- |
|Use local token as reviewer JWT	|Yes	|Yes	|Requires Vault (1.9.3+) to be deployed on the Kubernetes cluster|
|Use client JWT as reviewer JWT	|Yes	|Yes	|Operational overhead|
|Use long-lived token as reviewer JWT	|No	|Yes|	
|Use JWT auth instead	|Yes	|No|

#### Use local service account token as the reviewer JWT

在 Kubernetes pod 中運行 Vault 時，推薦的選項是使用 pod 的本地服務帳戶令牌。 Vault 將定期重新讀取文件以支持短期令牌。要使用本地令牌和 CA 證書，請在配置 auth 方法時省略 `token_reviewer_jwt` 和 `kubernetes_ca_cert`。 Vault 將嘗試分別從默認掛載文件夾 `/var/run/secrets/kubernetes.io/serviceaccount/` 中的 `token` 和 `ca.crt` 加載它們。

```bash
$ vault write auth/kubernetes/config \
    kubernetes_host=https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT
```

!!! warning
    注意：需要 Vault 1.9.3+。在早期版本中，服務帳戶令牌和 CA 證書被讀取一次並存儲在 Vault 存儲中。當服務帳戶令牌過期或被撤銷時，Vault 將無法再使用 TokenReview API，並且客戶端身份驗證將失敗。

#### Use the Vault client's JWT as the reviewer JWT

配置 Kubernetes auth 時，可以省略 `token_reviewer_jwt`，Vault 在與 Kubernetes TokenReview API 通信時會使用 Vault 客戶端的 JWT 作為自己的 auth token。如果 Vault 在 Kubernetes 中運行，您還需要設置 `disable_local_ca_jwt=true`。

這意味著 Vault 不存儲任何 JWT 並允許您在任何地方使用短期令牌，但會增加一些操作開銷來維護您希望能夠使用 Vault 進行身份驗證的服務帳戶集上的集群角色綁定。 Vault 的每個客戶端都需要 `system:auth-delegator` ClusterRole：

```bash
$ kubectl create clusterrolebinding vault-client-auth-delegator \
  --clusterrole=system:auth-delegator \
  --group=group1 \
  --serviceaccount=default:svcaccount1 \
  ...
```

#### Continue using long-lived tokens

您可以使用此處的說明創建一個長期存在的機密並將其用作 `token_reviewer_jwt`。在此示例中，`vault 服務帳戶`需要 `system:auth-delegator` ClusterRole：

```bash
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: vault-k8s-auth-secret
  annotations:
    kubernetes.io/service-account.name: vault
type: kubernetes.io/service-account-token
EOF
```

使用它可以維護以前的工作流程，但不會從短期令牌的改進安全狀態中受益。

#### Use JWT auth

Kubernetes auth 專門用於使用 Kubernetes 的 `TokenReview API`。但是，Kubernetes 生成的 JWT 令牌也可以使用 Kubernetes 作為 OIDC 提供程序進行驗證。 JWT auth 方法文檔包含使用 Kubernetes 作為 OIDC 提供程序設置 JWT auth 的說明。

此解決方案允許您為所有客戶端使用短期令牌，並且無需審查者 JWT。但是，客戶端令牌在其 TTL 到期之前不能被撤銷，因此建議在記住該限制的情況下保持 TTL 較短。

### 發現 Service account `issuer`

!!! info
    注意：從 Vault 1.9.0 開始，`disable_iss_validation` 和 `issuer` 已被棄用，並且 `disable_iss_validation` 的默認值已更改為 `true` 以用於新的 Kubernetes 身份驗證掛載。以下部分僅適用於您已設置 disable_iss_validation=false 或在 1.9 之前使用默認值創建掛載，但   `disable_iss_validation=true` 是所有 Vault 版本的新推薦值。


Kubernetes 1.21+ 集群可能需要將服務帳戶頒發者設置為與 kube-apiserver 的 `--service-account-issuer` 標誌相同的值。這是因為這些集群的服務帳戶 JWT 可能具有特定於集群本身的頒發者，而不是舊默認的 kubernetes/serviceaccount。如果您無法直接檢查此值，則可以運行以下命令並查找 `iss`字段以找到所需的值：

```bash
echo '{"apiVersion": "authentication.k8s.io/v1", "kind": "TokenRequest"}' \
  | kubectl create -f- --raw /api/v1/namespaces/default/serviceaccounts/default/token \
  | jq -r '.status.token' \
  | cut -d . -f2 \
  | base64 -D
```

大多數集群還將在 `.well-known/openid-configuration` 端點上提供該信息：

```bash
$ kubectl get --raw /.well-known/openid-configuration | jq -r .issuer
```

然後在配置 Kubernetes 身份驗證時使用此值，例如：

```bash
$ vault write auth/kubernetes/config \
  kubernetes_host="https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT" \
  issuer="\"test-aks-cluster-dns-d6cbb78e.hcp.uksouth.azmk8s.io\""
```

```json
{
  "issuer": "https://kubernetes.default.svc.cluster.local",
  "jwks_uri": "https://192.168.49.2:8443/openid/v1/jwks",
  "response_types_supported": [
    "id_token"
  ],
  "subject_types_supported": [
    "public"
  ],
  "id_token_signing_alg_values_supported": [
    "RS256"
  ]
}
```

## 配置 Kubernetes

此身份驗證方法訪問 [Kubernetes TokenReview API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.22/#tokenreview-v1-authentication-k8s-io) 以驗證提供的 JWT 是否仍然有效。 Kubernetes 應該使用 `--service-account-lookup` 運行。這在 Kubernetes 1.7 中默認為 true。否則 Kubernetes 中已刪除的令牌將無法正確撤銷，並且能夠通過此身份驗證方法進行身份驗證。

此身份驗證方法中使用的服務帳戶需要有權訪問 `TokenReview API`。如果 Kubernetes 配置為使用 RBAC 角色，則應授予服務帳戶訪問此 API 的權限。以下示例     `ClusterRoleBinding` 可用於授予這些權限：

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: role-tokenreview-binding
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
  - kind: ServiceAccount
    name: vault-auth
    namespace: default

```

## API

Kubernetes Auth 插件具有完整的 HTTP API。有關更多詳細信息，請參閱 [API 文檔](https://www.vaultproject.io/api-docs/auth/kubernetes)。

### 代碼示例

以下示例演示了使用 Vault 進行身份驗證的 Kubernetes auth 方法。

```golang
package main

import (
    "fmt"
    "os"

    vault "github.com/hashicorp/vault/api"
  auth "github.com/hashicorp/vault/api-docs/auth/kubernetes"
)

// Fetches a key-value secret (kv-v2) after authenticating to Vault with a Kubernetes service account.
// For a more in-depth setup explanation, please see the relevant readme in the hashicorp/vault-examples repo.
func getSecretWithKubernetesAuth() (string, error) {
    // If set, the VAULT_ADDR environment variable will be the address that
    // your pod uses to communicate with Vault.
    config := vault.DefaultConfig() // modify for more granular configuration

    client, err := vault.NewClient(config)
    if err != nil {
        return "", fmt.Errorf("unable to initialize Vault client: %w", err)
    }

    // The service-account token will be read from the path where the token's
    // Kubernetes Secret is mounted. By default, Kubernetes will mount it to
    // /var/run/secrets/kubernetes.io/serviceaccount/token, but an administrator
    // may have configured it to be mounted elsewhere.
    // In that case, we'll use the option WithServiceAccountTokenPath to look
    // for the token there.
    k8sAuth, err := auth.NewKubernetesAuth(
        "dev-role-k8s",
        auth.WithServiceAccountTokenPath("path/to/service-account-token"),
    )
    if err != nil {
        return "", fmt.Errorf("unable to initialize Kubernetes auth method: %w", err)
    }

    authInfo, err := client.Auth().Login(context.TODO(), k8sAuth)
    if err != nil {
        return "", fmt.Errorf("unable to log in with Kubernetes auth: %w", err)
    }
    if authInfo == nil {
        return "", fmt.Errorf("no auth info was returned after login")
    }

    // get secret from Vault, from the default mount path for KV v2 in dev mode, "secret"
    secret, err := client.KVv2("secret").Get(context.Background(), "creds")
    if err != nil {
        return "", fmt.Errorf("unable to read secret: %w", err)
    }

    // data map can contain more than one key-value pair,
    // in this case we're just grabbing one of them
    value, ok := secret.Data["password"].(string)
    if !ok {
        return "", fmt.Errorf("value type assertion failed: %T %#v", secret.Data["password"], secret.Data["password"])
    }

    return value, nil
}

```
