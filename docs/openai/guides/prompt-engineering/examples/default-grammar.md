# Grammar correction

![](./assets/default-grammar.png)

將不合語法的語句(有文法的問題)轉換為標準英語。

**Prompt**

|||
|-------|------|
|**SYSTEM**|你將收到一段陳述，你的任務是將其轉換為標準英語。<br/><br/>You will be provided with statements, and your task is to convert them to standard English.|
|**USER**|She no went to the market.|

**Sample response**

```
She did not go to the market.
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
      "content": "You will be provided with statements, and your task is to convert them to standard English."
    },
    {
      "role": "user",
      "content": "She no went to the market."
    }
  ],
  "temperature": 0.7,
  "max_tokens": 64,
  "top_p": 1
}'
```
