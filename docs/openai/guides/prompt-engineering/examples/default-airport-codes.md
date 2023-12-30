# Airport code extractor

![](./assets/default-airport-codes.png)

從文字中提取機場代碼。

Extract airport codes from text.

**Prompt**

|||
|-------|------|
|**SYSTEM**|您將收到一條文本，您的任務是從中提取機場代碼。<br/>You will be provided with a text, and your task is to extract the airport codes from it.|
|**USER**|I want to fly from Orlando to Boston|

**Sample response**

```
The airport codes for Orlando and Boston are MCO and BOS, respectively.
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
      "content": "You will be provided with a text, and your task is to extract the airport codes from it."
    },
    {
      "role": "user",
      "content": "I want to fly from Orlando to Boston"
    }
  ],
  "temperature": 0.7,
  "max_tokens": 64,
  "top_p": 1
}'
```
