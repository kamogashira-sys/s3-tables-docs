# 猫でもわかる Amazon S3 Tables (2025年11月15日版)

Amazon S3 Tablesおよび関連するS3サービス、テーブルフォーマット、外部アクセスに関する包括的な調査ドキュメント

> **Note**  
> 本ドキュメントは、JAWS-UG横浜 #89 AWS re:Invent 2025 "Pre":Cap LT発表用に2025年11月15日時点のAmazon S3 Tablesに関するAWS一次情報を取りまとめたものです。

## 目次

### 入門・基礎
- [概要](docs/01-getting-started/01-overview.md)
- [主要機能](docs/01-getting-started/02-features.md)
- [料金](docs/01-getting-started/03-pricing.md)
- [制限事項](docs/01-getting-started/04-limitations.md)
- [考慮事項](docs/01-getting-started/05-user-considerations.md)

### 運用・管理
- [メンテナンス](docs/02-operations/01-maintenance.md)
- [パフォーマンス最適化](docs/02-operations/02-performance-optimization.md)
- [トラブルシューティング](docs/02-operations/03-troubleshooting-guide.md)
- [監視・ログ記録](docs/02-operations/04-monitoring-logging.md)
- [コンソールデータプレビュー](docs/02-operations/05-console-data-preview.md)

### セキュリティ・コンプライアンス
- [セキュリティベストプラクティス](docs/03-security/01-security-best-practices.md)
- [データガバナンス](docs/03-security/02-data-governance.md)
- [コンプライアンス](docs/03-security/03-compliance.md)
- [SSE-KMS暗号化](docs/03-security/04-sse-kms-encryption.md)

### 統合・連携
- [AWS Glue統合](docs/04-integrations/01-glue-integration.md)
- [Data Firehose統合](docs/04-integrations/02-firehose-integration.md)
- [ストリーミングデータ取り込み](docs/04-integrations/03-streaming-data-ingestion.md)
- [機械学習統合](docs/04-integrations/04-machine-learning-integration.md)
- [SageMaker Unified Studio統合](docs/04-integrations/05-sagemaker-unified-studio.md)

### 高度な活用
- [マルチリージョン戦略](docs/05-advanced/01-multi-region-strategy.md)
- [災害復旧](docs/05-advanced/02-disaster-recovery.md)
- [データレイク移行](docs/05-advanced/03-migration-from-datalake.md)
- [コンパクション戦略](docs/05-advanced/04-compaction-strategies.md)
- [スナップショット管理](docs/05-advanced/05-snapshot-management.md)

### ユースケース・実践
- [ユースケース集](docs/06-use-cases/01-use-cases.md)

### リファレンス
- [用語集](docs/07-reference/01-glossary.md)
- [技術リファレンス](docs/07-reference/02-reference.md)
- [FAQ](docs/07-reference/03-faq.md)

### 比較・関係性
- [S3 Tables と Iceberg の関係](docs/08-comparison/01-s3tables-iceberg-relationship.md)
- [Iceberg on S3 との比較](docs/08-comparison/02-iceberg-library-s3.md)

## 調査状況

調査中...

## GA後の更新情報

| 日付 | 内容 |
| --- | --- |
| 2024年12月3日 | S3 Tables GA（一般提供開始） |
| 2025年3月13日 | S3コンソールでのテーブル作成・クエリサポート追加 |
| 2025年3月13日 | SageMaker Lakehouse統合GA |
| 2025年4月16日 | SSE-KMS（カスタマー管理キー）サポート追加 |
| 2025年6月18日 | コンパクション戦略（Sort、Z-order）追加 |
| 2025年6月30日 | CloudTrailイベント（メンテナンス操作）サポート追加 |
| 2025年11月6日 | タグサポート（ABAC、コスト配分）追加 |

## 主要な調査テーマ

### テーブルフォーマットの選択
- Apache Iceberg on AWS
- S3 Tables
- S3 TablesとIcebergフォーマットの関係
- Apache Icebergライブラリの役割
- 選択基準とユースケース

### マルチクラウド/ハイブリッド環境
- Snowflakeからの外部テーブルアクセス
- Databricksからの外部テーブルアクセス
- AWS Glue Data Catalogの活用

### 将来展望
- S3 Tablesのロードマップ予想
- 競合サービスとの機能ギャップ分析
- 業界トレンドとの関連

## 参考資料

- [AWS公式ドキュメント](https://docs.aws.amazon.com/)
- [AWS S3 Tables](https://aws.amazon.com/s3/features/tables/)
- [Apache Iceberg](https://iceberg.apache.org/)
- [AWS Glue Data Catalog](https://docs.aws.amazon.com/glue/latest/dg/catalog-and-crawler.html)

---

## 🤝 貢献者

- **調査担当**: Amazon Q Developer CLI
- **レビュー**: katoh

---

## 📝 ライセンス

このプロジェクトは [MIT License](LICENSE) の下で公開されています。

---

## ⚠️ 免責事項

> **重要な免責事項**
> 
> このプロジェクトは **非公式** のドキュメントです。Amazon Web Services, Inc.またはその関連会社によって作成、承認、または保証されたものではありません。
> 
> - **商標について**: "Amazon S3 Tables"、"AWS"、"Amazon Web Services" は Amazon.com, Inc. またはその関連会社の商標です
> - **情報の正確性**: 本ドキュメントの情報は調査時点のものであり、正確性を保証するものではありません
> - **責任の制限**: 本ドキュメントの使用により生じた損害について、作成者は一切の責任を負いません
> - **公式情報**: 最新かつ正確な情報は [Amazon S3 Tables 公式サイト](https://aws.amazon.com/s3/features/tables/) をご確認ください

---

## 管理方針

- 調査結果はGitHubで管理
- 定期的な更新（AWS re:Invent 2025後も継続）
- 構造化されたドキュメント形式での記録
- 変更履歴の適切な管理

---

## 🔗 関連リンク

- [Amazon S3 Tables 公式サイト](https://aws.amazon.com/s3/features/tables/)
- [AWS S3 Tables ドキュメント](https://docs.aws.amazon.com/s3-tables/)
- [Apache Iceberg 公式サイト](https://iceberg.apache.org/)
- [AWS Glue Data Catalog](https://docs.aws.amazon.com/glue/latest/dg/catalog-and-crawler.html)
- [JAWS-UG横浜](https://jaws-ug-yokohama.connpass.com/)

---

*このプロジェクトは継続的な調査と分析を通じて、Amazon S3 Tablesの効果的な活用を支援することを目指しています。*
