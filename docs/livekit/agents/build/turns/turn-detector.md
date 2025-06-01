# LiveKit turn detector plugin

> 用於情境感知語音 AI 對話輪次偵測的開放權重模型。

## Overview

LiveKit 對話輪次偵測外掛程式是一種自訂的開放權重語言模型，它將對話情境作為附加訊號添加到語音活動偵測 (VAD) 中，以改善語音 AI 應用中轉彎結束偵測。

傳統的 VAD 模型可以有效地確定語音的存在與否，但如果沒有語言理解能力，它們可能會提供糟糕的使用者體驗。例如，用戶可能會說“我需要考慮一下”，然後長時間停頓。用戶還有很多話要說，但光是 VAD 系統無論如何都會打斷他們。情境感知模型可以預測他們還有更多話要說，並等待他們說完再回應。

LiveKit 對話輪次偵測器外掛程式可免費與 Agents SDK 搭配使用，包括純英語和多語言模型。

- **[Turn detector demo](https://youtu.be/EYDrSSEP0h0)**: 展示 LiveKit 對話輪次檢測器所提供的改進的影片。

## Quick reference

以下部分簡要概述了對話輪次偵測器插件。有關詳細信息，請參閱[其他資源](#additional-resources)。

### Requirements

LiveKit 對話輪次偵測器設計用於 `AgentSession` 內部，並且還需要提供 [STT 插件](https://docs.livekit.io/agents/integrations/stt.md)。如果您使用即時 LLM，則必須包含單獨的 STT 外掛才能使用 LiveKit 轉彎偵測器外掛程式。

LiveKit 也建議使用 [Silero VAD 外掛程式](https://docs.livekit.io/agents/build/turns/vad.md) 以獲得最佳效能，但如果您願意，也可以依賴 STT 外掛程式的端點。

該模型在 CPU 上本地運行，即使使用共享推理伺服器進行多個並發作業，也只需要 <500 MB 的 RAM。

### Installation

從 PyPI 安裝插件：

```bash
pip install "livekit-agents[turn-detector]~=1.0"

```

### Download model weights

首次運行代理之前，您必須下載模型權重：

```bash
python agent.py download-files

```

### Usage

使用對話輪次偵測器初始化您的“AgentSession”，並使用相符的語言設定初始化您的 STT 外掛程式。這些範例使用了 Deepgram STT 插件，但還有 10 多個其他 STT 插件[可用](https://docs.livekit.io/agents/integrations/stt.md)。

#### English-only model

使用 `EnglishModel` 並確保您的 STT 插件配置匹配：

```python
from livekit.plugins.turn_detector.english import EnglishModel
from livekit.plugins import deepgram

session = AgentSession(
    turn_detection=EnglishModel(),
    stt=deepgram.STT(model="nova-3", language="en"),
    # ... vad, stt, tts, llm, etc.
)

```

#### Multilingual model

使用 `MultilingualModel` 並確保您的 STT 插件配置匹配。在這個範例中，Deepgram 執行自動語言偵測並將該值傳遞給對話輪次偵測器。

```python
from livekit.plugins.turn_detector.multilingual import MultilingualModel
from livekit.plugins import deepgram

session = AgentSession(
    turn_detection=MultilingualModel(),
    stt=deepgram.STT(model="nova-3", language="multi"),
    # ... vad, stt, tts, llm, etc.
)

```

### Parameters

轉彎檢測器本身沒有配置，但使用它的 `AgentSession` 支援以下相關參數：

- **`min_endpointing_delay`** _(float)_ (optional) - Default: `0.5`: 視為對話輪次完成前等待的秒數。當不存在對話輪次偵測器模型或模型指示可能的對話輪次邊界時，會話將使用此延遲。

- **`max_endpointing_delay`** _(float)_ (optional) - Default: `6.0`: 對話輪次偵測器模型指示使用者可能繼續說話後等待使用者說話的最長時間。如果沒有對話輪次偵測器模型，此參數則不起作用。

## Supported languages

`MultilingualModel` 支援英語和其他 13 種語言。此模型依賴您的 [STT 外掛程式](https://docs.livekit.io/agents/integrations/stt.md) 來報告使用者語音的語言。若要將語言設定為固定值，請使用特定語言設定 STT 外掛程式。例如，強制模型使用西班牙語：

```python
session = AgentSession(
    turn_detection=MultilingualModel(),
    stt=deepgram.STT(model="nova-2", language="es"),
    # ... vad, stt, tts, llm, etc.
)

```

該模型目前支援英語、西班牙語、法語、德語、義大利語、葡萄牙語、荷蘭語、中文、日語、韓語、印尼語、土耳其語和俄語。

## Realtime model usage

像 OpenAI Realtime API 這樣的即時模型會在回合結束後產生使用者記錄，而不是在使用者說話時逐步產生。對話輪次偵測器模型需要即時 STT 結果才能運行，因此您必須向 `AgentSession` 提供 STT 插件才能將其與即時模型一起使用。這會導致 STT 模型產生額外成本。

## Benchmarks

以下數據顯示了對話輪次偵測器模型的預期性能。

### Runtime performance

對話輪次偵測器模型的磁碟大小和典型 CPU 推理時間如下：

| Model | Base Model | Size on Disk | Per Turn Latency |
|-------|------------|--------------|------------------|
| English-only | [SmolLM2-135M](https://huggingface.co/HuggingFaceTB/SmolLM2-135M-Instruct) | 66 MB | ~15-45 ms |
| Multilingual | [Qwen2.5-0.5B](https://huggingface.co/Qwen/Qwen2.5-0.5B) | 281 MB | ~50-160 ms |

### Detection accuracy

下表顯示了每種受支援語言的對話輪次偵測器模型的準確度指標。

- **True positive** 表示模型正確辨識使用者已經說完了。
- **True negative** 意味著模型正確識別使用者將繼續說話。

#### English-only model

純英語模型的準確度指標：

| Language | True Positive Rate | True Negative Rate |
|----------|--------------------|--------------------|
| English | 98.8% | 87.5% |

#### Multilingual model

使用正確的語言配置時，多語言模型的準確度指標：

| Language | True Positive Rate | True Negative Rate |
|----------|--------------------|--------------------|
| French | 98.8% | 97.7% |
| Indonesian | 98.9% | 97.6% |
| Russian | 98.8% | 97.6% |
| Dutch | 98.8% | 97.4% |
| Turkish | 98.8% | 97.4% |
| Portuguese | 98.8% | 97.4% |
| Spanish | 98.8% | 97.1% |
| German | 98.8% | 96.9% |
| Italian | 98.8% | 96.8% |
| Korean | 98.8% | 91.0% |
| English | 98.8% | 89.8% |
| Japanese | 98.8% | 88.0% |
| Chinese | 98.8% | 81.6% |

## Additional resources

以下資源提供了有關使用 LiveKit 對話輪次偵測器外掛程式的更多資訊。

- **[Python package](https://pypi.org/project/livekit-plugins-turn-detector/)**: PyPI 上的 `livekit-plugins-turn-detector` 包。

- **[Plugin reference](https://docs.livekit.io/reference/python/v1/livekit/plugins/turn_detector/index.html.md#livekit.plugins.turn_detector.TurnDetector)**: LiveKit 對話輪次偵測器插件的參考。

- **[GitHub repo](https://github.com/livekit/agents/tree/main/livekit-plugins/livekit-plugins-turn-detector)**: 查看原始程式碼或為 LiveKit 對話輪次檢測器外掛程式做出貢獻。

- **[LiveKit Model License](https://huggingface.co/livekit/turn-detector/blob/main/LICENSE)**: 用於對話輪次偵測器模型的 LiveKit 模型許可證。
