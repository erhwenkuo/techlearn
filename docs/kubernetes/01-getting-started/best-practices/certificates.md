# PKI 證書和要求

原文:[PKI certificates and requirements](https://kubernetes.io/docs/setup/best-practices/certificates/)

Kubernetes 需要 PKI 證書才能進行基於 TLS 的身份驗證。如果你是使用 [kubeadm](https://kubernetes.io/zh-cn/docs/reference/setup-tools/kubeadm/) 安裝的 Kubernetes， 則會自動生成集群所需的證書。你還可以生成自己的證書。例如，不將私鑰存儲在 API 服務器上，可以讓私鑰更加安全。此頁面說明了集群必需的證書。

## 集群是如何使用證書的

Kubernetes 需要 PKI 才能執行以下操作：

- Kubelet 的客戶端證書，用於 API 服務器身份驗證
- Kubelet 服務端證書， 用於 API 服務器與 Kubelet 的會話
- API 服務器端點的證書
- 集群管理員的客戶端證書，用於 API 服務器身份認證
- API 服務器的客戶端證書，用於和 Kubelet 的會話
- API 服務器的客戶端證書，用於和 etcd 的會話
- 控制器管理器的客戶端證書或 kubeconfig，用於和 API 服務器的會話
- 調度器的客戶端證書或 kubeconfig，用於和 API 服務器的會話
- 前端代理的客戶端及服務端證書

etcd 還實現了雙向TLS 來對客戶端和對其他對等節點進行身份驗證。

## 證書存放的位置

假如通過 kubeadm 安裝 Kubernetes，大多數證書都存儲在 `/etc/kubernetes/pki`。本文檔中的所有路徑都是相對於該目錄的，但用戶賬戶證書除外，kubeadm 將其放在 `/etc/kubernetes` 中。

## 手動配置證書

如果你不想通過 kubeadm 生成這些必需的證書，你可以使用一個單一的根 CA 來創建這些證書或者直接提供所有證書。參見[證書](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/certificates/)以進一步了解創建自己的證書機構。關於管理證書的更多信息，請參見使用 kubeadm 進行證書管理。


## 根 CA

你可以創建由管理員控制的根 CA。該根 CA 可以創建多個中間 CA，並將所有進一步的創建委託給 Kubernetes。

需要這些CA：

|路徑	|默認CN	|描述|
|----|-------|----|
|ca.crt,key	|kubernetes-ca	|Kubernetes 通用CA|
|etcd/ca.crt,key	|etcd-ca	|與etcd 相關的所有功能|
|front-proxy-ca.crt,key	|kubernetes-front-proxy-ca	|用於前端代理|

除了上面的 CA 之外，還需要獲取用於服務賬戶管理的密鑰對，也就是 `sa.key` 和 `sa.pub`。

下面的例子說明了上表中所示的 CA 密鑰和證書文件。

```bash
/etc/kubernetes/pki/ca.crt
/etc/kubernetes/pki/ca.key
/etc/kubernetes/pki/etcd/ca.crt
/etc/kubernetes/pki/etcd/ca.key
/etc/kubernetes/pki/front-proxy-ca.crt
/etc/kubernetes/pki/front-proxy-ca.key
```

## 所有的證書

如果你不想將CA 的私鑰拷貝至你的集群中，你也可以自己生成全部的證書。

需要這些證書：

|默認 CN	|父級 CA	|O（位於 Subject 中）	|kind	|主機(SAN)|
|--------|---------|-------------------|-----|--------|
|kube-etcd	|etcd-ca	|	|server, client	|`<hostname>, <Host_IP>, localhost,127.0.0.1`|
|kube-etcd-peer	|etcd-ca	|	|server, client	|`<hostname>, <Host_IP>, localhost,127.0.0.1`|
|kube-etcd-healthcheck-client	|etcd-ca	|	|client||
|kube-apiserver-etcd-client	|etcd-ca	|system:masters	|client|
|kube-apiserver	|kubernetes-ca	|	|server	|`<hostname>, <Host_IP>, <advertise_IP>,[1]`|
|kube-apiserver-kubelet-client	|kubernetes-ca	|system:masters	|client|
|front-proxy-client	|kubernetes-front-proxy-ca	|	|client|

