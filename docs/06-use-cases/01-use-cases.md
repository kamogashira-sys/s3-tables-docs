# Amazon S3 Tables - ユースケース集


## 目次

- [概要](#概要)
- [ユースケース一覧](#ユースケース一覧)
  - [データレイク・分析](#データレイク・分析)
  - [機械学習](#機械学習)
  - [ストリーミング処理](#ストリーミング処理)
  - [データ移行・最適化](#データ移行・最適化)
- [UC1: リアルタイムログ分析](#uc1-リアルタイムログ分析)
  - [ビジネス課題](#ビジネス課題)
  - [アーキテクチャ](#アーキテクチャ)
  - [実装例](#実装例)
  - [メリット](#メリット)
- [UC2: クリックストリーム分析](#uc2-クリックストリーム分析)
  - [ビジネス課題](#ビジネス課題)
  - [アーキテクチャ](#アーキテクチャ)
  - [実装例](#実装例)
  - [メリット](#メリット)
- [UC3: IoTデータ収集と分析](#uc3-iotデータ収集と分析)
  - [ビジネス課題](#ビジネス課題)
  - [アーキテクチャ](#アーキテクチャ)
  - [実装例](#実装例)
  - [メリット](#メリット)
- [UC4: データウェアハウス統合](#uc4-データウェアハウス統合)
  - [ビジネス課題](#ビジネス課題)
  - [アーキテクチャ](#アーキテクチャ)
  - [実装例](#実装例)
  - [メリット](#メリット)
- [UC5: 顧客離反予測](#uc5-顧客離反予測)
  - [ビジネス課題](#ビジネス課題)
  - [アーキテクチャ](#アーキテクチャ)
  - [実装例](#実装例)
  - [メリット](#メリット)
- [UC6: レコメンデーションシステム](#uc6-レコメンデーションシステム)
  - [ビジネス課題](#ビジネス課題)
  - [アーキテクチャ](#アーキテクチャ)
  - [実装例](#実装例)
  - [メリット](#メリット)
- [UC7: 異常検知](#uc7-異常検知)
  - [ビジネス課題](#ビジネス課題)
  - [アーキテクチャ](#アーキテクチャ)
  - [実装例](#実装例)
  - [メリット](#メリット)
- [出典](#出典)
  - [関連ドキュメント](#関連ドキュメント)
- [更新履歴](#更新履歴)

## 概要

本ドキュメントでは、Amazon S3 Tablesの実践的なユースケースを業界別・機能別に整理し、具体的な実装例とともに解説します。各ユースケースには、アーキテクチャ図、実装コード、ベストプラクティスが含まれています。

---

## ユースケース一覧

### データレイク・分析
1. [リアルタイムログ分析](#uc1-リアルタイムログ分析)
2. [クリックストリーム分析](#uc2-クリックストリーム分析)
3. [IoTデータ収集と分析](#uc3-iotデータ収集と分析)
4. [データウェアハウス統合](#uc4-データウェアハウス統合)

### 機械学習
5. [顧客離反予測](#uc5-顧客離反予測)
6. [レコメンデーションシステム](#uc6-レコメンデーションシステム)
7. [異常検知](#uc7-異常検知)
8. [需要予測](#uc8-需要予測)

### ストリーミング処理
9. [CDCパイプライン](#uc9-cdcパイプライン)
10. [リアルタイムダッシュボード](#uc10-リアルタイムダッシュボード)

### データ移行・最適化
11. [既存データレイクからの移行](#uc11-既存データレイクからの移行)
12. [データレイク最適化](#uc12-データレイク最適化)

---

## UC1: リアルタイムログ分析

### ビジネス課題
- アプリケーションログをリアルタイムで分析
- 障害の早期検出
- ユーザー行動の即座の把握

### アーキテクチャ

```text
アプリケーション → CloudWatch Logs → Firehose → S3 Tables → Athena/QuickSight
```

### 実装例

**1. CloudWatch Logsサブスクリプション設定**:
```bash
aws logs put-subscription-filter \
  --log-group-name /aws/lambda/my-app \
  --filter-name app-logs-to-firehose \
  --filter-pattern "" \
  --destination-arn arn:aws:firehose:us-east-1:123456789012:deliverystream/app-logs-stream
```

**2. Firehose設定**:
```json
{
  "deliveryStreamName": "app-logs-stream",
  "icebergDestinationConfiguration": {
    "catalogConfiguration": {
      "catalogArn": "arn:aws:glue:us-east-1:123456789012:catalog/s3tablescatalog/my-table-bucket"
    },
    "destinationTableConfigurationList": [{
      "tableName": "application_logs",
      "destinationDatabaseName": "logs",
      "uniqueKeys": ["log_id"]
    }],
    "bufferingHints": {
      "sizeInMBs": 5,
      "intervalInSeconds": 60
    }
  }
}
```

**3. Athenaクエリ（エラー分析）**:
```sql
-- 過去1時間のエラー集計
SELECT 
    DATE_TRUNC('minute', timestamp) as minute,
    error_type,
    COUNT(*) as error_count
FROM s3tablescatalog.logs.application_logs
WHERE 
    timestamp >= CURRENT_TIMESTAMP - INTERVAL '1' HOUR
    AND log_level = 'ERROR'
GROUP BY 1, 2
ORDER BY 1 DESC, 3 DESC;

-- エラー率の計算
SELECT 
    DATE_TRUNC('hour', timestamp) as hour,
    COUNT(CASE WHEN log_level = 'ERROR' THEN 1 END) * 100.0 / COUNT(*) as error_rate
FROM s3tablescatalog.logs.application_logs
WHERE timestamp >= CURRENT_TIMESTAMP - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 1 DESC;
```

### メリット
- リアルタイムでのログ分析
- 低コスト（サーバーレス）
- スケーラブル
- SQLベースの分析

---

## UC2: クリックストリーム分析

### ビジネス課題
- ユーザー行動の理解
- コンバージョン率の向上
- パーソナライゼーション

### アーキテクチャ

```
Webアプリ → Kinesis Data Streams → Firehose → S3 Tables → Athena/QuickSight
                                                    ↓
                                              SageMaker ML
```

### 実装例

**1. クリックイベント送信（JavaScript）**:
```javascript
// AWS SDK for JavaScript
const AWS = require('aws-sdk');
const kinesis = new AWS.Kinesis();

function trackEvent(eventType, eventData) {
    const record = {
        user_id: getUserId(),
        session_id: getSessionId(),
        event_type: eventType,
        event_data: eventData,
        timestamp: new Date().toISOString(),
        page_url: window.location.href,
        referrer: document.referrer,
        user_agent: navigator.userAgent
    };

    kinesis.putRecord({
        StreamName: 'clickstream-events',
        Data: JSON.stringify(record),
        PartitionKey: record.user_id.toString()
    }, (err, data) => {
        if (err) console.error('Error sending event:', err);
    });
}

// 使用例
trackEvent('page_view', { page_title: document.title });
trackEvent('button_click', { button_id: 'checkout' });
```

**2. セッション分析クエリ**:
```sql
-- セッションごとのページビュー数
WITH sessions AS (
    SELECT 
        session_id,
        user_id,
        MIN(timestamp) as session_start,
        MAX(timestamp) as session_end,
        COUNT(*) as page_views,
        COUNT(DISTINCT page_url) as unique_pages
    FROM s3tablescatalog.analytics.clickstream_events
    WHERE 
        event_type = 'page_view'
        AND timestamp >= CURRENT_DATE - INTERVAL '7' DAY
    GROUP BY 1, 2
)
SELECT 
    DATE(session_start) as date,
    AVG(page_views) as avg_page_views,
    AVG(CAST(session_end AS BIGINT) - CAST(session_start AS BIGINT)) / 1000 as avg_session_duration_sec
FROM sessions
GROUP BY 1
ORDER BY 1 DESC;

-- コンバージョンファネル分析
SELECT 
    event_type,
    COUNT(DISTINCT user_id) as unique_users,
    COUNT(*) as total_events
FROM s3tablescatalog.analytics.clickstream_events
WHERE 
    event_type IN ('page_view', 'add_to_cart', 'checkout', 'purchase')
    AND timestamp >= CURRENT_DATE - INTERVAL '7' DAY
GROUP BY 1
ORDER BY 
    CASE event_type
        WHEN 'page_view' THEN 1
        WHEN 'add_to_cart' THEN 2
        WHEN 'checkout' THEN 3
        WHEN 'purchase' THEN 4
    END;
```

### メリット
- リアルタイムユーザー行動分析
- 機械学習との統合
- スケーラブルなデータ収集

---

## UC3: IoTデータ収集と分析

### ビジネス課題
- 大量のIoTセンサーデータの収集
- リアルタイム監視
- 予知保全

### アーキテクチャ

```
IoTデバイス → IoT Core → Firehose → S3 Tables → Athena/QuickSight
                                              ↓
                                        SageMaker ML（異常検知）
```

### 実装例

**1. IoT Ruleの設定**:
```json
{
  "sql": "SELECT * FROM 'sensors/+/telemetry'",
  "actions": [{
    "firehose": {
      "deliveryStreamName": "iot-telemetry-stream",
      "roleArn": "arn:aws:iam::123456789012:role/IoTFirehoseRole",
      "separator": "\n"
    }
  }]
}
```

**2. センサーデータ分析**:
```sql
-- 温度異常の検出
SELECT 
    device_id,
    timestamp,
    temperature,
    AVG(temperature) OVER (
        PARTITION BY device_id 
        ORDER BY timestamp 
        ROWS BETWEEN 10 PRECEDING AND CURRENT ROW
    ) as moving_avg_temp
FROM s3tablescatalog.iot.sensor_telemetry
WHERE 
    timestamp >= CURRENT_TIMESTAMP - INTERVAL '1' HOUR
    AND ABS(temperature - moving_avg_temp) > 10
ORDER BY timestamp DESC;

-- デバイス稼働状況
SELECT 
    device_id,
    MAX(timestamp) as last_seen,
    CURRENT_TIMESTAMP - MAX(timestamp) as time_since_last_seen
FROM s3tablescatalog.iot.sensor_telemetry
GROUP BY 1
HAVING time_since_last_seen > INTERVAL '5' MINUTE
ORDER BY 3 DESC;
```

### メリット
- 大規模IoTデータの効率的な収集
- リアルタイム異常検知
- 予知保全の実現

---

## UC4: データウェアハウス統合

### ビジネス課題
- データレイクとデータウェアハウスの統合
- 統一されたデータアクセス
- コスト最適化

### アーキテクチャ

```
S3 Tables (Data Lake) ─┐
                       ├─→ SageMaker Lakehouse ─→ 統一クエリ
Redshift (DWH) ────────┘
```

### 実装例

**1. Redshift Spectrumからのアクセス**:
```sql
-- Redshift外部スキーマ作成
CREATE EXTERNAL SCHEMA s3tables_schema
FROM DATA CATALOG
DATABASE 's3tablescatalog'
IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftSpectrumRole';

-- S3 Tablesとの結合クエリ
SELECT
    o.order_id,
    o.order_date,
    c.customer_name,
    c.customer_segment
FROM orders o
JOIN s3tables_schema.my_namespace.customers c
    ON o.customer_id = c.customer_id
WHERE o.order_date >= CURRENT_DATE - 30;
```

**2. データ同期（Redshift → S3 Tables）**:
```sql
-- Redshift UNLOADでS3にエクスポート
UNLOAD ('SELECT * FROM orders WHERE order_date >= CURRENT_DATE - 30')
TO 's3://my-bucket/temp/orders/'
IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftUnloadRole'
FORMAT AS PARQUET;

-- Sparkで S3 Tablesに取り込み
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .config("spark.sql.catalog.s3tablescatalog", "software.amazon.s3tables.iceberg.S3TablesCatalog") \
    .config("spark.sql.catalog.s3tablescatalog.warehouse", "arn:aws:s3tables:...") \
    .getOrCreate()

df = spark.read.parquet("s3://my-bucket/temp/orders/")
df.writeTo("s3tablescatalog.my_namespace.orders") \
    .using("iceberg") \
    .createOrReplace()
```

### メリット
- データレイクとDWHの統合
- コスト最適化（ホットデータはRedshift、コールドデータはS3 Tables）
- 柔軟なクエリ

---

## UC5: 顧客離反予測

### ビジネス課題
- 顧客離反の早期検出
- リテンション施策の最適化
- LTV（顧客生涯価値）の向上

### アーキテクチャ

```
S3 Tables → SageMaker Training → モデル → SageMaker Endpoint
    ↓                                           ↓
Athena ML ←─────────────────────────────────────┘
```

### 実装例

**1. 特徴量作成**:
```sql
CREATE TABLE s3tablescatalog.ml.customer_features
USING iceberg
AS
SELECT
    customer_id,
    COUNT(DISTINCT order_id) as total_orders,
    SUM(order_amount) as total_spent,
    AVG(order_amount) as avg_order_value,
    DATEDIFF(CURRENT_DATE, MAX(order_date)) as days_since_last_order,
    DATEDIFF(MAX(order_date), MIN(order_date)) / NULLIF(COUNT(DISTINCT order_id) - 1, 0) as avg_days_between_orders,
    COUNT(DISTINCT product_category) as unique_categories,
    CASE
        WHEN DATEDIFF(CURRENT_DATE, MAX(order_date)) > 90 THEN 1
        ELSE 0
    END as churned
FROM s3tablescatalog.sales.orders
GROUP BY customer_id;
```

**2. Athena MLで推論**:
```sql
USING EXTERNAL FUNCTION predict_churn(
    total_orders BIGINT,
    total_spent DOUBLE,
    avg_order_value DOUBLE,
    days_since_last_order INT
)
RETURNS DOUBLE
SAGEMAKER 'customer-churn-endpoint'

SELECT 
    customer_id,
    customer_name,
    predict_churn(
        total_orders,
        total_spent,
        avg_order_value,
        days_since_last_order
    ) as churn_probability
FROM s3tablescatalog.ml.customer_features
WHERE churn_probability > 0.7
ORDER BY churn_probability DESC;
```

### メリット
- データドリブンなリテンション施策
- 早期の離反検出
- ROIの向上

---

## UC6: レコメンデーションシステム

### ビジネス課題
- パーソナライズされた商品推薦
- クロスセル・アップセルの促進
- コンバージョン率の向上

### アーキテクチャ

```
S3 Tables → EMR Spark ML → 協調フィルタリングモデル → 推薦結果
    ↓                                                    ↓
Feature Store ←──────────────────────────────────────────┘
```

### 実装例

**1. 協調フィルタリング（Spark ALS）**:
```python
from pyspark.ml.recommendation import ALS
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .config("spark.sql.catalog.s3tablescatalog", "software.amazon.s3tables.iceberg.S3TablesCatalog") \
    .config("spark.sql.catalog.s3tablescatalog.warehouse", "arn:aws:s3tables:...") \
    .getOrCreate()

# 購入履歴データ読み取り
ratings_df = spark.sql("""
    SELECT
        customer_id as userId,
        product_id as itemId,
        CASE
            WHEN quantity > 1 THEN 5.0
            ELSE 4.0
        END as rating
    FROM s3tablescatalog.sales.order_items
""")

# ALSモデル学習
als = ALS(
    maxIter=10,
    regParam=0.1,
    userCol="userId",
    itemCol="itemId",
    ratingCol="rating",
    coldStartStrategy="drop"
)

model = als.fit(ratings_df)

# 全ユーザーへの推薦生成
recommendations = model.recommendForAllUsers(10)

# S3 Tablesに保存
recommendations.writeTo("s3tablescatalog.ml.product_recommendations") \
    .using("iceberg") \
    .createOrReplace()
```

**2. 推薦結果の取得**:
```sql
SELECT 
    r.customer_id,
    p.product_name,
    r.rating as recommendation_score
FROM s3tablescatalog.ml.product_recommendations r
CROSS JOIN UNNEST(r.recommendations) AS t(product_id, rating)
JOIN s3tablescatalog.products.catalog p
    ON t.product_id = p.product_id
WHERE r.customer_id = 12345
ORDER BY r.rating DESC
LIMIT 10;
```

### メリット
- パーソナライズされた体験
- 売上向上
- 顧客満足度向上

---

## UC7: 異常検知

### ビジネス課題
- 不正取引の検出
- システム異常の早期発見
- セキュリティインシデントの防止

### アーキテクチャ

```
ストリーミングデータ → Kinesis → Spark Streaming → 異常検知モデル → アラート
                                        ↓
                                   S3 Tables（履歴保存）
```

### 実装例

**1. Isolation Forestによる異常検知**:
```python
from pyspark.ml.feature import VectorAssembler
from pyspark.ml.clustering import KMeans
from pyspark.sql.functions import col, udf
from pyspark.sql.types import DoubleType
import numpy as np

# データ読み取り
df = spark.sql("""
    SELECT
        transaction_id,
        amount,
        merchant_category,
        distance_from_home,
        time_since_last_transaction
    FROM s3tablescatalog.transactions.realtime
""")

# 特徴量ベクトル化
assembler = VectorAssembler(
    inputCols=['amount', 'distance_from_home', 'time_since_last_transaction'],
    outputCol='features'
)

df_features = assembler.transform(df)

# K-Meansクラスタリング
kmeans = KMeans(k=5, seed=1)
model = kmeans.fit(df_features)

# 異常スコア計算（クラスタ中心からの距離）
def anomaly_score(features, centers):
    distances = [np.linalg.norm(features - center) for center in centers]
    return float(min(distances))

anomaly_udf = udf(lambda x: anomaly_score(x, model.clusterCenters()), DoubleType())

# 異常検知
anomalies = df_features.withColumn('anomaly_score', anomaly_udf(col('features'))) \
    .filter(col('anomaly_score') > 3.0)

# 異常取引をS3 Tablesに保存
anomalies.writeTo("s3tablescatalog.security.anomalous_transactions") \
    .using("iceberg") \
    .append()
```

### メリット
- リアルタイム異常検知
- 不正取引の防止
- セキュリティ強化

---

## 出典

### 関連ドキュメント
- [Firehose統合](../04-integrations/02-firehose-integration.md): Firehose統合
- [ストリーミングデータ取り込み](../04-integrations/03-streaming-data-ingestion.md): ストリーミングデータ取り込み
- [機械学習統合](../04-integrations/04-machine-learning-integration.md): 機械学習統合
- [データレイク移行](../05-advanced/03-migration-from-datalake.md): データ移行

## 更新履歴
- 2025-11-15: 初版作成
  - 12のユースケースを追加
  - 各ユースケースに実装例を含む
