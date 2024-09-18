# Ollama 環境變數

|變數|預設值|描述|
|---|-----|----|
|`OLLAMA_HOST`|http://127.0.0.1:11434|配置 Ollama 伺服器的 endpoint。用於連接 Ollama 伺服器的 URL。在分散式環境中部署 Ollama 或需要在特定網路介面上公開服務時非常有用。|
|`OLLAMA_ORIGINS`|[localhost, 127.0.0.1, 0.0.0.0] + app://, file://, tauri://|配置 CORS 允許的來源。控制允許哪些來源向 Ollama 伺服器發出請求。將 Ollama 與 Web 應用程式整合以防止來自不同網域的未經授權的存取時至關重要。|
|`OLLAMA_MODELS`|$HOME/.ollama/models|設定模型目錄的本地路徑。決定模型檔案的儲存和載入位置。對於管理不同磁碟機上的磁碟空間或在多用戶環境中設定共用模型儲存庫很有用。|
|`OLLAMA_KEEP_ALIVE`|5 minutes|設定模型在記憶體中載入的時間。控制模型在使用後保留在記憶體中的持續時間。較長的持續時間可以縮短頻繁查詢的回應時間，但會增加記憶體使用量。較短的持續時間可以釋放資源，但可能會增加初始回應時間。|
|`OLLAMA_DEBUG`|false|啟用附加偵錯資訊。效果：增加日誌記錄和調試輸出的詳細程度。場景：對於解決問題或了解開發或部署期間的系統行為非常有用。|
|`OLLAMA_FLASH_ATTENTION`|false|啟用實驗性 flash attention 功能。可能會提高相容硬體的效能，但可能會帶來不穩定。|
|`OLLAMA_NOHISTORY`|false|停用讀取行歷史記錄。防止保存命令歷史記錄。在不應該保留命令歷史記錄的安全敏感環境中很有用。|
|`OLLAMA_NOPRUNE`|false|禁用啟動時修剪模型 blob。保留所有模型 blob，可能會增加磁碟使用量。|
|`OLLAMA_SCHED_SPREAD`|false|允許跨所有 GPU 調度模型。啟用多 GPU 進行模型推理。有利於具有多個 GPU 的高效能運算環境，以最大限度地提高硬體利用率。|
|`OLLAMA_INTEL_GPU`|false|啟用實驗性 Intel GPU 偵測。允許使用 Intel GPU 進行模型推理。對於利用 Intel GPU 硬體處理 AI 工作負載的組織很有用。|
|`OLLAMA_LLM_LIBRARY`|"" (auto-detect)|設定要使用的 LLM 函式庫。覆蓋LLM函式庫的自動偵測。當您出於相容性或效能原因需要強制使用特定的程式庫版本或實作時很有用。|
|`OLLAMA_TMPDIR`|System default temp directory|設定臨時檔案的位置。確定暫存檔案的儲存位置。對於管理 I/O 效能或系統暫存目錄空間有限時很重要。|
|`CUDA_VISIBLE_DEVICES`|All available|設定哪些 NVIDIA 裝置可見。控制可以使用哪些 NVIDIA GPU。對於管理多使用者或多進程環境中的 GPU 分配至關重要。|
|`HIP_VISIBLE_DEVICES`|All available|設定哪些 AMD 裝置可見。控制可以使用哪些 AMD GPU。與 CUDA_VISIBLE_DEVICES 類似，但適用於 AMD 硬體。|
|`OLLAMA_RUNNERS_DIR`|System-dependent|設定 runner 的位置。決定執行程式可執行檔所在的位置。對於自訂部署或執行程式需要與主應用程式隔離時很重要。|
|`OLLAMA_NUM_PARALLEL`|0 (unlimited)|設定併行模型請求的數量。控制模型推理的併發度。對於管理系統負載和確保高流量環境中的回應能力至關重要。|
|`OLLAMA_MAX_LOADED_MODELS`|0 (unlimited)|設定載入模型的最大數量。限制可以同時載入的模型數量。幫助管理資源有限或許多不同模型的環境中的記憶體使用情況。|
|`OLLAMA_MAX_QUEUE`|512|設定排隊請求的最大數量。限制請求佇列的大小。防止流量高峰期間系統過載並確保及時處理請求。|
|`OLLAMA_MAX_VRAM`|0 (unlimited)|設定最大 VRAM 覆蓋（以位元組為單位）。效果：限制可以使用的 VRAM 量。場景：在共享 GPU 環境中有用，可防止單一程序獨佔 GPU 記憶體。|
