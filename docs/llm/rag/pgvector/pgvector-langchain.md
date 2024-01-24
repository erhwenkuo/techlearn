# 如何使用 LangChain 中的 pgvector 向量儲存來建立 LLM 應用程式

原文: [How to Build LLM Applications With pgvector Vector Store in LangChain](https://www.timescale.com/blog/how-to-build-llm-applications-with-pgvector-vector-store-in-langchain/)

![](./assets/pgvector-langchain.png)

## 環境啟動並運行

LangChain 是使用大型語言模型 (LLM) 建立應用程式和代理程式的最受歡迎的框架之一。本文介紹如何使用 Python 中的 LangChain 框架建立 LLM 應用程序，使用 PostgreSQL 和 pgvector 作為 OpenAI 嵌入資料的向量資料庫。

我們將使用建立聊天機器人來回答有關 Timescale 部落格中的部落格文章的問題的範例來說明以下概念:

- 如何使用 LangChain 文件轉換器 TextSplitter 準備文件插入 PostgreSQL 和 pgvector。
- 如何使用 OpenAI 嵌入模型從資料建立嵌入並將其插入到 PostgreSQL 和 pgvector 中。
- 如何使用從向量資料庫檢索的嵌入來增強 LLM 的生成。

總而言之：您可以使用 Python、PostgreSQL 和 pgvector 中的 LangChain 框架來建立 LLM 應用程序，用於儲存 OpenAI 嵌入資料。該過程涉及創建嵌入、儲存資料、拆分和載入 CSV 檔案、執行相似性搜尋以及使用檢索增強生成。

對於使用 Python 進行更高級的 LangChain 專案來說，這是一個很好的第一步。例如，為公司文件創建聊天機器人或用於回答上傳的 PDF 中的問題的應用程式。

!!!info
    Jupyter Notebook 和程式碼：您可以在 GitHub 上的 [Timescale Vector Cookbook 儲存庫](https://github.com/timescale/vector-cookbook/?ref=timescale.com)的 Jupyter Notebook 中找到本教學中使用的所有程式碼。我們建議 clone 存儲庫，並在閱讀本教程時執行程式碼單元。


## 設定和配置

- 註冊 OpenAI 開發者帳戶並建立 API 金鑰。請參閱 OpenAI 的開發者平台。
- 安裝 Python。
- 安裝並配置 Python 虛擬環境。
- 使用以下命令安裝此筆記本的要求：

```bash
pip install -r requirements.txt
```

或者，如果您已經安裝了 LangChain，請執行 `pip install --upgrade langchain`。

```python
import os
# Run export OPENAI_API_KEY=sk-YOUR_OPENAI_API_KEY...
# Get openAI api key by reading local .env file
from dotenv import load_dotenv, find_dotenv
_ = load_dotenv(find_dotenv())
OPENAI_API_KEY  = os.environ['OPENAI_API_KEY']
```

接下來，我們需要一種讓 LangChain 與 PostgreSQL 和 pgvector 互動的方式。這是透過從 `langchain.vectorstores` 套件匯入 PGVector 類別來實現的，如下所示。

```python
from langchain.vectorstores.pgvector import PGVector
```

接下來，我們將為 LangChain 建立連接字串以連接到 PostgreSQL 資料庫。

由於 LangChain 使用 [SQLAlchemy](https://www.sqlalchemy.org/?ref=timescale.com) 連接到 PostgreSQL 等 SQL 資料庫，因此我們需要以程式設計方式建立連接字串，從環境變數中讀取字串的每個組成部分（主機、資料庫名稱、密碼、連接埠等）。

在此範例中，我們將使用安裝了 pgvector 的 PostgreSQL 資料庫。您可以透過下面的 Docker Compose 來建立自己的 PostgreSQL 資料庫並繼續操作。

首先，我們需要建立一個包含必要服務的 `docker-compose.yml` 檔案。建立名為 `docker-compose.yml` 的檔案並新增以下內容：

```docker
services:
  db:
    container_name: pg_container
    hostname: db
    image: ankane/pgvector
    ports:
     - 5432:5432
    restart: always
    environment:
      - POSTGRES_DB=vectordb
      - POSTGRES_USER=testuser
      - POSTGRES_PASSWORD=testpwd
      - POSTGRES_HOST_AUTH_METHOD=trust
    volumes:
     - ./init.sql:/docker-entrypoint-initdb.d/init.sql
  pgadmin:
    container_name: pgadmin4_container
    image: dpage/pgadmin4
    restart: always
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@admin.com
      PGADMIN_DEFAULT_PASSWORD: root
    ports:
      - 5050:80
```

在這個 `docker-compose.yml` 檔案中，我們定義了一個名為 `db` 的服務，它是基於 `ankane/pgvector` Docker 映像。該服務公開連接埠 `5432` 與資料庫交互，並為資料庫名稱、使用者、密碼和身份驗證方法設定環境變數。此外，我們將 `init.sql` 檔案掛載到容器內的 `/docker-entrypoint-initdb.d` 目錄中以進行初始化。

接下來，建立一個名為 `init.sql` 的檔案並使用以下 SQL 語句填入該檔案：

```sql title="init.sql"
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE IF NOT EXISTS embeddings (
  id SERIAL PRIMARY KEY,
  embedding vector,
  text text,
  created_at timestamptz DEFAULT now()
);
```

在此 `init.sql` 腳本中，我們首先建立 `vector` 擴充（如果尚不存在）。然後，我們建立一個名為 `embeddings` 的表，其中包含 `id`、`embedding`、一些 `text` 資料和`created_at` 時間戳記的列。

現在我們已經準備好了 `docker-compose.yml` 檔案和 `init.sql` 腳本，我們可以繼續設定 PostgreSQL 資料庫。在儲存庫目錄中執行以下命令來設定資料庫：

```bash
docker compose up -d
```

此命令將根據 `docker-compose.yml` 檔案中的規格建立一個 Docker 容器，其中包含已安裝和設定的 PostgreSQL 伺服器和 **pgvector** 擴充功能。

```python
# Build the PGVector Connection String from params
# Found in the credential cheat-sheet or "Connection Info" in the Timescale console
# In terminal, run: export VAR_NAME=value for each of the values below
host= os.environ['PG_HOST']
port= os.environ['PG_PORT']
user= os.environ['PG_USER']
password= os.environ['PG_PASSWORD']
dbname= os.environ['PG_DBNAME']

# We use postgresql rather than postgres in the conn string since LangChain uses sqlalchemy under the hood
# You can add the ?sslmode=require if you have a local PostgreSQL instance running with SSL
CONNECTION_STRING = f"postgresql+psycopg2://{user}:{password}@{host}:{port}/{dbname}"
```

## Part 1: 使用 LangChain 將 CSV 檔案分割成更小的區塊，同時保留關聯的元資料

