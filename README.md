# 猫でもわかる Amazon S3 Tables (2025年11月15日版)

Amazon S3 Tablesおよび関連するS3サービス、テーブルフォーマット、外部アクセスに関する包括的な調査ドキュメント

> **Note**  
> 本ドキュメントは、JAWS-UG横浜 #89 AWS re:Invent 2025 "Pre":Cap LT発表用に2025年11月15日時点のAmazon S3 Tablesに関するAWS一次情報を取りまとめたものです。

## 目次

### S3サービス全体
- [S3サービス種類の比較](docs/s3/01-s3-service-types.md)
- [S3ストレージクラス比較](docs/s3/02-storage-classes.md)

### S3 Tables
- [概要](docs/s3-tables/01-overview.md)
- [主要機能](docs/s3-tables/02-features.md)
- [ユースケース](docs/s3-tables/03-usecases.md)
- [料金](docs/s3-tables/04-pricing.md)
- [制限事項（公式）](docs/s3-tables/05-limitations.md)
- [制限事項・注意事項（ユーザー目線）](docs/s3-tables/06-user-considerations.md)
- [バージョンアップ履歴](docs/s3-tables/07-version-history.md)
- [実装例](docs/s3-tables/08-examples.md)

### テーブルフォーマット比較
- [Apache Iceberg on AWS vs S3 Tables](docs/comparison/01-iceberg-vs-s3tables.md)
- [S3 TablesとIcebergテーブルフォーマットの関係](docs/comparison/02-s3tables-iceberg-relationship.md)
- [Apache IcebergライブラリとS3の関係](docs/comparison/03-iceberg-library-s3.md)

### 外部アクセス
- [Glue Data Catalogと外部テーブル](docs/external-access/01-glue-data-catalog.md)
- [Snowflakeからのアクセス](docs/external-access/02-snowflake.md)
- [Databricksからのアクセス](docs/external-access/03-databricks.md)
- [その他のコンピューティングリソース](docs/external-access/04-other-compute.md)

### 将来展望
- [今後の機能拡張予想](docs/future/01-roadmap-prediction.md)

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
