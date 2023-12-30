# Spreadsheet creator

![](./assets/default-spreadsheet-gen.png)

建立各種資料的電子表格。

Create spreadsheets of various kinds of data.

**Prompt**

|||
|-------|------|
|**USER**|建立包含頂級科幻電影以及發行年份的兩列 CSV。<br/><br/>Create a two-column CSV of top science fiction movies along with the year of release.|

**Sample response**

```
Movie,Year
2001: A Space Odyssey,1968
Blade Runner,1982
The Matrix,1999
Star Wars: Episode IV - A New Hope,1977
E.T. the Extra-Terrestrial,1982
The Terminator,1984
Inception,2010
Interstellar,2014
The Fifth Element,1997
Back to the Future,1985
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
      "content": "Create a two-column CSV of top science fiction movies along with the year of release."
    }
  ],
  "temperature": 0.5,
  "max_tokens": 64,
  "top_p": 1
}'
```
