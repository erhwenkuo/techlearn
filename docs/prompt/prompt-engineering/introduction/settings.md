# 模型參數設置

使用提示詞時，您會通過 API 或直接與大語言模型進行交互。你可以通過配置一些參數以獲得不同的提示結果。

**Temperature**：簡單來說，`temperature` 的參數值越小，模型就會返回越確定的一個結果。如果調高該參數值，大語言模型可能會返回更隨機的結果，也就是說這可能會帶來更多樣化或更具創造性的產出。我們目前也在增加其他可能 token 的權重。在實際應用方面，對於質量保障（QA）等任務，我們可以設置更低的 `temperature` 值，以促使模型基於事實返回更真實和簡潔的結果。對於詩歌生成或其他創造性任務，你可以適當調高 `temperature` 參數值。

**Top_p**：同樣，使用 `top_p`（與 `temperature` 一起稱為核採樣的技術），可以用來控制模型返回結果的真實性。如果你需要準確和事實的答案，就把參數值調低。如果你想要更多樣化的答案，就把參數值調高一些。

一般建議是改變其中一個參數就行，不用兩個都調整。

在我們開始一些基礎範例之前，請記住最終生成的結果可能會和使用的大語言模型的版本而異。