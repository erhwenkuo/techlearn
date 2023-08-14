# 介紹

**LangChain** 是一個用於開發由語言模型支持的應用程序的框架。它支持以下應用程序：

- **Data-aware**: 將語言模型連接到其他數據源
- **Agentic**: 允許語言模型與其環境交互

LangChain 的價值在下面兩點上特別突出:

1. **Components**: 用於處理語言模型的抽象，以及每個抽象的實現集合。無論您是否使用 LangChain 框架的其餘部分，組件都是模塊化且易於使用的

2. **Off-the-shelf chains**: 用於完成特定更高級別任務的結構化組件組合

現成的 chains 讓您可以輕鬆上手。對於更複雜的應用程序和細緻入微的用例，組件可以輕鬆定制現有鏈或構建新鏈。

## 開始使用

[以下](https://python.langchain.com/docs/get_started/installation.html)是如何安裝 LangChain、設置環境並開始構建。

我們建議您按照我們的[快速入門指南](https://python.langchain.com/docs/get_started/quickstart.html)構建您的第一個 LangChain 應用程序來熟悉該框架。

## 模組

LangChain 為以下模組提供標準的、可擴展的接口和外部集成，按照從最簡單到最複雜的順序列出：

[**Model I/O**](https://python.langchain.com/docs/modules/model_io/)

與語言模型的接口

[**Data connection**](https://python.langchain.com/docs/modules/data_connection/)

與應用程序特定數據的接口

[**Chains**](https://python.langchain.com/docs/modules/chains/)

構建調用序列

[**Agents**](https://python.langchain.com/docs/modules/agents/)

讓 LangChain 根據給定的高級指令選擇使用哪些工具

[**Memory**](https://python.langchain.com/docs/modules/memory/)

在鏈的運行之間保留應用程序狀態

[**Callbacks**](https://python.langchain.com/docs/modules/callbacks/)

記錄並流式傳輸任何鏈的中間步驟


## 範例、生態系統和資源

[**Use cases**](https://python.langchain.com/docs/use_cases/)

常見端到端用例的演練和最佳實踐，例如：

- [聊天機器人](https://python.langchain.com/docs/use_cases/chatbots/)
- [使用來源回答問題](https://python.langchain.com/docs/use_cases/question_answering/)
- [分析結構化數據](https://python.langchain.com/docs/use_cases/tabular.html)
- 以及更多...

[**Guides**](https://python.langchain.com/docs/guides/)

了解使用 LangChain 進行開發的最佳實踐。

[**Ecosystem**](https://python.langchain.com/docs/ecosystem/)

LangChain 是豐富的工具生態系統的一部分，它與我們的框架集成並建立在其之上。查看我們不斷增長的集成和依賴庫列表。

[**Additional resources**](https://python.langchain.com/docs/additional_resources/)

LangChain 社群充滿了多產的開發人員、富有創造力的建設者和出色的教師。查看 [YouTube 教程](https://python.langchain.com/docs/additional_resources/youtube.html)，了解社群人員提供的精彩教程，並查看 [Gallery](https://github.com/kyrolabs/awesome-langchain)，了解由 [KyroLabs](https://kyrolabs.com/) 人員編制的精彩 LangChain 專案列表。

## API 參考

請前往 [reference](https://api.python.langchain.com/) 部分，獲取 LangChain Python 包中所有類別和方法的完整文檔。