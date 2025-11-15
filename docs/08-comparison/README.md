# 比較・関係性

S3 TablesとApache Iceberg、他のストレージソリューションとの関係性と比較に関するドキュメントです。

## このフォルダの内容

| ドキュメント | 説明 |
|------------|------|
| [01-s3tables-iceberg-relationship.md](01-s3tables-iceberg-relationship.md) | S3 TablesとApache Icebergの関係 |
| [02-iceberg-library-s3.md](02-iceberg-library-s3.md) | Icebergライブラリを使用したS3との比較 |

## 対象読者

- アーキテクト
- 技術評価担当者
- データエンジニア
- 意思決定者

## 主要な比較ポイント

### S3 Tables vs 標準S3
- **管理**: マネージドサービス vs セルフマネージド
- **最適化**: 自動最適化 vs 手動最適化
- **メタデータ**: 統合カタログ vs 外部カタログ
- **パフォーマンス**: 最適化済み vs 要チューニング

### S3 Tables vs Iceberg on S3
- **運用負荷**: 低 vs 高
- **コスト**: 従量課金 vs インフラコスト
- **柔軟性**: 標準化 vs カスタマイズ可能
- **統合**: AWS統合 vs オープンソース

## 選択ガイド

### S3 Tablesが適している場合
- マネージドサービスを希望
- 運用負荷を最小化したい
- AWS統合を重視
- 迅速な導入が必要

### Iceberg on S3が適している場合
- 完全なコントロールが必要
- マルチクラウド対応が必要
- カスタマイズが重要
- オープンソースを優先

## 技術的な違い

### アーキテクチャ
- S3 Tables: AWSマネージドレイヤー
- Iceberg on S3: セルフホスト型

### メタデータ管理
- S3 Tables: AWS Glue Data Catalog統合
- Iceberg on S3: Hive Metastore / Glue / Nessie

### 最適化
- S3 Tables: 自動コンパクション、自動クリーンアップ
- Iceberg on S3: 手動またはスケジュール実行

## 移行パス

### Iceberg on S3 → S3 Tables
既存のIcebergテーブルをS3 Tablesに移行する方法

### S3 Tables → Iceberg on S3
S3 TablesからセルフマネージドIcebergへの移行（通常は非推奨）

## 関連ドキュメント

- **入門**: [概要](../01-getting-started/01-overview.md) - S3 Tablesの概要
- **移行**: [データレイク移行](../05-advanced/03-migration-from-datalake.md) - 移行ガイド
- **FAQ**: [FAQ](../07-reference/03-faq.md) - よくある質問
