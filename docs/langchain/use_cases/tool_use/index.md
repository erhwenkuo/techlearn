# Tool use

LLM 的一個令人興奮的用例是為其他 tool 建立自然語言接口，無論是 API、函數、資料庫等。LangChain 非常適合建立此類接口，因為它具有：

- 良好的模型輸出解析，可以輕鬆從模型輸出中提取 JSON、XML、OpenAI 函數呼叫等。
- 大量內建 [tools](https://python.langchain.com/docs/integrations/tools)。
- 為您如何呼叫這些工具提供了很大的靈活性。



工具的使用方式主要有兩種：[chains](https://python.langchain.com/docs/modules/chains) 和 [agents](https://python.langchain.com/docs/modules/agents/)。

Chains 允許您建立預先定義的工具使用順序。

![](./assets/tool_chain.svg)


Agents 讓模型循環使用工具，這樣它就可以決定使用工具的次數。

![](./assets/tool_agent.svg)

