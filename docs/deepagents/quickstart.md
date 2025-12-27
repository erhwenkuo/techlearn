# Quickstart

快速構建您的第一個 deep agents。

本指南將引導您使用規劃、檔案系統工具和子智能體功能來建立您的第一個 deep agents。您將建立一個能夠執行研究並撰寫報告的研究型智能體。


## Prerequisites

在開始之前，請確保您已從模型提供者（例如 Anthropic, OpenAI）處取得 API 金鑰。

### Step 1: Install dependencies

=== "pip"
    ```bash
    pip install deepagents tavily-python
    ```

=== "uv"

    ```bash
    uv init
    uv add deepagents tavily-python
    uv sync
    ```

=== "poetry"
    ```bash
    poetry add deepagents tavily-python
    ```

### Step 2: Set up your API keys

```bash
export ANTHROPIC_API_KEY="your-api-key"
export TAVILY_API_KEY="your-tavily-api-key"
```

### Step 3: Create a search tool

```python
import os
from typing import Literal
from tavily import TavilyClient
from deepagents import create_deep_agent

tavily_client = TavilyClient(api_key=os.environ["TAVILY_API_KEY"])

def internet_search(
    query: str,
    max_results: int = 5,
    topic: Literal["general", "news", "finance"] = "general",
    include_raw_content: bool = False,
):
    """Run a web search"""
    return tavily_client.search(
        query,
        max_results=max_results,
        include_raw_content=include_raw_content,
        topic=topic,
    )
```

### Step 4: Create a deep agent

```python
# System prompt to steer the agent to be an expert researcher
research_instructions = """You are an expert researcher. Your job is to conduct thorough research and then write a polished report.

You have access to an internet search tool as your primary means of gathering information.

## `internet_search`

Use this to run an internet search for a given query. You can specify the max number of results to return, the topic, and whether raw content should be included.
"""

agent = create_deep_agent(
    tools=[internet_search],
    system_prompt=research_instructions
)
```

### Step 5: Run the agent

```python
result = agent.invoke({"messages": [{"role": "user", "content": "What is langgraph?"}]})

# Print the agent's response
print(result["messages"][-1].content)
```

## What happened?

您的深度代理會自動執行以下操作：

1. **Planned its approach**: 使用內建的 `write_todos` 工具將研究任務分解成多個部分。
2. **Conducted research**: 呼叫互聯網搜尋工具來收集信息
3. **Managed context**: 使用檔案系統工具（`write_file`, `read_file`）卸載大量搜尋結果
4. **Spawned subagents** (if needed): 將複雜的子任務委派給專門的子代理人
5. **Synthesized a report**: 將調查結果匯總成一份連貫的回應
