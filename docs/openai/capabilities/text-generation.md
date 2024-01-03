# Text generation models

OpenAI 的文本生成模型（通常稱為生成式預訓練 Transformer 或大型語言模型）經過訓練可以理解自然語言、程式碼和圖像。這些模型提供文字輸出來回應其輸入。這些模型的輸入也稱為“提示”。設計提示本質上是如何「編程」大型語言模型，通常是透過提供說明或一些如何成功完成任務的範例。

使用 OpenAI 的文本生成模型，您可以建立應用程式來：

- 文件草案
- 編寫電腦程式碼
- 回答有關知識庫的問題
- 分析文字
- 為軟體提供自然語言介面
- 一系列科目的導師
- 翻譯語言
- 模擬遊戲角色

隨著 `gpt-4-vision-preview` 的發布，您現在可以建立一個還可以處理和理解圖像的系統。

若要透過 OpenAI API 使用這些模型之一，您將發送包含輸入和 API 金鑰的請求，並接收包含模型輸出的回應。我們的最新模型 `gpt-4` 和 `gpt-3.5-turbo` 可透過聊天完成 API 端點進行存取。

||MODEL FAMILIES|API ENDPOINT|
|---|---|---|
|Newer models (2023–)|gpt-4, gpt-4 turbo, gpt-3.5-turbo|[https://api.openai.com/v1/chat/completions](https://api.openai.com/v1/chat/completions)|
|Updated legacy models (2023)|gpt-3.5-turbo-instruct, babbage-002, davinci-002|[https://api.openai.com/v1/completions](https://api.openai.com/v1/completions)|
|Legacy models (2020–2022)|text-davinci-003, text-davinci-002, davinci, curie, babbage, ada|[https://api.openai.com/v1/completions](https://api.openai.com/v1/completions)|

您可以在 [chat playground](https://platform.openai.com/playground?mode=chat) 中嘗試各種模型。如果您不確定要使用哪個模型，請使用 `gpt-3.5-turbo` 或 `gpt-4`。

## Chat Completions API

聊天模型將訊息列表作為輸入，並傳回模型產生的訊息作為輸出。儘管聊天格式旨在使多輪對話變得容易，但它對於沒有任何對話的單輪任務也同樣有用。

聊天完成 API 呼叫範例如下所示：

=== "curl"

    ```bash
    curl https://api.openai.com/v1/chat/completions \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -d '{
        "model": "gpt-3.5-turbo",
        "messages": [
            {
                "role": "system",
                "content": "You are a helpful assistant."
            },
            {
                "role": "user",
                "content": "Who won the world series in 2020?"
            },
            {
                "role": "assistant",
                "content": "The Los Angeles Dodgers won the World Series in 2020."
            },
            {
                "role": "user",
                "content": "Where was it played?"
            }
        ]
    }'
    ```

=== "python"

    ```python
    from openai import OpenAI
    client = OpenAI()

    response = client.chat.completions.create(
        model="gpt-3.5-turbo",
        messages=[
            {"role": "system", "content": "You are a helpful assistant."},
            {"role": "user", "content": "Who won the world series in 2020?"},
            {"role": "assistant", "content": "The Los Angeles Dodgers won the World Series in 2020."},
            {"role": "user", "content": "Where was it played?"}
        ]
    )
    ```

=== "node.js"

    ```nodejsrepl
    import OpenAI from "openai";

    const openai = new OpenAI();

    async function main() {
        const completion = await openai.chat.completions.create({
            messages: [{"role": "system", "content": "You are a helpful assistant."},
                {"role": "user", "content": "Who won the world series in 2020?"},
                {"role": "assistant", "content": "The Los Angeles Dodgers won the World Series in 2020."},
                {"role": "user", "content": "Where was it played?"}],
            model: "gpt-3.5-turbo",
        });

        console.log(completion.choices[0]);
    }

    main();
    ```

要了解更多信息，您可以查看聊天 API 的完整 [API 參考文件](https://platform.openai.com/docs/api-reference/chat)。

主要輸入是 **messages** 參數。訊息必須是訊息物件的陣列，其中每個物件都有一個角色（"system", "user", 或 "assistant"）和內容。對話可以短至一則訊息，也可以來回多次。

通常，對話首先由系統訊息格式化，然後是交替的使用者訊息和助理訊息。

系統訊息有助於設定助手的行為。例如，您可以修改助手的個性或提供有關其在整個對話過程中應如何表現的具體說明。但請注意，系統訊息是可選的，沒有系統訊息的模型行為可能類似於使用通用訊息，例如「你是一個有用的助手」。

用戶訊息提供助理回應的請求或評論。助理訊息儲存先前的助理回應，但也可以由您編寫以給出所需行為的範例。

當使用者指令引用先前的消息時，包含對話歷史記錄非常重要。在上面的範例中，用戶的最後一個問題是“在哪裡播放的？”僅在 2020 年世界職棒大賽的先前訊息的上下文中才有意義。由於模型沒有過去請求的記憶，因此所有相關資訊必須作為每個請求中的對話歷史記錄的一部分提供。如果對話無法滿足模型的令牌限制，則需要以某種方式縮短。

**Chat Completions 回應格式**

聊天完成 API 回應範例如下所示：

```json
{
  "choices": [
    {
      "finish_reason": "stop",
      "index": 0,
      "message": {
        "content": "The 2020 World Series was played in Texas at Globe Life Field in Arlington.",
        "role": "assistant"
      },
      "logprobs": null
    }
  ],
  "created": 1677664795,
  "id": "chatcmpl-7QyqpwdfhqwajicIEznoc6Q47XAyW",
  "model": "gpt-3.5-turbo-0613",
  "object": "chat.completion",
  "usage": {
    "completion_tokens": 17,
    "prompt_tokens": 57,
    "total_tokens": 74
  }
}
```

助理的回覆可以透過以下方式提取：

=== "python"

    ```python
    response['choices'][0]['message']['content']
    ```

=== "node.js"

    ```nodejsrepl
    completion.choices[0].message.content
    ```

每個回應都將包含一個 `finish_reason`。 `finish_reason` 的可能值為：

- `stop`: API 傳回完整訊息，或透過 stop 參數提供的停止序列之一終止的訊息
- `length`: 由於 max_tokens 參數或令牌限制導致模型輸出不完整
- `function_call`: 模型決定呼叫一個函數
- `content_filter`: 由於我們的內容過濾器中的標記而省略了內容
- `null`: API 回應仍在進行中或不完整

根據輸入參數，模型響應可能包括不同的資訊。

## JSON 模式

使用聊天完成的常見方法是透過在系統訊息中指定，指示模型始終傳回對您的用例有意義的 JSON 物件。雖然這在某些情況下確實有效，但有時模型可能會產生無法解析為有效 JSON 物件的輸出。

為了防止這些錯誤並提高模型效能，在呼叫 `gpt-4-1106-preview` 或 `gpt-3.5-turbo-1106` 時，可以將 `response_format` 設定為 `{"type":"json_object"}` 以啟用 JSON 模式。啟用 JSON 模式時，模型僅限於產生解析為有效 JSON 物件的字串。

- 使用 JSON 模式時，請務必指示模型透過對話中的某些訊息（例如透過系統訊息）產生 JSON。如果您不包含產生 JSON 的明確指令，則模型可能會產生無休止的空白流，並且請求可能會持續運行，直到達到令牌限制。為了幫助確保您不會忘記，如果字串 "JSON" 沒有出現在上下文中的某個位置，API 將拋出錯誤。

- 如果 `finish_reason` 為 `length`，則模型傳回的訊息中的 JSON 可能是部分的（即被截斷），這表示產生超出了 `max_tokens` 或會話超出了令牌限制。為了防止這種情況，請在解析回應之前檢查 `finish_reason` 。

- JSON 模式不保證輸出符合任何特定模式，僅保證其有效且解析無錯誤。


=== "curl"

    ```bash
    curl https://api.openai.com/v1/chat/completions \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -d '{
        "model": "gpt-3.5-turbo-1106",
        "response_format": { "type": "json_object" },
        "messages": [
        {
            "role": "system",
            "content": "You are a helpful assistant designed to output JSON."
        },
        {
            "role": "user",
            "content": "Who won the world series in 2020?"
        }
        ]
    }'
    ```

=== "python"

    ```python
    from openai import OpenAI
    client = OpenAI()

    response = client.chat.completions.create(
        model="gpt-3.5-turbo-1106",
        response_format={ "type": "json_object" },
        messages=[
            {"role": "system", "content": "You are a helpful assistant designed to output JSON."},
            {"role": "user", "content": "Who won the world series in 2020?"}
        ]
    )
    print(response.choices[0].message.content)
    ```

=== "node.js"

    ```nodejsrepl
    import OpenAI from "openai";

    const openai = new OpenAI();

    async function main() {
        const completion = await openai.chat.completions.create({
            messages: [
                {
                    role: "system",
                    content: "You are a helpful assistant designed to output JSON.",
                },
                { role: "user", content: "Who won the world series in 2020?" },
            ],
            model: "gpt-3.5-turbo-1106",
            response_format: { type: "json_object" },
        });
        console.log(completion.choices[0].message.content);
    }

    main();
    ```

在此範例中，回應包含一個 JSON 對象，如下所示：

```json
"content": "{\"winner\": \"Los Angeles Dodgers\"}"`
```

!!! info
    請注意，當模型在函數呼叫過程中產生參數時，JSON 模式始終處於啟用狀態。


## 可重現呼叫結果

預設情況下，Chat Completions　是　non-deterministic（這意味著模型輸出可能會因請求而異）。話雖這麼說，我們透過讓您存取 `seed` 參數和 `system_fingerprint` 回應欄位來提供對確定性輸出的一些控制。

若要跨 API 呼叫接收確定性輸出，您可以：

- 將 `seed` 參數設定為您選擇的任何整數，並在您想要確定性輸出的請求中使用相同的值。
- 確保所有其他參數（如 `prompt` 或 `temperature`）在請求之間完全相同。

有時，由於 OpenAI 對我們端的模型配置進行必要的更改，確定性可能會受到影響。為了幫助您追蹤這些更改，我們公開了 `system_fingerprint` 欄位。如果該值不同，您可能會因我們對系統所做的變更而看到不同的輸出。

## 管理 tokens

語言模型以稱為標記的區塊的形式讀取和寫入文字。在英語中，標記可以短至一個字符，也可以長至一個單字（例如，`a` 或 `apple`），並且在某些語言中，標記甚至可以短於一個字符，甚至長於一個單字。

例如，字串 `ChatGPT is Great!` 被編碼為六個標記：[`Chat`、`G`、`PT`、`is`、`great`、`!`]。

API 呼叫中的令牌總數會影響：

- 您的 API 呼叫費用是多少（按令牌支付）
- API 呼叫需要多長時間，因為寫入更多令牌需要更多時間
- 您的 API 呼叫是否有效，因為令牌總數必須低於模型的最大限制（`gpt-3.5-turbo` 為 `4097` 個令牌）

輸入和輸出令牌都計入這些數量。例如，如果您的 API 呼叫在訊息輸入中使用了 10 個令牌，而您在訊息輸出中收到了 20 個令牌，則您將需要支付 30 個令牌。但請注意，對於某些模型，輸入中的令牌與輸出中的代幣的每個令牌的價格是不同的。

若要查看 API 呼叫使用了多少令牌，請檢查 API 回應中的使用欄位（例如，`response['usage']['total_tokens']`）。

像 `gpt-3.5-turbo` 和 `gpt-4` 這樣的聊天模型使用令牌的方式與完成 API 中可用的模型相同，但由於它們基於訊息的格式，因此更難計算對話將使用多少個令牌。

若要在不呼叫 API 的情況下查看文字字串中有多少個令牌，請使用 OpenAI 的 [tiktoken](https://github.com/openai/tiktoken) Python 函式庫。範例程式碼可以在 OpenAI Cookbook 的關於如何使用 tiktoken 計算令牌的指南中找到。

傳遞到 API 的每個訊息都會消耗內容、角色和其他欄位中的令牌數量，以及一些用於幕後格式化的額外令牌。這在未來可能會略有改變。

如果對話有太多令牌而無法滿足模型的最大限制（例如，`gpt-3.5-turbo` 超過 4097 個標記），您將必須截斷、省略或以其他方式縮小文本，直到適合為止。請注意，如果從訊息輸入中刪除訊息，模型將丟失它的所有知識。

請注意，很長的對話更有可能收到不完整的回應。例如，長度為 4090 個令牌的 `gpt-3.5-turbo` 對話將在僅 6 個令牌後就被切斷。


## Completions API (Legacy)

完成 API 端點於 2023 年 7 月收到最終更新，並且具有與新的聊天完成端點不同的介面。輸入不是訊息列表，而是稱為 `prompt` 的自由格式文字字串。

API 呼叫範例如下所示：

=== "python"

    ```python
    from openai import OpenAI
    client = OpenAI()

    response = client.completions.create(
        model="gpt-3.5-turbo-instruct",
        prompt="Write a tagline for an ice cream shop."
    )
    ```

=== "node.js"

    ```nodejsrepl
    const completion = await openai.completions.create({
        model: 'gpt-3.5-turbo-instruct',
        prompt: 'Write a tagline for an ice cream shop.'
    });
    ```


**Completions 回應格式**

完成 API 回應範例如下所示：

```json
{
  "choices": [
    {
      "finish_reason": "length",
      "index": 0,
      "logprobs": null,
      "text": "\n\n\"Let Your Sweet Tooth Run Wild at Our Creamy Ice Cream Shack"
    }
  ],
  "created": 1683130927,
  "id": "cmpl-7C9Wxi9Du4j1lQjdjhxBlO22M61LD",
  "model": "gpt-3.5-turbo-instruct",
  "object": "text_completion",
  "usage": {
    "completion_tokens": 16,
    "prompt_tokens": 10,
    "total_tokens": 26
  }
}
```

在 Python 中，可以使用 `response['choices'][0]['text']` 來提取輸出。

回應格式與聊天完成 API 的回應格式類似。

## Chat Completions 對比 Completions

透過使用單一使用者訊息建構請求，可以使聊天完成格式與完成格式類似。例如，可以透過以下完成提示將英文翻譯成法文：

```bash title="Completion"
Translate the following English text to French: "{text}"
```

```json title="Chat Completion"
[{"role": "user", "content": 'Translate the following English text to French: "{text}"'}]
```

這些 API 之間的差異在於每個 API 中可用的底層模型。Chat completions 是 OpenAI 最強大的模型 `gpt-4` 和最具成本效益的模型 `gpt-3.5-turbo` 的介面。