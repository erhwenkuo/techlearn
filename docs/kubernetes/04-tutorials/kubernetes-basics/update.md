# 更新你的應用

## 執行滾動更新

**目標:**

- 使用 kubectl 進行應用版本滾動更新。

### 更新應用程序

用戶希望應用程序始終可用，而開發人員則需要每天多次部署它們的新版本。在 Kubernetes 中，這些是通過滾動更新（Rolling Updates）完成的。**滾動更新** 允許通過使用新的實例逐步更新 Pod 實例，零停機進行 Deployment 更新。新的 Pod 將在具有可用資源的節點上進行調度。

在前面的教程中，我們將應用程序擴展為運行多個實例。這是在不影響應用程序可用性的情況下執行更新的要求。默認情況下，更新期間不可用的 pod 的最大值和可以創建的新 pod 數都是 1。這兩個選項都可以配置為（pod）數字或百分比。在 Kubernetes 中，更新是經過版本控制的，任何 Deployment 更新都可以恢復到以前的（穩定）版本。

![](./assets/module_06_rollingupdates1.svg)

![](./assets/module_06_rollingupdates2.svg)

![](./assets/module_06_rollingupdates3.svg)

![](./assets/module_06_rollingupdates4.svg)

與應用程序擴展類似，如果公開了 Deployment，服務將在更新期間僅對可用的 pod 進行負載均衡。可用 Pod 是應用程序用戶可用的實例。

滾動更新允許以下操作：

- 將應用程序從一個環境提升到另一個環境（通過容器鏡像更新）
- 回滾到以前的版本
- 持續集成和持續交付應用程序，無需停機

在下面的教程中，我們將應用程序更新為新版本，並執行回滾。

#### 更新應用程序的版本

要列出您的部署，請運行 `get deployments` 命令：

```bash
kubectl get deployments
```

輸出應類似於：

```
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   2/2     2            2           45h
```

要列出正在運行的 Pod，請運行 `get pods` 命令：

```bash
kubectl get pods
```

輸出應類似於：

```
NAME                                   READY   STATUS    RESTARTS   AGE
kubernetes-bootcamp-57978f5f5d-844f2   1/1     Running   0          111m
kubernetes-bootcamp-57978f5f5d-csrqd   1/1     Running   0          111m
```

要查看應用程序的當前映像版本，請運行 describe pods 命令並查找 Image 字段：

```bash
kubectl describe pods
```

輸出應類似於：

```
Name:         kubernetes-bootcamp-57978f5f5d-844f2
Namespace:    default
Priority:     0
Node:         k3d-k3s-default-server-0/172.24.0.2
Start Time:   Thu, 04 Aug 2022 13:46:51 +0800
Labels:       app=kubernetes-bootcamp
              pod-template-hash=57978f5f5d
Annotations:  <none>
Status:       Running
IP:           10.42.0.15
IPs:
  IP:           10.42.0.15
Controlled By:  ReplicaSet/kubernetes-bootcamp-57978f5f5d
Containers:
  kubernetes-bootcamp:
    Container ID:   containerd://152fce57df6ffd1e0be621be0b6852bde44d36e0a4f5e1574e342749638bccee
    Image:          gcr.io/google-samples/kubernetes-bootcamp:v1
    Image ID:       gcr.io/google-samples/kubernetes-bootcamp@sha256:0d6b8ee63bb57c5f5b6156f446b3bc3b3c143d233037f3a2f00e279c8fcc64af
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Thu, 04 Aug 2022 13:46:52 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-gk5rc (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  kube-api-access-gk5rc:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:                      <none>
```

要將應用程序的映像更新到 **版本 2**，請使用 `set image` 命令，後跟部署名稱和新的映像版本：


```bash
kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v2
```

該命令通知部署為您的應用程序使用不同的映像並啟動滾動更新。檢查新 Pod 的狀態，並使用 `get pods` 命令查看舊 Pod 終止：

```bash
kubectl get pods
```

輸出應類似於：

```
NAME                                  READY   STATUS    RESTARTS   AGE
kubernetes-bootcamp-769746fd4-mjdkd   1/1     Running   0          82s
kubernetes-bootcamp-769746fd4-r5rwb   1/1     Running   0          70s
```

#### 驗證更新

首先，檢查應用程序是否正在運行。要查找暴露的 IP 和端口，請運行 describe service 命令：

```bash
kubectl describe services/kubernetes-bootcamp
```

輸出應類似於：

```hl_lines="13"
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
Endpoints:                10.42.0.17:8080,10.42.0.18:8080
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

創建一個名為 NODE_PORT 的環境變量，該變量具有分配的節點端口的值：

```bash
export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')

echo NODE_PORT=$NODE_PORT
```

接下來，對暴露的 IP 和端口進行 curl：

```bash
$ curl localhost:$NODE_PORT
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-769746fd4-mjdkd | v=2

