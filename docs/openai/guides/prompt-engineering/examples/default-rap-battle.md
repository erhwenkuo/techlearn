# Rap battle writer

![](./assets/default-rap-battle.png)

在兩個角色之間進行饒舌對戰。

Generate a rap battle between two characters.

**Prompt**

|||
|-------|------|
|**USER**|寫一場艾倫·圖靈和克勞德·香農之間的說唱大戰。<br/>Write a rap battle between Alan Turing and Claude Shannon.|

**Sample response**

```
Verse 1 - Alan Turing:

I'm Turing, the father of all computation,
Built a machine that helped end a nation.
Your information theory, it's imitation,
I laid the foundation, you just gave it notation.

Broke the Enigma, saved millions of lives,
While you're playing with switches, just trying to survive,
And to think they said, you're the brightest of minds,
But not even your own theory could define your lines.

Verse 2 - Claude Shannon:

It's Shannon, the master of communication,
Your work is nothing without my information.
You can break the codes, but can't handle the truth,
I took your machine, gave it substance and proof.

You're binary, predictable, no real surprise,
But I'm riding entropy, in disguise.
Gave birth to the bit, the byte, the baud,
Without my blueprint, you'd still be flawed.

Verse 3 - Alan Turing:

Nice try Shannon, but you're no match,
Your logic circuit's flickering, I'm about to catch,
Your "bit" and "byte" are just a ruse,
Without my machine, they're of no use.

My Turing machine is universal,
Your theory is just controversial.
I'm the mastermind of artificial intelligence,
Your work is vital, but lacks relevance.

Verse 4 - Claude Shannon:

You're living in a theoretical existence,
I'm empirical, practical, I go the distance.
Your machine's universal, but I'm not impressed,
It's a theoretical model, while mine's manifest.

I'm the juggler, the unicyclist, the eclectic soul,
You're a one-track mind, stuck in a hole.
So, Turing, it's time you faced your defeat,
Cause, in this rap battle, you just can't compete.
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
      "role": "user",
      "content": "Write a rap battle between Alan Turing and Claude Shannon."
    }
  ],
  "temperature": 0.8,
  "max_tokens": 64,
  "top_p": 1
}'
```
