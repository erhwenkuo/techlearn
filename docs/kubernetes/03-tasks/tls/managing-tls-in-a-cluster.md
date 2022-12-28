# 管理集群中的 TLS 認證

原文: [Manage TLS Certificates in a Cluster](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/)

Kubernetes 提供 `certificates.k8s.io` API，可讓你配置由你控制的證書頒發機構（CA）簽名的 TLS 證書。你的工作負載可以使用這些 CA 和證書來建立信任。

`certificates.k8s.io` API使用的協議類似於 [ACME 草案](https://github.com/ietf-wg-acme/acme/)。

## 準備開始

你必須擁有一個 Kubernetes 的集群，同時你的 Kubernetes 集群必須帶有 kubectl 命令行工具。建議在至少有兩個節點的集群上運行本教程，且這些節點不作為控制平面主機。如果你還沒有集群，你可以通過 Minikube 構建一個你自己的集群。

本文中某些步驟使用 `jq` 工具。如果你沒有 `jq`，你可以通過操作系統的軟件源安裝， 或者從 `https://stedolan.github.io/jq/` 獲取。

## 集群中的 TLS 信任

信任 Pod 中運行的應用程序所提供的自定義 CA 通常需要一些額外的應用程序配置。你需要將 CA 證書包添加到 TLS 客戶端或服務器信任的 CA 證書列表中。例如，你可以使用 Golang TLS 配置通過解析證書鏈並將解析的證書添加到 `tls.Config` 結構中的 RootCAs 字段中。

!!! info
    說明：
    即使自定義 CA 證書可能包含在文件系統中（在 ConfigMap `kube-root-ca.crt` 中）， 除了驗證內部 Kubernetes 端點之外，你不應將該證書頒發機構用於任何目的。內部 Kubernetes 端點的一個示例是默認命名空間中名為 kubernetes 的服務。

    如果你想為你的工作負載使用自定義證書頒發機構，你應該單獨生成該 CA， 並使用你的 Pod 有讀權限的 ConfigMap 分發該 CA 證書。

## 請求證書

以下部分演示如何為通過 DNS 訪問的 Kubernetes 服務創建 TLS 證書。

## 創建證書籤名請求

通過運行以下命令生成私鑰和證書籤名請求（或CSR）:

```bash
cat <<EOF | cfssl genkey - | cfssljson -bare server
{
  "hosts": [
    "my-svc.my-namespace.svc.cluster.local",
    "my-pod.my-namespace.pod.cluster.local",
    "192.0.2.24",
    "10.0.34.2"
  ],
  "CN": "my-pod.my-namespace.pod.cluster.local",
  "key": {
    "algo": "ecdsa",
    "size": 256
  }
}
EOF
```

其中 `192.0.2.24` 是服務的集群 IP，`my-svc.my-namespace.svc.cluster.local` 是服務的 DNS 名稱，`10.0.34.2` 是 Pod 的 IP，而 `my-pod.my-namespace.pod.cluster.local` 是 Pod 的 DNS 名稱。你能看到的輸出類似於：

```bash
2022/02/01 11:45:32 [INFO] generate received request
2022/02/01 11:45:32 [INFO] received CSR
2022/02/01 11:45:32 [INFO] generating key: ecdsa-256
2022/02/01 11:45:32 [INFO] encoded CSR
```

此命令生成兩個文件；它生成包含PEM 編碼 PKCS#10 證書請求的 `server.csr`，以及 PEM 編碼密鑰的 server-key.pem，用於待生成的證書。

## 創建證書籤名請求（CSR）對象發送到 Kubernetes API

使用以下命令創建 CSR YAML 文件，並發送到 API 服務器：

```bash
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: my-svc.my-namespace
spec:
  request: $(cat server.csr | base64 | tr -d '\n')
  signerName: example.com/serving
  usages:
  - digital signature
  - key encipherment
  - server auth
EOF
```

請注意，在步驟1 中創建的 `server.csr` 文件是 base64 編碼並存儲在 `.spec.request` 字段中的。你還要求提供“digital signature（數字簽名）”， “密鑰加密（key encipherment）” 和“服務器身份驗證（server auth）” 密鑰用途， 由 `example.com/serving` 示例簽名程序簽名的證書。你也可以要求使用特定的`signerName`。更多信息可參閱支持的簽署者名稱。


在 API server 中可以看到這些 CSR 處於 Pending 狀態。執行下面的命令你將可以看到：

```bash
kubectl describe csr my-svc.my-namespace
```

結果:

```
Name:                   my-svc.my-namespace
Labels:                 <none>
Annotations:            <none>
CreationTimestamp:      Tue, 01 Feb 2022 11:49:15 -0500
Requesting User:        yourname@example.com
Signer:                 example.com/serving
Status:                 Pending
Subject:
        Common Name:    my-pod.my-namespace.pod.cluster.local
        Serial Number:
Subject Alternative Names:
        DNS Names:      my-pod.my-namespace.pod.cluster.local
                        my-svc.my-namespace.svc.cluster.local
        IP Addresses:   192.0.2.24
                        10.0.34.2
Events: <none>
```

## 批准證書簽名請求（CSR）

[證書簽名請求](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/certificate-signing-requests/) 的批准或者是通過自動批准過程完成的，或由集群管理員一次性完成。如果你被授權批准證書請求，你可以使用 kubectl 來手動完成此操作；例如：

```bash
$ kubectl certificate approve my-svc.my-namespace

certificatesigningrequest.certificates.k8s.io/my-svc.my-namespace approved
```

你現在應該能看到如下輸出：

```bash
$ kubectl get csr

NAME                  AGE   SIGNERNAME            REQUESTOR              REQUESTEDDURATION   CONDITION
my-svc.my-namespace   10m   example.com/serving   yourname@example.com   <none>              Approved
```

這意味著證書請求已被批准，並正在等待請求的簽名者對其簽名。


## 簽名證書籤名請求（CSR）

接下來，你將扮演證書籤署者的角色，頒發證書並將其上傳到API 服務器。

簽名者通常會使用其 `signerName` 查看對象的 `CertificateSigningRequest` API， 檢查它們是否已被批准，為這些請求簽署證書，並使用已頒發的證書更新 API 對象狀態。

### 創建證書頒發機構

你需要授權在新證書上提供數字簽名。

首先，通過運行以下命令創建簽名證書：

```bash
cat <<EOF | cfssl gencert -initca - | cfssljson -bare ca
{
  "CN": "My Example Signer",
  "key": {
    "algo": "rsa",
    "size": 2048
  }
}
EOF
```

你應該看到類似於以下的輸出：

```bash
2022/02/01 11:50:39 [INFO] generating a new CA key and certificate from CSR
2022/02/01 11:50:39 [INFO] generate received request
2022/02/01 11:50:39 [INFO] received CSR
2022/02/01 11:50:39 [INFO] generating key: rsa-2048
2022/02/01 11:50:39 [INFO] encoded CSR
2022/02/01 11:50:39 [INFO] signed certificate with serial number 263983151013686720899716354349605500797834580472
```

這會產生一個證書頒發機構密鑰文件（`ca-key.pem`）和證書（`ca.pem`）。

### 頒發證書

```json title="tls/server-signing-config.json"
{
    "signing": {
        "default": {
            "usages": [
                "digital signature",
                "key encipherment",
                "server auth"
            ],
            "expiry": "876000h",
            "ca_constraint": {
                "is_ca": false
            }
        }
    }
}
```

使用 `server-signing-config.json` 簽名配置、證書頒發機構密鑰文件和證書來簽署證書請求：

```bash
kubectl get csr my-svc.my-namespace -o jsonpath='{.spec.request}' | \
  base64 --decode | \
  cfssl sign -ca ca.pem -ca-key ca-key.pem -config server-signing-config.json - | \
  cfssljson -bare ca-signed-server
```

你應該看到類似於以下的輸出：

```bash
2022/02/01 11:52:26 [INFO] signed certificate with serial number 576048928624926584381415936700914530534472870337
```

這會生成一個簽名的服務證書文件，`ca-signed-server.pem`。

### 上傳簽名證書

最後，在API 對象的狀態中填充簽名證書：

```bash
kubectl get csr my-svc.my-namespace -o json | \
  jq '.status.certificate = "'$(base64 ca-signed-server.pem | tr -d '\n')'"' | \
  kubectl replace --raw /apis/certificates.k8s.io/v1/certificatesigningrequests/my-svc.my-namespace/status -f -
```

批准CSR 並上傳簽名證書後，運行：

```bash
kubectl get csr
```

輸入類似於：

```bash
NAME                  AGE   SIGNERNAME            REQUESTOR              REQUESTEDDURATION   CONDITION
my-svc.my-namespace   20m   example.com/serving   yourname@example.com   <none>              Approved,Issued
```

## 下載證書並使用它

現在，作為請求用戶，你可以通過運行以下命令下載頒發的證書並將其保存到 `server.crt` 文件中：

CSR 被簽署並獲得批准後，你應該看到以下內容：

```bash
kubectl get csr my-svc.my-namespace -o jsonpath='{.status.certificate}' \
    | base64 --decode > server.crt
```

現在你可以將 `server.crt` 和 `server-key.pem` 填充到 Secret 中， 稍後你可以將其掛載到 Pod 中（例如，用於提供 HTTPS 的網絡服務器）。

```bash
$ kubectl create secret tls server --cert server.crt --key server-key.pem

secret/server created
```

最後，你可以將 `ca.pem` 填充到 ConfigMap 並將其用作信任根來驗證服務證書：

```bash
$ kubectl create configmap example-serving-ca --from-file ca.crt=ca.pem

configmap/example-serving-ca created
```

## 批准證書籤名請求（CSR)

Kubernetes 管理員（具有適當權限）可以使用 kubectl certificate approve 和 kubectl certificate deny 命令手動批准（或拒絕）證書籤名請求（CSR）。但是，如果你打算大量使用此API，則可以考慮編寫自動化的證書控制器。

無論上述機器或人使用kubectl，“批准者”的作用是驗證CSR 滿足如下兩個要求：

1. CSR 的 subject 控制用於簽署CSR 的私鑰。這解決了偽裝成授權主體的第三方的威脅。在上述示例中，此步驟將驗證該Pod 控制了用於生成 CSR 的私鑰。
2. CSR 的 subject 被授權在請求的上下文中執行。這點用於處理不期望的主體被加入集群的威脅。在上述示例中，此步驟將是驗證該Pod 是否被允許加入到所請求的服務中。

當且僅當滿足這兩個要求時，審批者應該批准CSR，否則拒絕 CSR。

有關證書批准和訪問控制的更多信息， 請閱讀證書籤名請求參考頁。

