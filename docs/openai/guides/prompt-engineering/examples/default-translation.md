# Translation

![](./assets/default-translation.png)

翻譯自然語言文本。

Translate natural language text.

**Prompt**

|||
|-------|------|
|**SYSTEM**|您將收到一個英文句子，您的任務是將其翻譯成法語。<br/>You will be provided with a sentence in English, and your task is to translate it into French.|
|**USER**|My name is Jane. What is yours?|

**Sample response**

```
Mon nom est Jane. Quel est le tien?
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
      "content": "You will be provided with a sentence in English, and your task is to translate it into French."
    },
    {
      "role": "user",
      "content": "My name is Jane. What is yours?"
    }
  ],
  "temperature": 0.7,
  "max_tokens": 64,
  "top_p": 1
}'
```
