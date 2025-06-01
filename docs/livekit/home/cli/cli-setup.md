# CLI Setup

> 安裝 LiveKit CLI 並使用範例前端應用程式測試您的設定。

## Install LiveKit CLI

=== "macOS"

    ```text
    brew update && brew install livekit-cli

    ```

=== "Linux"

    ```text
    curl -sSL https://get.livekit.io/cli | bash

    ```

    !!! tip
        您也可以從[此處](https://github.com/livekit/livekit-cli/releases/latest)下載最新的預編譯二進位。

=== "Windows"

    ```text
    winget install LiveKit.LiveKitCLI

    ```

    !!! tip
        您也可以從[此處](https://github.com/livekit/livekit-cli/releases/latest)下載最新的預編譯二進位。


=== "From Source"

    此 repo 使用 [Git LFS](https://git-lfs.github.com/) 來嵌入影片資源。在繼續之前，請確保您的機器上安裝了 git-lfs。

    ```text
    git clone github.com/livekit/livekit-cli
    make install

    ```


`lk` 是 LiveKit 的 CLI 公用套件。它讓您可以方便地從命令列存取伺服器 API、建立令牌並產生測試流量。更多詳細信息，請參閱 `livekit-cli` [GitHub repo](https://github.com/livekit/livekit-cli#usage) 中的文檔。

## Authenticate with Cloud (optional)

對於 LiveKit Cloud 用戶，您可以使用 Cloud 專案驗證 CLI 以建立 API 金鑰和機密。這使您可以使用 CLI，而無需每次手動提供憑證。

```shell
lk cloud auth

```

然後，按照說明從瀏覽器登入。

!!! tip
    如果您想要探索 LiveKit 的 [Agents](https://docs.livekit.io/agents.md) 框架，或想要針對預先建置的前端或令牌伺服器為您的應用程式製作原型，請查看 [Sandboxes](https://docs.livekit.io/home/cloud/sandbox.md)。

## Generate access token

建立或加入 LiveKit [房間](https://docs.livekit.io/home/concepts/api-primitives.md) 的參與者需要 [訪問令牌](https://docs.livekit.io/home/concepts/authentication.md) 才能執行此操作。現在，讓我們透過 CLI 產生一個：

=== "Localhost"

    ```shell
    lk token create \
    --api-key devkey --api-secret secret \
    --join --room test_room --identity test_user \
    --valid-for 24h

    ```

    !!! tip
        確保您在 [開發模式](https://docs.livekit.io/home/self-hosting/local.md#dev-mode) 下本地執行 LiveKit 伺服器。


=== "Cloud"

    ```shell
    lk token create \
    --api-key <PROJECT_KEY> --api-secret <PROJECT_SECRET> \
    --join --room test_room --identity test_user \
    --valid-for 24h

    ```

    或者，您可以[從專案的儀表板產生令牌]（https://cloud.livekit.io/projects/p_/settings/keys）。



## Test with LiveKit Meet

!!! tip
    如果您正在測試 LiveKit Cloud 實例，您可以在專案設定中找到您的「專案 URL」（以 `wss://` 開頭）。

使用範例應用程式 [LiveKit Meet](https://meet.livekit.io) 預覽您的新 LiveKit 實例。在「自訂」標籤中輸入您[先前產生的](#generate-access-token)的令牌。一旦連接，您的麥克風和攝影機將即時傳輸到您的新 LiveKit 實例（以及連接到同一房間的任何其他參與者）！

如果有興趣，這裡是此範例應用的[完整原始碼](https://github.com/livekit-examples/meet)。

### Simulating another publisher

測試多用戶會話的一種方法是[生成](#generate-access-token)第二個令牌（確保 `--identity` 是唯一的），在另一個[瀏覽器選項卡](https://meet.livekit.io)中打開我們的範例應用程式並連接到同一個房間。

另一種方法是使用 CLI 作為模擬參與者並將預先錄製的影片發佈到房間。方法如下：

=== "Localhost"

    ```shell
    lk room join \
    --url ws://localhost:7880 \
    --api-key devkey --api-secret secret \
    --publish-demo --identity bot_user \
    my_first_room

    ```

=== "Cloud"

    ```shell
    lk room join \
    --url <PROJECT_SECURE_WEBSOCKET_ADDRESS> \
    --api-key <PROJECT_API_KEY> --api-secret <PROJECT_SECRET_KEY> \
    --publish-demo --identity bot_user \
    my_first_room

    ```

此指令將循環示範影片發佈到 `my-first-room`。由於文件的編碼方式，瀏覽器在獲得足夠的資料來呈現幀之前可能會出現短暫的延遲。
