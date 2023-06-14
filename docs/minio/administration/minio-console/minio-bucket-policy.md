# Minio Bucket Policy 教程

原文: [Minio Bucket Policy Notes](https://blog.nikhilbhardwaj.in/2020/02/25/minio-bucket-policy/)

Minio 是一個非常酷的開源項目，它使雲存儲大眾化。我最喜歡它的功能是 S3 兼容性，這意味著您可以將它與 AWS CLI 或任何其他 AWS SDK 一起使用。公司可以使用它來運行自己的分佈式對象存儲系統。

我在家裡將其與我的服務器一起使用，以便為我自己和我的家人提供數 TB 的存儲空間。我已經連接了 3 個分別為 4、2 和 1 TB 的硬盤驅動器，並通過我的家庭互聯網和公共互聯網無線訪問它們。將其視為我的安全自託管私有 S3。

即使這是一項私人存儲服務，我也不願意將所有驅動器暴露給任何具有單個訪問和秘密密鑰對的人。它變得太冒險了，任何用戶都可能不小心擦除每個人的數據。值得慶幸的是，minio 中的權限建模類似於 S3，但是文檔有點稀疏且很難找到。因此，我正在記錄我的工作流程，如果您嘗試做類似的事情，這可能會對您有所幫助，並且肯定會幫助我避免搜索我的 shell 歷史記錄。

一個示例用例是我想授予家庭成員訪問他們自己的私人存儲桶的權限。

## Step 0 - Create mc alias

以下命令為在 URL https://myminio.example.net 上運行的 MinIO 部署 myminio 添加別名。 mc 使用指定的用戶名和密碼對 MinIO 部署進行身份驗證：

```bash
mc alias set local http://localhost:9000 {SECRETKEY} {ACCESS_KEY}
```

## Step 1 - Create the bucket

```bash
mc mb local/wifey
```

## Step 2 - Create the credentials pair

```bash
mc admin user add local wifey-user wife-top-secret-key
```

## Step 3 - Create the policy to grant access to the bucket

```json title="sample-bucket-policy.json"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
            "s3:ListBucket",
            "s3:PutObject",
            "s3:GetObject",
            "s3:DeleteObject"
        ],
      "Effect": "Allow",
      "Resource": [
        "arn:aws:s3:::wifey/*", "arn:aws:s3:::wifey"
      ],
      "Sid": "BucketAccessForUser"
    }
  ]
}
```

這裡的 `Version` 並不是今天的日期，所以不要更改它。這是定義 IAM 策略支持的語法的版本。

將這個 `sample-bucket-policy.json` 文件保存在某個地方，接下來我們將把這個策略添加到 minio 實例中。

## Step 4 - Add policy to your minio instance

```bash
mc admin policy create local wifey-bucket-policy sample-bucket-policy.json
```

替換文件的路徑。

## Step 5 - Associate policy with your user

```bash
mc admin policy attach local wifey-bucket-policy --user wifey-user
```

現在，您與用戶共享的憑據將只允許他們訪問這個存儲桶。

您可以通過運行以下命令來驗證是否已按預期設置所有內容 -

```bash
mc admin user info local wifey-user
```

結果:

```bash
AccessKey: wifey-user
Status: enabled
PolicyName: wifey-bucket-policy
MemberOf: []
```
