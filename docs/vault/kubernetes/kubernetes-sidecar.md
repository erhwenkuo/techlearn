# 通過 Vault Agent 容器將 Secret 注入 Kubernetes Pod

原文: https://learn.hashicorp.com/tutorials/vault/kubernetes-sidecar

![](./assets/vault-injector.png)

部署使用 Vault 來保存與應用帳密的應用程序需要：

- 驗證並獲取客戶端令牌。
- 管理令牌的生命週期。
- 從 Vault 中檢索秘密。
- 管理任何動態機密的租約。

`Vault Agent` 負責這些任務並使您的應用程序感受不到 Vault 的存在。這樣的整合引入了一項新要求，也就是將 `Vault Agent` 與應用程序一起安裝和配置為 `sidecar`。

`Vault Helm chart` 使您能夠運行 `Vault` 和 `Vault Agent Injector` 服務。此注入器服務利用 Kubernetes mutating admission webhook 來攔截定義特定註釋的 pod，並註入 Vault Agent 容器來管理這些機密。這是很有幫助，因為：

- 應用程序仍然不知道 Vault，因為機密存儲在其容器的文件系統中。
- 現有部署無需更改；因為可以修補註釋。
- 可以通過 Kubernetes 服務帳戶和命名空間強制訪問機密

在本教程中，您將使用 Vault Helm chart 設置 Vault 和此注入器服務。然後，您將部署幾個應用程序來演示這個新的注入器服務如何檢索和寫入這些秘密以供應用程序使用。



## 先決條件

本教程需要安裝 `Kubernetes 命令行界面 (CLI)` 和 `Helm CLI`、`Minikube` 以及其他配置以將它們組合在一起。

## 啟動 Kubernetes

=== "Minikube"

    Minikube 是一個 CLI 工具，可在系統上的虛擬機 (VM) 中本地配置和管理單節點 Kubernetes 集群的生命週期。

    啟動 Kubernetes 集群。

    ```bash
    $ minikube start

    😄  minikube v1.19.0 on Darwin 11.2.3
    ✨  Using the docker driver based on existing profile
    👍  Starting control plane node minikube in cluster minikube
    🔄  Restarting existing docker container for "minikube" ...
    🐳  Preparing Kubernetes v1.20.2 on Docker 20.10.5 ...
    🔎  Verifying Kubernetes components...
        ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
    🌟  Enabled addons: storage-provisioner, default-storageclass
    🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
    ```

    初始化過程需要幾分鐘，因為它會檢索任何必要的依賴項並執行各種容器映像。

    驗證 Minikube 集群的狀態。

    ```bash
    $ minikube status
    minikube
    type: Control Plane
    host: Running
    kubelet: Running
    apiserver: Running
    kubeconfig: Configured
    ```

    主機、kubelet 和 apiserver 報告它們正在運行。 kubectl 是一個命令行界面 (CLI)，用於針對 Kubernetes 集群運行命令，它也被配置為與這個最近啟動的集群進行通信。

    Minikube 在基於 Web 的儀表板中提供狀態的可視化表示。該界面在可視化界面中顯示集群活動，有助於深入研究影響它的問題。

    在另一個終端中，啟動 minikube 儀表板。

    ```bash
    $ minikube dashboard
    ```

    操作系統的默認瀏覽器打開並顯示儀表板。

    ![](./assets/k8s-dashboard.png)

