# 數據分割

本文檔主要介紹 Doris 的建表和資料劃分，以及建表作業中可能遇到的問題和解決方法。

## 基本概念

在 Doris 中，資料都以表格（Table）的形式進行邏輯上的描述。

### Row & Column

一張表格包括行（Row）和列（Column）：

- Row：即使用者的一行資料；

- Column： 用於描述一行資料中不同的欄位。

  Column 可以分為兩大類：`Key` 和 `Value`。從業務角度來看，`Key` 和 `Value` 可以分別對應維度列和指標列。 Doris 的 `key` 列是建表語句中指定的列，建表語句中的關鍵字 `unique key` 或 `aggregate key` 或 `duplicate key` 後面的列就是 `key` 列，除了 `key` 列剩下的就是 `value` 列。從 Aggregate 模型的角度來說，`Key` 列相同的行，會聚集合成一行。其中 `Value` 欄位的聚合方式由使用者在建表時指定。更多 Aggregate 模型的介紹，可以參考 [Doris 資料模型](data-model.md)。

### Tablet & Partition

在 Doris 的儲存引擎中，使用者資料被水平劃分為若干個資料分片（Tablet，也稱為資料分桶）。每個 Tablet 包含若干資料行。各個 Tablet 之間的資料沒有交集，並且在實體上是獨立儲存的。

多個 Tablet 在邏輯上歸屬於不同的分割區（Partition）。一個 Tablet 只屬於一個 Partition。而一個 Partition 包含若干個 Tablet。因為 Tablet 在實體上是獨立儲存的，所以可以視為 Partition 在物理上也是獨立。 Tablet 是資料移動、複製等作業的最小實體儲存單元。

若干個 Partition 組成一個 Table。 Partition 可以視為邏輯上最小的管理單元。資料的匯入與刪除，僅能針對一個 Partition 進行。

## 數據劃分

我們以一個建表運算來說明 Doris 的資料劃分。

Doris 的建表是同步指令，SQL執行完成即回傳結果，指令回傳成功即表示建表成功。具體建表語法可以參考[CREATE TABLE](../sql-manual/sql-reference/Data-Definition-Statements/Create/CREATE-TABLE.md)，也可以透過`HELP CREATE TABLE;` 查看更多幫助。

本小節透過一個例子，來介紹 Doris 的建表方式。

```sql
-- Range Partition

CREATE TABLE IF NOT EXISTS example_db.example_range_tbl
(
    `user_id` LARGEINT NOT NULL COMMENT "使用者id",
    `date` DATE NOT NULL COMMENT "資料灌入日期時間",
    `timestamp` DATETIME NOT NULL COMMENT "資料灌入的時間戳記",
    `city` VARCHAR(20) COMMENT "使用者所在城市",
    `age` SMALLINT COMMENT "用戶年齡",
    `sex` TINYINT COMMENT "用戶性別",
    `last_visit_date` DATETIME REPLACE DEFAULT "1970-01-01 00:00:00" COMMENT "使用者最後一次造訪時間",
    `cost` BIGINT SUM DEFAULT "0" COMMENT "用戶總消費",
    `max_dwell_time` INT MAX DEFAULT "0" COMMENT "使用者最大停留時間",
    `min_dwell_time` INT MIN DEFAULT "99999" COMMENT "使用者最小停留時間"
)
ENGINE=OLAP
AGGREGATE KEY(`user_id`, `date`, `timestamp`, `city`, `age`, `sex`)
PARTITION BY RANGE(`date`)
(
    PARTITION `p201701` VALUES LESS THAN ("2017-02-01"),
    PARTITION `p201702` VALUES LESS THAN ("2017-03-01"),
    PARTITION `p201703` VALUES LESS THAN ("2017-04-01")
)
DISTRIBUTED BY HASH(`user_id`) BUCKETS 16
PROPERTIES
(
    "replication_num" = "3",
    "storage_medium" = "SSD",
    "storage_cooldown_time" = "2018-01-01 12:00:00"
);

-- List Partition

CREATE TABLE IF NOT EXISTS example_db.example_list_tbl
(
    `user_id` LARGEINT NOT NULL COMMENT "使用者id",
    `date` DATE NOT NULL COMMENT "資料灌入日期時間",
    `timestamp` DATETIME NOT NULL COMMENT "資料灌入的時間戳記",
    `city` VARCHAR(20) NOT NULL COMMENT "使用者所在城市",
    `age` SMALLINT COMMENT "用戶年齡",
    `sex` TINYINT COMMENT "用戶性別",
    `last_visit_date` DATETIME REPLACE DEFAULT "1970-01-01 00:00:00" COMMENT "使用者最後一次造訪時間",
    `cost` BIGINT SUM DEFAULT "0" COMMENT "用戶總消費",
    `max_dwell_time` INT MAX DEFAULT "0" COMMENT "使用者最大停留時間",
    `min_dwell_time` INT MIN DEFAULT "99999" COMMENT "使用者最小停留時間"
)
ENGINE=olap
AGGREGATE KEY(`user_id`, `date`, `timestamp`, `city`, `age`, `sex`)
PARTITION BY LIST(`city`)
(
    PARTITION `p_cn` VALUES IN ("Beijing", "Shanghai", "Hong Kong"),
    PARTITION `p_usa` VALUES IN ("New York", "San Francisco"),
    PARTITION `p_jp` VALUES IN ("Tokyo")
)
DISTRIBUTED BY HASH(`user_id`) BUCKETS 16
PROPERTIES
(
    "replication_num" = "3",
    "storage_medium" = "SSD",
    "storage_cooldown_time" = "2018-01-01 12:00:00"
);

```

