# llama.cpp

llama.cpp 是兩個重要場景中最好的後端：

- 你沒有 GPU。
- 您想要運行一個塞不進您的 GPU 顯存的模型。

## 模型設定

**預轉換**

- 將 GGUF 模型直接下載到 `text-generation-webui/models` 文件夾中。這將是一個文件。
- 建議使用 `q4_K_M` 量化。

**手建轉換 Llama 模型**

按照 llama.cpp README 中的說明生成 GGUF：

- https://github.com/ggerganov/llama.cpp#prepare-data--run

## GPU 加速

使用 `--n-gpu-layers` 參數啟用 GPU 加速。

- 如果您有足夠的 VRAM，請使用較高的數字（例如 `--n-gpu-layers 1000`）將所有層卸載到 GPU。
- 否則，從一個較小的數字開始，例如 `--n-gpu-layers 10`，然後逐漸增加它，直到內存耗盡。

此功能在 Linux (amd64) 或 Windows 上的 NVIDIA GPU 上開箱即用。對於其他 GPU，您需要卸載 `llama-cpp-python`