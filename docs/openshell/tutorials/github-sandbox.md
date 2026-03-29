# Grant GitHub Push Access to a Sandboxed Agent

本教學將逐步介紹迭代式沙箱策略工作流程。首先，啟動一個沙箱，讓 Claude Code 將程式碼推送到 GitHub，然後觀察預設網路策略如何拒絕該請求。接下來，分別從您的機器和沙箱內部診斷拒絕原因，套用政策更新，並驗證該政策更新是否已在沙箱中生效。

完成本教程後，您將擁有：

- 一個運行著 **Claude Code** 的沙箱環境，可以向 GitHub 程式碼庫推送程式碼。
- 一個自訂網路策略，用於授予對特定程式碼庫的 GitHub 存取權限。
- 策略迭代工作流程的實務經驗：失敗、診斷、更新、驗證。

!!! info
    本教學展示了 Claude Code 中的一些範例提示和回應。您看到的具體措辭可能因會話而異。請將這些範例作為互動類型的參考，而非預期輸出。

## Prerequisites

本教學需要以下組件：

- 一個可正常運作的 OpenShell 安裝。請先完成快速入門指南。
- 一個具有倉庫權限的 GitHub 個人存取權杖 (PAT)。您可以在 GitHub 個人存取令牌設定頁面中產生該令牌，方法是選擇 **Generate new token (classic)** 並啟用 `repo scope`。
- 一個擁有 Claude Code 存取權限的 Anthropic 帳戶。 OpenShell 提供的是 sandbox runtime，而非 agent 程式。您必須使用自己的帳戶進行身份驗證。
- 一個您擁有的 GitHub 倉庫，用作推送目標。一個臨時倉庫即可。如有需要，您可以建立一個包含 README 檔案的臨時倉庫。

本教學使用兩個終端機來示範迭代策略工作流程：

- **Terminal 1**: 沙箱終端。您可以在此終端機中執行 `openshell sandbox create` 命令建立沙箱，並在其中與 Claude Code 進行互動。
- **Terminal 2**: 這是您機器上沙箱外部的終端。您可以使用此終端機透過 `openshell term` 查看沙箱日誌，並透過 `openshell policy set` 應用更新後的政策。

以下各部分的說明都會指明了應使用的那一個終端機。

## Set Up a Sandbox with Your GitHub Token

根據您是建立新沙盒還是使用現有沙盒，選擇相應的選項卡並按照說明操作。

