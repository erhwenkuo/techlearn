# Write Your First Sandbox Network Policy

本教學將在五分鐘內示範 OpenShell 的網路策略系統如何運作。您將建立一個沙箱環境，觀察請求如何被預設拒絕策略阻止，應用細粒度的 L7 規則，並驗證在寫入被阻止的情況下允許讀取操作，所有操作均無需重新啟動任何服務。

完成本教學後，您將了解：

- 如何透過預設拒絕網路策略來阻止沙箱的所有出站流量？
- 如何應用網路策略，授予對特定 API 的唯讀存取權限？
- L7 層強制執行如何區分同一端點上的 HTTP 方法，例如 GET 和 POST？
- 如何檢查拒絕日誌以獲取完整的審計追蹤？

## Prerequisites

- 已安裝並可正常運作的 OpenShell。請先完成快速入門指南再繼續。
- 您的電腦上已執行 Docker Desktop 或 Docker Engine。

!!! tip
    要執行本教學的每個步驟，您也可以使用 NVIDIA OpenShell 程式碼庫 [examples/sandbox-policy-quickstart](https://github.com/NVIDIA/OpenShell/blob/main/examples/sandbox-policy-quickstart) 目錄中的自動化示範腳本。該腳本可在不到一分鐘的時間內完成整個教程，無需任何用戶互動。

    ```bash
    bash examples/sandbox-policy-quickstart/demo.sh
    ```

## Create a Sandbox

首先建立一個不包含任何網路策略的沙箱環境。這樣可以創造一個乾淨的環境來觀察預設拒絕行為。

```bash
openshell sandbox create --name demo --keep --no-auto-providers
```

- `--keep` 參數會在您退出後保持沙箱運行，以便您稍後重新連接。 
- `--no-auto-providers` 參數會跳過提供者設定提示，因為本教學使用 `curl` 而不是 AI 代理程式。

你會降落在一個沙盒內的互動式 shell 中：

```bash
sandbox@demo:~$
```

## Try to Reach the GitHub API

如果沒有設定網路策略，所有出站連線都會被封鎖。您可以從沙箱內部發起一個簡單的 API 呼叫來測試這一點：

```bash
curl https://api.github.com/zen
```

`https://api.github.com/zen` 是一個輕量級的、無需身份驗證的 GitHub REST 端點，每次呼叫都會傳回一條隨機格言。它不需要任何身份令牌或參數，因此非常適合作為驗證出站 HTTPS 連接的冒煙測試目標。

請求失敗。預設情況下，所有出站網路流量均被拒絕。沙盒代理攔截了發送到 `api.github.com:443` 的 HTTPS CONNECT 請求，並拒絕了該請求，因為沒有任何網路策略授權 `curl` 存取該主機。

```bash
curl: (56) CONNECT tunnel failed, response 403
```

退出沙箱。 `--keep` 旗標會保持 sandbox 實例的運作：

```bash
exit
```

## Check the Deny Log

每次連線被拒絕都會產生一條結構化的日誌條目。請查詢主機上的沙箱日誌，以確認拒絕連線並查看拒絕原因。

```bash
openshell logs demo --since 5m
```

你會看到這樣一行資訊：

```bash
action=deny dst_host=api.github.com dst_port=443 binary=/usr/bin/curl deny_reason="no matching network policy"
```

每次連線被拒絕都會被記錄，包括目標位址、嘗試連線所使用的二進位檔案、拒絕原因。不會有任何秘密洩漏。

## Apply a Read-Only GitHub API Policy

若要允許沙箱存取 GitHub API，請定義一個授予唯讀存取權限的網路策略。此策略指定允許的主機、連接埠、二進位和 HTTP 方法。建立一個名為 `github-readonly.yaml` 的文件，並加入以下內容：

```yml
version: 1

filesystem_policy:
  include_workdir: true
  read_only: [/usr, /lib, /proc, /dev/urandom, /app, /etc, /var/log]
  read_write: [/sandbox, /tmp, /dev/null]
landlock:
  compatibility: best_effort
process:
  run_as_user: sandbox
  run_as_group: sandbox

network_policies:
  github_api:
    name: github-api-readonly
    endpoints:
      - host: api.github.com
        port: 443
        protocol: rest
        enforcement: enforce
        access: read-only
    binaries:
      - { path: /usr/bin/curl }
```

`filesystem_policy`, `landlock` 和 `process` 區塊保留了預設的沙箱設定。這是必需的，因為策略集會替換整個策略。 `network_policies` 區塊是關鍵：
- `curl` 可以透過 HTTPS 向 `api.github.com` 發送 GET, HEAD 和 OPTIONS 請求。
- 其他所有請求都會被拒絕。
  
Sandbox 的 proxy 程式會自動偵測 HTTPS 端點上的 TLS，並在必要時終止 TLS 連接，以便檢查每個 HTTP 請求，並在方法層級強制執行唯讀存取預設。

套用這個新的 policy：

```bash
openshell policy set demo --policy github-readonly.yaml --wait
```

`--wait` 會阻塞，直到沙箱確認新策略已載入。無需重啟。策略支援熱重載。

!!! tip
    本教學使用 `cur`l 和唯讀存取權限，以簡化操作。在為實際工作負載建立策略時：

    - 要將策略限定於某個代理，請將 `binaries` 部分替換為代理的二進位文件，例如 `/usr/local/bin/claude`，而不是 `curl`。
    - 若要授予寫入權限，請將 `access: read-only` 變更為 `read-write`，或為特定路徑新增明確規則。請參閱[策略架構參考](https://docs.nvidia.com/openshell/latest/reference/policy-schema.html)。
    - 若要允許其他端點，請在相同檔案中堆疊多個策略，例如針對 PyPI, npm 或內部 API 的策略。有關範例，請參閱[自訂沙箱策略](https://docs.nvidia.com/openshell/latest/sandboxes/policies.html)。

## Verify If GET Requests Are Allowed

該策略現已生效。請重新連接到沙箱並重試相同的請求，以確認讀取存取權限是否有效。

```bash
openshell sandbox connect demo
```

重試相同的請求：

```bash
curl -s https://api.github.com/zen
```

```bash
Anything added dilutes everything else.
```

它奏效了。唯讀策略預設允許 GET 請求通過。

## Try a Write

唯讀策略預設允許 GET 請求，但會阻止 POST, PUT 和 DELETE 等修改方法。請在沙盒環境中向 GitHub API 發送 POST 請求來測試這一點：

```bash
curl -s -X POST https://api.github.com/repos/octocat/hello-world/issues \
    -H "Content-Type: application/json" \
    -d '{"title":"oops"}'
```

```bash
{"error":"policy_denied","policy":"github-api-readonly","detail":"POST /repos/octocat/hello-world/issues not permitted by policy"}
```

CONNECT 請求成功，因為 `api.github.com` 已被允許訪問，但 L7 代理程式檢查了 HTTP 方法並傳回了 403 錯誤。 POST 方法不在唯讀預設策略中。具有此策略的 proxy 程式可以從 GitHub 讀取程式碼，但無法建立 issue、推送 commit 或修改任何內容。

退出沙箱：

```bash
exit
```

## Check the L7 Deny Log

L7 層拒絕請求與連線層拒絕請求的日誌記錄是分開的。日誌條目包含代理程式拒絕的確切 HTTP 方法和路徑。

```bash
openshell logs demo --level warn --since 5m
```

```bash
l7_decision=deny dst_host=api.github.com l7_action=POST l7_target=/repos/octocat/hello-world/issues l7_deny_reason="POST /repos/octocat/hello-world/issues not permitted by policy"
```

日誌會記錄確切的 HTTP 方法、路徑和拒絕原因。在生產環境中，將這些日誌傳輸到您的 SIEM 系統，即可獲得代理程式發出的每個請求的完整稽核追蹤。

!!! tip
    若要在不封鎖要求的情況下記錄違規行為，請在策略中將 `forcement: enforce` 設定為 `affective: audit` 而非 `forcement: enforce`。這對於迭代建置策略非常有用：先以稽核模式部署，查看日誌，確認規則正確後再切換到強制執行模式。

## Clean Up

刪除沙箱以釋放資源。這將停止所有進程並清除所有注入的憑證。

```bash
openshell sandbox delete demo
```

!!! tip
    若要以非互動方式運行整個過程，請使用自動演示腳本：
     
    ```bash
    bash examples/sandbox-policy-quickstart/demo.sh
    ```

