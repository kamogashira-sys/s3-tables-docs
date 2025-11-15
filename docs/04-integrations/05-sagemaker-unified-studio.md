# Amazon SageMaker Unified Studioとの統合

## 概要

Amazon SageMaker Unified Studioは、データエンジニアリング、データサイエンス、機械学習を統合した開発環境です。S3 Tablesとの統合により、Apache Icebergテーブルを使用した高度なデータ分析とML開発が可能になります。

### SageMaker Unified Studioとは

SageMaker Unified Studioは、以下の機能を提供する統合プラットフォームです：

- **データカタログ管理**: メタデータの一元管理とガバナンス
- **データ探索**: インタラクティブなデータ探索とビジュアライゼーション
- **データ変換**: ETL/ELTパイプラインの構築と実行
- **機械学習**: モデル開発、トレーニング、デプロイ
- **コラボレーション**: チーム間でのデータとモデルの共有

### S3 Tables統合のメリット

1. **統一されたデータアクセス**
   - S3 Tablesを他のデータソースと同じインターフェースでアクセス
   - 一貫したメタデータ管理

2. **きめ細かいアクセス制御**
   - テーブルレベル、列レベルの権限管理
   - AWS Lake Formationとの統合

3. **高性能なクエリ実行**
   - Apache Icebergの最適化機能を活用
   - パーティショニングとコンパクションの自動管理

4. **シームレスなML統合**
   - テーブルデータを直接ML特徴量として使用
   - モデルトレーニングパイプラインへの統合

5. **データリネージ追跡**
   - データの流れと変換履歴の可視化
   - コンプライアンスと監査の支援

## 統合機能

### 1. テーブルバケット作成

SageMaker Unified Studioから直接S3テーブルバケットを作成できます。

**手順**:
1. SageMaker Unified Studioコンソールを開く
2. 「Data」セクションに移動
3. 「Create table bucket」を選択
4. バケット名、リージョン、暗号化設定を指定
5. 作成を実行

**AWS CLI例**:
```bash
aws s3tables create-table-bucket \
  --name my-sagemaker-table-bucket \
  --region us-east-1 \
  --encryption-configuration '{
    "type": "aws:kms",
    "kmsKeyArn": "arn:aws:kms:us-east-1:111122223333:key/1234abcd-12ab-34cd-56ef-1234567890ab"
  }'
```

### 2. Apache Icebergテーブル作成

SageMaker Unified Studio内でIcebergテーブルを作成し、データカタログに登録できます。

**手順**:
1. テーブルバケットを選択
2. 「Create table」を選択
3. テーブル名、スキーマ、パーティショニングを定義
4. 作成を実行

**Spark SQL例**:
```sql
CREATE TABLE sagemaker_catalog.my_namespace.customer_data (
  customer_id BIGINT,
  name STRING,
  email STRING,
  registration_date DATE,
  total_purchases DECIMAL(10,2),
  last_purchase_date TIMESTAMP
)
USING iceberg
PARTITIONED BY (months(registration_date))
TBLPROPERTIES (
  'write.format.default' = 'parquet',
  'write.metadata.compression-codec' = 'gzip'
);
```

### 3. AWS分析サービスとの連携

S3 Tablesは、SageMaker Unified Studioを通じて以下のAWSサービスと統合されます：

**Amazon Athena**
- SQLクエリによるデータ探索
- CTAS（Create Table As Select）によるテーブル作成
- ビューの作成と管理

**AWS Glue**
- ETLジョブでのデータ変換
- クローラーによるスキーマ検出
- Data Catalogとの同期

**Amazon EMR**
- Sparkジョブでの大規模データ処理
- Hiveクエリの実行
- Prestoクエリの実行

**Amazon Redshift**
- Redshift Spectrumによる外部テーブルアクセス
- データウェアハウスとの統合分析

### 4. テーブルクエリ

SageMaker Unified Studioのノートブック環境から、S3 Tablesに対してクエリを実行できます。

**Python (PySpark) 例**:
```python
from pyspark.sql import SparkSession

# Sparkセッションの作成
spark = SparkSession.builder \
    .appName("SageMaker S3 Tables") \
    .config("spark.sql.catalog.s3tables", "org.apache.iceberg.spark.SparkCatalog") \
    .config("spark.sql.catalog.s3tables.catalog-impl", "software.amazon.s3tables.iceberg.S3TablesCatalog") \
    .getOrCreate()

# テーブルの読み取り
df = spark.table("s3tables.my_namespace.customer_data")

# データ探索
df.show(10)
df.printSchema()
df.describe().show()

# フィルタリング
recent_customers = df.filter(df.registration_date >= '2024-01-01')
recent_customers.show()

# 集計
df.groupBy("registration_date").count().show()
```

