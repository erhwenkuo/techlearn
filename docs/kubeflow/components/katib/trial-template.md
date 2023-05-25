# Trial Templates 概述

**如何在 Katib 中指定 trial template 參數並支持自定義資源 (CRD)**

本指南介紹如何在 Katib 中配置 trial template 參數和使用自定義 Kubernetes CRD。您將了解更改 trial template 規範、如何使用 Kubernetes ConfigMaps 存儲模板以及如何修改 Katib 控制器以在 Katib 實驗中支持您的 Kubernetes CRD。

Katib 在上游有這些 CRD 範例：

- [Kubernetes `Job`](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
- [Kubeflow `TFJob`](https://www.kubeflow.org/docs/components/training/tftraining/)
- [Kubeflow `PyTorchJob`](https://www.kubeflow.org/docs/components/training/pytorch/)
- [Kubeflow `MXJob`](https://www.kubeflow.org/docs/components/training/mxnet)
- [Kubeflow `XGBoostJob`](https://www.kubeflow.org/docs/components/training/xgboost)
- [Kubeflow `MPIJob`](https://www.kubeflow.org/docs/components/training/mpi)
- [Tekton `Pipelines`](https://github.com/kubeflow/katib/tree/master/examples/v1beta1/tekton)
- [Argo `Workflows`](https://github.com/kubeflow/katib/tree/master/examples/v1beta1/argo)

要使用您自己的 Kubernetes 資源，請按照[以下步驟](https://www.kubeflow.org/docs/components/katib/trial-template/#custom-resource)操作。

有關如何配置和運行實驗的詳細信息，請遵循[運行實驗指南](./experiment.md)。

## 使用 Trial Templates 提交實驗

要運行 Katib 實驗，您必須為運行實際訓練的 worker job 指定一個 trial template 。在概述指南中了解有關 [Katib 概念](./overview.md)的更多信息。

### 配置 Trial Templates

Trial Templates 規範位於 experiment 的 `.spec.trialTemplate` 下。有關 API 概述，請參閱 [TrialTemplate 類型](https://github.com/kubeflow/katib/blob/318f66890ebee00eba9893f7145d366795caa1d0/pkg/apis/controller/experiments/v1beta1/experiment_types.go#L216-L278)。

```golang
// TrialTemplate describes structure of trial template
type TrialTemplate struct {
	// Retain indicates that trial resources must be not cleanup
	Retain bool `json:"retain,omitempty"`

	// Source for trial template (unstructured structure or config map)
	TrialSource `json:",inline"`

	// List of parameters that are used in trial template
	TrialParameters []TrialParameterSpec `json:"trialParameters,omitempty"`

	// Labels that determines if pod needs to be injected by Katib sidecar container.
	// If PrimaryPodLabels is omitted, metrics collector wraps all Trial's pods.
	PrimaryPodLabels map[string]string `json:"primaryPodLabels,omitempty"`

	// Name of training container where actual model training is running
	PrimaryContainerName string `json:"primaryContainerName,omitempty"`

	// Condition when trial custom resource is succeeded.
	// Condition must be in GJSON format, ref https://github.com/tidwall/gjson.
	// For example for BatchJob: status.conditions.#(type=="Complete")#|#(status=="True")#
	SuccessCondition string `json:"successCondition,omitempty"`

	// Condition when trial custom resource is failed.
	// Condition must be in GJSON format, ref https://github.com/tidwall/gjson.
	// For example for BatchJob: status.conditions.#(type=="Failed")#|#(status=="True")#
	FailureCondition string `json:"failureCondition,omitempty"`
}

// TrialSource represent the source for trial template
// Only one source can be specified
type TrialSource struct {

	// TrialSpec represents trial template in unstructured format
	TrialSpec *unstructured.Unstructured `json:"trialSpec,omitempty"`

	// ConfigMap spec represents a reference to ConfigMap
	ConfigMap *ConfigMapSource `json:"configMap,omitempty"`
}

// ConfigMapSource references the config map where trial template is located
type ConfigMapSource struct {
	// Name of config map where trial template is located
	ConfigMapName string `json:"configMapName,omitempty"`

	// Namespace of config map where trial template is located
	ConfigMapNamespace string `json:"configMapNamespace,omitempty"`

	// Path in config map where trial template is located
	TemplatePath string `json:"templatePath,omitempty"`
}

// TrialParameterSpec describes parameters that must be replaced in trial template
type TrialParameterSpec struct {
	// Name of the parameter that must be replaced in trial template
	Name string `json:"name,omitempty"`

	// Description of the parameter
	Description string `json:"description,omitempty"`

	// Reference to the parameter in search space
	Reference string `json:"reference,omitempty"`
}
```

要定義實驗的試驗，您應該在 `.spec.trialTemplate` 中指定這些參數：

- `trialParameters`: 在實驗執行期間在試驗模板中使用的參數列表。

    !!! info
        注意：您的 trialTemplate 必須包含 `trialParameters` 中的每個參數。您可以在模板的任何字段中設置這些參數，但 `.metadata.name` 和 `.metadata.namespace` 除外。在下方查看如何在模板中使用試用 `metadata` 參數。例如，您的訓練容器可以接收超參數作為命令行或參數或作為環境變量。

    您的實驗 suggestion 會在運行 trial 之前生成 `trialParameters`。每個 `trialParameter` 具有以下結構：

    - `name`- 在您的模板中替換的參數名稱。
    - `description`- 參數的描述。
    - `reference`- 實驗建議返回的參數名稱。通常，對於超參數調整參數參考等於實驗搜索空間。例如，在 grid example search space 中，搜索空間具有三個參數（`lr`、`num-layers` 和 `optimizer`），`trialParameters` 包含參考中的每個參數。

- 您必須在 `trialSpec` 或 `configMap sources` 之一中定義實驗的 trial templates。

    要從 `trialParameters` 設置參數，您需要在模板中使用此表達式：`${trialParameters.<parameter-name>}`。 Katib 會自動將其替換為實驗建議中的適當值。

    例如，`--lr=${trialParameters.learningRate}` 是學習率參數。

    - `trialSpec`- 非結構化格式的實驗試驗模板。模板應該是有效的 YAML格式。範例: [grid example](https://github.com/kubeflow/katib/blob/master/examples/v1beta1/hp-tuning/grid.yaml#L49-L65)。

    - `configMap`- 實驗的試用模板所在的 Kubernetes ConfigMap 規範。此 ConfigMap 必須具有標籤 `katib.kubeflow.org/component: trial-templates` 並包含鍵值對，其中鍵：`<template-name>`，值：`<template-yaml>`。檢查帶有[試用模板的 ConfigMap 範例](https://github.com/kubeflow/katib/blob/master/manifests/v1beta1/components/controller/trial-templates.yaml)。

        configMap 應該包含有：

        1. `configMapName`- 帶有試用模板的 ConfigMap 名稱。
        2. `configMapNamespace`- 帶有試用模板的 ConfigMap 命名空間。
        3. `templatePath`- ConfigMap 到模板的數據路徑。

        檢查試用模板的 [ConfigMap source](https://github.com/kubeflow/katib/blob/master/examples/v1beta1/trial-template/trial-configmap-source.yaml#L50-L53) 範例。


下面的 `.spec.trialTemplate` 參數用於控制試驗行為。如果 parameter 有默認值，在實驗YAML中可以省略。

- `retain`- 表示 trial 完成後不清理 trial 的資源。使用 `retain: true` 參數檢查範例。默認值為 `false`
- `primaryPodLabels`- trial worker Pod 標籤。這些 Pod 由 Katib 指標收集器注入。

    Kubeflow `TFJob`、`PyTorchJob`、`MXJob` 和 `XGBoostJob` 的默認值為 `job-role: master`。

- `primaryContainerName`- 運行實際模型訓練的訓練容器名稱。 Katib 指標收集器包裝此容器以收集單個實驗優化步驟所需的指標。

- `successCondition`- trial worker 成功的對象狀態。此條件必須採用 [`GJSON`](https://github.com/tidwall/gjson) 格式。使用 [successCondition](https://github.com/kubeflow/katib/blob/master/examples/v1beta1/tekton/pipeline-run.yaml#L36) 檢查範例。

    Kubernetes Job 的默認值是 `status.conditions.#(type=="Complete")#|#(status=="True")#`

    Kubeflow TFJob、PyTorchJob、MXJob 和 XGBoostJob 的默認值為 `status.conditions。#(type=="Succeeded")#|#(status=="True")#`

    僅當您在 .spec.trialTemplate.trialSpec` 中指定模板時，`successCondition` 默認值才有效。對於 `configMap` 模板源，您必須手動設置 `successCondition `。

- `failureCondition`- trial worker 失敗的物件狀態。此條件必須採用 [`GJSON`](https://github.com/tidwall/gjson)。檢查帶有 [failureCondition 的範例](https://github.com/kubeflow/katib/blob/master/examples/v1beta1/tekton/pipeline-run.yaml#L37)。

    Kubernetes Job 的默認值是 `status.conditions.#(type=="Failed")#|#(status=="True")#`

    Kubeflow TFJob、PyTorchJob、MXJob 和 XGBoostJob 的默認值為 `status.conditions。#(type=="Failed")#|#(status=="True")#`

    只有在 `.spec.trialTemplate.trialSpec` 中指定模板時，`failureCondition` 默認值才有效。對於 `configMap` 模板源，您必須手動設置 `failureCondition`。

### 在模板中使用 trial metadata

您不能在試用模板中指定 `.metadata.name` 和 `.metadata.namespace`，但您可以在實驗運行期間獲取此數據。例如，如果您想將試驗的名稱附加到您的模型存儲中。

為此，請將 `.trialParameters[x].reference` 指向適當的元數據參數並在您的試用模板中使用 `.trialParameters[x].name`。

下表顯示了 `.trialParameters[x].reference` 值和 trial metadata 之間的聯繫。

|Reference	|Trial metadata|
|-----------|--------------|
|`${trialSpec.Name}`	|Trial name|
|`${trialSpec.Namespace}`	|Trial namespace|
|`${trialSpec.Kind}`	|Kubernetes resource kind for the trial's worker|
|`${trialSpec.APIVersion}`	|Kubernetes resource APIVersion for the trial's worker|
|`${trialSpec.Labels[custom-key]}`	|Trial's worker label with `custom-key` key|
|`${trialSpec.Annotations[custom-key]}`	|Trial's worker annotation with `custom-key` key|

查看[使用trial metadata](https://github.com/kubeflow/katib/blob/master/examples/v1beta1/trial-template/trial-metadata-substitution.yaml) 的範例。

## 使用自定義 Kubernetes 資源作為 Trial Templates

在 Katib 範例中，您可以找到以下 trial worker type：`Kubernetes Job`、`Kubeflow TFJob`、`Kubeflow PyTorchJob`、`Kubeflow MXJob`、`Kubeflow XGBoostJob`、`Kubeflow MPIJob`、`Tekton Pipelines` 和 `Argo Workflows`。

可以使用您自己的 Kubernetes CRD 或其他 Kubernetes 資源（例如 Kubernetes 部署）作為 trial worker，而無需修改 Katib 控制器源代碼和構建新鏡像。只要你的 CRD 創建了 Kubernetes Pod，允許在這些 Pod 上註入 sidecar 容器，並且有成功和失敗狀態，你就可以在 Katib 中使用它。

為此，您需要先修改 Katib 組件，然後再將其安裝到 Kubernetes 集群上。因此，您必須知道您的 CRD API 組和版本，以及 CRD 對象的種類。此外，您需要知道您的自定義對象創建了哪些資源。查看 Kubernetes 指南以了解有關 CRD 的更多信息。

按照以下兩個簡單步驟將您的自定義 CRD 集成到 Katib 中：

1. 使用新規則修改 Katib 控制器 ClusterRole 的規則，使 Katib 能夠訪問試驗創建的所有資源。要了解有關 ClusterRole 的更多信息，請查看 Kubernetes 指南。

    對於 Tekton Pipelines，trial 創建 Tekton PipelineRun，然後 Tekton PipelineRun 創建 Tekton TaskRun。因此，Katib 控制器 ClusterRole 應該有權訪問 pipelineruns 和 taskruns：

    ```yaml
    - apiGroups:
        - tekton.dev
      resources:
        - pipelineruns
        - taskruns
      verbs:
        - "get"
        - "list"
        - "watch"
        - "create"
        - "delete"
    ```

2. 使用 new flag 修改 Katib 控制器部署的參數：`--trial-resources=<object-kind>.<object-API-version>.<object-API-group>`。

    例如，要支持 Tekton `Pipelines`：

    ```bash
    - "--trial-resources=PipelineRun.v1beta1.tekton.dev"
    ```

完成這些更改後，按照[入門指南](https://www.kubeflow.org/docs/components/katib/hyperparameter/#installing-katib)中的描述部署 Katib 並等待創建 `katib-controller` Pod。您可以檢查來自 Katib 控制器的日誌以驗證您的資源集成：

```bash
$ kubectl logs $(kubectl get pods -n kubeflow -o name | grep katib-controller) -n kubeflow | grep '"CRD Kind":"PipelineRun"'

{"level":"info","ts":1628032648.6285546,"logger":"trial-controller","msg":"Job watch added successfully","CRD Group":"tekton.dev","CRD Version":"v1beta1","CRD Kind":"PipelineRun"}
```

如果您成功運行了上述步驟，您應該能夠在實驗的試用模板源規範中使用您的自定義對象 YAML。