# Apache IcebergライブラリとS3の関係

## 調査日時
2025-11-15 09:17

## 【結論】

**AWSはApache Icebergライブラリのエコシステムを活用し、S3 Tables専用のカタログ実装を提供**

Apache IcebergライブラリはS3上でのテーブル管理の基盤技術であり、S3 TablesはIcebergライブラリと統合するための専用カタログ実装（Amazon S3 Tables Catalog for Apache Iceberg）を提供しています。

## 【根拠】

### 1. Amazon S3 Tables Catalog for Apache Iceberg

AWSはApache Iceberg用の専用カタログライブラリを提供：

> Amazon S3 Tables Catalog for Apache Iceberg is an open source library hosted by AWS Labs. It works by translating Apache Iceberg operations in your query engines (such as table discovery, metadata updates, and adding or removing tables) into S3 Tables API operations.

**配布形式**:
- Maven JAR: `s3-tables-catalog-for-iceberg.jar`
- GitHubリポジトリ: https://github.com/awslabs/s3-tables-catalog
- Maven Central: `software.amazon.s3tables:s3-tables-catalog-for-iceberg`

**出典**: [Accessing Amazon S3 tables with the Amazon S3 Tables Catalog for Apache Iceberg](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-client-catalog.html)

### 2. Icebergライブラリの役割

Apache Icebergライブラリは以下の機能を提供：

1. **テーブルメタデータ管理**
   - スキーマ定義
   - パーティション情報
   - スナップショット管理

2. **クエリエンジン統合**
   - Spark、Athena、Redshift等との統合
   - 標準SQLインターフェース

3. **データファイル管理**
   - Parquet、ORC、Avroファイルの管理
   - ファイルレベルの統計情報

