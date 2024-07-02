# 自訂模型的 predict 方法

在本教程中，我們將探索在 MLflow 的 PyFunc 風格的背景下自訂模型 `predict` 方法的過程。當您在使用 MLflow 部署模型後希望模型的行為具有更大的彈性時，這特別有用。

為了說明這一點，我們將使用著名的 Iris 資料集並使用 scikit-learn 建立基本的邏輯迴歸模型。

```python
from joblib import dump
from sklearn.datasets import load_iris
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import train_test_split

import mlflow
from mlflow.models import infer_signature
from mlflow.pyfunc import PythonModel
```

## 配置追蹤伺服器 uri

此步驟非常重要，可以確保我們將在此筆記本中執行的所有對 MLflow 的呼叫實際上都會記錄到 MLflow 追蹤伺服器。

你自建的 MLfloew: `mlflow.set_tracking_uri("http://my.company.mlflow.tracking.server:<port>)`

讓我們先載入 Iris 資料集並將其分為訓練集和測試集。然後，我們將在訓練資料上訓練一個簡單的邏輯迴歸模型。

```python
iris = load_iris()
x = iris.data[:, 2:]
y = iris.target

x_train, x_test, y_train, y_test = train_test_split(x, y, test_size=0.2, random_state=9001)

model = LogisticRegression(random_state=0, max_iter=5_000, solver="newton-cg").fit(x_train, y_train)
```

這是機器學習中的常見場景。我們有一個經過訓練的模型，我們想用它來進行預測。透過 scikit-learn，該模型提供了幾種方法來執行此操作：

- `predict` - 預測類別標籤
- `predict_proba` - 獲得類別成員機率
- `predict_log_proba` - 獲得每個類別的對數機率

我們可以預測類別標籤，如下所示。

```python
model.predict(x_test)[:5]
```

結果:

```
array([1, 2, 2, 1, 0])
```

我們還可以得到類別成員機率。

```python
model.predict_proba(x_test)[:5]
```

結果:

```
array([[2.64002987e-03, 6.62306827e-01, 3.35053144e-01],
       [1.24429110e-04, 8.35485037e-02, 9.16327067e-01],
       [1.30646549e-04, 1.37480519e-01, 8.62388835e-01],
       [3.70944840e-03, 7.13202611e-01, 2.83087941e-01],
       [9.82629868e-01, 1.73700532e-02, 7.88350143e-08]])
```

以及為每個類別產生對數機率。

```python
model.predict_log_proba(x_test)[:5]
```

結果:

```
array([[ -5.93696505,  -0.41202635,  -1.09346612],
       [ -8.99177441,  -2.48232793,  -0.08738192],
       [ -8.94301498,  -1.98427305,  -0.14804903],
       [ -5.59687209,  -0.33798973,  -1.26199768],
       [ -0.01752276,  -4.05300763, -16.35590859]])
```

雖然直接在同一個 Python 會話中使用模型很簡單，但當我們想要保存該模型並將其載入到其他地方時，尤其是在使用 MLflow 的 PyFunc 風格時，會發生什麼？讓我們來探討一下這個場景。

```python
mlflow.set_experiment("Overriding Predict Tutorial")

sklearn_path = "/tmp/sklearn_model"

with mlflow.start_run() as run:
    mlflow.sklearn.save_model(
        sk_model=model,
        path=sklearn_path,
        input_example=x_train[:2],
    )
```

一旦模型作為 pyfunc 加載，預設行為僅支援 `predict` 方法。當您嘗試呼叫其他方法（例如 `predict_proba`）時，這一點很明顯，導致 AttributeError。這可能是有限制的，特別是當您想要保留原始模型的全部功能時。

```python
loaded_logreg_model = mlflow.pyfunc.load_model(sklearn_path)

loaded_logreg_model.predict(x_test)
```

結果:

```
array([1, 2, 2, 1, 0, 1, 2, 0, 1, 0, 0, 1, 1, 1, 0, 0, 1, 0, 1, 1, 2, 1,
       1, 0, 1, 1, 0, 0, 1, 2])
```

這完全符合我們的預期。輸出與儲存前直接使用模型相同。

