# Amazon S3 Tables - ストリーミングデータ取り込み


## 目次

- [概要](#概要)
  - [ストリーミングデータ取り込みの重要性](#ストリーミングデータ取り込みの重要性)
- [ストリーミングデータ取り込みアーキテクチャ](#ストリーミングデータ取り込みアーキテクチャ)
  - [アーキテクチャパターン](#アーキテクチャパターン)
    - [パターン1: Amazon Data Firehose（推奨）](#パターン1-amazon-data-firehose推奨)
    - [パターン2: Spark Structured Streaming](#パターン2-spark-structured-streaming)
    - [パターン3: カスタムアプリケーション](#パターン3-カスタムアプリケーション)
- [パターン1: Amazon Data Firehose統合](#パターン1-amazon-data-firehose統合)
  - [概要](#概要)
  - [主要機能](#主要機能)
    - [1. Exactly-once配信保証](#1-exactly-once配信保証)
    - [2. マルチテーブルルーティング](#2-マルチテーブルルーティング)
    - [3. データ変換](#3-データ変換)
    - [4. CRUD操作サポート](#4-crud操作サポート)
  - [データソース統合](#データソース統合)
    - [Kinesis Data Streams](#kinesis-data-streams)
    - [Amazon MSK（Kafka）](#amazon-mskkafka)
    - [AWS IoT Core](#aws-iot-core)
    - [CloudWatch Logs](#cloudwatch-logs)
    - [Direct PUT（SDK/API）](#direct-putsdkapi)
  - [パフォーマンス最適化](#パフォーマンス最適化)
    - [バッファ設定](#バッファ設定)
    - [スループット最適化](#スループット最適化)
- [パターン2: Spark Structured Streaming](#パターン2-spark-structured-streaming)
  - [概要](#概要)
  - [アーキテクチャ](#アーキテクチャ)
  - [Kinesis Data Streamsからの読み取り](#kinesis-data-streamsからの読み取り)
  - [MSK（Kafka）からの読み取り](#mskkafkaからの読み取り)
  - [データ変換](#データ変換)
  - [EMRでの実行](#emrでの実行)
  - [AWS Glueでの実行](#aws-glueでの実行)
- [パターン3: カスタムアプリケーション](#パターン3-カスタムアプリケーション)
  - [概要](#概要)
  - [Python実装例](#python実装例)
  - [Java実装例](#java実装例)
- [モニタリングとトラブルシューティング](#モニタリングとトラブルシューティング)
  - [CloudWatch Metrics](#cloudwatch-metrics)
    - [Firehoseメトリクス](#firehoseメトリクス)
    - [Sparkメトリクス](#sparkメトリクス)
  - [トラブルシューティング](#トラブルシューティング)
    - [一般的な問題と解決策](#一般的な問題と解決策)
- [ベストプラクティス](#ベストプラクティス)
  - [1. データソース選択](#1-データソース選択)
  - [2. バッファ設定](#2-バッファ設定)
  - [3. エラーハンドリング](#3-エラーハンドリング)
  - [4. セキュリティ](#4-セキュリティ)
  - [5. パフォーマンス](#5-パフォーマンス)
  - [6. コスト最適化](#6-コスト最適化)
- [出典](#出典)
  - [AWS公式ドキュメント](#aws公式ドキュメント)
  - [関連ドキュメント](#関連ドキュメント)
- [更新履歴](#更新履歴)

## 概要

Amazon S3 Tablesは、リアルタイムストリーミングデータを効率的に取り込むための複数の方法を提供します。本ドキュメントでは、ストリーミングデータ取り込みのアーキテクチャ、実装方法、ベストプラクティスを詳細に解説します。

### ストリーミングデータ取り込みの重要性

**リアルタイム分析の実現**
- 秒単位でのデータ可視化
- 即座な意思決定
- リアルタイムアラート

**データ鮮度の向上**
- 最新データへの即座のアクセス
- バッチ処理の遅延削減
- ビジネスインサイトの迅速化

**スケーラビリティ**
- 大量データの連続処理
- 自動スケーリング
- コスト効率の向上

---

## ストリーミングデータ取り込みアーキテクチャ

### アーキテクチャパターン

S3 Tablesへのストリーミングデータ取り込みは、以下の3つの主要パターンで実装できます。

#### パターン1: Amazon Data Firehose（推奨）

**アーキテクチャ**:
```text
データソース → Firehose → S3 Tables
```

**特徴**:
- **マネージドサービス**: サーバーレス、自動スケーリング
- **Exactly-once配信**: 重複排除機能
- **データ変換**: Lambda関数による変換
- **マルチテーブルルーティング**: 動的ルーティング

**サポートデータソース**（20以上）:
- Amazon Kinesis Data Streams
- Amazon MSK（Kafka）
- AWS IoT Core
- CloudWatch Logs
- Direct PUT（SDK/API）
- その他のAWSサービス

**ユースケース**:
- リアルタイムログ分析
- IoTデータ収集
- クリックストリーム分析
- CDC（Change Data Capture）パイプライン

#### パターン2: Spark Structured Streaming

**アーキテクチャ**:
```text
Kinesis/MSK → Spark Structured Streaming → S3 Tables
```

**特徴**:
- **柔軟性**: 複雑なデータ変換
- **統合処理**: ストリーミング + バッチ
- **スケーラビリティ**: EMR/Glueでの実行

**サポートデータソース**:
- Amazon Kinesis Data Streams
- Amazon MSK（Kafka）
- その他のKafka互換ソース

**ユースケース**:
- 複雑なデータ変換が必要な場合
- ストリーミングとバッチの統合処理
- 既存Sparkパイプラインの拡張

#### パターン3: カスタムアプリケーション

**アーキテクチャ**:
```text
データソース → カスタムアプリ → S3 Tables（Iceberg API）
```

**特徴**:
- **完全制御**: すべての処理をカスタマイズ
- **Iceberg API**: 直接Iceberg APIを使用
- **柔軟性**: 任意のプログラミング言語

**サポートデータソース**:
- 任意のデータソース

**ユースケース**:
- 特殊な要件がある場合
- 既存システムとの統合
- 高度なカスタマイズが必要な場合

---

## パターン1: Amazon Data Firehose統合

### 概要

Amazon Data Firehoseは、S3 Tablesへのストリーミングデータ取り込みに最も推奨される方法です。

### 主要機能

#### 1. Exactly-once配信保証

**仕組み**:
- トランザクションID管理
- 重複検出と排除
- 自動リトライ

**メリット**:
- データ品質の保証
- 重複データの排除
- 信頼性の高い配信

#### 2. マルチテーブルルーティング

**仕組み**:
- レコード内容に基づく動的ルーティング
- 単一Streamから複数テーブルへ配信
- JSONパスによるルーティング

**設定例**:
```json
{
  "dynamicPartitioningConfiguration": {
    "enabled": true
  },
  "processingConfiguration": {
    "enabled": true,
    "processors": [
      {
        "type": "MetadataExtraction",
        "parameters": [
          {
            "parameterName": "JsonParsingEngine",
            "parameterValue": "JQ-1.6"
          },
          {
            "parameterName": "MetadataExtractionQuery",
            "parameterValue": "{table_name: .event_type}"
          }
        ]
      }
    ]
  }
}
```

#### 3. データ変換

**Lambda関数による変換**:
```python
import json
import base64

def lambda_handler(event, context):
    output = []

    for record in event['records']:
        # Base64デコード
        payload = base64.b64decode(record['data']).decode('utf-8')
        data = json.loads(payload)

        # データ変換
        transformed_data = {
            'user_id': data['userId'],
            'event_type': data['eventType'],
            'timestamp': data['timestamp'],
            'properties': json.dumps(data.get('properties', {}))
        }

        # Base64エンコード
        output_record = {
            'recordId': record['recordId'],
            'result': 'Ok',
            'data': base64.b64encode(
                json.dumps(transformed_data).encode('utf-8')
            ).decode('utf-8')
        }
        output.append(output_record)

    return {'records': output}
```

#### 4. CRUD操作サポート

**Insert操作**:
```json
{
  "user_id": 123,
  "name": "Alice",
  "email": "alice@example.com",
  "created_at": "2024-01-15T10:00:00Z"
}
```

**Update操作（Upsert）**:
```json
{
  "user_id": 123,
  "name": "Alice Smith",
  "email": "alice.smith@example.com",
  "updated_at": "2024-01-16T10:00:00Z",
  "_operation": "update"
}
```

**Delete操作**:
```json
{
  "user_id": 123,
  "_operation": "delete"
}
```

### データソース統合

#### Kinesis Data Streams

**アーキテクチャ**:
```
Producer → Kinesis Data Streams → Firehose → S3 Tables
```

**設定例**:
```bash
# Kinesis Data Stream作成
aws kinesis create-stream \
  --stream-name my-stream \
  --shard-count 2

# Firehose Delivery Stream作成（Kinesisソース）
aws firehose create-delivery-stream \
  --delivery-stream-name my-firehose-stream \
  --delivery-stream-type KinesisStreamAsSource \
  --kinesis-stream-source-configuration \
    KinesisStreamARN=arn:aws:kinesis:us-east-1:123456789012:stream/my-stream,\
    RoleARN=arn:aws:iam::123456789012:role/FirehoseRole \
  --iceberg-destination-configuration file://iceberg-config.json
```

**メリット**:
- 複数コンシューマーのサポート
- データ保持期間の設定（最大365日）
- Enhanced Fan-Out機能

**ユースケース**:
- 複数の処理パイプライン
- データの再処理が必要な場合
- リアルタイム分析とバッチ処理の併用

#### Amazon MSK（Kafka）

**アーキテクチャ**:
```text
Producer → MSK → Firehose → S3 Tables
```

**設定例**:
```bash
# Firehose Delivery Stream作成（MSKソース）
aws firehose create-delivery-stream \
  --delivery-stream-name my-msk-firehose \
  --delivery-stream-type MSKAsSource \
  --msk-source-configuration \
    MSKClusterARN=arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/...,\
    TopicName=my-topic,\
    AuthenticationConfiguration={RoleARN=arn:aws:iam::123456789012:role/FirehoseRole} \
  --iceberg-destination-configuration file://iceberg-config.json
```

**制限事項**:
- MSK Serverless非サポート
- MSK Provisioned/Serverless v2のみサポート

**メリット**:
- Kafkaエコシステムとの互換性
- 高スループット
- 複数トピックのサポート

**ユースケース**:
- 既存Kafkaパイプラインの拡張
- マイクロサービスアーキテクチャ
- イベント駆動アーキテクチャ

#### AWS IoT Core

**アーキテクチャ**:
```
IoTデバイス → IoT Core → Firehose → S3 Tables
```

**IoT Ruleの設定例**:
```json
{
  "sql": "SELECT * FROM 'iot/sensors/+'",
  "actions": [
    {
      "firehose": {
        "deliveryStreamName": "my-iot-firehose",
        "roleArn": "arn:aws:iam::123456789012:role/IoTFirehoseRole"
      }
    }
  ]
}
```

**メリット**:
- IoTデバイスとの直接統合
- デバイス管理機能
- セキュアな通信（MQTT over TLS）

**ユースケース**:
- IoTセンサーデータ収集
- テレメトリデータ分析
- デバイスモニタリング

#### CloudWatch Logs

**アーキテクチャ**:
```text
アプリケーション → CloudWatch Logs → Firehose → S3 Tables
```

**サブスクリプションフィルター設定**:
```bash
aws logs put-subscription-filter \
  --log-group-name /aws/lambda/my-function \
  --filter-name my-filter \
  --filter-pattern "[timestamp, request_id, level, msg]" \
  --destination-arn arn:aws:firehose:us-east-1:123456789012:deliverystream/my-firehose
```

**メリット**:
- アプリケーションログの自動収集
- フィルタリング機能
- 複数ログソースの統合

**ユースケース**:
- アプリケーションログ分析
- セキュリティ監査
- トラブルシューティング

#### Direct PUT（SDK/API）

**アーキテクチャ**:
```
アプリケーション → Firehose API → S3 Tables
```

**Python SDK例**:
```python
import boto3
import json

firehose = boto3.client('firehose')

def send_record(data):
    response = firehose.put_record(
        DeliveryStreamName='my-firehose-stream',
        Record={
            'Data': json.dumps(data).encode('utf-8')
        }
    )
    return response

# 使用例
data = {
    'user_id': 123,
    'event_type': 'page_view',
    'timestamp': '2024-01-15T10:00:00Z',
    'page_url': '/products/123'
}

send_record(data)
```

**バッチ送信例**:
```python
def send_batch(records):
    response = firehose.put_record_batch(
        DeliveryStreamName='my-firehose-stream',
        Records=[
            {'Data': json.dumps(record).encode('utf-8')}
            for record in records
        ]
    )
    return response

# 使用例（最大500レコード）
records = [
    {'user_id': i, 'event': 'login'}
    for i in range(100)
]

send_batch(records)
```

**メリット**:
- シンプルな統合
- 低レイテンシ
- 直接制御

**ユースケース**:
- カスタムアプリケーション
- マイクロサービス
- サーバーレスアプリケーション

### パフォーマンス最適化

#### バッファ設定

**推奨設定**:

| ユースケース | バッファサイズ | バッファインターバル | 理由 |
|------------|--------------|-------------------|------|
| リアルタイム分析 | 1-5 MiB | 60-120秒 | 低レイテンシ優先 |
| バッチ処理 | 64-128 MiB | 300-900秒 | スループット優先 |
| バランス型 | 16-32 MiB | 180-300秒 | レイテンシとスループットのバランス |

**設定例**:
```json
{
  "bufferingHints": {
    "sizeInMBs": 16,
    "intervalInSeconds": 180
  }
}
```

#### スループット最適化

**AppendOnlyフラグ**:
- Insert専用の場合に設定
- 自動スケールが有効化
- Update/Delete操作は使用不可

**設定例**:
```bash
aws firehose create-delivery-stream \
  --delivery-stream-name my-stream \
  --iceberg-destination-configuration \
    DestinationTableConfigurationList=[{
      TableName=my_table,
      AppendOnly=true
    }]
```

**リージョン別スループット制限**:

| リージョン | スループット制限 |
|-----------|----------------|
| US East (N. Virginia) | 5 MiB/秒 |
| US West (Oregon) | 5 MiB/秒 |
| Europe (Ireland) | 5 MiB/秒 |
| その他のリージョン | 1 MiB/秒 |


---

## パターン2: Spark Structured Streaming

### 概要

Spark Structured Streamingを使用して、Kinesis Data StreamsやMSKからデータを読み取り、S3 Tablesに書き込むことができます。

### アーキテクチャ

```
Kinesis/MSK → Spark Structured Streaming → S3 Tables
```

### Kinesis Data Streamsからの読み取り

**Spark設定**:
```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("KinesisToS3Tables") \
    .config("spark.sql.catalog.s3tablescatalog", "software.amazon.s3tables.iceberg.S3TablesCatalog") \
    .config("spark.sql.catalog.s3tablescatalog.warehouse", "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket") \
    .getOrCreate()
```

**ストリーム読み取り**:
```python
# Kinesis Data Streamsから読み取り
kinesis_df = spark.readStream \
    .format("kinesis") \
    .option("streamName", "my-stream") \
    .option("region", "us-east-1") \
    .option("initialPosition", "TRIM_HORIZON") \
    .option("format", "json") \
    .load()

# スキーマ定義
from pyspark.sql.types import *

schema = StructType([
    StructField("user_id", IntegerType(), False),
    StructField("event_type", StringType(), False),
    StructField("timestamp", TimestampType(), False),
    StructField("properties", StringType(), True)
])

# JSONパース
parsed_df = kinesis_df.selectExpr("CAST(data AS STRING) as json_data") \
    .select(from_json(col("json_data"), schema).alias("data")) \
    .select("data.*")
```

**S3 Tablesへの書き込み**:
```python
# ストリーミング書き込み
query = parsed_df.writeStream \
    .format("iceberg") \
    .outputMode("append") \
    .option("path", "s3tablescatalog.my_namespace.events") \
    .option("checkpointLocation", "s3://my-bucket/checkpoints/events") \
    .trigger(processingTime="1 minute") \
    .start()

query.awaitTermination()
```

### MSK（Kafka）からの読み取り

**ストリーム読み取り**:
```python
# MSKから読み取り
kafka_df = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "b-1.my-cluster.kafka.us-east-1.amazonaws.com:9092") \
    .option("subscribe", "my-topic") \
    .option("startingOffsets", "earliest") \
    .load()

# データ変換
parsed_df = kafka_df.selectExpr("CAST(value AS STRING) as json_data") \
    .select(from_json(col("json_data"), schema).alias("data")) \
    .select("data.*")
```

**S3 Tablesへの書き込み（パーティション付き）**:
```python
query = parsed_df.writeStream \
    .format("iceberg") \
    .outputMode("append") \
    .option("path", "s3tablescatalog.my_namespace.events") \
    .option("checkpointLocation", "s3://my-bucket/checkpoints/events") \
    .partitionBy("event_date") \
    .trigger(processingTime="1 minute") \
    .start()
```

### データ変換

**ウィンドウ集計**:
```python
from pyspark.sql.functions import window, count, avg

# 1分ごとの集計
windowed_df = parsed_df \
    .withWatermark("timestamp", "10 minutes") \
    .groupBy(
        window("timestamp", "1 minute"),
        "event_type"
    ) \
    .agg(
        count("*").alias("event_count"),
        avg("response_time").alias("avg_response_time")
    )

# S3 Tablesへの書き込み
query = windowed_df.writeStream \
    .format("iceberg") \
    .outputMode("append") \
    .option("path", "s3tablescatalog.my_namespace.event_metrics") \
    .option("checkpointLocation", "s3://my-bucket/checkpoints/metrics") \
    .trigger(processingTime="1 minute") \
    .start()
```

### EMRでの実行

**EMR Serverless**:
```bash
aws emr-serverless start-job-run \
  --application-id app-123456 \
  --execution-role-arn arn:aws:iam::123456789012:role/EMRServerlessRole \
  --job-driver '{
    "sparkSubmit": {
      "entryPoint": "s3://my-bucket/scripts/streaming_job.py",
      "sparkSubmitParameters": "--conf spark.sql.catalog.s3tablescatalog=software.amazon.s3tables.iceberg.S3TablesCatalog"
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

**EMR on EC2**:
```bash
aws emr create-cluster \
  --name "Streaming to S3 Tables" \
  --release-label emr-7.1.0 \
  --applications Name=Spark \
  --ec2-attributes KeyName=my-key \
  --instance-type m5.xlarge \
  --instance-count 3 \
  --use-default-roles \
  --steps Type=Spark,Name="Streaming Job",ActionOnFailure=CONTINUE,Args=[--deploy-mode,cluster,--master,yarn,s3://my-bucket/scripts/streaming_job.py]
```

### AWS Glueでの実行

**Glue Streaming Job**:
```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job

args = getResolvedOptions(sys.argv, ['JOB_NAME'])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# Kinesisから読み取り
kinesis_df = spark.readStream \
    .format("kinesis") \
    .option("streamName", "my-stream") \
    .option("region", "us-east-1") \
    .option("initialPosition", "TRIM_HORIZON") \
    .load()

# データ変換
transformed_df = kinesis_df.selectExpr("CAST(data AS STRING) as json_data") \
    .select(from_json(col("json_data"), schema).alias("data")) \
    .select("data.*")

# S3 Tablesへの書き込み
query = transformed_df.writeStream \
    .format("iceberg") \
    .outputMode("append") \
    .option("path", "s3tablescatalog.my_namespace.events") \
    .option("checkpointLocation", "s3://my-bucket/checkpoints/events") \
    .trigger(processingTime="1 minute") \
    .start()

query.awaitTermination()
job.commit()
```

---

## パターン3: カスタムアプリケーション

### 概要

Iceberg APIを直接使用して、カスタムアプリケーションからS3 Tablesにデータを書き込むことができます。

### Python実装例

**PyIcebergを使用**:
```python
from pyiceberg.catalog import load_catalog
from pyiceberg.table import Table
import json

# カタログ接続
catalog = load_catalog(
    "s3tables",
    **{
        "type": "rest",
        "uri": "https://s3tables.us-east-1.amazonaws.com",
        "warehouse": "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket"
    }
)

# テーブル取得
table = catalog.load_table("my_namespace.events")

# データ書き込み
def write_records(records):
    # レコードをDataFrameに変換
    import pandas as pd
    df = pd.DataFrame(records)
    
    # Icebergテーブルに書き込み
    table.append(df)

# 使用例
records = [
    {
        'user_id': 123,
        'event_type': 'page_view',
        'timestamp': '2024-01-15T10:00:00Z'
    },
    {
        'user_id': 456,
        'event_type': 'click',
        'timestamp': '2024-01-15T10:01:00Z'
    }
]

write_records(records)
```

### Java実装例

**Iceberg Java APIを使用**:
```java
import org.apache.iceberg.catalog.Catalog;
import org.apache.iceberg.catalog.TableIdentifier;
import org.apache.iceberg.Table;
import org.apache.iceberg.data.GenericRecord;
import org.apache.iceberg.data.parquet.GenericParquetWriter;
import org.apache.iceberg.io.OutputFile;

public class S3TablesWriter {
    private Catalog catalog;
    private Table table;

    public void writeRecords(List<Map<String, Object>> records) {
        TableIdentifier tableId = TableIdentifier.of("my_namespace", "events");
        table = catalog.loadTable(tableId);

        OutputFile outputFile = table.io().newOutputFile(
            table.location() + "/data/file-" + System.currentTimeMillis() + ".parquet"
        );

        try (GenericParquetWriter writer = new GenericParquetWriter(
                outputFile, table.schema())) {
            for (Map<String, Object> record : records) {
                GenericRecord genericRecord = GenericRecord.create(table.schema());
                record.forEach(genericRecord::setField);
                writer.write(genericRecord);
            }
        }
    }
}
```

---

## モニタリングとトラブルシューティング

### CloudWatch Metrics

#### Firehoseメトリクス

**主要メトリクス**:

| メトリクス | 説明 | 推奨閾値 |
|-----------|------|---------|
| `IncomingRecords` | 入力レコード数 | - |
| `IncomingBytes` | 入力バイト数 | - |
| `DeliveryToS3.Success` | 配信成功率 | > 99% |
| `DeliveryToS3.Records` | 配信レコード数 | - |
| `DeliveryToS3.DataFreshness` | データ鮮度（秒） | < 300秒 |
| `DeliveryToS3.Bytes` | 配信バイト数 | - |

**アラーム設定例**:
```bash
# 配信成功率のアラーム
aws cloudwatch put-metric-alarm \
  --alarm-name firehose-delivery-success-rate \
  --alarm-description "Firehose delivery success rate below 99%" \
  --metric-name DeliveryToS3.Success \
  --namespace AWS/Firehose \
  --statistic Average \
  --period 300 \
  --threshold 99 \
  --comparison-operator LessThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=DeliveryStreamName,Value=my-firehose-stream \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:my-sns-topic

# データ鮮度のアラーム
aws cloudwatch put-metric-alarm \
  --alarm-name firehose-data-freshness \
  --alarm-description "Firehose data freshness exceeds 5 minutes" \
  --metric-name DeliveryToS3.DataFreshness \
  --namespace AWS/Firehose \
  --statistic Maximum \
  --period 300 \
  --threshold 300 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=DeliveryStreamName,Value=my-firehose-stream \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:my-sns-topic
```

#### Sparkメトリクス

**主要メトリクス**:
- `streaming.lastReceivedBatch_records`: 最後に受信したバッチのレコード数
- `streaming.lastCompletedBatch_processingTime`: 最後に完了したバッチの処理時間
- `streaming.lastCompletedBatch_inputRowsPerSecond`: 入力行数/秒
- `streaming.lastCompletedBatch_processedRowsPerSecond`: 処理行数/秒

### トラブルシューティング

#### 一般的な問題と解決策

**問題1: 配信失敗率が高い**

**原因**:
- IAM権限不足
- KMS権限不足
- ネットワーク問題
- スキーマ不一致

**解決策**:
```bash
# CloudTrailログ確認
aws logs filter-log-events \
  --log-group-name /aws/kinesisfirehose/my-firehose-stream \
  --filter-pattern "ERROR" \
  --start-time $(date -u -d '1 hour ago' +%s)000

# エラーバケット確認
aws s3 ls s3://my-error-bucket/errors/ --recursive
```

**問題2: データ鮮度が悪い**

**原因**:
- バッファ設定が大きすぎる
- スループット制限
- 処理遅延

**解決策**:
- バッファサイズを小さくする（1-5 MiB）
- バッファインターバルを短くする（60-120秒）
- AppendOnlyフラグを有効化

**問題3: スキーマエラー**

**原因**:
- カラム名の大文字小文字混在
- データ型の不一致
- 必須カラムの欠落

**解決策**:
```python
# Lambda関数でスキーマ検証
def validate_schema(data):
    required_fields = ['user_id', 'event_type', 'timestamp']
    
    # 必須フィールド確認
    for field in required_fields:
        if field not in data:
            raise ValueError(f"Missing required field: {field}")
    
    # カラム名を小文字に変換
    normalized_data = {k.lower(): v for k, v in data.items()}
    
    return normalized_data
```

---

## ベストプラクティス

### 1. データソース選択

**推奨事項**:
- **Firehose**: ほとんどのユースケースに推奨
- **Spark Structured Streaming**: 複雑な変換が必要な場合
- **カスタムアプリケーション**: 特殊な要件がある場合

### 2. バッファ設定

**推奨事項**:
- リアルタイム分析: 小さいバッファ（1-5 MiB、60-120秒）
- バッチ処理: 大きいバッファ（64-128 MiB、300-900秒）
- バランス型: 中程度のバッファ（16-32 MiB、180-300秒）

### 3. エラーハンドリング

**推奨事項**:
- エラーバケットを設定
- CloudWatch Alarmsを設定
- 定期的なエラーログ確認
- 自動リトライの設定

### 4. セキュリティ

**推奨事項**:
- S3 Tables側でKMS暗号化を設定
- IAMロールに最小権限を付与
- VPCエンドポイントを使用
- CloudTrailログを有効化

### 5. パフォーマンス

**推奨事項**:
- AppendOnlyフラグを活用（Insert専用の場合）
- パーティション戦略を最適化
- 適切なバッファサイズを設定
- スループット制限を考慮

### 6. コスト最適化

**推奨事項**:
- バッファサイズを最適化（大きいほどコスト効率が良い）
- データ圧縮を有効化
- 不要なデータ変換を削減
- リージョン選択を最適化

---

## 出典

### AWS公式ドキュメント

1. **Streaming data to tables with Amazon Data Firehose**
   - URL: https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-integrating-firehose.html
   - アクセス日: 2025-11-15
   - 内容: Firehose統合の詳細

2. **What is Amazon Data Firehose?**
   - URL: https://docs.aws.amazon.com/firehose/latest/dev/what-is-this-service.html
   - アクセス日: 2025-11-15
   - 内容: Firehoseの概要と機能

3. **What is Amazon Kinesis Data Streams?**
   - URL: https://docs.aws.amazon.com/streams/latest/dev/introduction.html
   - アクセス日: 2025-11-15
   - 内容: Kinesis Data Streamsの概要

4. **Supported streaming connectors - Amazon EMR**
   - URL: https://docs.aws.amazon.com/emr/latest/EMR-Serverless-UserGuide/jobs-spark-streaming-connectors.html
   - アクセス日: 2025-11-15
   - 内容: EMRでのストリーミングコネクタ

5. **Apache Spark - Amazon Kinesis Data Streams**
   - URL: https://docs.aws.amazon.com/streams/latest/dev/using-other-services-read-spark.html
   - アクセス日: 2025-11-15
   - 内容: SparkとKinesisの統合

6. **Firehose integration for Amazon MSK**
   - URL: https://docs.aws.amazon.com/msk/latest/developerguide/integrations-kinesis-data-firehose.html
   - アクセス日: 2025-11-15
   - 内容: MSKとFirehoseの統合

7. **Data ingestion methods - Storage Best Practices**
   - URL: https://docs.aws.amazon.com/whitepapers/latest/building-data-lakes/data-ingestion-methods.html
   - アクセス日: 2025-11-15
   - 内容: データ取り込み方法のベストプラクティス

8. **Build Modern Data Streaming Architectures on AWS**
   - URL: https://docs.aws.amazon.com/whitepapers/latest/build-modern-data-streaming-analytics-architectures/what-is-a-modern-streaming-data-architecture.html
   - アクセス日: 2025-11-15
   - 内容: モダンストリーミングアーキテクチャ

### 関連ドキュメント
- [Firehose統合](02-firehose-integration.md): Firehose統合の詳細
- [セキュリティベストプラクティス](../03-security/01-security-best-practices.md): セキュリティベストプラクティス
- [パフォーマンス最適化](../02-operations/02-performance-optimization.md): パフォーマンス最適化

## 更新履歴
- 2025-11-15: 初版作成
  - ストリーミングデータ取り込みの3つのパターン
  - Firehose統合の詳細
  - Spark Structured Streamingの実装例
  - カスタムアプリケーションの実装例
  - モニタリングとトラブルシューティング
  - ベストプラクティス
