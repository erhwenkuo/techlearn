# Keras (keras)

keras 模型 flavor 支援記錄和載入 Keras 模型。它在 Python 和 R 用戶端中均可使用。在 R 中，您可以使用 `mlflow_save_model` 和 `mlflow_log_model` 來儲存或記錄模型。這些函數使用 Keras 函式庫的內建模型持久化函數將 Keras 模型序列化為 `HDF5` 檔案。您可以使用 R 中的 mlflow_load_model 函數將具有 keras 風格的 MLflow 模型載入為 Keras 模型物件。

## Keras pyfunc usage

對於最小順序模型，pyfunc `Predict()` 方法的範例配置是：

```python
import mlflow
import numpy as np
import pathlib
import shutil
from tensorflow import keras

# 啟動 autolog
mlflow.tensorflow.autolog()

# 訓練的數據
X = np.array([-2, -1, 0, 1, 2, 1]).reshape(-1, 1)
y = np.array([0, 0, 1, 1, 1, 0])

# 構建模型
model = keras.Sequential(
    [
        keras.Input(shape=(1,)),
        keras.layers.Dense(1, activation="sigmoid"),
    ]
)

# 編譯模型訓練
model.compile(loss="binary_crossentropy", optimizer="adam", metrics=["accuracy"])

# 模型訓練
model.fit(X, y, batch_size=3, epochs=5, validation_split=0.2)

# 定義MLflow 模型下載到本地的路徑
local_artifact_dir = "/tmp/mlflow/keras_model"
pathlib.Path(local_artifact_dir).mkdir(parents=True, exist_ok=True)

# 模型在 MLflow Tracking server 上的 uri
model_uri = f"runs:/{mlflow.last_active_run().info.run_id}/model"

# 加載模型
keras_pyfunc = mlflow.pyfunc.load_model(
    model_uri=model_uri, dst_path=local_artifact_dir
)

# 構建測試數據
data = np.array([-4, 1, 0, 10, -2, 1]).reshape(-1, 1)

# 進行推論
predictions = keras_pyfunc.predict(data)

# 清除相關的模型資料
shutil.rmtree(local_artifact_dir)
```
