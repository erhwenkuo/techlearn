# Model Evaluation

## 自動化的力量

在不斷發展的機器學習領域，模型開發的評估階段一如既往地重要。確保模型的準確性、可靠性和效率至關重要，以確保已訓練的模型在進入開發階段之前得到盡可能徹底的驗證。

然而，手動評估可能很乏味、容易出錯且耗時。

MLflow 正面應對這些挑戰，提供一套自動化工具來簡化評估流程、節省時間並提高準確性，幫助您確信您花費大量時間開發的解決方案將滿足客戶的需求您試圖解決的問題。

## LLM Model Evaluation

ChatGPT 等大型語言模型 (LLM) 的興起改變了文本生成的格局，在問答、翻譯和文本摘要中找到了應用。然而，評估 LLM 會帶來獨特的挑戰，主要是因為通常沒有單一的基本事實可供比較。 MLflow 的評估工具專為 LLM 量身定制，確保評估流程精簡且準確。

主要特色：

- **Versatile Model Evaluation**: MLflow 支援評估各種類型的 LLM，無論是 MLflow pyfunc 模型、指向已註冊 MLflow 模型的 URI，或是代表模型的任何 Python 呼叫物件。

- **Comprehensive Metrics**: MLflow 提供了一系列用於 LLM 評估的指標。從依賴 OpenAI 等 SaaS 模型進行評分的指標（例如，`mlflow.metrics.genai.answer_relevance()`）到基於函數的每行指標，例如 Rouge (`mlflow.metrics.rougeL()`) 或 Flesch Kincaid (`mlflow. metrics. flesch_kincaid_grade_level()`)。

- **Predefined Metric Collections**: 根據您的 LLM 使用案例，MLflow 提供預先定義的指標集合，例如 "question-answering" 或 "text-summarization"，從而簡化了評估過程。

- **Custom Metric Creation**: 除了預先定義的指標之外，MLflow 還允許使用者建立自訂的 LLM 評估指標。無論您是要評估回應的專業性還是任何其他自訂標準，MLflow 都提供了定義和實施這些指標的工具。

- **Evaluation with Static Datasets**: 從 MLflow 2.8.0 開始，您可以評估靜態資料集而無需指定模型。當您將模型輸出保存在資料集中並希望快速評估而不重新運行模型時，這尤其有用。

- **Integrated Results View**: MLflow 的 `mlflow.evaluate()` 傳回綜合評估結果，可以直接在程式碼中查看或透過 MLflow UI 進行更直觀的表示。

利用這些功能，MLflow 的 LLM 評估工具消除了與評估大型語言模型相關的複雜性和模糊性。透過自動化這些關鍵評估任務，MLflow 確保使用者可以自信地評估其 LLM 的表現，從而在這些模型的部署和應用中做出更明智的決策。

### LLM 模型評估指南和教程

若要詳細了解如何利用 MLflow 的評估功能來進行 LLM 支援的專案工作，請參閱以下教學課程：

- [RAG Evaluation](https://mlflow.org/docs/latest/llms/llm-evaluate/notebooks/rag-evaluation.html)
- [Question-Answering Evaluation](https://mlflow.org/docs/latest/llms/llm-evaluate/notebooks/question-answering-evaluation.html)
- [RAG Question Generation Evaluation](https://mlflow.org/docs/latest/llms/rag/notebooks/question-generation-retrieval-evaluation.html)

## Traditional ML Evaluation

從分類到回歸的傳統機器學習技術一直是許多行業的基石。 MLflow 認識到它們的重要性，並提供針對這些經典技術量身定制的自動化評估工具。

主要特色：

- [Evaluating a Function](https://mlflow.org/docs/latest/models.html#evaluating-with-a-function): 要立即獲得結果，您可以直接評估 python 函數，而無需記錄模型。當您想要快速評估而不需要日誌記錄時，這尤其有用。

- [Evaluating a Dataset](https://mlflow.org/docs/latest/models.html#evaluating-with-a-static-dataset): MLflow 也支援在不指定模型的情況下評估靜態資料集。當您將模型輸出保存在資料集中並希望快速評估而無需重新運行模型推理時，這是非常寶貴的。

- [Evaluating a Model](https://mlflow.org/docs/latest/models.html#performing-model-validation): 使用 MLflow，您可以為指標設定驗證閾值。如果模型與基線相比不滿足這些閾值，MLflow 將向您發出警報。這種自動驗證確保只有高品質的模型才能進入下一階段。

- [Common Metrics and Visualizations](https://mlflow.org/docs/latest/models.html#model-evaluation): MLflow 自動記錄常見指標，如準確度、精確度、召回率等。此外，還會自動記錄混淆矩陣、lift_curve_plot 等視覺化圖表，提供模型效能的全面視圖。

- **SHAP Integration**: MLflow 與 [SHAP](https://shap.readthedocs.io/en/latest/) 集成，允許在使用評估 API 時自動記錄 SHAP 的摘要重要性驗證視覺化。

