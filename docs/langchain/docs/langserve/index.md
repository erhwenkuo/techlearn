# 🦜️🏓 LangServe

## 概述

LangServe 協助開發人員將 LangChain 的 [runnables 與 chains](https://python.langchain.com/docs/expression_language/) 部署為 REST API。

LangServe 整合了 [FastAPI](https://fastapi.tiangolo.com/)，並使用 [pydantic](https://docs.pydantic.dev/latest/) 進行資料驗證。

此外，它還提供了一個可用於呼叫部署在伺服器上的可運行程式的用戶端。 [LangChain.js](https://js.langchain.com/docs/ecosystem/langserve) 中提供了 JavaScript 用戶端。

## 功能

- 輸入和輸出 schema 會自動從 LangChain 物件推斷出來，並在每個 API 呼叫上強制執行，並提供豐富的錯誤訊息
- 包含 JSONSchema 和 Swagger 的 API 文件頁面
- 高效的 `/invoke/`、`/batch/` 和 `/stream/` 端點，支援單一伺服器上的許多並發請求
- `/stream_log/` 端點，用於串流傳輸鏈/代理中的所有（或部分）中間步驟
- 自 0.0.40 起新增，支援 `astream_events`，以便更輕鬆地進行串流傳輸，而無需解析 `stream_log` 的輸出。
- Playground 頁面位於 `/playground/` ，具有串流輸出和中間步驟
- 內建（可選）追蹤 LangSmith，只需新增您的 API 金鑰
- 使用經過實戰考驗的開源 Python 庫（例如 FastAPI、Pydantic、uvloop 和 asyncio）構建的。
- 可使用[LangChain.js](https://js.langchain.com/docs/ecosystem/langserve)客戶端 SDK 呼叫 LangServe 伺服器，就像它是在本地運行的 Runnable 一樣（或直接呼叫 HTTP API）

## 限制

- 對於源自伺服器的事件尚不支援客戶端回調
- 使用 Pydantic V2 時不會產生 OpenAPI 文件。 Fast API 不支援混合 pydantic v1 和 v2 命名空間。有關更多詳細信息，請參閱下面的部分。

## 安裝

對於客戶端和伺服器:

```bash
pip install "langserve[all]"
```

或 `pip install "langserve[client]"` 對於客戶端程式碼，`pip install "langserve[server]"` 對於伺服器程式碼。

## LangChain CLI 🛠️

使用 LangChain CLI 快速 bootstrap `LangServe` project。

要使用 langchain CLI，請確保您安裝了最新版本的 `langchain-cli`。您可以使用 `pip install -U langchain-cli` 安裝它。

## 設定專案

注意：LangServe 使用 [poetry](https://python-poetry.org/) 進行依賴管理。請關注 [poetry](https://python-poetry.org/) 文檔以了解更多資訊。

**1. Create new app using langchain cli command**

```bash
langchain app new my-app
```

**2. Define the runnable in add_routes. Go to `server.py` and edit**

```python
add_routes(app. NotImplemented)
```

**3. Use poetry to add 3rd party packages (e.g., `langchain-openai` etc).**

```bash
poetry add [package-name] // e.g `poetry add langchain-openai`
```

**4. Set up relevant env variables.**

```bash
export OPENAI_API_KEY="sk-..."
```

**5. Serve your app**

```bash
poetry run langchain serve --port=8100
```

## Examples

使用 [LangChain Templates](https://github.com/langchain-ai/langchain/blob/master/templates/README.md) 快速啟動您的 LangServe 實例。

如需更多範例，請參閱範本索引或範例目錄。

|Description	|Links|
|---------------|-----|
|**LLMs** Minimal example that reserves OpenAI and Anthropic chat models. Uses async, supports batching and streaming.|[server](https://github.com/langchain-ai/langserve/tree/main/examples/llm/server.py), [client](https://github.com/langchain-ai/langserve/blob/main/examples/llm/client.ipynb)|
|**Retriever** Simple server that exposes a retriever as a runnable.|[server](https://github.com/langchain-ai/langserve/tree/main/examples/retrieval/server.py), [client](https://github.com/langchain-ai/langserve/tree/main/examples/retrieval/client.ipynb)|
|**Conversational Retriever** A Conversational Retriever exposed via LangServe|[server](https://github.com/langchain-ai/langserve/tree/main/examples/conversational_retrieval_chain/server.py), [client](https://github.com/langchain-ai/langserve/tree/main/examples/conversational_retrieval_chain/client.ipynb)|
|**Agent** without **conversation history** based on [OpenAI tools](https://python.langchain.com/docs/modules/agents/agent_types/openai_functions_agent)|[server](https://github.com/langchain-ai/langserve/tree/main/examples/agent/server.py), [client](https://github.com/langchain-ai/langserve/tree/main/examples/agent/client.ipynb)|
|**Agent** with **conversation history** based on [OpenAI tools](https://python.langchain.com/docs/modules/agents/agent_types/openai_functions_agent)|[server](https://github.com/langchain-ai/langserve/blob/main/examples/agent_with_history/server.py), [client](https://github.com/langchain-ai/langserve/blob/main/examples/agent_with_history/client.ipynb)|





