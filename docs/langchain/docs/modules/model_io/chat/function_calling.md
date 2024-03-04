# Function calling

越來越多的聊天模型（例如 OpenAI、Gemini 等）具有函數呼叫 API，可讓您描述函數及其參數，並讓模型傳回 JSON 對象，其中包含要呼叫的函數以及該函數的輸入。函數呼叫對於建立使用工具的 chain 和 agent 以及更普遍地從模型中獲取結構化輸出非常有用。

LangChain 附帶了許多實用程式來簡化函數呼叫。也就是說，它帶有：

- 將函數綁定到模型的簡單語法
- 用於將各種類型的物件格式化為預期函數模式的轉換器
- 用於從 API 回應中提取函數呼叫的輸出解析器
- 用於從模型取得結構化輸出的鏈，建構在函數呼叫之上

我們將重點放在前兩點。有關輸出解析的詳細指南，請查看 [OpenAI Tools 輸出解析器](https://python.langchain.com/docs/modules/model_io/output_parsers/types/openai_tools)，若要查看結構化輸出鏈，請查看[結構化輸出指南](https://python.langchain.com/docs/guides/structured_output)。

在開始之前，請確保您已安裝 `langchain-core`。

```bash
%pip install -qU langchain-core langchain-openai
```

```python
import getpass
import os
```

## Binding functions

許多模型實作了 helper methods，這些方法將負責格式化不同的類似函數的物件並將其綁定到模型。讓我們來看看如何採用以下 `Pydantic` 函數 schema 並獲得不同的模型來呼叫它：

```python
from langchain_core.pydantic_v1 import BaseModel, Field


# Note that the docstrings here are crucial, as they will be passed along
# to the model along with the class name.
class Multiply(BaseModel):
    """Multiply two integers together."""

    a: int = Field(..., description="First integer")
    b: int = Field(..., description="Second integer")
```

設定依賴項和 API 金鑰：

```bash
%pip install -qU langchain-openai
```

```python
os.environ["OPENAI_API_KEY"] = getpass.getpass()
```

我們可以使用 `ChatOpenAI.bind_tools()` 方法來處理將 `Multiply` 轉換為 OpenAI 函數並將其綁定到模型（即每次呼叫模型時將其傳遞）。

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-3.5-turbo-0125", temperature=0)
llm_with_tools = llm.bind_tools([Multiply])
llm_with_tools.invoke("what's 3 * 12")
```

結果:

```
AIMessage(content='', additional_kwargs={'tool_calls': [{'id': 'call_Q8ZQ97Qrj5zalugSkYMGV1Uo', 'function': {'arguments': '{"a":3,"b":12}', 'name': 'Multiply'}, 'type': 'function'}]})
```

我們可以新增一個工具解析器來將工具呼叫從產生的訊息中提取到 JSON：

```python
from langchain_core.output_parsers.openai_tools import JsonOutputToolsParser

tool_chain = llm_with_tools | JsonOutputToolsParser()
tool_chain.invoke("what's 3 * 12")
```

結果:

```
[{'type': 'Multiply', 'args': {'a': 3, 'b': 12}}]
```

或使用原來的 Pydantic 類別：

```python
from langchain_core.output_parsers.openai_tools import PydanticToolsParser

tool_chain = llm_with_tools | PydanticToolsParser(tools=[Multiply])
tool_chain.invoke("what's 3 * 12")
```

結果:

```
[Multiply(a=3, b=12)]
```

如果我們想要強制使用某個工具（並且只使用一次），我們可以設定 `tool_choice` 參數：

```python
llm_with_multiply = llm.bind_tools([Multiply], tool_choice="Multiply")

llm_with_multiply.invoke(
    "make up some numbers if you really want but I'm not forcing you"
)
```

結果:

```
AIMessage(content='', additional_kwargs={'tool_calls': [{'id': 'call_f3DApOzb60iYjTfOhVFhDRMI', 'function': {'arguments': '{"a":5,"b":10}', 'name': 'Multiply'}, 'type': 'function'}]})
```

有關更多信息，請參閱 [ChatOpenAI API 參考](https://api.python.langchain.com/en/latest/chat_models/langchain_openai.chat_models.base.ChatOpenAI.html#langchain_openai.chat_models.base.ChatOpenAI.bind_tools)。

## Defining functions schemas

如果您需要直接存取函數 schema，LangChain 有一個內建轉換器，可以將 Python 函數、Pydantic 類別和 LangChain Tools 轉換為 OpenAI 格式的 JSON schema：

### Python function

```python
import json

from langchain_core.utils.function_calling import convert_to_openai_tool


def multiply(a: int, b: int) -> int:
    """Multiply two integers together.

    Args:
        a: First integer
        b: Second integer
    """
    return a * b


print(json.dumps(convert_to_openai_tool(multiply), indent=2))
```

結果:

```json
{
  "type": "function",
  "function": {
    "name": "multiply",
    "description": "Multiply two integers together.",
    "parameters": {
      "type": "object",
      "properties": {
        "a": {
          "type": "integer",
          "description": "First integer"
        },
        "b": {
          "type": "integer",
          "description": "Second integer"
        }
      },
      "required": [
        "a",
        "b"
      ]
    }
  }
}
```

### Pydantic class

```python
from langchain_core.pydantic_v1 import BaseModel, Field


class multiply(BaseModel):
    """Multiply two integers together."""

    a: int = Field(..., description="First integer")
    b: int = Field(..., description="Second integer")


print(json.dumps(convert_to_openai_tool(multiply), indent=2))
```

結果:

```json
{
  "type": "function",
  "function": {
    "name": "multiply",
    "description": "Multiply two integers together.",
    "parameters": {
      "type": "object",
      "properties": {
        "a": {
          "description": "First integer",
          "type": "integer"
        },
        "b": {
          "description": "Second integer",
          "type": "integer"
        }
      },
      "required": [
        "a",
        "b"
      ]
    }
  }
}
```

### LangChain Tool

```python
from typing import Any, Type

from langchain_core.tools import BaseTool


class MultiplySchema(BaseModel):
    """Multiply tool schema."""

    a: int = Field(..., description="First integer")
    b: int = Field(..., description="Second integer")


class Multiply(BaseTool):
    args_schema: Type[BaseModel] = MultiplySchema
    name: str = "multiply"
    description: str = "Multiply two integers together."

    def _run(self, a: int, b: int, **kwargs: Any) -> Any:
        return a * b


# Note: we're passing in a Multiply object not the class itself.
print(json.dumps(convert_to_openai_tool(Multiply()), indent=2))
```

結果:

```json
{
  "type": "function",
  "function": {
    "name": "multiply",
    "description": "Multiply two integers together.",
    "parameters": {
      "type": "object",
      "properties": {
        "a": {
          "description": "First integer",
          "type": "integer"
        },
        "b": {
          "description": "Second integer",
          "type": "integer"
        }
      },
      "required": [
        "a",
        "b"
      ]
    }
  }
}
```

