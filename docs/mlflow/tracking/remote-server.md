# 使用 MLflow Tracking Server 進行遠端實驗追蹤

在本教學中，您將學習如何使用 MLflow Tracking Server 設定 MLflow Tracking 環境以進行團隊開發。

利用 MLflow Tracking Server 進行遠端實驗追蹤有很多好處：

- **Collaboration**: 多個使用者可以將運行記錄到同一端點，以及其他使用者記錄的查詢運行和模型。
- **Sharing Results**: 追蹤伺服器還提供追蹤 UI 端點，團隊成員可以輕鬆地探索彼此的結果。
- **Centralized Access**: 追蹤伺服器可以作為遠端存取元資料和工件的代理程式運行，從而更容易保護和審核對資料的存取。

## 它是如何運作的？

下圖描述了使用帶有 PostgreSQL 和 S3 的遠端 MLflow Tracking Server 的架構:

![](./assets/scenario_5.png)

當您開始將執行記錄到 MLflow Tracking Server 時，會發生以下情況：

- **Part 1a and 1b:**
    - MLflow 用戶端建立 `RestStore` 實例並傳送 REST API 請求以記錄 MLflow 實體
    - 追蹤伺服器建立 `SQLAlchemyStore` 的實例並連接到遠端主機以在資料庫中插入追蹤資訊（即指標、參數、標籤等）

- **Part 1c and 1d:**
    - 客戶端的檢索請求從配置的 `SQLAlchemyStore` 表傳回訊息

- **Part 2a and 2b:**
    - 要記錄 artifact 的事件是由客戶端使用 `HttpArtifactRepository` 將檔案寫入 MLflow 追蹤伺服器
    - 然後，追蹤伺服器使用假定的角色身份驗證將這些檔案寫入配置的物件儲存位置

- **Part 2c and 2d:**
    - 從配置的後端儲存中檢索使用者請求的 artifact 是使用伺服器啟動時配置的相同授權身份驗證完成的
    - Artifact 透過 `HttpArtifactRepository` 的介面透過追蹤伺服器傳遞給最終用戶

## 安裝

在實際的生產部署環境中，您將有多個遠端主機來運行追蹤伺服器和資料庫，如上圖所示。然而，就本教學而言，我們將僅使用一台機器，在不同連接埠上運行多個 Docker 容器，透過更簡單的評估教程設定來模仿遠端環境。我們還將使用 MinIO（與 S3 相容的物件儲存）作為工件存儲，這樣您就無需擁有 AWS 帳戶即可運行本教學。

### Step 1 - Get MLflow and additional dependencies

MLflow 可在 PyPI 上使用。使用 Python 存取 PostgreSQL 和 S3 還需要 `pyscopg2` 和 `boto3`。如果您的系統上尚未安裝它們，您可以使用以下命令安裝它們：

```bash
pip install mlflow psycopg2 boto3
```

### Step 2 - Set up remote data stores

MLflow Tracking Server 可以與各種資料儲存交互，以儲存實驗和運行資料以及工件。在本教學中，我們將使用 Docker Compose 啟動兩個容器，每個容器模擬實際環境中的遠端伺服器。

- PostgreSQL 資料庫作為後端儲存。
- MinIO 伺服器作為工件儲存。

**安裝 docker 和 docker-compose**

