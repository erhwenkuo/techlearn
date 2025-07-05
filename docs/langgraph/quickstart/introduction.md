![](./assets/langgraph_dark.svg)

LangGraph 是一個用於建置、管理和部署長期運作、有狀態代理的 low-level 編排框架。

## Get started

安裝 LangGraph：

```bash
pip install -U langgraph  langchain-openai
```

然後，使用預先建置的元件建立 agent：

```python
# pip install -qU "langchain-openai" to call the model

from langgraph.prebuilt import create_react_agent

def get_weather(city: str) -> str:
    """Get weather for a given city."""
    return f"It's always sunny in {city}!"

agent = create_react_agent(
    model="gpt-4.1-mini",
    tools=[get_weather],
    prompt="You are a helpful assistant"
)

# Run the agent
agent.invoke(
    {"messages": [{"role": "user", "content": "what is the weather in sf"}]}
)
```

更多信息，請參閱[快速入門](https://langchain-ai.github.io/langgraph/agents/agents/)。或者，若要了解如何建立具有可自訂架構、長期記憶和其他複雜任務處理的[代理程式工作流程](https://langchain-ai.github.io/langgraph/concepts/low_level/)，請參閱 [LangGraph 基礎教學](https://langchain-ai.github.io/langgraph/tutorials/get-started/1-build-basic-chatbot/)。


## Core benefits

LangGraph 為任何長期運作、有狀態的工作流程或代理提供底層支援基礎架構。 LangGraph 不會抽象化提示或架構，並提供以下核心優勢：

- [Durable execution](https://langchain-ai.github.io/langgraph/concepts/durable_execution/): 建置能夠經得起故障考驗並長時間運行的代理，並自動從中斷點恢復運行。
- [Human-in-the-loop](https://langchain-ai.github.io/langgraph/concepts/human_in_the_loop/): 透過在執行過程中的任何時間點檢查和修改代理狀態，無縫融入人工監督。
- [Comprehensive memory](https://langchain-ai.github.io/langgraph/concepts/memory/): 創建真正有狀態的代理，既具有用於持續推理的短期工作記憶，又具有跨會話的長期持久記憶。
- [Debugging with LangSmith](http://www.langchain.com/langsmith?_gl=1*3wafuw*_gcl_au*MTMzOTIyOTUzNS4xNzUwNTUxODU5*_ga*MTM5NTM1ODE0Mi4xNzQ3MDkyMTE0*_ga_47WX3HKKY2*czE3NTEwNzA5MDEkbzkkZzEkdDE3NTEwNzA5MjMkajM4JGwwJGgw): 使用視覺化工具深入了解複雜的代理行為，這些工具可以追蹤執行路徑、捕獲狀態轉換並提供詳細的運行時指標。
- [Production-ready deployment](https://langchain-ai.github.io/langgraph/concepts/deployment_options/): 使用可擴展的基礎架構，自信地部署複雜的代理系統，該基礎架構旨在應對有狀態、長時間運行的工作流程的獨特挑戰。

## LangGraph’s ecosystem

LangGraph 可以獨立使用，也可以與任何 LangChain 產品無縫集成，為開發者提供建立代理的全套工具。為了提升你的 LLM 應用程式開發效率，請將 LangGraph 與以下產品搭配使用：

- [LangSmith](http://www.langchain.com/langsmith?_gl=1*h07k7y*_gcl_au*MTMzOTIyOTUzNS4xNzUwNTUxODU5*_ga*MTM5NTM1ODE0Mi4xNzQ3MDkyMTE0*_ga_47WX3HKKY2*czE3NTEwNzA5MDEkbzkkZzEkdDE3NTEwNzA5MjMkajM4JGwwJGgw): 有助於代理評估和可觀察性。調試性能不佳的 LLM 應用運行，評估代理軌跡，獲得生產環境中的可見性，並逐步提升性能。
- [LangGraph Platform](https://langchain-ai.github.io/langgraph/concepts/#langgraph-platform): 使用專為長期運行、有狀態工作流程建置的部署平台，輕鬆部署和擴展代理程式。在團隊之間發現、重複使用、配置和共享代理，並使用 LangGraph Studio 中的視覺化原型設計快速迭代。
- [LangChain](https://python.langchain.com/docs/introduction/?_gl=1*18vttjc*_gcl_au*MTMzOTIyOTUzNS4xNzUwNTUxODU5*_ga*MTM5NTM1ODE0Mi4xNzQ3MDkyMTE0*_ga_47WX3HKKY2*czE3NTEwNzA5MDEkbzkkZzEkdDE3NTEwNzA5MjMkajM4JGwwJGgw): 提供整合和可組合組件以簡化 LLM 應用程式開發。

