# OpenAI functions

某些 OpenAI 模型（如 gpt-3.5-turbo-0613 和 gpt-4-0613）已經過微調，可以檢測何時應調用函數並回應應傳遞給函數的輸入。在 API 呼叫中，您可以描述函數並讓模型智慧地選擇輸出包含呼叫這些函數的參數的 JSON 物件。 OpenAI 函數 API 的目標是比通用文字完成或聊天 API 更可靠地傳回有效且有用的函數呼叫。

許多開源模型都採用了相同的函數呼叫格式，並且還對模型進行了微調以檢測何時應該呼叫函數。

OpenAI Functions Agent 旨在與這些模型搭配使用。

安裝 `openai`、`tavily-python` 軟體包，這些軟體包是 LangChain 軟體包在內部調用它們所必需的。

OpenAI API 已棄用 `functions`，轉而使用 `tools`。兩者之間的差異在於，`tools` API 允許模型請求一次呼叫多個函數，這可以減少某些架構中的回應時間。建議對 OpenAI 模型使用 tools agent。

請參閱以下連結以了解更多資訊：

- [OpenAI chat create](https://platform.openai.com/docs/api-reference/chat/create)
- [OpenAI function calling](https://platform.openai.com/docs/guides/function-calling)

函數(functions)格式仍然與採用它的開源模型和提供者相關，並且該 agent 程式也要搭配這些模型。

```bash
%pip install --upgrade --quiet  langchain-openai tavily-python
```

## Initialize Tools

我們將首先創建一些我們可以使用的工具

```python
from langchain import hub
from langchain.agents import AgentExecutor, create_openai_functions_agent
from langchain_community.tools.tavily_search import TavilySearchResults
from langchain_openai import ChatOpenAI

tools = [TavilySearchResults(max_results=1)]
```

## Create Agent

```python
# Get the prompt to use - you can modify this!
prompt = hub.pull("hwchase17/openai-functions-agent")

print(prompt.messages)
```
# Choose the LLM that will drive the agent
llm = ChatOpenAI(model="gpt-3.5-turbo-1106")

# Construct the OpenAI Functions agent
agent = create_openai_functions_agent(llm, tools, prompt)
結果:

```bash
[SystemMessagePromptTemplate(prompt=PromptTemplate(input_variables=[], template='You are a helpful assistant')),
 MessagesPlaceholder(variable_name='chat_history', optional=True),
 HumanMessagePromptTemplate(prompt=PromptTemplate(input_variables=['input'], template='{input}')),
 MessagesPlaceholder(variable_name='agent_scratchpad')]
```

```python
# Choose the LLM that will drive the agent
llm = ChatOpenAI(model="gpt-3.5-turbo-1106")

# Construct the OpenAI Functions agent
agent = create_openai_functions_agent(llm, tools, prompt)
```

## Run Agent

```python
# Create an agent executor by passing in the agent and tools
agent_executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

agent_executor.invoke({"input": "what is LangChain?"})
```

結果:

```bash


> Entering new AgentExecutor chain...

Invoking: `tavily_search_results_json` with `{'query': 'LangChain'}`


[{'url': 'https://www.ibm.com/topics/langchain', 'content': 'LangChain is essentially a library of abstractions for Python and Javascript, representing common steps and concepts  LangChain is an open source orchestration framework for the development of applications using large language models  other LangChain features, like the eponymous chains.  LangChain provides integrations for over 25 different embedding methods, as well as for over 50 different vector storesLangChain is a tool for building applications using large language models (LLMs) like chatbots and virtual agents. It simplifies the process of programming and integration with external data sources and software workflows. It supports Python and Javascript languages and supports various LLM providers, including OpenAI, Google, and IBM.'}]LangChain is a tool for building applications using large language models (LLMs) like chatbots and virtual agents. It simplifies the process of programming and integration with external data sources and software workflows. LangChain provides integrations for over 25 different embedding methods and for over 50 different vector stores. It is essentially a library of abstractions for Python and JavaScript, representing common steps and concepts. LangChain supports Python and JavaScript languages and various LLM providers, including OpenAI, Google, and IBM. You can find more information about LangChain [here](https://www.ibm.com/topics/langchain).

> Finished chain.
```

```
{'input': 'what is LangChain?',
 'output': 'LangChain is a tool for building applications using large language models (LLMs) like chatbots and virtual agents. It simplifies the process of programming and integration with external data sources and software workflows. LangChain provides integrations for over 25 different embedding methods and for over 50 different vector stores. It is essentially a library of abstractions for Python and JavaScript, representing common steps and concepts. LangChain supports Python and JavaScript languages and various LLM providers, including OpenAI, Google, and IBM. You can find more information about LangChain [here](https://www.ibm.com/topics/langchain).'}
 ```

## Using with chat history

```python
from langchain_core.messages import AIMessage, HumanMessage

agent_executor.invoke(
    {
        "input": "what's my name?",
        "chat_history": [
            HumanMessage(content="hi! my name is bob"),
            AIMessage(content="Hello Bob! How can I assist you today?"),
        ],
    }
)
```

結果:

```bash


> Entering new AgentExecutor chain...
Your name is Bob.

> Finished chain.
```

```bash
{'input': "what's my name?",
 'chat_history': [HumanMessage(content='hi! my name is bob'),
  AIMessage(content='Hello Bob! How can I assist you today?')],
 'output': 'Your name is Bob.'}
```
