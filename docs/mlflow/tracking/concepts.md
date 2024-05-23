# Concepts

## Runs

MLflow Tracking 圍繞著 run 的概念進行組織，run 是某些資料科學程式碼的執行，例如單一 python train.py 執行。每次 run 都會記錄元資料（有關 run 的各種信息，例如指標、參數、開始和結束時間）和工件（run 的輸出文件，例如模型權重、圖像等）。

## Experiments

實驗將針對特定任務分組執行。您可以使用 CLI、API 或 UI 建立實驗。 MLflow API 和 UI 還允許您建立和搜尋實驗。有關如何將運行組織到實驗中的更多詳細信息，請參閱將 run 組織到實驗中。

## Tracking Runs

MLflow Tracking API 提供了一組用於追蹤您的 run 的功能。例如，您可以呼叫 `mlflow.start_run()` 開始新的 run，然後呼叫 Logging Functions（例如 `mlflow.log_param()` 和 `mlflow.log_metric()`）分別記錄參數和指標。請造訪追蹤 API 文檔，以了解有關使用這些 API 的更多詳細資訊。

```python
import mlflow

with mlflow.start_run():
    mlflow.log_param("lr", 0.001)
    # Your ml code
    ...
    mlflow.log_metric("val_loss", val_loss)
```

或者，Auto-logging 提供了用於啟動 MLflow 追蹤的超快速設定。這個強大的功能可讓您記錄指標、參數和模型，而無需明確的日誌語句 - 您所需要做的就是在訓練程式碼之前呼叫 `mlflow.autolog()` 。Auto-logging 支援流行的庫，例如 Scikit-learn、XGBoost、PyTorch、Keras、Spark 等。請參閱 [Automatic Logging Documentation](https://mlflow.org/docs/latest/tracking/autolog.html#automatic-logging)，以了解受支援的庫以及如何將自動日誌記錄 API 與每個庫一起使用。

```python
import mlflow

mlflow.autolog()

# Your training code...
```

!!! info
    預設情況下，無需任何特定的伺服器/資料庫配置，MLflow Tracking 會將資料記錄到本機 `mlruns` 目錄。如果您想要將運行記錄到其他位置（例如遠端資料庫和雲端儲存）以與團隊共用結果，請按照設定 MLflow 追蹤環境部分中的說明進行操作。

## Tracking Datasets

MLflow 提供了追蹤與模型訓練事件相關的資料集的能力。可以透過使用 `[mlflow.log_input()](https://mlflow.org/docs/latest/python_api/mlflow.html#mlflow.log_input)` API 來儲存與資料集關聯的這些元資料。要了解更多信息，請訪問 MLflow 資料文檔以查看此 API 中可用的功能。