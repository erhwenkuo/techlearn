# Function from specification

![](./assets/default-function-from-spec.png)

根據規範建立 Python 函數。

Create a Python function from a specification.

**Prompt**

|||
|-------|------|
|**USER**|編寫一個 Python 函數，將圖像的檔案路徑作為輸入，將圖像作為 numpy 數組載入到記憶體中，然後在周邊的行和列比閾值暗時裁剪它們。使用行和列的平均值來決定是否應將它們標記為刪除。<br/><br/>Write a Python function that takes as input a file path to an image, loads the image into memory as a numpy array, then crops the rows and columns around the perimeter if they are darker than a threshold value. Use the mean value of rows and columns to decide if they should be marked for deletion.|

**Sample response**

```
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
      "content": "Write a Python function that takes as input a file path to an image, loads the image into memory as a numpy array, then crops the rows and columns around the perimeter if they are darker than a threshold value. Use the mean value of rows and columns to decide if they should be marked for deletion."
    }
  ],
  "temperature": 0.7,
  "max_tokens": 64,
  "top_p": 1
}'
```
