# Data Plane (V1)

KServe 的 `V1` 協議提供了跨所有模型框架的標準化預測工作流。此協議版本仍然受支持，但建議用戶遷移到 `V2` 協議以獲得更好的性能和服務運行時之間的標準化。但是，如果用例需要比協議 `v2` 提供的更靈活的模式，`v1` 協議仍然是一個選項。

|API	|Verb	|Path	|Request Payload	|Response Payload|
|-----|-----|-----|-----------------|----------------|
|List Models	|GET	|/v1/models		||{"models": [<model_name>]}|
|Model Ready	|GET	|/v1/models/<model_name>		||{"name": <model_name>,"ready": $bool}|
|Predict	|POST	|/v1/models/<model_name>:predict	|{"instances": []}|{"predictions": []}|
|Explain	|POST	|/v1/models/<model_name>:explain	|{"instances": []}|{"predictions": [], "explanations": []}|

!!! info
    注意：`V1` 協議中的響應負載並未嚴格執行。自定義服務器可以定義並返回自己的響應負載。我們鼓勵使用 KServe 定義的響應負載來保持一致性。

## API 定義

|API |Definition|
|----|----------|
|Predict |"predict" API 對模型執行推理。響應是預測結果。所有推理服務都使用 [Tensorflow V1 HTTP API](https://www.tensorflow.org/tfx/serving/api_rest#predict_api)。|
|Explain | "explain" API 是一個 optional 組件，除了預測之外還提供模型解釋。標準化的解釋器接口與 Tensorflow V1 HTTP API 相同，只是增加了一個 `:explain` 動詞。|
|Model Ready |"model ready" health API 指示特定模型是否已準備好進行推理。如果模型已下載並準備好為請求提供服務，模型就緒端點將返回可訪問列表。|
|List Models |"models" API 在模型註冊表中公開模型列表。
