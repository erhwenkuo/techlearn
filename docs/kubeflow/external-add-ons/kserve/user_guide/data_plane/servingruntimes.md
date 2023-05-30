# 模型推論 Runtime

KServe 使用兩個 CRD 來定義模型服務環境：

- `ServingRuntimes`
- `ClusterServingRuntimes`

兩者之間的唯一區別是一個是命名空間範圍的，另一個是集群範圍的。

`ServingRuntime` 為可以提供一種或多種特定模型格式的 Pod 定義模板。每個 `ServingRuntime` 都定義了運行時的容器鏡像、運行時支持的模型格式列表等關鍵信息。運行時的其他配置設置可以通過容器規範中的環境變量來傳達。

這些 CRD 允許提高靈活性和可擴展性，使用戶能夠快速定義或自定義可重用運行時，而無需修改控制器命名空間中的任何控制器代碼或任何資源。

以下是 `ServingRuntime` 的示例：

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: ServingRuntime
metadata:
  name: example-runtime
spec:
  supportedModelFormats:
    - name: example-format
      version: "1"
      autoSelect: true
  containers:
    - name: kserve-container
      image: examplemodelserver:latest
      args:
        - --model_dir=/mnt/models
        - --http_port=8080
```

KServe 提供了幾個開箱即用的 `ClusterServingRuntime`，這樣用戶就可以快速部署通用模型格式，而無需自己定義運行時。

|Name	|Supported Model Formats|
|-----|-----------------------|
|kserve-lgbserver	|LightGBM|
|kserve-mlserver	|SKLearn, XGBoost, LightGBM, MLflow|
|kserve-paddleserver	|Paddle|
|kserve-pmmlserver	|PMML|
|kserve-sklearnserver	|SKLearn|
|kserve-tensorflow-serving	|TensorFlow|
|kserve-torchserve	|PyTorch|
|kserve-tritonserver	|TensorFlow, ONNX, PyTorch, TensorRT|
|kserve-xgbserver	|XGBoost|

除了這些包含的運行時之外，您還可以通過添加自定義運行時來擴展 KServe 安裝。這在 [AMD Inference Server](https://kserve.github.io/website/0.10/modelserving/servingruntimes/v1beta1/amd/) 的示例中進行了演示。

## 規範屬性

`ServingRuntime` 規範中的可用屬性：

|Attribute	|Description|
|-----------|-----------|
|`multiModel`	|此 ServingRuntime 是否與 ModelMesh 兼容並用於多模型使用（與 KServe 單模型服務相對）。默認為 false|
|`disabled`	|禁用此運行時|
|`containers`	|與運行時關聯的容器列表|
|`containers[ ].image`	|當前容器的容器鏡像|
|`containers[ ].command`	|在提供的鏡像中找到的可執行命令|
|`containers[ ].args`	|作為字符串的命令行參數列表|
|`containers[ ].resources`	|Kubernetes limits 或 requests|
|`containers[ ].env`	|傳遞給容器的環境變量列表|
|`containers[ ].imagePullPolicy`	|容器鏡像拉取策略|
|`containers[ ].workingDir`	|當前容器的工作目錄|
|`containers[ ].livenessProbe`	|檢查容器 liveness 的探針|
|`containers[ ].readinessProbe`	|檢查容器 readness 的探針|
|`supportedModelFormats`	|當前運行時支持的模型類型列表|
|`supportedModelFormats[ ].name`	|模型格式名稱|
|`supportedModelFormats[ ].version`	|模型格式的版本。用於驗證運行時是否支持預測器。建議在此處僅包含主要版本，例如“1”而不是“1.15.4”|
|`storageHelper.disabled`	|禁用 storage helper|
|`nodeSelector`	|影響 Kubernetes 調度以將 Pod 分配給節點|
|`affinity`	|影響 Kubernetes 調度以將 Pod 分配給節點|
|`tolerations`	|允許 pod 被調度到具有匹配污點的節點上|

ModelMesh 利用此處未列出的其他欄位。更多信息在[這裡](https://github.com/kserve/modelmesh-serving/blob/main/docs/runtimes/custom_runtimes.md#spec-attributes)。

## 使用 ServingRuntimes

ServingRuntimes 可以顯式和隱式使用。

### 顯式：指定 ServingRuntime

當用戶在其 `InferenceServices` 中定義 `predictor` 時，他們可以明確指定 `ClusterServingRuntime` 或 `ServingRuntime` 的名稱。例如：

```yaml hl_lines="11"
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: example-sklearn-isvc
spec:
  predictor:
    model:
      modelFormat:
        name: sklearn
      storageUri: s3://bucket/sklearn/mnist.joblib
      runtime: kserve-mlserver
