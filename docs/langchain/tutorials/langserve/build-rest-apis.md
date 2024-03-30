# 使用 LangServe 為 LangChain 應用程式建立 REST API

原文: [Using LangServe to build REST APIs for LangChain Applications](https://www.koyeb.com/tutorials/using-langserve-to-build-rest-apis-for-langchain-applications)

## 介紹

[LangChain](https://www.langchain.com/) 是一個強大的框架，用於使用 AI 語言模型建立應用程式。它透過簡化提示範本、配置查詢上下文以及將離散流程連結在一起以形成複雜的管道，簡化了與本地或遠端 LLM 互動的過程。

[LangServe](https://www.langchain.com/langserve) 是一個 LangChain 項目，可協助您透過 REST API 建置和交付這些應用程式。在底層，它使用 [FastAPI](https://fastapi.tiangolo.com/) 來建立路由和建置 Web 服務，並利用 [Pydantic](https://docs.pydantic.dev/latest/) 來處理資料驗證。

在本文，我們將示範如何使用 LangChain 和 LangServe 建立應用程式。該應用程式將提供 REST API，用戶可以在其中提交查詢。它將這些資訊以及上下文資訊傳遞給 OpenAI 以產生回應。

## 步驟

要完成本指南並部署 LangServe 應用程序，您需要執行以下步驟：

1. 設定 Project 目錄
2. 建立應用程式目錄結構並安裝依賴項
3. 建立 LangServe 應用程式
4. 測試應用程式
5. 調整 Dockerfile


### 1.設定 Project 目錄

首先，建立並移動到一個專案目錄，該目錄將保存我們將要建立的應用程式:

```bash
mkdir example-langserve
cd example-langserve
```

在專案目錄裡面，建立一個名為 `.env` 的檔案。在內部，透過將 `OPENAI_API_KEY` 環境變數設定為您的 OpenAI API 金鑰:

``` title=".env"
OPENAI_API_KEY="<YOUR_OPENAI_API_KEY>"
```

我們的應用程式將從該檔案中讀取 API 金鑰，以驗證其對 OpenAI 服務的請求。

接下來，為專案創建並啟動新的 Python 虛擬環境。這會將我們專案的依賴項與系統包隔離，以避免衝突並提供更好的可重複性：

```bash
conda create -n example-langserve python=3.11
conda activate example-langserve
```

您的虛擬環境現在應該已啟動。

**匯出Python 虛擬環境到 Conda**

```bash
conda env export | grep -v "^prefix: " > environment.yml
```

### 2.建立應用程式目錄結構並安裝依賴項

現在我們正在虛擬環境中工作，我們可以開始安裝應用程式將使用的套件並設定專案目錄。

標準 Python 安裝預設包含 pip 套件管理器。然而，LangServe 專案預設使用 [poetry](https://python-poetry.org/)。因此，我們將分兩個階段安裝依賴項。

首先，安裝 `langchain-cli` 軟體包來取得 `langchain` 命令列工具。我們也將藉此機會安裝 `poetry` 並確保 `pip` 是最新的:

```bash
pip install -U pip langchain-cli poetry
```

接下來，使用新安裝的 `langchain` 指令，在目前目錄中初始化一個 LangChain 專案:

```bash
langchain app new .
```

!!!info
    注意：請務必包含 `.` 以定位目前目錄。

系統將詢問您是否要安裝任何軟體包。儘管有這樣的措辭，但這個提示實際上指的是 LangChain 模板而不是 Python 套件。按 ENTER 鍵繼續而不新增任何範本。

使用新的專案文件，您的目錄現在應該類似於以下內容：

```bash
.
├── app
│   ├── __init__.py
│   ├── __pycache__
│   │   ├── __init__.cpython-310.pyc
│   │   └── server.cpython-310.pyc
│   └── server.py
├── Dockerfile
├── environment.yml
├── packages
│   └── README.md
├── pyproject.toml
└── README.md
```

`pyproject.toml` 檔案是 `langchain` 指令和 `poetry` 都用來記錄依賴關係資訊和設定專案元資料的主要檔案。因為這現在使目錄成為有效的 `poetry` 項目，所以我們可以使用 `poetry` 來安裝其餘的依賴項:

- `langserve[all]`: LangServe 庫的伺服器和客戶端元件。
- `langchain-openai`: 該軟體包包含 LangChain 的 OpenAI 整合。
- `python-decouple`: 可用於讀取環境變數和 `.env` 檔案的套件。

```bash
poetry add "langserve[all]" langchain-openai python-decouple
```

我們的專案目錄現在擁有我們開始工作所需的所有依賴項和專案文件。

### 3.建立 LangServe 應用程式

若要建立基本的 LangServe 應用程序，請在文字編輯器中開啟 `app/server.py` 檔案。在裡面，將現有內容替換為以下程式碼:

```python title="app/server.py"
# app/server.py
from decouple import config
from fastapi import FastAPI
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from langserve import add_routes


app = FastAPI()

model = ChatOpenAI(openai_api_key=config("OPENAI_API_KEY"))
prompt = ChatPromptTemplate.from_template("Give me a summary about {topic} in a paragraph or less.")
chain = prompt | model

add_routes(app, chain, path="/openai")

if __name__ == "__main__":
    import uvicorn

    uvicorn.run(app, host="0.0.0.0", port=8000)
```

讓我們花點時間回顧一下這個應用程式的用途。

程式碼首先從我們安裝的套件中導入所有必需的類別、函數和其他材料。然後，它初始化一個 `FastAPI()` 實例，該實例將用作應用程式的主要應用程式物件。

接下來，我們初始化 `ChatOpenAI` 和 `ChatPromptTemplate` 類別的實例，並將它們分別指派給模型和提示變數。對於 `ChatOpenAI` 實例，我們使用 `python-decouple` 中的設定物件從 `.env` 檔案傳遞 OpenAI API 金鑰。對於 `ChatPromptTemplate`，我們設定提示以詢問給定主題的摘要。然後我們將這兩個連結在一個  `chain` 變數中。

我們新增了一條 route 來為 `/openai` 的 chain 提供服務。之後，我們使用 `uvicorn` 在使用連接埠 8000 的所有介面上提供應用程式服務。

### 4.測試應用程式

我們可以透過在主專案目錄中鍵入以下內容來測試應用程式是否按預期運作：

```bash
langchain serve
```

這將啟動應用程式伺服器。在網頁瀏覽器中導覽至 `127.0.0.1:8000/openai/playground` 以查看提示頁面。您可以透過輸入問題或主題來測試一切是否正常運作。

完成後，按 `CTRL-C` 停止伺服器。

### 5.調整 Dockerfile

在專案目錄裡面，我們建立一個名為 `.env` 的檔案。我們需要修改 Dockerfile 來將 `.env` 檔案複製進 Docker 映像中。

```docker hl_lines="17"
FROM python:3.11-slim

RUN pip install poetry==1.6.1

RUN poetry config virtualenvs.create false

WORKDIR /code

COPY ./pyproject.toml ./README.md ./poetry.lock* ./

COPY ./package[s] ./packages

RUN poetry install  --no-interaction --no-ansi --no-root

COPY ./app ./app

COPY ./.env ./.env

RUN poetry install --no-interaction --no-ansi

EXPOSE 8080

CMD exec uvicorn app.server:app --host 0.0.0.0 --port 8080
```

要部署 langserve 項目，首先建置 docker 映像:

```bash
docker build . -t example-langserve:latest
```

然後運行映像:

```bash
docker run -p 8080:8080  example-langserve:latest
```

這將使用 Docker 來啟動 Langserve 應用程式伺服器。在網頁瀏覽器中導覽至 `127.0.0.1:8080/openai/playground` 以查看提示頁面。您可以透過輸入問題或主題來測試一切是否正常運作。

![](./assets/langserve_playground.png)

