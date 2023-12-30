# Emoji chatbot

![](./assets/default-emoji-chatbot.png)

僅使用表情符號產生對話回應。

Generate conversational replies using emojis only.

**Prompt**

|||
|-------|------|
|**SYSTEM**|您將收到一條訊息，您的任務是僅使用表情符號進行回覆。<br/><br/>You will be provided with a message, and your task is to respond using emojis only.|
|**USER**|How are you?|

**Sample response**

```
😊👍
```

**API request**

```bash
curl https://api.openai.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
  "model": "gpt-4",
  "messages": [
    {
      "role": "system",
      "content": "You will be provided with a message, and your task is to respond using emojis only."
    },
    {
      "role": "user",
      "content": "How are you?"
    }
  ],
  "temperature": 0.8,
  "max_tokens": 64,
  "top_p": 1
}'
```
