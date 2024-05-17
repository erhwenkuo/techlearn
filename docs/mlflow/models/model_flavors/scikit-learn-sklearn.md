# Scikit-learn (sklearn)

sklearn 模型 flavor 提供了一個易於使用的介面，用於保存和載入 scikit-learn 模型。 `mlflow.sklearn` 模組定義了 `save_model()` 和 `log_model()` 函數，它們以 MLflow 格式儲存 scikit-learn 模型，使用 Python 的 pickle 模組 (Pickle) 或 CloudPickle 進行模型序列化。這些函數產生具有 python_function 風格的 MLflow 模型，允許它們作為通用 Python 函數加載，以便透過 `mlflow.pyfunc.load_model()` 進行推理。此載入的 PyFunc 模型只能使用 `DataFrame` 輸入進行評分。最後，您可以使用 `mlflow.sklearn.load_model()` 方法將具有 sklearn 風格的 MLflow 模型作為 scikit-learn 模型物件載入。

## Scikit-learn pyfunc usage

對於 Scikit-learn LogisticRegression 模型，pyfunc `Predict()` 方法的範例配置是：

```python
import mlflow
from mlflow.models import infer_signature
import numpy as np
from sklearn.linear_model import LogisticRegression

# 啟動 MLflow 對模型訓練過程的記錄
with mlflow.start_run():
    # 訓練用樣本數據
    X = np.array([-2, -1, 0, 1, 2, 1]).reshape(-1, 1)
    y = np.array([0, 0, 1, 1, 1, 0])

    # 選擇演算法
    lr = LogisticRegression()

    # 模型訓練
    lr.fit(X, y)

    # 推導出模型的 input / output
    signature = infer_signature(X, lr.predict(X))

    # 使用 MLflow 記錄模型資訊
    model_info = mlflow.sklearn.log_model(
        sk_model=lr, artifact_path="model", signature=signature
    )

# 從 MLflow Tracking Service 載入模型
sklearn_pyfunc = mlflow.pyfunc.load_model(model_uri=model_info.model_uri)

# 準備推論數據
data = np.array([-4, 1, 0, 10, -2, 1]).reshape(-1, 1)

# 進行推論
predictions = sklearn_pyfunc.predict(data)
```

有關更多信息，請參閱 [mlflow.sklearn](https://mlflow.org/docs/latest/python_api/mlflow.sklearn.html#module-mlflow.sklearn)。
