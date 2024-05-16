# Deploy Scikit-learn models with InferenceService

## Scikit-learn 介詔

Scikit-learn（曾叫做 scikits.learn 與 sklearn）是用於 Python 程式語言的自由軟體機器學習庫。它包含了各種分類、回歸和聚類算法，包括多層感知器、支持向量機、隨機森林、梯度提升、k-平均聚類和 DBSCAN，它被設計協同於 Python 數值庫 NumPy 和和科學庫 SciPy。

Scikit-learn 主要用 Python 編寫的，並廣泛使用 NumPy 進行高性能線性代數和數組運算。此外，一些核心算法用 Cython 書寫來以提高性能。在某些情況下，用 Python 擴展出特定方法是不可能的；比如支持向量機，是通過用 Cython 包裝 LIBSVM 實現；邏輯斯諦回歸和線性支持向量機，是通過對 LIBLINEAR 的類似的包裝實現的。

Scikit-learn 與很多其他 Python 庫可以良好的集成起來，比如用於繪圖的 matplotlib 和 plotly，用於陣列向量化的 NumPy，用於dataframe 的 pandas，用於科學計算的 SciPy 等等。

## 訓練 Scikit-learn 模型

**套件安裝:**

```bash
pip install scikit-learn
pip install joblib
```

要測試 Scikit-learn model serving，首先需要使用以下 python 程式碼訓練一個簡單的 Scikit-learn 模型。

```python
from sklearn import svm
from sklearn.datasets import load_iris
import joblib
import numpy
import os

model_dir = "."
MODEL_FILE = "model.joblib"

# 載入 iris 資料集
iris = load_iris()
X = iris['data']
y = iris['target']

# 選擇一個分類的演算法并配置超參數
clf = svm.SVC(gamma='scale')

# 進行模型訓練
clf.fit(X, y)

# 取得 input 數據
input = numpy.asarray([[3, 5, 4, 2], [5, 4, 3, 2]])

print(f'範例 input: {input}')
print(f'Data type: {type(input)}')
print(f'Data Shape: {input.shape}')

# 進行模型推論
output = clf.predict(input)

print(f'範例 output: {output}')
print(f'Data type: {type(output)}')
print(f'Data Shape: {output.shape}')

# 定義模型導出的路徑
model_file = os.path.join(model_dir, MODEL_FILE)

# scikit-learn 模型檔的導出
joblib.dump(clf, filename=model_file)
```

結果:

```bash
範例 input: [[3 5 4 2]
 [5 4 3 2]]
Data type: <class 'numpy.ndarray'>
Data Shape: (2, 4)
範例 output: [1 1]
Data type: <class 'numpy.ndarray'>
Data Shape: (2,)
```


在專案目錄下也會看到導出的模型檔案 `model.joblib`。

## 使用 v2 協定來佈署 Scikit-learn 模型

若要使用開放推理協定部署 Scikit-learn 模型，您需要將 protocolVersion 欄位設為 v2。

```yaml title="sklearn-v2-iris.yaml"
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "sklearn-v2-iris"
spec:
  predictor:
    model:
      modelFormat:
        name: sklearn
      protocolVersion: v2
      runtime: kserve-sklearnserver
      storageUri: "gs://kfserving-examples/models/sklearn/1.0/model"
```

### 佈署 InferenceService

**在 Kserve 中創建 InferenceService 物件:**

```bash
kubectl apply -f sklearn-v2-iris.yaml
```

結果:

```bash
inferenceservice.serving.kserve.io/sklearn-v2-iris created
```

查詢:

```bash
$ kubectl get all
NAME                                                              READY   STATUS    RESTARTS   AGE
pod/sklearn-v2-iris-predictor-00001-deployment-6d6744669b-jttbl   2/2     Running   0          41s

NAME                                              TYPE           CLUSTER-IP      EXTERNAL-IP                                            PORT(S)                                              AGE
service/sklearn-v2-iris-predictor-00001-private   ClusterIP      10.43.134.207   <none>                                                 80/TCP,443/TCP,9090/TCP,9091/TCP,8022/TCP,8012/TCP   42s
service/sklearn-v2-iris-predictor-00001           ClusterIP      10.43.255.6     <none>                                                 80/TCP,443/TCP                                       42s
service/sklearn-v2-iris-predictor                 ExternalName   <none>          knative-local-gateway.istio-system.svc.cluster.local   80/TCP                                               11s
service/sklearn-v2-iris                           ExternalName   <none>          knative-local-gateway.istio-system.svc.cluster.local   <none>                                               10s

NAME                                                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/sklearn-v2-iris-predictor-00001-deployment   1/1     1            1           42s

NAME                                                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/sklearn-v2-iris-predictor-00001-deployment-6d6744669b   1         1         1       42s

NAME                                                          LATESTCREATED                     LATESTREADY                       READY   REASON
configuration.serving.knative.dev/sklearn-v2-iris-predictor   sklearn-v2-iris-predictor-00001   sklearn-v2-iris-predictor-00001   True    

NAME                                                           CONFIG NAME                 K8S SERVICE NAME   GENERATION   READY   REASON   ACTUAL REPLICAS   DESIRED REPLICAS
revision.serving.knative.dev/sklearn-v2-iris-predictor-00001   sklearn-v2-iris-predictor                      1            True             1                 1

NAME                                                  URL                                                        READY   REASON
route.serving.knative.dev/sklearn-v2-iris-predictor   http://sklearn-v2-iris-predictor.kserve-test.example.com   True    

NAME                                                    URL                                                        LATESTCREATED                     LATESTREADY                       READY   REASON
service.serving.knative.dev/sklearn-v2-iris-predictor   http://sklearn-v2-iris-predictor.kserve-test.example.com   sklearn-v2-iris-predictor-00001   sklearn-v2-iris-predictor-00001   True 
```

