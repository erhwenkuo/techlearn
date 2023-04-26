# Multi-user 隔離

Kubeflow Pipelines 的多用戶隔離是 Kubeflow 整體多租戶功能的一部分。

## 資源如何隔離？

Kubeflow Pipelines 使用 Kubernetes 命名空間隔離資源，這些命名空間由 Kubeflow 的 **Profile** 來進行設定與管理。未經許可，其他用戶無法查看您的配置文件/命名空間中的資源，因為 Kubeflow Pipelines API 服務器拒絕對當前用戶無權訪問的命名空間的請求。

“Experiments” 直接屬於命名空間，運行和重複運行屬於其父實驗的命名空間。

“Pipeline Runs” 在用戶命名空間中執行，因此用戶可以利用 Kubernetes 命名空間隔離。例如，他們可以為不同命名空間中的其他服務配置不同的 secrets。

## 何時使用 UI

當您從 Kubeflow 儀表板訪問 Kubeflow Pipelines UI 時，它只會在您選擇的命名空間中顯示 “experiments”、“runs” 和 “recurring runs”。同樣，當您從 UI 創建資源時，它們也屬於您選擇的命名空間。

## 使用 SDK 時機

如何將 Pipelines SDK 連接到 Kubeflow Pipelines 將取決於您擁有的 Kubeflow 部署類型以及您從何處運行代碼。

- Full Kubeflow (from inside cluster)
- Full Kubeflow (from outside cluster)
- Standalone Kubeflow Pipelines (from inside cluster)
- Standalone Kubeflow Pipelines (from outside cluster)

以下 Python 代碼將從完整 Kubeflow 集群內的 Pod 創建實驗（和相關運行）。

```python
import kfp

# the namespace in which you deployed Kubeflow Pipelines
kubeflow_namespace = "kubeflow"

# the namespace of your pipelines user (where the pipeline will be executed)
user_namespace = "jane-doe"

# the KF_PIPELINES_SA_TOKEN_PATH environment variable is used when no `path` is set
# the default KF_PIPELINES_SA_TOKEN_PATH is /var/run/secrets/kubeflow/pipelines/token
credentials = kfp.auth.ServiceAccountTokenVolumeCredentials(path=None)

# create a client
client = kfp.Client(host=f"http://ml-pipeline-ui.{kubeflow_namespace}", credentials=credentials)

# create an experiment
client.create_experiment(name="<YOUR_EXPERIMENT_ID>", namespace=user_namespace)
print(client.list_experiments(namespace=user_namespace))

# create a pipeline run
client.run_pipeline(
    experiment_id="<YOUR_EXPERIMENT_ID>",  # the experiment determines the namespace
    job_name="<YOUR_RUN_NAME>",
    pipeline_id="<YOUR_PIPELINE_ID>"  # the pipeline definition to run
)
print(client.list_runs(experiment_id="<YOUR_EXPERIMENT_ID>"))
print(client.list_runs(namespace=user_namespace))
```

!!! tip
    要為 Pipelines SDK 命令設置默認命名空間，請使用 `kfp.Client().set_user_namespace()` 方法，此方法將您的用戶命名空間存儲在 `$HOME/.config/kfp/context.json` 的配置文件中。

    可以在 Kubeflow Pipelines SDK Reference 中找到 `kfp.Client()` 的[詳細文檔](https://kubeflow-pipelines.readthedocs.io/en/stable/source/kfp.client.html)。

## 使用 REST API 時機

調用 Kubeflow Pipelines REST API 時，實驗 API 需要命名空間參數。

命名空間由“資源引用”指定，其類型為 NAMESPACE 且 key.id 等於命名空間名稱。

以下代碼使用生成的 python API 客戶端創建實驗和管道運行。

```python
import kfp
from kfp_server_api import *

# the namespace in which you deployed Kubeflow Pipelines
kubeflow_namespace = "kubeflow"

# the namespace of your pipelines user (where the pipeline will be executed)
user_namespace = "jane-doe"

# the KF_PIPELINES_SA_TOKEN_PATH environment variable is used when no `path` is set
# the default KF_PIPELINES_SA_TOKEN_PATH is /var/run/secrets/kubeflow/pipelines/token
credentials = kfp.auth.ServiceAccountTokenVolumeCredentials(path=None)

# create a client
client = kfp.Client(host=f"http://ml-pipeline-ui.{kubeflow_namespace}", credentials=credentials)

# create an experiment
experiment: ApiExperiment = client._experiment_api.create_experiment(
    body=ApiExperiment(
        name="<YOUR_EXPERIMENT_ID>",
        resource_references=[
            ApiResourceReference(
                key=ApiResourceKey(
                    id=user_namespace,
                    type=ApiResourceType.NAMESPACE,
                ),
                relationship=ApiRelationship.OWNER,
            )
        ],
    )
)
print("-------- BEGIN: EXPERIMENT --------")
print(experiment)
print("-------- END: EXPERIMENT ----------")

# get the experiment by name (only necessary if you comment out the `create_experiment()` call)
# experiment: ApiExperiment = client.get_experiment(
#     experiment_name="<YOUR_EXPERIMENT_ID>",
#     namespace=user_namespace
# )

# create a pipeline run
run: ApiRunDetail = client._run_api.create_run(
    body=ApiRun(
        name="<YOUR_RUN_NAME>",
        pipeline_spec=ApiPipelineSpec(
            # replace <YOUR_PIPELINE_ID> with the UID of a pipeline definition you have previously uploaded
            pipeline_id="<YOUR_PIPELINE_ID>",
        ),
        resource_references=[ApiResourceReference(
            key=ApiResourceKey(
                id=experiment.id,
                type=ApiResourceType.EXPERIMENT,
            ),
            relationship=ApiRelationship.OWNER,
        )
        ],
    )
)
print("-------- BEGIN: RUN --------")
print(run)
print("-------- END: RUN ----------")

# view the pipeline run
runs: ApiListRunsResponse = client._run_api.list_runs(
    resource_reference_key_type=ApiResourceType.EXPERIMENT,
    resource_reference_key_id=experiment.id,
)
print("-------- BEGIN: RUNS --------")
print(runs)
print("-------- END: RUNS ----------")
```

## 當前限制

以下資源目前不支持隔離，在沒有訪問控制的情況下共享：

- 管道（Pipeline definitions）。
- 機器學習元數據 (MLMD) 中的工件、執行和其他元數據實體。
- 包含管道運行的輸入/輸出工件的 Minio 工件存儲。