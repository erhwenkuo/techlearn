# MLflow 身份驗證

!!! note
    此功能仍處於實驗階段，可能會在未來版本中更改。

## 概述

若要啟用 MLflow 身份驗證，請使用下列命令啟動 MLflow UI：

```bash
mlflow server --app-name basic-auth
```

MLflow Tracking service 管理員可以隨時透過重新啟動 tracking service 來選擇停用此功能。建立的任何使用者和權限都將保留在 SQL 資料庫中，並在重新啟用該功能後重新投入使用。

由於 HTTP 身份驗證的性質，它僅支持 Remote Tracking Server，其中用戶透過 REST API 向伺服器發送請求。

## 運作設計

### 權限

可用的權限有：

| Permission | Can read | Can update | Can delete | Can manage |
|------------|----------|------------|------------|------------|
|**READ**|Yes|No|No|No|
|**EDIT**|Yes|Yes|No|No|
|**MANAGE**|Yes|Yes|Yes|Yes|
|**NO_PERMISSIONS**|No|No|No|No|

{==所有使用者的預設權限都是 **READ**==}。這個預設的設定可以在[設定檔](https://mlflow.org/docs/latest/auth/index.html#configuration)中更改。

可以為每個使用者授予對單獨資源的權限。支援的資源包括 `Experiment` 和 `Registered Model`。要存取 API 端點，使用者必須擁有所需的權限。否則，將傳回 403 Forbidden 回應。

存取 `Experiment` 所需的權限：

| API | Endpoint | Method | Required permission |
|------------|----------|------------|------------|
|[Create Experiment](https://mlflow.org/docs/latest/rest-api.html#mlflowmlflowservicecreateexperiment)|2.0/mlflow/experiments/create|POST|None|
|[Get Experiment](https://mlflow.org/docs/latest/rest-api.html#mlflowmlflowservicegetexperiment)|2.0/mlflow/experiments/get|GET|can_read|
|[Get Experiment By Name](https://mlflow.org/docs/latest/rest-api.html#mlflowmlflowservicegetexperimentbyname)|2.0/mlflow/experiments/get-by-name|GET|can_read|
|[Delete Experiment](https://mlflow.org/docs/latest/rest-api.html#mlflowmlflowservicedeleteexperiment)|2.0/mlflow/experiments/delete|POST|can_delete|
|[Restore Experiment](https://mlflow.org/docs/latest/rest-api.html#mlflowmlflowservicerestoreexperiment)|2.0/mlflow/experiments/restore|POST|can_delete|
|[Update Experiment](https://mlflow.org/docs/latest/rest-api.html#mlflowmlflowserviceupdateexperiment)|2.0/mlflow/experiments/update|POST|can_update|
|[Search Experiments](https://mlflow.org/docs/latest/rest-api.html#mlflowmlflowservicesearchexperiments)|2.0/mlflow/experiments/search|POST|None|
|[Search Experiments](https://mlflow.org/docs/latest/rest-api.html#mlflowmlflowservicesearchexperiments)|2.0/mlflow/experiments/search|GET|None|
|[Set Experiment Tag](https://mlflow.org/docs/latest/rest-api.html#mlflowmlflowservicesetexperimenttag)|2.0/mlflow/experiments/set-experiment-tag|POST|can_update|
|[Create Run](https://mlflow.org/docs/latest/rest-api.html#mlflowmlflowservicecreaterun)|2.0/mlflow/runs/create|POST|can_update|
|[Get Run](https://mlflow.org/docs/latest/rest-api.html#mlflowmlflowservicegetrun)|2.0/mlflow/runs/get|GET|can_read|
|[Update Run](https://mlflow.org/docs/latest/rest-api.html#mlflowmlflowserviceupdaterun)|2.0/mlflow/runs/update|POST|can_update|
|[Delete Run](https://mlflow.org/docs/latest/rest-api.html#mlflowmlflowservicedeleterun)|2.0/mlflow/runs/delete|POST|can_delete|
|[Restore Run](https://mlflow.org/docs/latest/rest-api.html#mlflowmlflowservicerestorerun)|2.0/mlflow/runs/restore|POST|can_delete|
|[Search Runs](https://mlflow.org/docs/latest/rest-api.html#mlflowmlflowservicesearchruns)|2.0/mlflow/runs/search|POST|None|
|[Set Tag](https://mlflow.org/docs/latest/rest-api.html#mlflowmlflowservicesettag)|2.0/mlflow/runs/set-tag|POST|can_update|
|[Delete Tag](https://mlflow.org/docs/latest/rest-api.html#mlflowmlflowservicedeletetag)|2.0/mlflow/runs/delete-tag|POST|can_update|
|[Log Metric](https://mlflow.org/docs/latest/rest-api.html#mlflowmlflowservicelogmetric)|2.0/mlflow/runs/log-metric|POST|can_update|
|[Log Param](https://mlflow.org/docs/latest/rest-api.html#mlflowmlflowservicelogparam)|2.0/mlflow/runs/log-parameter|POST|can_update|
|[Log Batch](https://mlflow.org/docs/latest/rest-api.html#mlflowmlflowservicelogbatch)|2.0/mlflow/runs/log-batch|POST|can_update|
|[Log Model](https://mlflow.org/docs/latest/rest-api.html#mlflowmlflowservicelogmodel)|2.0/mlflow/runs/log-model|POST|can_update|
|[List Artifacts](https://mlflow.org/docs/latest/rest-api.html#mlflowmlflowservicelistartifacts)|2.0/mlflow/artifacts/list|GET|can_read|
|[Get Metric History](https://mlflow.org/docs/latest/rest-api.html#mlflowmlflowservicegetmetrichistory)|2.0/mlflow/metrics/get-history|GET|can_read|


存取 `Registered Model` 所需的權限：

| API | Endpoint | Method | Required permission |
|------------|----------|------------|------------|
|[Create Registered Model](https://mlflow.org/docs/latest/rest-api.html#mlflowmodelregistryservicecreateregisteredmodel)|2.0/mlflow/registered-models/create|POST|None|
|[Rename Registered Model](https://mlflow.org/docs/latest/rest-api.html#mlflowmodelregistryservicerenameregisteredmodel)|2.0/mlflow/registered-models/rename|POST|can_update|
|[Update Registered Model](https://mlflow.org/docs/latest/rest-api.html#mlflowmodelregistryserviceupdateregisteredmodel)|2.0/mlflow/registered-models/update|PATCH|can_update|
|[Delete Registered Model](https://mlflow.org/docs/latest/rest-api.html#mlflowmodelregistryservicedeleteregisteredmodel)|2.0/mlflow/registered-models/delete|DELETE|can_delete|
|[Get Registered Model](https://mlflow.org/docs/latest/rest-api.html#mlflowmodelregistryservicegetregisteredmodel)|2.0/mlflow/registered-models/get|GET|can_read|
|[Search Registered Models](https://mlflow.org/docs/latest/rest-api.html#mlflowmodelregistryservicesearchregisteredmodels)|2.0/mlflow/registered-models/search|GET|None|
|[Get Latest Versions](https://mlflow.org/docs/latest/rest-api.html#mlflowmodelregistryservicegetlatestversions)|2.0/mlflow/registered-models/get-latest-versions|POST|can_read|
|[Get Latest Versions](https://mlflow.org/docs/latest/rest-api.html#mlflowmodelregistryservicegetlatestversions)|2.0/mlflow/registered-models/get-latest-versions|GET|can_read|
|[Set Registered Model Tag](https://mlflow.org/docs/latest/rest-api.html#mlflowmodelregistryservicesetregisteredmodeltag)|2.0/mlflow/registered-models/set-tag|POST|can_update|
|[Delete Registered Model Tag](https://mlflow.org/docs/latest/rest-api.html#mlflowmodelregistryservicedeleteregisteredmodeltag)|2.0/mlflow/registered-models/delete-tag|DELETE|can_update|
|[Set Registered Model Alias](https://mlflow.org/docs/latest/rest-api.html#mlflowmodelregistryservicesetregisteredmodelalias)|2.0/mlflow/registered-models/alias|POST|can_update|
|[Delete Registered Model Alias](https://mlflow.org/docs/latest/rest-api.html#mlflowmodelregistryservicedeleteregisteredmodelalias)|2.0/mlflow/registered-models/alias|DELETE|can_delete|
|[Get Model Version By Alias](https://mlflow.org/docs/latest/rest-api.html#mlflowmodelregistryservicegetmodelversionbyalias)|2.0/mlflow/registered-models/alias|GET|can_read|
|[Create Model Version](https://mlflow.org/docs/latest/rest-api.html#mlflowmodelregistryservicecreatemodelversion)|2.0/mlflow/model-versions/create|POST|can_update|
|[Update Model Version](https://mlflow.org/docs/latest/rest-api.html#mlflowmodelregistryserviceupdatemodelversion)|2.0/mlflow/model-versions/update|PATCH|can_update|
|[Transition Model Version Stage](https://mlflow.org/docs/latest/rest-api.html#mlflowmodelregistryservicetransitionmodelversionstage)|2.0/mlflow/model-versions/transition-stage|POST|can_update|
|[Delete Model Version](https://mlflow.org/docs/latest/rest-api.html#mlflowmodelregistryservicedeletemodelversion)|2.0/mlflow/model-versions/delete|DELETE|can_delete|
|[Get Model Version](https://mlflow.org/docs/latest/rest-api.html#mlflowmodelregistryservicegetmodelversion)|2.0/mlflow/model-versions/get|GET|can_read|
|[Search Model Versions](https://mlflow.org/docs/latest/rest-api.html#mlflowmodelregistryservicesearchmodelversions)|2.0/mlflow/model-versions/search|GET|None|
|[Get Model Version Download Uri](https://mlflow.org/docs/latest/rest-api.html#mlflowmodelregistryservicegetmodelversiondownloaduri)|2.0/mlflow/model-versions/get-download-uri|GET|can_read|
|[Set Model Version Tag](https://mlflow.org/docs/latest/rest-api.html#mlflowmodelregistryservicesetmodelversiontag)|2.0/mlflow/model-versions/set-tag|POST|can_update|
|[Delete Model Version Tag](https://mlflow.org/docs/latest/rest-api.html#mlflowmodelregistryservicedeletemodelversiontag)|2.0/mlflow/model-versions/delete-tag|DELETE|can_delete|

MLflow 身份驗證引入了幾個新的 API 端點來管理使用者和權限。

| API | Endpoint | Method | Required permission |
|------------|----------|------------|------------|
|[Create User](https://mlflow.org/docs/latest/auth/rest-api.html#mlflowauthservicecreateuser)|2.0/mlflow/users/create|POST|None|
|[Get User](https://mlflow.org/docs/latest/auth/rest-api.html#mlflowauthservicegetuser)|2.0/mlflow/users/get|GET|Only readable by that user|
|[Update User Password](https://mlflow.org/docs/latest/auth/rest-api.html#mlflowauthserviceupdateuserpassword)|2.0/mlflow/users/update-password|PATCH|Only updatable by that user|
|[Update User Admin](https://mlflow.org/docs/latest/auth/rest-api.html#mlflowauthserviceupdateuseradmin)|2.0/mlflow/users/update-admin|PATCH|Only admin|
|[Delete User](https://mlflow.org/docs/latest/auth/rest-api.html#mlflowauthservicedeleteuser)|2.0/mlflow/users/delete|DELETE|Only admin|
|[Create Experiment Permission](https://mlflow.org/docs/latest/auth/rest-api.html#mlflowauthservicecreateexperimentpermission)|2.0/mlflow/experiments/permissions/create|POST|can_manage|
|[Get Experiment Permission](https://mlflow.org/docs/latest/auth/rest-api.html#mlflowauthservicegetexperimentpermission)|2.0/mlflow/experiments/permissions/get|GET|can_manage|
|[Update Experiment Permission](https://mlflow.org/docs/latest/auth/rest-api.html#mlflowauthserviceupdateexperimentpermission)|2.0/mlflow/experiments/permissions/update|PATCH|can_manage|
|[Delete Experiment Permission](https://mlflow.org/docs/latest/auth/rest-api.html#mlflowauthservicedeleteexperimentpermission)|2.0/mlflow/experiments/permissions/delete|DELETE|can_manage|
|[Create Registered Model Permission](https://mlflow.org/docs/latest/auth/rest-api.html#mlflowauthservicecreateregisteredmodelpermission)|2.0/mlflow/registered-models/permissions/create|POST|can_manage|
|[Get Registered Model Permission](https://mlflow.org/docs/latest/auth/rest-api.html#mlflowauthservicegetregisteredmodelpermission)|2.0/mlflow/registered-models/permissions/get|GET|can_manage|
|[Update Registered Model Permission](https://mlflow.org/docs/latest/auth/rest-api.html#mlflowauthserviceupdateregisteredmodelpermission)|2.0/mlflow/registered-models/permissions/update|PATCH|can_manage|
|[Delete Registered Model Permission](https://mlflow.org/docs/latest/auth/rest-api.html#mlflowauthservicedeleteregisteredmodelpermission)|2.0/mlflow/registered-models/permissions/delete|DELETE|can_manage|

有一些 API 權限會被幾個 API 觸發的結果而被修改。例如，`Experiment`　的創建者將自動被授予對該實驗的 MANAGE 權限，以便創建者可以授予或撤銷其他使用者對該 `Experiment` 的存取權限。

| API | Endpoint | Method | Effect |
|------------|----------|------------|------------|
|[Create Experiment](https://mlflow.org/docs/latest/rest-api.html#mlflowmlflowservicecreateexperiment)|2.0/mlflow/experiments/create|POST|{==自動向建立者授予 `MANAGE` 權限。==}|
|[Create Registered Model](https://mlflow.org/docs/latest/rest-api.html#mlflowmodelregistryservicecreateregisteredmodel)|2.0/mlflow/registered-models/create|POST|{==自動向建立者授予 `MANAGE` 權限。==}|
|[Search Experiments](https://mlflow.org/docs/latest/rest-api.html#mlflowmlflowservicesearchexperiments)|2.0/mlflow/experiments/search|POST|僅傳回使用者具有 `READ` 權限的實驗。|
|[Search Experiments](https://mlflow.org/docs/latest/rest-api.html#mlflowmlflowservicesearchexperiments)|2.0/mlflow/experiments/search|GET|僅傳回使用者具有 `READ` 權限的實驗。|
|[Search Runs](https://mlflow.org/docs/latest/rest-api.html#mlflowmlflowservicesearchruns)|2.0/mlflow/runs/search|POST|僅傳回使用者具有 `READ` 權限的實驗。|
|[Search Registered Models](https://mlflow.org/docs/latest/rest-api.html#mlflowmodelregistryservicesearchregisteredmodels)|2.0/mlflow/registered-models/search|GET|僅傳回使用者具有 `READ` 權限的註冊模型。|
|[Search Model Versions](https://mlflow.org/docs/latest/rest-api.html#mlflowmodelregistryservicesearchmodelversions)|2.0/mlflow/model-versions/search|GET|僅傳回使用者具有 `READ` 權限的註冊模型。|


### 權限資料庫

所有使用者和權限都儲存在 `basic_auth.db` 中的資料庫中，相對於 MLflow 伺服器啟動的目錄。可以在設定檔中變更該位置。若要執行 db migration，請使用下列命令：

```bash
python -m mlflow.server.auth db upgrade --url <database_url>
```

!!! tip
    由於 MLflow 使用 [sqlalchemy](https://www.sqlalchemy.org/) 來存取資料庫, 因此 `<database_url>` 需要根據不同的後端資料庫來進行定義與安裝相關連接資料庫的套件。
    參考: [SqlAlchemy - Database URLs](https://docs.sqlalchemy.org/en/20/core/engines.html#database-urls) 文件

### Admin Users

Admin 使用者可以不受限制地存取所有 MLflow 資源，包括建立或刪除使用者、更新其他使用者的密碼和管理狀態、授予或撤銷其他使用者的權限以及管理所有 MLflow 資源的權限，即使為該管理員明確設定 NO_PERMISSIONS 也是如此帳戶。

MLflow 有一個內建管理員用戶，該用戶將在首次啟用 MLflow 身份驗證功能時建立。

!!! note
    建議您在建立後儘快更新預設管理員密碼。

預設管理員使用者憑證如下：

|Username|Password|
|--------|--------|
|admin|password|

透過使用 `2.0/mlflow/users/update-admin` 端點可將其他使用者提升為管理員，MLflow 可以存在多個管理員使用者。

**環境變數:**

```bash
# authenticate as built-in admin user
export MLFLOW_TRACKING_USERNAME=admin
export MLFLOW_TRACKING_PASSWORD=password
```

**程式範例:**

```python
from mlflow.server import get_app_client

tracking_uri = "http://localhost:5000/"

auth_client = get_app_client("basic-auth", tracking_uri=tracking_uri)
auth_client.create_user(username="user1", password="pw1")
auth_client.update_user_admin(username="user1", is_admin=True)
```

### 管理權限

MLflow 提供 REST API 和客戶端類別 `AuthServiceClient` 來管理使用者和權限。若要實例化 `AuthServiceClient`，建議您使用 `mlflow.server.get_app_client()`。

**環境變數:**

```bash
# authenticate as built-in admin user
export MLFLOW_TRACKING_USERNAME=admin
export MLFLOW_TRACKING_PASSWORD=password
```

**程式範例:**

```python
from mlflow import MlflowClient
from mlflow.server import get_app_client

tracking_uri = "http://localhost:5000/"

auth_client = get_app_client("basic-auth", tracking_uri=tracking_uri)
auth_client.create_user(username="user1", password="pw1")
auth_client.create_user(username="user2", password="pw2")

client = MlflowClient(tracking_uri=tracking_uri)
experiment_id = client.create_experiment(name="experiment")

auth_client.create_experiment_permission(
    experiment_id=experiment_id, username="user2", permission="MANAGE"
)
```

## 向 MLflow 進行身份驗證

### 使用 MLflow UI

當使用者首次在瀏覽器上造訪 MLflow UI 時，系統會提示他們登入。

{==目前，MLflow UI 不顯示有關目前使用者的任何資訊。用戶登入後，登出的唯一方法是關閉瀏覽器。==}

![](./assets/auth_prompt.png)

### 使用環境變數

MLflow 提供兩個用於驗證的環境變數：`MLFLOW_TRACKING_USERNAME` 和 `MLFLOW_TRACKING_PASSWORD`。若要使用 basic authentication 模組，您必須設定這兩個環境變數。

**環境變數:**

```bash
# authenticate as built-in admin user
export MLFLOW_TRACKING_USERNAME=admin
export MLFLOW_TRACKING_PASSWORD=password
```

**程式範例:**

```python
import mlflow

mlflow.set_tracking_uri("https://<mlflow_tracking_uri>/")
with mlflow.start_run():
    ...
```

### 使用憑證文件

您可以將憑證保存在文件中，從而無需每次都設定環境變數。憑證應使用 **INI** 格式儲存在 `~/.mlflow/credentials` 中。請注意，密碼將以未加密的方式儲存在磁碟上，並且僅受檔案系統權限的保護。

如果配置了環境變數 `MLFLOW_TRACKING_USERNAME` 和 `MLFLOW_TRACKING_PASSWORD`，它們將覆蓋憑證檔案中提供的任何憑證。

**憑證文件格式:**

```bash
[mlflow]
mlflow_tracking_username = username
mlflow_tracking_password = password
```

### 使用 REST API

使用者可以使用 HTTP Authorization 標頭進行身份驗證。請參閱 https://developer.mozilla.org/en-US/docs/Web/HTTP/Authentication 以了解更多資訊。

在 Python 中，您可以使用 requests 庫：

```python
import requests

response = requests.get(
    "https://<mlflow_tracking_uri>/",
    auth=("username", "password"),
)
```

## 建立新用戶

!!! info
    若要建立新用戶，您需要使用管理員權限進行身份驗證。


### 使用 MLflow UI

MLflow UI 提供了一個簡單的頁面，用於在 `<tracking_uri>/signup` 處建立新使用者。

![](./assets/auth_signup_form.png)


### 使用 REST API

或者，您可以將 POST 請求傳送到追蹤伺服器端點 `2.0/users/create`。

在 Python 中，您可以使用 requests 庫：

```python
import requests

response = requests.post(
    "https://<mlflow_tracking_uri>/api/2.0/mlflow/users/create",
    json={
        "username": "username",
        "password": "password",
    },
)
```

### 使用 MLflow AuthServiceClient

MLflow [AuthServiceClient](https://mlflow.org/docs/latest/auth/python-api.html#mlflow.server.auth.client.AuthServiceClient) 提供了輕鬆建立新使用者的功能。

```python
import mlflow

auth_client = mlflow.server.get_app_client(
    "basic-auth", tracking_uri="https://<mlflow_tracking_uri>/"
)

auth_client.create_user(username="username", password="password")
```

## 如何配置

MLflow 身份驗證模組的配置檔案位於 [mlflow/server/auth/basic_auth.ini](https://github.com/mlflow/mlflow/blob/master/mlflow/server/auth/basic_auth.ini)：

```ini title="basic_auth.ini"
[mlflow]
default_permission = READ
database_uri = sqlite:///basic_auth.db
admin_username = admin
admin_password = password
authorization_function = mlflow.server.auth:authenticate_request_basic_auth
```

| Variable | Description |
|----------|-------------|
|`default_permission`|所有資源的預設權限, 從 MLflow 源始設定檔可得知預設權限的設定是 **READ**。<br/>對於多團隊共同使用同一個 Tracking Service 的情境, 建議設定為 **NO_PERMISSIONS**。|
|`database_uri`|儲存權限和使用者資料的資料庫位置, 參考: [SqlAlchemy - Database URLs](https://docs.sqlalchemy.org/en/20/core/engines.html#database-urls)|
|`admin_username`|預設管理員使用者名稱|
|`admin_password`|預設管理員使用者密碼|
|`authorization_function`|驗證請求的配置函數|

!!! tip
    **PostgreSQL**
    PostgreSQL dialect 使用 `psycopg2` 作為預設 DBAPI。建議先進行下列套件的安裝:

    ```bash
    pip install psycopg2-binary
    ```

    連接字串範例: 
    
    ```bash
    # default
    engine = create_engine("postgresql://scott:tiger@localhost/mydatabase")

    # psycopg2
    engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/mydatabase")
    ```

    有關連接到 PostgreSQL 的更多說明，請參閱 [PostgreSQL](https://docs.sqlalchemy.org/en/20/dialects/postgresql.html)。

    **MySQL**
    MySQL dialect 使用 `mysqlclient` 作為預設 DBAPI。建議先進行下列套件的安裝:

    ```bash
    pip install mysqlclient
    ```

    連接字串範例: 

    ```
    # default
    engine = create_engine("mysql://scott:tiger@localhost/foo")

    # mysqlclient (a maintained fork of MySQL-Python)
    engine = create_engine("mysql+mysqldb://scott:tiger@localhost/foo")
    ```

或者，指派環境變數 **MLFLOW_AUTH_CONFIG_PATH** 以指向您的自訂設定檔。

如果您想要使用 HTTP basic auth 之外的其他的身份驗證方法，則 `authorization_function` 設定支援可插入客制化的身份驗證方法。該值的設定結構是 `module_name:function_name`。此函數具有以下 input/oput 參數與回傳結果：

```python
def authenticate_request() -> Union[Authorization, Response]:
    ...
```

如果請求經過驗證，函數應傳回 `werkzeug.datastructs.Authorization` 物件；如果請求未經過驗證，則該函數應傳回 Response 物件（通常為 `401：未經授權`）。有關如何實作自訂驗證方法的範例，請參閱 [tests/server/auth/jwt_auth.py](https://github.com/mlflow/mlflow/blob/master/tests/server/auth/test_auth.py)。注意：此範例不適用於生產用途。


## 連接到資料庫

預設情況下，MLflow 身份認證模組使用本機 SQLite 資料庫來儲存使用者和權限資料。在多節點部署的情況下，建議使用集中式資料庫來儲存這些資料。

若要連接到集中式資料庫，您可以將 `database_uri` 配置變數設定為資料庫 URL。


**Example: `/path/to/my_auth_config.ini`**

```
[mlflow]
database_uri = postgresql://username:password@hostname:port/database
```

然後，啟動 MLflow Tracking Service，{==並將 `MLFLOW_AUTH_CONFIG_PATH` 環境變數設定為設定檔的路徑==}。

```bash
export MLFLOW_AUTH_CONFIG_PATH=/path/to/my_auth_config.ini

mlflow server --app-name basic-auth
```

必須在啟動 MLflow Tracking Service 之前先建立好資料庫。資料庫 schema 則會在 Tracking Service 啟動時自動建立。

## 客制化身份認證

MLflow 身份驗證設計為可擴充的。如果您的組織需要更進階的身份驗證邏輯（例如基於 token 的身份驗證），可以安裝第三方外掛程式或建立您自己的外掛程式。

您的插件應該是可安裝的 Python 套件。它應該包括一個擴展 MLflow 應用程式的應用程式工廠，並且可以選擇實現一個客戶端來管理權限。應用程式工廠函數名稱將傳遞給 Flask CLI 中的 `--app` 參數。請參閱 [Flask application-discovery](https://flask.palletsprojects.com/en/latest/cli/#application-discovery) 以了解更多資訊。

**Example: my_auth/__init__.py**

```python
from flask import Flask
from mlflow.server import app


def create_app(app: Flask = app):
    app.add_url_rule(...)
    return app


class MyAuthClient:
    ...
```

然後，該插件應該安裝在您的 Python 環境中：

```bash
pip install my_auth
```

然後，在 `mlflow/setup.py` 中註冊您的外掛程式：

```python
setup(
    ...,
    entry_points="""
        ...

        [mlflow.app]
        my-auth=my_auth:create_app

        [mlflow.app.client]
        my-auth=my_auth:MyAuthClient
    """,
)
```

然後，您可以啟動 MLflow 伺服器：

```
mlflow server --app-name my-auth
```

