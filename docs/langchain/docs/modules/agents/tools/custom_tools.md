# 定義 Custom Tools

在建立您自己的 agent 程式時，您需要為其提供可以使用的工具清單。除了呼叫的實際函數之外，該工具還包含幾個元件：

- `name` (str) 是必需的，並且在提供給 agent 的一組工具中必須是唯一的
- `description` (str) 是可選的，但建議使用，{==因為 agent 使用它來確定是否使用這個工具==}
- `args_schema` (Pydantic BaseModel) 是可選的，但建議使用，可用於提供更多資訊給 LLM 來從自然語中構建出呼叫工具所使用的參數。

定義工具有多種方法。在本指南中，我們將介紹如何執行兩個功能：

1. 一個始終返回字串 “LangChain” 的虛構搜尋函數
2. 將兩個數字相乘的乘數函數

這裡最大的差異是第一個函數只需要一個輸入，而第二個函數需要多個輸入。許多 agent 程式僅使用需要單一輸入的功能，因此了解如何使用這些功能非常重要。在大多數情況下，定義這些自訂工具是相同的，但也存在一些差異。

```python
# Import things that are needed generically
from langchain.pydantic_v1 import BaseModel, Field
from langchain.tools import BaseTool, StructuredTool, tool
```

## @tool decorator

這個 `@tool` 裝飾器是定義自訂工具最簡單的方法。裝飾器預設使用 **函數名稱** 作為工具名稱，但是可以透過傳遞字串作為第一個參數來覆寫它。此外，裝飾器將使用函數的 docstring 作為工具的描述 - 因此必須提供 docstring。

```python
@tool
def search(query: str) -> str:
    """Look up things online."""
    return "LangChain"
```

看一下這個工具的 name, description 與 args:

```python
print(search.name)

print(search.description)

print(search.args)
```

結果:

```bash
search

search(query: str) -> str - Look up things online.

{'query': {'title': 'Query', 'type': 'string'}}
```

讓我們再定義一個工具:

```python
@tool
def multiply(a: int, b: int) -> int:
    """Multiply two numbers."""
    return a * b
```

看一下這個工具的 name, description 與 args:

```python
print(multiply.name)

print(multiply.description)

print(multiply.args)
```

結果:

```bash
multiply

multiply(a: int, b: int) -> int - Multiply two numbers.

{'a': {'title': 'A', 'type': 'integer'}, 'b': {'title': 'B', 'type': 'integer'}}
```

您也可以透過將工具名稱和 JSON 參數傳遞到工具裝飾器中來自訂它們。

```python
class SearchInput(BaseModel):
    query: str = Field(description="should be a search query")


@tool("search-tool", args_schema=SearchInput, return_direct=True)
def search(query: str) -> str:
    """Look up things online."""
    return "LangChain"
```

看一下這個工具的 name, description 與 args:

```python
print(search.name)

print(search.description)

print(search.args)

print(search.return_direct)
```

結果:

```bash
search-tool

search-tool(query: str) -> str - Look up things online.

{'query': {'title': 'Query', 'description': 'should be a search query', 'type': 'string'}}

True
```

## 創建 BaseTool 的子類別

您也可以透過子類化 BaseTool 類別來明確定義自訂工具。這提供了對工具定義的最大控制，但工作量會多一些。

```python
from typing import Optional, Type

from langchain.callbacks.manager import (
    AsyncCallbackManagerForToolRun,
    CallbackManagerForToolRun,
)


class SearchInput(BaseModel):
    query: str = Field(description="should be a search query")


class CalculatorInput(BaseModel):
    a: int = Field(description="first number")
    b: int = Field(description="second number")


class CustomSearchTool(BaseTool):
    name = "custom_search"
    description = "useful for when you need to answer questions about current events"
    args_schema: Type[BaseModel] = SearchInput

    def _run(
        self, query: str, run_manager: Optional[CallbackManagerForToolRun] = None
    ) -> str:
        """Use the tool."""
        return "LangChain"

    async def _arun(
        self, query: str, run_manager: Optional[AsyncCallbackManagerForToolRun] = None
    ) -> str:
        """Use the tool asynchronously."""
        raise NotImplementedError("custom_search does not support async")


class CustomCalculatorTool(BaseTool):
    name = "Calculator"
    description = "useful for when you need to answer questions about math"
    args_schema: Type[BaseModel] = CalculatorInput
    return_direct: bool = True

    def _run(
        self, a: int, b: int, run_manager: Optional[CallbackManagerForToolRun] = None
    ) -> str:
        """Use the tool."""
        return a * b

    async def _arun(
        self,
        a: int,
        b: int,
        run_manager: Optional[AsyncCallbackManagerForToolRun] = None,
    ) -> str:
        """Use the tool asynchronously."""
        raise NotImplementedError("Calculator does not support async")
```

