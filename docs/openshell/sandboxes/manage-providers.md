# Manage Providers and Credentials

AI 代理通常需要憑證才能存取外部服務：例如，AI 模型提供者的 API 金鑰, GitHub 或 GitLab 的令牌等等。 OpenShell 將這些憑證作為名為「提供者 (Provider)」的 first-class 實體進行管理。

建立和管理向沙箱提供憑證的提供者。

## Create a Provider

可以透過本機環境變數或明確憑證值建立提供者。

### From Local Credentials

建立提供者最快的方法是讓 CLI 從 shell 環境中發現憑證：

```bash
openshell provider create --name my-claude --type claude --from-existing
```

此操作會從目前環境中讀取 `ANTHROPIC_API_KEY` 或 `CLAUDE_API_KEY`，並將其儲存在提供者中。

### With Explicit Credentials

直接提供憑證值：

```bash
openshell provider create --name my-api --type generic --credential API_KEY=sk-abc123
```

### Bare Key Form

傳遞一個不帶值的鍵名，即可從該鍵名的環境變數讀取值：

```bash
openshell provider create --name my-api --type generic --credential API_KEY
```

這段程式碼會在你的 shell 中尋找 `$API_KEY` 的目前值並將其儲存起來。

## Manage Providers

列出, 檢查, 更新和刪除活動群集中的提供者。

列出所有提供者：

```bash
openshell provider list
```

檢查服務提供者：

```bash
openshell provider get my-claude
```

更新服務提供者的憑證：

```bash
openshell provider update my-claude --type claude --from-existing
```

刪除服務提供者：

```bash
openshell provider delete my-claude
```

## Attach Providers to Sandboxes

建立沙箱時，可以傳遞一個或多個 `--provider` 旗標：

```bash
openshell sandbox create --provider my-claude --provider my-github -- claude
```

每個 `--provider` 旗標都會附加一個提供者。沙箱會在運作時接收所有已附加提供者的全部憑證。

!!! warning
    無法在正在運行的沙箱中新增提供者。如果需要新增其他提供程序，請刪除沙箱並重新創建，同時指定所有必要的提供程序。

### Auto-Discovery Shortcut

當 `openshell sandbox create` 命令的末尾命令是可識別的工具名稱（例如 `claude`, `codex` 或 `opencode`）時，如果所需的提供者尚不存在，CLI 會自動使用您的本機憑證建立該提供者。您無需單獨建立提供者：

```bash
openshell sandbox create -- claude
```

這會將 Claude 偵測為已知工具，找到您的 `ANTHROPIC_API_KEY`，建立一個提供程序，將其附加到沙箱，並啟動 Claude Code。

## Supported Provider Types

支援以下幾種提供者類型。

|類型|注入的環境變數|典型用途|
|---|------------|-------|
|`claude`|`ANTHROPIC_API_KEY`, `CLAUDE_API_KEY`|Claude Code, Anthropic API|
|`codex`|`OPENAI_API_KEY`|OpenAI Codex|
|`generic`|User-defined|任何具有自訂憑證的服務|
|`github`|`GITHUB_TOKEN`, `GH_TOKEN`|GitHub API, `gh` CLI — 請參閱「[授予沙盒代理 GitHub 推送存取權限](https://docs.nvidia.com/openshell/latest/tutorials/github-sandbox.html)」。|
|`gitlab`|`GITLAB_TOKEN`, `GLAB_TOKEN`, `CI_JOB_TOKEN`|GitLab API, `glab` CLI|
|`nvidia`|`NVIDIA_API_KEY`|NVIDIA API Catalog|
|`openai`|`OPENAI_API_KEY`|任何相容 OpenAI 的端點。設定 `--config OPENAI_BASE_URL` 以指向提供者。請參閱 [配置推理路由](https://docs.nvidia.com/openshell/latest/inference/configure.html)。|
|`opencode`|`OPENCODE_API_KEY`, `OPENROUTER_API_KEY`, `OPENAI_API_KEY`|opencode tool|

!!! tip
    對於未在上述清單中列出的任何服務，請使用 `generic` 類型。您可以使用 `--credential` 參數自行定義環境變數名稱和值。

## Supported Inference Providers

以下提供者已通過 `inference.local` 測試。任何提供 OpenAI 相容 API 的供應商均可與 `openai` 類型搭配使用。請將 `--config OPENAI_BASE_URL` 設定為提供者的基本 URL，並將 `--credential OPENAI_API_KEY` 設定為您的 API 金鑰。

|提供者|名稱|類型|Base URL|API 金鑰變數|
|-----|---|----|--------|----------|
|NVIDIA API Catalog|`nvidia-prod`|`nvidia`|`https://integrate.api.nvidia.com/v1`|`NVIDIA_API_KEY`|
|Anthropic|`anthropic-prod`|`anthropic`|`https://api.anthropic.com`|`ANTHROPIC_API_KEY`|
|Baseten|`baseten`|`openai`|`https://inference.baseten.co/v1`|`OPENAI_API_KEY`|
|Bitdeer AI|`bitdeer`|`openai`|`https://api-inference.bitdeer.ai/v1`|`OPENAI_API_KEY`|
|Deepinfra|`deepinfra`|`openai`|`https://api.deepinfra.com/v1/openai`|`OPENAI_API_KEY`|
|Groq|`groq`|`openai`|`https://api.groq.com/openai/v1`|`OPENAI_API_KEY`|
|Ollama(local)|`ollama`|`openai`|`http://host.openshell.internal:11434/v1`|`OPENAI_API_KEY`|
|LM Studio (local)|`lmstudio`|`openai`|`http://host.openshell.internal:1234/v1`|`OPENAI_API_KEY`|

請參閱服務提供者的文檔，以取得正確的基本 URL、可用模型和 API 金鑰設定。若要設定模型推論路由，請參閱「[設定推論路由](https://docs.nvidia.com/openshell/latest/inference/configure.html)」部分。