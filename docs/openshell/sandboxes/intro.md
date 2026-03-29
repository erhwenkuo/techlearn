# About Gateways and Sandboxes

每個 OpenShell 部署都始於一個 Gateway 和一個或多個沙箱。Gateway 是控制平面，負責管理沙箱的生命週期, 提供程序和策略。沙箱是資料平面，它是一個安全, 私密的執行環境，用於運行 AI 代理程式。

每個沙箱都運行多層保護，以防止未經授權的資料存取, 憑證外洩和網路資料外洩。這些保護層包括檔案系統限制（Landlock）, 系統呼叫過濾（seccomp）, 網路命名空間隔離, 強制執行隱私保護的 HTTP CONNECT 代理程式。

## Gateway Types

網關負責配置沙箱, 代理 CLI 請求, 執行策略以及管理提供者憑證。 OpenShell 支援三種部署模式，因此網關可以運行在工作負載所需的任何位置。

|類型|適用範圍|最佳用途|
|----|------|-------|
|**Local**|工作站上的 Docker Desktop|獨立開發，快速迭代。如果本機網關不存在，CLI 會自動啟動一個。|
|**Remote**|透過 SSH 在遠端主機上運行 Docker。|在性能更強大的機器（例如 DGX Spark）上運行沙箱，同時將 CLI 保留在筆記型電腦上。|
|**Cloud**|在反向代理（例如 Cloudflare Access）之後|單一使用者透過雲端虛擬機器存取 OpenShell。雲端網關目前尚不適用於團隊共享存取。|

這三種類型都提供相同的 API 介面。無論網關運作在何處，沙箱, 策略和提供者的工作方式都完全相同。唯一的區別在於 CLI 連接到網關的方式：是透過直接的 Docker socket, SSH tunnel，還是透過代理的 HTTPS。

!!! tip
    您無需手動部署網關。執行不含網關的 `openshell sandbox create` 指令會自動為您構建一個本機網關。

## Sandbox Lifecycle

每個沙盒都會經歷一連串既定的階段：

|階段|描述|
|----|---|
|**Provisioning**|Runtime 環境正在設定沙箱環境, 注入憑證並套用您的策略。|
|**Ready**|沙箱正在運行。代理進程處於活動狀態，所有隔離層均已啟用。您可以連線、同步檔案和查看日誌。|
|**Error**|配置或執行過程中出現錯誤。請查看 openshell 日誌以取得詳細資訊。|
|**Deleting**|沙箱環境正在拆除。系統將釋放資源並清除憑證。|

## Sandbox Policies

OpenShell 內建了一個策略，可直接用於常見的 agent 程式工作流程。如果您建立沙箱時未使用 `--policy` 參數，則會套用預設原則。此策略控制三個方面。

|層別|它控制什麼|它是如何工作的|
|---|-------|------------|
|**Filesystem**|Agent 程式可以存取磁碟上的哪些內容|路徑被分為唯讀路徑集和讀寫路徑集。 Landlock LSM 在核心層級強制執行這些限制。|
|**Network**|Agent 在網路上可以存取的內容|每個策略區塊將允許的目標位址（主機和連接埠）與允許的二進位檔案（可執行檔案路徑）配對。代理伺服器會將每個出站連線與開啟該連線的二進位檔案進行比對。兩者必須完全匹配，否則連接將被拒絕。|
|**Process**|Agent 擁有哪些權限|此代理程式以非特權使用者身分執行，並帶有 seccomp 過濾器，可阻止危險的系統呼叫。不使用 `sudo`, `setuid` 或任何其他提升權限的途徑。|

如需了解每個預設策略區塊的完整細則和代理程式相容性詳情，請參閱 [預設策略參考](https://docs.nvidia.com/openshell/latest/reference/default-policy.html)。

如需檢視包含註解的 YAML 範例的完整策略結構，請參閱 [自訂沙盒策略](https://docs.nvidia.com/openshell/latest/sandboxes/policies.html)。

