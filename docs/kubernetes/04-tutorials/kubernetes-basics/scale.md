# 縮放你的應用

## 運行應用程序的多個實例

**目標:**

- 用 kubectl 擴縮應用程序

### 擴縮應用程序

在之前的模塊中，我們創建了一個 Deployment，然後通過 Service 讓其可以被訪問。 Deployment 僅為跑這個應用程序創建了一個 Pod。當流量增加時，我們需要擴容應用程序滿足用戶需求。

**水平擴充** 是通過改變 Deployment 中的 **--replicas** 參數來實現的。


![](./assets/module_05_scaling1.svg)

![](./assets/module_05_scaling2.svg)


擴展(scale out)，Deployment 將創建新的 Pods，並將資源調度請求分配到有可用資源的節點上，收縮(scale in) 會將 Pods 數量減少至所需的狀態。 Kubernetes 還支持 Pods 的自動縮放，但這並不在本教程的討論範圍內。將 Pods 數量收縮到 0 也是可以的，但這會終止 Deployment 上所有已經部署的 Pods。

運行應用程序的多個實例需要在它們之間分配流量。服務 (Service)有一種負載均衡器類型，可以將網絡流量均衡分配到外部可訪問的 Pods 上。服務將會一直通過端點來監視 Pods 的運行，保證流量只分配到可用的 Pods 上。

一旦有了多個應用實例，就可以沒有宕機地滾動更新。我們將會在下面的模塊中介紹這些。現在讓我們體驗一下應用程序的擴縮過程。

#### 擴展部署

要列出您的部署，請使用 `kubectl get deployments` 命令:

```bash
kubectl get deployments
```

輸出應類似於：

```
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   1/1     1            1           11m
```

我們應該有 1 個 Pod。由此可見：

- **NAME** 列出集群中部署的名稱。
- **READY** 顯示 CURRENT/DESIRED 副本的比率
- **UP-TO-DATE** 顯示已更新以達到所需狀態的副本數。
- **AVAILABLE** 顯示有多少應用程序副本可供用戶使用。
- **AGE** 顯示應用程序運行的時間量。

要查看 Deployment 創建的 ReplicaSet，請運行 `kubectl get rs`:

```bash
kubectl get rs
```

輸出應類似於：

```
NAME                             DESIRED   CURRENT   READY   AGE
kubernetes-bootcamp-57978f5f5d   1         1         1       11m
```

請注意，ReplicaSet 的名稱始終格式化為 [DEPLOYMENT-NAME]-[RANDOM-STRING]。隨機字符串是隨機生成的，並使用 pod-template-hash 作為種子。

此命令結果會展示出兩個重要資訊列：

- **DESIRED** 顯示所需的應用程序副本數，您在創建部署時定義。這是期望的狀態。
- **CURRENT** 顯示當前正在運行的副本數。

接下來，讓我們將 Deployment 擴展到 4 個副本。我們將使用 kubectl scale 命令，後跟部署類型、名稱和所需的實例數量：

```bash
kubectl scale deployments/kubernetes-bootcamp --replicas=4
```

要出部署:

```bash
kubectl get deployments
```

輸出應類似於：

```
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   4/4     4            4           10m
```

更改已應用，我們有 4 個可用的應用程序實例。接下來，讓我們檢查 Pod 的數量是否發生了變化：

```bash
kubectl get pods -o wide
```

輸出應類似於：

```
NAME                                   READY   STATUS        RESTARTS   AGE    IP           NODE                       NOMINATED NODE   READINESS GATES
kubernetes-bootcamp-57978f5f5d-844f2   1/1     Running       0          16s    10.42.0.15   k3d-k3s-default-server-0   <none>           <none>
kubernetes-bootcamp-57978f5f5d-csrqd   1/1     Running       0          16s    10.42.0.14   k3d-k3s-default-server-0   <none>           <none>
kubernetes-bootcamp-57978f5f5d-lz4l7   1/1     Running       0          16s    10.42.0.16   k3d-k3s-default-server-0   <none>           <none>
kubernetes-bootcamp-57978f5f5d-n2lb8   1/1     Running       0          16s    10.42.0.13   k3d-k3s-default-server-0   <none>           <none>
```

