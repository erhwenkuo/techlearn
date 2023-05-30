# 部署 Tensorflow 模型

## V1 模型推論協議

### 創建 InferenceService

創建一個 `InferenceService` yaml，指定框架 `tensorflow` 和 `storageUri` 指向保存的 tensorflow 模型，並將其命名為 `tensorflow.yaml`。

```yaml title="tensorflow.yaml"
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "flower-sample"
spec:
  predictor:
    model:
      modelFormat:
        name: tensorflow
      storageUri: "gs://kfserving-examples/models/tensorflow/flowers"
```

應用 `tensorflow.yaml` 創建 `InferenceService`，默認情況下它公開一個 HTTP/REST 端點。

```bash
kubectl apply -f tensorflow.yaml 
```

結果:

```bash
inferenceservice.serving.kserve.io/flower-sample created
```

等待 InferenceService 進入就緒狀態

```bash
$ kubectl get isvc flower-sample

NAME            URL                                           READY   PREV   LATEST   PREVROLLEDOUTREVISION   LATESTREADYREVISION                     AGE
flower-sample   http://flower-sample.kserve-test.example.it   True           100                              flower-sample-predictor-default-00001   56s
```

### 進行模型推論

第一步是確定入口 IP 和端口並設置 `INGRESS_HOST` 和 `INGRESS_PORT`，推理請求輸入文件可以在這裡[下載](./assets/tensorflow-flower-input.json)。

```bash
MODEL_NAME=flower-sample
INPUT_PATH=@./tensorflow-flower-input.json
SERVICE_HOSTNAME=$(kubectl get inferenceservice ${MODEL_NAME} -o jsonpath='{.status.url}' | cut -d "/" -f 3)
curl -v -H "Host: ${SERVICE_HOSTNAME}" http://${INGRESS_HOST}:${INGRESS_PORT}/v1/models/$MODEL_NAME:predict -d $INPUT_PATH
```

結果:

```bash hl_lines="18-26"
* Connected to localhost (::1) port 8080 (#0)
> POST /v1/models/tensorflow-sample:predict HTTP/1.1
> Host: tensorflow-sample.default.example.com
> User-Agent: curl/7.73.0
> Accept: */*
> Content-Length: 16201
> Content-Type: application/x-www-form-urlencoded
> 
* upload completely sent off: 16201 out of 16201 bytes
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< content-length: 222
< content-type: application/json
< date: Sun, 31 Jan 2021 01:01:50 GMT
< x-envoy-upstream-service-time: 280
< server: istio-envoy
< 
{
    "predictions": [
        {
            "scores": [0.999114931, 9.20987877e-05, 0.000136786213, 0.000337257545, 0.000300532585, 1.84813616e-05],
            "prediction": 0,
            "key": "   1"
        }
    ]
}
```
