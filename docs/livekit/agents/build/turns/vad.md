# Silero VAD plugin

> 針對 LiveKit Agents 的高效能語音活動偵測。

## Overview

Silero VAD 外掛程式提供語音活動偵測 (VAD)，有助於在語音 AI 應用程式中實現準確的 [turn detection](https://docs.livekit.io/agents/build/turns.md) 偵測。

VAD 是語音 AI 應用程式的關鍵組件，因為它有助於確定使用者何時說話以及何時保持沉默。這使得對話能夠自然地進行，並且透過僅在用戶說話時執行語音轉文字來幫助優化資源使用。

LiveKit 建議將 Silero VAD 外掛程式與 [turn detector model](https://docs.livekit.io/agents/build/turns/turn-detector.md) 模型結合使用，以獲得最佳效能。

## Quick reference

以下部分簡要概述了 Silero VAD 插件。有關詳細信息，請參閱[其他資源](#additional-resources)。

### Requirements

該模型在 CPU 上本地運行，需要最少的系統資源。

### Installation

從 PyPI 安裝插件：

```bash
pip install "livekit-agents[silero]~=1.0"

```

### Download model weights

首次運行代理之前，您必須下載模型權重：

```bash
python agent.py download-files

```

### Usage

使用 Silero VAD 插件初始化您的 `AgentSession`：

```python
from livekit.plugins import silero

session = AgentSession(
    vad=silero.VAD.load(),
    # ... stt, tts, llm, etc.
)

```

## Prewarm

您可以 [prewarm](https://docs.livekit.io/agents/worker/options.md#prewarm) 該插件以縮短新作業的載入時間：

```python
async def entrypoint(ctx: agents.JobContext):
    await ctx.connect()

    session = AgentSession(
        vad=ctx.proc.userdata["vad"],
        # ... stt, tts, llm, etc.
    )

    # ... session.start etc ...

def prewarm(proc: agents.JobProcess):
    proc.userdata["vad"] = silero.VAD.load()

if __name__ == "__main__":
    agents.cli.run_app(
        agents.WorkerOptions(entrypoint_fnc=entrypoint, prewarm_fnc=prewarm)
    )


```

## Configuration

`load` 方法中有以下參數可用：

- **`min_speech_duration`** _(float)_ (optional) - Default: `0.05`: 開始新的語音區塊所需的最短語音持續時間。

- **`min_silence_duration`** _(float)_ (optional) - Default: `0.55`: 語音結束後等待的靜默時間長度，以確定使用者是否已講完。

- **`prefix_padding_duration`** _(float)_ (optional) - Default: `0.5`: 新增到每個語音區塊開頭的填充持續時間。

- **`max_buffered_speech`** _(float)_ (optional) - Default: `60.0`: 緩衝區中保留的語音的最大持續時間（以秒為單位）。

- **`activation_threshold`** _(float)_ (optional) - Default: `0.5`: 將某一幀視為語音的閾值。閾值越高，偵測結果越保守，但可能會錯過軟語音。較低的閾值可實現更靈敏的檢測，但可能會將噪音識別為語音。

- **`sample_rate`** _(Literal[8000, 16000])_ (optional) - Default: `16000`: 推理的取樣率（僅支援8KHz和16KHz）。

- **`force_cpu`** _(bool)_ (optional) - Default: `True`: 強制使用 CPU 進行推理。

## Additional resources

以下資源提供了有關使用 LiveKit Silero VAD 插件的更多資訊。

- **[Python package](https://pypi.org/project/livekit-plugins-silero/)**: PyPI 上的 `livekit-plugins-silero` 包。

- **[Plugin reference](https://docs.livekit.io/reference/python/v1/livekit/plugins/silero/index.html.md#livekit.plugins.silero.VAD)**: LiveKit Silero VAD 插件的參考。

- **[GitHub repo](https://github.com/livekit/agents/tree/main/livekit-plugins/livekit-plugins-silero)**: 查看原始程式碼或為 LiveKit Silero VAD 外掛程式做出貢獻。

- **[Silero VAD project](https://github.com/snakers4/silero-vad)**: 為 LiveKit Silero VAD 外掛程式提供支援的開源 VAD 模型。

- **[Transcriber](https://github.com/livekit-examples/python-agents-examples/tree/main/pipeline-stt/transcriber.py)**: 在 `AgentSession` 之外使用獨立 VAD 和 STT 的範例。
