# 使用 InferenceService 部署 Transformer

`Transformer` 是一個 `InferenceService` 組件，它與模型推理一起進行 pre/post processing。它通常採用 raw input 並將它們轉換為模型服務器期望的輸入張量 (input tensors)。在此示例中，我們演示了一個使用通過 REST 和 gRPC 協議進行通信的自定義 `Transformer` 運行推理的示例。

## 構建 Custom Transformer 鏡像

### 使用 KServe 模型 API 實施前/後處理

`KServe.Model` base-class 主要定義了 `preprocess`、`predict` 和 `postprocess` 三個 handler，這些 handler 按順序執行，其中 `preprocess handler` 的輸出作為輸入傳遞給`predict handler`。

當 `predictor_host` 被傳遞時，`predict handler` 調用 predictor 並取回響應，然後將其傳遞給 `postprocess handler`。KServe 自動為 Transformer 填入 `predictor_host`，將調用交給 Predictor。默認情況下，transformer 對預測器進行 REST 調用，如果要對預測器進行 gRPC 調用，您可以傳遞值為 `grpc-v2` 的 `--protocol` 參數。

要實現 Transformer，您可以從繼承 `Model` 類別，然後覆蓋 `preprocess` 和 `postprocess` 以擁有您自己的自定義轉換邏輯。對於 Open(v2) 推理協議，KServe 為預測、預處理、後處理處理程序提供 `InferRequest` 和 `InferResponse` API 對象，以抽像出 REST/gRPC 在線解碼和編碼的實現細節。

```python
from kserve import Model, ModelServer, model_server, InferInput, InferRequest
from typing import Dict
from PIL import Image
import torchvision.transforms as transforms
import logging
import io
import base64

logging.basicConfig(level=kserve.constants.KSERVE_LOGLEVEL)

def image_transform(byte_array):
    """converts the input image of Bytes Array into Tensor
    Args:
        instance (dict): The request input for image bytes.
    Returns:
        list: Returns converted tensor as input for predict handler with v1/v2 inference protocol.
    """
    image_processing = transforms.Compose([
        transforms.ToTensor(),
        transforms.Normalize((0.1307,), (0.3081,))
    ])
    image = Image.open(io.BytesIO(byte_array))
    tensor = image_processing(image).numpy()
    return tensor

# for v1 REST predictor the preprocess handler converts to input image bytes to float tensor dict in v1 inference REST protocol format
class ImageTransformer(kserve.Model):
    def __init__(self, name: str, predictor_host: str, headers: Dict[str, str] = None):
        super().__init__(name)
        self.predictor_host = predictor_host
        self.ready = True

    def preprocess(self, inputs: Dict, headers: Dict[str, str] = None) -> Dict:
        return {'instances': [image_transform(instance) for instance in inputs['instances']]}

    def postprocess(self, inputs: Dict, headers: Dict[str, str] = None) -> Dict:
        return inputs

# for v2 gRPC predictor the preprocess handler converts the input image bytes tensor to float tensor in v2 inference protocol format
class ImageTransformer(kserve.Model):
    def __init__(self, name: str, predictor_host: str, protocol: str, headers: Dict[str, str] = None):
        super().__init__(name)
        self.predictor_host = predictor_host
        self.protocol = protocol
        self.ready = True

    def preprocess(self, request: InferRequest, headers: Dict[str, str] = None) -> InferRequest:
        input_tensors = [image_transform(instance) for instance in request.inputs[0].data]
        input_tensors = np.asarray(input_tensors)
        infer_inputs = [InferInput(name="INPUT__0", datatype='FP32', shape=list(input_tensors.shape),
                                   data=input_tensors)]
        infer_request = InferRequest(model_name=self.model_name, infer_inputs=infer_inputs)
        return infer_request

```

