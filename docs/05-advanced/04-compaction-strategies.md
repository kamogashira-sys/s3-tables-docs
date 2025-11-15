# コンパクション戦略によるクエリパフォーマンス最適化

## 概要

コンパクション（Compaction）は、複数の小さなファイルを少数の大きなファイルに結合することで、Apache Icebergクエリのパフォーマンスを向上させるメンテナンス操作です。S3 Tablesでは、コンパクションがデフォルトで有効化されており、テーブルレベルで設定をカスタマイズできます。

### コンパクションが必要な理由

データレイクでは、頻繁な書き込み操作により多数の小さなファイルが生成されます。これにより以下の問題が発生します：

- **クエリパフォーマンスの低下**: 多数のファイルを読み取る必要があるため、クエリ実行時間が増加
- **メタデータオーバーヘッド**: ファイル数が多いほど、メタデータ管理のコストが増加
- **ストレージコスト**: 小さなファイルは効率的に圧縮されない可能性がある

コンパクションは、これらの問題を解決し、以下のメリットを提供します：

- クエリ実行時間の短縮
- メタデータ管理の効率化
- ストレージコストの削減
- 行レベル削除の適用

## 4つのコンパクション戦略

S3 Tablesは、クエリパターンとテーブル構造に応じて選択できる4つのコンパクション戦略を提供します。

### 1. Auto（デフォルト）

**概要**  
Amazon S3が自動的に最適なコンパクション戦略を選択します。これはすべてのテーブルのデフォルト戦略です。

**動作ロジック**
- テーブルメタデータに`sort_order`が定義されている場合 → **Sort**戦略を適用
- `sort_order`が定義されていない場合 → **Binpack**戦略を適用

**推奨シーン**
- 初期設定として推奨
- テーブル構造に応じた自動最適化が必要な場合
- 複数のテーブルで一貫した戦略を適用したい場合

**メリット**
- 設定不要で最適な戦略を自動選択
- テーブル構造の変更に自動対応
- 運用負荷の軽減

### 2. Binpack

**概要**  
小さなファイルを大きなファイルに結合し、通常100MB以上のサイズを目標とします。保留中の削除操作も同時に適用されます。

**動作の仕組み**
1. 小さなファイルを識別
2. ファイルを結合してターゲットサイズ（デフォルト512MB）に近づける
3. 行レベル削除を適用
4. 新しいスナップショットとして書き込み

**推奨シーン**
- ソート順序が定義されていないテーブル
- 頻繁な書き込みが発生するテーブル
- クエリパターンが予測不可能な場合

**メリット**
- シンプルで高速な処理
- 最小限のコスト
- すべてのファイル形式に対応

**対応ファイル形式**
- Apache Parquet
- Apache Avro
- Apache ORC

### 3. Sort

**概要**  
指定された列に基づいてデータを階層的にソートし、フィルタ操作のクエリパフォーマンスを向上させます。

**動作の仕組み**
1. テーブルプロパティの`sort_order`を読み取り
2. 指定された列の階層順にデータをソート
3. ソート済みデータをターゲットサイズのファイルに書き込み
4. 新しいスナップショットとして保存

**推奨シーン**
- 特定の列で頻繁にフィルタリングするクエリ
- 時系列データ（タイムスタンプ列でソート）
- カテゴリ別分析（カテゴリ列でソート）

**前提条件**
- テーブルプロパティに`sort_order`が定義されていること
- `s3tables:GetTableData`権限が必要

**メリット**
- フィルタクエリの大幅な高速化
- データスキップの効率化
- 範囲クエリのパフォーマンス向上

**設定例（Iceberg SQL）**
```sql
-- テーブル作成時にソート順序を定義
CREATE TABLE my_table (
  id BIGINT,
  timestamp TIMESTAMP,
  category STRING,
  value DOUBLE
)
USING iceberg
TBLPROPERTIES (
  'write.metadata.metrics.default' = 'full',
  'write.metadata.metrics.column.timestamp' = 'full'
)
PARTITIONED BY (days(timestamp))
SORTED BY (timestamp, category);
```

### 4. Z-order

**概要**  
複数の属性を単一のスカラー値にブレンドし、多次元クエリの効率的な実行を可能にします。

**動作の仕組み**
1. 複数の列の値をZ-orderカーブに沿ってマッピング
2. Z-order値に基づいてデータをソート
3. 近接するデータポイントを同じファイルに配置
4. 新しいスナップショットとして書き込み

**推奨シーン**
- 複数の列で同時にフィルタリングするクエリ
- 多次元分析（例: 地理座標、時間+カテゴリ）
- 複雑な結合条件を持つクエリ

**前提条件**
- Icebergテーブルプロパティに`sort_order`が定義されていること
- `s3tables:GetTableData`権限が必要

