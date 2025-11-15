# Amazon S3 Tables - 制限事項


## 目次

- [概要](#概要)
- [セキュリティとアクセス制御の制限事項](#セキュリティとアクセス制御の制限事項)
  - [サポートされていない機能](#サポートされていない機能)
    - [1. Public Access（パブリックアクセス）](#1-public-accessパブリックアクセス)
    - [2. Presigned URLs（署名付きURL）](#2-presigned-urls署名付きurl)
    - [3. HTTPリクエスト](#3-httpリクエスト)
    - [4. AWS Signature Version](#4-aws-signature-version)
    - [5. IPv6サポート](#5-ipv6サポート)
    - [6. ポリシーサイズ制限](#6-ポリシーサイズ制限)
  - [タグサポート（2025年11月6日追加）](#タグサポート2025年11月6日追加)
    - [サポート内容](#サポート内容)
    - [タグベースの条件キー](#タグベースの条件キー)
    - [注意事項](#注意事項)
- [テーブル構造の制限事項](#テーブル構造の制限事項)
  - [カラム名の制限](#カラム名の制限)
    - [混合ケース（Mixed Case）の非サポート](#混合ケースmixed-caseの非サポート)
    - [その他のカラム名制限](#その他のカラム名制限)
- [データ移行の制限事項](#データ移行の制限事項)
  - [In-place Migration（インプレース移行）の非サポート](#in-place-migrationインプレース移行の非サポート)
    - [制限内容](#制限内容)
    - [In-place Migrationとは](#in-place-migrationとは)
    - [S3 Tablesで使用できない理由](#s3-tablesで使用できない理由)
    - [推奨される移行方法](#推奨される移行方法)
- [Amazon Data Firehoseとの統合制限事項](#amazon-data-firehoseとの統合制限事項)
  - [Icebergライブラリバージョン](#icebergライブラリバージョン)
  - [スキーマの制限](#スキーマの制限)
    - [ネストレベル](#ネストレベル)
    - [データ形式](#データ形式)
  - [同時書き込みの制限](#同時書き込みの制限)
  - [スループット制限](#スループット制限)
    - [リージョン別の制限](#リージョン別の制限)
    - [スループット向上の方法](#スループット向上の方法)
  - [暗号化設定の制限](#暗号化設定の制限)
    - [重要な制限](#重要な制限)
  - [その他の制限](#その他の制限)
- [サービスクォータ（制限値）](#サービスクォータ制限値)
  - [クォータ引き上げ](#クォータ引き上げ)
- [メンテナンスジョブの制限事項](#メンテナンスジョブの制限事項)
  - [コンパクション（Compaction）](#コンパクションcompaction)
    - [サポートされるファイル形式](#サポートされるファイル形式)
    - [デフォルト動作](#デフォルト動作)
    - [非サポート項目](#非サポート項目)
    - [同時実行制御](#同時実行制御)
    - [無効化](#無効化)
    - [設定パラメータ](#設定パラメータ)
    - [関連するIcebergメンテナンスルーチン](#関連するicebergメンテナンスルーチン)
  - [スナップショット管理（Snapshot Management）](#スナップショット管理snapshot-management)
    - [保持条件](#保持条件)
    - [削除動作](#削除動作)
    - [非サポート項目](#非サポート項目)
    - [自動無効化の条件](#自動無効化の条件)
    - [設定パラメータ](#設定パラメータ)
    - [関連するIcebergメンテナンスルーチン](#関連するicebergメンテナンスルーチン)
  - [未参照ファイル削除（Unreferenced File Removal）](#未参照ファイル削除unreferenced-file-removal)
    - [削除対象](#削除対象)
    - [設定パラメータ](#設定パラメータ)
    - [関連するIcebergメンテナンスルーチン](#関連するicebergメンテナンスルーチン)
  - [その他のメンテナンス設定](#その他のメンテナンス設定)
    - [Parquet Row Group サイズ](#parquet-row-group-サイズ)
- [リージョンとエンドポイント](#リージョンとエンドポイント)
  - [サポートリージョン](#サポートリージョン)
  - [エンドポイント形式](#エンドポイント形式)
- [出典](#出典)
  - [AWS公式ドキュメント](#aws公式ドキュメント)
  - [関連API](#関連api)
- [更新履歴](#更新履歴)

## 概要

このドキュメントでは、Amazon S3 Tablesの制限事項と考慮事項について説明します。

## セキュリティとアクセス制御の制限事項

### サポートされていない機能

#### 1. Public Access（パブリックアクセス）
- **制限**: パブリックアクセスポリシーは非サポート
- **詳細**: ユーザーはバケットポリシーやテーブルポリシーを変更してパブリックアクセスを許可することはできません
- **理由**: セキュリティベストプラクティスに基づく設計

#### 2. Presigned URLs（署名付きURL）
- **制限**: テーブルに関連付けられたオブジェクトへのアクセスに署名付きURLは使用不可
- **影響**: 一時的なアクセス権限の付与には別の方法（IAMロール等）を使用する必要があります

#### 3. HTTPリクエスト
- **制限**: HTTPリクエストは非サポート
- **動作**: HTTPリクエストは自動的にHTTPSへリダイレクトされます
- **要件**: すべてのリクエストはHTTPSを使用する必要があります
- **理由**: 転送時のデータ暗号化を保証

#### 4. AWS Signature Version
- **要件**: REST APIを使用してアクセスポイントにリクエストを行う際は、AWS Signature Version 4を使用する必要があります

#### 5. IPv6サポート
- **制限**: IPv6は限定的にサポート
- **サポート範囲**: テーブルストレージエンドポイント経由のオブジェクトレベルアクションのみ
- **非サポート**: テーブルレベルおよびバケットレベルのアクション

#### 6. ポリシーサイズ制限
- **制限**: テーブルバケットおよびテーブルアクセスポリシーのサイズは20KBまで
- **影響**: 複雑なポリシーを作成する際は、サイズ制限に注意が必要

### タグサポート（2025年11月6日追加）

**重要**: 以前はタグがサポートされていませんでしたが、2025年11月6日のアップデートでタグサポートが追加されました。

#### サポート内容
- **Table Buckets**: タグサポートあり
- **Tables**: タグサポートあり
- **用途**:
  - コスト配分（Cost Allocation）
  - 属性ベースアクセス制御（ABAC: Attribute-Based Access Control）

#### タグベースの条件キー
- `aws:ResourceTag/key-name`: リソースに付与されたタグに基づくアクセス制御
- `aws:RequestTag/key-name`: リクエストに含まれるタグに基づくアクセス制御
- `aws:TagKeys`: 許可されるタグキーの制御
- `s3tables:TableBucketTag/tag-key`: テーブルバケットのタグに基づくアクセス制御

#### 注意事項
- タグの使用に追加料金はかかりません（標準のS3 APIリクエスト料金のみ）
- Lake Formationと統合する場合、IAMロールに同じプリンシパルタグを付与する必要があります

## テーブル構造の制限事項

### カラム名の制限

#### 混合ケース（Mixed Case）の非サポート
- **制限**: カラム名に大文字と小文字の混在は非サポート
- **推奨**: すべて小文字のカラム名を使用してください
- **理由**: Apache Icebergのスキーマ管理との互換性

**例**:
```sql
-- ❌ 非推奨（混合ケース）
CREATE TABLE my_table (
  UserId INT,
  UserName STRING,
  CreatedAt TIMESTAMP
);

-- ✅ 推奨（小文字）
CREATE TABLE my_table (
  user_id INT,
  user_name STRING,
  created_at TIMESTAMP
);
```

#### その他のカラム名制限
- 予約語の使用は避けてください
- 特殊文字の使用には注意が必要です
- スペースを含むカラム名は推奨されません

## データ移行の制限事項

### In-place Migration（インプレース移行）の非サポート

#### 制限内容
- **制限**: S3 Tablesへの移行では、In-place Migration（インプレース移行）は使用できません
- **対応方法**: Full Data Migration（完全データ移行）のみサポート

#### In-place Migrationとは
- 既存のデータファイルを書き直さずに、メタデータのみを変換してIcebergテーブルに移行する方法
- 以下の2つのアプローチがあります：
  1. **In-place snapshot**: メタデータのみ作成（データ書き直しなし）
  2. **In-place migrate**: メタデータ作成 + 段階的なデータ書き直し

#### S3 Tablesで使用できない理由
- S3 Tablesは最適化されたストレージアーキテクチャを使用
- 既存のS3バケットのデータを直接参照することはできません
- テーブルバケット内にデータを配置する必要があります

#### 推奨される移行方法

**Full Data Migration（完全データ移行）**:
- データを読み取り、S3 Tablesに書き込む
- 以下の方法が利用可能：
  1. **CTAS（CREATE TABLE AS SELECT）**: Athena、Spark
  2. **CREATE TABLE + INSERT**: Athena、Spark
  3. **Sparkプログラマティック**: DataFrame API
  4. **AWS Glue ETLジョブ**: マネージドETL

**利点**:
- データレイアウトの最適化
- スキーマの変更
- パーティション戦略の変更
- ソート順の最適化
- 小ファイル問題の解決

**詳細**: [データレイク移行](../05-advanced/03-migration-from-datalake.md)を参照してください。

## Amazon Data Firehoseとの統合制限事項

### Icebergライブラリバージョン
- **制限**: Firehoseは固定のIcebergライブラリバージョン（1.5.2）を使用
- **影響**: 最新のIceberg機能は利用できない可能性があります

### スキーマの制限

#### ネストレベル
- **制限**: 最大16レベルまでのネスト構造をサポート
- **影響**: 深くネストされたJSONデータは変換が必要な場合があります

#### データ形式
- **要件**: 1レコード = 1つのJSONオブジェクト
- **制限**: 複数のJSONオブジェクトを含む配列は非サポート

**例**:
```json
// ✅ サポート（1レコード = 1オブジェクト）
{"user_id": 123, "name": "Alice", "timestamp": "2024-01-15T10:00:00Z"}
{"user_id": 456, "name": "Bob", "timestamp": "2024-01-15T10:01:00Z"}

// ❌ 非サポート（配列形式）
[
  {"user_id": 123, "name": "Alice"},
  {"user_id": 456, "name": "Bob"}
]
```

### 同時書き込みの制限
- **非推奨**: 複数のFirehose Delivery Streamから同じテーブルへの同時書き込み
- **理由**: Apache Icebergの楽観的同時実行制御により、競合が発生する可能性があります
- **推奨**: 単一のDelivery Streamを使用するか、マルチテーブルルーティングを活用してください

### スループット制限

#### リージョン別の制限
| リージョン | スループット制限 |
|-----------|----------------|
| US East (N. Virginia) | 5 MiB/秒 |
| US West (Oregon) | 5 MiB/秒 |
| Europe (Ireland) | 5 MiB/秒 |
| その他のリージョン | 1 MiB/秒 |

#### スループット向上の方法
- **AppendOnlyフラグ**: Insert専用の場合に設定することで自動スケールが有効化されます
- **注意**: AppendOnlyフラグは`CreateDeliveryStream` APIでのみ設定可能（コンソールでは設定不可）

### 暗号化設定の制限

#### 重要な制限
- **制限**: Firehose側のKMS設定は使用されません
- **要件**: S3 Tables側でKMS暗号化を設定する必要があります

**設定例**:
```bash
# ❌ Firehose側のKMS設定は無視される
aws firehose create-delivery-stream \
  --delivery-stream-name my-stream \
  --delivery-stream-encryption-configuration-input \
    KeyType=CUSTOMER_MANAGED_CMK,KeyARN=arn:aws:kms:...

# ✅ S3 Tables側でKMS設定が必要
aws s3tables create-table-bucket \
  --name my-table-bucket \
  --encryption-configuration \
    KmsKeyArn=arn:aws:kms:...
```

### その他の制限
- **バッファサイズ**: 1-128 MiB（デフォルト: 128 MiB）
- **バッファインターバル**: 0-900秒（デフォルト: 900秒）
- **リトライ期間**: 0-7200秒（デフォルト: 300秒）

**詳細**: [Firehose統合](../04-integrations/02-firehose-integration.md)を参照してください。

## サービスクォータ（制限値）

| リソース | デフォルト値 | 調整可能 | 説明 |
|---------|------------|---------|------|
| Table Buckets | 10 | ✓ | リージョンごとのアカウントあたりのテーブルバケット数 |
| Namespaces | 10,000 | ✓ | テーブルバケットあたりのネームスペース数 |
| Tables | 10,000 | ✓ | テーブルバケットあたりのテーブル数 |

### クォータ引き上げ
- すべてのクォータは[AWSサポート](https://console.aws.amazon.com/support/home#/case/create?issueType=service-limit-increase)に連絡することで引き上げ可能です

## メンテナンスジョブの制限事項

### コンパクション（Compaction）

#### サポートされるファイル形式
- Apache Parquet
- Apache Avro
- Apache ORC

#### デフォルト動作
- コンパクション後のファイルはデフォルトでApache Parquet形式で書き込まれます
- AvroまたはORC形式で出力する場合は、`write.format.default`テーブルプロパティを`avro`または`orc`に設定します

#### 非サポート項目
- **データ型**: `Fixed`データ型は非サポート
- **圧縮タイプ**: `brotli`、`lz4`圧縮は非サポート

#### 同時実行制御
- Apache Icebergは楽観的同時実行制御モデルを使用
- ユーザートランザクションとコンパクショントランザクションが競合する可能性があります
- 競合が発生した場合、コンパクションジョブは失敗時に再試行します
- **推奨**: パイプラインにも再試行ロジックを実装してください

#### 無効化
- コンパクションは自動スケジュールで実行されます
- 料金を回避したい場合は、`PutTableMaintenanceConfiguration` APIを使用して手動で無効化できます

#### 設定パラメータ
| パラメータ | デフォルト値 | 最小値 | 設定レベル |
|-----------|------------|--------|-----------|
| targetFileSizeMB | 512MB | 64MB | テーブル |

#### 関連するIcebergメンテナンスルーチン
- `rewriteDataFiles`

### スナップショット管理（Snapshot Management）

#### 保持条件
- スナップショットは以下の**両方の条件**を満たす場合のみ保持されます：
  1. 最小スナップショット数
  2. 指定された保持期間

#### 削除動作
- 期限切れスナップショットのメタデータをApache Icebergから削除
- タイムトラベルクエリが期限切れスナップショットに対して実行できなくなります
- オプションで関連するデータファイルも削除可能

#### 非サポート項目
- `metadata.json`ファイルで設定した保持値は非サポート
- `ALTER TABLE SET TBLPROPERTIES` SQLコマンドで設定した保持値は非サポート
- ブランチベースの保持ポリシーは非サポート
- タグベースの保持ポリシーは非サポート

#### 自動無効化の条件
以下の場合、スナップショット管理は自動的に無効化されます：
- ブランチまたはタグベースの保持ポリシーを設定した場合
- `metadata.json`ファイルで`PutTableMaintenanceConfiguration` APIより長い保持ポリシーを設定した場合

**注意**: これらの場合、S3はスナップショットを期限切れにしたり削除したりしません。ストレージ料金を回避するには、手動でスナップショットを削除するか、Icebergテーブルからプロパティを削除する必要があります。

#### 設定パラメータ
| パラメータ | デフォルト値 | 最小値 | 設定レベル |
|-----------|------------|--------|-----------|
| minimumSnapshots | 1 | 1 | テーブル |
| maximumSnapshotAge | 120時間 | 1時間 | テーブル |

#### 関連するIcebergメンテナンスルーチン
- `ExpireSnapshots` (`retainLast`)
- `ExpireSnapshots` (`expireOlderThan`)

### 未参照ファイル削除（Unreferenced File Removal）

#### 削除対象
- Icebergメタデータから参照されなくなったデータファイルとメタデータファイル
- 作成時刻が保持期間より前のファイル

#### 設定パラメータ
| パラメータ | デフォルト値 | 最小値 | 設定レベル |
|-----------|------------|--------|-----------|
| unreferencedDays | 3日 | 1日 | テーブルバケット |
| nonCurrentDays | 10日 | 1日 | テーブルバケット |

#### 関連するIcebergメンテナンスルーチン
- `deleteOrphanFiles`

### その他のメンテナンス設定

#### Parquet Row Group サイズ
- S3 Tablesは128MBのParquet row-group-default サイズを適用します

## リージョンとエンドポイント

### サポートリージョン
- 利用可能なリージョンの一覧は[Amazon S3 endpoints](https://docs.aws.amazon.com/general/latest/gr/s3.html#s3_region)を参照してください

### エンドポイント形式
- **Dual-stackエンドポイント**: `s3tables.aws-region.api.aws`
- **IPv6サポート**: Dual-stackエンドポイント経由で利用可能（制限あり）
- **AWS PrivateLink**: サポートあり

## 出典

### AWS公式ドキュメント
1. [Security considerations and limitations for S3 Tables](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-restrictions.html)
   - アクセス日: 2025-11-15
   - 内容: セキュリティとアクセス制御の制限事項

2. [Considerations and limitations for maintenance jobs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-considerations.html)
   - アクセス日: 2025-11-15
   - 内容: メンテナンスジョブの制限事項と考慮事項

3. [S3 Tables AWS Regions, endpoints, and service quotas](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-regions-quotas.html)
   - アクセス日: 2025-11-15
   - 内容: リージョン、エンドポイント、サービスクォータ

4. [Using tags with S3 table buckets](https://docs.aws.amazon.com/AmazonS3/latest/userguide/table-bucket-tagging.html)
   - アクセス日: 2025-11-15
   - 内容: テーブルバケットのタグサポート

5. [Using tags with S3 tables](https://docs.aws.amazon.com/AmazonS3/latest/userguide/table-tagging.html)
   - アクセス日: 2025-11-15
   - 内容: テーブルのタグサポート

### 関連API
- [PutTableMaintenanceConfiguration](https://docs.aws.amazon.com/AmazonS3/latest/API/API_s3tables_PutTableMaintenanceConfiguration.html)
- [PutTableBucketMaintenanceConfiguration](https://docs.aws.amazon.com/AmazonS3/latest/API/API_s3tables_PutTableBucketMaintenanceConfiguration.html)

## 更新履歴
- 2025-11-15: 初版作成
  - セキュリティとアクセス制御の制限事項
  - サービスクォータ
  - メンテナンスジョブの制限事項
  - タグサポート情報（2025年11月6日追加機能）
- 2025-11-15: フェーズ0整合性修正
  - テーブル構造の制限事項（カラム名制限）を追加
  - データ移行の制限事項（In-place Migration非サポート）を追加
  - Amazon Data Firehoseとの統合制限事項を追加
