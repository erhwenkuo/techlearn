# XGBoost (xgboost)

xgboost 模型風格允許分別透過 python 中的 `mlflow.xgboost.save_model()` 和 `mlflow.xgboost.log_model()` 方法以及 R 中的 `mlflow_save_model` 和 `mlflow_log_model` 以 MLflow 格式記錄 `XGmlflow_save_model` 和 `mlflow_log_model` 以 MLflow 格式記錄 XGmlflow 模型。這些方法也將 python_function 風格添加到它們產生的 MLflow 模型中，允許將模型解釋為通用 Python 函數，以便透過 `mlflow.pyfunc.load_model()` 進行推理。

此載入的 PyFunc 模型只能使用 DataFrame 輸入進行評分。您也可以使用 `mlflow.xgboost.load_model()` 方法以本機 XGBoost 格式載入具有 xgboost 模型風格的 MLflow 模型。

請注意，xgboost 模型風格僅支援 xgboost.Booster 的實例，而不支援實作 scikit-learn API 的模型。

## XGBoost pyfunc usage

下面的例子

- 從 scikit-learn 載入 IRIS 資料集
- 訓練 XGBoost 分類器
- 使用 mlflow 記錄模型和參數
- 載入記錄的模型並進行預測

```python
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from xgboost import XGBClassifier
import mlflow
from mlflow.models import infer_signature

# 訓練用樣本數據
data = load_iris()
X_train, X_test, y_train, y_test = train_test_split(
    data["data"], data["target"], test_size=0.2
)

# 構建模型
xgb_classifier = XGBClassifier(
    n_estimators=10,
    max_depth=3,
    learning_rate=1,
    objective="binary:logistic",
    random_state=123,
)

# 啟動 MLflow 對模型訓練過程的記錄
with mlflow.start_run():
    # 模型訓練
    xgb_classifier.fit(X_train, y_train)

    # 取得參數
    clf_params = xgb_classifier.get_xgb_params()

    # 記錄參數
    mlflow.log_params(clf_params)

    # 推論模型簽章
    signature = infer_signature(X_train, xgb_classifier.predict(X_train))

    # 使用 MLflow 記錄模型資訊
    model_info = mlflow.xgboost.log_model(
        xgb_classifier, "iris-classifier", signature=signature
    )

# 從 MLflow Tracking Service 載入模型
xgb_classifier_saved = mlflow.pyfunc.load_model(model_info.model_uri)

# 進行推論
y_pred = xgb_classifier_saved.predict(X_test)
```
