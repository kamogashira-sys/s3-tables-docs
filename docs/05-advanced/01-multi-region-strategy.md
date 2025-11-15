# Amazon S3 Tables - マルチリージョン戦略


## 目次

- [概要](#概要)
  - [マルチリージョン戦略の重要性](#マルチリージョン戦略の重要性)
- [現在の制約事項](#現在の制約事項)
  - [S3 Tablesの制約](#s3-tablesの制約)
- [マルチリージョンアーキテクチャパターン](#マルチリージョンアーキテクチャパターン)
  - [パターン1: アクティブ-パッシブ構成](#パターン1-アクティブ-パッシブ構成)
  - [パターン2: アクティブ-アクティブ構成](#パターン2-アクティブ-アクティブ構成)
  - [パターン3: リージョン別データ分離](#パターン3-リージョン別データ分離)
- [データレプリケーション戦略](#データレプリケーション戦略)
  - [増分レプリケーション](#増分レプリケーション)
  - [フルレプリケーション](#フルレプリケーション)
- [グローバルアクセスパターン](#グローバルアクセスパターン)
  - [Route 53によるルーティング](#route-53によるルーティング)
  - [CloudFrontによるキャッシング](#cloudfrontによるキャッシング)
- [モニタリングとアラート](#モニタリングとアラート)
  - [レプリケーション遅延の監視](#レプリケーション遅延の監視)
  - [CloudWatch Metricsへの送信](#cloudwatch-metricsへの送信)
- [ベストプラクティス](#ベストプラクティス)
  - [1. レプリケーション頻度の最適化](#1-レプリケーション頻度の最適化)
  - [2. 競合解決戦略の明確化](#2-競合解決戦略の明確化)
  - [3. コスト最適化](#3-コスト最適化)
  - [4. セキュリティ](#4-セキュリティ)
- [出典](#出典)
  - [関連ドキュメント](#関連ドキュメント)
- [更新履歴](#更新履歴)

## 概要

Amazon S3 Tablesは現在シングルリージョンサービスですが、グローバルなデータアクセス、ディザスタリカバリ、コンプライアンス要件に対応するためのマルチリージョン戦略が重要です。本ドキュメントでは、S3 Tablesのマルチリージョン展開戦略、データレプリケーション、グローバルアクセスパターンについて解説します。

### マルチリージョン戦略の重要性

**グローバルアクセス**
- 世界中のユーザーへの低レイテンシアクセス
- リージョン障害時の可用性確保
- データ主権とコンプライアンス要件への対応

**ディザスタリカバリ**
- リージョン障害からの復旧
- データ損失の防止
- ビジネス継続性の確保

**パフォーマンス最適化**
- ユーザーに近いリージョンからのアクセス
- クロスリージョントラフィックの削減
- レイテンシの最小化

---

## 現在の制約事項

### S3 Tablesの制約

**シングルリージョンサービス**
- Table Bucketは単一リージョンに作成
- クロスリージョンレプリケーション機能なし
- リージョン間の自動フェイルオーバーなし

**対応方法**
- 手動でのマルチリージョン展開
- カスタムレプリケーションパイプライン
- アプリケーションレベルでのフェイルオーバー

---

## マルチリージョンアーキテクチャパターン

### パターン1: アクティブ-パッシブ構成

**アーキテクチャ**:
```text
プライマリリージョン (us-east-1)
  └─ S3 Tables (アクティブ)
       ↓ レプリケーション
セカンダリリージョン (us-west-2)
  └─ S3 Tables (パッシブ)
```

**特徴**:
- プライマリリージョンで全ての書き込み
- セカンダリリージョンは読み取り専用
- 障害時にセカンダリへフェイルオーバー

**実装方法**:

1. **両リージョンにTable Bucket作成**:
```bash
# プライマリリージョン
aws s3tables create-table-bucket \
  --name my-table-bucket \
  --region us-east-1

# セカンダリリージョン
aws s3tables create-table-bucket \
  --name my-table-bucket-replica \
  --region us-west-2
```

2. **Sparkによるレプリケーション**:
```python
from pyspark.sql import SparkSession

# プライマリリージョンのSpark Session
spark_primary = SparkSession.builder \
    .appName("MultiRegionReplication") \
    .config("spark.sql.catalog.primary", "software.amazon.s3tables.iceberg.S3TablesCatalog") \
    .config("spark.sql.catalog.primary.warehouse", "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket") \
    .config("spark.sql.catalog.secondary", "software.amazon.s3tables.iceberg.S3TablesCatalog") \
    .config("spark.sql.catalog.secondary.warehouse", "arn:aws:s3tables:us-west-2:123456789012:bucket/my-table-bucket-replica") \
    .getOrCreate()

# プライマリから読み取り
df = spark_primary.sql("SELECT * FROM primary.my_namespace.orders")

# セカンダリに書き込み
df.writeTo("secondary.my_namespace.orders") \
    .using("iceberg") \
    .createOrReplace()
```

3. **定期的なレプリケーションジョブ**:
```python
from datetime import datetime, timedelta

# 増分レプリケーション
last_replicated = datetime.now() - timedelta(hours=1)

df_incremental = spark_primary.sql(f"""
    SELECT * FROM primary.my_namespace.orders
    WHERE updated_at > TIMESTAMP '{last_replicated.isoformat()}'
""")

df_incremental.writeTo("secondary.my_namespace.orders") \
    .using("iceberg") \
    .append()
```

**メリット**:
- シンプルな構成
- コスト効率が良い
- 実装が容易

**デメリット**:
- フェイルオーバーに時間がかかる
- セカンダリリージョンのデータが遅延する可能性
- 手動フェイルオーバーが必要

### パターン2: アクティブ-アクティブ構成

**アーキテクチャ**:
```
リージョン1 (us-east-1)          リージョン2 (eu-west-1)
  └─ S3 Tables (アクティブ)  ←→  └─ S3 Tables (アクティブ)
       ↓ 双方向レプリケーション ↑
```

**特徴**:
- 両リージョンで読み書き可能
- 双方向レプリケーション
- 高可用性

**実装方法**:

1. **双方向レプリケーション**:
```python
# リージョン1 → リージョン2
def replicate_region1_to_region2():
    df = spark.sql("SELECT * FROM region1.my_namespace.orders WHERE updated_at > last_sync_time")
    df.writeTo("region2.my_namespace.orders").using("iceberg").append()

# リージョン2 → リージョン1
def replicate_region2_to_region1():
    df = spark.sql("SELECT * FROM region2.my_namespace.orders WHERE updated_at > last_sync_time")
    df.writeTo("region1.my_namespace.orders").using("iceberg").append()

# 定期実行
import schedule
schedule.every(5).minutes.do(replicate_region1_to_region2)
schedule.every(5).minutes.do(replicate_region2_to_region1)
```

2. **競合解決戦略**:
```python
from pyspark.sql.functions import col, max as spark_max

# Last Write Wins戦略
def resolve_conflicts(df1, df2):
    # 両方のデータフレームを結合
    combined = df1.union(df2)
    
    # 最新のレコードを保持
    result = combined.groupBy("order_id") \
        .agg(spark_max("updated_at").alias("max_updated_at")) \
        .join(combined, 
              (combined.order_id == col("order_id")) & 
              (combined.updated_at == col("max_updated_at"))) \
        .drop("max_updated_at")
    
    return result
```

**メリット**:
- 高可用性
- 低レイテンシ（ローカルアクセス）
- 自動フェイルオーバー

**デメリット**:
- 複雑な実装
- 競合解決が必要
- コストが高い

### パターン3: リージョン別データ分離

**アーキテクチャ**:
```
リージョン1 (us-east-1)     リージョン2 (eu-west-1)     リージョン3 (ap-northeast-1)
  └─ 北米データ              └─ 欧州データ               └─ アジアデータ
```

**特徴**:
- リージョンごとにデータを分離
- データ主権要件への対応
- クロスリージョンアクセスなし

**実装方法**:

1. **リージョン別Table Bucket**:
```bash
# 北米
aws s3tables create-table-bucket --name us-data-bucket --region us-east-1

# 欧州
aws s3tables create-table-bucket --name eu-data-bucket --region eu-west-1

# アジア
aws s3tables create-table-bucket --name ap-data-bucket --region ap-northeast-1
```

2. **リージョンルーティング**:
```python
def get_table_bucket_by_region(user_region):
    region_mapping = {
        'us': 'arn:aws:s3tables:us-east-1:123456789012:bucket/us-data-bucket',
        'eu': 'arn:aws:s3tables:eu-west-1:123456789012:bucket/eu-data-bucket',
        'ap': 'arn:aws:s3tables:ap-northeast-1:123456789012:bucket/ap-data-bucket'
    }
    return region_mapping.get(user_region)

# 使用例
warehouse_arn = get_table_bucket_by_region('eu')
spark = SparkSession.builder \
    .config("spark.sql.catalog.s3tablescatalog.warehouse", warehouse_arn) \
    .getOrCreate()
```

**メリット**:
- データ主権要件への対応
- シンプルな構成
- レプリケーション不要

**デメリット**:
- グローバルクエリが困難
- リージョン間のデータ統合が必要な場合に複雑

---

## データレプリケーション戦略

### 増分レプリケーション

**タイムスタンプベース**:
```python
from datetime import datetime, timedelta

def incremental_replication(source_table, target_table, last_sync_time):
    # 増分データ読み取り
    df = spark.sql(f"""
        SELECT * FROM {source_table}
        WHERE updated_at > TIMESTAMP '{last_sync_time.isoformat()}'
    """)

    # ターゲットに追加
    df.writeTo(target_table).using("iceberg").append()

    # 同期時刻を更新
    return datetime.now()

# 使用例
last_sync = datetime.now() - timedelta(hours=1)
new_sync_time = incremental_replication(
    "primary.my_namespace.orders",
    "secondary.my_namespace.orders",
    last_sync
)
```

**Icebergスナップショットベース**:
```python
def snapshot_based_replication(source_table, target_table):
    # ソーステーブルの最新スナップショット取得
    source_snapshot = spark.sql(f"""
        SELECT snapshot_id, committed_at
        FROM {source_table}.snapshots
        ORDER BY committed_at DESC
        LIMIT 1
    """).collect()[0]
    
    # ターゲットテーブルの最新スナップショット取得
    target_snapshot = spark.sql(f"""
        SELECT snapshot_id, committed_at
        FROM {target_table}.snapshots
        ORDER BY committed_at DESC
        LIMIT 1
    """).collect()[0]
    
    # 差分データを読み取り
    if source_snapshot.committed_at > target_snapshot.committed_at:
        df = spark.sql(f"""
            SELECT * FROM {source_table}
            VERSION AS OF {source_snapshot.snapshot_id}
        """)
        
        df.writeTo(target_table).using("iceberg").createOrReplace()
```

### フルレプリケーション

```python
def full_replication(source_table, target_table):
    # 全データ読み取り
    df = spark.sql(f"SELECT * FROM {source_table}")

    # ターゲットに書き込み（置き換え）
    df.writeTo(target_table).using("iceberg").createOrReplace()
```

---

## グローバルアクセスパターン

### Route 53によるルーティング

**地理的ルーティング**:
```json
{
  "Type": "A",
  "Name": "api.example.com",
  "GeoLocation": {
    "ContinentCode": "NA"
  },
  "ResourceRecords": [
    {
      "Value": "us-east-1-endpoint"
    }
  ]
}
```

### CloudFrontによるキャッシング

**Athenaクエリ結果のキャッシング**:
```bash
# CloudFront Distribution作成
aws cloudfront create-distribution \
  --origin-domain-name my-bucket.s3.amazonaws.com \
  --default-cache-behavior '{
    "TargetOriginId": "S3-my-bucket",
    "ViewerProtocolPolicy": "redirect-to-https",
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6"
  }'
```

---

## モニタリングとアラート

### レプリケーション遅延の監視

```python
from datetime import datetime

def check_replication_lag(primary_table, secondary_table):
    # プライマリの最新タイムスタンプ
    primary_max = spark.sql(f"""
        SELECT MAX(updated_at) as max_time
        FROM {primary_table}
    """).collect()[0].max_time
    
    # セカンダリの最新タイムスタンプ
    secondary_max = spark.sql(f"""
        SELECT MAX(updated_at) as max_time
        FROM {secondary_table}
    """).collect()[0].max_time
    
    # 遅延時間を計算
    lag_seconds = (primary_max - secondary_max).total_seconds()
    
    # アラート閾値チェック
    if lag_seconds > 3600:  # 1時間以上の遅延
        send_alert(f"Replication lag: {lag_seconds} seconds")
    
    return lag_seconds
```

### CloudWatch Metricsへの送信

```python
import boto3

cloudwatch = boto3.client('cloudwatch')

def publish_replication_metrics(lag_seconds):
    cloudwatch.put_metric_data(
        Namespace='S3Tables/Replication',
        MetricData=[
            {
                'MetricName': 'ReplicationLag',
                'Value': lag_seconds,
                'Unit': 'Seconds',
                'Dimensions': [
                    {
                        'Name': 'SourceRegion',
                        'Value': 'us-east-1'
                    },
                    {
                        'Name': 'TargetRegion',
                        'Value': 'us-west-2'
                    }
                ]
            }
        ]
    )
```

---

## ベストプラクティス

### 1. レプリケーション頻度の最適化

**推奨事項**:
- リアルタイム要件: 5-15分間隔
- 準リアルタイム: 1時間間隔
- バッチ: 日次

### 2. 競合解決戦略の明確化

**推奨戦略**:
- Last Write Wins（最終書き込み優先）
- Timestamp-based（タイムスタンプベース）
- Application-level（アプリケーションレベル）

### 3. コスト最適化

**推奨事項**:
- 増分レプリケーションを使用
- 圧縮を有効化
- 不要なデータのレプリケーションを避ける

### 4. セキュリティ

**推奨事項**:
- クロスリージョンIAMロールを使用
- VPCエンドポイントを活用
- 転送時の暗号化を確保

---

## 出典

### 関連ドキュメント
- [セキュリティベストプラクティス](../03-security/01-security-best-practices.md): セキュリティベストプラクティス
- [災害復旧](02-disaster-recovery.md): ディザスタリカバリ
- [パフォーマンス最適化](../02-operations/02-performance-optimization.md): パフォーマンス最適化

## 更新履歴
- 2025-11-15: 初版作成
  - マルチリージョンアーキテクチャパターン
  - データレプリケーション戦略
  - グローバルアクセスパターン
  - モニタリングとアラート
  - ベストプラクティス