讓我們嘗試使用 `predict_proba` 方法。

實際上它不會運行，因為它會引發異常。如果我們嘗試執行此操作，則會出現以下行為：

```python
loaded_logreg_model.predict_proba(x_text)
```

這將導致此錯誤：

```
---------------------------------------------------------------------------
AttributeError                            Traceback (most recent call last)
/var/folders/cd/n8n0rm2x53l_s0xv_j_xklb00000gp/T/ipykernel_15410/1677830262.py in <cell line: 1>()
----> 1 loaded_logreg_model.predict_proba(x_text)

AttributeError: 'PyFuncModel' object has no attribute 'predict_proba'
```

## 如何支援模型的原始行為？

我們可以建立一個自訂 pyfunc 來覆寫 `predict` 方法的行為。

對於下面的範例，我們將展示 pyfunc 的兩個功能，可用來處理自訂模型日誌記錄功能：

- override of the `predict` method
- custom loading of an artifact

需要注意的關鍵問題是使用 `joblib` 進行序列化。雖然 pickle 歷來用於序列化 scikit-learn 模型，但現在建議使用 joblib，因為它提供更好的性能和支持，特別是對於大型 numpy 數組。

我們將使用 joblib 及其轉儲和載入 API 來處理將模型物件載入到自訂 pyfunc 實作中。在實例化 pyfunc 物件時使用 `load_context` 方法來處理載入檔案的過程對於具有非常大或大量工件依賴項（例如 LLM）的模型特別有用，並且可以幫助顯著減少正在運行的 pyfunc 的總記憶體佔用量。到分散式系統（例如Apache Spark 或Ray）中。

```python
from joblib import dump

from mlflow.models import infer_signature
from mlflow.pyfunc import PythonModel
```

為了了解如何在自訂 Python 模型中利用 load_context 功能，我們首先使用 joblib 在本機序列化我們的模型。這裡使用 joblib 純粹是為了示範一種非標準方法（MLflow 本身不支援的方法），以說明 Python 模型實現的靈活性。如果我們在 load_context 中匯入該程式庫並使其在我們將載入該模型的環境中可用，則模型工件將正確反序列化。

```python
model_directory = "/tmp/sklearn_model.joblib"

dump(model, model_directory)
```

結果:

```
['/tmp/sklearn_model.joblib']
```

## 定義 custom PythonModel 類別

下面的 `ModelWrapper` 類別是擴充 MLflow 的 `PythonModel` 的自訂 pyfunc 範例。它透過使用 `predict` 方法的 **params** 參數來提供預測方法的靈活性。這樣，當我們在載入的 pyfunc 實例上呼叫 `predict` 方法時，我們可以指定是否需要常規的 `predict`、`predict_proba` 或 `predict_log_proba` 行為。

```python
class ModelWrapper(PythonModel):
    def __init__(self):
        self.model = None

    def load_context(self, context):
        from joblib import load

        self.model = load(context.artifacts["model_path"])

    def predict(self, context, model_input, params=None):
        params = params or {"predict_method": "predict"}
        predict_method = params.get("predict_method")

        if predict_method == "predict":
            return self.model.predict(model_input)
        elif predict_method == "predict_proba":
            return self.model.predict_proba(model_input)
        elif predict_method == "predict_log_proba":
            return self.model.predict_log_proba(model_input)
        else:
            raise ValueError(f"The prediction method '{predict_method}' is not supported.")
```

定義自訂 pyfunc 後，接下來的步驟涉及使用 MLflow 儲存模型，然後將其載入回來。載入的模型將保留我們在自訂 pyfunc 中內建的靈活性，讓我們可以動態選擇預測方法。

!!! info
    注意：下面的 `artifacts` 參考非常重要。為了讓 `load_context` 能夠存取我們指定為保存模型位置的路徑，必須將其作為字典提供，將適當的存取鍵對應到相關值。如果未能提供此字典作為 `mlflow.save_model()` 或 `mlflow.log_model()` 的一部分，將導致此自訂 pyfunc 模型無法正確載入。

```python
# Define the required artifacts associated with the saved custom pyfunc
artifacts = {"model_path": model_directory}

# Define the signature associated with the model
signature = infer_signature(x_train, params={"predict_method": "predict_proba"})
```

