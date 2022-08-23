# Cluster Generator

在 Argo CD 中，託管集群存儲在 Argo CD 命名空間的 [Secrets](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#clusters) 中。 ApplicationSet 控制器使用這些相同的 Secret 來生成參數來識別和定位可用的集群。

對於註冊進 Argo CD 的每個集群，集群生成器會根據在集群密鑰中找到的項目列表生成參數。

它會自動為每個集群的 `Application` 模板提供以下參數值：

- `name`
- `nameNormalized` （`name` 但規範化為僅包含小寫字母數字字符，`-` 或 `.`）
- `server`
- `metadata.labels.<key>` (對於 Secret 中的每個標籤)
- `metadata.annotations.<key>` (對於 Secret 中的每個註釋)

!!! info
    如果您的集群名稱包含對 Kubernetes 資源名稱無效的字符（例如下劃線），請使用 `nameNormalized` 參數。這可以防止呈現名稱為 `my_cluster-app1` 的無效 Kubernetes 資源，而是將它們轉換為 `my-cluster-app1`。

在 Argo CD 集群 Secrets 中是描述集群的數據字段：

```yaml
kind: Secret
data:
  # 在 Kubernetes 中，這些字段實際上是用 Base64 編碼的；為了方便起見，它們在這裡被解碼。
  # (它們在集群生成器作為參數傳遞時同樣被解碼)
  config: "{'tlsClientConfig':{'insecure':false}}"
  name: "in-cluster2"
  server: "https://kubernetes.default.svc"
metadata:
  labels:
    argocd.argoproj.io/secret-type: cluster
# (...)
```

Cluster Generator 將自動識別使用 Argo CD 定義的集群，並提取集群數據作為參數：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook
spec:
  generators:
  - clusters: {} # Automatically use all clusters defined within Argo CD
  template:
    metadata:
      name: '{{name}}-guestbook' # 'name' field of the Secret
    spec:
      project: "default"
      source:
        repoURL: https://github.com/argoproj/argocd-example-apps/
        targetRevision: HEAD
        path: guestbook
      destination:
        server: '{{server}}' # 'server' field of the secret
        namespace: guestbook
```

(完整的例子可以在[這裡](https://github.com/argoproj/applicationset/tree/master/examples/cluster)找到。)

在此示例中，集群密鑰的 `name` 和 `server` 字段用於填充 `Application` 資源 `name` 和 `server`（然後用於定位同一集群）。

## Label selector

Label selector 可用於將目標集群的範圍縮小到僅匹配特定標籤的集群：

```yaml
kind: ApplicationSet
metadata:
  name: guestbook
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          staging: true
  template:
  # (...)
```

這將匹配包含以下內容的 Argo CD 集群 secret：

```yaml
kind: Secret
data:
  # (... fields as above ...)
metadata:
  labels:
    argocd.argoproj.io/secret-type: cluster
    staging: "true"
# (...)
```

Cluster selector 還支持基於 set-based 的需求，供[多個核心 Kubernetes 資源使用](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#resources-that-support-set-based-requirements)。

## Deploying to the local cluster

在 Argo CD 中，“本地集群”是安裝 Argo CD（和 ApplicationSet 控制器）的集群。這是為了將其與“遠程集群”區分開來，後者是那些以[聲明方式](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#clusters)或通過 [Argo CD CLI](https://argo-cd.readthedocs.io/en/stable/getting_started/#5-register-a-cluster-to-deploy-apps-to-optional) 添加到 Argo CD 的集群。

對於與集群選擇器匹配的每個集群，集群生成器將自動定位本地和非本地集群。

如果您希望僅針對應用程序的遠程集群（例如，您想排除本地集群），則使用帶有標籤的集群選擇器，例如：

```yaml
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          argocd.argoproj.io/secret-type: cluster
```

此 selector 將與默認本地集群不匹配，因為默認本地集群沒有 Secret（因此該密鑰上沒有 `argocd.argoproj.io/secret-type` 標籤）。在該標籤上選擇的任何集群選擇器都將自動排除默認的本地集群。

但是，如果您確實希望同時針對本地和非本地集群，同時還使用標籤匹配，您可以在 Argo CD Web UI 中為本地集群創建一個秘密：

1. 在 Argo CD Web UI 中，選擇 `Settings`，然後選擇 `Clusters`。
2. 選擇您的本地集群，通常命名為 `in-cluster`。
3. 點擊 `Edit` 按鈕，然後將集群的名稱更改為另一個值，例如 `in-cluster-local`。這裡的任何其他值都可以。
4. 保持所有其他字段不變。
5. 點擊 `Save`。

這些步驟可能看起來違反直覺，但更改本地集群的默認值之一的行為會導致 Argo CD Web UI 為該集群創建一個新的秘密。在 Argo CD 命名空間中，您現在應該會看到一個名為 `cluster-(cluster suffix)`的 Secret 資源，其標籤為 `"argocd.argoproj.io/secret-type": "cluster"`。您也可以以聲明方式或使用 CLI 使用 `argocd cluster add "(context name)" --in-cluster`，而不是通過 Web UI。

## Pass additional key-value pairs via values field

您可以通過集群生成器的 `values` 字段傳遞額外的任意字符串鍵值對。通過值字段添加的值作為值添加。(field)

在此示例中，根據集群密鑰上的匹配標籤傳遞修訂參數值：

```yaml
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          type: 'staging'
      # A key-value map for arbitrary parameters
      values:
        revision: HEAD # staging clusters use HEAD branch
  - clusters:
      selector:
        matchLabels:
          type: 'production'
      values:
        # production uses a different revision value, for 'stable' branch
        revision: stable
  template:
    metadata:
      name: '{{name}}-guestbook'
    spec:
      project: "default"
      source:
        repoURL: https://github.com/argoproj/argocd-example-apps/
        # The cluster values field for each generator will be substituted here:
        targetRevision: '{{values.revision}}'
        path: guestbook
      destination:
        server: '{{server}}'
        namespace: guestbook
```

在此示例中，來自 `generators.clusters` 字段的修訂值作為 `values.revision` 傳遞到模板中，包含 `HEAD` 或 `stable`（基於生成參數集的生成器）。

!!! info
    前綴總是附加在通過 `generators.clusters.values` 字段提供的值之前。確保在使用模板時在參數名稱中包含此前綴。

    