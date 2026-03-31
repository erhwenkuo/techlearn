# Community Sandboxes

使用 OpenShell 社群目錄中預先建立的沙箱，或貢獻您自己的沙箱。

## What Are Community Sandboxes

社群沙箱是發佈在 [OpenShell 社群倉庫](https://github.com/NVIDIA/OpenShell-Community)中的即用型環境。每個沙箱都將 Dockerfile、策略、可選技能和啟動腳本打包到一個單獨的軟體包中，您只需一條命令即可啟動。

## Current Catalog

目錄中提供了以下社區沙盒。

|沙盒|描述|
|---|----|
|`base`|包含系統工具和開發環境的基礎鏡像|
|`ollama`|Ollama 支援雲端和本機模型，並預先安裝了 Claude Code, OpenCode 和 Codex。在沙箱環境中使用 `ollama launch` 指令即可啟動無需任何配置的編碼代理程式。|
|`openclaw`|開放代理操縱和控制|
|`sdg`|合成資料生成工作流程|

## Use a Community Sandbox

使用 `--from` 旗標按名稱啟動社區沙箱：

```bash
openshell sandbox create --from openclaw
```

當您使用 `--from` 參數指定社區沙箱名稱時，CLI 會執行下列操作：

1. 在 [OpenShell 社群倉庫](https://github.com/NVIDIA/OpenShell-Community)中解析名稱。
2. 拉取 Dockerfile、策略、技能以及所有啟動腳本。
3. 在本地建置容器鏡像。
4. 建立應用了捆綁配置的沙箱。
5. 最終你會得到一個運作中的沙箱，其鏡像、策略和工具都由社群軟體包預先配置好。

## Other Sources

`--from` 旗標也接受：

- **本機目錄路徑**：指向磁碟上包含 Dockerfile 和可選原則/技能的目錄：
  
    ```bash
    openshell sandbox create --from ./my-sandbox-dir
    ```

- **容器鏡像引用**：直接使用現有的容器鏡像：

    ```bash
    openshell sandbox create --from my-registry.example.com/my-image:latest
    ```

## Contribute a Community Sandbox

每個社區沙箱都是 OpenShell 社群倉庫 `sandboxes/` 目錄下的子目錄。沙箱目錄至少必須包含以下檔案：

- `Dockerfile` 它定義了容器鏡像。
- `README.md` 它描述了沙盒及其使用方法。

您也可以包含以下可選文件：

- `policy.yaml` 檔案定義了沙箱啟動時所應用的預設原則。
- `skills/` 檔案包含沙箱捆綁的代理技能定義。
- 啟動腳本是指 Dockerfile 或入口點呼叫的所有腳本。

要貢獻程式碼，請 fork 此倉庫，新增您的沙箱目錄，然後提交 pull request。提交指南請參閱倉庫的 [CONTRIBUTING.md](https://github.com/NVIDIA/OpenShell-Community/blob/main/CONTRIBUTING.md) 文件。

!!! info
    社區目錄旨在不斷擴充。如果您建立了一個支援特定工作流程（資料處理、模擬、程式碼審查或其他任何功能）的沙箱，請考慮將其貢獻出來，以便其他人也能使用。