=== "K3D"

    k3d 是一個輕量級的 kubernetes 包裝器，用於在 docker 中運行 k3s（Rancher Lab 的最小 Kubernetes 發行版）。

    k3d 使得在 docker 中創建單節點和多節點 k3s 集群變得非常容易，例如用於 Kubernetes 上的本地開發。

    啟動 Kubernetes 集群。

    ```bash
    $ mkdir -p /tmp/k3d/kubelet/pods
    $ k3d cluster create -v /tmp/k3d/kubelet/pods:/var/lib/kubelet/pods:shared

    WARN[0000] No node filter specified                     
    INFO[0000] Prep: Network                                
    INFO[0000] Created network 'k3d-k3s-default'            
    INFO[0000] Created image volume k3d-k3s-default-images  
    INFO[0000] Starting new tools node...                   
    INFO[0000] Starting Node 'k3d-k3s-default-tools'        
    INFO[0001] Creating node 'k3d-k3s-default-server-0'     
    INFO[0001] Creating LoadBalancer 'k3d-k3s-default-serverlb' 
    INFO[0001] Using the k3d-tools node to gather environment information 
    INFO[0001] HostIP: using network gateway 172.29.0.1 address 
    INFO[0001] Starting cluster 'k3s-default'               
    INFO[0001] Starting servers...                          
    INFO[0001] Starting Node 'k3d-k3s-default-server-0'     
    INFO[0005] All agents already running.                  
    INFO[0005] Starting helpers...                          
    INFO[0006] Starting Node 'k3d-k3s-default-serverlb'     
    INFO[0012] Injecting records for hostAliases (incl. host.k3d.internal) and for 2 network members into CoreDNS configmap... 
    INFO[0014] Cluster 'k3s-default' created successfully!  
    INFO[0014] You can now use it like this:                
    kubectl cluster-info
    ```

    驗證 K3D 集群的狀態。

    ```bash
    $ kubectl cluster-info
    ```

## 安裝 Vault Helm chart

