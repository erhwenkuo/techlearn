# 語嵌入 (Embeddings)

OpenLLM 提供用於嵌入計算的嵌入端點。這可以通過 `/v1/embeddings` 訪問。

要通過 CLI 使用，只需調用 `openllm embed`：

```bash
openllm embed --endpoint http://localhost:3000 "I like to eat apples" -o json
{
  "embeddings": [
    0.006569798570126295,
    -0.031249752268195152,
    -0.008072729222476482,
    0.00847396720200777,
    -0.005293501541018486,
    ...<many embeddings>...
    -0.002078012563288212,
    -0.00676426338031888,
    -0.002022686880081892
  ],
  "num_tokens": 9
}
```

要調用此端點，請使用 Python SDK 中的 `client.embed`：

```python
import openllm

client = openllm.client.HTTPClient("http://localhost:3000")

client.embed("I like to eat apples")
```

!!! info
    目前，以下模型系列支持嵌入計算：Llama、T5（Flan-T5、FastChat 等）、ChatGLM 對於其餘沒有特定嵌入實現的 LLM，我們將使用通用的 BertModel 來生成嵌入。實現很大程度上基於 `[bentoml/sentence-embedding-bento](https://github.com/bentoml/sentence-embedding-bento)`