### 列定義

這裡我們只以 `AGGREGATE KEY` 資料模型為例來說明。更多資料模型請參閱 [Doris 資料模型](./data-model.md)。

列的基本類型，可以透過在 mysql-client 中執行 `HELP CREATE TABLE;` 檢視。

AGGREGATE KEY 資料模型中，所有沒有指定聚合方式（SUM、REPLACE、MAX、MIN）的欄位視為 `Key` 列。而其餘則為 `Value` 列。

定義列時，可參考以下建議：

1. Key 欄位必須在所有 Value 欄位之前。
2. 盡量選擇整數類型。因為整數類型的計算和查找效率遠高於字串。
3. 對於不同長度的整型類型的選擇原則，遵循 **夠用即可**。
4. 對於 VARCHAR 和 STRING 類型的長度，遵循 **夠用即可**。

### 分區和分桶

Doris 支援兩層的資料劃分。第一層是 `Partition`，支援 `Range` 和 `List` 的分割方式。第二層是 `Bucket（Tablet）`，支援 `Hash` 和 `Random` 的分割方式。

也可以只使用一層分區，建表時如果不寫分區的語句即可，此時 Doris 會產生一個預設的分區，對使用者是透明的。使用一層分割區時，只支援 `Bucket` 劃分。下面我們來分別介紹下分區以及分桶：

#### 分區 (Partition)

- Partition 列可以指定一列或多列，分區列必須為 KEY 列。多列分區的使用方式在後面 **多列分區** 小結介紹。
- 不論分區列是什麼類型，在寫分區值時，都需要加雙引號。
- 分區數量理論上沒有上限。
- 當不使用 Partition 建置表時，系統會自動產生一個和表名同名的，全值範圍的 Partition。該 Partition 對使用者不可見，且不可刪改。
- 建立分割區時　**不可新增範圍重疊**　的分割區。

---

**1. Range 分區:**

