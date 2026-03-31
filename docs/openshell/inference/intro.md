# About Inference Routing

NVIDIA OpenShell 透過兩個端點處理推論流量：`inference.local` 端點和外部端點。下表概述了 OpenShell 如何處理推論流量。

|路徑|工作原理|
|---|-------|
|**External endpoints**|對 `api.openai.com` 或 `api.anthropic.com` 等主機的流量處理方式與其他出站請求相同，由 `network_policies` 決定是否允許。請參閱「[自訂沙盒策略](https://docs.nvidia.com/openshell/latest/sandboxes/policies.html)」。|
|`inference.local`|每個沙箱內部都暴露了一個特殊的端點，用於在本地路由推論流量，從而保護隱私和安全。[隱私路由器](https://docs.nvidia.com/openshell/latest/about/architecture.html)會剝離原始憑證，注入已設定的後端憑證，並將流量轉送到託管模型端點。|

## How `inference.local` Works

當沙箱內的程式碼呼叫 `https://inference.local` 時，隱私權路由器會將請求路由到為該閘道配置的後端。配置的模型應用於產生請求，並且提供者憑證由 OpenShell 提供，而不是由沙箱內的程式碼提供。

如果程式碼直接呼叫外部推論主機，則該流量僅由 `network_policies` 進行評估。

|特性|細節|
|---|----|
|**Credentials**|無需沙盒 API 金鑰。憑證來自已配置的提供者記錄。|
|**Configuration**|一個 provider 和一個 model 定義了活動網關的沙箱推論。該網關上的每個沙箱都使用相同的 `inference.local` 後端。|
|**Provider support**|NVIDIA、任何相容於 OpenAI 的供應商以及 Anthropic 都透過同一個端點運作。|
|**Hot-refresh**|OpenShell 能夠偵測提供者憑證的變更和推斷更新，而無需重新建立沙箱。預設情況下，變更會在大約 5 秒內傳播。|

## Supported API Patterns

支援的請求模式取決於 `inference.local` 設定的 provider。

=== "OpenAI-compatible"

    |Pattern|Method|Path|
    |-------|------|----|
    |Chat Completions|`POST`|`/v1/chat/completions`|
    |Completions|`POST`|`/v1/completions`|
    |Responses|`POST`|`/v1/responses`|
    |Model Discovery|`GET`|`/v1/models`|
    |Model Discovery|`GET`|`/v1/models/*`|

=== "Anthropic-compatible"

    |Pattern|Method|Path|
    |-------|------|----|
    |Messages|`POST`|`/v1/messages`|

拒絕向 `inference.local` 發出的與已配置提供者支援的模式不符的請求。


