# 創建集群

## 使用 K3D 創建集群

**目標:**

- 了解 Kubernetes 集群。
- 了解 K3D 。
- 使用 K3D 開啟一個 Kubernetes 集群。

### Kubernetes 集群

**Kubernetes 協調一個高可用計算機集群，每個計算機作為獨立單元互相連接工作**。 Kubernetes 中的抽象允許你將容器化的應用部署到集群，而無需將它們綁定到某個特定的獨立計算機。為了使用這種新的部署模型，應用需要以將應用與單個主機分離的方式打包：它們需要 **被容器化**。與過去的那種應用直接以包的方式深度與主機集成的部署模型相比，容器化應用更靈活、更可用。 **Kubernetes 以更高效的方式跨集群自動分發和調度應用容器**。 Kubernetes 是一個開源平台，並且可應用於生產環境。

一個 Kubernetes 集群包含兩種類型的資源:

- **Master** 調度整個集群
- **Nodes** 負責運行應用

![](./assets/module_01_cluster.svg)

**Master 負責管理整個集群**。 Master 協調集群中的所有活動，例如調度應用、維護應用的所需狀態、應用擴容以及推出新的更新。

**Node 是一個虛擬機或者物理機，它在 Kubernetes 集群中充當工作機器的角色** 每個Node都有 Kubelet , 它管理 Node 而且是 Node 與 Master 通信的代理。 Node 還應該具有用於​​處理容器操作的工具，例如 Docker 或 rkt 。處理生產級流量的 Kubernetes 集群至少應具有三個 Node，因為如果一個 Node 出現故障其對應的 etcd 成員和控制平面實例都會丟失，並且冗餘會受到影響。你可以通過添加更多控制平面節點來降低這種風險 。

在 Kubernetes 上部署應用時，你告訴 Master 啟動應用容器。 Master 就編排容器在集群的 Node 上運行。 **Node 使用 Master 暴露的 Kubernetes API 與 Master 通信**。終端用戶也可以使用 Kubernetes API 與集群交互。

Kubernetes 既可以部署在物理機上也可以部署在虛擬機上。你可以使用 K3D 開始部署 Kubernetes 集群。 K3D 是一種輕量級的 Kubernetes 實現，可在本地計算機上創建 dokcer 並部署僅包含一個節點的簡單集群。 K3D 可用於 Linux ， macOS 和 Windows 系統。 K3D CLI 提供了用於引導集群工作的多種操作，包括啟動、停止、查看狀態和刪除。

既然你已經知道 Kubernetes 是什麼，讓我們啟動我們的第一個 Kubernetes 集群！

### 創建 K3D 集群

K3D 默認使用 traefik，我們將通過 traefik 暴露佈署在 Kubernetes 裡的應用程序。如果您在遠程服務器上安裝了 K3D，則需要打開外部端口進行訪問。為此，我們使用以下命令打開負載均衡器。

```bash	
$ k3d cluster create -p 8081:80@loadbalancer -p 30000-30010:30000-30010@server:0
```

!!! info
    - 端口映射結構 8081:80@loadbalancer 的意思是：“將主機的 8081 端口映射到與 nodefilter 負載平衡器匹配的容器上的 80 端口”

檢查集群資訊：

```bash
kubectl cluster-info
```

輸出結果類似於這樣：

```
Kubernetes control plane is running at https://0.0.0.0:43893
CoreDNS is running at https://0.0.0.0:43893/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://0.0.0.0:43893/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

檢查集群節點資訊：

```bash
kubectl get nodes
```

輸出結果類似於這樣：

```
NAME                       STATUS   ROLES                  AGE   VERSION
k3d-k3s-default-server-0   Ready    control-plane,master   86s   v1.22.7+k3s1
```

此命令顯示可用於託管我們的應用程序的所有節點。現在我們只有一個節點，我們可以看到它的狀態是準備就緒（它準備好接受應用程序進行部署）。

