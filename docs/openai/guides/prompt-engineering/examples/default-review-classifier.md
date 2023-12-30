# Review classifier

![](./assets/default-review-classifier.png)

根據一組標籤對使用者評論進行分類。

Classify user reviews based on a set of tags.

**Prompt**

|||
|-------|------|
|**SYSTEM**|<pre>You will be presented with user reviews and your job is to provide a set of tags from the following list. Provide your answer in bullet point form. Choose ONLY from the list of tags provided here (choose either the positive or the negative tag but NOT both):<br/>    <br/>    - Provides good value for the price OR Costs too much<br/>    - Works better than expected OR Did not work as well as expected<br/>    - Includes essential features OR Lacks essential features<br/>    - Easy to use OR Difficult to use<br/>    - High quality and durability OR Poor quality and durability<br/>    - Easy and affordable to maintain or repair OR Difficult or costly to maintain or repair<br/>    - Easy to transport OR Difficult to transport<br/>    - Easy to store OR Difficult to store<br/>    - Compatible with other devices or systems OR Not compatible with other devices or systems<br/>    - Safe and user-friendly OR Unsafe or hazardous to use<br/>    - Excellent customer support OR Poor customer support<br/>    - Generous and comprehensive warranty OR Limited or insufficient warranty</pre>|
|**USER**|<pre>I recently purchased the Inflatotron 2000 airbed for a camping trip and wanted to share my experience with others. Overall, I found the airbed to be a mixed bag with some positives and negatives.<br/>    <br/>    Starting with the positives, the Inflatotron 2000 is incredibly easy to set up and inflate. It comes with a built-in electric pump that quickly inflates the bed within a few minutes, which is a huge plus for anyone who wants to avoid the hassle of manually pumping up their airbed. The bed is also quite comfortable to sleep on and offers decent support for your back, which is a major plus if you have any issues with back pain.<br/>    <br/>    On the other hand, I did experience some negatives with the Inflatotron 2000. Firstly, I found that the airbed is not very durable and punctures easily. During my camping trip, the bed got punctured by a stray twig that had fallen on it, which was quite frustrating. Secondly, I noticed that the airbed tends to lose air overnight, which meant that I had to constantly re-inflate it every morning. This was a bit annoying as it disrupted my sleep and made me feel less rested in the morning.<br/>    <br/>    Another negative point is that the Inflatotron 2000 is quite heavy and bulky, which makes it difficult to transport and store. If you'\''re planning on using this airbed for camping or other outdoor activities, you'\''ll need to have a large enough vehicle to transport it and a decent amount of storage space to store it when not in use.</pre>|

**Sample response**

```
- Easy to use
- Poor quality and durability
- Difficult to transport
- Difficult to store
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
      "content": "You will be presented with user reviews and your job is to provide a set of tags from the following list. Provide your answer in bullet point form. Choose ONLY from the list of tags provided here (choose either the positive or the negative tag but NOT both):\n    \n    - Provides good value for the price OR Costs too much\n    - Works better than expected OR Did not work as well as expected\n    - Includes essential features OR Lacks essential features\n    - Easy to use OR Difficult to use\n    - High quality and durability OR Poor quality and durability\n    - Easy and affordable to maintain or repair OR Difficult or costly to maintain or repair\n    - Easy to transport OR Difficult to transport\n    - Easy to store OR Difficult to store\n    - Compatible with other devices or systems OR Not compatible with other devices or systems\n    - Safe and user-friendly OR Unsafe or hazardous to use\n    - Excellent customer support OR Poor customer support\n    - Generous and comprehensive warranty OR Limited or insufficient warranty"
    },
    {
      "role": "user",
      "content": "I recently purchased the Inflatotron 2000 airbed for a camping trip and wanted to share my experience with others. Overall, I found the airbed to be a mixed bag with some positives and negatives.\n    \n    Starting with the positives, the Inflatotron 2000 is incredibly easy to set up and inflate. It comes with a built-in electric pump that quickly inflates the bed within a few minutes, which is a huge plus for anyone who wants to avoid the hassle of manually pumping up their airbed. The bed is also quite comfortable to sleep on and offers decent support for your back, which is a major plus if you have any issues with back pain.\n    \n    On the other hand, I did experience some negatives with the Inflatotron 2000. Firstly, I found that the airbed is not very durable and punctures easily. During my camping trip, the bed got punctured by a stray twig that had fallen on it, which was quite frustrating. Secondly, I noticed that the airbed tends to lose air overnight, which meant that I had to constantly re-inflate it every morning. This was a bit annoying as it disrupted my sleep and made me feel less rested in the morning.\n    \n    Another negative point is that the Inflatotron 2000 is quite heavy and bulky, which makes it difficult to transport and store. If you'\''re planning on using this airbed for camping or other outdoor activities, you'\''ll need to have a large enough vehicle to transport it and a decent amount of storage space to store it when not in use."
    }
  ],
  "temperature": 0.7,
  "max_tokens": 64,
  "top_p": 1
}'
```
