# Function Calling 功能

原文: [OpenAI — Function Calling Feature](https://levelup.gitconnected.com/openai-function-calling-feature-f846579c9c69)

OpenAI 為其 API 推出了一系列引人注目的更新，讓用戶驚嘆不已。在這些更新中，函數呼叫(Function Calling)功能是最重要的一項。透過利用此函數呼叫功能，模型現在能夠確定要呼叫的適當函數來解決特定問題。這項非凡的功能以前是 Langchain 獨有的，但現在可以透過 OpenAI 輕鬆取得。

本文旨在深入探討函數呼叫特性，全面了解其本質與功能。我們將首先提供函數呼叫功能的精確定義。隨後，我們將深入研究該功能被證明具有無價價值的各種應用程式和場景。最後，為了說明函數呼叫功能的實際實現，我們將提供一系列現實生活中的範例，供您從中汲取靈感。讀完本文後，您將充分掌握函數呼叫功能及其巨大潛力。

## 什麼是函數呼叫功能?

OpenAI 最近透過引入一系列新功能增強了其 API，其中函數呼叫功能是一項重大更新。透過整合函數呼叫功能，開發人員現在能夠為 `gpt-4-0613` 和 `gpt-3.5-turbo-0613` 模型定義函數。透過這樣做，開發人員可以巧妙地指示模型產生一個 JSON 對象，其中包含智慧執行這些函數所需的參數。這使開發人員在使用 API 及其功能時擁有更大的控制力和靈活性。

函數呼叫的結合代表了一種新穎且可靠的方法，可以將 GPT 的功能與外部工具和 API 無縫整合。這項突破使開發人員能夠利用 OpenAI API 創建廣泛的應用程式。模型的最新迭代經過了細緻的微調，使它們能夠根據使用者輸入有效地識別需要呼叫功能的情況。此外，模型已最佳化，可以產生符合指定函數簽章的 JSON 回應。

除了這些進步之外，函數呼叫還使開發人員能夠從模型獲取結構化資料輸出。此功能提供了一種結構化且有組織的格式來檢索相關信息，從而增強了 API 在各種應用程式中的可用性和效率。智慧函數呼叫與接收結構化資料響應的能力相結合，為開發人員利用 OpenAI 模型的強大功能並創建創新解決方案開闢了新的可能性。

## 2. 函數呼叫的用例

函數呼叫功能被譽為 OpenAI API 自推出以來最重要的更新，為開發人員提供了廣泛的強大功能來增強其應用程式。讓我們探討一些可能性：

1. **Chatbots** 透過函數呼叫功能，開發人員可以創建利用外部工具回答問題的聊天機器人。例如，像「給 Anya 發電子郵件詢問她下週五是否想喝咖啡」這樣的查詢可以轉換為諸如 `send_email(to: string, body: string)` 之類的函數呼叫。同樣，諸如「波士頓的天氣怎麼樣？」之類的問題可以透過函數呼叫 `get_current_weather(location: string, unit: 'celsius' | 'fahrenheit')` 來處理。

2. **Converting natural language into API calls or database queries** 函數呼叫功能可以將自然語言轉換為 API 呼叫或資料庫查詢。例如，「本月我的十大客戶是誰？」之類的問題。可以轉換為內部 API 呼叫，例如 `get_customers_by_revenue(start_date: string, end_date: string, limit: int)`。同樣，諸如「Acme, Inc. 上個月下了多少訂單？」之類的查詢。可以使用 `sql_query(query: string)` 轉換為 SQL 查詢。

3. **Extracting structured data from text** 利用函數呼叫功能，開發人員可以從文字內容中提取結構化資料。例如，定義一個名為 `extract_people_data(people: [{name: string,birthday: string, location: string}])` 的函數將能夠提取維基百科文章中提到的所有個人，包括他們的姓名、生日和位置。

## 3. 實作

要利用 OpenAI API 檢索特定城市今天的天氣，您可以按照以下步驟操作：

### Step 1

使用函數和使用者輸入進行 API 呼叫。

**Request**

```bash hl_lines="6-25"
curl https://api.openai.com/v1/chat/completions -u :$OPENAI_API_KEY -H 'Content-Type: application/json' -d '{
  "model": "gpt-3.5-turbo-0613",
  "messages": [
    {"role": "user", "content": "What is the weather like in Boston?"}
  ],
  "functions": [
    {
      "name": "get_current_weather",
      "description": "Get the current weather in a given location",
      "parameters": {
        "type": "object",
        "properties": {
          "location": {
            "type": "string",
            "description": "The city and state, e.g. San Francisco, CA"
          },
          "unit": {
            "type": "string",
            "enum": ["celsius", "fahrenheit"]
          }
        },
        "required": ["location"]
      }
    }
  ]
}'
```

在此範例中，我們向 OpenAI API 端點發出 POST 請求，將模型指定為 `gpt-3.5-turbo-0613`。我們包含一條詢問波士頓天氣的用戶訊息。此外，我們定義了一個名為 `get_current_weather` 的函數，其中包含必要的參數，例如 location（城市和州）和 unit（「攝氏度」或「華氏度」）。

**Response**

使用提供的請求進行 API 呼叫後，您將收到類似於以下內容的回應：

```json hl_lines="9-11"
{
  "id": "chatcmpl-123",
  "...": "...",
  "choices": [{
    "index": 0,
    "message": {
      "role": "assistant",
      "content": null,
      "function_call": {
        "name": "get_current_weather",
        "arguments": "{ \"location\": \"Boston, MA\"}"
      }
    },
    "finish_reason": "function_call"
  }]
}
```

在回應中，關鍵部分是 choices 欄位。它包含一系列潛在的模型響應。這裡，模型回應指示名稱為 `get_current_weather` 的函數呼叫以及指定位置為 `Boston, MA` 的對應參數。

### Step 2

第三方 API：使用模型回應呼叫您的 API

**Request**

```bash
curl https://weatherapi.com/...
```

**Response**

```json
{ "temperature": 22, "unit": "celsius", "description": "Sunny" }
```

### Step 3

OpenAI API：將回應傳回模型進行總結

**Request**

```bash
curl https://api.openai.com/v1/chat/completions -u :$OPENAI_API_KEY -H 'Content-Type: application/json' -d '{
  "model": "gpt-3.5-turbo-0613",
  "messages": [
    {"role": "user", "content": "What is the weather like in Boston?"},
    {"role": "assistant", "content": null, "function_call": {"name": "get_current_weather", "arguments": "{ \"location\": \"Boston, MA\"}"}},
    {"role": "function", "name": "get_current_weather", "content": "{\"temperature\": "22", \"unit\": \"celsius\", \"description\": \"Sunny\"}"}
  ],
  "functions": [
    {
      "name": "get_current_weather",
      "description": "Get the current weather in a given location",
      "parameters": {
        "type": "object",
        "properties": {
          "location": {
            "type": "string",
            "description": "The city and state, e.g. San Francisco, CA"
          },
          "unit": {
            "type": "string",
            "enum": ["celsius", "fahrenheit"]
          }
        },
        "required": ["location"]
      }
    }
  ]
}'
```

**Response**

```json
{
  "id": "chatcmpl-123",
  ...
  "choices": [{
    "index": 0,
    "message": {
      "role": "assistant",
      "content": "The weather in Boston is currently sunny with a temperature of 22 degrees Celsius.",
    },
    "finish_reason": "stop"
  }]
}
```

## 結論

OpenAI API 中引入的函數呼叫功能標誌著利用人工智慧模型的力量的一個重要里程碑。它使開發人員能夠無縫整合外部工具和 API，執行智慧函數呼叫並獲得結構化資料輸出。此更新為創建創新應用程式提供了無數的可能性，例如與外部系統互動的聊天機器人、將自然語言轉換為 API 或資料庫查詢以及從非結構化文字中提取結構化資料。

