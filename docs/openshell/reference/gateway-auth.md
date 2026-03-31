# Gateway Authentication

本頁介紹 CLI 如何解析網關、如何透過網關進行身份驗證以及憑證的儲存位置。有關如何部署或註冊網關，請參閱「[管理沙箱](https://docs.nvidia.com/openshell/latest/sandboxes/manage-sandboxes.html)」。

## Gateway Resolution

當任何 CLI 命令需要與網關通訊時，它會透過 priority chain 解析目標：

1. `--gateway-endpoint <URL>` 旗標（直接 URL）。
2. `-g <NAME>` 旗標。
3. `OPENSHELL_GATEWAY` 環境變數。
4. 被定義在 `~/.config/openshell/active_gateway` 的 active gateway。

CLI 從磁碟載入網關元數據，以決定端點 URL 和驗證模式。

## Authentication Modes

CLI 根據網關的身份驗證配置使用三種連線模式之一。

### mTLS (local and remote gateways)

這是自部署網關的預設模式。當您執行 `gateway start` 或 `gateway add --local` / `gateway add --remote` 命令時，CLI 會從正在執行的容器中提取 mTLS 憑證並將其儲存在本機上。之後的每個請求都會提供客戶端憑證來驗證身分。

CLI 從 `~/.config/openshell/gateways/<name>/mtls/` 載入三個 PEM 檔案：

|文件|用途|
|---|----|
|`ca.crt`|CA憑證。驗證網關伺服器憑證。|
|`tls.crt`|客戶端證書。用於向網關證明 CLI 的身份。|
|`tls.key`|客戶端私鑰。|

連結流程：

1. CLI 載入三個憑證檔案。
2. 開啟與網關端點的 TCP 連線。
3. 執行 TLS 握手，並提供客戶端憑證。
4. 網關根據其 CA 驗證客戶端憑證。
5. 建立 HTTP/2 通道。所有 CLI 指令均使用此通道。

### Edge JWT (cloud gateways)

對於位於處理驗證的反向代理（例如 Cloudflare Access）後面的網關，CLI 使用基於 `browser-based login flow`，並透過 WebSocket 隧道路由流量。

**Registration flow** (`openshell gateway add https://gateway.example.com`):

1. CLI 使用 edge authentication mode 儲存網關元資料。
2. 開啟瀏覽器並存取網關的認證端點。
3. 反向代理處理登入（SSO、身分提供者等）。
4. 認證完成後，瀏覽器透過本機主機回呼將授權令牌傳回給 CLI。
5. CLI 儲存令牌並將網關設定為活動狀態。

**Connection flow** (subsequent commands):

1. CLI 啟動一個監聽臨時連接埠的本機代理程式。
2. 該代理程式開啟一個到網關的 WebSocket 連線 (`wss://`)，並將儲存的 bearer token 附加到升級請求頭中。
3. 反向代理驗證 WebSocket 升級請求。
4. 網關將 WebSocket 連線橋接到處理直接 mTLS 連線的相同服務中。
5. CLI 指令透過本機代理以明文 HTTP/2 協定經由 tunnel 傳送請求。

對用戶而言，這是透明的。無論網關使用 mTLS 認證還是邊緣認證，所有 CLI 命令的工作方式都相同。

**Re-authentication**: 如果令牌過期，請執行 `openshell gateway login` 以重新開啟瀏覽器流程並更新儲存的令牌。

### Plaintext

當網關部署時使用 `--plaintext` 參數，TLS 將會完全停用。 CLI 將透過純 HTTP/2 連線。此模式適用於位於受信任反向代理或隧道之後的網關，這些代理程式或隧道會在外部處理 TLS termination。

## File Layout

所有網關憑證和元資料都儲存在 `~/.config/openshell/` 目錄下：

```bash
openshell/
  active_gateway                    # Plain text: active gateway name
  gateways/
    <name>/
      metadata.json                 # Gateway metadata (endpoint, auth mode, type)
      mtls/                         # mTLS bundle (local and remote gateways)
        ca.crt                      # CA certificate
        tls.crt                     # Client certificate
        tls.key                     # Client private key
      edge_token                    # Edge auth JWT (cloud gateways)
      last_sandbox                  # Last-used sandbox for this gateway
```


