# ArgoCD 入門

![](./assets/ST_argocd_2xpng.resized.png)

原文: [Getting Started](https://argo-cd.readthedocs.io/en/stable/getting_started/)

## 步驟 01 - 環境安裝

### 創建 K8S 集群

執行下列命令來創建實驗 Kubernetes 集群:

```bash title="執行下列命令  >_"
k3d cluster create --api-port 6443 \
--k3s-arg "--disable=traefik@server:0" \
--port 80:80@loadbalancer --port 443:443@loadbalancer \
--agents-memory=8G
```

```bash title="執行下列命令  >_"
k3d cluster create --api-port 6443 \
--port 8080:80@loadbalancer --port 8443:443@loadbalancer \
--agents-memory=8G
```

參數說明:

- `--disable=traefik@server:0` 安裝 Istio 後禁用 Traefik 負載均衡器
-  `--agents-memory=8G` 安裝 Istio 的額外增加一些內存


### 安裝 ArgoCD

添加 ArgoCD 的 helm 存儲庫並更新本地緩存：

```bash
helm repo add argo https://argoproj.github.io/argo-helm

helm repo update
```

創建要配置的 vlaues 檔案:

```yaml title="argo-cd-values.yaml"
server:
  ingress:
    # -- Enable an ingress resource for the Argo CD server
    enabled: true
    # -- List of ingress hosts
    ## Argo Ingress.
    ## Hostnames must be provided if Ingress is enabled.
    ## Secrets must be manually created in the namespace
    hosts: 
      - argocd.example.it
```

使用 Helm 在 `argocd` 的命名空間中部署 `argo/argo-cd` chart：

```bash
helm upgrade --install --create-namespace --namespace argocd  \
     argocd argo/argo-cd \
     --values argo-cd-values.yaml
```

**接到 ArgoCD Web UI:**

ArgoCD Web UI 可通過以下命令通過端口轉發訪問：

```bash
kubectl port-forward service/argocd-server -n argocd 8080:443
```

使用下列的命令來取得 argocd 初始化的密碼:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

結果:

```
9zjgYEekFFpN47ZB
```

打開瀏覽器並轉到 http://localhost:8080 並填寫用戶名 `admin` 與前一個命令所取得的密碼。

![](./assets/argocd-login.png)

成功登入後可看到:

![](./assets/argocd-logined.png)

### 下載 Argo CD CLI

下載 ArgoCD command line tools

```bash
wget https://github.com/argoproj/argo-cd/releases/download/v2.5.5/argocd-linux-amd64

sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

rm argocd-linux-amd64
```

您現在應該能夠運行 `argocd` 命令。

### 訪問 Argo CD API 服務器

默認情況下，Argo CD API 服務器不會暴露在外部 IP 中。要訪問 API 服務器，請選擇以下技術之一來公開 Argo CD API 服務器：

#### Service Type Load Balancer

將 `argocd-server` 服務類型更改為 `LoadBalancer`：

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

#### Ingress

按照 [ingress 文檔](https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/) 了解如何使用 `ingress` 來配置 Argo CD。

#### Port Forwarding

Kubectl 端口轉發也可用於連接到 API 服務器而不暴露服務。

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

然後可以使用 `https://localhost:8080` 來訪問 ArgoCD 的 API 服務器。

### 使用 CLI 登錄

管理員帳戶的初始密碼是自動生成的，並以明文形式存儲在 Argo CD 安裝命名空間中名為 `argocd-initial-admin-secret` 的密碼字段中。您可以使用 `kubectl` 簡單地檢索此密碼：

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

使用上面的用戶名 admin 和密碼，登錄到 Argo CD 的 IP 或主機名：

```bash
$ argocd login localhost:8080

WARNING: server certificate had error: x509: certificate signed by unknown authority. Proceed insecurely (y/n)? y
Username: admin
Password: 
'admin:login' logged in successfully
Context 'localhost:8080' updated
```

使用以下命令更改密碼：

```bash
argocd account update-password
```

### 從 Git 存儲庫創建應用程序

`https://github.com/argoproj/argocd-example-apps.git` 提供了一個包含留言簿應用程序的示例存儲庫，用來演示 Argo CD 的工作原理。

=== 通過 CLI 創建 Application

    首先，我們需要將當前命名空間設置為運行以下命令的 argocd：

    ```bash
    kubectl config set-context --current --namespace=argocd
    ```

    使用以下命令創建示例留言簿應用程序：

    ```bash
    argocd app create guestbook --repo https://github.com/argoproj/argocd-example-apps.git \
      --path guestbook --dest-server https://kubernetes.default.svc --dest-namespace default
    ```

=== 通過 UI 創建 Application

    打開瀏覽器訪問 Argo CD 外部 UI，並通過在瀏覽器中訪問 IP/主機名並使用在第 4 步中設置的憑據登錄。

    登錄後點擊 **+New App** 按鈕，如下圖：

    ![](./assets/new-app.webp)

    為Application Name 設成 `guestbook`，Project 設成 `default`，並將同步策略保留為 `Manual`：

    ![](./assets/app-ui-information.webp)

    通過將存儲庫 url 設置為 github 存儲庫 url，將 `https://github.com/argoproj/argocd-example-apps.git` 存儲庫連接到 Argo CD，將修訂保留為 `HEAD`，並設置路徑為 `guestbook`：

    ![](./assets/connect-repo.webp)

    對於 Destination，將集群 URL 設置為 `https://kubernetes.default.svc`（或集群名稱為 `in-cluster`），將命名空間設置為 `default`：

    ![](./assets/destination.webp)

    填寫完以上信息後，點擊 UI 上方的 `Create` 創建 guestbook 應用：

    ![](./assets/create-app.webp)

### 同步（部署）應用程序

=== 通過 CLI 同步

    創建 guestbook 應用程序後，您現在可以查看其狀態：

    ```bash
    $ argocd app get guestbook
    Name:               argocd/guestbook
    Project:            default
    Server:             https://kubernetes.default.svc
    Namespace:          default
    URL:                https://localhost:8080/applications/guestbook
    Repo:               https://github.com/argoproj/argocd-example-apps.git
    Target:             HEAD
    Path:               guestbook
    SyncWindow:         Sync Allowed
    Sync Policy:        <none>
    Sync Status:        OutOfSync from HEAD (53e28ff)
    Health Status:      Missing

    GROUP  KIND        NAMESPACE  NAME          STATUS     HEALTH   HOOK  MESSAGE
          Service     default    guestbook-ui  OutOfSync  Missing        
    apps   Deployment  default    guestbook-ui  OutOfSync  Missing 
    ```

    由於應用尚未部署，且尚未創建 Kubernetes 資源，因此應用狀態最初處於 `OutOfSync` 狀態。要同步（部署）應用程序，請運行：

    ```bash
    argocd app sync guestbook
    ```

    結果:

    ```bash
    TIMESTAMP                  GROUP        KIND   NAMESPACE                  NAME    STATUS    HEALTH        HOOK  MESSAGE
    2022-12-29T06:11:43+08:00            Service     default          guestbook-ui  OutOfSync  Missing              
    2022-12-29T06:11:43+08:00   apps  Deployment     default          guestbook-ui  OutOfSync  Missing              
    2022-12-29T06:11:43+08:00            Service     default          guestbook-ui    Synced  Healthy              
    2022-12-29T06:11:43+08:00            Service     default          guestbook-ui    Synced   Healthy              service/guestbook-ui created
    2022-12-29T06:11:43+08:00   apps  Deployment     default          guestbook-ui  OutOfSync  Missing              deployment.apps/guestbook-ui created
    2022-12-29T06:11:43+08:00   apps  Deployment     default          guestbook-ui    Synced  Progressing              deployment.apps/guestbook-ui created

    Name:               argocd/guestbook
    Project:            default
    Server:             https://kubernetes.default.svc
    Namespace:          default
    URL:                https://localhost:8080/applications/guestbook
    Repo:               https://github.com/argoproj/argocd-example-apps.git
    Target:             HEAD
    Path:               guestbook
    SyncWindow:         Sync Allowed
    Sync Policy:        <none>
    Sync Status:        Synced to HEAD (53e28ff)
    Health Status:      Progressing

    Operation:          Sync
    Sync Revision:      53e28ff20cc530b9ada2173fbbd64d48338583ba
    Phase:              Succeeded
    Start:              2022-12-29 06:11:43 +0800 CST
    Finished:           2022-12-29 06:11:43 +0800 CST
    Duration:           0s
    Message:            successfully synced (all tasks run)

    GROUP  KIND        NAMESPACE  NAME          STATUS  HEALTH       HOOK  MESSAGE
          Service     default    guestbook-ui  Synced  Healthy            service/guestbook-ui created
    apps   Deployment  default    guestbook-ui  Synced  Progressing        deployment.apps/guestbook-ui created
    ```

=== 通過 UI 同步

    在 guestbook 的 Application 視窗下點擊 `SYNC` 按鈕:

    ![](./assets/ui-sync-01.png)

    在左側滑出的設定中點擊 `SYNCHRONIZE` 按鈕:

    ![](./assets/ui-sync-02.png)


上述的命令/動件會驅動 ArgoCD 從 git 存儲庫中檢索 manifests 並使用 `kubectl` 來應用這些 manifests 到目的地的 Kuberntes 集群。

你可經由 ArgoCD 的 Web UI 來觀察 guestbook 應用程序是否順利佈署或運行狀態:

![](./assets/app-sync-status.png)

點擊 guestbook 應用程序視窗，ArgoCD 會展現您現在可以查看其資源組件、日誌、事件和評估的運行狀況。

![](./assets/app-sync-status-details.png)
