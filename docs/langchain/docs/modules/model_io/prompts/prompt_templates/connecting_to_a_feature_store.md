# 連接到 Feature Store

Featrue store 是傳統機器學習的一個概念，可確保輸入模型的數據是最新的且相關的。有關這方面的更多信息，請參見[此處](https://www.tecton.ai/blog/what-is-a-feature-store/)。

當考慮將 LLM 應用程序投入生產時，這個概念非常重要。為了客製化 LLM 應用，您可能希望將 LLM 與有關特定用戶的最新信息結合起來。Feature store 是保持數據新鮮的好方法，LangChain 提供了一種將數據與 LLM 結合起來的簡單方法。

在此筆記本中，我們將展示如何將提示模板連接到特徵存儲。基本思想是從提示模板內部調用特徵存儲來檢索然後格式化為提示的值。

## Feast

首先，我們將使用流行的開源特徵存儲框架 [Feast](https://github.com/feast-dev/feast)。

這假設您已經有關 LangChain 入門的相關設定步驟。我們將在以 quickstart 示例為基礎，並創建 `LLMChain` 來向特定驅動程序寫入有關其最新統計數據的註釋。

### Load Feast Store

