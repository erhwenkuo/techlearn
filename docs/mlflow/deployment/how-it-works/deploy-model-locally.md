# 用本地推論伺服器部署 MLflow 模型

MLflow 可讓您僅使用單一指令用本地推理伺服器部署 MLflow 模型。此方法非常適合輕量級應用程式或在將模型移至臨時或生產環境之前在本地測試模型。

如果您不熟悉 MLflow 模型部署，請先閱讀[MLflow 部署指南](../index.md)，以了解 MLflow 模型和部署的基本概念。

## 部署推論伺服器

在部署之前，您必須有一個 MLflow 模型。如果您沒有，可以按照 [MLflow 追蹤快速入門](https://mlflow.org/docs/latest/getting-started/intro-quickstart/index.html)建立範例 scikit-learn 模型。

請記得記下模型 URI，例如 `runs:/<run_id>/<artifact_path>`（如果您己經在 MLflow model registry 中註冊了模型，則為模型 URI　是 `models:/<model_name>/<model_version>`）。

準備好模型後，部署到推論伺服器就很簡單了。使用 [`mlflow models serve`](https://mlflow.org/docs/latest/cli.html#mlflow-models-serve) 指令進行推論服務部署。此命令啟動一個本機伺服器，該伺服器偵聽指定連接埠並為您的模型提供服務。

=== "Bash"

    ```bash
    mlflow models serve -m runs:/<run_id>/model -p 5000
    ```

=== "Python"

    ```python
    import mlflow

    model = mlflow.pyfunc.load_model("runs:/<run_id>/model")
    model.serve(port=5000)
    ```

然後您可以向伺服器發送測試請求，如下所示：

```bash
curl http://127.0.0.1:5000/invocations -H "Content-Type:application/json;"  --data '{"inputs": [[1, 2], [3, 4], [5, 6]]}'
```

有幾個命令列選項可用於自訂伺服器的行為。例如，`--env-manager` 選項可讓您選擇特定的環境管理器（例如 Anaconda）來建立虛擬環境。 `mlflow models` 模組還提供了其他有用的命令，例如建立 Docker 映像或產生 Dockerfile。有關全面的詳細信息，請參閱 [MLflow CLI 參考](https://mlflow.org/docs/latest/cli.html#mlflow-models)。

## Inference Server 規格

### Endpoints

推論伺服器提供 4 個端點：

- `/invocations`: 推論端點，接受帶有輸入資料的 POST 請求並傳回預測。
- `/ping`: 用於推論服務健康檢查。
- `/health`: 與上述的 `/ping` 的目的相同。
- `/version`: 返回 MLflow 版本。


### 接受的輸入格式

`/invocations` 端點接受 CSV 或 JSON 輸入。{==輸入格式必須在 **Content-Type** 標頭中指定為 `application/json` 或 `application/csv`。==}

**CSV Input**

CSV 輸入必須是有效的 `pandas.DataFrame CSV` 表示形式。例如：

```bash
curl http://127.0.0.1:5000/invocations -H 'Content-Type: application/csv' --data '1,2,3,4'
```

**JSON Input**

如果您的模型格式不受上述支持，或者您希望避免將輸入資料轉換為所需的有效負載格式，則可以利用下面的字典有效負載結構。

|Field|Description|Example|
|-----|-----------|-------|
|`dataframe_split`|Pandas DataFrames in the split orientation.|`{"dataframe_split": pandas_df.to_dict(orient="split")}`|
|`dataframe_records`|Pandas DataFrame in the records orientation. We do not recommend using this format because it is not guaranteed to preserve column ordering.|`{"dataframe_records": pandas_df.to_dict(orient="records")}`|
|`instances`|Tensor input formatted as described in TF Serving’s API docs where the provided inputs will be cast to Numpy arrays.|`{"instances": [1.0, 2.0, 5.0]}`|
|`inputs`|Same as instances but with a different key.|`{"inputs": [["Cheese"], ["and", "Crackers"]]}`|

**例子:**

```python
# Prerequisite: serve a custom pyfunc OpenAI model (not mlflow.openai) on localhost:5678
#   that defines inputs in the below format and params of `temperature` and `max_tokens`

import json
import requests

payload = json.dumps(
    {
        "inputs": {"messages": [{"role": "user", "content": "Tell a joke!"}]},
        "params": {
            "temperature": 0.5,
            "max_tokens": 20,
        },
    }
)
response = requests.post(
    url=f"http://localhost:5678/invocations",
    data=payload,
    headers={"Content-Type": "application/json"},
)
print(response.json())
```

JSON 輸入還可以包含一個可選的 `params` 字段，用於傳遞其他參數。有效參數類型為 `Union[DataType, List[DataType], None]`，其中 DataType 是 [MLflow 資料類型](https://mlflow.org/docs/latest/python_api/mlflow.types.html#mlflow.types.DataType)。要傳遞參數，必須定義帶有參數的有效模型簽章。

```python
curl http://127.0.0.1:5000/invocations -H 'Content-Type: application/json' -d '{
    "inputs": {"question": ["What color is it?"],
               "context": ["Some people said it was green but I know that it is pink."]},
    "params": {"max_answer_len": 10}
}'
```

**Encoding complex data**

複雜資料類型（例如日期或二進位）沒有 native JSON 表示形式。如果您包含模型簽名，MLflow 可以自動從 JSON 解碼支援的資料類型。支援以下資料類型轉換：

- `binary`: 資料預計採用 Base64 編碼，MLflow 將自動進行 Base64 解碼。
- `datetime`: 資料應根據 ISO 8601 規範編碼為字串。 MLflow 會將其解析為給定平台上適當的日期時間表示形式。

請求範例：

```python
# record-oriented DataFrame input with binary column "b"
curl http://127.0.0.1:5000/invocations -H 'Content-Type: application/json' -d '[
    {"a": 0, "b": "dGVzdCBiaW5hcnkgZGF0YSAw"},
    {"a": 1, "b": "dGVzdCBiaW5hcnkgZGF0YSAx"},
    {"a": 2, "b": "dGVzdCBiaW5hcnkgZGF0YSAy"}
]'

# record-oriented DataFrame input with datetime column "b"
curl http://127.0.0.1:5000/invocations -H 'Content-Type: application/json' -d '[
    {"a": 0, "b": "2020-01-01T00:00:00Z"},
    {"a": 1, "b": "2020-02-01T12:34:56Z"},
    {"a": 2, "b": "2021-03-01T00:00:00Z"}
]'
```

## Serving Frameworks

預設情況下，MLflow 使用 **Flask**（一個用於 Python 的輕量級 WSGI Web 應用程式框架）來服務推論端點。然而，Flask 主要是為輕量級應用程式設計的，可能不適合大規模生產用例。為了解決這一差距，MLflow 與 **MLServer** 整合作為替代服務引擎。 MLServer 透過利用非同步請求/回應範例和工作負載卸載來實現更高的效能和可擴充性。此外，MLServer 也用作 Seldon Core 和 [KServe](https://kserve.github.io/website/) 等 Kubernetes 原生框架中的核心 Python 推論伺服器，因此它提供了金絲雀部署和開箱即用自動擴展等高級功能。

||Flask|MLServer|
|----|----|-----|
|**Use Case**|輕量級目的，包括本地測試。|高規模生產環境。|
|**Set Up**|Flask 預設隨 MLflow 一起安裝。|需要單獨安裝。|
|**Performance**|適合輕量級應用程序，但未針對高效能進行最佳化，如 WSGI 應用程式。 WSGI 是基於同步請求/回應範例，由於阻塞性質，這對於 ML 工作負載來說並不理想。機器學習預測通常涉及大量計算，並且可能需要很長時間才能完成，因此在處理請求時阻塞伺服器並不理想。雖然 Flask 可以透過 Uvicorn 等非同步框架進行增強，但 MLflow 並不支援它們開箱即用，而只是使用 Flask 的預設同步行為。|專為高性能機器學習工作負載而設計，通常可提供更好的吞吐量和效率。 MLServer 支援非同步請求/回應範例，透過將 ML 推理工作負載卸載到單獨的工作池（進程），以便伺服器可以在處理推理時繼續接受新請求。請參閱 MLServer Parallel Inference 以了解有關它們如何實現這一點的更多詳細資訊。此外，MLServer 支援自適應 Bacthing，可以透明地批量處理請求，以提高吞吐量和效率。|
|**Scalability**|出於與性能相同的原因，本質上不具有可擴展性。|除了上述的平行推理的支援之外，MLServer 還被用作 Kubernetes 原生框架（例如 Seldon Core 和 KServe（以前稱為 KFServing））中的核心推理伺服器。透過使用 MLServer 將 MLflow 模型部署到 Kubernetes，您可以利用這些框架的進階功能（例如自動縮放）來實現高可擴充性。

MLServer 透過 `/invocations` 端點公開相同的推論 API。若要使用 MLServer 進行部署，請先使用 `pip install mlflow[extras]` 安裝其他依賴項，然後使用 `--enable-mlserver` 選項執行部署指令。例如:

=== "Bash"

    ```bash
    mlflow models serve -m runs:/<run_id>/model -p 5000 --enable-mlserver
    ```

=== "Python"

    ```python
    import mlflow

    model = mlflow.pyfunc.load_model("runs:/<run_id>/model")

    model.serve(port=5000, enable_mlserver=True)
    ```

    要了解有關 MLflow 和 MLServer 之間整合的更多信息，請查看 MLServer 文件中的[端到端範例](https://mlserver.readthedocs.io/en/latest/examples/mlflow/README.html)。您也可以在[將模型部署到 Kubernetes](https://mlflow.org/docs/latest/deployment/deploy-model-to-kubernetes/index.html) 中找到使用 MLServer 將 MLflow 模型部署到 Kubernetes 叢集的指南。

## Running Batch Inference

您可以使用 `mlflow models Predict` 指令對本機檔案執行單一批次推論作業，而不是執行線上推論端點。以下命令對 input.csv 運行模型預測並將結果輸出到 output.csv。

=== "Bash"

    ```bash
    mlflow models predict -m runs:/<run_id>/model -i input.csv -o output.csv
    ```

=== "Python"

    ```python
    import mlflow

    model = mlflow.pyfunc.load_model("runs:/<run_id>/model")
    
    predictions = model.predict(pd.read_csv("input.csv"))

    predictions.to_csv("output.csv")
    ```
