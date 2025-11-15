# Amazon S3 Tables - 監査とロギング


## 目次

- [概要](#概要)
- [CloudTrailログ](#cloudtrailログ)
  - [管理イベント](#管理イベント)
  - [データイベント](#データイベント)
  - [ログ分析](#ログ分析)
- [メンテナンスイベントログ](#メンテナンスイベントログ)
- [カスタムログ記録](#カスタムログ記録)
- [ベストプラクティス](#ベストプラクティス)
  - [1. ログの長期保存](#1-ログの長期保存)
  - [2. リアルタイム監視](#2-リアルタイム監視)
  - [3. 定期的なログ分析](#3-定期的なログ分析)
- [出典](#出典)
  - [関連ドキュメント](#関連ドキュメント)
- [更新履歴](#更新履歴)

## 概要

監査とロギングは、セキュリティ、コンプライアンス、トラブルシューティングに不可欠です。本ドキュメントでは、Amazon S3 Tablesの監査とロギングの実装方法を解説します。

---

## CloudTrailログ

### 管理イベント

**自動的に記録されるイベント**:
- CreateTableBucket
- DeleteTableBucket
- CreateTable
- DeleteTable
- PutTablePolicy
- GetTable

### データイベント

**設定方法**:
```bash
aws cloudtrail put-event-selectors \
  --trail-name my-trail \
  --event-selectors '[{
    "ReadWriteType": "All",
    "IncludeManagementEvents": true,
    "DataResources": [{
      "Type": "AWS::S3Tables::TableBucket",
      "Values": ["arn:aws:s3tables:us-east-1:123456789012:bucket/*"]
    }]
  }]'
```

### ログ分析

```python
# Athenaでログ分析
query = """
SELECT
    eventtime,
    useridentity.principalid,
    eventname,
    requestparameters,
    errorcode
FROM cloudtrail_logs
WHERE
    eventsource = 's3tables.amazonaws.com'
    AND eventtime >= CURRENT_DATE - INTERVAL '7' DAY
ORDER BY eventtime DESC
"""
```

---

## メンテナンスイベントログ

**自動メンテナンス操作の記録**:
- Compaction
- Snapshot Management
- Unreferenced File Removal

**ログ確認**:
```bash
aws logs filter-log-events \
  --log-group-name /aws/s3tables/maintenance \
  --filter-pattern "compaction" \
  --start-time $(date -u -d '1 day ago' +%s)000
```

---

## カスタムログ記録

```python
import logging
from datetime import datetime

# ロガー設定
logger = logging.getLogger('s3tables_audit')
logger.setLevel(logging.INFO)

# CloudWatch Logsハンドラー
handler = watchtower.CloudWatchLogHandler(
    log_group='/custom/s3tables/audit',
    stream_name=datetime.now().strftime('%Y-%m-%d')
)
logger.addHandler(handler)

# 使用例
def audit_log(action, table_name, user, details):
    logger.info({
        'timestamp': datetime.now().isoformat(),
        'action': action,
        'table': table_name,
        'user': user,
        'details': details
    })

audit_log('data_access', 'my_namespace.orders', 'user@example.com', {'rows': 1000})
```

---

## ベストプラクティス

### 1. ログの長期保存
- CloudTrailログを90日以上保存
- S3にアーカイブ
- Glacier Deep Archiveで長期保存

### 2. リアルタイム監視
- CloudWatch Logsでリアルタイム監視
- 異常パターンの検出
- 自動アラート

### 3. 定期的なログ分析
- 週次/月次レポート
- アクセスパターン分析
- セキュリティ監査

---

## 出典

### 関連ドキュメント
- [セキュリティベストプラクティス](../03-security/01-security-best-practices.md): セキュリティベストプラクティス
- [データガバナンス](../03-security/02-data-governance.md): データガバナンス

## 更新履歴
- 2025-11-15: 初版作成