=== "Starting a new sandbox"

    在 **Terminal 2** 中，使用 Claude Code 建立一個新的沙箱。系統會自動套用[預設策略](https://docs.nvidia.com/openshell/latest/reference/default-policy.html)，該策略允許對 GitHub 進行唯讀存取。

    建立一個憑證提供程序，會自動將您的 GitHub 令牌注入到沙箱中。此提供者會從您的主機環境中讀取 `GITHUB_TOKEN`，並將其設定為沙箱內的環境變數。

    ```bash
    export GITHUB_TOKEN=<your-token>

    openshell provider create --name my-github --type github --from-existing

    openshell sandbox create --provider my-github -- claude
    ```

    `openshell sandbox create` 指令會在 Claude Code 退出後繼續執行沙箱，這樣您就可以稍後套用原則更新而無需重新建立環境。如果您希望自動刪除沙箱，請新增 `--no-keep` 參數。

    Claude Code 會在沙盒環境中啟動，並列印一個身份驗證連結。請在瀏覽器中打開該鏈接，登入您的 Anthropic 帳戶，然後返回終端。出現提示時，請信任 `/sandbox` 工作區，以允許 Claude Code 讀取和寫入檔案。


=== "Using an existing sandbox"

    在 **Terminal 1** 中，連接到已運行的沙箱，並將您的 GitHub 令牌設定為環境變數：

    ```bash
    openshell sandbox connect <sandbox-name>

    export GITHUB_TOKEN=<your-token>
    ```

    若要尋找正在運行的沙箱的名稱，請在 **Terminal 2** 中執行 `openshell sandbox list`。

## Push Code to GitHub

在 **Terminal 1** 中，請 Claude Code 編寫一​​個簡單的腳本並將其推送到您的程式碼倉庫。將 `<org>` 替換為您的 GitHub 組織或使用者名，將 `<repo>` 替換為您的程式碼倉庫名稱。

!!! abstract "Prompt"

    ```bash
    Write a hello_world.py script and push it to https://github.com/<org>/<repo>.
    ```

Claude 辨識到需要 GitHub 憑證。它詢問您希望如何進行身份驗證。請將您的 GitHub 個人存取權令牌貼到對話中。 Claude 設定身份驗證並嘗試推送。

推送失敗。 Claude 報告了一個錯誤，但失敗並非身份驗證問題。預設的沙盒策略允許對 GitHub 進行唯讀存取並阻止寫入操作，因此代理程式在請求到達 GitHub 伺服器之前就被 proxy 拒絕了推送。

## Diagnose the Denial

在本節中，您將從您的機器和沙箱內部診斷拒絕存取問題。

### View the Logs from Your Machine

在 **Terminal 2** 中，啟動 OpenShell 終端機：

```bash
openshell term
```

儀錶板顯示沙箱狀態和策略決策的即時流。尋找 `l7_decision=deny` 的條目。選擇一個拒絕條目即可查看完整詳情：

```bash
l7_action:      PUT
l7_target:      /repos/<org>/<repo>/contents/hello_world.py
l7_decision:    deny
dst_host:       api.github.com
dst_port:       443
l7_protocol:    rest
policy:         github_rest_api
l7_deny_reason: PUT /repos/<org>/<repo>/contents/hello_world.py not permitted by policy
```

日誌顯示，沙箱 proxy 攔截了發往 `api.github.com` 的出站 PUT 請求並將其拒絕。 `github_rest_api` 策略允許 **讀取操作（GET）**，但阻止對 GitHub API 的 **寫入操作（PUT, POST, DELETE）**。如果 Claude 嘗試透過 HTTPS 向 `github.com` 推送 `git` 請求，也會出現類似的拒絕。

### Ask Claude Code to Check the Sandbox Logs

在 **Terminal 1** 中，請 Claude Code 檢查沙箱日誌中是否有被拒絕的請求：

!!! abstract "Prompt"

    ```bash
    Check the sandbox logs for any denied network requests. What is blocking the push?
    ```

Claude 讀取了拒絕條目並確定了根本原因。它解釋說，失敗原因是沙盒網路策略限制，而不是令牌權限問題。例如，以下是一個可能的回應：

!!! info "Response"
    沙箱運行一個代理，該代理會對出站流量強制執行策略。 `github_rest_api` 策略允許 GET 請求（用於讀取檔案），但阻止對 GitHub 的 PUT/寫入請求。這是沙箱層級的限制，並非令牌問題。無論您提供什麼令牌，透過 API 推送的內容都會被阻止，直到策略更新為止。

兩種觀點都證實了一點：proxy 伺服器運作正常。預設策略旨在限制推送。若要允許 GitHub 推送，您需要更新網路策略。

從 Claude 的回覆中複製拒絕原因。下一步，您需要將其貼到運行在您機器上的 proxy 程式中。

## Update the Policy from Your Machine

在 **Terminal 2** 中，將上一個步驟中的拒絕原因貼到您電腦上的程式碼代理程式（例如 Claude Code 或 Cursor）中，並要求其建議政策更新。拒絕原因為代理提供了產生正確策略規則所需的上下文資訊。貼上以下提示範例後，請正確提供您要推送程式碼的 GitHub repo 的組織名稱和 repo 名稱。

!!! abstract "Prompt"

    ```bash
    基於以下拒絕原因，建議更新沙箱策略，允許 GitHub 向 `https://github.com/<org>/<repo>` 推送程式碼，並將更新儲存到 `/tmp/sandbox-policy-update.yaml`：

    `filesystem_policy`, `landlock` 和 `process` 部分是靜態的。它們在沙箱創建時讀取一次，無法透過熱重載更改。此處包含這些部分是為了完整性，使文件自包含，但只有 `network_policies` 部分會在將此更新應用於正在運行的沙箱時生效。
    ```

以下步驟概述 claude code 應執行的流程：

1. 檢查拒絕原因。
2. 寫入更新後的策略，新增 github_git 和 github_api 程式碼區塊，授予對您的倉庫的寫入權限。
3. 將政策儲存到 `/tmp/sandbox-policy-update.yaml`。

## Review the Generated Policy

請參考以下策略範例，在應用策略之前將其與產生的策略進行比較。確認該策略僅授予您預期的存取權限。在本例中，該策略僅授予針對單一程式碼倉庫的 Git 推送操作和 GitHub REST API 存取權。

??? tip "Full reference policy"

    以下 YAML 檔案展示了一個完整的策略，它在預設策略的基礎上擴展了對單一 GitHub 倉庫的存取權。請將 `<org>` 替換為您的 GitHub 組織或使用者名，並將 `<repo>` 替換為您的倉庫名稱。

    `filesystem_policy`、`landlock` 和 `process` 部分是靜態的。它們在沙箱創建時讀取一次，並且無法透過熱重載進行更改。這裡包含這些部分是為了完整性，使文件自包含，但只有 `network_policies` 部分會在您將此原則套用至正在執行的沙箱時生效。

    ```yaml
    version: 1

    # ── Static (locked at sandbox creation) ──────────────────────────

    filesystem_policy:
    include_workdir: true
    read_only:
        - /usr
        - /lib
        - /proc
        - /dev/urandom
        - /app
        - /etc
        - /var/log
    read_write:
        - /sandbox
        - /tmp
        - /dev/null

    landlock:
    compatibility: best_effort

    process:
    run_as_user: sandbox
    run_as_group: sandbox

    # ── Dynamic (hot-reloadable) ─────────────────────────────────────

    network_policies:

    # Claude Code ↔ Anthropic API
    claude_code:
        name: claude-code
        endpoints:
        - { host: api.anthropic.com, port: 443, protocol: rest, enforcement: enforce, access: full }
        - { host: statsig.anthropic.com, port: 443 }
        - { host: sentry.io, port: 443 }
        - { host: raw.githubusercontent.com, port: 443 }
        - { host: platform.claude.com, port: 443 }
        binaries:
        - { path: /usr/local/bin/claude }
        - { path: /usr/bin/node }

    # NVIDIA inference endpoint
    nvidia_inference:
        name: nvidia-inference
        endpoints:
        - { host: integrate.api.nvidia.com, port: 443 }
        binaries:
        - { path: /usr/bin/curl }
        - { path: /bin/bash }
        - { path: /usr/local/bin/opencode }

    # ── GitHub: git operations (clone, fetch, push) ──────────────

    github_git:
        name: github-git
        endpoints:
        - host: github.com
            port: 443
            protocol: rest
            enforcement: enforce
            rules:
            - allow:
                method: GET
                path: "/<org>/<repo>.git/info/refs*"
            - allow:
                method: POST
                path: "/<org>/<repo>.git/git-upload-pack"
            - allow:
                method: POST
                path: "/<org>/<repo>.git/git-receive-pack"
        binaries:
        - { path: /usr/bin/git }

    # ── GitHub: REST API ─────────────────────────────────────────

    github_api:
        name: github-api
        endpoints:
        - host: api.github.com
            port: 443
            protocol: rest
            enforcement: enforce
            rules:
            # GraphQL API (used by gh CLI)
            - allow:
                method: POST
                path: "/graphql"
            # Full read-write access to the repository
            - allow:
                method: "*"
                path: "/repos/<org>/<repo>/**"
        binaries:
        - { path: /usr/local/bin/claude }
        - { path: /usr/local/bin/opencode }
        - { path: /usr/bin/gh }
        - { path: /usr/bin/curl }

    # ── Package managers ─────────────────────────────────────────

    pypi:
        name: pypi
        endpoints:
        - { host: pypi.org, port: 443 }
        - { host: files.pythonhosted.org, port: 443 }
        - { host: github.com, port: 443 }
        - { host: objects.githubusercontent.com, port: 443 }
        - { host: api.github.com, port: 443 }
        - { host: downloads.python.org, port: 443 }
        binaries:
        - { path: /sandbox/.venv/bin/python }
        - { path: /sandbox/.venv/bin/python3 }
        - { path: /sandbox/.venv/bin/pip }
        - { path: "/sandbox/.uv/python/**/python*" }
        - { path: /usr/local/bin/uv }
        - { path: "/sandbox/.uv/python/**" }

    # ── VS Code Remote ──────────────────────────────────────────

    vscode:
        name: vscode
        endpoints:
        - { host: update.code.visualstudio.com, port: 443 }
        - { host: "*.vo.msecnd.net", port: 443 }
        - { host: vscode.download.prss.microsoft.com, port: 443 }
        - { host: marketplace.visualstudio.com, port: 443 }
        - { host: "*.gallerycdn.vsassets.io", port: 443 }
        binaries:
        - { path: /usr/bin/curl }
        - { path: /usr/bin/wget }
        - { path: "/sandbox/.vscode-server/**" }
        - { path: "/sandbox/.vscode-remote-containers/**" }
    ```

    下表總結了兩個 GitHub 特有的模組：

    |Block|Endpoint|Behavior|
    |-----|--------|--------|
    |`github_git`|`github.com:443`|Git 智慧 HTTP 協定。代理程式會自動偵測並終止 TLS 連線以檢查請求。允許對指定倉庫執行 `info/refs`（複製/取得）、`git-upload-pack`（取得資料）和 `git-receive-pack`（推播）操作。拒絕對未列出的倉庫執行所有操作。|
    |`github_api`|`api.github.com:443`|REST API。代理程式會自動偵測並終止 TLS 連線以檢查請求。允許對指定儲存庫使用所有 HTTP 方法和 GraphQL 查詢。拒絕存取未列出的儲存庫。|

    其餘程式碼區塊（claude_code, nvidia_inference, pypi, vscode）與預設策略相同。預設策略中的 github_ssh_over_https 和 github_rest_api 程式碼區塊被替換為上述的 github_git 和 github_api 程式碼區塊，這些程式碼區塊會授予指定倉庫的寫入權限。 GitHub 作業之外的沙箱行為保持不變。


    有關策略區塊結構的詳細信息，請參閱 [策略](https://docs.nvidia.com/openshell/latest/sandboxes/policies.html) 部分。

## Apply the Policy

審核產生的策略後，在 **Terminal 2** 將其應用到正在運行的沙箱環境中：

```bash
openshell policy set <sandbox-name> --policy /tmp/sandbox-policy-update.yaml --wait
```

網路策略支援熱重載。 `--wait` 標誌會阻塞操作，直到策略引擎確認新版本已加載，更新會立即生效，無需重新啟動沙箱或重新連接 Claude Code。

## Retry the Push

在 **Terminal 1** 中，請 Claude Code 重試推送：

!!! note "Prompt"

    ```bash
    The sandbox policy has been updated. Try pushing to the repository again.
    ```

推送操作已成功完成。 openshell 終端儀表板現在顯示了 `api.github.com` 和 `github.com` 的 `l7_decision=allow` 條目，而先前顯示的是拒絕條目。

## Clean Up

完成後，刪除沙箱以釋放叢集資源：

```bash
openshell sandbox delete <sandbox-name>
```
