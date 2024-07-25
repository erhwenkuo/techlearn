# MLflow 模型推論服務

`mlserver` 開箱即用，支援 MLflow 模型的部署和服務，具有以下功能：

- MLflow 模型 artifacts 載入。
- 支援 dataframe, dict-tensor 和 tensor 張量輸入。

## 安裝相依套件

要使用 `mlserver` 開箱即用地支援 `scikit-learn` Runtime, 必需要先安裝下列套件:

```bash
pip install mlserver-mlflow
```

# 模型訓練

第一步是訓練和序列化 MLflow 模型。為此，我們將使用 MLflow 文件中的[線性迴歸範例](https://www.mlflow.org/docs/latest/tutorials-and-examples/tutorial.html)。

```python title="train.py"
# %load src/train.py
# Original source code and more details can be found in:
# https://www.mlflow.org/docs/latest/tutorials-and-examples/tutorial.html

# The data set used in this example is from
# http://archive.ics.uci.edu/ml/datasets/Wine+Quality
# P. Cortez, A. Cerdeira, F. Almeida, T. Matos and J. Reis.
# Modeling wine preferences by data mining from physicochemical properties.
# In Decision Support Systems, Elsevier, 47(4):547-553, 2009.

import warnings
import sys

import pandas as pd
import numpy as np
from sklearn.metrics import mean_squared_error, mean_absolute_error, r2_score
from sklearn.model_selection import train_test_split
from sklearn.linear_model import ElasticNet
from urllib.parse import urlparse
import mlflow
import mlflow.sklearn
from mlflow.models.signature import infer_signature

import logging

logging.basicConfig(level=logging.WARN)
logger = logging.getLogger(__name__)


def eval_metrics(actual, pred):
    rmse = np.sqrt(mean_squared_error(actual, pred))
    mae = mean_absolute_error(actual, pred)
    r2 = r2_score(actual, pred)
    return rmse, mae, r2


warnings.filterwarnings("ignore")
np.random.seed(40)

# Read the wine-quality csv file from the URL
csv_url = (
    "http://archive.ics.uci.edu/ml"
    "/machine-learning-databases/wine-quality/winequality-red.csv"
)
try:
    data = pd.read_csv(csv_url, sep=";")
except Exception as e:
    logger.exception(
        "Unable to download training & test CSV, "
        "check your internet connection. Error: %s",
        e,
    )

# Split the data into training and test sets. (0.75, 0.25) split.
train, test = train_test_split(data)

# The predicted column is "quality" which is a scalar from [3, 9]
train_x = train.drop(["quality"], axis=1)
test_x = test.drop(["quality"], axis=1)
train_y = train[["quality"]]
test_y = test[["quality"]]

alpha = float(sys.argv[1]) if len(sys.argv) > 1 else 0.5
l1_ratio = float(sys.argv[2]) if len(sys.argv) > 2 else 0.5

with mlflow.start_run():
    lr = ElasticNet(alpha=alpha, l1_ratio=l1_ratio, random_state=42)
    lr.fit(train_x, train_y)

    predicted_qualities = lr.predict(test_x)

    (rmse, mae, r2) = eval_metrics(test_y, predicted_qualities)

    print("Elasticnet model (alpha=%f, l1_ratio=%f):" % (alpha, l1_ratio))
    print("  RMSE: %s" % rmse)
    print("  MAE: %s" % mae)
    print("  R2: %s" % r2)

    mlflow.log_param("alpha", alpha)
    mlflow.log_param("l1_ratio", l1_ratio)
    mlflow.log_metric("rmse", rmse)
    mlflow.log_metric("r2", r2)
    mlflow.log_metric("mae", mae)

    tracking_url_type_store = urlparse(mlflow.get_tracking_uri()).scheme
    model_signature = infer_signature(train_x, train_y)

    # Model registry does not work with file store
    if tracking_url_type_store != "file":

        # Register the model
        # There are other ways to use the Model Registry,
        # which depends on the use case,
        # please refer to the doc for more information:
        # https://mlflow.org/docs/latest/model-registry.html#api-workflow
        mlflow.sklearn.log_model(
            lr,
            "model",
            registered_model_name="ElasticnetWineModel",
            signature=model_signature,
        )
    else:
        mlflow.sklearn.log_model(lr, "model", signature=model_signature)
```

訓練腳本也將利用 MLflow 模型格式序列化我們訓練的模型。預設情況下，我們應該能夠在 `mlruns` 資料夾下找到已儲存的工件。

```python
import os

[experiment_file_path] = !ls -td ./mlruns/0/* | head -1

model_path = os.path.join(experiment_file_path, "artifacts", "model")

print(model_path)
```

## 模型推論

現在我們已經訓練並保存了模型，下一步將是使用 `mlserver` 為其提供模型推論服務。為此，我們需要建立 2 個設定檔：


- `settings.json`: 儲存我們伺服器的配置（例如連接埠、日誌等級等）。
- `model-settings.json`: 儲存模型的配置（例如輸入類型、使用的執行時間等）。

**settings.json**

```json title="settings.json"
{
    "debug": "true"
}
```

**model-settings.json**

```json title="model-settings.json"
{
    "name": "wine-classifier",
    "implementation": "mlserver_mlflow.MLflowRuntime",
    "parameters": {
        "uri": "runs:/<run_id>/model"
    }
}
```

### 開始模型推論服務

現在我們已經有了配置，我們可以透過執行 `mlserver start` 來啟動伺服器。

```bash
mlserver start .
```

由於此命令將啟動伺服器並封鎖終端，等待請求，因此需要在單獨的終端機上在背景執行。

### 發送推理請求

現在我們的模型由 mlserver 提供服務。為了確保一切按預期工作，讓我們從測試數據集發送請求。

為此，我們可以使用 mlserver 提供的開箱即用的 Python 類型，也可以手動建立請求。

=== "JSON payload"
    ```python
    import requests

    inference_request = {
        "inputs": [
            {
            "name": "fixed acidity",
            "shape": [1],
            "datatype": "FP32",
            "data": [7.4],
            },
            {
            "name": "volatile acidity",
            "shape": [1],
            "datatype": "FP32",
            "data": [0.7000],
            },
            {
            "name": "citric acid",
            "shape": [1],
            "datatype": "FP32",
            "data": [0],
            },
            {
            "name": "residual sugar",
            "shape": [1],
            "datatype": "FP32",
            "data": [1.9],
            },
            {
            "name": "chlorides",
            "shape": [1],
            "datatype": "FP32",
            "data": [0.076],
            },
            {
            "name": "free sulfur dioxide",
            "shape": [1],
            "datatype": "FP32",
            "data": [11],
            },
            {
            "name": "total sulfur dioxide",
            "shape": [1],
            "datatype": "FP32",
            "data": [34],
            },
            {
            "name": "density",
            "shape": [1],
            "datatype": "FP32",
            "data": [0.9978],
            },
            {
            "name": "pH",
            "shape": [1],
            "datatype": "FP32",
            "data": [3.51],
            },
            {
            "name": "sulphates",
            "shape": [1],
            "datatype": "FP32",
            "data": [0.56],
            },
            {
            "name": "alcohol",
            "shape": [1],
            "datatype": "FP32",
            "data": [9.4],
            },
        ]
    }

    endpoint = "http://localhost:8080/v2/models/wine-classifier/infer"
    response = requests.post(endpoint, json=inference_request)

    print(response.json())
    ```

=== "NumPy Input Codec"

    ```python
    from mlserver.types import InferenceRequest
    from mlserver.codecs import NumpyCodec
    import requests

    x_0 = X_test[0:1]

    # We can use the `NumpyCodec` to encode a single input head with name `foo`
    # within a larger request
    inference_request = InferenceRequest(
    inputs=[
        NumpyCodec.encode_input("x_0", x_0)
    ]
    )

    endpoint = "http://localhost:8080/v2/models/mnist-svm/versions/v0.1.0/infer"

    response = requests.post(endpoint, json=inference_request.model_dump())

    print(response.json())
    ```

結果:

```json
{
  "model_name": "mnist-svm",
  "model_version": "v0.1.0",
  "id": "4c965aea-36ec-435a-8f57-0a66f202ef4d",
  "parameters": {},
  "outputs": [
    {
      "name": "predict",
      "shape": [
        1,
        1
      ],
      "datatype": "INT64",
      "parameters": {
        "content_type": "np"
      },
      "data": [
        8
      ]
    }
  ]
}
```

正如我們在上面看到的，模型將輸入預測為數字 `8`，這與測試集上的數字(target)相符。