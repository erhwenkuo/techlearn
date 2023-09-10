# Zero-Shot Prompting

如今，經過大量數據訓練並指令化調整的 LLM 能夠執行零樣本(zero-shot)任務。

我們在前一節中嘗試了一些零樣本示例。以下是我們使用的一個示例：

**Prompt**:

```console
Classify the text into neutral, negative or positive. 
Text: I think the vacation is okay.
Sentiment:
```

**Output**:

```console
Neutral
```

請注意，在上面的提示中，我們沒有向模型提供任何示例——這就是零樣本能力的作用。

指令調整已被證明可以改進零樣本學習 [Wei 等人（2022）](https://arxiv.org/pdf/2109.01652.pdf)。指令調整本質上是在通過指令的數據集上調節模型的概念。另外，[RLHF](https://arxiv.org/abs/1706.03741)已採用擴展指令調整，其中模型被調整以更好地適應人類偏好。這一最新發展推動了像 ChatGPT 這樣的模型。我們將在接下來的章節中討論所有這些方法和方法。

當零樣本失效時，建議在提示中提供演示或示例，這引出了少樣本提示。在下一節中，我們將演示少樣本​​提示。