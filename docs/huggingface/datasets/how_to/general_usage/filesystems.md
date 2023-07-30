# Cloud storage

🤗 數據集支持通過 `fsspec` 文件系統實現訪問雲存儲服務的存取。您可以以 Python 方式從任何雲存儲服務來保存和加載數據集。請查看下表，了解受支持的雲存儲服務的一些範例：

|Storage provider	|Filesystem implementation|
|-----------------|-------------------------|
|`Amazon S3`	|[s3fs](https://s3fs.readthedocs.io/en/latest/)|
|`Google Cloud Storage`	|[gcsfs](https://gcsfs.readthedocs.io/en/latest/)|
|`Azure Blob/DataLake`	|[adlfs](https://github.com/fsspec/adlfs)|
|`Dropbox`	|[dropboxdrivefs](https://github.com/MarineChap/dropboxdrivefs)|
|`Google Drive`	|[gdrivefs](https://github.com/intake/gdrivefs)|
|`Oracle Cloud Storage`	|[ocifs](https://ocifs.readthedocs.io/en/latest/)|

本指南將向您展示如何使用任何雲存儲服務保存和加載數據集。以下是 `S3`、`Google 雲存儲`、`Azure Blob 存儲`和 `Oracle 雲 Object 存儲`的範例。

##　Set up your cloud storage FileSystem

###　Amazon S3

1. 安裝 S3 FileSystem implementation：

    ```python
     pip install s3fs
    ```

2. 定義您的 credentials

    要使用匿名連接，請使用 `anon=True`。否則，每當您與私有 S3 存儲桶交互時，請包含您的 `aws_access_key_id` 和 `aws_secret_access_key`。

    === "AWS S3"

        ```python
        storage_options = {"anon": True}  # for anonymous connection
        storage_options = {"key": aws_access_key_id, "secret": aws_secret_access_key}  # for private buckets

        import aiobotocore.session

        s3_session = aiobotocore.session.AioSession(profile="my_profile_name")

        storage_options = {"session": s3_session}
        ```

    === "Minio"

        ```python
        storage_options = {
            "anon": False, # for anonymous connection
            "key": "*********", # access_key
            "secret": "*********", # secret_key
            "use_ssl": False, # use https or http
            "client_kwargs": {
                "endpoint_url": "http://localhost:9000" # minio endpoint
            }
        }
        ```

3. 創建你的 `FileSystem instance`

    ```python
    import s3fs
    fs = s3fs.S3FileSystem(**storage_options)
    ```

#### Class: `datasets.filesystems.S3FileSystem`

Parameters:

- `anon` (bool, default to False) — Whether to use anonymous connection (public buckets only). If False, uses the key/secret given, or boto’s credential - resolver (client_kwargs, environment, variables, config files, EC2 IAM server, in that order).
- `key` (str) — If not anonymous, use this access key ID, if specified.
- `secret` (str) — If not anonymous, use this secret access key, if specified.
- `token` (str) — If not anonymous, use this security token, if specified.
- `use_ssl` (bool, defaults to True) — Whether to use SSL in connections to S3; may be faster without, but insecure. If use_ssl is also set in client_kwargs, the value set in client_kwargs will take priority.
- `s3_additional_kwargs` (dict) — Parameters that are used when calling S3 API methods. Typically used for things like ServerSideEncryption.
- `client_kwargs` (dict) — Parameters for the botocore client.
- `requester_pays` (bool, defaults to False) — Whether RequesterPays buckets are supported.
- `default_block_size` (int) — If given, the default block size value used for open(), if no specific value is given at all time. The built-in default is 5MB.
- `default_fill_cache` (bool, defaults to True) — Whether to use cache filling with open by default. Refer to S3File.open.
- `default_cache_type` (str, defaults to bytes) — If given, the default cache_type value used for open(). Set to none if no caching is desired. See fsspec’s documentation for other available cache_type values.
- `version_aware` (bool, defaults to False) — Whether to support bucket versioning. If enable this will require the user to have the necessary IAM permissions for dealing with versioned objects.
- `cache_regions` (bool, defaults to False) — Whether to cache bucket regions. Whenever a new bucket is used, it will first find out which region it belongs to and then use the client for that region.
- `asynchronous` (bool, defaults to False) — Whether this instance is to be used from inside coroutines.
- `config_kwargs` (dict) — Parameters passed to botocore.client.Config. **kwargs — Other parameters for core session.
- `session` (aiobotocore.session.AioSession) — Session to be used for all connections. This session will be used inplace of creating a new session inside S3FileSystem. For example: aiobotocore.session.AioSession(profile='test_user').
- `skip_instance_cache` (bool) — Control reuse of instances. Passed on to fsspec.
- `use_listings_cache` (bool) — Control reuse of directory listings. Passed on to fsspec.
- `listings_expiry_time` (int or float) — Control reuse of directory listings. Passed on to fsspec.
- `max_paths` (int) — Control reuse of directory listings. Passed on to fsspec.

`datasets.filesystems.S3FileSystem` 是 `s3fs.S3FileSystem` 的子類。

用戶可以使用此 class 來訪問 S3，就像它是一個文件系統一樣。它在 S3 存儲之上公開了類似文件系統的 API（`ls`、`cp`、`open` 等）。顯式提供憑據　`(key=、secret=)` 或使用 `boto`` 的憑據方法。請參閱 `botocore` 文檔以獲取更多信息。如果沒有可用的憑據，請使用 `anon=True`。

範例:

**Listing files from public S3 bucket**

```python
import datasets

s3 = datasets.filesystems.S3FileSystem(anon=True)

s3.ls('public-datasets/imdb/train')
```

結果:

```bash
['dataset_info.json.json','dataset.arrow','state.json']
```

**Listing files from private S3 bucket using aws_access_key_id and aws_secret_access_key.**

```python
import datasets

s3 = datasets.filesystems.S3FileSystem(key=aws_access_key_id, secret=aws_secret_access_key)

s3.ls('my-private-datasets/imdb/train')
```

結果:

```bash
['dataset_info.json.json','dataset.arrow','state.json']
```

**Using S3Filesystem with `botocore.session.Session` and custom aws_profile.**

```python
import botocore
from datasets.filesystems import S3Filesystem

s3_session = botocore.session.Session(profile_name='my_profile_name')

s3 = S3FileSystem(session=s3_session)
```

**Loading dataset from S3 using S3Filesystem and `load_from_disk()`.**

=== "AWS S3"

    ```python
    from datasets import load_from_disk
    from datasets.filesystems import S3Filesystem

    s3 = S3FileSystem(key=aws_access_key_id, secret=aws_secret_access_key)

    dataset = load_from_disk('s3://my-private-datasets/imdb/train', storage_options=s3.storage_options)

    print(len(dataset))
    ```

    結果:

    ```bash
    25000
    ```

=== "Minio"

    ```python
    from datasets import load_from_disk

    storage_options = {
            "anon": False, # for anonymous connection
            "key": "minioadmin", # access_key
            "secret": "minioadmin", # secret_key
            "use_ssl": False, # use https or http
            "client_kwargs": {
                "endpoint_url": "http://localhost:9000" # minio endpoint
            }
        }

    dataset = load_from_disk('s3://my-private-datasets/imdb/train', storage_options=storage_options)

    print(len(dataset))
    ```

    結果:

    ```bash
    25000
    ```

**Saving dataset to S3 using S3Filesystem and `Dataset.save_to_disk()`.**

=== "AWS S3"

```python
from datasets import load_dataset
from datasets.filesystems import S3Filesystem

dataset = load_dataset("imdb")

s3 = S3FileSystem(key=aws_access_key_id, secret=aws_secret_access_key)

dataset.save_to_disk('s3://my-private-datasets/imdb/train', storage_options=s3.storage_options)
```

=== "Minio"

    ```python
    from datasets import load_dataset

    dataset = load_dataset("imdb")

    storage_options = {
                "anon": False, # for anonymous connection
                "key": "minioadmin", # access_key
                "secret": "minioadmin", # secret_key
                "use_ssl": False, # use https or http
                "client_kwargs": {
                    "endpoint_url": "http://localhost:9000" # minio endpoint
                }
            }

    dataset.save_to_disk('s3://my-private-datasets/imdb/train', storage_options=storage_options)
    ```

### Google Cloud Storage

1. 安裝 Google Cloud Storage implementation：

    ```python
    conda install -c conda-forge gcsfs

    # or install using pip
    pip install gcsfs
    ```

2. 定義您的 credentials

    ```python
    storage_options={"token": "anon"}  # for anonymous connection

    # or use your credentials of your default gcloud credentials or from the google metadata service
    storage_options={"project": "my-google-project"}

    # or use your credentials from elsewhere, see the documentation at https://gcsfs.readthedocs.io/
    storage_options={"project": "my-google-project", "token": TOKEN}
    ```

3. 創建你的 `FileSystem instance`

    ```python
    import gcsfs

    fs = gcsfs.GCSFileSystem(**storage_options)
    ```

### Azure Blob Storage

1. 安裝 Azure Blob Storage implementation：

    ```python
    conda install -c conda-forge adlfs

    # or install using pip
    pip install adlfs
    ```

2. 定義您的 credentials

    ```python
    storage_options = {"anon": True}  # for anonymous connection

    # or use your credentials
    storage_options = {"account_name": ACCOUNT_NAME, "account_key": ACCOUNT_KEY}  # gen 2 filesystem

    # or use your credentials with the gen 1 filesystem
    storage_options={"tenant_id": TENANT_ID, "client_id": CLIENT_ID, "client_secret": CLIENT_SECRET}
    ```

3. 創建你的 `FileSystem instance`

    ```python
    import adlfs

    fs = adlfs.AzureBlobFileSystem(**storage_options)
    ```

### Oracle Cloud Object Storage

1. 安裝 OCI FileSystem implementation：

    ```python
    conda install -c conda-forge ocifs

    # or install using pip
    pip install ocifs
    ```

2. 定義您的 credentials

    ```python
    storage_options = {"config": "~/.oci/config", "region": "us-ashburn-1"} 
    ```

3. 創建你的 `FileSystem instance`

    ```python
    import ocifs

    fs = ocifs.OCIFileSystem(**storage_options)
    ```

## Load and Save your datasets using your cloud storage FileSystem

### Download and prepare a dataset into a cloud storage

您可以通過在 `download_and_prepare` 中指定遠程 `output_dir` 將數據集下載並準備到雲存儲服務中。不要忘記使用之前定義的包含您的憑據的 `storage_options`。

`download_and_prepare` 方法分兩個步驟進行：

1. 它首先下載本地緩存中的原始數據文件（如果有）。您可以通過將 `cache_dir` 傳遞給 `load_dataset_builder()` 來設置緩存目錄
2. 然後它通過迭代原始數據文件在雲存儲中生成 `Arrow` 或 `Parquet` 格式的數據集。

範例:

從 Hugging Face Hub 加載數據集構建器(了解[how to load from the Hugging Face Hub](https://huggingface.co/docs/datasets/loading#hugging-face-hub)):

```python
from datasets import load_dataset_builder

output_dir = "s3://my-bucket/imdb"

builder = load_dataset_builder("imdb")

builder.download_and_prepare(output_dir, storage_options=storage_options, file_format="parquet")
```

另外一種手法是使用本地的 `loading script` 來加載 dataset builder（請參閱 [how to load a local loading script](https://huggingface.co/docs/datasets/loading#local-loading-script)):

```python
output_dir = "s3://my-bucket/imdb"

builder = load_dataset_builder("path/to/local/loading_script/loading_script.py")

builder.download_and_prepare(output_dir, storage_options=storage_options, file_format="parquet")
```

使用您自己的數據文件 (請參閱 [how to load local and remote files](https://huggingface.co/docs/datasets/loading#local-and-remote-files)):

```python
data_files = {"train": ["path/to/train.csv"]}

output_dir = "s3://my-bucket/imdb"

builder = load_dataset_builder("csv", data_files=data_files)

builder.download_and_prepare(output_dir, storage_options=storage_options, file_format="parquet")
```

強烈建議將文件保存為壓縮的 `Parquet` 文件，通過指定 `file_format="parquet"` 來優化 I/O。否則，數據集將保存為未壓縮的 `Arrow` 文件。

您還可以使用 `max_shard_size` 指定分片的大小（默認為 `500MB`）：

```python
builder.download_and_prepare(output_dir, storage_options=storage_options, file_format="parquet", max_shard_size="1GB")
```

### Dask

Dask 是一個並行計算庫，它有一個類似 pandas 的 API，用於並行處理大於內存的 Parquet 數據集。 Dask 可以在單台機器或機器集群上使用多個線程或進程來並行處理數據。 Dask 支持本地數據，也支持來自雲存儲的數據。

因此，您可以在 Dask 中加載保存為分片 Parquet 文件的數據集：

```python
import dask.dataframe as dd

df = dd.read_parquet(output_dir, storage_options=storage_options)

# or if your dataset is split into train/valid/test
df_train = dd.read_parquet(output_dir + f"/{builder.name}-train-*.parquet", storage_options=storage_options)
df_valid = dd.read_parquet(output_dir + f"/{builder.name}-validation-*.parquet", storage_options=storage_options)
df_test = dd.read_parquet(output_dir + f"/{builder.name}-test-*.parquet", storage_options=storage_options)
```

您可以在其[文檔](https://docs.dask.org/en/stable/dataframe.html)中找到有關 dask dataframes的更多信息。

## Saving serialized datasets

處理完數據集後，您可以使用 [`Dataset.save_to_disk()`](https://huggingface.co/docs/datasets/v2.14.1/en/package_reference/main_classes#datasets.Dataset.save_to_disk) 將其保存到雲存儲中：

```python
encoded_dataset.save_to_disk("s3://my-private-datasets/imdb/train", storage_options=storage_options)

encoded_dataset.save_to_disk("gcs://my-private-datasets/imdb/train", storage_options=storage_options)

encoded_dataset.save_to_disk("adl://my-private-datasets/imdb/train", storage_options=storage_options)
```

!!! info
    每當您與私有雲存儲交互時，請記住在[文件系統](https://huggingface.co/docs/datasets/filesystems#set-up-your-cloud-storage-filesystem)實例 fs 中定義您的 credentials。

範例:

```python
# load dataset from Hugging Hub
dataset = load_dataset("imdb")

# setup storage options for remote file system
storage_options = {
    "anon": False, # for anonymous connection
    "key": "minioadmin", # access_key
    "secret": "minioadmin", # secret_key
    "use_ssl": False, # use https or http
    "client_kwargs": {
        "endpoint_url": "http://localhost:9000" # minio endpoint
    }
}

# serialize dataset to remote file system
dataset.save_to_disk("s3://my-private-datasets/imdb", storage_options=storage_options)
```

###　Listing serialized datasets

使用 `fs.ls` 列出帶有文件系統實例 fs 的雲存儲中的文件：

```python
fs.ls("my-private-datasets/imdb", detail=False)
```

結果:

```bash
["dataset_info.json.json","dataset.arrow","state.json"]
```

## Load serialized datasets

當您準備好再次使用數據集時，請使用 [`Dataset.load_from_disk()`](https://huggingface.co/docs/datasets/v2.14.1/en/package_reference/main_classes#datasets.Dataset.load_from_disk) 重新加載它：

```python
from datasets import load_from_disk

dataset = load_from_disk("s3://my-private-datasets/imdb", storage_options=storage_options)

print(len(dataset))
```

範例:

```python
import datasets

# setup storage options for remote file system
storage_options = {
        "anon": False, # for anonymous connection
        "key": "minioadmin", # access_key
        "secret": "minioadmin", # secret_key
        "use_ssl": False, # use https or http
        "client_kwargs": {
            "endpoint_url": "http://localhost:9000" # minio endpoint
        }
    }

# load dataset from remote file system
dataset = datasets.load_from_disk("s3://my-private-datasets/imdb", storage_options=storage_options)

print(dataset)
```

結果:

```bash
DatasetDict({
    train: Dataset({
        features: ['text', 'label'],
        num_rows: 25000
    })
    test: Dataset({
        features: ['text', 'label'],
        num_rows: 25000
    })
    unsupervised: Dataset({
        features: ['text', 'label'],
        num_rows: 50000
    })
})
```