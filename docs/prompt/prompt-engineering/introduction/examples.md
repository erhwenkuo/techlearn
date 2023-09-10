# 提示詞範例

在上一節中，我們介紹並給出如何對大語言模型下提示語的基本示例。

在本節中，我們會提供更多示例，介紹如何使用提示詞來完成不同的任務，並介紹其中涉及的重要概念。通常，學習概念的最佳方法是通過示例進行學習。下面，我們將通過示例介紹說明如何使用精細的提示詞來執行不同類型的任務。

- 文本摘要 (Text Summarization)
- 信息提取 (Information Extraction)
- 問答 (Question Answering)
- 文本分類 (Text Classification)
- 對話 (Conversation)
- 程式碼生成 (Code Generation)
- 推理 (Reasoning)

## 文本摘要

自然語言生成的標準任務之一是文本摘要。文本摘要可能涉及到不同的風格和領域。事實上，語言模型最前景的應用場景之一就是能夠快速概括出易於理解的文章大意和相關概念。我們可以使用提示詞嘗試一個簡單的概括任務。

假設我想了解 "抗生素" 的相關信息，我可以給出這樣的提示：

**Prompt**:

```bash
Explain antibiotics
A:
```

**Output**:

```bash
Antibiotics are a type of medication used to treat bacterial infections. They work by either killing the bacteria or preventing them from reproducing, allowing the body’s immune system to fight off the infection. Antibiotics are usually taken orally in the form of pills, capsules, or liquid solutions, or sometimes administered intravenously. They are not effective against viral infections, and using them inappropriately can lead to antibiotic resistance.
```

在問答形式中，`A:` 是一種明確的提示格式。在這個示例中，我用它去提示模型，我想要該概念的進一步解釋。在這個例子中，我們可能還不清楚使用它是否有用，我們會在之後的示例中探討這一點。現在假設我們感覺模型給了太多的信息，想要進一步提煉它。我們可以指導模型幫我們用一句話總結相關內容：

**Prompt**:

```bash hl_lines="7"
Antibiotics are a type of medication used to treat bacterial infections. They work by either killing the bacteria or preventing them from reproducing, allowing the body’s immune system to fight off the infection. Antibiotics are usually taken orally in the form of pills, capsules, or liquid solutions, or sometimes administered intravenously. They are not effective against viral infections, and using them inappropriately can lead to antibiotic resistance.
Explain the above in one sentence:
```

**Output**:

```bash
Antibiotics are medications used to treat bacterial infections by either killing the bacteria or stopping them from reproducing, but they are not effective against viruses and overuse can lead to antibiotic resistance.
```

本示例是模型在沒有過多關注上文輸出內容的準確性的情況下，嘗試用一個句子來總結段落內容。關於上文準確性，我們可以通過指令或說明進一步改善它，這一點我們會在後續指南中進行探討。讀到這裡，您可以暫時停住並進行實驗，看看是否能獲得更好的結果。

## 信息提取 

語言模型通過訓練不僅可以用於執行自然語言生成相關任務，還可以用於執行文本分類和其他一系列自然語言處理 (NLP) 任務。

使用以下示例提示詞從指定段落中提取信息：

**Prompt**:

```bash
Author-contribution statements and acknowledgements in research papers should state clearly and specifically whether, and to what extent, the authors used AI technologies such as ChatGPT in the preparation of their manuscript and analysis. They should also indicate which LLMs were used. This will alert editors and reviewers to scrutinize manuscripts more carefully for potential biases, inaccuracies and improper source crediting. Likewise, scientific journals should be transparent about their use of LLMs, for example when selecting submitted manuscripts.
Mention the large language model based product mentioned in the paragraph above:
```

**Output**:

```bash
The large language model based product mentioned in the paragraph above is ChatGPT.
```

我們可以通過許多方式改進以上結果，但當前方式已經非常有用。

現在應該很明顯，您可以通過簡單地指示語言模型去執行不同的任務。 AI 研發人員也利用這種能力來構建強大的產品和體驗。

