# 用 HTTP 客戶端調用 Kubernetes API

原文: https://iximiuz.com/en/posts/kubernetes-api-call-simple-http-client/

使用 CLI（如 curl）或 GUI（如 postman）HTTP 客戶端調用 Kubernetes API 有很多使用情境。例如，您可能需要對 Kubernetes 對象進行比 kubectl 提供的更細粒度的控制，或者只是想在嘗試從代碼訪問 API 之前探索它。

本文不僅僅是一個方便的命令列表，而是一個深思熟慮的演練，揭示了一些您在從命令行調用 Kubernetes API 時可能會偶然發現的有趣問題。它涵蓋以下主題：

- 如何獲取 Kubernetes API 服務器地址
- 如何向客戶端驗證 API 服務器
- 如何使用證書 (certificate) 向 API 服務器驗證客戶端
- 如何使用令牌 (token) 向 API 服務器驗證客戶端
  - 如何從 Pod 內部調用 Kubernetes API
- 如何使用 curl 對 Kubernetes 對象執行基本的 CRUD 操作
- 如何使用 kubectl 的 raw 模式直接訪問 Kubernetes API
  - 如何查看哪些 API 請求 kubectl 命令（如 apply 發送）

![](./assets/k8s-api-http-client.png)

## Setting up Kubernetes playground

如果你沒有 Kubernetes 集群可以玩，這裡是你可以使用 `arkade` 快速創建本地遊樂場的方法：

```bash
$ curl -sLS https://get.arkade.dev | sudo sh
$ arkade get minikube kubectl
$ minikube start --profile cluster1
```

## How to get Kubernetes API host and port

要調用任何 API，您首先需要知道其服務器地址。對於 Kubernetes，每個集群都有一個 API 服務器。因此，查找 API 主機和端口的最簡單方法是查看 kubectl cluster-info 輸出。例如，在我的 Vagrant 盒子上，它會產生以下訊息：

```bash
$ kubectl cluster-info
Kubernetes control plane is running at https://192.168.49.2:8443
CoreDNS is running at https://192.168.49.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
...
```

`cluster-info` 命令顯示在[當前上下文](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)中選擇的集群的 API 地址。但是，如果您有多個集群怎麼辦？

```bash hl_lines="13"
$ kubectl config view

apiVersion: v1
clusters:
- cluster:
    certificate-authority: /home/dxlab/.minikube/ca.crt
    extensions:
    - extension:
        last-update: Tue, 28 Jun 2022 05:55:10 CST
        provider: minikube.sigs.k8s.io
        version: v1.25.2
      name: cluster_info
    server: https://192.168.49.2:8443
  name: minikube
contexts:
- context:
    cluster: minikube
    extensions:
    - extension:
        last-update: Tue, 28 Jun 2022 05:55:10 CST
        provider: minikube.sigs.k8s.io
        version: v1.25.2
      name: context_info
    namespace: default
    user: minikube
  name: minikube
current-context: minikube
kind: Config
preferences: {}
users:
- name: minikube
  user:
    client-certificate: /home/dxlab/.minikube/profiles/minikube/client.crt
    client-key: /home/dxlab/.minikube/profiles/minikube/client.key
```

!!! info
    默認情況下，`kubectl` 在 `$HOME/.kube` 目錄中查找名為 `config` 的文件。那麼，為什麼不直接從這個文件中獲取 API 地址呢？
    原因是潛在的配置合併。通過將 `KUBECONFIG env var` 設置為以冒號分隔的位置列表，可以指定多個 `kubeconfig` 文件。在訪問集群之前，`kubectl` 會嘗試將所有 `kubeconfig` 文件的內容合併到一個配置中。

因此，從上面的列表中選擇正確的集群，讓我們嘗試向其 API 服務器發送請求：

```bash
$ KUBE_API=$(kubectl config view -o jsonpath='{.clusters[0].cluster.server}')
```

## How to call Kubernetes API using curl

實際上，任何 HTTP 客戶端（curl、httpie、wget 甚至 postman）都可以，但我將在本節中使用 `curl` 。

### Authenticating API server to client

讓我們從查詢 API 的 /version 端點開始：

