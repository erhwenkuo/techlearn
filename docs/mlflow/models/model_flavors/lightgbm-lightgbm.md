# LightGBM (lightgbm)

lightgbm 模型風格允許透過 `mlflow.lightgbm.save_model()` 和 `mlflow.lightgbm.log_model()` 方法以 MLflow 格式記錄 LightGBM 模型。這些方法也將 python_function 風格添加到它們產生的 MLflow 模型中，允許將模型解釋為通用 Python 函數，以便透過 `mlflow.pyfunc.load_model()` 進行推理。您也可以使用 `mlflow.lightgbm.load_model()` 方法載入具有本機 LightGBM 格式的 lightgbm 模型風格的 MLflow 模型。

## LightGBM pyfunc usage

下面的例子

- 從 scikit-learn 載入 IRIS 資料集
- 訓練 LightGBM LGBMClassifier
- 使用 mlflow 記錄模型和特徵重要性
- 載入記錄的模型並進行預測

```python
from lightgbm import LGBMClassifier
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
import mlflow
from mlflow.models import infer_signature

# 訓練用樣本數據
data = load_iris()

# Remove special characters from feature names to be able to use them as keys for mlflow metrics
feature_names = [
    name.replace(" ", "_").replace("(", "").replace(")", "")
    for name in data["feature_names"]
]

X_train, X_test, y_train, y_test = train_test_split(
    data["data"], data["target"], test_size=0.2
)

# 構建模型
lgb_classifier = LGBMClassifier(
    n_estimators=10,
    max_depth=3,
    learning_rate=1,
    objective="binary:logistic",
    random_state=123,
)

# 啟動 MLflow 對模型訓練過程的記錄
with mlflow.start_run():
    # 模型訓練
    lgb_classifier.fit(X_train, y_train)
    
    feature_importances = dict(zip(feature_names, lgb_classifier.feature_importances_))

    feature_importance_metrics = {
        f"feature_importance_{feature_name}": imp_value
        for feature_name, imp_value in feature_importances.items()
    }

    # 記錄 metric
    mlflow.log_metrics(feature_importance_metrics)

    # 推論模型簽章
    signature = infer_signature(X_train, lgb_classifier.predict(X_train))

    # 使用 MLflow 記錄模型資訊
    model_info = mlflow.lightgbm.log_model(
        lgb_classifier, "iris-classifier", signature=signature
    )

# 從 MLflow Tracking Service 載入模型
lgb_classifier_saved = mlflow.pyfunc.load_model(model_info.model_uri)

# 進行推論
y_pred = lgb_classifier_saved.predict(X_test)

print(y_pred)
```
