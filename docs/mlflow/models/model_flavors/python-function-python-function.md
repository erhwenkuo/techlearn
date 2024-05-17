# Python Function (python_function)

`python_function` 模型 flavor 用來作 MLflow Python 模型的預設模型介面。任何 MLflow Python 模型都應可作為 巷`python_function` 模型載入。這使得其他 MLflow 工具能夠處理任何 Python 模型，無論使用哪個持久性模組或框架來產生模型。這種互通性非常強大，因為它允許任何 Python 模型在各種環境中進行生產。

此外，`python_function` 模型 flavor 為 Python 模型定義了通用檔案系統[模型格式(model format)](https://mlflow.org/docs/latest/python_api/mlflow.pyfunc.html#pyfunc-filesystem-format)，並提供了用於在該格式中儲存和載入模型的實用程式。該格式是獨立的，因為它包含載入和使用模型所需的所有資訊。依賴關係直接與模型一起儲存或透過 conda 環境引用。這種模型格式允許其他工具將其模型與 MLflow 整合。

??? info "Filesystem format"
    Pyfunc 格式定義為包含所有必要資料、程式碼和配置的目錄結構：

    ```bash
    ./dst-path/
        ./MLmodel: configuration
        <code>: code packaged with the model (specified in the MLmodel file)
        <data>: data packaged with the model (specified in the MLmodel file)
        <env>: Conda environment definition (specified in the MLmodel file)
    ```

    目錄結構可能包含可由 `MLmodel` 配置引用的其他內容。

## 如何將模型儲存為 Python 函數格式

大多數 `python_function` 模型都儲存為其他模型 flavor 的一部分 - 例如，所有 mlflow 內建 flavor 都在匯出的模型中包含 `python_function` 風格。此外，[mlflow.pyfunc](https://mlflow.org/docs/latest/python_api/mlflow.pyfunc.html#module-mlflow.pyfunc) 模組定義了用於明確建立 `python_function` 模型的函數。該模組還包括用於創建自訂 Python 模型的實用程序，這是向 ML models 添加自訂 Python 程式碼的便捷方法。有關更多信息，請參閱 [custom Python models documentation](https://mlflow.org/docs/latest/models.html#custom-python-models)。

## 如何載入 Python 函數模型並對其進行評分

您可以透過呼叫 `mlflow.pyfunc.load_model()` 函數在 Python 中載入 `python_function` 模型。請注意，`load_model` 函數假設所有相依性均已可用，且不會檢查或安裝任何相依性（有關使用自動相依性管理部署模型的工具，請參閱[模型部署部分](https://mlflow.org/docs/latest/models.html#built-in-deployment)）。

載入後，您可以透過呼叫 `predict` 方法對模型進行評分，該方法具有以下簽名：

```python
predict(data: Union[pandas.(Series | DataFrame), numpy.ndarray, csc_matrix, csr_matrix, List[Any], Dict[str, Any], str],
        params: Optional[Dict[str, Any]] = None) → Union[pandas.(Series | DataFrame), numpy.ndarray, list, str]
```

所有 PyFunc 模型都支援 `pandas.DataFrame` 作為輸入。除了 pandas.DataFrame 之外，DL PyFunc 模型還將支援 `numpy.ndarrays` 形式的張量輸入。若要驗證模型 flavor 是否支援張量輸入，請檢查 flavor 的文件。

對於具有 column-based schema 的模型，輸入通常以 `pandas.DataFrame` 的形式提供。如果提供將列名對應到值的字典作為具有命名列的模式的輸入，或者如果提供 `python List` 或 `numpy.ndarray` 作為具有未命名列的模式的輸入，則 MLflow 會將輸入轉換為 DataFrame。針對預期資料類型的架構強制執行和轉換是針對 DataFrame 執行的。

對於具有基於 tensor-based schema 的模型，輸入通常以 `numpy.ndarray` 或將張量名稱對應到其 `np.ndarray` 值的字典的形式提供。模式執行將根據模型模式中指定的形狀和類型檢查提供的輸入的形狀和類型，如果不匹配則拋出錯誤。

對於未定義 schema 的模型，不會對模型輸入和輸出進行任何變更。如果模型不接受提供的輸入類型，MLflow 將傳播模型引發的任何錯誤。

PyFunc 模型載入用於預測或推理的 Python 環境可能與其訓練環境不同。如果環境不匹配，呼叫 `mlflow.pyfunc.load_model()` 時將列印警告訊息。此警告聲明將識別訓練期間使用的軟體包與當前環境之間版本不匹配的軟體包。為了取得模型訓練環境的完整依賴關係，您可以呼叫 `mlflow.pyfunc.get_model_dependency()`。此外，如果您想在模型訓練中使用的相同環境中執行模型推理，您可以呼叫 `mlflow.pyfunc.spark_udf()` 並將 **env_manager** 參數設為 `conda`。這將從 `conda.yaml` 檔案產生環境，確保 python UDF 將使用訓練期間使用的確切套件版本執行。

某些 PyFunc 模型可能接受模型載入配置，該配置控制模型的載入方式和預測計算方式。您可以透過檢查模型的flavor 元資料來了解模型支援哪些配置：

```python
model_info = mlflow.models.get_model_info(model_uri)
model_info.flavors[mlflow.pyfunc.FLAVOR_NAME][mlflow.pyfunc.MODEL_CONFIG]
```

或者，您可以載入 PyFunc 模型並檢查 `model_config` 屬性：

```python
pyfunc_model = mlflow.pyfunc.load_model(model_uri)
pyfunc_model.model_config
```

可以透過在 `mlflow.pyfunc.load_model()` 方法中指示 **model_config** 參數在載入時更改模型配置：

```python
pyfunc_model = mlflow.pyfunc.load_model(model_uri, model_config=dict(temperature=0.93))
```

當模型配置值變更時，這些值將成為儲存模型時所使用的配置。指示模型的無效模型配置鍵會導致該配置被忽略。將顯示一條警告，提及被忽略的條目。