```bash
$ curl $KUBE_API/version

curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

很顯然……我們連接失敗了！ 🙈

當我第一次偶然發現類似的錯誤時，我真的很困惑。但仔細想想，上述錯誤實際上是有道理的。默認情況下，Kubernetes 通過 HTTPS 公開其 API，特別是為了向客戶端保證 API 服務器的強身份。但是，minikube 使用自簽名證書來設定本地集群。因此，Kubernetes API 服務器的 TLS 證書原來是由證書頒發機構 (CA) minikubeCA 簽署的(self-signed)，該證書頒發機構 (CA) minikubeCA 對 curl 是未知的 。curl 無法信任它，因此請求失敗。

!!! info
    默認情況下，curl 信任底層操作系統所信任的同一組 CA。例如，在 Ubuntu 或 Debian 上，可以在 `/etc/ssl/certs/ca-certificates.crt` 中找到受信任的 CA 列表。顯然，minikube 不會將其證書添加到此文件中。

幸運的是，minikube 將 CA 證書保存到 `~/.minikube/ca.crt`：

```bash
$ cat ~/.minikube/ca.crt | openssl x509 -text
```

結果:

```bash
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 1 (0x1)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = minikubeCA
        Validity
            Not Before: Jun 12 06:14:14 2022 GMT
            Not After : Jun 10 06:14:14 2032 GMT
        Subject: CN = minikubeCA
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (2048 bit)
                Modulus:
                    00:c2:ee:45:b9:c6:63:1f:15:d5:e7:22:5a:92:48:
                    86:23:db:0c:a2:0c:e5:e9:b4:9b:29:75:04:c0:e2:
                    2b:f2:8e:2a:8a:47:06:6e:81:d0:13:90:0a:08:71:
                    99:23:01:57:47:9b:1d:00:8f:95:50:20:97:13:6c:
                    32:42:b5:04:c8:3a:0b:5d:30:9c:55:bd:28:53:1b:
                    87:de:04:a7:04:18:e9:16:38:47:38:aa:a7:08:35:
                    1c:62:c3:e5:2d:89:ae:0b:e3:3b:9d:be:a2:b8:3b:
                    e6:81:ac:75:12:fe:f7:9b:a1:1b:a8:46:f7:9d:b8:
                    ce:41:2b:95:de:98:1b:52:a6:59:b9:54:ce:60:d6:
                    f3:2b:a5:b8:43:e1:ca:87:5c:8c:61:f8:59:4b:47:
                    1c:73:20:ad:3d:8e:fa:0e:a2:c5:b7:2d:8b:9a:85:
                    3f:6c:39:46:16:e8:4f:a9:91:e2:86:02:f3:2d:66:
                    7d:6c:74:66:23:7a:34:24:85:d9:f6:20:b0:39:7d:
                    68:86:d0:67:f1:90:df:ab:bd:52:85:5c:bb:99:a0:
                    ab:b9:a5:bd:1e:08:69:8e:7e:e2:e0:d5:f2:a5:40:
                    76:e7:03:de:aa:e3:f9:16:11:c9:ed:a3:8f:9a:c3:
                    48:44:20:5e:4c:8d:13:fa:a0:f7:68:86:f5:8f:5f:
                    97:3f
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment, Certificate Sign
            X509v3 Extended Key Usage: 
                TLS Web Client Authentication, TLS Web Server Authentication
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Subject Key Identifier: 
                70:53:86:8E:01:AE:E3:4F:57:8F:B0:30:C3:31:B0:5E:31:DB:7F:96
    Signature Algorithm: sha256WithRSAEncryption
         56:3c:de:59:09:ce:7b:bf:92:b0:fd:44:ca:e7:83:48:a2:f8:
         a7:3d:01:b2:62:50:62:cb:c0:7c:33:5d:25:a2:f4:46:af:88:
         b0:02:db:84:e6:4c:cb:a3:74:46:ed:ef:96:48:15:f9:cb:d7:
         07:42:0e:b0:3f:bf:bd:2b:e6:02:66:9c:a6:b4:5c:de:f9:0a:
         5b:68:cc:7a:0f:ea:1a:b5:4e:ff:ff:95:f7:b9:c4:a1:1f:e5:
         76:62:f4:c7:82:2d:9d:a6:79:99:95:f2:f7:77:58:11:87:0a:
         76:88:e3:32:fe:4a:18:99:fd:a2:b8:50:29:fb:4a:f5:74:4c:
         b2:0c:b5:42:75:93:6c:b9:80:db:10:8b:3e:49:c2:f2:15:84:
         e4:0f:fd:9b:21:05:ed:0e:66:85:04:58:ff:d8:c3:e9:e4:44:
         c4:30:28:fa:7f:b8:79:3c:8e:c3:28:64:cd:6c:ea:db:57:a2:
         e9:b3:26:2a:d2:18:4e:f5:b9:1d:23:48:cf:be:f7:85:6c:7e:
         dd:c8:85:78:ae:92:a7:1e:2c:0e:2a:c6:5f:3a:9c:84:8f:2d:
         cd:20:ac:30:a6:79:f3:4d:08:8e:e3:93:32:81:01:dc:c5:7b:
         0d:a8:d8:e3:13:0b:d6:68:1f:3e:69:10:16:4a:9e:77:4a:61:
         8f:94:e5:3c
