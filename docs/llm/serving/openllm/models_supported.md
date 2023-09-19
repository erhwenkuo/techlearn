# OpenLLM 支持的模型

OpenLLM 目前支持以下模型。默認情況下，OpenLLM 不包含運行所有模型的依賴項。可以按照以下說明安裝額外的特定於模型所需要的依賴套件。

## Llama

**安裝**:

要使用 OpenLLM 運行 Llama 模型，您需要安裝 llama 依賴項，因為默認情況下未安裝它。

```python
pip install openllm[llama]
```

**快速開始**:

運行以下命令快速啟動 `Llama 2` 服務器並向其發送請求。

```bash
openllm start llama --model-id meta-llama/Llama-2-7b-chat-hf
```

或是使用量化模型(`--quantize int8` 或 `--quantize int4`):

```bash
openllm start llama --model-id meta-llama/Llama-2-7b-chat-hf --quantize int4

openllm start llama --model-id ziqingyang/chinese-alpaca-2-7b --quantize int4 --serialisation legacy
```

或是使用 GTPQ 模型:

```bash
openllm start llama --model-id TheBloke/Llama-2-7b-Chat-GPTQ --quantize gptq
```

!!! tip
    要運行 `GPTQ` 量化過的模型必需要額外安裝下列的依賴套件:

    ```bash
    pip install openllm[gptq] --extra-index-url https://huggingface.github.io/autogptq-index/whl/cu118/ 
    ```

檢查:

```bash
export OPENLLM_ENDPOINT=http://localhost:3000

openllm query 'What are large language models?'
```

