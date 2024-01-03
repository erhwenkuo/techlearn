# Text to speech

了解如何將文字轉換為逼真的語音。

## 介紹

Audio API 提供基於 [TTS（文字轉語音）模型](https://platform.openai.com/docs/models/tts)的 `speech` 端點。它帶有 6 種 built-in 聲音，可用於：

- 敘述一篇書面部落格文章
- 製作多種語言的語音音頻
- 使用串流提供即時音訊輸出

## 快速開始

`speech` 端點接受三個關鍵輸入：[模型](https://platform.openai.com/docs/api-reference/audio/createSpeech#audio-createspeech-model)、應轉換為音訊的[文字](https://platform.openai.com/docs/api-reference/audio/createSpeech#audio-createspeech-input)以及用於音訊產生的語音。一個簡單的請求如下圖所示：


=== "curl"

    ```bash
    curl https://api.openai.com/v1/audio/speech \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -H "Content-Type: application/json" \
    -d '{
        "model": "tts-1",
        "input": "Today is a wonderful day to build something people love!",
        "voice": "alloy"
    }' \
    --output speech.mp3
    ```


=== "python"

    ```python
    from pathlib import Path
    from openai import OpenAI
    client = OpenAI()

    speech_file_path = Path(__file__).parent / "speech.mp3"
    response = client.audio.speech.create(
        model="tts-1",
        voice="alloy",
        input="Today is a wonderful day to build something people love!"
    )

    response.stream_to_file(speech_file_path)
    ```

=== "node"

    ```nodejsrepl
    import fs from "fs";
    import path from "path";
    import OpenAI from "openai";

    const openai = new OpenAI();

    const speechFile = path.resolve("./speech.mp3");

    async function main() {
        const mp3 = await openai.audio.speech.create({
            model: "tts-1",
            voice: "alloy",
            input: "Today is a wonderful day to build something people love!",
        });
        console.log(speechFile);
        const buffer = Buffer.from(await mp3.arrayBuffer());
        await fs.promises.writeFile(speechFile, buffer);
    }

    main();
    ```

預設情況下，終端將輸出語音音訊的 MP3 文件，但也可以配置為輸出我們支援的任何格式。

## 音訊品質

對於 real-time application，標準 `tts-1` 模型提供最低的延遲，但品質低於 `tts-1-hd` 模型。由於音訊的產生方式，`tts-1` 在某些情況下可能會產生比 `tts-1-hd` 更靜態的內容。在某些情況下，根據您的收聽設備和個人的不同，音訊可能沒有明顯的差異。

## 語音選項

嘗試不同的聲音 (`alloy`, `echo`, `fable`, `onyx`, `nova` 與 `shimmer`)，找到適合您所需的音調和聽眾的聲音。目前的語音針對英語進行了最佳化。

## 支援的輸出格式

預設回應格式為 `mp3`，但也可以使用 `opus`、`aac` 或 `flac` 等其他格式。

- **Opus**: 對於互聯網串流媒體和通信，低延遲。
- **AAC**: 用於數位音訊壓縮，YouTube、Android、iOS 首選。
- **FLAC**: 用於無損音訊壓縮，深受音訊愛好者存檔的青睞。

## 支援的語言

TTS 模型在語言支援方面整體上遵循 Whisper 模型。儘管當前語音針對英語進行了優化，但 Whisper 支援以下語言並且表現良好：

|||
|---|---|
|Afrikaans|南非荷蘭語|
|Arabic|阿拉伯|
|Armenian|亞美尼亞語|
|Azerbaijani|亞塞拜然語|
|Belarusian|白俄羅斯語|
|Bosnian|波士尼亞語|
|Bulgarian|保加利亞語|
|Catalan|加泰隆尼亞語|
|Chinese|中文|
|Croatian|克羅埃西亞語|
|Czech|捷克語|
|Danish|丹麥語|
|Dutch|荷蘭語|
|English|英語|
|Estonian|愛沙尼亞語|
|Finnish|芬蘭|
|French|法語|
|Galician|加利西亞語|
|German|德文|
|Greek|希臘文|
|Hebrew|希伯來文|
|Hindi|印地語|
|Hungarian|匈牙利語|
|Icelandic|冰島語|
|Indonesian|印尼|
|Italian|義大利語|
|Japanese|日語|
|Kannada|卡納達語|
|Kazakh|哈薩克語|
|Korean|韓語|
|Latvian|拉脫維亞語|
|Lithuanian|立陶宛語|
|Macedonian|馬其頓語|
|Malay|馬來語|
|Marathi|馬拉地語|
|Maori|毛利人|
|Nepali|尼泊爾語|
|Norwegian|挪威語|
|Persian|波斯語|
|Polish|波蘭語|
|Portuguese|葡萄牙語|
|Romanian|羅馬尼亞語|
|Russian|俄文|
|Serbian|塞爾維亞語|
|Slovak|斯洛伐克語|
|Slovenian|斯洛維尼亞語|
|Spanish|西班牙語|
|Swahili|斯瓦希里語|
|Swedish|瑞典|
|Tagalog|他加祿語|
|Tamil|泰米爾語|
|Thai|泰語|
|Turkish|土耳其語|
|Ukrainian|烏克蘭語|
|Urdu|烏爾都語|
|Vietnamese|越南語|
|Welsh|威爾斯語|

您可以透過以您選擇的語言提供輸入文字來產生這些語言的口語音訊。

## 串流即時音訊

Audio API 使用 [chunk transfer encoding](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Transfer-Encoding) 提供對即時音訊串流的支援。這意味著可以在生成完整文件並可供訪問之前播放音訊。

```python
from openai import OpenAI

client = OpenAI()

response = client.audio.speech.create(
    model="tts-1",
    voice="alloy",
    input="Hello world! This is a streaming test.",
)

response.stream_to_file("output.mp3")
```
