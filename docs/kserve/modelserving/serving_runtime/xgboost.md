# Deploying XGBoost models with Inference

## XGBoost 介詔

XGBoost (Extreme Gradient Boosting)，是一種 Gradient Boosted Tree(GBDT)，每一次保留原來的模型不變，並且加入一個新的函數至模型中，修正上一棵樹的錯誤，以提升整體的模型。主要用於解於監督是學習，可以用於分類也可以用於迴歸問題。

**1. 基於 Tree Ensemble 模型**

需要考慮多棵樹的參數優化問題，但是我們卻無法一次訓練所有的樹，因此會透過增量訓練 (additive training) 的方式，每一次保留原來的模型不變，並且加入一個新的函數至模型中，也就是說每一步皆會在前一步的基礎上增加一顆樹，以利修復上一顆樹的不足，有助於提升目標函數。

**2. XGBoost 有別於傳統的 GBDT**

選擇新增的樹、找到最好的樹以及減枝過程的方法差異。主要是透過貪心法，在樹的每層建構過程中優化目標函式的最大增益。

**3. 優點以及使用方式**

背後是 C++，所以速度很快，有競賽神器 (Kaggle 競賽最常被選擇使用的演算法之一) 之稱。儘管如此，雖許多隊伍都使用 XGBoost，得分的差異也非常大，涉及更多的是 XGBoost 背後的思想原理及數學模型的描述，因此只有深刻理解的人，才能發揮其最大價值。


## 訓練 XGBoost 模型

**套件安裝:**

```bash
pip install xgboost
```

要測試 XGBoost model serving，首先需要使用以下 python 程式碼訓練一個簡單的 XGBoost 模型。

```python
import xgboost as xgb
from sklearn.datasets import load_iris
import os

model_dir = "."
BST_FILE = "model.bst"

# 載入 iris 資料集
iris = load_iris()
y = iris['target']
X = iris['data']

# 準備 xgboost 訓練的資料集設定
dtrain = xgb.DMatrix(X, label=y)

# xgboost 模型的超參數
param = {'max_depth': 6,
         'eta': 0.1,
         'silent': 1,
         'nthread': 4,
         'num_class': 10,
         'objective': 'multi:softmax'
         }

# 進行模型訓練
xgb_model = xgb.train(params=param, dtrain=dtrain)

# 取得一筆 input 數據
input = X[:2]

print(f'範例 input: {input}')
print(f'Data type: {type(input)}')
print(f'Data Shape: {input.shape}')

# 進行模型推論
output = xgb_model.predict(xgb.DMatrix(input))

print(f'範例 output: {output}')
print(f'Data type: {type(output)}')
print(f'Data Shape: {output.shape}')

# 定義模型導出的路徑
model_file = os.path.join(model_dir, BST_FILE)

# xgboost 模型檔的導出
xgb_model.save_model(model_file)
```

結果:

```bash
範例 input: [[5.1 3.5 1.4 0.2]]
Data type: <class 'numpy.ndarray'>
Data Shape: (1, 4)
範例 output: [0.]
Data type: <class 'numpy.ndarray'>
Data Shape: (1,)
```

在專案目錄下也會看到導出的模型檔案 `model.bst`。

## 使用 v2 協定來佈署 XGBoost 模型

若要使用開放推理協定部署 XGBoost 模型，您需要將 protocolVersion 欄位設為 v2。

```yaml title="xgboost-v2.yaml"
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "xgboost-v2-iris"
spec:
  predictor:
    model:
      modelFormat:
        name: xgboost
      protocolVersion: v2
      runtime: kserve-xgbserver
      storageUri: "gs://kfserving-examples/models/xgboost/iris"
```

### 佈署 InferenceService

**在 Kserve 中創建 InferenceService 物件:**

```bash
kubectl apply -f xgboost-v2-iris.yaml
```

結果:

```bash
inferenceservice.serving.kserve.io/xgboost-v2-iris created
```

查詢:

```bash
$ kubectl get all
NAME                                                              READY   STATUS    RESTARTS   AGE
pod/xgboost-v2-iris-predictor-00001-deployment-7cf4b44588-p4rpg   2/2     Running   0          7m32s

NAME                                              TYPE           CLUSTER-IP      EXTERNAL-IP                                            PORT(S)                                              AGE
service/xgboost-v2-iris-predictor-00001-private   ClusterIP      10.43.138.185   <none>                                                 80/TCP,443/TCP,9090/TCP,9091/TCP,8022/TCP,8012/TCP   7m32s
service/xgboost-v2-iris-predictor-00001           ClusterIP      10.43.121.156   <none>                                                 80/TCP,443/TCP                                       7m32s
service/xgboost-v2-iris-predictor                 ExternalName   <none>          knative-local-gateway.istio-system.svc.cluster.local   80/TCP                                               5m51s
service/xgboost-v2-iris                           ExternalName   <none>          knative-local-gateway.istio-system.svc.cluster.local   <none>                                               5m51s

NAME                                                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/xgboost-v2-iris-predictor-00001-deployment   1/1     1            1           7m32s

NAME                                                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/xgboost-v2-iris-predictor-00001-deployment-7cf4b44588   1         1         1       7m32s

NAME                                                          LATESTCREATED                     LATESTREADY                       READY   REASON
configuration.serving.knative.dev/xgboost-v2-iris-predictor   xgboost-v2-iris-predictor-00001   xgboost-v2-iris-predictor-00001   True    

NAME                                                           CONFIG NAME                 K8S SERVICE NAME   GENERATION   READY   REASON   ACTUAL REPLICAS   DESIRED REPLICAS
revision.serving.knative.dev/xgboost-v2-iris-predictor-00001   xgboost-v2-iris-predictor                      1            True             1                 1

NAME                                                  URL                                                        READY   REASON
route.serving.knative.dev/xgboost-v2-iris-predictor   http://xgboost-v2-iris-predictor.kserve-test.example.com   True    

NAME                                                    URL                                                        LATESTCREATED                     LATESTREADY                       READY   REASON
service.serving.knative.dev/xgboost-v2-iris-predictor   http://xgboost-v2-iris-predictor.kserve-test.example.com   xgboost-v2-iris-predictor-00001   xgboost-v2-iris-predictor-00001   True    
```

從上面的訊息可發現 Kserve 構建這個 API 的 URL:

- `http://xgboost-v2-iris-predictor.kserve-test.example.com`

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

- `xgboost-v2-iris`

讓我們把模型名稱發佈成環境變數:

```bash
export MODEL_NAME="xgboost-v2-iris"
```

讓我們先取得一些 Istio IngressGateway 的訊息到環境變數:

```bash
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
```

同時也取得這個 InferenceService 所產生的 Service Hostname:

```bash
export SERVICE_HOSTNAME=$(kubectl get inferenceservice xgboost-v2-iris -o jsonpath='{.status.url}' | cut -d "/" -f 3)
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
{"name":"xgboost-v2-iris","ready":true}
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
{"name":"xgboost-v2-iris","versions":null,"platform":"","inputs":[],"outputs":[]}
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
  "model_name": "xgboost-v2-iris",
  "model_version": null,
  "id": "b509c4e8-6ff1-4225-a607-6a16bbafdf38",
  "parameters": null,
  "outputs": [
    {
      "name": "output-0",
      "shape": [
        2
      ],
      "datatype": "FP32",
      "parameters": null,
      "data": [
        1,
        1
      ]
    }
  ]
}
```
