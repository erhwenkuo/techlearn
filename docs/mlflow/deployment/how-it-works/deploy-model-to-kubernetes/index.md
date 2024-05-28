# 將 MLflow 模型部署到 Kubernetes

## 使用 MLServer 作為推論伺服器

預設情況下，MLflow 部署使用 [Flask](https://flask.palletsprojects.com/en/1.1.x/)（廣泛使用的 Python WSGI Web 應用程式框架）來服務推論端點。然而，Flask 主要是為輕量級應用程式設計的，可能不適合大規模生產用例。為了彌補這一差距，MLflow 與 [MLServer](https://mlserver.readthedocs.io/en/latest/) 整合作為替代部署選項，後者在 [Seldon Core](https://docs.seldon.io/projects/seldon-core/en/latest/) 和 [KServe](https://kserve.github.io/website/) 等 Kubernetes 原生框架中用作核心 Python 推論伺服器。

使用 [MLServer](https://mlserver.readthedocs.io/en/latest/)，您可以利用 Kubernetes 的可擴充性和可靠性來大規模為您的模型提供服務。請參閱[Serving Framework](https://mlflow.org/docs/latest/deployment/deploy-model-locally.html#serving-frameworks)，以了解 Flask 和 MLServer 之間的詳細比較，以及為什麼 MLServer 是 ML 生產用例的更好選擇。

### Serving Frameworks 比較

預設情況下，MLflow 使用 **Flask**（一個用於 Python 的輕量級 WSGI Web 應用程式框架）來服務推論端點。然而，Flask 主要是為輕量級應用程式設計的，可能不適合大規模生產用例。為了解決這一差距，MLflow 與 **MLServer** 整合作為替代服務引擎。 MLServer 透過利用非同步請求/回應範例和工作負載卸載來實現更高的效能和可擴充性。此外，MLServer 也用作 Seldon Core 和 [KServe](https://kserve.github.io/website/) 等 Kubernetes 原生框架中的核心 Python 推論伺服器，因此它提供了金絲雀部署和開箱即用自動擴展等高級功能。

![](./assets/flask-logo.png) ![](./assets/seldon-mlserver-logo.png)

||Flask|MLServer|
|----|----|-----|
|**Use Case**|輕量級目的，包括本地測試。|高規模生產環境。|
|**Set Up**|Flask 預設隨 MLflow 一起安裝。|需要單獨安裝。|
|**Performance**|適合輕量級應用程序，但未針對高效能進行最佳化，如 WSGI 應用程式。 WSGI 是基於同步請求/回應範例，由於阻塞性質，這對於 ML 工作負載來說並不理想。機器學習預測通常涉及大量計算，並且可能需要很長時間才能完成，因此在處理請求時阻塞伺服器並不理想。雖然 Flask 可以透過 Uvicorn 等非同步框架進行增強，但 MLflow 並不支援它們開箱即用，而只是使用 Flask 的預設同步行為。|專為高性能機器學習工作負載而設計，通常可提供更好的吞吐量和效率。 MLServer 支援非同步請求/回應範例，透過將 ML 推理工作負載卸載到單獨的工作池（進程），以便伺服器可以在處理推理時繼續接受新請求。請參閱 MLServer Parallel Inference 以了解有關它們如何實現這一點的更多詳細資訊。此外，MLServer 支援自適應 Bacthing，可以透明地批量處理請求，以提高吞吐量和效率。|
|**Scalability**|出於與性能相同的原因，本質上不具有可擴展性。|除了上述的平行推理的支援之外，MLServer 還被用作 Kubernetes 原生框架（例如 Seldon Core 和 KServe（以前稱為 KFServing））中的核心推理伺服器。透過使用 MLServer 將 MLflow 模型部署到 Kubernetes，您可以利用這些框架的進階功能（例如自動縮放）來實現高可擴充性。

## 為 MLflow 模型建立 Docker 鏡像

將 MLflow 模型部署到 Kubernetes 的基本步驟是建立包含 MLflow 模型和推論伺服器的 Docker 鏡像。這可以透過 `build-docker` CLI 命令或 Python API 來完成。

=== "CLI"

    ```bash
    mlflow models build-docker -m runs:/<run_id>/model -n <image_name> --enable-mlserver
    ```

    如果您想使用基本的 Flask 伺服器而不是 MLServer，請刪除 `--enable-mlserver` 標誌。有關其他選項，請參閱 [build-docker](https://mlflow.org/docs/latest/cli.html#mlflow-models-build-docker) 命令文件。

=== "Python"

    ```python
    import mlflow

    mlflow.models.build_docker(
        model_uri=f"runs:/{run_id}/model",
        name="<image_name>",
        enable_mlserver=True,
    )
    ```

    如果您想使用基本的 Flask 伺服器而不是 MLServer，請刪除 `enable_mlserver=True`。有關其他選項，請參閱 [mlflow.models.build_docker](https://mlflow.org/docs/latest/python_api/mlflow.models.html#mlflow.models.build_docker) 函數文件。


## 部署步驟

請參閱以下合作夥伴文檔，以了解如何使用 MLServer 將 MLflow 模型部署到 Kubernetes。您也可以按照下面的教學來學習端到端的流程，包括環境設定、模型訓練和部署。

- [Deploy MLflow models with KServe InferenceService](https://kserve.github.io/website/latest/modelserving/v1beta1/mlflow/v2/)
- [Deploy MLflow models to Seldon Core](https://docs.seldon.io/projects/seldon-core/en/latest/servers/mlflow.html)

