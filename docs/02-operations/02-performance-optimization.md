# Amazon S3 Tables パフォーマンス最適化ガイド


## 目次

- [概要](#概要)
  - [パフォーマンス特性](#パフォーマンス特性)
- [1. データレイアウトの最適化](#1-データレイアウトの最適化)
  - [1.1 ファイルサイズの最適化](#11-ファイルサイズの最適化)
    - [推奨ファイルサイズ](#推奨ファイルサイズ)
    - [ファイルサイズ設定](#ファイルサイズ設定)
    - [小さなファイルの問題](#小さなファイルの問題)
  - [1.2 圧縮フォーマットの選択](#12-圧縮フォーマットの選択)
    - [推奨圧縮アルゴリズム](#推奨圧縮アルゴリズム)
    - [圧縮設定](#圧縮設定)
  - [1.3 データ配置戦略](#13-データ配置戦略)
    - [ソート順序（Sort Order）](#ソート順序sort-order)
    - [Z-Order最適化](#z-order最適化)
- [2. パーティショニング戦略](#2-パーティショニング戦略)
  - [2.1 パーティションキーの選択](#21-パーティションキーの選択)
    - [選択基準](#選択基準)
    - [Hidden Partitioning](#hidden-partitioning)
  - [2.2 パーティション数の最適化](#22-パーティション数の最適化)
    - [推奨パーティション数](#推奨パーティション数)
  - [2.3 パーティション進化](#23-パーティション進化)
- [3. メタデータ最適化](#3-メタデータ最適化)
  - [3.1 カラム統計の管理](#31-カラム統計の管理)
    - [統計情報の重要性](#統計情報の重要性)
    - [選択的統計収集](#選択的統計収集)
  - [3.2 メタデータキャッシング](#32-メタデータキャッシング)
    - [クエリエンジンのメタデータキャッシュ](#クエリエンジンのメタデータキャッシュ)
  - [3.3 マニフェストファイルの管理](#33-マニフェストファイルの管理)
    - [マニフェストファイルの役割](#マニフェストファイルの役割)
- [4. クエリ最適化](#4-クエリ最適化)
  - [4.1 述語プッシュダウン（Predicate Pushdown）](#41-述語プッシュダウンpredicate-pushdown)
    - [概要](#概要)
    - [効果的な述語の書き方](#効果的な述語の書き方)
  - [4.2 プロジェクションプッシュダウン（Projection Pushdown）](#42-プロジェクションプッシュダウンprojection-pushdown)
    - [概要](#概要)
    - [効果的なSELECT句の書き方](#効果的なselect句の書き方)
  - [4.3 クエリ結果のキャッシング](#43-クエリ結果のキャッシング)
    - [Athena Query Result Reuse](#athena-query-result-reuse)
- [5. 書き込み最適化](#5-書き込み最適化)
  - [5.1 更新戦略の選択](#51-更新戦略の選択)
    - [Copy-on-Write（CoW）vs Merge-on-Read（MoR）](#copy-on-writecowvs-merge-on-readmor)
  - [5.2 配布モード（Distribution Mode）](#52-配布モードdistribution-mode)
    - [Sparkの配布モード設定](#sparkの配布モード設定)
  - [5.3 ファイルフォーマットの選択](#53-ファイルフォーマットの選択)
    - [書き込み時のフォーマット](#書き込み時のフォーマット)
- [6. コンパクション戦略](#6-コンパクション戦略)
  - [6.1 S3 Tablesの自動コンパクション](#61-s3-tablesの自動コンパクション)
    - [コンパクション戦略](#コンパクション戦略)
    - [設定方法](#設定方法)
  - [6.2 AWS Glue Data Catalogのコンパクション](#62-aws-glue-data-catalogのコンパクション)
    - [自動コンパクション](#自動コンパクション)
  - [6.3 手動コンパクション](#63-手動コンパクション)
    - [Sparkでの手動コンパクション](#sparkでの手動コンパクション)
- [7. ストレージ最適化](#7-ストレージ最適化)
  - [7.1 スナップショット管理](#71-スナップショット管理)
    - [スナップショット保持ポリシー](#スナップショット保持ポリシー)
  - [7.2 孤立ファイルの削除](#72-孤立ファイルの削除)
    - [孤立ファイルとは](#孤立ファイルとは)
  - [7.3 S3ストレージクラスの活用](#73-s3ストレージクラスの活用)
    - [S3 Intelligent-Tiering](#s3-intelligent-tiering)
- [8. ネットワークとI/O最適化](#8-ネットワークとio最適化)
  - [8.1 水平スケーリング](#81-水平スケーリング)
    - [複数接続の活用](#複数接続の活用)
  - [8.2 バイトレンジフェッチ](#82-バイトレンジフェッチ)
    - [部分読み取りの活用](#部分読み取りの活用)
  - [8.3 同一リージョン配置](#83-同一リージョン配置)
    - [レイテンシとコストの削減](#レイテンシとコストの削減)
- [9. パフォーマンス監視](#9-パフォーマンス監視)
  - [9.1 CloudWatchメトリクス](#91-cloudwatchメトリクス)
    - [S3メトリクス](#s3メトリクス)
  - [9.2 AWS Glue Data Catalogメトリクス](#92-aws-glue-data-catalogメトリクス)
    - [テーブルオプティマイザーメトリクス](#テーブルオプティマイザーメトリクス)
  - [9.3 クエリパフォーマンスの分析](#93-クエリパフォーマンスの分析)
    - [Athenaクエリメトリクス](#athenaクエリメトリクス)
- [10. ベストプラクティスまとめ](#10-ベストプラクティスまとめ)
  - [10.1 設計フェーズ](#101-設計フェーズ)
    - [テーブル設計](#テーブル設計)
    - [スキーマ設計](#スキーマ設計)
  - [10.2 運用フェーズ](#102-運用フェーズ)
    - [定期メンテナンス](#定期メンテナンス)
    - [自動メンテナンス設定](#自動メンテナンス設定)
  - [10.3 監視とアラート](#103-監視とアラート)
    - [CloudWatchアラーム設定](#cloudwatchアラーム設定)
  - [10.4 パフォーマンスチューニングチェックリスト](#104-パフォーマンスチューニングチェックリスト)
    - [初期設定](#初期設定)
    - [定期レビュー](#定期レビュー)
    - [トラブルシューティング](#トラブルシューティング)
- [11. 出典](#11-出典)
  - [AWS公式ドキュメント](#aws公式ドキュメント)
  - [AWS Prescriptive Guidance](#aws-prescriptive-guidance)
  - [その他の情報源](#その他の情報源)

## 概要

Amazon S3 Tablesは、Apache Icebergフォーマットを使用した分析ワークロード向けの最適化されたストレージサービスです。本ドキュメントでは、S3 Tablesのパフォーマンスを最大化するためのベストプラクティスと最適化戦略を説明します。

### パフォーマンス特性

S3 Tablesは、S3の高いスケーラビリティとパフォーマンスを継承しています。

| 項目 | 性能 | 備考 |
|------|------|------|
| **リクエストレート（PUT/POST/DELETE）** | 3,500リクエスト/秒/プレフィックス | プレフィックス数に制限なし |
| **リクエストレート（GET/HEAD）** | 5,500リクエスト/秒/プレフィックス | プレフィックス数に制限なし |
| **レイテンシ** | 100-200ミリ秒 | 小さなオブジェクト、同一リージョン |
| **スループット** | 最大100 Gb/s | 単一EC2インスタンス |
| **スケーラビリティ** | 無制限 | 自動スケーリング |

**重要な注意事項**:
- スケーリングは段階的に行われ、瞬時ではありません
- 新しい高いリクエストレートへのスケーリング中、503（Slow Down）エラーが発生する可能性があります
- 実際のパフォーマンスは、ワークロードの特性、使用パターン、システム構成によって異なります

## 1. データレイアウトの最適化

### 1.1 ファイルサイズの最適化

#### 推奨ファイルサイズ

Apache Icebergテーブルのパフォーマンスは、ファイルサイズに大きく依存します。

| テーブルサイズ | 推奨ファイルサイズ | 理由 |
|--------------|------------------|------|
| **小規模（< 1TB）** | 128-256 MB | メタデータオーバーヘッド削減 |
| **中規模（1-10TB）** | 256-512 MB | バランスの取れたパフォーマンス |
| **大規模（> 10TB）** | 512 MB - 1 GB | クエリプランニング時間短縮 |

**AWS Prescriptive Guidanceの推奨**:
- **最小ファイルサイズ**: 100 MB以上
- **デフォルト目標サイズ**: 512 MB（`write.target-file-size-bytes`プロパティ）
- **定期的なコンパクション**: 小さなファイルを大きなファイルに統合

#### ファイルサイズ設定

```sql
-- テーブル作成時にファイルサイズを設定
CREATE TABLE my_table (
  id BIGINT,
  name STRING,
  created_at TIMESTAMP
)
USING iceberg
TBLPROPERTIES (
  'write.target-file-size-bytes' = '536870912'  -- 512 MB
);

-- 既存テーブルのファイルサイズ設定を変更
ALTER TABLE my_table SET TBLPROPERTIES (
  'write.target-file-size-bytes' = '1073741824'  -- 1 GB
);
```

#### 小さなファイルの問題

小さなファイルが多数存在すると、以下の問題が発生します:

1. **メタデータオーバーヘッド増加**: 各ファイルのメタデータ管理コスト
2. **クエリプランニング時間増加**: クエリエンジンがスキャンするファイル数の増加
3. **S3リクエスト数増加**: ファイルごとにS3 APIコールが必要
4. **読み取りパフォーマンス低下**: ファイルオープン/クローズのオーバーヘッド

**AWS Glue Data Catalogのコンパクション閾値**:
- **ファイル数**: 100ファイル以上
- **ファイルサイズ**: 目標ファイルサイズの75%未満
- **自動実行**: 閾値を超えると自動的にコンパクション開始

### 1.2 圧縮フォーマットの選択

#### 推奨圧縮アルゴリズム

| 圧縮形式 | 圧縮率 | 速度 | 推奨用途 |
|---------|--------|------|---------|
| **ZSTD** | 高 | 高速 | **推奨**: バランスの取れた選択 |
| **GZIP** | 高 | 中速 | 互換性重視 |
| **Snappy** | 中 | 非常に高速 | 書き込み重視 |
| **LZ4** | 低 | 非常に高速 | リアルタイム処理 |

**AWS Prescriptive Guidanceの推奨**:
- **ZSTD（Zstandard）**: GZIPと比較して高速で同等の圧縮率
- **Parquetフォーマット**: カラムナーストレージで分析クエリに最適

#### 圧縮設定

```sql
-- テーブル作成時に圧縮を設定
CREATE TABLE my_table (
  id BIGINT,
  name STRING,
  data STRING
)
USING iceberg
TBLPROPERTIES (
  'write.parquet.compression-codec' = 'zstd',
  'write.parquet.compression-level' = '3'
);
```

**S3 Tablesの制限事項**:
- **サポート圧縮**: Parquet、Avro、ORC
- **非サポート圧縮**: brotli、lz4（コンパクション時）
- **自動圧縮**: S3 Tablesメンテナンスジョブで自動実行可能

### 1.3 データ配置戦略

#### ソート順序（Sort Order）

ソート順序を定義することで、ファイルプルーニング効率が向上し、S3リクエスト数が削減されます。

```sql
-- ソート順序を設定
ALTER TABLE my_table WRITE ORDERED BY (date, region);

-- 複数カラムのソート順序
ALTER TABLE my_table WRITE ORDERED BY (
  year DESC,
  month DESC,
  day DESC,
  customer_id
);
```

**メリット**:
1. **ファイルプルーニング**: クエリ条件に基づいて不要なファイルをスキップ
2. **S3リクエスト削減**: 読み取るファイル数の削減
3. **クエリパフォーマンス向上**: データの局所性向上

#### Z-Order最適化

Z-Order最適化は、複数カラムを同時にクエリする場合に効果的です。

```sql
-- Z-Order最適化を設定（AWS Glue）
-- AWS GlueコンソールまたはAPIで設定
```

**Z-Orderの特徴**:
- **多次元クエリ**: 複数カラムのフィルタリングに最適
- **データクラスタリング**: 類似データを物理的に近くに配置
- **クエリコスト削減**: スキャンするデータ量の削減

**使用例**:
- 日付と地域で頻繁にフィルタリングするテーブル
- 複数のディメンションでスライス&ダイスするOLAPワークロード

## 2. パーティショニング戦略

### 2.1 パーティションキーの選択

#### 選択基準

効果的なパーティションキーの選択は、クエリパフォーマンスに大きな影響を与えます。

| 基準 | 説明 | 例 |
|------|------|-----|
| **低カーディナリティ** | 値の種類が少ない | 日付、地域、カテゴリ |
| **クエリフィルタ頻度** | WHERE句で頻繁に使用 | `WHERE date = '2024-01-01'` |
| **データ分布の均等性** | パーティション間でデータ量が均等 | 日付（毎日同程度のデータ） |
| **時系列データ** | 時間ベースのパーティション | 年、月、日 |

**AWS Prescriptive Guidanceの推奨**:
- **低カーディナリティカラム**: 高カーディナリティ（例: ユーザーID）は避ける
- **Hidden Partitioning**: Icebergの隠しパーティショニング機能を活用
- **パーティション進化**: スキーマ進化と同様にパーティション戦略も進化可能

#### Hidden Partitioning

Icebergの隠しパーティショニングは、ユーザーがパーティションカラムを意識せずにクエリできます。

```sql
-- 隠しパーティショニングの例
CREATE TABLE events (
  event_id BIGINT,
  event_time TIMESTAMP,
  user_id STRING,
  event_type STRING
)
USING iceberg
PARTITIONED BY (days(event_time));

-- クエリ時はパーティションカラムを意識不要
SELECT * FROM events
WHERE event_time >= '2024-01-01'
  AND event_time < '2024-02-01';
```

**メリット**:
- **クエリの簡素化**: パーティションカラムを明示的に指定不要
- **パーティション進化**: パーティション戦略の変更が容易
- **エラー削減**: パーティション指定ミスの防止

### 2.2 パーティション数の最適化

#### 推奨パーティション数

| テーブルサイズ | 推奨パーティション数 | パーティションあたりのデータ量 |
|--------------|---------------------|----------------------------|
| **小規模（< 1TB）** | 10-100 | 10-100 GB |
| **中規模（1-10TB）** | 100-1,000 | 1-10 GB |
| **大規模（> 10TB）** | 1,000-10,000 | 1-10 GB |

**過剰なパーティショニングの問題**:
1. **メタデータオーバーヘッド**: パーティションごとのメタデータ管理コスト
2. **クエリプランニング時間**: パーティション数に比例して増加
3. **小さなファイル問題**: パーティションあたりのデータ量が少ない

**不十分なパーティショニングの問題**:
1. **フルスキャン**: パーティションプルーニングの効果が低い
2. **大きなファイル**: パーティションあたりのファイルサイズが大きすぎる
3. **並列処理の制限**: パーティション数が並列度の上限

### 2.3 パーティション進化

Icebergのパーティション進化機能により、既存データを書き換えずにパーティション戦略を変更できます。

```sql
-- 初期パーティション: 月単位
CREATE TABLE sales (
  sale_id BIGINT,
  sale_date DATE,
  amount DECIMAL(10,2)
)
USING iceberg
PARTITIONED BY (months(sale_date));

-- パーティション進化: 日単位に変更
ALTER TABLE sales
ADD PARTITION FIELD days(sale_date);

-- 古いパーティションフィールドを削除
ALTER TABLE sales
DROP PARTITION FIELD months(sale_date);
```

**メリット**:
- **データ再書き込み不要**: 既存データはそのまま
- **段階的移行**: 新しいデータから新しいパーティション戦略を適用
- **柔軟性**: ワークロードの変化に応じて最適化


## 3. メタデータ最適化

### 3.1 カラム統計の管理

#### 統計情報の重要性

Icebergは、カラムごとに統計情報（最小値、最大値、NULL数など）を保持し、クエリ最適化に活用します。

**デフォルト設定**:
- **統計収集カラム数**: 100カラム（`write.metadata.metrics.default`）
- **自動収集**: 書き込み時に自動的に収集

#### 選択的統計収集

すべてのカラムで統計を収集する必要はありません。クエリで頻繁に使用するカラムのみに絞ることで、メタデータサイズを削減できます。

```sql
-- 特定カラムの統計収集を無効化
ALTER TABLE my_table SET TBLPROPERTIES (
  'write.metadata.metrics.column.large_text_column' = 'none'
);

-- 特定カラムの統計収集を有効化
ALTER TABLE my_table SET TBLPROPERTIES (
  'write.metadata.metrics.column.filter_column' = 'full'
);
```

**統計収集レベル**:
- `full`: すべての統計を収集（デフォルト）
- `counts`: NULL数とレコード数のみ
- `truncate(N)`: 文字列をN文字に切り詰めて統計収集
- `none`: 統計収集を無効化

### 3.2 メタデータキャッシング

#### クエリエンジンのメタデータキャッシュ

クエリエンジン（Athena、EMR、Glue）は、Icebergメタデータをキャッシュしてクエリプランニング時間を短縮します。

**Athena v3のメタデータキャッシュ**:
- **自動キャッシュ**: メタデータファイルを自動的にキャッシュ
- **キャッシュ期間**: クエリセッション中有効
- **更新検知**: テーブル更新時に自動的にキャッシュ無効化

**EMR Sparkのメタデータキャッシュ**:
```python
# Sparkセッションでメタデータキャッシュを有効化
spark.conf.set("spark.sql.catalog.my_catalog.cache-enabled", "true")
```

### 3.3 マニフェストファイルの管理

#### マニフェストファイルの役割

マニフェストファイルは、データファイルのリストとメタデータを含みます。

**マニフェストファイルの最適化**:
- **コンパクション**: 小さなマニフェストファイルを統合
- **定期的なクリーンアップ**: 古いマニフェストファイルの削除
- **適切なサイズ**: 1マニフェストあたり数千ファイル

```sql
-- マニフェストファイルのコンパクション（Spark）
CALL system.rewrite_manifests('my_catalog.my_db.my_table');
```

## 4. クエリ最適化

### 4.1 述語プッシュダウン（Predicate Pushdown）

#### 概要

述語プッシュダウンは、WHERE句の条件をストレージレイヤーに押し下げ、読み取るデータ量を削減します。

**Icebergの述語プッシュダウン**:
1. **パーティションプルーニング**: パーティション条件に基づいてパーティション全体をスキップ
2. **ファイルプルーニング**: ファイル統計に基づいて不要なファイルをスキップ
3. **行グループプルーニング**: Parquetの行グループ統計に基づいてスキップ

#### 効果的な述語の書き方

```sql
-- 良い例: パーティションカラムでフィルタリング
SELECT * FROM sales
WHERE sale_date = '2024-01-01'
  AND region = 'us-east-1';

-- 良い例: 統計情報を活用
SELECT * FROM sales
WHERE amount > 1000
  AND sale_date BETWEEN '2024-01-01' AND '2024-01-31';

-- 悪い例: 関数を使用（統計情報を活用できない）
SELECT * FROM sales
WHERE YEAR(sale_date) = 2024;  -- 代わりに sale_date >= '2024-01-01' を使用

-- 悪い例: OR条件（パーティションプルーニングが効かない）
SELECT * FROM sales
WHERE region = 'us-east-1' OR region = 'us-west-2';  -- 代わりに IN を使用
```

### 4.2 プロジェクションプッシュダウン（Projection Pushdown）

#### 概要

プロジェクションプッシュダウンは、必要なカラムのみを読み取ることで、I/Oを削減します。

**Parquetのカラムナーストレージ**:
- **カラム単位の読み取り**: 必要なカラムのみをS3から読み取り
- **I/O削減**: 不要なカラムのデータ転送を回避
- **ネットワーク帯域幅の節約**: データ転送量の削減

#### 効果的なSELECT句の書き方

```sql
-- 良い例: 必要なカラムのみを選択
SELECT customer_id, order_date, total_amount
FROM orders
WHERE order_date = '2024-01-01';

-- 悪い例: SELECT *（すべてのカラムを読み取り）
SELECT *
FROM orders
WHERE order_date = '2024-01-01';
```

**パフォーマンス改善例**:
- テーブル: 100カラム、各カラム10MB
- SELECT *: 1,000MB読み取り
- SELECT 3カラム: 30MB読み取り（97%削減）

### 4.3 クエリ結果のキャッシング

#### Athena Query Result Reuse

Athenaは、同一クエリの結果を再利用してパフォーマンスを向上させます。

**設定方法**:
```sql
-- Athenaワークグループ設定でクエリ結果の再利用を有効化
-- AWS Management Console > Athena > Workgroups > Settings
-- Query result reuse: Enabled
-- Maximum age: 60 minutes
```

**メリット**:
- **レイテンシ削減**: キャッシュヒット時は数秒で結果を返す
- **コスト削減**: S3スキャンコストの削減
- **スループット向上**: 同時実行クエリ数の増加

## 5. 書き込み最適化

### 5.1 更新戦略の選択

#### Copy-on-Write（CoW）vs Merge-on-Read（MoR）

| 戦略 | 書き込み | 読み取り | 推奨用途 |
|------|---------|---------|---------|
| **Copy-on-Write** | 遅い | 高速 | 読み取り重視ワークロード |
| **Merge-on-Read** | 高速 | 中速 | 書き込み重視ワークロード |

**Copy-on-Write（デフォルト）**:
```sql
-- CoWはデフォルト設定
CREATE TABLE my_table (
  id BIGINT,
  name STRING
)
USING iceberg
TBLPROPERTIES (
  'write.update.mode' = 'copy-on-write',
  'write.delete.mode' = 'copy-on-write',
  'write.merge.mode' = 'copy-on-write'
);
```

**Merge-on-Read**:
```sql
-- MoRを有効化
CREATE TABLE my_table (
  id BIGINT,
  name STRING
)
USING iceberg
TBLPROPERTIES (
  'write.update.mode' = 'merge-on-read',
  'write.delete.mode' = 'merge-on-read',
  'write.merge.mode' = 'merge-on-read'
);
```

**AWS Prescriptive Guidanceの推奨**:
- **読み取り最適化**: Copy-on-Write（分析クエリが多い場合）
- **書き込み最適化**: Merge-on-Read（頻繁な更新がある場合）
- **定期的なコンパクション**: MoR使用時は必須

### 5.2 配布モード（Distribution Mode）

#### Sparkの配布モード設定

Sparkでデータを書き込む際、配布モードを設定することでパフォーマンスを最適化できます。

```python
# 配布モードの設定
spark.conf.set("spark.sql.iceberg.distribution-mode", "hash")

# オプション:
# - none: 配布なし（高速だがファイルサイズが不均等）
# - hash: ハッシュ配布（パーティション内でデータを均等に分散）
# - range: レンジ配布（ソート順序を維持）
```

**推奨設定**:
- **パーティションテーブル**: `hash`（均等なファイルサイズ）
- **非パーティションテーブル**: `range`（ソート順序維持）
- **高速書き込み**: `none`（ただしコンパクションが必要）

### 5.3 ファイルフォーマットの選択

#### 書き込み時のフォーマット

**Avro vs Parquet**:
- **Avro**: 書き込み高速、スキーマ進化に強い
- **Parquet**: 読み取り高速、圧縮率高い

**AWS Prescriptive Guidanceの推奨**:
- **ストリーミング書き込み**: Avro（高速書き込み）
- **バッチ処理**: Parquet（読み取り最適化）
- **ハイブリッド**: Avroで書き込み、後でParquetに変換

```python
# Avroで書き込み
df.write.format("iceberg") \
  .option("write-format", "avro") \
  .save("my_catalog.my_db.my_table")

# Parquetに変換（コンパクション時）
spark.sql("""
  CALL system.rewrite_data_files(
    table => 'my_catalog.my_db.my_table',
    options => map('target-file-size-bytes', '536870912')
  )
""")
```

## 6. コンパクション戦略

### 6.1 S3 Tablesの自動コンパクション

#### コンパクション戦略

S3 Tablesは、4種類のコンパクション戦略をサポートしています。

| 戦略 | 説明 | 推奨用途 |
|------|------|---------|
| **Auto** | 自動選択 | 一般的なワークロード |
| **Binpack** | ファイル統合 | 小さなファイルが多い場合 |
| **Sort** | ソート順序維持 | 範囲クエリが多い場合 |
| **Z-order** | 多次元最適化 | 複数カラムでフィルタリング |

#### 設定方法

```json
// S3 Tablesメンテナンス設定ファイル
{
  "icebergCompaction": {
    "enabled": true,
    "settings": {
      "targetFileSizeMB": 512,
      "strategy": "auto"
    }
  }
}
```

**AWS CLIでの設定**:
```bash
# コンパクション設定の更新
aws s3tables put-table-maintenance-configuration \
  --table-bucket-arn arn:aws:s3tables:us-east-1:123456789012:bucket/my-bucket \
  --namespace my-namespace \
  --name my-table \
  --value file://maintenance-config.json
```

### 6.2 AWS Glue Data Catalogのコンパクション

#### 自動コンパクション

AWS Glue Data Catalogは、Icebergテーブルのマネージドコンパクションを提供します。

**コンパクション開始条件**:
- **ファイル数**: 100ファイル以上
- **ファイルサイズ**: 目標ファイルサイズの75%未満
- **フォーマット**: Parquetのみサポート

**設定方法**:
```bash
# AWS CLIでコンパクションを有効化
aws glue create-table-optimizer \
  --catalog-id 123456789012 \
  --database-name my_database \
  --table-name my_table \
  --table-optimizer-configuration '{
    "enabled": true,
    "roleArn": "arn:aws:iam::123456789012:role/GlueTableOptimizerRole"
  }' \
  --type compaction
```

**コンパクション戦略の選択**:
```bash
# Binpack戦略
aws glue update-table-optimizer \
  --catalog-id 123456789012 \
  --database-name my_database \
  --table-name my_table \
  --type compaction \
  --table-optimizer-configuration '{
    "enabled": true,
    "roleArn": "arn:aws:iam::123456789012:role/GlueTableOptimizerRole",
    "configuration": {
      "compactionStrategy": "binpack"
    }
  }'

# Sort戦略（ソートカラム指定）
aws glue update-table-optimizer \
  --catalog-id 123456789012 \
  --database-name my_database \
  --table-name my_table \
  --type compaction \
  --table-optimizer-configuration '{
    "enabled": true,
    "roleArn": "arn:aws:iam::123456789012:role/GlueTableOptimizerRole",
    "configuration": {
      "compactionStrategy": "sort",
      "sortColumns": ["date", "region"]
    }
  }'

# Z-order戦略
aws glue update-table-optimizer \
  --catalog-id 123456789012 \
  --database-name my_database \
  --table-name my_table \
  --type compaction \
  --table-optimizer-configuration '{
    "enabled": true,
    "roleArn": "arn:aws:iam::123456789012:role/GlueTableOptimizerRole",
    "configuration": {
      "compactionStrategy": "zorder",
      "zorderColumns": ["date", "region", "customer_id"]
    }
  }'
```

### 6.3 手動コンパクション

#### Sparkでの手動コンパクション

```python
# Binpackコンパクション
spark.sql("""
  CALL system.rewrite_data_files(
    table => 'my_catalog.my_db.my_table',
    options => map(
      'target-file-size-bytes', '536870912',
      'min-input-files', '2'
    )
  )
""")

# Sortコンパクション
spark.sql("""
  CALL system.rewrite_data_files(
    table => 'my_catalog.my_db.my_table',
    strategy => 'sort',
    sort_order => 'date,region',
    options => map('target-file-size-bytes', '536870912')
  )
""")

# Z-orderコンパクション
spark.sql("""
  CALL system.rewrite_data_files(
    table => 'my_catalog.my_db.my_table',
    strategy => 'zorder',
    zorder_columns => array('date', 'region'),
    options => map('target-file-size-bytes', '536870912')
  )
""")
```


## 7. ストレージ最適化

### 7.1 スナップショット管理

#### スナップショット保持ポリシー

古いスナップショットを削除することで、メタデータサイズとストレージコストを削減できます。

**S3 Tablesの設定**:
```json
{
  "icebergSnapshotManagement": {
    "enabled": true,
    "settings": {
      "minimumSnapshots": 3,
      "maximumSnapshotAge": 7
    }
  }
}
```

**AWS Glue Data Catalogの設定**:
```bash
# スナップショット保持オプティマイザーを有効化
aws glue create-table-optimizer \
  --catalog-id 123456789012 \
  --database-name my_database \
  --table-name my_table \
  --table-optimizer-configuration '{
    "enabled": true,
    "roleArn": "arn:aws:iam::123456789012:role/GlueTableOptimizerRole",
    "configuration": {
      "retainSnapshots": 5,
      "retainDays": 7
    }
  }' \
  --type retention
```

**手動でのスナップショット削除**:
```python
# Sparkでスナップショットを削除
spark.sql("""
  CALL system.expire_snapshots(
    table => 'my_catalog.my_db.my_table',
    older_than => TIMESTAMP '2024-01-01 00:00:00',
    retain_last => 5
  )
""")
```

### 7.2 孤立ファイルの削除

#### 孤立ファイルとは

孤立ファイルは、Icebergメタデータから参照されなくなったデータファイルです。

**発生原因**:
- 失敗したETLジョブ
- テーブル削除
- コンパクション後の古いファイル

**S3 Tablesの自動削除**:
```json
{
  "icebergUnreferencedFileRemoval": {
    "enabled": true,
    "settings": {
      "unreferencedDays": 3,
      "nonCurrentDays": 10
    }
  }
}
```

**AWS Glue Data Catalogの設定**:
```bash
# 孤立ファイル削除オプティマイザーを有効化
aws glue create-table-optimizer \
  --catalog-id 123456789012 \
  --database-name my_database \
  --table-name my_table \
  --table-optimizer-configuration '{
    "enabled": true,
    "roleArn": "arn:aws:iam::123456789012:role/GlueTableOptimizerRole",
    "configuration": {
      "olderThan": 3
    }
  }' \
  --type orphan_file_deletion
```

**手動での孤立ファイル削除**:
```python
# Sparkで孤立ファイルを削除
spark.sql("""
  CALL system.remove_orphan_files(
    table => 'my_catalog.my_db.my_table',
    older_than => TIMESTAMP '2024-01-01 00:00:00'
  )
""")
```

### 7.3 S3ストレージクラスの活用

#### S3 Intelligent-Tiering

S3 Intelligent-Tieringは、アクセスパターンに基づいて自動的にストレージクラスを変更します。

**メリット**:
- **コスト削減**: アクセス頻度の低いデータを低コストストレージに移動
- **自動化**: 手動でのライフサイクル管理不要
- **パフォーマンス維持**: 頻繁にアクセスされるデータは高速ストレージに保持

**AWS Prescriptive Guidanceの推奨**:
- **長期保存データ**: S3 Intelligent-Tieringを有効化
- **アーカイブ層**: Deep Archive Access層を有効化（90日以上アクセスなし）

**注意事項**:
- S3 Tablesは現在、S3 Intelligent-Tieringを直接サポートしていません
- S3汎用バケットでセルフマネージドIcebergを使用する場合に適用可能

## 8. ネットワークとI/O最適化

### 8.1 水平スケーリング

#### 複数接続の活用

S3は、複数の並列接続をサポートしており、スループットを向上させることができます。

**ベストプラクティス**:
- **並列リクエスト**: 複数のスレッド/プロセスで同時にリクエスト
- **プレフィックスの分散**: 異なるプレフィックスに分散してリクエスト
- **接続数の制限なし**: S3は接続数に制限なし

**Sparkの並列度設定**:
```python
# Sparkの並列度を設定
spark.conf.set("spark.sql.shuffle.partitions", "200")
spark.conf.set("spark.default.parallelism", "200")

# S3接続の並列度
spark.conf.set("spark.hadoop.fs.s3a.connection.maximum", "100")
spark.conf.set("spark.hadoop.fs.s3a.threads.max", "64")
```

### 8.2 バイトレンジフェッチ

#### 部分読み取りの活用

バイトレンジフェッチを使用することで、必要な部分のみを読み取ることができます。

**推奨サイズ**:
- **8-16 MB**: 一般的な推奨サイズ
- **マルチパートアップロードと同じサイズ**: パート境界に合わせる

**Sparkの設定**:
```python
# バイトレンジフェッチのサイズ設定
spark.conf.set("spark.hadoop.fs.s3a.readahead.range", "8388608")  # 8 MB
```

### 8.3 同一リージョン配置

#### レイテンシとコストの削減

S3 TablesとコンピュートリソースEMR、Glue）を同一リージョンに配置することで、パフォーマンスとコストを最適化できます。

**メリット**:
- **レイテンシ削減**: ネットワークレイテンシの最小化
- **データ転送コスト削減**: 同一リージョン内の転送は無料
- **スループット向上**: リージョン内の高速ネットワーク

**推奨構成**:
```
S3 Tables (us-east-1)
  ↓ 同一リージョン
Amazon Athena (us-east-1)
Amazon EMR (us-east-1)
AWS Glue (us-east-1)
```

## 9. パフォーマンス監視

### 9.1 CloudWatchメトリクス

#### S3メトリクス

S3 Tablesのパフォーマンスを監視するために、CloudWatchメトリクスを活用します。

**主要メトリクス**:
- `BucketSizeBytes`: バケットサイズ
- `NumberOfObjects`: オブジェクト数
- `AllRequests`: 総リクエスト数
- `GetRequests`: GETリクエスト数
- `PutRequests`: PUTリクエスト数
- `4xxErrors`: クライアントエラー数
- `5xxErrors`: サーバーエラー数

**503エラーの監視**:
```bash
# CloudWatch Logsでの503エラー監視
aws logs put-metric-filter \
  --log-group-name /aws/s3/tables \
  --filter-name S3Tables503Errors \
  --filter-pattern '[... status_code=503 ...]' \
  --metric-transformations \
    metricName=503Errors,metricNamespace=S3Tables,metricValue=1
```

### 9.2 AWS Glue Data Catalogメトリクス

#### テーブルオプティマイザーメトリクス

AWS Glue Data Catalogは、テーブルオプティマイザーのメトリクスを提供します。

**主要メトリクス**:
- `CompactionJobsSucceeded`: 成功したコンパクションジョブ数
- `CompactionJobsFailed`: 失敗したコンパクションジョブ数
- `CompactionJobDuration`: コンパクションジョブの実行時間
- `FilesCompacted`: コンパクトされたファイル数
- `BytesCompacted`: コンパクトされたバイト数

**CloudWatchダッシュボード**:
```json
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/Glue", "CompactionJobsSucceeded"],
          [".", "CompactionJobsFailed"]
        ],
        "period": 300,
        "stat": "Sum",
        "region": "us-east-1",
        "title": "Compaction Jobs"
      }
    }
  ]
}
```

### 9.3 クエリパフォーマンスの分析

#### Athenaクエリメトリクス

Athenaは、クエリパフォーマンスの詳細なメトリクスを提供します。

**主要メトリクス**:
- `DataScannedInBytes`: スキャンされたデータ量
- `EngineExecutionTimeInMillis`: クエリ実行時間
- `QueryPlanningTimeInMillis`: クエリプランニング時間
- `QueryQueueTimeInMillis`: キュー待ち時間
- `TotalExecutionTimeInMillis`: 総実行時間

**クエリ実行統計の取得**:
```bash
# AWS CLIでクエリ実行統計を取得
aws athena get-query-execution \
  --query-execution-id <query-execution-id> \
  --query 'QueryExecution.Statistics'
```

**パフォーマンス分析のポイント**:
1. **DataScannedInBytes**: パーティションプルーニングの効果を確認
2. **QueryPlanningTimeInMillis**: メタデータサイズの影響を確認
3. **EngineExecutionTimeInMillis**: クエリ最適化の効果を確認

## 10. ベストプラクティスまとめ

### 10.1 設計フェーズ

#### テーブル設計

| 項目 | 推奨事項 |
|------|---------|
| **パーティション** | 低カーディナリティカラム、クエリフィルタ頻度の高いカラム |
| **ファイルサイズ** | 512 MB（中規模テーブル）、100 MB以上（小規模テーブル） |
| **圧縮** | ZSTD（バランス）、Snappy（書き込み重視） |
| **ソート順序** | 範囲クエリが多いカラム |
| **カラム統計** | クエリで使用するカラムのみ |

#### スキーマ設計

```sql
-- 推奨テーブル設計例
CREATE TABLE optimized_table (
  -- 主キー
  id BIGINT,
  
  -- パーティションキー（低カーディナリティ）
  date DATE,
  region STRING,
  
  -- ソートキー（範囲クエリ用）
  timestamp TIMESTAMP,
  
  -- データカラム
  user_id STRING,
  event_type STRING,
  properties MAP<STRING, STRING>
)
USING iceberg
PARTITIONED BY (days(date), region)
TBLPROPERTIES (
  'write.target-file-size-bytes' = '536870912',  -- 512 MB
  'write.parquet.compression-codec' = 'zstd',
  'write.metadata.metrics.column.properties' = 'none'  -- 大きなカラムの統計を無効化
);

-- ソート順序を設定
ALTER TABLE optimized_table WRITE ORDERED BY (timestamp, user_id);
```

### 10.2 運用フェーズ

#### 定期メンテナンス

| タスク | 頻度 | 目的 |
|--------|------|------|
| **コンパクション** | 日次 | 小さなファイルの統合 |
| **スナップショット削除** | 週次 | メタデータサイズ削減 |
| **孤立ファイル削除** | 週次 | ストレージコスト削減 |
| **統計情報更新** | 月次 | クエリ最適化 |
| **パーティション見直し** | 四半期 | ワークロード変化への対応 |

#### 自動メンテナンス設定

**S3 Tables**:
```json
{
  "icebergCompaction": {
    "enabled": true,
    "settings": {
      "targetFileSizeMB": 512,
      "strategy": "auto"
    }
  },
  "icebergSnapshotManagement": {
    "enabled": true,
    "settings": {
      "minimumSnapshots": 3,
      "maximumSnapshotAge": 7
    }
  },
  "icebergUnreferencedFileRemoval": {
    "enabled": true,
    "settings": {
      "unreferencedDays": 3,
      "nonCurrentDays": 10
    }
  }
}
```

**AWS Glue Data Catalog**:
```bash
# すべてのオプティマイザーを有効化
for optimizer_type in compaction retention orphan_file_deletion; do
  aws glue create-table-optimizer \
    --catalog-id 123456789012 \
    --database-name my_database \
    --table-name my_table \
    --table-optimizer-configuration '{
      "enabled": true,
      "roleArn": "arn:aws:iam::123456789012:role/GlueTableOptimizerRole"
    }' \
    --type $optimizer_type
done
```

### 10.3 監視とアラート

#### CloudWatchアラーム設定

```bash
# 503エラーのアラーム
aws cloudwatch put-metric-alarm \
  --alarm-name S3Tables-503Errors \
  --alarm-description "Alert when 503 errors exceed threshold" \
  --metric-name 503Errors \
  --namespace S3Tables \
  --statistic Sum \
  --period 300 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2

# コンパクション失敗のアラーム
aws cloudwatch put-metric-alarm \
  --alarm-name Glue-CompactionFailed \
  --alarm-description "Alert when compaction jobs fail" \
  --metric-name CompactionJobsFailed \
  --namespace AWS/Glue \
  --statistic Sum \
  --period 3600 \
  --threshold 1 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1
```

### 10.4 パフォーマンスチューニングチェックリスト

#### 初期設定

- [ ] パーティション戦略の決定（低カーディナリティカラム）
- [ ] ファイルサイズの設定（512 MB推奨）
- [ ] 圧縮アルゴリズムの選択（ZSTD推奨）
- [ ] ソート順序の設定（範囲クエリ用）
- [ ] カラム統計の最適化（不要なカラムは無効化）

#### 定期レビュー

- [ ] クエリパフォーマンスの分析（Athenaメトリクス）
- [ ] ファイル数とサイズの確認（コンパクション必要性）
- [ ] パーティション分布の確認（データスキュー）
- [ ] スナップショット数の確認（メタデータサイズ）
- [ ] 孤立ファイルの確認（ストレージコスト）

#### トラブルシューティング

- [ ] 503エラーの監視（スケーリング中）
- [ ] クエリプランニング時間の確認（メタデータ肥大化）
- [ ] データスキャン量の確認（パーティションプルーニング）
- [ ] コンパクション失敗の確認（同時実行制御）
- [ ] ストレージコストの確認（未参照ファイル）

## 11. 出典

本ドキュメントは、以下のAWS公式ドキュメントに基づいて作成されました。

### AWS公式ドキュメント

1. **Best practices design patterns: optimizing Amazon S3 performance**
   - URL: https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance.html
   - アクセス日: 2025-11-15
   - 内容: S3パフォーマンス最適化のベストプラクティス

2. **Performance guidelines for Amazon S3**
   - URL: https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance-guidelines.html
   - アクセス日: 2025-11-15
   - 内容: S3パフォーマンスガイドライン

3. **Compaction optimization - AWS Glue**
   - URL: https://docs.aws.amazon.com/glue/latest/dg/compaction-management.html
   - アクセス日: 2025-11-15
   - 内容: AWS Glue Data Catalogのマネージドコンパクション

4. **Optimizing Iceberg tables - AWS Glue**
   - URL: https://docs.aws.amazon.com/glue/latest/dg/table-optimizers.html
   - アクセス日: 2025-11-15
   - 内容: AWS Glueのテーブル最適化オプション

5. **Considerations and limitations for maintenance jobs - Amazon S3 Tables**
   - URL: https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-considerations.html
   - アクセス日: 2025-11-15
   - 内容: S3 Tablesメンテナンスジョブの制限事項

### AWS Prescriptive Guidance

6. **Best practices for optimizing Apache Iceberg workloads**
   - URL: https://docs.aws.amazon.com/prescriptive-guidance/latest/apache-iceberg-on-aws/
   - アクセス日: 2025-11-15（前回取得）
   - 内容: Apache Iceberg on AWSのベストプラクティス

7. **Optimizing read performance**
   - URL: https://docs.aws.amazon.com/prescriptive-guidance/latest/apache-iceberg-on-aws/best-practices-read.html
   - アクセス日: 2025-11-15（前回取得）
   - 内容: 読み取りパフォーマンスの最適化

8. **Optimizing storage**
   - URL: https://docs.aws.amazon.com/prescriptive-guidance/latest/apache-iceberg-on-aws/best-practices-storage.html
   - アクセス日: 2025-11-15（前回取得）
   - 内容: ストレージの最適化

9. **Optimizing write performance**
   - URL: https://docs.aws.amazon.com/prescriptive-guidance/latest/apache-iceberg-on-aws/best-practices-write.html
   - アクセス日: 2025-11-15（前回取得）
   - 内容: 書き込みパフォーマンスの最適化

10. **Maintaining tables by using compaction**
    - URL: https://docs.aws.amazon.com/prescriptive-guidance/latest/apache-iceberg-on-aws/best-practices-compaction.html
    - アクセス日: 2025-11-15（前回取得）
    - 内容: コンパクションによるテーブルメンテナンス

### その他の情報源

11. **AWS公式ブログ記事**
    - 投稿者: Sotaro Hikita (ソリューションアーキテクト 疋田)
    - 投稿日: 2025年7月2日
    - イベント日: 2025年5月14日
    - 内容: AWS Glue Data Catalogのテーブル最適化、Amazon Data Firehose統合

---

**確実性**: 高

本ドキュメントは、AWS公式ドキュメントとAWS Prescriptive Guidanceに基づいて作成されており、情報の正確性は高いです。ただし、S3 Tablesは比較的新しいサービス（2024年12月GA）であるため、機能追加や変更が行われる可能性があります。最新情報は、AWS公式ドキュメントで確認してください。

**最終更新**: 2025-11-15
