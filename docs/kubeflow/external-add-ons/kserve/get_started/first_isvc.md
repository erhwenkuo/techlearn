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

如果你是使用 Kubeflow, 那麼就可直接使用特定使用者的命名空間, 比如: `kubeflow-user-example-com`

### 2. 創建 `InferenceService`

接下來，為模型定義一個新的 `InferenceService` YAML 並將其應用於 Kubernetes 集群。

```yaml
kubectl apply -f - <<EOF
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

### 3. 檢查 `InferenceService` 狀態。

```bash
kubectl get inferenceservices sklearn-iris
```

結果:

```bash
NAME           URL                                                               READY   PREV   LATEST   PREVROLLEDOUTREVISION   LATESTREADYREVISION                    AGE
sklearn-iris   http://sklearn-iris.kubeflow-user-example-com.svc.cluster.local   True           100                              sklearn-iris-predictor-default-00001   2m44s
```

如果您的 DNS 包含 `example.com`，請諮詢您的管理員以配置 DNS 或使用[自定義網域](https://knative.dev/docs/serving/using-a-custom-domain)。

### 4. 確定 ingress IP 和端口

執行以下命令以確定您的 kubernetes 集群是否運行在支持外部負載均衡器的環境中

```bash
kubectl get svc istio-ingressgateway -n istio-system
```

結果:

```bash
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                                      AGE
istio-ingressgateway   LoadBalancer   10.43.211.196   172.20.0.15   15021:30544/TCP,80:30665/TCP,443:31348/TCP,31400:30165/TCP,15443:30738/TCP   8d
```

### 5. 執行模型推理

首先，在文件中準備您的推理輸入請求：

```bash
cat <<EOF > "./iris-input.json"
{
  "instances": [
    [6.8,  2.8,  4.8,  1.4],
    [6.0,  3.4,  4.5,  1.6]
  ]
}
EOF
```

根據您的設置，使用以下命令之一來呼叫 InferenceService：

```bash
curl -v http://sklearn-iris.kubeflow-user-example-com.172.20.0.15.xip.io/v1/models/sklearn-iris:predict -d @./iris-input.json
```

```bash
curl -v http://sklearn-iris.kubeflow-user-example-com.svc.cluster.local/v1/models/sklearn-iris:predict -d @./iris-input.json
```

### 6. 運行性能測試（optional）

如果要對已部署的模型進行負載測試，請嘗試部署以下 Kubernetes 作業以驅動模型的負載：

```bash
# use kubectl create instead of apply because the job template is using generateName which doesn't work with kubectl apply
kubectl create -f https://raw.githubusercontent.com/kserve/kserve/release-0.8/docs/samples/v1beta1/sklearn/v1/perf.yaml -n kserve-test
```

執行以下命令查看輸出：

```
kubectl logs load-test8b58n-rgfxr -n kserve-test
```
