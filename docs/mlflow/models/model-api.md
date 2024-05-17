# Model API

您可以透過多種方式儲存和載入 MLflow 模型。首先，MLflow 包含與多個通用程式庫的整合。例如，`mlflow.sklearn` 包含 scikit-learn 模型的 `save_model()`、`log_model()` 和 `load_model()` 函數。其次，您可以使用 `mlflow.models.Model` 類別來建立和編寫模型。該類別有四個關鍵功能：

- [`add_flavor`](https://mlflow.org/docs/latest/python_api/mlflow.models.html#mlflow.models.Model.add_flavor) 為模型添加支援不同框架的 flavor。每個 flavor都有一個字串名稱和一個鍵值屬性字典，其中值可以是任何可以序列化為 YAML 的物件。
- `save` 將模型儲存到本地目錄。
- `log` 使用 MLflow Tracking 將模型記錄為目前運行中的工件。
- `load` 從本機目錄或先前執行的工件載入模型。

## MLfflow 內建 Model Flavors

MLflow 提供了幾種可能對您的應用程式有用的標準 flavor。具體來說，它的許多部署工具都支援這些 flavor，因此您可以以這些 flavor 之一導出自己的模型，以從所有這些工具中受益：

- Python Function (python_function)
- R Function (crate)
- H2O (h2o)
- Keras (keras)
- MLeap (mleap)
- PyTorch (pytorch)
- Scikit-learn (sklearn)
- Spark MLlib (spark)
- TensorFlow (tensorflow)
- ONNX (onnx)
- MXNet Gluon (gluon)
- XGBoost (xgboost)
- LightGBM (lightgbm)
- CatBoost (catboost)
- Spacy(spaCy)
- Fastai(fastai)
- Statsmodels (statsmodels)
- Prophet (prophet)
- Pmdarima (pmdarima)
- OpenAI (openai) (Experimental)
- LangChain (langchain) (Experimental)
- John Snow Labs (johnsnowlabs) (Experimental)
- Diviner (diviner)
- Transformers (transformers) (Experimental)
- SentenceTransformers (sentence_transformers) (Experimental)
- Promptflow (promptflow) (Experimental)