請依照官方說明安裝 [Docker](https://docs.docker.com/install/) 和 [Docker Compose](https://docs.docker.com/compose/install/)。然後，執行 `docker --version` 和 `docker-compose --version` 以確保它們安裝正確。

**創建 compose.yaml**

建立一個名為 `compose.yaml` 的文件，其中包含以下內容：

```yaml title="compose.yaml"
version: '3.7'
services:
  # PostgreSQL database
  postgres:
    image: postgres:latest
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mlflowdb
    ports:
      - 5432:5432
    volumes:
      - ./postgres-data:/var/lib/postgresql/data
  # MinIO server
  minio:
    image: minio/minio
    expose:
      - "9000"
    ports:
      - "9000:9000"
      # MinIO Console is available at http://localhost:9001
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: "minio_user"
      MINIO_ROOT_PASSWORD: "minio_password"
    healthcheck:
      test: timeout 5s bash -c ':> /dev/tcp/127.0.0.1/9000' || exit 1
      interval: 1s
      timeout: 10s
      retries: 5
    command: server /data --console-address ":9001"
  # Create a bucket named "bucket" if it doesn't exist
  minio-create-bucket:
    image: minio/mc
    depends_on:
      minio:
        condition: service_healthy
    entrypoint: >
      bash -c "
      mc alias set minio http://minio:9000 minio_user minio_password &&
      if ! mc ls minio | grep --quiet bucket; then
        mc mb minio/bucket
      else
        echo 'bucket already exists'
      fi
      "
```

**啟動容器**

從 `compose.yaml` 檔案所在的相同目錄執行以下命令來啟動容器。這將在後台啟動 PostgreSQL 和 Minio 伺服器的容器，並在 Minio 中建立一個名為 **bucket** 的新儲存桶。

```bash
docker compose up -d
```

### Step 3 - Start the Tracking Server

**配置存取權限**

為了讓追蹤伺服器存取遠端存儲，需要配置必要的憑證。

```bash
export MLFLOW_S3_ENDPOINT_URL=http://localhost:9000 # Replace this with remote storage endpoint e.g. s3://my-bucket in real use cases
export AWS_ACCESS_KEY_ID=minio_user
export AWS_SECRET_ACCESS_KEY=minio_password
```

您可以在支援的儲存體中找到有關如何配置其他儲存的憑證的說明。

**啟動追蹤伺服器**

若要指定後端儲存和工件存儲，您可以分別使用 `--backend-store-uri` 和 `--artifacts-store-uri` 選項。

```bash
mlflow server \
  --backend-store-uri postgresql://user:password@localhost:5432/mlflowdb \
  --artifacts-destination s3://bucket \
  --host 0.0.0.0 \
  --port 5000
```

將 localhost 替換為實際環境中資料庫伺服器的遠端主機名稱或 IP 位址。

### Step 4: Logging to the Tracking Server

追蹤伺服器運行後，您可以透過將 MLflow 追蹤 URI 設定為追蹤伺服器的 URI 來記錄運行情況。或者，您可以使用 `mlflow.set_tracking_uri()` API 設定來追蹤 URI。

```bash
export MLFLOW_TRACKING_URI=http://127.0.0.1:5000  # Replace with remote host name or IP address in an actual environment
```

然後像往常一樣使用 MLflow 追蹤 API 運行您的程式碼。以下程式碼在糖尿病資料集上執行 scikit-learn RandomForest 模型的訓練：


```python
import mlflow

from sklearn.model_selection import train_test_split
from sklearn.datasets import load_diabetes
from sklearn.ensemble import RandomForestRegressor

# 啟動 auto-log 功能
mlflow.sklearn.autolog()

# 載入資料集
db = load_diabetes()
X_train, X_test, y_train, y_test = train_test_split(db.data, db.target)

# 創建和訓練模型。
rf = RandomForestRegressor(n_estimators=100, max_depth=6, max_features=3)
rf.fit(X_train, y_train)

# 使用模型對測試資料集進行預測。
predictions = rf.predict(X_test)
```

### Step 5: View logged Run in Tracking UI

我們的偽遠端 MLflow 追蹤伺服器也會在同一端點上託管追蹤 UI。在具有遠端追蹤伺服器的實際部署環境中，也是如此。您可以在瀏覽器中輸入 `http://127.0.0.1:5000`（已取代為實際環境中的遠端主機名稱或IP位址）來存取UI。

### Step 6: Download artifacts

MLflow Tracking Server 也可作為工件存取的代理主機。透過代理 URI（例如 `runs:/`、`mlflow-artifacts:/`）啟用工件訪問，使用戶可以存取此位置，而無需管理直接存取的憑證或權限。

```python
import mlflow

run_id = "YOUR_RUN_ID"  # 您可以在追蹤 UI 中找到run ID
artifact_path = "model"

# 透過追蹤伺服器下載工件
mlflow_artifact_uri = f"runs://{run_id}/{artifact_path}"

local_path = mlflow.artifacts.download_artifacts(mlflow_artifact_uri)

# 載入模型
model = mlflow.sklearn.load_model(local_path)
```
