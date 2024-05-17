# Storage Format

每個 MLflow 模型都是一個包含任意文件的 **目錄**，以及該目錄根目錄中的一個 `MLmodel` 文件，該文件可以定義可以在其中查看模型是使用那種 "flavor"。

Flavor 是使 MLflow 模型變得強大的關鍵概念: 它們是部署工具可以用來理解模型的約定，這使得編寫與任何 ML library 中的模型一起使用的工具成為可能，而無需將每個工具與每個庫整合。 MLflow 定義了其所有內建部署工具都支援的幾種 "standard" flavors，例如描述如何將模型作為 Python 函數運行的 "Python function" flavor。然而，使用者也可以定義和使用其他 flavor。例如，MLflow 的 `mlflow.sklearn` 函式庫允許將模型作為 scikit-learn Pipeline 物件載入回來，以便在支援 scikit-learn 的程式碼中使用，或作為通用 Python 函數在只需要應用模型的工具中使用（例如，帶有選項 `-t sagemaker` 的 mlflow deployments，用於將模型部署到 Amazon SageMaker）。

## MLmodel 檔案

特定模型支援的所有 flavor 均以 YAML 格式在其 `MLmodel` 檔案中定義。例如，`mlflow.sklearn` 輸出模型如下：

```bash
# Directory written by mlflow.sklearn.save_model(model, "my_model")
my_model/
├── MLmodel
├── model.pkl
├── conda.yaml
├── python_env.yaml
└── requirements.txt
```

它的 MLmodel 檔案描述了兩種 flavor：

```yaml
time_created: 2018-05-25T17:28:53.35

flavors:
  sklearn:
    sklearn_version: 0.19.1
    pickled_model: model.pkl
  python_function:
    loader_module: mlflow.sklearn
```

除了列出模型 flavors 的欄位之外，`MLmodel` YAML 格式還可以包含以下欄位：

- `time_created`: 建立模型的日期和時間，採用 UTC ISO 8601 格式。
- `run_id`: 建立模型 run 的 ID（如果模型是使用 MLflow Tracking 保存的）。
- `signature`: JSON 格式的模型簽章。
- `input_example`: 模型 input 的參考範例。
- `mlflow_version`: 用於記錄模型的 MLflow 版本。


## Additional Logged 檔案

為了重新建立環境，每當記錄模型時，MLflow 都會自動記錄 `conda.yaml`、`python_env.yaml` 和 `requirements.txt` 檔案。然後，這些檔案可用於使用 `conda` 或 `virtualenv` 與 `pip` 重新安裝相依性。有關這些文件的更多詳細信息，請參閱 [MLflow 模型如何記錄依賴關係](https://mlflow.org/docs/latest/model/dependencies.html#how-mlflow-records-dependencies)。

記錄模型時，模型元資料檔（`MLmodel`、`conda.yaml`、`python_env.yaml`、`requirements.txt`）將複製到名為 **metadata** 的子目錄。對於 wheeled models，也複製`original_requirements.txt` 檔案。

!!! notes
    當被註冊在 MLflow Model Registry 的模型被下載時，名為 `registered_model_meta` 的 YAML 檔案將會加入下載者端的模型目錄。該檔案包含 MLflow 模型註冊表中引用的模型的 name 和 version，並將用於部署和其他目的。
