# 開發者快速入門

## 使用 OpenAI API 啟動並執行

OpenAI API 為開發人員提供了一個簡單的介面，可在其應用程式中建立智慧層，並由 OpenAI 最先進的模型提供支援。 Chat Completions 端點為 ChatGPT 提供支持，並提供了一種簡單的方法來將文字作為輸入並使用 GPT-4 等模型來產生輸出。

本快速入門旨在協助您設定本機開發環境並發送您的第一個 API 請求。如果您是經驗豐富的開發人員或只想深入使用 OpenAI API，GPT 指南的 [API 參考](https://platform.openai.com/docs/api-reference)是一個很好的起點。透過本快速入門，您將了解到：

- 如何設定您的開發環境
- 如何安裝最新的 SDK
- OpenAI API 的一些基本概念
- 如何發送您的第一個 API 請求

## 帳戶設定

首先，[建立 OpenAI 帳戶](https://platform.openai.com/signup)或[登入](https://platform.openai.com/login)。接下來，導覽至 [API 金鑰頁面](https://platform.openai.com/account/api-keys)和 "Create new secret key"，可以選擇命名金鑰。請務必將其保存在安全的地方，並且不要與任何人共享。

## 快速入門 - curl

Curl 是一種流行的命令列工具，開發人員使用它向 API 發送 HTTP 請求。它需要最少的設定時間，但功能不如 Python 或 JavaScript 等功能齊全的程式語言。

### Step 1: 設定 curl

許多作業系統預設提供 `curl`。您可以透過開啟終端機或命令列，然後輸入命令來檢查是否安裝了 `curl`:

```bash
curl https://platform.openai.com
```

如果安裝了 `curl` 並且您已連接到互聯網，它將發送 HTTP 請求來獲取 `platform.openai.com` 的內容。

如果出現找不到 `curl` 的錯誤，可以依照 `curl` 主頁上的指示進行安裝。

### Step 2: 設定您的 API 金鑰

現在我們已經可以使用 `curl` 了，下一步是在終端機或命令列中設定 API 金鑰。您可以選擇跳過此步驟，只需在請求中包含您的 API 金鑰，如 Step 3 所述。

根據不同的作業系統的特性來設定一個環境變數 `OPENAI_API_KEY` 來保存 API 金鑰。

### Step 3: 發送 API 請求

設定 API 金鑰後，最後一步是發送 API 請求。為此，下面包含了對聊天完成、嵌入和圖像 API 的範例請求。由於 API 金鑰是在 `Step 2` 中設定的，因此應透過終端機或命令列中的 `OPENAI_API_KEY` 自動引用它。您也可以手動將 `OPENAI_API_KEY` 替換為您的 API 金鑰，但如果包含您的 API 金鑰，請務必將 curl 命令隱藏。

=== "Chat Completions"

    ```bash
    curl https://api.openai.com/v1/chat/completions   -H "Content-Type: application/json"   -H "Authorization: Bearer $OPENAI_API_KEY"   -d '{
        "model": "gpt-3.5-turbo",
        "messages": [
        {
            "role": "system",
            "content": "You are a poetic assistant, skilled in explaining complex programming concepts with creative flair."
        },
        {
            "role": "user",
            "content": "Compose a poem that explains the concept of recursion in programming."
        }
        ]
    }'
    ```

=== "Embeddings"

    ```bash
    curl https://api.openai.com/v1/embeddings   -H "Authorization: Bearer $OPENAI_API_KEY"   -H "Content-Type: application/json"   -d '{
        "input": "The food was delicious and the waiter...",
        "model": "text-embedding-ada-002"
    }'
    ```

=== "Images"

    ```bash
    curl https://api.openai.com/v1/images/generations   -H "Content-Type: application/json"   -H "Authorization: Bearer $OPENAI_API_KEY"   -d '{
        "prompt": "A cute baby sea otter",
        "n": 2,
        "size": "1024x1024"
    }'
    ```

要運行程式碼，請在終端機/命令列中輸入 `python openai-test.py`。

聊天完成範例僅突出了 OpenAI 模型的一個優勢領域：創造力。用一首格式良好的詩來解釋遞歸（程式主題）是最好的開發人員和最好的詩人都會遇到的問題。在這種情況下，`gpt-3.5-turbo` 可以毫不費力地做到這一點。

## 快速入門 - Python

Python 是一種流行的程式語言，由於其易用性，通常用於資料應用程式、Web 開發和許多其他程式設計任務。 OpenAI 提供了一個自訂 Python 函式庫，使得在 Python 中使用 OpenAI API 變得簡單且有效率。

### Step 1: 設定 Python

**安裝 Python**

要使用 OpenAI Python 程式庫，您需要確保已安裝 Python。有些計算機預先安裝了 Python，而其他計算機則要求您自行設定。要測試是否安裝了 Python，您可以導航到終端機或命令列：

- MacOS：開啟終端機：您可以在 Applications 資料夾中找到它或使用 Spotlight（Command + Space）進行搜尋。
- Windows：開啟命令提示字元：您可以透過在開始功能表中搜尋 "cmd" 來找到它。

接下來，輸入 python，然後按回車鍵。如果你進入 Python 解釋器，那麼你的電腦上已經安裝了 Python，你可以進入下一步。如果您收到類似 "Error: command python not found" 的錯誤訊息，您可能需要安裝 Python 並使其在終端機/命令列中可用。

若要下載 Python，請前往 [Python 官方網站](https://www.python.org/downloads/)並下載最新版本。要使用 OpenAI Python 函式庫，您至少需要 Python 3.7.1 或更高版本。如果您是第一次安裝Python，可以按照[官方的 Python 初學者安裝指南](https://wiki.python.org/moin/BeginnersGuide/Download)進行操作。

**設定虛擬環境（optional）**

安裝 Python 後，最好先建立一個虛擬 Python 環境來安裝 OpenAI Python 函式庫。虛擬環境為要安裝的 Python 套件提供了一個乾淨的工作空間，讓您不會與其他專案安裝的其他程式庫發生衝突。您不需要使用虛擬環境，因此如果您不想設定虛擬環境，請跳至 `Step 3`。

為了創建虛擬環境，Python 提供了內建的 `venv` 模組，該模組提供虛擬環境設定所需的基本功能。執行以下命令將在您在終端機/命令列中選擇的目前資料夾內建立一個名為 `openai-env` 的虛擬環境：

```bash
python -m venv openai-env
```

創建虛擬環境後，您需要啟動它。在 Windows 上，運行：

```bash
openai-env\Scripts\activate
```

在 Unix 或 MacOS 上，運行：

```bash
source openai-env/bin/activate
```

啟動虛擬環境後，您應該會看到終端機/命令列介面略有變化，它現在應該在遊標輸入部分的左側顯示 `openai-env`。有關使用虛擬環境的更多詳細信息，請參閱[官方 Python 文件](https://docs.python.org/3/tutorial/venv.html#creating-virtual-environments)。

**安裝 OpenAI Python 函式庫**

一旦安裝了 Python 3.7.1 或更高版本以及（可選）虛擬環境設置，就可以安裝 OpenAI Python 庫。從終端機/命令列運行：

```bash
pip install --upgrade openai
```

完成後，執行 `pip list` 將顯示目前環境中已安裝的 Python 庫，這應該可以確認 OpenAI Python 庫已成功安裝。

### Step 2: 設定 API 金鑰

**為所有專案設定 API 金鑰（建議）**

讓所有專案都可以存取 API 金鑰的主要優點是，Python 庫將自動偵測並使用它，而無需編寫任何程式碼。

根據不同的作業系統的特性來設定一個環境變數 `OPENAI_API_KEY` 來保存 API 金鑰。

**為單一專案設定 API 金鑰**

如果您只希望單一專案可以存取 API 金鑰，則可以建立一個包含 API 金鑰的本機 `.env` 文件，然後透過後續步驟中顯示的 Python 程式碼明確使用該 API 金鑰。

首先轉到要在其中建立 `.env` 檔案的專案資料夾。`.env` 檔案應如下所示：

```bash
# Once you add your API key below, make sure to not share it with anyone! The API key should remain private.
OPENAI_API_KEY=abc123
```

可以透過執行以下程式碼導入 API 金鑰：

```python
from openai import OpenAI

client = OpenAI()
# defaults to getting the key using os.environ.get("OPENAI_API_KEY")
# if you saved the key under a different environment variable name, you can do something like:
# client = OpenAI(
#   api_key=os.environ.get("CUSTOM_ENV_NAME"),
# )
```

### Step 3: 發送 API 請求

配置 Python 並設定 API 金鑰後，最後一步是使用 Python 庫向 OpenAI API 發送請求。為此，請使用終端機或 IDE 建立一個名為 `openai-test.py` 的檔案。

在文件內，複製並貼上以下範例之一：


=== "Chat Completions"

    ```python
    from openai import OpenAI
    client = OpenAI()

    completion = client.chat.completions.create(
        model="gpt-3.5-turbo",
        messages=[
            {"role": "system", "content": "You are a poetic assistant, skilled in explaining complex programming concepts with creative flair."},
            {"role": "user", "content": "Compose a poem that explains the concept of recursion in programming."}
        ]
    )

    print(completion.choices[0].message)
    ```

=== "Embeddings"

    ```python
    from openai import OpenAI
    client = OpenAI()

    response = client.embeddings.create(
        model="text-embedding-ada-002",
        input="The food was delicious and the waiter..."
    )

    print(response)
    ```

=== "Images"

    ```python
    from openai import OpenAI
    client = OpenAI()

    response = client.images.generate(
        prompt="A cute baby sea otter",
        n=2,
        size="1024x1024"
    )

    print(response)
    ```

要運行程式碼，請在終端機/命令列中輸入 `python openai-test.py`。

聊天完成範例僅突出了 OpenAI 模型的一個優勢領域：創造力。用一首格式良好的詩來解釋遞歸（程式主題）是最好的開發人員和最好的詩人都會遇到的問題。在這種情況下，`gpt-3.5-turbo` 可以毫不費力地做到這一點。


## 快速入門 - Node.js

Node.js 是一種流行的 JavaScript 框架，常用於 Web 開發。 OpenAI 提供了一個自訂 [Node.js / TypeScript 函式庫](https://github.com/openai/openai-node)，使得在 JavaScript 中使用 OpenAI API 變得簡單且有效率。

### Step 1: 設定 Node

**安裝 Node.js**

要使用 OpenAI Node.js 函式庫，您需要確保安裝了 Node.js。

若要下載 Node.js，請前往 [Node 官方網站](https://nodejs.org/en/download)並下載 LTS 版本。如果您是第一次安裝 Node.js，可以依照 [Node.js 官方使用指南](https://nodejs.org/api/synopsis.html#usage)開始安裝。

**安裝 OpenAI Node.js 函式庫**

安裝 Node.js 後，就可以安裝 OpenAI Node.js 函式庫。從終端機/命令列運行：

```bash
npm install --save openai
# or
yarn add openai
```

### Step 2: 設定 API 金鑰

**為所有專案設定 API 金鑰（建議）**

讓您的 API 金鑰可供所有專案存取的主要優點是，我們的 SDK 將自動偵測並使用它，而無需編寫任何程式碼。

根據不同的作業系統的特性來設定一個環境變數 `OPENAI_API_KEY` 來保存 API 金鑰。

### Step 3: 發送 API 請求

配置 Node.js 並設定 API 金鑰後，最後一步是使用 Node.js 函式庫向 OpenAI API 發送請求。為此，請使用終端機或 IDE 建立一個名為 `openai-test.js` 的檔案。

在文件內，複製並貼上以下範例之一：

=== "Chat Completions"

    ```nodejsrepl
    import OpenAI from "openai";

    const openai = new OpenAI();

    async function main() {
        const completion = await openai.chat.completions.create({
            messages: [{ role: "system", content: "You are a helpful assistant." }],
            model: "gpt-3.5-turbo",
        });

        console.log(completion.choices[0]);
    }

    main();
    ```

=== "Embeddings"

    ```nodejsrepl
    import OpenAI from "openai";

    const openai = new OpenAI();

    async function main() {
        const embedding = await openai.embeddings.create({
            model: "text-embedding-ada-002",
            input: "The quick brown fox jumped over the lazy dog",
        });

        console.log(embedding);
    }

    main();
    ```

=== "Images"

    ```nodejsrepl
    import OpenAI from "openai";

    const openai = new OpenAI();

    async function main() {
        const image = await openai.images.generate({ prompt: "A cute baby sea otter" });

        console.log(image.data);
    }
    main();
    ```

要運行程式碼，請在終端機/命令列中輸入 `node openai-test.js`。

聊天完成範例僅突出了 OpenAI 模型的一個優勢領域：創造力。用一首格式良好的詩來解釋遞歸（程式主題）是最好的開發人員和最好的詩人都會遇到的問題。在這種情況下，`gpt-3.5-turbo` 可以毫不費力地做到這一點。