- Range 分區列通常為 **時間列**，以方便的管理新舊資料。
- Range 分區支援的欄位類型有: `[DATE,DATETIME,TINYINT,SMALLINT,INT,BIGINT,LARGEINT]`
- Partition 支援透過 `VALUES LESS THAN (...)` 僅指定上界，系統會將前一個分區的上界作為該分區的下界，產生一個左閉右開的區間。也支援透過 `VALUES [...)` 指定上下界，產生一個左閉右開的區間。
- 同時，也支援透過 `FROM(...) TO (...) INTERVAL ...` 來批次建立分區。
- 透過 `VALUES [...)` 同時指定上下界比較容易理解。這裡舉例說明，當使用 `VALUES LESS THAN (...)` 語句進行分區的增刪操作時，分區範圍的變化情況：

    - 如上 `example_range_tbl` 範例，當建表完成後，會自動產生如下 3 個分區：

    ```text
    p201701: [MIN_VALUE, 2017-02-01)
    p201702: [2017-02-01, 2017-03-01)
    p201703: [2017-03-01, 2017-04-01)
    ```

    - 當我們增加一個分區 `p201705 VALUES LESS THAN ("2017-06-01")`，分區結果如下：

    ```text
    p201701: [MIN_VALUE, 2017-02-01)
    p201702: [2017-02-01, 2017-03-01)
    p201703: [2017-03-01, 2017-04-01)
    p201705: [2017-04-01, 2017-06-01)
    ```

    - 此時我們刪除分割區 `p201703`，則分割區結果如下：

    ```text
    p201701: [MIN_VALUE, 2017-02-01)
    p201702: [2017-02-01, 2017-03-01)
    p201705: [2017-04-01, 2017-06-01)
    ```

    > 注意到 `p201702` 和 `p201705` 的分區範圍並沒有發生變化，而這兩個分區之間，出現了一個空洞：`[2017-03-01, 2017-04-01)`。即如果導入的資料範圍在這個空洞範圍內，是無法導入的。

    - 繼續刪除分割區 `p201702`，分割區結果如下：

    ```text
    p201701: [MIN_VALUE, 2017-02-01)
    p201705: [2017-04-01, 2017-06-01)
    ```

    > 空洞範圍變成：`[2017-02-01, 2017-04-01)`

    - 現在增加一個分割區 `p201702new VALUES LESS THAN ("2017-03-01")`，分割區結果如下：

    ```text
    p201701: [MIN_VALUE, 2017-02-01)
    p201702new: [2017-02-01, 2017-03-01)
    p201705: [2017-04-01, 2017-06-01)
    ```

    > 可以看到空洞範圍縮小為：`[2017-03-01, 2017-04-01)`

    - 現在刪除分割區 `p201701`，並新增分割區 `p201612 VALUES LESS THAN ("2017-01-01")`，分割區結果如下：

    ```text
    p201612: [MIN_VALUE, 2017-01-01)
    p201702new: [2017-02-01, 2017-03-01)
    p201705: [2017-04-01, 2017-06-01)
    ```

    > 即出現了一個新的空洞：`[2017-01-01, 2017-02-01)`


    綜合上述的範例，分割區的刪除不會改變已存在分割區的範圍。刪除分割區可能出現空洞。透過 `VALUES LESS THAN` 語句增加分區時，分區的下界緊接上一個分區的上界。

    Range　分區除了上述我們看到的單列分區，也支援　**多列分區**，範例如下：

    ```text
   PARTITION BY RANGE(`date`, `id`)
   (
       PARTITION `p201701_1000` VALUES LESS THAN ("2017-02-01", "1000"),
       PARTITION `p201702_2000` VALUES LESS THAN ("2017-03-01", "2000"),
       PARTITION `p201703_all` VALUES LESS THAN ("2017-04-01")
   )
   ```
   
   在上述範例中，我們指定 `date`(DATE 類型) 和 `id`(INT 類型) 作為分區列。以上範例最後得到的分區如下：
   
   ```text
       * p201701_1000: [(MIN_VALUE, MIN_VALUE), ("2017-02-01", "1000") )
       * p201702_2000: [("2017-02-01", "1000"), ("2017-03-01", "2000") )
       * p201703_all: [("2017-03-01", "2000"), ("2017-04-01", MIN_VALUE))
   ```
   
   注意，最後一個分區使用者缺省只指定了 `date` 列的分區值，所以 `id` 列的分區值會預設填入 `MIN_VALUE`。當使用者插入資料時，分區列值會依照順序依序比較，最終得到對應的分區。舉例如下：
   
   ``` text
       * 資料 --> 分割區
       * 2017-01-01, 200 --> p201701_1000
       * 2017-01-01, 2000 --> p201701_1000
       * 2017-02-01, 100 --> p201701_1000
       * 2017-02-01, 2000 --> p201702_2000
       * 2017-02-15, 5000 --> p201702_2000
       * 2017-03-01, 2000 --> p201703_all
       * 2017-03-10, 1 --> p201703_all
       * 2017-04-01, 1000 --> 無法匯入
       * 2017-05-01, 1000 --> 無法匯入
   ```

    Range分區同樣支援　**批次分區**， 透過語句　`FROM ("2022-01-03") TO ("2022-01-06") INTERVAL 1 DAY` 批次建立按天劃分的分區：`2022-01- 03` 到 `2022-01-06（不含2022-01-06日）`，分區結果如下：

   ```text
   p20220103: [2022-01-03, 2022-01-04)
   p20220104: [2022-01-04, 2022-01-05)
   p20220105: [2022-01-05, 2022-01-06)
   ```

