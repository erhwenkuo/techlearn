# pgvector 簡介

原文: [Github-pgvector/pgvector](https://github.com/pgvector/pgvector)

Postgres 的開源向量相似度搜尋(vector similarity search)。

**pgvector** 將向量與其餘資料一起儲存在 Postgres 數據庫。它支持:

- 精確和近似最近鄰搜索
- L2 距離、內積與餘弦距離
- 使用 Postgres 客戶端的任何語言

加上 ACID 合規性、時間點恢復、JOIN 以及 Postgres 的所有其他出色功能。

## 安裝

編譯並安裝擴充（支援Postgres 11+）。

```bash
cd /tmp
git clone --branch v0.5.1 https://github.com/pgvector/pgvector.git
cd pgvector
make
make install # may need sudo
```

如果遇到問題，請參閱[安裝說明](https://github.com/pgvector/pgvector#installation-notes)。

## 入門

啟用擴充（在每個要使用它的資料庫中執行一次）

```bash
CREATE EXTENSION vector;
```

建立 3 維向量列(row)

```bash
CREATE TABLE items (id bigserial PRIMARY KEY, embedding vector(3));
```

插入向量

```bash
INSERT INTO items (embedding) VALUES ('[1,2,3]'), ('[4,5,6]');
```

透過L2距離獲取最近鄰

```bash
SELECT * FROM items ORDER BY embedding <-> '[3,1,2]' LIMIT 5;
```

也支持內積 (`<#>`) 和餘弦距離 (`<=>`)

!!!info
    注意：`<#>` 傳回負內積，因為 Postgres 僅支援對運算子進行 ASC 順序索引掃描


## 儲存

建立一個帶有向量列的新 table

```bash
CREATE TABLE items (id bigserial PRIMARY KEY, embedding vector(3));
```

或在既有的 table 裡增加一個新的向量列

```bash
ALTER TABLE items ADD COLUMN embedding vector(3);
```

插入(insert)向量

```bash
INSERT INTO items (embedding) VALUES ('[1,2,3]'), ('[4,5,6]');
```

更新插入(upsert)向量

```bash
INSERT INTO items (id, embedding) VALUES (1, '[1,2,3]'), (2, '[4,5,6]')
    ON CONFLICT (id) DO UPDATE SET embedding = EXCLUDED.embedding;
```

更新向量

```bash
UPDATE items SET embedding = '[1,2,3]' WHERE id = 1;
```

刪除向量

```bash
DELETE FROM items WHERE id = 1;
```

## 查詢

取得向量的最近鄰

```bash
SELECT * FROM items ORDER BY embedding <-> '[3,1,2]' LIMIT 5;
```

取得一行中最近的鄰居

```bash
SELECT * FROM items WHERE id != 1 ORDER BY embedding <-> (SELECT embedding FROM items WHERE id = 1) LIMIT 5;
```

取得一定距離內的行

```bash
SELECT * FROM items WHERE embedding <-> '[3,1,2]' < 5;
```

!!!info
    注意：與 `ORDER BY` 和 `LIMIT` 結合使用索引

**Distances**

取得距離

```bash
SELECT embedding <-> '[3,1,2]' AS distance FROM items;
```

對於內積，乘以 `-1`（因為 `<#>` 返回負內積）

```bash
SELECT (embedding <#> '[3,1,2]') * -1 AS inner_product FROM items;
```

對於餘弦相似度，使用 `1 - 餘弦距離`

```bash
SELECT 1 - (embedding <=> '[3,1,2]') AS cosine_similarity FROM items;
```

**Aggregates**

平均向量

```bash
SELECT AVG(embedding) FROM items;
```

平均向量組

```bash
SELECT category_id, AVG(embedding) FROM items GROUP BY category_id;
```

## 索引

預設情況下，pgvector 執行精確的最近鄰搜索，從而提供完美的召回率。

您可以添加索引以使用近似最近鄰搜索，這會犧牲一些召回率來換取速度。與典型索引不同，新增近似索引後，您將看到不同的查詢結果。

支援的索引類型有：

- [HNSW](https://github.com/pgvector/pgvector#hnsw)
- [IVFFlat](https://github.com/pgvector/pgvector#ivfflat)

### HNSW

HNSW 索引建立多層圖。它具有比 IVFFlat 更好的查詢效能（在速度呼叫權衡方面），但建置時間較慢並且使用更多記憶體。另外，由於沒有像 IVFFlat 這樣的訓練步驟，因此可以在沒有任何資料的情況下建立索引。

為您要使用的每個距離函數新增索引。

**L2 distance**

```bash
CREATE INDEX ON items USING hnsw (embedding vector_l2_ops);
```

**Inner product**

```bash
CREATE INDEX ON items USING hnsw (embedding vector_ip_ops);
```

**Cosine distance**

```bash
CREATE INDEX ON items USING hnsw (embedding vector_cosine_ops);
```

最多可以索引 2,000 個維度的向量。

#### 索引選項

指定 HNSW 參數:

- `m` - 每層最大連接數（預設為 16）
- `ef_construction` - 用於構造圖的動態候選清單的大小（預設為 64）

```bash
CREATE INDEX ON items USING hnsw (embedding vector_l2_ops) WITH (m = 16, ef_construction = 64);
```

`ef_construction` 的值越高，可以提供更好的召回率，但代價是索引建置時間/插入速度。

#### 查詢選項

指定搜尋的動態候選清單的大小（預設為40）

```bash
SET hnsw.ef_search = 100;
```

較高的值提供更好的召回率，但代價是速度。

在事務內使用 SET LOCAL 將其設定為單一查詢。

```bash
BEGIN;
SET LOCAL hnsw.ef_search = 100;
SELECT ...
COMMIT;
```

#### 索引建構時間

當 graph 適合 `Maintenance_work_mem` 時，索引建構速度顯著加快

```bash
SET maintenance_work_mem = '8GB';
```

當 graph 不再適合時會顯示一則通知

```bash
NOTICE:  hnsw graph no longer fits into maintenance_work_mem after 100000 tuples
DETAIL:  Building will take significantly more time.
HINT:  Increase maintenance_work_mem to speed up builds.
```

!!!info
    注意：不要將 `maintenance_work_mem` 設定得太高，否則會耗盡伺服器上的內存

#### 索引進度

使用 Postgres 12+ 檢查索引進度

```bash
SELECT phase, round(100.0 * blocks_done / nullif(blocks_total, 0), 1) AS "%" FROM pg_stat_progress_create_index;
```

HNSW 的階段是：

1. initializing
2. loading tuples

### IVFFlat

IVFFlat 索引將向量分割成列表，然後搜尋這些列表中最接近查詢向量的子集。與 HNSW 相比，它的建置時間更快，使用的記憶體更少，但查詢效能較低（在速度與呼叫權衡方面）。

實現良好召回率的三個關鍵是：

1. 當資料表有一些資料後建立索引
2. 選擇適當數量的清單 - 一個好的起始位置是 `rows / 1000`（最多 1M 行）和 `sqrt(rows)`（超過 1M 行）
3. 查詢時，指定適當的探針數量（召回率越高，速度越低） - 一個好的起點是 `sqrt(lists)`

為您要使用的每個距離函數新增索引。

**L2 distance**

```bash
CREATE INDEX ON items USING ivfflat (embedding vector_l2_ops) WITH (lists = 100);
```

**Inner product**

```bash
CREATE INDEX ON items USING ivfflat (embedding vector_ip_ops) WITH (lists = 100);
```

**Cosine distance**

```bash
CREATE INDEX ON items USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
```

最多可以索引 2,000 個維度的向量。

#### 查詢選項

指定探針數量（預設為1）

```bash
SET ivfflat.probes = 10;
```

較高的值以速度為代價提供更好的召回率，並且可以將其設定為精確最近鄰搜尋的清單數量（此時規劃器將不會使用索引）

在事務內使用 `SET LOCAL` 將其設定為單一查詢:

```bash
BEGIN;
SET LOCAL ivfflat.probes = 10;
SELECT ...
COMMIT;
```

#### 索引建構時間

透過增加並行工作線程的數量（預設為 2）來加速大型表上的索引創建

```bash
SET max_parallel_maintenance_workers = 7; -- plus leader
```

對於大量 worker，您可能還需要增加 `max_parallel_workers` （預設為 8）

#### 索引進度

使用 Postgres 12+ 檢查索引進度

```bash
SELECT phase, round(100.0 * tuples_done / nullif(tuples_total, 0), 1) AS "%" FROM pg_stat_progress_create_index;
```

IVFFlat 的階段是：

1. initializing
2. performing k-means
3. assigning tuples
4. loading tuples

!!!info
    注意：% 僅在載入元組階段填充

## 過濾

有幾種方法可以使用 WHERE 子句對最近鄰居查詢進行索引

```bash
SELECT * FROM items WHERE category_id = 123 ORDER BY embedding <-> '[3,1,2]' LIMIT 5;
```

在一個或多個 WHERE 列上建立索引以進行精確搜索

```bash
CREATE INDEX ON items (category_id);
```

或向量列上的部分索引以進行近似搜索

```bash
CREATE INDEX ON items USING hnsw (embedding vector_l2_ops) WHERE (category_id = 123);
```

使用分區對 WHERE 列的許多不同值進行近似搜索

```bash
CREATE TABLE items (embedding vector(3), category_id int) PARTITION BY LIST(category_id);
```

## 混合搜尋

與 Postgres [全文搜尋](https://www.postgresql.org/docs/current/textsearch-intro.html)一起使用以進行混合搜尋。

```bash
SELECT id, content FROM items, plainto_tsquery('hello search') query
    WHERE textsearch @@ query ORDER BY ts_rank_cd(textsearch, query) DESC LIMIT 5;
```

您可以使用倒數排名融合([Reciprocal Rank Fusion](https://github.com/pgvector/pgvector-python/blob/master/examples/hybrid_search_rrf.py))或交叉編碼器([cross-encoder](https://github.com/pgvector/pgvector-python/blob/master/examples/hybrid_search.py))來組合結果。

## 效能

使用 `EXPLAIN ANALYZE` 調試效能。

```bash
EXPLAIN ANALYZE SELECT * FROM items ORDER BY embedding <-> '[3,1,2]' LIMIT 5;
```

### 精確搜尋

若要加快沒有索引的查詢速度，請增加 `max_parallel_workers_per_gather`。

```bash
SET max_parallel_workers_per_gather = 4;
```

如果向量標準化為長度 1（如 [OpenAI 嵌入](https://platform.openai.com/docs/guides/embeddings/which-distance-function-should-i-use)），請使用內積以獲得最佳效能。

```bash
SELECT * FROM items ORDER BY embedding <#> '[3,1,2]' LIMIT 5;
```

### 大致搜尋

若要加快使用 IVFFlat 索引的查詢速度，請增加倒排列表的數量（以召回為代價）。

```bash
CREATE INDEX ON items USING ivfflat (embedding vector_l2_ops) WITH (lists = 1000);
```

## 參考

### Vector 型別

每個向量佔用 `4 * 維度 + 8` 個位元組的儲存空間。每個元素都是單精確度浮點數（就像 Postgres 中的 `real` 類型），並且所有元素都必須是有限的（沒有 `NaN`、`Infinity` 或 `-Infinity`）。向量最多可以有 16,000 個維度。

### Vector 操作子

|Operator	|Description	|Added|
|-----------|---------------|-----|
|+	|element-wise addition	||
|-	|element-wise subtraction	||
|*	|element-wise multiplication|	0.5.0|
|<->	|Euclidean distance	||
|<#>	|negative inner product	||
|<=>	|cosine distance	||

### Vector 函數


|Function	|Description	|Added|
|-----------|---------------|-----|
|cosine_distance(vector, vector) → double precision	|cosine distance	||
|inner_product(vector, vector) → double precision	|inner product	||
|l2_distance(vector, vector) → double precision	|Euclidean distance	||
|l1_distance(vector, vector) → double precision	|taxicab distance	|0.5.0|
|vector_dims(vector) → integer	|number of dimensions	||
|vector_norm(vector) → double precision	|Euclidean norm	||

### Aggregate 函數

|Function	|Description	|Added|
|-----------|---------------|-----|
|avg(vector) → vector	|average||	
|sum(vector) → vector	|sum	|0.5.0|



