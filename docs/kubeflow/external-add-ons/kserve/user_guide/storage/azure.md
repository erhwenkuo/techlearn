# 在 Azure 上使用保存的模型部署 InferenceService

## 使用公共 Azure Blob

默認情況下，KServe 使用匿名客戶端下載工件。要指向 Azure Blob，請指定 `StorageUri` 以指向 Azure Blob 存儲，格式為：`https://{$STORAGE_ACCOUNT_NAME}.blob.core.windows.net/{$CONTAINER}/{$PATH}`

- e.g. `https://modelstoreaccount.blob.core.windows.net/model-store/model.joblib`

## 使用私有 Blob

KServe 支持使用 Azure 服務原則進行身份驗證。

創建授權的 Azure 服務主體

- 要創建 Azure 服務原則，請按照[此處](https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli?view=azure-cli-latest)的步驟操作。
- 為 SP 分配您的 blob 上的存儲 Storage Blob Data Owner（KServe 需要此權限，因為它需要在 blob 路徑中列出內容以過濾要下載的項目）。
- 在[此處](https://docs.microsoft.com/en-us/azure/storage/common/storage-auth-aad)分配存儲角色的詳細信息。

```bash
az ad sp create-for-rbac --name model-store-sp --role "Storage Blob Data Owner" \
    --scopes /subscriptions/2662a931-80ae-46f4-adc7-869c1f2bcabf/resourceGroups/cognitive/providers/Microsoft.Storage/storageAccounts/modelstoreaccount
```

## 創建 Azure Secret 並附加到 Service Account

### 創建 Azure secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: azcreds
type: Opaque
stringData: # use `stringData` for raw credential string or `data` for base64 encoded string
  AZ_CLIENT_ID: xxxxx
  AZ_CLIENT_SECRET: xxxxx
  AZ_SUBSCRIPTION_ID: xxxxx
  AZ_TENANT_ID: xxxxx
```

### 附加 Ｓecret 到 Service Account

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa
secrets:
- name: azcreds
```

```bash
kubectl apply -f create-azure-secret.yaml
```

## 使用 InferenceService 在 Azure 上部署模型

使用 azure `storageUri` 和附加了 azure 憑據的服務帳戶創建 `InferenceService`。

```yaml
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "sklearn-azure"
spec:
  predictor:
    serviceAccountName: sa
    model:
      modelFormat:
        name: sklearn
      storageUri: "https://modelstoreaccount.blob.core.windows.net/model-store/model.joblib"
```

應用 `sklearn-azure.yaml`。

```bash
kubectl apply -f sklearn-azure.yaml
```

## 運行預測

現在，可以通過 `${INGRESS_HOST}:${INGRESS_PORT}` 訪問入口，或按照此說明查找入口 IP 和端口。

```bash
SERVICE_HOSTNAME=$(kubectl get inferenceservice sklearn-azure -o jsonpath='{.status.url}' | cut -d "/" -f 3)

MODEL_NAME=sklearn-azure
INPUT_PATH=@./input.json
curl -v -H "Host: ${SERVICE_HOSTNAME}" http://${INGRESS_HOST}:${INGRESS_PORT}/v1/models/$MODEL_NAME:predict -d $INPUT_PATH
```

```bash
*   Trying 127.0.0.1:8080...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8080 (#0)
> POST /v1/models/sklearn-azure:predict HTTP/1.1
> Host: sklearn-azure.default.example.com
> User-Agent: curl/7.68.0
> Accept: */*
> Content-Length: 84
> Content-Type: application/x-www-form-urlencoded
>
* upload completely sent off: 84 out of 84 bytes
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< content-length: 23
< content-type: application/json; charset=UTF-8
< date: Mon, 20 Sep 2021 04:55:50 GMT
< server: istio-envoy
< x-envoy-upstream-service-time: 6
<
* Connection #0 to host localhost left intact
{"predictions": [1, 1]}
```