!!! info
    要使用官方 Llama 2 模型，您必須通過訪問 [Meta AI 網站](https://ai.meta.com/resources/models-and-libraries/llama-downloads/)並接受其許可條款和可接受的使用政策來獲得訪問權限。您還需要在 Hugging Face 上獲取這些模型的訪問權限。請注意，如果您可訪問官方 Llama 2 模型，則任何 Llama 2 變體都可以使用 OpenLLM 進行部署。訪問 [Hugging Face 模型中心](https://huggingface.co/models?sort=trending&search=llama2)查看更多 Llama 2 兼容模型。

### 模型支持

您可以使用 `--model-id` 指定以下任何 Llama 模型。

- [meta-llama/Llama-2-70b-chat-hf](https://huggingface.co/meta-llama/Llama-2-70b-chat-hf)
- [meta-llama/Llama-2-13b-chat-hf](https://huggingface.co/meta-llama/Llama-2-13b-chat-hf)
- [meta-llama/Llama-2-7b-chat-hf](https://huggingface.co/meta-llama/Llama-2-7b-chat-hf)
- [meta-llama/Llama-2-70b-hf](https://huggingface.co/meta-llama/Llama-2-70b-hf)
- [meta-llama/Llama-2-13b-hf](https://huggingface.co/meta-llama/Llama-2-13b-hf)
- [meta-llama/Llama-2-7b-hf](https://huggingface.co/meta-llama/Llama-2-7b-hf)
- [NousResearch/llama-2-70b-chat-hf](https://huggingface.co/NousResearch/llama-2-70b-chat-hf)
- [NousResearch/llama-2-13b-chat-hf](https://huggingface.co/NousResearch/llama-2-13b-chat-hf)
- [NousResearch/llama-2-7b-chat-hf](https://huggingface.co/NousResearch/llama-2-7b-chat-hf)
- [NousResearch/llama-2-70b-hf](https://huggingface.co/NousResearch/llama-2-70b-hf)
- [NousResearch/llama-2-13b-hf](https://huggingface.co/NousResearch/llama-2-13b-hf)
- [NousResearch/llama-2-7b-hf](https://huggingface.co/NousResearch/llama-2-7b-hf)
- [openlm-research/open_llama_7b_v2](https://huggingface.co/openlm-research/open_llama_7b_v2)
- [openlm-research/open_llama_3b_v2](https://huggingface.co/openlm-research/open_llama_3b_v2)
- [openlm-research/open_llama_13b](https://huggingface.co/openlm-research/open_llama_13b)
- [huggyllama/llama-65b](https://huggingface.co/huggyllama/llama-65b)
- [huggyllama/llama-30b](https://huggingface.co/huggyllama/llama-30b)
- [huggyllama/llama-13b](https://huggingface.co/huggyllama/llama-13b)
- [huggyllama/llama-7b](https://huggingface.co/huggyllama/llama-7b)
- 任何其他嚴格遵循 [LlamaForCausalLM](https://huggingface.co/docs/transformers/main/model_doc/llama#transformers.LlamaForCausalLM) 架構的模型

### 推論後台支援

- PyTorch (Default):

    ```bash
    openllm start llama --model-id meta-llama/Llama-2-7b-chat-hf --backend pt
    ```

- vLLM (Recommended):


    ```bash
    pip install "openllm[llama, vllm]"

    openllm start llama --model-id meta-llama/Llama-2-7b-chat-hf --backend vllm
    ```

    !!! info
        目前，使用 vLLM 後端時，不支持量化(quantization)和適配器(adapter)。

## ChatGLM

**安裝**:

要使用 OpenLLM 運行 ChatGLM 模型，您需要安裝 `chatglm` 依賴項，因為默認情況下未安裝它。

```python
pip install openllm[chatglm]
```

**快速開始**:

運行以下命令快速啟動 `ChatGLM` 服務器並向其發送請求。

```bash
openllm start chatglm --model-id thudm/chatglm-6b
```

檢查:

```bash
export OPENLLM_ENDPOINT=http://localhost:3000

openllm query 'What are large language models?'
```

### 模型支持

您可以使用 `--model-id` 指定以下任何 ChatGLM 模型。

- [thudm/chatglm-6b](https://huggingface.co/thudm/chatglm-6b)
- [thudm/chatglm-6b-int8](https://huggingface.co/thudm/chatglm-6b-int8)
- [thudm/chatglm-6b-int4](https://huggingface.co/thudm/chatglm-6b-int4)
- [thudm/chatglm2-6b](https://huggingface.co/thudm/chatglm2-6b)
- [thudm/chatglm2-6b-int4](https://huggingface.co/thudm/chatglm2-6b-int4)
- 嚴格遵循 [ChatGLMForConditionalGeneration](https://github.com/THUDM/ChatGLM-6B) 架構的任何其他模型

### 推論後台支援

- PyTorch (Default):

    ```bash
    openllm start chatglm --model-id thudm/chatglm-6b --backend pt
    ```

## Dolly-v2

**安裝**:

安裝 openllm 後，`Dolly-v2` 模型不需要安裝任何特定於模型的依賴項。

```python
pip install openllm
```

**快速開始**:

運行以下命令快速啟動 `Dolly-v2` 服務器並向其發送請求。

```bash
openllm start dolly-v2 --model-id databricks/dolly-v2-3b
```

檢查:

```bash
export OPENLLM_ENDPOINT=http://localhost:3000

openllm query 'What are large language models?'
```

### 模型支持

您可以使用 `--model-id` 指定以下任何 Dolly-v2 模型。

- [databricks/dolly-v2-3b](https://huggingface.co/databricks/dolly-v2-3b)
- [databricks/dolly-v2-7b](https://huggingface.co/databricks/dolly-v2-7b)
- [databricks/dolly-v2-12b](https://huggingface.co/databricks/dolly-v2-12b)
- 嚴格遵循 [GPTNeoXForCausalLM](https://huggingface.co/docs/transformers/main/model_doc/gpt_neox#transformers.GPTNeoXForCausalLM) 架構的任何其他模型

### 推論後台支援

- PyTorch (Default):

    ```bash
    openllm start dolly-v2 --model-id databricks/dolly-v2-3b --backend pt
    ```

- vLLM (Recommended):


    ```bash
    pip install "openllm[vllm]"

    openllm start dolly-v2 --model-id databricks/dolly-v2-3b --backend vllm
    ```

    !!! info
        目前，使用 vLLM 後端時，不支持量化(quantization)和適配器(adapter)。


## Falcon

**安裝**:

要使用 OpenLLM 運行 Falcon 模型，您需要安裝 `falcon` 依賴項，因為默認情況下未安裝它。

```python
pip install openllm[falcon]
```

**快速開始**:

運行以下命令快速啟動 `Falcon` 服務器並向其發送請求。

```bash
openllm start falcon --model-id tiiuae/falcon-7b --serialisation legacy
```

或是使用量化模型(`--quantize int8` 或 `--quantize int4`):

```bash
openllm start falcon --model-id tiiuae/falcon-7b --quantize int4 --serialisation legacy
```

或是使用 GTPQ 模型:

```bash
openllm start falcon --model-id TheBloke/falcon-7b-instruct-GPTQ --quantize gptq
```

!!! tip
    要運行 `GPTQ` 量化過的模型必需要額外安裝下列的依賴套件:

    ```bash
    pip install openllm[gptq] --extra-index-url https://huggingface.github.io/autogptq-index/whl/cu118/ 
    ```

檢查:

```bash
export OPENLLM_ENDPOINT=http://localhost:3000

openllm query 'What are large language models?'
```

### 模型支持

您可以使用 `--model-id` 指定以下任何 Falcon 模型。

- [tiiuae/falcon-7b](https://huggingface.co/tiiuae/falcon-7b)
- [tiiuae/falcon-40b](https://huggingface.co/tiiuae/falcon-40b)
- [tiiuae/falcon-7b-instruct](https://huggingface.co/tiiuae/falcon-7b-instruct)
- [tiiuae/falcon-40b-instruct](https://huggingface.co/tiiuae/falcon-40b-instruct)
- 任何其他嚴格遵循 [FalconForCausalLM](https://falconllm.tii.ae/) 架構的模型

### 推論後台支援

- PyTorch (Default):

    ```bash
    openllm start falcon --model-id tiiuae/falcon-7b --backend pt
    ```

- vLLM (Recommended):


    ```bash
    pip install "openllm[falcon, vllm]"

    openllm start falcon --model-id tiiuae/falcon-7b --backend vllm
    ```

    !!! info
        目前，使用 vLLM 後端時，不支持量化(quantization)和適配器(adapter)。


## Flan-T5

**安裝**:

要使用 OpenLLM 運行 Flan-T5 模型，您需要安裝 `flan-t5` 依賴項，因為默認情況下未安裝它。

```python
pip install openllm[flan-t5]
```

**快速開始**:

運行以下命令快速啟動 `Flan-T5` 服務器並向其發送請求。

```bash
openllm start flan-t5 --model-id google/flan-t5-large
```

或是使用量化模型(`--quantize int8` 或 `--quantize int4`):

```bash
openllm start flan-t5 --model-id google/flan-t5-large --quantize int4
```

檢查:

```bash
export OPENLLM_ENDPOINT=http://localhost:3000

openllm query 'What are large language models?'
```

### 模型支持

您可以使用 `--model-id` 指定以下任何 Flan-T5 模型。

- [google/flan-t5-small](https://huggingface.co/google/flan-t5-small)
- [google/flan-t5-base](https://huggingface.co/google/flan-t5-base)
- [google/flan-t5-large](https://huggingface.co/google/flan-t5-large)
- [google/flan-t5-xl](https://huggingface.co/google/flan-t5-xl)
- [google/flan-t5-xxl](https://huggingface.co/google/flan-t5-xxl)
- 任何其他嚴格遵循 [T5ForConditionalGeneration](https://huggingface.co/docs/transformers/main/model_doc/t5#transformers.T5ForConditionalGeneration) 架構的模型

### 推論後台支援

- PyTorch (Default):

    ```bash
    openllm start flan-t5 --model-id google/flan-t5-large --backend pt
    ```

- Flax:

    ```bash
    pip install "openllm[flan-t5, flax]"
    openllm start flan-t5 --model-id google/flan-t5-large --backend flax
    ```

- TensorFlow:

    ```bash
    pip install "openllm[flan-t5, tf]"
    openllm start flan-t5 --model-id google/flan-t5-large --backend tf
    ```

## GPT-NeoX

**安裝**:

安裝 openllm 後，GPT-NeoX 模型不需要安裝任何特定於模型的依賴項。

```python
pip install openllm
```

**快速開始**:

運行以下命令快速啟動 `GPT-NeoX` 服務器並向其發送請求。

```bash
openllm start gpt-neox --model-id eleutherai/gpt-neox-20b
```

檢查:

```bash
export OPENLLM_ENDPOINT=http://localhost:3000

openllm query 'What are large language models?'
```

### 模型支持

您可以使用 `--model-id` 指定以下任何 Falcon 模型。

- [eleutherai/gpt-neox-20b](https://huggingface.co/eleutherai/gpt-neox-20b)
- 任何其他嚴格遵循 [GPTNeoXForCausalLM](https://huggingface.co/docs/transformers/main/model_doc/gpt_neox#transformers.GPTNeoXForCausalLM) 架構的模型

### 推論後台支援

- PyTorch (Default):

    ```bash
    openllm start gpt-neox --model-id eleutherai/gpt-neox-20b --backend pt
    ```

- vLLM (Recommended):


    ```bash
    pip install openllm[vllm]

    openllm start gpt-neox --model-id eleutherai/gpt-neox-20b --backend vllm
    ```

    !!! info
        目前，使用 vLLM 後端時，不支持量化(quantization)和適配器(adapter)。

## MPT

**安裝**:

要使用 OpenLLM 運行 MPT 模型，您需要安裝 `mpt` 依賴項，因為默認情況下未安裝它。

```python
pip install openllm[mpt]
```

**快速開始**:

運行以下命令快速啟動 `MPT` 服務器並向其發送請求。

```bash
openllm start mpt --model-id mosaicml/mpt-7b-chat
```

檢查:

```bash
export OPENLLM_ENDPOINT=http://localhost:3000

openllm query 'What are large language models?'
```

### 模型支持

您可以使用 `--model-id` 指定以下任何 MPT 模型。

- [mosaicml/mpt-7b](https://huggingface.co/mosaicml/mpt-7b)
- [mosaicml/mpt-7b-instruct](https://huggingface.co/mosaicml/mpt-7b-instruct)
- [mosaicml/mpt-7b-chat](https://huggingface.co/mosaicml/mpt-7b-chat)
- [mosaicml/mpt-7b-storywriter](https://huggingface.co/mosaicml/mpt-7b-storywriter)
- [mosaicml/mpt-30b](https://huggingface.co/mosaicml/mpt-30b)
- [mosaicml/mpt-30b-instruct](https://huggingface.co/mosaicml/mpt-30b-instruct)
- [mosaicml/mpt-30b-chat](https://huggingface.co/mosaicml/mpt-30b-chat)
- 任何其他嚴格遵循 [MPTForCausalLM](https://huggingface.co/mosaicml) 架構的模型

### 推論後台支援

- PyTorch (Default):

    ```bash
    openllm start mpt --model-id mosaicml/mpt-7b-chat --backend pt

    ```

- vLLM (Recommended):


    ```bash
    pip install openllm[mpt, vllm]

    openllm start mpt --model-id mosaicml/mpt-7b-chat --backend vllm
    ```

    !!! info
        目前，使用 vLLM 後端時，不支持量化(quantization)和適配器(adapter)。

## OPT

**安裝**:

要使用 OpenLLM 運行 OPT 模型，您需要安裝 `opt` 依賴項，因為默認情況下未安裝它。

```python
pip install openllm[opt]
```

**快速開始**:

運行以下命令快速啟動 `OPT` 服務器並向其發送請求。

```bash
openllm start opt --model-id facebook/opt-2.7b
```

檢查:

```bash
export OPENLLM_ENDPOINT=http://localhost:3000

openllm query 'What are large language models?'
```

### 模型支持

您可以使用 `--model-id` 指定以下任何 OPT 模型。

-[facebook/opt-125m](https://huggingface.co/facebook/opt-125m)
-[facebook/opt-350m](https://huggingface.co/facebook/opt-350m)
-[facebook/opt-1.3b](https://huggingface.co/facebook/opt-1.3b)
-[facebook/opt-2.7b](https://huggingface.co/facebook/opt-2.7b)
-[facebook/opt-6.7b](https://huggingface.co/facebook/opt-6.7b)
-[facebook/opt-66b](https://huggingface.co/facebook/opt-66b)
- 任何其他嚴格遵循 [OPTForCausalLM](https://huggingface.co/docs/transformers/main/model_doc/opt#transformers.OPTForCausalLM) 架構的模型

### 推論後台支援

- PyTorch (Default):

    ```bash
    openllm start opt --model-id facebook/opt-2.7b --backend pt
    ```

- vLLM (Recommended):


    ```bash
    pip install "openllm[opt, vllm]"
    openllm start opt --model-id facebook/opt-2.7b --backend vllm
    ```

    !!! info
        目前，使用 vLLM 後端時，不支持量化(quantization)和適配器(adapter)。

- TensorFlow:

    ```bash
    pip install "openllm[opt, tf]"
    openllm start opt --model-id facebook/opt-2.7b --backend tf
    ```

- Flax:

    ```bash
    pip install "openllm[opt, flax]"
    openllm start opt --model-id facebook/opt-2.7b --backend flax
    ```

## StableLM

**安裝**:

安裝 openllm 後，StableLM 模型不需要安裝任何特定於模型的依賴項。

```python
pip install openllm
```

**快速開始**:

運行以下命令快速啟動 `StableLM` 服務器並向其發送請求。

```bash
openllm start stablelm --model-id stabilityai/stablelm-tuned-alpha-7b
```

檢查:

```bash
export OPENLLM_ENDPOINT=http://localhost:3000

openllm query 'What are large language models?'
```

### 模型支持

您可以使用 `--model-id` 指定以下任何 StableLM 模型。

-[stabilityai/stablelm-tuned-alpha-3b](https://huggingface.co/stabilityai/stablelm-tuned-alpha-3b)
-[stabilityai/stablelm-tuned-alpha-7b](https://huggingface.co/stabilityai/stablelm-tuned-alpha-7b)
-[stabilityai/stablelm-base-alpha-3b](https://huggingface.co/stabilityai/stablelm-base-alpha-3b)
-[stabilityai/stablelm-base-alpha-7b](https://huggingface.co/stabilityai/stablelm-base-alpha-7b)
- 任何其他嚴格遵循 [GPTNeoXForCausalLM](https://huggingface.co/docs/transformers/main/model_doc/gpt_neox#transformers.GPTNeoXForCausalLM) 架構的模型

### 推論後台支援

- PyTorch (Default):

    ```bash
    openllm start stablelm --model-id stabilityai/stablelm-tuned-alpha-7b --backend pt
    ```

- vLLM (Recommended):


    ```bash
    openllm start stablelm --model-id stabilityai/stablelm-tuned-alpha-7b --backend vllm
    ```

    !!! info
        目前，使用 vLLM 後端時，不支持量化(quantization)和適配器(adapter)。

## StarCoder

**安裝**:

要使用 OpenLLM 運行 StarCoder 模型，您需要安裝 `starcoder` 依賴項，因為默認情況下未安裝它。

```python
pip install openllm[starcoder]
```

**快速開始**:

運行以下命令快速啟動 `StarCoder` 服務器並向其發送請求。

```bash
openllm start startcoder --model-id bigcode/starcoder
```

檢查:

```bash
export OPENLLM_ENDPOINT=http://localhost:3000

openllm query 'What are large language models?'
```

### 模型支持

您可以使用 `--model-id` 指定以下任何 StarCoder 模型。

-[bigcode/starcoder](https://huggingface.co/bigcode/starcoder)
-[bigcode/starcoderbase](https://huggingface.co/bigcode/starcoderbase)
- 任何其他嚴格遵循 [GPTBigCodeForCausalLM](https://huggingface.co/docs/transformers/main/model_doc/gpt_bigcode#transformers.GPTBigCodeForCausalLM) 架構的模型

### 推論後台支援

- PyTorch (Default):

    ```bash
    openllm start startcoder --model-id bigcode/starcoder --backend pt
    ```

- vLLM (Recommended):

    ```bash
    pip install "openllm[startcoder, vllm]"
    openllm start startcoder --model-id bigcode/starcoder --backend vllm
    ```

    !!! info
        目前，使用 vLLM 後端時，不支持量化(quantization)和適配器(adapter)。

## Baichuan

**安裝**:

要使用 OpenLLM 運行 Baichuan 模型，您需要安裝 `baichuan` 依賴項，因為默認情況下未安裝它。

```python
pip install openllm[baichuan]
```

**快速開始**:

運行以下命令快速啟動 `Baichuan` 服務器並向其發送請求。

```bash
openllm start baichuan --model-id baichuan-inc/baichuan-13b-base
```

檢查:

```bash
export OPENLLM_ENDPOINT=http://localhost:3000

openllm query 'What are large language models?'
```

### 模型支持

您可以使用 `--model-id` 指定以下任何 Baichuan 模型。

-[baichuan-inc/baichuan-7b](https://huggingface.co/baichuan-inc/baichuan-7b)
-[baichuan-inc/baichuan-13b-base](https://huggingface.co/baichuan-inc/baichuan-13b-base)
-[baichuan-inc/baichuan-13b-chat](https://huggingface.co/baichuan-inc/baichuan-13b-chat)
-[fireballoon/baichuan-vicuna-chinese-7b](https://huggingface.co/fireballoon/baichuan-vicuna-chinese-7b)
-[fireballoon/baichuan-vicuna-7b](https://huggingface.co/fireballoon/baichuan-vicuna-7b)
-[hiyouga/baichuan-7b-sft](https://huggingface.co/hiyouga/baichuan-7b-sft)
- 任何其他嚴格遵循 [BaiChuanForCausalLM ](https://github.com/baichuan-inc/Baichuan-7B) 架構的模型

### 推論後台支援

- PyTorch (Default):

    ```bash
    openllm start baichuan --model-id baichuan-inc/baichuan-13b-base --backend pt
    ```

- vLLM (Recommended):

    ```bash
    pip install "openllm[baichuan, vllm]"
    openllm start baichuan --model-id baichuan-inc/baichuan-13b-base --backend vllm
    ```

    !!! info
        目前，使用 vLLM 後端時，不支持量化(quantization)和適配器(adapter)。


更多模型將與 OpenLLM 集成，如果您想將自定義 LLM 納入生態系統，我們歡迎您的貢獻。查看[添加新模型指南](https://github.com/bentoml/OpenLLM/blob/main/openllm-python/ADDING_NEW_MODEL.md)以了解更多信息。
