# 標準部署

該文件主要介紹了部署 Doris 所需軟硬體環境、建議的部署方式、叢集擴容縮容，以及叢集搭建到運行過程中的常見問題。
在閱讀本文檔前，請先根據編譯文檔編譯 Doris。

## 軟硬體需求

### 概述

Doris 作為一個開源的 MPP 架構 OLAP 資料庫，能夠運作在絕大多數主流的商用伺服器上。為了能夠充分運用 MPP 架構的並發優勢，以及 Doris 的高可用特性，我們建議 Doris 的部署遵循以下需求：

#### Linux 作業系統版本需求

| Linux 系統 | 版本 |
|---|---|
| CentOS | 7.1 以上 |
| Ubuntu | 16.04 以上 |

#### 軟體需求

| 軟體 | 版本 |
|---|---|
| Java | 1.8 |
| GCC | 4.8.2 以上 |

#### 作業系統安裝需求

##### 設定系統最大開啟檔案句柄數

```
vi /etc/security/limits.conf
* soft nofile 65536
* hard nofile 65536
```

##### 時鐘同步

Doris 的元資料要求時間精度要小於 5000ms，所以所有群集所有機器要進行時脈同步，避免因為時脈問題引發的元資料不一致而導致服務出現異常。

##### 關閉交換分割區（swap）

Linux 交換分割區會為 Doris 帶來嚴重的效能問題，需要在安裝前停用交換分割區

##### Linux檔案系統

`ext4` 和 `xfs` 檔案系統皆支援。

#### 開發測試環境

| 模組 | CPU | 記憶體 | 磁碟 | 網路 | 實例數量 |
|---|---|---|---|---|---|
| Frontend | 8核心+ | 8GB+ | SSD 或 SATA，10GB+ * | 千兆網路卡 | 1 |
| Backend | 8核心+ | 16GB+ | SSD 或 SATA，50GB+ * | 千兆網路卡 | 1-3 * |

#### 生產環境

| 模組 | CPU | 記憶體 | 磁碟 | 網路 | 實例數量（最低需求）​​ |
|---|---|---|---|---|------------|
| Frontend | 16核心+ | 64GB+ | SSD 或 RAID 卡，100GB+ * | 萬兆網路卡 | 1-3 * |
| Backend | 16核心+ | 64GB+ | SSD 或 SATA，100G+ * | 萬兆網路卡 | 3 * |

!!! info
    註1:

    - FE 的磁碟空間主要用於儲存元數據，包括日誌和 image。通常從幾百 MB 到幾個 GB 不等。
    - BE 的磁碟空間主要用於存放用戶數據，總磁碟空間按用戶總數據量 * 3（3副本）計算，然後再預留額外 40% 的空間用作後台 compaction 以及一些中間數據的存放。
    - 一台機器上雖然可以部署多個 BE，**但只建議部署一個實例**，同時 **只能部署一個 FE**。如果需要 3 副本數據，那麼至少需要 3 台機器各部署一個 BE 實例（而不是 1 台機器部署 3 個 BE 實例）。 **多個 FE 所在伺服器的時鐘必須保持一致（允許最多 5 秒的時鐘偏差）**
    - 測試環境也可以只適用一個 BE 來測試。實際生產環境，BE 實例數量直接決定了整體查詢延遲。
    - 所有部署節點關閉 Swap。

!!! info
    註2：FE 節點的數量

    - FE 角色分為 Follower 和 Observer，（Leader 為 Follower 組中選出的一種角色，以下統稱 Follower）。
    - FE 節點資料至少為1（1 個 Follower）。當部署 1 個 Follower 和 1 個 Observer 時，可以實現讀高可用。當部署 3 個 Follower 時，可以實現讀寫高可用（HA）。
    - Follower 的數量 **必須** 為奇數，Observer 數量隨意。
    - 根據以往經驗，當叢集可用性要求很高時（例如提供線上業務），可以部署 3 個 Follower 和 1-3 個 Observer。如果是離線業務，建議部署 1 個 Follower 和 1-3 個 Observer。

