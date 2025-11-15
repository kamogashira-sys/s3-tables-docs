# スナップショット管理によるストレージ最適化

## 概要

スナップショット管理は、Apache Icebergテーブルのスナップショット（時点のデータ状態）を自動的に管理し、古いスナップショットを削除することでストレージコストを最適化する機能です。S3 Tablesでは、テーブルレベルでスナップショット管理を設定できます。

### スナップショットとは

Apache Icebergでは、テーブルへの各書き込み操作（INSERT、UPDATE、DELETE）が新しいスナップショットを作成します。スナップショットは以下の情報を含みます：

- **スナップショットID**: 一意の識別子
- **タイムスタンプ**: 作成日時
- **マニフェストリスト**: データファイルへの参照
- **スキーマ**: テーブル構造
- **パーティション情報**: パーティショニング設定

### なぜスナップショット管理が必要か

スナップショットが蓄積すると、以下の問題が発生します：

1. **ストレージコストの増加**
   - 古いスナップショットが参照するデータファイルが保持される
   - メタデータファイルが増加

2. **メタデータ管理の複雑化**
   - スナップショット数の増加によるメタデータ読み取りの遅延
   - テーブルメタデータファイルのサイズ増大

3. **クエリパフォーマンスへの影響**
   - メタデータ処理のオーバーヘッド増加

スナップショット管理により、これらの問題を自動的に解決できます。

## デフォルト設定

S3 Tablesでは、すべてのテーブルに対してスナップショット管理がデフォルトで有効化されています。

### デフォルト値

| 設定項目 | デフォルト値 | 説明 |
|---------|------------|------|
| MinimumSnapshots | 1 | 保持する最小スナップショット数 |
| MaximumSnapshotAge | 120時間（5日間） | スナップショットの最大保持期間 |
| Status | enabled | スナップショット管理の有効/無効 |

### 動作ロジック

スナップショット管理は、以下のロジックでスナップショットを削除します：

1. **最小スナップショット数の確認**
   - 現在のスナップショット数が`MinimumSnapshots`以下の場合、削除しない

2. **スナップショット年齢の確認**
   - スナップショットの作成時刻が`MaximumSnapshotAge`を超えているか確認

3. **削除対象の決定**
   - 最小スナップショット数を維持しつつ、古いスナップショットを削除

4. **非現行オブジェクトのマーク**
   - 削除されたスナップショットのみが参照するデータファイルを非現行としてマーク

5. **非現行オブジェクトの削除**
   - テーブルバケットの`NoncurrentDays`設定に基づき、非現行オブジェクトを削除

## 動作の仕組み

### スナップショット有効期限

```
現在時刻: 2024-12-01 10:00:00
MaximumSnapshotAge: 120時間（5日間）

スナップショット一覧:
1. 2024-11-25 08:00:00 (6日前) → 削除対象
2. 2024-11-26 09:00:00 (5日前) → 削除対象
3. 2024-11-27 10:00:00 (4日前) → 保持
4. 2024-11-28 11:00:00 (3日前) → 保持
5. 2024-11-29 12:00:00 (2日前) → 保持
6. 2024-11-30 13:00:00 (1日前) → 保持
7. 2024-12-01 09:00:00 (1時間前) → 保持

MinimumSnapshots: 1
→ スナップショット1と2を削除（5個のスナップショットが残る）
```

### 非現行オブジェクトのマーク

スナップショットが削除されると、そのスナップショットのみが参照するデータファイルが非現行としてマークされます。

```
スナップショット1が参照するファイル:
- file_a.parquet (スナップショット1のみが参照) → 非現行
- file_b.parquet (スナップショット1と3が参照) → 現行のまま
- file_c.parquet (スナップショット1のみが参照) → 非現行

スナップショット1削除後:
- file_a.parquet → 非現行オブジェクトとしてマーク
- file_c.parquet → 非現行オブジェクトとしてマーク
```

### 削除プロセス

非現行オブジェクトは、テーブルバケットの`NoncurrentDays`設定に基づいて削除されます。

```
NoncurrentDays: 7日

非現行オブジェクト:
- file_a.parquet (非現行になってから3日) → 保持
- file_d.parquet (非現行になってから8日) → 削除
```

## 設定方法

### スナップショット管理の設定変更

```bash
# MinimumSnapshotsとMaximumSnapshotAgeを変更
aws s3tables put-table-maintenance-configuration \
  --table-bucket-arn arn:aws:s3tables:us-east-1:111122223333:bucket/my-table-bucket \
  --namespace my-namespace \
  --name my-table \
  --type icebergSnapshotManagement \
  --value '{
    "status": "enabled",
    "settings": {
      "icebergSnapshotManagement": {
        "minSnapshotsToKeep": 10,
        "maxSnapshotAgeHours": 2500
      }
    }
  }'
```

