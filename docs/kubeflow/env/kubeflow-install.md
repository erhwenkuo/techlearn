# 手動安裝 Kubeflow

## 概述

Repo: https://github.com/kubeflow/manifests

Kubeflow Manifests 存儲庫組織在三個主要目錄下，其中包括用於安裝的清單：

| 目錄 | 目的 |
| --------- | ------- |
| `apps` | Kubeflow 的官方組件 |
| `common` | 共用服務組件|
| `contrib` | 第 3 方貢獻的應用程序 |

從 Kubeflow 1.3 開始，所有組件都應該只能使用 [kustomize](https://kustomize.io/) 部署。任何用於在清單之上部署的自動化工具都應該由發行版所有者在外部維護。

## Kubeflow 組件版本

該存儲庫會定期從各自的上游存儲庫同步所有官方 Kubeflow 組件。以下矩陣顯示了 Kubeflow 為每個組件包含的 git 版本：

| Component | Local Manifests Path | Upstream Revision |
| - | - | - |
| Training Operator | apps/training-operator/upstream | [v1.6.0-rc.0](https://github.com/kubeflow/training-operator/tree/v1.6.0-rc.0/manifests) |
| Notebook Controller | apps/jupyter/notebook-controller/upstream | [v1.6.0-rc.1](https://github.com/kubeflow/kubeflow/tree/v1.6.0-rc.1/components/notebook-controller/config) |
| Tensorboard Controller | apps/tensorboard/tensorboard-controller/upstream | [v1.6.0-rc.1](https://github.com/kubeflow/kubeflow/tree/v1.6.0-rc.1/components/tensorboard-controller/config) |
| Central Dashboard | apps/centraldashboard/upstream | [v1.6.0-rc.1](https://github.com/kubeflow/kubeflow/tree/v1.6.0-rc.1/components/centraldashboard/manifests) |
| Profiles + KFAM | apps/profiles/upstream | [v1.6.0-rc.1](https://github.com/kubeflow/kubeflow/tree/v1.6.0-rc.1/components/profile-controller/config) |
| PodDefaults Webhook | apps/admission-webhook/upstream | [v1.6.0-rc.1](https://github.com/kubeflow/kubeflow/tree/v1.6.0-rc.1/components/admission-webhook/manifests) |
| Jupyter Web App | apps/jupyter/jupyter-web-app/upstream | [v1.6.0-rc.1](https://github.com/kubeflow/kubeflow/tree/v1.6.0-rc.1/components/crud-web-apps/jupyter/manifests) |
| Tensorboards Web App | apps/tensorboard/tensorboards-web-app/upstream | [v1.6.0-rc.1](https://github.com/kubeflow/kubeflow/tree/v1.6.0-rc.1/components/crud-web-apps/tensorboards/manifests) |
| Volumes Web App | apps/volumes-web-app/upstream | [v1.6.0-rc.1](https://github.com/kubeflow/kubeflow/tree/v1.6.0-rc.1/components/crud-web-apps/volumes/manifests) |
| Katib | apps/katib/upstream | [v0.14.0-rc.0](https://github.com/kubeflow/katib/tree/v0.14.0-rc.0/manifests/v1beta1) |
| KServe | contrib/kserve/kserve | [release-0.8](https://github.com/kserve/kserve/tree/8079f375cbcedc4d45a1b4aade2e2308ea6f9ae8/install/v0.8.0) |
| KServe Models Web App | contrib/kserve/models-web-app | [v0.8.1](https://github.com/kserve/models-web-app/tree/v0.8.1/config) |
| Kubeflow Pipelines | apps/pipeline/upstream | [2.0.0-alpha.3](https://github.com/kubeflow/pipelines/tree/2.0.0-alpha.3/manifests/kustomize) |
| Kubeflow Tekton Pipelines | apps/kfp-tekton/upstream | [v1.5.1](https://github.com/kubeflow/kfp-tekton/tree/v1.5.1/manifests/kustomize) |

以下也是一個矩陣，其中包含來自常見組件的版本
從 Kubeflow 的不同項目中使用：

| Component | Local Manifests Path | Upstream Revision |
| - | - | - |
| Istio | common/istio-1-16 | [1.16.0](https://github.com/istio/istio/releases/tag/1.16.0) |
| Knative | common/knative/knative-serving <br /> common/knative/knative-eventing | [1.8.1](https://github.com/knative/serving/releases/tag/knative-v1.8.1) <br /> [1.8.1](https://github.com/knative/eventing/releases/tag/knative-v1.8.1) |
| Cert Manager | common/cert-manager | [1.10.1](https://github.com/cert-manager/cert-manager/releases/tag/v1.10.1) |

## 安裝

從 Kubeflow 1.3 開始，Manifests WG 提供了兩種安裝 Kubeflow 官方組件和使用 kustomize 的常用服務的選項。目的是幫助最終用戶輕鬆安裝 Kubeflow:

1. Single-command 安裝 `apps` 和 `common` 下的所有組件
2. Multi-command 安裝 `apps` 和 `common` 下的組件

選項 1 旨在簡化最終用戶的部署。
選項 2 的目標是自定義和挑選單個組件的能力。

示例目錄 `example` 包含使用單一個命令 (利用 kustomize) 能夠運行的 Kubelfow 的範例。

!!! warn
    在這兩個選項中，都使用了預設的電子郵件 `user@example.com` 和密碼 `12341234` 來作範例使用者帳密。對於任何生產環境的 Kubeflow 部署，您應該按照相關參考文件來更改預設的密碼。

### 先決條件

- 具有預設 `StorageClass` 的 Kubernetes（最高版本 **1.21**）
    - Kubeflow 1.5.0 與 Kubernetes 1.22 及更高版本不兼容。您可以在 [kubeflow/kubeflow#6353](https://github.com/kubeflow/kubeflow/issues/6353) 中跟踪 K8s 1.22 支持的剩餘工作
    - Kubeflow 1.6.0 與 Kubernetes 1.22

- kustomize (version 3.2.0)
    - https://github.com/kubernetes-sigs/kustomize/releases/tag/v3.2.0

- kubectl

### Single Command 安裝

您可以使用以下命令安裝所有 Kubeflow 官方組件(位於 `apps`)和所有公共服務(位於 `common`):

```bash
git clone https://github.com/kubeflow/manifests.git

cd manifests

while ! kustomize build example | kubectl apply -f -; do echo "Retrying to apply resources"; sleep 10; done
```

!!! tip
    安裝的時候可能會發現下列的錯誤訊息:

    ```
    error: resource mapping not found for name: "kubeflow-user-example-com" namespace: "" from "STDIN": no matches for kind "Profile" in version "kubeflow.org/v1beta1" ensure CRDs are installed first
    ```

    根據 [Profiles + KFAM](https://github.com/kubeflow/manifests#profiles--kfam) 來安裝下面的組件:

    - Profile Controller
    - Kubeflow Access-Management (KFAM)

    ```bash
    kustomize build apps/profiles/upstream/overlays/kubeflow | kubectl apply -f -
    ```

當所有的元件都安裝成功之後，您可以通過登錄到您的集群來訪問 Kubeflow Central Dashboard。

```bash
kubectl port-forward --address 0.0.0.0 svc/istio-ingressgateway -n istio-system 8080:80
```

輸入默認用戶名和密碼：

- user@example.com 
- 12341234

