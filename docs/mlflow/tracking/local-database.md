# 使用本地資料庫追蹤實驗

在本教程中，您將學習如何使用本機資料庫透過 MLflow 追蹤實驗元資料。預設情況下，MLflow Tracking 將運行資料記錄到本地文件，這可能會由於破碎的小文件和缺乏簡單的存取介面而導致一些挫折感。此外，如果您使用 Python，則可以使用在本機檔案系統（例如 `mlruns.db`）上執行並具有內建客戶端 sqlite3 的 SQLite，從而無需安裝任何其他依賴項和設定資料庫伺服器。

## Step 1 - Get MLflow

MLflow 可在 PyPI 上使用。如果您尚未在本機上安裝它，您可以使用以下命令安裝它：

```bash
pip install mlflow
```

## Step 2 - Configure MLflow environment varialbles

**將追蹤 URI 設定為本機 SQLite 資料庫:**

若要將 MLflow 指向本機 SQLite 資料庫，您需要將環境變數 `MLFLOW_TRACKING_URI` 設定為 `sqlite:///mlruns.db`。(這將在目前目錄中建立一個名為 `mlruns.db` 的 SQLite 資料庫檔案。如果要將資料庫檔案儲存在不同位置，請指定不同的路徑。)

```bash
export MLFLOW_TRACKING_URI=sqlite:///mlruns.db
```

如果您使用的是 Notebook，則可以執行以下：

```bash
%env MLFLOW_TRACKING_URI=sqlite:///mlruns.db
```

!!! info
    對於使用 SQLite 資料庫，MLflow 會自動建立一個新資料庫（如果不存在）。如果要使用不同的資料庫，需要先建立資料庫。

## Step 3: Start logging

現在您已準備好開始記錄實驗運行。例如，以下程式碼在糖尿病資料集上執行 scikit-learn RandomForest 模型的訓練：

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

## Step 4: View your logged Run in the Tracking UI

訓練作業完成後，您可以執行下列命令來啟動 MLflow UI（您必須使用 `--backend-store-uri` 選項指定 SQLite 資料庫檔案的路徑）:

```bash
mlflow ui --port 8080 --backend-store-uri sqlite:///mlruns.db
```

然後，在瀏覽器中導航至 `http://localhost:8080` 以查看結果。