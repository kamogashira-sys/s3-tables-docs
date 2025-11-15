# S3 Tables ドキュメント

Amazon S3 Tablesの包括的なドキュメントセットへようこそ。このドキュメントは、S3 Tablesの基本から高度な活用方法まで、体系的に学習できるように構成されています。

> **Note**  
> 本ドキュメントは、JAWS-UG横浜 #89 AWS re:Invent 2025 "Pre":Cap LT発表用に2025年11月15日時点のAmazon S3 Tablesに関するAWS一次情報を取りまとめたものです。

## 📚 ドキュメント構成

### [00. S3基礎知識](00-s3-fundamentals/)
S3 Tablesを理解するための前提知識です。

- [S3サービス種類の比較](00-s3-fundamentals/01-s3-service-types.md) - 4つのバケットタイプ
- [ストレージクラス比較](00-s3-fundamentals/02-storage-classes.md) - 8つのストレージクラス

**対象読者**: S3初学者、S3 Tables評価担当者

---

### [01. 入門・基礎](01-getting-started/)
S3 Tablesを初めて使う方向けの入門ドキュメントです。

- [概要](01-getting-started/01-overview.md) - S3 Tablesとは何か
- [機能](01-getting-started/02-features.md) - 主要機能の紹介
- [料金](01-getting-started/03-pricing.md) - 料金体系の理解
- [制限事項](01-getting-started/04-limitations.md) - 技術的制約
- [考慮事項](01-getting-started/05-user-considerations.md) - 利用時の注意点

**対象読者**: 初学者、評価担当者、意思決定者

---

### [02. 運用・管理](02-operations/)
日常的な運用、パフォーマンス最適化、トラブルシューティングに関するドキュメントです。

- [メンテナンス](02-operations/01-maintenance.md) - 定期メンテナンス作業
- [パフォーマンス最適化](02-operations/02-performance-optimization.md) - チューニング手法
- [トラブルシューティング](02-operations/03-troubleshooting-guide.md) - 問題解決ガイド
- [監視・ログ記録](02-operations/04-monitoring-logging.md) - 監視と監査
- [コンソールデータプレビュー](02-operations/05-console-data-preview.md) - S3コンソールでのデータ確認 🆕

**対象読者**: システム管理者、DevOpsエンジニア、SREエンジニア

---

### [03. セキュリティ・コンプライアンス](03-security/)
セキュリティベストプラクティスとコンプライアンス要件に関するドキュメントです。

- [セキュリティベストプラクティス](03-security/01-security-best-practices.md) - 推奨設定
- [データガバナンス](03-security/02-data-governance.md) - アクセス制御
- [コンプライアンス](03-security/03-compliance.md) - 規制対応
- [SSE-KMS暗号化](03-security/04-sse-kms-encryption.md) - カスタマー管理キーによる暗号化 🆕

**対象読者**: セキュリティエンジニア、コンプライアンス担当者

---

### [04. 統合・連携](04-integrations/)
他のAWSサービスとの統合方法に関するドキュメントです。

- [AWS Glue統合](04-integrations/01-glue-integration.md) - ETLとData Catalog
- [Data Firehose統合](04-integrations/02-firehose-integration.md) - ストリーミング取り込み
- [ストリーミングデータ取り込み](04-integrations/03-streaming-data-ingestion.md) - リアルタイム処理
- [機械学習統合](04-integrations/04-machine-learning-integration.md) - ML/AIサービス連携
- [SageMaker Unified Studio統合](04-integrations/05-sagemaker-unified-studio.md) - Lakehouseアーキテクチャ統合 🆕

**対象読者**: データエンジニア、MLエンジニア、アプリケーション開発者

---

### [05. 高度な活用](05-advanced/)
エンタープライズ向けの高度な活用方法に関するドキュメントです。

- [マルチリージョン戦略](05-advanced/01-multi-region-strategy.md) - グローバル展開
- [災害復旧](05-advanced/02-disaster-recovery.md) - DR計画
- [データレイク移行](05-advanced/03-migration-from-datalake.md) - 既存環境からの移行
- [コンパクション戦略](05-advanced/04-compaction-strategies.md) - クエリパフォーマンス最適化 🆕
- [スナップショット管理](05-advanced/05-snapshot-management.md) - ストレージコスト最適化 🆕

**対象読者**: ソリューションアーキテクト、エンタープライズアーキテクト

---

### [06. ユースケース・実践](06-use-cases/)
実践的なユースケースと実装例を紹介するドキュメントです。

- [ユースケース集](06-use-cases/01-use-cases.md) - 業界別・用途別の活用例

**対象読者**: すべてのユーザー、ビジネスアナリスト

---

### [07. リファレンス](07-reference/)
用語集、技術リファレンス、FAQなどの参照情報です。

- [用語集](07-reference/01-glossary.md) - 用語の定義
- [技術リファレンス](07-reference/02-reference.md) - API仕様と制限値
- [FAQ](07-reference/03-faq.md) - よくある質問

**対象読者**: すべてのユーザー

---

### [08. 比較・関係性](08-comparison/)
S3 Tablesと他のソリューションとの比較に関するドキュメントです。

- [S3 Tables と Iceberg の関係](08-comparison/01-s3tables-iceberg-relationship.md) - 技術的関係性
- [Iceberg on S3 との比較](08-comparison/02-iceberg-library-s3.md) - 選択ガイド

**対象読者**: アーキテクト、技術評価担当者

---

### [09. メタ情報](09-meta/)
ドキュメントの変更履歴とメタ情報です。

- [変更履歴](09-meta/changelog.md) - ドキュメントの更新履歴

**対象読者**: ドキュメント管理者、コントリビューター

---

## 🚀 クイックスタート

### 初めての方
1. [概要](01-getting-started/01-overview.md)でS3 Tablesの全体像を理解
2. [機能](01-getting-started/02-features.md)で提供される機能を確認
3. [ユースケース](06-use-cases/01-use-cases.md)で活用イメージを把握

### 実装を検討中の方
1. [料金](01-getting-started/03-pricing.md)でコストを見積もり
2. [制限事項](01-getting-started/04-limitations.md)で技術的制約を確認
3. [統合](04-integrations/)で必要な統合方法を確認

### 運用担当者の方
1. [メンテナンス](02-operations/01-maintenance.md)で運用タスクを確認
2. [監視・ログ記録](02-operations/04-monitoring-logging.md)で監視設定を実施
3. [トラブルシューティング](02-operations/03-troubleshooting-guide.md)で問題解決方法を把握

---

## 📖 学習パス

### 初級（1-2時間）
1. 入門・基礎セクション全体
2. ユースケース集
3. FAQ

### 中級（3-4時間）
1. 運用・管理セクション
2. セキュリティセクション
3. 統合セクション

### 上級（5-8時間）
1. 高度な活用セクション
2. 比較・関係性セクション
3. 技術リファレンス

---

## 🔗 外部リソース

- [AWS公式ドキュメント](https://docs.aws.amazon.com/s3-tables/)
- [Apache Iceberg公式サイト](https://iceberg.apache.org/)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)

---

## 📝 ドキュメントへの貢献

ドキュメントの改善提案や誤字脱字の報告を歓迎します。変更を行った場合は、[変更履歴](09-meta/changelog.md)を更新してください。

---

## 📄 ライセンス

このドキュメントは、AWSのドキュメントガイドラインに従って作成されています。

---

**最終更新**: 2025-11-15
