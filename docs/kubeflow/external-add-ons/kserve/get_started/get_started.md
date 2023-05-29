# KServe 入門

## 開始之前

KServe 快速啟動環境僅供實驗使用。對於生產安裝，請參閱我們的[管理員指南](https://kserve.github.io/website/0.10/admin/serverless/serverless/)。

在開始使用 KServe Quickstart 部署之前，您必須安裝 kind 和 Kubernetes CLI。

## 安裝 Kind（Docker 中的 Kubernetes）

您可以使用 [kind](https://kind.sigs.k8s.io/docs/user/quick-start)（Docker 中的 Kubernetes）來運行具有 Docker 容器節點的本地 Kubernetes 集群。

## 安裝 K3D（Docker 中的 Kubernetes）

### 創建本機 Docker Network

使用 docker 創建一個虛擬的網路來做為本次教程的網路架構。

|   |   |
|--- |---|
|CIDR|172.22.0.0/24|
|CIDR IP Range|172.20.0.0 - 172.20.0.255|
|IPs|256|
|Subnet Mask|255.255.255.0|
|Gateay|172.20.0.1|

```bash
docker network create \
  --driver=bridge \
  --subnet=172.20.0.0/24 \
  --gateway=172.20.0.1 \
  lab-network
```

檢查 Docker 虛擬網絡 `lab-network` 的設定。

```bash
docker network inspect lab-network
```

結果:

```json hl_lines="14-15"
[
    {
        "Name": "lab-network",
        "Id": "2e2ca22fbb712cbc19d93acb16fc4e1715488c4c18b82d12dba4c1634ac5b1b6",
        "Created": "2023-02-09T23:07:33.186003336+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.20.0.0/24",
                    "Gateway": "172.20.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
```

讓我們從這個虛擬網段裡的 CIDR IP Range 中保留 5 個 IP (`172.20.0.10-172.20.0.15`) 來做本次的練習。

### 創建 K8S 集群

執行下列命令來創建實驗 Kubernetes 集群:

```bash title="執行下列命令  >_"
k3d cluster create  --api-port 6443 \
  --port 8080:80@loadbalancer --port 8443:443@loadbalancer \
  --k3s-arg "--disable=traefik@server:0" \
  --k3s-arg "--disable=servicelb@server:0" \
  --network lab-network
```

參數說明:

- `--k3s-arg "--disable=servicelb@server:0"` 不安裝 K3D 預設的 traefik (IngressController), 我們將使用　nginx ingress controller
- `--k3s-arg "--disable=traefik@server:0"` 不安裝 K3D 預設的 servicelb (klipper-lb), 我們將使用 metallb
- `--network lab-network` 使用預先創建的 docker 虛擬網段

### 安裝/設定 MetalLB

在本次的結構裡, 有兩個負責南北向 traffic 的亓件, 主要原因是 Nginx Ingress Controller 是很多團隊己經使用的 ingress 的元件, 雖說 Istio Ingress Gateway 也可負責對外開放在 Kubernetes 裡頭的服務, 但是對大多數的團隊來說這樣的轉換需要熟悉與過渡。因此在本次的 lab 架構會同時並存這兩個元件並且使用 metallb 來配置固定的 IP。

![](./assets/dual-north-south-traffic.png)

#### Helm 安裝

使用 Helm 的手法來進行 Ｍetallb 安裝:

```bash
#　setup helm repo
helm repo add metallb https://metallb.github.io/metallb

helm repo update

# install metallb to specific namespace
helm upgrade --install --create-namespace --namespace metallb-system \
  metallb metallb/metallb
```

!!! tips
    Ｍetallb 在[Version 0.13.2](https://metallb.universe.tf/release-notes/#version-0-13-2) 版本有一個很重大的修改:

    - 新功能:　支持CRD！期待已久的功能 MetalLB 現在可通過 CR 進行配置。
    - 行為變化:　最大的變化是引入了 CRD 並刪除了對通過 ConfigMap 進行配置的支持。

#### 設定 IP Adress Pool

我們將使用 MetalLB 的 Layer 2 模式是最簡單的配置：在大多數的情況下，你不需要任何特定於協議的配置，只需要 IP 地址範圍。

Layer 2 模式模式不需要將 IP 綁定到工作程序節點的網絡接口。它通過直接響應本地網絡上的 ARP 請求來工作，將機器的 MAC 地址提供給客戶端。

讓我們使用 CRD 來設定 Metallb:

```bash hl_lines="9"
cat <<EOF | kubectl apply -n metallb-system -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: ip-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.20.0.10-172.20.0.15
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2advertise
  namespace: metallb-system
spec:
  ipAddressPools:
  - ip-pool
EOF
```

結果:

```bash
ipaddresspool.metallb.io/ip-pool created
l2advertisement.metallb.io/l2advertise created
```

!!! tip
    如果只有一個 IP 要讓 Metallb 來給予，那麼 CIDR 的設定可設成 172.20.0.5/32 (也就是只有一個 IP: `172.20.0.5` 可被指派使用)

### 安裝/設定 Nginx Ingress Controller

#### Helm 安裝

使用以下命令添加 Nginx Ingress Controller 的 chart 存儲庫：

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

helm repo update
```

設定 `ingress-nginx` 要從 metallb 取得特定的預設 IP (`172.20.0.13`):

```yaml title="ingress-nginx-values.yaml"
controller:
  # add annotations to get ip from metallb
  service:
    annotations:
      metallb.universe.tf/address-pool: ip-pool
    loadBalancerIP: "172.20.0.13"
  # set ingressclass as default
  ingressClassResource:
    default: true
```

將 Nginx Ingress Controller 安裝到 kube-system 命名空間中：

```bash
helm upgrade --install \
     --create-namespace --namespace kube-system \
     ingress-nginx ingress-nginx/ingress-nginx \
     --values ingress-nginx-values.yaml
```

檢查:

```bash
kubectl get svc -n kube-system
```

結果:

```
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.43.160.250   172.20.0.13    80:30672/TCP,443:30990/TCP   91s
```

!!! tip
    特別注意 `ingress-nginx-controller` 的 EXTERNAL-IP 是否從 metallb 取得 `172.20.0.13`

#### 驗證 Ingress 設定

創建一個 Nginx 的 Deployment 與 Service:

```bash
kubectl create deployment nginx --image=nginx

kubectl create service clusterip nginx --tcp=80:80
```

創建 Ingress 來曝露這個測試的 Nginx 網站:

```bash
kubectl apply -f -<<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-nginx-svc
spec:
  rules:
  - host: "nginx.example.it"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: nginx
            port:
              number: 80
EOF
```

檢查看這個 ingress 是否有取得 IP ADDRESS:

```bash
kubectl get ing/ingress-nginx-svc
```

結果:

```
NAME                CLASS    HOSTS              ADDRESS       PORTS   AGE
ingress-nginx-svc   <none>   nginx.example.it   172.20.0.13   80      21s
```

修改 `/etc/hosts` 來增加一筆 entry 來模擬 DNS 解析:

``` title="/etc/hosts"
...
172.20.0.13  nginx.example.it
...
```

使用瀏覽器瀏覽 `http://nginx.example.it`:

![](./assets/ingress-test-nginx.png)

### 安裝 Kubernetes CLI

[Kubernetes CLI (kubectl)](https://kubernetes.io/docs/tasks/tools/install-kubectl) 允許您針對 Kubernetes 集群運行命令。您可以使用 kubectl 部署應用程序、檢查和管理集群資源以及查看日誌。

## 安裝 KServe “Quickstart” 環境

您可以使用 Kind 上的 KServe 快速安裝腳本開始本地部署 KServe：

```bash
curl -s "https://raw.githubusercontent.com/kserve/kserve/release-0.10/hack/quick_install.sh" | bash
```

預設的佈署模式是 `serverless`, 而且這個安裝腳本會安裝下列關鍵組件:

- Istio (v1.15.0)
- Knative (v1.7.0)
- KServe (v0.10.1)
- CertManager (v1.3.0)

??? info "quick_install.sh"

    ```bash
    set -e
    ############################################################
    # Help                                                     #
    ############################################################
    Help()
    {
      # Display Help
      echo "KServe quick install script."
      echo
      echo "Syntax: [-s|-r]"
      echo "options:"
      echo "s Serverless Mode."
      echo "r RawDeployment Mode."
      echo
    }

    deploymentMode=serverless
    while getopts ":hsr" option; do
      case $option in
          h) # display Help
            Help
            exit;;
          r) # skip knative install
      deploymentMode=kubernetes;;
          s) # install knative
            deploymentMode=serverless;;
        \?) # Invalid option
            echo "Error: Invalid option"
            exit;;
      esac
    done

    export ISTIO_VERSION=1.15.0
    export KNATIVE_VERSION=knative-v1.7.0
    export KSERVE_VERSION=v0.10.1
    export CERT_MANAGER_VERSION=v1.3.0
    export SCRIPT_DIR="$( dirname -- "${BASH_SOURCE[0]}" )"

    KUBE_VERSION=$(kubectl version --short=true | grep "Server Version" | awk -F '.' '{print $2}')
    if [ ${KUBE_VERSION} -lt 22 ];
    then
      echo "😱 install requires at least Kubernetes 1.22";
      exit 1;
    fi

    curl -L https://istio.io/downloadIstio | sh -
    cd istio-${ISTIO_VERSION}

    # Create istio-system namespace
    cat <<EOF | kubectl apply -f -
    apiVersion: v1
    kind: Namespace
    metadata:
      name: istio-system
      labels:
        istio-injection: disabled
    EOF

    cat << EOF > ./istio-minimal-operator.yaml
    apiVersion: install.istio.io/v1beta1
    kind: IstioOperator
    spec:
      values:
        global:
          proxy:
            autoInject: disabled
          useMCP: false

      meshConfig:
        accessLogFile: /dev/stdout

      components:
        ingressGateways:
          - name: istio-ingressgateway
            enabled: true
            k8s:
              podAnnotations:
                cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
        pilot:
          enabled: true
          k8s:
            resources:
              requests:
                cpu: 200m
                memory: 200Mi
            podAnnotations:
              cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
            env:
            - name: PILOT_ENABLE_CONFIG_DISTRIBUTION_TRACKING
              value: "false"
    EOF

    bin/istioctl manifest apply -f istio-minimal-operator.yaml -y;

    echo "😀 Successfully installed Istio"

    # Install Knative
    if [ $deploymentMode = serverless ]; then
      kubectl apply --filename https://github.com/knative/serving/releases/download/${KNATIVE_VERSION}/serving-crds.yaml
      kubectl apply --filename https://github.com/knative/serving/releases/download/${KNATIVE_VERSION}/serving-core.yaml
      kubectl apply --filename https://github.com/knative/net-istio/releases/download/${KNATIVE_VERSION}/release.yaml
      echo "😀 Successfully installed Knative"
    fi

    # Install Cert Manager
    kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/${CERT_MANAGER_VERSION}/cert-manager.yaml
    kubectl wait --for=condition=available --timeout=600s deployment/cert-manager-webhook -n cert-manager
    cd ..
    echo "😀 Successfully installed Cert Manager"

    # Install KServe
    KSERVE_CONFIG=kserve.yaml
    MAJOR_VERSION=$(echo ${KSERVE_VERSION:1} | cut -d "." -f1)
    MINOR_VERSION=$(echo ${KSERVE_VERSION} | cut -d "." -f2)
    if [ ${MAJOR_VERSION} -eq 0 ] && [ ${MINOR_VERSION} -le 6 ]; then KSERVE_CONFIG=kfserving.yaml; fi

    # Retry inorder to handle that it may take a minute or so for the TLS assets required for the webhook to function to be provisioned
    kubectl apply -f https://github.com/kserve/kserve/releases/download/${KSERVE_VERSION}/${KSERVE_CONFIG}

    # Install KServe built-in servingruntimes
    kubectl wait --for=condition=ready pod -l control-plane=kserve-controller-manager -n kserve --timeout=300s
    kubectl apply -f https://github.com/kserve/kserve/releases/download/${KSERVE_VERSION}/kserve-runtimes.yaml
    echo "😀 Successfully installed KServe"

    # Clean up
    rm -rf istio-${ISTIO_VERSION}
    ```