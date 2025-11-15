# Amazon S3 Tables - コンプライアンス


## 目次

- [概要](#概要)
- [主要なコンプライアンス規制](#主要なコンプライアンス規制)
  - [GDPR (General Data Protection Regulation)](#gdpr-general-data-protection-regulation)
  - [HIPAA (Health Insurance Portability and Accountability Act)](#hipaa-health-insurance-portability-and-accountability-act)
  - [PCI DSS (Payment Card Industry Data Security Standard)](#pci-dss-payment-card-industry-data-security-standard)
  - [SOC 2 (Service Organization Control 2)](#soc-2-service-organization-control-2)
- [データ保持とアーカイブ](#データ保持とアーカイブ)
  - [保持ポリシーの実装](#保持ポリシーの実装)
- [監査とレポーティング](#監査とレポーティング)
  - [コンプライアンスレポートの生成](#コンプライアンスレポートの生成)
- [ベストプラクティス](#ベストプラクティス)
  - [1. データ分類の徹底](#1-データ分類の徹底)
  - [2. 暗号化の強制](#2-暗号化の強制)
  - [3. アクセス制御](#3-アクセス制御)
  - [4. 監査証跡の維持](#4-監査証跡の維持)
- [出典](#出典)
  - [関連ドキュメント](#関連ドキュメント)
- [更新履歴](#更新履歴)

## 概要

コンプライアンスは、法規制、業界標準、社内ポリシーへの準拠を確保するための重要な要素です。本ドキュメントでは、Amazon S3 Tablesにおける主要なコンプライアンス要件への対応方法を解説します。

---

## 主要なコンプライアンス規制

### GDPR (General Data Protection Regulation)

**対応要件**:
- 個人データの識別と分類
- データ主体の権利（アクセス、削除、訂正）
- データ処理の記録
- データ保護影響評価（DPIA）

**実装例**:
```python
# 個人データの識別
def identify_personal_data(table_name):
    pii_columns = []
    df = spark.sql(f"DESCRIBE {table_name}")
    
    for row in df.collect():
        column_name = row['col_name'].lower()
        if any(keyword in column_name for keyword in ['name', 'email', 'phone', 'address', 'ssn']):
            pii_columns.append(row['col_name'])
    
    return pii_columns

# データ主体アクセス要求（DSAR）への対応
def handle_dsar(customer_id):
    tables = ['customers', 'orders', 'interactions']
    customer_data = {}
    
    for table in tables:
        df = spark.sql(f"""
            SELECT * FROM s3tablescatalog.my_namespace.{table}
            WHERE customer_id = {customer_id}
        """)
        customer_data[table] = df.toPandas().to_dict('records')
    
    return customer_data
```

### HIPAA (Health Insurance Portability and Accountability Act)

**対応要件**:
- PHI（Protected Health Information）の暗号化
- アクセスログの記録
- 監査証跡の維持
- データ保持期間の遵守

**実装例**:
```bash
# KMS暗号化の有効化
aws s3tables create-table-bucket \
  --name hipaa-compliant-bucket \
  --encryption-configuration \
    KmsKeyArn=arn:aws:kms:us-east-1:123456789012:key/hipaa-key

# CloudTrailデータイベントの有効化
aws cloudtrail put-event-selectors \
  --trail-name my-trail \
  --event-selectors '[{
    "ReadWriteType": "All",
    "IncludeManagementEvents": true,
    "DataResources": [{
      "Type": "AWS::S3Tables::TableBucket",
      "Values": ["arn:aws:s3tables:us-east-1:123456789012:bucket/hipaa-compliant-bucket"]
    }]
  }]'
```

### PCI DSS (Payment Card Industry Data Security Standard)

**対応要件**:
- カード情報の暗号化
- アクセス制御
- ネットワークセキュリティ
- 定期的なセキュリティテスト

**実装例**:
```python
# カード情報のマスキング
from pyspark.sql.functions import col, regexp_replace

def mask_card_number(df):
    return df.withColumn(
        'card_number',
        regexp_replace(col('card_number'), r'(\d{4})\d{8}(\d{4})', r'\1********\2')
    )

# トークン化
def tokenize_card_data(card_number):
    # 外部トークン化サービスを使用
    import hashlib
    token = hashlib.sha256(card_number.encode()).hexdigest()
    return token
```

### SOC 2 (Service Organization Control 2)

**対応要件**:
- セキュリティ
- 可用性
- 処理の整合性
- 機密性
- プライバシー

**実装例**:
```python
# セキュリティコントロールの実装
def implement_soc2_controls():
    controls = {
        'encryption_at_rest': True,
        'encryption_in_transit': True,
        'access_logging': True,
        'mfa_enabled': True,
        'backup_enabled': True,
        'incident_response_plan': True
    }
    return controls
```

---

## データ保持とアーカイブ

### 保持ポリシーの実装

```python
from datetime import datetime, timedelta

# 業界別保持期間
RETENTION_POLICIES = {
    'financial': 7 * 365,  # 7年
    'healthcare': 6 * 365,  # 6年
    'general': 3 * 365      # 3年
}

def apply_retention_policy(table_name, industry):
    retention_days = RETENTION_POLICIES.get(industry, 365)
    cutoff_date = datetime.now() - timedelta(days=retention_days)
    
    # 古いデータをアーカイブテーブルに移動
    spark.sql(f"""
        INSERT INTO s3tablescatalog.archive.{table_name}
        SELECT * FROM s3tablescatalog.active.{table_name}
        WHERE created_at < TIMESTAMP '{cutoff_date.isoformat()}'
    """)
    
    # アクティブテーブルから削除
    spark.sql(f"""
        DELETE FROM s3tablescatalog.active.{table_name}
        WHERE created_at < TIMESTAMP '{cutoff_date.isoformat()}'
    """)
```

---

## 監査とレポーティング

### コンプライアンスレポートの生成

```python
def generate_compliance_report(start_date, end_date):
    report = {
        'period': f"{start_date} to {end_date}",
        'data_access': get_data_access_summary(start_date, end_date),
        'data_modifications': get_data_modification_summary(start_date, end_date),
        'security_incidents': get_security_incidents(start_date, end_date),
        'policy_violations': get_policy_violations(start_date, end_date)
    }
    return report

def get_data_access_summary(start_date, end_date):
    # CloudTrailログから集計
    query = f"""
    SELECT
        useridentity.principalid as user,
        COUNT(*) as access_count,
        COUNT(DISTINCT requestparameters) as unique_tables
    FROM cloudtrail_logs
    WHERE
        eventsource = 's3tables.amazonaws.com'
        AND eventname LIKE '%GetTable%'
        AND eventtime BETWEEN '{start_date}' AND '{end_date}'
    GROUP BY useridentity.principalid
    """
    return spark.sql(query).toPandas().to_dict('records')
```

---

## ベストプラクティス

### 1. データ分類の徹底
- すべてのデータに分類レベルを設定
- 規制対象データを明確に識別
- 定期的な分類レビュー

### 2. 暗号化の強制
- 保存時の暗号化（SSE-KMS）
- 転送時の暗号化（TLS 1.2+）
- キー管理の適切な実施

### 3. アクセス制御
- 最小権限の原則
- Lake Formationできめ細かい制御
- 定期的な権限レビュー

### 4. 監査証跡の維持
- CloudTrailログの有効化
- ログの長期保存
- 定期的なログ分析

---

## 出典

### 関連ドキュメント
- [データガバナンス](02-data-governance.md): データガバナンス
- [セキュリティベストプラクティス](01-security-best-practices.md): セキュリティベストプラクティス

## 更新履歴
- 2025-11-15: 初版作成