**2. List 分區:**

- 分區列支援 `BOOLEAN, TINYINT, SMALLINT, INT, BIGINT, LARGEINT, DATE, DATETIME, CHAR, VARCHAR` 資料類型，分區值為枚舉值。只有當資料為目標分區枚舉值其中之一時，才可以命中分區。

- Partition 支援透過 `VALUES IN (...)` 來指定每個分區所包含的枚舉值。

- 以下透過範例說明，進行分區的增刪操作時，分區的變更。

    - 如上 `example_list_tbl` 範例，當建表完成後，會自動產生如下3個分區：

    ```text
    p_cn: ("Beijing", "Shanghai", "Hong Kong")
    p_usa: ("New York", "San Francisco")
    p_jp: ("Tokyo")
    ```

    - 當我們增加一個分區 `p_uk VALUES IN ("London")`，分區結果如下：

    ```text
    p_cn: ("Beijing", "Shanghai", "Hong Kong")
    p_usa: ("New York", "San Francisco")
    p_jp: ("Tokyo")
    p_uk: ("London")
    ```

    - 當我們刪除分區 `p_jp`，分區結果如下：

    ```text
    p_cn: ("Beijing", "Shanghai", "Hong Kong")
    p_usa: ("New York", "San Francisco")
    p_uk: ("London")
    ```

- List 分區也支援 **多列分區**，範例如下：

    ```text
    PARTITION BY LIST(`id`, `city`)
    (
        PARTITION `p1_city` VALUES IN (("1", "Beijing"), ("1", "Shanghai")),
        PARTITION `p2_city` VALUES IN (("2", "Beijing"), ("2", "Shanghai")),
        PARTITION `p3_city` VALUES IN (("3", "Beijing"), ("3", "Shanghai"))
    )
    ```

    在上述範例中，我們指定 `id`(INT 類型) 和 `city`(VARCHAR 類型) 作為分區列。以上範例最後得到的分區如下：

    ```text
    * p1_city: [("1", "Beijing"), ("1", "Shanghai")]
    * p2_city: [("2", "Beijing"), ("2", "Shanghai")]
    * p3_city: [("3", "Beijing"), ("3", "Shanghai")]
    ```

    當使用者插入資料時，分區列值會依照順序依序比較，最終得到對應的分區。舉例如下：
      
    ```text
    * 資料 ---> 分割區
    * 1, Beijing ---> p1_city
    * 1, Shanghai ---> p1_city
    * 2, Shanghai ---> p2_city
    * 3, Beijing ---> p3_city
    * 1, Tianjin ---> 無法匯入
    * 4, Beijing ---> 無法匯入
    ```

---

#### 分桶 (Bucket)

- 如果使用了 `Partition`，則 `DISTRIBUTED ...` 語句描述的是資料在 **各個分區內** 的劃分規則。如果不使用 `Partition`，則描述的是整個表格的資料的分割規則。
- 分桶列可以是多列，Aggregate 和 Unique 模型必須為 Key 列，Duplicate 模型可以是 key 列和 value 列。分桶欄位可以和 Partition 欄位相同或不同。
- 分桶列的選擇，是在 **查詢吞吐** 和 **查詢並發** 之間的一種權衡:
    1. 如果選擇多個分桶列，則資料分佈較均勻。如果一個查詢條件不包含所有分桶列的等值條件，那麼該查詢會觸發所有分桶同時掃描，這樣查詢的吞吐會增加，單一查詢的延遲隨之降低。這個方式適合大吞吐低並發的查詢場景。
    2. 如果只選擇一個或少數分桶列，則對應的點查詢可以只觸發一個分桶掃描。此時，當多個點查詢並發時，這些查詢有較大的機率分別觸發不同的分桶掃描，各個查詢之間的 IO 影響較小（尤其當不同桶分佈在不同磁碟上時），所以這種方式適合高併發的點查詢場景。
- `AutoBucket`: 請 Doris 根據資料量，計算分桶數。對於分區表，可以根據歷史分區的資料量、機器數、盤數，來自動決定一個分桶。
- 分桶的數量理論上沒有上限。

