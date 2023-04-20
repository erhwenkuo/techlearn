# Jupyter TensorFlow 範例

在 Kubeflow Notebooks 中使用 Jupyter 和 TensorFlow 的範例。

## Mnist Example

1. 創建 "Notebook" 時，選擇一個安裝了 Jupyter 和 TensorFlow 的容器鏡像
    - 例如, `jupyter-tensorflow-full:v1.7.0`

2. 使用 Jupyter 的界面創建一個新的 `Python 3` 筆記本。

    ![](./assets/nb-tesorflow-mnist.png)

3. 複製以下程碼並將其粘貼到您的筆記本中：

    - 參考: [TensorFlow 2 quickstart for beginners](https://www.tensorflow.org/tutorials/quickstart/beginner)

    ```python
    # Set up TensorFlow
    import tensorflow as tf

    print("TensorFlow version:", tf.__version__)

    # Load a dataset
    mnist = tf.keras.datasets.mnist

    (x_train, y_train), (x_test, y_test) = mnist.load_data()

    x_train, x_test = x_train / 255.0, x_test / 255.0

    # Build a tf.keras.Sequential model
    model = tf.keras.models.Sequential([
    tf.keras.layers.Flatten(input_shape=(28, 28)),
    tf.keras.layers.Dense(128, activation='relu'),
    tf.keras.layers.Dropout(0.2),
    tf.keras.layers.Dense(10)
    ])

    predictions = model(x_train[:1]).numpy()

    tf.nn.softmax(predictions).numpy()

    loss_fn = tf.keras.losses.SparseCategoricalCrossentropy(from_logits=True)

    loss_fn(y_train[:1], predictions).numpy()

    model.compile(optimizer='adam',
                loss=loss_fn,
                metrics=['accuracy'])

    # Train and evaluate your model
    model.fit(x_train, y_train, epochs=5)

    model.evaluate(x_test,  y_test, verbose=2)

    probability_model = tf.keras.Sequential([
    model,
    tf.keras.layers.Softmax()
    ])

    probability_model(x_test[:5])
    ```

