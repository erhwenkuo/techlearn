# 日誌架構

應用日誌可以讓你了解應用內部的運行狀況。日誌對調試問題和監控集群活動非常有用。大部分現代化應用都有某種日誌記錄機制。同樣地，容器引擎也被設計成支持日誌記錄。針對容器化應用，最簡單且最廣泛採用的日誌記錄方式就是寫入標準輸出和標準錯誤流。

但是，由容器引擎或運行時提供的原生功能通常不足以構成完整的日誌記錄方案。例如，如果發生容器崩潰、Pod 被逐出或節點宕機等情況，你可能想訪問應用日誌。在集群中，日誌應該具有獨立的存儲和生命週期，與節點、Pod 或容器的生命週期相獨立。這個概念叫 **集群級的日誌**。

集群級日誌架構需要一個獨立的後端用來存儲、分析和查詢日誌。 Kubernetes 並不為日誌數據提供原生的存儲解決方案。相反，有很多現成的日誌方案可以集成到 Kubernetes 中。下面各節描述如何在節點上處理和存儲日誌。

## Kubernetes 中的基本日誌記錄

這裡的示例使用包含一個容器的 Pod spec，每秒鐘向標準輸出(stdout)寫入數據。

```yaml title="debug/counter-pod.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox:1.28
    args: [/bin/sh, -c,
            'i=0; while true; do echo "$i: $(date)"; i=$((i+1)); sleep 1; done']
```

用下面的命令運行 Pod：

```bash
kubectl apply -f https://k8s.io/examples/debug/counter-pod.yaml
```

輸出結果為：

```
pod/counter created
```

像下面這樣，使用 `kubectl logs` 命令獲取日誌:

```bash
kubectl logs counter
```

輸出結果為：

```
0: Mon Jan  1 00:00:00 UTC 2001
1: Mon Jan  1 00:00:01 UTC 2001
2: Mon Jan  1 00:00:02 UTC 2001
...
```

你可以使用命令 `kubectl logs --previous` 檢索之前容器實例的日誌。如果 Pod 有多個容器，你應該為該命令附加容器名以訪問對應容器的日誌， 使用 -c 標誌來指定要訪問的容器的日誌，如下所示：

```bash
kubectl logs counter -c count
```

