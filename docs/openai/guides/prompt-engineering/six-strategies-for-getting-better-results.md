# 六個提示工程策略

下列六項策略可幫助你從 LLM 模型取得好的結果:

|策略 (Strategy)|手法 (Tactic)|
|--------|------|
|[**寫下清晰的指示**](#1)<br/><br/>Write clear instructions|- [在查詢中包含詳細資訊以獲得更相關的答案](#_2)<br/>- [為 LLM 模型設定角色(人格設定)](#llm)<br/>- [使用分隔符號清楚指示輸入的不同部分](#_3)<br/>- [指定完成任務所需的步驟](#_4)<br/>- [提供範例](#_5)<br/>- [指定所需的輸出長度](#_6)|
|[**提供參考文本**](#2)<br/><br/>Provide reference text|- [指示模型使用參考文本回答](#_7)<br/>- [指示模型透過引用參考文本來回答](#_8)|
|[**將複雜的任務拆分為更簡單的子任務**](#3)<br/><br/>Split complex tasks into simpler subtasks|- [使用意圖分類來識別使用者查詢最相關的指示](#_9)<br/>- [對於需要很長對話的對話應用，總結或過濾先前的對話](#_10)<br/>- [分段總結長文檔並逐漸建構完整摘要](#_11)|
|[**給模型時間"思考"**](#4)<br/><br/>Give models time to "think"|- [指示語言模型在急於下結論之前找出自己的解決方案](#_12)<br/>- [使用內在獨白(自言自語)或一系列查詢來隱藏模型的推理過程](#_13)<br/>- [詢問語言模型在先前的過程中是否遺漏了任何內容](#_14)|
|[**使用外部工具**](#5)<br/><br/>Use external tools|- [使用基於嵌入(embedding)的搜尋實現高效的知識檢索](#_15)<br/>- [使用 code execution 來執行更準確的計算或呼叫外部 API](#code-execution-api)<br/>- [授予模型存取特定外部預定義 function 的權力](#function)|
|[**有系統地測試提示變更**](#6)<br/><br/>Test changes systematically|- [參考黃金標準答案評估模型輸出](#_16)|


## 1. 寫下清晰的指示

**Strategy: Write clear instructions**

如果沒有清晰的指示，這些 LLM 模型就無法了解你的想法。如果輸出太長，請要求簡短答案。如果輸出太簡單，請要求專家級的寫作。如果您不喜歡這種格式，請示範您希望看到的格式。模型猜測你想要什麼意圖的模糊程度越小，你得到想要的結果的可能性就越大。- 使用基於嵌入(embedding)的搜尋實現高效的知識檢索
- 使用 code execution 來執行更準確的計算或呼叫外部 API
- 授予模型存取特定外部預定義 function 的權力

手法:

- 在查詢中包含詳細資訊以獲得更相關的答案
- 為 LLM 模型設定角色(人格設定)
- 使用分隔符號清楚指示輸入的不同部分
- 指定完成任務所需的步驟
- 提供範例
- 指定所需的輸出長度

### 在查詢中包含詳細資訊以獲得更相關的答案

**Tactic: Include details in your query to get more relevant answers**

為了獲得高度相關的回應，請確保在對模型的提示請求中提供與回應相關的重要的詳細資訊或上下文。否則，你將讓模型來猜測你的想法。


|Worse	|Better|
|-------|------|
|How do I add numbers in Excel?	<br/><br/>如何在 Excel 中加總數字?|How do I add up a row of dollar amounts in Excel? I want to do this automatically for a whole sheet of rows with all the totals ending up on the right in a column called "Total".<br/><br/>如何在 Excel 中對一行裡的所有數字進行金額的加總? 我想對整個工作表自動執行此操作，所有總計都在右側名為 "Total" 的列中結束。|
|Who’s president?<br/><br/>誰是總統?|Who was the president of Mexico in 2021, and how frequently are elections held?<br/><br/>誰是 2021 年墨西哥總統?選舉頻率如何?|
|Write code to calculate the Fibonacci sequence.<br/><br/>寫程式來計算斐波那契數列。|Write a TypeScript function to efficiently calculate the Fibonacci sequence. Comment the code liberally to explain what each piece does and why it's written that way.<br/><br/>編寫一個 TypeScript 函數來有效率地計算斐波那契數列。自由地註釋程式碼來解釋每部分的作用以及為什麼這樣編寫。|
|Summarize the meeting notes.<br/><br/>總結會議記錄。|Summarize the meeting notes in a single paragraph. Then write a markdown list of the speakers and each of their key points. Finally, list the next steps or action items suggested by the speakers, if any.<br/><br/>用一個段落總結會議記錄。然後寫下演講者的 Markdown 清單以及他們的每個要點。最後，列出發言人建議的後續步驟或行動項目（如果有的話）。|

### 為 LLM 模型設定角色

**Tactic: Ask the model to adopt a persona**

`System message` 可用於指定模型在其回覆使用者提問中所使用的角色(人格設定)。

|||
|-------|------|
|**SYSTEM**|When I ask for help to write something, you will reply with a document that contains at least one joke or playful comment in every paragraph.<br/><br/>當我請求幫助寫一些東西時，你會回覆一份文檔，其中每個段落至少包含一個笑話或有趣的評論。|
|**USER**|Write a thank you note to my steel bolt vendor for getting the delivery in on time and in short notice. This made it possible for us to deliver an important order.<br/><br/>給我的螺栓供應商寫一封感謝信，感謝他們在短時間內準時交貨。這使我們能夠交付一份重要的訂單。|

### 使用分隔符號清楚指示輸入的不同部分

**Tactic: Use delimiters to clearly indicate distinct parts of the input**

引號、XML 標籤、section 標題等分隔符號可以幫助區別想要讓語言模型參考的文本內容。

|||
|-------|------|
|**USER**|Summarize the text delimited by triple quotes with a haiku.<br/><br/>用俳句總結由引號所分隔的文本。<br/><br/>`"""insert text here"""`|

??? info
    俳句（Haiku）是一種日本詩歌形式，通常由三行組成，分為5、7、5音節。傳統的俳句通常捕捉自然景象，表達季節變化，或者突顯一瞬間的感覺。這種短小而精簡的形式旨在喚起讀者的感觸，讓他們感受到自然和生活的深層情感。

    傳統的俳句通常包括季節的提示，這被稱為「季語」（kigo）。而且，俳句還強調「切れ」（kire），即「切斷」或「切分」，這是指在詩的結構中創造出一種突然轉折或對比，讓詩歌更有深度和意義。

    這種形式的詩歌傳統上與日本文學和文化相關聯，但它在全球范圍內也受到了廣泛的歡迎，並且有許多現代的俳句是用其他語言寫成的。


|||
|-------|------|
|**SYSTEM**|You will be provided with a pair of articles (delimited with XML tags) about the same topic. First summarize the arguments of each article. Then indicate which of them makes a better argument and explain why.<br/><br/>您將獲得兩篇關於相同主題的文章（以 XML 標籤分隔）。首先總結每篇文章的論點。然後指出哪一個提出了更好的論點並解釋原因。|
|**USER**|`<article> insert first article here </article>`<br/><br/>`<article> insert second article here </article>`|


|||
|-------|------|
|**SYSTEM**|You will be provided with a thesis abstract and a suggested title for it. The thesis title should give the reader a good idea of the topic of the thesis but should also be eye-catching. If the title does not meet these criteria, suggest 5 alternatives.<br/><br/>您將獲得論文摘要和建議的標題。論文標題應該讓讀者清楚了解論文的主題，但也應該引人注目。如果標題不符合這些標準，請提出 5 個替代方案。|
|**USER**|Abstract: `insert abstract here`<br/><br/>Title: `insert title here`|

!!! tip
    對於一些的簡單任務，使用分隔符號可能不會對輸出品質產生影響。然而，當任務越複雜，消除任務細節的歧義就越重要。要讓模型準確地理解你對他們的要求。

### 指定完成任務所需的步驟

**Tactic: Specify the steps required to complete a task**

有些任務最好使用一系列的步驟來進行指示。明確地寫出這些步驟可以使模型更容易遵循它們。

|||
|-------|------|
|**SYSTEM**|Use the following step-by-step instructions to respond to user inputs.<br/><br/>Step 1 - The user will provide you with text in triple quotes. Summarize this text in one sentence with a prefix that says "Summary: ".<br/><br/>Step 2 - Translate the summary from Step 1 into Spanish, with a prefix that says "Translation: ".<br/><br/>使用以下逐步說明來回應使用者輸入。<br/><br/>步驟 1 - 使用者將為您提供引號中的文字。用一個句子總結這段文字，並加上前綴 "Summary:"。<br/><br/>步驟 2 - 將步驟 1 中的摘要翻譯成西班牙語，並加上前綴 "Translation:"。|
|**USER**|`"""insert text here"""`|

### 提供範例

**Tactic: Provide examples**

提供適用於所有範例的一般說明通常比透過範例示範任務的所有回應結果來的更有效，但在某些情況下提供範例可能更容易。例如，如果您打算讓模型複製回應使用者查詢的特定風格，而這種風格很難明確描述。這稱為 `"few-shot"` 提示。

|||
|-------|------|
|**SYSTEM**|Answer in a consistent style.<br/><br/>以一致的風格回答。|
|**USER**|Teach me about patience.<br/><br/>教導我何謂耐心。|
|**ASSISTANT**|The river that carves the deepest valley flows from a modest spring; the grandest symphony originates from a single note; the most intricate tapestry begins with a solitary thread.<br/><br/>發源於溫和的泉水所匯集而成的河流會沖刷出最深的山谷；最偉大的交響樂源自一個音符；最複雜的掛毯都是從一條單獨的線開始​​的。|
|**USER**|Teach me about the ocean.<br/><br/>教導我何謂大海。|

### 指定所需的輸出長度

**Tactic: Specify the desired length of the output**

您可以要求模型產生給定目標長度的輸出。目標輸出長度可以根據單字、句子、段落、要點等的計數來指定。但請注意，指示模型產生特定長度的回應內容時, 因長度有限制, 因此並不能精確地陳述結果。要求語言模型產生具有特定數量的段落或要點的輸出可以更可靠地陳述結果。

|||
|-------|------|
|**USER**|Summarize the text delimited by triple quotes in about 50 words.<br/><br/>用大約 50 個單字總結由三引號分隔的文本。<br/><br/>`"""insert text here"""`|

|||
|-------|------|
|**USER**|Summarize the text delimited by triple quotes in 2 paragraphs.<br/><br/>總結兩段中用三引號分隔的文本。<br/><br/>`"""insert text here"""`|

|||
|-------|------|
|**USER**|Summarize the text delimited by triple quotes in 3 bullet points.<br/><br/>將由三引號分隔的文字總結為 3 個要點。<br/><br/>`"""insert text here"""`|

## 2. 提供參考文本

**Strategy: Provide reference text**

語言模型可以自信地發明假答案，特別是當被問及深奧的主題或引文和 URL 時。就像一個好的學習筆記可以幫助學生在考試中取得更好的成績一樣，為這些模型提供參考文本可以幫助減少作答次數。

手法:

- 指示模型使用參考文本回答
- 指示模型透過引用參考文本來回答

### 指示模型使用參考文本回答

**Tactic: Instruct the model to answer using a reference text**

如果我們可以為模型提供與當前查詢相關的可信信息，那麼我們可以指示模型使用提供的資訊來組成其答案。

|||
|-------|------|
|**SYSTEM**|Use the provided articles delimited by triple quotes to answer questions. If the answer cannot be found in the articles, write "I could not find an answer.<br/><br/>使用提供的由三重引號引起來的文章來回答問題。如果在文章中找不到答案，請回答 "我找不到答案"。|
|**USER**|`<insert articles, each delimited by triple quotes>`<br/><br/>Question: `<insert question here>`|

鑑於所有模型的上下文視窗都有其限制，我們需要某種方法來動態地尋找與所提出的問題相關的資訊。[嵌入(embedding)](https://platform.openai.com/docs/guides/embeddings/what-are-embeddings)可用於實現高效率的知識檢索。有關如何實現這一點的更多詳細信息，請參閱策略 [使用基於嵌入的搜尋來實現高效的知識檢索]("Use embeddings-based search to implement efficient knowledge retrieval" )。

### 指示模型透過引用參考文本來回答

**Tactic: Instruct the model to answer with citations from a reference text**

如果輸入已補充相關知識，則可以直接要求模型透過引用所提供文件中的段落來為其答案新增引用。請注意，輸出中的引用可以透過所提供文件中的字串匹配以程式設計方式進行驗證。

|||
|-------|------|
|**SYSTEM**|You will be provided with a document delimited by triple quotes and a question. Your task is to answer the question using only the provided document and to cite the passage(s) of the document used to answer the question. If the document does not contain the information needed to answer this question then simply write: "Insufficient information." If an answer to the question is provided, it must be annotated with a citation. Use the following format for to cite relevant passages ({"citation": …}).<br/><br/>您將獲得一份由三重引號和一個問題分隔的文檔。您的任務是僅使用提供的文件回答問題，並引用用於回答問題的文件段落。如果文件不包含回答此問題所需的信息，則只需回應: "資訊不足"。如果提供了問題的答案，則必須附有引文註釋。使用以下格式引用相關段落（({"citation": …})。|
|**USER**|`"""<insert document here>"""`<br/><br/>Question: `<insert question here>`|

## 3. 將複雜的任務拆分為更簡單的子任務

**Strategy: Split complex tasks into simpler subtasks**

正如軟體工程中將複雜系統分解為一組模組化元件是良好實踐一樣，提交給語言模型的任務也是如此。複雜的任務往往比簡單的任務有更高的錯誤率。此外，複雜的任務通常可以被重新定義為更簡單任務的工作流程，其中早期任務的輸出可用於建構後續任務的輸入。

手法:

- 使用意圖分類來識別使用者查詢最相關的指示
- 對於需要很長對話的對話應用，總結或過濾先前的對話
- 分段總結長文檔並逐漸建構完整摘要

### 使用意圖分類來識別使用者查詢最相關的指示

**Tactic: Use intent classification to identify the most relevant instructions for a user query**

對於需要大量指令集來處理不同情況的複雜任務，首先對查詢類型進行分類並使用該分類來確定需要哪些指令可能是有益的。

這可以透過定義與處理給定類別中的任務相關的固定類別和硬編碼指令來實現。該過程還可以遞歸地應用以將任務分解為一系列階段。

這種方法的優點是每個查詢僅包含執行任務下一階段所需的指令，與使用單一查詢執行整個任務相比，這可以降低錯誤率。

這種方法在某種情形下也可以降低成本，因為較大的提示運行成本更高(見 [pricing information](https://openai.com/pricing))。

例如，假設對於客戶服務應用程序，查詢可以有效地分類如下：

|||
|-------|------|
|**SYSTEM**|You will be provided with customer service queries. Classify each query into a primary category and a secondary category. Provide your output in json format with the keys: primary and secondary.<br/><br/>Primary categories: Billing, Technical Support, Account Management, or General Inquiry.<br/><br/>Billing secondary categories:<br/>- Unsubscribe or upgrade<br/>- Add a payment method<br/>- Explanation for charge<br/>- Dispute a charge<br/><br/>Technical Support secondary categories:<br/>- Troubleshooting<br/>- Device compatibility<br/>- Software updates<br/><br/>Account Management secondary categories:<br/>- Password reset<br/>- Update personal information<br/>- Close account<br/>- Account security<br/><br/>General Inquiry secondary categories:<br/>- Product information<br/>- Pricing<br/>- Feedback<br/>- Speak to a human|
|**USER**|I need to get my internet working again.|

根據客戶查詢的分類，可以向語言模型提供一組更具體的指令，以供其處理後續步驟。例如，假設客戶需要 "troubleshooting" 方面的協助。

|||
|-------|------|
|**SYSTEM**|You will be provided with customer service inquiries that require troubleshooting in a technical support context. Help the user by:<br/><br/>- Ask them to check that all cables to/from the router are connected. Note that it is common for cables to come loose over time.<br/>- If all cables are connected and the issue persists, ask them which router model they are using<br/>- Now you will advise them how to restart their device:<br/>-- If the model number is MTD-327J, advise them to push the red button and hold it for 5 seconds, then wait 5 minutes before testing the connection.<br/>-- If the model number is MTD-327S, advise them to unplug and replug it, then wait 5 minutes before testing the connection.<br/>- If the customer's issue persists after restarting the device and waiting 5 minutes, connect them to IT support by outputting {"IT support requested"}.<br/>- If the user starts asking questions that are unrelated to this topic then confirm if they would like to end the current chat about troubleshooting and classify their request according to the following scheme:<br/><br/>`<insert primary/secondary classification scheme from above here>`|
|**USER**|I need to get my internet working again.|

請注意，在上述的範例中語言模型已被指示發出特殊字串來指示對話狀態何時發生變化。這使我們能夠將我們的系統變成一個狀態機，其中狀態決定要注入哪些指令。透過追蹤狀態、哪些指令與該狀態相關，以及可選地允許從該狀態進行哪些狀態轉換。

### 對於需要很長對話的對話應用，總結或過濾先前的對話

**Tactic: For dialogue applications that require very long conversations, summarize or filter previous dialogue**

由於模型具有固定的上下文長度，因此使用者和助手之間的對話（其中整個對話都包含在上下文視窗中）無法無限期地繼續。

解決此問題有多種解決方法，其中之一是總結對話中的內容。一旦輸入的大小達到預定的閾值長度，就是觸發總結部分對話的時機點，並且先前對話的摘要可以作為系統訊息的一部分包括在內。或者，可以在整個對話過程中在後台非同步總結先前的對話。

另一個解決方案是動態選擇與目前查詢最相關的對話的先前部分。參考 ["Use embeddings-based search to implement efficient knowledge retrieval"](https://platform.openai.com/docs/guides/prompt-engineering/tactic-use-embeddings-based-search-to-implement-efficient-knowledge-retrieval)。

### 分段總結長文檔並逐漸建構完整摘要

**Tactic: Summarize long documents piecewise and construct a full summary recursively**

由於模型具有固定的上下文長度，因此它們不能用於總結長於上下文長度減去單一查詢中產生的摘要長度的文字。

要總結一個很長的文件（例如一本書），我們可以使用一系列查詢來總結文件的每個部分。章節摘要可以連結和總結，產生摘要的摘要。這個過程可以遞歸地進行，直到總結整個文件。

如果有必要使用前面部分的資訊來理解後面的部分，那麼另一個有用的技巧是在書中任何給定點之前包含文本的運行摘要，同時總結該點的內容。 OpenAI 在先前的研究中已經使用 GPT-3 的變體研究了這種總結書籍的過程的有效性。


## 4. 給模型時間"思考"

**Strategy: Give models time to "think"**

如果要求你馬上直接給出 `17 * 28 = ???` 的答案時，您可能不會立即算的出來，但若給你足夠的時間來計算那麼得出對的答案的機率會大增。同樣，語言模型在嘗試立即回答而不是花時間找出答案時會犯更多推理錯誤。若是我們要語言模型使用 **思路鏈 (chain of thought)** 的手法來一步一步地來進行思考，這樣可以幫助模型更可靠地推理出正確答案。

手法:

- 指示語言模型在急於下結論之前找出自己的解決方案
- 使用內在獨白(自言自語)或一系列查詢來隱藏模型的推理過程
- 詢問語言模型在先前的過程中是否遺漏了任何內容

### 指示語言模型在急於下結論之前找出自己的解決方案

**Tactic: Instruct the model to work out its own solution before rushing to a conclusion**

有時，當我們明確指示模型在得出結論之前從基本原理進行推理時，我們會得到更好的結果。假設我們想要一個模型來評估學生對數學問題的解。解決這個問題最明顯的方法是簡單地詢問模型學生的解決方案是否正確。

|||
|-------|------|
|**SYSTEM**|Determine if the student's solution is correct or not.<br/><br/>確定學生的解決方案是否正確。|
|**USER**|Problem Statement: I'm building a solar power installation and I need help working out the financials.<br/>- Land costs $100 / square foot<br/>- I can buy solar panels for $250 / square foot<br/>- I negotiated a contract for maintenance that will cost me a flat $100k per year, and an additional $10 / square foot<br/><br/>What is the total cost for the first year of operations as a function of the number of square feet.<br/><br/>Student's Solution: Let x be the size of the installation in square feet.<br/>1. Land cost: 100x<br/>2. Solar panel cost: 250x<br/>3. Maintenance cost: 100,000 + 100x<br/>
Total cost: 100x + 250x + 100,000 + 100x = 450x + 100,000<br/>|
|**ASSISTANT**|The student's solution is correct.|

運行範例會從語言模型得到 `The student's solution is correct.` 的回覆。

然而同學的解法其實不正確！我們可以透過提示模型先產生自己的解決方案來讓模型成功注意到這一點。


|||
|-------|------|
|**SYSTEM**|First work out your own solution to the problem. Then compare your solution to the student's solution and evaluate if the student's solution is correct or not. Don't decide if the student's solution is correct until you have done the problem yourself.|
|**USER**|Problem Statement: I'm building a solar power installation and I need help working out the financials.<br/>- Land costs $100 / square foot<br/>- I can buy solar panels for $250 / square foot<br/>- I negotiated a contract for maintenance that will cost me a flat $100k per year, and an additional $10 / square foot<br/><br/>What is the total cost for the first year of operations as a function of the number of square feet.<br/><br/>Student's Solution: Let x be the size of the installation in square feet.<br/>1. Land cost: 100x<br/>2. Solar panel cost: 250x<br/>3. Maintenance cost: 100,000 + 100x<br/>
Total cost: 100x + 250x + 100,000 + 100x = 450x + 100,000<br/>|
|**ASSISTANT**|Let x be the size of the installation in square feet.<br/>1. Land cost: 100x<br/>2. Solar panel cost: 250x<br/>3. Maintenance cost: 100,000 + 10x<br/>
Total cost: 100x + 250x + 100,000 + 10x = 360x + 100,000<br/><br/>The student's solution is incorrect. They made an error in the maintenance cost calculation, using 100x instead of 10x. The correct total cost for the first year of operations is 360x + 100,000.|

### 使用內在獨白或一系列查詢來隱藏推理過程

**Tactic: Use inner monologue or a sequence of queries to hide the model's reasoning process**

前面的策略表明，模型有時在回答特定問題之前詳細推理問題很重要。對於某些應用程序，模型用於得出最終答案的推理過程不適合與用戶共享。例如，在教學應用程式中，我們可能希望鼓勵學生得出自己的答案，但模型關於學生解決方案的推理過程可能會向學生揭示答案。

內心獨白(inner monologue)是一種可以用來緩解這種情況的策略。內心獨白的想法是指示模型將原本對使用者隱藏的部分輸出放入結構化格式中，以便於解析它們。然後，在向使用者呈現輸出之前，將解析輸出並且僅使部分輸出可見。

|||
|-------|------|
|**SYSTEM**|Follow these steps to answer the user queries.<br/><br/>Step 1 - First work out your own solution to the problem. Don't rely on the student's solution since it may be incorrect. Enclose all your work for this step within triple quotes (""").<br/><br/>Step 2 - Compare your solution to the student's solution and evaluate if the student's solution is correct or not. Enclose all your work for this step within triple quotes (""").<br/><br/>Step 3 - If the student made a mistake, determine what hint you could give the student without giving away the answer. Enclose all your work for this step within triple quotes (""").<br/><br/>Step 4 - If the student made a mistake, provide the hint from the previous step to the student (outside of triple quotes). Instead of writing "Step 4 - ..." write "Hint:".|
|**USER**|Problem Statement: `<insert problem statement>`<br/><br/>Student Solution: `<insert student solution>`|

或者，這可以透過一系列查詢來實現，其中除了最後一個查詢之外，所有查詢的輸出都對最終用戶隱藏。

首先，我們可以要求模型自己解決問題。由於這個初始查詢不需要學生的解決方案，因此可以省略。這提供了額外的優勢，即模型的解決方案不會因學生嘗試的解決方案而產生偏差。

|||
|-------|------|
|**USER**|`<insert problem statement>`|

接下來，我們可以讓模型使用所有可用資訊來評估學生解決方案的正確性。

|||
|-------|------|
|**SYSTEM**|Compare your solution to the student's solution and evaluate if the student's solution is correct or not.|
|**USER**|Problem statement: `"""<insert problem statement>"""`<br/><br/>Your solution: `"""<insert model generated solution>"""`<br/><br/>
Student’s solution: `"""<insert student's solution>"""`|

最後，我們可以讓模型使用自己的分析來以樂於助人的導師的角色建立回應。

|||
|-------|------|
|**SYSTEM**|You are a math tutor. If the student made an error, offer a hint to the student in a way that does not reveal the answer. If the student did not make an error, simply offer them an encouraging comment.|
|**USER**|Problem statement: `"""<insert problem statement>"""`<br/><br/>Your solution: `"""<insert model generated solution>"""`<br/><br/>Student’s solution: `"""<insert student's solution>"""`<br/><br/>Analysis: `"""<insert model generated analysis from previous step>"""`|

### 詢問語言模型在先前的過程中是否遺漏了任何內容

**Tactic: Ask the model if it missed anything on previous passes**

假設我們使用一個模型來列出與特定問題相關的來源的摘錄。列出每個摘錄後，模型需要確定是否應該開始編寫另一個摘錄或是否應該停止。如果來源文件很大，模型通常會過早停止並且無法列出所有相關摘錄。在這種情況下，透過使用後續查詢提示模型來尋找先前傳遞中錯過的任何摘錄，通常可以獲得更好的效能。

|||
|-------|------|
|**SYSTEM**|You will be provided with a document delimited by triple quotes. Your task is to select excerpts which pertain to the following question: "What significant paradigm shifts have occurred in the history of artificial intelligence."<br/><br/>Ensure that excerpts contain all relevant context needed to interpret them - in other words don't extract small snippets that are missing important context. Provide output in JSON format as follows:<br/><br/>[{"excerpt": "..."},<br/>...<br/>{"excerpt": "..."}]|
|**USER**|`"""<insert document here>"""`|
|**ASSISTANT**|[{"excerpt": "the model writes an excerpt here"},<br/>...<br/>{"excerpt": "the model writes another excerpt here"}]|
|**USER**|Are there more relevant excerpts? Take care not to repeat excerpts. Also ensure that excerpts contain all relevant context needed to interpret them - in other words don't extract small snippets that are missing important context.|

## 5. 使用外部工具

**Strategy: Use external tools**

透過向語言模型提供其他工具的輸出來彌補模型的弱點。例如，文字檢索系統（有時稱為 RAG 或檢索增強生成）可以告訴模型用來回答使用者問題的相關文件知識。像 OpenAI 的程式碼解釋器這樣的程式碼執行引擎可以幫助模型進行數學運算並運行程式碼。如果一項任務可以透過工具而不是語言模型更可靠或更有效地完成，那麼可以充分利用兩者的能力來進行互補。

手法:

- 使用基於嵌入(embedding)的搜尋實現高效的知識檢索
- 使用 code execution 來執行更準確的計算或呼叫外部 API
- 授予模型存取特定外部預定義 function 的權力

### 使用基於嵌入的搜尋實現高效的知識檢索

**Tactic: Use embeddings-based search to implement efficient knowledge retrieval**

如果作為輸入的一部分提供，模型可以利用外部資訊來源。這可以幫助模型產生更明智和最新的回應。例如，如果用戶詢問有關特定電影的問題，則將有關電影的高品質資訊（例如演員、導演等）添加到模型的輸入中可能會很有用。嵌入(embedding)可用於實現高效的知識檢索，從而可以在運行時動態地將相關資訊添加到模型輸入中。

文字嵌入(embedding)是一個可以衡量文字字串之間相關性的向量。相似或相關的字串比不相關的字串更接近。這一事實以及快速向量搜尋演算法的存在意味著嵌入可以用於實現高效的知識檢索。特別地，文字語料庫可以被分割成區塊，並且每個區塊可以被嵌入和儲存。然後可以嵌入給定的查詢，並且可以執行向量搜尋以從語料庫中找到與查詢最相關的嵌入文字區塊（即在嵌入空間中最接近的文字區塊）。

範例實作可以在 [OpenAI Cookbook](https://cookbook.openai.com/examples/vector_databases/readme) 中找到。請參閱策略 [“指示模型使用檢索到的知識來回答查詢”](https://platform.openai.com/docs/guides/prompt-engineering/tactic-instruct-the-model-to-use-retrieved-knowledge-to-answer-queries)，以了解如何使用知識檢索來最大程度地減少模型編造不正確事實的可能性的範例。

### 使用 code execution 來執行更準確的計算或呼叫外部 API

**Tactic: Use code execution to perform more accurate calculations or call external APIs**

不能依賴語言模型自行準確地執行算術或長時間計算。在需要的情況下，可以指示模型編寫和運行程式碼，而不是進行自己的計算。特別是，可以指示模型將要執行的程式碼放入指定的格式，例如三重反引號。產生輸出後，可以提取程式碼並運行。最後，如有必要，可以將程式碼執行引擎（即 Python 解釋器）的輸出作為下一個查詢的模型的輸入。

|||
|-------|------|
|**SYSTEM**|You can write and execute Python code by enclosing it in triple backticks, e.g. ```code goes here```. Use this to perform calculations.|
|**USER**|Find all real-valued roots of the following polynomial: 3*x**5 - 5*x**4 - 3*x**3 - 7*x - 10.|

程式碼執行的另一個很好的用例是呼叫外部 API。如果模型接受如何正確使用 API 的指導，它就可以編寫使用該 API 的程式碼。透過向模型提供展示如何使用 API 的文件和/或程式碼範例，可以指導模型如何使用 API。

|||
|-------|------|
|**SYSTEM**|You can write and execute Python code by enclosing it in triple backticks. Also note that you have access to the following module to help users send messages to thei這是一個滿足這兩點的範例輸入：r friends:<br/><br/>```python<br/>import message<br/>message.write(to="John", message="Hey, want to meetup after work?")```|

### 授予模型存取特定外部預定義 function 的權力

**Tactic: Give the model access to specific functions**

聊天完成(Chat Completions) API 允許在請求中傳遞對所需功能(function)的描述清單。這使得模型能夠根據提供的模式產生函數參數。產生的函數參數由 API 以 JSON 格式傳回，可用於執行函數呼叫。然後，可以將函數呼叫提供的輸出回饋到以下請求中的模型以關閉循環。這是使用 OpenAI 模型呼叫外部函數的建議方式。要了解更多信息，請參閱我們的介紹性文字生成指南中的[函數呼叫](https://platform.openai.com/docs/guides/function-calling)部分以及 OpenAI Cookbook 中的更多[函數呼叫範例](https://cookbook.openai.com/examples/how_to_call_functions_with_chat_models)。

## 6. 有系統地測試提示變更

**Strategy: Test changes systematically**

如果您可以衡量效果，那麼提高效能就會更容易點。在某些情況下，對提示的修改對於某些特定的用例上會實現較好的效果，但會可能導致在另外一組更具代表性的用例上整體效能變差。因此，為了確保變更對效能產生正面影響，可能有必要定義一個全面的測試（也稱為「評估」）。

手法:

- 參考黃金標準答案評估模型輸出

### 參考黃金標準答案評估模型輸出

**Tactic: Evaluate model outputs with reference to gold-standard answers**

有時很難判斷更改（例如新指令或新設計）是否使您的系統變得更好或更差。看幾個例子可能會暗示哪個更好，但由於樣本量較小，很難區分真正的改進或隨機運氣。也許這種變化有助於某些輸入的性能，但會損害其他輸入的性能。

評估程序（或“evals”）對於優化系統設計非常有用。好的評估有下列的特性：

- 代表現實世界的使用情況（或至少是多樣化的）
- 包含許多測試案例以獲得更大的統計能力（有關指南，請參閱下表）
- 易於自動化或重複


|DIFFERENCE TO DETECT	|SAMPLE SIZE NEEDED FOR 95% CONFIDENCE|
|-------|------|
|30%	|~10|
|10%	|~100|
|3%	|~1,000|
|1%	|~10,000|

輸出的評估可以由電腦、人類或兩者混合來完成。電腦可以使用客觀標準（例如，具有單一正確答案的問題）以及一些主觀或模糊標準自動進行評估，其中模型輸出由其他模型查詢進行評估。 [OpenAI Evals](https://github.com/openai/evals) 是一個開源軟體框架，提供用於建立自動化評估的工具。

當存在一系列可能的輸出被認為品質同樣高時（例如，對於答案很長的問題），基於模型的評估可能會很有用。透過基於模型的評估可以實際評估的內容與需要人類評估的內容之間的界限是模糊的，並且隨著模型變得更加強大而不斷變化。我們鼓勵進行實驗，以確定基於模型的評估對您的用例的效果如何。

假設已知問題的正確答案應參考一組特定的已知事實。然後我們可以使用模型查詢來計算答案中包含多少必需的事實。

例如，使用以下系統訊息：

|||
|-------|------|
|**SYSTEM**|You will be provided with text delimited by triple quotes that is supposed to be the answer to a question. Check if the following pieces of information are directly contained in the answer:<br/><br/>- Neil Armstrong was the first person to walk on the moon.<br/>- The date Neil Armstrong first walked on the moon was July 21, 1969.<br/><br/>For each of these points perform the following steps:<br/><br/>1 - Restate the point.<br/>2 - Provide a citation from the answer which is closest to this point.<br/>3 - Consider if someone reading the citation who doesn't know the topic could directly infer the point. Explain why or why not before making up your mind.<br/>4 - Write "yes" if the answer to 3 was yes, otherwise write "no".<br/><br/>Finally, provide a count of how many "yes" an|||swers there are. Provide this count as {"count": <insert count here>}.|


這是一個滿足這兩點的範例輸入：

|||
|-------|------|
|**SYSTEM**|`<insert system message above>`|
|**USER**|"""Neil Armstrong is famous for being the first human to set foot on the Moon. This historic event took place on July 21, 1969, during the Apollo 11 mission."""|

以下是僅滿足一個點的範例輸入：

|||
|-------|------|
|**SYSTEM**|`<insert system message above>`|
|**USER**|"""Neil Armstrong made history when he stepped off the lunar module, becoming the first person to walk on the moon."""|

這是一個不滿足任何條件的範例輸入：

|||
|-------|------|
|**SYSTEM**|`<insert system message above>`|
|**USER**|"""In the summer of '69, a voyage grand, Apollo 11, bold as legend's hand. Armstrong took a step, history unfurled, "One small step," he said, for a new world."""|

這種基於模型的評估有許多可能的變體。考慮以下變體，它追蹤候選答案和黃金標準答案之間的重疊類型，並且還追蹤候選答案是否與黃金標準答案的任何部分相矛盾。

|||
|-------|------|
|**SYSTEM**|Use the following steps to respond to user inputs. Fully restate each step before proceeding. i.e. "Step 1: Reason...".<br/><br/>Step 1: Reason step-by-step about whether the information in the submitted answer compared to the expert answer is either: disjoint, equal, a subset, a superset, or overlapping (i.e. some intersection but not subset/superset).<br/><br/>Step 2: Reason step-by-step about whether the submitted answer contradicts any aspect of the expert answer.<br/><br/>Step 3: Output a JSON object structured like: {"type_of_overlap": "disjoint" or "equal" or "subset" or "superset" or "overlapping", "contradiction": true or false}|

這是一個範例輸入，其答案不合格，但與專家答案並不矛盾：

|||
|-------|------|
|**SYSTEM**|`<insert system message above>`|
|**USER**|Question: """What event is Neil Armstrong most famous for and on what date did it occur? Assume UTC time."""<br/><br/>Submitted Answer: """Didn't he walk on the moon or something?"""<br/><br/>Expert Answer: """Neil Armstrong is most famous for being the first person to walk on the moon. This historic event occurred on July 21, 1969."""|

這是一個範例輸入，其答案與專家答案直接矛盾：

|||
|-------|------|
|**SYSTEM**|`<insert system message above>`|
|**USER**|Question: """What event is Neil Armstrong most famous for and on what date did it occur? Assume UTC time."""<br/><br/>Submitted Answer: """On the 21st of July 1969, Neil Armstrong became the second person to walk on the moon, following after Buzz Aldrin."""<br/><br/>Expert Answer: """Neil Armstrong is most famous for being the first person to walk on the moon. This historic event occurred on July 21, 1969."""|

下面是一個帶有正確答案的範例輸入，它還提供了比必要的更多的細節：

|||
|-------|------|
|**SYSTEM**|`<insert system message above>`|
|**USER**|Question: """What event is Neil Armstrong most famous for and on what date did it occur? Assume UTC time."""<br/><br/>Submitted Answer: """At approximately 02:56 UTC on July 21st 1969, Neil Armstrong became the first human to set foot on the lunar surface, marking a monumental achievement in human history."""<br/><br/>Expert Answer: """Neil Armstrong is most famous for being the first person to walk on the moon. This historic event occurred on July 21, 1969."""|

## 其他資源

如需更多靈感，請訪問 [OpenAI Cookbook](https://cookbook.openai.com/)，其中包含範例程式碼以及第三方資源的鏈接，例如：

- [提示函式庫和工具](https://cookbook.openai.com/related_resources#prompting-libraries--tools)
- [提示指南](https://cookbook.openai.com/related_resources#prompting-guides)
- [視訊課程](https://cookbook.openai.com/related_resources#video-courses)
- [關於高級提示以提高推理能力的論文](https://cookbook.openai.com/related_resources#papers-on-advanced-prompting-to-improve-reasoning)

