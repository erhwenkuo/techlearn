# 在 Minio 上使用保存的模型部署 InferenceService

### 創建 Service Account & Secret

使用您的 minio 用戶 credential 來創建一個 Secret，KServe 讀取 Secret 註釋以將 S3 環境變量注入存儲初始化程序或模型代理以從 minio 存儲下載模型。

```yaml title="kserve-minio-sa.yaml"
apiVersion: v1
kind: Secret
metadata:
  name: minio-kserve-secret
  annotations:
     serving.kserve.io/s3-endpoint: "minio-service.kubeflow:9000" # replace with your minio endpoint
     serving.kserve.io/s3-usehttps: "0" # by default 1, if testing with minio you can set to 0
     serving.kserve.io/s3-useanoncredential: "false" # omitting this is the same as false, if true will ignore provided credential and use anonymous credentials
type: Opaque
stringData: # use `stringData` for raw credential string or `data` for base64 encoded string
  AWS_ACCESS_KEY_ID: "minio"
  AWS_SECRET_ACCESS_KEY: "minio123"
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa-minio-kserve
secrets:
- name: minio-kserve-secret
```

執行下列命令來套用

```
kubectl apply -f kserve-minio-sa.yaml
```

!!! info
    如果您在啟用 istio sidecars 的情況下運行 kserve，則 istio 代理準備就緒和代理拉取模型之間可能存在競爭條件。當代理嘗試從 s3 下載時，這將導致 tcp 撥號連接被拒絕錯誤。

    為了解決這個問題，istio 允許阻塞 pod 中的其他容器，直到代理容器準備就緒。

    您可以通過在 `istio-sidecar-injector` configmap 中設置 `proxy.holdApplicationUntilProxyStarts: true` 來啟用此功能，`proxy.holdApplicationUntilProxyStarts` 標誌在 Istio 1.7 中作為實驗性功能引入，默認情況下處於關閉狀態。

## 使用 InferenceService 在 Minio 上部署模型

使用 s3 `storageUri` 和附加了 s3 secret 的 service acccount 來創建 `InferenceService`。

```yaml hl_lines="7" title="mnist-minio.yaml"
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "mnist-minio"
spec:
  predictor:
    serviceAccountName: sa-minio-kserve
    model:
      modelFormat:
        name: tensorflow
      storageUri: "s3://kserve-examples/mnist"
```

執行下列命令來套用

```bash
kubectl apply -f mnist-minio.yaml
```

## 運行模型推論

現在，可以通過 `${INGRESS_HOST}:${INGRESS_PORT}` 訪問入口，或按照此說明查找入口 IP 和端口。

```bash
SERVICE_HOSTNAME=$(kubectl get inferenceservice mnist-s3 -o jsonpath='{.status.url}' | cut -d "/" -f 3)

MODEL_NAME=mnist-minio
INPUT_PATH=@./input.json
curl -v -H "Host: ${SERVICE_HOSTNAME}" http://${INGRESS_HOST}:${INGRESS_PORT}/v1/models/$MODEL_NAME:predict -d $INPUT_PATH
```

結果:

```bash
Note: Unnecessary use of -X or --request, POST is already inferred.
*   Trying 35.237.217.209...
* TCP_NODELAY set
* Connected to mnist-s3.default.35.237.217.209.xip.io (35.237.217.209) port 80 (#0)
> POST /v1/models/mnist-s3:predict HTTP/1.1
> Host: mnist-s3.default.35.237.217.209.xip.io
> User-Agent: curl/7.55.1
> Accept: */*
> Content-Length: 2052
> Content-Type: application/x-www-form-urlencoded
> Expect: 100-continue
>
< HTTP/1.1 100 Continue
* We are completely uploaded and fine
< HTTP/1.1 200 OK
< content-length: 251
< content-type: application/json
< date: Sun, 04 Apr 2021 20:06:27 GMT
< x-envoy-upstream-service-time: 5
< server: istio-envoy
<
* Connection #0 to host mnist-s3.default.35.237.217.209.xip.io left intact
{
    "predictions": [
        {
            "predictions": [0.327352405, 2.00153053e-07, 0.0113353515, 0.203903764, 3.62863029e-05, 0.416683704, 0.000281196437, 8.36911859e-05, 0.0403052084, 1.82206513e-05],
            "classes": 5
        }
    ]
}
```
