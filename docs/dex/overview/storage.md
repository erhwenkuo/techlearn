# 存儲選項

Dex 需要持久狀態來執行各種任務，例如跟踪刷新令牌、防止重放和輪換密鑰。本文檔是 dex 支持的存儲配置的總結。

存儲漏洞很嚴重，因為它們會影響依賴 dex 的應用程序。 Dex 將敏感數據保存在其後端存儲中，包括簽名密鑰和 `bcrypt` 密碼。因此，無論選擇哪種存儲選項，都應使用傳輸安全和數據庫 ACL。

## Etcd

Dex 支持將狀態持久化到 [etcd v3](https://github.com/coreos/etcd)。

一個示例 etcd 配置使用這些值：

```yaml
storage:
  type: etcd
  config:
    # list of etcd endpoints we should connect to
    endpoints:
      - http://localhost:2379
    namespace: my-etcd-namespace/
```

可以使用以下選項進一步自定義 Etcd 存儲：

- `endpoints`：我們應該連接到的 etcd 端點列表
- `namespace`：要為連接設置的 etcd 命名空間。 etcd 存儲創建的所有鍵都將以命名空間為前綴。當您在多個應用程序之間共享 etcd 集群時，這很有用。另一種設置命名空間的方法是使用 etcd 代理
- `username`：etcd 認證的用戶名
- `password`：etcd 認證的密碼
- `ssl`：用於 etcd 連接的 ssl 設置
    - `serverName`：確保證書與客戶端連接到的給定主機名匹配。
    - `caFile`：ca 的路徑
    - `keyFile`：私鑰路徑
    - `certFile`：證書路徑

## Kubernetes 客制資源定義 (CRDs)

Kubernetes 客制資源定義是應用程序在 Kubernetes API 中創建新資源類型的一種方式。

客制資源定義 (CRD) API 物件是在 Kubernetes 1.7 版中引入的，用於替代第三方資源 (TPR) 擴展。 CRD 允許 dex 在現有 Kubernetes 集群之上運行，而無需外部數據庫。雖然這種存儲可能不適合大量用戶，但它對許多 Kubernetes 用例非常有效。

本節的其餘部分將探討 dex 如何使用 CRD 的內部細節。管理員不應直接與這些資源直接交互，調試時除外。這些資源僅用於存儲狀態，並不打算供最終用戶使用。如需動態修改 dex 的狀態，請參閱 [API 文檔](https://dexidp.io/docs/api/)。

下面是 dex 管理的 `AuthCode` 資源的例子：

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  creationTimestamp: 2017-09-13T19:56:28Z
  name: authcodes.dex.coreos.com
  resourceVersion: "288893"
  selfLink: /apis/apiextensions.k8s.io/v1beta1/customresourcedefinitions/authcodes.dex.coreos.com
  uid: a1cb72dc-98bd-11e7-8f6a-02d13336a01e
spec:
  group: dex.coreos.com
  names:
    kind: AuthCode
    listKind: AuthCodeList
    plural: authcodes
    singular: authcode
  scope: Namespaced
  version: v1
status:
  acceptedNames:
    kind: AuthCode
    listKind: AuthCodeList
    plural: authcodes
    singular: authcode
  conditions:
  - lastTransitionTime: null
    message: no conflicts found
    reason: NoConflicts
    status: "True"
    type: NamesAccepted
  - lastTransitionTime: 2017-09-13T19:56:28Z
    message: the initial names have been accepted
    reason: InitialNamesAccepted
    status: "True"
    type: Established
```

創建 `CustomResourceDefinition` 後，可以在名稱空間級別創建和存儲自定義資源。可以像使用 `kubectl` 的任何其他資源一樣查詢、刪除和編輯 CRD 類型和自定義資源。

dex 需要訪問非命名空間的 CustomResourceDefinition 類型。例如，使用 RBAC 授權的集群需要創建以下角色和綁定：

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: dex
rules:
- apiGroups: ["dex.coreos.com"] # API group created by dex
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["apiextensions.k8s.io"]
  resources: ["customresourcedefinitions"]
  verbs: ["create"] # To manage its own resources identity must be able to create customresourcedefinitions.
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: dex
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: dex
subjects:
- kind: ServiceAccount
  name: dex                 # Service account assigned to the dex pod.
  namespace: dex-namespace  # The namespace dex is running in.
```

### 配置

存儲配置非常有限，因為在 Kubernetes 集群之外運行的安裝可能更喜歡不同的存儲選項。 dex 在 Kubernetes 中運行的示例配置：

```yaml
storage:
  type: kubernetes
  config:
    inCluster: true
```

Dex 通過解析自動掛載到其 pod 中的服務帳戶令牌來確定其運行的名稱空間。


## SQL

Dex 支持三種風格的 SQL：`SQLite3`、`Postgres` 和 `MySQL`。

遷移在第一次連接到 SQL 服務器時自動執行（它不支持回滾）。因為這個 dex 需要權限來為其數據庫添加和更改資料表。

### SQLite3

SQLite3 是推薦給想快速立起 dex 的用戶的存儲。它不適用於實際工作負載。

SQLite3 配置只有一個參數，即數據庫文件。

```yaml
storage:
  type: sqlite3
  config:
    file: /var/dex/dex.db
```

因為 SQLite3 使用文件鎖來防止競爭條件，如果提供了 “:memory:” 值，dex 將自動禁用對併發數據庫查詢的支持。

### Postgres

使用 Postgres 時，出於以下原因，管理員可能希望將數據庫專用於 dex：

- Dex 需要對其數據庫的特權訪問，因為它執行遷移。
- Dex 的數據庫表名是不可配置的；當與其他應用程序共享時，可能會出現資料表名衝突。

```sql
CREATE DATABASE dex_db;
CREATE USER dex WITH PASSWORD '66964843358242dbaaa7778d8477c288';
GRANT ALL PRIVILEGES ON DATABASE dex_db TO dex;
```

使用這些值的 Postgres 設置示例配置：

```yaml
storage:
  type: postgres
  config:
    host: localhost
    port: 5432
    database: dex_db
    user: dex
    password: 66964843358242dbaaa7778d8477c288
    ssl:
      mode: verify-ca
      caFile: /etc/dex/postgres.ca
```

SSL “模式” 對應於 `github.com/lib/pq` 包連接選項。如果未指定，dex 默認為最嚴格的模式 “verify-full”。

### MySQL

Dex 需要 MySQL 5.7 或更高版本。使用 MySQL 時，出於以下原因，管理員可能希望將數據庫專用於 dex：

- Dex 需要對其數據庫的特權訪問，因為它執行遷移。
- Dex 的數據庫表名是不可配置的；當與其他應用程序共享時，可能會出現資料表名衝突。

```sql
CREATE DATABASE dex_db;
CREATE USER dex IDENTIFIED BY '66964843358242dbaaa7778d8477c288';
GRANT ALL PRIVILEGES ON dex_db.* TO dex;
```

使用這些值的 MySQL 設置示例配置：

```yaml
storage:
  type: mysql
  config:
    database: dex_db
    user: dex
    password: 66964843358242dbaaa7778d8477c288
    ssl:
      mode: custom
      caFile: /etc/dex/mysql.ca
```

SSL “模式” 對應於 `github.com/go-sql-driver/mysql` 包連接選項。如果未指定，dex 默認為最嚴格的模式 “true”。
