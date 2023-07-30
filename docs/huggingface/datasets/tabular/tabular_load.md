# Load tabular data

Ｔabular dataset 是用於描述存儲在行和列中的任何數據的通用數據集，其中行(row)代表示例，列(column)代表特徵（可以是連續的或分類的）。這些數據集通常存儲在 CSV 文件、Pandas DataFrame 和數據庫表中。本指南將向您展示如何從以下位置加載和創建表格數據集：

- CSV files
- Pandas DataFrames
- Databases

## CSV files

🤗 數據集可以通過在 `load_dataset()` 方法中指定通用 `csv` dataset builder name 來讀取 CSV 文件。要加載多個 CSV 文件，請將它們作為列表傳遞給 `data_files` 參數：

```python
from datasets import load_dataset

# load single CSV file
dataset = load_dataset("csv", data_files="my_file.csv")

# load multiple CSV files
dataset = load_dataset("csv", data_files=["my_file_1.csv", "my_file_2.csv", "my_file_3.csv"])
```

您還可以將特定的 CSV 文件映射到 `train` 和 `test` 的 split：

```python
dataset = load_dataset("csv", data_files={"train": ["my_train_file_1.csv", "my_train_file_2.csv"], "test": "my_test_file.csv"})
```

要加載遠程 CSV 文件，請傳遞 URL：

```python
base_url = "https://huggingface.co/datasets/lhoestq/demo1/resolve/main/data/"

dataset = load_dataset('csv', data_files={"train": base_url + "train.csv", "test": base_url + "test.csv"})
```

要加載遠程 CSV 文件(s3)，請傳遞 URL 並設　`storage_options`：

```python
# setup s3(mino) storage option
storage_options = {
    "anon": False,  # for anonymous connection
    "key": "minioadmin",  # access_key
    "secret": "minioadmin",  # secret_key
    "use_ssl": False,  # use https or http
    "client_kwargs": {
        "endpoint_url": "http://localhost:9000"  # minio endpoint
    }
}

base_url = "s3://private_bucket/datasets/titanic/"

dataset = load_dataset("csv", data_files= base_url + "train.csv", storage_options=storage_options)
```

要加載壓縮的 CSV 文件：

```python
url = "https://domain.org/train_data.zip"

data_files = {"train": url}

dataset = load_dataset("csv", data_files=data_files)
```

## Pandas DataFrames

🤗 Datasets 還支持使用 `from_pandas()` 方法從 Pandas DataFrames 加載數據集：

```python
from datasets import Dataset
import pandas as pd

# create a Pandas DataFrame
df = pd.read_csv("https://huggingface.co/datasets/imodels/credit-card/raw/main/train.csv")

df = pd.DataFrame(df)

# load Dataset from Pandas DataFrame
dataset = Dataset.from_pandas(df)
```

使用 `splits` 參數指定數據集拆分的名稱：

```python
train_ds = Dataset.from_pandas(train_df, split="train")

test_ds = Dataset.from_pandas(test_df, split="test")
```

如果數據集看起來不符合預期，您應該顯式指定數據集特徵。 `pandas.Series` 可能並不總是攜帶足夠的信息供 Arrow 自動推斷數據類型。例如，如果 DataFrame 的長度為 0 或者 Series 僅包含 None/NaN 物件，則類型設置為 `null`。

##　Databases

存儲在數據庫中的數據集通常通過 SQL 查詢進行訪問。使用🤗數據集，您可以連接到數據庫，查詢所需的數據，並從中創建數據集。然後，您可以使用🤗數據集的所有處理功能來準備訓練數據集。

###　SQLite

SQLite 是一個小型、輕量級數據庫，快速且易於設置。如果您願意，可以使用現有數據庫，或者從頭開始。

首先使用來自《紐約時報》的 Covid-19 數據創建一個快速 SQLite 數據庫：

```python
import sqlite3
import pandas as pd

conn = sqlite3.connect("us_covid_data.db")

df = pd.read_csv("https://raw.githubusercontent.com/nytimes/covid-19-data/master/us-states.csv")

df.to_sql("states", conn, if_exists="replace")
```

這會在 us_covid_data.db 數據庫中創建一個 states 表，您現在可以將其加載到數據集中。

要連接到數據庫，您需要標識數據庫的 URI 字符串。使用 URI 連接到數據庫會緩存返回的數據集。每種數據庫 dialect 的 URI 字符串都不同，因此請務必檢查您使用的數據庫的[數據庫 URL](https://docs.sqlalchemy.org/en/13/core/engines.html#database-urls)。

對於 SQLite 來說，它是：

```bash
uri = "sqlite:///us_covid_data.db"
```

通過將表名和 URI 傳遞給 `from_sql()` 來加載資料表：

```python
from datasets import Dataset

ds = Dataset.from_sql("states", uri)

print(ds)
```

結果:

```bash
Dataset({
    features: ['index', 'date', 'state', 'fips', 'cases', 'deaths'],
    num_rows: 54382
})
```

然後您可以使用所有🤗數據集處理功能，例如 `filter()`：

```python
ds.filter(lambda x: x["state"] == "California")
```

您還可以從 SQL 查詢加載數據集而不是整個表，這對於查詢和聯接多個表非常有用。

通過將查詢和 URI 傳遞給 `from_sql()` 來加載數據集：

```python
from datasets import Dataset

ds = Dataset.from_sql('SELECT * FROM states WHERE state="California";', uri)

print(ds)
```

結果:

```bash
Dataset({
    features: ['index', 'date', 'state', 'fips', 'cases', 'deaths'],
    num_rows: 1019
})
```

然後您可以使用所有🤗數據集處理功能，例如 `filter()`：

```python
ds.filter(lambda x: x["cases"] > 10000)
```

### PostgreSQL

您還可以從 PostgreSQL 數據庫連接並加載數據集，但是我們不會在文檔中直接演示如何操作，因為該示例僅適用於在筆記本中運行。相反，看看如何在此筆記本中安裝和設置 PostgreSQL 服務器！

設置 PostgreSQL 數據庫後，您可以使用 `from_sql()` 方法從表或查詢加載數據集。