* 通常我們建議 10 ~ 100 台左右的機器，來充分發揮 Doris 的性能（其中 3 台部署 FE（HA），剩餘的部署 BE）
* 當然，Doris 的效能與節點數量及配置呈正相關。在最少 4 台機器（一台 FE，三台 BE，其中一台 BE 混部一個 Observer FE 提供元資料備份），以及較低配置的情況下，依然可以平穩的運行 Doris。
* 如果 FE 和 BE 混部，需注意資源競爭問題，並確保元資料目錄和資料目錄分屬不同磁碟。

#### Broker 部署

Broker 是用於存取外部資料來源（如 hdfs）的進程。通常，在每台機器上部署一個 broker 實例即可。

#### 網路需求

Doris 各個實例直接透過網路進行通訊。以下表格展示了所有需要的端口

| 實例名稱 | 連接埠名稱 | 預設連接埠 | 通訊方向 | 說明 |
|---|---|---|---| ---|
| BE | be_port | 9060 | FE --> BE | BE 上 thrift server 的端口，用於接收來自 FE 的請求 |
| BE | webserver_port | 8040 | BE <--> BE | BE 上的 http server 的連接埠 |
| BE | heartbeat\_service_port | 9050 | FE --> BE | BE 上心跳服務端口（thrift），用於接收來自 FE 的心跳 |
| BE | brpc\_port | 8060 | FE <--> BE, BE <--> BE | BE 上的 brpc 端口，用於 BE 之間通訊 |
| FE | http_port | 8030 | FE <--> FE，使用者 <--> FE |FE 上的 http server 連接埠 |
| FE | rpc_port | 9020 | BE --> FE, FE <--> FE | FE 上的 thrift server 端口，每個fe的配置需要保持一致|
| FE | query_port | 9030 | 使用者 <--> FE | FE 上的 mysql server 埠 |
| FE | arrow_flight_sql_port | 9040 | 使用者 <--> FE | FE 上的 Arrow Flight SQL server 連接埠 |
| FE | edit\_log_port | 9010 | FE <--> FE | FE 上的 bdbje 之間通訊用的埠 |
| Broker | broker\_ipc_port | 8000 | FE --> Broker, BE --> Broker | Broker 上的 thrift server，用於接收請求 |

> 註：
> 1. 當部署多個 FE 實例時，請確保 FE 的 http\_port 配置相同。
> 2. 部署前請確保各個連接埠在應有方向上的存取權限。

#### IP 綁定

因為有多網卡的存在，或因為安裝過 docker 等環境而導致的虛擬網卡的存在，同一個主機可能存在多個不同的 ip。目前 Doris 並不能自動辨識可用 IP。所以當遇到部署主機上有多個 IP 時，必須透過 priority\_networks 設定項目來強制指定正確的 IP。

priority\_networks 是 FE 和 BE 都有的一個配置，配置項目需寫在 `fe.conf` 和 `be.conf` 中。此組態項目用於在 FE 或 BE 啟動時，告訴程序應該綁定哪個IP。範例如下：

`priority_networks=10.1.3.0/24`

這是一種 [CIDR](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing) 的表示方法。 FE 或 BE 會根據這個設定項來尋找符合的 IP，作為自己的 localIP。

**注意**：當配置完 priority\_networks 並啟動 FE 或 BE 後，只是保證了 FE 或 BE 自身的 IP 進行了正確的綁定。而在使用 ADD BACKEND 或 ADD FRONTEND 語句中，也需要指定和 priority\_networks 配置相符的 IP，否則叢集無法建立。舉例：

BE 的設定為：`priority_networks=10.1.3.0/24`

但在 ADD BACKEND 時使用的是：`ALTER SYSTEM ADD BACKEND "192.168.0.1:9050";`

則 FE 和 BE 將無法正常通訊。

這時，必須 DROP 掉這個新增錯誤的 BE，重新使用正確的 IP 執行 ADD BACKEND。

