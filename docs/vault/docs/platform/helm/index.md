# Helm Chart

原文: https://www.vaultproject.io/docs/platform/k8s/helm

Vault Helm chart 是在 Kubernetes 上安裝和配置 Vault 的推薦方式。除了運行 Vault 本身之外，Helm chart 是安裝和配置 Vault 以與其他服務集成的主要方法，例如 Consul for High Availability (HA) 部署。

此頁面假定您具有 Helm 的一般知識以及如何使用它。使用 Helm 安裝 Vault 需要使用 Kubernetes 集群正確安裝和配置 Helm。

## 使用 Helm Chart

必須在您的機器上安裝和配置 Helm。請參閱 [Helm 文檔](https://helm.sh/) 或通過 [Helm 安裝到 Minikube 的 Vault 教程](https://learn.hashicorp.com/tutorials/vault/kubernetes-minikube)。

要使用 Helm chart，請添加 Hashicorp helm 存儲庫並檢查您是否有權訪問該 chart：

```bash
$ helm repo add hashicorp https://helm.releases.hashicorp.com

"hashicorp" has been added to your repositories

$ helm search repo hashicorp/vault

NAME            CHART VERSION   APP VERSION DESCRIPTION
hashicorp/vault 0.20.1          1.10.3      Official HashiCorp Vault Chart
```

!!! warning
    重要提示：Helm chart 會持續被 Hashicorpt 開發更版中。請始終在安裝或升級之前使用 `--dry-run` 運行 Helm 以驗證更改。

Helm chart 用法示例：

安裝最新版本的 Vault Helm chart，其中 pod 以名稱 `vault` 為前綴。

```bash
$ helm install vault hashicorp/vault
```

安裝特定版本的 chart。

```bash
# List the available releases
$ helm search repo hashicorp/vault -l

NAME           	CHART VERSION	APP VERSION	DESCRIPTION                               
hashicorp/vault	0.20.1       	1.10.3     	Official HashiCorp Vault Chart            
hashicorp/vault	0.20.0       	1.10.3     	Official HashiCorp Vault Chart            
hashicorp/vault	0.19.0       	1.9.2      	Official HashiCorp Vault Chart            
hashicorp/vault	0.18.0       	1.9.0      	Official HashiCorp Vault Chart            
hashicorp/vault	0.17.1       	1.8.4      	Official HashiCorp Vault Chart            
hashicorp/vault	0.17.0       	1.8.4      	Official HashiCorp Vault Chart            
hashicorp/vault	0.16.1       	1.8.3      	Official HashiCorp Vault Chart            
hashicorp/vault	0.16.0       	1.8.2      	Official HashiCorp Vault Chart            
hashicorp/vault	0.15.0       	1.8.1      	Official HashiCorp Vault Chart            
hashicorp/vault	0.14.0       	1.8.0      	Official HashiCorp Vault Chart            
hashicorp/vault	0.13.0       	1.7.3      	Official HashiCorp Vault Chart            
hashicorp/vault	0.12.0       	1.7.2      	Official HashiCorp Vault Chart            
hashicorp/vault	0.11.0       	1.7.0      	Official HashiCorp Vault Chart            
hashicorp/vault	0.10.0       	1.7.0      	Official HashiCorp Vault Chart            
hashicorp/vault	0.9.1        	1.6.2      	Official HashiCorp Vault Chart            
hashicorp/vault	0.9.0        	1.6.1      	Official HashiCorp Vault Chart            
hashicorp/vault	0.8.0        	1.5.4      	Official HashiCorp Vault Chart            
hashicorp/vault	0.7.0        	1.5.2      	Official HashiCorp Vault Chart            
hashicorp/vault	0.6.0        	1.4.2      	Official HashiCorp Vault Chart            
hashicorp/vault	0.5.0        	           	Install and configure Vault on Kubernetes.
hashicorp/vault	0.4.0        	           	Install and configure Vault on Kubernetes.

# Install version 0.20.1
$ helm install vault hashicorp/vault --version 0.20.1
```

!!! info
    安全警告：默認情況下，chart 以 `standalong mode` 運行。此模式使用帶有文件存儲後端的單個 Vault 服務器。這是一種簡易地安裝，不適合生產環境的設置。強烈建議使用適當保護的 Kubernetes 集群，了解可用的[配置選項](https://www.vaultproject.io/docs/platform/k8s/helm/configuration)，並閱讀[生產部署清單](https://www.vaultproject.io/docs/platform/k8s/helm/run#architecture)。

`helm install` 命令接受參數以覆蓋事先在 Helm 中定義的默認配置值。

覆蓋 `server.dev.enabled` 配置值：

```bash
$ helm install vault hashicorp/vault \
    --set "server.dev.enabled=true"
```

在 `override-values.yml` 文件中的設定要覆蓋的配置值：

```yaml title="override-values.yml"
server:
  ha:
    enabled: true
    replicas: 5
```

使用 `override-values.yml` 文件中的設定來覆蓋默認配置值：

```bash
$ helm install vault hashicorp/vault \
    --values override-values.yml
```

### Dev mode

Helm chart 可以運行一個用來讓開發/驗證的 Vault 服務器。這將安裝一個使用 memory 作為後端存儲的 Vault 服務器。

!!! info
    Dev mode：這是學習和演示環境的理想選擇，但不推薦用於生產環境。

使用在 `Dev mode` 下安裝 Vault。

```bash
$ helm install vault hashicorp/vault \
    --set "server.dev.enabled=true"
```

### Standalone mode

Helm chart 默認以 `Standalone mode` 安裝 Vault。這將安裝一個使用檔案文件作為後端存儲的 Vault 服務器。

在`Standalone mode` 下安裝 Vault。HA

```bash
$ helm install vault hashicorp/vault
```

### HA mode

Helm chart 可以安裝一個高可用性 (HA) mode/ 的 Vault 服務器。這將安裝三個具有使用 Consul 存儲後端的 Vault 服務器。建議通過 [Consul Helm chart](https://github.com/hashicorp/consul-helm) 安裝 Consul。

在 `HA mode` 下安裝 Vault。

```bash
$ helm install vault hashicorp/vault \
    --set "server.ha.enabled=true"
```

請參閱[通過 Helm 將 Vault 安裝到 Minikube 教程](https://learn.hashicorp.com/vault/getting-started-k8s/minikube)，了解如何在 HA 模式下設置 Consul 和 Vault。

### External mode

Helm chart可以在 `External mode` 下運行。這不會安裝 Vault 服務器，而是依賴於一個外部 Vault 服務器。

在 `External mode` 下安裝 Vault。

```bash
$ helm install vault hashicorp/vault \
    --set "injector.externalVaultAddr=http://external-vault:8200"
```

請參閱[將 Kubernetes 集群與外部 Vault 集成](https://learn.hashicorp.com/vault/getting-started-k8s/external-vault)教程，了解如何在 Kubernetes 集群中使用外部 Vault。

## 啟動 Vault UI

Vault UI 已啟用，但出於安全原因未作為服務公開。 Vault UI 也可以通過端口轉發或通過 `ui` [配置值](https://www.vaultproject.io/docs/platform/k8s/helm/configuration/#ui)公開。

使用 port-forwarding 公開 Vault UI：

```bash
$ kubectl port-forward vault-0 8200:8200
```

## Initialize 與 `Unseal` Vault

在以 `standalone` 或 `ha` 模式安裝 Vault Helm chart後，需要[初始化](https://www.vaultproject.io/docs/commands/operator/init)其中一個 Vault 服務器。初始化會生成 [unseal](https://www.vaultproject.io/docs/concepts/seal#why) 所有 Vault 服務器所需的憑據。

### CLI initialize 與 unseal

查看當前命名空間中的所有 Vault pod：

```bash
$ kubectl get pods -l app.kubernetes.io/name=vault

NAME                                    READY   STATUS    RESTARTS   AGE
vault-0                                 0/1     Running   0          1m49s
vault-1                                 0/1     Running   0          1m49s
vault-2                                 0/1     Running   0          1m49s
```

使用默認的密鑰共享數量和默認的密鑰閾值初始化一個 Vault 服務器：

```bash
$ kubectl exec -ti vault-0 -- vault operator init

Unseal Key 1: MBFSDepD9E6whREc6Dj+k3pMaKJ6cCnCUWcySJQymObb
Unseal Key 2: zQj4v22k9ixegS+94HJwmIaWLBL3nZHe1i+b/wHz25fr
Unseal Key 3: 7dbPPeeGGW3SmeBFFo04peCKkXFuuyKc8b2DuntA4VU5
Unseal Key 4: tLt+ME7Z7hYUATfWnuQdfCEgnKA2L173dptAwfmenCdf
Unseal Key 5: vYt9bxLr0+OzJ8m7c7cNMFj7nvdLljj0xWRbpLezFAI9

Initial Root Token: s.zJNwZlRrqISjyBHFMiEca6GF
##...
```

!!! info
    記得先把這些解封的密鑰與 Root Token保存起來


輸出顯示密鑰共享和生成的初始根密鑰。

使用密鑰共享解封 Vault 服務器，直到滿足密鑰閾值：

```bash
## Unseal the first vault server until it reaches the key threshold
$ kubectl exec -ti vault-0 -- vault operator unseal # ... Unseal Key 1
$ kubectl exec -ti vault-0 -- vault operator unseal # ... Unseal Key 2
$ kubectl exec -ti vault-0 -- vault operator unseal # ... Unseal Key 3
```

對所有 Vault 服務器 pod 重複解封過程。當所有 Vault 服務器 pod 都打開時，它們會報告 READY 1/1。

```bash
$ kubectl get pods -l app.kubernetes.io/name=vault
NAME                                    READY   STATUS    RESTARTS   AGE
vault-0                                 1/1     Running   0          1m49s
vault-1                                 1/1     Running   0          1m49s
vault-2                                 1/1     Running   0          1m49s
```

## 探針 Probes

探針 Probes 對於在 Kubernetes 中檢測故障、重新調度和使用 pod 至關重要。 helm chart 提供可配置的就緒和活躍度探針，可以針對各種用例進行定制。

可以自定義 Vault 的 `/sys/health` 端點以更改運行狀況檢查的行為。例如，我們可以使用以下探針更改 Vault 就緒探測以顯示 Vault pod 已準備就緒，即使它們仍未初始化和密封狀態：

```yaml
server:
  readinessProbe:
    enabled: true
    path: '/v1/sys/health?standbyok=true&sealedcode=204&uninitcode=204'
```

使用這個定制的探針，一旦 pod 準備好進行其他設置，postStart 腳本就可以自動運行。


## 保護敏感的 Vault 配置資訊

Vault Helm 會在安裝 Vault 的過程中生成一佪 Vault 配置文件，並將該文件存儲在 **Kubernetes configmap** 中。在配置文件中可能包含了一些敏感數據配置資訊，並且一旦在 Kubernetes 中創建之後就不會被靜態加密。

以下示例展示瞭如何向 Vault Helm 添加額外的配置文件，以保護敏感配置不使用 Kubernetes 機密以靜態明文形式出現。

首先，使用啟動期間 Vault 加載的敏感設置創建部分 Vault 配置：

```hcl title="config.hcl"
storage "mysql" {
    username = "user1234"
    password = "secret123!"
    database = "vault"
}
```

接下來，創建一個包含此部分配置的 Kubernetes secret：

```bash
$ kubectl create secret generic vault-storage-config \
    --from-file=config.hcl
```

最後，將此 `secret` 掛載為一個額外的 volume，並在 Vault 啟動命令中添加一個額外的 `-config` 標誌：

```bash
$ helm install vault hashicorp/vault \
  --set='server.volumes[0].name=userconfig-vault-storage-config' \
  --set='server.volumes[0].secret.defaultMode=420' \
  --set='server.volumes[0].secret.secretName=vault-storage-config' \
  --set='server.volumeMounts[0].mountPath=/vault/userconfig/vault-storage-config' \
  --set='server.volumeMounts[0].name=userconfig-vault-storage-config' \
  --set='server.volumeMounts[0].readOnly=true' \
  --set='server.extraArgs=-config=/vault/userconfig/vault-storage-config/config.hcl'
```

