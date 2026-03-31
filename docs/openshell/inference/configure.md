# Configure Inference Routing

本頁面介紹託管的本地模型推論端點 (`https://inference.local`)。外部推論端點透過沙盒 `network_policies` 進行通訊。詳情請參閱「[策略](https://docs.nvidia.com/openshell/latest/sandboxes/policies.html)」部分。

此配置包含兩個值：

|值|描述|
|--|---|
|Provider record|OpenShell 使用的憑證後端，用於向上游模型主機進行身份驗證。|
|Model ID|用於產生請求的模型。|

有關已測試 provider 及其基本 URL 的列表，請參閱「[支援的推理提供者](https://docs.nvidia.com/openshell/latest/sandboxes/manage-providers.html#supported-inference-providers)」。

## Create a Provider

建立一個提供程序，用於保存您希望 OpenShell 使用的後端憑證。

=== "NVIDIA API Catalog"
    ```bash
    openshell provider create --name nvidia-prod --type nvidia --from-existing
    ```

    這會從您的環境變數中讀取 `NVIDIA_API_KEY`。

=== "OpenAI-compatible Provider"
    任何提供與 OpenAI 相容 API 的雲端服務提供者都支援 openai 提供者類型。您需要從提供者取得三個值：base URL, API 金鑰和模型名稱。

    ```bash
    openshell provider create \
      --name my-cloud-provider \
      --type openai \
      --credential OPENAI_API_KEY=<your_api_key> \
      --config OPENAI_BASE_URL=https://api.example.com/v1
    ```

    請將 base URL 和 API 金鑰替換為您提供者提供的值。對於開箱即用的支援供應商，請參閱「[受支援的推理提供者](https://docs.nvidia.com/openshell/latest/sandboxes/manage-providers.html#supported-inference-providers)」頁面。對於其他提供者，請參閱其文件以取得正確的 base URL、可用模型和 API 金鑰設定。

=== "Local Endpoint"

    ```bash
    openshell provider create \
      --name my-local-model \
      --type openai \
      --credential OPENAI_API_KEY=empty-if-not-required \
      --config OPENAI_BASE_URL=http://host.openshell.internal:11434/v1
    ```
    
    使用 `--config OPENAI_BASE_URL` 指向網關執行所在位置的任何相容 OpenAI 的伺服器。對於主機支援的本機推理，請使用 `host.openshell.internal` 或主機的區域網路 IP 位址。避免使用 `127.0.0.1` 和 `localhost`。如果伺服器不需要身份驗證，請將 `OPENAI_API_KEY` 設定為一個虛擬值。

    !!! tip
        對於獨立部署，Ollama 社區沙箱將 Ollama 打包在沙箱內部——無需主機級提供者。詳情請參閱「[使用 Ollama 運行本地推理](https://docs.nvidia.com/openshell/latest/tutorials/inference-ollama.html)」。
        
        Ollama 也支援使用 `:cloud` 標籤後綴的雲端託管模型（例如，`qwen3.5:cloud`）。

=== "Anthropic"

    ```bash
    openshell provider create --name anthropic-prod --type anthropic --from-existing
    ```

    這會從您的環境中讀取 `ANTHROPIC_API_KEY`。

## Set Inference Routing

將 `inference.local` 指向該提供程序，然後選擇要使用的模型：

```bash
openshell inference set \
    --provider nvidia-prod \
    --model nvidia/nemotron-3-nano-30b-a3b
```

## Verify the Active Config

請確認 provider 和 model 設定正確：

```bash
openshell inference get
```

結果:

```bash
Gateway inference:

  Provider: nvidia-prod
  Model: nvidia/nemotron-3-nano-30b-a3b
  Version: 1
```

## Update Part of the Config

當您只想更改一個欄位時，請使用 `update` 命令：

```bash
openshell inference update --model nvidia/nemotron-3-nano-30b-a3b
```

或更換服務 provider，而無需重複當前的 model：

```bash
openshell inference update --provider openai-prod
```

## Use the Local Endpoint from a Sandbox

配置好模型推論之後，任何沙箱內的程式碼都可以直接呼叫 `https://inference.local`：

```bash
from openai import OpenAI

client = OpenAI(base_url="https://inference.local/v1", api_key="unused")

response = client.chat.completions.create(
    model="anything",
    messages=[{"role": "user", "content": "Hello"}],
)
```

客戶端提供的 `model` 和 `api_key` 不會傳送到上游。隱私路由器(privacy router)會從已設定的 provider 注入真實憑證，並在轉送前重寫 model。

某些 SDK 需要非空的 API key，即使 `inference.local` 不使用沙箱提供的值。在這種情況下，請傳遞任何占位符，例如 `test` 或 `unused`。

出於隱私和安全考慮，當推論過程應僅限於本地主機時，請使用此端點。需要直接存取的外部提供者則應放在 `network_policies` 中。

當上游伺服器與網關運行在同一台機器上時，將其綁定到 `0.0.0.0`，並將 provider 指向 `host.openshell.internal` 或主機的區域網路 IP 位址。 `127.0.0.1` 和 `localhost` 通常會失敗，因為請求源自網關或沙箱執行時，而不是您的 shell。

如果網關運行在遠端主機上或雲端部署環境中，則 `host.openshell.internal` 指向的是該遠端主機，而不是您的筆記型電腦。除非您新增自己的隧道或共用網路路徑，否則無法從遠端網關存取本機執行的 Ollama 或 vLLM 進程。 Ollama 也支援不需要本地硬體的雲端託管模型。

## Verify the Endpoint from a Sandbox

`openshell inference set` 和 `openshell inference update` 預設會在儲存配置前驗證已解析的上游端點。如果端點尚未生效，請使用 `--no-verify` 重試，以便在不進行探測的情況下持久化路由。

`openshell inference get` 會確認目前已儲存的設定。若要從沙箱環境確認端對端連接，請執行：

```bash
curl https://inference.local/v1/responses \
    -H "Content-Type: application/json" \
    -d '{
      "instructions": "You are a helpful assistant.",
      "input": "Hello!"
    }'
```

成功回應確認隱私路由器可以連接到設定的後端，並且模型正在處理請求。

- **Gateway-scoped**：所有使用相同網關的沙箱都存取相同的 `inference.local` 後端。
- **HTTPS only**：`inference.local` 僅攔截 HTTPS 流量。
- **Hot reload**：預設情況下，provider 和 inference 變更會在大約 5 秒內生效。