FE 同理。

BROKER 目前沒有，也不需要 `priority\_networks` 這個選項。 Broker 的服務預設綁定在 `0.0.0.0` 上。只需在 ADD BROKER 時，執行正確可存取的 BROKER IP 即可。

#### 表名大小寫敏感性設置

doris 預設為表名大小寫敏感，如有表名大小寫不敏感的需求需在叢集初始化時進行設定。{==表名大小寫敏感度在叢集初始化完成後不可再修改==}。

詳細參考 [變數](../advanced/variables.md##支援的變數) 中關於 `lower_case_table_names` 變數的介紹。

## 叢集部署

### 手動部署

#### FE 部署

- 拷貝 FE 部署檔案到指定節點

    將原始碼編譯產生的 output 下的 fe 資料夾拷貝到 FE 的節點指定部署路徑下並進入該目錄。

- 配置 FE

    1. 設定檔為 `conf/fe.conf`。其中注意：`meta_dir`　是元資料存放位置。預設值為 `${DORIS_HOME}/doris-meta`。需 **手動建立** 該目錄。

        **注意：生產環境強烈建議單獨指定目錄不要放在Doris安裝目錄下，最好是單獨的磁碟（如果有SSD最好），測試開發環境可以使用預設配置**

    2. `fe.conf` 中 `JAVA_OPTS` 預設 java 最大堆記憶體為 8GB。

- 啟動FE

    `bin/start_fe.sh --daemon`

    FE  進程啟動進入後台執行。日誌預設存放在 `log/` 目錄下。如啟動失敗，可以透過檢視 `log/fe.log` 或 `log/fe.out` 查看錯誤訊息。

- 如需部署多 FE，請參閱 "FE 擴容與縮容" 章節

#### BE 部署

* 拷貝 BE 部署檔案到所有要部署 BE 的節點

    將原始碼編譯產生的 output 下的 be 資料夾拷貝到 BE 的節點的指定部署路徑下。

    > 注意：`output/be/lib/debug_info/` 目錄下為調試資訊文件，文件較大，但實際運行不需要這些文件，可以不部署。

* 修改所有 BE 的配置

    修改 be/conf/be.conf。主要是設定 `storage_root_path`：資料存放目錄。預設在be/storage下，若需要指定目錄的話，需要**預先建立目錄**。多個路徑之間使用英文狀態的分號 `;` 分隔。
    可以透過路徑區別節點內的冷熱資料儲存目錄，HDD（冷資料目錄）或 SSD（熱資料目錄）。如果不需要 BE 節點內的冷熱機制，那麼只需要配置路徑即可，無需指定 medium 類型；也不需要修改FE的預設儲存媒體配置

    **注意：**
    1. 如果未指定儲存路徑的儲存類型，則預設全部為 HDD（冷資料目錄）。
    2. 這裡的 HDD 和 SSD 與實體儲存媒體無關，只為了區分儲存路徑的儲存類型，也就是可以在 HDD 媒體的磁碟上標記某個目錄為 SSD（熱資料目錄）。

    範例 1 如下：

    `storage_root_path=/home/disk1/doris;/home/disk2/doris;/home/disk2/doris`

    範例 2 如下：

    **使用 storage_root_path 參數裡指定 medium**

    `storage_root_path=/home/disk1/doris,medium:HDD;/home/disk2/doris,medium:SSD`

    **說明**

      - /home/disk1/doris,medium:HDD： 表示該目錄儲存冷資料;
      - /home/disk2/doris,medium:SSD： 表示該目錄儲存熱資料;

* BE webserver_port埠配置

    如果 be 部署在 hadoop 叢集中，注意調整 be.conf 中的 `webserver_port = 8040` ,以免造成連接埠衝突

* 配置 JAVA_HOME 環境變量
 
    由於從 1.2 版本開始支援 Java UDF 函數，BE 依賴 Java 環境。所以要預先配置 `JAVA_HOME` 環境變量，也可以在 `start_be.sh` 啟動腳本第一行加入 `export JAVA_HOME=your_java_home_path` 來新增環境變數。

* 安裝 Java UDF 函數

    因為從 1.2 版本開始支援 Java UDF 函數，需要從官網下載 Java UDF 函數的 JAR 套件放到 BE 的 lib 目錄下，否則可能會啟動失敗。

* 在 FE 中新增所有 BE 節點

    BE 節點需要先在 FE 中加入，才可加入叢集。可以使用 mysql-client([下載MySQL 5.7](https://dev.mysql.com/downloads/mysql/5.7.html)) 連接到 FE：

    `./mysql-client -h fe_host -P query_port -uroot`

    其中 fe_host 為 FE 所在節點 ip；query_port 在 `fe/conf/fe.conf` 中的；預設使用 root 帳戶，無密碼登入。

    登入後，執行以下命令來新增每一個 BE：

    `ALTER SYSTEM ADD BACKEND "be_host:heartbeat-service_port";`

    其中 be_host 為 BE 所在節點 ip；heartbeat_service_port 在 `be/conf/be.conf` 中。

* 啟動 BE

    `bin/start_be.sh --daemon`

    BE 進程將啟動並進入背景執行。日誌預設存放在 be/log/ 目錄下。如啟動失敗，可以透過查看 be/log/be.log 或 be/log/be.out 查看錯誤訊息。

* 查看BE狀態

    使用 mysql-client 連接到 FE，並執行 `SHOW PROC '/backends';` 查看 BE 運行情況。如一切正常，`isAlive` 列應為 `true`。

####（可選）FS_Broker 部署

Broker 以插件的形式，獨立於 Doris 部署。如果需要從第三方儲存系統匯入數據，則需要部署相應的 Broker，預設提供了讀取 HDFS 、物件儲存的 fs_broker。 fs_broker 是無狀態的，建議每一個 FE 和 BE 節點部署一個 Broker。

* 拷貝原始碼 fs_broker 的 output 目錄下的對應 Broker 目錄到需要部署的所有節點上。建議和 BE 或 FE 目錄保持同級。

* 修改對應 Broker 配置

    在對應 broker/conf/ 目錄下對應的設定檔中，可以修改對應設定。

* 啟動 Broker

    `bin/start_broker.sh --daemon`

* 新增 Broker

    要讓 Doris 的 FE 和 BE 知道 Broker 在哪些節點上，透過 sql 指令新增 Broker 節點清單。

    使用 mysql-client 連線啟動的 FE，執行以下指令：

    `ALTER SYSTEM ADD BROKER broker_name "broker_host1:broker_ipc_port1","broker_host2:broker_ipc_port2",...;`

    其中 broker_host 為 Broker 所在節點 ip；broker_ipc_port 在 Broker 設定檔中的conf/apache_hdfs_broker.conf。

* 查看 Broker 狀態

    使用 mysql-client 連線任一已啟動的 FE，執行下列指令查看 Broker 狀態：`SHOW PROC "/brokers";`

#### FE 和 BE 的啟動方式

##### 版本 >=2.0.2

1. 使用 `start_xx.sh` 啟動，該方式會將日誌輸出至文件，同時不會退出啟動腳本進程，通常使用 supervisor 等工具自動拉起時建議採用這種方式。
2. 使用 `start_xx.sh --daemon` 啟動，FE/BE 將作為一個後台程序在後台運行，並且日誌輸出將預設寫入到指定的日誌檔案中。這種啟動方式適用於生產環境。
3. 使用 `start_xx.sh --console` 啟動，該參數用於以控制台模式啟動 FE/BE 。當使用 --console 參數啟動時，伺服器將在目前終端會話中啟動，並將日誌輸出和控制台互動列印到該終端。這種啟動方式適用於開發和測試場景.

##### 版本 < 2.0.2

1. 使用 `start_xx.sh --daemon` 啟動，FE/BE 將作為一個後台程序在後台運行，並且日誌輸出將預設寫入到指定的日誌檔案中。這種啟動方式適用於生產環境。
2. 使用 `start_xx.sh` 啟動，該參數用於以控制台模式啟動 FE/BE 。當使用 --console 參數啟動時，伺服器將在目前終端會話中啟動，並將日誌輸出和控制台互動列印到該終端。
   這種啟動方式適用於開發和測試場景.

**註：在生產環境中，所有實例都應使用守護程序啟動，以確保程序退出後，會自動拉起，如 [Supervisor](http://supervisord.org/)。如需使用守護程式啟動，在 0.9.0 及之前版本中，需要修改各個 start_xx.sh 腳本，去掉最後的 & 符號**。從 0.10.0 版本開始，直接呼叫 `sh start_xx.sh` 啟動即可。也可參考 [這裡](https://www.cnblogs.com/lenmom/p/9973401.html)

## 常見問題

### 進程相關

1. 如何確定 FE 進程啟動成功

    FE 進程啟動後，會先載入元數據，根據 FE 角色的不同，在日誌中會看到 ```transfer from UNKNOWN to MASTER/FOLLOWER/OBSERVER```。最終會看到 ```thrift server started``` 日誌，並且可以透過 mysql 用戶端連線到 FE，則​​表示 FE 啟動成功。

    也可以透過以下連線查看是否啟動成功：
    `http://fe_host:fe_http_port/api/bootstrap`

    如果返回：
    `{"status":"OK","msg":"Success"}`

    則表示啟動成功，其餘情況，則可能有問題。

    > 註：如果在 fe.log 中查看不到啟動失敗的訊息，也許在 fe.out 中可以看到。

2. 如何確定 BE 進程啟動成功

    BE 進程啟動後，如果之前有數據，可能有數分鐘不等的數據索引載入時間。

    如果是 BE 的第一次啟動，或者該 BE 尚未加入任何集群，則 BE 日誌會定期滾動 ```waiting to receive first heartbeat from frontend``` 字樣。表示 BE 還未透過 FE 的心跳收到 Master 的地址，正在被動等待。這種錯誤日誌，在 FE 中 ADD BACKEND 並發送心跳後，就會消失。如果接到心跳後，又重複出現``````master client, get client from cache failed.host: , port: 0, code: 7`````` 字樣，表示FE 成功連接了BE，但BE 無法主動連接FE。可能需要檢查 BE 到 FE 的 rpc_port 的連通性。

    如果BE 已經加入集群，日誌中應該每隔5 秒滾動來自FE 的心跳日誌：```get heartbeat, host: xx.xx.xx.xx, port: 9020, cluster id: xxxxxx```，表示心跳正常。

    其次，日誌中應該每隔 10 秒滾動 ```finish report task success. return code: 0``` 的字樣，表示 BE 向 FE 的通訊正常。

    同時，如果有資料查詢，應該可以看到不停滾動的日誌，並且有 ```execute time is xxx``` 日誌，表示 BE 啟動成功，並且查詢正常。

    也可以透過以下連線查看是否啟動成功：
    `http://be_host:webserver_port/api/health`

    如果返回：
    `{"status": "OK","msg": "To Be Added"}`

    則表示啟動成功，其餘情況，則可能有問題。

    > 注意：如果在 be.INFO 中查看不到啟動失敗的訊息，也許在 be.out 中可以看到。

3. 搭建系統後，如何確定 FE、BE 連通性正常

    首先確認 FE 和 BE 程序都已經單獨正常啟動，並確認已經透過 `ADD BACKEND` 或 `ADD FOLLOWER/OBSERVER` 語句新增了所有節點。

    如果心跳正常，BE 的日誌中會顯示 ```get heartbeat, host: xx.xx.xx.xx, port: 9020, cluster id: xxxxxx```。如果心跳失敗，在FE 的日誌中會出現```backend[10001] got Exception: org.apache.thrift.transport.TTransportException``` 類似的字樣，或是其他thrift 通訊異常日誌，表示FE 向10001 這個BE的心跳失敗。這裡需要檢查 FE 向 BE host 的心跳埠的連通性。

    如果 BE 向 FE 的通訊正常，則 BE 日誌中會顯示 ```finish report task success. return code: 0``` 的字樣。否則會出現 ```master client, get client from cache failed``` 的字樣。在這種情況下，需要檢查 BE 向 FE 的 rpc_port 的連通性。

4. Doris 各節點認證機制

    除了 Master FE 以外，其餘角色節點（Follower FE，Observer FE，Backend），都需要透過 `ALTER SYSTEM ADD` 語句先註冊到集群，然後才能加入集群。

    Master FE 在第一次啟動時，會在 doris-meta/image/VERSION 檔案中產生一個 cluster_id。

    FE 在第一次加入叢集時，會先從 Master FE 取得這個檔案。之後每次 FE 之間的重新連接（FE 重啟），都會校驗自身 cluster id 是否與已存在的其它 FE 的 cluster id 相同。如果不同，則該 FE 會自動退出。

    BE 在第一次接收到 Master FE 的心跳時，會從心跳中取得到 cluster id，並記錄到資料目錄的 `cluster_id` 檔案中。之後的每次心跳都會比對 FE 發來的 cluster id。如果 cluster id 不相等，則 BE 會拒絕回應 FE 的心跳。

    心跳中同時會包含 Master FE 的 ip。當 FE 切主時，新的 Master FE 會攜帶自身的 ip 傳送心跳給 BE，BE 會更新自身保存的 Master FE 的 ip。

    > **priority\_network**
    >
    > priority\_network 是 FE 和 BE 都有一個配置，其主要目的是在多網卡的情況下，協助 FE 或 BE 識別自身 ip 位址。 priority\_network 採用 CIDR 表示法：[RFC 4632](https://tools.ietf.org/html/rfc4632)
    >
    > 當確認 FE 和 BE 連通性正常後，如果仍然出現建表 Timeout 的情況，並且 FE 的日誌中有 `backend does not found. host: xxx.xxx.xxx.xxx` 字樣的錯誤訊息。則表示 Doris 自動辨識的 IP 位址有問題，需要手動設定 priority\_network 參數。
    >
    > 出現這個問題的主要原因是：當使用者透過 `ADD BACKEND` 語句新增 BE 後，FE 會辨識該語句中指定的是 hostname 還是 IP。如果是 hostname，則 FE 會自動將其轉換為 IP 位址並儲存到元資料中。當 BE 在報告任務完成資訊時，會攜帶自己的 IP 位址。而如果 FE 發現 BE 報告的 IP 位址和元資料中不一致時，就會出現如上錯誤。
    >
    > 這個錯誤的解決方法：1）分別在 FE 和 BE 設定 **priority\_network** 參數。通常 FE 和 BE 都處於一個網段，所以該參數設定為相同即可。 2）在 `ADD BACKEND` 語句中直接填寫 BE 正確的 IP 位址而不是 hostname，以避免 FE 取得到錯誤的 IP 位址。

5. BE 進程文件句柄數

    BE 進程檔案句柄數，受 `min_file_descriptor_number`/`max_file_descriptor_number` 兩個參數控制。

    如果不在 [`min_file_descriptor_number`, `max_file_descriptor_number`] 區間內，BE 進程啟動會出錯，可以使用 `ulimit` 進行設定。

    `min_file_descriptor_number` 的預設值為 65536。

    `max_file_descriptor_number` 的預設值為 131072.

    舉例而言：`ulimit -n 65536`; 表示將檔案句柄設為 65536。

    啟動 BE 程序之後，可以透過 `cat /proc/$pid/limits` 查看程序實際生效的句柄數

    如果使用了 supervisord，遇到句柄數錯誤，可以透過修改 supervisord 的 `minfds` 參數來解決。

    ```shell
    vim /etc/supervisord.conf

    minfds=65535 ; (min. avail startup file descriptors;default 1024)
    ```