我們可以看到如何在簽名定義中使用定義的參數。如下所示，參數在記錄時會發生輕微的變化。我們有一個已定義的參數鍵（predict_method）、預期類型（string）和預設值。此 params 定義的最終意義是：

- 我們只能為關鍵的 `predict_method` 提供參數覆蓋。除此之外的任何內容都將被忽略，並且會顯示一條警告，指示未知參數不會傳遞到底層模型。
- 與 Predict_method 關聯的值必須是字串。不允許任何其他類型，並將針對意外類型引發異常。
- 如果在呼叫預測時未提供 predict_method 的值，則模型將使用 predict_proba 的預設值。

```python
print(signature)
```

結果:

```
inputs:
  [Tensor('float64', (-1, 2))]
outputs:
  None
params:
  ['predict_method': string (default: predict_proba)]
```

我們現在可以保存我們的自訂模型。我們提供了一個保存它的路徑，以及包含我們透過 joblib 儲存的手動序列化實例的位置的工件定義。還包括 signature，這是使該範例正常工作的關鍵組件；如果沒有在 signarture 中定義的參數，我們將無法覆寫預測方法將使用的預測方法。

!!! info
    請注意，我們在這裡重寫 `pip_requirements` 以確保指定兩個依賴庫的要求：`joblib` 和 `sklearn`。這有助於確保我們將此模型部署到的任何環境都會在載入此儲存的模型之前預先載入這兩個依賴項。

```python
pyfunc_path = "/tmp/dynamic_regressor"

with mlflow.start_run() as run:
    mlflow.pyfunc.save_model(
        path=pyfunc_path,
        python_model=ModelWrapper(),
        input_example=x_train,
        signature=signature,
        artifacts=artifacts,
        pip_requirements=["joblib", "sklearn"],
    )
```

現在，我們可以使用 `mlflow.pyfunc.load_model` API 載入模型。

```python
loaded_dynamic = mlflow.pyfunc.load_model(pyfunc_path)
```

讓我們看看 pyfunc 模型在不覆蓋 `params` 參數的情況下會產生什麼。

```python
loaded_dynamic.predict(x_test)
```

結果:

```
array([[2.64002987e-03, 6.62306827e-01, 3.35053144e-01],
       [1.24429110e-04, 8.35485037e-02, 9.16327067e-01],
       [1.30646549e-04, 1.37480519e-01, 8.62388835e-01],
       [3.70944840e-03, 7.13202611e-01, 2.83087941e-01],
       [9.82629868e-01, 1.73700532e-02, 7.88350143e-08],
       [6.54171552e-03, 7.54211950e-01, 2.39246334e-01],
       [2.29127680e-06, 1.29261337e-02, 9.87071575e-01],
       [9.71364952e-01, 2.86348857e-02, 1.62618524e-07],
       [3.36988442e-01, 6.61070371e-01, 1.94118691e-03],
       [9.81908726e-01, 1.80911360e-02, 1.38374097e-07],
       [9.70783357e-01, 2.92164276e-02, 2.15395762e-07],
       [6.54171552e-03, 7.54211950e-01, 2.39246334e-01],
       [1.06968794e-02, 8.88253152e-01, 1.01049969e-01],
       [3.35084116e-03, 6.57732340e-01, 3.38916818e-01],
       [9.82272901e-01, 1.77269948e-02, 1.04445227e-07],
       [9.82629868e-01, 1.73700532e-02, 7.88350143e-08],
       [1.62626101e-03, 5.43474542e-01, 4.54899197e-01],
       [9.82629868e-01, 1.73700532e-02, 7.88350143e-08],
       [5.55685308e-03, 8.02036140e-01, 1.92407007e-01],
       [1.01733783e-02, 8.62455340e-01, 1.27371282e-01],
       [1.43317140e-08, 1.15653085e-03, 9.98843455e-01],
       [4.33536629e-02, 9.32351526e-01, 2.42948113e-02],
       [3.97007654e-02, 9.08506559e-01, 5.17926758e-02],
       [9.19762712e-01, 8.02357267e-02, 1.56085268e-06],
       [4.21970838e-02, 9.26463030e-01, 3.13398863e-02],
       [3.13635521e-02, 9.17295925e-01, 5.13405229e-02],
       [9.77454643e-01, 2.25452265e-02, 1.30412321e-07],
       [9.71364952e-01, 2.86348857e-02, 1.62618524e-07],
       [3.23802803e-02, 9.27626313e-01, 3.99934070e-02],
       [1.21876019e-06, 1.79695714e-02, 9.82029210e-01]])
```

