# 如何使用 Continue 小幫手

## 簡介

Continue 使用了 LLM 模型來幫助來進行程式開發。 LLM 有時會產生幻覺，因此可能會編造一個函式庫或發明一些不存在的語法。如果它所建議的某些內容不能正確運行或對您來說它所建議訊息似乎很奇怪的時候，最好仔細檢查或使用 Google 搜索其它資訊進行輔助。

當您持續地使用 Continue 時，您將了解何時信任它與利用它。一個很好的入門方法就是直接使用它並開始了解什麼有效，什麼無效。您可接受/拒絕它建議的任何更改。

如果您嘗試將它用於新任務，並且不知道 Continue 可以在多大程度上幫助您完成它，那麼像這樣開始通常會很有幫助：

1. 在編輯器上框出(highlight)您不理解的程式碼部分，然後在 Continue 的對話輸入框中輸入 "tell me how this code works"

2. 如果解釋看起來合理，那麼，在仍然有框出程式碼部分的同時，輸入 "how would you change this code to [INSERT TASK]?"

3. 如果這個解釋也很好，那麼，在仍然框出程式碼部分的同時，輸入 "/edit [INSERT TASK]"

4. 如果第一次嘗試效果不好，請按一下 `reject` 拒絕其建議，然後重試，通常每次 LLM 都會提出不同的建議

5. 如果在再次嘗試後它沒有給您想要的東西，請點擊 `reject`，然後使用更具體/清晰的說明來重試，LLM 需要您更準確闡明希望它做什麼和不做什麼

6. 如果這仍然不起作用，那麼您可能需要將任務分解為更小的子任務，並要求 LLM 一次完成其中每一項任務，或者自己手動完成

請記住：您需要對您發佈的所有程式碼負責，所以無論程式碼是由您編寫還是由您指導的 LLM 編寫。這意味著您查看 LLM 所寫的內容是否合理與有效至關重要。

## 何時使用 Ｃontinue

以下是 Continue 擅長幫助您完成的任務:

### Laborious edits

**簡單但耗時的編輯**

在 `find` 和 `replace` 不起作用的情況下，Contineu 效果很好（比如 "/edit change all of these to be like that"）

範例:

- "/edit Use 'Union' instead of a vertical bar here"
- "/edit Make this use more descriptive variable names"

### Writing files from scratch

**從無到有地構建新程式碼**

Continue 可以幫助您開始建立 React 元件、Python 腳本、Shell 腳本、Makefile、單元測試等。

範例:

- "/edit write a python script to get posthog events"
- "/edit add a react component for syntax highlighted code"

### Creating boilerplate from scratch

**從頭開始建立樣板**

Continue 可以幫助建立套件的腳手架(boilerplate)。

範例:

- "/edit use this schema to write me a SQL query that gets recently churned users"
- "/edit create a shell script to back up my home dir to /tmp/"

### Fix highlighted code

**修復程式碼**

選擇某部份的程式碼後，請嘗試使用　Continue　對其進行重構（例如 "/edit change the function to work like this" 或 "/edit do this everywhere"）

範例:

- "/edit migrate this digital ocean terraform file into one that works for GCP"
- "/edit rewrite this function to be async"

### Ask about highlighted code or an entire file

**詢問選定的程式碼或整個文件**

如果您不明白某些程式碼是如何工作的，請在編輯器上框出(highlight)您不理解的程式碼部分並詢問 "how does this code work?"

範例:

- “where in the page should I be making this request to the backend?”
- "how can I communicate between these iframes?"

### Ask about errors

**詢問錯誤**

Continue 還可以幫助解釋錯誤/異常並提供可能的解決方案。當您在終端機中遇到錯誤/異常時，請按 `cmd+shift+r` (MacOS) / `ctrl+shift+r` (Windows)。

這會將 stack trace 放入 Continue 中，並要求它向您解釋問題。

### Figure out what shell command to run

**找出要執行的 shell 指令**

您可以問 "How do I find running process on port 8000?" 之類的問題，而不是切換視窗並分心。

範例:

- "what is the load_dotenv library name?"
- "how do I find running process on port 8000?"

### Ask single-turn open-ended questions

**提出單輪開放式問題**

您可以提出開放式問題，而不是離開 IDE，而您不希望這些問題會變成多輪對話。

範例:

- "how can I set up a Prisma schema that cascades deletes?"
- "what is the difference between dense and sparse embeddings?"

### Editing small existing files

**編輯現有的小型文件**

您可以突出顯示整個文件並要求繼續改進它，只要文件不是太大。

範例:

- "/edit here is a connector for postgres, now write one for kafka"
- "/edit Rewrite this API call to grab all pages"

### Using context from multiple other files

**使用多個文件來作為上下文**

與手動進行更改的方式類似，一次只專注於一個文件。但是，如果其他文件中有關鍵訊息，請突出顯示這些代碼部分以用作附加上下文

### Tasks with a few steps

**只需幾個步驟即可完成任務**

Continue 可以幫助您完成更多任務。通常，這些任務不需要太多的步驟來完成。

範例:

- "/edit make an IAM policy that creates a user with read-only access to S3"
- "/edit change this plot into a bar chart in this dashboard component"

## 何時不適合使用 Ｃontinue

以下是 Continue 在這個階段沒有幫助的任務:

### Deep debugging

**深度除錯**

如果您得花 20 分鐘來調試多個文件中牽扯的複雜問題，那麼　Continue　將無法幫助您將各個問題點連接起來。

### Multi-file edits in parallel

**平行編輯多文件**

目前 Continue 一次只能編輯一個檔案。

### Using context of the entire file

**使用整個文件作為上下文**

如果檔案太大，Continue 可能很難將它們放入有限的 LLM 上下文視窗中。嘗試突出顯示包含相關上下文的程式碼部分。您很少需要整個文件。

### Editing large files

**編輯大文件**

同樣，如果您嘗試一次編輯太多行，則可能會遇到上下文視窗限制。應用這些建議也可能會非常緩慢。

### Highlighting really long lines

**突出顯示很長的行**

如果您突出顯示很長的行（例如複雜的 SVG），您也可能會遇到類似上述的問題。

### Tasks with many steps

**具有多個步驟的任務**

LLM 無法解決太複雜的程式任務。但是，通常情況下，如果您弄清楚如何將任務分解為子任務，則可以從 Continue 中獲得協助。