!!! tip
    關於 Partition 和 Bucket 的數量和資料量的建議:

    - 一個表格的 Tablet 總數量等於 (Partition num * Bucket num)。
    - 一個表格的 Tablet 數量，在不考慮擴容的情況下，建議略多於整個叢集的磁碟數量。
    - 單一 Tablet 的資料量理論上沒有上下界，但建議在 1G - 10G 的範圍內。如果單一 Tablet 資料量太小，則資料的聚合效果不佳，且元資料管理壓力大。如果資料量過大，則不利於副本的遷移、補齊，且會增加 Schema Change 或 Rollup 操作失敗重試的代價（這些操作失敗重試的粒度是 Tablet）。
    - 當 Tablet 的資料量原則和數量原則衝突時，建議優先考慮資料量原則。
    - 在建表時，每個分區的 Bucket 數量統一指定。但在動態增加分割區時（`ADD PARTITION`），可以單獨指定新分割區的 Bucket 數量。可以利用這個功能方便的因應資料縮小或膨脹。
    - 一個 Partition 的 Bucket 數量一旦指定，就不會改變。所以在決定 Bucket 數量時，需要預先考慮集群擴容的情況。例如目前只有 3 台 host，每台 host 有 1 塊碟。如果 Bucket 的數量只設定為 3 或更小，那麼後期即使再增加機器，也不能提高並發度。
    - 舉一些例子：假設在有10台BE，每台BE一塊磁碟的情況下。如果一個表總大小為 500MB，則可以考慮4-8個分片。 5GB：8-16分片。 50GB：32個分片。 500GB：建議分割區，每個分割區大小在 50GB 左右，每個分割區16-32個分片。 5TB：建議分割區，每個分割區大小在 50GB 左右，每個分割區16-32個分片。

    註：表格的資料量可以透過[`SHOW DATA`](../sql-manual/sql-reference/Show-Statements/SHOW-DATA.md) 指令查看，結果除以副本數，即表格的資料量。


!!! tip
    關於 Random Distribution 的設定以及使用場景:

    - 如果 OLAP 表沒有更新類型的欄位，將表的資料分桶模式設為 RANDOM，則可以避免嚴重的資料傾斜(資料在匯入表對應的分區的時候，單次匯入作業每個 batch 的資料將隨機選擇一個tablet 進行寫入)。
    - 當表格的分桶模式被設定為 RANDOM 時，因為沒有分桶列，無法根據分桶列的值僅對幾個分桶查詢，對資料表進行查詢的時候將對命中分區的全部分桶同時掃描，此設定適合對資料表資料整體的聚合查詢分析而不適合高並發的點查詢。
    - 如果 OLAP 表的是 Random Distribution 的資料分佈，那麼在資料匯入的時候可以設定單分片匯入模式（將 `load_to_single_tablet` 設為 `true`），那麼在大資料量的匯入的時候，一個任務在將資料寫入對應的分區時將只寫入一個分片，這樣將能提高資料導入的並發度和吞吐量，減少資料導入和 Compaction 導致的寫放大問題，保障集群的穩定性。

#### 複合分區與單一分區

`複合分區`

- 第一級稱為 Partition，即分區。使用者可以指定某一維度列作為分區列（目前只支援整數和時間類型的列），並指定每個分區的取值範圍。
- 第二級稱為 Distribution，即分桶。使用者可以指定一個或多個維度列以及桶數對資料進行 HASH 分佈 或不指定分桶列設定成 Random Distribution 對資料進行隨機分佈。

以下場景推薦使用複合分區:

- 有時間維度或類似有有序值的維度，可以以這類維度列作為分區列。分區粒度可以根據導入頻率、分區資料量等進行評估。
- 歷史資料刪除需求：如有刪除歷史資料的需求（例如僅保留最近 N 天的資料）。使用複合分區，可以透過刪除歷史分區來達到目的。也可以透過在指定分割區內傳送 DELETE 語句進行資料刪除。
- 解決資料傾斜問題：每個分區可以單獨指定分桶數量。如按天分區，當每天的資料量差異很大時，可以透過指定分區的分桶數，合理劃分不同分區的資料,分桶列建議選擇區分度大的列。

使用者也可以不使用複合分區，即使用單一分區。則資料只做 HASH 分佈。

### PROPERTIES

在建表語句的最後 PROPERTIES 中，關於 PROPERTIES 中可以設定的相關參數，我們可以查看 [CREATE TABLE](../sql-manual/sql-reference/Data-Definition-Statements/Create/CREATE-TABLE.md) 中查看詳細的介紹。

### ENGINE

