# MLflow Model Registry

MLflow 模型註冊元件是集中式模型儲存、一組 API 和 UI，用於協作管理 MLflow 模型的整個生命週期。它提供模型沿襲（MLflow 實驗和運行生成模型）、模型版本控制、模型別名、模型標記和註解。

## 概念

模型註冊(model registry)引入了一些描述和促進 MLflow 模型的整個生命週期的概念。

**Model**

MLflow 模型是根據使用模型的 `mlflow.<model_flavor>.log_model()` 方法來記錄 experiment 或 run 來建立起來的。記錄後，該模型就可以在模型註冊元件中進行註冊。

**Registered Model**

MLflow 模型可以在 Model Registry 中註冊。註冊的模型具有唯一的名稱，包含版本、別名、標籤和其他元資料。

**Model Version**

每個註冊的模型可以有一個或多個版本。將新模型新增至　Model Registry　時，它會新增為版本 1。模型版本具有標籤，可用於追蹤模型版本的屬性（例如 `pre_deploy_checks：PASSED`）

**Model Alias**

模型別名可讓您將可變的命名參考指派給已註冊模型的特定版本。透過為特定模型版本指派別名，您可以使用別名透過模型 URI 或模型登錄 API 引用該模型版本。例如，您可以建立一個名為 `Champion` 的別名，它指向名為 MyModel 的模型的版本 1。然後，您可以使用 URI `models:/MyModel@champion` 來引用 MyModel 的版本 1。

別名對於部署模型特別有用。例如，您可以將 `Champion` 別名分配給用於生產流量的模型版本，並在生產工作負載中定位該別名。然後，您可以透過將冠軍別名重新指派給不同的模型版本來更新服務生產流量的模型。

**Tags**

標籤是與註冊模型和模型版本關聯的鍵值對，可讓您按功能或狀態對它們進行標記和分類。例如，您可以將具有鍵 `task` 和值 `question-answering`（在 UI 中顯示為 `task:question-answering`）的標籤套用於問答任務的註冊模型。在模型版本級別，您可以使用 `validation_status:pending` 標記正在進行部署前驗證的版本，並使用 `validation_status:approved` 標記已清除部署的版本。

**Annotations and Descriptions**

您可以使用 Markdown 單獨註釋頂級模型和每個版本，包括描述和對團隊有用的任何相關信息，例如演算法描述、使用的資料集或方法。

