# Amazon S3 Tables - トラブルシューティングガイド


## 目次

- [概要](#概要)
- [一般的な問題と解決策](#一般的な問題と解決策)
  - [問題1: テーブルが見つからない](#問題1-テーブルが見つからない)
  - [問題2: 権限エラー](#問題2-権限エラー)
  - [問題3: クエリパフォーマンスが遅い](#問題3-クエリパフォーマンスが遅い)
  - [問題4: データが更新されない](#問題4-データが更新されない)
  - [問題5: Firehose配信失敗](#問題5-firehose配信失敗)
- [デバッグ手順](#デバッグ手順)
  - [1. ログ確認](#1-ログ確認)
  - [2. メトリクス確認](#2-メトリクス確認)
  - [3. 設定確認](#3-設定確認)
- [パフォーマンスチューニング](#パフォーマンスチューニング)
  - [クエリ最適化](#クエリ最適化)
  - [ファイル最適化](#ファイル最適化)
- [よくある質問への回答](#よくある質問への回答)
  - [Q: マネジメントコンソールからテーブルを削除できない](#q-マネジメントコンソールからテーブルを削除できない)
  - [Q: カラム名に大文字を使用したらAthenaから見えなくなった](#q-カラム名に大文字を使用したらathenaから見えなくなった)
- [サポートへの問い合わせ](#サポートへの問い合わせ)
  - [必要な情報](#必要な情報)
  - [問い合わせ方法](#問い合わせ方法)
- [出典](#出典)
  - [関連ドキュメント](#関連ドキュメント)
- [更新履歴](#更新履歴)

## 概要

本ドキュメントでは、Amazon S3 Tablesの一般的な問題と解決方法を解説します。

---

## 一般的な問題と解決策

### 問題1: テーブルが見つからない

**症状**:
```yaml
Table not found: s3tablescatalog.my_namespace.orders
```

**原因**:
- テーブル名の誤り
- Namespaceの誤り
- カタログ設定の誤り

**解決策**:
```sql
-- テーブル一覧確認
SHOW TABLES IN s3tablescatalog.my_namespace;

-- カタログ設定確認
SHOW CATALOGS;
```

### 問題2: 権限エラー

**症状**:
```
Access Denied: User does not have permission to access table
```

**原因**:
- IAM権限不足
- Lake Formation権限不足
- テーブルポリシーによる制限

**解決策**:
```bash
# IAM権限確認
aws iam get-role-policy \
  --role-name MyRole \
  --policy-name S3TablesPolicy

# Lake Formation権限確認
aws lakeformation list-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::123456789012:role/MyRole
```

### 問題3: クエリパフォーマンスが遅い

**症状**:
- クエリ実行時間が長い
- タイムアウトエラー

**原因**:
- パーティション設定の不備
- 小ファイル問題
- 統計情報の欠如

**解決策**:
```sql
-- パーティション確認
SHOW PARTITIONS s3tablescatalog.my_namespace.orders;

-- ファイル統計確認
SELECT 
    file_path,
    file_size_in_bytes,
    record_count
FROM s3tablescatalog.my_namespace.orders.files
ORDER BY file_size_in_bytes;

-- コンパクション実行
CALL s3tablescatalog.system.rewrite_data_files(
    table => 'my_namespace.orders',
    strategy => 'binpack',
    options => map('target-file-size-bytes', '536870912')
);
```

### 問題4: データが更新されない

**症状**:
- 書き込んだデータが見えない
- 古いデータが表示される

**原因**:
- トランザクションの未コミット
- メタデータキャッシュ
- スナップショットの問題

**解決策**:
```sql
-- メタデータリフレッシュ
REFRESH TABLE s3tablescatalog.my_namespace.orders;

-- 最新スナップショット確認
SELECT * FROM s3tablescatalog.my_namespace.orders.snapshots
ORDER BY committed_at DESC
LIMIT 1;
```

### 問題5: Firehose配信失敗

**症状**:
- データが配信されない
- エラーバケットにデータが蓄積

**原因**:
- スキーマ不一致
- IAM権限不足
- KMS権限不足

**解決策**:
```bash
# エラーログ確認
aws logs filter-log-events \
  --log-group-name /aws/kinesisfirehose/my-stream \
  --filter-pattern "ERROR"

# IAM権限確認
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:role/FirehoseRole \
  --action-names s3tables:PutTableData \
  --resource-arns arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket/table/*
```

---

## デバッグ手順

### 1. ログ確認

```bash
# CloudTrailログ
aws logs filter-log-events \
  --log-group-name /aws/cloudtrail/logs \
  --filter-pattern "s3tables"

# CloudWatch Logs
aws logs tail /aws/s3tables/maintenance --follow
```

### 2. メトリクス確認

```bash
# CloudWatch Metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/S3Tables \
  --metric-name RequestCount \
  --start-time 2024-01-15T00:00:00Z \
  --end-time 2024-01-15T23:59:59Z \
  --period 3600 \
  --statistics Sum
```

### 3. 設定確認

```bash
# Table Bucket設定
aws s3tables get-table-bucket \
  --table-bucket-arn arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket

# テーブル設定
aws s3tables get-table \
  --table-bucket-arn arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket \
  --namespace my_namespace \
  --name orders
```

---

## パフォーマンスチューニング

### クエリ最適化

```sql
-- パーティションプルーニング
SELECT * FROM s3tablescatalog.my_namespace.orders
WHERE order_date = '2024-01-15';  -- パーティションキーを使用

-- カラムプルーニング
SELECT order_id, customer_id, order_amount  -- 必要なカラムのみ
FROM s3tablescatalog.my_namespace.orders;
```

### ファイル最適化

```sql
-- コンパクション
CALL s3tablescatalog.system.rewrite_data_files(
    table => 'my_namespace.orders',
    strategy => 'binpack'
);

-- Z-Orderingによる最適化
CALL s3tablescatalog.system.rewrite_data_files(
    table => 'my_namespace.orders',
    strategy => 'sort',
    sort_order => 'customer_id,order_date'
);
```

---

## よくある質問への回答

### Q: マネジメントコンソールからテーブルを削除できない

**A**: S3 TablesはCLI/APIからのみ削除可能です。

```bash
aws s3tables delete-table \
  --table-bucket-arn arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket \
  --namespace my_namespace \
  --name orders
```

### Q: カラム名に大文字を使用したらAthenaから見えなくなった

**A**: S3 Tablesはすべて小文字に変換されます。小文字のカラム名を使用してください。

```sql
-- ❌ 非推奨
CREATE TABLE my_table (UserId INT, UserName STRING);

-- ✅ 推奨
CREATE TABLE my_table (user_id INT, user_name STRING);
```

---

## サポートへの問い合わせ

### 必要な情報

1. Table Bucket ARN
2. テーブル名
3. エラーメッセージ
4. CloudTrailログ
5. 再現手順

### 問い合わせ方法

```bash
# AWS Support Case作成
aws support create-case \
  --subject "S3 Tables Issue" \
  --service-code "amazon-s3-tables" \
  --severity-code "normal" \
  --category-code "technical" \
  --communication-body "詳細な問題説明"
```

---

## 出典

### 関連ドキュメント
- [制限事項](../01-getting-started/04-limitations.md): 制限事項
- [パフォーマンス最適化](02-performance-optimization.md): パフォーマンス最適化
- [セキュリティベストプラクティス](../03-security/01-security-best-practices.md): セキュリティベストプラクティス

## 更新履歴
- 2025-11-15: 初版作成
