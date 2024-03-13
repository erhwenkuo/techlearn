# Quickstart

在本文中，我們將介紹建立呼叫工具的 chain 和 agent 的基本方法。工具可以是任何東西——API、函數、資料庫等。工具使我們能夠擴展模型的功能，而不僅僅是輸出文字/訊息。將模型與工具結合使用的關鍵是{==正確提示模型並解析其響應，以便它選擇正確的工具並為它們提供正確的輸入。==}

## Setup

我們需要為本本安裝以下套件包：

```bash
pip install --upgrade --quiet langchain langchain-openai
```

並設定這些環境變數：

```python
import getpass
import os

os.environ["OPENAI_API_KEY"] = getpass.getpass()

# If you'd like to use LangSmith, uncomment the below
# os.environ["LANGCHAIN_TRACING_V2"] = "true"
# os.environ["LANGCHAIN_API_KEY"] = getpass.getpass()
```

## Create a tool

首先，我們需要建立一個呼叫的工具。對於此範例，我們將從函數建立自訂工具。有關創建自訂工具的更多信息，請參閱[本指南](https://python.langchain.com/docs/modules/agents/tools/)。

```python
from langchain_core.tools import tool


@tool
def multiply(first_int: int, second_int: int) -> int:
    """Multiply two integers together."""
    return first_int * second_int

# 觀察下面三種重要的 properties
print(multiply.name)
print(multiply.description)
print(multiply.args)
```

結果:

```bash
multiply
multiply(first_int: int, second_int: int) -> int - Multiply two integers together.
{'first_int': {'title': 'First Int', 'type': 'integer'}, 'second_int': {'title': 'Second Int', 'type': 'integer'}}
```

執行看看:

```python
multiply.invoke({"first_int": 4, "second_int": 5})
```

結果:

```bash
20
```

## Chains

如果我們知道我們只需要使用某個工具固定次數，我們就可以建立一個 chain 來執行此操作。讓我們建立一個簡單的 chain，僅將使用者指定的數字相乘。

![](./assets/tool_chain.svg)