**SQL例**:
```sql
-- テーブルの確認
SHOW TABLES IN s3tables.my_namespace;

-- データクエリ
SELECT 
  customer_id,
  name,
  total_purchases,
  last_purchase_date
FROM s3tables.my_namespace.customer_data
WHERE registration_date >= CURRENT_DATE - INTERVAL '30' DAY
ORDER BY total_purchases DESC
LIMIT 10;

-- 集計クエリ
SELECT 
  DATE_TRUNC('month', registration_date) AS month,
  COUNT(*) AS new_customers,
  SUM(total_purchases) AS total_revenue
FROM s3tables.my_namespace.customer_data
GROUP BY DATE_TRUNC('month', registration_date)
ORDER BY month DESC;
```

## Lakehouseアーキテクチャ統合

### アーキテクチャ概要

SageMaker Unified StudioのLakehouseアーキテクチャは、S3 Tablesを中核として以下のコンポーネントで構成されます：

```
┌─────────────────────────────────────────────────────────┐
│         SageMaker Unified Studio                        │
├─────────────────────────────────────────────────────────┤
│  Data Catalog  │  Query Engine  │  ML Workbench        │
└────────┬────────┴────────┬───────┴──────────┬───────────┘
         │                 │                  │
         ▼                 ▼                  ▼
┌─────────────────────────────────────────────────────────┐
│              AWS Lake Formation                         │
│         (Fine-grained Access Control)                   │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│              S3 Tables (Iceberg)                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │ Table 1  │  │ Table 2  │  │ Table 3  │             │
│  └──────────┘  └──────────┘  └──────────┘             │
└─────────────────────────────────────────────────────────┘
```

### データソースの結合

S3 Tablesと他のデータソースを統合したクエリが可能です：

```sql
-- S3 TablesとRedshiftの結合
SELECT 
  t.customer_id,
  t.name,
  r.order_count,
  r.total_amount
FROM s3tables.my_namespace.customer_data t
JOIN redshift_catalog.public.orders_summary r
  ON t.customer_id = r.customer_id
WHERE t.registration_date >= '2024-01-01';

-- S3 TablesとAthenaの結合
SELECT 
  t.product_id,
  t.product_name,
  a.page_views,
  a.conversion_rate
FROM s3tables.my_namespace.products t
LEFT JOIN athena_catalog.analytics.product_metrics a
  ON t.product_id = a.product_id;
```

### きめ細かいアクセス権限管理

AWS Lake Formationを使用して、テーブル、列、行レベルでアクセス制御を設定できます。

**テーブルレベル権限**:
```bash
aws lakeformation grant-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::111122223333:role/DataAnalystRole \
  --resource '{"Table":{"DatabaseName":"my_namespace","Name":"customer_data"}}' \
  --permissions SELECT DESCRIBE
```

**列レベル権限**:
```bash
aws lakeformation grant-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::111122223333:role/DataAnalystRole \
  --resource '{"TableWithColumns":{"DatabaseName":"my_namespace","Name":"customer_data","ColumnNames":["customer_id","name","total_purchases"]}}' \
  --permissions SELECT
```

**行レベルフィルタ**:
```bash
aws lakeformation create-data-cells-filter \
  --table-catalog-id 111122223333 \
  --database-name my_namespace \
  --table-name customer_data \
  --name regional_filter \
  --row-filter '{"FilterExpression":"region = '\''us-east-1'\''"}'
```

## セットアップ手順

### 前提条件

1. **AWSアカウント**
   - SageMaker Unified Studioが有効化されていること
   - 必要なIAM権限が付与されていること

2. **S3テーブルバケット**
   - 既存のテーブルバケット、または新規作成

3. **AWS Lake Formation設定**
   - Lake Formationが有効化されていること
   - データレイク管理者権限

### ステップ1: S3 Tables統合の有効化

```bash
# Lake FormationでS3 Tables統合を有効化
aws lakeformation register-resource \
  --resource-arn arn:aws:s3tables:us-east-1:111122223333:bucket/my-table-bucket \
  --use-service-linked-role
```

### ステップ2: テーブルバケットのオンボーディング

```bash
# SageMaker Unified StudioにテーブルバケットをOnboard
aws sagemaker create-data-source \
  --domain-id d-xxxxxxxxxxxx \
  --name s3-tables-datasource \
  --data-source-config '{
    "S3TablesConfig": {
      "TableBucketArn": "arn:aws:s3tables:us-east-1:111122223333:bucket/my-table-bucket"
    }
  }'
```

### ステップ3: IAMロールの設定

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3tables:GetTable",
        "s3tables:GetTableData",
        "s3tables:ListTables",
        "s3tables:ListNamespaces"
      ],
      "Resource": [
        "arn:aws:s3tables:*:*:bucket/*/table/*",
        "arn:aws:s3tables:*:*:bucket/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "lakeformation:GetDataAccess"
      ],
      "Resource": "*"
    }
  ]
}
```

### ステップ4: テーブルの作成とデータロード

```python
# SageMaker Unified Studioノートブックで実行
from pyspark.sql import SparkSession
from pyspark.sql.types import *