在本範例中，ENGINE 的類型是 `olap`，即預設的 ENGINE 類型。在 Doris 中，只有這個 ENGINE 類型是由 Doris 負責資料管理和儲存的。其他 ENGINE 類型，如 `mysql`、`broker`、`es` 等等，本質上只是對外部其他資料庫或系統中的表的映射，以保證 Doris 可以讀取這些資料。而 Doris 本身並不會創建、管理和儲存任何非 olap ENGINE 類型的表和資料。

### 其他

```text
`IF NOT EXISTS` 表示如果沒有建立過該表，則建立。注意這裡只判斷表名是否存在，而不會判斷新建表結構是否與已存在的表結構相同。所以如果存在一個同名但不同構的表，該指令也會回傳成功，但不代表已經建立了新的表和新的結構。
```

## 常見問題

### 建表操作常見問題

1. 如果在較長的建表語句中出現語法錯誤，可能會出現語法錯誤提示不全的現象。這裡羅列可能的文法錯誤供手動糾錯：

   - 語法結構錯誤。請仔細閱讀 `HELP CREATE TABLE;`，檢查相關語法結構。
   - 保留字。當使用者自訂名稱遇到保留字時，就需要用反引號 `` 引起來。建議所有自訂名稱使用這個符號引起來。
   - 中文字元或全角字元。非 utf8 編碼的中文字符，或隱藏的全角字符（空格，標點等）會導致語法錯誤。建議使用帶有顯示不可見字元的文字編輯器進行檢查。

2. `Failed to create partition [xxx] . Timeout`

   Doris 建表是依照 Partition 粒度依序建立的。當一個 Partition 建立失敗時，可能會回報這個錯誤。即使不使用 Partition，當建表出現問題時，也會報 `Failed to create partition`，因為如前文所述，Doris 會為沒有指定 Partition 的表建立一個不可變更的預設的 Partition。

   當遇到這個錯誤是，通常是 BE 在建立資料分片時遇到了問題。可參考以下步驟排查：

      - 在 fe.log 中，尋找對應時間點的 `Failed to create partition` 日誌。在該日誌中，會出現一系列類似 `{10001-10010}` 字樣的數字對。數字對的第一個數字表示 Backend ID，第二個數字表示 Tablet ID。如上這個數字對，表示 ID 為 10001 的 Backend 上，建立 ID 為 10010 的 Tablet 失敗了。
      - 前往對應 Backend 的 be.INFO 日誌，尋找對應時間段內，tablet id 相關的日誌，可以找到錯誤訊息。
      - 以下羅列一些常見的 tablet 建立失敗錯誤，包括但不限於：
      - BE 沒有收到相關 task，此時無法在 be.INFO 中找到 tablet id 相關日誌或 BE 建立成功，但報告失敗。以上問題，請參閱 [安裝與部署](../install/standard-deployment.md) 檢查 FE 和 BE 的連結性。
      - 預分配記憶體失敗。可能是表中一行的位元組長度超過了 100KB。
      - `Too many open files`。開啟的檔案句柄數超過了 Linux 系統限制。需修改 Linux 系統的句柄數限制。

  如果建立資料分片時逾時，也可以透過在 `fe.conf` 中設定 `tablet_create_timeout_second=xxx` 以及 `max_create_table_timeout_second=xxx` 來延長逾時時間。其中`tablet_create_timeout_second` 預設為 1 秒, `max_create_table_timeout_second` 預設為 60 秒，整體的超時時間為 `min(tablet_create_timeout_second * replication_num, max_create_table_timeout_second)`，具體設定可參考 [FE 配置項](../admin-manual/config/fe-config.md)。

3. 建表命令長時間不回傳結果。

   Doris 的建表指令是同步指令。此指令的超時時間目前設定的比較簡單，即 `(tablet num * replication num)` 秒。如果建立較多的資料分片，且其中有分片建立失敗，則可能導致等待較長逾時後，才會傳回錯誤。

   正常情況下，建表語句會在幾秒鐘或十幾秒內回傳。如果超過一分鐘，建議直接取消掉這個操作，前往 FE 或 BE 的日誌查看相關錯誤。

## 更多幫助

關於資料分割更多的詳細說明，我們可以在　[CREATE TABLE](../sql-manual/sql-reference/Data-Definition-Statements/Create/CREATE-TABLE.md) 命令手冊中查閱，也可以在 Mysql 客戶端下輸入 `HELP CREATE TABLE;` 取得更多的協助資訊。