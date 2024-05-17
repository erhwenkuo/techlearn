# 管理 MLflow 模型中的相依性

一個 MLflow 模型可根據 model flavor 來推斷所需的依賴關係並自動記錄它們。但是，它還允許您定義額外的依賴項或 custom Python 程式碼，並提供在沙箱環境中驗證它們的工具。

MLflow 模型是一種標準格式，它將機器學習模型及其依賴項和其他元資料打包在一起。使用其依賴項建構模型可以實現跨各種平台和工具的可重複性和可移植性。

當您使用 MLflow Tracking API（例如 `mlflow.pytorch.log_model()`）建立 MLflow 模型時，MLflow 會自動推斷您正在使用的模型 flavor 所需的依賴項，並將它們記錄為模型元資料的一部分。然後，當您提供模型進行預測時，MLflow 會自動將相依性安裝到環境中。因此，您通常不需要擔心管理 MLflow 模型中的依賴關係。

但是，在某些情況下，您可能需要新增或修改一些依賴項。本文提供了 MLflow 如何管理依賴項的高級描述以及如何為您的用例自訂依賴項的指南。

!!! tip
    提高 MLflow 推理依賴準確性的一個技巧是在儲存模型時添加 `input_example`。這使得 MLflow 能夠在保存模型之前執行模型預測，從而捕獲預測期間使用的依賴關係。有關此參數的其他詳細用法，請參閱 [Model Input Example](https://mlflow.org/docs/latest/model/signatures.html#input-example)。

## MLflow 如何記錄模型依賴關係

MLflow 模型保存在指定目錄中，結構如下：

```bash
my_model/
├── MLmodel
├── model.pkl
├── conda.yaml
├── python_env.yaml
└── requirements.txt
```

模型依賴項由以下文件定義（對於其他文件，請參閱討論儲存格式部分中提供的指南）：

- `python_env.yaml` - 該檔案包含使用 virtualenv 恢復模型環境所需的資訊 (1) python 版本 (2) pip、setuptools 和 wheel 等建置工具 (3) 模型的 pip requirements（對requirements.txt 的引用）
- `requirements.txt` - 定義運行模型所需的 pip 相依性集。
- `conda.yaml` - 定義運行模型所需的 conda 環境。當您指定 conda 作為用於復原模型環境的環境管理器時，將使用此選項。

請注意，**不建議手動編輯這些檔案** 以新增或刪除依賴項。它們由 MLflow 自動生成，當您再次儲存模型時，您手動所做的任何更改都將被覆蓋。相反，您應該使用以下各節中描述的建議方法之一。

### Example

以下顯示了使用 `mlflow.sklearn.log_model()` 記錄模型時 MLflow 產生的環境檔案範例：

- `python_env.yaml`

    ```bash
    python: 3.9.8
    build_dependencies:
    - pip==23.3.2
    - setuptools==69.0.3
    - wheel==0.42.0
    dependencies:
    - -r requirements.txt
    ```

- `requirements.txt`

    ```bash
    mlflow==2.9.2
    scikit-learn==1.3.2
    cloudpickle==3.0.0
    ```

- `conda.yaml`

    ```bash
    name: mlflow-env
    channels:
    - conda-forge
    dependencies:
    - python=3.9.8
    - pip
    - pip:
    - mlflow==2.9.2
    - scikit-learn==1.3.2
    - cloudpickle==3.0.0
    ```

## 在 MLflow 模型中新增額外的依賴項

MLflow 推斷模型 flavor 所需的依賴關係，但您的模型可能依賴其他函式庫，例如資料預處理。在這種情況下，您可以透過在記錄模型時指定 **extra_pip_requirements** 參數來為模型新增額外的依賴項。例如，

```python hl_lines="18"
import mlflow


class CustomModel(mlflow.pyfunc.PythonModel):
    def predict(self, context, model_input):
        # your model depends on pandas
        import pandas as pd

        ...
        return prediction


# 記錄模型
with mlflow.start_run() as run:
    mlflow.pyfunc.log_model(
        python_model=CustomModel(),
        artifact_path="model",
        extra_pip_requirements=["pandas==2.0.3"], # 為模型新增額外的依賴項
        input_example=input_data,
    )
```

額外的依賴項將會加入 `requirements.txt` 中，如下所示（與 `conda.yaml` 類似）：

```bash
mlflow==2.9.2
cloudpickle==3.0.0
pandas==2.0.3  # 被自動加進來
```

在這種情況下，MLflow 在為模型提供預測服務時，除了推斷的依賴項之外，還將安裝 Pandas 2.0.3。

!!! notes
    一旦您在記錄時模型宣告了額外的依賴項，建議在沙箱環境中進行測試，以避免將模型部署到生產時出現任何依賴項問題。從 MLflow 2.10.0 開始，您可以使用 `mlflow.models.predict()` API 在虛擬環境中快速測試模型。有關更多詳細信息，請參閱 [Validating Environment for Prediction](https://mlflow.org/docs/latest/model/dependencies.html#validating-environment-for-prediction)。

## 自己定義所有依賴項

或者，您也可以從頭開始定義所有依賴項，而不是新增額外的依賴項。為此，請在記錄模型時指定 **pip_requirements**。例如，

```python hl_lines="8-12"
import mlflow

# 記錄模型
with mlflow.start_run() as run:
    mlflow.sklearn.log_model(
        sk_model=model,
        artifact_path="model",
        pip_requirements=[
            "mlflow-skinny==2.9.2",
            "cloudpickle==2.5.8",
            "scikit-learn==1.3.1",
        ],
    )
```

手動定義的依賴項將覆蓋 MLflow 從模型 flavor 中檢測到的預設依賴項：

```bash
mlflow-skinny==2.9.2
cloudpickle==2.5.8
scikit-learn==1.3.1
```

!!! warning
    聲明與訓練期間使用的依賴項不同的依賴項時請小心，因為它可能很危險且容易出現意外行為。確保一致性最安全的方法是依賴 MLflow 推斷的預設依賴關係。

## 儲存額外程式碼

MLflow 也支援將自訂 Python 程式碼儲存為模型的依賴項。當您想要部署模型預測所需的自訂模組時，這特別有用。為此，請在記錄模型時指定 **code_paths**。例如，如果您的專案中有以下文件結構：

```bash
my_project/
├── utils.py
└── train.py
```

```python title="train.py"
import mlflow


class MyModel(mlflow.pyfunc.PythonModel):
    def predict(self, context, model_input):
        from utils import my_func

        x = my_func(model_input)
        # .. your prediction logic
        return prediction


# Log the model
with mlflow.start_run() as run:
    mlflow.pyfunc.log_model(
        python_model=MyModel(),
        artifact_path="model",
        input_example=input_data,
        code_paths=["utils.py"], # 宣告要儲存額外程式碼
    )
```

然後 MLflow 會將 `utils.py` 保存在模型目錄中的 `code/` 目錄下：

```bash
model/
├── MLmodel
├── ...
└── code/
    └── utils.py
```

當 MLflow 載入模型進行服務時，`code/` 目錄將會新增到系統路徑中，以便您可以在模型程式碼中使用該模組，例如 `from utils import my_func`。也可以指定一個目錄路徑為**code_paths**，以在該目錄下儲存多個檔案：

### 使用 `code_paths` 注意事項

使用 `code_paths` 選項時，請注意指定的檔案或目錄 {==必須與模型腳本位於同一目錄中的限制==}。如果指定的檔案或目錄位於父目錄或子目錄（例如 `my_project/src/utils.py`）中，則模型服務將失敗並出現 ModuleNotFoundError。例如，假設您的專案中有以下文件結構:

```bash
my_project/
|── train.py
└── src/
    └──  utils.py
```

那麼下面的模型程式碼就不起作用了：

```python
class MyModel(mlflow.pyfunc.PythonModel):
    def predict(self, context, model_input):
        from src.utils import my_func

        # .. your prediction logic
        return prediction


with mlflow.start_run() as run:
    mlflow.pyfunc.log_model(
        python_model=MyModel(),
        artifact_path="model",
        input_example=input_data,
        code_paths=[
            "src/utils.py"
        ],  # the file will be saved at code/utils.py not code/src/utils.py
    )

# => Model serving will fail with ModuleNotFoundError: No module named 'src'
```

此限制是由於 MLflow 保存和載入指定檔案和目錄的方式造成的。當它複製 `code/` 中的指定檔案或目錄時，它不會保留它們最初所在的相對路徑。例如，在上面的範例中，MLflow 會將 `utils.py` 複製到 `code/utils.py`，而不是 `code/src/utils.py`。因此，必須將其導入為 `from utils import my_func`，而不是 `from src.utils import my_func`。然而，這可能並不令人愉快，因為導入路徑與原始訓練腳本不同。

若要解決此問題，**code_paths** 應指定父目錄，在此範例中為 `code_paths=["src"]`。這樣，MLflow 將複製 `code/` 下的整個 `src/` 目錄，並且您的模型程式碼將能夠匯入 `src.utils`。

```python hl_lines="14"
class MyModel(mlflow.pyfunc.PythonModel):
    def predict(self, context, model_input):
        from src.utils import my_func

        # .. your prediction logic
        return prediction


with mlflow.start_run() as run:
    mlflow.pyfunc.log_model(
        python_model=model,
        artifact_path="model",
        input_example=input_data,
        code_paths=["src"],  # the whole /src directory will be saved at code/src
    )
```

!!! warning
    基於相同的原因，**code_paths** 選項不處理 `code_paths=["../src"]` 的相對導入。

## 推薦專案結構

考慮到 **code_paths** 的這項限制，建議的專案結構如下所示：

```bash
my_project/
|-- model.py # Defines the custom pyfunc model
|── train.py # Trains and logs the model
|── core/    # Required modules for prediction
|   |── preprocessing.py
|   └── ...
└── helper/  # Other helper modules used for training, evaluation
    |── evaluation.py
    └── ...
```

這樣，您可以使用 `code_paths=["core"]` 來記錄模型，以包含預測所需的模組，同時排除僅用於開發的幫助程式模組。

## 驗證推論運行環境

在部署之前驗證模型是確保生產準備就緒的關鍵步驟。 MLflow 提供了幾種在虛擬環境或 Docker 容器中本地測試模型的方法。如果您在驗證過程中發現任何依賴關係問題，請按照如何在提供模型時修復依賴關係錯誤？

### 使用虛擬環境測試離線預測

您可以透過 Python 或 CLI 使用 MLflow Models **predict** API 對您的模型進行測試推論。這將從模型 URI 載入模型，建立具有模型依賴項（在 MLflow Model 中定義）的虛擬環境，並使用模型執行離線預測。請參閱 [mlflow.models.predict()](https://mlflow.org/docs/latest/python_api/mlflow.models.html#mlflow.models.predict) 或 [CLI 參考](https://mlflow.org/docs/latest/cli.html#mlflow-models)，以了解 **predict** API 的更詳細用法。

=== "Python"

    ```python
    import mlflow

    mlflow.models.predict(
        model_uri="runs:/<run_id>/model",
        input_data=<input_data>,
    )
    ```

=== "Bash"

    ```bash
    mlflow models predict -m runs:/<run_id>/model-i <input_path>
    ```

使用 `mlflow.models.predict()` API 可以輕鬆快速測試模型和推論環境。然而，它可能不是服務的完美模擬，因為它沒有啟動線上推理伺服器。也就是說，這是測試預測輸入格式是否正確的好方法。

格式取決於記錄模型的 Predict() 方法支援的類型。如果使用簽章記錄模型，則應可從 MLflow UI 或透過具有欄位簽署的 `mlflow.models.get_model_info()` 查看輸入資料。

更一般地說，MLflow 能夠支援各種特定 flavor 的輸入類型，例如 TensorFlow 張量。 MLflow 也支援不特定於給定 flavor 的類型，例如 pandas DataFrame、numpy ndarray、python Dict、python List、scipy.sparse 矩陣和 Spark dataframe。

## 使用虛擬環境測試線上推論端點

如果您想透過實際執行線上推論伺服器來測試您的模型，可以使用 MLflow **serve** API。這將使用您的模型和相依性建立一個虛擬環境，類似於 **predict** API，但將啟動推論伺服器並公開 REST 端點。然後您可以發送測試請求並驗證回應。有關 **serve** API 的更詳細用法，請參閱 [CLI 參考](https://mlflow.org/docs/latest/cli.html#mlflow-models)。

```bash
mlflow models serve -m runs:/<run_id>/model -p <port>

# In another terminal
curl -X POST -H "Content-Type: application/json" \
    --data '{"inputs": [[1, 2], [3, 4]]}' \
    http://localhost:<port>/invocations
```

雖然這是在部署之前測試模型的一種可靠方法，但需要注意的是，虛擬環境無法吸收電腦與生產環境之間的作業系統層級差異。例如，如果您使用 MacOS 作為本機開發計算機，但部署目標在 Linux 上運行，則可能會遇到一些在虛擬環境中無法重現的問題。

在這種情況下，您可以使用 Docker 容器來測試您的模型。雖然它不提供與虛擬機器不同的完整作業系統級隔離，例如我們無法在 Linux 機器上執行 Windows 容器，Docker 涵蓋了一些流行的測試場景，例如運行不同版本的 Linux 或在 Mac 或 Windows 上模擬 Linux 環境。

## 使用 Docker 容器測試線上推論端點

適用於 CLI 和 Python 的 MLflow **build-docker** API 能夠建立基於 Ubuntu 的 Docker 映像來為您的模型提供服務。該映像將包含您的模型和依賴項，以及用於啟動推理伺服器的入口點。與服務 API 類似，您可以發送測試請求並驗證回應。有關 **build-docker** API 的更詳細用法，請參閱 [CLI 參考](https://mlflow.org/docs/latest/cli.html#mlflow-models)。

```bash
mlflow models build-docker -m runs:/<run_id>/model -n <image_name>

docker run -p <port>:8080 <image_name>

# In another terminal
curl -X POST -H "Content-Type: application/json" \
    --data '{"inputs": [[1, 2], [3, 4]]}' \
    http://localhost:<port>/invocations
```

## 故障排除

### 如何修復為模型提供服務時的依賴錯誤

模型部署過程中遇到的最常見問題之一是依賴問題。記錄或儲存模型時，MLflow 會嘗試推斷模型依賴性並將其儲存為 MLflow 模型元資料的一部分。然而，這可能並不總是完整的，並且會遺漏一些依賴項，例如[extras] 某些函式庫的依賴項。這可能會在提供模型時導致錯誤，例如 “ModuleNotFoundError” 或 “ImportError”。以下是一些有助於診斷和修復缺失依賴項錯誤的步驟。

!!! tip
    為了減少依賴錯誤的可能性，您可以在儲存模型時新增 **input_example**。這使得 MLflow 能夠在保存模型之前執行模型預測，從而捕獲預測期間使用的依賴關係。有關此參數的其他詳細用法，請參閱 [Model Input Example](https://mlflow.org/docs/latest/model/signatures.html#input-example)。

**1. 檢查缺少的依賴項**

錯誤訊息中列出了缺少的依賴項。例如，如果您看到以下錯誤訊息：

```bash
ModuleNotFoundError: No module named 'cv2'
```

**2. 嘗試使用 predict API 新增依賴項**

現在您知道了缺少的依賴項，您可以建立具有正確依賴項的新模型版本。然而，建立一個新模型來嘗試新的依賴關係可能有點乏味，特別是因為您可能需要多次迭代才能找到正確的解決方案。相反，您可以使用 `mlflow.models.predict()` API 來測試您的更改，而在排除安裝錯誤時實際上不需要重複重新記錄模型。

為此，請使用 **pip-requirements-override** 選項指定 pip 依賴項，例如 opencv-python==4.8.0。

=== "Python"

    ```python
    import mlflow

    mlflow.models.predict(
        model_uri="runs:/<run_id>/<model_path>",
        input_data=<input_data>,
        pip_requirements_override=["opencv-python==4.8.0"],
    )
    ```

=== "Bash"

    ```bash
    mlflow models predict \
        -m runs:/<run_id>/<model_path> \
        -I <input_path> \
        --pip-requirements-override opencv-python==4.8.0
    ```

除了（或代替）模型元資料中定義的依賴項之外，指定的依賴項也會安裝到虛擬環境中。由於這不會改變模型，因此您可以快速、安全地迭代以找到正確的依賴項。

請注意，對於 python 實作中的 **input_data** 參數，函數採用模型的 `Predict()` 函數支援的 Python 物件。一些範例可能包括特定 flavor 的輸入類型，例如 tensorflow 張量，或更通用的類型，例如 pandas DataFrame、numpy ndarray、python Dict 或 python List。使用 CLI 時，我們無法傳遞 python 對象，而是希望傳遞包含輸入負載的 CSV 或 JSON 檔案的路徑。

**3. 更新模型元數據**

一旦找到正確的依賴關係，您就可以建立具有正確依賴關係的新模型。為此，請在記錄模型時指定 **extra_pip_requirements** 選項。

```python hl_lines="6"
import mlflow

mlflow.pyfunc.log_model(
    artifact_path="model",
    python_model=python_model,
    extra_pip_requirements=["opencv-python==4.8.0"],
    input_example=input_data,
)
```

請注意，您還可以利用 CLI 就地更新模型依賴項，從而避免重新記錄模型。

```bash
mlflow models update-pip-requirements -m runs:/<run_id>/<model_path> add "opencv-python==4.8.0"
```

### 如何遷移 Anaconda 依賴項以進行許可證更改

Anaconda Inc. 更新了 anaconda.org 頻道的服務條款。根據新的服務條款，如果您依賴 Anaconda 的打包和分發，您可能需要商業許可。有關更多信息，請參閱 Anaconda 商業版常見問題解答。您對任何 Anaconda 管道的使用均受其服務條款的約束。

v1.18 先前記錄的 MLflow 模型預設使用 conda **defaults** 通道 (https://repo.anaconda.com/pkgs/) 作為依賴項進行記錄。由於此許可證更改，MLflow 已停止使用使用 MLflow v1.18 及更高版本記錄的模型的 **defaults** 通道。現在記錄的預設通道是 **conda-forge**，它指向社群管理的 https://conda-forge.org/。

如果您在 MLflow v1.18 之前記錄了模型，但沒有從模型的 conda 環境中排除預設通道，則該模型可能會依賴您可能不想要的預設通道。若要手動確認模型是否具有此依賴性，您可以檢查與記錄的模型一起打包的 `conda.yaml` 檔案中的通道值。例如，具有 defaults 通道相依性的模型的 `conda.yaml` 可能如下所示：

```yaml title="conda.yaml"
name: mlflow-env
channels:
- defaults
dependencies:
- python=3.8.8
- pip
- pip:
    - mlflow==2.3
    - scikit-learn==0.23.2
    - cloudpickle==1.6.0
```

如果您想要變更模型環境中使用的頻道，可以使用新的 `conda.yaml` 將模型重新註冊到 model registry。您可以透過在 `log_model()` 的 **conda_env** 參數中指定通道來完成此操作。

有關 `log_model()` API 的更多信息，請參閱您正在使用的模型 flavor 的 MLflow 文檔，例如 `mlflow.sklearn.log_model()`。