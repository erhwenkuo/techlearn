# FAQ

### How can I upgrade Ollama?

在 Linux 上，重新執行安裝腳本：

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

### How can I specify the context window size?

預設情況下，Ollama 使用 2048 個 token 的上下文視窗大小。

若要在使用 `ollama run` 時變更此設置，請使用 `/set parameter`：

```bash
/set parameter num_ctx 4096
```

使用 API ​​時，指定 `num_ctx` 參數：

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.1",
  "prompt": "Why is the sky blue?",
  "options": {
    "num_ctx": 4096
  }
}'
```

### How can I tell if my model was loaded onto the GPU?

使用 `ollama ps` 指令查看目前載入到記憶體中的模型。

```bash
ollama ps
NAME      	ID          	SIZE 	PROCESSOR	UNTIL
llama3:70b	bcfb190ca3a7	42 GB	100% GPU 	4 minutes from now
```

處理器列將顯示模型載入到的記憶體：

- `100% GPU` 意味著模型完全載入到 GPU 中
- `100% CPU` 意味著模型完全載入到系統記憶體中
- `48%/52% CPU/GPU` 表示模型已部分載入到 GPU 和系統記憶體中

### How do I configure Ollama server?

Ollama 伺服器可以設定環境變數。

如果 Ollama 作為 systemd 服務運行，則應使用 `systemctl` 設定環境變數：

1. 透過呼叫 `systemctl edit ollama.service` 編輯 `systemd` 服務。這將打開一個編輯器。
2. 對於每個環境變量，在 `[Service]` 部分下新增一行 Environment：
    ```bash
    [Service]
    Environment="OLLAMA_HOST=0.0.0.0"
    ```
3. 儲存並退出。
4. 重新載入 systemd 並重新啟動 Ollama：
    ```bash
    systemctl daemon-reload
    systemctl restart ollama
    ```

### How do I use Ollama behind a proxy?

Ollama 從 Internet 拉取模型，並且可能需要代理伺服器(Http Proxy Server)才能存取模型。使用 `HTTPS_PROXY` 透過代理程式重新導向出站請求。確保代理證書作為系統證書安裝。有關如何在您的平台上使用環境變數的信息，請參閱上面的部分。

> 避免設定 `HTTP_PROXY`。 Ollama 不使用 HTTP 進行模型拉取，僅使用 HTTPS。設定 `HTTP_PROXY` 可能會中斷客戶端與伺服器的連線。

### How can I expose Ollama on my network?

Ollama 預設綁定 `127.0.0.1` 連接埠 `11434`。使用 `OLLAMA_HOST` 環境變數變更綁定位址。

### How can I allow additional web origins to access Ollama?

Ollama 預設允許來自 `127.0.0.1` 和 `0.0.0.0` 的跨域請求。可以使用 `OLLAMA_ORIGINS` 來配置其他來源。

### Where are models stored?

- Linux: `/usr/share/ollama/.ollama/models`

### How do I set them to a different location?

如果需要使用不同的目錄，請將環境變數 `OLLAMA_MODELS` 設定為所選目錄。

> 注意：在使用標準安裝程式的 Linux 上，ollama 使用者需要對指定目錄具有讀寫權限。若要將目錄指派給 ollama 用戶，請執行 `sudo chown -R ollama:ollama <directory>`。

### How can I preload a model into Ollama to get faster response times?

如果您使用 API，您可以透過向 Ollama 伺服器發送空請求來預先載入模型。這適用於 `/api/generate` 和 `/api/chat` API 端點。

若要使用 `generate` 端點預先載入 Mistra 模型，請使用：

```bash
curl http://localhost:11434/api/generate -d '{"model": "mistral"}'
```

若要使用 `chat completions` 端點，請使用：

```bash
curl http://localhost:11434/api/chat -d '{"model": "mistral"}'
```

若要使用 CLI 預載模型，請使用下列命令：

```bash
ollama run llama3.1 ""
```

### How do I keep a model loaded in memory or make it unload immediately?

預設情況下，模型在卸載之前會在記憶體中保留 5 分鐘。如果您向 LLM 提出大量請求，這可以加快回應時間。但是，您可能希望在 5 分鐘過去之前釋放記憶體或無限期地保持模型載入。將 `keep_alive` 參數與 `/api/generate` 和 `/api/chat` API 端點結合使用來控制模型在記憶體中保留的時間。

`keep_alive` 參數可以設定為：

- 持續時間字串（例如 `10m` 或 `24h`）
- 以秒為單位的數字（例如 `3600`）
- 設定任何負數（例如 `-1` 或 `-1m`）來將模型載入到記憶體中而不卸載
- `0` 將在產生回應後立即卸載模型

例如，要預先載入模型並將其保留在記憶體中：

```bash
curl http://localhost:11434/api/generate -d '{"model": "llama3.1", "keep_alive": -1}'
```

要卸載模型並釋放記憶體使用：

```bash
curl http://localhost:11434/api/generate -d '{"model": "llama3.1", "keep_alive": 0}'
```

或者，您可以透過在啟動 Ollama 伺服器時設定 `OLLAMA_KEEP_ALIVE` 環境變數來變更所有模型載入到記憶體中的時間。 `OLLAMA_KEEP_ALIVE` 變數使用與上面提到的 `keep_alive` 參數類型相同的參數類型。

如果您希望覆蓋 `OLLAMA_KEEP_ALIVE` 設定，請將 `keep_alive` API 參數與 `/api/generate` 或 `/api/chat` API 端點結合使用。


### How do I manage the maximum number of requests the Ollama server can queue?

如果發送到伺服器的請求太多，它將回應 503 錯誤，表示伺服器過載。您可以透過設定 `OLLAMA_MAX_QUEUE` 來調整可以排隊的請求數量。

### How does Ollama handle concurrent requests?

Ollama 支援兩個層級的同時處理。如果您的系統有足夠的可用內存（使用 CPU 推理時的系統內存，或用於 GPU 推理的 VRAM），則可以同時加載多個模型。對於給定模型，如果載入模型時有足夠的可用內存，則將其配置為允許並行請求處理。

如果在一個或多個模型已載入時沒有足夠的可用記憶體來載入新模型請求，則所有新請求將排隊，直到可以載入新模型。當先前的模型閒置時，一個或多個模型將被卸載，為新模型騰出空間。排隊的請求將按順序處理。使用 GPU 推理時，新模型必須能夠完全適合 VRAM，以允許並發模型載入。

給定模型的平行請求處理會導致上下文大小增加並行請求的數量。例如，具有 4 個平行請求的 2K 上下文將導致 8K 上下文和額外的記憶體分配。

以下伺服器設定可用於調整 Ollama 在大多數平台上處理並發請求的方式：

- `OLLAMA_MAX_LOADED_MODELS` - 可以同時載入的模型的最大數量（前提是它們適合可用記憶體）。預設值為 `3` * GPU 數量或 `3`（用於 CPU 推理）。
- `OLLAMA_NUM_PARALLEL` - 每個模型同時處理的最大平行請求數。預設將根據可用記憶體自動選擇 `4` 或 `1`。
- `OLLAMA_MAX_QUEUE` - Ollama 在拒絕其他請求之前在繁忙時排隊的最大請求數。預設為 `512`

### How does Ollama load models on multiple GPUs?

安裝同一品牌的多個 GPU 是增加可用 VRAM 以載入更大模型的好方法。當您載入新模型時，Ollama 會根據目前可用的 VRAM 評估模型所需的 VRAM。如果模型完全適合任何單一 GPU，Ollama 會將模型載入到該 GPU 上。這通常可以提供最佳效能，因為它減少了推理期間透過 PCI 總線傳輸的資料量。如果模型不能完全適合一個 GPU，那麼它將分佈在所有可用的 GPU 上。