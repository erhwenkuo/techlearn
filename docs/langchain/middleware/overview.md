# Overview

控制和自訂 agent 執行的每一步。

Middleware 提供了一種更嚴格地控制 agent 內部運作機制的方法。Middleware 的用途包括：

- 透過日誌記錄、分析和調試追蹤 agent 行為。
- 調整提示訊息、工具選擇和輸出格式。
- 新增重試、回退和提前終止邏輯。
- 應用速率限制、防護措施和個人識別資訊 (PII) 檢測。

透過向 `create_agent()` 傳遞 middleware 實例來新增 middleware 在 agent 的運作：

```python
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware, HumanInTheLoopMiddleware

agent = create_agent(
    model="gpt-4o",
    tools=[...],
    middleware=[
        SummarizationMiddleware(...),
        HumanInTheLoopMiddleware(...)
    ],
)
```

## The agent loop

核心代理循環包括呼叫模型，讓模型選擇要執行的工具，然後在不再需要呼叫任何工具時結束：

![](./assets/core_agent_loop.avif)

中間件在每個步驟之前和之後都會暴露 hook：

![](./assets/middleware_final.avif)