從上面的訊息可發現 Kserve 構建這個 API 的 URL:

- `http://sklearn-v2-iris-predictor.kserve-test.example.com`

### 測試 v2 協定 RestAPI

接下來我們使用 open-inference-protocol 來取得相關的模型資料并且執行推論:

| API  | Verb | Path |
| ------------- | ------------- | ------------- |
| Inference | POST | v2/models/<model_name>[/versions/\<model_version\>]/infer |
| Model Metadata | GET | v2/models/\<model_name\>[/versions/\<model_version\>] |
| Server Ready | GET | v2/health/ready |
| Server Live | GET | v2/health/live |
| Server Metadata | GET | v2 |
| Model Ready| GET   | v2/models/\<model_name\>[/versions/<model_version>]/ready |

根據上述的 `InferenceService` 的宣告, 這個 API 的模型的名稱:

- `lightgbm-v2-iris`

讓我們把模型名稱發佈成環境變數:

```bash
export MODEL_NAME="sklearn-v2-iris"
```

讓我們先取得一些 Istio IngressGateway 的訊息到環境變數:

```bash
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
```

同時也取得這個 InferenceService 所產生的 Service Hostname:

```bash
export SERVICE_HOSTNAME=$(kubectl get inferenceservice sklearn-v2-iris -o jsonpath='{.status.url}' | cut -d "/" -f 3)
```

#### Health

**`GET v2/health/live`**

執行下列指令:

```bash
curl -v\
  -H "Host: ${SERVICE_HOSTNAME}" \
  -H "Content-Type: application/json" \
  http://${INGRESS_HOST}:${INGRESS_PORT}/v2/health/live
```

結果:

```json
{"live":true}
```

**`GET v2/health/ready`**

執行下列指令:

```bash
curl -v\
  -H "Host: ${SERVICE_HOSTNAME}" \
  -H "Content-Type: application/json" \
  http://${INGRESS_HOST}:${INGRESS_PORT}/v2/health/ready
```

結果:

```json
{"ready":true}
```

**`GET v2/models/${MODEL_NAME}[/versions/${MODEL_VERSION}]/ready`**

執行下列指令:

```bash
curl -v\
  -H "Host: ${SERVICE_HOSTNAME}" \
  -H "Content-Type: application/json" \
  http://${INGRESS_HOST}:${INGRESS_PORT}/v2/models/${MODEL_NAME}/ready
```

結果:

```json
{"name":"sklearn-v2-iris","ready":true}
```

#### Server Metadata

**`GET v2`**

執行下列指令:

```bash
curl -v\
  -H "Host: ${SERVICE_HOSTNAME}" \
  -H "Content-Type: application/json" \
  http://${INGRESS_HOST}:${INGRESS_PORT}/v2
```

結果:

```json
{"name":"kserve","version":"0.12.1","extensions":["model_repository_extension"]}
```

#### Model Metadata

**`GET v2/models/${MODEL_NAME}[/versions/${MODEL_VERSION}]`**

執行下列指令:

```bash
curl -v\
  -H "Host: ${SERVICE_HOSTNAME}" \
  -H "Content-Type: application/json" \
  http://${INGRESS_HOST}:${INGRESS_PORT}/v2/models/${MODEL_NAME}
```

結果:

```json
{"name":"sklearn-v2-iris","versions":null,"platform":"","inputs":[],"outputs":[]}
```

!!! info
    由本次的呼叫的產出來看, 使用者并無法由此 Metadata 來了解要使用本模型 API 來進行推論時的 `inputs` 與 `outputs` 結構。


#### Inference

**`POST v2/models/${MODEL_NAME}[/versions/${MODEL_VERSION}]/infer`**

Open Inference Protocol 定義了一個通用的模型 input 的結構, 主要要包含幾個關鍵資訊:

- `name` Input 的名稱
- `shape` Tensor 的資料型狀,通常是多維的宣告
- `datatype` 數據的型別
- `data` 要推論的數據

??? info

    下表顯示了張量(tensor)資料類型以及每種類型的大小（以位元組為單位）。

    | Data Type | Size (bytes) |
    | --------- | ------------ |
    | BOOL      | 1            |
    | UINT8     | 1            |
    | UINT16    | 2            |
    | UINT32    | 4            |
    | UINT64    | 8            |
    | INT8      | 1            |
    | INT16     | 2            |
    | INT32     | 4            |
    | INT64     | 8            |
    | FP16      | 2            |
    | FP32      | 4            |
    | FP64      | 8            |
    | BYTES     | Variable (max 2<sup>32</sup>) |
    | BF16      | 2            |


```json title="iris-input-v2.json"
{
  "inputs": [
    {
      "name": "input-0",
      "shape": [2, 4],
      "datatype": "FP32",
      "data": [
        [6.8, 2.8, 4.8, 1.4],
        [6.0, 3.4, 4.5, 1.6]
      ]
    }
  ]
}
```

執行下列指令:

```bash
curl -v \
  -H "Host: ${SERVICE_HOSTNAME}" \
  -H "Content-Type: application/json" \
  -d @./iris-input-v2.json \
  http://${INGRESS_HOST}:${INGRESS_PORT}/v2/models/${MODEL_NAME}/infer
```

結果:

```json
{
  "model_name": "sklearn-v2-iris",
  "model_version": null,
  "id": "930a2a63-f5ac-4e77-8fa8-02c0530439af",
  "parameters": null,
  "outputs": [
    {
      "name": "output-0",
      "shape": [
        2
      ],
      "datatype": "INT32",
      "parameters": null,
      "data": [
        1,
        1
      ]
    }
  ]
}
```
