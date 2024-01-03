# Speech to text

了解如何將音訊轉換為文字。

## 介紹

Audio API 基於 OpenAI 最先進的開源大型 v2 [Whisper 模型](https://openai.com/blog/whisper/)，提供兩種語音到文字端點、轉錄和翻譯。它們可用於：

- 將音訊轉錄成音訊所使用的任何語言。
- 將音訊翻譯並轉錄成英文。

文件上傳目前限制為 25 MB，並支援以下輸入檔類型：

- `mp3`
- `mp4`
- `mpeg`
- `mpga`
- `m4a`
- `wav`
- `webm`

## 快速開始

### 轉錄

Transcriptions API 將您想要轉錄的音訊檔案以及音訊轉錄所需的輸出檔案格式作為輸入。OpenAI 目前支援多種輸入和輸出檔案格式。

=== "Python"

    ```python
    from openai import OpenAI
    client = OpenAI()

    audio_file= open("/path/to/file/audio.mp3", "rb")

    transcript = client.audio.transcriptions.create(
    model="whisper-1", 
    file=audio_file
    )
    ```

=== "curl"

    ```bash
    curl --request POST \
    --url https://api.openai.com/v1/audio/transcriptions \
    --header 'Authorization: Bearer TOKEN' \
    --header 'Content-Type: multipart/form-data' \
    --form file=@/path/to/file/openai.mp3 \
    --form model=whisper-1
    ```

預設情況下，回應類型將為 json，其中包含原始文字。

```json
{
  "text": "Imagine the wildest idea that you've ever had, and you're curious about how it might scale to something that's a 100, a 1,000 times bigger.
....
}
```

Audio API 還允許您在請求中設定其他參數。例如，如果您想將 `response_format` 設定為 `text`，您的請求將如下所示：

=== "Python"

    ```python
    from openai import OpenAI
    client = OpenAI()

    audio_file = open("speech.mp3", "rb")
    transcript = client.audio.transcriptions.create(
    model="whisper-1", 
    file=audio_file, 
    response_format="text"
    )
    ```

=== "curl"

    ```bash
    curl --request POST \
    --url https://api.openai.com/v1/audio/transcriptions \
    --header 'Authorization: Bearer TOKEN' \
    --header 'Content-Type: multipart/form-data' \
    --form file=@openai.mp3 \
    --form model=whisper-1 \
    --form response_format=text
    ```

[API 參考](https://platform.openai.com/docs/api-reference/audio)包含可用參數的完整清單。

### 翻譯

Translations API 將任何支援語言的音訊檔案作為輸入，並在必要時將音訊轉錄為英文。這與我們的 `/Transcriptions` 端點不同，因為輸出不是原始輸入語言，而是翻譯為英文文字。

=== "Python"

    ```python
    from openai import OpenAI
    client = OpenAI()

    audio_file= open("/path/to/file/german.mp3", "rb")
    transcript = client.audio.translations.create(
        model="whisper-1", 
        file=audio_file
    )
    ```

=== "curl"

    ```bash
    curl --request POST   --url https://api.openai.com/v1/audio/translations   --header 'Authorization: Bearer TOKEN'   --header 'Content-Type: multipart/form-data'   --form file=@/path/to/file/german.mp3   --form model=whisper-1
    ```

在本例中，輸入的音訊是 `german`，輸出的文字如下所示：

```
Hello, my name is Wolfgang and I come from Germany. Where are you heading today?
```

目前 OpenAI 僅支援翻譯成英文。

## 支援的語言

目前，OpenAI 透過 `transcriptions` 和 `translations` 端點支援以下語言：


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

雖然底層模型是在 98 種語言上進行訓練的，但我們只列出了單字錯誤率 (WER) 超過 50% 的語言，這是語音到文字模型準確性的行業標準基準。該模型將傳回上面未列出的語言的結果，但品質較低。

## 更長的輸入

預設情況下，Whisper API 僅支援小於 `25 MB` 的檔案。如果您的音訊檔案比該長度長，則需要將其分成 25 MB 或更少的區塊，或使用壓縮音訊格式。為了獲得最佳性能，我們建議您避免在句子中間中斷音頻，因為這可能會導致丟失一些上下文。

處理此問題的一種方法是使用 [PyDub 開源 Python 套件](https://github.com/jiaaro/pydub)來分割音訊：

```python
from pydub import AudioSegment

song = AudioSegment.from_mp3("good_morning.mp3")

# PyDub handles time in milliseconds
ten_minutes = 10 * 60 * 1000

first_10_minutes = song[:ten_minutes]

first_10_minutes.export("good_morning_10.mp3", format="mp3")
```

## 提示

您可以使用 **提示** 來提高 Whisper API 產生的轉錄文本的品質。模型將嘗試匹配提示的樣式，因此如果提示也這樣做，則更有可能使用大寫和標點符號。然而，目前的提示系統比我們的其他語言模型受到更多限制，並且僅提供對生成的音訊的有限控制。以下是提示如何在不同情況下提供幫助的一些範例：

1. 提示對於糾正模型在音訊中經常錯誤識別的特定單字或首字母縮寫非常有幫助。

2. 若要保留分割為多個片段的檔案的上下文，您可以使用前一個片段的轉錄本來提示模型。這將使轉錄更加準確，因為模型將使用先前音訊中的相關資訊。模型將僅考慮提示的最後 224 個標記，並忽略先前的任何內容。對於多語言輸入，Whisper 使用自訂標記器。對於僅英語輸入，它使用標準 GPT-2 分詞器，這兩種分詞器都可以透過開源 Whisper Python 套件進行存取。

3. 有時，模型可能會跳過記錄中的標點符號。您可以透過使用包含標點符號的簡單提示來避免這種情況：“您好，歡迎來到我的講座。”

4. 該模型還可能遺漏音訊中常見的填充詞。如果您想在成績單中保留填充詞，您可以使用包含它們的提示：“嗯，讓我這樣想，嗯......好吧，這就是我的想法。”

5. 有些語言可以用不同的方式書寫，例如簡體中文或繁體中文。預設情況下，模型可能不會總是使用您想要的成績單寫作風格。您可以透過使用您喜歡的寫作風格的提示來改進這一點。

## 提高可靠性

正如我們在提示部分中探討的那樣，使用 Whisper 時面臨的最常見挑戰之一是模型通常無法識別不常見的單字或首字母縮寫。為了解決這個問題，我們重點介紹了在這些情況下提高 Whisper 可靠性的不同技術：

**使用提示參數**

第一種方法涉及使用可選的提示參數來傳遞正確拼字的字典。

由於 Whisper 沒有使用指令追蹤技術進行訓練，因此其運作方式更像是基本 GPT 模型。請務必記住，Whisper 僅考慮提示的前 244 個標記。

```python
transcribe(filepath, prompt="ZyntriQix, Digique Plus, CynapseFive, VortiQore V8, EchoNix Array, OrbitalLink Seven, DigiFractal Matrix, PULSE, RAPT, B.R.I.C.K., Q.U.A.R.T.Z., F.L.I.N.T.")
```

雖然它會提高可靠性，但該技術僅限於 244 個字符，因此您的 SKU 列表需要相對較小，才能成為可擴展的解決方案。


**使用 GPT-4 進行後處理**

第二種方法涉及使用 `GPT-4` 或 `GPT-3.5-Turbo` 的後處理步驟。

我們先透過 `system_prompt` 變數提供 GPT-4 的指令。與我們先前對提示參數所做的類似，我們可以定義我們的公司和產品名稱。

```python
system_prompt = "You are a helpful assistant for the company ZyntriQix. Your task is to correct any spelling discrepancies in the transcribed text. Make sure that the names of the following products are spelled correctly: ZyntriQix, Digique Plus, CynapseFive, VortiQore V8, EchoNix Array, OrbitalLink Seven, DigiFractal Matrix, PULSE, RAPT, B.R.I.C.K., Q.U.A.R.T.Z., F.L.I.N.T. Only add necessary punctuation such as periods, commas, and capitalization, and use only the context provided."

def generate_corrected_transcript(temperature, system_prompt, audio_file):
    response = client.chat.completions.create(
        model="gpt-4",
        temperature=temperature,
        messages=[
            {
                "role": "system",
                "content": system_prompt
            },
            {
                "role": "user",
                "content": transcribe(audio_file, "")
            }
        ]
    )
    return response['choices'][0]['message']['content']

corrected_text = generate_corrected_transcript(0, system_prompt, fake_company_filepath)
```

如果您在自己的音訊檔案上嘗試此操作，您會發現 GPT-4 設法修正了記錄中的許多拼字錯誤。由於其更大的上下文窗口，該方法可能比使用 Whisper 的提示參數更具可擴展性，並且更可靠，因為 GPT-4 可以以 Whisper 無法實現的方式進行指示和引導，因為缺乏後續指令。