```

此處指定的運行時是 `kserve-mlserver`，因此 KServe 控制器將首先在命名空間中搜索具有該名稱的 `ServingRuntime`。如果不存在，控制器將搜索 `ClusterServingRuntimes` 列表。如果找到，控制器將首先驗證預測器中提供的模型格式是否在支持的模型格式列表中。如果是，那麼運行時提供的容器和 pod 信息將用於模型部署。

### 隱式：自動選擇

在 supportedModelFormats 列表的每個條目中，可以選擇指定 `autoSelect: true` 以指示如果沒有明確指定運行時，可以考慮給定的 `ServingRuntime` 自動選擇具有相應模型格式的預測器。例如，`kserve-sklearnserver` ClusterServingRuntime 支持 SKLearn 版本 1 並啟用了 `autoSelect`：

```yaml hl_lines="9"
apiVersion: serving.kserve.io/v1alpha1
kind: ClusterServingRuntime
metadata:
  name: kserve-sklearnserver
spec:
  supportedModelFormats:
    - name: sklearn
      version: "1"
      autoSelect: true
...
```

當部署以下 `InferenceService` 且未指定運行時時，控制器將查找支持 `sklearn` 的運行時：

```yaml hl_lines="9"
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: example-sklearn-isvc
spec:
  predictor:
    model:
      modelFormat:
        name: sklearn
      storageUri: s3://bucket/sklearn/mnist.joblib
```

由於 `kserve-sklearnserver` 在其 `supportedModelFormats` 列表中有一個 `sklearn` 和 `autoSelect: true` 條目，這個 `ClusterServingRuntime` 將用於模型部署。

如果還指定了版本：

```yaml
...
spec:
  predictor:
    model:
      modelFormat:
        name: sklearn
        version: "0"
...
```

然後，`supportedModelFormat` 的版本也必須匹配。在此示例中，`kserve-sklearnserver` 不符合選擇條件，因為它僅列出了對 sklearn 版本 `1` 的支持。

!!! warning
    如果多個運行時列出相同的格式和/或版本作為自動選擇，則無法保證將選擇哪個運行時。因此，用戶和集群管理員應謹慎啟用 `autoSelect`。

### 前一版本的 schema

目前，如果用戶使用舊模式部署預測器，您將 framework/format 指定為 key，則 KServe webhook 會自動將其映射到開箱即用的 `ClusterServingRuntimes` 之一。這是為了向後兼容。例如：

=== "Previous Schema"

    ```yaml
    apiVersion: serving.kserve.io/v1beta1
    kind: InferenceService
    metadata:
      name: example-sklearn-isvc
    spec:
      predictor:
        sklearn:
          storageUri: s3://bucket/sklearn/mnist.joblib
    ```

=== "Equivalent New Schema"

    ```yaml
    apiVersion: serving.kserve.io/v1beta1
    kind: InferenceService
    metadata:
      name: example-sklearn-isvc
    spec:
      predictor:
        model:
          modelFormat:
            name: sklearn
          storageUri: s3://bucket/sklearn/mnist.joblib
          runtime: kserve-sklearnserver
    ```

先前的 schema 將轉變為新 schema，其中明確指定了 `kserve-sklearnserver` ClusterServingRuntime。

!!! warning
    舊 schema 最終將被刪除，以支持新的模型規範，用戶可以在其中指定模型格式和可選的相應版本。在以前的 KServe 版本中，支持的預測器格式和容器鏡像是在控制平面命名空間的 ConfigMap 中定義的。從 v0.7、v0.8、v0.9 升級的現有 `InferenceServices` 需要轉換為新模型規範，因為預測器配置在 v0.10 中被淘汰。