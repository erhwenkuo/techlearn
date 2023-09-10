# 基本概念

## 基礎提示詞

您可以通過簡單的提示詞（Prompts）獲得大量結果，但結果的質量與您提供的信息數量和完善度有關。一個提示詞可以包含您傳遞到模型的 **instruction** 或 **question** 等信息，也可以包含其他詳細信息，如 **context**、**input** 或 **example**_等。您可以通過這些元素來更好地指導模型，並因此獲得更好的結果。

看下面一個簡單的示例：

**Prompt**:

```bash
The sky is
```

**Output**:

```bash
blue
The sky is blue on a clear day. On a cloudy day, the sky may be gray or white.
```

如以上示例，語言模型能夠基於我們給出的上下文內容 "The sky is" 完成句子接續。而輸出的結果可能是出人意料的，或遠高於我們的任務要求。

基於以上示例，如果想要實現更具體的目標，我們還必須提供更多的背景信息或說明信息。

可以按如下示例試著完善一下：

**Prompt**:

```bash
Complete the sentence: 
The sky is
```

**Output**:

```bash
so  beautiful today.
```

結果是不是要好一些了? 本例中，我們告知模型去完善句子，因此輸出的結果和我們最初的輸入是完全符合的。提示工程(Prompt Engineering)就是探討如何設計出最佳提示詞，用於指導語言模型幫助我們高效完成某項任務。

以上示例基本說明了現階段的大語言模型能夠發揮的功能作用。它們可以用於執行各種高級任務，如文本概括、數學推理、程式碼生成等。

## 提示詞格式

前文中我們還是採取的比較簡單的提示詞。標準提示詞應該遵循以下格式：

```bash
<Question>?
```

或

```bash
<Instruction>
```

這種可以被格式化為標準的問答格式，如：

```bash
Q: <Question>?
A: 
```

以上的提示方式，也被稱為 **零樣本提示(zero-shot prompting)**，即用戶不提供任務結果相關的示範，直接提示語言模型給出任務相關的回答。某些大型語言模式有能力實現零樣本提示，但這也取決於任務的複雜度和已有的知識範圍。

基於以上標準範式，目前業界普遍使用的還是更高效的 **小樣本提示(Few-shot Prompting)** 範式，即用戶提供少量的提示範例，如任務說明等。小樣本提示一般遵循以下格式：

```bash
<Question>?
<Answer>
<Question>?
<Answer>
<Question>?
<Answer>
<Question>?
```

而問答格式(QA format)如下所示：

```bash
Q: <Question>?
A: <Answer>
Q: <Question>?
A: <Answer>
Q: <Question>?
A: <Answer>
Q: <Question>?
A:
```

注意，使用問答格式並不是必須的。你可以根據任務需求調整提示範式。比如，您可以按以下示例執行一個簡單的分類任務，並對任務做簡單說明：

**Prompt**:

```bash
This is awesome! // Positive
This is bad! // Negative
Wow that movie was rad! // Positive
What a horrible show! //
```

**Output**:

```bash
Negative
```

語言模型可以基於一些說明了解和學習某些任務，而 **小樣本提示** 正好可以賦與上下文學習能力。