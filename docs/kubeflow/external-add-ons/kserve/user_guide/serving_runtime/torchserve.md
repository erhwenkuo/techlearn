# 部署 PyTorch 模型

在此示例中，我們部署了經過訓練的 PyTorch MNIST 模型，通過使用 TorchServe 運行時運行 `InferenceService` 來預測手寫數字，[TorchServe runtime](https://github.com/pytorch/serve) 是 PyTorch 模型默認安裝的模型推論服務 runtime。

模型可解釋性也是一個重要方面，它有助於理解哪些輸入特徵對特定分類很重要。 [Captum](https://captum.ai/) 是一個模型可解釋性庫。在此示例中，TorchServe explain endpoint 是使用 Captum 最先進的算法實現的，包括集成梯度，為用戶提供一種簡單的方法來了解哪些特徵對模型輸出有貢獻。您可以參考 [Captum 教程](https://captum.ai/tutorials/)以獲取更多示例。

## 使用模型存檔文件和配置創建模型存儲

`KServe/TorchServe` 整合需要以下模型的檔案使用特定的佈局。

```bash
├── config
│   ├── config.properties
├── model-store
│   ├── densenet_161.mar
│   ├── mnist.mar
```

`TorchServe` 提供了一個工具程序，可將所有模型工件打包到單個 [TorchServe 模型存檔文件 (MAR)](https://github.com/pytorch/serve/blob/master/model-archiver/README.md) 中。

將模型工件打包成 MAR 文件後，再上傳到模型存儲路徑下的 `model-store` 。

您可以將模型和相關文件存儲在遠程存儲或本地持久卷上。 MNIST 模型和依賴文件可以從[這裡](https://github.com/pytorch/serve/tree/master/examples/image_classifier/mnist)獲得。

!!! info
    對於遠程存儲，您可以選擇使用存儲在 KServe 示例 GCS 存儲桶 `gs://kfserving-examples/models/torchserve/image_classifier` 上的預構建 MNIST MAR 文件啟動示例，或者使用 `torch-model-archiver` 生成 MAR 文件並創建根據上述佈局將模型存儲在遠程存儲上。

    ```bash
    torch-model-archiver --model-name mnist --version 1.0 \
    --model-file model-archiver/model-store/mnist/mnist.py \
    --serialized-file model-archiver/model-store/mnist/mnist_cnn.pt \
    --handler model-archiver/model-store/mnist/mnist_handler.py \
    ```

TorchServe 使用 `config.properties` 文件來存儲配置。有關配置文件支持的屬性的更多詳細信息，請參閱[此處](https://pytorch.org/serve/configuration.html#config-properties-file)。以下是 KServe 的示例文件：

```properties title="config.properties" hl_lines="10"
inference_address=http://0.0.0.0:8085
management_address=http://0.0.0.0:8085
metrics_address=http://0.0.0.0:8082
grpc_inference_port=7070
grpc_management_port=7071
enable_metrics_api=true
metrics_format=prometheus
number_of_netty_threads=4
job_queue_size=10
enable_envvars_config=true
install_py_dep_per_model=true
model_store=/mnt/models/model-store
model_snapshot={"name":"startup.cfg","modelCount":1,"models":{"mnist":{"1.0":{"defaultVersion":true,"marName":"mnist.mar","minWorkers":1,"maxWorkers":5,"batchSize":1,"maxBatchDelay":10,"responseTimeout":120}}}}
```

KServe/TorchServe 集成支持 KServe v1/v2 REST 協議。在 `config.properties` 中，我們需要打開標誌 `enable_envvars_config` 以啟用使用環境變量設置 KServe。

## V1 模型推論協議

### 創建 InferenceService

當您在模型規範上指定模型格式 `pytorch` 時，KServe 默認選擇 `TorchServe` runtime。

```yaml title="torchserve-v1.yaml"
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "torchserve"
spec:
  predictor:
    model:
      modelFormat:
        name: pytorch
      storageUri: gs://kfserving-examples/models/torchserve/image_classifier/v1
```

要在 CPU 上部署模型，請應用以下 `torchserve-v1.yaml` 來創建 `InferenceService`。

```bash
kubectl apply -f torchserve-v1.yaml
```

要在 GPU 上部署模型，請應用以下 `torchserve-v1-gpu.yaml` 來創建 `InferenceService`。

```yaml title="torchserve-v1-gpu.yaml"
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "torchserve"
spec:
  predictor:
    model:
      modelFormat:
        name: pytorch
      storageUri: gs://kfserving-examples/models/torchserve/image_classifier/v1
      resources:
        limits:
          memory: 4Gi
          nvidia.com/gpu: "1"
```

結果:

```bash
inferenceservice.serving.kserve.io/torchserve created
```

### 進行模型推論

第一步是確定入口 IP 和端口並設置 INGRESS_HOST 和 INGRESS_PORT。

```bash
MODEL_NAME=mnist
SERVICE_HOSTNAME=$(kubectl get inferenceservice torchserve -o jsonpath='{.status.url}' | cut -d "/" -f 3)
```

您可以使用 [image converter](https://github.com/kserve/kserve/tree/master/docs/samples/v1beta1/torchserve/v1/imgconv) 將圖像轉換為 `base64`byte array。

```bash
curl -v -H "Host: ${SERVICE_HOSTNAME}" http://${INGRESS_HOST}:${INGRESS_PORT}/v1/models/${MODEL_NAME}:predict -d @./mnist.json
```

結果:

```console hl_lines="22"
$ curl -v -H "Host: ${SERVICE_HOSTNAME}" http://${INGRESS_HOST}:${INGRESS_PORT}/v1/models/${MODEL_NAME}:predict -d @./mnist.json
*   Trying 172.20.0.10:80...
* TCP_NODELAY set
* Connected to 172.20.0.10 (172.20.0.10) port 80 (#0)
> POST /v1/models/mnist:predict HTTP/1.1
> Host: torchserve.kserve-test.example.it
> User-Agent: curl/7.68.0
> Accept: */*
> Content-Length: 401
> Content-Type: application/x-www-form-urlencoded
> 
* upload completely sent off: 401 out of 401 bytes
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< content-length: 20
< content-type: application/json; charset=UTF-8
< date: Tue, 30 May 2023 08:02:02 GMT
< server: istio-envoy
< x-envoy-upstream-service-time: 56
< 
* Connection #0 to host 172.20.0.10 left intact
{"predictions": [2]}
```

## V2 模型推論協議

### 創建 InferenceService

當您在新模型規範上指定模型格式 `pytorch` 並啟用 KServe `v1` 推理協議時，KServe 默認選擇 TorchServe runtime。要啟用 `v2` 推理協議，請使用值 `v2` 指定 `protocolVersion` 欄位。

```yaml title="torchserve-v2.yaml" hl_lines="10"
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "torchserve-mnist-v2"
spec:
  predictor:
    model:
      modelFormat:
        name: pytorch
      protocolVersion: v2  
      storageUri: gs://kfserving-examples/models/torchserve/image_classifier/v2
```

要在 CPU 上部署模型，請應用 `torchserve-v2.yaml` 來創建 InferenceService。

```bash
kubectl apply -f torchserve-v2.yaml
```

結果:

```console
inferenceservice.serving.kserve.io/torchserve-mnist-v2 created
```

### 進行模型推論

第一步是確定入口 IP 和端口並設置 INGRESS_HOST 和 INGRESS_PORT。

```bash
MODEL_NAME=mnist
SERVICE_HOSTNAME=$(kubectl get inferenceservice torchserve-mnist-v2 -o jsonpath='{.status.url}' | cut -d "/" -f 3)
```

您可以使用 v2 協議發送 **byte array** 或是使用 **tensor**，對於 byte array，使用 [image converter](https://github.com/kserve/kserve/tree/master/docs/samples/v1beta1/torchserve/v2/bytes_conv) 將圖像轉換為 byte array 輸入。這裡我們使用 [`mnist_v2_bytes.json`](https://kserve.github.io/website/0.10/modelserving/v1beta1/torchserve/mnist_v2_bytes.json) 文件來運行示例推理。

```bash
curl -v -H "Host: ${SERVICE_HOSTNAME}" http://${INGRESS_HOST}:${INGRESS_PORT}/v2/models/${MODEL_NAME}/infer -d @./mnist_v2_bytes.json
```

結果:

```json
{"id": "813203d0-5deb-4c7c-8286-17fbbe7529d7", "model_name": "mnist", "model_version": "1.0", "outputs": [{"name": "predict", "shape": [1], "datatype": "INT64", "data": [0]}]}
```

對於張量輸入，使用 [tensor image converter](https://github.com/kserve/kserve/tree/master/docs/samples/v1beta1/torchserve/v2/tensor_conv) 將圖像轉換為張量輸入，這裡我們使用 [`mnist_v2.json`](https://kserve.github.io/website/0.10/modelserving/v1beta1/torchserve/mnist_v2.json) 文件運行示例推理。

```bash
curl -v -H "Host: ${SERVICE_HOSTNAME}" http://${INGRESS_HOST}:${INGRESS_PORT}/v2/models/${MODEL_NAME}/infer -d @./mnist_v2.json
```

結果:

```json
{"id": "d3b15cad-50a2-4eaf-80ce-8b0a428bd298", "model_name": "mnist", "model_version": "1.0", "outputs": [{"name": "predict", "shape": [1], "datatype": "INT64", "data": [1]}]}
```

