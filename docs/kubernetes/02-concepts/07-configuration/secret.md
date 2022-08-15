# Secret

Secret 是一種包含少量敏感信息例如密碼、令牌或密鑰的對象。這樣的信息可能會被放在 Pod 規約中或者鏡像中。使用 Secret 意味著你不需要在應用程序代碼中包含機密數據。

由於創建 Secret 可以獨立於使用它們的 Pod， 因此在創建、查看和編輯 Pod 的工作流程中暴露 Secret（及其數據）的風險較小。 Kubernetes 和在集群中運行的應用程序也可以對 Secret 採取額外的預防措施， 例如避免將機密數據寫入非易失性存儲。

Secret 類似於 ConfigMap 但專門用於保存機密數據。

!!! warning
    注意：
      默認情況下，Kubernetes Secret 未加密地存儲在 API 服務器的底層數據存儲（etcd）中。任何擁有 API 訪問權限的人都可以檢索或修改 Secret，任何有權訪問 etcd 的人也可以。此外，任何有權限在命名空間中創建 Pod 的人都可以使用該訪問權限讀取該命名空間中的任何 Secret； 這包括間接訪問，例如創建 Deployment 的能力。

      為了安全地使用 Secret，請至少執行以下步驟：

      1. 為 Secret [啟用靜態加密](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/encrypt-data/)；
      2. [啟用或配置 RBAC 規則](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/authorization/)來限制讀取和寫入 Secret 的數據（包括通過間接方式）。需要注意的是，被准許創建 Pod 的人也隱式地被授權獲取 Secret 內容。
      3. 在適當的情況下，還可以使用 RBAC 等機制來限制允許哪些主體創建新 Secret 或替換現有 Secret。

