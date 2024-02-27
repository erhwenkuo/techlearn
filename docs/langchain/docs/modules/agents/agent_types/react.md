# ReAct

本演練展示如何使用 agent 來實作 ReAct 邏輯。

```python
from langchain import hub
from langchain.agents import AgentExecutor, create_react_agent
from langchain_community.tools.tavily_search import TavilySearchResults
from langchain_openai import OpenAI
```

## Initialize tools

讓我們載入一些工具來使用。

```python
tools = [TavilySearchResults(max_results=1)]
```

## Create Agent

取得要使用的 prompt:

```python
# Get the prompt to use - you can modify this!
prompt = hub.pull("hwchase17/react")
```

或是自己構建 prompt:

```python
prompt = PromptTemplate.from_template('''
Answer the following questions as best you can. You have access to the following tools:

{tools}hwchase17/react-chat

Use the following format:

Question: the input question you must answer
Thought: you should always think about what to do
Action: the action to take, should be one of [{tool_names}]
Action Input: the input to the action
Observation: the result of the action
... (this Thought/Action/Action Input/Observation can repeat N times)
Thought: I now know the final answer
Final Answer: the final answer to the original input question

Begin!

Question: {input}
Thought:{agent_scratchpad}
''')

!!! info
    **ReAct 提示**任務任務

    Yao等人，於 2022 引入了一個框架，其中 LLMs 以交錯的方式產生 {==推理軌跡==} 和 {==特定任務==}操作 。

    產生推理軌跡使模型能夠誘導、追蹤和更新任務操作計劃，甚至處理異常情況。操作步驟允許與外部來源（如知識庫或環境）進行互動並且收集資訊。

    ReAct 框架允許 LLMs 與外部工具互動來獲取額外信息，從而給出更可靠和實際的回應。

    結果表明，ReAct 可以在語言和決策任務上的表現高於幾個最先進水準要求的的基線。 ReAct 也提高了 LLMs 的人類可解釋性和可信度。總的來說，作者發現了將 ReAct 和鍊式思考 (CoT) 結合使用的最佳方法是在推理過程中同時使用內部知識和獲取到的外部資訊。

    參考: [ReAct Prompting](https://www.promptingguide.ai/techniques/react)


```python
# Choose the LLM to use
llm = OpenAI()

# Construct the ReAct agent
agent = create_react_agent(llm, tools, prompt)
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
 I should research LangChain to learn more about it.
Action: tavily_search_results_json
Action Input: "LangChain"[{'url': 'https://www.ibm.com/topics/langchain', 'content': 'LangChain is essentially a library of abstractions for Python and Javascript, representing common steps and concepts  LangChain is an open source orchestration framework for the development of applications using large language models  other LangChain features, like the eponymous chains.  LangChain provides integrations for over 25 different embedding methods, as well as for over 50 different vector storesLangChain is a tool for building applications using large language models (LLMs) like chatbots and virtual agents. It simplifies the process of programming and integration with external data sources and software workflows. It supports Python and Javascript languages and supports various LLM providers, including OpenAI, Google, and IBM.'}] I should read the summary and look at the different features and integrations of LangChain.
Action: tavily_search_results_json
Action Input: "LangChain features and integrations"[{'url': 'https://www.ibm.com/topics/langchain', 'content': "LangChain provides integrations for over 25 different embedding methods, as well as for over 50 different vector stores  LangChain is an open source orchestration framework for the development of applications using large language models  other LangChain features, like the eponymous chains.  LangChain is essentially a library of abstractions for Python and Javascript, representing common steps and conceptsLaunched by Harrison Chase in October 2022, LangChain enjoyed a meteoric rise to prominence: as of June 2023, it was the single fastest-growing open source project on Github. 1 Coinciding with the momentous launch of OpenAI's ChatGPT the following month, LangChain has played a significant role in making generative AI more accessible to enthusias..."}] I should take note of the launch date and popularity of LangChain.
Action: tavily_search_results_json
Action Input: "LangChain launch date and popularity"[{'url': 'https://www.ibm.com/topics/langchain', 'content': "LangChain is an open source orchestration framework for the development of applications using large language models  other LangChain features, like the eponymous chains.  LangChain provides integrations for over 25 different embedding methods, as well as for over 50 different vector stores  LangChain is essentially a library of abstractions for Python and Javascript, representing common steps and conceptsLaunched by Harrison Chase in October 2022, LangChain enjoyed a meteoric rise to prominence: as of June 2023, it was the single fastest-growing open source project on Github. 1 Coinciding with the momentous launch of OpenAI's ChatGPT the following month, LangChain has played a significant role in making generative AI more accessible to enthusias..."}] I now know the final answer.
Final Answer: LangChain is an open source orchestration framework for building applications using large language models (LLMs) like chatbots and virtual agents. It was launched by Harrison Chase in October 2022 and has gained popularity as the fastest-growing open source project on Github in June 2023.

