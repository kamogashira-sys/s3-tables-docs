# Amazon S3 Tables - 機械学習統合


## 目次

- [概要](#概要)
  - [機械学習統合の重要性](#機械学習統合の重要性)
- [統合アーキテクチャ](#統合アーキテクチャ)
  - [全体像](#全体像)
  - [統合パターン](#統合パターン)
    - [パターン1: SageMaker統合](#パターン1-sagemaker統合)
    - [パターン2: Athena ML統合](#パターン2-athena-ml統合)
    - [パターン3: EMR Spark ML統合](#パターン3-emr-spark-ml統合)
- [SageMaker統合](#sagemaker統合)
  - [SageMaker Feature Store統合](#sagemaker-feature-store統合)
    - [概要](#概要)
    - [Feature Storeの構成](#feature-storeの構成)
    - [S3 Tablesとの統合方法](#s3-tablesとの統合方法)
  - [SageMaker Training統合](#sagemaker-training統合)
    - [S3 Tablesからの学習データ読み取り](#s3-tablesからの学習データ読み取り)
  - [SageMaker Lakehouse統合](#sagemaker-lakehouse統合)
    - [概要](#概要)
    - [アーキテクチャ](#アーキテクチャ)
    - [統合設定](#統合設定)
  - [SageMaker Data Wrangler統合](#sagemaker-data-wrangler統合)
    - [データ変換とエクスポート](#データ変換とエクスポート)
- [Athena ML統合](#athena-ml統合)
  - [概要](#概要)
  - [アーキテクチャ](#アーキテクチャ)
  - [実装例](#実装例)
    - [1. SageMakerモデルのデプロイ](#1-sagemakerモデルのデプロイ)
    - [2. Athena MLクエリ](#2-athena-mlクエリ)
    - [3. 複数モデルの使用](#3-複数モデルの使用)
  - [ユースケース](#ユースケース)
- [EMR Spark ML統合](#emr-spark-ml統合)
  - [概要](#概要)
  - [アーキテクチャ](#アーキテクチャ)
  - [実装例](#実装例)
    - [1. Spark MLlibを使用した学習](#1-spark-mllibを使用した学習)
    - [2. 推論結果をS3 Tablesに保存](#2-推論結果をs3-tablesに保存)
    - [3. ストリーミングML推論](#3-ストリーミングml推論)
  - [EMR Serverlessでの実行](#emr-serverlessでの実行)
- [AWS Glue ML統合](#aws-glue-ml統合)
  - [Glue DataBrewとの統合](#glue-databrewとの統合)
    - [データプロファイリングと準備](#データプロファイリングと準備)
  - [Glue ML Transformsとの統合](#glue-ml-transformsとの統合)
- [ベストプラクティス](#ベストプラクティス)
  - [1. データ準備](#1-データ準備)
  - [2. 特徴量エンジニアリング](#2-特徴量エンジニアリング)
  - [3. モデル学習](#3-モデル学習)
  - [4. モデル推論](#4-モデル推論)
  - [5. パフォーマンス最適化](#5-パフォーマンス最適化)
  - [6. コスト最適化](#6-コスト最適化)
- [モニタリングとトラブルシューティング](#モニタリングとトラブルシューティング)
  - [CloudWatch Metrics](#cloudwatch-metrics)
    - [SageMaker Metrics](#sagemaker-metrics)
    - [EMR Metrics](#emr-metrics)
  - [トラブルシューティング](#トラブルシューティング)
    - [一般的な問題と解決策](#一般的な問題と解決策)
- [出典](#出典)
  - [AWS公式ドキュメント](#aws公式ドキュメント)
  - [関連ドキュメント](#関連ドキュメント)
- [更新履歴](#更新履歴)

## 概要

Amazon S3 Tablesは、AWSの機械学習サービスとシームレスに統合し、データレイクから機械学習パイプラインへの効率的なデータフローを実現します。本ドキュメントでは、S3 Tablesと各種機械学習サービスの統合方法、ベストプラクティス、実装例を詳細に解説します。

### 機械学習統合の重要性

**データからインサイトへ**
- 構造化されたテーブルデータを直接ML学習に利用
- データ準備時間の短縮
- モデル学習の効率化

**エンドツーエンドMLパイプライン**
- データ取り込み → 特徴量エンジニアリング → モデル学習 → 推論
- 一貫したデータフォーマット（Apache Iceberg）
- スケーラブルなアーキテクチャ

**リアルタイムとバッチの統合**
- ストリーミングデータの即座の学習
- バッチ推論の効率化
- オンライン/オフライン特徴量の統合

---

## 統合アーキテクチャ

### 全体像

```text
データソース → S3 Tables → ML統合レイヤー → MLサービス
                    ↓
              [Iceberg Tables]
                    ↓
        ┌───────────┼───────────┐
        ↓           ↓           ↓
   SageMaker   Athena ML    EMR Spark
   Feature Store              ML
```

### 統合パターン

#### パターン1: SageMaker統合

**アーキテクチャ**:
```text
S3 Tables → SageMaker Feature Store → SageMaker Training/Inference
```

**特徴**:
- Feature Storeのオフラインストアとしてのiceberg利用
- オンライン/オフライン特徴量の統合
- 低レイテンシの特徴量取得

#### パターン2: Athena ML統合

**アーキテクチャ**:
```text
S3 Tables → Athena → SageMaker Model → 推論結果
```

**特徴**:
- SQLベースのML推論
- バッチ推論の簡素化
- 既存SQLワークフローへの統合

#### パターン3: EMR Spark ML統合

**アーキテクチャ**:
```text
S3 Tables → EMR Spark → MLlib/Spark ML → モデル学習/推論
```

**特徴**:
- 大規模データ処理
- 分散機械学習
- カスタムアルゴリズムの実装

---

## SageMaker統合

### SageMaker Feature Store統合

#### 概要

SageMaker Feature Storeは、機械学習の特徴量を一元管理するサービスです。オフラインストアとしてIcebergテーブルフォーマットをサポートしており、S3 Tablesと統合できます。

#### Feature Storeの構成

**オンラインストア**:
- 低レイテンシの特徴量取得（ミリ秒単位）
- リアルタイム推論用
- DynamoDBベース

**オフラインストア**:
- 履歴データの保存
- モデル学習用
- S3ベース（Iceberg/Glueフォーマット）

#### S3 Tablesとの統合方法

**方法1: Feature StoreのオフラインストアとしてS3 Tablesを使用**

現在、Feature StoreはIcebergフォーマットをサポートしていますが、S3 Tablesへの直接統合は制限があります。以下の方法で統合できます。

**方法2: S3 TablesからFeature Storeへのデータ取り込み**

```python
import boto3
import pandas as pd
from sagemaker.feature_store.feature_group import FeatureGroup
from sagemaker.session import Session

# SageMaker Session
sagemaker_session = Session()
region = boto3.Session().region_name

# Feature Group作成
feature_group_name = "customer-features"
feature_group = FeatureGroup(
    name=feature_group_name,
    sagemaker_session=sagemaker_session
)

# S3 Tablesからデータ読み取り（Athena経由）
athena_client = boto3.client('athena')

query = """
SELECT 
    customer_id,
    total_purchases,
    avg_purchase_amount,
    last_purchase_date,
    customer_segment,
    CAST(current_timestamp AS BIGINT) as event_time
FROM s3tablescatalog.my_namespace.customers
"""

# Athenaクエリ実行
response = athena_client.start_query_execution(
    QueryString=query,
    QueryExecutionContext={'Database': 's3tablescatalog'},
    ResultConfiguration={'OutputLocation': 's3://my-bucket/athena-results/'}
)

# 結果取得
query_execution_id = response['QueryExecutionId']

# 結果をDataFrameに変換
df = pd.read_csv(f's3://my-bucket/athena-results/{query_execution_id}.csv')

# Feature Storeに取り込み
feature_group.ingest(
    data_frame=df,
    max_workers=3,
    wait=True
)
```

**方法3: Sparkを使用した統合**

```python
from pyspark.sql import SparkSession

# Spark Session
spark = SparkSession.builder \
    .appName("S3TablesToFeatureStore") \
    .config("spark.sql.catalog.s3tablescatalog", "software.amazon.s3tables.iceberg.S3TablesCatalog") \
    .config("spark.sql.catalog.s3tablescatalog.warehouse", "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket") \
    .getOrCreate()

# S3 Tablesから読み取り
df = spark.sql("""
    SELECT
        customer_id,
        total_purchases,
        avg_purchase_amount,
        last_purchase_date,
        customer_segment,
        unix_timestamp() as event_time
    FROM s3tablescatalog.my_namespace.customers
""")

# Feature Storeに書き込み
# Feature Store SDKを使用
from sagemaker.feature_store.feature_group import FeatureGroup

feature_group = FeatureGroup(name="customer-features", sagemaker_session=sagemaker_session)

# DataFrameをPandas DataFrameに変換
pandas_df = df.toPandas()

# Feature Storeに取り込み
feature_group.ingest(
    data_frame=pandas_df,
    max_workers=3,
    wait=True
)
```

### SageMaker Training統合

#### S3 Tablesからの学習データ読み取り

**方法1: Athena経由でCSV/Parquetに変換**

```python
import boto3
import sagemaker
from sagemaker.estimator import Estimator

# Athenaでデータをエクスポート
athena_client = boto3.client('athena')

query = """
UNLOAD (
    SELECT * FROM s3tablescatalog.my_namespace.training_data
)
TO 's3://my-bucket/training-data/'
WITH (format = 'PARQUET', compression = 'SNAPPY')
"""

response = athena_client.start_query_execution(
    QueryString=query,
    QueryExecutionContext={'Database': 's3tablescatalog'},
    ResultConfiguration={'OutputLocation': 's3://my-bucket/athena-results/'}
)

# SageMaker Training Job
estimator = Estimator(
    image_uri='<training-image>',
    role='<iam-role>',
    instance_count=1,
    instance_type='ml.m5.xlarge',
    output_path='s3://my-bucket/model-output/'
)

estimator.fit({'training': 's3://my-bucket/training-data/'})
```

**方法2: Spark経由でSageMaker Trainingに渡す**

```python
from pyspark.sql import SparkSession
import sagemaker
from sagemaker.spark import SageMakerEstimator

# Spark Session
spark = SparkSession.builder \
    .appName("S3TablesToSageMaker") \
    .config("spark.sql.catalog.s3tablescatalog", "software.amazon.s3tables.iceberg.S3TablesCatalog") \
    .config("spark.sql.catalog.s3tablescatalog.warehouse", "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket") \
    .getOrCreate()

# S3 Tablesから読み取り
training_df = spark.sql("""
    SELECT * FROM s3tablescatalog.my_namespace.training_data
""")

# Parquetとして保存
training_df.write.mode("overwrite").parquet("s3://my-bucket/training-data/")

# SageMaker Training
estimator = Estimator(
    image_uri='<training-image>',
    role='<iam-role>',
    instance_count=1,
    instance_type='ml.m5.xlarge',
    output_path='s3://my-bucket/model-output/'
)

estimator.fit({'training': 's3://my-bucket/training-data/'})
```

### SageMaker Lakehouse統合

#### 概要

SageMaker Lakehouseは、S3データレイク、Redshiftデータウェアハウス、S3 Tablesを統合し、Apache Icebergを使用して統一されたデータアクセスを提供します。

#### アーキテクチャ

```text
S3 Tables (Iceberg) ─┐
                     ├─→ SageMaker Lakehouse ─→ SageMaker Studio
Redshift ────────────┘                          ↓
                                          ML Workloads
```

#### 統合設定

**Table Bucketの統合**:
```bash
# S3コンソールからSageMaker Lakehouseとの統合を有効化
# または AWS CLIを使用

aws s3tables create-table-bucket \
  --name my-lakehouse-bucket \
  --region us-east-1

# SageMaker Lakehouseカタログに登録
# （現在はコンソール経由で設定）
```

**SageMaker Studioからのアクセス**:
```python
import pandas as pd
from sagemaker import Session

# SageMaker Session
sagemaker_session = Session()

# S3 Tablesへのクエリ（Athena経由）
query = """
SELECT * FROM s3tablescatalog.my_namespace.customers
LIMIT 1000
"""

# Athena経由でクエリ実行
df = pd.read_sql(query, con=athena_connection)

# データ分析・ML処理
print(df.head())
```

### SageMaker Data Wrangler統合

#### データ変換とエクスポート

```python
# Data Wranglerフロー定義
# S3 Tablesからデータを読み取り、変換、Feature Storeにエクスポート

# 1. S3 Tablesからデータ読み取り（Athena Data Source）
# 2. Data Wranglerで変換
# 3. Feature Storeにエクスポート

# Data Wrangler Flowの例（JSON形式）
{
  "metadata": {
    "version": 1
  },
  "nodes": [
    {
      "node_id": "source_node",
      "type": "SOURCE",
      "parameters": {
        "dataset_definition": {
          "datasetSourceType": "Athena",
          "name": "s3tables_source",
          "catalogName": "s3tablescatalog",
          "databaseName": "my_namespace",
          "queryString": "SELECT * FROM customers",
          "s3OutputLocation": "s3://my-bucket/data-wrangler-output/"
        }
      }
    },
    {
      "node_id": "transform_node",
      "type": "TRANSFORM",
      "parents": ["source_node"],
      "parameters": {
        "transform_type": "Custom Pandas",
        "code": "df['total_value'] = df['quantity'] * df['price']"
      }
    },
    {
      "node_id": "export_node",
      "type": "EXPORT",
      "parents": ["transform_node"],
      "parameters": {
        "destination": "FeatureStore",
        "feature_group_name": "customer_features"
      }
    }
  ]
}
```

---

## Athena ML統合

### 概要

Athena MLを使用すると、SQLクエリ内でSageMakerモデルを呼び出し、S3 Tablesデータに対してML推論を実行できます。

### アーキテクチャ

```
S3 Tables → Athena SQL → SageMaker Model → 推論結果
```

### 実装例

#### 1. SageMakerモデルのデプロイ

```python
import sagemaker
from sagemaker.model import Model

# モデルの作成
model = Model(
    model_data='s3://my-bucket/model/model.tar.gz',
    image_uri='<inference-image>',
    role='<iam-role>',
    name='customer-churn-model'
)

# エンドポイントのデプロイ
predictor = model.deploy(
    initial_instance_count=1,
    instance_type='ml.m5.xlarge',
    endpoint_name='customer-churn-endpoint'
)
```

#### 2. Athena MLクエリ

**ML関数の作成**:
```sql
-- SageMakerモデルをAthena関数として登録
USING EXTERNAL FUNCTION predict_churn(
    total_purchases DOUBLE,
    avg_purchase_amount DOUBLE,
    days_since_last_purchase INT,
    customer_segment VARCHAR
)
RETURNS DOUBLE
SAGEMAKER 'customer-churn-endpoint'

-- S3 Tablesデータに対して推論実行
SELECT 
    customer_id,
    customer_name,
    predict_churn(
        total_purchases,
        avg_purchase_amount,
        days_since_last_purchase,
        customer_segment
    ) as churn_probability
FROM s3tablescatalog.my_namespace.customers
WHERE churn_probability > 0.7
ORDER BY churn_probability DESC
LIMIT 100;
```

**バッチ推論**:
```sql
-- 推論結果を新しいテーブルに保存
CREATE TABLE s3tablescatalog.my_namespace.customer_churn_predictions
USING iceberg
AS
SELECT
    customer_id,
    customer_name,
    total_purchases,
    avg_purchase_amount,
    predict_churn(
        total_purchases,
        avg_purchase_amount,
        days_since_last_purchase,
        customer_segment
    ) as churn_probability,
    CASE
        WHEN predict_churn(...) > 0.7 THEN 'High Risk'
        WHEN predict_churn(...) > 0.4 THEN 'Medium Risk'
        ELSE 'Low Risk'
    END as risk_category,
    current_timestamp as prediction_timestamp
FROM s3tablescatalog.my_namespace.customers;
```

#### 3. 複数モデルの使用

```sql
-- 複数のML関数を定義
USING EXTERNAL FUNCTION predict_churn(...) RETURNS DOUBLE SAGEMAKER 'churn-endpoint'
USING EXTERNAL FUNCTION predict_ltv(...) RETURNS DOUBLE SAGEMAKER 'ltv-endpoint'
USING EXTERNAL FUNCTION predict_segment(...) RETURNS VARCHAR SAGEMAKER 'segment-endpoint'

-- 複数モデルの推論を統合
SELECT 
    customer_id,
    predict_churn(...) as churn_probability,
    predict_ltv(...) as lifetime_value,
    predict_segment(...) as predicted_segment
FROM s3tablescatalog.my_namespace.customers;
```

### ユースケース

**顧客離反予測**:
- 離反リスクの高い顧客を特定
- リテンション施策の優先順位付け

**レコメンデーション**:
- 商品推薦
- パーソナライズされたコンテンツ

**異常検知**:
- 不正取引の検出
- システム異常の検出


---

## EMR Spark ML統合

### 概要

EMR Sparkを使用して、S3 Tablesデータに対して大規模な機械学習処理を実行できます。MLlibやSpark MLを使用した分散機械学習が可能です。

### アーキテクチャ

```
S3 Tables → EMR Spark → MLlib/Spark ML → モデル学習/推論
```

### 実装例

#### 1. Spark MLlibを使用した学習

```python
from pyspark.sql import SparkSession
from pyspark.ml.feature import VectorAssembler, StandardScaler
from pyspark.ml.classification import RandomForestClassifier
from pyspark.ml.evaluation import BinaryClassificationEvaluator
from pyspark.ml import Pipeline

# Spark Session
spark = SparkSession.builder \
    .appName("S3TablesMLTraining") \
    .config("spark.sql.catalog.s3tablescatalog", "software.amazon.s3tables.iceberg.S3TablesCatalog") \
    .config("spark.sql.catalog.s3tablescatalog.warehouse", "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket") \
    .getOrCreate()

# S3 Tablesからデータ読み取り
df = spark.sql("""
    SELECT
        customer_id,
        total_purchases,
        avg_purchase_amount,
        days_since_last_purchase,
        customer_segment,
        churned
    FROM s3tablescatalog.my_namespace.customer_training_data
""")

# 特徴量エンジニアリング
feature_cols = ['total_purchases', 'avg_purchase_amount', 'days_since_last_purchase']
assembler = VectorAssembler(inputCols=feature_cols, outputCol="features_raw")
scaler = StandardScaler(inputCol="features_raw", outputCol="features")

# モデル定義
rf = RandomForestClassifier(
    featuresCol="features",
    labelCol="churned",
    numTrees=100,
    maxDepth=10
)

# パイプライン
pipeline = Pipeline(stages=[assembler, scaler, rf])

# 学習データと検証データに分割
train_df, test_df = df.randomSplit([0.8, 0.2], seed=42)

# モデル学習
model = pipeline.fit(train_df)

# 予測
predictions = model.transform(test_df)

# 評価
evaluator = BinaryClassificationEvaluator(labelCol="churned", metricName="areaUnderROC")
auc = evaluator.evaluate(predictions)
print(f"AUC: {auc}")

# モデル保存
model.write().overwrite().save("s3://my-bucket/models/churn-model")
```

#### 2. 推論結果をS3 Tablesに保存

```python
# 全顧客に対して推論
all_customers = spark.sql("""
    SELECT 
        customer_id,
        total_purchases,
        avg_purchase_amount,
        days_since_last_purchase,
        customer_segment
    FROM s3tablescatalog.my_namespace.customers
""")

# 推論実行
predictions = model.transform(all_customers)

# 結果を整形
from pyspark.sql.functions import col, current_timestamp

results = predictions.select(
    col("customer_id"),
    col("prediction").alias("churn_prediction"),
    col("probability").getItem(1).alias("churn_probability"),
    current_timestamp().alias("prediction_timestamp")
)

# S3 Tablesに保存
results.writeTo("s3tablescatalog.my_namespace.churn_predictions") \
    .using("iceberg") \
    .createOrReplace()
```

#### 3. ストリーミングML推論

```python
from pyspark.sql.functions import from_json, col
from pyspark.sql.types import *

# スキーマ定義
schema = StructType([
    StructField("customer_id", IntegerType()),
    StructField("total_purchases", DoubleType()),
    StructField("avg_purchase_amount", DoubleType()),
    StructField("days_since_last_purchase", IntegerType()),
    StructField("customer_segment", StringType())
])

# Kinesisからストリーミングデータ読み取り
stream_df = spark.readStream \
    .format("kinesis") \
    .option("streamName", "customer-events") \
    .option("region", "us-east-1") \
    .option("initialPosition", "TRIM_HORIZON") \
    .load()

# JSONパース
parsed_df = stream_df.selectExpr("CAST(data AS STRING) as json_data") \
    .select(from_json(col("json_data"), schema).alias("data")) \
    .select("data.*")

# モデル読み込み
from pyspark.ml import PipelineModel
model = PipelineModel.load("s3://my-bucket/models/churn-model")

# ストリーミング推論
predictions = model.transform(parsed_df)

# 結果をS3 Tablesに書き込み
query = predictions.select(
    col("customer_id"),
    col("prediction").alias("churn_prediction"),
    col("probability").getItem(1).alias("churn_probability"),
    current_timestamp().alias("prediction_timestamp")
).writeStream \
    .format("iceberg") \
    .outputMode("append") \
    .option("path", "s3tablescatalog.my_namespace.realtime_churn_predictions") \
    .option("checkpointLocation", "s3://my-bucket/checkpoints/churn") \
    .trigger(processingTime="1 minute") \
    .start()

query.awaitTermination()
```

### EMR Serverlessでの実行

```bash
# EMR Serverless Application作成
aws emr-serverless create-application \
  --name "ML Training Application" \
  --type SPARK \
  --release-label emr-7.1.0

# ジョブ実行
aws emr-serverless start-job-run \
  --application-id app-123456 \
  --execution-role-arn arn:aws:iam::123456789012:role/EMRServerlessRole \
  --job-driver '{
    "sparkSubmit": {
      "entryPoint": "s3://my-bucket/scripts/ml_training.py",
      "sparkSubmitParameters": "--conf spark.sql.catalog.s3tablescatalog=software.amazon.s3tables.iceberg.S3TablesCatalog --conf spark.executor.memory=4g --conf spark.executor.cores=2"
    }
  }' \
  --configuration-overrides '{
    "monitoringConfiguration": {
      "s3MonitoringConfiguration": {
        "logUri": "s3://my-bucket/logs/"
      }
    }
  }'
```

---

## AWS Glue ML統合

### Glue DataBrewとの統合

#### データプロファイリングと準備

```python
import boto3

databrew = boto3.client('databrew')

# DataBrewプロジェクト作成
response = databrew.create_project(
    Name='s3tables-ml-prep',
    DatasetName='customer-data',
    RecipeName='ml-feature-engineering'
)

# データセット定義（Athena経由でS3 Tablesにアクセス）
databrew.create_dataset(
    Name='customer-data',
    Format='PARQUET',
    Input={
        'DataCatalogInputDefinition': {
            'CatalogId': '123456789012',
            'DatabaseName': 's3tablescatalog',
            'TableName': 'my_namespace.customers'
        }
    }
)

# レシピ定義（特徴量エンジニアリング）
databrew.create_recipe(
    Name='ml-feature-engineering',
    Steps=[
        {
            'Action': {
                'Operation': 'FILL_WITH_NULL',
                'Parameters': {
                    'sourceColumn': 'last_purchase_date'
                }
            }
        },
        {
            'Action': {
                'Operation': 'CREATE_COLUMN',
                'Parameters': {
                    'targetColumn': 'days_since_last_purchase',
                    'expression': 'DATEDIFF(CURRENT_DATE, last_purchase_date)'
                }
            }
        }
    ]
)
```

### Glue ML Transformsとの統合

```python
import boto3

glue = boto3.client('glue')

# FindMatches ML Transform作成
response = glue.create_ml_transform(
    Name='customer-deduplication',
    InputRecordTables=[
        {
            'DatabaseName': 's3tablescatalog',
            'TableName': 'my_namespace.customers',
            'CatalogId': '123456789012'
        }
    ],
    Parameters={
        'TransformType': 'FIND_MATCHES',
        'FindMatchesParameters': {
            'PrimaryKeyColumnName': 'customer_id',
            'PrecisionRecallTradeoff': 0.5,
            'AccuracyCostTradeoff': 0.5
        }
    },
    Role='arn:aws:iam::123456789012:role/GlueMLRole'
)

# ML Transform学習
glue.start_ml_evaluation_task_run(
    TransformId=response['TransformId']
)
```

---

## ベストプラクティス

### 1. データ準備

**推奨事項**:
- データ品質チェックを実施
- 欠損値処理を適切に行う
- 外れ値を検出・処理
- 特徴量スケーリングを実施

**実装例**:
```python
from pyspark.sql.functions import col, when, isnan, count

# データ品質チェック
df.select([count(when(isnan(c) | col(c).isNull(), c)).alias(c) for c in df.columns]).show()

# 欠損値処理
df_cleaned = df.fillna({
    'total_purchases': 0,
    'avg_purchase_amount': df.agg({'avg_purchase_amount': 'mean'}).collect()[0][0]
})
```

### 2. 特徴量エンジニアリング

**推奨事項**:
- ドメイン知識を活用
- 時系列特徴量の作成
- カテゴリカル変数のエンコーディング
- 特徴量の相関分析

**実装例**:
```python
from pyspark.ml.feature import StringIndexer, OneHotEncoder

# カテゴリカル変数のエンコーディング
indexer = StringIndexer(inputCol="customer_segment", outputCol="segment_index")
encoder = OneHotEncoder(inputCol="segment_index", outputCol="segment_encoded")

# 時系列特徴量
from pyspark.sql.functions import datediff, current_date

df = df.withColumn(
    "days_since_last_purchase",
    datediff(current_date(), col("last_purchase_date"))
)
```

### 3. モデル学習

**推奨事項**:
- 適切な学習/検証/テストデータ分割
- クロスバリデーション実施
- ハイパーパラメータチューニング
- モデルのバージョン管理

**実装例**:
```python
from pyspark.ml.tuning import CrossValidator, ParamGridBuilder

# パラメータグリッド
paramGrid = ParamGridBuilder() \
    .addGrid(rf.numTrees, [50, 100, 200]) \
    .addGrid(rf.maxDepth, [5, 10, 15]) \
    .build()

# クロスバリデーション
cv = CrossValidator(
    estimator=pipeline,
    estimatorParamMaps=paramGrid,
    evaluator=evaluator,
    numFolds=5
)

# 学習
cv_model = cv.fit(train_df)
```

### 4. モデル推論

**推奨事項**:
- バッチ推論とリアルタイム推論の使い分け
- 推論結果のモニタリング
- モデルドリフトの検出
- A/Bテストの実施

**実装例**:
```python
# バッチ推論
batch_predictions = model.transform(batch_df)

# 推論結果の保存
batch_predictions.writeTo("s3tablescatalog.my_namespace.predictions") \
    .using("iceberg") \
    .partitionedBy("prediction_date") \
    .createOrReplace()
```

### 5. パフォーマンス最適化

**推奨事項**:
- データのパーティショニング
- 適切なファイルサイズ
- キャッシング戦略
- 並列処理の最適化

**実装例**:
```python
# データキャッシング
df.cache()

# パーティション最適化
df.repartition(100, "customer_segment")

# ブロードキャスト結合
from pyspark.sql.functions import broadcast
result = large_df.join(broadcast(small_df), "customer_id")
```

### 6. コスト最適化

**推奨事項**:
- Spot Instancesの活用
- 適切なインスタンスタイプ選択
- 自動スケーリング設定
- 不要なデータの削除

**実装例**:
```bash
# EMR Serverless with Spot
aws emr-serverless start-job-run \
  --application-id app-123456 \
  --execution-role-arn arn:aws:iam::123456789012:role/EMRServerlessRole \
  --job-driver '{...}' \
  --configuration-overrides '{
    "applicationConfiguration": [{
      "classification": "spark-defaults",
      "properties": {
        "spark.executor.instances": "10",
        "spark.executor.memory": "4g"
      }
    }]
  }'
```

---

## モニタリングとトラブルシューティング

### CloudWatch Metrics

#### SageMaker Metrics

**主要メトリクス**:
- `ModelLatency`: モデル推論レイテンシ
- `Invocations`: 推論リクエスト数
- `ModelSetupTime`: モデルセットアップ時間
- `CPUUtilization`: CPU使用率
- `MemoryUtilization`: メモリ使用率

**アラーム設定**:
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name sagemaker-high-latency \
  --alarm-description "SageMaker endpoint latency is high" \
  --metric-name ModelLatency \
  --namespace AWS/SageMaker \
  --statistic Average \
  --period 300 \
  --threshold 1000 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=EndpointName,Value=customer-churn-endpoint \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:my-sns-topic
```

#### EMR Metrics

**主要メトリクス**:
- `AppsRunning`: 実行中のアプリケーション数
- `AppsPending`: 待機中のアプリケーション数
- `ContainerAllocated`: 割り当てられたコンテナ数
- `MemoryAvailableMB`: 利用可能メモリ

### トラブルシューティング

#### 一般的な問題と解決策

**問題1: メモリ不足エラー**

**原因**:
- データサイズが大きすぎる
- パーティション数が少ない
- キャッシュの過剰使用

**解決策**:
```python
# パーティション数を増やす
df = df.repartition(200)

# メモリ設定を調整
spark.conf.set("spark.executor.memory", "8g")
spark.conf.set("spark.driver.memory", "4g")

# 不要なキャッシュをクリア
spark.catalog.clearCache()
```

**問題2: 推論レイテンシが高い**

**原因**:
- モデルサイズが大きい
- インスタンスタイプが不適切
- バッチサイズが小さい

**解決策**:
```python
# バッチ推論を使用
batch_size = 100
predictions = model.transform(df.limit(batch_size))

# より高性能なインスタンスを使用
# ml.m5.xlarge → ml.c5.2xlarge
```

**問題3: データ品質の問題**

**原因**:
- 欠損値
- 外れ値
- スキーマの不一致

**解決策**:
```python
# データ品質チェック
from pyspark.sql.functions import col, isnan, when, count

df.select([
    count(when(isnan(c) | col(c).isNull(), c)).alias(c) 
    for c in df.columns
]).show()

# 外れ値検出
from pyspark.sql.functions import percentile_approx

quantiles = df.approxQuantile("total_purchases", [0.25, 0.75], 0.05)
iqr = quantiles[1] - quantiles[0]
lower_bound = quantiles[0] - 1.5 * iqr
upper_bound = quantiles[1] + 1.5 * iqr

df_cleaned = df.filter(
    (col("total_purchases") >= lower_bound) & 
    (col("total_purchases") <= upper_bound)
)
```

---

## 出典

### AWS公式ドキュメント

1. **Offline store - Amazon SageMaker**
   - URL: https://docs.aws.amazon.com/sagemaker/latest/dg/feature-store-storage-configurations-offline-store.html
   - アクセス日: 2025-11-15
   - 内容: Feature StoreのオフラインストアとIceberg統合

2. **Amazon SageMaker Feature Store offline store data format**
   - URL: https://docs.aws.amazon.com/sagemaker/latest/dg/feature-store-offline.html
   - アクセス日: 2025-11-15
   - 内容: Feature StoreのIcebergフォーマットサポート

3. **Support for the Apache Iceberg open standard in the lakehouse architecture**
   - URL: https://docs.aws.amazon.com/sagemaker-lakehouse-architecture/latest/userguide/lakehouse-iceberg.html
   - アクセス日: 2025-11-15
   - 内容: SageMaker LakehouseのIceberg統合

4. **Use Machine Learning (ML) with Amazon Athena**
   - URL: https://docs.aws.amazon.com/athena/latest/ug/querying-mlmodel.html
   - アクセス日: 2025-11-15
   - 内容: Athena MLの使用方法

5. **Train a Model with Amazon SageMaker**
   - URL: https://docs.aws.amazon.com/sagemaker/latest/dg/how-it-works-training.html
   - アクセス日: 2025-11-15
   - 内容: SageMaker Trainingの概要

6. **Create, store, and share features with Feature Store**
   - URL: https://docs.aws.amazon.com/sagemaker/latest/dg/feature-store.html
   - アクセス日: 2025-11-15
   - 内容: Feature Storeの概要

7. **Use Feature Store with SDK for Python (Boto3)**
   - URL: https://docs.aws.amazon.com/sagemaker/latest/dg/feature-store-create-feature-group.html
   - アクセス日: 2025-11-15
   - 内容: Feature Store SDK使用方法

8. **Export - Amazon SageMaker AI**
   - URL: https://docs.aws.amazon.com/sagemaker/latest/dg/data-wrangler-data-export.html
   - アクセス日: 2025-11-15
   - 内容: Data Wranglerのエクスポート機能

### 関連ドキュメント
-
- [パフォーマンス最適化](../02-operations/02-performance-optimization.md): パフォーマンス最適化
- [ストリーミングデータ取り込み](03-streaming-data-ingestion.md): ストリーミングデータ取り込み

## 更新履歴
- 2025-11-15: 初版作成
  - SageMaker統合（Feature Store、Training、Lakehouse、Data Wrangler）
  - Athena ML統合
  - EMR Spark ML統合
  - AWS Glue ML統合
  - ベストプラクティス
  - モニタリングとトラブルシューティング
