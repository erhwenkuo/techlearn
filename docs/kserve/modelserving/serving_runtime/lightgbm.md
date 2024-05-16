# Deploy LightGBM model with InferenceService

## LightGBM 介詔

LightGBM 是由微軟公司於2017年四月釋出的一款基於決策樹(Decision Tree)學習算法的梯度提升框架。

它被設計為分散式且高效的，具有以下優點:

- Faster training speed and higher efficiency.
- Lower memory usage.
- Better accuracy.
- Support of parallel, distributed, and GPU learning.
- Capable of handling large-scale data.

其中, 決策樹是機器學習演算法中很重要的一環。在 Kaggle 的比賽中，有超過一半的勝利者都使用 XGBoost。最近微軟發布了 Gradient Boosting 的框架 LightGBM ; 很多 Kaggle 的參賽者都開始使用 LightGBM 多於 XGBoost 雖然 XGBoost 有更高的準確率，但是 LightGBM 比 XGBoost 運算的速度快10倍(以前), 目前則是快6倍。

### 什麼是 LightGBM 

- **Light** = light weight (輕量)
- **G** = gradient (梯度)
- **B** = boosting (加速)
- **M** = machines (機器學習)

## 訓練 LightGBM 模型

**套件安裝:**

```bash
pip install lightgbm
```

要測試 LightGBM model serving，首先需要使用以下 python 程式碼訓練一個簡單的 LightGBM 模型。

```python
import lightgbm as lgb
from sklearn.datasets import load_iris
import os

model_dir = "."
BST_FILE = "model.bst"

# 載入 iris 資料集
iris = load_iris()
y = iris['target']
X = iris['data']

# 準備 lgb 訓練的資料集設定
dtrain = lgb.Dataset(X, label=y, feature_name=iris['feature_names'])

# lgb 模型的超參數
params = {
    'objective': 'multiclass',
    'metric': 'softmax',
    'num_class': 3
}

# 進行模型訓練
lgb_model = lgb.train(params=params, train_set=dtrain)

# 取得一筆 input 數據
input = X[:1]

print(f'範例 input: {input}')
print(f'Data type: {type(input)}')
print(f'Data Shape: {input.shape}')

# 進行模型推論
output = lgb_model.predict(input)

print(f'範例 output: {output}')
print(f'Data type: {type(output)}')
print(f'Data Shape: {output.shape}')

# 定義模型導出的路徑
model_file = os.path.join(model_dir, BST_FILE)

# lgb 模型檔的導出
lgb_model.save_model(model_file)
```

結果:

```bash
範例 input: [[5.1 3.5 1.4 0.2]]
Data type: <class 'numpy.ndarray'>
Data Shape: (1, 4)

範例 output: [[9.99985204e-01 1.38238969e-05 9.72063744e-07]]
Data type: <class 'numpy.ndarray'>
Data Shape: (1, 3)
```


在專案目錄下也會看到導出的模型檔案 `model.bst`。

## 使用 v2 協定來佈署 LightGBM 模型

若要使用開放推理協定部署 LightGBM 模型，您需要將 protocolVersion 欄位設為 v2。

```yaml title="lightgbm-v2.yaml"
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "lightgbm-v2-iris"
spec:
  predictor:
    model:
      modelFormat:
        name: lightgbm
      runtime: kserve-lgbserver
      protocolVersion: v2
      storageUri: "gs://kfserving-examples/models/lightgbm/v2/iris"
```

### 佈署 InferenceService

**在 Kserve 中創建 InferenceService 物件:**

```bash
kubectl apply -f lightgbm-v2.yaml
```

結果:

```bash
inferenceservice.serving.kserve.io/lightgbm-v2-iris created
```

查詢:

```bash
$ kubectl get all
NAME                                                               READY   STATUS    RESTARTS   AGE
pod/lightgbm-v2-iris-predictor-00001-deployment-794d949b75-jccdq   2/2     Running   0          2m51s

NAME                                               TYPE           CLUSTER-IP     EXTERNAL-IP                                            PORT(S)                                              AGE
service/lightgbm-v2-iris-predictor-00001-private   ClusterIP      10.43.148.70   <none>                                                 80/TCP,443/TCP,9090/TCP,9091/TCP,8022/TCP,8012/TCP   2m51s
service/lightgbm-v2-iris-predictor-00001           ClusterIP      10.43.106.98   <none>                                                 80/TCP,443/TCP                                       2m51s
service/lightgbm-v2-iris-predictor                 ExternalName   <none>         knative-local-gateway.istio-system.svc.cluster.local   80/TCP                                               100s
service/lightgbm-v2-iris                           ExternalName   <none>         knative-local-gateway.istio-system.svc.cluster.local   <none>                                               100s

NAME                                                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/lightgbm-v2-iris-predictor-00001-deployment   1/1     1            1           2m51s

NAME                                                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/lightgbm-v2-iris-predictor-00001-deployment-794d949b75   1         1         1       2m51s

NAME                                                           LATESTCREATED                      LATESTREADY                        READY   REASON
configuration.serving.knative.dev/lightgbm-v2-iris-predictor   lightgbm-v2-iris-predictor-00001   lightgbm-v2-iris-predictor-00001   True    

NAME                                                            CONFIG NAME                  K8S SERVICE NAME   GENERATION   READY   REASON   ACTUAL REPLICAS   DESIRED REPLICAS
revision.serving.knative.dev/lightgbm-v2-iris-predictor-00001   lightgbm-v2-iris-predictor                      1            True             1                 1

NAME                                                   URL                                                         READY   REASON
route.serving.knative.dev/lightgbm-v2-iris-predictor   http://lightgbm-v2-iris-predictor.kserve-test.example.com   True    

NAME                                                     URL                                                         LATESTCREATED                      LATESTREADY                        READY   REASON
service.serving.knative.dev/lightgbm-v2-iris-predictor   http://lightgbm-v2-iris-predictor.kserve-test.example.com   lightgbm-v2-iris-predictor-00001   lightgbm-v2-iris-predictor-00001   True    
```

從上面的訊息可發現 Kserve 構建這個 API 的 URL:

- `http://lightgbm-v2-iris-predictor.kserve-test.example.com`

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
export MODEL_NAME="lightgbm-v2-iris"
```

讓我們先取得一些 Istio IngressGateway 的訊息到環境變數:

```bash
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
```

同時也取得這個 InferenceService 所產生的 Service Hostname:

```bash
export SERVICE_HOSTNAME=$(kubectl get inferenceservice lightgbm-v2-iris -o jsonpath='{.status.url}' | cut -d "/" -f 3)
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
{"name":"lightgbm-v2-iris","ready":true}
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
{"name":"lightgbm-v2-iris","versions":null,"platform":"","inputs":[],"outputs":[]}
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
curl -v\
  -H "Host: ${SERVICE_HOSTNAME}" \
  -H "Content-Type: application/json" \
  http://${INGRESS_HOST}:${INGRESS_PORT}/v2/models/${MODEL_NAME}

curl -v \
  -H "Host: ${SERVICE_HOSTNAME}" \
  -H "Content-Type: application/json" \
  -d @./iris-input-v2.json \
  http://${INGRESS_HOST}:${INGRESS_PORT}/v2/models/${MODEL_NAME}/infer
```

結果:

```json
{
  "model_name": "lightgbm-v2-iris",
  "model_version": null,
  "id": "3e2aac4b-c463-4e3a-bb97-48f2c58cec93",
  "parameters": null,
  "outputs": [
    {
      "name": "output-0",
      "shape": [
        2,
        3
      ],
      "datatype": "FP64",
      "parameters": null,
      "data": [
        0.000008796664107010673,
        0.9992300031041593,
        0.0007612002317336916,
        0.000004974786820804187,
        0.9999919650711493,
        0.0000030601420299625077
      ]
    }
  ]
}
```