如預期的那樣，它傳回了參數 `predict_method` 的預設值，即 `predict_proba` 的預設值。我們現在可以嘗試覆寫該功能以傳回類別預測。

```python
loaded_dynamic.predict(x_test, params={"predict_method": "predict"})
```

結果:

```
array([1, 2, 2, 1, 0, 1, 2, 0, 1, 0, 0, 1, 1, 1, 0, 0, 1, 0, 1, 1, 2, 1,
       1, 0, 1, 1, 0, 0, 1, 2])
```

我們也可以重寫它以傳回類別成員資格的 `predict_log_proba` 對數機率。

```python
loaded_dynamic.predict(x_test, params={"predict_method": "predict_log_proba"})
```

結果:

```
array([[-5.93696505e+00, -4.12026346e-01, -1.09346612e+00],
       [-8.99177441e+00, -2.48232793e+00, -8.73819177e-02],
       [-8.94301498e+00, -1.98427305e+00, -1.48049026e-01],
       [-5.59687209e+00, -3.37989732e-01, -1.26199768e+00],
       [-1.75227629e-02, -4.05300763e+00, -1.63559086e+01],
       [-5.02955584e+00, -2.82081850e-01, -1.43026157e+00],
       [-1.29864013e+01, -4.34850415e+00, -1.30127244e-02],
       [-2.90530299e-02, -3.55312953e+00, -1.56318587e+01],
       [-1.08770665e+00, -4.13894984e-01, -6.24445569e+00],
       [-1.82569224e-02, -4.01233318e+00, -1.57933050e+01],
       [-2.96519488e-02, -3.53302414e+00, -1.53507887e+01],
       [-5.02955584e+00, -2.82081850e-01, -1.43026157e+00],
       [-4.53780322e+00, -1.18498496e-01, -2.29214015e+00],
       [-5.69854387e+00, -4.18957208e-01, -1.08200058e+00],
       [-1.78861062e-02, -4.03266667e+00, -1.60746030e+01],
       [-1.75227629e-02, -4.05300763e+00, -1.63559086e+01],
       [-6.42147176e+00, -6.09772414e-01, -7.87679430e-01],
       [-1.75227629e-02, -4.05300763e+00, -1.63559086e+01],
       [-5.19272332e+00, -2.20601610e-01, -1.64814232e+00],
       [-4.58798095e+00, -1.47971911e-01, -2.06064898e+00],
       [-1.80607910e+01, -6.76233040e+00, -1.15721450e-03],
       [-3.13836408e+00, -7.00453618e-02, -3.71749248e+00],
       [-3.22638481e+00, -9.59531718e-02, -2.96050653e+00],
       [-8.36395634e-02, -2.52278639e+00, -1.33702783e+01],
       [-3.16540417e+00, -7.63811370e-02, -3.46286367e+00],
       [-3.46210882e+00, -8.63251488e-02, -2.96927492e+00],
       [-2.28033892e-02, -3.79223192e+00, -1.58525647e+01],
       [-2.90530299e-02, -3.55312953e+00, -1.56318587e+01],
       [-3.43020568e+00, -7.51263075e-02, -3.21904066e+00],
       [-1.36176765e+01, -4.01907543e+00, -1.81342258e-02]])
```

我們成功創建了一個 pyfunc 模型，該模型保留了原始 scikit-learn 模型的全部功能，同時使用避開標準 pickle 方法的自訂載入器方法。

本教學重點介紹了 MLflow 的 PyFunc 風格的強大功能和靈活性，示範如何對其進行自訂以滿足您的特定需求。當您繼續建置和部署模型時，請考慮如何使用自訂 pyfunc 來增強模型的功能並適應各種場景。
