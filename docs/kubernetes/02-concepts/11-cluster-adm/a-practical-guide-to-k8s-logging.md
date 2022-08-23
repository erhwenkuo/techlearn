# Kubernetes 日誌記錄實用指南

原文: https://logz.io/blog/a-practical-guide-to-kubernetes-logging/

![](./assets/kubernetes-wide.jpg)

Kubernetes 已成為容器編排事實上的行業標準。它提供了有效管理具有聲明式配置、簡單部署機制以及擴展和自我修復功能的大型容器化應用程序所需的抽象。

與任何系統一樣，**日誌** 可以幫助工程師獲得對容器和它們所運行的 Kubernetes 集群的可觀察性，並且它們所起的關鍵作用在許多以 Kubernetes 故障為特徵的事件中是顯而易見的。然而 要完善地運維　Kubernetes 必需面監一系列獨特的日誌記錄挑戰。

Kubernetes 是一個高度分佈式和動態的環境。在生產環境中，您很可能會運行數十台機器和數百個容器，這些容器可以隨時終止、重新啟動或重新調度。系統的這種瞬態和動態特性本身就是一個挑戰。

Kubernetes 集群還包含需要監控的多個抽象層，每個抽象層層都會產生不同類型的日誌。

值得慶幸的是，有很多關於如何了解 Kubernetes 的文獻。還有各種日誌工具可以與 Kubernetes 原生集成，以簡化任務。在本文中，我們將回顧其中一些工具以及 Kubernetes 日誌架構。

## 一個簡單的例子：使用 Kubelet 進行容器化應用程序日誌記錄

### Logging to stdout and stderr standard output streams

可以從 Kubernetes 集群收集的第一層日誌是由 **容器化應用程序** 生成的日誌。{==最佳實踐是將應用程序日誌寫入標準輸出 (**stdout**) 和標準錯誤 (**stderr**) 流==}。您不必擔心丟失這些日誌，因為 Kubernetes 的節點代理 kubelet 會在後台收集這些流並將它們寫入本地文件，因此您可以使用 Kubernetes 訪問它們。

```yaml title="example.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: example
spec:
  containers:
  - name: example
    image: busybox
    args: [/bin/sh, -c, 'while true; do echo $(date); sleep 1; done']
```

要應用清單，請運行：

```bash
kubectl apply -f example.yaml
```

查看此容器的日誌：

```bash
kubectl logs pod/example
```

該命令調用該節點上的 kubelet 服務來檢索日誌。如您所見，日誌被收集並與 Kubernetes 一起呈現。這是針對集群中 pod 中的每個容器完成的。使用 kubectl 檢索日誌使您無需訪問集群中的各個節點。

Kubectl 一次只能顯示一個 pod 的日誌。如果您需要將多個 pod 聚合到一個流中，則需要使用 `kubetail` 命令，或者我們將在本文後面討論的更高級別的日誌聚合和管理工具。

### Using a sidecar for logging

如果您的應用程序不輸出到 stdout 和 stderr，那麼您可以在應用程序旁邊部署一個 sidecar 容器，它將獲取應用程序日誌並將它們分別流式傳輸到 stdout 和 stderr。

這樣的 sidecar 模式還可以執行一些日誌操作，例如將節點上的多個日誌流聚合為一個，或者將一個應用程序日誌流分成多個邏輯流（每個邏輯流由專用的 sidecar 實例處理）。

對於持久化容器日誌，常見的做法是將日誌寫入日誌文件，然後使用 sidecar 容器：

```yaml title="example2.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: example
spec:
  containers:
  - name: example
    image: busybox
    args:
    - /bin/sh
    - -c
    - >
      while true;
      do
      echo "$(date)\n" >> /var/log/example.log;
      sleep 1;
      done
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: sidecar
    image: busybox
    args: [/bin/sh, -c, 'tail -f /var/log/example.log']
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  volumes:
  - name: varlog
    emptyDir: {}
```

如上面的 pod 配置所示，sidecar 容器將與應用程序容器一起運行在同一個 pod 中，掛載相同的捲並分別處理日誌。

## Kubernetes 日誌架構

如前所述，記錄 Kubernetes 的一個主要挑戰是了解生成的日誌以及如何使用它們。在以下部分中，我將研究節點日誌記錄和集群日誌記錄。