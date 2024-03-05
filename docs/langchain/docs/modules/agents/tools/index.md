# Tools

Tool 是 agent 可以用來與外部世界互動的介面。它們結合了一些東西：

1. 工具名稱
2. 該工具是什麼的描述
3. 描述工具輸入參數內容的 JSON schema
4. 要呼叫的函數
5. 工具的結果是否應直接回傳給用戶

擁有所有這些資訊非常有用，因為這些資訊可用於建立採取行動的系統！名稱、描述和 JSON schema 可以用來給 LLM 模型一些提示 ，以便它知道如何指定要採取的操作，然後指出要呼叫的函數。

工具的輸入越簡單，LLM 就越容易使用它。許多 agent 只能使用具有單一字串輸入的工具。有關代理類型清單以及哪些代理類型可處理更複雜的輸入，請參閱此[文檔](https://python.langchain.com/docs/modules/agents/agent_types)。

重要的是，名稱、描述和 JSON schema 都會在構建提示(prompt)中被使用。因此，{==清晰並準確地描述如何使用該工具非常重要。如果 LLM 不了解如何使用該工具，您可能需要變更預設名稱、描述或 JSON schema==}。

## Default Tools

讓我們看看如何使用工具。為此，我們將使用 Langchain 內建工具。

```python
from langchain_community.tools import WikipediaQueryRun
from langchain_community.utilities import WikipediaAPIWrapper
```

現在我們初始化該工具。這是我們可以根據需要進行配置的地方:

```python
api_wrapper = WikipediaAPIWrapper(top_k_results=1, doc_content_chars_max=100)

tool = WikipediaQueryRun(api_wrapper=api_wrapper)
```

讓我們來看一下這個工具的名稱

```python
print(tool.name)
```

結果:

```bash
Wikipedia
```

這是預設的工具描述

```python
print(tool.description)
```

結果:

```bash
A wrapper around Wikipedia. Useful for when you need to answer general questions about people, places, companies, facts, historical events, or other subjects. Input should be a search query.
```

接下來讓我們來看一下這個工具要被呼叫時的輸入(input)的 JSON schema:

```python
print(tool.args)
```

結果:

```bash
{'query': {'title': 'Query', 'type': 'string'}}
```

我們可以看看該工具是否應該直接返回給用戶

```python
print(tool.return_direct)
```

結果:

```bash
False
```

我們可以用字典輸入來呼叫這個工具

```python
tool.run({"query": "langchain"})
```

結果:

```bash
Page: LangChain\nSummary: LangChain is a framework designed to simplify the creation of applications
```

## Customizing Default Tools

我們還可以修改內建名稱、描述和參數的 JSON schema。

定義參數的 JSON schema 時，輸入與函數保持相同非常重要，因此您不應更改它。但您可以輕鬆地為每個輸入定義自訂描述。

```python
from langchain_core.pydantic_v1 import BaseModel, Field


class WikiInputs(BaseModel):
    """Inputs to the wikipedia tool."""

    query: str = Field(description="query to look up in Wikipedia, should be 3 or less words")


tool = WikipediaQueryRun(
    name="wiki-tool",
    description="look up things in wikipedia",
    args_schema=WikiInputs,
    api_wrapper=api_wrapper,
    return_direct=True,
)
```

讓我們來看一下這個工具的名稱

```python
print(tool.name)
```

結果:

```bash
wiki-tool
```

工具描述

```python
print(tool.description)
```

結果:

```bash
look up things in wikipedia
```

接下來讓我們來看一下這個工具要被呼叫時的輸入(input)的 JSON schema:

```python
print(tool.args)
```

結果:

```bash
{
  'query': {'title': 'Query',
  'description': 'query to look up in Wikipedia, should be 3 or less words',
  'type': 'string'}
}
```

我們可以看看該工具是否應該直接返回給用戶

```python
print(tool.return_direct)
```

結果:

```bash
True
```

呼叫這個工具

```python
tool.run("langchain")
```

結果:

```bash
Page: LangChain\nSummary: LangChain is a framework designed to simplify the creation of applications
```

## 更多主題

這是對 LangChain tools 的快速介紹，但還有很多東西需要學習

- **Built-In Tools** 有關所有內建工具的列表，請參閱此[頁面](https://python.langchain.com/docs/integrations/tools/)
- **Custom Tools** 儘管內建工具很有用，但您很可能必須定義自己的工具。請參閱[本指南](https://python.langchain.com/docs/modules/agents/tools/custom_tools)以取得有關如何定義自己的工具的說明。
- **Toolkits** 工具包是可以很好地協同工作的工具的集合。有關更深入的描述以及所有內建工具包的列表，請參閱此[頁面](https://python.langchain.com/docs/modules/agents/tools/toolkits)
- **Tools as OpenAI Functions** 工具與 OpenAI Function calling 的定義非常相似，並且可以輕鬆轉換為該格式。[請參閱此筆記本](https://python.langchain.com/docs/modules/agents/tools/tools_as_openai_functions)以取得有關如何執行此操作的說明。