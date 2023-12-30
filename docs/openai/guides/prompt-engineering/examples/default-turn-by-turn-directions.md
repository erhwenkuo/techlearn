# Turn by turn directions

![](./assets/default-turn-by-turn-directions.png)

將自然語言轉換為逐步導航。

Convert natural language to turn-by-turn directions.

**Prompt**

|||
|-------|------|
|**SYSTEM**|您將獲得一段文本，您的任務是根據該文本建立逐段路線的編號清單。<br/>You will be provided with a text, and your task is to create a numbered list of turn-by-turn directions from it.|
|**USER**|Go south on 95 until you hit Sunrise boulevard then take it east to us 1 and head south. Tom Jenkins bbq will be on the left after several miles.|

**Sample response**

```
1. Start by going south on 95.
2. Continue on 95 until you reach Sunrise Boulevard.
3. Take a right onto Sunrise Boulevard and head east.
4. Follow Sunrise Boulevard until you reach US 1.
5. Take a left onto US 1 and head south.
6. After several miles, you will find Tom Jenkins BBQ on the left.
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
      "content": "You will be provided with a text, and your task is to create a numbered list of turn-by-turn directions from it."
    },
    {
      "role": "user",
      "content": "Go south on 95 until you hit Sunrise boulevard then take it east to us 1 and head south. Tom Jenkins bbq will be on the left after several miles."
    }
  ],
  "temperature": 0.3,
  "max_tokens": 64,
  "top_p": 1
}'
```
