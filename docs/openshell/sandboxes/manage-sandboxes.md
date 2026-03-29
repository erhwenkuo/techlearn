# Manage Sandboxes

本頁面介紹如何建立和管理沙箱。有關沙箱的定義和隔離機制，請參閱 [關於沙箱](https://docs.nvidia.com/openshell/latest/sandboxes/index.html) 頁面。

!!! info
    在建立網關或沙箱之前，Docker 必須先運作。否則，CLI 將回傳連線被拒絕錯誤（`os error 61`），且不提供錯誤原因說明。請啟動 Docker 並重試。

## Create a Sandbox

使用一條指令即可建立沙箱。例如，要使用 Claude 建立沙箱，請執行：

```bash
openshell sandbox create -- claude
```

每個沙箱都需要網關。如果您執行 `openshell sandbox create` 命令時沒有指定網關，命令列介面 (CLI) 會自動構建一個本機網關。

## Remote Gateways

如果您打算在遠端主機或雲端託管網關上執行沙箱，請先設定網關。有關部署選項和多網關管理，請參閱 [部署和管理網關](https://docs.nvidia.com/openshell/latest/sandboxes/manage-gateways.html)。

## GPU Resources

若要請求 GPU 資源，請新增 `--gpu`：

```bash
openshell sandbox create --gpu -- claude
```

## Custom Containers

使用 `--from` 參數可以從預先建置的社群軟體包、本地目錄或容器映像建立沙箱：

```bash
openshell sandbox create --from openclaw
openshell sandbox create --from ./my-sandbox-dir
openshell sandbox create --from my-registry.example.com/my-image:latest
```

使用 `--from <FROM>` 的旗標時有幾個變形:

- **沙箱簡名**: Community 沙箱名稱（例如 `openclaw`）。社群名稱解析為 `ghcr.io/nvidia/openshell-community/sandboxes/<name>:latest`（使用 `OPENSHELL_COMMUNITY_REGISTRY` 覆蓋前綴）。
- **Dockerfile 或包含該沙箱的目錄路徑**: 如果提供 Dockerfile 或目錄，則在建立沙箱之前，容器鏡像會自動被建置並推送到叢集。
- **完整的容器映像**: 使用完整的容器鏡像名稱可切換到企業內部或是指定的容器 Registry 來載入沙箱容器。

CLI 會根據 OpenShell [社群目錄](https://github.com/NVIDIA/OpenShell-Community)解析社群名稱，拉取捆綁的 Dockerfile 和策略，在本地建立映像，並建立沙箱。有關完整目錄以及如何貢獻您自己的沙箱，請參閱 [Community Sandboxes](https://docs.nvidia.com/openshell/latest/sandboxes/community-sandboxes.html) 頁面。


## Connect to a Sandbox

開啟一個連接到正在運作的沙箱的 SSH 會話：

```bash
openshell sandbox connect my-sandbox
```

直接在沙盒工作區啟動 VS Code 或 Cursor：

```bash
openshell sandbox create --editor vscode --name my-sandbox

openshell sandbox connect my-sandbox --editor cursor
```

使用 `--editor` 時，OpenShell 會保持沙箱運行，並安裝一個由 OpenShell 管理的 SSH 包含文件，而不是用生成的 host 塊來污染你的主 `~/.ssh/config` 文件。


## Monitor and Debug

列出所有沙盒：

```bash
openshell sandbox list
```

取得特定沙盒的詳細資訊：

```bash
openshell sandbox get my-sandbox
```

串流沙箱日誌，以監控代理程式活動並診斷策略決策：

```bash
openshell logs my-sandbox
```

|旗標|用途|範例|
|---|----|---|
|`--tail`|即時日誌串流|`openshell logs my-sandbox --tail`|
|`--source`|按日誌來源過濾|`openshell logs my-sandbox --source sandbox`|
|`--level`|按嚴重性過濾|`openshell logs my-sandbox --level warn`|
|`--since`|顯示特定時間段內的日誌|`openshell logs my-sandbox --since 5m`|

OpenShell Terminal 將沙箱狀態和即時日誌整合到一個即時儀表板中：

```bash
openshell term
```

使用終端查找標記為 `action=deny` 的被阻止連線以及與推論相關的代理活動。如果連線意外被阻止，請將主機新增至網路原則。有關工作流程，請參閱「[自訂沙盒策略](https://docs.nvidia.com/openshell/latest/sandboxes/policies.html)」。

## Port Forwarding

將本機連接埠轉送到正在執行的沙箱，以存取其中的服務，例如 Web 伺服器或資料庫：

```bash
openshell forward start 8000 my-sandbox

openshell forward start 8000 my-sandbox -d    # run in background
```

列出並停止活躍的轉發：

```bash
openshell forward list

openshell forward stop 8000 my-sandbox
```

!!! tip
    您也可以在建立時使用 `--forward` 參數轉送連接埠：

    ```bash
    openshell sandbox create --forward 8000 -- claude
    ```

## SSH Config

為沙箱產生 SSH 設定條目，以便 VS Code Remote-SSH 等工具可以直接連接：

```bash
openshell sandbox ssh-config my-sandbox
```

將輸出附加到 `~/.ssh/config` 或使用 `--editor` 參數在 `sandbox create/sandbox connect` 中進行自動設定。

## Transfer Files

將主機上的檔案上傳到沙盒：

```bash
openshell sandbox upload my-sandbox ./src /sandbox/src
```

將檔案從沙箱下載到主機：

```bash
openshell sandbox download my-sandbox /sandbox/output ./local
```

!!! note
    您也可以使用 `openshell sandbox create` 指令的 `--upload` 旗標在建立時上傳檔案。

## Delete Sandboxes

刪除沙箱會停止所有程序、釋放資源並清除注入的憑證。

```bash
openshell sandbox delete my-sandbox
```

