# 使用來自 URI 的已保存模型部署 InferenceService

本文檔指導如何通過通過 `http` 或 `https` 端點公開的模型對象的 URI（統一資源標識符）指定模型對象。

此 `storageUri` 選項支持單個文件模型，例如由 `joblib` 文件指定的 `sklearn`，或包含其他模型類型（例如 `tensorflow` 或 `pytorch`）的所有必要依賴項的工件（例如 `tar` 或 `zip`）。在這裡，我們將展示上述兩者的示例。

## 創建 HTTP/HTTPS 標頭 Secret 並附加到服務帳戶

HTTP/HTTPS 服務請求標頭可以定義為 Secret 並附加到 Service Account。這是 optional 的。

```yaml title="create-uri-secret.yaml"
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  https-host: ZXhhbXBsZS5jb20=
  headers: |-
    ewoiYWNjb3VudC1uYW1lIjogInNvbWVfYWNjb3VudF9uYW1lIiwKInNlY3JldC1rZXkiOiAic29tZV9zZWNyZXRfa2V5Igp9
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa
secrets:
  - name: mysecret
```

執行下列命令來套用

```bash
kubectl apply -f create-uri-secret.yaml
```

!!! info
    在推理服務的預測器中指定的 serviceAccountName。這些標頭將應用於具有相同主機的任何 http/https 請求。

header 和 host 應該是 base64 編碼格式。

```bash
example.com
# echo -n "example.com" | base64
ZXhhbXBsZS5jb20=
---
{
  "account-name": "some_account_name",
  "secret-key": "some_secret_key"
}
# echo -n '{\n"account-name": "some_account_name",\n"secret-key": "some_secret_key"\n}' | base64
ewoiYWNjb3VudC1uYW1lIjogInNvbWVfYWNjb3VudF9uYW1lIiwKInNlY3JldC1rZXkiOiAic29tZV9zZWNyZXRfa2V5Igp9
```

## Sklearn

### 訓練並凍結模型

在這裡，我們將訓練一個簡單的 iris 模型。請注意，KServe 需要 `sklearn==0.20.3`。

```python
from sklearn import svm
from sklearn import datasets
import joblib

def train(X, y):
    clf = svm.SVC(gamma='auto')
    clf.fit(X, y)
    return clf

def freeze(clf, path='../frozen'):
    joblib.dump(clf, f'{path}/model.joblib')
    return True

if __name__ == '__main__':
    iris = datasets.load_iris()
    X, y = iris.data, iris.target
    clf = train(X, y)
    freeze(clf)
```

現在，可以將凍結的模型對象放在 Web 上的某個位置以公開它。例如，將 `model.joblib` 文件推送到 GitHub 上的某個存儲庫。

### 指定並創建 InferenceService

```yaml title="sklearn-from-uri.yaml"
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: sklearn-from-uri
spec:
  predictor:
    model:
      modelFormat:
        name: sklearn
      storageUri: https://github.com/tduffy000/kfserving-uri-examples/blob/master/sklearn/frozen/model.joblib?raw=true
```

執行下列命令來套用

```bash
kubectl apply -f sklearn-from-uri.yaml
```

### 運行模型推論

