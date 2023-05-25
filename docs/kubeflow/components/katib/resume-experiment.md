# 回復實驗

**如何重新啟動和修改正在運行的實驗**

本指南描述如何修改正在運行的實驗和重新啟動已完成的實驗。您將了解如何更改實驗執行過程並為 Katib 實驗使用各種恢復策略。

有關如何配置和運行實驗的詳細信息，請遵循[running an experiment guide](./experiment.md)。

## 修改正在運行的實驗

在實驗運行時，您可以更改 trial count 參數。例如，您可以減少併行訓練的超參數集的最大數量。

您只能更改 `parallelTrialCount`、`maxTrialCount` 和 `maxFailedTrialCount` 實驗參數。

使用 Kubernetes API 或 kubectl 就地更新資源來進行實驗更改。例如，運行：

```bash
kubectl edit experiment <experiment-name> -n <experiment-namespace>
```

進行適當的更改並保存。控制器自動處理新參數並進行必要的更改。

- 如果要增加或減少並行試驗執行，請修改 `parallelTrialCount`。控制器根據 `parallelTrialCount` 值相應地創建或刪除試驗。
- 如果要增加或減少最大試驗次數，請修改 `maxTrialCount`。 `maxTrialCount` 應該大於當前成功試驗的計數。您可以刪除 `maxTrialCount` 參數，如果您的實驗應該使用並行試驗的 `parallelTrialCount` 無限運行，直到實驗達到目標或 `maxFailedTrialCount`。
- 如果要增加或減少最大失敗試驗次數，請修改 `maxFailedTrialCount`。如果實驗不應達到 `Failed` 狀態，您可以刪除 `maxFailedTrialCount` 參數。

## 回復成功的實驗

僅當 Katib 實驗處於 **`Succeeded`** 狀態時才可重新啟動，因為已達到 `maxTrialCount`。要檢查當前實驗狀態運行：`kubectl get experiment <experiment-name> -n <experiment-namespace>`。

要重新開始實驗，您只能更改 `parallelTrialCount`、`maxTrialCount` 和 `maxFailedTrialCount`，如上所述。

要控制各種 resume 策略，您可以為實驗指定 `.spec.resumePolicy`。請參考 [`ResumePolicy 類型`](https://github.com/kubeflow/katib/blob/318f66890ebee00eba9893f7145d366795caa1d0/pkg/apis/controller/experiments/v1beta1/experiment_types.go#L60)。

```golang
// ResumePolicyType describes how the experiment should be resumed.
// Only one of the following resume policies may be specified.
// If none of the following policies is specified, the default one is LongRunning.
type ResumePolicyType string

const (
	// NeverResume indicates that experiment can't be resumed.
	NeverResume ResumePolicyType = "Never"
	// LongRunning indicates that experiment's suggestion resources
	// (deployment and service) are always running.
	LongRunning ResumePolicyType = "LongRunning"
	// FromVolume indicates that volume is attached to experiment's
	// suggestion. Suggestion data can be retained in the volume.
	// When experiment is succeeded suggestion deployment and service are deleted.
	FromVolume ResumePolicyType = "FromVolume"
)
```

### Resume policy: Never

如果您的實驗不應在任何時候恢復，請使用此策略。實驗結束後，建議的 Deployment 和 Service 被刪除，無法重啟實驗。在[概述指南](https://www.kubeflow.org/docs/components/katib/overview/#katib-concepts)中了解有關 Katib 概念的更多信息。

這是所有 Katib 實驗的默認策略。您可以省略該功能的 `.spec.resumePolicy` 參數。

### Resume policy: LongRunning

如果您打算重新啟動實驗，請使用此策略。實驗完成後，建議的 Deployment 和 Service 會保持運行。修改實驗的試驗計數參數以重新啟動實驗。

當您刪除實驗時，建議的 Deployment 和 Service 也會被刪除。

查看 [`long-running-resume.yaml`](https://github.com/kubeflow/katib/blob/master/examples/v1beta1/resume-experiment/long-running-resume.yaml#L19) 範例以獲取更多詳細信息。

### Resume policy: FromVolume

如果您打算重新啟動實驗，請使用此策略。在這種情況下，卷會附加到建議的 Deployment。

除了建議的部署和服務之外，Katib 控制器還創建 PersistentVolumeClaim (PVC)。

注意：您的 Kubernetes 集群必須具有用於動態卷配置的 `StorageClass`，以便為創建的 PVC 自動配置存儲。否則，您必須在 Katib 配置設置中定義建議的 PersistentVolume (PV) 規範，Katib 控制器將創建 PVC 和 PV。按照 Katib 配置指南設置建議的 volume 設定。

- PVC 在建議命名空間中以名稱部署：`<suggestion-name>-<suggestion-algorithm>`。
- PV 以名稱部署：`<suggestion-name>-<suggestion-algorithm>-<suggestion-namespace>`。

實驗完成後，建議的 Deployment 和 Service 被刪除。Suggestion 的數據被保留在 volume 中。當您重新啟動實驗時，將創建建議的 Deployment 和 Service，並且可以從 volume 中恢復建議統計信息。

刪除實驗時，建議的 Deployment、Service、PVC 和 PV 會自動刪除。

查看 [`from-volume-resume.yaml`](https://github.com/kubeflow/katib/blob/master/examples/v1beta1/resume-experiment/from-volume-resume.yaml#L19) 範例以獲取更多詳細信息。