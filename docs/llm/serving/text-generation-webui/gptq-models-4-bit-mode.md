# GTPQ 量化模型

GPTQ 是一種巧妙的量化算法，它在量化過程中稍微重新優化權重，從而相對於舍入到最近的量化來補償精度損失。有關更多詳細信息，請參閱論文：https://arxiv.org/abs/2210.17323

4-bit GPTQ 模型可將 VRAM 使用量減少約 75%。因此 LLaMA-7B 就可塞入只有 6GB 的 GPU，而 LLaMA-30B 也可使用 24GB 的 GPU 來跑。

## 概述

目前在 Web UI 中加載 GPTQ 模型有兩種方法：

1. 使用 [AutoGPTQ](https://github.com/PanQiWei/AutoGPTQ)

    - 支持更多模型
    - 標準化（無需猜測任何參數）
    - 是一個合適的 Python 套件
    - 支持加載 triton 和 cuda 模型

2. 直接使用 [GPTQ-for-LLaMa](https://github.com/qwopqwop200/GPTQ-for-LLaMa)

    - 更快的 CPU 卸載
    - 更快的多 GPU 推理
    - 支持使用猴子補丁加載 LoRA
    - 需要您手動找出模型的 wbits/groupsize/model_type 參數才能加載它
    - 僅支持 cuda 或僅支持 Triton，具體取決於分支

建議使用 [AutoGPTQ](https://github.com/PanQiWei/AutoGPTQ)!

## AutoGPTQ

**安裝:**

無需執行其他步驟，因為 AutoGPTQ 已包含在 WebUI 的 `requirements.txt` 中。如果您出於某種原因仍然想要或需要手動安裝它，請使用以下命令：

```bash
conda activate textgen

git clone https://github.com/PanQiWei/AutoGPTQ.git && cd AutoGPTQ

pip install .
```

最後一個命令需要安裝 `nvcc`（請參閱上面的說明）。

**用法:**

當您使用 AutoGPTQ 量化模型時，將生成一個包含名為 `quantize_config.json` 的文件的文件夾。將該文件夾放入 `models/` 文件夾中，並使用 `--autogptq` 標誌加載它：

```bash
python server.py --autogptq --model model_name
```

或者，在加載模型之前檢查 UI 的 "Model" 頁籤中的 `autogptq` box。