spark = SparkSession.builder \
    .appName("S3TablesSetup") \
    .getOrCreate()

# スキーマ定義
schema = StructType([
    StructField("customer_id", LongType(), False),
    StructField("name", StringType(), True),
    StructField("email", StringType(), True),
    StructField("registration_date", DateType(), True),
    StructField("total_purchases", DecimalType(10,2), True)
])

# サンプルデータ作成
data = [
    (1, "Alice", "alice@example.com", "2024-01-15", 1250.50),
    (2, "Bob", "bob@example.com", "2024-02-20", 890.25),
    (3, "Charlie", "charlie@example.com", "2024-03-10", 2100.75)
]

df = spark.createDataFrame(data, schema)

# テーブルに書き込み
df.writeTo("s3tables.my_namespace.customer_data") \
    .using("iceberg") \
    .createOrReplace()
```

## 使用例

### 例1: データ分析ワークフロー

```python
# データ探索
df = spark.table("s3tables.my_namespace.customer_data")

# 基本統計
df.describe().show()

# 顧客セグメンテーション
from pyspark.sql.functions import when, col

df_segmented = df.withColumn(
    "segment",
    when(col("total_purchases") >= 2000, "Premium")
    .when(col("total_purchases") >= 1000, "Standard")
    .otherwise("Basic")
)

# セグメント別集計
df_segmented.groupBy("segment").agg({
    "customer_id": "count",
    "total_purchases": "avg"
}).show()

# 結果を新しいテーブルに保存
df_segmented.writeTo("s3tables.my_namespace.customer_segments") \
    .using("iceberg") \
    .createOrReplace()
```

### 例2: MLモデル学習での利用

```python
from sagemaker.spark import SageMakerEstimator
from pyspark.ml.feature import VectorAssembler
from pyspark.ml.classification import RandomForestClassifier

# 特徴量エンジニアリング
df = spark.table("s3tables.my_namespace.customer_data")

# 特徴量ベクトルの作成
assembler = VectorAssembler(
    inputCols=["total_purchases", "days_since_registration"],
    outputCol="features"
)

df_features = assembler.transform(df)

# トレーニングデータとテストデータに分割
train_df, test_df = df_features.randomSplit([0.8, 0.2], seed=42)

# モデルトレーニング
rf = RandomForestClassifier(
    featuresCol="features",
    labelCol="churn_label",
    numTrees=100
)

model = rf.fit(train_df)

# 予測
predictions = model.transform(test_df)

# 結果をテーブルに保存
predictions.select("customer_id", "prediction", "probability") \
    .writeTo("s3tables.my_namespace.churn_predictions") \
    .using("iceberg") \
    .createOrReplace()
```

## ベストプラクティス

### 1. データカタログの整理

- 明確な命名規則を使用
- メタデータを充実させる
- タグを活用した分類

### 2. アクセス制御の設計

- 最小権限の原則を適用
- 列レベルのマスキングを活用
- 定期的な権限レビュー

### 3. パフォーマンス最適化

- 適切なパーティショニング戦略
- コンパクション設定の調整
- クエリパターンに応じたソート順序

### 4. コスト管理

- 不要なスナップショットの削除
- ストレージクラスの最適化
- クエリ実行の監視

### 5. データガバナンス

- データリネージの追跡
- データ品質チェックの自動化
- コンプライアンス要件の遵守

## トラブルシューティング

### 問題: テーブルが表示されない

**原因**: Lake Formation権限不足

**解決策**:
```bash
aws lakeformation grant-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::111122223333:role/SageMakerRole \
  --resource '{"Database":{"Name":"my_namespace"}}' \
  --permissions DESCRIBE
```

### 問題: クエリ実行時にアクセス拒否

**原因**: IAMロールまたはLake Formation権限不足

**解決策**: 必要な権限を確認し、付与

### 問題: パフォーマンスが遅い

**原因**: パーティショニングまたはコンパクション設定が不適切

**解決策**: テーブル設計を見直し、最適化

## 参考リンク

- [AWS公式ドキュメント: Get started with Amazon S3 Tables in Amazon SageMaker Unified Studio](https://docs.aws.amazon.com/next-generation-sagemaker/latest/userguide/s3-tables-integration.html)
- [AWS公式ドキュメント: Amazon S3 tables integration - The lakehouse architecture of Amazon SageMaker](https://docs.aws.amazon.com/sagemaker-lakehouse-architecture/latest/userguide/lakehouse-s3-tables-integration.html)
- [SageMaker Unified Studio公式ドキュメント](https://docs.aws.amazon.com/sagemaker-unified-studio/latest/userguide/)
- [データガバナンス](../03-security/02-data-governance.md)
- [AWS Glue統合](./01-glue-integration.md)
