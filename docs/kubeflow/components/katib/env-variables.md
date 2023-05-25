# Katib 組件的環境變量

**如何為每個 Katib 組件設置環境變量**

本指南描述了每個 Katib 組件的環境變量。如果你想改變你的 Katib 安裝，你可以修改其中的一些變量。

在下表中，您可以找到每個 Katib 組件中所有環境變量的描述、默認值和強制屬性。如果變量具有強制屬性，則需要在適當的 Katib 組件的清單中設置相關的環境變量。

## Katib Controller

下面是 [Katib Controller](https://github.com/kubeflow/katib/blob/master/manifests/v1beta1/components/controller/controller.yaml) 部署的環境變量：

|Variable	|Description	|Default Value	|Mandatory|
|---------|-------------|---------------|---------|
|`KATIB_CORE_NAMESPACE`	|所有 Katib 組件和默認實驗的基本命名空間	|`metadata.namespace`	|Yes|
|`KATIB_SUGGESTION_COMPOSER`	|Katib Suggestions 的 [Composer](https://github.com/kubeflow/katib/blob/master/pkg/controller.v1beta1/suggestion/composer/composer.go)。 你可以用你自己的 Composer	|general	|No|
|`KATIB_DB_MANAGER_SERVICE_NAMESPACE`	|Katib DB Manager 命名空間	|`KATIB_CORE_NAMESPACE` env variable	|No|
|`KATIB_DB_MANAGER_SERVICE_IP`	|Katib DB Manager IP	|katib-db-manager	|No|
|`KATIB_DB_MANAGER_SERVICE_PORT`	|Katib DB Manager Port	|6789	|No|

Katib Controller 使用此地址表達式調用 Katib DB Manager：

- **`KATIB_DB_MANAGER_SERVICE_IP.KATIB_DB_MANAGER_SERVICE_NAMESPACE:KATIB_DB_MANAGER_SERVICE_PORT`**

如果您設置 `KATIB_DB_MANAGER_SERVICE_NAMESPACE=""`，Katib Controller 使用此地址調用 Katib DB Manager：

- **`KATIB_DB_MANAGER_SERVICE_IP:KATIB_DB_MANAGER_SERVICE_PORT`**

如果您想使用自己的數據庫管理器來報告 Katib 指標，您可以更改 `KATIB_DB_MANAGER_SERVICE_NAMESPACE`、`KATIB_DB_MANAGER_SERVICE_IP` 和 `KATIB_DB_MANAGER_SERVICE_PORT` 變量。


## Katib UI

以下是 Katib UI 部署的環境變量：

|Variable	|Description	|Default Value	|Mandatory|
|---------|-------------|---------------|---------|
|`KATIB_CORE_NAMESPACE`	|所有 Katib 組件和默認實驗的基本命名空間	|`metadata.namespace`	|Yes|
|`KATIB_DB_MANAGER_SERVICE_NAMESPACE`	|Katib DB Manager 命名空間	|`KATIB_CORE_NAMESPACE` env variable	|No|
|`KATIB_DB_MANAGER_SERVICE_IP`	|Katib DB Manager IP	|katib-db-manager	|No|
|`KATIB_DB_MANAGER_SERVICE_PORT`	|Katib DB Manager Port	|6789	|No|

Katib UI 使用與 Katib Controller 相同的地址表達式調用 Katib DB Manager。

## Katib DB Manager

下面是 Katib DB Manager 部署的環境變量：

|Variable	|Description	|Default Value	|Mandatory|
|---------|-------------|---------------|---------|
|`DB_NAME`	|Katib DB Name: 'mysql' or 'postgres'|		|Yes|
|`DB_PASSWORD`	|Katib DB Password	|test (MySQL)<br/>katib (Postgres)	|Yes|
|`DB_USER`	|Katib DB User	|root (MySQL) <br/>katib (Postgres)	|No|
|`KATIB_MYSQL_DB_HOST`	|Katib MySQL Host	|katib-mysql	|No|
|`KATIB_MYSQL_DB_PORT`	|Katib MySQL Port	|3306	|No|
|`KATIB_MYSQL_DB_DATABASE`	|Katib MySQL Database name	|katib	|No|
|`KATIB_POSTGRESQL_DB_HOST`	|Katib Postgres Host	|katib-postgres|	No|
|`KATIB_POSTGRESQL_DB_PORT`	|Katib Postgres Port	|5432	|No|
|`KATIB_POSTGRESQL_DATABASE`	|Katib Postgres Database name	|katib	|No|

目前，Katib DB Manager 僅支持 MySQL 和 Postgres 數據庫。 （`DB_NAME` 環境變量必須填寫 `mysql` 或 `postgres` 之一）
但是，您可以使用自己的數據庫管理器和數據庫通過實現 katib 數據庫接口來報告指標。

對於 Katib DB Manager，您可以將 `DB_PASSWORD` 更改為您自己的數據庫密碼。

Katib DB Manager 通過 DB 的類型創建到 DB 的 DB 連接。

如果 `DB_NAME=mysql`，則使用 `mysql` 驅動程序和此數據源名稱：

- `DB_USER:DB_PASSWORD@tcp(KATIB_MYSQL_DB_HOST:KATIB_MYSQL_DB_PORT)/KATIB_MYSQL_DB_DATABASE?timeout=5s`

如果 `DB_NAME=postgres`，則使用 `pg` 驅動程序和此數據源名稱：

- `postgresql://[DB_USER[:DB_PASSWORD]@][KATIB_POSTGRESQL_DB_HOST][:KATIB_POSTGRESQL_DB_PORT][/KATIB_POSTGRESQL_DB_DATABASE]`

## Katib DB

Katib 數據庫組件支持 MySQL 和 Postgres。

### Katib MySQL

對於 Katib MySQL，您需要設置這些環境變量：

- `MYSQL_ROOT_PASSWORD` 為來自 `katib-mysql-secrets` 的值，等於 “test”。
- `MYSQL_ALLOW_EMPTY_PASSWORD` 為 `true`
- `MYSQL_DATABASE` 為 `katib`。

Katib MySQL 環境變量必須與 Katib DB Manager 環境變量匹配，這意味著：

- `MYSQL_ROOT_PASSWORD` = `DB_PASSWORD`
- `MYSQL_DATABASE` = `KATIB_MYSQL_DB_DATABASE`

### Katib Postgres

對於 Katib Postgres，您需要設置這些環境變量：

- `POSTGRES_USER`、`POSTGRES_PASSWORD` 和 `POSTGRES_DB` 為來自 `katib-postgres-secrets` 的值，等於 “katib”。

Katib Postgres 環境變量必須與 Katib DB Manager 環境變量匹配，這意味著：

- `POSTGRES_USER` = `DB_USER`
- `POSTGRES_PASSWORD` = `DB_PASSWORD`
- `POSTGRES_DB` = `KATIB_POSTGRESQL_DB_DATABASE`



