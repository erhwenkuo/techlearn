# LangGraph CLI

**LangGraph CLI** 是一個命令列工具，用於在本地建置和運行 [Agent Server](https://docs.langchain.com/langsmith/agent-server)。產生的 Agent Server 會公開所有用於 runs、threads、assistants 等的 API 端點，並包含一些支援服務，例如用於檢查點和儲存的託管資料庫。

## Installation

1. 確保已安裝 Docker (e.g., `docker --version`)
2. 安裝 CLI
   
    === "Python (pip)"
        ```bash
        pip install langgraph-cli
        ```

    === "JavaScript"
        ```bash
        # Use latest on demand
        npx @langchain/langgraph-cli

        # Or install globally (available as `langgraphjs`)
        npm install -g @langchain/langgraph-cli
        ```

3. 驗證安裝

    === "Python (pip)"
        ```bash
        langgraph --help
        ```

        !!! info
            ```bash
            Usage: langgraph [OPTIONS] COMMAND [ARGS]...

            Options:
            --version  Show the version and exit.
            --help     Show this message and exit.

            Commands:
            build       📦 Build LangGraph API server Docker image.
            dev         🏃‍♀️‍➡️ Run LangGraph API server in development mode with...
            dockerfile  🐳 Generate a Dockerfile for the LangGraph API server, with...
            new         🌱 Create a new LangGraph project from a template.
            up          🚀 Launch LangGraph API server
            ```


    === "JavaScript"
        ```bash
        npx @langchain/langgraph-cli --help
        ```
    
### Quick commands

|Command	|What it does|
|-----------|------------|
|`langgraph dev`	|啟動一個輕量級的本機開發伺服器（無需 Docker），非常適合快速測試。|
|`langgraph build`	|建置 LangGraph API 伺服器的 Docker 映像以進行部署。|
|`langgraph dockerfile`	|產生一個基於你的配置的 Dockerfile，用於自訂建置。|
|`langgraph up`	|在本機 Docker 容器中啟動 LangGraph API 伺服器。需要 Docker 運作；本機開發環境需要 LangSmith API 金鑰；生產環境需要授權。|

對於 JS，請使用 `npx @langchain/langgraph-cli <command>`（如果 globally 安裝了 langgraphjs，則可以使用 langgraphjs）。


## Configuration file

要建置和運行有效的應用程序，LangGraph CLI 需要一個符合以下 [schema](https://raw.githubusercontent.com/langchain-ai/langgraph/refs/heads/main/libs/cli/schemas/schema.json) 的 JSON 設定檔。它包含以下屬性：

!!! info
    LangGraph CLI 預設使用目前目錄中名為 `langgraph.json` 的設定檔。

=== "Python"

    | Key |Description|
    |-----|-----------|
    |`dependencies`| **Required**. LangSmith API 伺服器的依賴項列垃。依賴項可以是以下幾種： <ul><li>一個句點（"."），它將查找本地 Python 套件。</li><li>`pyproject.toml`、`setup.py` 或 `requirements.txt` 所在的目錄路徑。<br />Example: 如果 `requirements.txt` 位於專案目錄的根目錄，則指定 `"./"`。如果它位於名為 `local_package` 的子目錄中，則指定 `"./local_package"`。設定上不要指定設定檔名字串 `"requirements.txt"` 本身。</li><li>Python 套件名。</li></ul>|
    |`graphs`         | **Required**. 將 graph ID 對應到已編譯 graph 或產生 graph 的函數所在的路徑。 Example: <ul><li>`./your_package/your_file.py:variable`，其中 `variable` 是 `langgraph.graph.state.CompiledStateGraph` 的一個實例。</li><li>`./your_package/your_file.py:make_graph`，其中 `make_graph` 是一個函數，它接受一個配置字典（`langchain_core.runnables.RunnableConfig`），並傳回一個 `langgraph.graph.state.StateGraph` 或 `langgraph.graph.state.CompiledStateGraph` 的實例。 查看 [how to rebuild a graph at runtime](https://docs.langchain.com/langsmith/graph-rebuild) 了解更多詳情。</li></ul>|
    |`auth`             | *(Added in v0.0.11)* 身份驗證配置，包含身份驗證處理程序的路徑。Example: `./your_package/auth.py:auth`，其中 `auth` 是 `langgraph_sdk.Auth` 查看 [authentication guide](https://docs.langchain.com/langsmith/auth) 了解更多詳情。|
    |`base_image`       | Optional. 用於 LangGraph API 伺服器的基礎鏡像。預設為 `langchain/langgraph-api` 或 `langchain/langgraphjs-api`。使用此選項可將建置版本鎖定至特定版本的 LangGraph API，例如 `"langchain/langgraph-server:0.2"`。 查看 [https://hub.docker.com/r/langchain/langgraph-server/tags](https://hub.docker.com/r/langchain/langgraph-server/tags) 了解更多詳情。(added in `langgraph-cli==0.2.8`)|
    |`image_distro`     | Optional. 基礎鏡像的 Linux 發行版。必須是 `"debian"`、`"wolfi"`、`"bookworm"` 或 `"bullseye"` 之一。如果省略，則預設為 `"debian"`。適用於 `langgraph-cli>=0.2.11`。|
    |`env`              | `.env` 檔案的路徑或環境變數與其值的對應。 |
    |`store`            | 用於在 BaseStore 中新增語義搜尋和/或生存時間 (TTL) 的配置。包含以下欄位: <ul><li>`index` (optional): 用於語義搜尋索引的配置，包含欄位 `embed`、`dims` 和可選的 `fields`。</li><li>`ttl` (optional): 用於配置專案過期時間。一個包含可選欄位的物件：`refresh_on_read`（布林值，預設為`true`）、`default_ttl`（浮點數，生存時間，單位為 **分鐘**；僅應用於新建立的項目；現有項目保持不變；預設為無過期時間）和`sweep_interval_minutes`（預設值，檢查過期清理項目的頻率，預設為清除項目的頻率，預設值，檢查過期清理項目。</li></ul> |
    |`ui`               | Optional. 由 agent 程式發出的 UI 元件的命名定義，每個定義都指向一個 JS/TS 檔案。(added in `langgraph-cli==0.1.84`)|
    |`python_version`   | `3.11`, `3.12`, 或 `3.13`. 預設為 `3.11`。|
    |`node_version`     | 指定 `node_version: 20` 以使用 LangGraph.js。|
    |`pip_config_file`  | `pip` 設定檔的路徑。|
    |`pip_installer`    | *(Added in v0.3)* Optional. Python 套件安裝程式選擇器。它可以設定為 `"auto"`、`"pip"` 或 `"uv"`。從 0.3 版本開始，預設策略是運行 `uv pip`，這通常可以加快建置速度，同時保持無縫替換。在極少數情況下，如果 `uv` 無法處理您的依賴關係或 `pyproject.toml` 檔案的結構，請在此處指定 `"pip"` 以恢復到先前的行為。|
    |`keep_pkg_tools`   | *(Added in v0.3.4)* Optional. 控制是否在最終映像中保留 Python 打包工具（`pip`、`setuptools`、`wheel`）。可接受的值: <ul><li><code>true</code> : 保留所有三個工具（跳過卸載）。</li><li><code>false</code> / omitted : 卸載所有三個工具（預設行為）。</li><li><code>list\[str]</code> : 要保留的工具名稱。每個值必須是 `pip`, `setuptools` 或 `wheel` 之一。預設情況下，這三個工具都會被卸載。|
    |`dockerfile_lines` | 從父映像匯入後，要新增至 Dockerfile 中的附加行列表。 |
    |`checkpointer`     | Checkpointer 配置。支持：<ul><li>`ttl` (optional): 具有 `strategy`、`sweep_interval_minutes`、`default_ttl` 等參數的 Object，用於控制檢查點過期時間。</li><li>`serde` (optional, 0.5+): 包含 `allowed_json_modules` 和 `pickle_fallback` 的 Object，用於調整反序列化行為。</li></ul>|
    |`http`             | HTTP 伺服器配置，包含以下欄位: <ul><li>`app`: 客制化 Starlette/FastAPI app 的路徑 (e.g., `"./src/agent/webapp.py:app"`)。查看 [custom routes guide](https://docs.langchain.com/langsmith/custom-routes).</li><li>`cors`: CORS 配置及其欄位比如 `allow_origins`, `allow_methods`, `allow_headers`, `allow_credentials`, `allow_origin_regex`, `expose_headers`, 與 `max_age`.</li><li>`configurable_headers`: 透過 `includes` / `excludes` 模式定義要公開為可設定值的請求標頭。</li><li>`logging_headers`: `configurable_headers` 的鏡像，用於從日誌中排除敏感標頭。</li><li>`middleware_order`: 選擇自訂中間件和身份驗證如何互動。 `auth_first` 會在自訂中間件運作之前執行驗證鉤子，而 `middleware_first`（預設值）會先執行您的中間件。</li><li>`enable_custom_route_auth`: 對透過 `app` 新增的路由套用驗證檢查。</li><li>`disable_assistants`, `disable_mcp`, `disable_a2a`, `disable_meta`, `disable_runs`, `disable_store`, `disable_threads`, `disable_ui`, `disable_webhooks`: 禁用內置路由或鉤子。</li><li>`mount_prefix`: 已掛載路由的前綴（例如，"/my-deployment/api“）。</li></ul> |
    |`api_version`      | *(Added in v0.3.7)* 要使用哪個語意版本的 LangGraph API 伺服器 (e.g., `"0.3"`)。 預設使用最新版本。請查看伺服器[變更日誌](https://docs.langchain.com/langsmith/agent-server-changelog)以了解每個版本的詳細資訊。|


=== "JS"

    | Key | Description |
    |-----|-------------|
    | `graphs`           | **Required**. 將 graph ID 對應到已編譯 graph 或產生 graph 的函數所在的路徑。 Example: <ul><li>`./src/graph.ts:variable`，其中 variable 是 `CompiledStateGraph` 的一個實例。</li><li>`./src/graph.ts:makeGraph`，其中 `makeGraph` 是一個函數，它接受一個設定字典（`LangGraphRunnableConfig`），並傳回 `StateGraph` 或 `CompiledStateGraph` 的實例。</li></ul>|
    | `env`              | `.env` 檔案的路徑或環境變數與其值的對應。|
    | `store`            | 用於在 BaseStore 中新增語義搜尋和/或生存時間 (TTL) 的配置。包含以下欄位:<ul><li>`index` (optional): 用於語義搜尋索引的配置，包含欄位 `embed`、`dims` 和可選的 `fields`。</li><li>`ttl` (optional): 用於配置專案過期時間。一個包含可選欄位的物件：`refresh_on_read`（布林值，預設為`true`）、`default_ttl`（浮點數，生存時間，單位為 **分鐘**；僅應用於新建立的項目；現有項目保持不變；預設為無過期時間）和`sweep_interval_minutes`（預設值，檢查過期清理項目的頻率，預設為清除項目的頻率，預設值，檢查過期清理項目。</li></ul> |
    | `node_version`     | 指定 `node_version: 20` 以使用 LangGraph.js。 |
    | `dockerfile_lines` | 從父映像匯入後，要新增至 Dockerfile 中的附加行列表。|
    | `checkpointer`     | 檢查點配置。支持: <ul><li>`ttl` (optional): 具有 `strategy`、`sweep_interval_minutes`、`default_ttl` 等參數的 Object，用於控制檢查點過期時間。</li><li>`serde` (optional, 0.5+): 包含 `allowed_json_modules` 和 `pickle_fallback` 的 Object，用於調整反序列化行為。</li></ul>|
    | `http`             | HTTP 伺服器配置與 Python 選項相對應: <ul><li>`cors` 包含了 `allow_origins`, `allow_methods`, `allow_headers`, `allow_credentials`, `allow_origin_regex`, `expose_headers`, `max_age`.</li><li>`configurable_headers` 與 `logging_headers` pattern 列表。</li><li>`middleware_order` (`auth_first` or `middleware_first`)。</li><li>`enable_custom_route_auth` 此外，還包含與上述相同的布林路由切換。</li></ul> |
    | `api_version`      | *(Added in v0.3.7)* 要使用哪個語意版本的 LangGraph API 伺服器 (e.g., `"0.3"`)。 efaults 更新至最新版本。請查看伺服器[變更日誌](https://docs.langchain.com/langsmith/agent-server-changelog)以了解每個版本的詳細資訊。 |

### Examples

=== "Python"

    **Basic configuration**

    ```json
    {
        "$schema": "https://langgra.ph/schema.json",
        "dependencies": ["."],
        "graphs": {
            "chat": "chat.graph:graph"
        }
    }
    ```


    **Using Wolfi base images**

    您可以使用 `image_distro` 欄位指定基礎鏡像的 Linux 發行版。有效選項包括 `debian`, `wolfi`, `bookwor` 和 `bullseye`。建議使用 Wolfi，因為它提供的鏡像體積更小、安全性更高。此功能在 langgraph-cli >= 0.2.11 版本中可用。

    ```json
    {
        "$schema": "https://langgra.ph/schema.json",
        "dependencies": ["."],
        "graphs": {
            "chat": "chat.graph:graph"
        },
        "image_distro": "wolfi"
    }
    ```

    **Adding semantic search to the store**

    所有部署都包含一個基於資料庫的 `BaseStore`。在 `langgraph.json` 檔案中新增 `index`配置即可在部署的 `BaseStore` 中啟用語意搜尋。

    `index.fields` 配置決定了要嵌入文件的哪些部分:
      - 如果省略或設定為 `["$"]`，則會嵌入整個文件。
      - 若要嵌入特定字段，請使用 JSON 路徑表示法：`["metadata.title", "content.text"]`
      - 缺少指定欄位的文件仍會被存儲，但不會嵌入這些欄位。
      - 您仍然可以在建立文件時使用 `index` 參數來覆蓋要嵌入的欄位。

    ```json
    {
        "dependencies": ["."],
        "graphs": {
            "memory_agent": "./agent/graph.py:graph"
        },
        "store": {
            "index": {
                "embed": "openai:text-embedding-3-small",
                "dims": 1536,
                "fields": ["$"]
            }
        }
    }
    ```

    !!! info "Common model dimensions"
        - openai:text-embedding-3-large: 3072
        - openai:text-embedding-3-small: 1536
        - openai:text-embedding-ada-002: 1536
        - cohere:embed-english-v3.0: 1024
        - cohere:embed-english-light-v3.0: 384
        - cohere:embed-multilingual-v3.0: 1024
        - cohere:embed-multilingual-light-v3.0: 384

    **Semantic search with a custom embedding function**

    如果要將語義搜尋與自訂嵌入函數一起使用，可以將自訂嵌入函數的路徑傳遞給它:

    ```json
    {
        "dependencies": ["."],
        "graphs": {
            "memory_agent": "./agent/graph.py:graph"
        },
        "store": {
            "index": {
                "embed": "./embeddings.py:embed_texts",
                "dims": 768,
                "fields": ["text", "summary"]
            }
        }
    }
    ```

    Store 配置中的 embed 欄位可以引用自訂函數，該函數接受字串列表並傳回一個嵌入列表。範例實作：

    ```python
    # embeddings.py
    def embed_texts(texts: list[str]) -> list[list[float]]:
        """Custom embedding function for semantic search."""
        # Implementation using your preferred embedding model
        return [[0.1, 0.2, ...] for _ in texts]  # dims-dimensional vectors
    ```

    **Adding custom authentication**

    ```json
    {
        "$schema": "https://langgra.ph/schema.json",
        "dependencies": ["."],
        "graphs": {
            "chat": "chat.graph:graph"
        },
        "auth": {
            "path": "./auth.py:auth",
            "openapi": {
                "securitySchemes": {
                    "apiKeyAuth": {
                        "type": "apiKey",
                        "in": "header",
                        "name": "X-API-Key"
                    }
                },
                "security": [{ "apiKeyAuth": [] }]
            },
            "disable_studio_auth": false
        }
    }
    ```

    有關詳細信息，請參閱[身份驗證概念指南](https://docs.langchain.com/langsmith/auth)；有關流程的實際操作步驟，請參閱[設定自訂身份驗證指南](https://docs.langchain.com/langsmith/set-up-custom-auth)。

    **Configuring store item Time-to-Live**

    您可以使用 `store.ttl` 鍵為 BaseStore 中的項目/記憶體配置預設資料過期時間。此鍵決定了專案在上次存取後保留的時間（讀取操作可能會根據 `refresh_on_read` 刷新計時器）。請注意，這些預設值可以透過修改 `get`, `search` 等方法中的對應參數來逐次覆寫。

    `ttl` 配置是一個包含可選字段的物件：

    - `refresh_on_read`: 如果為真（預設值），則透過 `get` 或 `search` 存取項目時會重設其過期計時器。設定為 `false` 則僅在寫入 `put` 操作時刷新 TTL。
    - `default_ttl`: Item 的預設生存時間（以分鐘為單位）。僅適用於新創建的 item；現有 item 不會被修改。如果未設置，item 預設不會過期。
    - `sweep_interval_minutes`: 系統執行後台程序刪除過期項目的頻率（以分鐘為單位）。如果未設置，則不會自動執行清理操作。

    以下範例啟用 7 天 TTL（10080 分鐘），讀取時刷新，並每小時進行一次掃描：

    ```json
    {
        "$schema": "https://langgra.ph/schema.json",
        "dependencies": ["."],
        "graphs": {
            "memory_agent": "./agent/graph.py:graph"
        },
        "store": {
            "ttl": {
                "refresh_on_read": true,
                "sweep_interval_minutes": 60,
                "default_ttl": 10080
                }
        }
    }
    ```

    **Configuring checkpoint Time-to-Live**
    您可以使用檢查點鍵配置檢查點的生存時間 (TTL)。這決定了檢查點資料在根據指定策略（例如，刪除）自動處理之前保留多長時間。支援兩個可選子物件：

    - `ttl`: 包括 strategy、sweep_interval_minutes 和 default_ttl，它們共同決定檢查點的過期時間。
    - `serde` (Agent server 0.5+) : 讓您控制檢查點有效負載的反序列化行為。
    
    以下範例將預設 TTL 設定為 30 天（43200 分鐘）：
    ```json
    {
        "$schema": "https://langgra.ph/schema.json",
        "dependencies": ["."],
        "graphs": {
            "chat": "chat.graph:graph"
        },
        "checkpointer": {
            "ttl": {
                "strategy": "delete",
                "sweep_interval_minutes": 10,
                "default_ttl": 43200
            }
        }
    }
    ```

    在這個例子中，超過 30 天的檢查點將被刪除，檢查每 10 分鐘運行一次。

    **Configuring checkpointer serde**

    `checkpointer.serde` 物件定義了反序列化過程：

    - `allowed_json_modules` 定義一個允許列表，用於指定伺服器可以從以 `json` 模式保存的有效負載中反序列化的自訂 Python 物件。這是一個包含 [path, to, module, file, symbol] 序列的清單。如果省略，則僅允許 LangChain 安全的預設值。您可以將其設為 true（不安全），以允許反序列化任何模組。
    - `pickle_fallback`: 當 JSON 解碼失敗時，是否回退到 pickle 反序列化。

    ```json
    {
    "checkpointer": {
        "serde": {
            "allowed_json_modules": [
                ["my_agent", "auth", "SessionState"]
                ]
            }
        }
    }
    ```

    **Customizing HTTP middleware and headers**

    http 配置區塊可讓您微調請求處理：

    - `middleware_order`: 選擇 `auth_first` 可在中間件之前執行身份驗證，或選擇 `middleware_first`（預設）以反轉該順序。
    - `enable_custom_route_auth`: 將身份驗證擴展到透過 http.app 掛載的路由。
    - `configurable_headers / logging_headers`: 每個函數都接受一個 object，該 object 帶有可選的包含和排除數組；支援通配符，排除操作在包含操作之前執行。
    - `cors`: 除了 `allow_origins`, `allow_methods` 和 `allow_headers` 之外，您還可以設定 `allow_credentials`, `allow_origin_regex`, `expose_headers` 和 `max_age` 來進行更詳細的瀏覽器控制。

    **Pinning API version**

    您可以使用 `api_version` 鍵來鎖定代理伺服器的 API 版本。如果您希望確保伺服器使用特定版本的 API，這將非常有用。預設情況下，雲端部署中的建置使用伺服器的最新穩定版本。您可以將 `api_version` 鍵設定為特定版本來鎖定此版本。

    ```json
    {
        "$schema": "https://langgra.ph/schema.json",
        "dependencies": ["."],
        "graphs": {
            "chat": "chat.graph:graph"
        },
        "api_version": "0.2"
    }
    ```




=== "JS"

    **Basic configuration**

    ```json
    {
        "$schema": "https://langgra.ph/schema.json",
        "graphs": {
            "chat": "./src/graph.ts:graph"
        }
    }
    ```


    **Pinning API version**

    您可以使用 `api_version` 鍵來鎖定 Agent Server 的 API 版本。如果您希望確保伺服器使用特定版本的 API，這將非常有用。預設情況下，雲端部署中的建置使用伺服器的最新穩定版本。您可以將 `api_version` 鍵設定為特定版本來鎖定此版本。

    ```json
    {
        "$schema": "https://langgra.ph/schema.json",
        "dependencies": ["."],
        "graphs": {
            "chat": "./src/chat/graph.ts:graph"
        },
        "api_version": "0.2"
    }
    ```

## Commands

**Usage**

=== "Python"

    LangGraph CLI 的基本指令是 `langgraph`。

    ```bash
    langgraph [OPTIONS] COMMAND [ARGS]
    ```
    
=== "JS"

    LangGraph.js CLI 的基本指令是 `langgraphjs`。

    ```bash
    npx @langchain/langgraph-cli [OPTIONS] COMMAND [ARGS]
    ```

    我們建議使用 **npx** 來始終使用最新版本的 CLI。

### `dev`

=== "Python"

    以開發模式運行 LangGraph API 伺服器，支援熱重載和調試功能。這款輕量級伺服器無需安裝 Docker，適用於開發和測試。狀態持久化到本地目錄。

    !!! info
        目前，CLI 僅支援 Python >= 3.11。

    **Installation**

    此命令需要安裝 "inmem" 附加元件：

    ```bash
    pip install -U "langgraph-cli[inmem]"
    ```

    **Usage**

    ```bash
    langgraph dev [OPTIONS]
    ```

=== "JS"

    以開發模式運行 LangGraph API 伺服器，並啟用熱重載功能。這款輕量級伺服器無需安裝 Docker，適用於開發和測試。狀態持久化到本地目錄。

    **Usage**

    ```bash
    npx @langchain/langgraph-cli dev [OPTIONS]
    ```

**Options**

|Option	|Default	|Description|
|-------|-----------|-----------|
|`-c, --config FILE`	|`langgraph.json`	|配置檔案路徑，該檔案聲明依賴項、graph 和環境變數|
|`--host TEXT`	|`127.0.0.1`	|要將伺服器綁定到的 host name|
|`--port INTEGER`	|`2024`	|伺服器綁定的 port|
|`--no-reload`	|	|停用自動重新加載|
|`--n-jobs-per-worker INTEGER`	|	|每位 worker 負責的工作數。預設值為 10。|
|`--debug-port INTEGER`	|	|調試器監聽的 port|
|`--wait-for-client`	|`False`	|等待 debugger client 連接到調試連接埠後再啟動伺服器。|
|`--no-browser`	|	|伺服器啟動時跳過自動開啟瀏覽器的操作|
|`--studio-url TEXT`	|	|要連線的 LangSmith Studio 實例的 URL。預設為 https://smith.langchain.com|
|`--allow-blocking`	|`False`	|不要在程式碼中對同步 I/O 阻塞操作引發錯誤（0.2.6 版本新增）|
|`--tunnel`	|`False`	|透過公共 tunnel（例如 Cloudflare）暴露本機伺服器，以便進行遠端前端存取。這樣可以避免 Safari 等瀏覽器或網路阻止本地主機連線的問題。|
|`--help`	|	|顯示 command 文檔|

### `build`

=== "Python"

    建置 LangSmith API 伺服器 Docker 映像。

    **Usage**

    ```bash
    langgraph build [OPTIONS]
    ```

=== "JS"

    建置 LangSmith API 伺服器 Docker 映像。

    **Usage**

    ```bash
    npx @langchain/langgraph-cli build [OPTIONS]
    ```

**Options**

|Option	|Default	|Description|
|-------|-----------|-----------|
|`--platform TEXT`	|	|要建置 Docker 映像的目標平台。例如: `langgraph build --platform linux/amd64,linux/arm64`|
|`-t, --tag TEXT`	|	|**Required.** Docker映像的標籤。例如: `langgraph build -t my-image`|
|`--no-pull`	|	|使用本地建置的鏡像。預設為 `false`，即使用最新的遠端 Docker 映像進行建置。|
|`-c, --config FILE`	|`langgraph.json`	|設定檔路徑，用於聲明依賴項、graph 和環境變數。|
|`--help`	|	|顯示 command 文件。|


### `up`

=== "Python"

    啟動 LangGraph API 伺服器。本地測試需要具有 LangSmith 存取權限的 LangSmith API 金鑰。生產環境使用需要 license key。

    **Usage**

    ```bash
    langgraph up [OPTIONS]
    ```

=== "JS"

    啟動 LangGraph API 伺服器。本地測試需要具有 LangSmith 存取權限的 LangSmith API 金鑰。生產環境使用需要 license key。

    **Usage**

    ```bash
    npx @langchain/langgraph-cli up [OPTIONS]
    ```

**Options**

|Option	|Default	|Description|
|-------|-----------|-----------|
|`--wait`	|	|等待服務啟動後再返回。|
|`--base-image TEXT`	|`langchain/langgraph-api`	|用於 LangGraph API 伺服器的基礎鏡像。使用版本標籤鎖定到特定版本。|
|`--image TEXT`	|	|用於 langgraph-api 服務的 Docker 映像。如果指定，則跳過建置步驟並直接使用此鏡像。|
|`--postgres-uri TEXT`	|`Local database`	|要用於資料庫的Postgres URI。|
|`--watch`	|	|文件更改後重新啟動|
|`-c, --config FILE`	|`langgraph.json`	|設定檔路徑，用於聲明依賴項、graph 和環境變數。|
|`-d, --docker-compose FILE`	|	|包含要啟動的其他服務的 docker-compose.yml 檔案路徑。|
|`-p, --port INTEGER`	|`8123`	|要暴露的連接埠。例如：`langgraph up --port 8000`|
|`--no-pull`	|	|使用本地建置的鏡像。預設為 false，即使用最新的遠端 Docker 映像進行建置。|
|`--recreate`	|	|即使容器的配置和鏡像沒有改變，也要重新建立容器。|
|`--help`	|	|顯示 command 文件。|

### `dockerfile`

=== "Python"

    產生用於建置 LangSmith API 伺服器 Docker 映像的 Dockerfile。

    **Usage**

    ```bash
    langgraph dockerfile [OPTIONS] SAVE_PATH
    ```

    **Options**

    |Option	|Default	|Description|
    |-------|-----------|-----------|
    |`-c, --config FILE`	|`langgraph.json`	|設定檔路徑，該檔案聲明依賴項、graph 和環境變數。|
    |`--help`	|	|顯示此訊息並退出。|

    **Example**

    ```bash
    langgraph dockerfile -c langgraph.json Dockerfile
    ```

    這將產生一個類似於以下內容的 Dockerfile：

    ```bash
    FROM langchain/langgraph-api:3.11

    ADD ./pipconf.txt /pipconfig.txt

    RUN PIP_CONFIG_FILE=/pipconfig.txt PYTHONDONTWRITEBYTECODE=1 pip install --no-cache-dir -c /api/constraints.txt langchain_community langchain_anthropic langchain_openai wikipedia scikit-learn

    ADD ./graphs /deps/__outer_graphs/src
    RUN set -ex && \
        for line in '[project]' \
                    'name = "graphs"' \
                    'version = "0.1"' \
                    '[tool.setuptools.package-data]' \
                    '"*" = ["**/*"]'; do \
            echo "$line" >> /deps/__outer_graphs/pyproject.toml; \
        done

    RUN PIP_CONFIG_FILE=/pipconfig.txt PYTHONDONTWRITEBYTECODE=1 pip install --no-cache-dir -c /api/constraints.txt -e /deps/*

    ENV LANGSERVE_GRAPHS='{"agent": "/deps/__outer_graphs/src/agent.py:graph", "storm": "/deps/__outer_graphs/src/storm.py:graph"}'
    ```

    !!! info
        `langgraph dockerfile` 指令會將 langgraph.json 檔案中的所有設定轉換為 Dockerfile 指令。使用此命令時，每次更新 langgraph.json 檔案後都需要重新執行它。否則，在建置或執行 Dockerfile 時，您的變更將不會生效。

=== "JS"

    產生用於建置 LangSmith API 伺服器 Docker 映像的 Dockerfile。

    **Usage**

    ```bash
    npx @langchain/langgraph-cli dockerfile [OPTIONS] SAVE_PATH
    ```

    **Options**

    |Option	|Default	|Description|
    |-------|-----------|-----------|
    |`-c, --config FILE`	|`langgraph.json`	|設定檔路徑，該檔案聲明依賴項、graph 和環境變數。|
    |`--help`	|	|顯示此訊息並退出。|

    **Example**

    ```bash
    npx @langchain/langgraph-cli dockerfile -c langgraph.json Dockerfile
    ```

    這將產生一個類似於以下內容的 Dockerfile：

    ```bash
    FROM langchain/langgraphjs-api:20

    ADD . /deps/agent

    RUN cd /deps/agent && yarn install

    ENV LANGSERVE_GRAPHS='{"agent":"./src/react_agent/graph.ts:graph"}'

    WORKDIR /deps/agent

    RUN (test ! -f /api/langgraph_api/js/build.mts && echo "Prebuild script not found, skipping") || tsx /api/langgraph_api/js/build.mts
    ```

    !!! info
        `npx @langchain/langgraph-cli dockerfile` 指令會將 `langgraph.json` 檔案中的所有設定轉換為 Dockerfile 指令。使用此命令後，每次更新 `langgraph.json` 檔案時都需要重新執行它。否則，在建置或執行 Dockerfile 時，您的變更將不會生效。
