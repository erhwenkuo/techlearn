# 模型推論 Runtime 簡介

KServe 提供了一個簡單的 Kubernetes CRD，可以將單個或多個經過訓練的模型部署到模型服務運行時，例如 [TFServing](https://www.tensorflow.org/tfx/guide/serving)、[TorchServe](https://pytorch.org/serve/server.html)、[Triton 推理服務器](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs)。

此外，[ModelServer](https://github.com/kserve/kserve/tree/master/python/kserve/kserve) 是使用預測 `v1` 協議在 KServe 本身中實現的 Python 模型服務運行時，[MLServer](https://github.com/SeldonIO/MLServer) 使用 REST 和 gRPC 實現預測 `v2` 協議。

這些模型推論 Runtime 能夠提供開箱即用的模型服務，但您也可以選擇為更複雜的用例構建自己的模型服務器。 KServe 提供基本 API 原語，讓您輕鬆構建自定義模型服務運行時，您可以使用其他工具（如 [BentoML](https://docs.bentoml.org/en/latest)）構建您的自定義模型服務鏡像。

使用 InferenceService 部署模型後，您將獲得 KServe 提供的以下所有 serverless 功能。

- Scale to and from Zero
- Request based Autoscaling on CPU/GPU
- Revision Management
- Optimized Container
- Batching
- Request/Response logging
- Traffic management
- Security with AuthN/AuthZ
- Distributed Tracing
- Out-of-the-box metrics
- Ingress/Egress control

下表標識了 KServe 支持的每個模型服務運行時。 HTTP 和 gRPC 列指示服務運行時支持的預測協議版本。 KServe 預測協議被標記為 `v1` 或 `v2`。一些服務運行時也支持他們自己的預測協議，這些用 * 表示。

默認服務運行時版本列定義了服務運行時的來源和版本——MLServer、KServe 或其自身。這些版本也可以在運行時 kustomization YAML 中找到。所有 KServe 本機模型服務運行時都使用當前的 KServe 發布版本 (v0.10)。

支持的框架版本列列出了支持的模型的主要版本。這些也可以在 `supportedModelFormats` 字段下的相應運行時 YAML 中找到。對於使用 KServe 服務運行時的模型框架，具體的默認版本可以在 `kserve/python` 中找到。在給定的服務運行時目錄中，`setup.py` 文件包含所使用的確切模型框架版本。例如，在 kserve/python/lgbserver 中，setup.py 文件將模型框架版本設置為 3.3.2，lightgbm == 3.3.2。

|Model Serving Runtime	|Exported model	|HTTP	|gRPC	|Default Serving Runtime Version	|Supported Framework (Major) Version(s)	|Examples|
|-----------------------|---------------|-----|-----|---------------------------------|---------------------------------------|--------|
|[Custom ModelServer](https://github.com/kserve/kserve/tree/master/python/kserve/kserve)	|--	|v1, v2	|v2	|--	|--	|[Custom Model](https://kserve.github.io/website/0.10/modelserving/v1beta1/serving_runtime/custom/custom_model)|
|[LightGBM MLServer](https://mlserver.readthedocs.io/en/latest/runtimes/lightgbm.html)	|[Saved LightGBM Model](https://lightgbm.readthedocs.io/en/latest/pythonapi/lightgbm.Booster.html#lightgbm.Booster.save_model)	|v2	|v2	|v1.0.0 (MLServer)	|3	|[LightGBM Iris V2](https://kserve.github.io/website/0.10/modelserving/v1beta1/serving_runtime/lightgbm)|
|[LightGBM ModelServer](https://github.com/kserve/kserve/tree/master/python/lgbserver)	|[Saved LightGBM Model](https://lightgbm.readthedocs.io/en/latest/pythonapi/lightgbm.Booster.html#lightgbm.Booster.save_model)	|v1	|--	|v0.10 (KServe)	|3	|[LightGBM Iris](https://kserve.github.io/website/0.10/modelserving/v1beta1/serving_runtime/lightgbm)|
|[MLFlow ModelServer](https://docs.seldon.io/projects/seldon-core/en/latest/servers/mlflow.html)	|[Saved MLFlow Model](https://www.mlflow.org/docs/latest/python_api/mlflow.sklearn.html#mlflow.sklearn.save_model)	|v2	|v2	|v1.0.0 (MLServer)	|1	|[MLFLow wine-classifier](https://kserve.github.io/website/0.10/modelserving/v1beta1/serving_runtime/mlflow)|
|[PMML ModelServer](https://github.com/kserve/kserve/tree/master/python/pmmlserver)	|[PMML](http://dmg.org/pmml/v4-4-1/GeneralStructure.html)	|v1	|--	|v0.10 (KServe)	|3, 4 (PMML4.4.1)	|[SKLearn PMML](https://kserve.github.io/website/0.10/modelserving/v1beta1/serving_runtime/pmml)|
|[SKLearn MLServer](https://github.com/SeldonIO/MLServer)	|[Pickled Model](https://scikit-learn.org/stable/modules/model_persistence.html)	|v2	|v2	|v1.0.0 (MLServer)	|1	|[SKLearn Iris V2](https://kserve.github.io/website/0.10/modelserving/v1beta1/serving_runtime/sklearn/v2)|
|[SKLearn ModelServer](https://github.com/kserve/kserve/tree/master/python/sklearnserver)	|[Pickled Model](https://scikit-learn.org/stable/modules/model_persistence.html)	|v1	|--	|v0.10 (KServe)	|1	|[SKLearn Iris](https://kserve.github.io/website/0.10/modelserving/v1beta1/serving_runtime/sklearn/v2)|
|[TFServing](https://www.tensorflow.org/tfx/guide/serving)	|[TensorFlow SavedModel](https://www.tensorflow.org/guide/saved_model)	|v1	|tensorflow	|2.6.2 (TFServing Versions)	|2	|[TensorFlow flower](https://kserve.github.io/website/0.10/modelserving/v1beta1/serving_runtime/tensorflow)|
|[TorchServe](https://pytorch.org/serve/server.html)	|[Eager Model/TorchScript](https://pytorch.org/docs/master/generated/torch.save.html)	|v1, v2, torchserve	|torchserve	0.7.0 (TorchServe)	|1	|[TorchServe mnist](https://kserve.github.io/website/0.10/modelserving/v1beta1/serving_runtime/torchserve)|
|[Triton Inference Server](https://github.com/triton-inference-server/server)	|[TensorFlow,TorchScript,ONNX](https://github.com/triton-inference-server/server/blob/r21.09/docs/model_repository.md)	|v2	|v2	|21.09-py3 (Triton)	|8 (TensoRT), 1, 2 (TensorFlow), 1 (PyTorch), 2 (Triton) Compatibility Matrix	|[Torchscript cifar](https://kserve.github.io/website/0.10/modelserving/v1beta1/serving_runtime/triton/torchscript)|
|[XGBoost MLServer](https://github.com/SeldonIO/MLServer)	|[Saved Model](https://xgboost.readthedocs.io/en/latest/tutorials/saving_model.html)	|v2	|v2	|v1.0.0 (MLServer)	|1	|[XGBoost Iris V2](https://kserve.github.io/website/0.10/modelserving/v1beta1/serving_runtime/xgboost)|
|[XGBoost ModelServer](https://github.com/kserve/kserve/tree/master/python/xgbserver)	|[Saved Model](https://xgboost.readthedocs.io/en/latest/tutorials/saving_model.html)	|v1	|--	|v0.10 (KServe)	|1	|[XGBoost Iris](https://kserve.github.io/website/0.10/modelserving/v1beta1/serving_runtime/xgboost)|

!!! info
    服務運行時版本的模型可以用 `InferenceService` yaml 上的 `runtimeVersion` 欄位來覆蓋，我們強烈建議為生產服務設置此欄位。

    ```yaml
    apiVersion: "serving.kserve.io/v1beta1"
    kind: "InferenceService"
    metadata:
      name: "torchscript-cifar"
    spec:
      predictor:
        triton:
          storageUri: "gs://kfserving-examples/models/torchscript"
          runtimeVersion: 21.08-py3
    ```