### スナップショット管理の無効化

```bash
aws s3tables put-table-maintenance-configuration \
  --table-bucket-arn arn:aws:s3tables:us-east-1:111122223333:bucket/my-table-bucket \
  --namespace my-namespace \
  --name my-table \
  --type icebergSnapshotManagement \
  --value '{
    "status": "disabled",
    "settings": {
      "icebergSnapshotManagement": {
        "minSnapshotsToKeep": 1,
        "maxSnapshotAgeHours": 120
      }
    }
  }'
```

### 設定の確認

```bash
aws s3tables get-table-maintenance-configuration \
  --table-bucket-arn arn:aws:s3tables:us-east-1:111122223333:bucket/my-table-bucket \
  --namespace my-namespace \
  --name my-table \
  --type icebergSnapshotManagement
```

### スナップショット一覧の確認

```sql
-- Sparkでスナップショット履歴を確認
SELECT * FROM s3tables.my_namespace.my_table.snapshots
ORDER BY committed_at DESC;

-- 特定期間のスナップショット
SELECT 
  snapshot_id,
  committed_at,
  operation,
  summary
FROM s3tables.my_namespace.my_table.snapshots
WHERE committed_at >= CURRENT_TIMESTAMP - INTERVAL '7' DAY
ORDER BY committed_at DESC;
```

## Icebergテーブルプロパティとの関係

### metadata.jsonでの設定

Icebergテーブルの`metadata.json`ファイルでもスナップショット保持設定を定義できますが、S3 Tablesのスナップショット管理とは**独立して動作**します。

**重要な注意事項**:
- S3 Tablesのスナップショット管理は、`metadata.json`の設定を**無視**します
- `metadata.json`で長い保持期間を設定すると、S3 Tablesのスナップショット管理が**無効化**されます

### ALTER TABLE SET TBLPROPERTIESとの関係

SQLの`ALTER TABLE SET TBLPROPERTIES`コマンドで設定したスナップショット保持ポリシーも、S3 Tablesのスナップショット管理とは独立しています。

```sql
-- この設定はS3 Tablesのスナップショット管理に影響しない
ALTER TABLE my_table SET TBLPROPERTIES (
  'history.expire.max-snapshot-age-ms' = '432000000'  -- 5日間
);
```

### 競合時の動作

以下の場合、S3 Tablesのスナップショット管理が**無効化**されます：

1. **Branch/tag-based retentionの設定**
   ```sql
   -- Branchベースの保持設定
   ALTER TABLE my_table 
   SET TBLPROPERTIES (
     'write.wap.enabled' = 'true',
     'write.wap.branch' = 'staging'
   );
   ```

2. **metadata.jsonでの長期保持設定**
   ```json
   {
     "properties": {
       "history.expire.max-snapshot-age-ms": "864000000"  -- 10日間
     }
   }
   ```
   この場合、S3 Tablesの`MaximumSnapshotAge`（120時間=5日間）より長いため、スナップショット管理が無効化されます。

### 推奨アプローチ

S3 Tablesのスナップショット管理を使用する場合：
- `metadata.json`やSQL TBLPROPERTIESでスナップショット保持設定を**行わない**
- S3 Tablesの`PutTableMaintenanceConfiguration` APIのみで管理
- Branch/tag-based retentionを**使用しない**

## 注意事項

### 1. 削除の永続性

**重要**: 非現行オブジェクトの削除は永続的であり、復旧できません。

削除されたデータを復元する方法はありません。重要なデータの場合は、以下の対策を検討してください：

- `MinimumSnapshots`を増やす
- `MaximumSnapshotAge`を延長
- 定期的なバックアップの実施

### 2. 非現行オブジェクトの確認

非現行としてマークされたオブジェクトを確認または復旧するには、**AWS Supportへの連絡が必要**です。

