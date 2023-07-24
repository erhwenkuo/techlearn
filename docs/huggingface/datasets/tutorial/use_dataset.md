# Preprocess

除了加載數據集之外，🤗 數據集的另一個主要目標是提供一組多樣化的預處理函數，將數據集轉換為適當的格式，以便使用機器學習框架進行訓練。

預處理數據集的方法有很多種，這完全取決於您的具體數據集。有時您可能需要重命名列，有時您可能需要取消展平嵌套字段。 🤗 數據集提供了一種完成大部分這些事情的方法。但在幾乎所有預處理情況下，根據您的數據集模式，您需要：

- 對文本數據集進行 tokenize。
- 對音頻數據集重新採樣。
- 將變換應用於圖像數據集。

{==最後的預處理步驟通常是將數據集格式設置為與機器學習框架的預期輸入格式兼容==}。

在本教程中，您還需要安裝 🤗 Transformers 套件：

```python
pip install transformers
```

##　Tokenize text

模型無法處理原始文本，因此您需要將文本轉換為數字。Tokenization 提供了一種通過將文本切割成為 token 來實現此目的的方法。Token 最終會轉換為數字。

!!! tip
    查看 Hugging Face 課程第 2 章中的 [Tokenizers](https://huggingface.co/course/chapter2/4?fw=pt) 部分，了解有關 tokenization 和不同的 tokenization 演算法的更多信息。

1. 首先加載 [rotten_tomatoes](https://huggingface.co/datasets/rotten_tomatoes) 數據集和與預訓練 [BERT 模型](https://huggingface.co/bert-base-uncased)對應的分詞器。使用與預訓練模型相同的分詞器非常重要，因為您希望確保文本以相同的方式分割。

    ```python
    from transformers import AutoTokenizer
    from datasets import load_dataset

    tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")
    dataset = load_dataset("rotten_tomatoes", split="train")
    ```

2. 對數據集中第一行文本調用 tokenizer：

    ```python
    tokenizer(dataset[0]["text"])
    ```

    結果:

    ```bash
    {'input_ids': [101, 1103, 2067, 1110, 17348, 1106, 1129, 1103, 6880, 1432, 112, 188, 1207, 107, 14255, 1389, 107, 1105, 1115, 1119, 112, 188, 1280, 1106, 1294, 170, 24194, 1256, 3407, 1190, 170, 11791, 5253, 188, 1732, 7200, 10947, 12606, 2895, 117, 179, 7766, 118, 172, 15554, 1181, 3498, 6961, 3263, 1137, 188, 1566, 7912, 14516, 6997, 119, 102], 
    'token_type_ids': [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0], 
    'attention_mask': [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]}
    ```

    分詞器返回一個包含三個項目的字典：

    - `input_ids`：代表文本中 token 的 id。
    - `token_type_ids`：如果有多個序列，則表示該 token 屬於哪個序
    - `token_type_ids`：如果有多個序列，則表示該 token 屬於哪個序列。
    - `attention_mask`：指示是否應 masked token。

    這些值實際上是模型輸入。

3. 對整個數據集進行 tokenize 的最快方法是使用 `map()` 函數。此函數通過將 tokenizer 應用於批量數據而不是單個數據來加速分詞。將批處理參數設置為 `True`：

    ```python
    def tokenization(example):
        return tokenizer(example["text"])

    dataset = dataset.map(tokenization, batched=True)
    ```

4. 設置數據集的格式以與您的機器學習框架兼容：

    !!! tip "Pytorch"

        使用 `set_format()` 函數設置數據集格式以與 PyTorch 兼容：

        ```python
        dataset.set_format(type="torch", columns=["input_ids", "token_type_ids", "attention_mask", "labels"])

        dataset.format['type']
        ```

    !!! tip "Tensorflow"

        使用 `to_tf_dataset()` 函數設置數據集格式以與 TensorFlow 兼容。您還需要從 🤗 Transformers 導入 data collator，以將不同的序列長度組合成一批相同長度的：

        ```python
        from transformers import DataCollatorWithPadding

        data_collator = DataCollatorWithPadding(tokenizer=tokenizer, return_tensors="tf")
        tf_dataset = dataset.to_tf_dataset(
            columns=["input_ids", "token_type_ids", "attention_mask"],
            label_cols=["labels"],
            batch_size=2,
            collate_fn=data_collator,
            shuffle=True
        )
        ```

5. 數據集現已準備好使用您的機器學習框架進行訓練！

## Resample audio signals

像文本數據集這樣的音頻輸入需要分為離散的數據點。這稱為抽樣；採樣率告訴您每秒捕獲多少語音信號。確保數據集的採樣率與用於預訓練您正在使用的模型的數據的採樣率相匹配非常重要。如果採樣率不同，預訓練模型可能在數據集上表現不佳，因為它無法識別採樣率的差異。

1. 首先加載 [MInDS-14](https://huggingface.co/datasets/PolyAI/minds14) 數據集、音頻特徵以及與預訓練的 [Wav2Vec2 模型](https://huggingface.co/facebook/wav2vec2-base-960h)對應的特徵提取器：

    ```python
    from transformers import AutoFeatureExtractor
    from datasets import load_dataset, Audio

    feature_extractor = AutoFeatureExtractor.from_pretrained("facebook/wav2vec2-base-960h")
    dataset = load_dataset("PolyAI/minds14", "en-US", split="train")
    ```

2. 索引數據集的第一行。當您調用數據集的音頻列時，它會自動解碼並重新採樣：

    ```python
    dataset[0]["audio"]
    ```

    結果:

    ```bash
    {'array': array([ 0.        ,  0.00024414, -0.00024414, ..., -0.00024414,
            0.        ,  0.        ], dtype=float32),
    'path': '/root/.cache/huggingface/datasets/downloads/extracted/f14948e0e84be638dd7943ac36518a4cf3324e8b7aa331c5ab11541518e9368c/en-US~JOINT_ACCOUNT/602ba55abb1e6d0fbce92065.wav',
    'sampling_rate': 8000}
    ```

3. 閱讀數據集訊息卡非常有用，可以為您提供有關數據集的關鍵信息。快速瀏覽一下 MInDS-14 數據集訊息卡，您會發現採樣率為 8kHz。同樣，您可以從模型訊息卡中獲取有關模型的許多詳細信息。 Wav2Vec2 模型訊息卡表示它是在 16kHz 語音音頻上採樣的。這意味著您需要對 MInDS-14 數據集進行上採樣以匹配模型的採樣率。

    使用 `cast_column()` 函數並設置音頻功能中的 `sampling_rate` 參數來對音頻信號進行上採樣。當您現在調用音頻數據列時，它會被解碼並重新採樣到 16kHz：

    ```python
    dataset = dataset.cast_column("audio", Audio(sampling_rate=16_000))

    dataset[0]["audio"]
    ```

    結果:

    ```bash
    {'array': array([ 2.3443763e-05,  2.1729663e-04,  2.2145823e-04, ...,
            3.8356509e-05, -7.3497440e-06, -2.1754686e-05], dtype=float32),
    'path': '/root/.cache/huggingface/datasets/downloads/extracted/f14948e0e84be638dd7943ac36518a4cf3324e8b7aa331c5ab11541518e9368c/en-US~JOINT_ACCOUNT/602ba55abb1e6d0fbce92065.wav',
    'sampling_rate': 16000}
    ```

4. 使用  `map()` 函數將整個數據集重新採樣至 16kHz。該函數通過將特徵提取器應用於批量樣本數據而不是單個樣本數據來加速重採樣。將批處理參數設置為 `True`：

    ```python
    def preprocess_function(examples):
        audio_arrays = [x["array"] for x in examples["audio"]]
        inputs = feature_extractor(
            audio_arrays, sampling_rate=feature_extractor.sampling_rate, max_length=16000, truncation=True
        )
        return inputs

    dataset = dataset.map(preprocess_function, batched=True)
    ```

5. 數據集現已準備好使用您的機器學習框架進行訓練！

## Apply data augmentations

對圖像數據集進行的最常見的預處理是 **數據增強**，這是一個在不改變數據含義的情況下向圖像引入隨機變化的過程。這可能意味著更改圖像的顏色屬性或隨機裁剪圖像。您可以自由使用您喜歡的任何數據增強庫，🤗 數據集將幫助您將數據增強應用到數據集。

1. 首先加載 [Beans](https://huggingface.co/datasets/beans) 數據集、Image 特徵以及與預訓練 [ViT 模型](https://huggingface.co/google/vit-base-patch16-224-in21k)對應的特徵提取器：

    ```python
    from transformers import AutoFeatureExtractor
    from datasets import load_dataset, Image

    feature_extractor = AutoFeatureExtractor.from_pretrained("google/vit-base-patch16-224-in21k")

    dataset = load_dataset("beans", split="train")
    ```

2. 索引數據集的第一行。當您調用數據集的圖像列時，底層 PIL 對象會自動解碼為圖像。

```python
dataset[0]["image"]
```

3. 現在，您可以對圖像應用一些變換。請隨意查看 torchvision 中提供的[各種變換](https://pytorch.org/vision/stable/auto_examples/plot_transforms.html#sphx-glr-auto-examples-plot-transforms-py)，然後選擇您想要嘗試的變換。此示例應用隨機旋轉圖像的變換：

    ```python
    from torchvision.transforms import RandomRotation

    rotate = RandomRotation(degrees=(0, 90))

    def transforms(examples):
        examples["pixel_values"] = [rotate(image.convert("RGB")) for image in examples["image"]]
        return examples
    ```

4. 使用 `set_transform()` 函數即時應用變換。當您索引圖像像素值時，會應用變換，並且圖像會旋轉。

    ```python
    dataset.set_transform(transforms)
    dataset[0]["pixel_values"]
    ```

5. 數據集現已準備好使用您的機器學習框架進行訓練！
