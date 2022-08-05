# 了解你的應用

## 查看 pod 和工作節點

**目標:**

- 了解 Kubernetes Pod。
- 了解 Kubernetes 工作節點。
- 對已部署的應用程序故障排除。

### Kubernetes Pods

[模塊 2](deploy-app.md) 時，Kubernetes 添加了一個 Pod 在這些容器中來創建這些部署的應用實例。是 Kubernetes 部署的，表示一個或多個應用程序），以及容器的一些共享資源。資源包括：

在[模塊 2]((deploy-app.md)) 創建 Deployment 時, Kubernetes 添加了一個 Pod 來託管你的應用實例。 Pod 是 Kubernetes 抽象出來的，表示一組一個或多個應用程序容器（如 Docker），以及這些容器的一些共享資源。這些資源包括:

- 共享存儲 Volume
- 網絡，作為唯一的集群 IP 地址
- 有關每個容器如何運行的信息，例如容器鏡像版本或要使用的特定端口。

Pod 為特定於應用程序的“邏輯主機”建模，並且可以包含相對緊耦合的不同應用容器。例如，Pod 可能既包含帶有 Node.js 應用的容器，也包含另一個不同的容器，用於提供 Node.js 網絡服務器要發布的數據。 Pod 中的容器共享 IP 地址和端口，始終位於同一位置並且共同調度，並在同一工作節點上的共享上下文中運行。

Pod是 Kubernetes 平台上的原子單元。當我們在 Kubernetes 上創建 Deployment 時，該 Deployment 會在其中創建包含容器的 Pod （而不是直接創建容器）。每個 Pod 都與調度它的工作節點綁定，並保持在那裡直到終止（根據重啟策略）或刪除。如果工作節點發生故障，則會在集群中的其他可用工作節點上調度相同的 Pod。

**Pod 概覽**

![](./assets/module_03_pods.svg)

### 工作節點

一個 pod 總是運行在 **工作節點**。工作節點是 Kubernetes 中的參與計算的機器，可以是虛擬機或物理計算機，具體取決於集群。每個工作節點由主節點管理。工作節點可以有多個 pod ，Kubernetes 主節點會自動處理在集群中的工作節點上調度 pod 。主節點的自動調度考量了每個工作節點上的可用資源。

每個 Kubernetes 工作節點至少運行:

- Kubelet，負責 Kubernetes 主節點和工作節點之間通信的過程; 它管理 Pod 和機器上運行的容器。
- 容器運行 runtime（如 Docker）負責從倉庫中提取容器鏡像，解壓縮容器以及運行應用程序。

**工作節點概覽**

![](./assets/module_03_nodes.svg)

### 使用 kubectl 進行故障排除

在模塊 2,你使用了 Kubectl 命令行界面。你將繼續在第3單元中使用它來獲取有關已部署的應用程序及其環境的信息。最常見的操作可以使用以下 kubectl 命令完成：

- **kubectl get** - 列出資源
- **kubectl describe** - 顯示有關資源的詳細信息
- **kubectl logs** - 打印 pod 和其中容器的日誌
- **kubectl exec** - 在 pod 中的容器上執行命令

你可以使用這些命令查看應用程序的部署時間，當前狀態，運行位置以及配置。

現在我們了解了有關集群組件和命令行的更多信息，讓我們來探索一下我們的應用程序。

#### 檢查應用程序配置

讓我們驗證我們在前面場景中部署的應用程序是否正在運行。我們將使用 `kubectl get` 命令並查找現有的 Pod：

```bash
kubectl get pods
```

輸出結果類似於這樣：

```
NAME                                   READY   STATUS    RESTARTS   AGE
kubernetes-bootcamp-57978f5f5d-jsdct   1/1     Running   0          42m
```

接下來，要查看該 Pod 內的容器以及用於構建這些容器的鏡像，我們運行 `describe pods` 命令：

```bash
kubectl describe pods
```

輸出結果類似於這樣：

```
Name:         kubernetes-bootcamp-57978f5f5d-jsdct
Namespace:    default
Priority:     0
Node:         k3d-k3s-default-server-0/172.22.0.2
Start Time:   Tue, 02 Aug 2022 15:29:45 +0800
Labels:       app=kubernetes-bootcamp
              pod-template-hash=57978f5f5d
Annotations:  <none>
Status:       Running
IP:           10.42.0.9
IPs:
  IP:           10.42.0.9
Controlled By:  ReplicaSet/kubernetes-bootcamp-57978f5f5d
Containers:
  kubernetes-bootcamp:
    Container ID:   containerd://83aa4f85a8bbbc41a77169f003d3caaf7c7481fd2f479dc8d033ee491b786cbf
    Image:          gcr.io/google-samples/kubernetes-bootcamp:v1
    Image ID:       gcr.io/google-samples/kubernetes-bootcamp@sha256:0d6b8ee63bb57c5f5b6156f446b3bc3b3c143d233037f3a2f00e279c8fcc64af
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Tue, 02 Aug 2022 15:30:21 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-zmfmz (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  kube-api-access-zmfmz:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  44m   default-scheduler  Successfully assigned default/kubernetes-bootcamp-57978f5f5d-jsdct to k3d-k3s-default-server-0
  Normal  Pulling    44m   kubelet            Pulling image "gcr.io/google-samples/kubernetes-bootcamp:v1"
  Normal  Pulled     43m   kubelet            Successfully pulled image "gcr.io/google-samples/kubernetes-bootcamp:v1" in 34.864007947s
  Normal  Created    43m   kubelet            Created container kubernetes-bootcamp
  Normal  Started    43m   kubelet            Started container kubernetes-bootcamp
```

