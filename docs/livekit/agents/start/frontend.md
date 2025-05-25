# Web and mobile frontends

> 透過網頁或行動應用程式讓您的AI Voice 代理變得生動活潑。

## Overview

LiveKit Agents 已準備好使用適用於 `JavaScript`、 `Swift`、 `Android`、 `Flutter`、 `React Native` 等的 [LiveKit SDK](https://docs.livekit.io/home/client/connect.md) 來與您首選的前端平台整合。您的代理可以透過 LiveKit WebRTC 與您的前端進行通信，它提供快速可靠的即時連接。

例如，一個簡單的語音代理訂閱用戶的 microphone track 並發布自己的軌道。[文本轉錄](https://docs.livekit.io/agents/build/text.md) 也可作為文字串流使用。具有視覺功能的更複雜的代理可以訂閱從用戶的 webcam 或共享螢幕發布的視訊軌道。代理還可以發布自己的影片來實現虛擬化身或其他功能。

在所有這些情況下，LiveKit SDK 都是生產級別的並且易於使用，因此您可以建立有用且高級的代理，而不必擔心即時媒體傳輸的複雜性。本主題包含為您的代理建立高品質前端的資源和技巧。

## Starter apps

LiveKit 建議使用以下入門應用程式之一在您首選的平台上快速啟動並運行。每個應用程式都是根據 MIT 許可證開源的，因此您可以根據自己的需求自由修改它。行動應用程式需要託管令牌伺服器，在 [LiveKit Cloud Sandbox](https://cloud.livekit.io/projects/p_/sandbox/templates/token-server) 中包含一個以用於開發目的。

- **[SwiftUI Voice Assistant](https://github.com/livekit-examples/voice-assistant-swift)**: SwiftUI 內建的原生 iOS、macOS 和 visionOS 語音 AI 助理。

- **[Next.js Voice Assistant](https://github.com/livekit-examples/voice-assistant-frontend)**: 使用 React 和 Next.js 建構的網路語音 AI 助理。

- **[Flutter Voice Assistant](https://github.com/livekit-examples/voice-assistant-flutter)**: 使用 Flutter 建立的跨平台語音 AI 助理應用程式。

- **[React Native Voice Assistant](https://github.com/livekit-examples/voice-assistant-react-native)**: 使用 React Native 和 Expo 建立的原生語音 AI 助理應用程式。

- **[Android Voice Assistant](https://github.com/livekit-examples/android-voice-assistant)**: 使用 Kotlin 和 Jetpack Compose 建立的原生 Android 語音 AI 助理應用程式。

## Media and text

要了解有關即時媒體和文字流的更多信息，請參閱以下文件。

- **[Media tracks](https://docs.livekit.io/home/client/tracks.md)**: 與您的代理一起使用麥克風、揚聲器、相機和螢幕共用。
- **[Text streams](https://docs.livekit.io/home/client/data/text-streams.md)**: 發送和接收即時文字和轉錄。

## Data sharing

要在前端和代理之間共享圖片、檔案或任何其他類型的數據，您可以使用以下功能。

- **[Byte streams](https://docs.livekit.io/home/client/data/byte-streams.md)**: 發送和接收圖像、文件或任何其他資料。

- **[Data packets](https://docs.livekit.io/home/client/data/packets.md)**: 用於發送和接收任何類型資料的低階 API。

## State and control

在某些情況下，您的代理程式和前端程式碼可能需要自訂狀態和配置整合以滿足應用程式的要求。在這些情況下，LiveKit 即時狀態和資料功能可用於建立緊密耦合且反應迅速的體驗。

`AgentSession` 自動管理 `lk.agent.state` 參與者屬性以包含 `initializing`、`listening`、`thinking` 或 `speaking` 中的適當字串值。

- **[State synchronization](https://docs.livekit.io/home/client/state.md)**: 在您的前端和代理之間共用自訂狀態。

- **[RPC](https://docs.livekit.io/home/client/data/rpc.md)**: 從另一端定義並呼叫代理或前端上的方法。

## Audio visualizer

`React`、 `SwiftUI`、 `Android Compose` 和 `Flutter` 的 LiveKit 元件 SDK 包含一個音訊視覺化元件，可用於讓您的語音代理程式在您的應用程式中呈現視覺效果。

有關完整範例，請參閱上面列出的範例應用程式。以下文件是使用這些元件的快速指南：

**React**:

安裝 [React 元件](https://github.com/livekit/components-js/tree/main/packages/react) 和 [styles](https://github.com/livekit/components-js/tree/main/packages/styles) 軟體包，以使用[useVoiceAssistant](https://docs.livekit.io/reference/components/react/hook/usevoiceassistant.md) hook 和 [BarVisualizer](https://docs.livekit.io/reference/components/react/component/barvisualizer.md)。

這些元件在 [LiveKitRoom](https://docs.livekit.io/reference/components/react/component/livekitroom.md) 或 [RoomContext.Provider](https://docs.livekit.io/reference/components/react/component/roomcontextmd)) 中自動運作。

另請參閱 [VoiceAssistantControlBar](https://docs.livekit.io/reference/components/react/component/voiceassistantcontrolbar.md)，它為語音代理應用程式提供了一組簡單的通用 UI 控制項。

```typescript
"use client";

import "@livekit/components-styles";

import {
  useVoiceAssistant,
  BarVisualizer,
} from "@livekit/components-react";

export default function SimpleVoiceAssistant() {
  // Get the agent's audio track and current state
  const { state, audioTrack } = useVoiceAssistant();
  return (
    <div className="h-80">
      <BarVisualizer state={state} barCount={5} trackRef={audioTrack} style={{}} />
      <p className="text-center">{state}</p>
    </div>
  );
}

```

---

**Swift**:

首先從 [https://github.com/livekit/components-swift](https://github.com/livekit/components-swift) 安裝元件包。

然後您可以使用 `AgentBarAudioVisualizer` 視圖顯示代理程式的音訊和狀態：

```swift
struct AgentView: View {
    // Load the room from the environment
    @EnvironmentObject private var room: Room

    // Find the first agent participant in the room
    private var agentParticipant: RemoteParticipant? {
        for participant in room.remoteParticipants.values {
            if participant.kind == .agent {
                return participant
            }
        }
        
        return nil
    }

    // Reads the agent state property
    private var agentState: AgentState {
        agentParticipant?.agentState ?? .initializing
    }

    var body: some View {
          AgentBarAudioVisualizer(audioTrack: participant.firstAudioTrack, agentState: agentState, barColor: .primary, barCount: 5)
              .id(participant.firstAudioTrack?.id)
    }
}

```

---

**Android**:

首先從 [https://github.com/livekit/components-android](https://github.com/livekit/components-android) 安裝元件包。

然後，您可以使用 `rememberVoiceAssistant` 和 `VoiceAssistantBarVisualizer` 可組合項目來顯示視覺化器，假設您已經在 `RoomScope` 可組合項目內。

```kotlin
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.padding
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import io.livekit.android.compose.state.rememberVoiceAssistant
import io.livekit.android.compose.ui.audio.VoiceAssistantBarVisualizer

@Composable
fun AgentAudioVisualizer(modifier: Modifier = Modifier) {
    // Get the voice assistant instance
    val voiceAssistant = rememberVoiceAssistant()
    
    // Display the audio visualization
    VoiceAssistantBarVisualizer(
        voiceAssistant = voiceAssistant,
        modifier = modifier
            .padding(8.dp)
            .fillMaxWidth()
    )
}

```

---

**Flutter**:

首先從 [https://github.com/livekit/components-flutter](https://github.com/livekit/components-flutter) 安裝元件包。

```bash
flutter pub add livekit_components

```

創建 `room` 時啟用音訊視覺化：

```dart
// Enable audio visualization when creating the Room
final room = Room(roomOptions: const RoomOptions(enableVisualizer: true));

```
然後，您可以使用 `SoundWaveformWidget` 來顯示代理程式的音訊視覺化，假設您正在使用 `RoomContext`：

```dart
import 'package:flutter/material.dart';
import 'package:livekit_client/livekit_client.dart';
import 'package:livekit_components/livekit_components.dart' hide ParticipantKind;
import 'package:provider/provider.dart';

/// Shows a simple audio visualizer for an agent participant
class AgentView extends StatelessWidget {
  const AgentView({super.key});

  @override
  Widget build(BuildContext context) {
    return Consumer<RoomContext>(
      builder: (context, roomContext, child) {
        // Find the agent participant in the room
        final agentParticipant = roomContext.room.remoteParticipants.values
            .where((p) => p.kind == ParticipantKind.AGENT)
            .firstOrNull;
        
        if (agentParticipant == null) {
          return const SizedBox.shrink();
        }
        
        // Get the agent's audio track for visualization
        final audioTrack = agentParticipant.audioTrackPublications
            .firstOrNull?.track as AudioTrack?;
            
        if (audioTrack == null) {
          return const SizedBox.shrink();
        }
        
        // Show the waveform visualization
        return SoundWaveformWidget(
          audioTrack: audioTrack,
          options: AudioVisualizerOptions(
            width: 32,
            minHeight: 32,
            maxHeight: 256,
            color: Theme.of(context).colorScheme.primary,
            count: 7,
          ),
        );
      },
    );
  }
}

```

## Authentication

LiveKit SDK 需要 [token](https://docs.livekit.io/home/get-started/authentication.md) 才能連接到房間。在 Web 應用程式中，您通常可以將簡單的令牌端點作為應用程式的一部分。對於行動應用，您需要一個單獨的 [令牌伺服器](https://docs.livekit.io/home/server/generating-tokens.md)。

## Virtual avatars

您的前端可以使用來自受支援提供者的虛擬化身來包含您的代理程式的視訊表示。 LiveKit 全面支援所有支援平台上的影片渲染。[啟動應用程式](#starter-apps) 包含對虛擬化身的支援。有關更多資訊和受支援的提供者列表，請參閱文件：

- **[Virtual avatars](https://docs.livekit.io/agents/integrations/avatar.md)**: 使用虛擬化身讓您的代理在您的應用程式中具有視覺存在。

## Responsiveness tips

本節包含一些建議，可讓您的應用程式對使用者更具回應能力。

### Minimize connection time

若要將您的使用者連接到您的 agent，必須執行以下步驟：

1. 獲取訪問令牌 (access token)。
2. 用戶連接到房間。
3. 派遣 (dispatch) 代理進程。
4. 代理連接到房間。
5. 用戶和代理發布並訂閱彼此的媒體 tracks。

如果按順序進行，則需要幾秒鐘才能完成。您可以透過消除或併行化這些步驟來減少這段時間。

**Option 1: "Warm" token**

在這種情況下，您的應用程式將在使用者登入時為使用者產生具有較長有效期的令牌。當您需要連接到房間時，令牌已經在您的前端可用。

**Option 2: Dispatch agent during token generation**

在這種情況下，您的應用程式將樂觀地創建一個房間，並在生成令牌的同時調度代理，使用[顯式代理調度]（https://docs.livekit.io/agents/worker/agent-dispatch.md#explicit）。這允許用戶和代理同時連接到房間。

### Connection indicators

透過將各種事件連結到使用者的一個或兩個狀態指示器（而不是多個離散步驟和 UI 變更），使您的應用程式反應更快，即使連線速度較慢。  有關如何監視連接狀態和其他事件的更多信息，請參閱[事件處理](https://docs.livekit.io/home/client/events.md)文件。

如果您的代理無法連接，您應該通知用戶並允許他們重試，而不是讓他們對著空房間講話。

- **Room connection**: 大多數 SDK 中都可以使用 `room.connect` 方法來等待結果，並且大多數還提供 `room.connectionState` 屬性。也可監視 `Disconnected` 事件以了解連線何時遺失。
- **Agent presence**: 使用 `participant.kind === ParticipantKind.AGENT` 監視 `ParticipantConnected` 事件。
- **Agent state**: 訪問代理的狀態 (`initializing`, `listening`, `thinking`, or `speaking`)。
- **Track subscription**: 監聽 `TrackSubscribed` 事件以了解您的媒體何時被訂閱。

### Effects

您應該使用聲音效果、觸覺回饋和視覺效果來讓您的代理感覺更靈敏。這在長時間思考狀態下尤其重要（例如，執行外部尋找或使用工具時）。[視覺化工具](#audio-visualizer) 包含基本的 `thinking` 狀態指示，並且還允許使用者注意到他們的音訊何時無法運作。如需更進階的效果，請使用[狀態與控制](#state-control)功能在前端觸發效果。