# Amazon S3 Tables - リファレンス


## 目次

- [概要](#概要)
- [サービスクォータ](#サービスクォータ)
- [ARN形式](#arn形式)
  - [Table Bucket](#table-bucket)
  - [Table](#table)
  - [Catalog](#catalog)
- [IAMアクション](#iamアクション)
  - [Table Bucketレベル](#table-bucketレベル)
  - [Tableレベル](#tableレベル)
- [CLI コマンド](#cli-コマンド)
  - [Table Bucket操作](#table-bucket操作)
  - [Table操作](#table操作)
- [メンテナンス設定パラメータ](#メンテナンス設定パラメータ)
  - [Compaction](#compaction)
  - [Snapshot Management](#snapshot-management)
  - [Unreferenced File Removal](#unreferenced-file-removal)
- [Firehose設定パラメータ](#firehose設定パラメータ)
  - [バッファ設定](#バッファ設定)
  - [リトライ設定](#リトライ設定)
  - [スループット制限](#スループット制限)
- [Spark設定](#spark設定)
  - [カタログ設定](#カタログ設定)
  - [パフォーマンス設定](#パフォーマンス設定)
- [Athena設定](#athena設定)
  - [Icebergテーブル作成](#icebergテーブル作成)
- [エンドポイント](#エンドポイント)
  - [Dual-Stack](#dual-stack)
  - [VPC Endpoint](#vpc-endpoint)
- [サポートリージョン](#サポートリージョン)
- [エラーコード](#エラーコード)
  - [Firehoseエラー](#firehoseエラー)
- [関連リンク](#関連リンク)
  - [AWS公式ドキュメント](#aws公式ドキュメント)
  - [Apache Iceberg](#apache-iceberg)
  - [関連ドキュメント](#関連ドキュメント)
- [更新履歴](#更新履歴)

## 概要

本ドキュメントでは、Amazon S3 Tablesの技術仕様、API、CLI、設定パラメータをまとめています。

---

## サービスクォータ

| リソース | デフォルト値 | 調整可能 |
|---------|------------|---------|
| Table Buckets（リージョンごと） | 10 | ✓ |
| Namespaces（Table Bucketごと） | 10,000 | ✓ |
| Tables（Table Bucketごと） | 10,000 | ✓ |

---

## ARN形式

### Table Bucket
```text
arn:aws:s3tables:region:account-id:bucket/table-bucket-name
```

### Table
```text
arn:aws:s3tables:region:account-id:bucket/table-bucket-name/table/table-id
```

### Catalog
```text
arn:aws:glue:region:account-id:catalog/s3tablescatalog/table-bucket-name
```

---

## IAMアクション

### Table Bucketレベル
- `s3tables:CreateTableBucket`
- `s3tables:GetTableBucket`
- `s3tables:ListTableBuckets`
- `s3tables:DeleteTableBucket`
- `s3tables:PutTableBucketPolicy`
- `s3tables:GetTableBucketPolicy`
- `s3tables:DeleteTableBucketPolicy`

### Tableレベル
- `s3tables:CreateTable`
- `s3tables:GetTable`
- `s3tables:ListTables`
- `s3tables:DeleteTable`
- `s3tables:GetTableData`
- `s3tables:PutTableData`
- `s3tables:UpdateTableMetadataLocation`

---

## CLI コマンド

### Table Bucket操作

```bash
# 作成
aws s3tables create-table-bucket \
  --name my-table-bucket \
  --region us-east-1

# 取得
aws s3tables get-table-bucket \
  --table-bucket-arn arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket

# 一覧
aws s3tables list-table-buckets

# 削除
aws s3tables delete-table-bucket \
  --table-bucket-arn arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket
```

### Table操作

```bash
# 作成（SQLで実行）
# CLIでの直接作成は非サポート

# 取得
aws s3tables get-table \
  --table-bucket-arn arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket \
  --namespace my_namespace \
  --name orders

# 一覧
aws s3tables list-tables \
  --table-bucket-arn arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket \
  --namespace my_namespace

# 削除
aws s3tables delete-table \
  --table-bucket-arn arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket \
  --namespace my_namespace \
  --name orders
```

---

## メンテナンス設定パラメータ

### Compaction

| パラメータ | デフォルト値 | 最小値 | 最大値 |
|-----------|------------|--------|--------|
| targetFileSizeMB | 512 | 64 | 1024 |

### Snapshot Management

| パラメータ | デフォルト値 | 最小値 |
|-----------|------------|--------|
| minimumSnapshots | 1 | 1 |
| maximumSnapshotAge | 120時間 | 1時間 |

### Unreferenced File Removal

| パラメータ | デフォルト値 | 最小値 |
|-----------|------------|--------|
| unreferencedDays | 3日 | 1日 |
| nonCurrentDays | 10日 | 1日 |

---

## Firehose設定パラメータ

### バッファ設定

| パラメータ | 最小値 | 最大値 | デフォルト値 |
|-----------|--------|--------|------------|
| BufferSizeInMBs | 1 | 128 | 128 |
| BufferIntervalInSeconds | 0 | 900 | 900 |

### リトライ設定

| パラメータ | 最小値 | 最大値 | デフォルト値 |
|-----------|--------|--------|------------|
| DurationInSeconds | 0 | 7200 | 300 |

### スループット制限

| リージョン | 制限 |
|-----------|------|
| US East (N. Virginia) | 5 MiB/秒 |
| US West (Oregon) | 5 MiB/秒 |
| Europe (Ireland) | 5 MiB/秒 |
| その他 | 1 MiB/秒 |

---

## Spark設定

### カタログ設定

```python
spark.sql.catalog.s3tablescatalog = software.amazon.s3tables.iceberg.S3TablesCatalog
spark.sql.catalog.s3tablescatalog.warehouse = arn:aws:s3tables:region:account-id:bucket/table-bucket-name
```

### パフォーマンス設定

```python
spark.sql.adaptive.enabled = true
spark.sql.adaptive.coalescePartitions.enabled = true
spark.sql.files.maxPartitionBytes = 134217728  # 128MB
```

---

## Athena設定

### Icebergテーブル作成

```sql
CREATE TABLE s3tablescatalog.my_namespace.orders (
    order_id BIGINT,
    customer_id BIGINT,
    order_date DATE,
    order_amount DECIMAL(10,2)
)
USING iceberg
PARTITIONED BY (bucket(16, customer_id), days(order_date))
LOCATION 'arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket';
```

---

## エンドポイント

### Dual-Stack
```
s3tables.region.api.aws
```

### VPC Endpoint
- **Gateway Endpoint**: サポート
- **Interface Endpoint**: サポート

---

## サポートリージョン

主要リージョン:
- US East (N. Virginia) - us-east-1
- US West (Oregon) - us-west-2
- Europe (Ireland) - eu-west-1
- Asia Pacific (Tokyo) - ap-northeast-1

最新のリージョン一覧は[AWS公式ドキュメント](https://docs.aws.amazon.com/general/latest/gr/s3.html#s3_region)を参照してください。

---

## エラーコード

### Firehoseエラー

| エラーコード | 説明 | 対処方法 |
|------------|------|---------|
| Iceberg.NoSuchTable | テーブルが存在しない | テーブル名を確認 |
| Iceberg.InvalidTableName | テーブル名が無効 | 小文字のテーブル名を使用 |
| S3.AccessDenied | S3アクセス拒否 | IAM権限を確認 |
| Glue.AccessDenied | Glueアクセス拒否 | Glue権限を確認 |

---

## 関連リンク

### AWS公式ドキュメント
- [S3 Tables User Guide](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables.html)
- [S3 Tables API Reference](https://docs.aws.amazon.com/AmazonS3/latest/API/API_Operations_Amazon_S3_Tables.html)
- [AWS CLI Reference](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/s3tables/index.html)

### Apache Iceberg
- [Apache Iceberg Documentation](https://iceberg.apache.org/docs/latest/)
- [Iceberg Spec](https://iceberg.apache.org/spec/)

### 関連ドキュメント
- [制限事項](../01-getting-started/04-limitations.md): 制限事項
- [セキュリティベストプラクティス](../03-security/01-security-best-practices.md): セキュリティベストプラクティス
- [用語集](01-glossary.md): 用語集

## 更新履歴
- 2025-11-15: 初版作成
