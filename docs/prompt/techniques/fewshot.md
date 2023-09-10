# Few-Shot Prompting

雖然大型語言模型展示了驚人的零樣本能力，但在使用零樣本設置時，它們在更複雜的任務上仍然表現不佳。少樣本提示可以作為一種技術，以啟用上下文學習，我們在提示中提供演示以引導模型實現更好的性能。演示作為後續示例的條件，我們希望模型生成響應。

讓我們通過 [Brown 等人 2020年](https://arxiv.org/pdf/2302.13971.pdf)提出的一個例子來演示少樣本提示。在這個例子中，任務是在句子中正確使用一個新詞。

**Prompt**:

```console
A "whatpu" is a small, furry animal native to Tanzania. An example of a sentence that uses
the word whatpu is:
We were traveling in Africa and we saw these very cute whatpus.
To do a "farduddle" means to jump up and down really fast. An example of a sentence that uses
the word farduddle is:
```

**Output**:

```console
When we won the game, we all started to farduddle in celebration.
```

我們可以觀察到，模型通過提供一個示例（即 `1-shot`）已經學會瞭如何執行任務。對於更困難的任務，我們可以嘗試增加演示（例如 `3-shot`、`5-shot`、`10-shot` 等）。

根據 [Min 等人（2022）](https://arxiv.org/abs/2202.12837)的研究結果，以下是在進行少樣本學習時關於演示/範例的一些額外提示：

- “標籤空間和演示指定的輸入文本的分佈都很重要（無論標籤是否對單個輸入正確）”
- 使用的格式也對性能起著關鍵作用，即使只是使用隨機標籤，這也比沒有標籤好得多。
- 其他結果表明，從真實標籤分佈（而不是均勻分佈）中選擇隨機標籤也有幫助。

讓我們嘗試一些例子。讓我們首先嘗試一個隨機標籤的例子(意味著將標籤 Negative 和 Positive 隨機分配給輸入)：

**Prompt**:

```console
This is awesome! // Negative
This is bad! // Positive
Wow that movie was rad! // Positive
What a horrible show! //
```

**Output**:

```console
Negative
```

即使標籤已經隨機化，我們仍然得到了正確的答案。請注意，我們還保留了格式，這也有所幫助。實際上，通過進一步的實驗，我們發現我們正在嘗試的新GPT模型甚至對隨機格式也變得更加穩健。例如：

**Prompt**:

```console
Positive This is awesome! 
This is bad! Negative
Wow that movie was rad!
Positive
What a horrible show! --
```

**Output**:

```console
Negative
```

上面的格式不一致，但模型仍然預測了正確的標籤。我們必須進行更徹底的分析，以確認這是否適用於不同和更複雜的任務，包括提示的不同變體。

## 少樣本提示的限制

標準的少樣本提示對許多任務都有效，但仍然不是一種完美的技術，特別是在處理更複雜的推理任務時。讓我們演示為什麼會這樣。您是否還記得之前提供的任務：

**Prompt**:

```console
The odd numbers in this group add up to an even number: 15, 32, 5, 13, 82, 7, 1. 
A: 
```

**Output**:

```console
Yes, the odd numbers in this group add up to 107, which is an even number.
```

這不是正確的答案，這不僅突顯了這些系統的局限性，而且需要更高級的提示工程。

讓我們嘗試添加一些示例，看看少樣本提示是否可以改善結果。

**Prompt**:

```console
The odd numbers in this group add up to an even number: 4, 8, 9, 15, 12, 2, 1.
A: The answer is False.
The odd numbers in this group add up to an even number: 17,  10, 19, 4, 8, 12, 24.
A: The answer is True.
The odd numbers in this group add up to an even number: 16,  11, 14, 4, 8, 13, 24.
A: The answer is True.
The odd numbers in this group add up to an even number: 17,  9, 10, 12, 13, 4, 2.
A: The answer is False.
The odd numbers in this group add up to an even number: 15, 32, 5, 13, 82, 7, 1. 
A: 
```

**Output**:

```console
The answer is True.
```

這沒用。似乎少樣本提示不足以獲得這種類型的推理問題的可靠響應。上面的示例提供了任務的基本信息。如果您仔細觀察，我們引入的任務類型涉及幾個更多的推理步驟。換句話說，如果我們將問題分解成步驟並向模型演示，這可能會有所幫助。最近，[思維鏈（CoT）提示](https://arxiv.org/abs/2201.11903)已經流行起來，以解決更複雜的算術、常識和符號推理任務。

總的來說，提供示例對解決某些任務很有用。當零樣本提示和少樣本提示不足時，這可能意味著模型學到的東西不足以在任務上表現良好。從這裡開始，建議開始考慮微調您的模型或嘗試更高級的提示技術。接下來，我們將討論一種流行的提示技術，稱為思維鏈提示(chain-of-thought prompting)，它已經獲得了很多關注。

