# Deep Agents CLI

用於建立 Deep Agents 的互動式命令列介面。

一個用於建立具有持久記憶體的 agent 的終端介面。agent 能夠在 session 之間保持上下文，學習專案約定，並在批准控制下執行程式碼。

Deep Agents CLI 有以下內建功能：

 - :material-file-outline: **File operations** - 使用工具讀取、寫入和編輯專案中的文件，使代理程式能夠管理和修改程式碼和文件。
 - :octicons-command-palette-16: **Shell command execution** - 執行 shell 命令來執行測試、建置專案、管理依賴項以及與版本控制系統互動。
 - :octicons-search-16: **Web search** - 在網路上搜尋最新資訊和文件（需要 Tavily API 金鑰）。
 - :material-web: **HTTP requests** - 向 API 和外部服務發出 HTTP 請求，以進行資料擷取和整合任務。
 - :octicons-tasklist-16: **Task planning and tracking** - 將複雜任務分解成離散步驟，並透過內建的待辦事項系統追蹤進度。
 - :material-memory: **Memory storage and retrieval** - 跨會話儲存和檢索訊息，使代理能夠記住專案約定和學習模式。
 - :octicons-people-16: **Human-in-the-loop** - 敏感工具操作需要人工核准。

!!! tip
    [觀看演示視頻](https://youtu.be/IrnacLa9PJc?si=3yUnPbxnm2yaqVQb)，了解 Deep Agents CLI 的工作原理。

## Quick start

:octicons-key-16: **Set your API key**

設定環境變數:

```bash
export ANTHROPIC_API_KEY="your-api-key"
```

或在專案根目錄下建立一個 `.env` 檔案：

```ini
ANTHROPIC_API_KEY=your-api-key
```

:octicons-terminal-16: **Run the CLI**

```bash
uvx deepagents-cli
```

:material-code-tags-check: **Give the agent a task**

```bash
> Create a Python script that prints "Hello, World!"
```

Agent 會在修改文件之前提出帶有差異的更改建議，供您批准。

!!! tip "Additional installation and configuration options"

    如有需要，請本地安裝:

    === "pip"

        ```bash
        pip install deepagents-cli
        ```

    === "uv"

        ```bash
        uv add deepagents-cli
        ```

    CLI 預設使用 Anthropic Claude Sonnet 4。要使用 OpenAI:

    ```bash
    export OPENAI_API_KEY="your-key"
    ```

    啟用網路搜尋 (optional):

    ```bash
    export TAVILY_API_KEY="your-key"
    ```

    API 金鑰可以設定為環境變數或 `.env` 檔案。


## Configuration

:fontawesome-regular-flag: **Command-line options**

| Option | Description |
| -------| ----------- |
| `--agent NAME` | 使用有獨立記憶體的 named agent |
| `--auto-approve` | 跳過工具確認提示（按 `Ctrl+T` 切換）|
| `--sandbox TYPE` | 在遠端沙箱中執行：`modal`, `daytona` 或 `runloop` |
| `--sandbox-id ID` | 重複使用現有沙箱 |
| `--sandbox-setup PATH` | 在沙箱中執行安裝腳本 |

:octicons-terminal-16: **CLI commands**

| Option | Description |
| -------| ----------- |
| `deepagents list` | 列出所有 agent |
| `deepagents help` | 顯示 help |
| `deepagents reset --agent NAME` | 清除 agent memory並重置為預設設置 |
| `deepagents reset --agent NAME --target SOURCE` | 從另一個 agent 複製 memory |

## Interactive mode

:material-slash-forward: **Slash commands**

在 CLI 會話​​中使用以下命令：

- `/tokens` - 顯示 token 使用狀況
- `/clear` - 清除通話記錄
- `/exit` - 退出 CLI

:octicons-terminal-16: **Bash commands**

透過在指令前加上 `!` 來直接執行 shell 指令：

```bash
!git status
!npm test
!ls -la
```

:material-keyboard-outline: **Keyboard shortcuts**

| Option | Description |
| -------| ----------- |
| `Enter` | Submit |
| `Alt+Enter` | Newline |
| `Ctrl+E` | External editor |
| `Ctrl+T` | Toggle auto-approve |
| `Ctrl+C` | Interrupt |
| `Ctrl+D` | Exit |

## Set project conventions with memories

Agents 使用 memory-first 協議，將資訊以 markdown 檔案的形式儲存在 `~/.deepagents/AGENT_NAME/memories/` 目錄中：

- **Research**: 在開始任務之前，搜尋記憶體以獲取相關上下文。
- **Response**: 執行過程中如果不確定該如何進行，則檢查記憶體。
- **Learning**: 自動儲存新資訊以供日後會話使用

按主題整理 memories，並使用描述性檔案名稱：

```bash
~/.deepagents/backend-dev/memories/
├── api-conventions.md
├── database-schema.md
└── deployment-process.md
```

只需一次性設定 agent 運行慣例:

```bash
uvx deepagents-cli --agent backend-dev
> Our API uses snake_case and includes created_at/updated_at timestamps
```

它會記住這些資訊以供後續會話使用：

```bash
> Create a /users endpoint
# Applies conventions without prompting
```

## Use remote sandboxes

為了安全性和靈活性，可以在隔離的遠端環境中執行程式碼。遠程沙箱具有以下優勢：

- **Safety**: 保護您的本機電腦免受潛在有害程式碼執行的影響
- **Clean environments**: 使用特定的依賴項或作業系統配置，無需本機設定
- **Parallel execution**: 在隔離環境中同時運行多個代理
- **Long-running tasks**: 執行耗時操作，而不會阻塞您的機器。
- **Reproducibility**: 確保各團隊執行環境的一致性

若要使用遠端沙箱，請依照下列步驟操作：

1. 配置您的沙盒提供者（[Runloop](https://www.runloop.ai/), [Daytona](https://www.daytona.io/) 或 [Modal](https://modal.com/)）：

    ```bash
    # Runloop
    export RUNLOOP_API_KEY="your-key"

    # Daytona
    export DAYTONA_API_KEY="your-key"

    # Modal
    modal setup
    ```

2. 在沙箱環境下執行 CLI:

    ```bash
    uvx deepagents-cli --sandbox runloop --sandbox-setup ./setup.sh
    ```

    agent 程式在本地運行，但所有程式碼操作都在遠端沙箱中執行。可選的設定腳本可以配置環境變數、克隆程式碼庫並準備依賴項。


3. 建立 `setup.sh` 檔案來配置沙箱環境 (Optional):

    ```bash
    #!/bin/bash
    set -e

    # Clone repository using GitHub token
    git clone https://x-access-token:${GITHUB_TOKEN}@github.com/username/repo.git $HOME/workspace
    cd $HOME/workspace

    # Make environment variables persistent
    cat >> ~/.bashrc <<'EOF'
    export GITHUB_TOKEN="${GITHUB_TOKEN}"
    export OPENAI_API_KEY="${OPENAI_API_KEY}"
    cd $HOME/workspace
    EOF

    source ~/.bashrc
    ```

    將金鑰儲存在本機 `.env` 檔案中，以供安裝腳本存取。


