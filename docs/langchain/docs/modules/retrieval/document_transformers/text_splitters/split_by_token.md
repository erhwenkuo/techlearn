# Split by tokens

語言模型有一個 token 數的限制。您不應超出 token 限制。因此，當您將文字拆分為區塊時，最好計算 token 的數量。當您計算文字中的 token 數時，您應該使用與語言模型中使用的相同的 tokenizer。

## tiktoken

---

> [tiktoken](https://github.com/openai/tiktoken) 是 `OpenAI` 創建的快速 `BPE` tokenizer。

我們可以用它來估計所使用的 token。對於 OpenAI 模型來說，它可能會更準確。

- 文字如何分割：按傳入的字元。
- 如何測量區塊大小：透過 `tiktoken tokenizer`。

```bash
pip install tiktoken
```

```python
# 這是一個很長的文檔，我們可以將其拆分。
with open("../../../state_of_the_union.txt") as f:
    state_of_the_union = f.read()

from langchain.text_splitter import CharacterTextSplitter
```

**API Reference:**

- [CharacterTextSplitter](https://api.python.langchain.com/en/latest/text_splitter/langchain.text_splitter.CharacterTextSplitter.html)

```python
text_splitter = CharacterTextSplitter.from_tiktoken_encoder(
    chunk_size=100, chunk_overlap=0
)

texts = text_splitter.split_text(state_of_the_union)

print(texts[0])
```

結果:

```python
Madam Speaker, Madam Vice President, our First Lady and Second Gentleman. Members of Congress and the Cabinet. Justices of the Supreme Court. My fellow Americans.  

Last year COVID-19 kept us apart. This year we are finally together again. 

Tonight, we meet as Democrats Republicans and Independents. But most importantly as Americans. 

With a duty to one another to the American people to the Constitution.
```

!!! info
    請注意，如果我們使用 `CharacterTextSplitter.from_tiktoken_encoder`，則文字僅由 `CharacterTextSplitter` 分割，並且 `tiktoken` 分詞器用於合併分割。這意味著 split 可以大於 `tiktoken` tokenizer 測量的區塊大小。
    
    我們可以使用 `RecursiveCharacterTextSplitter.from_tiktoken_encoder` 來確保分割不大於語言模型允許的標記區塊大小，其中如果每個分割具有更大的大小，則每個分割將被遞歸分割。

我們也可以直接載入一個 `tiktoken` splitter，這確保每個 split 都小於 chunk 大小。

```python
from langchain.text_splitter import TokenTextSplitter

text_splitter = TokenTextSplitter(chunk_size=10, chunk_overlap=0)

texts = text_splitter.split_text(state_of_the_union)

print(texts[0])
```

**API Reference:**

- [TokenTextSplitter](https://api.python.langchain.com/en/latest/text_splitter/langchain.text_splitter.TokenTextSplitter.html)


## spaCy

---

> [spaCy](https://spacy.io/) 是一個用於高階自然語言處理的開源軟體套件，以程式語言 Python 和 Cython 編寫。

NLTK 的另一種替代方法是使用 [spaCy tokenizer](https://spacy.io/api/tokenizer)。

1. 文本如何分割：透過 `spaCy` tokenizer。
2. 如何測量區塊大小：按字元數。

```bash
pip install spacy
```

```python
# This is a long document we can split up.
with open("../../../state_of_the_union.txt") as f:
    state_of_the_union = f.read()

from langchain.text_splitter import SpacyTextSplitter

text_splitter = SpacyTextSplitter(chunk_size=1000)
```

**API Reference:**

- [SpacyTextSplitter](https://api.python.langchain.com/en/latest/text_splitter/langchain.text_splitter.SpacyTextSplitter.html)

```python
texts = text_splitter.split_text(state_of_the_union)

print(texts[0])
```

結果:

```bash
    Madam Speaker, Madam Vice President, our First Lady and Second Gentleman.
    
    Members of Congress and the Cabinet.
    
    Justices of the Supreme Court.
    
    My fellow Americans.  
    
    
    
    Last year COVID-19 kept us apart.
    
    This year we are finally together again. 
    
    
    
    Tonight, we meet as Democrats Republicans and Independents.
    
    But most importantly as Americans. 
    
    
    
    With a duty to one another to the American people to the Constitution. 
    
    
    
    And with an unwavering resolve that freedom will always triumph over tyranny. 
    
    
    
    Six days ago, Russia’s Vladimir Putin sought to shake the foundations of the free world thinking he could make it bend to his menacing ways.
    
    But he badly miscalculated. 
    
    
    
    He thought he could roll into Ukraine and the world would roll over.
    
    Instead he met a wall of strength he never imagined. 
    
    
    
    He met the Ukrainian people. 
    
    
    
    From President Zelenskyy to every Ukrainian, their fearlessness, their courage, their determination, inspires the world.
```

## SentenceTransformers

---

`SentenceTransformersTokenTextSplitter` 是一個專門的文字分割器，用於句子轉換器模型。預設行為是將文字分割成適合您要使用的句子轉換器模型的標記視窗的區塊。

```python
from langchain.text_splitter import SentenceTransformersTokenTextSplitter
```

**API Reference:**

- [SentenceTransformersTokenTextSplitter](https://api.python.langchain.com/en/latest/text_splitter/langchain.text_splitter.SentenceTransformersTokenTextSplitter.html)

```python
splitter = SentenceTransformersTokenTextSplitter(chunk_overlap=0)

text = "Lorem "

count_start_and_stop_tokens = 2

text_token_count = splitter.count_tokens(text=text) - count_start_and_stop_tokens

print(text_token_count)

token_multiplier = splitter.maximum_tokens_per_chunk // text_token_count + 1

# `text_to_split` does not fit in a single chunk
text_to_split = text * token_multiplier

print(f"tokens in text to split: {splitter.count_tokens(text=text_to_split)}")

text_chunks = splitter.split_text(text=text_to_split)

print(text_chunks[1])
```

## NLTK

---

> [自然語言工具包](https://en.wikipedia.org/wiki/Natural_Language_Toolkit)（或更常見的是 [NLTK](https://www.nltk.org/)）是一套用 Python 程式語言編寫的用於英語符號和統計自然語言處理 (NLP) 的函式套件和程式。

我們可以使用 `NLTK` 基於 [NLTK tokenizer](https://www.nltk.org/api/nltk.tokenize.html)進行分割，而不是僅根據 `\n\n` 進行分割。

1. 文本如何分割：通過 `NLTK` tokenizer。
2. 如何測量區塊大小：按字元數。

```bash
pip install nltk
```

```python
# This is a long document we can split up.
with open("../../../state_of_the_union.txt") as f:
    state_of_the_union = f.read()

from langchain.text_splitter import NLTKTextSplitter

text_splitter = NLTKTextSplitter(chunk_size=1000)
```

**API Reference:**

- [NLTKTextSplitter](https://api.python.langchain.com/en/latest/text_splitter/langchain.text_splitter.NLTKTextSplitter.html)


```python
texts = text_splitter.split_text(state_of_the_union)

print(texts[0])
```

結果:

```python
    Madam Speaker, Madam Vice President, our First Lady and Second Gentleman.
    
    Members of Congress and the Cabinet.
    
    Justices of the Supreme Court.
    
    My fellow Americans.
    
    Last year COVID-19 kept us apart.
    
    This year we are finally together again.
    
    Tonight, we meet as Democrats Republicans and Independents.
    
    But most importantly as Americans.
    
    With a duty to one another to the American people to the Constitution.
    
    And with an unwavering resolve that freedom will always triumph over tyranny.
    
    Six days ago, Russia’s Vladimir Putin sought to shake the foundations of the free world thinking he could make it bend to his menacing ways.
    
    But he badly miscalculated.
    
    He thought he could roll into Ukraine and the world would roll over.
    
    Instead he met a wall of strength he never imagined.
    
    He met the Ukrainian people.
    
    From President Zelenskyy to every Ukrainian, their fearlessness, their courage, their determination, inspires the world.
    
    Groups of citizens blocking tanks with their bodies.
```

## Hugging Face tokenizer

---

> [Hugging Face](https://huggingface.co/docs/tokenizers/index) 有很多 tokenizer。

我們使用 Hugging Face tokenizer（[GPT2TokenizerFast](https://huggingface.co/Ransaka/gpt2-tokenizer-fast)）來計算標記中的文字長度。

1. 文字如何分割：按傳入的字元。
2. 如何測量區塊大小：透過 `Hugging Face` tokenizer 計算的 token 數量。

```python
from transformers import GPT2TokenizerFast

tokenizer = GPT2TokenizerFast.from_pretrained("gpt2")

# This is a long document we can split up.
with open("../../../state_of_the_union.txt") as f:
    state_of_the_union = f.read()

from langchain.text_splitter import CharacterTextSplitter
```

**API Reference:**

- [CharacterTextSplitter](https://api.python.langchain.com/en/latest/text_splitter/langchain.text_splitter.CharacterTextSplitter.html)

```python
text_splitter = CharacterTextSplitter.from_huggingface_tokenizer(
    tokenizer, chunk_size=100, chunk_overlap=0
)
texts = text_splitter.split_text(state_of_the_union)

print(texts[0])
```

結果:

```
Madam Speaker, Madam Vice President, our First Lady and Second Gentleman. Members of Congress and the Cabinet. Justices of the Supreme Court. My fellow Americans.  

Last year COVID-19 kept us apart. This year we are finally together again. 

Tonight, we meet as Democrats Republicans and Independents. But most importantly as Americans. 

With a duty to one another to the American people to the Constitution.
```