在 Kubernetes 上運行 Vault 的推薦方法是通過 [Helm chart](https://www.vaultproject.io/docs/platform/k8s/helm)。 Helm 是一個包管理器，它安裝和配置所有必要的組件以在幾種不同的模式下運行 Vault。 Helm chart 包括啟用條件和參數化執行的模板。這些參數可以通過命令行參數設置或在 YAML 中定義。

添加 HashiCorp Helm 存儲庫。

```bash
$ helm repo add hashicorp https://helm.releases.hashicorp.com
```

更新所有存儲庫以確保 helm 知道最新版本。

```bash
$ helm repo update
```

安裝在開發模式下運行的最新版本的 Vault 服務器。

```bash
$ helm install vault hashicorp/vault --set "server.dev.enabled=true"
```

Vault pod 和 Vault Agent Injector pod 部署在默認命名空間中。

顯示默認命名空間中的所有 pod。

```bash
$ kubectl get pods

NAME                                    READY   STATUS    RESTARTS   AGE
vault-0                                 1/1     Running   0          80s
vault-agent-injector-5945fb98b5-tpglz   1/1     Running   0          80s
```

`vault-0` pod 在開發模式下運行 Vault 服務器。 `vault-agent-injector` pod 根據部署中存在或修補的註釋執行注入。

!!! info
    開發模式：在開發中運行 Vault 服務器會自動初始化和解封。這在學習環境中是理想的，但不建議在生產環境中使用。

等到 `vault-0` pod 和 `vault-agent-injector` pod 正在運行並準備就緒 (1/1)。

## 在 Vault 中設定 Secret

您在將機密注入 pod 部分中部署的應用程序期望 Vault 存儲用戶名和密碼，該用戶名和密碼存儲在路徑 `internal/database/config` 中。要創建此密鑰，需要啟用鍵值密鑰引擎並將用戶名和密碼放在指定的路徑中。

在 `vault-0` pod 上啟動交互式 shell 會話。

```bash
$ kubectl exec -it vault-0 -- /bin/sh
```

您的系統提示符將替換為新的提示符 `/$`。在此提示下發出的命令在 vault-0 容器上執行。

在 `internal` 路徑啟用 `kv-v2` 機密存儲。

```bash
/ $ vault secrets enable -path=internal kv-v2

Success! Enabled the kv-v2 secrets engine at: internal/
```

使用用戶名和密碼在路徑 `internal/database/config` 創建一個秘密。

```bash
/ $ vault kv put internal/database/config username="db-readonly-username" password="db-secret-password"

======== Secret Path ========
internal/data/database/config

======= Metadata =======
Key                Value
---                -----
created_time       2022-07-09T23:49:33.330145404Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1
```

驗證密鑰是否在路徑 `internal/database/config` 中定義。

```bash
/ $ vault kv get internal/database/config

======== Secret Path ========
internal/data/database/config

======= Metadata =======
Key                Value
---                -----
created_time       2022-07-09T23:49:33.330145404Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

====== Data ======
Key         Value
---         -----
password    db-secret-password
username    db-readonly-username
```

秘密已準備好用於應用程序。

最後，退出 `vault-0` pod。

```bash
/ $ exit
```

## 配置 Kubernetes auth

Vault 提供了一種 Kubernetes authentication method，使客戶端能夠使用 Kubernetes **服務帳戶令牌**進行身份驗證。此令牌在創建時提供給每個 pod。

在 `vault-0` pod 上啟動交互式 shell 會話。

```bash
$ kubectl exec -it vault-0 -- /bin/sh

/ $
```

在 Vault 中啟動 Kubernetes authentication method。

```bash
/ $ vault auth enable kubernetes

Success! Enabled kubernetes auth method at: kubernetes/
```

Vault 接受來自 Kubernetes 集群中任何客戶端的服務令牌。在身份驗證期間，Vault 通過查詢令牌審查 Kubernetes 端點來驗證服務帳戶令牌是否有效。

配置 Kubernetes 身份驗證方法以使用 Kubernetes API 的位置。

```bash
/ $ vault write auth/kubernetes/config \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"

Success! Data written to: auth/kubernetes/config
```

定義了環境變量 `KUBERNETES_PORT_443_TCP_ADDR` 並引用了 Kubernetes 主機的內部網絡地址。

對於客戶端讀取定義在 `internal/database/config` 中的秘密數據，需要為路徑 `internal/data/database/config` 授予讀取能力。這是 Vault Policy 的一個例子。Policy 定義了一組能力。

寫出名為 `internal-app` 的 Policy，該 Policy 啟用對路徑 `internal/data/database/config` 中機密的讀取功能。

```bash
/ $ vault policy write internal-app - <<EOF
path "internal/data/database/config" {
  capabilities = ["read"]
}
EOF
```

創建一個名為 `internal-app` 的 Kubernetes 身份驗證角色。

```bash
/ $ vault write auth/kubernetes/role/internal-app \
    bound_service_account_names=internal-app \
    bound_service_account_namespaces=default \
    policies=internal-app \
    ttl=24h

```

該命令的成功輸出類似於以下示例：

```bash
Success! Data written to: auth/kubernetes/role/internal-app
```

該角色將 Kubernetes 服務帳戶、internal-app 和命名空間（默認）與 Vault 策略 internal-app 連接起來。認證後返回的令牌有效期為 24 小時。

最後，退出 `vault-0` pod。

```bash
/ $ exit
```

## 定義一個 Kubernetes service account

Vault Kubernetes 身份驗證角色定義了一個名為 `internal-app` 的 Kubernetes 服務帳戶。

獲取默認命名空間中的所有服務帳戶。

```bash
$ kubectl get serviceaccounts

NAME                   SECRETS   AGE
default                1         98m
vault                  1         39m
vault-agent-injector   1         39m
```

讓我們在默認命名空間中創建一個名為 `internal-app` 的 Kubernetes 服務帳戶。

```bash
$ kubectl create sa internal-app

serviceaccount/internal-app created
```

驗證服務帳戶是否已創建。

```bash
$ kubectl get serviceaccounts

NAME                   SECRETS   AGE
default                1         101m
internal-app           1         69s
vault                  1         42m
vault-agent-injector   1         42m
```

此處的服務帳戶名稱與創建 `internal-app` 角色時分配給 `bound_service_account_names` 字段的名稱一致。

## 啟動一個應用程序

我們創建了一個示例應用程序，將其發佈到 DockerHub，並創建了一個啟動該應用程序的 Kubernetes 部署。

顯示 `orgchart` 應用程序的部署。

```bash
$ cat deployment-orgchart.yaml
```

jwissig源碼: https://github.com/jweissig/app/blob/master/main.go

```yaml title="deployment-orgchart.yaml" hl_lines="18"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orgchart
  labels:
    app: orgchart
spec:
  selector:
    matchLabels:
      app: orgchart
  replicas: 1
  template:
    metadata:
      annotations:
      labels:
        app: orgchart
    spec:
      serviceAccountName: internal-app
      containers:
        - name: orgchart
          image: jweissig/app:0.0.1
```

此部署的名稱是 orgchart。 spec.template.spec.serviceAccountName 定義了服務帳戶 internal-app 來運行這個容器。

應用在 deployment-orgchart.yaml 中定義的部署到 Kubernetes 中。

```
$ kubectl apply --filename deployment-orgchart.yaml

deployment.apps/orgchart created
```

獲取默認命名空間中的所有 pod。

```bash
$ kubectl get pods

NAME                                    READY   STATUS    RESTARTS   AGE
orgchart-97b5b656d-4658m                1/1     Running   0          57s
vault-0                                 1/1     Running   0          53m
vault-agent-injector-659b4488df-thv9x   1/1     Running   0          53m
```

orgchart pod 在此處顯示為以 orgchart 為前綴的 pod。

Vault-Agent 注入器查找定義特定註釋的部署。當前部署中不存在這些註釋。這意味著 orgchart pod 中的 orgchart 容器上不存在任何需要從 Vault 中取得的機密。

驗證 Vault-Agent 沒有將機密寫入 orgchart pod 中的 orgchart 容器。

```
$ kubectl exec \
    $(kubectl get pod -l app=orgchart -o jsonpath="{.items[0].metadata.name}") \
    --container orgchart -- ls /vault/secrets
```

輸出顯示沒找到名為 `/vault/secrets` 的此類文件或目錄：

```bash
ls: /vault/secrets: No such file or directory
command terminated with exit code 1
```

## 在 pod 裡頭注入 secret

部署使用默認命名空間中的 `internal-app` Kubernetes 服務帳戶運行 pod。 Vault Agent Injector 只有在部署包含一組特定註釋時才會修改它。現有部署可能會對其定義進行修補以包含必要的註釋。

顯示部署補丁 `patch-inject-secrets.yaml`。

```yaml title="patch-inject-secrets.yaml" hl_lines="5-7"
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: 'true'
        vault.hashicorp.com/role: 'internal-app'
        vault.hashicorp.com/agent-inject-secret-database-config.txt: 'internal/data/database/config'
```

這些註釋定義了部署模式的部分結構，並以 `vault.hashicorp.com` 為前綴。

- `agent-inject` 啟用 Vault Agent Injector service
- `role` 是 Vault Kubernetes 身份驗證角色
- `agent-inject-secret-FILEPATH` 文件路徑的前綴，`database-config.txt` 寫入 `/vault/secrets` 目錄。該值是 Vault 中定義的機密的路徑。

修補在 patch-inject-secrets.yaml 中定義的 orgchart 部署。

```bash
$ kubectl patch deployment orgchart --patch "$(cat patch-inject-secrets.yaml)"

deployment.apps/orgchart patched
```

一個新的 orgchart pod 在現有 pod 旁邊開始。當它準備好時，原始節點終止並將自己從活動 pod 列表中刪除。

獲取默認命名空間中的所有 pod。

```bash
$ kubectl get pods

NAME                                    READY   STATUS    RESTARTS   AGE
orgchart-6f88c9f9f4-c2pdb               2/2     Running   0          42s
vault-0                                 1/1     Running   0          66m
vault-agent-injector-659b4488df-thv9x   1/1     Running   0          66m
```

等到重新部署的 orgchart pod 報告它正在運行並準備就緒 (2/2)。

這個新的 pod 現在啟動了兩個容器。名為 `orgchart` 的應用程序容器和名為 `vault-agent` 的 Vault Agent 容器。

在新的 orgchart pod 中顯示 Vault-agent 容器的日誌。

```bash
$ kubectl logs \
    $(kubectl get pod -l app=orgchart -o jsonpath="{.items[0].metadata.name}") \
    --container vault-agent


==> Vault agent started! Log data will stream in below:

==> Vault agent configuration:

                     Cgo: disabled
               Log Level: info
                 Version: Vault v1.10.3
             Version Sha: af866591ee60485f05d6e32dd63dde93df686dfb

2022-07-10T00:31:28.780Z [INFO]  sink.file: creating file sink
2022-07-10T00:31:28.780Z [INFO]  sink.file: file sink configured: path=/home/vault/.vault-token mode=-rw-r-----
2022-07-10T00:31:28.781Z [INFO]  template.server: starting template server
2022-07-10T00:31:28.781Z [INFO]  auth.handler: starting auth handler
2022-07-10T00:31:28.781Z [INFO]  auth.handler: authenticating
2022-07-10T00:31:28.781Z [INFO] (runner) creating new runner (dry: false, once: false)
2022-07-10T00:31:28.781Z [INFO]  sink.server: starting sink server
2022-07-10T00:31:28.784Z [INFO] (runner) creating watcher
2022-07-10T00:31:28.801Z [INFO]  auth.handler: authentication successful, sending token to sinks
2022-07-10T00:31:28.801Z [INFO]  auth.handler: starting renewal process
2022-07-10T00:31:28.801Z [INFO]  sink.file: token written: path=/home/vault/.vault-token
2022-07-10T00:31:28.801Z [INFO]  template.server: template server received new token
2022-07-10T00:31:28.801Z [INFO] (runner) stopping
2022-07-10T00:31:28.801Z [INFO] (runner) creating new runner (dry: false, once: false)
2022-07-10T00:31:28.801Z [INFO] (runner) creating watcher
2022-07-10T00:31:28.801Z [INFO] (runner) starting
2022-07-10T00:31:28.803Z [INFO]  auth.handler: renewed auth token
```

Vault Agent 管理令牌生命週期和秘密檢索。機密在路徑 `/vault/secrets/database-config.txt` 的 orgchart 容器中呈現。

最後，顯示寫入 orgchart 容器的秘密。

```bash
$ kubectl exec \
    $(kubectl get pod -l app=orgchart -o jsonpath="{.items[0].metadata.name}") \
    --container orgchart -- cat /vault/secrets/database-config.txt

```

未格式化的秘密數據存在於容器中：

```
data: map[password:db-secret-password username:db-readonly-user]
metadata: map[created_time:2019-12-20T18:17:50.930264759Z deletion_time: destroyed:false version:2]
```

## 將模板應用於注入的機密

注入的秘密的結構可能需要以應用程序使用的方式進行結構化。在將機密寫入文件系統之前，模板可以構造數據。要應用此模板，需要應用一組新的註釋。

顯示包含模板定義的註釋文件。

```yaml title="patch-inject-secrets-as-template.yaml"
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: 'true'
        vault.hashicorp.com/agent-inject-status: 'update'
        vault.hashicorp.com/role: 'internal-app'
        vault.hashicorp.com/agent-inject-secret-database-config.txt: 'internal/data/database/config'
        vault.hashicorp.com/agent-inject-template-database-config.txt: |
          {{- with secret "internal/data/database/config" -}}
          postgresql://{{ .Data.data.username }}:{{ .Data.data.password }}@postgres:5432/wizard
          {{- end -}}
```

此補丁包含兩個新註釋：

- `agent-inject-status` 設置為更新通知注入器重新註入這些值。
- `agent-inject-template-FILEPATH` 文件路徑的前綴。該值定義要應用於機密數據的 Vault Agent 模板。

該模板將用戶名和密碼格式化為 PostgreSQL 連接字符串。

應用更新的註釋。

```bash
$ kubectl patch deployment orgchart --patch "$(cat patch-inject-secrets-as-template.yaml)"
```

獲取默認命名空間中的所有 pod。

```bash
kubectl get pods

NAME                                    READY   STATUS    RESTARTS   AGE
orgchart-554db4579d-w6565               2/2     Running   0          16s
vault-0                                 1/1     Running   0          126m
vault-agent-injector-5945fb98b5-tpglz   1/1     Running   0          126m
```

等到重新部署的 orgchart pod 報告它正在運行並準備就緒 (2/2)。

最後，在 orgchart pod 中顯示寫入 orgchart 容器的 secret。

```bash
$ kubectl exec \
    $(kubectl get pod -l app=orgchart -o jsonpath="{.items[0].metadata.name}") \
    -c orgchart -- cat /vault/secrets/database-config.txt

```

秘密呈現在容器上的 PostgreSQL 連接字符串中：

```
postgresql://db-readonly-user:db-secret-password@postgres:5432/wizard
```

## 帶有註釋的 Pod

註釋可以將這些秘密修補到任何部署中。 Pod 要求將註釋包含在其初始定義中。

顯示 `payroll` 應用程序的 pod 定義。

```yaml title="pod-payroll.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: payroll
  labels:
    app: payroll
  annotations:
    vault.hashicorp.com/agent-inject: 'true'
    vault.hashicorp.com/role: 'internal-app'
    vault.hashicorp.com/agent-inject-secret-database-config.txt: 'internal/data/database/config'
    vault.hashicorp.com/agent-inject-template-database-config.txt: |
      {{- with secret "internal/data/database/config" -}}
      postgresql://{{ .Data.data.username }}:{{ .Data.data.password }}@postgres:5432/wizard
      {{- end -}}
spec:
  serviceAccountName: internal-app
  containers:
    - name: payroll
      image: jweissig/app:0.0.1
```

應用 `pod-payroll.yaml` 中定義的 pod。

```bash
$ kubectl apply --filename pod-payroll.yaml
```

獲取默認命名空間中的所有 pod。

```bash
$ kubectl get pods

NAME                                    READY   STATUS    RESTARTS   AGE
orgchart-b6c98776d-r7m5k                2/2     Running   0          4h18m
payroll                                 2/2     Running   0          3m
vault-0                                 1/1     Running   0          5h34m
vault-agent-injector-659b4488df-thv9x   1/1     Running   0          5h34m
```

等到 payroll pod 報告它正在運行並準備就緒 (2/2)。

最後，在 payroll pod 中顯示寫入 payroll 容器的 secret。

```bash
$ kubectl exec \
    payroll \
    --container payroll -- cat /vault/secrets/database-config.txt
```

秘密呈現在容器上的 PostgreSQL 連接字符串中：

```
postgresql://db-readonly-username:db-secret-password@postgres:5432/wizard
```

## Secret 綁定到 Service account

使用 Vault Kubernetes 身份驗證角色中定義的帳戶以外的 Kubernetes 服務帳戶運行的 Pod 無法訪問在該路徑中定義的機密。

顯示 `website` 應用程序的部署和服務帳戶。

```yaml title="deployment-website.yaml" hl_lines="26 34"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: website
  labels:
    app: website
spec:
  selector:
    matchLabels:
      app: website
  replicas: 1
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: 'true'
        vault.hashicorp.com/role: 'internal-app'
        vault.hashicorp.com/agent-inject-secret-database-config.txt: 'internal/data/database/config'
        vault.hashicorp.com/agent-inject-template-database-config.txt: |
          {{- with secret "internal/data/database/config" -}}
          postgresql://{{ .Data.data.username }}:{{ .Data.data.password }}@postgres:5432/wizard
          {{- end -}}
      labels:
        app: website
    spec:
      # This service account does not have permission to request the secrets.
      serviceAccountName: website
      containers:
        - name: website
          image: jweissig/app:0.0.1
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: website
```

應用 `deployment-website.yaml` 中定義的部署和服務帳戶。

```bash
$ kubectl apply --filename deployment-website.yaml
```

獲取默認命名空間中的所有 pod。

```bash hl_lines="8"
$ kubectl get pods

NAME                                    READY   STATUS     RESTARTS   AGE
orgchart-554db4579d-w6565               2/2     Running    0          29m
payroll                                 2/2     Running    0          12s
vault-0                                 1/1     Running    0          155m
vault-agent-injector-5945fb98b5-tpglz   1/1     Running    0          155m
website-7fc8b69645-527rf                0/2     Init:0/1   0          76s
```

`web-site` 部署創建了一個 pod，但它在Kubernetes裡的狀態總是無法像其它的 pod 轉換成 READY。

在網站 pod 中顯示 vault-agent-init 容器的日誌。

```bash
$ kubectl logs \
    $(kubectl get pod -l app=website -o jsonpath="{.items[0].metadata.name}") \
    --container vault-agent-init
```

初始化過程失敗，因為服務帳戶名稱未經授權：


```
...
[INFO]  auth.handler: authenticating
[ERROR] auth.handler: error authenticating: error="Error making API request.

URL: PUT http://vault.default.svc:8200/v1/auth/kubernetes/login
Code: 500. Errors:

* service account name not authorized" backoff=1.562132589
```

服務帳戶 `website` 未分配給任何 Vault Kubernetes 身份驗證角色。此身份驗證失敗會導致部署初始化失敗。

顯示部署補丁 `patch-website.yaml`。

```yaml title="patch-website.yaml"
spec:
  template:
    spec:
      serviceAccountName: internal-app
```

該補丁修改了部署定義以使用服務帳戶 `internal-app`。此 Kubernetes 服務帳戶由 Vault Kubernetes 身份驗證角色授權。

修補 `patch-website.yaml` 中定義的網站部署。

```bash
$ kubectl patch deployment website --patch "$(cat patch-website.yaml)"
```

獲取默認命名空間中的所有 pod。

```bash
$ kubectl get pods

NAME                                    READY   STATUS        RESTARTS   AGE
orgchart-b6c98776d-r7m5k                2/2     Running       0          8h
payroll                                 2/2     Running       0          4h9m
vault-0                                 1/1     Running       0          9h
vault-agent-injector-659b4488df-thv9x   1/1     Running       0          9h
website-54cf7dcdfd-fv9vh                2/2     Running       0          23s
website-5899c7c66b-drhv7                0/2     Terminating   0          34m
```

等到網站 pod 報告它正在運行並準備就緒 (2/2)。

最後，在 `website` pod 中顯示寫入網站容器的 secret。

```bash
$ kubectl exec \
    $(kubectl get pod -l app=website -o jsonpath="{.items[0].metadata.name}") \
    --container website -- cat /vault/secrets/database-config.txt
```

Secret 呈現在容器上的 PostgreSQL 連接字符串中：

```
postgresql://db-readonly-user:db-secret-password@postgres:5432/wizard
```

!!! info
    Vault Kubernetes 角色：或者，您可以定義一個新的 Vault Kubernetes 角色，該角色啟用原始服務帳戶訪問，並修補部署。

## Secret 綁定到 Namespace

使用 Vault Kubernetes 身份驗證角色中定義的帳戶以外的 Kubernetes 服務帳戶運行的 Pod 無法訪問在該路徑中定義的機密。

創建 `offsite` 命名空間。

```bash
$ kubectl create namespace offsite

namespace/offsite created
```

將當前上下文設置為 `offsite` 命名空間。

```bash
$ kubectl config set-context --current --namespace offsite

Context "minikube" modified.
```

在 `offsite` 命名空間中創建一個名為 `internal-app` 的 Kubernetes 服務帳戶。

```bash
$ kubectl create sa internal-app

serviceaccount/internal-app created
```

顯示 `issue` 應用程序的部署 manifest。

```yaml title="deployment-issues.yaml"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: issues
  labels:
    app: issues
spec:
  selector:
    matchLabels:
      app: issues
  replicas: 1
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: 'true'
        vault.hashicorp.com/role: 'internal-app'
        vault.hashicorp.com/agent-inject-secret-database-config.txt: 'internal/data/database/config'
        vault.hashicorp.com/agent-inject-template-database-config.txt: |
          {{- with secret "internal/data/database/config" -}}
          postgresql://{{ .Data.data.username }}:{{ .Data.data.password }}@postgres:5432/wizard
          {{- end -}}
      labels:
        app: issues
    spec:
      serviceAccountName: internal-app
      containers:
        - name: issues
          image: jweissig/app:0.0.1
```

應用 `deployment-issues.yaml` 中定義的部署。

```bash
$ kubectl apply --filename deployment-issues.yaml

deployment.apps/issues created
```

獲取 `offsite` 命名空間中的所有 pod。

```bash
$ kubectl get pods

NAME                     READY   STATUS     RESTARTS   AGE
issues-cd8958bc6-7hrc5   0/2     Init:0/1   0          111s
```

`deployment-issues.yaml` 部署創建了一個 pod，但它在 Kubernetes 裡的狀態總是無法像其它的 pod 轉換成 READY。

在網站 pod 中顯示 vault-agent-init 容器的日誌。

```bash
$ kubectl logs \
    $(kubectl get pod -l app=issues -o jsonpath="{.items[0].metadata.name}") \
    --container vault-agent-init

==> Vault agent started! Log data will stream in below:

==> Vault agent configuration:

                     Cgo: disabled
               Log Level: info
                 Version: Vault v1.10.3
             Version Sha: af866591ee60485f05d6e32dd63dde93df686dfb

2022-07-10T11:51:47.276Z [INFO]  sink.file: creating file sink
2022-07-10T11:51:47.276Z [INFO]  sink.file: file sink configured: path=/home/vault/.vault-token mode=-rw-r-----
2022-07-10T11:51:47.277Z [INFO]  template.server: starting template server
2022-07-10T11:51:47.277Z [INFO]  auth.handler: starting auth handler
2022-07-10T11:51:47.277Z [INFO] (runner) creating new runner (dry: false, once: false)
2022-07-10T11:51:47.278Z [INFO]  auth.handler: authenticating
2022-07-10T11:51:47.277Z [INFO]  sink.server: starting sink server
2022-07-10T11:51:47.278Z [INFO] (runner) creating watcher
2022-07-10T11:52:47.279Z [ERROR] auth.handler: error authenticating: error="context deadline exceeded" backoff=1s
2022-07-10T11:52:48.280Z [INFO]  auth.handler: authenticating
2022-07-10T11:53:48.281Z [ERROR] auth.handler: error authenticating: error="context deadline exceeded" backoff=1.93s
2022-07-10T11:53:50.218Z [INFO]  auth.handler: authenticating
2022-07-10T11:54:50.219Z [ERROR] auth.handler: error authenticating: error="context deadline exceeded" backoff=3.36s
2022-07-10T11:54:53.586Z [INFO]  auth.handler: authenticating

```

命名空間 `offsite` 未分配給任何 Vault Kubernetes 身份驗證角色。此身份驗證失敗會導致部署初始化失敗。

在默認命名空間中的 `vault-0` pod 上啟動交互式 shell 會話。

```bash
$ kubectl exec --namespace default -it vault-0 -- /bin/sh

/ $
```

創建一個名為 `offsite-app` 的 Kubernetes 身份驗證角色。

```bash
/ $ vault write auth/kubernetes/role/offsite-app \
    bound_service_account_names=internal-app \
    bound_service_account_namespaces=offsite \
    policies=internal-app \
    ttl=24h
```

該命令的成功輸出類似於以下示例：

```bash
Success! Data written to: auth/kubernetes/role/offsite-app
```

退出 `vault-0` pod。

```bash
$ exit
```

顯示部署補丁 `patch-issues.yaml`。

```yaml title="patch-issues.yaml"
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: 'true'
        vault.hashicorp.com/agent-inject-status: 'update'
        vault.hashicorp.com/role: 'offsite-app'
        vault.hashicorp.com/agent-inject-secret-database-config.txt: 'internal/data/database/config'
        vault.hashicorp.com/agent-inject-template-database-config.txt: |
          {{- with secret "internal/data/database/config" -}}
          postgresql://{{ .Data.data.username }}:{{ .Data.data.password }}@postgres:5432/wizard
          {{- end -}}
```

該補丁執行更新以將 `vault.hashicorp.com/role` 設置為 Vault Kubernetes 角色 `offsite-app`。

修補 `patch-issues.yaml` 中定義的 `issues` 部署。

```bash
$ kubectl patch deployment issues --patch "$(cat patch-issues.yaml)"

deployment.apps/issues patched
```

新問題 pod 與現有 pod 一起開始。當它準備好時，原始節點終止並將自己從活動 pod 列表中刪除。

獲取 `offsite` 命名空間中的所有 pod。

```bash
$ kubectl get pods

NAME                    READY   STATUS    RESTARTS   AGE
issues-d8596bb9-jkzzr   2/2     Running   0          66s
```

等到重新部署的問題 pod 報告它正在運行並準備就緒 (2/2)。

最後，在問題 pod 中顯示寫入問題容器的秘密。

```bash
$ kubectl exec \
    $(kubectl get pod -l app=issues -o jsonpath="{.items[0].metadata.name}") \
    --container issues -- cat /vault/secrets/database-config.txt
```

秘密呈現在容器上的 PostgreSQL 連接字符串中：

```
postgresql://db-readonly-user:db-secret-password@postgres:5432/wizard
```
    