-----BEGIN CERTIFICATE-----
MIIDBjCCAe6gAwIBAgIBATANBgkqhkiG9w0BAQsFADAVMRMwEQYDVQQDEwptaW5p
a3ViZUNBMB4XDTIyMDYxMjA2MTQxNFoXDTMyMDYxMDA2MTQxNFowFTETMBEGA1UE
AxMKbWluaWt1YmVDQTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAMLu
RbnGYx8V1eciWpJIhiPbDKIM5em0myl1BMDiK/KOKopHBm6B0BOQCghxmSMBV0eb
HQCPlVAglxNsMkK1BMg6C10wnFW9KFMbh94EpwQY6RY4Rziqpwg1HGLD5S2Jrgvj
O52+org75oGsdRL+95uhG6hG9524zkErld6YG1KmWblUzmDW8yuluEPhyodcjGH4
WUtHHHMgrT2O+g6ixbcti5qFP2w5RhboT6mR4oYC8y1mfWx0ZiN6NCSF2fYgsDl9
aIbQZ/GQ36u9UoVcu5mgq7mlvR4IaY5+4uDV8qVAducD3qrj+RYRye2jj5rDSEQg
XkyNE/qg92iG9Y9flz8CAwEAAaNhMF8wDgYDVR0PAQH/BAQDAgKkMB0GA1UdJQQW
MBQGCCsGAQUFBwMCBggrBgEFBQcDATAPBgNVHRMBAf8EBTADAQH/MB0GA1UdDgQW
BBRwU4aOAa7jT1ePsDDDMbBeMdt/ljANBgkqhkiG9w0BAQsFAAOCAQEAVjzeWQnO
e7+SsP1EyueDSKL4pz0BsmJQYsvAfDNdJaL0Rq+IsALbhOZMy6N0Ru3vlkgV+cvX
B0IOsD+/vSvmAmacprRc3vkKW2jMeg/qGrVO//+V97nEoR/ldmL0x4ItnaZ5mZXy
93dYEYcKdojjMv5KGJn9orhQKftK9XRMsgy1QnWTbLmA2xCLPknC8hWE5A/9myEF
7Q5mhQRY/9jD6eRExDAo+n+4eTyOwyhkzWzq21ei6bMmKtIYTvW5HSNIz773hWx+
3ciFeK6Spx4sDirGXzqchI8tzSCsMKZ5800IjuOTMoEB3MV7DajY4xML1mgfPmkQ
Fkqed0phj5TlPA==
-----END CERTIFICATE-----
```

因此，要修復 `GET /version` 請求的問題，我只需要通過手動將其指向 minikubeCA 證書來使 curl 信任 API 服務器證書的頒發者：

```bash
$ curl --cacert ~/.minikube/ca.crt $KUBE_API/version

