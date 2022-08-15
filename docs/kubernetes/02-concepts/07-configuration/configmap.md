# ConfigMap

ConfigMap 是一種 API 對象，用來將非機密性的數據保存到鍵值對中。使用時， **Pods** 可以將其用作環境變量、命令行參數或者存儲卷中的配置文件。

ConfigMap 將你的環境配置信息和 **容器鏡像** 解耦，便於應用配置的修改。

!!! warning
    ConfigMap 並不提供保密或者加密功能。如果你想存儲的數據是機密的，請使用 Secret， 或者使用其他第三方工具來保證你的數據的私密性，而不是用 ConfigMap。

## 動機

使用 ConfigMap 來將你的配置數據和應用程序代碼分開。

比如，假設你正在開發一個應用，它可以在你自己的電腦上（用於開發）和在雲上 （用於實際流量）運行。你的代碼裡有一段是用於查看環境變量 DATABASE_HOST，在本地運行時， 你將這個變量設置為 localhost，在雲上，你將其設置為引用 Kubernetes 集群中的 公開數據庫組件的 服務。

這讓你可以獲取在雲中運行的容器鏡像，並且如果有需要的話，在本地調試完全相同的代碼。

ConfigMap 在設計上不是用來保存大量數據的。{==在 ConfigMap 中保存的數據不可超過 1 MiB==}。如果你需要保存超出此尺寸限制的數據，你可能希望考慮掛載存儲卷 或者使用獨立的數據庫或者文件服務。

## ConfigMap 對象

ConfigMap 是一個 API [對象](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/kubernetes-objects/)， 讓你可以存儲其他對象所需要使用的配置。和其他 Kubernetes 對象都有一個 `spec` 不同的是，ConfigMap 使用 `data` 和 `binaryData` 字段。這些字段能夠接收鍵-值對作為其取值。 `data` 和 `binaryData` 字段都是可選的。 data 字段設計用來保存 UTF-8 字符串，而 binaryData 則被設計用來保存二進制數據作為 base64 編碼的字串。

