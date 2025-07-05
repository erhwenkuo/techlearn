# Run a local server

本指南向您展示如何在本機上運行 LangGraph 應用程式。

## Prerequisites

在開始之前，請確保您已具備以下條件：

- LangSmith 的 API 金鑰 
  
## 1. Install the LangGraph CLI

```bash
# Python >= 3.11 is required.

pip install --upgrade "langgraph-cli[inmem]"
```

## 2. Create a LangGraph app 🌱

使用 [new-langgraph-project-python](https://github.com/langchain-ai/new-langgraph-project) 範本建立一個新應用程式。此模板演示了一個單節點應用程序，您可以使用自己的邏輯進行擴展。

```bash
langgraph new path/to/your/app --template new-langgraph-project-python
```

!!! info
    如果您使用 `langgraph new` 而不指定模板，您將看到一個互動式選單，可讓您從可用模板清單中進行選擇。

## 3. Install dependencies

在新的 LangGraph 應用程式的根目錄中，以編輯模式安裝依賴項，以便伺服器使用您的本機變更：

```bash
cd path/to/your/app
pip install -e .
```

## 4. Create a `.env` file

您將在新的 LangGraph 應用的根目錄中找到一個 `.env.example` 檔案。在新的 LangGraph 應用的根目錄中建立一個 `.env` 文件，並將 `.env.example` 文件的內容複製到其中，並填寫必要的 API 金鑰：

```bash
LANGSMITH_API_KEY=lsv2...
```

## 5. Launch LangGraph Server 🚀

在本地啟動 LangGraph API 伺服器：

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

`langgraph dev` 指令會以記憶體模式啟動 LangGraph 伺服器。此模式適用於開發和測試。對於生產環境，請部署 LangGraph 伺服器並使其能夠存取持久性儲存後端。更多信息，請參閱[部署選項](https://langchain-ai.github.io/langgraph/concepts/deployment_options/)。

## 6. Test your application in LangGraph Studio

LangGraph Studio 是一個專門的 UI，您可以連接到 LangGraph API 伺服器，在本地視覺化、互動和調試您的應用程式。透過存取 `langgraph dev` 命令輸出中提供的 URL，在 LangGraph Studio 中測試您的 graph：

```bash
>    - LangGraph Studio Web UI: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024
```

對於在自訂主機/連接埠上執行的 LangGraph 伺服器，請更新 `baseURL` 參數。

## 7. Test the API

=== "Python SDK (async)"

    1. 安裝 LangGraph Python SDK：

        ```bash
        pip install langgraph-sdk
        ```

    2. 向 assistant 發送訊息：

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

    1. 安裝 LangGraph Python SDK：

      ```bash
      pip install langgraph-sdk
    ```

    2. 向 assistant 發送訊息：

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

=== "Javascript SDK"

    1. 安裝 LangGraph JS SDK：

        ```bash
        npm install @langchain/langgraph-sdk
        ```

    2. 向 assistant 發送訊息：

        ```bash
        const { Client } = await import("@langchain/langgraph-sdk");

        // only set the apiUrl if you changed the default port when calling langgraph dev
        const client = new Client({ apiUrl: "http://localhost:2024"});

        const streamResponse = client.runs.stream(
            null, // Threadless run
            "agent", // Assistant ID
            {
                input: {
                    "messages": [
                        { "role": "user", "content": "What is LangGraph?"}
                    ]
                },
                streamMode: "messages-tuple",
            }
        );

        for await (const chunk of streamResponse) {
            console.log(`Receiving new event of type: ${chunk.event}...`);
            console.log(JSON.stringify(chunk.data));
            console.log("\n\n");
        }
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
