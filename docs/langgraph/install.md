# Install LangGraph

安裝基礎 LangGraph 套件：

=== "pip"
    ```bash
    pip install -U langgraph
    ```

=== "uv"
    ```bash
    uv add langgraph
    ```

要使用 LangGraph，通常需要存取語言學習模型 (LLM) 並定義工具。您可以根據自己的需求選擇合適的方式。

一種方法（我們將在文件中使用）是使用 [LangChain](https://docs.langchain.com/oss/python/langchain/overview)。

使用以下指令安裝 LangChain：

=== "pip"
    ```bash
    pip install -U langchain
    # Requires Python 3.10+
    ```

=== "uv"
    ```bash
    uv add langchain
    # Requires Python 3.10+
    ```

要使用特定的LLM提供者套件包，您需要單獨安裝它們。

請參閱 [integrations](https://docs.langchain.com/oss/python/integrations/providers/overview) 頁面，以取得特定提供者的安裝說明。

