# Recursively split by character

對於一般文本，建議使用此文本分割器。它由字元列表參數化。它嘗試按順序分割它們，直到區塊足夠小。預設清單為 `["\n\n", "\n", " ", ""]`。這樣做的效果是嘗試將所有段落（然後是句子，然後是單字）盡可能地放在一起，因為這些通常看起來是語義相關性最強的文本片段。

1. 文字如何分割：按字元清單。
2. 如何測量區塊大小：按字元數。

```python
# 這是一個很長的文檔，我們可以將其拆分。
with open('../../../state_of_the_union.txt') as f:
    state_of_the_union = f.read()

from langchain.text_splitter import RecursiveCharacterTextSplitter

text_splitter = RecursiveCharacterTextSplitter(
    # 設定一個非常小的區塊大小，只是為了範例顯示。
    chunk_size = 100,
    chunk_overlap  = 20,
    length_function = len,
    is_separator_regex = False,
)

texts = text_splitter.create_documents([state_of_the_union])

print(texts[0])
print(texts[1])
```

結果:

```python
page_content='Madam Speaker, Madam Vice President, our First Lady and Second Gentleman. Members of Congress and' lookup_str='' metadata={} lookup_index=0
page_content='of Congress and the Cabinet. Justices of the Supreme Court. My fellow Americans.' lookup_str='' metadata={} lookup_index=0
```

```python
text_splitter.split_text(state_of_the_union)[:2]
```

結果:

```python
['Madam Speaker, Madam Vice President, our First Lady and Second Gentleman. Members of Congress and',
    'of Congress and the Cabinet. Justices of the Supreme Court. My fellow Americans.']
```
