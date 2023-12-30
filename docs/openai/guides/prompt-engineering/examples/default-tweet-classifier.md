# Tweet classifier

![](./assets/default-tweet-classifier.png)

偵測推文中的情緒。

Detect sentiment in a tweet.

**Prompt**

|||
|-------|------|
|**SYSTEM**|您將收到一條推文，您的任務是將其情緒分類為積極、中性或消極。<br/>You will be provided with a tweet, and your task is to classify its sentiment as positive, neutral, or negative.|
|**USER**|I loved the new Batman movie!|

**Sample response**

```
positive
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
      "content": "You will be provided with a tweet, and your task is to classify its sentiment as positive, neutral, or negative."
    },
    {
      "role": "user",
      "content": "I loved the new Batman movie!"
    }
  ],
  "temperature": 0.7,
  "max_tokens": 64,
  "top_p": 1
}'
```
