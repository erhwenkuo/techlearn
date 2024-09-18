# 導入客制模型

本文將引導您匯入 GGUF、PyTorch 或 Safetensors 模型。

## 導入 (GGUF)

### Step 1: Write a Modelfile

首先建立一個 `Modelfile` 檔案。該檔案是模型的藍圖，指定模型的權重、參數、提示範本等。

```bash
FROM ./mistral-7b-v0.1.Q4_0.gguf
```

(Optional) 許多 chat 模型需要 prompt 模板才能正確回答。可以使用 `Modelfile` 中的 TEMPLATE 指令指定預設提示範本：

```bash
FROM ./mistral-7b-v0.1.Q4_0.gguf

TEMPLATE "[INST] {{ .Prompt }} [/INST]"
```

### Step 2: Create the Ollama model

最後，使用 `ollama` 的命例來從 `Modelfile` 建立一個模型：

```bash
ollama create example -f Modelfile
```

### Step 3: Run your model

接下來，使用 `ollama run` 測試模型：

```bash
ollama run example "What is your favourite condiment?"
```