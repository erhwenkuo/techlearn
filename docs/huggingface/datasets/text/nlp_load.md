# Load text data

本指南向您展示如何加載文本數據集。要了解如何加載任何類型的數據集，請查看[一般加載](https://huggingface.co/docs/datasets/v2.14.1/en/loading)指南。

文本文件是存儲數據集最常見的文件類型之一。默認情況下，🤗 Datasets 逐行對文本文件進行採樣以構建數據集。

```python
from datasets import load_dataset

dataset = load_dataset("text", data_files={"train": ["my_text_1.txt", "my_text_2.txt"], "test": "my_test_file.txt"})

# Load from a directory
dataset = load_dataset("text", data_dir="path/to/text/dataset")
```

要按段落甚至整個文檔對文本文件進行採樣，請使用 `sample_by` 參數：

```python
# Sample by paragraph
dataset = load_dataset("text", data_files={"train": "my_train_file.txt", "test": "my_test_file.txt"}, sample_by="paragraph")

# Sample by document
dataset = load_dataset("text", data_files={"train": "my_train_file.txt", "test": "my_test_file.txt"}, sample_by="document")
```

您還可以使用 `grep` 模式來加載特定文件：

```python
from datasets import load_dataset

c4_subset = load_dataset("allenai/c4", data_files="en/c4-train.0000*-of-01024.json.gz")
```

要通過 HTTP 加載遠程文本文件，請傳遞 URL：

```python
dataset = load_dataset("text", data_files="https://huggingface.co/datasets/lhoestq/test/resolve/main/some_text.txt")
```

