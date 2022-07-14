# Kubernetes 與外部 Vault 整合

原文: https://learn.hashicorp.com/tutorials/vault/kubernetes-external-vault

Vault 可以從集群外部管理 Kubernetes 應用程序 pod 的機密。這可能是 HashiCorp 雲平台 (HCP) Vault 或您組織內的另一個 Vault 服務。

!!! info
    在 Kubernetes 中運行 Vault：在 [Vault Installation to Minikube via Helm](https://learn.hashicorp.com/tutorials/vault/kubernetes-minikube-raft) 和 [Injecting Secrets into Kubernetes Pods via Vault Helm Sidecar](kubernetes-sidecar.md) 教程中探討了在 K8S 集群中運行 Vault 服務。

在本教程中，您將在本地運行 Vault，使用 Minikube 啟動 Kubernetes 集群，部署一個應用程序，該應用程序可以通過 Kubernetes 服務直接從 Vault 檢索機密，並通過 Vault Agent Injector 進行機密注入。

## 先決條件

本教程需要安裝 Kubernetes 命令行界面 (CLI) 和 Helm CLI、Minikube 以及其他配置以將它們組合在一起。

Docker version.

```bash
$ docker --version

Docker version 20.10.17, build 100c701
```

Minikube version.

```bash
$ minikube version

minikube version: v1.25.2
commit: 362d5fdc0a3dbee389b3d3f1034e8023e72bd3a7
```

Helm version.

```bash
$ helm version

version.BuildInfo{Version:"v3.9.0", GitCommit:"7ceeda6c585217a19a1131663d8cd1f7d641b2a7", GitTreeState:"clean", GoVersion:"go1.17.5"}
```

這些是推薦的軟件版本，顯示的輸出可能會因您的環境和您使用的軟件版本而異。

接下來，通過從 GitHub 來 clone [hashcorp/vault-guides](https://github.com/hashicorp/vault-guides) 存儲庫，裡頭包含本教程用來驗證的範例 Web 應用程序和其他配置檔案。

此存儲庫包含所有 Vault 學習指南的支持內容。本教程的特定內容可以在子目錄中找到。

進入 `vault-guides/operations/provision-vault/kubernetes/minikube/external-vault` 目錄。

```bash
$ cd vault-guides/operations/provision-vault/kubernetes/minikube/external-vault
```

!!! important
    工作目錄：本教程假設其餘命令都在此目錄中執行。


## 啟動 Vault

只要 Vault 服務器是網址是可被 Kubernetes 連接到的，那麼運行在 Kubernetes 集群外部的 Vault 就可以被它的任何被佈署在 Kubernetes 裡的 pod 所連接。

**1. 啟動 Vault 服務**

打開一個新終端，啟動一個以 root 作為 root 令牌的 Vault 開發服務器，它在 0.0.0.0:8200 處偵聽請求。**

```bash
$ vault server -dev -dev-root-token-id root -dev-listen-address 0.0.0.0:8200
```

將 `-dev-listen-address` 設置為 `0.0.0.0:8200` 會覆蓋 Vault 開發服務器的默認地址 (`127.0.0.1:8200`)，並使 Vault 可以被 Kubernetes 集群及其 Pod 尋址，因為它綁定到共享網絡.

**2. 設定 VAULT_ADDR 環境變數**

為 Vault CLI 設定環境變量來暴露出 Vault 服務器網路地址。

```bash
$ export VAULT_ADDR=http://0.0.0.0:8200
```

您部署的 Web 應用程序需要 Vault 存儲用戶名和密碼，這些用戶名和密碼存儲在路徑 `secret/devwebapp/config` 中。要創建此密鑰，需要啟用鍵值密鑰引擎並將用戶名和密碼放在指定的路徑中。默認情況下，Vault 開發服務器以在以 `secret` 為前綴的路徑上啟用鍵值機密引擎啟動。

**3. 便用 root 令牌登入 Vault**

在啟動 Vault 時我們設定的 root 令牌是 `root`

```bash
$ vault login root

Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                root
token_accessor       OJU8So8T5wuCassRAJpNrfN4
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```

**4. 在 `secret/devwebapp/config` 設定 credential**

使用用戶名 `username` 和密碼 `password`在被創建在路徑 `secret/devwebapp/config` 上。

```bash
$ vault kv put secret/devwebapp/config username='giraffe' password='salsa'

======== Secret Path ========
secret/data/devwebapp/config

======= Metadata =======
Key                Value
---                -----
created_time       2022-07-11T22:35:03.103203218Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1
```

**5. 驗證 Secret**

驗證密鑰是否存儲在路徑 `secret/devwebapp/config` 中。

```bash
$ vault kv get -format=json secret/devwebapp/config | jq ".data.data"

{
  "password": "salsa",
  "username": "giraffe"
}
```

帶有密鑰的 Vault 服務器已準備好由 Kubernetes 集群和部署在其中的 pod 進行尋址。


## 啟動 Minikube

Minikube 是一個 CLI 工具，用於配置和管理單節點 Kubernetes 集群的生命週期。這些集群在虛擬機 (VM) 中本地運行。

啟動 Kubernetes 集群。

```bash
$ minikube start

😄  minikube v1.25.2 on Ubuntu 21.10
🎉  minikube 1.26.0 is available! Download it: https://github.com/kubernetes/minikube/releases/tag/v1.26.0
💡  To disable this notice, run: 'minikube config set WantUpdateNotification false'

✨  Automatically selected the docker driver. Other choices: virtualbox, ssh
👍  Starting control plane node minikube in cluster minikube
🚜  Pulling base image ...
🔥  Creating docker container (CPUs=2, Memory=3900MB) ...
🐳  Preparing Kubernetes v1.23.3 on Docker 20.10.12 ...
    ▪ kubelet.housekeeping-interval=5m
    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default

```

顯示 Kubernetes 集群的版本。

```bash
$ kubectl version --output=json

{
  "clientVersion": {
    "major": "1",
    "minor": "24",
    "gitVersion": "v1.24.1",
    "gitCommit": "3ddd0f45aa91e2f30c70734b175631bec5b5825a",
    "gitTreeState": "clean",
    "buildDate": "2022-05-24T12:26:19Z",
    "goVersion": "go1.18.2",
    "compiler": "gc",
    "platform": "linux/amd64"
  },
  "kustomizeVersion": "v4.5.4",
  "serverVersion": {
    "major": "1",
    "minor": "23",
    "gitVersion": "v1.23.3",
    "gitCommit": "816c97ab8cff8a1c72eccca1026f7820e93e0d25",
    "gitTreeState": "clean",
    "buildDate": "2022-01-25T21:19:12Z",
    "goVersion": "go1.17.6",
    "compiler": "gc",
    "platform": "linux/amd64"
  }
}
```

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

主機、kubelet、apiserver 報告它們正在運行。 kubectl 是一個命令行界面 (CLI)，用於針對 Kubernetes 集群運行命令，它也被配置為與這個最近啟動的集群進行通信。


```bash
$ minikube status

minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured  
```

主機、kubelet、apiserver 報告它們正在運行。 kubectl 是一個命令行界面 (CLI)，用於針對 Kubernetes 集群運行命令，它也被配置為與這個最近啟動的集群進行通信。

## 決定 Vault address

正如您配置 Vault 一樣，綁定到主機上所有網絡的服務可以通過 Minikube 集群中的 pod 向 Kubernetes 集群的`網關`地址發送請求來尋址。

**1. 啟動 minikube SSH 會話**

```bash
$ minikube ssh
## ... minikube ssh login
```

**2. 檢索 Minikube 集群內部網關**

```bash hl_lines="10"
cat /etc/hosts

127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
192.168.49.2	minikube
192.168.49.1	host.minikube.internal
192.168.49.2	control-plane.minikube.internal
```

**3. 驗證與 Vault 的網絡連接**

```bash
docker@minikube:~$ curl -s http://host.minikube.internal:8200/v1/sys/seal-status

{"type":"shamir","initialized":true,"sealed":false,"t":1,"n":1,"progress":0,"nonce":"","version":"1.11.0","build_date":"2022-06-17T15:48:44Z","migration":false,"cluster_name":"vault-cluster-494486db","cluster_id":"20c81ca5-5293-a454-2af1-aa1ba5825400","recovery_seal":false,"storage_type":"inmem"}
```

輸出顯示 Vault 已初始化且未密封。這確認了集群中的 pod 能夠訪問 Vault，因為每個 pod 都配置為使用網關地址。

**4. 退出 Minikube SSH 會話**

```bash
$ exit
```

**5. 設定 EXTERNAL_VAULT_ADDR 環境**

創建一個名為 EXTERNAL_VAULT_ADDR 的變量來捕獲 Minikube 網關地址。

```bash
$ EXTERNAL_VAULT_ADDR=192.168.49.1
```

**6. 驗證變量**

驗證變量是否包含您在 minikube shell 中執行時看到的 ip 地址。

```bash
$ echo $EXTERNAL_VAULT_ADDR
```

## 使用 hard-coded 的 Vault 地址佈署應用程序

集群中的 pod 找尋 Vault 服務位址的最直接方法是使用在應用程序代碼中定義的hard-coded 網絡地址或作為環境變量提供。我們已經創建並發布了一個允許覆蓋 Vault 地址的 Web 應用程序。

!!! example "exampleapp"

    === "config.ru"

        ``` ruby hl_lines="34"
        # This file is a configuration file for a Rackup application.

        # Load the service file found in the lib directory.
        require './lib/service'

        # This Sinatra web application was built in the Classic way. Which
        # is great for simple web application. Sinatra automatically provides
        # a class you can provide to Rackup's `run` method to start the
        # application.
        #
        # Read about the 'Modular vs. Classic Style' in the Sinatra README.
        # @see http://www.sinatrarb.com/intro.html
        run ExampleApp

        ```
    === "lib/service.rb"

        ``` ruby hl_lines="34"
        require "sinatra"
        require "faraday"
        require "json"
        require "logger"

        $stdout.sync = true

        class ExampleApp < Sinatra::Base

        set :port, ENV['SERVICE_PORT'] || "8080"
        set :vault_token, ENV["VAULT_TOKEN"] || "root"

        configure :development do
            logger = Logger.new(STDOUT)
            logger.level = Logger::DEBUG
            set :raise_errors, true
            set :logger, logger
            set :vault_url, ENV["VAULT_ADDR"] || "http://host.docker.internal:8200"
        end

        configure :production do
            logger = Logger.new(STDOUT)
            logger.level = Logger::INFO
            set :vault_url, ENV["VAULT_ADDR"] || "http://vault:8200"
        end

        # GET "/"
        get "/" do
            logger.info "Received Request."

            # Set up an undefined state and set the vault server and secrets path
            secrets = { "username" => "undefined", "password" => "undefined" }

            secrets_path = "secret/data/devwebapp/config"

            # Ask for the secret at the path
            vault_response = Faraday.get "#{settings.vault_url}/v1/#{secrets_path}" do |req|
            req.headers['Content-Type'] = 'application/json'
            req.headers['X-Vault-Token'] = settings.vault_token
            end

            if vault_response.status != 200
            raise Exception.new "The secret request failed: #{vault_response.body}"
            end

            # Parse the JSON
            content = JSON.parse(vault_response.body) rescue {}

            # Traverse the response to find the secrets in the response
            if content.key?('data') and content['data'].key?('data')
            secrets = content['data']['data']
            end

            # Return secret
            secrets.to_s
        end

        end
        ```
    === "Dockerfile"

        ``` dockerfile
        FROM ruby:2.6.2

        RUN apt-get update && \
            apt-get install -y net-tools

        # Install gems
        ENV APP_HOME /app
        ENV HOME /root
        RUN mkdir $APP_HOME
        WORKDIR $APP_HOME
        COPY Gemfile* $APP_HOME/
        RUN bundle install

        # Upload source
        COPY config.ru $APP_HOME
        RUN mkdir $APP_HOME/lib
        COPY lib/* $APP_HOME/lib

        # Start server
        ENV PORT 8080
        EXPOSE 8080
        CMD ["rackup", "--port", "8080", "--env", "production" ]
        ```

**1. 創建 `internal-app` 的 Kubernetes 服務帳戶**

```bash
$ kubectl create sa internal-app

serviceaccount/internal-app created
```

**2. 設置 `EXTERNAL_VAULT_ADDR` 環境變數**

使用將 `VAULT_ADDR` 設置為 `EXTERNAL_VAULT_ADDR` 的 Web 應用程序定義一個名為 devwebapp 的 pod。

```bash hl_lines="14-17"
$ cat > devwebapp.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: devwebapp
  labels:
    app: devwebapp
spec:
  serviceAccountName: internal-app
  containers:
    - name: app
      image: burtlo/devwebapp-ruby:k8s
      env:
      - name: VAULT_ADDR
        value: "http://$EXTERNAL_VAULT_ADDR:8200"
      - name: VAULT_TOKEN
        value: root
EOF
```

**3. 創建 devwebapp pod**

```bash
$ kubectl apply --filename devwebapp.yaml

pod/devwebapp created
```

**4. 獲取 `default` 命名空間中的所有 pod**

```bash
$ kubectl get pods

NAME        READY   STATUS    RESTARTS   AGE
devwebapp   1/1     Running   0          2m11s
```

等到 devwebapp pod 報告正在運行並準備就緒 (1/1)。

**5. 從 devwebapp pod 中請求在 `localhost:8080` 的內容**

```bash
$ kubectl exec devwebapp -- curl -s localhost:8080 ; echo
```

結果顯示 Secret 是在 Vault 路徑 `secret/data/devwebapp/config` 中定義的。

```
{"password"=>"salsa", "username"=>"giraffe"}
```

Web 應用程序使用根令牌向外部 Vault 服務器進行身份驗證，並返回在 Vault 路徑 `secret/data/devwebapp/config` 中定義的密鑰。如果 Vault 服務器的地址不變，這種硬編碼方法是一種有效的解決方案。

## 在 K8S 部署服務和端點來映射至外部 Vault

外部 Vault 可能沒有 K8S 集群的 Service 可以讓相關的 Pod 來依賴。當 Vault 的網路地址更改時，每個 Pod 也需要更改才能繼續運行。管理此網路地址的另一種方法是定義 Kubernetes 的 `service` 和 `endpoint`。

`Service` 物件是對 Kubernetes 外部服務的抽象。當在 pod 中運行的應用程序請求 `service` 時，該請求將被路由到共享 `service` 的 `endpoint`。

**1. 定義 `service` 與 `endpoint` 的物件**

定義一個名為 `external-vault` 的服務和一個配置為尋址 `EXTERNAL_VAULT_ADDR` 的相應端點。

```bash
$ cat > external-vault.yaml <<EOF
---
apiVersion: v1
kind: Service
metadata:
  name: external-vault
  namespace: default
spec:
  ports:
  - protocol: TCP
    port: 8200
---
apiVersion: v1
kind: Endpoints
metadata:
  name: external-vault
subsets:
  - addresses:
      - ip: $EXTERNAL_VAULT_ADDR
    ports:
      - port: 8200
EOF
```

**2. 創建 `external-vault` 服務**

```bash
$ kubectl apply --filename external-vault.yaml

service/external-vault created
endpoints/external-vault created
```

**3.驗證 `external-vault` 服務是否可從 devwebapp pod 中尋址**

```bash
$ kubectl exec devwebapp -- curl -s http://external-vault:8200/v1/sys/seal-status | jq
```

結果顯示 Vault 服務器的狀態。

```bash
{
  "type": "shamir",
  "initialized": true,
  "sealed": false,
  "t": 1,
  "n": 1,
  "progress": 0,
  "nonce": "",
  "version": "1.11.0",
  "build_date": "2022-06-17T15:48:44Z",
  "migration": false,
  "cluster_name": "vault-cluster-494486db",
  "cluster_id": "20c81ca5-5293-a454-2af1-aa1ba5825400",
  "recovery_seal": false,
  "storage_type": "inmem"
}
```

**4. 創建一個將 `VAULT_ADDR` 設置為 `external-vault` 服務的 pod**

```yaml title="pod-devwebapp-through-service.yaml" hl_lines="14"
apiVersion: v1
kind: Pod
metadata:
  name: devwebapp-through-service
  labels:
    app: devwebapp-through-service
spec:
  serviceAccountName: internal-app
  containers:
    - name: app
      image: burtlo/devwebapp-ruby:k8s
      env:
      - name: VAULT_ADDR
        value: "http://external-vault:8200"
      - name: VAULT_TOKEN
        value: root
```

`devwebapp-through-service` pod 通過Service name而不是硬編碼的網絡地址來尋址 Vault。

```bash
$ kubectl apply --filename=pod-devwebapp-through-service.yaml

pod/devwebapp-through-service created
```

**5. 獲取默認命名空間中的所有 pod**

```bash
$ kubectl get pods

NAME                        READY   STATUS    RESTARTS   AGE
devwebapp                   1/1     Running   0          153m
devwebapp-through-service   1/1     Running   0          87s
```

等到 `devwebapp-through-service` pod 正在運行並準備就緒 (1/1)。

**6. 從 devwebapp-through-service pod 中請求在 localhost:8080 提供的內容**

```bash
$ kubectl exec devwebapp-through-service -- curl -s localhost:8080 ; echo

{"password"=>"salsa", "username"=>"giraffe"}
```

Web 應用程序對其通過 external-Vault 服務找到的外部 Vault 服務器進行身份驗證和請求。

## 安裝配置為尋址外部 Vault 的 Vault Helm chart

Vault Helm chart 只能部署配置為以外部 Vault 為目標的 Vault Agent Injector 服務。注入器服務通過在將 Vault Agent 容器自動寫入 pod 時添加它們來支持應用程序的身份驗證和秘密檢索，它包含特定的註釋。

在本節中，您將安裝 Vault Helm chart 並啟動 Vault agent injector 服務、配置 Vault 的 Kubernetes 身份驗證、創建角色以訪問機密以及修補部署。

### 安裝 Vault Helm chart

Vault Helm chart 啟動 Vault Agent Injector 服務。

**1. 添加 HashiCorp Helm 存儲庫**

```bash
$ helm repo add hashicorp https://helm.releases.hashicorp.com

"hashicorp" already exists with the same configuration, skipping
```

**2. 更新所有存儲庫以確保 helm 知道最新版本**

```bash
$ helm repo update

Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "hashicorp" chart repository
...Successfully got an update from the "secrets-store-csi-driver" chart repository
Update Complete. ⎈Happy Helming!⎈
```

**3. 安裝在外部模式下運行的最新版本的 Vault 服務器**

```bash
$ helm install vault hashicorp/vault \
    --set "injector.externalVaultAddr=http://external-vault:8200"


NAME: vault
LAST DEPLOYED: Wed Jul 13 16:37:28 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thank you for installing HashiCorp Vault!

Now that you have deployed Vault, you should look over the docs on using
Vault with Kubernetes available here:

https://www.vaultproject.io/docs/


Your release is named vault. To learn more about the release, try:

  $ helm status vault
  $ helm get manifest vault

```

為 `injector.externalVaultAddr` 分配 Deploy 服務和端點中定義的 Kubernetes 服務的地址，以解決外部 Vault 步驟。

`Vault Agent Injector` pod 部署在默認命名空間中。

**4. 獲取默認命名空間中的所有 pod**

```bash
$ kubectl get pods

NAME                                    READY   STATUS    RESTARTS   AGE
devwebapp                               1/1     Running   0          31h
devwebapp-through-service               1/1     Running   0          29h
vault-agent-injector-8679c9f654-trwvq   1/1     Running   0          14m
```

等到 `vault-agent-injector` pod 報告它正在運行並準備就緒 (1/1)。

Helm chart 創建保管庫服務帳戶。服務帳戶密碼是配置 Vault 的 Kubernetes 身份驗證方法所必需的。

**5. 描述 Vault 服務帳戶**

```bash hl_lines="12-13"
$ kubectl describe serviceaccount vault

Name:                vault
Namespace:           default
Labels:              app.kubernetes.io/instance=vault
                     app.kubernetes.io/managed-by=Helm
                     app.kubernetes.io/name=vault
                     helm.sh/chart=vault-0.20.1
Annotations:         meta.helm.sh/release-name: vault
                     meta.helm.sh/release-namespace: default
Image pull secrets:  <none>
Mountable secrets:   <none>
Tokens:              <none>
Events:              <none>
```

**6. 僅限 Kubernetes 1.24+**

`Mountable secrets` 的名稱顯示在 Kubernetes 1.23 中。在 Kubernetes 1.24+ 中，令牌不會自動創建，您必須顯式創建它。

```bash
cat > vault-secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: vault-token-g955r
  annotations:
    kubernetes.io/service-account.name: vault
type: kubernetes.io/service-account-token
EOF
```

創建一個 `Secret`。

```bash
$ kubectl apply -f vault-secret.yaml

secret/vault-token-g955r created
```

```bash hl_lines="13"
$ kubectl describe sa vault

Name:                vault
Namespace:           default
Labels:              app.kubernetes.io/instance=vault
                     app.kubernetes.io/managed-by=Helm
                     app.kubernetes.io/name=vault
                     helm.sh/chart=vault-0.20.1
Annotations:         meta.helm.sh/release-name: vault
                     meta.helm.sh/release-namespace: default
Image pull secrets:  <none>
Mountable secrets:   <none>
Tokens:              vault-token-g955r
Events:              <none>
```

**7. 創建一個名為 `VAULT_HELM_SECRET_NAME` 的變量來存儲密鑰名稱**

```bash
$ VAULT_HELM_SECRET_NAME=$(kubectl get secrets --output=json | jq -r '.items[].metadata | select(.name|startswith("vault-token-")).name')
```

此命令過濾以 vault-token- 開頭的秘密並返回令牌的名稱。

**8. 描述 Vault 令牌秘密**

```bash
$ kubectl describe secret $VAULT_HELM_SECRET_NAME
```

示例輸出：

```bash
$ kubectl describe secret $VAULT_HELM_SECRET_NAME

Name:         vault-token-g955r
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: vault
              kubernetes.io/service-account.uid: ab246d63-abae-4244-a173-5eb926e78913

Type:  kubernetes.io/service-account-token

Data
====
namespace:  7 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6Im1qb3FKczdnM1dFZnpDekd1N2tEU1B5YmhXZTFOVFpScjRxVm9oTUppQU0ifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6InZhdWx0LXRva2VuLWc5NTVyIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6InZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiYWIyNDZkNjMtYWJhZS00MjQ0LWExNzMtNWViOTI2ZTc4OTEzIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmRlZmF1bHQ6dmF1bHQifQ.dVWwFxGLa-lxCqK2L0hllUI6KWAetnzAVMkjF3I7ybGxDq7JJcLyAil8Aym1rRK3gQBkwh1th0yHVrSfezBo0eEUgfPvzY4Qttqc1um3FN8dmPUdjnV4Gewr6hHEYijYtJJW7PWJYS7SmE3tnLuaaoyPFpXJBM-RyTW-aws9O5cZIVD18AI2G23MLhyt1D8nqZr39yyNPsPiwKPa-Ba0QYzMZUOh0AmcLEIP7yGnMM8OLKVeN7XMz7yKbCV1f85Nz0BgH-aPF1rYHr7gCZ5I0jRVPqqpmAl8Gje1OggWxekCnWhZZTmQtMXCR3L2e4q08P2LYTeiSxIiBFeyqjPZVg
ca.crt:     1111 bytes
```

## 配置 Kubernetes 身份驗證

Vault 提供了一種 Kubernetes 身份驗證方法，使客戶端能夠使用 Kubernetes 服務帳戶令牌進行身份驗證。

**1. 啟用 Kubernetes 身份驗證方法**

```bash
$ vault auth enable kubernetes

Success! Enabled kubernetes auth method at: kubernetes/
```

Vault 接受來自 Kubernetes 集群中任何客戶端的此服務令牌。在身份驗證期間，Vault 通過查詢配置的 Kubernetes 端點來驗證服務帳戶令牌是否有效。要正確配置它，需要捕獲服務帳戶的 JSON Web 令牌 (JWT)、Kubernetes CA 證書和 Kubernetes 主機 URL。

**2. 從密鑰中獲取 JSON Web 令牌 (JWT)**

```bash
$ TOKEN_REVIEW_JWT=$(kubectl get secret $VAULT_HELM_SECRET_NAME --output='go-template={{ .data.token }}' | base64 --decode)
```

**3. 檢索 Kubernetes CA 證書**

```bash
$ KUBE_CA_CERT=$(kubectl config view --raw --minify --flatten --output='jsonpath={.clusters[].cluster.certificate-authority-data}' | base64 --decode)
```

**4. 檢索 Kubernetes 主機 URL**

```bash
$ KUBE_HOST=$(kubectl config view --raw --minify --flatten --output='jsonpath={.clusters[].cluster.server}')
```

**5. 配置 Kubernetes 身份驗證方法以使用服務帳戶令牌、Kubernetes 主機的位置、其證書及其服務帳戶頒發者名稱**

```bash
$ vault write auth/kubernetes/config \
     token_reviewer_jwt="$TOKEN_REVIEW_JWT" \
     kubernetes_host="$KUBE_HOST" \
     kubernetes_ca_cert="$KUBE_CA_CERT"

Success! Data written to: auth/kubernetes/config
```

```bash
$ vault write auth/kubernetes/config \
     token_reviewer_jwt="$TOKEN_REVIEW_JWT" \
     kubernetes_host="$KUBE_HOST" \
     kubernetes_ca_cert="$KUBE_CA_CERT" \
     issuer="https://kubernetes.default.svc.cluster.local"
```

要讓 Vault 客戶端讀取 Start Vault 部分中定義的機密數據，需要為路徑 `secret/data/devwebapp/config` 授予讀取能力。

**6. 定義 `devwebapp` 的策略，該策略在路徑 `secret/data/devwebapp/config` 處啟用對密鑰的讀取功能**

```bash
$ vault policy write devwebapp - <<EOF
path "secret/data/devwebapp/config" {
  capabilities = ["read"]
}
EOF

Success! Uploaded policy: devwebapp
```

**7. 在 Vault 裡創建一個名為 `devweb-app` 的 Kubernetes 身份驗證 `role`**

```bash
$ vault write auth/kubernetes/role/devweb-app \
     bound_service_account_names=internal-app \
     bound_service_account_namespaces=default \
     policies=devwebapp \
     ttl=24h

Success! Data written to: auth/kubernetes/role/devweb-app
```

該角色將 Kubernetes 服務帳戶、內部應用程序和命名空間（默認）與 Vault 策略 devwebapp 連接起來。認證後返回的令牌有效期為 24 小時。

## 將 Secret 注入進 pod

Vault Agent Injector 僅在 Pod 或部署包含一組特定註釋時才對其進行修改。

**1.使用您喜歡的文本編輯器並使用註釋 `pod-devwebapp-with-annotations.yaml` 檢查 pod**

```yaml title="pod-devwebapp-with-annotations.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: devwebapp-with-annotations
  labels:
    app: devwebapp-with-annotations
  annotations:
    vault.hashicorp.com/agent-inject: 'true'
    vault.hashicorp.com/role: 'devweb-app'
    vault.hashicorp.com/agent-inject-secret-credentials.txt: 'secret/data/devwebapp/config'
spec:
  serviceAccountName: internal-app
  containers:
    - name: app
      image: burtlo/devwebapp-ruby:k8s
```

這些註釋定義了部署模式的部分結構，並以 `vault.hashicorp.com` 為前綴。

- `agent-inject` 啟用 Vault Agent Injector 服務
- `role`是 Vault Kubernetes 身份驗證`role`
- `agent-inject-secret-FILEPATH` 作為文件路徑的前綴，`credentials.txt` 寫入 `/vault/secrets` 目錄。該值是 Vault 中定義的機密的路徑。

**2. 創建 devwebapp-with-annotations pod**

```bash
$ kubectl apply --filename pod-devwebapp-with-annotations.yaml

pod/devwebapp-with-annotations created
```

**3. 獲取默認命名空間中的所有 pod**

```bash
$ kubectl get pods

NAME                                    READY   STATUS     RESTARTS   AGE
devwebapp                               1/1     Running    0          33h
devwebapp-through-service               1/1     Running    0          30h
devwebapp-with-annotations              0/2     Init:0/1   0          38s
vault-agent-injector-8679c9f654-trwvq   1/1     Running    0          104m

```

等到 `devwebapp-with-annotations` pod 報告它正在運行並準備就緒 (2/2)。

Vault Agent Injector 服務在 pod 中創建了一個附加容器，該容器自動將機密寫入文件路徑 `/vault/secrets/credentials.txt` 處的應用程序容器。

**4. 在 devwebapp-with-annotations pod 上顯示寫入文件 /vault/secrets/secret-credentials.txt 的機密**

```bash
$ kubectl exec -it devwebapp-with-annotations -c app -- cat /vault/secrets/credentials.txt
```

結果顯示容器上存在的未格式化的秘密數據。

```bash
data: map[password:salsa username:giraffe]
metadata: map[created_time:2022-06-06T18:26:14.070155Z custom_metadata:<nil> deletion_time: destroyed:false version:1]
```

!!! tip
    格式化數據：可以應用[模板](https://learn.hashicorp.com/tutorials/vault/kubernetes-sidecar#apply-a-template-to-the-injected-secrets)來格式化這些數據以滿足應用程序的需要。

這個 pod 中的應用程序仍然直接檢索秘密，但現在註入器服務已部署並且能夠檢索應用程序的秘密，未來的更新可以刪除該應用程序邏輯。