現在，可以通過 `${INGRESS_HOST}:${INGRESS_PORT}` 訪問入口，或按照此[說明](https://kserve.github.io/website/master/get_started/first_isvc/#4-determine-the-ingress-ip-and-ports)查找入口 IP 和端口。

下面的示例有效負載：

```json
{
  "instances": [
    [6.8,  2.8,  4.8,  1.4],
    [6.0,  3.4,  4.5,  1.6]
  ]
}
```

執行下列命令:

```bash
SERVICE_HOSTNAME=$(kubectl get inferenceservice sklearn-from-uri -o jsonpath='{.status.url}' | cut -d "/" -f 3)

MODEL_NAME=sklearn-from-uri
INPUT_PATH=@./input.json
curl -v -H "Host: ${SERVICE_HOSTNAME}" http://${INGRESS_HOST}:${INGRESS_PORT}/v1/models/$MODEL_NAME:predict -d $INPUT_PATH
```

結果:

```bash
$ *   Trying 127.0.0.1:8080...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8080 (#0)
> POST /v1/models/sklearn-from-uri:predict HTTP/1.1
> Host: sklearn-from-uri.default.example.com
> User-Agent: curl/7.68.0
> Accept: */*
> Content-Length: 76
> Content-Type: application/x-www-form-urlencoded
>
* upload completely sent off: 76 out of 76 bytes
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< content-length: 23
< content-type: application/json; charset=UTF-8
< date: Mon, 06 Sep 2021 15:52:55 GMT
< server: istio-envoy
< x-envoy-upstream-service-time: 7
<
* Connection #0 to host localhost left intact
{"predictions": [1, 1]}
```

## Tensorflow

這將作為一個示例，說明還可以拉入包含所有必需模型依賴項的壓縮包的能力，例如，tensorflow 需要嚴格目錄結構中的多個文件才能提供服務。

### 訓練並凍結模型

```python
from sklearn import datasets
import numpy as np
import tensorflow as tf

def _ohe(targets):
    y = np.zeros((150, 3))
    for i, label in enumerate(targets):
        y[i, label] = 1.0
    return y

def train(X, y, epochs, batch_size=16):
    model = tf.keras.Sequential([
        tf.keras.layers.InputLayer(input_shape=(4,)),
        tf.keras.layers.Dense(16, activation=tf.nn.relu),
        tf.keras.layers.Dense(16, activation=tf.nn.relu),
        tf.keras.layers.Dense(3, activation='softmax')
    ])
    model.compile(tf.keras.optimizers.RMSprop(learning_rate=0.001), loss='categorical_crossentropy', metrics=['accuracy'])
    model.fit(X, y, epochs=epochs)
    return model

def freeze(model, path='../frozen'):
    model.save(f'{path}/0001')
    return True

if __name__ == '__main__':
    iris = datasets.load_iris()
    X, targets = iris.data, iris.target
    y = _ohe(targets)
    model = train(X, y, epochs=50)
    freeze(model)
```

模型訓練完成後要導出模型的手法與 Sklearn 有些不同。我們不需要將凍結的輸出直接推送到某個 URI，而是需要將它們打包到一個 `tarball` 中。

```bash
cd ../frozen
tar -cvf artifacts.tar 0001/
gzip < artifacts.tar > artifacts.tgz
```

我們假設 `0001/` 目錄具有以下結構：

```bash
|-- 0001/
|-- saved_model.pb
|-- variables/
|--- variables.data-00000-of-00001
|--- variables.index
```

!!! tip
    tensorflow 需要從指定一個版本號的目錄來構建 tarball。


### 指定並創建 InferenceService

接下來 Kserve 應該能夠拉取 tarball 打包的模型檔案並公開模型推論端點。

```yaml title="tensorflow-from-uri-gzip.yaml"
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: tensorflow-from-uri-gzip
spec:
  predictor:
    tensorflow:
      storageUri: https://raw.githubusercontent.com/tduffy000/kfserving-uri-examples/master/tensorflow/frozen/model_artifacts.tar.gz
```

執行下列命令來套用

```bash
kubectl apply -f tensorflow-from-uri-gzip.yaml
```

### 運行模型推論

現在，可以通過 `${INGRESS_HOST}:${INGRESS_PORT}` 訪問入口，或按照此[說明](https://kserve.github.io/website/master/get_started/first_isvc/#4-determine-the-ingress-ip-and-ports)查找入口 IP 和端口。

下面的示例有效負載：

```json
{
  "instances": [
    [6.8,  2.8,  4.8,  1.4],
    [6.0,  3.4,  4.5,  1.6]
  ]
}
```

執行下列命令:

```bash
SERVICE_HOSTNAME=$(kubectl get inferenceservice tensorflow-from-uri-gzip -o jsonpath='{.status.url}' | cut -d "/" -f 3)

MODEL_NAME=tensorflow-from-uri-gzip
INPUT_PATH=@./input.json
curl -v -H "Host: ${SERVICE_HOSTNAME}" http://${INGRESS_HOST}:${INGRESS_PORT}/v1/models/$MODEL_NAME:predict -d $INPUT_PATH
```

結果:

```
$ *   Trying 10.0.1.16...
* TCP_NODELAY set
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Connected to 10.0.1.16 (10.0.1.16) port 30749 (#0)
> POST /v1/models/tensorflow-from-uri:predict HTTP/1.1
> Host: tensorflow-from-uri.default.example.com
> User-Agent: curl/7.58.0
> Accept: */*
> Content-Length: 86
> Content-Type: application/x-www-form-urlencoded
>
} [86 bytes data]
* upload completely sent off: 86 out of 86 bytes
< HTTP/1.1 200 OK
< content-length: 112
< content-type: application/json
< date: Thu, 06 Aug 2020 23:21:19 GMT
< x-envoy-upstream-service-time: 151
< server: istio-envoy
<
{ [112 bytes data]
100   198  100   112  100    86    722    554 --:--:-- --:--:-- --:--:--  1285
* Connection #0 to host 10.0.1.16 left intact
{
  "predictions": [
    [
      0.0204100646,
      0.680984616,
      0.298605353
    ],
    [
      0.0296604875,
      0.658412039,
      0.311927497
    ]
  ]
}
```