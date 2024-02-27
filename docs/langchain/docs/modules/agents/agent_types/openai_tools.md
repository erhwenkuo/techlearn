# OpenAI tools

較新的 OpenAI 模型經過了微調，可以偵測何時應呼叫一個或多個函數，並回應應傳遞給函數的輸入。在 API 呼叫中，您可以描述函數並讓模型智慧地選擇輸出包含呼叫這些函數的參數的 JSON 物件。 OpenAI 工具 API 的目標是比使用通用文字完成或聊天 API 更可靠地傳回有效且有用的函數呼叫。

OpenAI 將呼叫單一函數的能力稱為 functions，將呼叫一個或多個函數的能力稱為 tools。

在 OpenAI Chat API 中，functions 現在被視為遺留選項，已被棄用，取而代之的是 tools。

如果您使用 OpenAI 模型建立 agent，則應該使用此 OpenAI Tools agent 而不是 OpenAI functions agent。

使用 tools 允許模型在適當的時候請求呼叫多個函數。

見:

- [OpenAI chat create](https://platform.openai.com/docs/api-reference/chat/create)
- [OpenAI function calling](https://platform.openai.com/docs/guides/function-calling)

```bash
%pip install --upgrade --quiet  langchain-openai tavily-python
```

```python
from langchain import hub
from langchain.agents import AgentExecutor, create_openai_tools_agent
from langchain_community.tools.tavily_search import TavilySearchResults
from langchain_openai import ChatOpenAI
```

## Initialize Tools

對於這個 agent，我們賦予它使用 Tavilly 搜尋網路的能力。

```python
tools = [TavilySearchResults(max_results=1)]

```

## Create Agent

```python
# Get the prompt to use - you can modify this!
prompt = hub.pull("hwchase17/openai-tools-agent")

# Choose the LLM that will drive the agent
# Only certain models support this
llm = ChatOpenAI(model="gpt-3.5-turbo-1106", temperature=0)

# Construct the OpenAI Tools agent
agent = create_openai_tools_agent(llm, tools, prompt)
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


[{'url': 'https://www.ibm.com/topics/langchain', 'content': 'LangChain is essentially a library of abstractions for Python and Javascript, representing common steps and concepts  LangChain is an open source orchestration framework for the development of applications using large language models  other LangChain features, like the eponymous chains.  LangChain provides integrations for over 25 different embedding methods, as well as for over 50 different vector storesLangChain is a tool for building applications using large language models (LLMs) like chatbots and virtual agents. It simplifies the process of programming and integration with external data sources and software workflows. It supports Python and Javascript languages and supports various LLM providers, including OpenAI, Google, and IBM.'}]LangChain is an open source orchestration framework for the development of applications using large language models. It is essentially a library of abstractions for Python and Javascript, representing common steps and concepts. LangChain simplifies the process of programming and integration with external data sources and software workflows. It supports various large language model providers, including OpenAI, Google, and IBM. You can find more information about LangChain on the IBM website: [LangChain - IBM](https://www.ibm.com/topics/langchain)

> Finished chain.
```

```bash
{'input': 'what is LangChain?',
 'output': 'LangChain is an open source orchestration framework for the development of applications using large language models. It is essentially a library of abstractions for Python and Javascript, representing common steps and concepts. LangChain simplifies the process of programming and integration with external data sources and software workflows. It supports various large language model providers, including OpenAI, Google, and IBM. You can find more information about LangChain on the IBM website: [LangChain - IBM](https://www.ibm.com/topics/langchain)'}
```

## Using with chat history

```python
from langchain_core.messages import AIMessage, HumanMessage

agent_executor.invoke(
    {
        "input": "what's my name? Don't use tools to look this up unless you NEED to",
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
{'input': "what's my name? Don't use tools to look this up unless you NEED to",
 'chat_history': [HumanMessage(content='hi! my name is bob'),
  AIMessage(content='Hello Bob! How can I assist you today?')],
 'output': 'Your name is Bob.'}
```
