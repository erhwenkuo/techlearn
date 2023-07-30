# Process text data

本指南展示了處理文本數據集的具體方法。了解如何：

- 使用 `map()` 對數據集進行 tokenize。
- 將數據集標籤與 NLI 數據集的標籤 ID 對齊。

有關如何處理任何類型的數據集的指南，請查看[一般流程](https://huggingface.co/docs/datasets/v2.14.1/en/process)指南。

## Map

`map()` 函數支持一次處理批量示例，從而加快 tokenization 速度。

從 🤗 Transformers 加載 tokenizer：

```python
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("bert-base-cased")
```

在 `map()` 函數中將 `batched` 參數設置為 `True`，將 tokenizer 應用於批量 examples：

```python
dataset = dataset.map(lambda examples: tokenizer(examples["text"]), batched=True)

dataset[0]
```

結果:

```bash
{'text': 'the rock is destined to be the 21st century\'s new " conan " and that he\'s going to make a splash even greater than arnold schwarzenegger , jean-claud van damme or steven segal .', 
 'label': 1, 
 'input_ids': [101, 1996, 2600, 2003, 16036, 2000, 2022, 1996, 7398, 2301, 1005, 1055, 2047, 1000, 16608, 1000, 1998, 2008, 2002, 1005, 1055, 2183, 2000, 2191, 1037, 17624, 2130, 3618, 2084, 7779, 29058, 8625, 13327, 1010, 3744, 1011, 18856, 19513, 3158, 5477, 4168, 2030, 7112, 16562, 2140, 1012, 102], 
 'token_type_ids': [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0], 
 'attention_mask': [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]}
```

`map()` 函數將返回的值轉換為 `PyArrow` 支持的格式。但顯式返回張量作為 NumPy 數組速度更快，因為它是原生支持的 PyArrow 格式。當您標記文本時設置 `return_tensors="np"`：

```python
dataset = dataset.map(lambda examples: tokenizer(examples["text"], return_tensors="np"), batched=True)
```

## Align

`align_labels_with_mapping()` 函數將數據集標籤 id 與標籤名稱對齊。並非所有🤗 Transformers 模型都遵循原始數據集規定的標籤映射，特別是對於 NLI 數據集。例如，[MNLI 數據集](https://huggingface.co/datasets/glue)使用以下標籤映射：

```python
label2id = {"entailment": 0, "neutral": 1, "contradiction": 2}
```

要將數據集標籤映射與模型使用的映射對齊，請創建要對齊的標籤名稱和 id 的字典：

```python
label2id = {"contradiction": 0, "neutral": 1, "entailment": 2}
```

將標籤映射的字典傳遞給 `align_labels_with_mapping()` 函數，以及要對齊的列：

```python
from datasets import load_dataset

mnli = load_dataset("glue", "mnli", split="train")

mnli_aligned = mnli.align_labels_with_mapping(label2id, "label")
```

您還可以使用此函數將標籤的自定義映射分配給 id。