Paragraph source: [ChatGPT: five priorities for research](https://www.nature.com/articles/d41586-023-00288-7)

## 問答

提高模型響應精確度的最佳方法之一是改進提示詞的格式。如前所述，提示詞可以通過指令、上下文、輸入和輸出指示以改進響應結果。雖然這些要素不是必需的，但如果您的指示越明確，響應的結果就會越好。以下示例可以說明結構化提示詞的重要性。


**Prompt**:

```bash
Answer the question based on the context below. Keep the answer short and concise. Respond "Unsure about answer" if not sure about the answer.
Context: Teplizumab traces its roots to a New Jersey drug company called Ortho Pharmaceutical. There, scientists generated an early version of the antibody, dubbed OKT3. Originally sourced from mice, the molecule was able to bind to the surface of T cells and limit their cell-killing potential. In 1986, it was approved to help prevent organ rejection after kidney transplants, making it the first therapeutic antibody allowed for human use.
Question: What was OKT3 originally sourced from?
Answer:
```

**Output**:

```bash
Mice.
```

Context obtained from [Nature](https://www.nature.com/articles/d41586-023-00400-x)。

## 文本分類

目前，我們已經會使用簡單的指令來執行任務。作為提示工程師，您需要提供更好的指令。此外， 您也會發現，對於更負責的使用場景，僅提供指令是遠遠不夠的。所以，您需要思考如何在提示詞中包含相關上下文和其他不同要素。同樣，你還可以提供其他的信息，如 `input data` 和 `examples`。

可以通過以下示例體驗文本分類：


**Prompt**:

```bash
Classify the text into neutral, negative or positive. 
Text: I think the food was okay. 
Sentiment:
```

**Output**:

```bash
Neutral
```

我們給出了對文本進行分類的指令，語言模型做出了正確響應，判斷文本類型為 `Neutral`。如果我們想要語言模型以指定格式做出響應， 比如，我們想要它返回 `neutral` 而不是 `Neutral`，那我們要如何做呢?

我們有多種方法可以實現這一點。此例中，我們主要是關注絕對特性，因此，我們提示詞中包含的信息越多，響應結果就會越好。我們可以使用以下示例來校正響應結果：

**Prompt**:

```bash
Classify the text into neutral, negative or positive. 
Text: I think the vacation is okay.
Sentiment: neutral 
Text: I think the food was okay. 
Sentiment:
```

**Output**:

```bash
neutral
```

完美！這次模型返回了 `neutral`，這正是我們想要的特定標籤。提示詞中的示例使得模型可以給出更具體的響應。有時給出具體的指令十分重要，可以通過以下示例感受這一點：

**Prompt**:

```bash
Classify the text into nutral, negative or positive. 
Text: I think the vacation is okay.
Sentiment:
```

**Output**:

```bash
Neutral
```

這裡有什麼問題？作為提示，模型完全忽略了虛構的中性標籤。相反，模型輸出中性，因為它對該標籤有一些偏見。但我們假設你真正想要的是中性的。你會如何解決這個問題？也許您可以嘗試在標籤中添加描述或在提示中添加更多示例？如果您不確定，我們將在接下來的部分中討論一些想法。

現在你知道給出具體指令的重要性了吧？

## 對話

你可以通過提示工程進行更有趣的實驗，比如指導大語言模型系統如何表現，指定它的行為意圖和身份。如果你正在構建客服聊天機器人之類的對話系統時，這個功能尤其有用。

例如，可以通過以下示例創建一個對話系統，該系統能夠基於問題給出技術性和科學性的答案。您可以關注我們是如何通過指示明確告訴模型應該如何表現。這種應用場景有時也被稱為角色提示(role prompting)。

**Prompt**:

```bash
The following is a conversation with an AI research assistant. The assistant tone is technical and scientific.
Human: Hello, who are you?
AI: Greeting! I am an AI research assistant. How can I help you today?
Human: Can you tell me about the creation of blackholes?
AI:
```

**Output**:

```bash
Sure! Black holes are regions of spacetime where the gravitational force is so strong that nothing, not even light, can escape from it. They are created when a very massive star dies and its core collapses in on itself, forming a singularity of infinite density. The intense gravity of the singularity pulls in all the matter and radiation around it, creating the black hole.
```

我們的人工智能助理給出的答案非常技術性對吧？ 下面，我們給出更容易理解的答案。

**Prompt**:

```bash
The following is a conversation with an AI research assistant. The assistant answers should be easy to understand even by primary school students.
Human: Hello, who are you?
AI: Greeting! I am an AI research assistant. How can I help you today?
Human: Can you tell me about the creation of black holes?
AI: 
```

**Output**:

```bash
Sure! Black holes are created when a very massive star runs out of fuel and collapses in on itself. The gravity of the star is so strong that nothing, not even light, can escape its pull. This creates a point in space-time with an intense gravitational force called a black hole.
```

我想我們應該取得了一些進展，你也可以繼續改進它。如果你再添加更多的示例，可能會得到更好的結果。

## 程式碼生成

大語言模型另外一個有效的應用場景是程式碼生成。在這方面，Copilot 就是一個很好的例子。下面你可以通過一些有效的提示詞讓程式碼生成任務。我們來看一下例子。

我們先用它寫一個簡單的用戶歡迎程序：


**Prompt**:

```bash
/*
Ask the user for their name and say "Hello"
*/
```

**Output**:

```bash
let name = prompt("What is your name?");
console.log(`Hello, ${name}!`);
```

你看，我們甚至都不需要指定要使用的編程語言。

來，我們再稍微升級一下。下面的例子將向你展示提示詞如何讓大型語言模型構建起來。

**Prompt**:

```bash
"""
Table departments, columns = [DepartmentId, DepartmentName]
Table students, columns = [DepartmentId, StudentId, StudentName]
Create a MySQL query for all students in the Computer Science Department
"""
```

**Output**:

```bash
SELECT StudentId, StudentName 
FROM students 
WHERE DepartmentId IN (SELECT DepartmentId FROM departments WHERE DepartmentName = 'Computer Science');
```

挺厲害的嘛。本例中，我們提供了有關數據庫架構並要求它生成有效的 MySQL 查詢。

## 推理

目前對於大語言模型來說，推理任務是最令人興奮的了。推理任務最讓人興奮的地方就是可以促使各種複雜的應用程序從大語言模型中誕生。

目前，涉及數學能力的推理任務已經有了一些改進。對於當前的大型語言模型來說，執行推理任務可能會有一些涉及，因此就需要更高級的提示詞工程技術。我們會在後面的指南中介紹這些高級技術。現在，我們將介紹一些基本示例來展示算術功能。


**Prompt**:

```bash
What is 9,000 * 9,000?
```

**Output**:

```bash
81,000,000
```

來，我們加大難度：

**Prompt**:

```bash
The odd numbers in this group add up to an even number: 15, 32, 5, 13, 82, 7, 1. 
A: 
```

**Output**:

```bash
No, the odd numbers in this group add up to an odd number: 119.
```

這結果不對！我們試著用提示詞去改進它：

**Prompt**:

```bash
The odd numbers in this group add up to an even number: 15, 32, 5, 13, 82, 7, 1. 
Solve by breaking the problem into steps. First, identify the odd numbers, add them, and indicate whether the result is odd or even. 
```

**Output**:

```bash
Odd numbers: 15, 5, 13, 7, 1
Sum: 41 
41 is an odd number.
```

好多了吧？順便說一句，我試過幾次，有時還是會失敗。如果你可以用示例給模型說明一下，可能會獲得更準確的結果。

我們後面還會在本章節中介紹更多常見應用示例。

在後面的章節，我們將介紹更高級的提示工程概念和技術，以完成更困難任務。