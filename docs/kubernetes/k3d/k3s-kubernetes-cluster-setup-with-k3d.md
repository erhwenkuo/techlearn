# 使用 K3D 設置 Kubernetes 集群

原文: https://kartaca.com/en/k3s-kubernetes-cluster-setup-with-k3d/

在本文中，將討論如何使用 Rancher 的工具 K3D 在基於 Linux 的操作系統中創建 Kubernetes 集群。

## K3S (輕量級 Kubernetes)

K3S 是 Rancher 於 2019 年開源發布的輕量級 Kubernetes 發行版。 K3S 被設計為 100MB 以下的二進製文件。此外，它是一個擁有 CNCF（Cloud Native Computing Foundation）證書的 Kubernetes 工具。由於 K3S 的輕量級，我們甚至可以在配置低的系統中安裝 Kubernetes 集群。 K3S的最低系統要求如下：

- Linux 3.10
- 512MB RAM (server)
- 75MB RAM (node)
- 200MB range

## K3D

K3D 是一個為我們提供快速搭建 Kubernetes 集群的工具。創建 K3D 的集群是輕量級的，因為它們基於我上面提到的 K3S。換句話說，使用 K3D，我們可以快速搭建一個 K3S 集群，用最少的工作量和最少的資源使用。此外，由於 K3D 是跨平台的，它可以在 Windows、Mac OS 和 Linux 設備上運行，並支持創建多個集群。

### 安裝

Requirements：

- Docker
- Kubectl
- K3D

#### Docker 設定安裝

要快速安裝 Docker，你可以使用 Rancher 準備的腳本和我在下面給出的命令，也可以按照適合你將要安裝的環境的安裝步驟從這個地址安裝。

```bash
[Rancher Script]
$ curl https://releases.rancher.com/install-docker/19.03.sh | sh
```

#### Kubectl 設定安裝

您需要設置 Kubectl 以與 Kubernetes API 通信，您可以按照以下步驟輕鬆完成此操作。

```bash
$ curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
$ chmod +x ./kubectl
$ sudo mv ./kubectl /usr/local/bin/kubectl
```

在安裝完之後，您可以使用以下命令進行 Kubectl 版本檢查。

```bash
$ kubectl version --client
```

#### K3D 設定安裝

K3D 設置非常簡單，您唯一需要做的就是在終端輸入以下命令。如果您願意，可以根據您的環境從此地址進行設置步驟。

```bash
$ wget -q -O - https://raw.githubusercontent.com/rancher/k3d/main/install.sh | bash
```

您可以通過版本檢查來驗證您的設置，為此您需要在終端中輸入以下指令。

```bash
$ k3d --version
```

### K3D 指令

K3D 提供了一系列的指令來幫助我們控制 Kubernetes 集群的相關設定與操作。

```bash
$ k3d 
Usage:
  k3d [flags]
  k3d [command]

Available Commands:
  cluster      Manage cluster(s)
  completion   Generate completion scripts for [bash, zsh, fish, powershell | psh]
  config       Work with config file(s)
  help         Help about any command
  image        Handle container images.
  kubeconfig   Manage kubeconfig(s)
  node         Manage node(s)
  registry     Manage registry/registries
  version      Show k3d and default k3s version

Flags:
  -h, --help         help for k3d
      --timestamps   Enable Log timestamps
      --trace        Enable super verbose output (trace logging)
      --verbose      Enable verbose output (debug logging)
      --version      Show k3d and default k3s version

Use "k3d [command] --help" for more information about a command.
```

使用 `k3d cluster` 命令:

- `create`      創建一個新集群
- `delete`      刪除集群。
- `edit`        [EXPERIMENTAL] 編輯集群。
- `list`        列出集群
- `start`       啟動現有的 k3d 集群
- `stop`        停止現有的 k3d 集群

## 設置 Kubernetes 集群

現在我們可以繼續進行集群設置。使用以下命令，您可以開始使用您選擇的名稱創建集群。

```bash
$ k3d cluster create [NAME]
```

由於它在第一次安裝時會下載大約 150–200MB 的 docker 映像，因此安裝時間可能需要一些時間，具體取決於您的互聯網速度，但除非您刪除這些映像，否則您的下一次集群安裝將花費更少的時間。


```bash
INFO[0000] Prep: Network                                
INFO[0000] Created network 'k3d-k3s-default'            
INFO[0000] Created image volume k3d-k3s-default-images  
INFO[0000] Starting new tools node...                   
INFO[0000] Starting Node 'k3d-k3s-default-tools'        
INFO[0001] Creating node 'k3d-k3s-default-server-0'     
INFO[0001] Creating LoadBalancer 'k3d-k3s-default-serverlb' 
INFO[0001] Using the k3d-tools node to gather environment information 
INFO[0001] HostIP: using network gateway 172.27.0.1 address 
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

使用 `k3d cluster list` 命令，您可以查看已創建的集群及其狀態。

```bash
$ k3d cluster list

