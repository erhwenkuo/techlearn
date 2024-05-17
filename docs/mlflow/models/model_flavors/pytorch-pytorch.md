# PyTorch (pytorch)

pytorch 模型 flavor 可以記錄和載入 PyTorch 模型。

[mlflow.pytorch](https://mlflow.org/docs/latest/python_api/mlflow.pytorch.html#module-mlflow.pytorch) 模組定義了用於保存和載入具有 pytorch 風格的 MLflow 模型的實用程式。您可以使用 [mlflow.pytorch.save_model()](https://mlflow.org/docs/latest/python_api/mlflow.pytorch.html#mlflow.pytorch.save_model) 和 [mlflow.pytorch.log_model()](https://mlflow.org/docs/latest/python_api/mlflow.pytorch.html#mlflow.pytorch.log_model) 方法以 MLflow 格式儲存 PyTorch 模型；這兩個函數都使用 [torch.save()](https://pytorch.org/docs/stable/torch.html#torch.save) 方法來序列化 PyTorch 模型。此外，您可以使用 [mlflow.pytorch.load_model()](https://mlflow.org/docs/latest/python_api/mlflow.pytorch.html#mlflow.pytorch.load_model) 方法將具有 pytorch 風格的 MLflow 模型載入為 PyTorch 模型物件。

這個載入的 PyFunc 模型可以使用 `DataFrame` 輸入和 `numpy` 陣列輸入進行評分。最後，由 `mlflow.pytorch.save_model()` 和 `mlflow.pytorch.log_model()` 產生的模型包含 python_function 風格，可讓您將它們載入為通用 Python 函數，以便透過 `mlflow.pyfunc.load_model()` 進行推理。

!!! info
    使用 PyTorch flavor 時，如果 GPU 在預測時可用，則會使用預設 GPU 來執行推理。若要停用此行為，使用者可以使用 `MLFLOW_DEFAULT_PREDICTION_DEVICE` 或透過預測函數的裝置參數傳入裝置。

## PyTorch pyfunc usage

對於最小 PyTorch 模型，pyfunc `Predict()` 方法的範例配置是：

```python
import numpy as np
import mlflow
from mlflow.models import infer_signature
import torch
from torch import nn

# 構建一個小模型
net = nn.Linear(6, 1)

# 定義 loss function
loss_function = nn.L1Loss()

# 選擇優化器
optimizer = torch.optim.Adam(net.parameters(), lr=1e-4)

# 訓練用樣本數據
X = torch.randn(6)
y = torch.randn(1)

# 定義訓練的 epoch
epochs = 5

# 進行模型訓練
for epoch in range(epochs):
    optimizer.zero_grad()
    outputs = net(X)

    loss = loss_function(outputs, y)
    loss.backward()

    optimizer.step()

# 使用 MLflow 來追蹤記錄
with mlflow.start_run() as run:
    # 推導出模型的 input 與 output 的資料結構
    signature = infer_signature(X.numpy(), net(X).detach().numpy())
    # 記錄模型
    model_info = mlflow.pytorch.log_model(net, "model", signature=signature)

# 載入模型
pytorch_pyfunc = mlflow.pyfunc.load_model(model_uri=model_info.model_uri)

# 進行推論
predictions = pytorch_pyfunc.predict(torch.randn(6).numpy())

# 打印結果
print(predictions)
```

有關更多信息，請參閱 [mlflow.pytorch](https://mlflow.org/docs/latest/python_api/mlflow.pytorch.html#module-mlflow.pytorch)。