現在有 4 個 Pod，具有不同的 IP 地址。更改已在部署事件日誌中註冊。要檢查這一點，請使用 `describe` 命令：

```bash
kubectl describe deployments/kubernetes-bootcamp
```

輸出應類似於：

```hl_lines="31"
Name:                   kubernetes-bootcamp
Namespace:              default
CreationTimestamp:      Tue, 02 Aug 2022 18:04:17 +0800
Labels:                 app=kubernetes-bootcamp
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=kubernetes-bootcamp
Replicas:               4 desired | 4 updated | 4 total | 4 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=kubernetes-bootcamp
  Containers:
   kubernetes-bootcamp:
    Image:        gcr.io/google-samples/kubernetes-bootcamp:v1
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Progressing    True    NewReplicaSetAvailable
  Available      True    MinimumReplicasAvailable
OldReplicaSets:  <none>
NewReplicaSet:   kubernetes-bootcamp-57978f5f5d (4/4 replicas created)
Events:
  Type    Reason             Age                   From                   Message
  ----    ------             ----                  ----                   -------
  Normal  ScalingReplicaSet  2m1s (x2 over 4m47s)  deployment-controller  Scaled up replica set kubernetes-bootcamp-57978f5f5d to 4
```

#### 負載均衡

讓我們檢查一下 Service 是否正在對流量進行負載平衡。要找出暴露的 IP 和端口，我們可以使用我們在上一個模塊中學習的描述服務：

```bash
kubectl describe services/kubernetes-bootcamp
```

輸出應類似於：

```hl_lines="9"
Name:                     kubernetes-bootcamp
Namespace:                default
Labels:                   app=kubernetes-bootcamp
Annotations:              <none>
Selector:                 app=kubernetes-bootcamp
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.43.10.162
IPs:                      10.43.10.162
Port:                     <unset>  8080/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  30000/TCP
Endpoints:                10.42.0.13:8080,10.42.0.14:8080,10.42.0.15:8080 + 1 more...
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

創建一個名為 NODE_PORT 的環境變量，其值為 Node 端口：

```bash
export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')

echo NODE_PORT=$NODE_PORT

```

接下來，我們將對暴露的 IP 和端口進行 curl。多次執行命令：

```bash
$ curl localhost:$NODE_PORT
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-57978f5f5d-844f2 | v=1

$ curl localhost:$NODE_PORT
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-57978f5f5d-lz4l7 | v=1

$ curl localhost:$NODE_PORT
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-57978f5f5d-844f2 | v=1

$ curl localhost:$NODE_PORT
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-57978f5f5d-lz4l7 | v=1

$ curl localhost:$NODE_PORT
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-57978f5f5d-844f2 | v=1

$ curl localhost:$NODE_PORT
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-57978f5f5d-844f2 | v=1

$ curl localhost:$NODE_PORT
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-57978f5f5d-n2lb8 | v=1

$ curl localhost:$NODE_PORT
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-57978f5f5d-lz4l7 | v=1
```

我們針對每個請求訪問不同的 Pod。這表明負載平衡正在工作。

### 縮小 Scale down

要將服務縮減為 2 個副本，請再次運行 scale 命令：

```bash
kubectl scale deployments/kubernetes-bootcamp --replicas=2
```

列出部署以檢查是否使用 `get deployments` 命令應用了更改：

```bash
kubectl get deployments
```

輸出應類似於：

```
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   2/2     2            2           43h
```

副本數減少到 2。列出 Pod 的數量，用 `get pods`：

```bash
kubectl get pods -o wide
```

輸出應類似於：

```
NAME                                   READY   STATUS    RESTARTS   AGE   IP           NODE                       NOMINATED NODE   READINESS GATES
kubernetes-bootcamp-57978f5f5d-844f2   1/1     Running   0          99m   10.42.0.15   k3d-k3s-default-server-0   <none>           <none>
kubernetes-bootcamp-57978f5f5d-csrqd   1/1     Running   0          99m   10.42.0.14   k3d-k3s-default-server-0   <none>           <none>
```

這證實了 2 個 Pod 被終止。


