# Quickstart

本頁面只需兩個指令即可從零開始建立一個運作中的、策略強制執行的沙箱。

## Prerequisites

在開始之前，請確保您已具備以下條件之一：

- 您的電腦上有可執行 Docker Desktop。
- 您的電腦上有可執行 Docker Engine。

有關完整的系統需求列表，請參閱[支援矩陣](https://docs.nvidia.com/openshell/latest/reference/support-matrix.html)。

## Install the OpenShell CLI

運行安裝腳本：

```bash
curl -LsSf https://raw.githubusercontent.com/NVIDIA/OpenShell/main/install.sh | sh
```

如果您更喜歡 [`uv`](https://docs.astral.sh/uv/)：

```bash
uv tool install -U openshell
```

安裝 CLI 後，在終端機中執行 `openshell --help` 查看完整的 CLI 參考，包括所有命令和 flags。

!!! tip
    您也可以複製 [NVIDIA OpenShell GitHub 儲存庫](https://github.com/NVIDIA/OpenShell)，並使用 `/openshell-cli` 技能將 CLI 參考載入到您的代理程式中。

## Create Your First OpenShell Sandbox

建立一個沙箱並在其中啟動一個代理程式。選擇與您的代理程式對應的選項卡：

=== "Claude Code"
    執行以下命令，建立沙箱來使用 Claude Code:

    ```bash
    openshell sandbox create -- claude
    ```

    CLI 會提示您使用本機憑證建立提供者。輸入`yes` 繼續。如果您的環境中已設定 `ANTHROPIC_API_KEY`，CLI 會自動識別。否則，您可以在沙箱啟動後進行設定。

=== "OpenCode"
    執行以下命令，建立沙箱來使用 OpenCode:

    ```bash
    openshell sandbox create -- opencode
    ```

    CLI 會提示您使用本機憑證建立提供者。輸入 `yes` 繼續。如果您的環境中已設定 `OPENAI_API_KEY` 或 `OPENROUTER_API_KEY`，CLI 將自動識別。否則，您可以在沙箱啟動後進行設定。

=== "Codex"
    執行以下命令，建立沙箱來使用 Codex:

    ```bash
    openshell sandbox create -- codex
    ```

    CLI 會提示您使用本機憑證建立提供者。輸入 `yes` 繼續。如果您的環境中已設定 `OPENAI_API_KEY`，CLI 會自動識別。否則，您可以在沙箱啟動後進行設定。

=== "OpenClaw"
    執行以下命令，建立沙箱來使用 OpenClaw:

    ```bash
    openshell sandbox create --from openclaw
    ```

    `--from` 旗標會從 [OpenShell 社群目錄](https://github.com/NVIDIA/OpenShell-Community)中拉取預先建置的沙箱定義。每個定義都將容器鏡像、自訂策略和可選技能打包到一個單獨的套件中。

=== "Community Sandbox"
    使用 `--from` 旗標可以從 [NVIDIA 容器註冊表](https://registry.nvidia.com/)中拉取其他 OpenShell 沙箱鏡像。例如，要拉取基礎鏡像，請執行以下命令：

    ```bash
    openshell sandbox create --from base
    ```

## Deploy a Gateway (Optional)

執行 `openshell sandbox create` 指令而不指定 gateway 會自動啟動本機 gateway 實例服務。若要明確啟動 gatway 或部署到遠端主機，請選擇與您的設定相符的設定。

=== "Brev"

    !!! info
        在 [Brev](https://brev.nvidia.com) 上部署 OpenShell 網關，方法是按一下 [OpenShell Launchable](https://brev.nvidia.com/launchable/deploy/now?launchableID=env-3Ap3tL55zq4a8kew1AuW0FpSLsg) 上的「部署」按鈕。

    執行個體啟動執行後，在 Brev 控制台的「使用安全連結」下找到網關 URL。複製連接埠 8080 的共用 URL，該連接埠即為網關端點。

    ```bash
    openshell gateway add https://<your-port-8080-url>.brevlab.com

    openshell status
    ```

=== "DGX Spark"

    !!! info
        首先使用 NVIDIA Sync 設定 Spark，或確保已設定 SSH 存取（例如，已將 SSH 金鑰新增至主機）。

    透過 SSH 部署到 DGX Spark 伺服器：

    ```bash
    openshell gateway start --remote <username>@<spark-ssid>.local
    
    openshell status
    ```

    openshell 狀態顯示網關運作正常後，所有後續命令都將透過 SSH 隧道路由。

