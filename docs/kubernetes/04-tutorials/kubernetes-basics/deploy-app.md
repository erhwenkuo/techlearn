# 部署應用

## 使用 kubectl 創建部署

**目標:**

- 學習了解應用的部署
- 使用 kubectl 在 Kubernetes 上部署第一個應用

### Kubernetes 部署

一旦運行了 Kubernetes 集群，就可以在其上部署容器化應用程序。為此，你需要創建 Kubernetes **Deployment** 配置。 Deployment 指揮 Kubernetes 如何創建和更新應用程序的實例。創建 Deployment 後，Kubernetes master 將應用程序實例調度到集群中的各個節點上。

創建應用程序實例後，Kubernetes Deployment 控制器會持續監視這些實例。如果託管實例的節點關閉或被刪除，則 Deployment 控制器會將該實例替換為集群中另一個節點上的實例。**這提供了一種自我修復機制來解決機器故障維護問題**。

在沒有 Kubernetes 這種編排系統之前，安裝腳本通常用於啟動應用程序，但它們不允許從機器故障中恢復。通過創建應用程序實例並使它們在節點之間運行， Kubernetes Deployments 提供了一種與眾不同的應用程序管理方法。

### 部署你在 Kubernetes 上的第一個應用程序

![](./assets/module_02_first_app.svg)

你可以使用 Kubernetes 命令行界面 **Kubectl** 創建和管理 Deployment。 Kubectl 使用 Kubernetes API 與集群進行交互。在本單元中，你將學習創建在 Kubernetes 集群上運行應用程序的 Deployment 所需的最常見的 Kubectl 命令。

創建 Deployment 時，你需要指定應用程序的容器鏡像以及要運行的副本數。你可以稍後通過更新 Deployment 來更改該信息; 模塊 5 和 6 討論瞭如何擴展和更新 Deployments。

對於我們的第一次部署，我們將使用打包在 Docker 容器中的 Node.js 應用程序。要創建 Node.js 應用程序並部署 Docker 容器，請參考 [你好 K3D 教程](../hello-k3d.md).

現在你已經了解了 Deployment 的內容，讓我們開始部署我們的第一個應用程序！

=== "Declarative method"

    首先構建 Deployment 的配置檔 `k8s-bootcamp-deployment.yaml`:

    ```bash title="k8s-bootcamp-deployment.yaml"
    cat > k8s-bootcamp-deployment.yaml <<EOF
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app: kubernetes-bootcamp
      name: kubernetes-bootcamp
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: kubernetes-bootcamp
      template:
        metadata:
          labels:
            app: kubernetes-bootcamp
        spec:
          containers:
          - image: gcr.io/google-samples/kubernetes-bootcamp:v1
            name: kubernetes-bootcamp
    EOF
    ```

    套用 Deployment 的設置到 Kubernetes 集群:

    ```bash
    kubectl apply -f k8s-bootcamp-deployment.yaml
    ```

=== "Imperative method"

    ```bash
    kubectl create deployment kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1
    ```

太好了！您剛剛通過創建 deployment 物件來部署了您的第一個應用程序。在這個過程 Kubernetes 為您做了一些事情：

- 搜索可以運行應用程序實例的合適節點（我們只有 1 個可用節點）
- 安排應用程序在該節點上運行
- 將集群配置為在需要時在新節點上重新調度實例

要列出您的部署，請使用 `get deployments` 命令：

```bash
kubectl get deployments
```

輸出結果類似於這樣：

```
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   1/1     1            1           70s
```

### 查看剛佈署的應用

在 Kubernetes 中運行的 Pod 運行在一個私有、隔離的網絡上。默認情況下，它們對同一 kubernetes 集群中的其他 pod 和服務可見，但在該網絡之外不可見。當我們使用 kubectl 時，我們通過 API 端點進行交互以與我們的應用程序進行通信。我們將在模塊 4 中介紹如何在 Kubernetes 集群之外公開您的應用程序的其他選項。

暫時我們先使用 `kubectl port-forward` 命令來暴露出集群中的應用：

```bash
kubectl port-forward deployment.apps/kubernetes-bootcamp :8080 --address='0.0.0.0'
```

kubectl 工具會找到一個未被使用的本地端口號。輸出類似於：

```
Forwarding from 0.0.0.0:35729 -> 8080
```

!!! tip
    請注意在 `port-forward` 輸出中與 8080 對應的長度為 5 位的端口號。此端口號是隨機生成的，可能與你的不同。對應於上面的例子，集群應用被暴露出的本地端口號是 35729。

執行下列命令來取得服務的網頁:

```bash
curl localhost:35729
```

輸出結果類似於這樣：

```
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-57978f5f5d-kmvpk | v=1
```