構建:

```python
search = CustomSearchTool()

print(search.name)
print(search.description)
print(search.args)
```

結果:

```bash
custom_search
useful for when you need to answer questions about current events
{'query': {'title': 'Query', 'description': 'should be a search query', 'type': 'string'}}
```

構建:

```python
multiply = CustomCalculatorTool()

print(multiply.name)
print(multiply.description)
print(multiply.args)
print(multiply.return_direct)
```

結果:

```bash
Calculator
useful for when you need to answer questions about math
{'a': {'title': 'A', 'description': 'first number', 'type': 'integer'}, 'b': {'title': 'B', 'description': 'second number', 'type': 'integer'}}
True
```

## StructuredTool dataclass

您也可以使用 `StructuredTool` 資料類別。這種方法是前兩種方法的混合。它比繼承 `BaseTool` 類別更方便，但提供的功能比僅使用裝飾器更多。

```python
def search_function(query: str):
    return "LangChain"


search = StructuredTool.from_function(
    func=search_function,
    name="Search",
    description="useful for when you need to answer questions about current events",
    # coroutine= ... <- you can specify an async method if desired as well
)

print(search.name)
print(search.description)
print(search.args)
```

結果:

```
Search
Search(query: str) - useful for when you need to answer questions about current events
{'query': {'title': 'Query', 'type': 'string'}}
```

您也可以定義自訂 `args_schema` 以提供有關輸入參數更詳細資訊。

```python
class CalculatorInput(BaseModel):
    a: int = Field(description="first number")
    b: int = Field(description="second number")


def multiply(a: int, b: int) -> int:
    """Multiply two numbers."""
    return a * b


calculator = StructuredTool.from_function(
    func=multiply,
    name="Calculator",
    description="multiply numbers",
    args_schema=CalculatorInput,
    return_direct=True,
    # coroutine= ... <- you can specify an async method if desired as well
)

print(calculator.name)
print(calculator.description)
print(calculator.args)
```

結果:

```
Calculator
Calculator(a: int, b: int) -> int - multiply numbers
{'a': {'title': 'A', 'description': 'first number', 'type': 'integer'}, 'b': {'title': 'B', 'description': 'second number', 'type': 'integer'}}
```

## 處理 Tool Errors

當工具遇到錯誤且未捕獲異常時，代理將停止執行。如果您希望代理繼續執行，您可以引發 `ToolException` 並相應地設定 `handle_tool_error`。

當拋出 `ToolException` 時，代理不會停止工作，而是根據工具的 `handle_tool_error` 變數處理異常，並將處理結果傳回代理作為觀察，並以紅色列印。

您可以將 `handle_tool_error` 設定為 `True`，將其設定為統一的字串值，或將其設為函數。如果將其設為函數，則函數應採用 `ToolException` 作為參數並傳回 `str` 值。

請注意，僅引發 `ToolException` 是無效的。您需要先設定工具的 `handle_tool_error`，因為它的預設值為 `False`。

```python
from langchain_core.tools import ToolException


def search_tool1(s: str):
    raise ToolException("The search tool1 is not available.")
```

首先，讓我們看看如果我們不設定 `handle_tool_error` 會發生什麼事——它會出錯。

```python
search = StructuredTool.from_function(
    func=search_tool1,
    name="Search_tool1",
    description="A bad tool",
)

search.run("test")
```

結果:

```bash
ToolException: The search tool1 is not available.
```

現在，讓我們將 `handle_tool_error` 設定為 `True`

```python
search = StructuredTool.from_function(
    func=search_tool1,
    name="Search_tool1",
    description="A bad tool",
    handle_tool_error=True,
)

search.run("test")
```

結果:

```bash
The search tool1 is not available.
```

我們也可以定義自訂方式來處理工具錯誤:

```python
def _handle_error(error: ToolException) -> str:
    return (
        "The following errors occurred during tool execution:"
        + error.args[0]
        + "Please try another tool."
    )


search = StructuredTool.from_function(
    func=search_tool1,
    name="Search_tool1",
    description="A bad tool",
    handle_tool_error=_handle_error,
)

search.run("test")
```

結果:

```bash
The following errors occurred during tool execution:The search tool1 is not available.Please try another tool.
```