**メリット**
- 多次元クエリの大幅な高速化
- データの局所性向上
- 複数列フィルタの効率化

**注意事項**
- Binpackより高いコストが発生
- 処理時間が長い
- 適切な列選択が重要

**設定例（Iceberg SQL）**
```sql
-- Z-order用のソート順序を定義
ALTER TABLE my_table 
SET TBLPROPERTIES (
  'write.distribution-mode' = 'hash',
  'write.target-file-size-bytes' = '536870912'
);

-- Z-orderでソートする列を指定
ALTER TABLE my_table 
WRITE ORDERED BY z_order(latitude, longitude, timestamp);
```

## 戦略の選び方

### クエリパターン別推奨

| クエリパターン | 推奨戦略 | 理由 |
|--------------|---------|------|
| 特定列でのフィルタリング | Sort | データスキップによる高速化 |
| 複数列での同時フィルタリング | Z-order | 多次元データの局所性向上 |
| 全テーブルスキャン | Binpack | シンプルで低コスト |
| 予測不可能なクエリ | Auto | 自動最適化 |
| 時系列分析 | Sort（timestamp列） | 範囲クエリの効率化 |
| 地理空間クエリ | Z-order（lat, lon） | 空間的局所性の活用 |

### パフォーマンス比較

| 戦略 | 処理速度 | コスト | クエリ高速化 | 複雑性 |
|------|---------|--------|-------------|--------|
| Auto | 中 | 中 | 中 | 低 |
| Binpack | 高 | 低 | 低 | 低 |
| Sort | 中 | 中 | 高（単一列） | 中 |
| Z-order | 低 | 高 | 高（多次元） | 高 |

## 設定方法

### コンパクション戦略の変更

```bash
# Sort戦略に変更
aws s3tables put-table-maintenance-configuration \
  --table-bucket-arn arn:aws:s3tables:us-east-1:111122223333:bucket/my-table-bucket \
  --type icebergCompaction \
  --namespace my-namespace \
  --name my-table \
  --value '{
    "status": "enabled",
    "settings": {
      "icebergCompaction": {
        "strategy": "sort"
      }
    }
  }'

# Z-order戦略に変更
aws s3tables put-table-maintenance-configuration \
  --table-bucket-arn arn:aws:s3tables:us-east-1:111122223333:bucket/my-table-bucket \
  --type icebergCompaction \
  --namespace my-namespace \
  --name my-table \
  --value '{
    "status": "enabled",
    "settings": {
      "icebergCompaction": {
        "strategy": "z-order"
      }
    }
  }'

# Auto戦略に戻す
aws s3tables put-table-maintenance-configuration \
  --table-bucket-arn arn:aws:s3tables:us-east-1:111122223333:bucket/my-table-bucket \
  --type icebergCompaction \
  --namespace my-namespace \
  --name my-table \
  --value '{
    "status": "enabled",
    "settings": {
      "icebergCompaction": {
        "strategy": "auto"
      }
    }
  }'
```

### ターゲットファイルサイズの変更

```bash
# ターゲットファイルサイズを256MBに設定
aws s3tables put-table-maintenance-configuration \
  --table-bucket-arn arn:aws:s3tables:us-east-1:111122223333:bucket/my-table-bucket \
  --type icebergCompaction \
  --namespace my-namespace \
  --name my-table \
  --value '{
    "status": "enabled",
    "settings": {
      "icebergCompaction": {
        "targetFileSizeMB": 256
      }
    }
  }'
```

**ターゲットファイルサイズの範囲**
- **最小**: 64MB
- **最大**: 512MB
- **デフォルト**: 512MB

**推奨値**
- 小規模テーブル（< 1TB）: 128MB - 256MB
- 中規模テーブル（1TB - 10TB）: 256MB - 384MB
- 大規模テーブル（> 10TB）: 384MB - 512MB

### コンパクションの無効化

```bash
aws s3tables put-table-maintenance-configuration \
  --table-bucket-arn arn:aws:s3tables:us-east-1:111122223333:bucket/my-table-bucket \
  --type icebergCompaction \
  --namespace my-namespace \
  --name my-table \
  --value '{
    "status": "disabled",
    "settings": {
      "icebergCompaction": {
        "targetFileSizeMB": 512
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
  --type icebergCompaction
```

## コスト考慮事項

### コンパクション実行コスト

コンパクションは以下のAWSコストを発生させます：

1. **S3リクエストコスト**
   - GETリクエスト（既存ファイル読み取り）
   - PUTリクエスト（新ファイル書き込み）
   - DELETEリクエスト（古いファイル削除）

2. **S3ストレージコスト**
   - 一時的な重複ストレージ（コンパクション実行中）
   - 新旧ファイルの同時存在期間

3. **データ転送コスト**
   - S3内でのデータ移動（通常は無料）

