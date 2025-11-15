# Amazon Data Firehoseとの統合


## 目次

- [概要](#概要)
  - [主要な機能](#主要な機能)
  - [サポートされるデータソース](#サポートされるデータソース)
- [アーキテクチャ](#アーキテクチャ)
  - [基本的なデータフロー](#基本的なデータフロー)
  - [コンポーネント](#コンポーネント)
- [セットアップ手順](#セットアップ手順)
  - [前提条件](#前提条件)
  - [ステップ1: IAMサービスロールの作成](#ステップ1-iamサービスロールの作成)
    - [IAMポリシーの作成](#iamポリシーの作成)
    - [信頼ポリシー](#信頼ポリシー)
  - [ステップ2: Firehose Streamの作成](#ステップ2-firehose-streamの作成)
    - [コンソールでの作成手順](#コンソールでの作成手順)
    - [AWS CLIでの作成例](#aws-cliでの作成例)
- [データルーティング](#データルーティング)
  - [単一テーブルへのルーティング](#単一テーブルへのルーティング)
    - [Unique Key設定](#unique-key設定)
  - [複数テーブルへのルーティング](#複数テーブルへのルーティング)
    - [JQ Expressionを使用したルーティング](#jq-expressionを使用したルーティング)
    - [Lambda関数を使用したルーティング](#lambda関数を使用したルーティング)
- [データ操作](#データ操作)
  - [サポートされる操作](#サポートされる操作)
  - [操作の指定方法](#操作の指定方法)
    - [デフォルト動作（Insert）](#デフォルト動作insert)
    - [Update操作](#update操作)
    - [Delete操作](#delete操作)
  - [Unique Keyの設定](#unique-keyの設定)
    - [Firehose Stream作成時の設定](#firehose-stream作成時の設定)
    - [Icebergテーブル作成時の設定](#icebergテーブル作成時の設定)
- [データ変換](#データ変換)
  - [Lambda関数を使用したデータ変換](#lambda関数を使用したデータ変換)
    - [基本的な変換例](#基本的な変換例)
    - [エラーハンドリング](#エラーハンドリング)
- [バッファリング設定](#バッファリング設定)
  - [バッファサイズとインターバル](#バッファサイズとインターバル)
    - [設定パラメータ](#設定パラメータ)
    - [トレードオフ](#トレードオフ)
    - [推奨設定](#推奨設定)
- [リトライ設定](#リトライ設定)
  - [リトライ期間の設定](#リトライ期間の設定)
  - [リトライ動作](#リトライ動作)
- [エラーハンドリング](#エラーハンドリング)
  - [エラーバケットの設定](#エラーバケットの設定)
  - [エラータイプ](#エラータイプ)
  - [CloudWatch Logsでのモニタリング](#cloudwatch-logsでのモニタリング)
- [パフォーマンス最適化](#パフォーマンス最適化)
  - [スループット制限](#スループット制限)
    - [Direct PUTソースの制限](#direct-putソースの制限)
    - [AppendOnlyフラグ](#appendonlyフラグ)
  - [S3 TPSの最適化](#s3-tpsの最適化)
  - [コンパクション戦略](#コンパクション戦略)
    - [AWS Glue Data Catalogの自動コンパクション](#aws-glue-data-catalogの自動コンパクション)
    - [Athena OPTIMIZEコマンド](#athena-optimizeコマンド)
    - [VACUUMコマンド](#vacuumコマンド)
- [制限事項と考慮事項](#制限事項と考慮事項)
  - [リージョンサポート](#リージョンサポート)
  - [データフォーマット制限](#データフォーマット制限)
    - [ネストされたJSON](#ネストされたjson)
    - [レコードフォーマット](#レコードフォーマット)
  - [同時書き込み制限](#同時書き込み制限)
  - [Icebergライブラリバージョン](#icebergライブラリバージョン)
  - [命名規則制限](#命名規則制限)
  - [暗号化設定](#暗号化設定)
  - [パーティション制限](#パーティション制限)
  - [その他の制限](#その他の制限)
- [ユースケース](#ユースケース)
  - [リアルタイムログ分析](#リアルタイムログ分析)
    - [AWS WAFログのストリーミング](#aws-wafログのストリーミング)
  - [IoTデータの取り込み](#iotデータの取り込み)
    - [センサーデータのストリーミング](#センサーデータのストリーミング)
  - [CDCデータの取り込み](#cdcデータの取り込み)
    - [データベース変更のストリーミング](#データベース変更のストリーミング)
- [ベストプラクティス](#ベストプラクティス)
  - [1. 適切なバッファサイズの選択](#1-適切なバッファサイズの選択)
  - [2. エラーハンドリングの実装](#2-エラーハンドリングの実装)
  - [3. パーティション戦略](#3-パーティション戦略)
  - [4. コンパクションの自動化](#4-コンパクションの自動化)
  - [5. セキュリティ](#5-セキュリティ)
  - [6. モニタリングとアラート](#6-モニタリングとアラート)
    - [主要メトリクス](#主要メトリクス)
    - [CloudWatch Dashboardの作成](#cloudwatch-dashboardの作成)
  - [7. コスト最適化](#7-コスト最適化)
- [トラブルシューティング](#トラブルシューティング)
  - [一般的な問題と解決方法](#一般的な問題と解決方法)
    - [問題1: データが配信されない](#問題1-データが配信されない)
    - [問題2: スキーマ不一致エラー](#問題2-スキーマ不一致エラー)
    - [問題3: スループット制限](#問題3-スループット制限)
    - [問題4: 高レイテンシ](#問題4-高レイテンシ)
- [出典](#出典)
  - [AWS公式ドキュメント](#aws公式ドキュメント)
  - [関連リソース](#関連リソース)

## 概要

Amazon Data Firehoseは、リアルタイムストリーミングデータをAmazon S3 Tablesに配信するためのフルマネージドサービスです。Firehoseを使用することで、アプリケーションの開発やリソース管理を行うことなく、ストリーミングデータをS3 Tablesに自動的に配信できます。

### 主要な機能

- **フルマネージドサービス**: アプリケーション開発やリソース管理が不要
- **マルチソース対応**: 20以上のデータソースからのストリーミングデータ配信
- **自動データ変換**: Lambda関数を使用したデータ変換
- **マルチテーブルルーティング**: 単一ストリームから複数のIcebergテーブルへのルーティング
- **CRUD操作サポート**: Insert、Update、Delete操作の自動適用
- **Exactly-once配信保証**: データの重複や欠損を防止
- **Lake Formation統合**: きめ細かなアクセス制御

### サポートされるデータソース

- Amazon Kinesis Data Streams
- Amazon MSK（Amazon Managed Streaming for Apache Kafka）
- Direct PUT（アプリケーションから直接データを送信）
- AWS WAFログ
- Amazon CloudWatch Logs
- AWS IoT
- その他20以上のソース

## アーキテクチャ

### 基本的なデータフロー

```text
データソース → Firehose Stream → AWS Glue Data Catalog → S3 Tables
                      ↓
                Lambda変換（オプション）
                      ↓
                バッファリング
                      ↓
                エラーバケット（失敗時）
```

### コンポーネント

1. **データソース**: ストリーミングデータの送信元
2. **Firehose Stream**: データの受信、バッファリング、配信を管理
3. **AWS Glue Data Catalog**: Icebergテーブルのメタデータ管理
4. **S3 Tables**: データの最終的な保存先
5. **Lambda関数（オプション）**: データ変換とルーティング
6. **S3エラーバケット**: 配信失敗時のデータ保存

## セットアップ手順

### 前提条件

1. **Table Bucketの統合**
   - Table BucketをAWS分析サービスと統合済みであること
   - Namespaceが作成済みであること
   - Tableが作成済みであること

2. **IAMサービスロールの作成**
   - FirehoseがS3 Tablesにアクセスするための権限
   - AWS Glue Data Catalogへのアクセス権限
   - Lake Formation権限

3. **Lake Formation権限の付与**
   - FirehoseサービスロールにTable/Namespaceへの明示的な権限付与

### ステップ1: IAMサービスロールの作成

#### IAMポリシーの作成

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3TableAccessViaGlueFederation",
      "Effect": "Allow",
      "Action": [
        "glue:GetTable",
        "glue:GetDatabase",
        "glue:UpdateTable"
      ],
      "Resource": [
        "arn:aws:glue:us-east-1:111122223333:catalog/s3tablescatalog/*",
        "arn:aws:glue:us-east-1:111122223333:catalog/s3tablescatalog",
        "arn:aws:glue:us-east-1:111122223333:catalog",
        "arn:aws:glue:us-east-1:111122223333:database/*",
        "arn:aws:glue:us-east-1:111122223333:table/*/*"
      ]
    },
    {
      "Sid": "S3DeliveryErrorBucketPermission",
      "Effect": "Allow",
      "Action": [
        "s3:AbortMultipartUpload",
        "s3:GetBucketLocation",
        "s3:GetObject",
        "s3:ListBucket",
        "s3:ListBucketMultipartUploads",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::error-delivery-bucket",
        "arn:aws:s3:::error-delivery-bucket/*"
      ]
    },
    {
      "Sid": "RequiredWhenUsingKinesisDataStreamsAsSource",
      "Effect": "Allow",
      "Action": [
        "kinesis:DescribeStream",
        "kinesis:GetShardIterator",
        "kinesis:GetRecords",
        "kinesis:ListShards"
      ],
      "Resource": "arn:aws:kinesis:us-east-1:111122223333:stream/stream-name"
    },
    {
      "Sid": "RequiredWhenDoingMetadataReadsANDDataAndMetadataWriteViaLakeformation",
      "Effect": "Allow",
      "Action": [
        "lakeformation:GetDataAccess"
      ],
      "Resource": "*"
    },
    {
      "Sid": "RequiredWhenUsingKMSEncryptionForS3ErrorBucketDelivery",
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": [
        "arn:aws:kms:us-east-1:111122223333:key/KMS-key-id"
      ],
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "s3.us-east-1.amazonaws.com"
        },
        "StringLike": {
          "kms:EncryptionContext:aws:s3:arn": "arn:aws:s3:::error-delivery-bucket/prefix*"
        }
      }
    },
    {
      "Sid": "LoggingInCloudWatch",
      "Effect": "Allow",
      "Action": [
        "logs:PutLogEvents"
      ],
      "Resource": [
        "arn:aws:logs:us-east-1:111122223333:log-group:log-group-name:log-stream:log-stream-name"
      ]
    },
    {
      "Sid": "RequiredWhenAttachingLambdaToFirehose",
      "Effect": "Allow",
      "Action": [
        "lambda:InvokeFunction",
        "lambda:GetFunctionConfiguration"
      ],
      "Resource": [
        "arn:aws:lambda:us-east-1:111122223333:function:function-name:function-version"
      ]
    }
  ]
}
```

#### 信頼ポリシー

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sts:AssumeRole"
      ],
      "Principal": {
        "Service": [
          "firehose.amazonaws.com"
        ]
      }
    }
  ]
}
```

### ステップ2: Firehose Streamの作成

#### コンソールでの作成手順

1. **Firehoseコンソールを開く**
   - https://console.aws.amazon.com/firehose/

2. **Create Firehose streamを選択**

3. **ソースの設定**
   - Amazon Kinesis Data Streams
   - Amazon MSK
   - Direct PUT

4. **デスティネーションの設定**
   - Destination: **Apache Iceberg Tables**を選択
   - Firehose stream name: ストリーム名を入力

5. **カタログの設定**
   - Current account: 同一アカウント内のテーブル
   - Cross-account: 別アカウントのテーブル
   - Catalog ARN形式: `arn:aws:glue:<region>:<account-id>:catalog/s3tablescatalog/<table-bucket-name>`

6. **ルーティング設定**
   - Unique Key configuration
   - JSONQuery expressions
   - Lambda function

7. **バックアップ設定**
   - S3 backup bucket: エラー時のバックアップ先
   - S3 backup bucket error output prefix: エラーデータのプレフィックス

8. **IAMロールの設定**
   - Advanced settings → Existing IAM roles
   - 作成したIAMロールを選択

9. **Create Firehose streamを実行**

#### AWS CLIでの作成例

```bash
aws firehose create-delivery-stream \
  --delivery-stream-name my-s3-tables-stream \
  --delivery-stream-type DirectPut \
  --iceberg-destination-configuration '{
    "RoleARN": "arn:aws:iam::111122223333:role/FirehoseS3TablesRole",
    "CatalogConfiguration": {
      "CatalogARN": "arn:aws:glue:us-east-1:111122223333:catalog/s3tablescatalog/my-table-bucket"
    },
    "DestinationTableConfigurationList": [
      {
        "DestinationDatabaseName": "my-namespace",
        "DestinationTableName": "my-table",
        "UniqueKeys": ["id"]
      }
    ],
    "S3Configuration": {
      "RoleARN": "arn:aws:iam::111122223333:role/FirehoseS3TablesRole",
      "BucketARN": "arn:aws:s3:::error-delivery-bucket",
      "Prefix": "errors/",
      "ErrorOutputPrefix": "errors/failed/"
    },
    "BufferingHints": {
      "SizeInMBs": 128,
      "IntervalInSeconds": 300
    },
    "RetryOptions": {
      "DurationInSeconds": 300
    }
  }'
```


## データルーティング

### 単一テーブルへのルーティング

#### Unique Key設定

```json
[
  {
    "DestinationDatabaseName": "my-namespace",
    "DestinationTableName": "my-table",
    "UniqueKeys": ["customer_id"],
    "S3ErrorOutputPrefix": "errors/my-table/"
  }
]
```

### 複数テーブルへのルーティング

#### JQ Expressionを使用したルーティング

```json
{
  "DatabaseName": ".metadata.database",
  "TableName": ".metadata.table",
  "Operation": ".metadata.operation"
}
```

入力データ例：

```json
{
  "metadata": {
    "database": "sales",
    "table": "orders",
    "operation": "insert"
  },
  "data": {
    "order_id": "12345",
    "customer_id": "67890",
    "amount": 99.99
  }
}
```

#### Lambda関数を使用したルーティング

```python
import json

def lambda_handler(event, context):
    output = []
    
    for record in event['records']:
        # Base64デコード
        payload = json.loads(base64.b64decode(record['data']))
        
        # ルーティング情報を追加
        routing_info = {
            'database': determine_database(payload),
            'table': determine_table(payload),
            'operation': determine_operation(payload)
        }
        
        # データとルーティング情報を結合
        result = {
            'recordId': record['recordId'],
            'result': 'Ok',
            'data': base64.b64encode(
                json.dumps({
                    **payload,
                    '_metadata': routing_info
                }).encode('utf-8')
            ).decode('utf-8')
        }
        
        output.append(result)
    
    return {'records': output}
```

## データ操作

### サポートされる操作

Firehoseは3つのIceberg操作をサポートしています：

1. **Insert（デフォルト）**: 新しい行を追加
2. **Update**: 既存の行を更新（Unique Keyが必要）
3. **Delete**: 既存の行を削除（Unique Keyが必要）

### 操作の指定方法

#### デフォルト動作（Insert）

操作を指定しない場合、すべてのレコードはInsertとして処理されます。

```json
{
  "customer_id": "12345",
  "name": "John Doe",
  "email": "john@example.com"
}
```

#### Update操作

```json
{
  "operation": "update",
  "customer_id": "12345",
  "name": "John Smith",
  "email": "john.smith@example.com"
}
```

**注意**: Update操作は、Unique Keyで指定されたカラムを使用して既存の行を検索します。行が見つからない場合は、自動的にInsertされます。

#### Delete操作

```json
{
  "operation": "delete",
  "customer_id": "12345"
}
```

### Unique Keyの設定

#### Firehose Stream作成時の設定

```json
[
  {
    "DestinationDatabaseName": "sales",
    "DestinationTableName": "customers",
    "UniqueKeys": ["customer_id"]
  }
]
```

#### Icebergテーブル作成時の設定

```sql
CREATE TABLE sales.customers (
  customer_id STRING,
  name STRING,
  email STRING
)
USING iceberg
TBLPROPERTIES (
  'write.metadata.metrics.default' = 'full',
  'write.metadata.metrics.column.customer_id' = 'full'
)
PARTITIONED BY (bucket(16, customer_id));

-- Identifier fieldsの設定
ALTER TABLE sales.customers
SET IDENTIFIER FIELDS customer_id;
```

## データ変換

### Lambda関数を使用したデータ変換

#### 基本的な変換例

```python
import json
import base64
from datetime import datetime

def lambda_handler(event, context):
    output = []
    
    for record in event['records']:
        # デコード
        payload = json.loads(base64.b64decode(record['data']))
        
        # データ変換
        transformed = {
            'id': payload['id'],
            'timestamp': datetime.utcnow().isoformat(),
            'value': float(payload['value']) * 1.1,  # 10%増加
            'category': payload.get('category', 'unknown').upper()
        }
        
        # エンコード
        result = {
            'recordId': record['recordId'],
            'result': 'Ok',
            'data': base64.b64encode(
                json.dumps(transformed).encode('utf-8')
            ).decode('utf-8')
        }
        
        output.append(result)
    
    return {'records': output}
```

#### エラーハンドリング

```python
def lambda_handler(event, context):
    output = []

    for record in event['records']:
        try:
            payload = json.loads(base64.b64decode(record['data']))

            # バリデーション
            if not validate_payload(payload):
                raise ValueError("Invalid payload")

            # 変換処理
            transformed = transform_data(payload)

            result = {
                'recordId': record['recordId'],
                'result': 'Ok',
                'data': base64.b64encode(
                    json.dumps(transformed).encode('utf-8')
                ).decode('utf-8')
            }
        except Exception as e:
            # エラー時はDropped
            result = {
                'recordId': record['recordId'],
                'result': 'ProcessingFailed',
                'data': record['data']
            }

        output.append(result)

    return {'records': output}
```

## バッファリング設定

### バッファサイズとインターバル

Firehoseは、メモリ内でストリーミングデータをバッファリングしてからIcebergテーブルに配信します。

#### 設定パラメータ

| パラメータ | 範囲 | デフォルト | 説明 |
|----------|------|----------|------|
| Buffering size | 1-128 MiB | 5 MiB | バッファサイズ |
| Buffering interval | 0-900秒 | 300秒 | バッファ時間 |

#### トレードオフ

**大きいバッファ値（高スループット、高レイテンシ）**:
- S3書き込み回数の削減
- コンパクションコストの削減
- より大きなデータファイル
- クエリパフォーマンスの向上
- 配信レイテンシの増加

**小さいバッファ値（低スループット、低レイテンシ）**:
- 低レイテンシ配信
- S3書き込み回数の増加
- より小さなデータファイル
- コンパクション頻度の増加
- クエリパフォーマンスの低下

#### 推奨設定

```json
{
  "BufferingHints": {
    "SizeInMBs": 128,
    "IntervalInSeconds": 300
  }
}
```

**ユースケース別の推奨値**:

| ユースケース | Size (MiB) | Interval (秒) | 理由 |
|------------|-----------|--------------|------|
| リアルタイム分析 | 1-5 | 60-120 | 低レイテンシ優先 |
| バッチ処理 | 64-128 | 300-900 | スループット優先 |
| バランス型 | 32-64 | 180-300 | レイテンシとコストのバランス |

## リトライ設定

### リトライ期間の設定

```json
{
  "RetryOptions": {
    "DurationInSeconds": 300
  }
}
```

- **範囲**: 0-7200秒
- **デフォルト**: 300秒

### リトライ動作

1. Firehoseは、Icebergテーブルへの書き込みに失敗した場合、指定された期間リトライを試みます
2. リトライ期間が経過すると、失敗したレコードはS3エラーバケットに配信されます
3. エラーログはCloudWatch Logsに記録されます

## エラーハンドリング

### エラーバケットの設定

```json
{
  "S3Configuration": {
    "RoleARN": "arn:aws:iam::111122223333:role/FirehoseS3TablesRole",
    "BucketARN": "arn:aws:s3:::error-delivery-bucket",
    "Prefix": "errors/",
    "ErrorOutputPrefix": "errors/failed/"
  }
}
```

### エラータイプ

| エラーメッセージ | 説明 | 対処方法 |
|----------------|------|---------|
| `Iceberg.NoSuchTable` | テーブルが存在しない、またはV2フォーマットではない | テーブルの存在とフォーマットを確認 |
| `Iceberg.InvalidTableName` | テーブル名がnullまたは空、またはV2フォーマットではない | テーブル名を確認 |
| `S3.AccessDenied` | S3アクセス権限不足 | IAMロールの権限を確認 |
| `Glue.AccessDenied` | Glueアクセス権限不足 | IAMロールとLake Formation権限を確認 |

### CloudWatch Logsでのモニタリング

```bash
# ログストリームの確認
aws logs describe-log-streams \
  --log-group-name /aws/kinesisfirehose/my-stream \
  --order-by LastEventTime \
  --descending

# ログイベントの取得
aws logs get-log-events \
  --log-group-name /aws/kinesisfirehose/my-stream \
  --log-stream-name my-log-stream
```

## パフォーマンス最適化

### スループット制限

#### Direct PUTソースの制限

| リージョン | デフォルトスループット |
|----------|-------------------|
| US East (N. Virginia) | 5 MiB/秒 |
| US West (Oregon) | 5 MiB/秒 |
| Europe (Ireland) | 5 MiB/秒 |
| その他のリージョン | 1 MiB/秒 |

#### AppendOnlyフラグ

Insert専用（UpdateとDeleteなし）の場合、`AppendOnly`フラグを`True`に設定することで、Firehoseが自動的にスループットをスケールします。

```bash
aws firehose create-delivery-stream \
  --delivery-stream-name my-stream \
  --delivery-stream-type DirectPut \
  --iceberg-destination-configuration '{
    "AppendOnly": true,
    ...
  }'
```

**注意**: `AppendOnly`フラグは、CreateDeliveryStream APIでのみ設定可能です（コンソールでは設定不可）。

### S3 TPSの最適化

Kinesis Data StreamsまたはAmazon MSKをソースとして使用する場合：

1. **適切なパーティションキーの使用**
   - 同じIcebergテーブルにルーティングされるレコードを1つまたは少数のシャードにマッピング
   - 異なるIcebergテーブルへのレコードを異なるシャードに分散

2. **シャード数の最適化**
   - すべてのシャードの集約スループットを活用
   - テーブルごとにシャードを分離

### コンパクション戦略

Firehoseは書き込みごとにスナップショット、データファイル、削除ファイルを生成します。多数の小さなファイルはメタデータオーバーヘッドを増加させ、読み取りパフォーマンスに影響します。

#### AWS Glue Data Catalogの自動コンパクション

```bash
# テーブルオプティマイザーの有効化
aws glue create-table-optimizer \
  --catalog-id 111122223333 \
  --database-name my-namespace \
  --table-name my-table \
  --type compaction \
  --table-optimizer-configuration '{
    "enabled": true,
    "roleArn": "arn:aws:iam::111122223333:role/GlueTableOptimizerRole"
  }'
```

#### Athena OPTIMIZEコマンド

```sql
-- 手動コンパクション
OPTIMIZE my_table REWRITE DATA USING BIN_PACK;

-- パーティション指定
OPTIMIZE my_table REWRITE DATA USING BIN_PACK
WHERE year = 2024 AND month = 1;
```

#### VACUUMコマンド

```sql
-- スナップショット期限切れ
VACUUM my_table EXPIRE SNAPSHOTS OLDER THAN TIMESTAMP '2024-01-01 00:00:00';

-- 孤立ファイル削除
VACUUM my_table REMOVE ORPHAN FILES OLDER THAN TIMESTAMP '2024-01-01 00:00:00';
```


## 制限事項と考慮事項

### リージョンサポート

Firehoseは、以下を除くすべてのAWSリージョンでApache Icebergテーブルをサポートしています：
- 中国リージョン
- AWS GovCloud (US)リージョン
- Asia Pacific (Malaysia)

### データフォーマット制限

#### ネストされたJSON

- **サポートレベル**: 16レベルまでのネスト
- **カラム名とデータ型**: ソースデータとターゲットテーブルで完全に一致する必要がある
- **追加フィールド**: ターゲットテーブルに存在しないフィールドはスキップされる
- **不一致時の動作**: エラーが発生し、データはS3エラーバケットに配信される

ネストされたJSONの例：

```json
{
  "version": "2016-04-01",
  "deviceId": "device-12345",
  "sensorId": "sensor-67890",
  "timestamp": "2024-01-11T20:42:45.000Z",
  "value": 23.5,
  "position": {
    "x": 143.595901,
    "y": 476.399628,
    "z": 0.24234876
  }
}
```

対応するIcebergテーブルスキーマ：

```sql
CREATE TABLE sensors.readings (
  version STRING,
  deviceId STRING,
  sensorId STRING,
  timestamp TIMESTAMP,
  value DOUBLE,
  position STRUCT<x: DOUBLE, y: DOUBLE, z: DOUBLE>
)
USING iceberg;
```

#### レコードフォーマット

- **1レコード = 1 JSONオブジェクト**: 1つのFirehoseレコードに複数のJSONオブジェクトを含めることはできない
- **KPL集約**: Kinesis Producer Library (KPL)で集約されたレコードは、Firehoseが自動的に分解する

### 同時書き込み制限

**推奨されない**: 複数のFirehoseストリームから同じIcebergテーブルへの同時書き込み

**理由**: Apache Icebergは楽観的同時実行制御（OCC）を使用しているため：
- 同時に1つのストリームのみがコミットに成功
- 他のストリームはバックオフしてリトライ
- リトライ期間が経過すると、データはS3エラープレフィックスに送信される

### Icebergライブラリバージョン

- **サポートバージョン**: 1.5.2
- **テーブルフォーマット**: V2のみサポート（V1は非サポート）

### 命名規則制限

- **ハイフン（`-`）**: データベース名とテーブル名でサポートされない
- **GlueCatalog API**: Iceberg GlueCatalog APIで作成されたテーブルのみサポート
- **Glue SDK**: Glue SDKで作成されたテーブルは非サポート

参考: [Glue Database Regex](https://github.com/apache/iceberg/blob/main/aws/src/main/java/org/apache/iceberg/aws/glue/IcebergToGlueConverter.java#L62)

### 暗号化設定

**S3 Tablesへの配信時の暗号化**:
- S3 Tables側でKMS設定を行う
- Firehose設定でKMSパラメータを設定しても使用されない

参考: [Using server-side encryption with AWS KMS keys](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-kms-encryption.html)

### パーティション制限

- **パーティション計算**: すべてのファイル（データファイルと削除ファイル）は、レコード内のパーティション情報を使用して計算される
- **グローバル削除**: パーティションテーブルに対する非パーティション削除ファイルの書き込みは非サポート

### その他の制限

- **Amazon MSK Serverless**: Apache Icebergテーブルのデスティネーションとして非サポート
- **Update操作のコスト**: Update操作は削除ファイルの作成後にInsertを実行するため、S3 PUTリクエストが追加で発生

## ユースケース

### リアルタイムログ分析

#### AWS WAFログのストリーミング

```bash
# Firehose Stream作成
aws firehose create-delivery-stream \
  --delivery-stream-name waf-logs-stream \
  --delivery-stream-type DirectPut \
  --iceberg-destination-configuration '{
    "RoleARN": "arn:aws:iam::111122223333:role/FirehoseRole",
    "CatalogConfiguration": {
      "CatalogARN": "arn:aws:glue:us-east-1:111122223333:catalog/s3tablescatalog/security-logs"
    },
    "DestinationTableConfigurationList": [
      {
        "DestinationDatabaseName": "security",
        "DestinationTableName": "waf_logs"
      }
    ],
    "BufferingHints": {
      "SizeInMBs": 5,
      "IntervalInSeconds": 60
    }
  }'
```

### IoTデータの取り込み

#### センサーデータのストリーミング

```python
import boto3
import json
from datetime import datetime

firehose = boto3.client('firehose')

def send_sensor_data(device_id, sensor_readings):
    record = {
        'device_id': device_id,
        'timestamp': datetime.utcnow().isoformat(),
        'readings': sensor_readings
    }

    response = firehose.put_record(
        DeliveryStreamName='iot-sensor-stream',
        Record={
            'Data': json.dumps(record).encode('utf-8')
        }
    )

    return response
```

### CDCデータの取り込み

#### データベース変更のストリーミング

```json
{
  "operation": "update",
  "database": "ecommerce",
  "table": "orders",
  "data": {
    "order_id": "12345",
    "status": "shipped",
    "updated_at": "2024-01-15T10:30:00Z"
  }
}
```

Lambda関数でのルーティング：

```python
def lambda_handler(event, context):
    output = []

    for record in event['records']:
        payload = json.loads(base64.b64decode(record['data']))

        # CDC操作をIceberg操作にマッピング
        operation_mapping = {
            'insert': 'insert',
            'update': 'update',
            'delete': 'delete'
        }

        transformed = {
            '_metadata': {
                'database': payload['database'],
                'table': payload['table'],
                'operation': operation_mapping[payload['operation']]
            },
            **payload['data']
        }

        result = {
            'recordId': record['recordId'],
            'result': 'Ok',
            'data': base64.b64encode(
                json.dumps(transformed).encode('utf-8')
            ).decode('utf-8')
        }

        output.append(result)

    return {'records': output}
```

## ベストプラクティス

### 1. 適切なバッファサイズの選択

- **リアルタイム要件**: 小さいバッファサイズ（1-5 MiB）と短いインターバル（60-120秒）
- **コスト最適化**: 大きいバッファサイズ（64-128 MiB）と長いインターバル（300-900秒）
- **バランス型**: 中程度のバッファサイズ（32-64 MiB）とインターバル（180-300秒）

### 2. エラーハンドリングの実装

- **S3エラーバケットの設定**: 必ず設定する
- **CloudWatch Logsの有効化**: エラーログを記録
- **アラームの設定**: エラー発生時の通知

```bash
# CloudWatch Alarmの作成
aws cloudwatch put-metric-alarm \
  --alarm-name firehose-delivery-errors \
  --alarm-description "Alert on Firehose delivery errors" \
  --metric-name DeliveryToS3.DataFreshness \
  --namespace AWS/Firehose \
  --statistic Average \
  --period 300 \
  --threshold 900 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2
```

### 3. パーティション戦略

- **適切なパーティションキーの選択**: クエリパターンに基づく
- **カーディナリティの考慮**: 過度なパーティション分割を避ける
- **時系列データ**: 日付ベースのパーティショニング

```sql
CREATE TABLE events.user_actions (
  user_id STRING,
  action STRING,
  timestamp TIMESTAMP,
  event_date DATE
)
USING iceberg
PARTITIONED BY (event_date);
```

### 4. コンパクションの自動化

- **AWS Glue Data Catalogの自動コンパクション**: 有効化を推奨
- **定期的なVACUUM**: スナップショット期限切れと孤立ファイル削除
- **モニタリング**: ファイル数とサイズの監視

### 5. セキュリティ

- **最小権限の原則**: IAMロールに必要最小限の権限のみ付与
- **Lake Formation権限**: きめ細かなアクセス制御の実装
- **暗号化**: S3 Tables側でKMS暗号化を設定
- **VPCエンドポイント**: プライベートネットワーク経由でのアクセス

### 6. モニタリングとアラート

#### 主要メトリクス

| メトリクス | 説明 | 推奨閾値 |
|----------|------|---------|
| `DeliveryToS3.Success` | 成功した配信数 | - |
| `DeliveryToS3.DataFreshness` | データの鮮度（秒） | < 900 |
| `IncomingBytes` | 受信バイト数 | - |
| `IncomingRecords` | 受信レコード数 | - |
| `ThrottledRecords` | スロットルされたレコード数 | 0 |

#### CloudWatch Dashboardの作成

```bash
aws cloudwatch put-dashboard \
  --dashboard-name firehose-monitoring \
  --dashboard-body file://dashboard.json
```

dashboard.json:

```json
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/Firehose", "DeliveryToS3.Success", {"stat": "Sum"}],
          [".", "IncomingRecords", {"stat": "Sum"}]
        ],
        "period": 300,
        "stat": "Average",
        "region": "us-east-1",
        "title": "Firehose Delivery Metrics"
      }
    }
  ]
}
```

### 7. コスト最適化

- **AppendOnlyフラグの活用**: Insert専用の場合に使用
- **バッファサイズの最適化**: S3リクエスト数を削減
- **自動コンパクション**: ストレージコストを削減
- **リージョン選択**: データソースと同じリージョンを使用

## トラブルシューティング

### 一般的な問題と解決方法

#### 問題1: データが配信されない

**症状**: Firehoseストリームにデータを送信しているが、Icebergテーブルにデータが表示されない

**確認事項**:
1. IAMロールの権限確認
2. Lake Formation権限の確認
3. テーブルの存在確認
4. CloudWatch Logsでエラー確認

```bash
# テーブルの存在確認
aws glue get-table \
  --catalog-id 111122223333 \
  --database-name my-namespace \
  --name my-table

# Lake Formation権限の確認
aws lakeformation list-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::111122223333:role/FirehoseRole \
  --resource '{
    "Table": {
      "CatalogId": "111122223333",
      "DatabaseName": "my-namespace",
      "Name": "my-table"
    }
  }'
```

#### 問題2: スキーマ不一致エラー

**症状**: `Column type mismatch`エラーが発生

**解決方法**:
1. ソースデータのスキーマ確認
2. Icebergテーブルのスキーマ確認
3. データ型の一致確認

```sql
-- テーブルスキーマの確認
DESCRIBE FORMATTED my_namespace.my_table;
```

#### 問題3: スループット制限

**症状**: `ThrottlingException`エラーが発生

**解決方法**:
1. AppendOnlyフラグの設定（Insert専用の場合）
2. スループット制限の引き上げリクエスト
3. 複数のストリームへの分散

```bash
# スループット制限の引き上げリクエスト
# AWS Support Centerから申請
```

#### 問題4: 高レイテンシ

**症状**: データ配信に時間がかかる

**解決方法**:
1. バッファサイズとインターバルの調整
2. リージョンの確認（データソースと同じリージョンを使用）
3. ネットワーク設定の確認

## 出典

### AWS公式ドキュメント

1. **Streaming data to tables with Amazon Data Firehose**
   - URL: https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-integrating-firehose.html
   - アクセス日: 2025-11-15
   - 内容: S3 TablesへのFirehose統合の概要と設定手順

2. **Deliver data to Apache Iceberg Tables with Amazon Data Firehose**
   - URL: https://docs.aws.amazon.com/firehose/latest/dev/apache-iceberg-destination.html
   - アクセス日: 2025-11-15
   - 内容: Apache IcebergテーブルへのFirehose配信の詳細

3. **Considerations and limitations - Amazon Data Firehose**
   - URL: https://docs.aws.amazon.com/firehose/latest/dev/apache-iceberg-considerations.html
   - アクセス日: 2025-11-15
   - 内容: Firehose Iceberg統合の制限事項と考慮事項

4. **Set up the Firehose stream**
   - URL: https://docs.aws.amazon.com/firehose/latest/dev/apache-iceberg-stream.html
   - アクセス日: 2025-11-15
   - 内容: Firehoseストリームの詳細設定

5. **Working with Iceberg tables by using Amazon Data Firehose - AWS Prescriptive Guidance**
   - URL: https://docs.aws.amazon.com/prescriptive-guidance/latest/apache-iceberg-on-aws/iceberg-firehose.html
   - アクセス日: 2025-11-15
   - 内容: IcebergテーブルとFirehoseの統合ガイダンス

### 関連リソース

- **AWS Blog**: Stream real-time data into Apache Iceberg tables in Amazon S3 using Amazon Data Firehose
  - URL: https://aws.amazon.com/blogs/big-data/stream-real-time-data-into-apache-iceberg-tables-in-amazon-s3-using-amazon-data-firehose/

- **AWS Glue Data Catalog**: Compaction management
  - URL: https://docs.aws.amazon.com/glue/latest/dg/compaction-management.html

- **Apache Iceberg**: Optimistic Concurrency Control
  - URL: https://iceberg.apache.org/docs/1.6.0/reliability/#concurrent-write-operations

---

**最終更新日**: 2025-11-15
**ドキュメントバージョン**: 1.0
