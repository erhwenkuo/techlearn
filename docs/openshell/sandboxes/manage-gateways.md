# Deploy and Manage Gateways

網關是 OpenShell 的控制平面。 CLI 與運作中的沙箱之間所有控制平面流量都透過網關傳輸。

網關負責：

- 沙箱的配置和管理，包括建立、刪除和狀態監控。
- 儲存提供者憑證（API 金鑰、令牌），並在啟動時將其交付給沙箱。
- 向沙箱交付網路和檔案系統策略。策略執行本身在每個沙箱內部透過代理、OPA、Landlock 和 seccomp 進行。
- 管理模形推論配置並提供推理包，以便沙箱可以將請求路由到正確的後端。
- 提供 SSH tunnel 端點，以便您可以在不直接暴露沙箱的情況下連接到沙箱。

此網關運行在 Docker 容器內，並暴露一個連接埠（gRPC 和 HTTP 多路復用），預設使用 mTLS 進行安全性保護。無需單獨安裝 Kubernetes。它可以部署在本地端、透過 SSH 部署在遠端主機上，或部署在雲端反向代理程式之後。

## Deploy a Local Gateway

在工作站上部署網關。唯一的前提條件是 Docker daemon 程式正在執行。

```bash
openshell gateway start
```

網關可透過 `https://127.0.0.1:8080` 存取。請驗證是否運作正常：

```bash
openshell status
```

!!! tip
    您無需手動部署網關。如果您在未指定網關的情況下執行 `openshell sandbox create` 命令，CLI 會自動為您引導一個本機網關。

若要使用不同的連接埠或名稱：

```bash
openshell gateway start --port 9090
openshell gateway start --name dev-local
```

## Deploy a Remote Gateway

在可透過 SSH 存取的遠端機器上部署網關。遠端主機唯一相依性是 Docker。

```bash
openshell gateway start --remote user@hostname
```

可透過 `https://<主機名稱>:8080` 存取網關。

若要指定 SSH 金鑰：

```bash
openshell gateway start --remote user@hostname --ssh-key ~/.ssh/my_key
```

## Register an Existing Gateway

使用 `openshell gateway add` 註冊一個已經在執行的網關。

### Cloud Gateway

在反向代理（例如 Cloudflare Access）後面註冊網關：

```bash
openshell gateway add https://gateway.example.com
```

這將打開瀏覽器，進入代理伺服器的登入流程。身份驗證成功後，CLI 會儲存持有者令牌，並將網關設為活動狀態。

若要為網關指定一個特定名稱，而不是從主機名稱派生，請使用 `--name` 參數。

```bash
openshell gateway add https://gateway.example.com --name production
```

如果令牌稍後過期，請使用以下方式重新驗證：

```bash
openshell gateway login
```

## Remote Gateway

在具有 SSH 存取權限的遠端主機上註冊網關：

```bash
openshell gateway add https://remote-host:8080 --remote user@remote-host
```

或使用 `ssh://` 方案將 SSH 目標位址和閘道連接埠組合在一起：

```bash
openshell gateway add ssh://user@remote-host:8080
```

## Local Gateway

註冊一個在本機運作且未透過 CLI 啟動的網關：

```bash
openshell gateway add https://127.0.0.1:8080 --local
```

## Manage Multiple Gateways

總是會有一個網關處於活動狀態。所有 CLI 命令預設都以該網關為目標。 `gateway start` 和 `gateway add` 指令都會自動將新閘道設定為活動閘道。

**列出所有已註冊的網關**：

```bash
openshell gateway select
```

**切換活動網關**：

```bash
openshell gateway select my-remote-cluster
```

使用 `-g` 參數覆寫單一指令的活動網關：

```bash
openshell status -g my-other-cluster
```

顯示網關的部署詳情，包括端點, 認證模式和連接埠：

```bash
openshell gateway info

openshell gateway info --name my-remote-cluster
```

## Advanced Start Options

|旗標|用途|
|---|----|
|`--gpu`|啟用 NVIDIA GPU 直通。需要在主機上安裝 NVIDIA 驅動程式和容器工具包。|
|`--plaintext`|監聽 HTTP 協定而非 mTLS 協定。使用 TLS 終止反向代理。|
|`--disable-gateway-auth`|跳過 mTLS 用戶端憑證檢查。當反向代理無法轉發客戶端憑證時使用。|
|`--registry-username`|用於註冊表身份驗證的使用者名稱。如果設定了 `--registry-token`，則預設為 `__token__`。僅私有註冊表需要此設定。也可以使用 `OPENSHELL_REGISTRY_USERNAME` 進行配置。|
|`--registry-token`|用於拉取容器鏡像的身份驗證令牌。對於 GHCR，需要具有 `read:packages` 權限的 GitHub PAT。僅私有鏡像倉庫需要。也可以使用 **OPENSHELL_REGISTRY_TOKEN** 進行配置。|

## Stop and Destroy

停止網關，同時保留其狀態以便稍後重新啟動：

```bash
openshell gateway stop
```

永久銷毀網關及其所有狀態：

```bash
openshell gateway destroy
```

對於雲端網關，網關銷毀操作只會移除本機註冊訊息，不會影響遠端部署。

使用 `--name` 指定目標閘道：

```bash
openshell gateway stop --name my-gateway

openshell gateway destroy --name my-gateway
```

## Troubleshoot

檢查網關健康狀況：

```bash
openshell status
```

查看網關日誌：

```bash
openshell doctor logs
openshell doctor logs --tail              # stream live
openshell doctor logs --lines 50          # last 50 lines
```

在網關容器內執行命令以進行更深入的檢查：

```bash
openshell doctor exec -- kubectl get pods -A
openshell doctor exec -- sh
```

如果網關狀態異常，請重新建立它：

```bash
openshell gateway start --recreate
```
