# 使用 MLflow 的 Pyfunc 建立自訂 Python 模型的基礎知識

## 簡介

在本教程中，將向您介紹 MLflow pyfunc 的基本概念。我們將說明如何在 MLflow 生態系統中建立、儲存和呼叫自訂 Python 函數模型的簡單性和適應性。最後，您將親身了解一個為 DataFrame 欄位新增指定數值的模型，突顯了 pyfunc 風格的靈活性。

## 你將學到什麼?

- **Simplicity of Custom PyFunc Models**: 掌握 PythonModel 類別的基本結構以及它如何構成 MLflow 中自訂模型的支柱。
- **Model Persistence**: 了解保存和檢索自訂模型的簡單過程。
- **Invoking Predictions**: 了解如何使用已載入的自訂 pyfunc 模型進行預測推論的機制。

## 逐步指南

1. **Model Definition**: 首先建立一個 Python 類，封裝我們簡單的 "Add N" 模型的邏輯。
2. **Persisting the Model**: 使用 MLflow 的功能保存定義的模型，確保以後可以檢索它。
3. **Model Retrieval**: 從已儲存的位置載入模型並準備進行預測。
4. **Model Evaluation**: 在範例資料上使用檢索到的模型來見證其功能。

我們將建立一個模型，將指定的數值 n 加到 Pandas DataFrame 輸入的所有欄位。這將演示定義自訂模型、保存模型、載入模型以及執行預測的過程。

### Step 1: Define the Model Class

我們首先為我們的模型定義一個 Python 類別。此類別應繼承自 `mlflow.pyfunc.PythonModel` 並實作必要的方法。

```python
import mlflow.pyfunc


class AddN(mlflow.pyfunc.PythonModel):
    """
    A custom model that adds a specified value `n` to all columns of the input DataFrame.
    一個自訂模型，將指定值 'n' 新增至輸入 DataFrame 的所有欄位。

    Attributes:
    -----------
    n : int
        The value to add to input columns.
    """

    def __init__(self, n):
        """
        Constructor method. Initializes the model with the specified value `n`.
        構造方法。使用指定值 'n' 初始化模型。

        Parameters:
        -----------
        n : int
            The value to add to input columns.
        """
        self.n = n

    def predict(self, context, model_input, params=None):
        """
        Prediction method for the custom model.
        自訂模型的預測方法。

        Parameters:
        -----------
        context : Any
            Ignored in this example. It's a placeholder for additional data or utility methods.

        model_input : pd.DataFrame
            The input DataFrame to which `n` should be added.

        params : dict, optional
            Additional prediction parameters. Ignored in this example.

        Returns:
        --------
        pd.DataFrame
            The input DataFrame with `n` added to all columns.
        """
        return model_input.apply(lambda column: column + self.n)
```

### Step 2: Save the Model

現在我們的模型類別已定義，我們可以實例化它並使用 MLflow 保存它。

```python
# Define the path to save the model
model_path = "/tmp/add_n_model"

# Create an instance of the model with `n=5`
add5_model = AddN(n=5)

# Save the model using MLflow
mlflow.pyfunc.save_model(path=model_path, python_model=add5_model)
```

### Step 3: Load the Model

儲存模型後，我們可以使用 MLflow 將其載入回來，然後將其用於預測。

```python
# Load the saved model
loaded_model = mlflow.pyfunc.load_model(model_path)
```

### Step 4: Evaluate the Model

現在讓我們使用載入的模型對樣本輸入執行預測並驗證其正確性。

```python
import pandas as pd

# Define a sample input DataFrame
model_input = pd.DataFrame([range(10)])

# Use the loaded model to make predictions
model_output = loaded_model.predict(model_input)
```

## 結論

完成本教學後，您將了解 MLflow 的自訂 pyfunc 提供的易用性和一致性，即使對於最簡單的模型也是如此。它為您可能在後續教程中探索的更高級功能和用例奠定了基礎。