# Amazon S3 Tables - データガバナンス


## 目次

- [概要](#概要)
  - [データガバナンスの重要性](#データガバナンスの重要性)
- [データガバナンスフレームワーク](#データガバナンスフレームワーク)
  - [5つの柱](#5つの柱)
    - [1. データカタログ](#1-データカタログ)
    - [2. データ分類](#2-データ分類)
    - [3. データ品質](#3-データ品質)
    - [4. データリネージ](#4-データリネージ)
    - [5. データライフサイクル管理](#5-データライフサイクル管理)
- [Lake Formationとの統合](#lake-formationとの統合)
  - [きめ細かいアクセス制御](#きめ細かいアクセス制御)
  - [データマスキング](#データマスキング)
- [データ品質モニタリング](#データ品質モニタリング)
  - [継続的なモニタリング](#継続的なモニタリング)
  - [アラート設定](#アラート設定)
- [コンプライアンス](#コンプライアンス)
  - [GDPR対応](#gdpr対応)
  - [監査証跡](#監査証跡)
- [ベストプラクティス](#ベストプラクティス)
  - [1. データ分類の徹底](#1-データ分類の徹底)
  - [2. アクセス制御の最小権限化](#2-アクセス制御の最小権限化)
  - [3. データ品質の継続的監視](#3-データ品質の継続的監視)
  - [4. ライフサイクル管理の自動化](#4-ライフサイクル管理の自動化)
- [出典](#出典)
  - [関連ドキュメント](#関連ドキュメント)
- [更新履歴](#更新履歴)

## 概要

データガバナンスは、データの品質、セキュリティ、コンプライアンス、ライフサイクル管理を確保するための包括的なフレームワークです。本ドキュメントでは、Amazon S3 Tablesにおけるデータガバナンスの実装方法、ベストプラクティス、ツールについて解説します。

### データガバナンスの重要性

**データ品質**
- データの正確性と一貫性の確保
- データの信頼性向上
- ビジネス意思決定の質向上

**コンプライアンス**
- 規制要件への対応
- データプライバシーの保護
- 監査証跡の維持

**リスク管理**
- データ漏洩の防止
- 不正アクセスの検出
- データ損失の防止

---

## データガバナンスフレームワーク

### 5つの柱

#### 1. データカタログ

**AWS Glue Data Catalogとの統合**:
```python
import boto3

glue = boto3.client('glue')

# データベース作成
glue.create_database(
    DatabaseInput={
        'Name': 's3tablescatalog',
        'Description': 'S3 Tables catalog for data governance',
        'Parameters': {
            'classification': 'iceberg',
            'data_owner': 'data-team@example.com',
            'data_steward': 'governance-team@example.com'
        }
    }
)

# テーブルメタデータ取得
response = glue.get_table(
    DatabaseName='s3tablescatalog',
    Name='my_namespace.orders'
)

# メタデータにガバナンス情報を追加
glue.update_table(
    DatabaseName='s3tablescatalog',
    TableInput={
        'Name': 'my_namespace.orders',
        'Parameters': {
            'data_classification': 'confidential',
            'pii_fields': 'customer_name,email,phone',
            'retention_period': '7_years',
            'data_owner': 'sales-team@example.com'
        }
    }
)
```

#### 2. データ分類

**データ分類レベル**:
- **Public**: 公開可能なデータ
- **Internal**: 社内限定データ
- **Confidential**: 機密データ
- **Restricted**: 厳重管理データ

**実装例**:
```sql
-- テーブルプロパティで分類を設定
ALTER TABLE s3tablescatalog.my_namespace.orders
SET TBLPROPERTIES (
    'data_classification' = 'confidential',
    'contains_pii' = 'true',
    'pii_fields' = 'customer_name,email,phone,address'
);

-- 分類に基づくアクセス制御
-- Lake Formationで実装
```

#### 3. データ品質

**データ品質チェック**:
```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, count, when, isnan

spark = SparkSession.builder \
    .config("spark.sql.catalog.s3tablescatalog", "software.amazon.s3tables.iceberg.S3TablesCatalog") \
    .config("spark.sql.catalog.s3tablescatalog.warehouse", "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket") \
    .getOrCreate()

def data_quality_check(table_name):
    df = spark.sql(f"SELECT * FROM {table_name}")
    
    # 欠損値チェック
    null_counts = df.select([
        count(when(isnan(c) | col(c).isNull(), c)).alias(c)
        for c in df.columns
    ])
    
    # 重複チェック
    total_rows = df.count()
    distinct_rows = df.distinct().count()
    duplicate_rate = (total_rows - distinct_rows) / total_rows * 100
    
    # データ型チェック
    schema_violations = []
    for field in df.schema.fields:
        if field.dataType.typeName() == 'string':
            # 文字列長チェック
            max_length = df.agg({field.name: 'max'}).collect()[0][0]
            if max_length and len(max_length) > 1000:
                schema_violations.append(f"{field.name}: exceeds max length")
    
    # レポート作成
    quality_report = {
        'table': table_name,
        'total_rows': total_rows,
        'duplicate_rate': duplicate_rate,
        'null_counts': null_counts.collect()[0].asDict(),
        'schema_violations': schema_violations
    }
    
    return quality_report

# 使用例
report = data_quality_check('s3tablescatalog.my_namespace.orders')
print(report)
```

**AWS Glue Data Qualityとの統合**:
```python
# Glue Data Quality Ruleset作成
glue.create_data_quality_ruleset(
    Name='orders_quality_rules',
    Ruleset='''
        Rules = [
            RowCount > 1000,
            IsComplete "customer_id",
            IsComplete "order_date",
            IsUnique "order_id",
            ColumnValues "order_amount" > 0,
            ColumnValues "order_status" in ["pending", "completed", "cancelled"]
        ]
    ''',
    TargetTable={
        'DatabaseName': 's3tablescatalog',
        'TableName': 'my_namespace.orders'
    }
)
```

#### 4. データリネージ

**AWS Glue Data Lineageの活用**:
```python
# データリネージ情報の取得
def get_data_lineage(table_name):
    glue = boto3.client('glue')
    
    # テーブルのリネージ情報取得
    response = glue.get_table_versions(
        DatabaseName='s3tablescatalog',
        TableName=table_name
    )
    
    lineage = []
    for version in response['TableVersions']:
        lineage.append({
            'version': version['VersionId'],
            'update_time': version['Table']['UpdateTime'],
            'updated_by': version['Table'].get('Parameters', {}).get('updated_by', 'unknown')
        })
    
    return lineage

# 使用例
lineage = get_data_lineage('my_namespace.orders')
for entry in lineage:
    print(f"Version {entry['version']}: {entry['update_time']} by {entry['updated_by']}")
```

**カスタムリネージトラッキング**:
```python
from datetime import datetime

def track_data_transformation(source_table, target_table, transformation_type):
    # リネージ情報をメタデータテーブルに記録
    lineage_record = {
        'timestamp': datetime.now().isoformat(),
        'source_table': source_table,
        'target_table': target_table,
        'transformation_type': transformation_type,
        'user': os.environ.get('USER', 'unknown'),
        'job_id': os.environ.get('JOB_ID', 'manual')
    }

    # リネージテーブルに挿入
    spark.createDataFrame([lineage_record]).writeTo(
        "s3tablescatalog.governance.data_lineage"
    ).using("iceberg").append()

# 使用例
track_data_transformation(
    'raw.orders',
    'curated.orders',
    'data_cleansing'
)
```

#### 5. データライフサイクル管理

**保持ポリシーの実装**:
```python
from datetime import datetime, timedelta

def apply_retention_policy(table_name, retention_days):
    # 保持期間を超えたデータを削除
    cutoff_date = datetime.now() - timedelta(days=retention_days)
    
    spark.sql(f"""
        DELETE FROM {table_name}
        WHERE created_at < TIMESTAMP '{cutoff_date.isoformat()}'
    """)
    
    # 削除ログを記録
    deletion_log = {
        'table': table_name,
        'cutoff_date': cutoff_date.isoformat(),
        'deleted_at': datetime.now().isoformat()
    }
    
    spark.createDataFrame([deletion_log]).writeTo(
        "s3tablescatalog.governance.deletion_log"
    ).using("iceberg").append()

# 使用例
apply_retention_policy('s3tablescatalog.my_namespace.logs', 90)
```

---

## Lake Formationとの統合

### きめ細かいアクセス制御

**データベースレベルの権限**:
```bash
# データベースへのアクセス許可
aws lakeformation grant-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::123456789012:role/DataAnalystRole \
  --resource '{"Database":{"Name":"s3tablescatalog"}}' \
  --permissions "DESCRIBE"
```

**テーブルレベルの権限**:
```bash
# テーブルへのアクセス許可
aws lakeformation grant-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::123456789012:role/DataAnalystRole \
  --resource '{"Table":{"DatabaseName":"s3tablescatalog","Name":"my_namespace.orders"}}' \
  --permissions "SELECT" "DESCRIBE"
```

**カラムレベルの権限**:
```bash
# 特定カラムへのアクセス許可
aws lakeformation grant-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::123456789012:role/DataAnalystRole \
  --resource '{
    "TableWithColumns": {
      "DatabaseName": "s3tablescatalog",
      "Name": "my_namespace.customers",
      "ColumnNames": ["customer_id", "order_count", "total_spent"]
    }
  }' \
  --permissions "SELECT"
```

**行レベルのフィルタ**:
```bash
# データフィルタ作成
aws lakeformation create-data-cells-filter \
  --table-data '{
    "TableCatalogId": "123456789012",
    "DatabaseName": "s3tablescatalog",
    "TableName": "my_namespace.orders",
    "Name": "region_filter",
    "RowFilter": {
      "FilterExpression": "region = '\''us-east-1'\''"
    }
  }'
```

### データマスキング

**カラムマスキング**:
```python
from pyspark.sql.functions import col, when, regexp_replace

def mask_pii_data(df, pii_columns):
    masked_df = df

    for column in pii_columns:
        if column in df.columns:
            # メールアドレスのマスキング
            if 'email' in column.lower():
                masked_df = masked_df.withColumn(
                    column,
                    regexp_replace(col(column), r'(.{3}).*(@.*)', r'\1***\2')
                )
            # 電話番号のマスキング
            elif 'phone' in column.lower():
                masked_df = masked_df.withColumn(
                    column,
                    regexp_replace(col(column), r'(\d{3})\d{4}(\d{4})', r'\1****\2')
                )
            # その他のPIIは完全マスキング
            else:
                masked_df = masked_df.withColumn(
                    column,
                    when(col(column).isNotNull(), '***MASKED***').otherwise(None)
                )

    return masked_df

# 使用例
df = spark.sql("SELECT * FROM s3tablescatalog.my_namespace.customers")
masked_df = mask_pii_data(df, ['email', 'phone', 'ssn'])
```

---

## データ品質モニタリング

### 継続的なモニタリング

**CloudWatch Metricsへの送信**:
```python
import boto3

cloudwatch = boto3.client('cloudwatch')

def publish_data_quality_metrics(table_name, quality_report):
    cloudwatch.put_metric_data(
        Namespace='S3Tables/DataQuality',
        MetricData=[
            {
                'MetricName': 'DuplicateRate',
                'Value': quality_report['duplicate_rate'],
                'Unit': 'Percent',
                'Dimensions': [
                    {'Name': 'TableName', 'Value': table_name}
                ]
            },
            {
                'MetricName': 'NullValueCount',
                'Value': sum(quality_report['null_counts'].values()),
                'Unit': 'Count',
                'Dimensions': [
                    {'Name': 'TableName', 'Value': table_name}
                ]
            }
        ]
    )
```

### アラート設定

```bash
# データ品質アラーム
aws cloudwatch put-metric-alarm \
  --alarm-name data-quality-duplicate-rate-high \
  --alarm-description "Duplicate rate exceeds threshold" \
  --metric-name DuplicateRate \
  --namespace S3Tables/DataQuality \
  --statistic Average \
  --period 3600 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --dimensions Name=TableName,Value=my_namespace.orders \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:data-quality-alerts
```

---

## コンプライアンス

### GDPR対応

**個人データの識別**:
```sql
-- PII含有テーブルの一覧
SELECT 
    table_name,
    table_properties['contains_pii'] as contains_pii,
    table_properties['pii_fields'] as pii_fields
FROM s3tablescatalog.information_schema.tables
WHERE table_properties['contains_pii'] = 'true';
```

**データ削除要求への対応**:
```python
def handle_gdpr_deletion_request(customer_id):
    # 関連テーブルからデータ削除
    tables_with_customer_data = [
        's3tablescatalog.my_namespace.customers',
        's3tablescatalog.my_namespace.orders',
        's3tablescatalog.my_namespace.customer_interactions'
    ]

    for table in tables_with_customer_data:
        spark.sql(f"""
            DELETE FROM {table}
            WHERE customer_id = {customer_id}
        """)

    # 削除ログを記録
    deletion_record = {
        'customer_id': customer_id,
        'deleted_at': datetime.now().isoformat(),
        'deleted_by': os.environ.get('USER', 'system'),
        'reason': 'GDPR_deletion_request'
    }

    spark.createDataFrame([deletion_record]).writeTo(
        "s3tablescatalog.compliance.gdpr_deletions"
    ).using("iceberg").append()
```

### 監査証跡

**CloudTrailログの分析**:
```python
def analyze_audit_trail(start_date, end_date):
    # CloudTrailログからS3 Tables操作を抽出
    athena = boto3.client('athena')
    
    query = f"""
    SELECT 
        eventtime,
        useridentity.principalid,
        eventname,
        requestparameters,
        responseelements
    FROM cloudtrail_logs
    WHERE 
        eventsource = 's3tables.amazonaws.com'
        AND eventtime BETWEEN '{start_date}' AND '{end_date}'
    ORDER BY eventtime DESC
    """
    
    response = athena.start_query_execution(
        QueryString=query,
        QueryExecutionContext={'Database': 'default'},
        ResultConfiguration={'OutputLocation': 's3://audit-results/'}
    )
    
    return response['QueryExecutionId']
```

---

## ベストプラクティス

### 1. データ分類の徹底

**推奨事項**:
- すべてのテーブルに分類レベルを設定
- PII含有フィールドを明示
- 定期的な分類レビュー

### 2. アクセス制御の最小権限化

**推奨事項**:
- Lake Formationできめ細かい権限設定
- 定期的な権限レビュー
- 不要な権限の削除

### 3. データ品質の継続的監視

**推奨事項**:
- 自動化されたデータ品質チェック
- CloudWatch Metricsでの監視
- 品質低下時のアラート

### 4. ライフサイクル管理の自動化

**推奨事項**:
- 保持ポリシーの明確化
- 自動削除の実装
- 削除ログの記録

---

## 出典

### 関連ドキュメント
- [セキュリティベストプラクティス](01-security-best-practices.md): セキュリティベストプラクティス
- [コンプライアンス](03-compliance.md): コンプライアンス
- [監視・ログ記録](../02-operations/04-monitoring-logging.md): 監視とロギング

## 更新履歴
- 2025-11-15: 初版作成
  - データガバナンスフレームワーク
  - Lake Formationとの統合
  - データ品質モニタリング
  - コンプライアンス対応
  - ベストプラクティス