{
  "major": "1",
  "minor": "23",
  "gitVersion": "v1.23.3",
  "gitCommit": "816c97ab8cff8a1c72eccca1026f7820e93e0d25",
  "gitTreeState": "clean",
  "buildDate": "2022-01-25T21:19:12Z",
  "goVersion": "go1.17.6",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

!!! info
    或者，您可以在　**curl** 命令中使用 `--insecure` 標誌或其短別名 `-k`。在安全的環境中，我更喜歡 insecure mode，它比試圖找到頒發者證書更簡單。

### Authenticating using Certificates

好的，讓我們嘗試一些更複雜的東西。列出集群中的所有部署怎麼樣？

```bash
$ curl --cacert ~/.minikube/ca.crt $KUBE_API/apis/apps/v1/deployments

{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "deployments.apps is forbidden: User \"system:anonymous\" cannot list resource \"deployments\" in API group \"apps\" at the cluster scope",
  "reason": "Forbidden",
  "details": {
    "group": "apps",
    "kind": "deployments"
  },
  "code": 403
}
```

與明顯不受保護的 `/version` 端點不同，Kubernetes 通常會限制對其 API 端點的訪問。

從錯誤消息中可以清楚地看出，該請求被驗證為用戶 `system:anonymous`，顯然，該用戶未授權可呼叫列出部署資源的 API 權限。

這個失敗的請求導因於我們並未檢附任何身份驗證資訊（儘管如此，它已經過身份驗證，但作為匿名用戶），所以我需要提供一些額外的信息來獲得所需的訪問級別。

Kubernetes 支持多種[身份驗證機制](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)，我將從使用客戶端證書(client certificate)對請求進行身份驗證開始。

但是等一下！什麼是客戶證書(client certificate)？

當 minikube 設定集群時，它還創建了一個`用戶`。該`用戶`獲得了由同一個 minikubeCA 頒發機構簽署的證書。由於 Kubernetes API 服務器信任此 CA，因此在請求中提供此證書將使其作為所述用戶進行身份驗證。

!!! info
    Kubernetes 沒有代表`用戶`的對象。即不能通過 API 調用將用戶添加到集群中。但是，任何提供由集群的證書頒發機構簽名的有效證書的用戶都被視為已通過身份驗證。 Kubernetes 從證書主題中的通用名稱字段中獲取用戶名（例如，CN = minikube-user）。然後，Kubernetes RBAC 子系統判斷用戶是否有權對資源執行特定操作。

用戶證書通常可以在我們已經熟悉的 kubectl 配置視圖輸出中找到：

```bash hl_lines="3"
$ kubectl config view -o jsonpath='{.users[0]}' | jq
{
  "name": "minikube",
  "user": {
    "client-certificate": "/home/dxlab/.minikube/profiles/minikube/client.crt",
    "client-key": "/home/dxlab/.minikube/profiles/minikube/client.key"
  }
}
```

讓我們快速檢查證書內容以確保它是由同一個 CA 簽名的：

```bash hl_lines="12"
$ cat ~/.minikube/profiles/cluster1/client.crt | openssl x509 -text

Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 2 (0x2)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = minikubeCA
        Validity
            Not Before: Jun 26 21:54:59 2022 GMT
            Not After : Jun 26 21:54:59 2025 GMT
        Subject: O = system:masters, CN = minikube-user
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (2048 bit)
                Modulus:
                    00:a6:eb:74:10:82:be:42:94:2b:db:72:a6:17:7d:
                    a5:9e:b4:d1:1b:34:c5:a4:71:ff:03:68:1b:1e:cb:
                    77:57:bb:6f:fd:0e:d7:b5:62:8f:c8:66:22:36:7d:
                    01:d3:f3:4b:3d:84:6a:03:0b:6f:e4:8e:3d:50:24:
                    a4:ad:e0:89:1a:3f:05:af:d8:67:63:9d:44:92:73:
                    93:ef:65:08:7e:05:76:45:69:6e:e7:b8:50:81:b6:
                    8b:82:a7:85:9b:9d:84:2d:db:d5:f5:73:a9:eb:08:
                    a5:0b:f2:61:b8:66:a5:f9:84:09:68:8d:9f:56:8e:
                    1f:a7:71:f6:4f:67:31:03:8d:a2:17:2a:42:f4:de:
                    96:11:2e:22:da:fa:f6:26:f2:de:7b:99:bb:66:b3:
                    d0:3e:4a:f6:c3:7e:6d:0f:d5:ef:44:54:fd:7f:c1:
                    c9:65:f7:7f:d4:50:73:2b:2a:ce:84:19:b0:a7:8a:
                    66:cb:ed:f8:60:3a:4a:ac:36:e6:5a:8f:21:eb:4f:
                    f8:fd:db:d3:55:d8:41:a9:34:00:20:9f:eb:9c:58:
                    fd:32:0c:d5:a8:3d:13:82:5c:ef:5c:e6:a5:ac:a8:
                    fb:71:ec:1a:d1:db:ad:f9:2c:40:3b:ee:59:0f:70:
                    36:b1:8a:ab:95:1e:e6:0b:1e:01:c4:ab:a7:5c:c6:
                    10:43
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Authority Key Identifier: 
                keyid:70:53:86:8E:01:AE:E3:4F:57:8F:B0:30:C3:31:B0:5E:31:DB:7F:96

    Signature Algorithm: sha256WithRSAEncryption
         a3:4e:82:61:63:b9:bc:5c:3f:ba:4d:00:ae:b4:f0:a7:a3:9f:
         39:5b:2b:82:0c:4f:d8:a6:ef:07:45:de:b1:0f:85:70:50:8c:
         b3:9f:1f:a6:31:34:20:ee:59:89:3b:c5:03:5f:f0:8a:1c:b5:
         c4:44:c6:45:c4:f4:51:1c:28:4b:b0:0b:52:41:55:4a:04:06:
         38:0d:ab:65:e7:7a:6f:0b:eb:3c:7e:92:7f:f8:30:b8:73:dd:
         62:1c:19:f4:66:ff:88:57:f8:10:23:1b:c1:42:04:41:a1:94:
         13:d3:51:bb:8f:52:bf:f3:ff:50:66:cf:9e:e0:d4:1e:0b:44:
         7f:e3:8e:2a:b9:07:62:a3:98:e1:50:c6:82:71:de:da:c8:83:
         4a:6d:8f:78:f7:aa:53:5c:ed:68:db:b5:09:b2:1b:94:80:50:
         71:12:cb:92:c2:a3:5f:f1:9d:12:4f:b0:0a:08:39:25:d9:19:
         b3:ca:15:51:d6:01:b6:7d:58:81:cc:b5:85:47:b0:ff:d4:b3:
         63:ce:2e:b9:12:eb:c6:0e:c3:5f:88:62:7c:39:5f:2d:c5:d1:
         56:a6:68:7b:c1:29:5c:69:dc:ee:cf:af:92:6b:ad:3c:e8:bd:
         6c:b9:1e:9d:35:3a:93:c1:4a:f2:64:d2:1b:bb:79:48:bc:89:
         51:81:0d:da
-----BEGIN CERTIFICATE-----
MIIDITCCAgmgAwIBAgIBAjANBgkqhkiG9w0BAQsFADAVMRMwEQYDVQQDEwptaW5p
a3ViZUNBMB4XDTIyMDYyNjIxNTQ1OVoXDTI1MDYyNjIxNTQ1OVowMTEXMBUGA1UE
ChMOc3lzdGVtOm1hc3RlcnMxFjAUBgNVBAMTDW1pbmlrdWJlLXVzZXIwggEiMA0G
CSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQCm63QQgr5ClCvbcqYXfaWetNEbNMWk
cf8DaBsey3dXu2/9Dte1Yo/IZiI2fQHT80s9hGoDC2/kjj1QJKSt4IkaPwWv2Gdj
nUSSc5PvZQh+BXZFaW7nuFCBtouCp4WbnYQt29X1c6nrCKUL8mG4ZqX5hAlojZ9W
jh+ncfZPZzEDjaIXKkL03pYRLiLa+vYm8t57mbtms9A+SvbDfm0P1e9EVP1/wcll
93/UUHMrKs6EGbCnimbL7fhgOkqsNuZajyHrT/j929NV2EGpNAAgn+ucWP0yDNWo
PROCXO9c5qWsqPtx7BrR2635LEA77lkPcDaxiquVHuYLHgHEq6dcxhBDAgMBAAGj
YDBeMA4GA1UdDwEB/wQEAwIFoDAdBgNVHSUEFjAUBggrBgEFBQcDAQYIKwYBBQUH
AwIwDAYDVR0TAQH/BAIwADAfBgNVHSMEGDAWgBRwU4aOAa7jT1ePsDDDMbBeMdt/
ljANBgkqhkiG9w0BAQsFAAOCAQEAo06CYWO5vFw/uk0ArrTwp6OfOVsrggxP2Kbv
B0XesQ+FcFCMs58fpjE0IO5ZiTvFA1/wihy1xETGRcT0URwoS7ALUkFVSgQGOA2r
Zed6bwvrPH6Sf/gwuHPdYhwZ9Gb/iFf4ECMbwUIEQaGUE9NRu49Sv/P/UGbPnuDU
HgtEf+OOKrkHYqOY4VDGgnHe2siDSm2PePeqU1ztaNu1CbIblIBQcRLLksKjX/Gd
Ek+wCgg5JdkZs8oVUdYBtn1Ygcy1hUew/9SzY84uuRLrxg7DX4hifDlfLcXRVqZo
e8EpXGnc7s+vkmutPOi9bLkenTU6k8FK8mTSG7t5SLyJUYEN2g==
-----END CERTIFICATE-----
```

下面是如何使用 curl 向 Kubernetes API 服務器發送由該證書認證的請求：

```bash
$ curl $KUBE_API/apis/apps/v1/deployments \
  --cacert ~/.minikube/ca.crt \
  --cert ~/.minikube/profiles/minikube/client.crt \
  --key ~/.minikube/profiles/minikube/client.key

{
  "kind": "DeploymentList",
  "apiVersion": "apps/v1",
  "metadata": {
    "resourceVersion": "654514"
  },
  "items": [...]
- "iss" (Issuer) Claim - The "iss" (issuer) claim identifies the principal that issued the JWT. 
- "sub" (Subject) Claim -The "sub" (subject) claim identifies the principal that is the subject of the JWT.
}
```

### Authenticating using Service Account Tokens

另一種驗證 API 請求的方法是使用`服務帳戶`的 JWT 令牌。與用戶非常相似，不同的`服務帳戶`具有不同級別的訪問權限。讓我們看看使用默認命名空間中的默認服務帳戶可以實現什麼：

```bash
$ JWT_TOKEN_DEFAULT_DEFAULT=$(kubectl get secrets \
    $(kubectl get serviceaccounts/default -o jsonpath='{.secrets[0].name}') \
    -o jsonpath='{.data.token}' | base64 --decode)
```

把 JWT 令牌打印出來看:

```bash
$ echo $JWT_TOKEN_DEFAULT_DEFAULT

eyJhbGciOiJSUzI1NiIsImtpZCI6IjFVa2pVaUVUS25JSzFNT1V0RW1zTWEyT0F3MTFSaFNneUVobXk0dUhjUEUifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlZmF1bHQtdG9rZW4tNng1ZHEiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGVmYXVsdCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImRiZjA0ZDczLWZlOTctNDMzMi04NDM2LWVjNDdlMDZkYzE2MSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.pkJ-AZufRuGcMGXwtL7jQu0BOa6DGgylzTjGcKQcOESjXeTsZfw8pnwHMOJsiV760QMLY33GtpzLy0om5l4gfU1O-CJZXKTaYCHzjyEb6ZVLK0xLeLwsW--LI_Weo0X9KiAjNBqDSUmj8NYHn2-c-wygN6IYWqbT8MmXL-YC8dejeqUlLzAQUw8hZasdWN6KLnmNxhI0-cntp0C8O2I5SPumOxq7IEX8s2Erir2cU2xGI-LPaMm8ccAk5wrKS9rqk23G24vXFgi9u1SwtujC_hZrki_rm8yrMUWvOa5SPZqAag_dkjLXR5YSxF_nJO69XGJ1fId3hr2ruHBTivbfEQ
```

![](./assets/default-serviceaccount-jwt.png)

其中 `payload` 的資訊如下:

```json title="payload"
{
  "iss": "kubernetes/serviceaccount",
  "kubernetes.io/serviceaccount/namespace": "default",
  "kubernetes.io/serviceaccount/secret.name": "default-token-6x5dq",
  "kubernetes.io/serviceaccount/service-account.name": "default",
  "kubernetes.io/serviceaccount/service-account.uid": "dbf04d73-fe97-4332-8436-ec47e06dc161",
  "sub": "system:serviceaccount:default:default"
}
```

從一個簡單的任務開始 - 在 `apps/v1` 中列出已知的 API 資源類型：

```bash
$ curl $KUBE_API/apis/apps/v1/ \
  --cacert ~/.minikube/ca.crt  \
  --header "Authorization: Bearer $JWT_TOKEN_DEFAULT_DEFAULT"

{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "apps/v1",
  "resources": [...]
}
```

接著讓我們嘗試在默認命名空間中列出實際的部署對象：

```bash
$ curl $KUBE_API/apis/apps/v1/namespaces/default/deployments \
  --cacert ~/.minikube/ca.crt  \
  --header "Authorization: Bearer $JWT_TOKEN_DEFAULT_DEFAULT"

{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "deployments.apps is forbidden: User \"system:serviceaccount:default:default\" cannot list resource \"deployments\" in API group \"apps\" in the namespace \"default\"",
  "reason": "Forbidden",
  "details": {
    "group": "apps",
    "kind": "deployments"
  },
  "code": 403
}
```

顯然，用戶 `system:serviceaccount:default:default` 甚至不足以在其自己的命名空間中列出 Kubernetes 對象。

讓我們嘗試一個強大的 `kube-system` 服務帳戶：

```bash
$ JWT_TOKEN_KUBESYSTEM_DEFAULT=$(kubectl -n kube-system get secrets \
    $(kubectl -n kube-system get serviceaccounts/default -o jsonpath='{.secrets[0].name}') \
    -o jsonpath='{.data.token}' | base64 --decode)
```

把 JWT 令牌打印出來看:

```bash
$ echo $JWT_TOKEN_KUBESYSTEM_DEFAULT

eyJhbGciOiJSUzI1NiIsImtpZCI6IjFVa2pVaUVUS25JSzFNT1V0RW1zTWEyT0F3MTFSaFNneUVobXk0dUhjUEUifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkZWZhdWx0LXRva2VuLW16bWo3Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImRlZmF1bHQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJhN2E4YTc1Mi1jNjQ5LTRkOGYtOTQzMi03MDgxYjE4YzBiNTIiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06ZGVmYXVsdCJ9.talXOCcsaH5VaZEOJo-VilVZUhM6YWrChmKScXHcTwDfVBmX6XlrelomnYMUtwahnF1aGe_vJpLzk-SJhvdLvFSAvw0pIWxsabes7ITfCdOfAMliPhiCIOC4XwawKWmbVJYcvhHppNA41qzNAbtav_e9qB6z84fmAQHQJLITMmMfe-ZC38I3sWh5ezoo-1uiV3iY6SHxd4WGXmuRBqkvWxJM7JiC2TmUU9Z6K4gfrRtfBrB2-A3Ti6v7Jcqwxakqq092X7VpstmJTHgh6KExm-0laIUSOKwUFlFvs4-MYKwmbYdz4IYDxJejZjcfC8XkTM6nQPaXdaTA6olKBzKHKg
```

![](./assets/kube-system-jwt.png)

其中 `payload` 的資訊如下:

```json title="payload"
{
  "iss": "kubernetes/serviceaccount",
  "kubernetes.io/serviceaccount/namespace": "kube-system",
  "kubernetes.io/serviceaccount/secret.name": "default-token-mzmj7",
  "kubernetes.io/serviceaccount/service-account.name": "default",
  "kubernetes.io/serviceaccount/service-account.uid": "a7a8a752-c649-4d8f-9432-7081b18c0b52",
  "sub": "system:serviceaccount:kube-system:default"
}
```

強大的帳戶值得一項具有挑戰性的任務 - 列出集群級資源：

```bash
$ curl $KUBE_API/apis/apps/v1/deployments \
  --cacert ~/.minikube/ca.crt  \
  --header "Authorization: Bearer $JWT_TOKEN_KUBESYSTEM_DEFAULT"

{
  "kind": "DeploymentList",
  "apiVersion": "apps/v1",
  "metadata": {
    "resourceVersion": "656580"
  },
  "items": [...]
}
```

是的，符合我們的期待地完成工作👌

#### Call Kubernetes API inside a Pod

與任何其他 Kubernetes 服務非常相似，Kubernetes API 服務地址可通過環境變量提供給 Pod：

```bash
$ kubectl run -it --image curlimages/curl --restart=Never mypod -- sh

$ env | grep KUBERNETES
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_SERVICE_HOST=10.96.0.1
```

Pod 通常還會在 `/var/run/secrets/kubernetes.io/serviceaccount/` 上安裝 `Kubernetes CA` 證書和`服務帳戶` credentials。因此，應用以上部分的知識，從 Pod 調用 Kubernetes API 服務器的 curl 命令可以如下所示：

```bash
$ curl https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}/apis/apps/v1 \
  --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  --header "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)"
```

### Creat, Read, Watch, Update, Patch, and Delete objects

Kubernetes API 支持對 Kubernetes 對象進行以下[操作](https://github.com/kubernetes/community/blob/7f3f3205448a8acfdff4f1ddad81364709ae9b71/contributors/devel/sig-architecture/api-conventions.md#verbs-on-resources)：

- `GET    /<resourcePlural>` - Retrieve a list of type <resourceName>.
- `POST   /<resourcePlural>` - Create a new resource from the JSON object provided by the client.
- `GET    /<resourcePlural>/<name>` - Retrieves a single resource with the given name.
- `DELETE /<resourcePlural>/<name>` - Delete the single resource with the given name.
- `DELETE /<resourcePlural>` - Deletes a list of type <resourceName>.
- `PUT    /<resourcePlural>/<name>` - Update or create the resource with the given name with the JSON object provided by client.
- `PATCH  /<resourcePlural>/<name>` - Selectively modify the specified fields of the resource.
- `GET    /<resourcePlural>?watch=true` - Receive a stream of JSON objects corresponding to changes made to any resource of the given kind over time.

API 是 RESTful 的型式，因此上述 HTTP 方法在資源操作上的映射應該看起來很熟悉。

!!! info
    即使文檔僅提及 JSON 對象，如果 Content-Type 標頭設置為 `application/yaml`，則支持提交 YAML 有效負載。

**CREATE**

以下是使用 curl 和 YAML 清單創建新對象的方法：

```bash
$ curl $KUBE_API/apis/apps/v1/namespaces/default/deployments \
  --cacert ~/.minikube/ca.crt \
  --cert ~/.minikube/profiles/minikube/client.crt \
  --key ~/.minikube/profiles/minikube/client.key \
  -X POST \
  -H 'Content-Type: application/yaml' \
  -d '---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sleep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sleep
  template:
    metadata:
      labels:
        app: sleep
    spec:
      containers:
      - name: sleep
        image: curlimages/curl
        command: ["/bin/sleep", "365d"]
'
```

**READ**

以下是如何獲取默認命名空間中的所有對象：

```bash
$ curl $KUBE_API/apis/apps/v1/namespaces/default/deployments \
  --cacert ~/.minikube/ca.crt \
  --cert ~/.minikube/profiles/minikube/client.crt \
  --key ~/.minikube/profiles/minikube/client.key
```

以及如何通過名稱和命名空間獲取對象：

```bash
$ curl $KUBE_API/apis/apps/v1/namespaces/default/deployments/sleep \
  --cacert ~/.minikube/ca.crt \
  --cert ~/.minikube/profiles/minikube/client.crt \
  --key ~/.minikube/profiles/minikube/client.key
```

一種更高級的檢索 Kubernetes 資源的方法是持續觀察它們的變化：

```bash
$ curl $KUBE_API/apis/apps/v1/namespaces/default/deployments?watch=true \
  --cacert ~/.minikube/ca.crt \
  --cert ~/.minikube/profiles/minikube/client.crt \
  --key ~/.minikube/profiles/minikube/client.key
```

!!! info
    請注意，只能監視一組資源。但是，您可以通過提供標籤或字段選擇器將結果集縮小到單個資源。

**UPDATE**

以下是更新現有對象的方法：

```bash
$ curl $KUBE_API/apis/apps/v1/namespaces/default/deployments/sleep \
  --cacert ~/.minikube/ca.crt \
  --cert ~/.minikube/profiles/minikube/client.crt \
  --key ~/.minikube/profiles/minikube/client.key \
  -X PUT \
  -H 'Content-Type: application/yaml' \
  -d '---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sleep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sleep
  template:
    metadata:
      labels:
        app: sleep
    spec:
      containers:
      - name: sleep
        image: curlimages/curl
        command: ["/bin/sleep", "730d"]  # <-- Making it sleep twice longer
'
```

以下是如何修補現有對象：

```bash
$ curl $KUBE_API/apis/apps/v1/namespaces/default/deployments/sleep \
  --cacert ~/.minikube/ca.crt \
  --cert ~/.minikube/profiles/minikube/client.crt \
  --key ~/.minikube/profiles/minikube/client.key \
  -X PATCH \
  -H 'Content-Type: application/merge-patch+json' \
  -d '{
  "spec": {
    "template": {
      "spec": {
        "containers": [
          {
            "name": "sleep",
            "image": "curlimages/curl",
            "command": ["/bin/sleep", "1d"]
          }
        ]
      }
    }
  }
}'
```

!!! info
    請注意 UPDATE 和 PATCH 是相當棘手的操作。第一個受到各種版本衝突的影響，第二個的行為因使用的補丁策略而異。

**DELETE**

最後但同樣重要的是 - 以下是如何刪除對象集合：

```bash
$ curl $KUBE_API/apis/apps/v1/namespaces/default/deployments \
  --cacert ~/.minikube/ca.crt \
  --cert ~/.minikube/profiles/minikube/client.crt \
  --key ~/.minikube/profiles/minikube/client.key \
  -X DELETE
```

以下是如何刪除單個對象：

```bash
$ curl $KUBE_API/apis/apps/v1/namespaces/default/deployments/sleep \
  --cacert ~/.minikube/ca.crt \
  --cert ~/.minikube/profiles/minikube/client.crt \
  --key ~/.minikube/profiles/minikube/client.key \
  -X DELETE
```

## How to call Kubernetes API using kubectl

上面使用證書和令牌來呼叫 Kubernetes API 很有趣。至少經歷一次是一個很好的練習，可以鞏固對客戶端和服務器移動部件的理解。但是，當你有一個可以使用的 `kubectl` 時，每天都這樣做可能會有點矯枉過正。

### Calling Kubernetes API using kubectl proxy

使用正確配置的 `kubectl` 工具，您可以使用 `kubectl proxy` 命令大大簡化 API 訪問。

`kubectl proxy` 命令在您的 `localhost` 和 `Kubernetes API 服務器`之間創建一個代理服務器（或應用程序級網關）。但它必須不止於此。不然怎麼會這麼方便？

代理 kubectl 從調用者那裡卸載了相互的客戶端-服務器身份驗證責任。由於調用者和代理之間的通信發生在本地主機上，因此它被認為是安全的。並且代理本身使用 kubeconfig 文件中選擇的當前上下文中的信息來處理客戶端-服務器身份驗證。

```bash
$ kubectl config current-context
minikube

$ kubectl proxy --port=8080 &
```

啟動代理服務器後，調用 Kubernetes API 服務器就變得簡單多了：

```bash
$ curl localhost:8080/apis/apps/v1/deployments
{
  "kind": "DeploymentList",
  "apiVersion": "apps/v1",
  "metadata": {
    "resourceVersion": "660883"
  },
  "items": [...]
}
```

## Calling Kubernetes API using kubectl raw mode


我最近學到的另一個很酷的技巧是一些 kubectl 命令支持的原始模式：

```bash
# Sends HTTP GET request
$ kubectl get --raw /api/v1/namespaces/default/pods

# Sends HTTP POST request
$ kubectl create --raw /api/v1/namespaces/default/pods -f file.yaml

# Sends HTTP PUT request
$ kubectl replace --raw /api/v1/namespaces/default/pods/mypod -f file.json

# Sends HTTP DELETE request
$ kubectl delete --raw /api/v1/namespaces/default/pods
```

kubectl 是一個非常先進的工具，即使是像 kubectl 這樣的簡單命令也有大量代碼。但是，當使用 --raw 標誌時，實現歸結為將唯一的參數轉換為 API 端點 URL 並調用原始 REST API 客戶端。

## Wrapping up

第一次訪問 Kubernetes API 的需求可能很可怕 - 有很多新概念，如資源、API 組、種類、對象、集群、上下文、證書，哦，天哪！但是一旦你在構建塊上分解它並通過執行一些瑣碎的任務（比如找出 API 服務器地址或使用 curl 調用一堆端點）獲得一些實踐經驗，你很快就會意識到這個想法動物園並不是真的一些新的東西——它只是多年來為我們服務的眾所周知的機制的組合 — REST 架構風格、TLS 證書、JWT 令牌、對象方案等。

所以，不要害怕並運行一些查詢！

!!! info
    迫不及待想嘗試從代碼中調用 API？在 GitHub 上查看我收集的 [Kubernetes 客戶端示例](https://github.com/iximiuz/client-go-examples) 😉

### Resources

- [Kubernetes Documentation - Access Clusters Using the Kubernetes API](https://kubernetes.io/docs/tasks/administer-cluster/access-cluster-api/)
- [Kubernetes Documentation - Configure Access to Multiple Clusters](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)
- [Kubernetes Documentation - Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
- [Kubernetes Documentation - Controlling Access to the Kubernetes API](https://kubernetes.io/docs/concepts/security/controlling-access/)
- [Kubernetes Documentation - Configure Service Accounts for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
- [Building stuff with the Kubernetes API — Exploring API objects](https://medium.com/programming-kubernetes/building-stuff-with-the-kubernetes-api-1-cc50a3642)

