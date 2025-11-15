# Amazon S3 Tables - 概要

## 前提知識

S3 Tablesを理解する前に、以下のS3基礎知識を確認することを推奨します：

- [S3サービス種類の比較](../00-s3-fundamentals/01-s3-service-types.md) - S3の4つのバケットタイプ
- [ストレージクラス比較](../00-s3-fundamentals/02-storage-classes.md) - S3の8つのストレージクラス

---

## 目次
- [Amazon S3 Tablesとは](#amazon-s3-tablesとは)
- [基本概念](#基本概念)
  - [Table Bucket](#table-bucket)
  - [Namespace](#namespace)
  - [Table](#table)
- [Apache Icebergフォーマット](#apache-icebergフォーマット)
- [アーキテクチャ](#アーキテクチャ)
- [関連サービス](#関連サービス)
- [出典](#出典)

---

## Amazon S3 Tablesとは

Amazon S3 Tablesは、分析ワークロードに最適化されたS3ストレージです。テーブルのクエリパフォーマンスを継続的に向上させ、ストレージコストを削減する機能を備えています。

### 主な特徴

- **表形式データ専用ストレージ**: 日次購入トランザクション、ストリーミングセンサーデータ、広告インプレッションなどの表形式データ（列と行で表現されるデータ）を保存
- **新しいバケットタイプ**: データは新しいバケットタイプである**Table Bucket**に保存され、テーブルはサブリソースとして格納
- **Apache Icebergフォーマット**: テーブルはApache Icebergフォーマットで保存され、標準SQLでクエリ可能
- **高パフォーマンス**: S3汎用バケットのセルフマネージドテーブルと比較して、より高いTPS（Transactions Per Second）とクエリスループットを提供
- **同等の耐久性・可用性**: 他のAmazon S3バケットタイプと同じ耐久性、可用性、スケーラビリティを提供

---

## 基本概念

Amazon S3 Tablesは、3つの主要な概念で構成されています。

### Table Bucket

**Table Bucket**は、テーブルをS3リソースとして作成・保存するためのS3バケットタイプです。

#### 特徴

- **専用設計**: 表形式データとメタデータをオブジェクトとして保存し、分析ワークロードに使用
- **自動メンテナンス**: S3がテーブルバケット内で自動的にメンテナンスを実行し、ストレージコストを削減
- **Apache Iceberg統合**: Apache Icebergをサポートする分析アプリケーションと統合可能
- **AWS分析サービス統合**: AWS Glue Data Catalogを通じてAWS分析サービスと統合
- **オープンソース統合**: Amazon S3 Tables Catalog for Apache Icebergを使用してオープンソースクエリエンジンと統合

#### ARN形式

```text
arn:aws:s3tables:Region:OwnerAccountID:bucket/bucket-name
```

#### プライバシーとアクセス制御

- **常にプライベート**: すべてのTable BucketとTableはプライベートであり、パブリックにできない
- **明示的なアクセス許可**: 明示的にアクセスを許可されたユーザーのみがアクセス可能
- **IAMポリシー**: Table BucketとTableのリソースベースポリシー、ユーザーとロールのアイデンティティベースポリシーを使用してアクセスを許可

#### サービスクォータ

- **デフォルト**: AWSアカウントごと、AWSリージョンごとに最大10個のTable Bucketを作成可能
- **クォータ引き上げ**: [AWS Support](https://console.aws.amazon.com/support/home#/case/create?issueType=service-limit-increase)に連絡してクォータ引き上げをリクエスト可能

#### Table Bucketの種類

| 種類 | 説明 | 管理者 | 用途 |
|------|------|--------|------|
| **Customer-managed table buckets** | 顧客が作成・管理するTable Bucket | 顧客 | 顧客が作成したAmazon S3 Tablesを保存。バケット名を選択し、テーブルとネームスペースを完全に制御。暗号化やメンテナンスオプションをカスタマイズ可能 |
| **AWS managed table buckets** | AWSサービスが自動的に作成するTable Bucket | AWS | S3 Metadataが作成するライブインベントリテーブルやジャーナルテーブルなど、システム生成テーブルを保存。標準的な命名規則、標準ネームスペース、プリセットのメンテナンスと暗号化設定を使用。顧客は読み取り専用アクセスのみ |

### Namespace

**Namespace**は、Table Bucket内でテーブルを論理的なグループに整理するための構成要素です。

#### 特徴

- **リソースではない**: NamespaceはS3 TablesやTable Bucketsとは異なり、リソースではなく、テーブルを整理・管理するための構成要素
- **論理的なグループ化**: 例えば、会社の人事部門に属するすべてのテーブルを共通のNamespace値`hr`でグループ化可能
- **アクセス制御**: Table Bucketリソースポリシーを使用して、特定のNamespaceへのアクセスを制御可能

#### ルール

- **一意性**: 各NamespaceはTable Bucket内で一意である必要がある
- **数量制限**: Table Bucketごとに最大10,000個のNamespaceを作成可能
- **テーブル名の一意性**: 各テーブル名はNamespace内で一意である必要がある
- **単一レベル**: 各テーブルは1レベルのNamespaceのみを持つ。Namespaceはネストできない
- **単一所属**: 各テーブルは単一のNamespaceに属する
- **移動可能**: テーブルをNamespace間で移動可能

#### 用語のマッピング

S3 TablesのNamespaceは、各種AWSサービスやクエリエンジンで異なる用語で呼ばれます。

| サービス/エンジン | 用語 |
|------------------|------|
| AWS Lake Formation | Database |
| AWS Glue Data Catalog | Database |
| Amazon Athena | Database |
| Apache Spark | Namespace |

### Table

**Table**は、基礎となるテーブルデータと関連メタデータで構成される構造化データセットを表します。

#### 特徴

- **サブリソース**: TableはTable Bucket内にサブリソースとして保存
- **Apache Icebergフォーマット**: すべてのTableはApache Icebergテーブルフォーマットで保存
- **自動メンテナンス**: Amazon S3が自動ファイルコンパクションとスナップショット管理を通じてテーブルをメンテナンス
- **AWS分析サービス統合**: Amazon SageMaker Lakehouseとの統合により、Amazon AthenaやAmazon Redshiftなどのサービスが自動的にテーブルデータを検出・アクセス可能

#### Warehouse Location

テーブルを作成すると、Amazon S3は自動的にテーブルの**Warehouse Location**を生成します。これは、テーブルに関連するオブジェクトを保存する一意のS3ロケーションです。

**形式例**:
```text
s3://63a8e430-6e0b-46f5-k833abtwr6s8tmtsycedn8s4yc3xhuse1b--table-s3
```

#### ARN形式

```text
arn:aws:s3tables:region:owner-account-id:bucket/bucket-name/table/table-id
```

#### 識別子

- **一意のARN**: 各テーブルは独自の一意のAmazon Resource Name (ARN)を持つ
- **一意のTable ID**: 各テーブルは一意のTable IDを持つ
- **リソースポリシー**: 各テーブルにはリソースポリシーがアタッチされ、テーブルへのアクセスを管理可能
- **名前変更可能**: テーブルは名前変更可能

#### サービスクォータ

- **デフォルト**: Table Bucketごとに最大10,000個のテーブルを作成可能
- **クォータ引き上げ**: [AWS Support](https://console.aws.amazon.com/support/home#/case/create?issueType=service-limit-increase)に連絡してクォータ引き上げをリクエスト可能

#### テーブルの種類

| 種類 | 説明 | アクセス権限 |
|------|------|-------------|
| **Customer tables** | 顧客が読み書き可能なテーブル。統合クエリエンジンを使用してデータを取得可能。S3 API操作または統合クエリエンジンを使用してデータの挿入、更新、削除が可能 | 読み取り・書き込み |
| **AWS tables** | AWSサービスが顧客に代わって生成する読み取り専用テーブル。Amazon S3が管理し、Amazon S3以外のIAMプリンシパルによる変更は不可。S3 Metadataテーブル（S3汎用バケット内のオブジェクトから取得したメタデータを含む）を含む | 読み取りのみ |

#### 命名規則の重要な注意事項

**すべて小文字を使用する必要があります**:
- テーブル名とテーブル定義（カラム名を含む）にはすべて小文字を使用する必要があります
- 大文字を含む場合、AWS Lake FormationやAWS Glue Data Catalogでサポートされません
- この場合、Table BucketがAWS分析サービスと統合されていても、Amazon Athenaなどのサービスからテーブルが見えません

**エラー例**:
```yaml
GENERIC_INTERNAL_ERROR: Get table request failed:
com.amazonaws.services.glue.model.ValidationException:
Unsupported Federation Resource - Invalid table or column names.
```

---

## Apache Icebergフォーマット

Amazon S3 TablesはApache Icebergフォーマットを使用してテーブルを保存します。

### Icebergの主要機能

#### スキーマ進化（Schema Evolution）
- データの再編成方法を変更可能
- クエリの書き直しやデータ構造の再構築が不要
- 時間の経過とともにデータを進化させることが可能

#### パーティション進化（Partition Evolution）
- パーティション戦略を変更可能
- 既存データへの影響を最小限に抑える

#### トランザクションサポート
- データの一貫性と信頼性を確保
- ACID保証を提供

#### タイムトラベルクエリ
- データの変更を時系列で追跡
- 履歴バージョンにロールバック可能
- 問題の修正や時点クエリの実行が可能

### クエリエンジンのサポート

Apache Icebergをサポートするクエリエンジンで標準SQLを使用してテーブルをクエリ可能:
- Amazon Athena
- Amazon Redshift
- Apache Spark
- その他のIceberg対応クエリエンジン

---

## アーキテクチャ

### 階層構造

```text
AWS Account
└── AWS Region
    └── Table Bucket (最大10個/リージョン)
        ├── Namespace (最大10,000個/バケット)
        │   └── Table (最大10,000個/バケット)
        │       ├── Warehouse Location (S3ロケーション)
        │       ├── Table Data (Apache Icebergフォーマット)
        │       └── Metadata
        └── Maintenance Configuration
```

### データフロー

```text
データソース
    ↓
Table Bucket
    ↓
Namespace (論理的なグループ化)
    ↓
Table (Apache Icebergフォーマット)
    ↓
Warehouse Location (S3ストレージ)
    ↓
クエリエンジン (Athena, Redshift, Spark等)
```

### 統合パターン

#### AWS分析サービス統合

```text
Table Bucket
    ↓
Amazon SageMaker Lakehouse
    ↓
AWS Glue Data Catalog
    ↓
AWS分析サービス
    ├── Amazon Athena
    ├── Amazon Redshift
    ├── Amazon QuickSight
    ├── AWS Glue
    └── Amazon EMR
```

#### オープンソース統合

```text
Table Bucket
    ↓
Amazon S3 Tables Iceberg REST endpoint
    ↓
Amazon S3 Tables Catalog for Apache Iceberg
    ↓
オープンソースクエリエンジン
    ├── Apache Spark
    ├── PyIceberg
    ├── Presto/Trino
    └── その他のIceberg対応エンジン
```

---

## 関連サービス

Amazon S3 Tablesは、以下のAWSサービスと連携して特定の分析アプリケーションをサポートします。

### Amazon Athena
- **概要**: S3内のデータを標準SQLで直接分析できるインタラクティブクエリサービス
- **Apache Sparkサポート**: リソースの計画、設定、管理なしでApache Sparkを使用したデータ分析を実行可能
- **統合**: S3 Tablesのテーブルに対してSQLクエリを実行

### AWS Glue
- **概要**: 複数のソースからデータを検出、準備、移動、統合できるサーバーレスデータ統合サービス
- **用途**: 分析、機械学習（ML）、アプリケーション開発
- **追加機能**: データ操作ツール、ジョブ実行、ビジネスワークフロー実装のための生産性ツール

### Amazon EMR
- **概要**: Apache HadoopやApache Sparkなどのビッグデータフレームワークの実行を簡素化するマネージドクラスタープラットフォーム
- **用途**: 大量のデータの処理と分析
- **統合**: S3 TablesのIcebergテーブルに対してSparkジョブを実行

### Amazon Redshift
- **概要**: クラウド内のペタバイト規模のデータウェアハウスサービス
- **Redshift Serverless**: プロビジョニングされたデータウェアハウスの設定なしでデータにアクセス・分析
- **自動スケーリング**: リソースが自動的にプロビジョニングされ、最も要求の厳しいワークロードに対しても高速なパフォーマンスを提供
- **コスト効率**: データウェアハウスがアイドル状態の場合は課金されず、使用した分のみ支払い
- **統合**: Redshift query editor v2またはBIツールでS3 Tablesのデータをクエリ

### Amazon QuickSight
- **概要**: ビジュアライゼーションの構築、アドホック分析の実行、データからビジネスインサイトを迅速に取得するビジネス分析サービス
- **自動検出**: AWSデータソースをシームレスに検出
- **SPICE**: Super-fast, Parallel, In-Memory, Calculation Engineを使用した高速で応答性の高いクエリパフォーマンス
- **統合**: S3 Tablesのデータを可視化

### AWS Lake Formation
- **概要**: データレイクのセットアップ、セキュリティ保護、管理のプロセスを効率化するマネージドサービス
- **機能**: データソースの検出、カタログ化、クレンジング、変換
- **アクセス制御**: Amazon S3上のデータレイクデータとAWS Glue Data Catalog内のメタデータに対するきめ細かいアクセス制御を管理
- **統合**: S3 Tablesのテーブルに対する権限管理

---

## 出典

本ドキュメントは、以下のAWS公式ドキュメントに基づいて作成されました。

### 主要ドキュメント

1. **Working with Amazon S3 Tables and table buckets**
   - URL: https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables.html
   - アクセス日: 2025-11-15
   - 内容: S3 Tablesの概要、主要機能、関連サービス

2. **Table buckets**
   - URL: https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-buckets.html
   - アクセス日: 2025-11-15
   - 内容: Table Bucketの詳細、種類、ARN形式、アクセス制御

3. **Table namespaces**
   - URL: https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-namespace.html
   - アクセス日: 2025-11-15
   - 内容: Namespaceの概念、ルール、用語マッピング

4. **Tables in S3 table buckets**
   - URL: https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-tables.html
   - アクセス日: 2025-11-15
   - 内容: Tableの詳細、種類、ARN形式、Warehouse Location

5. **Creating an Amazon S3 table**
   - URL: https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-create.html
   - アクセス日: 2025-11-15
   - 内容: テーブル作成方法、命名規則、スキーマ定義

### 情報の信頼性

- **情報源**: AWS公式ドキュメント（一次情報）
- **確実性**: 高
- **最終確認日**: 2025-11-15

---

**注意事項**:
- 本ドキュメントはAWS公式ドキュメントに基づいて作成されていますが、最新情報はAWS公式ドキュメントを参照してください
- サービスの仕様や制限は予告なく変更される場合があります
- 実装前には必ず最新のAWS公式ドキュメントを確認してください
