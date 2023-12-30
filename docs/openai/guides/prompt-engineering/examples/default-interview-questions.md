# Interview questions

![](./assets/default-interview-questions.png)

建立面試問題。

Create interview questions.

**Prompt**

|||
|-------|------|
|**USER**|建立一個包含 8 個問題的清單，用於訪問一位科幻作家。<br/><br/>Create a list of 8 questions for an interview with a science fiction author.|

**Sample response**

```
1. Can you tell us a bit about your background and what inspired you to become a science fiction author?
2. How do you approach world-building in your science fiction novels? What elements do you prioritize when creating a believable and immersive fictional universe?
3. Science fiction often explores societal or technological advancements. What themes or ideas do you find most fascinating to explore in your writing?
4. How do you balance scientific accuracy with imaginative storytelling in your science fiction works?
5. Many science fiction authors use their stories as a platform to comment on contemporary issues. How do you incorporate social or political commentary into your writing, if at all?
6. Can you share any unique challenges or difficulties you've encountered while writing science fiction, and how you overcame them?
7. Science fiction often explores the potential impact of technology on society. In your opinion, what responsibilities do science fiction authors have in shaping public perception or understanding of emerging technologies?
8. Can you discuss any upcoming projects or ideas you're currently working on, and what readers can expect from your future science fiction novels?

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
      "role": "user",
      "content": "Create a list of 8 questions for an interview with a science fiction author."
    }
  ],
  "temperature": 0.5,
  "max_tokens": 64,
  "top_p": 1
}'
```
