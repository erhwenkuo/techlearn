# OpenLLM

🦾 [OpenLLM](https://github.com/bentoml/OpenLLM) 是一個用於在生產中操作大型語言模型 (LLM) 的開放平台。它使開發人員能夠輕鬆地使用任何開源 LLMs 運行推理、部署到雲端或本地，並構建強大的人工智能應用程序。

## 安裝

通過 [PyPI](https://pypi.org/project/openllm/) 安裝 `openllm`

```bash
pip install openllm
```

## 本地啟動 OpenLLM 服務器

要啟動 LLM 服務器，請使用 `openllm start` 命令。例如，要啟動 `dolly-v2` 服務器，請從終端(terminal)運行以下命令：

```bash
openllm start dolly-v2
```

詳細命令請參考: [OpenLLM 模型支援](../../../../llm/serving/openllm/models_supported.md)

## Wrapper 類別

連接遠端的 OpenLLM 服務器實例:

```python
from langchain.llms import OpenLLM

server_url = "http://localhost:3000"  # Replace with remote host if you are running on a remote server
llm = OpenLLM(server_url=server_url)
```

API Reference:

- [OpenLLM](https://api.python.langchain.com/en/latest/llms/langchain.llms.openllm.OpenLLM.html)

### 本地 LLM 推論

您還可以選擇從當前進程本地初始化由 OpenLLM 管理的 LLM。這對於開發目的很有用，並允許開發人員快速嘗試不同類型的 LLM。

將 LLM 應用程序轉移到生產環境時，我們建議單獨部署 OpenLLM 服務器並通過上面演示的 `server_url` 選項進行訪問。

要通過 LangChain 包裝器在本地加載 LLM：

```python
from langchain.llms import OpenLLM

llm = OpenLLM(
    model_name="dolly-v2",
    model_id="databricks/dolly-v2-3b",
    temperature=0.94,
    repetition_penalty=1.2,
)
```

API Reference:

- [OpenLLM](https://api.python.langchain.com/en/latest/llms/langchain.llms.openllm.OpenLLM.html)

### 與 LLMChain 整合

```python
from langchain.prompts import PromptTemplate
from langchain.chains import LLMChain

# 構建 prompt 範本
template = "What is a good name for a company that makes {product}?"

prompt = PromptTemplate(template=template, input_variables=["product"])

# 構建 llm_chain 的實例
llm_chain = LLMChain(prompt=prompt, llm=llm)

# 傳入 prompt 範本參數與觸動 LLM 來生成結果
generated = llm_chain.run(product="mechanical keyboard")

print(generated)
```

API Reference:

- [PromptTemplate]()
- [LLMChain]()