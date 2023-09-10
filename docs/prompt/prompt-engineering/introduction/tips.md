# 設計提示的通用技巧

以下是設計提示時需要記住的一些技巧：

## 從簡單開始

在設計提示時，需要記住這是一個迭代的過程，需要大量的實驗來獲得最佳結果。使用像 OpenAI 或 Cohere 這樣的簡單平台是一個很好的起點。

您可以從簡單的提示開始，隨著您的目標是獲得更好的結果，不斷添加更多的元素和上下文。在此過程中對您的提示進行版本控制是至關重要的。當您閱讀本指南時，您會看到許多例子，其中具體性、簡潔性和簡明性通常會給您帶來更好的結果。

當您有一個涉及許多不同子任務的大任務時，您可以嘗試將任務分解為更簡單的子任務，並隨著獲得更好的結果而不斷構建。這避免了在提示設計過程中一開始就添加過多的複雜性。

## 指令

您可以使用命令來指示模型執行各種簡單任務，例如 “寫入”、“分類”、“總結”、“翻譯”、“排序” 等，從而為各種簡單任務設計有效的提示。

請記住，您還需要進行大量的實驗，以查看哪種方法最有效。嘗試使用不同的關鍵字、上下文和數據嘗試不同的指令，看看哪種方法最適合您的特定用例和任務。通常情況下，上下文與您要執行的任務越具體和相關，效果越好。我們將在即將推出的指南中介紹採樣和添加更多上下文的重要性。

其他人建議將指令放在提示的開頭。建議使用一些清晰的分隔符，如 `###`，來分隔指令和上下文。

例如：

**Prompt**:

```bash
### Instruction ###
Translate the text below to Spanish:
Text: "hello!"
```

**Output**:

```bash
¡Hola!
```

## 具體性

對您希望模型執行的指令和任務非常具體。提示越具體和詳細，結果就越好。當您有所期望的結果或生成樣式時，這一點尤為重要。沒有特定的令牌或關鍵字會導致更好的結果。更重要的是具有良好的格式和描述性提示。實際上，在提示中提供示例非常有效，可以以特定格式獲得所需的輸出。

在設計提示時，您還應考慮提示的長度，因為提示的長度有限制。考慮到您應該具體和詳細的程度是需要考慮的。包含太多不必要的細節並不一定是一個好方法。這些細節應該是相關的，並有助於完成手頭的任務。這是您需要進行大量實驗的事情。我們鼓勵大量實驗和迭代，以優化您的應用程序的提示。

例如，讓我們嘗試從一段文本中提取特定信息的簡單提示。

**Prompt**:

```bash
Extract the name of places in the following text. 
Desired format:
Place: <comma_separated_list_of_company_names>
Input: "Although these developments are encouraging to researchers, much is still a mystery. “We often have a black box between the brain and the effect we see in the periphery,” says Henrique Veiga-Fernandes, a neuroimmunologist at the Champalimaud Centre for the Unknown in Lisbon. “If we want to use it in the therapeutic context, we actually need to understand the mechanism.""
```

**Output**:

```bash
Place: Champalimaud Centre for the Unknown, Lisbon
```

輸入文本來自[這篇 Nature 文章](https://www.nature.com/articles/d41586-023-00509-z)。

## 避免不精確

在上面關於詳細和格式改進的提示中，很容易陷入想要過於聰明的提示陷阱，從而可能創建不精確的描述。通常最好是具體和直接。這裡的類比非常類似於有效的溝通——越直接，信息傳遞就越有效。

例如，您可能有興趣了解提示工程的概念。您可以嘗試這樣做：

```bash
Explain the concept prompt engineering. Keep the explanation short, only a few sentences, and don't be too descriptive.
```

從上面的提示中不清楚要使用多少句話和什麼樣的風格。您可能仍然可以通過上面的提示獲得良好的響應，但更好的提示是非常具體、簡潔和直接的。例如：

```bash
Use 2-3 sentences to explain the concept of prompt engineering to a high school student.
```

## 做還是不做？

設計提示時的另一個常見技巧是避免說不要做什麼，而是說要做什麼。這鼓勵更具體化，並關注導致模型產生良好響應的細節。

以下是一個電影推薦聊天機器人的示例，因為我寫的指令——關注於不要做什麼，而失敗了。

**Prompt**:

```bash
The following is an agent that recommends movies to a customer. DO NOT ASK FOR INTERESTS. DO NOT ASK FOR PERSONAL INFORMATION.
Customer: Please recommend a movie based on my interests.
Agent: 
```

**Output**:

```bash
Sure, I can recommend a movie based on your interests. What kind of movie would you like to watch? Do you prefer action, comedy, romance, or something else?
```

以下是更好的提示：

**Prompt**:

```bash
The following is an agent that recommends movies to a customer. The agent is responsible to recommend a movie from the top global trending movies. It should refrain from asking users for their preferences and avoid asking for personal information. If the agent doesn't have a movie to recommend, it should respond "Sorry, couldn't find a movie to recommend today.".
Customer: Please recommend a movie based on my interests.
Agent:
```

**Output**:

```bash
Sorry, I don't have any information about your interests. However, here's a list of the top global trending movies right now: [list of movies]. I hope you find something you like!
```

上述示例中的一些內容摘自 [Best practices for prompt engineering with OpenAI API](https://help.openai.com/en/articles/6654000-best-practices-for-prompt-engineering-with-openai-api) 文章.