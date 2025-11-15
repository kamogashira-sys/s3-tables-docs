# S3 TablesとApache Icebergテーブルフォーマットの関係

## 調査日時
2025-11-15 09:13

## 【結論】

**S3 TablesはApache Icebergフォーマットを使用している**

S3 Tablesは、Apache Icebergテーブルフォーマットをネイティブにサポートする専用のストレージサービスです。

## 【根拠】

### 1. S3 TablesはIcebergフォーマットを使用

AWS公式ドキュメントより：

> Table buckets support storing tables in the Apache Iceberg format. Using standard SQL statements, you can query your tables with query engines that support Iceberg, such as Amazon Athena, Amazon Redshift, and Apache Spark.

> Tables in your table buckets are stored in Apache Iceberg format. You can query these tables using standard SQL in query engines that support Iceberg.

**出典**: [Working with Amazon S3 Tables and table buckets](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables.html)

### 2. Iceberg REST API仕様への準拠

S3 TablesはApache Iceberg REST Catalog Open API仕様を実装しています：

> The endpoint implements a set of standardized Iceberg REST APIs specified in the Apache Iceberg REST Catalog Open API specification. The endpoint works by translating Iceberg REST API operations into corresponding S3 Tables operations.

**出典**: [Accessing tables using the Amazon S3 Tables Iceberg REST endpoint](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-integrating-open-source.html)

### 3. Iceberg REST Endpointの提供

S3 TablesはIceberg REST Endpointを提供し、標準的なIcebergクライアントからアクセス可能：

```
https://s3tables.<REGION>.amazonaws.com/iceberg
```

サポートされるIceberg REST API操作：
- `getConfig`
- `listNamespaces`, `createNamespace`, `loadNamespaceMetadata`, `dropNamespace`
- `listTables`, `createTable`, `loadTable`, `updateTable`, `dropTable`, `renameTable`
- `tableExists`, `namespaceExists`

**出典**: [Accessing tables using the Amazon S3 Tables Iceberg REST endpoint](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-integrating-open-source.html)

### 4. Icebergクライアントライブラリとの互換性

PyIcebergやApache Sparkなどの標準Icebergクライアントから直接アクセス可能：

**PyIcebergの例**:
```python
rest_catalog = load_catalog(
  catalog_name,
  **{
    "type": "rest",    
    "warehouse":"arn:aws:s3tables:<Region>:<accountID>:bucket/<bucketname>",
    "uri": "https://s3tables.<Region>.amazonaws.com/iceberg",
    "rest.sigv4-enabled": "true",
    "rest.signing-name": "s3tables",
    "rest.signing-region": "<Region>"
  }
)
```

**出典**: [Accessing tables using the Amazon S3 Tables Iceberg REST endpoint](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-integrating-open-source.html)

## 【S3 TablesとIcebergの関係性】

### アーキテクチャ上の位置づけ

```
┌─────────────────────────────────────────────────┐
│         Query Engines (Athena, Spark, etc)      │
└────────────────┬────────────────────────────────┘
                 │
                 │ Iceberg REST API / SQL
                 │
┌────────────────▼────────────────────────────────┐
│         S3 Tables (Iceberg REST Endpoint)       │
│  - Iceberg REST API実装                         │
│  - メタデータ管理                                │
│  - 自動最適化                                    │
└────────────────┬────────────────────────────────┘
                 │
                 │ Apache Iceberg Format
                 │
┌────────────────▼────────────────────────────────┐
│         Table Bucket (S3 Storage)               │
│  - Icebergフォーマットでデータ保存               │
│  - メタデータファイル                            │
│  - データファイル (Parquet等)                    │
└─────────────────────────────────────────────────┘
```

### S3 TablesがIcebergを採用する理由

1. **スキーマ進化 (Schema Evolution)**
   - テーブル構造の変更が容易
   - クエリの書き換えやデータ再構築が不要

2. **パーティション進化 (Partition Evolution)**
   - パーティション戦略の変更が可能
   - データの再編成なしで最適化

3. **トランザクションサポート**
   - ACID特性の保証
   - データの一貫性と信頼性

4. **タイムトラベルクエリ**
   - 過去のデータバージョンへのアクセス
   - データ変更の追跡とロールバック

**出典**: [Working with Amazon S3 Tables and table buckets](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables.html)

## 【注意点・例外】

### 1. S3 Tables固有の制限事項

#### 単一レベルのネームスペースのみサポート
> S3 Tables only supports single-level namespaces.

Icebergは多階層ネームスペースをサポートしますが、S3 Tablesは単一レベルのみです。

#### CreateTable APIの制限
> The `stage-create` option is not supported for this operation, and results in a `400 Bad Request` error. This means you cannot create a table from query results using `CREATE TABLE AS SELECT` (CTAS).

#### DeleteTable APIの制限
> You can only drop tables with purge enabled. Dropping tables with `purge=false` is not supported and results in a `400 Bad Request` error.

**出典**: [Accessing tables using the Amazon S3 Tables Iceberg REST endpoint](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-integrating-open-source.html)

### 2. S3 Tables独自の機能

S3 TablesはIcebergフォーマットを使用しながら、AWS独自の機能を追加：

- **自動メンテナンス**: コンパクション、スナップショット管理、未参照ファイル削除
- **s3tablesネームスペース**: S3とは異なる独自のIAMネームスペース
- **AWS Glue Data Catalogとの統合**: Lake Formationによる細かいアクセス制御

**出典**: [Working with Amazon S3 Tables and table buckets](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables.html)

## 【出典】

1. [Working with Amazon S3 Tables and table buckets - Amazon Simple Storage Service](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables.html) - アクセス日: 2025-11-15
2. [Accessing tables using the Amazon S3 Tables Iceberg REST endpoint - Amazon Simple Storage Service](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-integrating-open-source.html) - アクセス日: 2025-11-15
3. [Apache Iceberg REST Catalog Open API specification](https://github.com/apache/iceberg/blob/main/open-api/rest-catalog-open-api.yaml) - 参照元: AWS公式ドキュメント

## 【確実性：高】

AWS公式ドキュメントに明確に記載されており、技術仕様も詳細に文書化されています。