詳見 [kubectl logs 文檔](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#logs)。

## 節點級日誌記錄

![](./assets/logging-node-level.png)

{==容器化應用寫入 `stdout` 和 `stderr` 的任何數據，都會被容器引擎捕獲並被重定向到某個位置。==}

默認情況下，如果容器重啟，kubelet 會保留被終止的容器日誌。如果 Pod 在工作節點被驅逐，該 Pod 中所有的容器也會被驅逐，包括容器日誌。

節點級日誌記錄中，需要重點考慮實現 **日誌的輪轉**，以此來保證日誌不會消耗節點上全部可用空間。 Kubernetes 並不負責輪轉日誌，而是通過部署工具建立一個解決問題的方案。例如，在用 kube-up.sh 部署的 Kubernetes 集群中，存在一個 [`logrotate`](https://linux.die.net/man/8/logrotate)，每小時運行一次。你也可以設置容器運行時來自動地輪轉應用日誌。

當使用某 CRI 容器 runtime 時，**kubelet 要負責對日誌進行輪換，並管理日誌目錄的結構**。 kubelet 將此信息發送給 CRI 容器運行時，後者將容器日誌寫入到指定的位置。在 kubelet 配置文件 中的兩個 kubelet 參數 **containerLogMaxSize** 和 **containerLogMaxFiles** 可以用來配置每個日誌文件的最大長度和每個容器可以生成的日誌文件個數上限。

當運行 `kubectl logs` 時， 節點上的 kubelet 處理該請求並直接讀取日誌文件，同時在響應中返回日誌文件內容。

!!! info
    說明： 如果有外部系統執行日誌輪轉或者使用了 CRI 容器運行時，那麼 `kubectl logs` 僅可查詢到最新的日誌內容。比如，對於一個 10MB 大小的文件，通過 logrotate 執行輪轉後生成兩個文件， 一個 10MB 大小，一個為空，kubectl logs 返回最新的日誌文件，而該日誌文件在這個例子中為空。

### 系統組件日誌

系統組件有兩種類型：在容器中運行的和不在容器中運行的。例如：

- 在容器中運行的 kube-scheduler 和 kube-proxy。
- 不在容器中運行的 kubelet 和容器 runtime。

在使用 systemd 機制的服務器上，kubelet 和容器運行時將日誌寫入到 journald 中。如果沒有 systemd，它們將日誌寫入到 `/var/log` 目錄下的 `.log` 文件中。容器中的系統組件通常將日誌寫到 `/var/log` 目錄，繞過了默認的日誌機制。他們使用 [klog](https://github.com/kubernetes/klog) 日誌庫。你可以在[日誌開發文檔](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-instrumentation/logging.md) 找到這些組件的日誌告警級別約定。

和容器日誌類似，`/var/log` 目錄中的系統組件日誌也應該被輪轉。通過腳本 `kube-up.sh` 啟動的 Kubernetes 集群中，日誌被工具 `logrotate` 執行每日輪轉，或者日誌大小超過 100MB 時觸發輪轉。

## 集群級日誌架構

雖然 Kubernetes 沒有為集群級日誌記錄提供原生的解決方案，但你可以考慮幾種常見的方法。以下是一些選項：

- 使用在每個節點上運行的節點級日誌記錄代理。
- 在應用程序的 Pod 中，包含專門記錄日誌的邊車（Sidecar）容器。
- 將日誌直接從應用程序中推送到日誌記錄後端。

### 使用節點級日誌代理

![](./assets/logging-with-node-agent.png)

你可以通過在每個節點上使用 **節點級的日誌記錄代理** 來實現集群級日誌記錄。`日誌記錄代理`是一種用於暴露日誌或將日誌推送到後端的專用工具。通常`日誌記錄代理`程序是一個容器，它可以訪問包含該節點上所有應用程序容器的日誌文件的目錄。

由於`日誌記錄代理`必須在每個節點上運行，通常可以用 DaemonSet 的形式運行該代理。節點級日誌在每個節點上僅創建一個代理，不需要對節點上的應用做修改。

容器向標準輸出(stdout)和標準錯誤輸出(stderr)寫出數據，但在格式上並不統一。`節點級代理`收集這些日誌並將其進行轉發以完成匯總。

### 使用 sidecar 容器運行日誌代理

你可以通過以下方式之一使用邊車（Sidecar）容器：

- 邊車容器將應用程序日誌傳送到自己的標準輸出。
- 邊車容器運行一個日誌代理，配置該日誌代理以便從應用容器收集日誌。

![](./assets/logging-with-streaming-sidecar.png)

利用邊車容器向自己的 stdout 和 stderr 傳輸流的方式， 你就可以利用每個節點上的 kubelet 和日誌代理來處理日誌。邊車容器從文件、套接字或 journald 讀取日誌。每個邊車容器向自己的 stdout 和 stderr 流中輸出日誌。

這種方法允許你將日誌流從應用程序的不同部分分離開，其中一些可能缺乏對寫入 stdout 或 stderr 的支持。重定向日誌背後的邏輯是最小的，因此它的開銷幾乎可以忽略不計。另外，因為 stdout 和 stderr 由 kubelet 處理，所以你可以使用內置的工具 kubectl logs。

例如，某 Pod 中運行一個容器，該容器向兩個文件寫不同格式的日誌。下面是這個 Pod 的配置文件:

```yaml title="admin/logging/two-files-counter-pod.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox:1.28
    args:
    - /bin/sh
    - -c
    - >
      i=0;
      while true;
      do
        echo "$i: $(date)" >> /var/log/1.log;
        echo "$(date) INFO $i" >> /var/log/2.log;
        i=$((i+1));
        sleep 1;
      done      
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  volumes:
  - name: varlog
    emptyDir: {}
```

不建議在同一個日誌流中寫入不同格式的日誌條目，即使你成功地將其重定向到容器的 stdout 流。相反，你可以創建兩個邊車容器。每個邊車容器可以從共享卷跟踪特定的日誌文件， 並將文件內容重定向到各自的 stdout 流。

下面是運行兩個邊車容器的 Pod 的配置文件：

```yaml title="admin/logging/two-files-counter-pod-streaming-sidecar.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox:1.28
    args:
    - /bin/sh
    - -c
    - >
      i=0;
      while true;
      do
        echo "$i: $(date)" >> /var/log/1.log;
        echo "$(date) INFO $i" >> /var/log/2.log;
        i=$((i+1));
        sleep 1;
      done      
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-log-1
    image: busybox:1.28
    args: [/bin/sh, -c, 'tail -n+1 -F /var/log/1.log']
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-log-2
    image: busybox:1.28
    args: [/bin/sh, -c, 'tail -n+1 -F /var/log/2.log']
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  volumes:
  - name: varlog
    emptyDir: {}
```

現在當你運行這個 Pod 時，你可以運行如下命令分別訪問每個日誌流：

```bash
kubectl logs counter count-log-1
```

輸出為：

```
0: Mon Jan  1 00:00:00 UTC 2001
1: Mon Jan  1 00:00:01 UTC 2001
2: Mon Jan  1 00:00:02 UTC 2001
...
```

```bash
kubectl logs counter count-log-2
```

輸出為：

```
Mon Jan  1 00:00:00 UTC 2001 INFO 0
Mon Jan  1 00:00:01 UTC 2001 INFO 1
Mon Jan  1 00:00:02 UTC 2001 INFO 2
...
```

集群中安裝的節點級代理會自動獲取這些日誌流，而無需進一步配置。如果你願意，你也可以配置代理程序來解析源容器的日誌行。

注意，儘管 CPU 和內存使用率都很低（以多個 CPU 毫核指標排序或者按內存的兆字節排序）， 向文件寫日誌然後輸出到 stdout 流仍然會成倍地增加磁盤使用率。如果你的應用向單一文件寫日誌，通常最好設置 /dev/stdout 作為目標路徑， 而不是使用流式的邊車容器方式。

應用本身如果不具備輪轉日誌文件的功能，可以通過邊車容器實現。該方式的一個例子是運行一個小的、定期輪轉日誌的容器。然而，還是推薦直接使用 stdout 和 stderr，將日誌的輪轉和保留策略交給 kubelet。

### 具有日誌代理功能的邊車容器

![](./assets/logging-with-sidecar-agent.png)

如果節點級日誌記錄代理程序對於你的場景來說不夠靈活， 你可以創建一個帶有單獨日誌記錄代理的邊車容器，將代理程序專門配置為與你的應用程序一起運行。

!!! info
    說明：
      在邊車容器中使用日誌代理會帶來嚴重的資源損耗。此外，你不能使用 kubectl logs 命令訪問日誌，因為日誌並沒有被 kubelet 管理。

下面是兩個配置文件，可以用來實現一個帶日誌代理的邊車容器。第一個文件包含用來配置 fluentd 的 ConfigMap。

```yaml title="admin/logging/fluentd-sidecar-config.yaml"
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
data:
  fluentd.conf: |
    <source>
      type tail
      format none
      path /var/log/1.log
      pos_file /var/log/1.log.pos
      tag count.format1
    </source>

    <source>
      type tail
      format none
      path /var/log/2.log
      pos_file /var/log/2.log.pos
      tag count.format2
    </source>

    <match **>
      type google_cloud
    </match>    
```

!!! info
    說明：
      要進一步了解如何配置 fluentd，請參考 [fluentd 官方文檔](https://docs.fluentd.org/)。

第二個文件描述了運行 fluentd 邊車容器的 Pod 。 flutend 通過 Pod 的掛載卷獲取它的配置數據。

```yaml title="admin/logging/two-files-counter-pod-agent-sidecar.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox:1.28
    args:
    - /bin/sh
    - -c
    - >
      i=0;
      while true;
      do
        echo "$i: $(date)" >> /var/log/1.log;
        echo "$(date) INFO $i" >> /var/log/2.log;
        i=$((i+1));
        sleep 1;
      done      
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-agent
    image: k8s.gcr.io/fluentd-gcp:1.30
    env:
    - name: FLUENTD_ARGS
      value: -c /etc/fluentd-config/fluentd.conf
    volumeMounts:
    - name: varlog
      mountPath: /var/log
    - name: config-volume
      mountPath: /etc/fluentd-config
  volumes:
  - name: varlog
    emptyDir: {}
  - name: config-volume
    configMap:
      name: fluentd-config
```

在示例配置中，你可以將 fluentd 替換為任何日誌代理，從應用容器內的任何來源讀取數據。

### 從應用中直接暴露日誌目錄

![](./assets/logging-from-application.png)

從各個應用中直接暴露和推送日誌數據的集群日誌機制已超出 Kubernetes 的範圍。
