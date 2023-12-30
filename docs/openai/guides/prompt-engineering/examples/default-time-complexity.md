# Calculate time complexity

![](./assets/default-time-complexity.png)

求函數的時間複雜度。

Find the time complexity of a function.

**Prompt**

|||
|-------|------|
|**SYSTEM**|您將獲得 Python 程式碼，您的任務是計算其時間複雜度。<br/>You will be provided with Python code, and your task is to calculate its time complexity.|
|**USER**|def foo(n, k):<br/>        accum = 0<br/>        for i in range(n):<br/>            for l in range(k):<br/>                accum += i<br/>        return accum|

**Sample response**

```
The time complexity of this code is O(n * k), where n is the value of the variable `n` and k is the value of the variable `k`. This is because there are two nested loops, one iterating `n` times and the other iterating `k` times. The `accum += i` statement inside the inner loop is executed `n * k` times, resulting in a time complexity of O(n * k).
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
      "content": "You will be provided with Python code, and your task is to calculate its time complexity."
    },
    {
      "role": "user",
      "content": "def foo(n, k):\n        accum = 0\n        for i in range(n):\n            for l in range(k):\n                accum += i\n        return accum"
    }
  ],
  "temperature": 0.7,
  "max_tokens": 64,
  "top_p": 1
}'
```
