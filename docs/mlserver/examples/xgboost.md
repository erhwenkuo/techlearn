# XGBoost 模型推論服務

`mlserver` 開箱即用地支援 `xgboost` 模型的部署和服務。預設情況下，它會假設這些模型已使用 [bst.save_model() method 進行序列化](https://xgboost.readthedocs.io/en/latest/tutorials/saving_model.html)。

在此範例中，我們將介紹如何訓練和序列化一個簡單的模型，然後使用 `mlserver` 為其提供模型推論服務。

## 安裝相依套件

要使用 `mlserver` 開箱即用地支援 `xgboost` Runtime, 必需要先安裝下列套件:

```bash
pip install mlserver-xgboost
```

## 模型訓練

第一步是訓練一個簡單的 `xgboost` 模型。為此，我們將使用 [xgboost 入門指南中的蘑菇範例](https://xgboost.readthedocs.io/en/latest/get_started.html#python)。

```python title="train.py"
# Original code and extra details can be found in:
# https://xgboost.readthedocs.io/en/latest/get_started.html#python

import os
import xgboost as xgb
import requests

from urllib.parse import urlparse
from sklearn.datasets import load_svmlight_file


TRAIN_DATASET_URL = 'https://raw.githubusercontent.com/dmlc/xgboost/master/demo/data/agaricus.txt.train'
TEST_DATASET_URL = 'https://raw.githubusercontent.com/dmlc/xgboost/master/demo/data/agaricus.txt.test'


def _download_file(url: str) -> str:
    parsed = urlparse(url)
    file_name = os.path.basename(parsed.path)
    file_path = os.path.join(os.getcwd(), file_name)
    
    res = requests.get(url)
    
    with open(file_path, 'wb') as file:
        file.write(res.content)
    
    return file_path

train_dataset_path = _download_file(TRAIN_DATASET_URL)
test_dataset_path = _download_file(TEST_DATASET_URL)

# NOTE: Workaround to load SVMLight files from the XGBoost example
X_train, y_train = load_svmlight_file(train_dataset_path)
X_test, y_test = load_svmlight_file(test_dataset_path)
X_train = X_train.toarray()
X_test = X_test.toarray()

# read in data
dtrain = xgb.DMatrix(data=X_train, label=y_train)

# specify parameters via map
param = {'max_depth':2, 'eta':1, 'objective':'binary:logistic' }
num_round = 2

bst = xgb.train(param, dtrain, num_round)
```

### 保存模型

為了保存經過訓練的模型，我們將使用 `bst.save_model()` 進行序列化。這是 XGBoost 專案推薦的方法。

我們的模型將儲存為名為 `mushroom-xgboost.json` 的文件

```python title="train.py"
model_file_name = 'mushroom-xgboost.json'

bst.save_model(model_file_name)
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
    "name": "mushroom-xgboost",
    "implementation": "mlserver_xgboost.XGBoostModel",
    "parameters": {
        "uri": "./mushroom-xgboost.json",
        "version": "v0.1.0"
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
    x_0 = X_test[0:1]

    inference_request = {
        "inputs": [
            {
                "name": "predict",
                "shape": x_0.shape,
                "datatype": "FP32",
                "data": x_0.tolist()
            }
        ]
    }

    endpoint = "http://localhost:8080/v2/models/mushroom-xgboost/versions/v0.1.0/infer"

    response = requests.post(endpoint, json=inference_request)

    print(response.json())
    ```

=== "NumPy Request Codec"

    ```python
    from mlserver.codecs import NumpyRequestCodec

    x_0 = X_test[0:1]

    # Encode an entire V2 request
    inference_request = NumpyRequestCodec.encode_request(x_0)

    endpoint = "http://localhost:8080/v2/models/mushroom-xgboost/versions/v0.1.0/infer"

    response = requests.post(endpoint, json=inference_request.model_dump())

    print(response.json())
    ```

=== "NumPy Input Codec"

    ```python
    from mlserver.types import InferenceRequest
    from mlserver.codecs import NumpyCodec

    x_0 = X_test[0:1]

    # Encode an entire V2 request
    inference_request = InferenceRequest(
        inputs=[
            NumpyCodec.encode_input("x_0", x_0)
        ]
    )

    endpoint = "http://localhost:8080/v2/models/mushroom-xgboost/versions/v0.1.0/infer"

    response = requests.post(endpoint, json=inference_request.model_dump())

    print(response.json())
    ```

結果:

```json
{
  "model_name": "mushroom-xgboost",
  "model_version": "v0.1.0",
  "id": "69ba8342-f7fe-4d7e-a483-1ab252495793",
  "parameters": {},
  "outputs": [
    {
      "name": "predict",
      "shape": [
        1,
        1
      ],
      "datatype": "FP32",
      "parameters": {
        "content_type": "np"
      },
      "data": [
        0.285208523273468
      ]
    }
  ]
}
```

正如我們在下面看到的，模型預測輸入接近 `0`，這與測試數據集上的結果相符。

```python
print(y_test[0])
```

結果:

```
0
```