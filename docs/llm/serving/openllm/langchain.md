# LangChain 整合

要使用 langchain 快速啟動本地 LLM，只需執行以下操作：

```python
from langchain.llms import OpenLLM

llm = OpenLLM(model_name="llama", model_id='meta-llama/Llama-2-7b-hf')

llm("What is the difference between a duck and a goose? And why there are so many Goose in Canada?")
```

!!! tips
    默認情況下，OpenLLM 使用 `safetensors` 格式來保存模型。如果模型不支持 `safetensors`，請確保傳遞`serialization="legacy"`以使用舊版 PyTorch bin 格式模型權重檔。

`langchain.llms.OpenLLM` 能夠與遠程 OpenLLM 服務器交互。鑑於在其他地方部署了 OpenLLM 服務器，您可以通過指定其 `URL` 來連接到它：

```python
from langchain.llms import OpenLLM

llm = OpenLLM(server_url='http://44.23.123.1:3000', server_type='grpc')

llm("What is the difference between a duck and a goose? And why there are so many Goose in Canada?")
```

