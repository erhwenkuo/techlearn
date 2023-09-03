# SQL

## 使用案例

企業數據通常存儲在 SQL 數據庫中。

LLM 讓使用自然語言與 SQL 數據庫交互成為可能。

LangChain 提供 **SQL Chains** 和 **Agents** 基於自然語言提示構建和運行 SQL 查詢。

它們與 SQLAlchemy 支持的任何 SQL dialect 兼容（例如 MySQL、PostgreSQL、Oracle SQL、Databricks、SQLite）。

它們支持以下用例：

- 生成將根據自然語言問題運行的查詢(query)
- 創建可以根據數據庫數據回答問題的聊天機器人
- 根據用戶想要分析的見解構建自定義儀表板

## 概述

LangChain 提供了與 SQL 數據庫交互的工具：

1. 基於自然語言用戶問題構建 SQL 查詢
2. 使用鏈來查詢 SQL 數據庫以創建和執行查詢
3. 使用代理與 SQL 數據庫交互，以實現健壯且靈活的查詢

![](./assets/sql_usecase.png)

## Quickstart

首先，獲取所需的套件並設置環境變數：

```bash
pip install langchain langchain-experimental openai

# Set env var OPENAI_API_KEY or load from a .env file
# import dotenv

# dotenv.load_env()
```

下面的範例將使用 SQLite 與 [Chinook 數據庫](https://github.com/lerocha/chinook-database)連接。

按照[安裝步驟](https://database.guide/2-sample-databases-sqlite/)在與此筆記本相同的目錄中創建 `Chinook.db`：

- 將[此文件](https://raw.githubusercontent.com/lerocha/chinook-database/master/ChinookDatabase/DataSources/Chinook_Sqlite.sql)保存到目錄 `Chinook_Sqlite.sql`
- 在 terminal 下運行 `sqlite3 Chinook.db`
- 接著在 sqllite3 的 prompt 介面中運行 `.read Chinook_Sqlite.sql`
- 接著就可直接來測試 `SELECT * FROM Artist LIMIT 10;`