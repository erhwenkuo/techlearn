# TensorFlow (tensorflow)

下面的簡單範例展示如何使用低階 TensorFlow API 在 mlflow 中記錄自訂訓練循環的參數和指標。請參閱 tf-keras-範例。有關 mlflow 和 tf.keras 模型的範例。

```python
import numpy as np
import tensorflow as tf

import mlflow

# 訓練的數據
x = np.linspace(-4, 4, num=512)
y = 3 * x + 10

# 估計 w 和 b，其中 y = w * x + b
learning_rate = 0.1
x_train = tf.Variable(x, trainable=False, dtype=tf.float32)
y_train = tf.Variable(y, trainable=False, dtype=tf.float32)

# 初始值
w = tf.Variable(1.0)
b = tf.Variable(1.0)

# 啟動 MLflow 對模型訓練過程的記錄
with mlflow.start_run():
    # 使用 MLflow 記錄超參數
    mlflow.log_param("learning_rate", learning_rate)

    for i in range(1000):
        with tf.GradientTape(persistent=True) as tape:
            # 計算 MSE = 0.5 * (y_predict - y_train)^2
            y_predict = w * x_train + b
            loss = 0.5 * tf.reduce_mean(tf.square(y_predict - y_train))
            # 使用 MLflow 記錄 metrci
            mlflow.log_metric("loss", value=loss.numpy(), step=i)

        # Update the trainable variables
        # w = w - learning_rate * gradient of loss function w.r.t. w
        # b = b - learning_rate * gradient of loss function w.r.t. b
        w.assign_sub(learning_rate * tape.gradient(loss, w))
        b.assign_sub(learning_rate * tape.gradient(loss, b))

print(f"W = {w.numpy():.2f}, b = {b.numpy():.2f}")
```