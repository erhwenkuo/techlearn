# The Kubernetes API call inside the cluster

原文: https://medium.com/@pczarkowski/the-kubernetes-api-call-is-coming-from-inside-the-cluster-f1a115bd2066

有時從 Pod 內部訪問 Kubernetes API 很有用。 Kubernetes 試圖通過為每個 pod 掛載一個 serviceaccount secret 來簡化此操作。

為了探索這一點，我使用 minikube 創建了一個 Kubernetes 集群並啟動了一個交互式 alpine Linux pod：

```bash
$ minikube start
$ kubectl run -i --tty alpine --image=alpine --restart=Never -- sh
/ #
```

您會在路徑中找到幾個文件

`/var/run/secrets/kubernetes.io/serviceaccount` 可用於對 Kubernetes API 進行身份驗證（身份驗證令牌、ca 證書和命名空間）：

```bash
$ ls /var/run/secrets/kubernetes.io/serviceaccount

ca.crt     namespace  token

$ cat /var/run/secrets/kubernetes.io/serviceaccount/token 

eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZ...
...
l25cDJo2xgnz02W-FVznBJ46A/
```

然而，這些文件都沒有提供 Kubernetes API 的 URL，幸好這可以通過環境變量獲得：

```bash
$ env | grep KUBE

KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_SERVICE_HOST=10.96.0.1
```

結合使用這些，您將能夠驗證和訪問 Kubernetes API。不幸的是，alpine 附帶的 wget 無法訪問 TLS 端點，因此您需要先安裝 curl 或類似的東西才能訪問它：

```bash
$ apk update

fetch https://dl-cdn.alpinelinux.org/alpine/v3.16/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.16/community/x86_64/APKINDEX.tar.gz
v3.16.0-242-ge8a8292729 [https://dl-cdn.alpinelinux.org/alpine/v3.16/main]
v3.16.0-240-g6e6a2bc8ec [https://dl-cdn.alpinelinux.org/alpine/v3.16/community]
OK: 17029 distinct packages available

$ apk add curl

(1/5) Installing ca-certificates (20211220-r0)
(2/5) Installing brotli-libs (1.0.9-r6)
(3/5) Installing nghttp2-libs (1.47.0-r0)
(4/5) Installing libcurl (7.83.1-r2)
(5/5) Installing curl (7.83.1-r2)
Executing busybox-1.35.0-r13.trigger
Executing ca-certificates-20211220-r0.trigger
OK: 8 MiB in 19 packages
```

將 curl 安裝到我們的 alpine 容器中後，我們可以執行一些非常基本的 API 操作來查找信息。

```bash
$ K8S=https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT
$ TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
$ CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
$ NAMESPACE=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)

$ curl -H "Authorization: Bearer $TOKEN" --cacert $CACERT $K8S/healthz
```

kubectl get rolebinding,clusterrolebinding --all-namespaces -o jsonpath='{range .items[?(@.subjects[0].name=="default")]}[{.roleRef.kind},{.roleRef.name}]{end}'