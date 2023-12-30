# Product name generator

![](./assets/default-product-name-gen.png)

根據描述和種子字詞產生產品名稱。

Generate product names from a description and seed words.

**Prompt**

|||
|-------|------|
|**SYSTEM**|您將獲得產品描述和種子詞，您的任務是產生產品名稱。<br/>You will be provided with a product description and seed words, and your task is to generate product names.|
|**USER**|Product description: A home milkshake maker<br/>Seed words: fast, healthy, compact.|

**Sample response**

```
1. QuickBlend
2. FitShake
3. MiniMix
4. TurboShake
5. HealthWhiz
6. MightyMixer
7. SlimBlend
8. SpeedyShake
9. NutriBlend
10. CompactMilkshake
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
      "content": "You will be provided with a product description and seed words, and your task is to generate product names."
    },
    {
      "role": "user",
      "content": "Product description: A home milkshake maker\n    Seed words: fast, healthy, compact."
    }
  ],
  "temperature": 0.8,
  "max_tokens": 64,
  "top_p": 1
}'
```