$ curl localhost:$NODE_PORT
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-769746fd4-r5rwb | v=2

$ curl localhost:$NODE_PORT
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-769746fd4-mjdkd | v=2

$ curl localhost:$NODE_PORT
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-769746fd4-r5rwb | v=2
```

每次運行 curl 命令時，都會遇到不同的 Pod。請注意，所有 Pod 都在運行最新版本 (v2)。

您還可以通過運行 `rollout status` 命令來確認更新：

```bash
kubectl rollout status deployments/kubernetes-bootcamp
```

輸出應類似於：

```
deployment "kubernetes-bootcamp" successfully rolled out
```

要查看應用程序的當前映像版本，請運行 `describe pods` 命令：

```bash
kubectl describe pods
```

在輸出的 Image 字段中，確認您正在運行最新的映像版本 (v2)。

#### 回滾更新

讓我們執行另一個更新，並部署一個帶有 v10 標記的映像：

```bash
kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=gcr.io/google-samples/kubernetes-bootcamp:v10
```

使用 get deployments 查看部署狀態：

```bash
kubectl get deployments
```

輸出應類似於：

```
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   2/2     1            2           45h
```

請注意，輸出沒有列出所需的可用 Pod 數量。運行 `get pods` 命令列出所有 Pod：

```bash
kubectl get pods
```

輸出應類似於：

```
NAME                                  READY   STATUS             RESTARTS   AGE
kubernetes-bootcamp-769746fd4-mjdkd   1/1     Running            0          11m
kubernetes-bootcamp-769746fd4-r5rwb   1/1     Running            0          11m
kubernetes-bootcamp-597654dbd-nwv76   0/1     ImagePullBackOff   0          2m20s
```

請注意，一些 Pod 的狀態為 ==ImagePullBackOff==。

要更深入地了解問題，請運行 `describe pods` 命令：

```bash
kubectl describe pods
```

輸出應類似於：

```hl_lines="9 22 50"
Name:         kubernetes-bootcamp-597654dbd-nwv76
Namespace:    default
Priority:     0
Node:         k3d-k3s-default-server-0/172.24.0.2
Start Time:   Thu, 04 Aug 2022 15:52:13 +0800
Labels:       app=kubernetes-bootcamp
              pod-template-hash=597654dbd
Annotations:  <none>
Status:       Pending
IP:           10.42.0.19
IPs:
  IP:           10.42.0.19
Controlled By:  ReplicaSet/kubernetes-bootcamp-597654dbd
Containers:
  kubernetes-bootcamp:
    Container ID:   
    Image:          gcr.io/google-samples/kubernetes-bootcamp:v10
    Image ID:       
    Port:           <none>
    Host Port:      <none>
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-cqlzt (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             False 
  ContainersReady   False 
  PodScheduled      True 
Volumes:
  kube-api-access-cqlzt:
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
  Type     Reason     Age                   From               Message
  ----     ------     ----                  ----               -------
  Normal   Scheduled  5m4s                  default-scheduler  Successfully assigned default/kubernetes-bootcamp-597654dbd-nwv76 to k3d-k3s-default-server-0
  Normal   Pulling    3m36s (x4 over 5m4s)  kubelet            Pulling image "gcr.io/google-samples/kubernetes-bootcamp:v10"
  Warning  Failed     3m36s (x4 over 5m3s)  kubelet            Failed to pull image "gcr.io/google-samples/kubernetes-bootcamp:v10": rpc error: code = NotFound desc = failed to pull and unpack image "gcr.io/google-samples/kubernetes-bootcamp:v10": failed to resolve reference "gcr.io/google-samples/kubernetes-bootcamp:v10": gcr.io/google-samples/kubernetes-bootcamp:v10: not found
  Warning  Failed     3m36s (x4 over 5m3s)  kubelet            Error: ErrImagePull
  Warning  Failed     3m20s (x6 over 5m2s)  kubelet            Error: ImagePullBackOff
  Normal   BackOff    3m9s (x7 over 5m2s)   kubelet            Back-off pulling image "gcr.io/google-samples/kubernetes-bootcamp:v10"
```

在受影響 Pod 的輸出的事件部分中，請注意存儲庫中不存在 v10 映像版本。

要將部署回滾到上一個工作版本，請使用 `rollout undo` 命令：

```bash
kubectl rollout undo deployments/kubernetes-bootcamp
```

`rollout undo` 命令將部署恢復到以前的已知狀態（映像的 v2）。更新是版本化的，您可以恢復到任何以前已知的部署狀態。

使用 get pods 命令再次列出 Pod：

```bash
kubectl get pods
```

四個 Pod 正在運行。要檢查部署在這些 Pod 上的映像，請使用 `describe pods` 命令：

```bash
kubectl describe pods
```

部署再次使用應用程序的穩定版本 (v2)。回滾成功。