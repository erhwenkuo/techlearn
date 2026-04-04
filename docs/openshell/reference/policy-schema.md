# Policy Schema Reference

完整的沙箱策略 YAML 欄位參考。每個欄位都記錄了其類型、是否必要以及是靜態（在沙箱建立時鎖定）還是動態（在運行中的沙箱上可熱重載）。

## Top-Level Structure

策略 YAML 檔案包含以下頂層欄位：

```yaml
version: 1
filesystem_policy: { ... }
landlock: { ... }
process: { ... }
network_policies: { ... }
```

|欄位|類型|必填|類別|描述|
|----|---|---|----|---|
|`version`|integer|Yes|-|策略架構版本。必須為 `1`。|
|`filesystem_policy`|object|No|Static|控制代理可以讀取和寫入哪些目錄。|
|`landlock`|object|No|Static|配置 Landlock LSM 強制執行行為。|
|`process`|object|No|Static|設定 agent 進程運行的 user 和 group。|
|`network_policies`|map|No|Dynamic|聲明哪些二進位檔案可以存取哪些網路端點。|

靜態字段在沙箱建立時設定。更改這些欄位需要銷毀並重新建立沙箱。動態欄位可以在已執行且設定了 openshell 策略的沙箱上更新(`openshell policy set`)，並且無需重新啟動即可生效。

## Version

版本(`version`)欄位用於標識策略使用的 scheam：

|欄位|類型|必填|描述|
|----|---|---|----|
|`version`|integer|Yes|Schema 版本號。目前必須為 `1`。|

## Filesystem Policy

**Category**: Static

控製沙箱內的檔案系統存取。未列入唯讀 `read_only` 或讀寫 `read_write` 清單的路徑將無法存取。

|欄位|類型|必填|描述|
|----|---|---|----|
|`include_workdir`|bool|No|如果為 `true`，則自動將 agent 程式的工作目錄新增至 `read_write` 權限。|
|`read_only`|list of strings|No|Agent 可以讀取但不能修改的路徑。通常是系統目錄，例如 `/usr`, `/lib`, `/etc`。|
|`read_write`|list of strings|No|Agent 程式可以讀取和寫入的路徑。通常是 `/sandbox`（工作目錄）和 `/tmp`。|

**Validation constraints**:

- 所有路徑必須是絕對路徑（以 `/` 開頭）。
- 路徑中不得包含 `..` 遍歷組件。伺服器會在儲存路徑前進行標準化，但會拒絕遍歷範圍超出預期的策略。
- 讀寫路徑(Read-write paths)不能過寬（例如，僅包含 `/` 的路徑會被拒絕）。
- 每個路徑的長度不得超過 4096 個字元。
- `read_only` 和 `read_write` 路徑的總長度不得超過 256 個字元。

違反這些約束的策略在建立或更新時會被拒絕，並傳回 `INVALID_ARGUMENT` 錯誤。磁碟載入的 YAML 策略如果驗證失敗，則會回退到限制性更強的預設值。

範例:

```yaml
filesystem_policy:
  include_workdir: true
  read_only:
    - /usr
    - /lib
    - /proc
    - /dev/urandom
    - /etc
  read_write:
    - /sandbox
    - /tmp
    - /dev/null
```

## Landlock

**Category**: Static

在核心層級配置 [Landlock LSM](https://docs.kernel.org/security/landlock.html) 強制執行。 Landlock 提供低於 UNIX 權限允許的強制性檔案系統存取控制。

|欄位|類型|必填|設定|描述|
|----|---|---|----|---|
|`compatibility`|string|No|`best_effort`, `hard_requirement`|OpenShell 如何處理 Landlock 故障。請參閱下表。|

**Compatibility modes**:

|值|內核 ABI 不可用|單一路徑不可存取|所有路徑不可存取|
|--|-------------|-------------|--------------|
|`best_effort`|警告並繼續執行無 Landlock。|跳過該路徑，套用剩餘規則。|發出警告並繼續執行，不應用 Landlock（拒絕應用空規則集）。|
|`hard_requirement`|中止沙箱啟動。|中止沙箱啟動。|中止沙箱啟動。|

`best_effort`（預設值）適用於大多數部署。它能優雅地處理缺失路徑——例如，`/app` 路徑可能並非存在於每個容器鏡像中，但對於存在該路徑的容器，它會被包含在基線路徑集中。對於個別缺少的路徑，系統會跳過處理，同時仍強制執行其餘檔案系統規則。

`hard_requirement` 適用於檔案系統隔離中任何漏洞都無法接受的環境。如果列出的路徑因任何原因（缺失、權限被拒絕、符號連結循環）無法打開，沙箱啟動將立即失敗，而不是以降低的保護等級運作。

當在 `best_effort` 模式下跳過某個路徑時，沙箱會記錄一條警告，其中包含該路徑、具體錯誤以及人類可讀的原因（例如，“path does not exist” 或 “permission denied”）。

範例:

```yaml
landlock:
  compatibility: best_effort
```

## Process

**Category**: Static

設定沙箱內 agent 程式的作業系統級身分。

|欄位|類型|必填|描述|
|----|---|---|----|
|`run_as_user`|string|No|Agent 進程運行所使用的使用者名稱或 UID。預設值：`sandbox`。|
|`run_as_group`|string|No|Agent 程式運行所使用的群組名稱或 GID。預設值：`sandbox`。|

**Validation constraint**：`run_as_user` 和 `run_as_group` 都不能設定為 `root` 或 `0`。請求 root 進程身分的策略在建立或更新時將被拒絕。

範例:

```yaml
process:
  run_as_user: sandbox
  run_as_group: sandbox
```

## Network Policies

**Category**: Dynamic

一個包含命名網路策略條目的映射表。每個條目聲明一組端點和一組二進位檔案。只有列出的二進位檔案才能連接到列出的端點。映射表的鍵是一個邏輯標識符。條目中的 `name` 欄位是日誌中使用的顯示名稱。

### Network Policy Entry

`network_policies` 映射中的每個條目都包含以下欄位：

|欄位|類型|必填|描述|
|----|---|---|----|
|`name`|string|No|策略條目的顯示名稱。用於日誌輸出。預設為 map key。|
|`endpoints`|list of endpoint objects|Yes|此條目允許的主機和連接埠。|
|`binaries`|list of binary objects|Yes|允許連接到這些端點的可執行檔。|

