# Tools 轉換成 OpenAI Functions

本文介紹如何使用 LangChain tools 轉換成為 OpenAI function calling 的參數。

```bash
pip install -qU langchain-community langchain-openai
```

```python
from langchain_community.tools import MoveFileTool
from langchain_core.messages import HumanMessage
from langchain_core.utils.function_calling import convert_to_openai_function
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-3.5-turbo")

# 設定要使用的 tool
tools = [MoveFileTool()]

# 使用　convert_to_openai_function　方法來轉換成　OpenAI function calling 的參數
functions = [convert_to_openai_function(t) for t in tools]

# 打印出來檢查
print(functions[0])
```

結果:

```bash
{'name': 'move_file',
 'description': 'Move or rename a file from one location to another',
 'parameters': {'type': 'object',
  'properties': {'source_path': {'description': 'Path of the file to move',
    'type': 'string'},
   'destination_path': {'description': 'New path for the moved file',
    'type': 'string'}},
  'required': ['source_path', 'destination_path']}}
```

呼叫 OpenAI chat-completion 的 API:

```python
message = model.invoke(
    [HumanMessage(content="move file foo to bar")], functions=functions
)

# 打印出來檢查
print(message)
```

結果:

```bash
AIMessage(content='', additional_kwargs={'function_call': {'arguments': '{\n  "source_path": "foo",\n  "destination_path": "bar"\n}', 'name': 'move_file'}})
```

 !!! tip
    如果有使用 Function calling 的參數時, OpenAI 的模型會根據使用者的 input 來決定需不需要使用 tools / functions 來取得額外的資訊。

    如果需要的話, 在 API 的 response 物件中的訊息 content 是空的, 而 `additional_kwargs` 會有要呼叫的 function 名稱與呼叫用的參數。

借助 OpenAI chat models，我們也可以使用 `bind_functions` 自動綁定和轉換類似函數的物件。

```python
model_with_functions = model.bind_functions(tools)

message = model_with_functions.invoke([HumanMessage(content="move file foo to bar")])

# 打印出來檢查
print(message)
```

結果:

```bash
AIMessage(content='', additional_kwargs={'function_call': {'arguments': '{\n  "source_path": "foo",\n  "destination_path": "bar"\n}', 'name': 'move_file'}})
```

或者我們可以透過使用更新的 OpenAI API 的 巷`ChatOpenAI.bind_tools` ，該 API 使用 `tools` 和 `tool_choice` 而不是 `functions` 和 `function_call`：

```python
model_with_tools = model.bind_tools(tools)

message = model_with_tools.invoke([HumanMessage(content="move file foo to bar")])

# 打印出來檢查
print(message)
```

結果:

```bash
AIMessage(content='', additional_kwargs={'tool_calls': [{'id': 'call_btkY3xV71cEVAOHnNa5qwo44', 'function': {'arguments': '{\n  "source_path": "foo",\n  "destination_path": "bar"\n}', 'name': 'move_file'}, 'type': 'function'}]})
```