我們在這裡看到有關 Pod 容器的詳細信息：IP 地址、使用的端口以及與 Pod 生命週期相關的事件列表。

describe 命令的輸出內容很豐富，涵蓋了一些我們尚未解釋的概念，但不用擔心，在本訓練營結束時它們會變得熟悉。

#### 查看容器日誌

應用程序發送到 STDOUT 的任何內容都會成為 Pod 中容器的 **日誌**。我們可以使用 `kubectl logs` 命令檢索這些日誌：

```bash
kubectl logs pod/kubernetes-bootcamp-57978f5f5d-jsdct
```

!!! tip
    每個 Deployment 所建構的 Pod 的名稱的後綴是由 Kubernetes 自動給號組合而成。

輸出結果類似於這樣：

```
Kubernetes Bootcamp App Started At: 2022-08-02T07:30:21.472Z | Running On:  kubernetes-bootcamp-57978f5f5d-jsdct 

Running On: kubernetes-bootcamp-57978f5f5d-jsdct | Total Requests: 1 | App Uptime: 446.976 seconds | Log Time: 2022-08-02T07:37:48.448Z
Running On: kubernetes-bootcamp-57978f5f5d-jsdct | Total Requests: 2 | App Uptime: 447.093 seconds | Log Time: 2022-08-02T07:37:48.565Z
Running On: kubernetes-bootcamp-57978f5f5d-jsdct | Total Requests: 3 | App Uptime: 569.043 seconds | Log Time: 2022-08-02T07:39:50.515Z
Running On: kubernetes-bootcamp-57978f5f5d-jsdct | Total Requests: 4 | App Uptime: 569.151 seconds | Log Time: 2022-08-02T07:39:50.623Z
```

#### 在容器上執行命令

一旦 Pod 啟動並運行，我們就可以直接在容器上執行命令。為此，我們使用 exec 命令並使用 Pod 的名稱作為參數。讓我們列出環境變量：

```bash
kubectl exec pod/kubernetes-bootcamp-57978f5f5d-jsdct -- env
```

輸出結果類似於這樣：

```
ATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=kubernetes-bootcamp-57978f5f5d-jsdct
NPM_CONFIG_LOGLEVEL=info
NODE_VERSION=6.3.1
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=10.43.0.1
KUBERNETES_SERVICE_HOST=10.43.0.1
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10.43.0.1:443
KUBERNETES_PORT_443_TCP=tcp://10.43.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
HOME=/root
```

同樣值得一提的是，容器本身的名稱可以省略，因為我們在 Pod 中只有一個容器。

接下來讓我們在 Pod 的容器中啟動一個 bash 會話：

```bash
kubectl exec -it kubernetes-bootcamp-57978f5f5d-jsdct -- bash
```

我們現在在運行 NodeJS 應用程序的容器上有一個開放的控制台。該應用程序的源代碼在 server.js 文件中：

```bash
root@kubernetes-bootcamp-57978f5f5d-jsdct:/# cat server.js
```

輸出結果類似於這樣：

```
root@kubernetes-bootcamp-57978f5f5d-jsdct:/# cat server.js
var http = require('http');
var requests=0;
var podname= process.env.HOSTNAME;
var startTime;
var host;
var handleRequest = function(request, response) {
  response.setHeader('Content-Type', 'text/plain');
  response.writeHead(200);
  response.write("Hello Kubernetes bootcamp! | Running on: ");
  response.write(host);
  response.end(" | v=1\n");
  console.log("Running On:" ,host, "| Total Requests:", ++requests,"| App Uptime:", (new Date() - startTime)/1000 , "seconds", "| Log Time:",new Date());
}
var www = http.createServer(handleRequest);
www.listen(8080,function () {
    startTime = new Date();;
    host = process.env.HOSTNAME;
    console.log ("Kubernetes Bootcamp App Started At:",startTime, "| Running On: " ,host, "\n" );
});
```

使用下列命令跳離Pod 的容器的 bash 會話:

```bash
exit
```


