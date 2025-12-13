# Run a local server

本指南將向您展示如何在本機上運行 LangGraph 應用程式。

## Prerequisites

在開始之前，請確保您已準備好以下物品：
- LangSmith API 金鑰 - 免費註冊

## 1. Install the LangGraph CLI


=== "pip"
    ```bash
    # Python >= 3.11 is required.
    pip install -U "langgraph-cli[inmem]"
    ```

=== "uv"
    ```bash
    # Python >= 3.11 is required.
    uv add 'langgraph-cli[inmem]'
    ```


## 2. Create a LangGraph app

使用 [new-langgraph-project-python 範本](https://github.com/langchain-ai/new-langgraph-project) 建立一個新應用程式。此範本示範了一個 single-node 應用，您可以根據自己的邏輯進行擴充。

```bash
langgraph new path/to/your/app --template new-langgraph-project-python
```

!!! tip "Additional templates"
    如果您使用 `langgraph new` 而不指定 `--template`，則會顯示一個互動式選單，您可以從中選擇可用模板。

    ```bash
    $ langgraph new path/to/your/app

    🌟 Please select a template:
    1. New LangGraph Project - A simple, minimal chatbot with memory.
    2. ReAct Agent - A simple agent that can be flexibly extended to many tools.
    3. Memory Agent - A ReAct-style agent with an additional tool to store memories for use across conversational threads.
    4. Retrieval Agent - An agent that includes a retrieval-based question-answering system.
    5. Data-enrichment Agent - An agent that performs web searches and organizes its findings into a structured format.
    Enter the number of your template choice (default is 1):
    ```

## 3. Install dependencies

在新建的 LangGraph 應用的根目錄中，以編輯模式安裝依賴項，以便伺服器能夠使用您本地所做的變更：

=== "pip"
    ```bash
    cd path/to/your/app
    pip install -e .
    ```

=== "uv"
    ```bash
    cd path/to/your/app
    uv sync
    ```

## 4. Create a .env file

您會在新建的 LangGraph 應用程式根目錄下找到一個 `.env.example` 檔案。在新建的 LangGraph 應用程式根目錄下建立一個 `.env` 文件，並將 `.env.example` 檔案的內容複製到該檔案中，並填寫必要的 API 金鑰：

``` title=".env"
# To separate your traces from other application
LANGSMITH_PROJECT=new-agent
LANGSMITH_API_KEY=lsv2....
# Add API keys for connecting to LLM providers, data sources, and other integrations here
GOOGLE_API_KEY=AIzaS....
```

## 5. Launch Agent server

在本地啟動 LangGraph API 伺服器:

```bash
langgraph dev
```

範例輸出：

```bash
>    Ready!
>
>    - API: [http://localhost:2024](http://localhost:2024/)
>
>    - Docs: http://localhost:2024/docs
>
>    - LangGraph Studio Web UI: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024
```

`langgraph dev` 指令會以 in-memory mode 啟動 Agent Server。此模式適用於 **開發** 和 **測試** 用途。對於生產環境，請將 Agent Server 部署到持久性儲存後端。更多資訊請參閱 [Platform setup overview](https://docs.langchain.com/langsmith/platform-setup)。

## 6. Test your application in Studio

[Studio](https://docs.langchain.com/langsmith/studio) 是一個專用的使用者介面，您可以透過它連接到 LangGraph API 伺服器，在本地視覺化、互動和調試您的應用程式。若要測試您的 graph，請造訪 `langgraph dev` 命令輸出中提供的 URL:

```bash
>    - LangGraph Studio Web UI: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024
```

對於執行在自訂主機/連接埠上的 Agent Server，請更新 URL 中的 `baseUrl` 查詢參數。例如，如果您的伺服器運行在 `http://myhost:3000`:

```bash
https://smith.langchain.com/studio/?baseUrl=http://myhost:3000
```

## 7. Test the API

=== "Python SDK (async)"
    1. 安裝 LangGraph Python SDK:
   
        ```bash
        pip install langgraph-sdk
        ```

    2. 向 assistant 發送訊息（threadless 運行）

        ```python
        from langgraph_sdk import get_client
        import asyncio

        client = get_client(url="http://localhost:2024")

        async def main():
            async for chunk in client.runs.stream(
                None,  # Threadless run
                "agent", # Name of assistant. Defined in langgraph.json.
                input={
                "messages": [{
                    "role": "human",
                    "content": "What is LangGraph?",
                    }],
                },
            ):
                print(f"Receiving new event of type: {chunk.event}...")
                print(chunk.data)
                print("\n\n")

        asyncio.run(main())
        ```


=== "Python SDK (sync)"
    1. 安裝 LangGraph Python SDK:
   
        ```bash
        pip install langgraph-sdk
        ```

    2. 向 assistant 發送訊息（threadless 運行）:

        ```python
        from langgraph_sdk import get_sync_client

        client = get_sync_client(url="http://localhost:2024")

        for chunk in client.runs.stream(
            None,  # Threadless run
            "agent", # Name of assistant. Defined in langgraph.json.
            input={
                "messages": [{
                    "role": "human",
                    "content": "What is LangGraph?",
                }],
            },
            stream_mode="messages-tuple",
        ):
            print(f"Receiving new event of type: {chunk.event}...")
            print(chunk.data)
            print("\n\n")
        ```

=== "Rest API"
    ```bash
    curl -s --request POST \
        --url "http://localhost:2024/runs/stream" \
        --header 'Content-Type: application/json' \
        --data "{
            \"assistant_id\": \"agent\",
            \"input\": {
                \"messages\": [
                    {
                        \"role\": \"human\",
                        \"content\": \"What is LangGraph?\"
                    }
                ]
            },
            \"stream_mode\": \"messages-tuple\"
        }"
    ```

## Next steps

現在您的 LangGraph 應用程式已經在本地運行，接下來可以探索部署和進階功能，進一步提升您的學習體驗：

- [Deployment quickstart](https://docs.langchain.com/langsmith/deployment-quickstart): Deploy your LangGraph app using LangSmith.
- [LangSmith](https://docs.langchain.com/langsmith/home): Learn about foundational LangSmith concepts.
- [SDK Reference](https://reference.langchain.com/python/langsmith/deployment/sdk/?_gl=1*1uimwvn*_gcl_au*MTA0NTc2NjA2OC4xNzYzNzI5OTg2*_ga*MTM5NTM1ODE0Mi4xNzQ3MDkyMTE0*_ga_47WX3HKKY2*czE3NjQ5ODM4ODMkbzUwJGcxJHQxNzY0OTg1MzYzJGo2MCRsMCRoMA..): Explore the SDK API Reference.