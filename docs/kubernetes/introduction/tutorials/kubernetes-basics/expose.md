# 公開曝露你的應用

## 使用 Service 曝露你的應用

**目標:**

- 了解 Kubernetes 中的 Service
- 了解 標籤(Label) 和 標籤選擇器(Label Selector) 對像如何與 Service 關聯
- 在 Kubernetes 集群外用 Service 暴露應用

### Kubernetes Service 總覽

Kubernetes [Pod](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/) 是轉瞬即逝的。 Pod 實際上擁有 [生命週期](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/pod-lifecycle/)。

當一個工作 Node 掛掉後, 在 Node 上運行的 Pod 也會消亡。 [ReplicaSet](https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/replicaset/) 會自動地通過創建新的 Pod 驅動集群回到目標狀態，以保證應用程序正常運行。

換一個例子，考慮一個具有3個副本數的用作圖像處理的後端程序。這些副本是可替換的; 前端系統不應該關心後端副本，即使 Pod 丟失或重新創建。
也就是說，Kubernetes 集群中的每個 Pod (即使是在同一個 Node 上的 Pod )都有一個唯一的 IP 地址，因此需要一種方法自動協調 Pod 之間的變更，以便應用程序保持運行。

Kubernetes 中的服務(Service)是一種抽象概念，它定義了 Pod 的邏輯集和訪問 Pod 的協議。 Service 使從屬 Pod 之間的松耦合成為可能。和其他 Kubernetes 對像一樣, Service 用 YAML (更推薦) 或者 JSON 來定義. Service 下的一組 Pod 通常由 LabelSelector (請參閱下面的說明為什麼你可能想要一個 spec 中不包含 selector 的服務)來標記。

儘管每個 Pod 都有一個唯一的 IP 地址，但是如果沒有 Service ，這些 IP 不會暴露在集群外部。 Service 允許你的應用程序接收流量。 Service 也以用 ServiceSpec 標記 type 的方式來選撢暴露的型式:

- **ClusterIP** (默認) - 在集群的內部 IP 上公開 Service 。這種類型使得 Service 只能從集群內訪問。
- **NodePort** - 使用 NAT 在集群中每個選定 Node 的相同端口上公開 Service 。使用<NodeIP>:<NodePort> 從集群外部訪問 Service。是 ClusterIP 的超集。
- **LoadBalancer** - 在當前雲中創建一個外部負載均衡器(如果支持的話)，並為 Service 分配一個固定的外部IP。是 NodePort 的超集。
- **ExternalName** - 通過返回帶有該名稱的 CNAME 記錄，使用任意名稱(由 spec 中的externalName指定)公開 Service。不使用代理。這種類型需要kube-dns的v1.7或更高版本。

更多關於不同 Service 類型的信息可以在使用[Source IP 教程](https://kubernetes.io/zh-cn/docs/tutorials/services/source-ip/)。也請參閱 [連接應用程序和 Service](https://kubernetes.io/zh-cn/docs/concepts/services-networking/connect-applications-service) 。

另外，需要注意的是有一些 Service 的用例沒有在 spec 中定義 `selector`。**一個沒有 selector 創建的 Service 也不會創建相應的端點對象**。這允許用戶手動將服務映射到特定的端點。沒有 `selector` 的另一種可能是你嚴格使用type: ExternalName 來標記。

### Service 和 Label

![](./assets/module_04_services.svg)

Service 通過一組 Pod 路由通信。 Service 是一種抽象，它允許 Pod 死亡並在 Kubernetes 中復制，而不會影響應用程序。在依賴的 Pod (如應用程序中的前端和後端組件)之間進行發現和路由是由 Kubernetes Service 處理的。

Service 匹配一組 Pod 是使用 [標籤(Label)和選擇器(Selector)](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/labels), 它們是允許對 Kubernetes 中的對象進行邏輯操作的一種分組原語。標籤(Label)是附加在對像上的鍵/值對，可以以多種方式使用:

- 指定用於開發，測試和生產的對象
- 嵌入版本標籤
- 使用 Label 將對象進行分類

![](./assets/module_04_labels.svg)

標籤(Label)可以在創建時或之後附加到對象上。他們可以隨時被修改。現在使用 Service 發布我們的應用程序並添加一些 Label 。

#### 創建新服務

讓我們驗證我們的應用程序是否正在運行。我們將使用 kubectl get 命令並查找現有的 Pod：

```bash
kubectl get pods
```

輸出結果類似於這樣：

```
NAME                                   READY   STATUS    RESTARTS   AGE
kubernetes-bootcamp-57978f5f5d-jsdct   1/1     Running   0          124m
```

接下來，讓我們列出集群中的當前服務：

```bash
kubectl get services
```

輸出結果類似於這樣：

```
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.43.0.1    <none>        443/TCP   3h54m
```

我們有一個名為 kubernetes 的服務，它在 K3D 啟動集群時默認創建。要創建新服務並將其公開給外部流量，我們將使用帶有 NodePort 作為參數的公開命令。


```bash title="k8s-bootcamp-service-nodeport.yaml" hl_lines="11"
cat > k8s-bootcamp-service-nodeport.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  labels:
    app: kubernetes-bootcamp
  name: kubernetes-bootcamp
spec:
  ports:
  - port: 8080
    nodePort: 30000 # assign specific host port
    protocol: TCP
    targetPort: 8080
  selector:
    app: kubernetes-bootcamp
  type: NodePort
EOF
```

套用 Service 的設置到 Kubernetes 集群:

```bash
kubectl apply -f k8s-bootcamp-service-nodeport.yaml
```

讓我們再次運行 get services 命令：

```bash
kubectl get services
```

輸出結果類似於這樣：

```
NAME                  TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
kubernetes            ClusterIP   10.43.0.1      <none>        443/TCP          32m
kubernetes-bootcamp   NodePort    10.43.10.162   <none>        8080:30000/TCP   4m18s
```

我們現在有一個名為 kubernetes-bootcamp 的正在運行的服務。在這裡，我們看到服務收到了一個唯一的集群 IP、一個內部端口和一個外部 IP（節點的 IP）。

要找出外部打開的端口（通過 NodePort 選項），我們將運行 describe service 命令：

```bash
kubectl describe services/kubernetes-bootcamp
```

輸出結果類似於這樣：

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
Endpoints:                10.42.0.7:8080
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

現在我們可以使用 curl 來測試應用程序是否暴露在集群之外：

```bash
curl localhost:30000
```

輸出結果類似於這樣：

```
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-57978f5f5d-cmbm4 | v=1
```