ConfigMap 的名字必須是一個合法的 [DNS 子域名](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/names#dns-subdomain-names)。

`data` 或 `binaryData` 字段下面的每個鍵的名稱都必須由字母數字字符或者 `-`、`_` 或 `.` 組成。在 `data` 下保存的鍵名不可以與在 `binaryData` 下出現的鍵名有重疊。

從 v1.19 開始，你可以添加一個 `immutable` 字段到 ConfigMap 定義中， 創建[不可變更的 ConfigMap](https://kubernetes.io/zh-cn/docs/concepts/configuration/configmap/#configmap-immutable)。

## ConfigMaps 和 Pods

你可以寫一個引用 ConfigMap 的 Pod 的 `spec`，並根據 ConfigMap 中的數據在該 Pod 中配置容器。這個 Pod 和 ConfigMap 必須要在同一個 **命名空間** 中。

這是一個 ConfigMap 的示例，它的一些鍵只有一個值，其他鍵的值看起來像是 配置的片段格式。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # 類屬性鍵；每一個鍵都映射到一個簡單的值
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"

  # 類文件鍵
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5    
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true    
```

你可以使用四種方式來使用 ConfigMap 配置 Pod 中的容器：

1. 在容器命令和參數內
2. 容器的環境變量
3. 在只讀卷裡面添加一個文件，讓應用來讀取
4. 編寫代碼在 Pod 中運行，使用 Kubernetes API 來讀取 ConfigMap

這些不同的方法適用於不同的數據使用方式。對前三個方法，kubelet 使用 ConfigMap 中的數據在 Pod 中啟動容器。

第四種方法意味著你必須編寫代碼才能讀取 ConfigMap 和它的數據。然而， 由於你是直接使用 Kubernetes API，因此只要 ConfigMap 發生更改， 你的應用就能夠通過訂閱來獲取更新，並且在這樣的情況發生的時候做出反應。通過直接進入 Kubernetes API，這個技術也可以讓你能夠獲取到不同的名字空間裡的 ConfigMap。

下面是一個 Pod 的示例，它通過使用 `game-demo` 中的值來配置一個 Pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
    - name: demo
      image: alpine
      command: ["sleep", "3600"]
      env:
        # 定義環境變量
        - name: PLAYER_INITIAL_LIVES # 請注意這里和 ConfigMap 中的鍵名是不一樣的
          valueFrom:
            configMapKeyRef:
              name: game-demo           # 這個值來自 ConfigMap
              key: player_initial_lives # 需要取值的键
        - name: UI_PROPERTIES_FILE_NAME
          valueFrom:
            configMapKeyRef:
              name: game-demo
              key: ui_properties_file_name
      volumeMounts:
      - name: config
        mountPath: "/config"
        readOnly: true
  volumes:
    # 你可以在 Pod 級別設置卷，然後將其掛載到 Pod 內的容器中
    - name: config
      configMap:
        # 提供你想要掛載的 ConfigMap 的名字
        name: game-demo
        # 來自 ConfigMap 的一組鍵，將被創建為文件
        items:
        - key: "game.properties"
          path: "game.properties"
        - key: "user-interface.properties"
          path: "user-interface.properties"
```

ConfigMap 不會區分單行屬性值和多行類似文件的值，重要的是 Pods 和其他對像如何使用這些值。

上面的例子定義了一個卷並將它作為 `/config` 文件夾掛載到 demo 容器內， 創建兩個文件，`/config/game.properties` 和 `/config/user-interface.properties`， 儘管 ConfigMap 中包含了四個鍵。這是因為 Pod 定義中在 `volumes` 節指定了一個 `items` 數組。如果你完全忽略 `items` 數組，則 ConfigMap 中的每個鍵都會變成一個與該鍵同名的文件， 因此你會得到四個文件。

## 使用 ConfigMap

ConfigMap 可以作為數據卷掛載。 ConfigMap 也可被系統的其他組件使用， 而不一定直接暴露給 Pod。例如，ConfigMap 可以保存系統中其他組件要使用的配置數據。

ConfigMap 最常見的用法是為同一命名空間裡某 Pod 中運行的容器執行配置。你也可以單獨使用 ConfigMap。

比如，你可能會遇到基於 ConfigMap 來調整其行為的 插件 或者 operator。

### 在 Pod 中將 ConfigMap 當做文件使用

要在一個 Pod 的存儲卷中使用 ConfigMap:

1. 創建一個 ConfigMap 對像或者使用現有的 ConfigMap 對象。多個 Pod 可以引用同一個 ConfigMap。
2. 修改 Pod 定義，在 `spec.volumes[]` 下添加一個卷。為該卷設置任意名稱，之後將 `spec.volumes[].configMap.name` 字段設置為對你的 ConfigMap 對象的引用。
3. 為每個需要該 ConfigMap 的容器添加一個 `.spec.containers[].volumeMounts[]`。設置 `.spec.containers[].volumeMounts[].readOnly=true` 並將 `.spec.containers[].volumeMounts[].mountPath` 設置為一個未使用的目錄名， ConfigMap 的內容將出現在該目錄中。
4. 更改你的鏡像或者命令行，以便程序能夠從該目錄中查找文件。 ConfigMap 中的每個 `data` 鍵會變成 mountPath 下面的一個文件名。

下面是一個將 ConfigMap 以卷的形式進行掛載的 Pod 示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    configMap:
      name: myconfigmap
```

你希望使用的每個 ConfigMap 都需要在 `spec.volumes` 中被引用到。

如果 Pod 中有多個容器，則每個容器都需要自己的 `volumeMounts` 塊，但針對每個 ConfigMap，你只需要設置一個 `spec.volumes` 塊。

### 被掛載的 ConfigMap 內容會被自動更新

當卷中使用的 ConfigMap 被更新時，所投射的鍵最終也會被更新。 kubelet 組件會在每次週期性同步時檢查所掛載的 ConfigMap 是否為最新。不過，kubelet 使用的是其本地的高速緩存來獲得 ConfigMap 的當前值。高速緩存的類型可以通過 KubeletConfiguration 結構. 的 `ConfigMapAndSecretChangeDetectionStrategy` 字段來配置。

ConfigMap 既可以通過 watch 操作實現內容傳播（默認形式），也可實現基於 TTL 的緩存，還可以直接經過所有請求重定向到 API 服務器。因此，從 ConfigMap 被更新的那一刻算起，到新的主鍵被投射到 Pod 中去， 這一時間跨度可能與 kubelet 的同步週期加上高速緩存的傳播延遲相等。這裡的傳播延遲取決於所選的高速緩存類型 （分別對應 watch 操作的傳播延遲、高速緩存的 TTL 時長或者 0）。

以環境變量方式使用的 ConfigMap 數據不會被自動更新。更新這些數據需要重新啟動 Pod。

!!! info
    說明： 使用 ConfigMap 作為 subPath 卷掛載的容器將不會收到 ConfigMap 的更新。

## 不可變更的 ConfigMap

Kubernetes 特性 Immutable Secret 和 ConfigMaps 提供了一種將各個 Secret 和 ConfigMap 設置為不可變更的選項。對於大量使用 ConfigMap 的集群 （至少有數万個各不相同的 ConfigMap 給 Pod 掛載）而言，禁止更改 ConfigMap 的數據有以下好處：

- 保護應用，使之免受意外（不想要的）更新所帶來的負面影響。
- 通過大幅降低對 kube-apiserver 的壓力提升集群性能， 這是因為系統會關閉對已標記為不可變更的 ConfigMap 的監視操作。
  
此功能特性由 `ImmutableEphemeralVolumes` [特性門控](https://kubernetes.io/zh-cn/docs/reference/command-line-tools-reference/feature-gates/)來控制。你可以通過將 `immutable` 字段設置為 `true` 創建不可變更的 ConfigMap。例如：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  ...
data:
  ...
immutable: true
```

一旦某 ConfigMap 被標記為不可變更，則 無法 逆轉這一變化，，也無法更改 `data` 或 `binaryData` 字段的內容。你只能刪除並重建 ConfigMap。因為現有的 Pod 會維護一個已被刪除的 ConfigMap 的掛載點，建議重新創建這些 Pods。

