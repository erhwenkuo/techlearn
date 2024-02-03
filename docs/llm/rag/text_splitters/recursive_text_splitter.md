# 按字元遞歸分割

原文: [Recursively split by character](https://python.langchain.com/docs/modules/data_connection/document_transformers/recursive_text_splitter)

對於一般文本，建議使用此文本分割器。它由字元列表參數化。它嘗試按順序分割它們，直到區塊足夠小。預設清單為 `["\n\n", "\n", " ", ""]`。這樣做的效果是嘗試將所有段落（然後是句子，然後是單字）在長度限制的允許下盡可能地放在一起，因為這些通常看起來是語義相關性最強的文本片段。

1. **文字如何分割**: 按字元清單。
2. **如何測量區塊(chunk)大小**: 按字元數。