**連絡先**:
- [AWS Support](https://aws.amazon.com/contact-us/)
- [AWS Support Documentation](https://aws.amazon.com/documentation/aws-support/)

### 3. 外部参照の考慮

スナップショット管理は、テーブル内の参照のみを考慮します。テーブル外からの参照（例: 他のテーブルからのシンボリックリンク）は考慮されません。

### 4. Branch/tag-based retention非対応

以下の機能は、S3 Tablesのスナップショット管理と**互換性がありません**：

- Iceberg Branchベースの保持ポリシー
- Iceberg Tagベースの保持ポリシー
- `metadata.json`での長期保持設定

これらを使用すると、スナップショット管理が自動的に無効化されます。

### 5. ストレージコストへの影響

スナップショット管理を無効化すると、古いスナップショットとデータファイルが蓄積し、ストレージコストが増加します。定期的な手動削除が必要になります。

## ベストプラクティス

### 1. 推奨設定値

| ユースケース | MinimumSnapshots | MaximumSnapshotAge | 理由 |
|------------|-----------------|-------------------|------|
| 開発環境 | 1 | 24時間（1日） | 最小限のストレージコスト |
| テスト環境 | 3 | 72時間（3日） | 最近のテストデータを保持 |
| 本番環境（低頻度更新） | 5 | 168時間（7日） | 週次のロールバック対応 |
| 本番環境（高頻度更新） | 10 | 120時間（5日） | 頻繁な更新に対応 |
| コンプライアンス要件 | 30 | 720時間（30日） | 長期保持が必要 |

### 2. ストレージコスト最適化

**定期的な見直し**
- 月次でスナップショット数を確認
- 不要なスナップショットの削除
- 設定値の調整

**NoncurrentDaysの最適化**
```bash
# テーブルバケットのNoncurrentDays設定
aws s3tables put-table-bucket-maintenance-configuration \
  --table-bucket-arn arn:aws:s3tables:us-east-1:111122223333:bucket/my-table-bucket \
  --type unreferencedFileRemoval \
  --value '{
    "status": "enabled",
    "settings": {
      "unreferencedFileRemoval": {
        "unreferencedDays": 1,
        "noncurrentDays": 3
      }
    }
  }'
```

### 3. ロールバック戦略

スナップショット管理を設定する際は、ロールバック要件を考慮します：

```sql
-- 特定のスナップショットにロールバック
CALL s3tables.system.rollback_to_snapshot(
  's3tables.my_namespace.my_table',
  1234567890
);

-- 特定の時刻にロールバック
CALL s3tables.system.rollback_to_timestamp(
  's3tables.my_namespace.my_table',
  TIMESTAMP '2024-12-01 10:00:00'
);
```

**推奨**:
- ロールバック期間を考慮して`MaximumSnapshotAge`を設定
- 重要な変更前にスナップショットIDを記録

### 4. 監視とアラート

```sql
-- スナップショット数の監視
SELECT COUNT(*) AS snapshot_count
FROM s3tables.my_namespace.my_table.snapshots;

-- 古いスナップショットの確認
SELECT 
  snapshot_id,
  committed_at,
  DATEDIFF(CURRENT_TIMESTAMP, committed_at) AS age_days
FROM s3tables.my_namespace.my_table.snapshots
WHERE committed_at < CURRENT_TIMESTAMP - INTERVAL '5' DAY
ORDER BY committed_at;
```

**アラート設定**:
- スナップショット数が閾値を超えた場合
- 古いスナップショットが残っている場合
- ストレージ使用量が急増した場合

### 5. テスト環境での検証

本番環境に適用する前に、テスト環境で設定を検証します：

1. 設定を適用
2. スナップショット削除の動作を確認
3. ストレージ使用量の変化を測定
4. ロールバック機能をテスト

## トラブルシューティング

### 問題: スナップショットが削除されない

**原因1**: `metadata.json`で長期保持が設定されている

**解決策**: Icebergテーブルプロパティを確認し、削除
```sql
ALTER TABLE my_table UNSET TBLPROPERTIES ('history.expire.max-snapshot-age-ms');
```

**原因2**: Branch/tag-based retentionが設定されている

**解決策**: Branch/tagを削除
```sql
ALTER TABLE my_table DROP BRANCH staging;
```

### 問題: ストレージコストが減らない

**原因**: 非現行オブジェクトがまだ削除されていない

**解決策**: `NoncurrentDays`の経過を待つ、または設定を短縮

### 問題: 必要なスナップショットが削除された

**原因**: `MinimumSnapshots`または`MaximumSnapshotAge`の設定が不適切

**解決策**: 
1. AWS Supportに連絡して復旧を試みる
2. 設定値を見直す
3. バックアップから復元

### 問題: スナップショット管理が無効化されている

**原因**: 競合する設定が存在

**解決策**: 
1. テーブルプロパティを確認
```sql
SHOW TBLPROPERTIES my_table;
```
2. 競合する設定を削除
3. スナップショット管理を再有効化

## 参考リンク

- [AWS公式ドキュメント: Maintenance for tables](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-maintenance.html)
- [AWS公式ドキュメント: Maintenance for table buckets](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-table-buckets-maintenance.html)
- [Apache Iceberg公式ドキュメント: Snapshot Management](https://iceberg.apache.org/docs/latest/maintenance/#expire-snapshots)
- [テーブルメンテナンス](../02-operations/01-maintenance.md)
- [コンパクション戦略](./04-compaction-strategies.md)
- [災害復旧](./02-disaster-recovery.md)
