# 組件

組件是 KFP 管道的構建積木。一個組件是一個遠程函數定義；它指定**輸入**，在其主體中具有用戶定義的邏輯，並且可以創建**輸出**。當使用輸入參數實例化組件模板時，我們將其稱為 task。

KFP 提供了兩種編寫組件的高級方法：`Python Components` 和 `Container Components`。

`Python Components` 是編寫以純 Python 實現的組件的便捷方式。有兩種特定類型的 Python 組件：`Lightweight Python Components` 和 `Containerized Python Components`。

`Container Components` 允許您使用任意容器來定義組件，從而公開了一種更靈活、更高級的創作方法。對於不是在純 Python 中實現的組件，這是推薦的方法。

`Importer Components` 是 KFP 提供的一種特殊的“預烘焙”組件，當 artifact 0不是由管道中的任務創建時，它允許您將 artifact 導入到管道中。

## 詳細說明

**Lightweight Python Components**

從獨立的 Python 函數創建組件

**Containerized Python Components**

創建具有更複雜依賴關係的 Python 組件

**Container Components**

通過任意容器定義創建組件

**Special Case: Importer Components**

從管道外部導入 artifact

**Additional Functionality**

有關創作 KFP 組件的更多信息