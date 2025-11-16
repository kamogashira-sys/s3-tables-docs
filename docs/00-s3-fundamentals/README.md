# S3基礎知識

Amazon S3の基本的なサービス構成とストレージクラスについて解説します。

## 目次

- [S3サービス種類の比較](01-s3-service-types.md)
- [ストレージクラス比較](02-storage-classes.md)

---

## 概要

S3 Tablesを理解する前に、Amazon S3全体のサービス構成を理解することが重要です。

### なぜS3基礎知識が必要か

S3 Tablesは、Amazon S3の4つのバケットタイプの1つです。S3 Tablesの特徴や利点を正しく理解するには、他のバケットタイプとの違いを知る必要があります。

また、S3 Tablesで使用できるストレージクラスは限定されています。適切なストレージクラスを選択するには、S3の全ストレージクラスを理解することが重要です。

---

## 本セクションで学べること

### S3サービス種類の比較

- **4つのバケットタイプ**: General Purpose、Directory、Table、Vector
- **各タイプの特徴**: 用途、パフォーマンス、制限事項
- **選択ガイド**: ユースケース別の推奨バケットタイプ

### ストレージクラス比較

- **8つのストレージクラス**: Standard、Express One Zone、Intelligent-Tiering、IA、Glacier
- **料金比較**: ストレージ料金、リクエスト料金、取得料金
- **選択ガイド**: アクセスパターン別の推奨ストレージクラス

---

## S3 Tablesとの関係

### Table Bucketsの位置づけ

S3 Tablesは、Table Bucketsという専用のバケットタイプを使用します。Table Bucketsは、Apache Iceberg形式の表形式データに最適化されており、分析ワークロードに特化しています。

### 使用可能なストレージクラス

**⚠️ 重要**: S3 Tablesは`S3 Standard`ストレージクラスのみをサポートします。

- **S3 Standard**: デフォルト、変更不可
  - 頻繁なアクセスに最適化
  - 低レイテンシアクセス
  - 自動メンテナンスに必要

**非サポート**:
- ❌ S3 Standard-IA
- ❌ S3 Glacier Instant Retrieval
- ❌ その他すべてのストレージクラス

**理由**: 分析ワークロードの性能要件と自動メンテナンス機能のため

**詳細**: [ストレージクラス比較](02-storage-classes.md#s3-tablesとの関係)を参照

---

## 次のステップ

S3の基礎知識を理解したら、以下のドキュメントに進んでください：

1. [S3 Tables概要](../01-getting-started/01-overview.md) - S3 Tablesの詳細
2. [主要機能](../01-getting-started/02-features.md) - S3 Tablesの機能
3. [料金](../01-getting-started/03-pricing.md) - S3 Tablesの料金体系

---

**出典**: AWS公式ドキュメント  
**最終更新**: 2025年11月16日
