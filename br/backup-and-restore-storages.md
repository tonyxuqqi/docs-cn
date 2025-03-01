---
title: 备份存储
summary: 了解 BR 支持的备份存储服务的 URL 格式、鉴权方案和使用方式。
aliases: ['/docs-cn/dev/br/backup-and-restore-storages/','/zh/tidb/dev/backup-storage-S3/','/zh/tidb/dev/backup-storage-azblob/','/zh/tidb/dev/backup-storage-gcs/','/zh/tidb/dev/external-storage/']
---

# 备份存储

TiDB 支持 Amazon S3、Google Cloud Storage (GCS)、Azure Blob Storage 和 NFS 作为备份恢复的存储。具体来说，可以在 `br` 的 `--storage` 或 `-s` 选项中指定备份存储的 URL。本文介绍不同外部存储服务中 [URL 的定义格式](#url-格式)、存储过程中的[鉴权方案](#鉴权)以及[存储服务端加密](#存储服务端加密)。

## BR 向 TiKV 发送凭证

| 命令行参数 | 描述 | 默认值 |
|:----------|:-------|:-------|
| `--send-credentials-to-tikv` | 是否将 BR 获取到的权限凭证发送给 TiKV。 | `true` |

在默认情况下，使用 Amazon S3、Google Cloud Storage (GCS)、Azure Blob Storage 存储时，BR 会将凭证发送到每个 TiKV 节点，以减少设置的复杂性。该操作由参数 `--send-credentials-to-tikv`（或简写为 `-c`）控制。

但是，这个操作不适合云端环境，如果采用了 IAM Role 方式授权，那么每个节点都有自己的角色和权限。在这种情况下，你需要设置 `--send-credentials-to-tikv=false`（或简写为 `-c=0`）来禁止发送凭证：

```bash
./br backup full -c=0 -u pd-service:2379 --storage 's3://bucket-name/prefix'
```

使用 SQL 进行[备份](/sql-statements/sql-statement-backup.md)[恢复](/sql-statements/sql-statement-restore.md)时，可加上 `SEND_CREDENTIALS_TO_TIKV = FALSE` 选项：

```sql
BACKUP DATABASE * TO 's3://bucket-name/prefix' SEND_CREDENTIALS_TO_TIKV = FALSE;
```

## URL 格式

### 格式说明

本部分介绍存储服务的 URL 格式：

```shell
[scheme]://[host]/[path]?[parameters]
```

<SimpleTab groupId="storage">
<div label="Amazon S3" value="amazon">

- `scheme`：`s3`
- `host`：`bucket name`
- `parameters`：

    - `access-key`：访问密钥
    - `secret-access-key`：秘密访问密钥
    - `use-accelerate-endpoint`：是否在 Amazon S3 上使用加速端点，默认为 `false`
    - `endpoint`：Amazon S3 兼容服务自定义端点的 URL，例如 `<https://s3.example.com/>`
    - `force-path-style`：使用路径类型 (path-style)，而不是虚拟托管类型 (virtual-hosted-style)，默认为 `true`
    - `storage-class`：上传对象的存储类别，例如 `STANDARD`、`STANDARD_IA`
    - `sse`：加密上传的服务端加密算法，可以设置为空、`AES256` 或 `aws:kms`
    - `sse-kms-key-id`：如果 `sse` 设置为 `aws:kms`，则使用该参数指定 KMS ID
    - `acl`：上传对象的标准 ACL (Canned ACL)，例如 `private`、`authenticated-read`

</div>
<div label="GCS" value="gcs">

- `scheme`：`gcs` 或 `gs`
- `host`：`bucket name`
- `parameters`：

    - `credentials-file`：迁移工具节点上凭证 JSON 文件的路径
    - `storage-class`：上传对象的存储类别，例如 `STANDARD` 或 `COLDLINE`
    - `predefined-acl`：上传对象的预定义 ACL，例如 `private` 或 `project-private`

</div>
<div label="Azure Blob Storage" value="azure">

- `scheme`：`azure` 或 `azblob`
- `host`：`container name`
- `parameters`：

    - `account-name`：存储账户名
    - `account-key`：访问密钥
    - `access-tier`：上传对象的存储类别，例如 `Hot`、`Cool`、`Archive`，默认为 `Hot`

</div>
</SimpleTab>

### URL 示例

本部分示例以 `host`（上表中 `bucket name`、`container name`）为 `external` 为例进行介绍。

<SimpleTab groupId="storage">
<div label="Amazon S3" value="amazon">

**备份快照数据到 Amazon S3**

```shell
./br backup full -u "${PD_IP}:2379" \
--storage "s3://external/backup-20220915?access-key=${access-key}&secret-access-key=${secret-access-key}"
```

**从 Amazon S3 恢复快照备份数据**

```shell
./br restore full -u "${PD_IP}:2379" \
--storage "s3://external/backup-20220915?access-key=${access-key}&secret-access-key=${secret-access-key}"
```

</div>
<div label="GCS" value="gcs">

**备份快照数据到 GCS**

```shell
./br backup full --pd "${PD_IP}:2379" \
--storage "gcs://external/backup-20220915?credentials-file=${credentials-file-path}"
```

**从 GCS 恢复快照备份数据**

```shell
./br restore full --pd "${PD_IP}:2379" \
--storage "gcs://external/backup-20220915?credentials-file=${credentials-file-path}"
```

</div>
<div label="Azure Blob Storage" value="azure">

**备份快照数据到 Azure Blob Storage**

```shell
./br backup full -u "${PD_IP}:2379" \
--storage "azure://external/backup-20220915?account-name=${account-name}&account-key=${account-key}"
```

**从 Azure Blob Storage 恢复快照备份数据中 `test` 数据库**

```shell
./br restore db --db test -u "${PD_IP}:2379" \
--storage "azure://external/backup-20220915account-name=${account-name}&account-key=${account-key}"
```

</div>
</SimpleTab>

## 鉴权

将数据存储到云服务存储系统时，根据云服务供应商的不同，需要设置不同的鉴权参数。本部分介绍使用 Amazon S3、GCS 及 Azure Blob Storage 时所用存储服务的鉴权方式以及如何配置访问相应存储服务的账户。

<SimpleTab groupId="storage">
<div label="Amazon S3" value="amazon">

在备份之前，需要为 br 命令行工具访问 Amazon S3 中的备份目录设置相应的访问权限：

- 备份时 TiKV 和 br 命令行工具需要的访问备份数据目录的最小权限：`s3:ListBucket`、`s3:PutObject` 和 `s3:AbortMultipartUpload`。
- 恢复时 TiKV 和 br 命令行工具需要的访问备份数据目录的最小权限：`s3:ListBucket` 和 `s3:GetObject`。

如果你还没有创建备份数据保存目录，可以参考[创建存储桶](https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/user-guide/create-bucket.html)在指定的区域中创建一个 S3 存储桶。如果需要使用文件夹，可以参考[使用文件夹在 Amazon S3 控制台中组织对象](https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/user-guide/create-folder.html)在存储桶中创建一个文件夹。

配置访问 Amazon S3 的账户可以通过以下两种方式：

- 方式一：指定访问密钥

    如果指定访问密钥和秘密访问密钥，将按照指定的访问密钥和秘密访问密钥进行鉴权。除了在 URL 中指定密钥外，还支持以下方式：

    - br 命令行工具读取 `$AWS_ACCESS_KEY_ID` 和 `$AWS_SECRET_ACCESS_KEY` 环境变量
    - br 命令行工具读取 `$AWS_ACCESS_KEY` 和 `$AWS_SECRET_KEY` 环境变量
    - br 命令行工具读取共享凭证文件，路径由 `$AWS_SHARED_CREDENTIALS_FILE` 环境变量指定
    - br 命令行工具读取共享凭证文件，路径为 `~/.aws/credentials`

- 方式二：基于 IAM Role 进行访问

    为运行 TiKV 和 br 命令行工具的 EC2 实例关联一个配置了访问 S3 访问权限的 IAM role。正确设置后，br 命令行工具可以直接访问对应的 S3 中的备份目录，而不需要额外的设置。

    ```shell
    br backup full --pd "${PD_IP}:2379" \
    --storage "s3://${host}/${path}"
    ```

</div>
<div label="GCS" value="gcs">

配置访问 GCS 的账户可以通过指定访问密钥的方式。如果指定了 `credentials-file` 参数，将按照指定的 `credentials-file` 进行鉴权。除了在 URL 中指定密钥文件外，还支持以下方式：

- br 命令行工具读取位于 `$GOOGLE_APPLICATION_CREDENTIALS` 环境变量所指定路径的文件内容
- br 命令行工具读取位于 `~/.config/gcloud/application_default_credentials.json` 的文件内容
- 在 GCE 或 GAE 中运行时，从元数据服务器中获取的凭证

</div>
<div label="Azure Blob Storage" value="azure">

- 方式一：指定访问密钥

    在 URL 配置 `account-name` 和 `account-key`，则使用该参数指定的密钥。除了在 URL 中指定密钥文件外，还支持 br 命令行工具读取 `$AZURE_STORAGE_KEY` 的方式。

- 方式二：使用 Azure AD 备份恢复

    在 br 命令行工具运行环境配置环境变量 `$AZURE_CLIENT_ID`、`$AZURE_TENANT_ID` 和 `$AZURE_CLIENT_SECRET`。

    - 当集群使用 TiUP 启动时，TiKV 会使用 systemd 服务。以下示例介绍如何为 TiKV 配置上述三个环境变量：

        > **注意：**
        >
        > 该流程在第 3 步中需要重启 TiKV。如果你的集群不适合重启，请使用**指定访问密钥的方式**进行备份恢复。

        1. 假设该节点上 TiKV 端口为 `24000`，即 systemd 服务名为 `tikv-24000`：

            ```shell
            systemctl edit tikv-24000
            ```

        2. 编辑三个环境变量的信息：

            ```
            [Service]
            Environment="AZURE_CLIENT_ID=aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"
            Environment="AZURE_TENANT_ID=aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"
            Environment="AZURE_CLIENT_SECRET=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
            ```

        3. 重新加载配置并重启 TiKV：

            ```shell
            systemctl daemon-reload
            systemctl restart tikv-24000
            ```

    - 为命令行启动的 TiKV 和 br 命令行工具配置 Azure AD 的信息，只需要确定运行环境中存在 `$AZURE_CLIENT_ID`、`$AZURE_TENANT_ID` 和 `$AZURE_CLIENT_SECRET`。通过运行下列命令行，可以确认 br 命令行工具和 TiKV 运行环境中是否存在这三个环境变量：

        ```shell
        echo $AZURE_CLIENT_ID
        echo $AZURE_TENANT_ID
        echo $AZURE_CLIENT_SECRET
        ```

    - 使用 br 命令行工具将数据备份至 Azure Blob Storage：

        ```shell
        ./br backup full -u "${PD_IP}:2379" \
        --storage "azure://external/backup-20220915?account-name=${account-name}"
        ```

</div>
</SimpleTab>

## 存储服务端加密

### Amazon S3 存储服务端加密备份数据

TiDB 备份恢复功能支持对备份到 Amazon S3 的数据进行 S3 服务端加密 (SSE)。S3 服务端加密也支持使用用户自行创建的 AWS KMS 密钥，详细信息请参考 [BR S3 服务端加密](/encryption-at-rest.md#br-s3-服务端加密)。

## 存储服务其他功能支持

TiDB 备份恢复功能从 v6.3.0 支持 AWS S3 Object Lock 功能。你可以在 AWS 中开启 [S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html) 功能来防止备份数据写入后被修改或者删除。
