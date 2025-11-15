# 統合・連携

S3 Tablesと他のAWSサービスとの統合方法に関するドキュメントです。

## このフォルダの内容

| ドキュメント | 説明 |
|------------|------|
| [01-glue-integration.md](01-glue-integration.md) | AWS Glueとの統合（ETL、Data Catalog） |
| [02-firehose-integration.md](02-firehose-integration.md) | Amazon Data Firehoseとの統合 |
| [03-streaming-data-ingestion.md](03-streaming-data-ingestion.md) | ストリーミングデータの取り込み |
| [04-machine-learning-integration.md](04-machine-learning-integration.md) | 機械学習サービスとの統合 |
| [05-sagemaker-unified-studio.md](05-sagemaker-unified-studio.md) | Amazon SageMaker Unified Studioとの統合 |

## 対象読者

- データエンジニア
- MLエンジニア
- アプリケーション開発者
- ソリューションアーキテクト

## 統合パターン

### データ取り込み
- **バッチ処理**: AWS Glue ETL
- **ストリーミング**: Amazon Data Firehose、Kinesis Data Streams
- **リアルタイム**: Kafka、MSK

### データ処理
- **クエリエンジン**: Amazon Athena、Amazon Redshift
- **ETL**: AWS Glue、EMR
- **分析**: QuickSight

### 機械学習
- **特徴量ストア**: SageMaker Feature Store
- **学習データ**: SageMaker Training
- **推論**: Athena ML、SageMaker

## 統合アーキテクチャ例

### リアルタイムデータパイプライン
```
データソース → Kinesis → Firehose → S3 Tables → Athena
```

### バッチETLパイプライン
```
データソース → Glue ETL → S3 Tables → Redshift Spectrum
```

### MLパイプライン
```
S3 Tables → SageMaker Feature Store → SageMaker Training → モデル
```

## 関連ドキュメント

- **入門**: [機能](../01-getting-started/02-features.md) - 統合機能の概要
- **ユースケース**: [../06-use-cases/](../06-use-cases/) - 実践的な統合例
- **高度な活用**: [../05-advanced/](../05-advanced/) - マルチリージョン統合
