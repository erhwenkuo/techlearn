# Set up the MLflow Tracking Environment

MLflow Tracking 支援您的開發工作流程的許多不同場景。本部分將引導您了解如何為您的特定用例設定 MLflow Tracking 環境。從鳥瞰角度來看，MLflow Tracking 環境由以下組件組成。

## 元件

### [MLflow Tracking APIs](https://mlflow.org/docs/latest/tracking/tracking-api.html)

您可以在 ML 程式碼中呼叫 MLflow Tracking API 來記錄運行並在必要時與 MLflow Tracking Server 進行通訊。

### [Backend Store](https://mlflow.org/docs/latest/tracking/backend-stores.html)

後端儲存保存每次 run 的各種元數據，例如 run ID、開始和結束時間、參數、指標等。MLflow 支援兩種類型的後端儲存：基於 **file-system-based**（如本機檔案）和 **database-based**（如 PostgreSQL）。

### [Artifact Store](https://mlflow.org/docs/latest/tracking/artifacts-stores.html)

工件儲存會在每次運行時保留（通常很大）工件，例如模型權重（例如醃製的 scikit-learn 模型）、圖像（例如 PNG）、模型和資料檔案（例如 Parquet 檔案）。 MLflow 預設將工件儲存在本機檔案 (`mlruns`) 中，但也支援不同的儲存選項，例如 Amazon S3 和 Azure Blob 儲存體。

### [MLflow Tracking Server](https://mlflow.org/docs/latest/tracking/server.html)

MLflow Tracking Server 是一個獨立的 HTTP 伺服器，提供用於存取後端和/或工件儲存的 REST API。追蹤伺服器還可以靈活地設定要伺服器的資料、管理存取控制、版本控制等。

## 常見設定

透過正確配置這些元件，您可以建立適合您團隊的開發工作流程的 MLflow Tracking 環境。下圖和表格顯示了 MLflow Tracking 環境的一些常見設定。

![](./assets/tracking-setup-overview.png)

||1. Localhost (default)|2. Local Tracking with Local Database|3. Remote Tracking with [MLflow Tracking Server](https://mlflow.org/docs/latest/tracking.html#tracking-server)|
|--------|----------------------|-------------------------------------|-----------------------------------------------------|
|Scenario|Solo development|Solo development|Team development|
|Use Case|預設情況下，MLflow 將每次執行的元資料和工件記錄到本機目錄 `mlruns`。這是開始使用 MLflow Tracking 的最簡單方法，無需設定任何外部伺服器、資料庫和儲存。|MLflow 用戶端可以與後端的 SQLAlchemy 相容資料庫（例如 SQLite、PostgreSQL、MySQL）連接。將元數據儲存到資料庫可以讓您更清晰地管理實驗數據，同時跳過設定伺服器的工作。|MLflow Tracking Server 可以配置工件 HTTP 代理，透過追蹤伺服器傳遞工件請求以儲存和檢索工件，而無需與底層物件儲存服務互動。這對於您想要在具有適當存取控制的共用位置儲存工件和實驗元資料的團隊開發場景特別有用。|
|Tutorial|[QuickStart](https://mlflow.org/docs/latest/getting-started/intro-quickstart/index.html)|[Tracking Experiments with a Local Database](https://mlflow.org/docs/latest/tracking/tutorials/local-database.html)|[Remote Experiment Tracking with MLflow Tracking Server](https://mlflow.org/docs/latest/tracking/tutorials/remote-server.html)|