# ExLlama

## 關於

ExLlama 是針對 LLaMA 模型極其優化的 GPTQ 後端。由於不依賴未優化的 `transformers` 代碼，它的 VRAM 使用率要低得多，推論速度要高得多。

## 用法

配置 `text-generation-webui` 以通過 UI 或命令行使用 exllama：

- 在 "Model" 頁籤, 設定 "Loader" 成 "exllama"
- 在啟動的命令行中加入 `--loader exllama` 設定引數

## 手動設置

由於 exllama 套件已包含在 `requirements.txt` 中，因此不需要額外的安裝步驟。如果此套件由於某種原因無法安裝，您可以通過將原始存儲庫克隆到您的 `repositories/` 文件夾中來手動安裝它：

```bash
mkdir repositories

cd repositories

git clone https://github.com/turboderp/exllama
```