# 擴展插件

擴展由 `text-generation-webui/extensions` 子文件夾內名為 `script.py` 的文件來定義。如果在服務啟動的命令行加上 `--extensions` 旗標後指定了文件夾名稱，則它們會在啟動時被加載。

例如，`extensions/silero_tts/script.py` 使用 `python server.py --extensions silero_tts` 來加載。

## 擴展存儲庫

- [text-generation-webui-extensions](https://github.com/oobabooga/text-generation-webui-extensions)

上面的存儲庫包含用戶擴展的目錄。

如果您創建擴展，歡迎您將其託管在 GitHub 存儲庫中並提交 PR 將其添加到列表中。

## 內建擴展

|Extension|Description|
|---------|-----------|
|[api](https://github.com/oobabooga/text-generation-webui/tree/main/extensions/api)| 創建一個具有兩個端點的 API，一個用於在 `/api/v1/stream` 端口 `5005` 處進行流式傳輸，另一個用於在 `/api/v1/generate` 端口 `5000` 處進行阻塞。這是 webui 的主要API 。|
|[openai](https://github.com/oobabooga/text-generation-webui/tree/main/extensions/openai)| 創建一個模仿 OpenAI API 的 API，可用作直接替代品。|
|[multimodal](https://github.com/oobabooga/text-generation-webui/tree/main/extensions/multimodal) | 添加多模態支持（文本+圖像）。有關詳細說明，請參閱擴展目錄中的 README.md。|
|[google_translate](https://github.com/oobabooga/text-generation-webui/tree/main/extensions/google_translate)| 使用 Google Translate 自動翻譯輸入和輸出。|
|[silero_tts](https://github.com/oobabooga/text-generation-webui/tree/main/extensions/silero_tts)| 使用 Silero 的文本轉語音擴展。在聊天模式下使用時，響應將替換為音頻小部件。|
|[elevenlabs_tts](https://github.com/oobabooga/text-generation-webui/tree/main/extensions/elevenlabs_tts)| 使用 ElevenLabs API 的文本轉語音擴展。您需要一個 API 密鑰才能使用它。|
|[whisper_stt](https://github.com/oobabooga/text-generation-webui/tree/main/extensions/whisper_stt)| 允許您在聊天模式下使用麥克風輸入內容。 |
|[sd_api_pictures](https://github.com/oobabooga/text-generation-webui/tree/main/extensions/sd_api_pictures)| 允許您在聊天模式下向機器人請求圖片，這些圖片將使用 AUTOMATIC1111 Stable Diffusion API 生成。請參閱此處的示例。|
|[character_bias](https://github.com/oobabooga/text-generation-webui/tree/main/extensions/character_bias)| 這只是一個非常簡單的示例，在聊天模式下機器人回复的開頭添加隱藏字符串。|
|[send_pictures](https://github.com/oobabooga/text-generation-webui/blob/main/extensions/send_pictures/)| 創建一個圖像上傳字段，可用於在聊天模式下將圖像發送到機器人。字幕是使用 BLIP 自動生成的。|
|[gallery](https://github.com/oobabooga/text-generation-webui/blob/main/extensions/gallery/)| 創建一個包含聊天角色及其圖片的圖庫。|
|[superbooga](https://github.com/oobabooga/text-generation-webui/tree/main/extensions/superbooga)| 使用 ChromaDB 創建任意大的偽上下文的擴展，將文本文件、URL 或粘貼文本作為輸入。基於 https://github.com/kaiokendev/superbig。|
|[ngrok](https://github.com/oobabooga/text-generation-webui/tree/main/extensions/ngrok)| 允許您使用 ngrok 反向隧道服務（免費）遠程訪問 Web UI。它是內置 Gradio `--share` 功能的替代方案。 |
|[perplexity_colors](https://github.com/oobabooga/text-generation-webui/tree/main/extensions/perplexity_colors)| 根據從模型 logits 導出的相關概率，為輸出文本中的每個標記著色。|