請在[此處](https://github.com/kserve/kserve/tree/release-0.10/python/custom_transformer)查看代碼示例。

### Transformer Server 入口點

對於單個模型，您只需創建一個轉換器對象並將其註冊到模型服務器。

```python
if __name__ == "__main__":
    model = ImageTransformer(args.model_name, predictor_host=args.predictor_host,
                             protocol=args.protocol)
    ModelServer().start(models=[model])
```

對於多模型情況，如果所有模型都可以共享相同的轉換器，您可以為不同的模型註冊相同的轉換器，或者如果每個模型需要自己的轉換，則可以註冊不同的轉換器。

```python
if __name__ == "__main__":
    for model_name in model_names:
        transformer = ImageTransformer(model_name, predictor_host=args.predictor_host)
        models.append(transformer)
    kserve.ModelServer().start(models=models)
```

### 構建 Transformer 容器鏡像

在 `kserve/python` 目錄下，使用 [Dockerfile](https://github.com/kserve/kserve/blob/release-0.10/python/custom_transformer.Dockerfile) 構建 transformer docker 鏡像

```bash
cd python
docker build -t $DOCKER_USER/image-transformer:latest -f transformer.Dockerfile .

docker push {username}/image-transformer:latest
```

## 使用 REST Predictor 部署 InferenceService

### 創建 InferenceService

默認情況下，`InferenceService` 使用 TorchServe 為 PyTorch 模型提供服務，並且可以根據 TorchServe 模型存儲庫佈局從雲存儲中的模型存儲庫加載模型。在此示例中，模型存儲庫包含一個 MNIST 模型，但您可以在其中存儲多個模型。

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: torch-transformer
spec:
  predictor:
    model:
      modelFormat:
        name: pytorch
      storageUri: gs://kfserving-examples/models/torchserve/image_classifier/v1
  transformer:
    containers:
      - image: kserve/image-transformer:latest
        name: kserve-container
        command:
          - "python"
          - "-m"
          - "model"
        args:
          - --model_name
          - mnist
```

下載的工件存儲在 `/mnt/models` 下。

應用 `transformer-new.yaml` InferenceService:

```bash
kubectl apply -f transformer-new.yaml
```

結果:

```
$ inferenceservice.serving.kserve.io/torch-transformer created
```

### 運行預測

首先，下載請求[輸入負載](https://kserve.github.io/website/master/modelserving/v1beta1/transformer/torchserve_image_transformer/input.json)。

```json
{
    "instances":[
       {
          "image":{
            "b64": "iVBORw0KGgoAAAANSUhEUgAAABwAAAAcCAAAAABXZoBIAAAAw0lEQVR4nGNgGFggVVj4/y8Q2GOR83n+58/fP0DwcSqmpNN7oOTJw6f+/H2pjUU2JCSEk0EWqN0cl828e/FIxvz9/9cCh1zS5z9/G9mwyzl/+PNnKQ45nyNAr9ThMHQ/UG4tDofuB4bQIhz6fIBenMWJQ+7Vn7+zeLCbKXv6z59NOPQVgsIcW4QA9YFi6wNQLrKwsBebW/68DJ388Nun5XFocrqvIFH59+XhBAxThTfeB0r+vP/QHbuDCgr2JmOXoSsAAKK7bU3vISS4AAAAAElFTkSuQmCC"
          }
       }
    ]
}
```

然後，確定入口 IP 和端口並設置 INGRESS_HOST 和 INGRESS_PORT。

```bash
SERVICE_NAME=torch-transformer
MODEL_NAME=mnist
INPUT_PATH=@./input.json
SERVICE_HOSTNAME=$(kubectl get inferenceservice $SERVICE_NAME -o jsonpath='{.status.url}' | cut -d "/" -f 3)

curl -v -H "Host: ${SERVICE_HOSTNAME}" -d $INPUT_PATH http://${INGRESS_HOST}:${INGRESS_PORT}/v1/models/$MODEL_NAME:predict
```

結果:

```console
> POST /v1/models/mnist:predict HTTP/1.1
> Host: torch-transformer.default.example.com
> User-Agent: curl/7.73.0
> Accept: */*
> Content-Length: 401
> Content-Type: application/x-www-form-urlencoded
>
* upload completely sent off: 401 out of 401 bytes
Handling connection for 8080
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< content-length: 20
< content-type: application/json; charset=UTF-8
< date: Tue, 12 Jan 2021 09:52:30 GMT
< server: istio-envoy
< x-envoy-upstream-service-time: 83
<
* Connection #0 to host localhost left intact
{"predictions": [2]}
```

## 用 Grpc Predictor 部署 InferenceService

與 REST 相比，gRPC 速度更快，因為協議緩衝區的緊密包裝以及 gRPC 使用 HTTP/2。在許多情況下，gRPC 可以成為 Transformer 和 Predictor 之間更高效的通信協議，因為您可能需要在它們之間傳輸大型張量。

### 創建 InferenceService

使用以下 yaml 創建 InferenceService，其中包括一個 Transformer 和一個 Triton Predictor。由於 KServe 默認為 PyTorch 模型使用 `TorchServe` 服務運行時，因此您需要將服務運行時覆蓋為 `kserve-tritonserver` 以使用 gRPC 協議。轉換器通過指定 `--protocol` 參數調用使用 `V2` gRPC 協議的預測器。

```yaml title="grpc_transformer.yaml" hl_lines="11"
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: torch-grpc-transformer
spec:
  predictor:
    model:
      modelFormat: 
        name: pytorch
      storageUri: gs://kfserving-examples/models/torchscript
      runtime: kserve-tritonserver
      runtimeVersion: 20.10-py3
      ports:
      - name: h2c
        protocol: TCP
        containerPort: 9000
  transformer:
    containers:
    - image: kserve/image-transformer:latest
      name: kserve-container
      command:
      - "python"
      - "-m"
      - "model"
      args:
      - --model_name
      - cifar10
      - --protocol
      - grpc-v2
```

應用推理服務 `grpc_transformer.yaml`

```bash
kubectl apply -f grpc_transformer.yaml
```

結果:

```
$ inferenceservice.serving.kserve.io/torch-grpc-transformer created
```

### 運行預測

首先，下載請求 [input payload](https://kserve.github.io/website/master/modelserving/v1beta1/transformer/torchserve_image_transformer/image.json)。

然後，確定入口 IP 和端口並設置 `INGRESS_HOST` 和 `INGRESS_PORT`

```bash
SERVICE_NAME=torch-grpc-transformer
MODEL_NAME=cifar10
INPUT_PATH=@./image.json
SERVICE_HOSTNAME=$(kubectl get inferenceservice $SERVICE_NAME -o jsonpath='{.status.url}' | cut -d "/" -f 3)

curl -v -H "Host: ${SERVICE_HOSTNAME}" -d $INPUT_PATH http://${INGRESS_HOST}:${INGRESS_PORT}/v1/models/$MODEL_NAME:predict
```

結果:

```console
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 8080 (#0)
> POST /v1/models/cifar10:predict HTTP/1.1
> Host: torch-transformer.default.example.com
> User-Agent: curl/7.64.1
> Accept: */*
> Content-Length: 3394
> Content-Type: application/x-www-form-urlencoded
> Expect: 100-continue
>
Handling connection for 8080
< HTTP/1.1 100 Continue
* We are completely uploaded and fine
< HTTP/1.1 200 OK
< content-length: 222
< content-type: application/json; charset=UTF-8
< date: Thu, 03 Feb 2022 01:50:07 GMT
< server: istio-envoy
< x-envoy-upstream-service-time: 73
<
* Connection #0 to host localhost left intact
{"predictions": [[-1.192867636680603, -0.35750141739845276, -2.3665435314178467, 3.9186441898345947, -2.0592284202575684, 4.091977119445801, 0.1266237050294876, -1.8284690380096436, 2.628898859024048, -4.255198001861572]]}* Closing connection 0
```

## gRPC 和 REST 的性能比較

從以下轉換器和預測器的延遲統計數據中，您可以看到轉換器到預測器的調用對於 REST 比 gRPC 花費更長的時間（92 毫秒對 55 毫秒），REST 序列化和反序列化 3*32*32 形狀張量和使用 gRPC 需要更多時間作為緊密打包的 numpy 數組序列化字節傳輸。

```bash
# from REST v1 transformer log
2023-01-09 07:15:55.263 79476 root INFO [__call__():128] requestId: N.A., preprocess_ms: 6.083965302, explain_ms: 0, predict_ms: 92.653036118, postprocess_ms: 0.007867813
# from REST v1 predictor log
2023-01-09 07:16:02.581 79402 root INFO [__call__():128] requestId: N.A., preprocess_ms: 13.532876968, explain_ms: 0, predict_ms: 48.450231552, postprocess_ms: 0.006914139
```

```bash
# from REST v1 transformer log
2023-01-09 07:27:52.172 79715 root INFO [__call__():128] requestId: N.A., preprocess_ms: 2.567052841, explain_ms: 0, predict_ms: 55.0532341, postprocess_ms: 0.101804733
# from gPPC v2 predictor log
2023-01-09 07:27:52.171 79711 root INFO [__call__():128] requestId: , preprocess_ms: 0.067949295, explain_ms: 0, predict_ms: 51.237106323, postprocess_ms: 0.049114227
```