**出典**: 
- [Query Apache Iceberg tables - Amazon Athena](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg.html)
- [Creating Apache Iceberg tables - AWS Lake Formation](https://docs.aws.amazon.com/lake-formation/latest/dg/creating-iceberg-tables.html)

### 3. S3上でのIcebergテーブル管理パターン

#### パターン1: S3 Tables（マネージド）

```
┌─────────────────────────────────────────┐
│      Apache Spark / Query Engine       │
└──────────────┬──────────────────────────┘
               │
               │ Iceberg API
               │
┌──────────────▼──────────────────────────┐
│  S3 Tables Catalog for Iceberg         │
│  (software.amazon.s3tables.iceberg)    │
└──────────────┬──────────────────────────┘
               │
               │ S3 Tables API
               │
┌──────────────▼──────────────────────────┐
│         S3 Tables Service              │
│  - 自動メンテナンス                     │
│  - メタデータ管理                       │
└──────────────┬──────────────────────────┘
               │
               │ Iceberg Format
               │
┌──────────────▼──────────────────────────┐
│         Table Bucket (S3)              │
└─────────────────────────────────────────┘
```

#### パターン2: セルフマネージドIceberg on S3

```
┌─────────────────────────────────────────┐
│      Apache Spark / Query Engine       │
└──────────────┬──────────────────────────┘
               │
               │ Iceberg API
               │
┌──────────────▼──────────────────────────┐
│    Iceberg Catalog (Glue/Hive/REST)    │
│    (org.apache.iceberg.*)              │
└──────────────┬──────────────────────────┘
               │
               │ 直接S3アクセス
               │
┌──────────────▼──────────────────────────┐
│    S3 General Purpose Bucket           │
│  - 手動メンテナンス必要                 │
│  - メタデータ管理はカタログ側           │
└─────────────────────────────────────────┘
```

**出典**: [Accessing Amazon S3 tables with the Amazon S3 Tables Catalog for Apache Iceberg](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-client-catalog.html)

### 4. S3 TablesでのIcebergライブラリ利用

#### Sparkセッションの初期化例

```bash
spark-shell \
  --packages org.apache.iceberg:iceberg-spark-runtime-3.5_2.12:1.6.1,\
software.amazon.s3tables:s3-tables-catalog-for-iceberg-runtime:0.1.4 \
  --conf spark.sql.catalog.s3tablesbucket=org.apache.iceberg.spark.SparkCatalog \
  --conf spark.sql.catalog.s3tablesbucket.catalog-impl=software.amazon.s3tables.iceberg.S3TablesCatalog \
  --conf spark.sql.catalog.s3tablesbucket.warehouse=arn:aws:s3tables:us-east-1:111122223333:bucket/bucket-name \
  --conf spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions
```

**使用するライブラリ**:
1. `org.apache.iceberg:iceberg-spark-runtime-3.5_2.12:1.6.1`
   - Apache Iceberg公式のSparkランタイム
2. `software.amazon.s3tables:s3-tables-catalog-for-iceberg-runtime:0.1.4`
   - AWS提供のS3 Tables専用カタログ実装

**出典**: [Accessing Amazon S3 tables with the Amazon S3 Tables Catalog for Apache Iceberg](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-client-catalog.html)

#### Spark SQLでの操作例

```sql
-- ネームスペース作成
CREATE NAMESPACE IF NOT EXISTS s3tablesbucket.my_namespace;

-- テーブル作成
CREATE TABLE IF NOT EXISTS s3tablesbucket.my_namespace.my_table 
  (id INT, name STRING, value INT) 
  USING iceberg;

-- データ挿入
INSERT INTO s3tablesbucket.my_namespace.my_table 
  VALUES (1, 'ABC', 100), (2, 'XYZ', 200);

-- クエリ実行
SELECT * FROM s3tablesbucket.my_namespace.my_table;
```

**出典**: [Accessing Amazon S3 tables with the Amazon S3 Tables Catalog for Apache Iceberg](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-client-catalog.html)

## 【Icebergライブラリの役割まとめ】

### 1. メタデータ管理
- **スキーマ進化**: テーブル構造の変更を追跡
- **パーティション進化**: パーティション戦略の変更をサポート
- **スナップショット管理**: データのバージョン管理

### 2. トランザクション管理
- **ACID保証**: データの一貫性を保証
- **楽観的並行制御**: 複数の書き込みを安全に処理

### 3. クエリ最適化
- **ファイルプルーニング**: 不要なファイルの読み込みを回避
- **パーティションプルーニング**: 不要なパーティションをスキップ
- **統計情報**: クエリプランナーへの情報提供

**出典**: 
- [Query Apache Iceberg tables - Amazon Athena](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg.html)
- [Optimize Iceberg tables - Amazon Athena](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg-data-optimization.html)

## 【S3 TablesとセルフマネージドIcebergの違い】

| 項目 | S3 Tables | セルフマネージドIceberg on S3 |
|------|-----------|------------------------------|
| **カタログ実装** | S3 Tables Catalog (AWS提供) | Glue/Hive/REST Catalog |
| **メンテナンス** | 自動（コンパクション、スナップショット管理） | 手動実装が必要 |
| **ストレージ** | Table Bucket（専用バケットタイプ） | General Purpose Bucket |
| **パフォーマンス** | 最適化済み（高TPS、高スループット） | 標準S3パフォーマンス |
| **IAMネームスペース** | s3tables:* | s3:* |
| **統合** | SageMaker Lakehouse、Glue自動統合 | 手動統合 |
| **Icebergライブラリ** | 標準Icebergライブラリ + S3 Tablesカタログ | 標準Icebergライブラリ |

**出典**: 
- [Working with Amazon S3 Tables and table buckets](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables.html)
- [Accessing Amazon S3 tables with the Amazon S3 Tables Catalog for Apache Iceberg](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-client-catalog.html)

## 【AWSサービスでのIcebergサポート】

### 1. Amazon Athena
- Icebergテーブルのクエリ、作成、更新
- タイムトラベルクエリ
- スキーマ進化
- テーブル最適化（OPTIMIZE、VACUUM）

**出典**: [Query Apache Iceberg tables - Amazon Athena](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg.html)

### 2. Amazon Redshift
- Icebergテーブルへの書き込み
- AWS Glueカタログとの統合
- レイクハウスアーキテクチャ

**出典**: [Writing to Apache Iceberg tables - Amazon Redshift](https://docs.aws.amazon.com/redshift/latest/dg/iceberg-writes.html)

### 3. AWS Glue
- Icebergテーブルのソース/ターゲット
- Data Catalogとの統合
- ETLジョブでのIceberg操作

**出典**: [Using Apache Iceberg framework in AWS Glue Studio](https://docs.aws.amazon.com/glue/latest/dg/gs-data-lake-formats-iceberg.html)

### 4. Amazon EMR
- SparkでのIcebergテーブルアクセス
- S3 Tablesとの統合

**出典**: [Accessing Amazon S3 tables with Amazon EMR](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-integrating-emr.html)

### 5. Amazon Data Firehose
- Icebergテーブルへのストリーミングデータ配信

**出典**: [Prerequisites to use Apache Iceberg Tables as a destination](https://docs.aws.amazon.com/firehose/latest/dev/apache-iceberg-prereq.html)

## 【注意点・例外】

### 1. カタログ実装の選択

S3 Tablesにアクセスする方法は2つ：

1. **S3 Tables Catalog for Iceberg**（推奨）
   - AWS Labs提供のオープンソースライブラリ
   - Iceberg操作をS3 Tables APIに変換
   - 最新バージョン: 0.1.4（2025-11-15時点）

2. **Iceberg REST Endpoint**
   - 標準Iceberg REST APIクライアント使用
   - 基本的な読み書きアクセスのみ
   - 単一テーブルバケットアクセス向け

**推奨**: 統合管理、ガバナンス、細かいアクセス制御が必要な場合はAWS Glue Iceberg REST endpointを使用

**出典**: [Accessing tables using the Amazon S3 Tables Iceberg REST endpoint](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-integrating-open-source.html)

### 2. Icebergバージョン互換性

- S3 Tablesは最新のIcebergライブラリと互換性あり
- 推奨バージョン: Apache Iceberg 1.6.1以降
- Spark 3.5系との組み合わせを推奨

**出典**: [Accessing Amazon S3 tables with the Amazon S3 Tables Catalog for Apache Iceberg](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-client-catalog.html)

### 3. セルフマネージドIcebergからの移行

既存のセルフマネージドIcebergテーブルをS3 Tablesに移行する場合：
- メタデータの移行が必要
- カタログ実装の変更が必要
- データファイル自体はIcebergフォーマットのため互換性あり

## 【出典】

1. [Accessing Amazon S3 tables with the Amazon S3 Tables Catalog for Apache Iceberg - Amazon Simple Storage Service](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-client-catalog.html) - アクセス日: 2025-11-15
2. [Query Apache Iceberg tables - Amazon Athena](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg.html) - アクセス日: 2025-11-15
3. [Creating Apache Iceberg tables - AWS Lake Formation](https://docs.aws.amazon.com/lake-formation/latest/dg/creating-iceberg-tables.html) - アクセス日: 2025-11-15
4. [Optimize Iceberg tables - Amazon Athena](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg-data-optimization.html) - アクセス日: 2025-11-15
5. [Writing to Apache Iceberg tables - Amazon Redshift](https://docs.aws.amazon.com/redshift/latest/dg/iceberg-writes.html) - アクセス日: 2025-11-15
6. [Using Apache Iceberg framework in AWS Glue Studio - AWS Glue](https://docs.aws.amazon.com/glue/latest/dg/gs-data-lake-formats-iceberg.html) - アクセス日: 2025-11-15
7. [AWS Labs GitHub - s3-tables-catalog](https://github.com/awslabs/s3-tables-catalog) - 参照元: AWS公式ドキュメント
8. [Maven Central - s3-tables-catalog-for-iceberg](https://mvnrepository.com/artifact/software.amazon.s3tables/s3-tables-catalog-for-iceberg) - 参照元: AWS公式ドキュメント

## 【確実性：高】

AWS公式ドキュメントおよびAWS Labs GitHubリポジトリから取得した一次情報です。