參見 [Secret 的信息安全](https://kubernetes.io/zh-cn/docs/concepts/configuration/secret/#information-security-for-secrets)了解詳情。

## Secret 的使用

Pod 可以用三種方式之一來使用 Secret：

- 作為掛載到一個或多個容器上的捲 中的[文件](https://kubernetes.io/zh-cn/docs/concepts/configuration/secret/#using-secrets-as-files-from-a-pod)。
- 作為[容器的環境變量](https://kubernetes.io/zh-cn/docs/concepts/configuration/secret/#using-secrets-as-environment-variables)。
- 由 [kubelet 在為 Pod 拉取鏡像時使用](https://kubernetes.io/zh-cn/docs/concepts/configuration/secret/#using-imagepullsecrets)。
  
Kubernetes 控制面也使用 Secret； 例如，[引導令牌 Secret](https://kubernetes.io/zh-cn/docs/concepts/configuration/secret/#bootstrap-token-secrets) 是一種幫助自動化節點註冊的機制。

### Secret 的替代方案

除了使用 Secret 來保護機密數據，你也可以選擇一些替代方案。

下面是一些選項：

- 如果你的雲原生組件需要執行身份認證來訪問你所知道的、在同一 Kubernetes 集群中運行的另一個應用， 你可以使用 [ServiceAccount](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/authentication/#service-account-tokens) 及其令牌來標識你的客戶端身份。
  
- 你可以運行的第三方工具也有很多，這些工具可以運行在集群內或集群外，提供機密數據管理。例如，這一工具可能是 Pod 通過 HTTPS 訪問的一個服務，該服務在客戶端能夠正確地通過身份認證 （例如，通過 ServiceAccount 令牌）時，提供機密數據內容。
  
- 就身份認證而言，你可以為 X.509 證書實現一個定制的簽名者，並使用 [CertificateSigningRequest](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/certificate-signing-requests/) 來讓該簽名者為需要證書的 Pod 發放證書。
  
- 你可以使用一個[設備插件](https://kubernetes.io/zh-cn/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/) 來將節點本地的加密硬件暴露給特定的 Pod。例如，你可以將可信任的 Pod 調度到提供可信平台模塊（Trusted Platform Module，TPM）的節點上。這類節點是另行配置的。

你還可以將如上選項的兩種或多種進行組合，包括直接使用 Secret 對象本身也是一種選項。

例如：實現（或部署）一個 operator， 從外部服務取回生命期很短的會話令牌，之後基於這些生命期很短的會話令牌來創建 Secret。運行在集群中的 Pod 可以使用這些會話令牌，而 Operator 則確保這些令牌是合法的。這種責權分離意味著你可以運行那些不了解會話令牌如何發放與刷新的確切機制的 Pod。

## 使用 Secret

### 創建 Secret

- [使用 kubectl 命令來創建 Secret](https://kubernetes.io/zh-cn/docs/tasks/configmap-secret/managing-secret-using-kubectl/)
- [基於配置文件來創建 Secret](https://kubernetes.io/zh-cn/docs/tasks/configmap-secret/managing-secret-using-config-file/)
- [使用 kustomize 來創建 Secret](https://kubernetes.io/zh-cn/docs/tasks/configmap-secret/managing-secret-using-kustomize/)

#### 對 Secret 名稱與數據的約束

Secret 對象的名稱必須是合法的 [DNS 子域名](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/names#dns-subdomain-names)。

在為創建 Secret 編寫配置文件時，你可以設置 `data` 與/或 `stringData` 字段。 `data` 和 `stringData` 字段都是可選的。 `data` 字段中所有鍵值都必須是 base64 編碼的字符串。如果不希望執行這種 base64 字符串的轉換操作，你可以選擇設置 `stringData` 字段，其中可以使用任何字符串作為其取值。

`data` 和 `stringData` 中的鍵名只能包含字母、數字、`-`、`_` 或 `.` 字符。 `stringData` 字段中的所有鍵值對都會在內部被合併到 `data` 字段中。如果某個主鍵同時出現在 `data` 和 `stringData` 字段中，`stringData` 所指定的鍵值具有高優先級。

#### 尺寸限制

{==每個 Secret 的尺寸最多為 1 MiB==}。施加這一限制是為了避免用戶創建非常大的 Secret， 進而導致 API 服務器和 kubelet 內存耗盡。不過創建很多小的 Secret 也可能耗盡內存。你可以使用資源配額來約束每個名字空間中 Secret（或其他資源）的個數。

### 編輯 Secret

你可以使用 kubectl 來編輯一個已有的 Secret：

```bash
kubectl edit secrets mysecret
```

這一命令會啟動你的默認編輯器，允許你更新 data 字段中存放的 base64 編碼的 Secret 值； 例如：

```yaml
# 請編輯以下對象。以 `#` 開頭的幾行將被忽略，
# 且空文件將放棄編輯。如果保存此文件時出錯，
# 則重新打開此文件時也會有相關故障。
apiVersion: v1
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: { ... }
  creationTimestamp: 2020-01-22T18:41:56Z
  name: mysecret
  namespace: default
  resourceVersion: "164619"
  uid: cfee02d6-c137-11e5-8d73-42010af00002
type: Opaque
```

這一示例清單定義了一個 Secret，其 `data` 字段中包含兩個主鍵：`username` 和 `password`。清單中的字段值是 Base64 字符串，不過，當你在 Pod 中使用 Secret 時，kubelet 為 Pod 及其中的容器提供的是 **解碼** 後的數據。

你可以在一個 Secret 中打包多個主鍵和數值，也可以選擇使用多個 Secret， 完全取決於哪種方式最方便。

### 使用 Secret

Secret 可以以數據卷的形式掛載，也可以作為環境變量 暴露給 Pod 中的容器使用。 Secret 也可用於系統中的其他部分，而不是一定要直接暴露給 Pod。例如，Secret 也可以包含系統中其他部分在替你與外部系統交互時要使用的憑證數據。

Kubernetes 會檢查 Secret 的捲數據源，確保所指定的對象引用確實指向類型為 Secret 的對象。因此，如果 Pod 依賴於某 Secret，該 Secret 必須先於 Pod 被創建。

如果 Secret 內容無法取回（可能因為 Secret 尚不存在或者臨時性地出現 API 服務器網絡連接問題），kubelet 會周期性地重試 Pod 運行操作。 kubelet 也會為該 Pod 報告 Event 事件，給出讀取 Secret 時遇到的問題細節。

#### 可選的(Optional) Secret

當你定義一個基於 Secret 的環境變量時，你可以將其標記為可選。默認情況下，所引用的 Secret 都是必需的。

只有所有非可選的 Secret 都可用時，Pod 中的容器才能啟動運行。

如果 Pod 引用了 Secret 中的特定主鍵，而雖然 Secret 本身存在，對應的主鍵不存在， Pod 啟動也會失敗。

#### 在 Pod 中以文件形式使用 Secret

如果你希望在 Pod 中訪問 Secret 內的數據，一種方式是讓 Kubernetes 將 Secret 以 Pod 中一個或多個容器的文件系統中的文件的形式呈現出來。

要配置這種行為，你需要：

1. 創建一個 Secret 或者使用已有的 Secret。多個 Pod 可以引用同一個 Secret。
2. 更改 Pod 定義，在 `.spec.volumes[]` 下添加一個卷。根據需要為卷設置其名稱， 並將 `.spec.volumes[].secret.secretName` 字段設置為 Secret 對象的名稱。
3. 為每個需要該 Secret 的容器添加 `.spec.containers[].volumeMounts[]`。並將 `.spec.containers[].volumeMounts[].readOnly` 設置為 `true`， 將 `.spec.containers[].volumeMounts[].mountPath` 設置為希望 Secret 被放置的、目前尚未被使用的路徑名。
4. 更改你的鏡像或命令行，以便程序讀取所設置的目錄下的文件。 Secret 的 `data` 映射中的每個主鍵都成為 `mountPath` 下面的文件名。

下面是一個通過卷來掛載名為 mysecret 的 Secret 的 Pod 示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      optional: false # 默認設置，意味著 "mysecret" 必須已經存在
```

你要訪問的每個 Secret 都需要通過 `.spec.volumes` 來引用。

如果 Pod 中包含多個容器，則每個容器需要自己的 `volumeMounts` 塊， 不過針對每個 Secret 而言，只需要一份 `.spec.volumes` 設置。

!!! info
    說明：
      Kubernetes v1.22 版本之前都會自動創建用來訪問 Kubernetes API 的憑證。這一老的機制是基於創建可被掛載到 Pod 中的令牌 Secret 來實現的。在最近的版本中，包括 Kubernetes v1.24 中，API 憑據是直接通過 [TokenRequest API](https://kubernetes.io/zh-cn/docs/reference/kubernetes-api/authentication-resources/token-request-v1/) 來獲得的，這一憑據會使用[投射卷](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/service-accounts-admin/#bound-service-account-token-volume) 掛載到 Pod 中。使用這種方式獲得的令牌有確定的生命期，並且在掛載它們的 Pod 被刪除時自動作廢。

      你仍然可以[手動創建](https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/configure-service-account/#manually-create-a-service-account-api-token) 服務賬號令牌。例如，當你需要一個永遠都不過期的令牌時。不過，仍然建議使用 [TokenRequest](https://kubernetes.io/zh-cn/docs/reference/kubernetes-api/authentication-resources/token-request-v1/) 子資源來獲得訪問 API 服務器的令牌。你可以使用 `kubectl create token` 命令調用 `TokenRequest` API 獲得令牌。

#### 將 Secret 鍵投射到特定目錄

你也可以控制 Secret 鍵所投射到的卷中的路徑。你可以使用 `.spec.volumes[].secret.items` 字段來更改每個主鍵的目標路徑：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      items:
      - key: username
        path: my-group/my-username
```

將發生的事情如下：

- `mysecret` 中的鍵 `username` 會出現在容器中的路徑為 `/etc/foo/my-group/my-username`， 而不是 `/etc/foo/username`。
- Secret 對象的 `password` 鍵不會被投射。

如果使用了 `.spec.volumes[].secret.items`，則只有 `items` 中指定了的主鍵會被投射。如果要使用 Secret 中的所有主鍵，則需要將它們全部枚舉到 `items` 字段中。

如果你顯式地列舉了主鍵，則所列舉的主鍵都必須在對應的 Secret 中存在。否則所在的卷不會被創建。

#### Secret 文件的訪問權限

你可以為某個 Secret 主鍵設置 POSIX 文件訪問權限位。如果你不指定訪問權限，默認會使用 0644。你也可以為整個 Secret 卷設置默認的訪問模式，然後再根據需要在主鍵層面重載。

例如，你可以像下面這樣設置默認的模式：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      defaultMode: 0400
```

該 Secret 被掛載在 /etc/foo 下，Secret 卷掛載所創建的所有文件的訪問模式都是 0400。

#### 使用來自卷中的 Secret 值

在掛載了 Secret 卷的容器內，Secret 的主鍵都呈現為文件。 Secret 的取值都是 Base64 編碼的，保存在這些文件中。

下面是在上例中的容器內執行命令的結果：

```bash
ls /etc/foo/
```

輸出類似於：

```
username
password
```

```bash
cat /etc/foo/username
```

輸出類似於：

```
admin
```

```bash
cat /etc/foo/password
```


輸出類似於：

```
1f2d1e2e67df
```

容器中的程序要負責根據需要讀取 Secret 數據。

#### 掛載的 Secret 是被自動更新的

當卷中包含來自 Secret 的數據，而對應的 Secret 被更新，Kubernetes 會跟踪到這一操作並更新卷中的數據。更新的方式是保證最終一致性。

!!! info
    說明：
      對於以 subPath 形式掛載 Secret 卷的容器而言， 它們無法收到自動的 Secret 更新。

Kubelet 組件會維護一個緩存，在其中保存節點上 Pod 卷中使用的 Secret 的當前主鍵和取值。你可以配置 kubelet 如何檢測所緩存數值的變化。 kubelet 配置中的 `configMapAndSecretChangeDetectionStrategy` 字段控制 kubelet 所採用的策略。默認的策略是 Watch。

對 Secret 的更新操作既可以通過 API 的 watch 機制（默認）來傳播， 基於設置了生命期的緩存獲取，也可以通過 kubelet 的同步迴路來從集群的 API 服務器上輪詢獲取。

因此，從 Secret 被更新到新的主鍵被投射到 Pod 中，中間存在一個延遲。這一延遲的上限是 kubelet 的同步週期加上緩存的傳播延遲， 其中緩存的傳播延遲取決於所選擇的緩存類型。對應上一段中提到的幾種傳播機制，延遲時長為 watch 的傳播延遲、所配置的緩存 TTL 或者對於直接輪詢而言是零。

#### 以環境變量的方式使用 Secret

如果需要在 Pod 中以環境變量 的形式使用 Secret：

1. 創建 Secret（或者使用現有 Secret）。多個 Pod 可以引用同一個 Secret。
2. 更改 Pod 定義，在要使用 Secret 鍵值的每個容器中添加與所使用的主鍵對應的環境變量。讀取 Secret 主鍵的環境變量應該在 `env[].valueFrom.secretKeyRef` 中填寫 Secret 的名稱和主鍵名稱。
3. 更改你的鏡像或命令行，以便程序讀取環境變量中保存的值。
   
下面是一個通過環境變量來使用 Secret 的示例 Pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: mycontainer
    image: redis
    env:
      - name: SECRET_USERNAME
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: username
            optional: false # 此值為默認值；意味著 "mysecret"
                            # 必須存在且包含名為 "username" 的主鍵
      - name: SECRET_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: password
            optional: false # 此值為默認值；意味著 "mysecret"
                            # 必須存在且包含名為 "username" 的主鍵
  restartPolicy: Never
```

#### 非法環境變量

對於通過 `envFrom` 字段來填充環境變量的 Secret 而言， 如果其中包含的主鍵不能被當做合法的環境變量名，這些主鍵會被忽略掉。 Pod 仍然可以啟動。

如果你定義的 Pod 中包含非法的變量名稱，則 Pod 可能啟動失敗， 會形成 reason 為 `InvalidVariableNames` 的事件，以及列舉被略過的非法主鍵的消息。下面的例子中展示了一個 Pod，引用的是名為 mysecret 的 Secret， 其中包含兩個非法的主鍵：`1badkey` 和 `2alsobad`。

```bash
kubectl get events
```

輸出類似於：

```
LASTSEEN   FIRSTSEEN   COUNT     NAME            KIND      SUBOBJECT                         TYPE      REASON
0s         0s          1         dapi-test-pod   Pod                                         Warning   InvalidEnvironmentVariableNames   kubelet, 127.0.0.1      Keys [1badkey, 2alsobad] from the EnvFrom secret default/mysecret were skipped since they are considered invalid environment variable names.
```

#### 通過環境變量使用 Secret 值

在通過環境變量來使用 Secret 的容器中，Secret 主鍵展現為普通的環境變量。這些變量的取值是 Secret 數據的 Base64 解碼值。

下面是在前文示例中的容器內執行命令的結果：

```bash
echo "$SECRET_USERNAME"
```

輸出類似於：

```
admin
```

```bash
echo "$SECRET_PASSWORD"
```

輸出類似於：

```
1f2d1e2e67df
```

!!! info
    說明：
      如果容器已經在通過環境變量來使用 Secret，Secret 更新在容器內是看不到的， 除非容器被重啟。有一些第三方的解決方案，能夠在 Secret 發生變化時觸發容器重啟。


#### 容器鏡像拉取 Secret

如果你嘗試從私有倉庫拉取容器鏡像，你需要一種方式讓每個節點上的 kubelet 能夠完成與鏡像庫的身份認證。你可以配置 **鏡像拉取 Secret** 來實現這點。 Secret 是在 Pod 層面來配置的。

Pod 的 `imagePullSecrets` 字段是一個對 Pod 所在的名字空間中的 Secret 的引用列表。你可以使用 `imagePullSecrets` 來將鏡像倉庫訪問憑據傳遞給 kubelet。 kubelet 使用這個信息來替你的 Pod 拉取私有鏡像。參閱 [Pod API](https://kubernetes.io/zh-cn/docs/reference/kubernetes-api/workload-resources/pod-v1/#PodSpec) 參考 中的 PodSpec 進一步了解 `imagePullSecrets` 字段。

**使用 imagePullSecrets**

imagePullSecrets 字段是一個列表，包含對同一名字空間中 Secret 的引用。你可以使用 imagePullSecrets 將包含 Docker（或其他）鏡像倉庫密碼的 Secret 傳遞給 kubelet。 kubelet 使用此信息來替 Pod 拉取私有鏡像。參閱 PodSpec API 進一步了解 `imagePullSecrets` 字段。

**手動設定 imagePullSecret**

你可以通過閱讀[容器鏡像](https://kubernetes.io/zh-cn/docs/concepts/containers/images/#specifying-imagepullsecrets-on-a-pod) 文檔了解如何設置 `imagePullSecrets`。

**設置 imagePullSecrets 為自動掛載**

你可以手動創建 `imagePullSecret`，並在一個 ServiceAccount 中引用它。對使用該 ServiceAccount 創建的所有 Pod，或者默認使用該 ServiceAccount 創建的 Pod 而言，其 `imagePullSecrets` 字段都會設置為該服務賬號。請閱讀[向服務賬號添加 ImagePullSecrets](https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/configure-service-account/#add-imagepullsecrets-to-a-service-account) 來詳細了解這一過程。

#### 在靜態 Pod 中使用 Secret

你不可以在靜態 Pod 中使用 ConfigMap 或 Secret。

## 使用場景

### 使用場景：作為容器環境變量

創建 Secret：

```yaml title="mysecret.yaml"
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  USER_NAME: YWRtaW4=
  PASSWORD: MWYyZDFlMmU2N2Rm
```

```bash
kubectl apply -f mysecret.yaml
```

使用 `envFrom` 來將 Secret 的所有數據定義為容器的環境變量。來自 Secret 的主鍵成為 Pod 中的環境變量名稱：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-test-pod
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh", "-c", "env" ]
      envFrom:
      - secretRef:
          name: mysecret
  restartPolicy: Never
```

### 使用場景：帶 SSH 密鑰的 Pod

創建包含一些 SSH 密鑰的 Secret：

```bash
kubectl create secret generic ssh-key-secret --from-file=ssh-privatekey=/path/to/.ssh/id_rsa --from-file=ssh-publickey=/path/to/.ssh/id_rsa.pub
```

輸出類似於：

```
secret "ssh-key-secret" created
```

你也可以創建一個 `kustomization.yaml` 文件，在其 `secretGenerator` 字段中包含 SSH 密鑰。

現在你可以創建一個 Pod，在其中訪問包含 SSH 密鑰的 Secret，並通過卷的方式來使用它：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-test-pod
  labels:
    name: secret-test
spec:
  volumes:
  - name: secret-volume
    secret:
      secretName: ssh-key-secret
  containers:
  - name: ssh-test-container
    image: mySshImage
    volumeMounts:
    - name: secret-volume
      readOnly: true
      mountPath: "/etc/secret-volume"
```

容器命令執行時，秘鑰的數據可以在下面的位置訪問到：

```
/etc/secret-volume/ssh-publickey
/etc/secret-volume/ssh-privatekey
```

容器就可以隨便使用 Secret 數據來建立 SSH 連接。

### 使用場景：帶有生產、測試環境憑據的 Pod

這一示例所展示的一個 Pod 會使用包含生產環境憑據的 Secret，另一個 Pod 使用包含測試環境憑據的 Secret。

你可以創建一個帶有 `secretGenerator` 字段的 `kustomization.yaml` 文件或者運行 `kubectl create secret` 來創建 Secret。

```bash
kubectl create secret generic prod-db-secret --from-literal=username=produser --from-literal=password=Y4nys7f11
```

輸出類似於：

```
secret "prod-db-secret" created
```

你也可以創建一個包含測試環境憑據的 Secret：

```bash
kubectl create secret generic test-db-secret --from-literal=username=testuser --from-literal=password=iluvtests
```

輸出類似於：

```
secret "test-db-secret" created
```

現在生成 Pod：

```bash
cat <<EOF > pod.yaml
apiVersion: v1
kind: List
items:
- kind: Pod
  apiVersion: v1
  metadata:
    name: prod-db-client-pod
    labels:
      name: prod-db-client
  spec:
    volumes:
    - name: secret-volume
      secret:
        secretName: prod-db-secret
    containers:
    - name: db-client-container
      image: myClientImage
      volumeMounts:
      - name: secret-volume
        readOnly: true
        mountPath: "/etc/secret-volume"
- kind: Pod
  apiVersion: v1
  metadata:
    name: test-db-client-pod
    labels:
      name: test-db-client
  spec:
    volumes:
    - name: secret-volume
      secret:
        secretName: test-db-secret
    containers:
    - name: db-client-container
      image: myClientImage
      volumeMounts:
      - name: secret-volume
        readOnly: true
        mountPath: "/etc/secret-volume"
EOF
```

將 Pod 添加到同一 `kustomization.yaml` 文件中：

```bash
cat <<EOF >> kustomization.yaml
resources:
- pod.yaml
EOF
```

通過下面的命令在 API 服務器上應用所有這些對象：

```bash
kubectl apply -k .
```

兩個文件都會在其文件系統中出現下面的文件，文件中內容是各個容器的環境值：

```
/etc/secret-volume/username
/etc/secret-volume/password
```

注意這兩個 Pod 的規約中只有一個字段不同。這便於基於相同的 Pod 模板生成具有不同能力的 Pod。

你可以通過使用兩個服務賬號來進一步簡化這一基本的 Pod 規約：

1. `prod-user` 服務賬號使用 `prod-db-secret`
2. `test-user` 服務賬號使用 `test-db-secret`

Pod 規約簡化為：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: prod-db-client-pod
  labels:
    name: prod-db-client
spec:
  serviceAccount: prod-db-client
  containers:
  - name: db-client-container
    image: myClientImage
```

### 使用場景：在 Secret 卷中帶句點的文件

通過定義以句點（.）開頭的主鍵，你可以“隱藏”你的數據。這些主鍵代表的是以句點開頭的文件或“隱藏”文件。例如，當下面的 Secret 被掛載到 `secret-volume` 卷中時：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dotfile-secret
data:
  .secret-file: dmFsdWUtMg0KDQo=
---
apiVersion: v1
kind: Pod
metadata:
  name: secret-dotfiles-pod
spec:
  volumes:
  - name: secret-volume
    secret:
      secretName: dotfile-secret
  containers:
  - name: dotfile-test-container
    image: k8s.gcr.io/busybox
    command:
    - ls
    - "-l"
    - "/etc/secret-volume"
    volumeMounts:
    - name: secret-volume
      readOnly: true
      mountPath: "/etc/secret-volume"
```

卷中會包含一個名為 `.secret-file` 的文件，並且容器 `dotfile-test-container` 中此文件位於路徑 `/etc/secret-volume/.secret-file` 處。

!!! tip
    說明：
      以句點開頭的文件會在 `ls -l` 的輸出中被隱藏起來； 列舉目錄內容時你必須使用 `ls -la` 才能看到它們。

### 使用場景：僅對 Pod 中一個容器可見的 Secret

考慮一個需要處理 HTTP 請求，執行某些複雜的業務邏輯，之後使用 HMAC 來對某些消息進行簽名的程序。因為這一程序的應用邏輯很複雜， 其中可能包含未被注意到的遠程服務器文件讀取漏洞， 這種漏洞可能會把私鑰暴露給攻擊者。

這一程序可以分隔成兩個容器中的兩個進程：前端容器要處理用戶交互和業務邏輯， 但無法看到私鑰；簽名容器可以看到私鑰，並對來自前端的簡單簽名請求作出響應 （例如，通過本地主機網絡）。

採用這種劃分的方法，攻擊者現在必須欺騙應用服務器來做一些其他操作， 而這些操作可能要比讀取一個文件要復雜很多。

## Secret 的類型

創建 Secret 時，你可以使用 Secret 資源的 `type` 字段，或者與其等價的 `kubectl` 命令行參數（如果有的話）為其設置類型。 Secret 類型有助於對 Secret 數據進行編程處理。

Kubernetes 提供若干種內置的類型，用於一些常見的使用場景。針對這些類型，Kubernetes 所執行的合法性檢查操作以及對其所實施的限制各不相同。

|類型	|用例|
|---|---|
|**Opaque**	|預設的的類型。Base64 編碼格式的 Secret，用來存儲密碼、密鑰等；數據可通過 `base64 –decode` 解碼得到原始數據，所以加密性很弱。|
|**kubernetes.io/service-account-token**|用於被 serviceaccount 引用。 serviceaccout 創建時 Kubernetes 會默認創建對應的 secret。 Pod 如果使用了serviceaccount，對應的 secret 會自動掛載到 Pod 目錄 `/run/secrets/kubernetes.io/serviceaccount` 中。|
|**kubernetes.io/dockercfg**|`~/.dockercfg` 文件的序列化形式，用來存儲私有 docker registry 的認證信息|
|**kubernetes.io/dockerconfigjson**|`~/.docker/config.json` 文件的序列化形式，用來存儲私有 docker registry 的認證信息|
|**kubernetes.io/basic-auth**|用於基本身份認證 (Basic Auth)的憑據|
|**kubernetes.io/ssh-auth**|用於 SSH 身份認證的憑據|
|**kubernetes.io/tls**|用於 TLS 客戶端或者服務器端的數據|
|**bootstrap.kubernetes.io/token**|K8s 節點 bootstrap 令牌數據|

通過為 Secret 對象的 type 字段設置一個非空的字符串值，你也可以定義並使用自己 Secret 類型（如果 type 值為空字符串，則被視為 Opaque 類型）。

Kubernetes 並不對類型的名稱作任何限制。不過，如果你要使用內置類型之一， 則你必須滿足為該類型所定義的所有要求。

如果你要定義一種公開使用的 Secret 類型，請遵守 Secret 類型的約定和結構， 在類型名簽名添加域名，並用 `/` 隔開。例如：`cloud-hosting.example.net/cloud-api-credentials`。

### Opaque Secret

當 Secret 配置文件中未作顯式設定時，默認的 Secret 類型是 Opaque。當你使用 kubectl 來創建一個 Secret 時，你會使用 generic 子命令來標明要創建的是一個 Opaque 類型 Secret。例如，下面的命令會創建一個空的 Opaque 類型 Secret 對象：

```bash
kubectl create secret generic empty-secret
kubectl get secret empty-secret
```

輸出類似於

```
NAME           TYPE     DATA   AGE
empty-secret   Opaque   0      2m6s
```

`DATA` 列顯示 Secret 中保存的數據條目個數。在這個例子種，`0` 意味著你剛剛創建了一個空的 Secret。

### 服務賬號令牌 Secret

類型為 `kubernetes.io/service-account-token` 的 Secret 用來存放標識某服務賬號的令牌憑據。

從 v1.22 開始，這種類型的 Secret 不再被用來向 Pod 中加載憑據數據， 建議通過 `TokenRequest API`(https://kubernetes.io/zh-cn/docs/reference/kubernetes-api/authentication-resources/token-request-v1/) 來獲得令牌，而不是使用服務賬號令牌 Secret 對象。通過 `TokenRequest API` 獲得的令牌比保存在 Secret 對像中的令牌更加安全， 因為這些令牌有著被限定的生命期，並且不會被其他 API 客戶端讀取。你可以使用 `kubectl create token` 命令調用 `TokenRequest API` 獲得令牌。

只有在你無法使用 `TokenRequest API` 來獲取令牌， 並且你能夠接受因為將永不過期的令牌憑據寫入到可讀取的 API 對象而帶來的安全風險時， 才應該創建服務賬號令牌 Secret 對象。

使用這種 Secret 類型時，你需要確保對象的註解 `kubernetes.io/service-account-name` 被設置為某個已有的服務賬號名稱。如果你同時負責 ServiceAccount 和 Secret 對象的創建，應該先創建 ServiceAccount 對象。

當 Secret 對像被創建之後，某個 Kubernetes 控制器會填寫 Secret 的其它字段，例如 `kubernetes.io/service-account.uid` 註解以及 `data` 字段中的 `token` 鍵值，使之包含實際的令牌內容。

下面的配置實例聲明了一個服務賬號令牌 Secret：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-sa-sample
  annotations:
    kubernetes.io/service-account.name: "sa-name"
type: kubernetes.io/service-account-token
data:
  # 你可以像 Opaque Secret 一样在这里添加额外的键/值偶对
  extra: YmFyCg==
```

創建了 Secret 之後，等待 Kubernetes 在 `data` 字段中填充 token 主鍵。

參考 ServiceAccount 文檔了解服務賬號的工作原理。你也可以查看 Pod 資源中的 `automountServiceAccountToken` 和 `serviceAccountName` 字段文檔， 進一步了解從 Pod 中引用服務賬號憑據。

### Docker 配置 Secret

你可以使用下面兩種 type 值之一來創建 Secret，用以存放用於訪問容器鏡像倉庫的憑據：

- `kubernetes.io/dockercfg`
- `kubernetes.io/dockerconfigjson`

  kubernetes.io/dockercfg` 是一種保留類型，用來存放 `~/.dockercfg` 文件的序列化形式。該文件是配置 Docker 命令行的一種老舊形式。使用此 Secret 類型時，你需要確保 Secret 的 `data` 字段中包含名為 `.dockercfg` 的主鍵，其對應鍵值是用 base64 編碼的某 `~/.dockercfg` 文件的內容。

類型 `kubernetes.io/dockerconfigjson` 被設計用來保存 JSON 數據的序列化形式， 該 JSON 也遵從 `~/.docker/config.json` 文件的格式規則，而後者是 `~/.dockercfg` 的新版本格式。使用此 Secret 類型時，Secret 對象的 `data` 字段必須包含 `.dockerconfigjson` 鍵，其鍵值為 base64 編碼的字符串包含 `~/.docker/config.json` 文件的內容。

下面是一個 `kubernetes.io/dockercfg` 類型 Secret 的示例：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-dockercfg
type: kubernetes.io/dockercfg
data:
  .dockercfg: |
        "<base64 encoded ~/.dockercfg file>"
```

當你使用清單文件來創建這兩類 Secret 時，API 服務器會檢查 `data` 字段中是否存在所期望的主鍵， 並且驗證其中所提供的鍵值是否是合法的 JSON 數據。不過，API 服務器不會檢查 JSON 數據本身是否是一個合法的 Docker 配置文件內容。

當你沒有 Docker 配置文件，或者你想使用 `kubectl` 創建一個 Secret 來訪問容器倉庫時，你可以這樣做：

```bash
kubectl create secret docker-registry secret-tiger-docker \
  --docker-email=tiger@acme.example \
  --docker-username=tiger \
  --docker-password=pass1234 \
  --docker-server=my-registry.example:5000
```

上面的命令創建一個類型為 `kubernetes.io/dockerconfigjson` 的 Secret。如果你對 `.data.dockerconfigjson` 內容進行轉儲並執行 base64 解碼：

```bash
kubectl get secret secret-tiger-docker -o jsonpath='{.data.*}' | base64 -d
```

那麼輸出等價於這個 JSON 文檔（這也是一個有效的 Docker 配置文件）：

```json
{
  "auths": {
    "my-registry.example:5000": {
      "username": "tiger",
      "password": "pass1234",
      "email": "tiger@acme.example",
      "auth": "dGlnZXI6cGFzczEyMzQ="
    }
  }
}
```

### 基本身份認證 Secret

`kubernetes.io/basic-auth` 類型用來存放用於基本身份認證所需的憑據信息。使用這種 Secret 類型時，Secret 的 `data` 字段必須包含以下兩個鍵之一：

- `username`: 用於身份認證的用戶名；
- `password`: 用於身份認證的密碼或令牌。
- 
以上兩個鍵的鍵值都是 base64 編碼的字符串。當然你也可以在創建 Secret 時使用 `stringData` 字段來提供明文形式的內容。以下清單是基本身份驗證 Secret 的示例：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-basic-auth
type: kubernetes.io/basic-auth
stringData:
  username: admin      # kubernetes.io/basic-auth 類型的必需字段
  password: t0p-Secret # kubernetes.io/basic-auth 類型的必需字段
```

提供基本身份認證類型的 Secret 僅僅是出於方便性考慮。你也可以使用 Opaque 類型來保存用於基本身份認證的憑據。不過，使用預定義的、公開的 Secret 類型（`kubernetes.io/basic-auth`） 有助於幫助其他用戶理解 Secret 的目的，並且對其中存在的主鍵形成一種約定。 API 服務器會檢查 Secret 配置中是否提供了所需要的主鍵。

### SSH 身份認證 Secret

Kubernetes 所提供的內置類型 `kubernetes.io/ssh-auth` 用來存放 SSH 身份認證中所需要的憑據。使用這種 Secret 類型時，你就必須在其 `data` （或 `stringData`） 字段中提供一個 `ssh-privatekey` 鍵值對，作為要使用的 SSH 憑據。

下面的清單是一個 SSH 公鑰/私鑰身份認證的 Secret 示例：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-ssh-auth
type: kubernetes.io/ssh-auth
data:
  # 此例中的數據被截斷
  ssh-privatekey: |
          MIIEpQIBAAKCAQEAulqb/Y ...
```

提供 SSH 身份認證類型的 Secret 僅僅是出於用戶方便性考慮。你也可以使用 Opaque 類型來保存用於 SSH 身份認證的憑據。不過，使用預定義的、公開的 Secret 類型（ `kubernetes.io/ssh-auth`） 有助於其他人理解你的 Secret 的用途，也可以就其中包含的主鍵名形成約定。 API 服務器確實會檢查 Secret 配置中是否提供了所需要的主鍵。

### TLS Secret

Kubernetes 提供一種內置的 `kubernetes.io/tls` Secret 類型，用來存放 TLS 場合通常要使用的證書及其相關密鑰。 TLS Secret 的一種典型用法是為 Ingress 資源配置傳輸過程中的數據加密，不過也可以用於其他資源或者直接在負載中使用。當使用此類型的 Secret 時，Secret 配置中的 `data` （或 `stringData`）字段必須包含 `tls.key` 和 `tls.crt` 主鍵，儘管 API 服務器實際上並不會對每個鍵的取值作進一步的合法性檢查。`

下面的 YAML 包含一個 TLS Secret 的配置示例：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-tls
type: kubernetes.io/tls
data:
  # 此例中的數據被截斷
  tls.crt: |
        MIIC2DCCAcCgAwIBAgIBATANBgkqh ...
  tls.key: |
        MIIEpgIBAAKCAQEA7yn3bRHQ5FHMQ ...
```

提供 TLS 類型的 Secret 僅僅是出於用戶方便性考慮。你也可以使用 Opaque 類型來保存用於 TLS 服務器與/或客戶端的憑據。不過，使用內置的 Secret 類型的有助於對憑據格式進行歸一化處理，並且 API 服務器確實會檢查 Secret 配置中是否提供了所需要的主鍵。

當使用 `kubectl` 來創建 TLS Secret 時，你可以像下面的例子一樣使用 tls 子命令：

```bash
kubectl create secret tls my-tls-secret \
  --cert=path/to/cert/file \
  --key=path/to/key/file
```

這裡的公鑰/私鑰對都必須事先已存在。用於 `--cert` 的公鑰證書必須是 [RFC 7468 中 5.1 節](https://datatracker.ietf.org/doc/html/rfc7468#section-5.1) 中所規定的 DER 格式，且與 `--key` 所給定的私鑰匹配。私鑰必須是 DER 格式的 PKCS #8 （參見 [RFC 7468 第 11節](https://datatracker.ietf.org/doc/html/rfc7468#section-11)）。

!!! info
    說明：
      類型為 `kubernetes.io/tls` 的 Secret 中包含密鑰和證書的 DER 數據，以 Base64 格式編碼。如果你熟悉私鑰和證書的 PEM 格式，base64 與該格式相同，只是你需要略過 PEM 數據中所包含的第一行和最後一行。

      例如，對於證書而言，你 不要 包含 --------BEGIN CERTIFICATE----- 和 -------END CERTIFICATE---- 這兩行。

### 啟動引導令牌 Secret

通過將 Secret 的 type 設置為 `bootstrap.kubernetes.io/token` 可以創建啟動引導令牌類型的 Secret。這種類型的 Secret 被設計用來支持節點的啟動引導過程。其中包含用來為周知的 ConfigMap 簽名的令牌。

啟動引導令牌 Secret 通常創建於 `kube-system` 名字空間內，並以 `bootstrap-token-<令牌 ID>` 的形式命名； 其中 `<令牌 ID>` 是一個由 6 個字符組成的字符串，用作令牌的標識。

以 Kubernetes 清單文件的形式，某啟動引導令牌 Secret 可能看起來像下面這樣：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: bootstrap-token-5emitj
  namespace: kube-system
type: bootstrap.kubernetes.io/token
data:
  auth-extra-groups: c3lzdGVtOmJvb3RzdHJhcHBlcnM6a3ViZWFkbTpkZWZhdWx0LW5vZGUtdG9rZW4=
  expiration: MjAyMC0wOS0xM1QwNDozOToxMFo=
  token-id: NWVtaXRq
  token-secret: a3E0Z2lodnN6emduMXAwcg==
  usage-bootstrap-authentication: dHJ1ZQ==
  usage-bootstrap-signing: dHJ1ZQ==
```

啟動引導令牌類型的 Secret 會在 `data` 字段中包含如下主鍵：

- `token-id`：由 6 個隨機字符組成的字符串，作為令牌的標識符。必需。
- `token-secret`：由 16 個隨機字符組成的字符串，包含實際的令牌機密。必需。
- `description`：供用戶閱讀的字符串，描述令牌的用途。可選。
- `expiration`：一個使用 RFC3339 來編碼的 UTC 絕對時間，給出令牌要過期的時間。可選。
- `usage-bootstrap-<usage>`：布爾類型的標誌，用來標明啟動引導令牌的其他用途。
- `auth-extra-groups`：用逗號分隔的組名列表，身份認證時除被認證為 `system:bootstrappers` 組之外，還會被添加到所列的用戶組中。

上面的 YAML 文件可能看起來令人費解，因為其中的數值均為 base64 編碼的字符串。實際上，你完全可以使用下面的 YAML 來創建一個一模一樣的 Secret：

```yaml
apiVersion: v1
kind: Secret
metadata:
  # 注意 Secret 的命名方式
  name: bootstrap-token-5emitj
  # 啟動引導令牌 Secret 通常位於 kube-system 名字空間
  namespace: kube-system
type: bootstrap.kubernetes.io/token
stringData:
  auth-extra-groups: "system:bootstrappers:kubeadm:default-node-token"
  expiration: "2020-09-13T04:39:10Z"
  # 此令牌 ID 被用於生成 Secret 名稱
  token-id: "5emitj"
  token-secret: "kq4gihvszzgn1p0r"
  # 此令牌還可用於 authentication （身份認證）
  usage-bootstrap-authentication: "true"
  # 且可用於 signing （證書籤名）
  usage-bootstrap-signing: "true"
```

### 不可更改的 Secret

Kubernetes 允許你將特定的 Secret（和 ConfigMap）標記為 不可更改（Immutable）。禁止更改現有 Secret 的數據有下列好處：

防止意外（或非預期的）更新導致應用程序中斷
（對於大量使用 Secret 的集群而言，至少數万個不同的 Secret 供 Pod 掛載）， 通過將 Secret 標記為不可變，可以極大降低 kube-apiserver 的負載，提升集群性能。 `kubelet` 不需要監視那些被標記為不可更改的 Secret。

### 將 Secret 標記為不可更改

你可以通過將 Secret 的 `immutable` 字段設置為 `true` 創建不可更改的 Secret。例如：

```yaml
apiVersion: v1
kind: Secret
metadata:
  ...
data:
  ...
immutable: true
```

你也可以更改現有的 Secret，令其不可更改。

!!! info
    說明：
      一旦一個 Secret 或 ConfigMap 被標記為不可更改，撤銷此操作或者更改 data 字段的內容都是 不 可能的。只能刪除並重新創建這個 Secret。現有的 Pod 將維持對已刪除 Secret 的掛載點 -- 建議重新創建這些 Pod。

## Secret 的信息安全問題

儘管 ConfigMap 和 Secret 的工作方式類似，但 Kubernetes 對 Secret 有一些額外的保護。

Secret 通常保存重要性各異的數值，其中很多都可能會導致 Kubernetes 中 （例如，服務賬號令牌）或對外部系統的特權提升。即使某些個別應用能夠推導它期望使用的 Secret 的能力， 同一名字空間中的其他應用可能會讓這種假定不成立。

只有當某個節點上的 Pod 需要某 Secret 時，對應的 Secret 才會被發送到該節點上。如果將 Secret 掛載到 Pod 中，kubelet 會將數據的副本保存在在 tmpfs 中， 這樣機密的數據不會被寫入到持久性存儲中。一旦依賴於該 Secret 的 Pod 被刪除，kubelet 會刪除來自於該 Secret 的機密數據的本地副本。

同一個 Pod 中可能包含多個容器。默認情況下，你所定義的容器只能訪問默認 ServiceAccount 及其相關 Secret。你必須顯式地定義環境變量或者將捲映射到容器中，才能為容器提供對其他 Secret 的訪問。

針對同一節點上的多個 Pod 可能有多個 Secret。不過，只有某個 Pod 所請求的 Secret 才有可能對 Pod 中的容器可見。因此，一個 Pod 不會獲得訪問其他 Pod 的 Secret 的權限。

### 針對開發人員的安全性建議

- 應用在從環境變量或卷中讀取了機密信息內容之後仍要對其進行保護。例如， 你的應用應該避免用明文的方式將 Secret 數據寫入日誌，或者將其傳遞給不可信的第三方。
- 如果你在一個 Pod 中定義了多個容器，而只有一個容器需要訪問某 Secret， 定義卷掛載或環境變量配置時，應確保其他容器無法訪問該 Secret。
- 如果你通過清單來配置某 Secret， Secret 數據以 Base64 的形式編碼，將此文件共享，或者將其檢入到某源碼倉庫， 都意味著 Secret 對於任何可以讀取清單的人都是可見的。 Base64 - 編碼 不是 一種加密方法，與明文相比沒有任何安全性提升。
- 部署與 Secret API 交互的應用時，你應該使用 [RBAC](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/rbac/) 這類[鑑權策略](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/authorization/)來限制訪問。
- 在 Kubernetes API 中，名字空間內對 Secret 對象的 `watch` 和 `list` 請求是非常強大的能力。在可能的時候應該避免授予這類訪問權限，因為通過列舉 Secret， 客戶端能夠查看對應名字空間內所有 Secret 的取值。

### 針對集群管理員的安全性建議

- 保留（使用 Kubernetes API）對集群中所有 Secret 對象執行 `watch` 或 `list` 操作的能力， 這樣只有特權級最高、系統級別的組件能夠執行這類操作。
- 在部署需要通過 Secret API 交互的應用時，你應該通過使用 RBAC 這類鑑權策略來限制訪問。
- 在 API 服務器上，對象（包括 Secret）會被持久化到 etcd 中； 因此：
    - 只應准許集群管理員訪問 etcd（包括只讀訪問）；
    - 為 Secret 對象啟用[靜態加密](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/encrypt-data/)， 這樣這些 Secret 的數據就不會以明文的形式保存到 etcd 中；
    - 當 etcd 的持久化存儲不再被使用時，請考慮徹底擦除存儲介質；
    - 如果存在多個 etcd 實例，請確保 etcd 使用 SSL/TLS 來完成其對等通信。