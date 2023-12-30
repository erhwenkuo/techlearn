# Tweet classifier

![](./assets/default-mood-color.png)

將文字描述轉換為顏色。

Turn a text description into a color.

**Prompt**

|||
|-------|------|
|**SYSTEM**|您將獲得一種心情的描述，您的任務是產生與其匹配的顏色的 CSS 程式碼。使用名為 “css_code” 的單一鍵將輸出寫入 json。<br/>You will be provided with a description of a mood, and your task is to generate the CSS code for a color that matches it. Write your output in json with a single key called "css_code".|
|**USER**|Blue sky at dusk.|

**Sample response**

```
{
  "css_code": "background-color: #1a237e;"
}
```

**API request**

```bash
curl https://api.openai.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
  "model": "gpt-3.5-turbo",
  "messages": [
    {
      "role": "system",
      "content": "You will be provided with a description of a mood, and your task is to generate the CSS code for a color that matches it. Write your output in json with a single key called \"css_code\"."
    },
    {
      "role": "user",
      "content": "Blue sky at dusk."
    }
  ],
  "temperature": 0.7,
  "max_tokens": 64,
  "top_p": 1
}'
```