> Finished chain.
```

最後的回答:

```bash
{'input': 'what is LangChain?',
 'output': 'LangChain is an open source orchestration framework for building applications using large language models (LLMs) like chatbots and virtual agents. It was launched by Harrison Chase in October 2022 and has gained popularity as the fastest-growing open source project on Github in June 2023.'}
```

## Using with chat history

當使用 chat history 時，我們需要一個考慮到這一點的提示

取得要使用的 prompt:

```python
# Get the prompt to use - you can modify this!
prompt = hub.pull("hwchase17/react-chat")
```

參考: [hwchase17/react-chat](https://smith.langchain.com/hub/hwchase17/react-chat)



或是自己構建 prompt:

```python
prompt = PromptTemplate.from_template('''
Assistant is a large language model trained by OpenAI.

Assistant is designed to be able to assist with a wide range of tasks, from answering simple questions to providing in-depth explanations and discussions on a wide range of topics. As a language model, Assistant is able to generate human-like text based on the input it receives, allowing it to engage in natural-sounding conversations and provide responses that are coherent and relevant to the topic at hand.

Assistant is constantly learning and improving, and its capabilities are constantly evolving. It is able to process and understand large amounts of text, and can use this knowledge to provide accurate and informative responses to a wide range of questions. Additionally, Assistant is able to generate its own text based on the input it receives, allowing it to engage in discussions and provide explanations and descriptions on a wide range of topics.

Overall, Assistant is a powerful tool that can help with a wide range of tasks and provide valuable insights and information on a wide range of topics. Whether you need help with a specific question or just want to have a conversation about a particular topic, Assistant is here to assist.

TOOLS:
------

Assistant has access to the following tools:

{tools}

To use a tool, please use the following format:

```
Thought: Do I need to use a tool? Yes
Action: the action to take, should be one of [{tool_names}]
Action Input: the input to the action
Observation: the result of the action
```

When you have a response to say to the Human, or if you do not need to use a tool, you MUST use the format:

```
Thought: Do I need to use a tool? No
Final Answer: [your response here]
```

Begin!

Previous conversation history:
{chat_history}

New input: {input}
{agent_scratchpad}
''')
```

構建 agent 并執行:

```python
# Construct the ReAct agent
agent = create_react_agent(llm, tools, prompt)

agent_executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

agent_executor.invoke(
    {
        "input": "what's my name?",
        # Notice that chat_history is a string, since this prompt is aimed at LLMs, not chat models
        "chat_history": "Human: Hi! My name is Bob\nAI: Hello Bob! Nice to meet you",
    }
)
```

執行過程:

```bash


> Entering new AgentExecutor chain...
Thought: Do I need to use a tool? No
Final Answer: Your name is Bob.

> Finished chain.
```

最後結果:

```bash
{'input': "what's my name?",
 'chat_history': 'Human: Hi! My name is Bob\nAI: Hello Bob! Nice to meet you',
 'output': 'Your name is Bob.'}
```




