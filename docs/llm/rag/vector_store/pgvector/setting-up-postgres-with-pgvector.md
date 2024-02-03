# PostgreSQL 資料庫 pgvector 擴展實作

原文: [Implementing the pgvector extension for a PostgreSQL database](https://medium.com/@johannes.ocean/setting-up-a-postgres-database-with-the-pgvector-extension-10ab7ff212cc)

![](./assets/pgvector.png)

在本文中，我們將探討如何使用 **pgvector** 擴充功能設定 PostgreSQL 資料庫並利用它來執行向量化搜尋。

首先，我們將使用一個名為 [pgvector-demo](https://github.com/johannesocean/pgvector-demo) 的示範儲存庫，它展示了向量化搜尋(vector search)的實作。您可以在 GitHub 上找到該儲存庫並將其複製到本機電腦 (https://github.com/johannesocean/pgvector-demo)。

```bash
pgvector-demo/
├── app
│   ├── db
│   │   ├── config.py
│   │   ├── connect.py
│   │   ├── data
│   │   │   ├── add_test_data.py
│   │   │   └── __init__.py
│   │   └── __init__.py
│   ├── __init__.py
│   └── test_search.py
├── database.ini
├── docker-compose.yml
├── init.sql
├── README.md
└── requirements.txt
```

## 啟動運行環境

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

## 新增 Embedding 至數據庫

**環境變數設定**

在　`pgvector-demo/` 的目錄下修改 `.example.env` 檔案成為 `.env`:

``` title=".env"
OPENAI_API_KEY=sk-******************
EMBEDDING_MODEL=text-embedding-ada-002
```

- OPENAI_API_KEY: 從 OpenAI 取得的有效 API Key
- EMBEDDING_MODEL: 定義要使用的 OpenAI 的模型

**安裝相關Python套件**

在　`pgvector-demo/` 的目錄下執行下列命令:

```bash
pip install -r requirements.txt
```

**轉換文本成 Embedding 並新增至數據庫**

一旦資料庫容器啟動並運行，我們就可以開始向其中插入資料。這個儲存庫提供了一個名為 `app/db/data/add_test_data.py` 的 Python 腳本，您可以執行該腳本將一些範例資料插入到資料庫中。

```python title="app/db/data/add_test_data.py"
# add_test_data.py
# 編寫五個將轉換為 embedding 的例句
texts = [
    "I like to eat broccoli and bananas.",
    "I ate a banana and spinach smoothie for breakfast.",
    "Chinchillas and kittens are cute.",
    "My sister adopted a kitten yesterday.",
    "Look at this cute hamster munching on a piece of broccoli.",
]

embeddings = get_embeddings(texts, os.getenv('EMBEDDING_MODEL'))

# 將文字和 embedding 寫入資料庫
connection = create_db_connection()
cursor = connection.cursor()
try:
    for text, embedding in zip(texts, embeddings):
        cursor.execute(
            "INSERT INTO embeddings (embedding, text) VALUES (%s, %s)",
            (embedding, text)
        )
    connection.commit()
except (Exception, psycopg2.Error) as error:
    print("Error while writing to DB", error)
finally:
    if cursor:
        cursor.close()
    if connection:
        connection.close()
```

## 向量搜尋

插入資料後，我們可以繼續測試向量化搜尋(vectorized search)。該儲存庫包含另一個名為 `app/test_search.py`​​ 的 Python 腳本，它允許您執行向量化搜尋。

```python title="app/test_search.py"
# test_search.py
text = "Did anyone adopt a cat this weekend?"
embedding = get_embedding(text, os.getenv('EMBEDDING_MODEL'))

connection = create_db_connection()
cursor = connection.cursor()
try:
    cursor.execute(f"""
        SELECT text,  1 - (embedding <=> '{embedding}') AS cosine_similarity
        FROM embeddings
        ORDER BY cosine_similarity desc
        LIMIT 3
    """)
    for r in cursor.fetchall():
        print(f"Text: {r[0]}; Similarity: {r[1]}")

except Exception as error:
    print("Error..", error)
finally:
    cursor.close()
    connection.close()
```

結果:

```bash
# PRINTS
Text: My sister adopted a kitten yesterday.; Similarity: 0.876654677191657
Text: Chinchillas and kittens are cute.; Similarity: 0.7983014583587703
Text: Look at this cute hamster munching on a piece of broccoli.; Similarity: 0.7650997902720689
```

這個腳本示範如何利用 PostgreSQL 中的 **pgvector** 擴充功能來執行向量化搜尋。這是建立您自己的向量化搜尋功能的一個很好的起點。

## 結論

實現向量化搜尋可以顯著提高各種應用程式中搜尋的效能和準確性，例如推薦系統、相似性匹配等。透過利用 PostgreSQL 和 **pgvector** 擴展，您可以有效地處理向量化資料並執行高速搜尋。

透過本文概述的步驟，您現在已經了解如何使用 **pgvector** 擴充功能設定 PostgreSQL 資料庫。請隨意進一步探索提供的演示存儲庫並對其進行自訂以滿足您的特定需求。