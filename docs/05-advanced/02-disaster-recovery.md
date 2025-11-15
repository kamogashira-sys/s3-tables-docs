# Amazon S3 Tables - ディザスタリカバリ


## 目次

- [概要](#概要)
  - [DR戦略の重要性](#dr戦略の重要性)
- [DR目標の定義](#dr目標の定義)
  - [RPO (Recovery Point Objective)](#rpo-recovery-point-objective)
  - [RTO (Recovery Time Objective)](#rto-recovery-time-objective)
- [バックアップ戦略](#バックアップ戦略)
  - [Icebergスナップショットの活用](#icebergスナップショットの活用)
  - [クロスリージョンバックアップ](#クロスリージョンバックアップ)
  - [S3バケットへのエクスポート](#s3バケットへのエクスポート)
- [復旧手順](#復旧手順)
  - [スナップショットからの復旧](#スナップショットからの復旧)
  - [バックアップからの復旧](#バックアップからの復旧)
  - [部分復旧](#部分復旧)
- [フェイルオーバー手順](#フェイルオーバー手順)
  - [手動フェイルオーバー](#手動フェイルオーバー)
  - [自動フェイルオーバー](#自動フェイルオーバー)
- [DR演習](#dr演習)
  - [定期的なDR演習の実施](#定期的なdr演習の実施)
- [モニタリングとアラート](#モニタリングとアラート)
  - [ヘルスチェック](#ヘルスチェック)
  - [アラート設定](#アラート設定)
- [ベストプラクティス](#ベストプラクティス)
  - [1. 定期的なバックアップ](#1-定期的なバックアップ)
  - [2. 自動化](#2-自動化)
  - [3. ドキュメント化](#3-ドキュメント化)
  - [4. テスト](#4-テスト)
- [出典](#出典)
  - [関連ドキュメント](#関連ドキュメント)
- [更新履歴](#更新履歴)

## 概要

ディザスタリカバリ（DR）は、災害やシステム障害からビジネスを保護し、データとサービスの継続性を確保するための重要な戦略です。本ドキュメントでは、Amazon S3 Tablesのディザスタリカバリ戦略、バックアップ、復旧手順について解説します。

### DR戦略の重要性

**ビジネス継続性**
- サービス停止時間の最小化
- データ損失の防止
- ビジネスへの影響軽減

**コンプライアンス**
- 規制要件への対応
- データ保持ポリシーの遵守
- 監査証跡の維持

**リスク管理**
- リージョン障害への対応
- 人的エラーからの復旧
- セキュリティインシデントへの対応

---

## DR目標の定義

### RPO (Recovery Point Objective)

**定義**: 許容可能なデータ損失の時間

**S3 Tablesでの実現**:
- **RPO = 0**: 同期レプリケーション（コスト高）
- **RPO < 1時間**: 15分間隔の増分レプリケーション
- **RPO < 24時間**: 日次バックアップ

### RTO (Recovery Time Objective)

**定義**: 許容可能なサービス停止時間

**S3 Tablesでの実現**:
- **RTO < 1時間**: アクティブ-アクティブ構成
- **RTO < 4時間**: アクティブ-パッシブ構成（自動フェイルオーバー）
- **RTO < 24時間**: 手動フェイルオーバー

---

## バックアップ戦略

### Icebergスナップショットの活用

**スナップショット作成**:
```sql
-- 手動スナップショット作成
CALL s3tablescatalog.system.create_snapshot(
    'my_namespace.orders',
    'daily_backup_2024_01_15'
);

-- スナップショット一覧
SELECT * FROM s3tablescatalog.my_namespace.orders.snapshots
ORDER BY committed_at DESC;
```

**スナップショット保持ポリシー**:
```bash
# メンテナンス設定でスナップショット保持期間を設定
aws s3tables put-table-maintenance-configuration \
  --table-bucket-arn arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket \
  --namespace my_namespace \
  --table-name orders \
  --value '{
    "icebergCompaction": {
      "settings": {
        "targetFileSizeMB": 512
      }
    },
    "icebergSnapshotManagement": {
      "settings": {
        "minSnapshotsToKeep": 7,
        "maxSnapshotAgeHours": 168
      }
    }
  }'
```

### クロスリージョンバックアップ

**Sparkによるバックアップ**:
```python
from pyspark.sql import SparkSession
from datetime import datetime

def backup_to_secondary_region(source_table, backup_region):
    # プライマリリージョンのSpark Session
    spark = SparkSession.builder \
        .config("spark.sql.catalog.primary", "software.amazon.s3tables.iceberg.S3TablesCatalog") \
        .config("spark.sql.catalog.primary.warehouse", "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket") \
        .config("spark.sql.catalog.backup", "software.amazon.s3tables.iceberg.S3TablesCatalog") \
        .config("spark.sql.catalog.backup.warehouse", f"arn:aws:s3tables:{backup_region}:123456789012:bucket/backup-bucket") \
        .getOrCreate()
    
    # データ読み取り
    df = spark.sql(f"SELECT * FROM primary.{source_table}")
    
    # バックアップリージョンに書き込み
    backup_table = f"backup.{source_table}_{datetime.now().strftime('%Y%m%d')}"
    df.writeTo(backup_table).using("iceberg").create()
    
    return backup_table

# 使用例
backup_table_name = backup_to_secondary_region("my_namespace.orders", "us-west-2")
print(f"Backup created: {backup_table_name}")
```

### S3バケットへのエクスポート

**Athena UNLOADによるエクスポート**:
```sql
-- Parquet形式でエクスポート
UNLOAD (
    SELECT * FROM s3tablescatalog.my_namespace.orders
)
TO 's3://backup-bucket/exports/orders/2024-01-15/'
WITH (
    format = 'PARQUET',
    compression = 'SNAPPY',
    partitioned_by = ARRAY['order_date']
);
```

**Sparkによるエクスポート**:
```python
def export_to_s3(source_table, backup_path):
    df = spark.sql(f"SELECT * FROM {source_table}")
    
    # Parquet形式で保存
    df.write \
        .mode("overwrite") \
        .partitionBy("order_date") \
        .parquet(backup_path)
    
    # メタデータも保存
    metadata = spark.sql(f"DESCRIBE EXTENDED {source_table}").toPandas()
    metadata.to_json(f"{backup_path}/_metadata.json")

# 使用例
export_to_s3(
    "s3tablescatalog.my_namespace.orders",
    "s3://backup-bucket/exports/orders/2024-01-15/"
)
```

---

## 復旧手順

### スナップショットからの復旧

**特定スナップショットへのロールバック**:
```sql
-- スナップショット一覧確認
SELECT snapshot_id, committed_at, summary
FROM s3tablescatalog.my_namespace.orders.snapshots
ORDER BY committed_at DESC;

-- 特定スナップショットにロールバック
CALL s3tablescatalog.system.rollback_to_snapshot(
    'my_namespace.orders',
    1234567890123456789
);
```

**タイムトラベルクエリ**:
```sql
-- 1時間前の状態を確認
SELECT * FROM s3tablescatalog.my_namespace.orders
FOR SYSTEM_TIME AS OF TIMESTAMP '2024-01-15 10:00:00';

-- 特定スナップショットの状態を確認
SELECT * FROM s3tablescatalog.my_namespace.orders
FOR SYSTEM_VERSION AS OF 1234567890123456789;
```

### バックアップからの復旧

**クロスリージョンバックアップからの復旧**:
```python
def restore_from_backup(backup_table, target_table):
    # バックアップリージョンから読み取り
    df = spark.sql(f"SELECT * FROM backup.{backup_table}")

    # プライマリリージョンに復元
    df.writeTo(f"primary.{target_table}") \
        .using("iceberg") \
        .createOrReplace()

    print(f"Restored {target_table} from {backup_table}")

# 使用例
restore_from_backup(
    "my_namespace.orders_20240115",
    "my_namespace.orders"
)
```

**S3エクスポートからの復旧**:
```python
def restore_from_s3_export(backup_path, target_table):
    # Parquetファイルから読み取り
    df = spark.read.parquet(backup_path)
    
    # メタデータ読み取り
    import json
    with open(f"{backup_path}/_metadata.json") as f:
        metadata = json.load(f)
    
    # テーブル復元
    df.writeTo(target_table) \
        .using("iceberg") \
        .createOrReplace()

# 使用例
restore_from_s3_export(
    "s3://backup-bucket/exports/orders/2024-01-15/",
    "s3tablescatalog.my_namespace.orders"
)
```

### 部分復旧

**特定パーティションの復旧**:
```python
def restore_partition(backup_table, target_table, partition_value):
    # バックアップから特定パーティションを読み取り
    df = spark.sql(f"""
        SELECT * FROM backup.{backup_table}
        WHERE order_date = '{partition_value}'
    """)

    # ターゲットテーブルの該当パーティションを削除
    spark.sql(f"""
        DELETE FROM primary.{target_table}
        WHERE order_date = '{partition_value}'
    """)

    # 復元
    df.writeTo(f"primary.{target_table}") \
        .using("iceberg") \
        .append()

# 使用例
restore_partition(
    "my_namespace.orders_20240115",
    "my_namespace.orders",
    "2024-01-15"
)
```

---

## フェイルオーバー手順

### 手動フェイルオーバー

**手順**:

1. **プライマリリージョンの状態確認**:
```bash
# プライマリリージョンのTable Bucket確認
aws s3tables get-table-bucket \
  --table-bucket-arn arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket \
  --region us-east-1
```

2. **セカンダリリージョンへの切り替え**:
```python
# アプリケーション設定を更新
def switch_to_secondary_region():
    # 環境変数を更新
    import os
    os.environ['TABLE_BUCKET_ARN'] = 'arn:aws:s3tables:us-west-2:123456789012:bucket/my-table-bucket-replica'
    os.environ['AWS_REGION'] = 'us-west-2'

    # Spark設定を更新
    spark.conf.set(
        "spark.sql.catalog.s3tablescatalog.warehouse",
        "arn:aws:s3tables:us-west-2:123456789012:bucket/my-table-bucket-replica"
    )
```

3. **DNSレコードの更新**:
```bash
# Route 53レコード更新
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "TTL": 60,
        "ResourceRecords": [{"Value": "secondary-region-endpoint"}]
      }
    }]
  }'
```

### 自動フェイルオーバー

**Lambda関数による自動フェイルオーバー**:
```python
import boto3
import os

def lambda_handler(event, context):
    # ヘルスチェック
    primary_healthy = check_primary_region_health()

    if not primary_healthy:
        # セカンダリリージョンに切り替え
        switch_to_secondary()

        # アラート送信
        send_alert("Failover to secondary region initiated")

    return {
        'statusCode': 200,
        'body': 'Health check completed'
    }

def check_primary_region_health():
    try:
        s3tables = boto3.client('s3tables', region_name='us-east-1')
        response = s3tables.get_table_bucket(
            tableBucketARN='arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket'
        )
        return True
    except Exception as e:
        print(f"Primary region unhealthy: {e}")
        return False

def switch_to_secondary():
    # Route 53レコード更新
    route53 = boto3.client('route53')
    route53.change_resource_record_sets(
        HostedZoneId='Z1234567890ABC',
        ChangeBatch={
            'Changes': [{
                'Action': 'UPSERT',
                'ResourceRecordSet': {
                    'Name': 'api.example.com',
                    'Type': 'A',
                    'TTL': 60,
                    'ResourceRecords': [{'Value': 'secondary-endpoint'}]
                }
            }]
        }
    )
```

---

## DR演習

### 定期的なDR演習の実施

**演習計画**:

1. **月次演習**: スナップショットからの復旧
2. **四半期演習**: クロスリージョンバックアップからの復旧
3. **年次演習**: 完全なフェイルオーバー演習

**演習手順書**:
```markdown
# DR演習手順書

## 目的
セカンダリリージョンへのフェイルオーバーと復旧を検証

## 前提条件
- プライマリリージョン: us-east-1
- セカンダリリージョン: us-west-2
- 最新のバックアップが存在すること

## 手順

### 1. 事前確認（15分）
- [ ] プライマリリージョンのデータ整合性確認
- [ ] セカンダリリージョンのレプリケーション遅延確認
- [ ] バックアップの存在確認

### 2. フェイルオーバー実行（30分）
- [ ] プライマリリージョンへのアクセス停止
- [ ] セカンダリリージョンへの切り替え
- [ ] DNSレコード更新
- [ ] アプリケーション動作確認

### 3. 復旧確認（30分）
- [ ] データ整合性確認
- [ ] クエリ性能確認
- [ ] エラーログ確認

### 4. フェイルバック（30分）
- [ ] プライマリリージョンの復旧
- [ ] データ同期
- [ ] プライマリリージョンへの切り戻し

### 5. 事後確認（15分）
- [ ] 全システムの動作確認
- [ ] ログ分析
- [ ] 改善点の洗い出し
```

---

## モニタリングとアラート

### ヘルスチェック

**CloudWatch Syntheticsによる監視**:
```python
# Canaryスクリプト
from aws_synthetics.selenium import synthetics_webdriver as webdriver
from aws_synthetics.common import synthetics_logger as logger

def main():
    # Athenaクエリ実行
    import boto3
    athena = boto3.client('athena')

    response = athena.start_query_execution(
        QueryString='SELECT COUNT(*) FROM s3tablescatalog.my_namespace.orders',
        QueryExecutionContext={'Database': 's3tablescatalog'},
        ResultConfiguration={'OutputLocation': 's3://query-results/'}
    )

    # クエリ完了待機
    query_id = response['QueryExecutionId']
    status = athena.get_query_execution(QueryExecutionId=query_id)

    if status['QueryExecution']['Status']['State'] == 'SUCCEEDED':
        logger.info("Health check passed")
        return True
    else:
        logger.error("Health check failed")
        return False

def handler(event, context):
    return main()
```

### アラート設定

**SNS通知**:
```bash
# CloudWatch Alarm作成
aws cloudwatch put-metric-alarm \
  --alarm-name s3tables-health-check-failure \
  --alarm-description "S3 Tables health check failed" \
  --metric-name SuccessPercent \
  --namespace CloudWatchSynthetics \
  --statistic Average \
  --period 300 \
  --threshold 90 \
  --comparison-operator LessThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:dr-alerts
```

---

## ベストプラクティス

### 1. 定期的なバックアップ

**推奨事項**:
- 日次フルバックアップ
- 時間単位の増分バックアップ
- クロスリージョンバックアップ

### 2. 自動化

**推奨事項**:
- バックアップの自動化
- 復旧手順の自動化
- DR演習の自動化

### 3. ドキュメント化

**推奨事項**:
- 復旧手順書の作成
- 連絡先リストの維持
- 定期的な更新

### 4. テスト

**推奨事項**:
- 定期的なDR演習
- 復旧時間の測定
- 改善点の特定

---

## 出典

### 関連ドキュメント
- [マルチリージョン戦略](01-multi-region-strategy.md): マルチリージョン戦略
- [セキュリティベストプラクティス](../03-security/01-security-best-practices.md): セキュリティベストプラクティス
- [トラブルシューティング](../02-operations/03-troubleshooting-guide.md): トラブルシューティングガイド

## 更新履歴
- 2025-11-15: 初版作成
  - DR目標の定義（RPO/RTO）
  - バックアップ戦略
  - 復旧手順
  - フェイルオーバー手順
  - DR演習
  - モニタリングとアラート
  - ベストプラクティス
