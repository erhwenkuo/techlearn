# Bootstrapping an application

> 透過一組方便的模板建立並初始化應用程式。

!!! info
    在開始之前，請確保您已建立雲端帳戶或是構建了 self-host 的 Livekit Server 實例、[安裝了 LiveKit CLI](https://docs.livekit.io/home/cli/cli-setup.md)，並且已驗證或手動設定了您選擇的 LiveKit 專案。

LiveKit CLI 可以幫助您從許多方便的範本儲存庫中引導應用程序，使用您的專案憑證自動設定所需的環境變數和其他配置。若要從範本建立應用程序，請執行以下命令：

```shell
lk app create --template <template_name> my-app

```

然後按照 CLI 提示完成設定。

可以省略 `--template` 標誌來查看所有可用模板的列表，或者可以從我們的第一方模板中選擇：

| **Template Name** | **Language/Framework** | **Description** |
|-------------------|------------------------|-----------------|
| [voice-assistant-frontend](https://github.com/livekit-examples/voice-assistant-frontend) | TypeScript/Next.js | 帶有整合令牌伺服器的語音助理前端 |
| [meet](https://github.com/livekit-examples/meet) | TypeScript/Next.js | 具有整合令牌伺服器的視訊會議前端 |
| [multimodal-agent-python](https://github.com/livekit-examples/multimodal-agent-python) | Python | 具有語音轉語音和轉錄功能的多模態式代理 |
| [voice-pipeline-agent-python](https://github.com/livekit-examples/voice-pipeline-agent-python) | Python | 使用模組化 TTS、LLM 和 STT 功能的語音代理 |
| [multimodal-agent-node](https://github.com/livekit-examples/multimodal-agent-node) | Node.js/TypeScript | 具有語音轉語音和轉錄功能的多模式代理 |
| [token-server-node](https://github.com/livekit-examples/token-server-node) | Node.js/TypeScript | 用於產生存取令牌的令牌伺服器 |
| [android-voice-assistant](https://github.com/livekit-examples/android-voice-assistant) | Kotlin/Android | 語音助理行動應用 |

!!! tip
    如果您想要探索 LiveKit 的 [Agents](https://docs.livekit.io/agents.md) 框架，或想要針對預先建置的前端或令牌伺服器為您的應用程式製作原型，請查看 [Sandboxes](https://docs.livekit.io/home/cloud/sandbox.md)。

有關模板的更多信息，請參閱 [LiveKit 模板索引](https://github.com/livekit-examples/index?tab=readme-ov-file)。