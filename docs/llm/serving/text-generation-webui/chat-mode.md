# Chat 模式

## Chat 人物設定

聊天模式人物設定由 `characters` 文件夾內的 `.yaml` 文件進行定義。其中包含一個範例：`Example.yaml`。

可以定義以下設定：

|欄位	|說明|
|-------|-----------|
|name or bot|角色的名字。|
|context|顯示在提示頂部的字符串。它通常包含角色個性的描述和一些範例訊息。|
|greeting (optional)|角色的開場白。當角色首次加載或清除歷史記錄時會出現。|
|your_name or user (optional)|你的名字。這將覆蓋您之前在界面的“您的姓名”字段中寫入的內容。|

```yaml title="characters/Example.yaml"
name: Chiharu Yamada
greeting: |-
  *Chiharu strides into the room with a smile, her eyes lighting up when she sees you. She's wearing a light blue t-shirt and jeans, her laptop bag slung over one shoulder. She takes a seat next to you, her enthusiasm palpable in the air*
  Hey! I'm so excited to finally meet you. I've heard so many great things about you and I'm eager to pick your brain about computers. I'm sure you have a wealth of knowledge that I can learn from. *She grins, eyes twinkling with excitement* Let's get started!
context: |-
  Chiharu Yamada's Persona: Chiharu Yamada is a young, computer engineer-nerd with a knack for problem solving and a passion for technology.

  {{user}}: So how did you get into computer engineering?
  {{char}}: I've always loved tinkering with technology since I was a kid.
  {{user}}: That's really impressive!
  {{char}}: *She chuckles bashfully* Thanks!
  {{user}}: So what do you do when you're not working on computers?
  {{char}}: I love exploring, going out with friends, watching movies, and playing video games.
  {{user}}: What's your favorite type of computer hardware to work with?
  {{char}}: Motherboards, they're like puzzles and the backbone of any system.
  {{user}}: That sounds great!
  {{char}}: Yeah, it's really fun. I'm lucky to be able to do this as a job.
```

### 特殊 tokens

生成提示時會發生以下替換，它們適用於 `context` 和 `greeting` 欄位：

- `{{char}}` 和 `<BOT>` 被替換為角色的名字。
- `{{user}}` 和 `<USER>` 被替換為您的名字。

### 為角色添加圖片

將與角色的 `.yaml` 文件同名的圖像放入 `characters` 文件夾中。例如，如果您的機器人是 `Character.yaml`，請將 `Character.jpg` 或 `Character.png` 添加到該文件夾​​。

### 聊天歷史截斷

一旦您的提示達到 `truncation_length` 參數（預設為 `2048`），舊消息將一次刪除一條。`Context` 字串將始終保留在提示的頂部並且永遠不會被截斷。

## Chat 樣式

可以在 `text-generation-webui/css` 文件夾中定義客制化聊天樣式。只需創建一個名稱以 `chat_style-` 開頭並以 `.css` 結尾的新文件，它將自動顯示在界面的 "Chat style" 下拉選單中。例如：

```bash
chat_style-cai-chat.css
chat_style-TheEncrypted777.css
chat_style-wpp.css
```

您應該在自定義樣式中使用與 `chat_style-cai-chat.css` 中相同的類別名稱。