NAME          SERVERS   AGENTS   LOADBALANCER
k3s-default   1/1       0/0      true
```

如果您按照這些步驟在遠程服務器上安裝 K3D 並在此環境中創建集群，則在安裝應用程序時無法訪問此應用程序。因為我們不允許任何端口，所以只能從集群內部訪問它。

在下一步中，我將討論如何訪問我們的應用程序。

```bash
# Deletes all clusters 
$ k3d cluster delete -a

# Deletes specified cluster
$ k3d cluster delete [name]
```

## 服務暴露

在使用 K3D 時，我們需要預先打開一個端口，但它為我們提供了很多方法。我們可以作為 node-port 或通過 Load Balancer 訪問我們的應用程序。

### K3D 負載均衡器設置

K3D 默認使用 traefik，我們將通過 traefik 暴露佈署在 Kubernetes 裡的應用程序。如果您在遠程服務器上安裝了 K3D，則需要打開外部端口進行訪問。為此，我們使用以下命令打開負載均衡器。

```bash	
$ k3d cluster create -p 8081:80@loadbalancer
```

!!! info
    - the port-mapping construct 8081:80@loadbalancer means: “map port 8081 from the host to port 80 on the container which matches the nodefilter loadbalancer“


安裝集群後，讓我們啟動一個示例 Nginx 應用程序。為此，您可以運行下面的 kubectl 命令。

```bash
$ kubectl create deployment nginx --image=nginx --port 80
```

使用此命令，我們創建了一個內部端口為 80 並命名為 demo 的 nginx 映像。現在，讓我們創建一個服務來訪問我們創建的部署。

```bash
$ kubectl create service clusterip nginx --tcp=80:80
```

檢查結果:

```bash hl_lines="5"
$ kubectl get svc

NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.43.0.1      <none>        443/TCP   88s
nginx        ClusterIP   10.43.72.227   <none>        80/TCP    15s
```

成功創建服務後，我們現在可以設置負載均衡器設置。現在讓我們創建一個名為 `demo-ingress.yaml` 的文件並編輯其內容，如下所示。

詳細的 `ingress` 設定規則見: [Kubernetes - Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

```yaml title="demo-ingress.yaml"
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
  annotations:
    ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80
```


在 Kubernetes 上部署 ingress:

```bash
$ kubectl create -f demo-ingress.yaml
```

這樣，我們就完成了剛剛創建的演示 nginx 應用的負載均衡器設置。我們可以看到我們使用 `kubectl get pod,svc,ing` 命令創建的 Kubernetes 對象。

```bash
$ kubectl get pod,svc,ing

NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-7848d4b86f-xdbfx   1/1     Running   0          111s

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.43.0.1      <none>        443/TCP   2m37s
service/nginx        ClusterIP   10.43.72.227   <none>        80/TCP    84s

NAME                              CLASS    HOSTS   ADDRESS      PORTS   AGE
ingress.networking.k8s.io/nginx   <none>   *       172.31.0.2   80      21s
```

如果您有上述輸出，我們現在可以對其進行測試。如果您在自己的計算機上使用 localhost 完成了安裝，則可以使用服務器 IP 地址訪問 nginx 頁面，如果您在遠程服務器上完成安裝，則可以驗證安裝。

```bash
$ curl localhost:8081/
```

### K3D NodePort 設置

K3D 需要顯示地告訴它打開某個端口範圍。讓我繼續下面的例子。

```
$ k3d cluster create --agents 1 -p 30000-30010:30000-30010@agent:0 -p 8081:80@loadbalancer
```

我們在上面的命令中添加了一個新參數。 Docker 用戶經常使用“-p”（發布）參數。有了這個參數，我們就可以打開主機容器端口了。我們在上面做了這個，在主機端和容器端都打開了 30000-30010 端口範圍。(Kubernetes default node port range: 30000–32767)

!!! info
    注意：我不建議配置系統打開 30000-32767 範圍，因為打開端口時 RAM 使用量會顯著增加。

安裝集群後，讓我們啟動一個示例 Nginx 應用程序。為此，您可以運行下面的 kubectl 命令。

```bash
$ kubectl create deployment nginx --image=nginx --port 80
```

使用此命令，我們創建了一個內部端口為 80 並命名為 demo 的 nginx 映像。現在，讓我們使用 `nodeport` 創建一個服務來訪問我們部署的 nginx 應用程序。

創建一個名為 `demo-nodeport.yaml` 的文件並添加以下行並保存它。

```yaml title="demo-nodeport.yaml" hl_lines="10 16"
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  ports:
  - name: 80-80
    nodePort: 30008
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  type: NodePort
```

在 Kubernetes 上部署 service:

```bash
$ kubectl create -f demo-nodeport.yaml
```

使用服務器 IP 地址與 nodeport 所映射的端口來訪問 nginx 頁面。

```bash
$ curl localhost:30008/
```
