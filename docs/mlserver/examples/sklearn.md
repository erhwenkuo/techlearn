# Scikit-Learn 模型推論服務

`mlserver` 開箱即用地支援 `scikit-learn` 模型的部署和服務。預設情況下，它會假設這些模型已使用 [joblib 進行序列化](https://scikit-learn.org/stable/modules/model_persistence.html)。

在此範例中，我們將介紹如何訓練和序列化一個簡單的模型，然後使用 `mlserver` 為其提供模型推論服務。

## 安裝相依套件

要使用 `mlserver` 開箱即用地支援 `scikit-learn` Runtime, 必需要先安裝下列套件:

```bash
pip install mlserver-sklearn
```

## 模型訓練

第一步是訓練一個簡單的 `scikit-learn` 模型。為此，我們將使用 [scikit-learn 文件中的 MNIST 範例](https://scikit-learn.org/stable/auto_examples/classification/plot_digits_classification.html)來訓練 SVM 模型。

```python
# Original source code and more details can be found in:
# https://scikit-learn.org/stable/auto_examples/classification/plot_digits_classification.html

# Import datasets, classifiers and performance metrics
from sklearn import datasets, svm, metrics
from sklearn.model_selection import train_test_split

# The digits dataset
digits = datasets.load_digits()

# To apply a classifier on this data, we need to flatten the image, to
# turn the data in a (samples, feature) matrix:
n_samples = len(digits.images)
data = digits.images.reshape((n_samples, -1))

# Create a classifier: a support vector classifier
classifier = svm.SVC(gamma=0.001)

# Split data into train and test subsets
X_train, X_test, y_train, y_test = train_test_split(
    data, digits.target, test_size=0.5, shuffle=False)

# We learn the digits on the first half of the digits
classifier.fit(X_train, y_train)

# Predict the value of the digit on the test subset
predicted = clf.predict(X_test)

print(
    f"Classification report for classifier {clf}:\n"
    f"{metrics.classification_report(y_test, predicted)}\n"
)
```

### 保存模型

為了保存經過訓練的模型，我們將使用 `joblib` 進行序列化。雖然這不是一個完美的方法，但它是目前 scikit-learn 文件中推薦的將模型儲存到磁碟的方法。

我們的模型將儲存為名為 `mnist-svm.joblib` 的文件

```python
import joblib

model_file_name = "mnist-svm.joblib"
joblib.dump(classifier, model_file_name)
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
    "name": "mnist-svm",
    "implementation": "mlserver_sklearn.SKLearnModel",
    "parameters": {
        "uri": "./mnist-svm.joblib",
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
    import requests

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

    endpoint = "http://localhost:8080/v2/models/mnist-svm/versions/v0.1.0/infer"
    response = requests.post(endpoint, json=inference_request)

    response.json()
    ```

=== "NumPy Request Codec"

    ```python
    from mlserver.codecs import NumpyRequestCodec
    import requests

    # Encode an entire V2 request
    inference_request = NumpyRequestCodec.encode_request(x_0)

    endpoint = "http://localhost:8080/v2/models/mnist-svm/versions/v0.1.0/infer"

    response = requests.post(endpoint, json=inference_request.model_dump())

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

### 解析推理請求結果

除了可使用 JSON 的格式來取得最後推論的結果之外, mlserver 也提供 Python 的一些類別可幫助解析結果:

```python
import numpy as np
from sklearn import datasets
from sklearn.model_selection import train_test_split
from mlserver.types import InferenceRequest
from mlserver.types import InferenceResponse
from mlserver.codecs import NumpyCodec
import requests

# 載入預處理好的數據集
digits = datasets.load_digits()

# 取得數據集的圖像數據(用 numpy ndarray 格式儲存)的樣本數
n_samples = len(digits.images)

# 根據 SVM 模型的 INPUT 格式來展平 tensor 的 shape
data = digits.images.reshape((n_samples, -1))

# 將資料拆分為訓練子集和測試子集
X_train, X_test, y_train, y_test = train_test_split(data, digits.target, test_size=0.5, shuffle=False)

# 取得五筆的測試數據
x_0_5 = X_test[0:5]
y_0_5 = y_test[0:5]

# 打印測試數據的 shape
print(f"Input Tensor Shape: {x_0_5.shape}")

# 使用 NumpyCodec 來編碼成 v2 Inference Protocol 的格式
inference_request = InferenceRequest(
  inputs=[
    NumpyCodec.encode_input("x_0_5", x_0_5)
  ]
)

# v2 Inference 端點
endpoint = "http://localhost:8080/v2/models/mnist-svm/versions/v0.1.0/infer"

# 呼叫推論服務
response = requests.post(endpoint, json=inference_request.model_dump())

# 重新構建成 InferenceResponse 物件
raw_response = response.json()

inference_response = InferenceResponse(**raw_response)

# 解析推論服務結果
for response in inference_response.outputs:
    result = NumpyCodec.decode_output(response)
    # 打印推論服務結果的 shape
    print(f"Output Tensor Shape: {result.shape}")
    # 重新 reshape 推論服務結果並且打印出來
    print(f'Inference result: {result.reshape((len(result),))}')
    # 對比真實答案
    print(f'Ground truth: {y_0_5}') 
```

結果:

```bash
Input Tensor Shape: (5, 64)
Output Tensor Shape: (5, 1)
Inference result: [8 8 4 9 0]
Ground truth: [8 8 4 9 0]
```

