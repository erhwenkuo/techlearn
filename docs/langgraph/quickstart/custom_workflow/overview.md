# Overview

LangGraph 專為希望建立強大且適應性強的 AI 代理程式的開發者而打造。開發者選擇 LangGraph 的理由如下：

- **Reliability and controllability.** 透過審核檢查(moderation checks)和人工審核(human-in-the-loop )來引導 agent 的操作。 LangGraph 會儲存長期運作工作流程的上下文，確保您的 agent 會按計畫進行。
- **Low-level and extensible.** 建立 custom agents，使用完全描述性的低階原語，擺脫限制客製化的僵化抽象。設計可擴展的多代理系統，每個代理程式都可根據您的用例自訂特定角色。
- **First-class streaming support.** 透過逐個 token 的串流和中間步驟的串流傳輸，LangGraph 讓使用者能夠清楚地即時了解代理推論和操作。

## Learn LangGraph basics

若要熟悉 LangGraph 的關鍵概念和功能，請完成以下 LangGraph 基礎教學系列：

1. [Build a basic chatbot](https://langchain-ai.github.io/langgraph/tutorials/get-started/1-build-basic-chatbot/)
2. [Add tools](https://langchain-ai.github.io/langgraph/tutorials/get-started/2-add-tools/)
3. [Add memory](https://langchain-ai.github.io/langgraph/tutorials/get-started/3-add-memory/)
4. [Add human-in-the-loop controls](https://langchain-ai.github.io/langgraph/tutorials/get-started/4-human-in-the-loop/)
5. [Customize state](https://langchain-ai.github.io/langgraph/tutorials/get-started/5-customize-state/)
6. [Time travel](https://langchain-ai.github.io/langgraph/tutorials/get-started/6-time-travel/)

完成本系列教學後，您將在 LangGraph 中建立一個支援聊天機器人，它可以：

- ✅ 透過搜尋網頁回答常見問題
- ✅ 跨通話維護對話狀態
- ✅ 將複雜查詢路由給人工審核
- ✅ 使用自訂狀態控制其行為
- ✅ 回退並探索其他對話路徑