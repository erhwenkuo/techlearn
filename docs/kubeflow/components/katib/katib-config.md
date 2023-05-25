# Katib Configuration 簡介

**如何更改 Katib 配置**

本指南描述了 [Katib config](https://github.com/kubeflow/katib/blob/master/manifests/v1beta1/components/controller/katib-config.yaml) — 包含以下信息的 Kubernetes 配置映射：

- 當前的 [metrics collectors](https://www.kubeflow.org/docs/components/katib/experiment/#metrics-collector) (`key = metrics-collector-sidecar`)。
- 當前的 [algorithms](https://www.kubeflow.org/docs/components/katib/experiment/#search-algorithms-in-detail) (suggestions) (`key = suggestion`)。
- 當前的 [early stopping algorithms](https://www.kubeflow.org/docs/components/katib/early-stopping/#early-stopping-algorithms-in-detail) (`key = early-stopping`)。

Katib Config Map 必須以 katib-config 名稱部署在 KATIB_CORE_NAMESPACE 命名空間中。當您提交實驗時，Katib 控制器會解析 Katib 配置。

即使在部署 Katib 之後，您也可以編輯此 Config Map。

如果您在 Kubeflow 命名空間中部署 Katib，請運行此命令來編輯您的 Katib 配置：

```bash
kubectl edit configMap katib-config -n kubeflow
```

## Metrics Collector Sidecar 配置

這些設置與 Katib 指標收集器相關，其中：

- `key`: **metrics-collector-sidecar**
- `value`: 每個指標收集器類型對應的 JSON 設置

File metrics collector 範例：

```yaml
metrics-collector-sidecar: |-
{
  "File": {
    "image": "docker.io/kubeflowkatib/file-metrics-collector",
    "imagePullPolicy": "Always",
    "resources": {
      "requests": {
        "memory": "200Mi",
        "cpu": "250m",
        "ephemeral-storage": "200Mi"
      },
      "limits": {
        "memory": "1Gi",
        "cpu": "500m",
        "ephemeral-storage": "2Gi"
      }
    },
    "waitAllProcesses": false
  },
  ...
}
```

除鏡像外，所有這些設置都可以省略。如果您不指定任何其他設置，則會自動設置默認值。

1. `image` - 文件指標收集器容器的 Docker 鏡像（必須指定）。
2. `imagePullPolicy` - 文件指標收集器容器的鏡像拉取策略。默認值為 “IfNotPresent”。
3. `resources` - 文件指標收集器容器的資源。在上面的範例中，您可以檢查如何設定限制和請求。目前，您只能指定內存、cpu 和臨時存儲資源。

    `requests` 的默認值為：

    - `memory = 10Mi`
    - `cpu = 50m`
    - `ephemeral-storage = 500Mi`

    `limits` 的默認值為：

    - `memory = 100Mi`
    - `cpu = 500m`
    - `ephemeral-storage = 5Gi`


    要從指標收集器的容器中刪除特定資源，請在您的 Katib 配置中的請求和限制中設置負值，如下所示：

    ```json
    "requests": {
      "cpu": "-1",
      "memory": "-1",
      "ephemeral-storage": "-1"
    },
    "limits": {
      "cpu": "-1",
      "memory": "-1",
      "ephemeral-storage": "-1"
    }
    ```

4. `waitAllProcesses` - 一個旗標，用於定義指標收集器是否應等到訓練容器中的所有進程完成後再開始收集指標。默認值是 `true`。

## Suggestion 配置

這些設置與 Katib suggestion 相關，其中：

- `key`: **suggestion**
- `value`: 每個算法名稱對應的 JSON 配置

如果你想使用新的算法，你需要更新 Katib 配置。例如，使用具有所有設置的隨機算法如下所示：

```yaml
suggestion: |-
{
  "random": {
    "image": "docker.io/kubeflowkatib/suggestion-hyperopt",
    "imagePullPolicy": "Always",
    "resources": {
      "requests": {
        "memory": "100Mi",
        "cpu": "100m",
        "ephemeral-storage": "100Mi"
      },
      "limits": {
        "memory": "500Mi",
        "cpu": "500m",
        "ephemeral-storage": "3Gi"
      }
    },
    "serviceAccountName": "random-sa"
  },
  ...
}
```

除鏡像外，所有這些設定都可以省略。如果您不指定任何其他設定，則會自動設定成默認值。

1. `image` - 建議容器的 Docker 鏡像，採用隨機算法（必須指定）。

    Image 範例: `docker.io/kubeflowkatib/<suggestion-name>`

    對於每個算法（suggestion），您可以在 Docker 鏡像中指定以下建議名稱之一：

|Suggestion name	|List of supported algorithms	|Description|
|-----------------|-----------------------------|-----------|
|`suggestion-hyperopt`	|`random`, `tpe`	|[Hyperopt](https://github.com/hyperopt/hyperopt) optimization framework|
|`suggestion-skopt`	|`bayesianoptimization`	|[Scikit-optimize](https://github.com/scikit-optimize/scikit-optimize) optimization framework|
|`suggestion-goptuna`	|`cmaes`, `random`, `tpe`, `sobol`	|[Goptuna](https://github.com/c-bata/goptuna) optimization framework|
|`suggestion-optuna`	|`multivariate-tpe`, `tpe`, `cmaes`, `random`, `grid`	|[Optuna](https://github.com/optuna/optuna) optimization framework|
|`suggestion-hyperband`	|`hyperband`	|[Katib Hyperband](https://github.com/kubeflow/katib/tree/master/pkg/suggestion/v1beta1/hyperband) implementation|
|`suggestion-pbt`	|`pbt`	|[Katib PBT](https://github.com/kubeflow/katib/tree/master/pkg/suggestion/v1beta1/pbt) implementation|
|`suggestion-enas`	|`enas`	|[Katib ENAS](https://github.com/kubeflow/katib/tree/master/pkg/suggestion/v1beta1/nas/enas) implementation|
|`suggestion-darts`	|`darts`	|[Katib DARTS](https://github.com/kubeflow/katib/tree/master/pkg/suggestion/v1beta1/nas/darts) implementation|

2. `imagePullPolicy` - 使用隨機算法的建議容器的鏡像拉取策略。。默認值為 “IfNotPresent”。
3. `resources` - 使用隨機算法的建議容器的資源。在上面的示例中，您可以檢查如何指定限制和請求。

    `requests` 的默認值為：

    - `memory = 10Mi`
    - `cpu = 50m`
    - `ephemeral-storage = 500Mi`

    `limits` 的默認值為：

    - `memory = 100Mi`
    - `cpu = 500m`
    - `ephemeral-storage = 5Gi`

    要從建議的容器中刪除特定資源，請在您的 Katib 配置中的請求和限制中設置負值，如下所示：

    ```json
    "requests": {
      "cpu": "-1",
      "memory": "-1",
      "ephemeral-storage": "-1"
    },
    "limits": {
      "cpu": "-1",
      "memory": "-1",
      "ephemeral-storage": "-1"
    }
    ```

4. `serviceAccountName` - 使用隨機算法的建議容器的服務帳戶。

    在上面的示例中，隨機算法為每個實驗的建議附加了 `random-sa` 服務帳戶，直到您從 Katib 配置中更改或刪除該服務帳戶。

    默認情況下，建議 Pod 沒有任何特定的服務帳戶，在這種情況下，Pod 使用默認服務帳戶。

    !!! tip
        注意：如果您想運行提前停止的實驗，建議的部署必須有權更新實驗的試用狀態。如果您沒有在 Katib 配置中指定服務帳戶，Katib 控制器會為建議創建所需的基於 Kubernetes 角色的訪問控制。

    如果您需要自己的服務帳戶來提供提前停止實驗的建議，則必須遵循以下規則：
    
    - 服務帳號名稱不能等於 `<experiment-name>-<experiment-algorithm>`
    - 服務帳戶必須具有足夠的權限才能更新實驗的試用狀態。

## Suggestion volume 配置

當您使用 `FromVolume` 恢復策略創建實驗時，您可以為實驗建議指定 PersistentVolume (PV) 和 PersistentVolumeClaim (PVC) 設置。在概述指南中了解有關 Katib 概念的更多信息。

如果 PV 設置為空，Katib 控制器只創建 PVC。如果你想使用默認的 volume 規格，你可以省略這些設置。

下列是 `random` 算法的示例：

```yaml
suggestion: |-
{
  "random": {
    "image": "docker.io/kubeflowkatib/suggestion-hyperopt",
    "volumeMountPath": "/opt/suggestion/data",
    "persistentVolumeClaimSpec": {
      "accessModes": [
        "ReadWriteMany"
      ],
      "resources": {
        "requests": {
          "storage": "3Gi"
        }
      },
      "storageClassName": "katib-suggestion"
    },
    "persistentVolumeSpec": {
      "accessModes": [
        "ReadWriteMany"
      ],
      "capacity": {
        "storage": "3Gi"
      },
      "hostPath": {
        "path": "/tmp/suggestion/unique/path"
      },
      "storageClassName": "katib-suggestion"
    },
    "persistentVolumeLabels": {
      "type": "local"
    }
  },
  ...
}
```

1. `volumeMountPath` - 使用隨機算法的建議容器的安裝路徑。

    默認值為 `/opt/katib/data`

2. `persistentVolumeClaimSpec` - Suggestion 的 PVC 規範。

    如果您未指定以下任何設置，則會設置默認值：

    - `persistentVolumeClaimSpec.accessModes[0]` - 默認值是 `ReadWriteOnce`
    - `persistentVolumeClaimSpec.resources.requests.storage` - 默認值是 `1Gi`

3. `persistentVolumeSpec` - Suggestion 的 PV 規範。

    PV `persistentVolumeReclaimPolicy` 始終等於 Delete，以便在刪除 Katib 實驗後正確刪除所有資源。要了解有關 PV 回收策略的更多信息，請查看 Kubernetes 文檔。

4. `persistentVolumeLabels` - Suggestion 的 PV 標籤。

## Early stopping 配置

這些設置與 Katib early stopping 有關，其中：

- `key`: **early-stopping**
- `value`: 每個提前停止算法名稱對應的 JSON 設置

如果你想使用新的提前停止算法，你需要更新 Katib 配置。例如，使用具有所有設置的 `meidanstop` 早停算法如下所示：

```yaml
early-stopping: |-
{
  "medianstop": {
    "image": "docker.io/kubeflowkatib/earlystopping-medianstop",
    "imagePullPolicy": "Always"
  },
  ...
}
```

除鏡像外，所有這些設置都可以省略。如果您不指定任何其他設置，則會自動設置默認值。

1. `image` - 用於早期停止容器的 Docker 鏡像，使用中值停止算法（必須指定）。

    鏡像示例：`docker.io/kubeflowkatib/<early-stopping-name>`

    對於每個提前停止算法，您可以在 Docker 鏡像中指定以下提前停止名稱之一：

    |Early stopping name	|Early stopping algorithm	|Description|
    |---------------------|-------------------------|-----------|
    |`earlystopping-medianstop`	|`medianstop`	|[Katib Median Stopping](https://github.com/kubeflow/katib/tree/master/pkg/earlystopping/v1beta1/medianstop) implementation|

2. `imagePullPolicy` - 用於早期停止容器的鏡像拉取策略，採用中值停止算法。

    默認值為 `IfNotPresent`