### 戦略別コスト比較

| 戦略 | 相対コスト | コスト要因 |
|------|-----------|-----------|
| Binpack | 低 | シンプルなファイル結合のみ |
| Auto | 中 | 状況に応じて変動 |
| Sort | 中〜高 | ソート処理のオーバーヘッド |
| Z-order | 高 | 複雑な計算とソート処理 |

### コスト最適化のヒント

1. **適切な戦略選択**
   - クエリパターンに合わない高コスト戦略を避ける
   - 初期はAutoまたはBinpackから開始

2. **ターゲットファイルサイズの調整**
   - 大きすぎるサイズは無駄なデータ読み取りを増やす
   - 小さすぎるサイズはファイル数を増やす

3. **コンパクション頻度の調整**
   - 書き込み頻度に応じて調整
   - 低頻度の書き込みでは無効化も検討

4. **スナップショット管理との連携**
   - 古いスナップショットを適切に削除
   - ストレージコストを削減

## パフォーマンスチューニング

### 1. ソート列の選択（Sort/Z-order）

**効果的な列の特徴**
- 高いカーディナリティ（多様な値）
- クエリで頻繁にフィルタリングされる
- 範囲クエリで使用される

**避けるべき列**
- 低いカーディナリティ（例: boolean列）
- ほとんど使用されない列
- 頻繁に更新される列

### 2. Z-order列の組み合わせ

**推奨**
- 2〜4列程度に制限
- クエリで同時に使用される列を選択
- カーディナリティのバランスを考慮

**例: IoTデータ**
```sql
-- 良い例: 時間と地域で頻繁にフィルタリング
WRITE ORDERED BY z_order(timestamp, region_id, device_type)

-- 悪い例: 列が多すぎる
WRITE ORDERED BY z_order(timestamp, region_id, device_type, sensor_id, status, value)
```

### 3. パーティショニングとの組み合わせ

```sql
-- パーティショニングとソートの併用
CREATE TABLE sensor_data (
  sensor_id STRING,
  timestamp TIMESTAMP,
  region STRING,
  value DOUBLE
)
USING iceberg
PARTITIONED BY (days(timestamp))
SORTED BY (region, sensor_id);
```

**ベストプラクティス**
- パーティション列はソート列に含めない
- パーティション内でのソートを最適化
- パーティション数を適切に管理（推奨: 100〜1000パーティション）

## トラブルシューティング

### 問題: Sort/Z-order戦略が適用されない

**原因1**: `sort_order`が定義されていない

**解決策**:
```sql
-- ソート順序を追加
ALTER TABLE my_table 
WRITE ORDERED BY (timestamp, category);
```

**原因2**: `s3tables:GetTableData`権限不足

**解決策**:
IAMポリシーに以下を追加：
```json
{
  "Effect": "Allow",
  "Action": [
    "s3tables:GetTableData"
  ],
  "Resource": "arn:aws:s3tables:*:*:bucket/*/table/*"
}
```

### 問題: コンパクションが完了しない

**原因**: テーブルサイズが大きすぎる

**解決策**:
1. ターゲットファイルサイズを増やす
2. パーティショニングを追加
3. 戦略をBinpackに変更（一時的）

### 問題: クエリパフォーマンスが改善しない

**原因**: 不適切な戦略選択

**解決策**:
1. クエリパターンを分析
2. 適切な戦略に変更
3. ソート列を見直す

### 問題: コストが予想以上に高い

**原因**: Z-order戦略の過剰使用

**解決策**:
1. クエリパターンを再評価
2. SortまたはBinpackに変更
3. コンパクション頻度を調整

## ベストプラクティス

### 1. 段階的なアプローチ
1. Autoで開始
2. クエリパターンを分析
3. 必要に応じてSortまたはZ-orderに移行

### 2. 定期的な見直し
- クエリパターンの変化を監視
- パフォーマンスメトリクスを追跡
- 戦略を定期的に再評価

### 3. テスト環境での検証
- 本番適用前にテスト環境で検証
- パフォーマンスとコストを測定
- 複数の戦略を比較

### 4. ドキュメント化
- 選択した戦略の理由を記録
- ソート列の選択根拠を文書化
- パフォーマンス改善結果を記録

## 参考リンク

- [AWS公式ドキュメント: Maintenance for tables](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-maintenance.html)
- [AWS公式ドキュメント: S3 Tables maintenance overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-maintenance-overview.html)
- [AWS公式ドキュメント: Considerations and limitations for maintenance jobs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-considerations.html)
- [Apache Iceberg公式ドキュメント: Table Maintenance](https://iceberg.apache.org/docs/latest/maintenance/)
- [テーブルメンテナンス](../02-operations/01-maintenance.md)
- [スナップショット管理](./05-snapshot-management.md)
