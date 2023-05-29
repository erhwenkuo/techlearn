# InferenceService 初試

## 運行 InferenceService

在本教程中，您將部署一個帶有預測器的 `InferenceService`，該預測器將加載使用 [iris 數據集](https://archive.ics.uci.edu/ml/datasets/iris)訓練的 `scikit-learn` 模型。

該數據集具有三個輸出類別：

- Iris Setosa
- Iris Versicolour
- Iris Virginica。

然後，您將向部署的模型發送推理請求，以便預測您的請求對應的鳶尾花類別。

由於您的模型被部署為 `InferenceService`，而不是原始的 Kubernetes 服務，您只需提供模型的存儲位置，它就會獲得一些開箱即用的超能力🚀。


### 1. 創建命名空間

首先，創建一個用於部署 KServe 資源的命名空間：

```bash
kubectl create namespace kserve-test
```

結果:

```bash
namespace/kserve-test created
```

如果你是使用 Kubeflow, 那麼就可直接使用特定使用者的命名空間, 比如: `kubeflow-user-example-com`

### 2. 創建 `InferenceService`

接下來，為模型定義一個新的 `InferenceService` YAML 並將其應用於 Kubernetes 集群。

```yaml
kubectl apply -n kserve-test -f - <<EOF
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
EOF
```

結果:

```bash
inferenceservice.serving.kserve.io/sklearn-iris created
```

### 3. 檢查 `InferenceService` 狀態。

```bash
kubectl get inferenceservices sklearn-iris -n kserve-test
```

結果:

```bash
NAME           URL                                                READY   PREV   LATEST   PREVROLLEDOUTREVISION   LATESTREADYREVISION                    AGE
sklearn-iris   http://sklearn-iris.kserve-test.example.it   True           100                              sklearn-iris-predictor-default-00001   27m
```

如果您的 DNS 包含 `example.com`，請諮詢您的管理員以配置 DNS 或使用[自定義網域](https://knative.dev/docs/serving/using-a-custom-domain)。

??? tip "配置域名"

    您可以自定義單個 Knative 服務的 domain，或為集群設置全局 default domain。默認情況下，路由的完全限定域名為 `{route}.{namespace}.svc.cluster.local`。

    **為單個 Knative 服務配置 domain**

    如果要定義單一個服務的 domain，請參閱有關 [DomainMapping](https://knative.dev/docs/serving/services/custom-domains/) 的文檔。

    **為集群上的所有 Knative 服務配置默認 domain**

    您可以通過修改 [`config-domain` ConfigMap](https://github.com/knative/serving/blob/main/config/core/configmaps/domain.yaml) 來更改集群上所有 Knative 服務的默認域。

    步驟:

    1. 在文本編輯器中打開 `config-domain` ConfigMap：

    ```bash
    kubectl edit configmap config-domain -n knative-serving
    ```

    2. 編輯該文件以將 `svc.cluster.local` 替換為您要使用的 domain，然後刪除 `_example` 鍵並保存您的更改。在此示例中，`istio.example.it` 配置為所有路由的 domain：

    ```yaml
    apiVersion: v1
    data:
      example.it: ""
    kind: ConfigMap
    [...]
    ```


### 4. 確定 ingress IP 和端口

執行以下命令以確定您的 kubernetes 集群是否運行在支持外部負載均衡器的環境中

```bash
kubectl get svc istio-ingressgateway -n istio-system
```

結果:

```bash
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                                      AGE
istio-ingressgateway   LoadBalancer   10.43.211.196   172.20.0.10   15021:30544/TCP,80:30665/TCP,443:31348/TCP,31400:30165/TCP,15443:30738/TCP   8d
```

=== "Load Balancer"

    如果設置了 EXTERNAL-IP 值，則您的環境具有可用於入口網關的外部負載均衡器。

    ```bash
    export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
    export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
    ```

    驗證:

    ```bash
    $ echo $INGRESS_HOST
    172.20.0.10
    $ echo $INGRESS_PORT
    80
    ```

=== "Port Forward"

    或者，您可以出於測試目的進行端口轉發。

    ```bash
    INGRESS_GATEWAY_SERVICE=$(kubectl get svc --namespace istio-system --selector="app=istio-ingressgateway" --output jsonpath='{.items[0].metadata.name}')
    kubectl port-forward --namespace istio-system svc/${INGRESS_GATEWAY_SERVICE} 8080:80
    ```

    打開另一個終端，輸入以下內容：

    ```bash
    export INGRESS_HOST=localhost
    export INGRESS_PORT=8080
    ```

### 5. 執行模型推理

首先，在文件中準備您的推理輸入請求：

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

根據您的設置，使用以下命令之一來呼叫 InferenceService：

=== "From Ingress gateway with HOST Header"

    如果您沒有 DNS，您仍然可以使用 HOST 標頭使用入口網關外部 IP 進行 curl。

    ```bash
    SERVICE_HOSTNAME=$(kubectl get inferenceservice sklearn-iris -n kserve-test -o jsonpath='{.status.url}' | cut -d "/" -f 3)
    curl -v -H "Host: ${SERVICE_HOSTNAME}" "http://${INGRESS_HOST}:${INGRESS_PORT}/v1/models/sklearn-iris:predict" -d @./iris-input-v1.json
    ```

    結果:

    ```json
    {"predictions":[1,1]}
    ```

### 6. 運行性能測試（optional）

如果要對已部署的模型進行負載測試，請嘗試部署以下 Kubernetes 作業以驅動模型的負載：

```bash
# use kubectl create instead of apply because the job template is using generateName which doesn't work with kubectl apply
kubectl create -f https://raw.githubusercontent.com/kserve/kserve/release-0.8/docs/samples/v1beta1/sklearn/v1/perf.yaml -n kserve-test
```

檢查 job:

```bash
kubectl get jobs -n kserve-test
```

結果:

```bash
NAME             COMPLETIONS   DURATION   AGE
load-testhd5bg   1/1           70s        74s
```

執行以下命令查看輸出：

```
kubectl logs job/load-testhd5bg -n kserve-test
```

結果:

```bash
Requests      [total, rate, throughput]         30000, 500.02, 498.65
Duration      [total, attack, wait]             1m0s, 59.998s, 162.614ms
Latencies     [min, mean, 50, 90, 95, 99, max]  2.037ms, 63.362ms, 49.049ms, 107.261ms, 145.613ms, 197.936ms, 395.942ms
Bytes In      [total, mean]                     630052, 21.00
Bytes Out     [total, mean]                     2460000, 82.00
Success       [ratio]                           100.00%
Status Codes  [code:count]                      200:29999  502:1  
Error Set:
502 Bad Gateway
```

