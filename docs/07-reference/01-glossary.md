# Amazon S3 Tables - 用語集


## 目次

- [概要](#概要)
- [A](#a)
  - [ACID](#acid)
  - [Apache Iceberg](#apache-iceberg)
  - [AppendOnly](#appendonly)
- [B](#b)
  - [Block Public Access](#block-public-access)
  - [Binpack](#binpack)
- [C](#c)
  - [Catalog](#catalog)
  - [CDC (Change Data Capture)](#cdc-change-data-capture)
  - [Compaction](#compaction)
  - [CTAS (CREATE TABLE AS SELECT)](#ctas-create-table-as-select)
- [D](#d)
  - [Data Lake](#data-lake)
  - [Data Lineage](#data-lineage)
  - [DR (Disaster Recovery)](#dr-disaster-recovery)
- [E](#e)
  - [ETL (Extract, Transform, Load)](#etl-extract-transform-load)
  - [Exactly-once Delivery](#exactly-once-delivery)
- [F](#f)
  - [Firehose](#firehose)
  - [Full Data Migration](#full-data-migration)
- [G](#g)
  - [GDPR (General Data Protection Regulation)](#gdpr-general-data-protection-regulation)
- [H](#h)
  - [HIPAA (Health Insurance Portability and Accountability Act)](#hipaa-health-insurance-portability-and-accountability-act)
- [I](#i)
  - [IAM (Identity and Access Management)](#iam-identity-and-access-management)
  - [Iceberg Snapshot](#iceberg-snapshot)
  - [In-place Migration](#in-place-migration)
- [K](#k)
  - [KMS (Key Management Service)](#kms-key-management-service)
- [L](#l)
  - [Lake Formation](#lake-formation)
- [M](#m)
  - [Manifest File](#manifest-file)
  - [Metadata](#metadata)
- [N](#n)
  - [Namespace](#namespace)
- [O](#o)
  - [ORC (Optimized Row Columnar)](#orc-optimized-row-columnar)
- [P](#p)
  - [Parquet](#parquet)
  - [Partition Evolution](#partition-evolution)
  - [PII (Personally Identifiable Information)](#pii-personally-identifiable-information)
- [R](#r)
  - [RPO (Recovery Point Objective)](#rpo-recovery-point-objective)
  - [RTO (Recovery Time Objective)](#rto-recovery-time-objective)
- [S](#s)
  - [Schema Evolution](#schema-evolution)
  - [Snapshot Management](#snapshot-management)
  - [SSE-KMS](#sse-kms)
  - [SSE-S3](#sse-s3)
- [T](#t)
  - [Table Bucket](#table-bucket)
  - [Time Travel Query](#time-travel-query)
  - [TPS (Transactions Per Second)](#tps-transactions-per-second)
- [U](#u)
  - [Unreferenced File Removal](#unreferenced-file-removal)
  - [Upsert](#upsert)
- [V](#v)
  - [VPC Endpoint](#vpc-endpoint)
- [Z](#z)
  - [Z-Ordering](#z-ordering)
- [関連ドキュメント](#関連ドキュメント)
- [更新履歴](#更新履歴)

## 概要

本ドキュメントでは、Amazon S3 Tablesおよび関連技術で使用される専門用語を定義します。

---

## A

### ACID
**Atomicity, Consistency, Isolation, Durability**の略。トランザクション処理の4つの特性。Apache Icebergはこれらの特性をサポートします。

### Apache Iceberg
オープンソースのテーブルフォーマット。大規模な分析データセットのための高性能なテーブル管理機能を提供します。

### AppendOnly
Firehoseの設定フラグ。Insert専用の場合に設定することで自動スケールが有効化されます。

---

## B

### Block Public Access
S3のセキュリティ機能。S3 Tablesでは常に有効で、無効化できません。

### Binpack
コンパクション戦略の一つ。小さなファイルを大きなファイルに統合します。

---

## C

### Catalog
テーブルのメタデータを管理するシステム。S3 TablesはAWS Glue Data Catalogと統合されます。

### CDC (Change Data Capture)
データベースの変更をキャプチャし、他のシステムに伝播する技術。

### Compaction
小さなファイルを大きなファイルに統合する処理。クエリパフォーマンスを向上させます。

### CTAS (CREATE TABLE AS SELECT)
既存のテーブルからデータを読み取り、新しいテーブルを作成するSQL文。

---

## D

### Data Lake
様々な形式のデータを大規模に保存するストレージリポジトリ。

### Data Lineage
データの起源、移動、変換の履歴を追跡すること。

### DR (Disaster Recovery)
災害やシステム障害からの復旧計画。

---

## E

### ETL (Extract, Transform, Load)
データを抽出、変換、ロードする処理。

### Exactly-once Delivery
各レコードが正確に1回だけ配信されることを保証する配信セマンティクス。

---

## F

### Firehose
Amazon Data Firehoseの略。ストリーミングデータをリアルタイムで配信するサービス。

### Full Data Migration
既存データを完全に書き直して移行する方法。

---

## G

### GDPR (General Data Protection Regulation)
EU一般データ保護規則。個人データの保護に関する規制。

---

## H

### HIPAA (Health Insurance Portability and Accountability Act)
医療保険の相互運用性と説明責任に関する法律。

---

## I

### IAM (Identity and Access Management)
AWSのアクセス制御サービス。

### Iceberg Snapshot
テーブルの特定時点の状態を表すメタデータ。

### In-place Migration
既存データを書き直さずにメタデータのみを変換する移行方法。S3 Tablesでは非サポート。

---

## K

### KMS (Key Management Service)
AWSの暗号化キー管理サービス。

---

## L

### Lake Formation
AWSのデータレイク管理サービス。きめ細かいアクセス制御を提供します。

---

## M

### Manifest File
Icebergテーブルのメタデータファイルの一つ。データファイルのリストを含みます。

### Metadata
データに関するデータ。テーブル構造、パーティション情報などを含みます。

---

## N

### Namespace
テーブルを論理的にグループ化する単位。データベースに相当します。

---

## O

### ORC (Optimized Row Columnar)
列指向のファイルフォーマット。

---

## P

### Parquet
列指向のファイルフォーマット。分析ワークロードに最適化されています。

### Partition Evolution
パーティション戦略を動的に変更できるIcebergの機能。

### PII (Personally Identifiable Information)
個人を特定できる情報。

---

## R

### RPO (Recovery Point Objective)
許容可能なデータ損失の時間。

### RTO (Recovery Time Objective)
許容可能なサービス停止時間。

---

## S

### Schema Evolution
データ構造を変更できるIcebergの機能。既存データへの影響を最小限に抑えます。

### Snapshot Management
古いスナップショットを削除してストレージコストを削減する機能。

### SSE-KMS
AWS KMSを使用したサーバーサイド暗号化。

### SSE-S3
S3管理キーを使用したサーバーサイド暗号化。

---

## T

### Table Bucket
S3 Tablesの最上位リソース。テーブルを格納するコンテナ。

### Time Travel Query
過去の特定時点のデータを照会する機能。

### TPS (Transactions Per Second)
1秒あたりのトランザクション数。

---

## U

### Unreferenced File Removal
Icebergメタデータから参照されなくなったファイルを削除する機能。

### Upsert
Update + Insertの造語。レコードが存在する場合は更新、存在しない場合は挿入する操作。

---

## V

### VPC Endpoint
VPC内からAWSサービスにプライベートアクセスするためのエンドポイント。

---

## Z

### Z-Ordering
複数のカラムでデータをソートする最適化手法。クエリパフォーマンスを向上させます。

---

## 関連ドキュメント
- [概要](../01-getting-started/01-overview.md): 概要
- [機能](../01-getting-started/02-features.md): 主要機能
- 

## 更新履歴
- 2025-11-15: 初版作成
