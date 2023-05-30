# 部署 Scikit-learn 模型

此示例將向您介紹如何使用 InferenceService CRD 部署 scikit-learn 模型。

## V1 模型推論協議

### 創建 InferenceService

創建一個 `InferenceService` yaml，指定模型格式 `sklearn` 和 `storageUri` 指向保存的 sklearn 模型，並將其命名為 `scikitlearn-v1.yaml`。

```yaml title="scikitlearn-v1.yaml"
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "sklearn-iris"
spec:
  predictor:
    model:
      modelFormat:
        name: sklearn
      storageUri: "gs://kfserving-examples/models/sklearn/1.0/model"
```

應用 `scikitlearn-v1.yaml` 創建 `InferenceService`，默認情況下它公開一個 HTTP/REST 端點。

```bash
kubectl apply -f scikitlearn-v1.yaml
```

結果:

```bash
inferenceservice.serving.kserve.io/sklearn-iris created
```

等待 InferenceService 進入就緒狀態

```bash
$ kubectl get isvc sklearn-iris

NAME           URL                                          READY   PREV   LATEST   PREVROLLEDOUTREVISION   LATESTREADYREVISION                    AGE
sklearn-iris   http://sklearn-iris.kserve-test.example.it   True           100                              sklearn-iris-predictor-default-00001   29s
```

### 進行模型推論

第一步是確定入口 IP 和端口並設置 `INGRESS_HOST` 和 `INGRESS_PORT`。

```bash
MODEL_NAME=sklearn-iris
SERVICE_HOSTNAME=$(kubectl get inferenceservice sklearn-iris -o jsonpath='{.status.url}' | cut -d "/" -f 3)
```

準備推論用的 request payload:

```bash
cat <<EOF > "./iris-input-v1.json"
{
  "instances": [
    [6.8,  2.8,  4.8,  1.4],
    [6.0,  3.4,  4.5,  1.6]
  ]
}
EOF
```

使用 curl 來呼叫 v1 推論 API:

```bash
curl -v -H "Host: ${SERVICE_HOSTNAME}" "http://${INGRESS_HOST}:${INGRESS_PORT}/v1/models/sklearn-iris:predict" -d @./iris-input-v1.json
```

結果:

```bash hl_lines="21"
*   Trying 172.20.0.10:80...
* TCP_NODELAY set
* Connected to 172.20.0.10 (172.20.0.10) port 80 (#0)
> POST /v1/models/sklearn-iris:predict HTTP/1.1
> Host: sklearn-iris.kserve-test.example.it
> User-Agent: curl/7.68.0
> Accept: */*
> Content-Length: 76
> Content-Type: application/x-www-form-urlencoded
> 
* upload completely sent off: 76 out of 76 bytes
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< content-length: 21
< content-type: application/json
< date: Tue, 30 May 2023 09:34:25 GMT
< server: istio-envoy
< x-envoy-upstream-service-time: 4
< 
* Connection #0 to host 172.20.0.10 left intact
{"predictions":[1,1]}
```

## V2 模型推論協議

### 創建 InferenceService


創建一個 `InferenceService` yaml，指定模型格式 `sklearn`, 指定 serving runtime 到支援 'v2' API 的 `kserve-mlserver` 和 `storageUri` 指向保存的 sklearn 模型，並將其命名為 `scikitlearn-v2.yaml`。

```yaml title="scikitlearn-v2.yaml" hl_lines="10"
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "sklearn-iris-v2"
spec:
  predictor:
    model:
      modelFormat:
        name: sklearn
      runtime: kserve-mlserver
      storageUri: "gs://kfserving-examples/models/sklearn/1.0/model"
```

應用 `scikitlearn-v2.yaml` 創建 `InferenceService`，默認情況下它公開一個 HTTP/REST 端點。

```bash
kubectl apply -f scikitlearn-v2.yaml
```

結果:

```bash
inferenceservice.serving.kserve.io/sklearn-iris-v2 created
```

等待 InferenceService 進入就緒狀態

```bash
$ kubectl get isvc sklearn-iris-v2

NAME              URL                                             READY   PREV   LATEST   PREVROLLEDOUTREVISION   LATESTREADYREVISION                       AGE
sklearn-iris-v2   http://sklearn-iris-v2.kserve-test.example.it   True           100                              sklearn-iris-v2-predictor-default-00001   19s
```

### 進行模型推論

第一步是確定入口 IP 和端口並設置 `INGRESS_HOST` 和 `INGRESS_PORT`。

```bash
MODEL_NAME=sklearn-iris-v2
SERVICE_HOSTNAME=$(kubectl get inferenceservice sklearn-iris-v2 -o jsonpath='{.status.url}' | cut -d "/" -f 3)
```

準備推論用的 request payload:

```bash
cat <<EOF > "./iris-input-v2.json"
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
EOF
```

使用 curl 來呼叫 v2 推論 API:

```bash
curl -v -H "Host: ${SERVICE_HOSTNAME}" "http://${INGRESS_HOST}:${INGRESS_PORT}/v2/models/sklearn-iris-v2/infer" -d @./iris-input-v2.json
```

結果:
