# 訪問管理

## 概述

MinIO 使用基於策略 policy 的訪問控制 (PBAC) 來定義經過身份驗證的用戶有權訪問的授權操作和資源。每個策略都描述一個或多個操作和條件，概述一個 user 或一組 user 的權限(group)。

MinIO PBAC 專為與 AWS IAM 策略語法、結構和行為兼容而構建。 MinIO 文檔盡力涵蓋 IAM 特定的行為和功能。請考慮遵循 [IAM 文檔](https://docs.aws.amazon.com/IAM/latest/UserGuide/)，以獲取有關 AWS IAM 特定主題的更完整文檔。

[`mc admin policy`](https://min.io/docs/minio/linux/reference/minio-mc-admin/mc-admin-policy.html#command-mc.admin.policy) 命令支持在 MinIO 部署上創建和管理策略。有關使用示例，請參閱命令參考。

## Tag-Based 策略條件

策略(policy)可以使用條件(condition)來限制用戶僅訪問具有特定標籤的 objects。

MinIO 支持針對選定操作的策略的基於標籤 [tag-based conditionals](https://docs.aws.amazon.com/AmazonS3/latest/userguide/tagging-and-policies.html)的條件。在策略的 `Condition` 語句中使用 `s3:ExistingObjectTag/<key>`。

## Built-In 策略

MinIO 提供以下內置策略可用於分配給 user 或 group：

**consoleAdmin**

授予對 MinIO 部署上所有資源的所有 S3 和管理 API 操作的完全訪問權限。相當於以下一組操作：

- `s3:*`
- `admin:*`

**readonly**

授予對 MinIO 部署上任何對象的唯讀權限。 GET 操作必須應用於特定對象，而不需要任何列表。相當於以下一組操作：

- `s3:GetBucketLocation`
- `s3:GetObject`

例如，該策略專門支持對特定路徑上的對象進行 GET 操作（例如 `GET play/mybucket/object.file`），例如：

- `mc cp`
- `mc stat`
- `mc head`
- `mc cat`

排除列出權限是有意為之的，因為典型的用例並不打算讓“唯讀”角色在對象存儲資源上具有完全的可發現性（列出所有存儲桶和對象）。

**readwrite**

授予 MinIO 服務器上所有存儲桶和對象的讀寫權限。相當於 `s3:*`。

**diagnostics**

授予對 MinIO 部署執行診斷操作的權限。具體包括以下幾個動作：

- `admin:ServerTrace`
- `admin:Profiling`
- `admin:ConsoleLog`
- `admin:ServerInfo`
- `admin:TopLocksInfo`
- `admin:OBDInfo`
- `admin:BandwidthMonitor`
- `admin:Prometheus`

**writeonly**

授予對 MinIO 部署的任何命名空間（存儲桶和對象路徑）的唯寫權限。 PUT 操作必須應用於特定的對象位置，而不需要任何列表。相當於 `s3:PutObject` 操作。

使用 `mc admin policy Attach` 將策略關聯到 MinIO 部署上的 user 或 group。

例如，請考慮以下用戶表。每個用戶都被分配了一個內置策略或支持的操作。該表描述了客戶端在通過該用戶身份驗證後可以執行的操作子集：

|User |Policy |Operations|
|-----|-------|----------|
|Operations|`readwrite` on `finance` bucket<br/>`readonly` on `audit` bucket|`PUT` and `GET` on `finance` bucket.<br/>`PUT` on `audit` bucket.|
|Auditing|`readonly` on `audit` bucket|`GET` on `audit` bucket|
|Admin|`admin:*`|All `mc admin` commands.|

每個用戶只能訪問內置角色明確授予的資源和操作。默認情況下，MinIO 拒絕訪問任何其他資源或操作。

## Policy 文件結構

MinIO 策略文檔使用與 [AWS IAM 策略](https://docs.aws.amazon.com/IAM/latest/UserGuide/access.html)文檔相同的架構。

以下範例文檔提供了一個模板，用於創建用於 MinIO 部署的自定義策略。有關 IAM 策略元素的更完整文檔，請參閱 [IAM JSON 策略元素參考](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements.html)。Policy 文檔的最大大小為 2048 個字符。

```json
{
   "Version" : "2012-10-17",
   "Statement" : [
      {
         "Effect" : "Allow",
         "Action" : [ "s3:<ActionName>", ... ],
         "Resource" : "arn:aws:s3:::*",
         "Condition" : { ... }
      },
      {
         "Effect" : "Deny",
         "Action" : [ "s3:<ActionName>", ... ],
         "Resource" : "arn:aws:s3:::*",
         "Condition" : { ... }
      }
   ]
}
```

- 對於 `Statement.Action` 陣列，可指定一個或多個支持的 S3 API actions。


## S3 API 支持的 Actions

MinIO 策略文檔支持 [IAM S3 Action keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/list_amazons3.html#amazons3-actions-as-permissions) 的子集。

以下列出常見的 S3 操作的 actions:

**s3:***

所有 MinIO S3 操作的選擇器。將此操作應用於給定資源允許用戶針對該資源執行任何 S3 操作。

**s3:CreateBucket**

控制對 [CreateBucket](https://docs.aws.amazon.com/AmazonS3/latest/API/API_CreateBucket.html) S3 API 操作的訪問。

**s3:DeleteBucket**

控制對 [DeleteBucket](https://docs.aws.amazon.com/AmazonS3/latest/API/API_DeleteBucket.html) S3 API 操作的訪問。

**s3:ForceDeleteBucket**

控制對帶有 `x-minio-force-delete` 標誌的操作的 [DeleteBucket](https://docs.aws.amazon.com/AmazonS3/latest/API/API_DeleteBucket.html) S3 API 操作的訪問。刪除非空存儲桶時需要。

**s3:GetBucketLocation**

控制對 [GetBucketLocation](https://docs.aws.amazon.com/AmazonS3/latest/API/API_GetBucketLocation.html) S3 API 操作的訪問。

**s3:ListAllMyBuckets**

控制對 [ListBuckets](https://docs.aws.amazon.com/AmazonS3/latest/API/API_ListBuckets.html) S3 API 操作的訪問。

**s3:ListBucket**

控制對 [ListObjectsV2](https://docs.aws.amazon.com/AmazonS3/latest/API/API_ListObjectsV2.html) S3 API 操作的訪問。

**s3:PutObject**

控制對 [PutObject](https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutObject.html) S3 API 操作的訪問。

**s3:DeleteObject**

控制對 [DeleteObject](https://docs.aws.amazon.com/AmazonS3/latest/API/API_DeleteObject.html) S3 API 操作的訪問。

**s3:GetObject**

控制對 [GetObject](https://docs.aws.amazon.com/AmazonS3/latest/API/API_GetObject.html) S3 API 操作的訪問。

**s3:PutObjectTagging**

控制對 [PutObjectTagging](https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutObjectTagging.html) S3 API 操作的訪問。

**s3:DeleteObjectTagging**

控制對 [DeleteObjectTagging](https://docs.aws.amazon.com/AmazonS3/latest/API/API_DeleteObjectTagging.html) S3 API 操作的訪問。

**s3:GetObjectTagging**

控制對 [GetObjectTagging](https://docs.aws.amazon.com/AmazonS3/latest/API/API_GetObjectTagging.html) S3 API 操作的訪問。

### Bucket Configuration

**s3:GetBucketPolicy**

控制對 [GetBucketPolicy](https://docs.aws.amazon.com/AmazonS3/latest/API/API_GetBucketPolicy.html) S3 API 操作的訪問。

**s3:PutBucketPolicy**

控制對 [PutBucketPolicy](https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutBucketPolicy.html) S3 API 操作的訪問。

**s3:DeleteBucketPolicy**

控制對 [DeleteBucketPolicy](https://docs.aws.amazon.com/AmazonS3/latest/API/API_DeleteBucketPolicy.html) S3 API 操作的訪問。

**s3:GetBucketTagging**

控制對 [GetBucketTagging](https://docs.aws.amazon.com/AmazonS3/latest/API/API_GetBucketTagging.html) S3 API 操作的訪問。

**s3:PutBucketTagging**

控制對 [PutBucketTagging](https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutBucketTagging.html) S3 API 操作的訪問。

### Multipart Upload

**s3:AbortMultipartUpload**

控制對 [AbortMultipartUpload](https://docs.aws.amazon.com/AmazonS3/latest/API/API_AbortMultipartUpload.html) S3 API 操作的訪問。

**s3:ListMultipartUploadParts**

控制對 [ListParts](https://docs.aws.amazon.com/AmazonS3/latest/API/API_ListParts.html) S3 API 操作的訪問。

**s3:ListBucketMultipartUploads**

控制對 [ListMultipartUploads](https://docs.aws.amazon.com/AmazonS3/latest/API/API_ListMultipartUploads.html) S3 API 操作的訪問。

### Versioning and Retention

**s3:PutBucketVersioning**

控制對 [PutBucketVersioning](https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutBucketVersioning.html) S3 API 操作的訪問。

**s3:GetBucketVersioning**

控制對 [GetBucketVersioning](https://docs.aws.amazon.com/AmazonS3/latest/API/API_GetBucketVersioning.html) S3 API 操作的訪問。

**s3:DeleteObjectVersion**

控制對 [DeleteObjectVersion](https://docs.aws.amazon.com/AmazonS3/latest/API/API_DeleteObjectVersion.html) S3 API 操作的訪問。

**s3:DeleteObjectVersionTagging**

控制對 [DeleteObjectVersionTagging](https://docs.aws.amazon.com/AmazonS3/latest/API/API_DeleteObjectVersionTagging.html) S3 API 操作的訪問。

**s3:GetObjectVersion**

控制對 [GetObjectVersion](https://docs.aws.amazon.com/AmazonS3/latest/API/API_GetObjectVersion.html) S3 API 操作的訪問。

**s3:BypassGovernanceRetention**

控制對在 `GOVERNANCE` retention 模式下鎖定的 object 的 S3 API 操作的訪問：

- `PutObjectRetention`
- `PutObject`
- `DeleteObject`

有關更多信息，請參閱有關 [s3:BypassGovernanceRetention](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock-managing.html#object-lock-managing-bypass) 的 S3 文檔。

**s3:PutObjectRetention**

控制對 [PutObjectRetention](https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutObjectRetention.html) S3 API 操作的訪問。

**s3:GetObjectRetention**

控制對 [GetObjectRetention](https://docs.aws.amazon.com/AmazonS3/latest/API/API_GetObjectRetention.html) S3 API 操作的訪問。

**s3:GetObjectLegalHold**

控制對 [GetObjectLegalHold](https://docs.aws.amazon.com/AmazonS3/latest/API/API_GetObjectLegalHold.html) S3 API 操作的訪問。

**s3:PutObjectLegalHold**

控制對 [PutObjectLegalHold](https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutObjectLegalHold.html) S3 API 操作的訪問。

**s3:GetBucketObjectLockConfiguration**

控制對 [GetObjectLockConfiguration](https://docs.aws.amazon.com/AmazonS3/latest/API/API_GetObjectLockConfiguration.html) S3 API 操作的訪問。

**s3:PutBucketObjectLockConfiguration**

控制對 [PutObjectLockConfiguration](https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutObjectLockConfiguration.html) S3 API 操作的訪問。

### Bucket Notifications

**s3:GetBucketNotification**

控制對 [GetBucketNotification](https://docs.aws.amazon.com/AmazonS3/latest/API/API_GetBucketNotification.html) S3 API 操作的訪問。

**s3:PutBucketNotification**

控制對 [PutBucketNotification](https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutBucketNotification.html) S3 API 操作的訪問。

### Object Lifecycle Management

**s3:PutLifecycleConfiguration**

控制對 [PutLifecycleConfiguration](https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutBucketLifecycleConfiguration.html) S3 API 操作的訪問。

**s3:GetEncryptionConfiguration**

控制對 [GetEncryptionConfiguration](https://docs.aws.amazon.com/AmazonS3/latest/API/API_GetBucketEncryption.html) S3 API 操作的訪問。

### Bucket Replication

**s3:GetReplicationConfiguration**

控制對 [GetBucketReplication](https://docs.aws.amazon.com/AmazonS3/latest/API/API_GetBucketReplication.html) S3 API 操作的訪問。

**s3:PutReplicationConfiguration**

控制對 [PutBucketReplication](https://docs.aws.amazon.com/AmazonS3/latest/API/PutBucketReplication.html) S3 API 操作的訪問。

## 支持 S3 Policy Condition Keys

MinIO 策略文檔支持 IAM [conditional statements](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_condition.html)。

每個條件元素由 [operators](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_condition_operators.html) 和 condition keys 組成。 MinIO 支持 IAM 條件鍵的子集。有關任何列出的條件鍵的完整信息，請參閱 [IAM Condition Element Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_condition.html)。

MinIO 支持所有受支持操作的以下條件鍵：

- `aws:Referer`
- `aws:SourceIp`
- `aws:UserAgent`
- `aws:SecureTransport`
- `aws:CurrentTime`
- `aws:EpochTime`
- `aws:PrincipalType`
- `aws:userid`
- `aws:username`
- `x-amz-content-sha256`

下表列出了特定操作的其他支持的條件鍵：

|Action Key |Condition Keys|
|-----------|--------------|
|`s3:GetObject`|`x-amz-server-side-encryption`<br/>`x-amz-server-side-encryption-customer-algorithm`<br/>`s3:ExistingObjectTag/<key>`|
|`s3:ListBucket`|`prefix`<br/>`delimiter`<br/>`max-keys`|
|`s3:PutObject`|`x-amz-copy-source`<br/>`x-amz-server-side-encryption`<br/>`x-amz-server-side-encryption-customer-algorithm`<br/>`x-amz-metadata-directive`<br/>`x-amz-storage-class`<br/>`object-lock-retain-until-date`<br/>`object-lock-mode`<br/>`object-lock-legal-hold`<br/>`s3:ExistingObjectTag/<key>`|
|`s3:PutObjectRetention`|`x-amz-object-lock-remaining-retention-days`<br/>`x-amz-object-lock-retain-until-date`<br/>`x-amz-object-lock-mode`|
|`s3:PutObjectLegalHold`|`object-lock-legal-hold`|
|`s3:BypassGovernanceRetention`|`object-lock-remaining-retention-days`<br/>`object-lock-retain-until-date`<br/>`object-lock-mode`<br/>`object-lock-legal-hold`|
|`s3:GetObjectVersion`|`versionid`|
|`s3:DeleteObjectVersion`|`versionid`|
|`s3:PutObjectTagging`|`s3:ExistingObjectTag/<key>`|
|`s3:DeleteObjectTagging`|`s3:ExistingObjectTag/<key>`|

## Policy 變數

MinIO 支持使用 policy variable 自動將經過身份驗證的用戶和/或操作的上下文替換為用戶分配的一個或多個策略。使用 `${POLICYVARIABLE}` 格式將策略的變量指定為條件或資源定義的一部分。 MinIO 策略變量的功能與 AWS IAM 策略元素類似：variables 和 tags。

每個 MinIO 身份提供者都支持自己的一組 policy variable：

### MinIO Policy 變數

下表包含推薦的策略變量列表，用於授權 MinIO 管理的用戶：

|Variable|Description|
|--------|-----------|
|`aws:referrer`|The referrer in the HTTP header for the authenticated API call.|
|`aws:SourceIp`|The source IP in the HTTP header for the authenticated API call.|
|`aws:username`|The name of the user associated with the authenticated API call.|

例如，以下策略使用變量來替換經過身份驗證的用戶的用戶名作為資源字段的一部分，以便用戶只能訪問與其用戶名匹配的前綴：

```json hl_lines="8 16"
{
"Version": "2012-10-17",
"Statement": [
      {
         "Action": ["s3:ListBucket"],
         "Effect": "Allow",
         "Resource": ["arn:aws:s3:::mybucket"],
         "Condition": {"StringLike": {"s3:prefix": ["${aws:username}/*"]}}
      },
      {
         "Action": [
         "s3:GetObject",
         "s3:PutObject"
         ],
         "Effect": "Allow",
         "Resource": ["arn:aws:s3:::mybucket/${aws:username}/*"]
      }
   ]
}
```

MinIO 將資源字段中的 `${aws:username}` 變量替換為用戶名。然後，MinIO 評估該策略並授予或撤銷對所請求的 API 和資源的訪問權限。

### OpenID Policy 變數

下表包含支持的策略變量的列表，用於授權 OIDC 管理的用戶。

每個變量對應於作為經過身份驗證的用戶的 JWT 令牌的一部分返回的 claim：

|Variable	|Description|
|---------|-----------|
|`jwt:sub`	|Returns the `sub` claim for the user.|
|`jwt:iss`	|Returns the `Issuer Identifier` claim from the ID token.|
|`jwt:aud`	|Returns the `Audience` claim from the ID token.|
|`jwt:jti`	|Returns the `JWT ID` claim from the client authentication information.|
|`jwt:upn`	|Returns the `User Principal Name` claim from the client authentication information.|
|`jwt:name`	|Returns the `name` claim for the user.|
|`jwt:groups`	|Returns the `groups` claim for the user.|
|`jwt:given_name`	|Returns the `given_name` claim for the user.|
|`jwt:family_name`	|Returns the `family_name` claim for the user.|
|`jwt:middle_name`	|Returns the `middle_name` claim for the user.|
|`jwt:nickname`	|Returns the `nickname` claim for the user.|
|`jwt:preferred_username`	|Returns the `preferred_username` claim for the user.|
|`jwt:profile`	|Returns the `profile` claim for the user.|
|`jwt:picture`	|Returns the `picture` claim for the user.|
|`jwt:website`	|Returns the `website` claim for the user.|
|`jwt:email`	|Returns the `email` claim for the user.|
|`jwt:gender`	|Returns the `gender` claim for the user.|
|`jwt:birthdate`	|Returns the `birthdate` claim for the user.|
|`jwt:phone_number`	|Returns the `phone_number` claim for the user.|
|`jwt:address`	|Returns the `address` claim for the user.|
|`jwt:scope`	|Returns the `scope` claim for the user.|
|`jwt:client_id`	|Returns the `client_id` claim for the user.|

有關這些範圍的更多信息，請參閱 OpenID Connect Core 1.0 文檔。您選擇的 OIDC 提供商可能有更具體的文檔。

例如，以下策略使用變量來替換經過身份驗證的用戶的 `PreferredUsername` 作為資源字段的一部分，以便用戶只能訪問與其用戶名匹配的前綴：

```json hl_lines="8 16"
{
"Version": "2012-10-17",
"Statement": [
      {
         "Action": ["s3:ListBucket"],
         "Effect": "Allow",
         "Resource": ["arn:aws:s3:::mybucket"],
         "Condition": {"StringLike": {"s3:prefix": ["${jwt:PreferredUsername}/*"]}}
      },
      {
         "Action": [
         "s3:GetObject",
         "s3:PutObject"
         ],
         "Effect": "Allow",
         "Resource": ["arn:aws:s3:::mybucket/${jwt:PreferredUsername}/*"]
      }
   ]
}
```

MinIO 將 Resource 字段中的 `${jwt:PreferredUsername}` 變量替換為 JWT 令牌中的 `PreferredUsername` 的值。然後，MinIO 評估該策略並授予或撤銷對所請求的 API 和資源的訪問權限。

### Active Directory/LDAP Policy 變數

下表包含用於授權 AD/LDAP 用戶的受支持策略變量的列表：

|Variable	|Description|
|---------|-----------|
|`ldap:username`|The simple username (name) for the authenticated user.This is distinct from the user’s DistinguishedName or CommonName.|
|`ldap:user`|The Distinguished Name (DN) used by the authenticated user.|
|`ldap:groups`|The Group Distinguished Name (DN) for the authenticated user.|

例如，以下策略使用變量來替換經過身份驗證的用戶的名稱作為資源字段的一部分，以便用戶只能訪問與其名稱匹配的前綴：

```json hl_lines="8 16"
{
"Version": "2012-10-17",
"Statement": [
      {
         "Action": ["s3:ListBucket"],
         "Effect": "Allow",
         "Resource": ["arn:aws:s3:::mybucket"],
         "Condition": {"StringLike": {"s3:prefix": ["${ldap:username}/*"]}}
      },
      {
         "Action": [
         "s3:GetObject",
         "s3:PutObject"
         ],
         "Effect": "Allow",
         "Resource": ["arn:aws:s3:::mybucket/${ldap:username}/*"]
      }
   ]
}
```

MinIO 將 Resource 字段中的 `${ldap:username}` 變量替換為經過身份驗證的用戶名的值。然後，MinIO 評估該策略並授予或撤銷對所請求的 API 和資源的訪問權限。

