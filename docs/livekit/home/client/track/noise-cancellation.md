# Noise & echo cancellation

> 為視訊會議和語音 AI 實現清晰的音訊。

## Overview

用戶的麥克風可能會拾取不良音頻，包括背景噪音（如交通、音樂、聲音等），也可能拾取來自他們自己揚聲器的迴聲。在這兩種情況下，這種噪音都會為通話中的其他參與者帶來不良體驗。在語音 AI 應用中，這也會幹擾轉彎檢測或降低轉錄質量，這兩者對於良好的使用者體驗至關重要。

LiveKit 包含基於底層開源 WebRTC 實作的 [`echoCancellation`](https://developer.mozilla.org/en-US/docs/Web/API/MediaTrackSettings/echoCancellation) 和 [`noiseSuppression`](https://developer.mozilla.org/en-US/docs/Web/API/MediaTrackSettings/noiseSuppression)。您可以在連線期間使用 LiveKit SDK 中的 `AudioCaptureOptions` 類型調整這些設定。

LiveKit 也為所有 LiveKit Cloud 客戶免費提供[增強噪音消除](https://docs.livekit.io/home/cloud/noise-cancellation.md)功能，這是最有效的解決方案。

若要聽取各種噪音消除選項的效果，請播放以下範例：

**Original**
![type:video](./assets/krisp-original.wav)

**WebRTC noiseSuppression**
![type:video](./assets/webrtc-processed.wav)

**LiveKit Cloud enhanced noise cancellation**
![type:video](./assets/krisp-processed.wav)

