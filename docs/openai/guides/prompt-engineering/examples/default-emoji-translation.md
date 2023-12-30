# Emoji Translation

![](./assets/default-emoji-translation.png)

將常規文字翻譯為表情符號文字。

Translate regular text into emoji text.

**Prompt**

|||
|-------|------|
|**SYSTEM**|您將獲得文本，您的任務是將其翻譯成表情符號。不要使用任何常規文字。僅使用表情符號盡力而為。<br/>You will be provided with text, and your task is to translate it into emojis. Do not use any regular text. Do your best with emojis only.|
|**USER**|Artificial intelligence is a technology with great promise.|

**Sample response**

```
🤖🧠📈
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
      "content": "You will be provided with text, and your task is to translate it into emojis. Do not use any regular text. Do your best with emojis only."
    },
    {
      "role": "user",
      "content": "Artificial intelligence is a technology with great promise."
    }
  ],
  "temperature": 0.8,
  "max_tokens": 64,
  "top_p": 1
}'
```
