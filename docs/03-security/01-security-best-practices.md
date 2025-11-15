# Amazon S3 Tables セキュリティベストプラクティス


## 目次

- [概要](#概要)
  - [セキュリティの5つの柱](#セキュリティの5つの柱)
- [アイデンティティとアクセス管理（IAM）](#アイデンティティとアクセス管理iam)
  - [リソースとARN形式](#リソースとarn形式)
    - [Table Bucket ARN](#table-bucket-arn)
    - [Table ARN](#table-arn)
  - [IAMアクション一覧](#iamアクション一覧)
    - [Table Bucketレベルのアクション](#table-bucketレベルのアクション)
    - [Namespaceレベルのアクション](#namespaceレベルのアクション)
    - [Tableレベルのアクション](#tableレベルのアクション)
    - [データアクセスとS3 APIの関係](#データアクセスとs3-apiの関係)
  - [IAM Condition Keys](#iam-condition-keys)
    - [s3tables:tableName](#s3tablestablename)
    - [s3tables:namespace](#s3tablesnamespace)
    - [s3tables:SSEAlgorithm](#s3tablesssealgorithm)
    - [s3tables:KMSKeyArn](#s3tableskmskeyarn)
  - [Identity-Based Policies（アイデンティティベースポリシー）](#identity-based-policiesアイデンティティベースポリシー)
    - [読み取り専用アクセスポリシー例](#読み取り専用アクセスポリシー例)
    - [フルアクセスポリシー例](#フルアクセスポリシー例)
    - [Namespace別アクセス制御ポリシー例](#namespace別アクセス制御ポリシー例)
  - [Resource-Based Policies（リソースベースポリシー）](#resource-based-policiesリソースベースポリシー)
    - [Table Bucket Policy](#table-bucket-policy)
    - [Table Policy](#table-policy)
  - [AWS Organizations Service Control Policies (SCPs)](#aws-organizations-service-control-policies-scps)
- [データ保護: 暗号化](#データ保護-暗号化)
  - [転送中のデータ保護](#転送中のデータ保護)
  - [保管中のデータ保護](#保管中のデータ保護)
    - [SSE-S3（Amazon S3管理キー）](#sse-s3amazon-s3管理キー)
    - [SSE-KMS（AWS KMS管理キー）](#sse-kmsaws-kms管理キー)
  - [SSE-KMS権限要件](#sse-kms権限要件)
    - [必須権限](#必須権限)
    - [メンテナンスサービスプリンシパルへの権限付与](#メンテナンスサービスプリンシパルへの権限付与)
    - [AWS分析サービス統合での権限付与](#aws分析サービス統合での権限付与)
    - [直接アクセスでの権限付与](#直接アクセスでの権限付与)
    - [S3 Metadataサービスプリンシパルへの権限付与](#s3-metadataサービスプリンシパルへの権限付与)
    - [クロスアカウントKMSキーの権限](#クロスアカウントkmsキーの権限)
  - [暗号化の強制](#暗号化の強制)
    - [Table BucketレベルでSSE-KMSを強制](#table-bucketレベルでsse-kmsを強制)
    - [特定のKMSキーの使用を強制](#特定のkmsキーの使用を強制)
- [インフラストラクチャ保護: VPC接続](#インフラストラクチャ保護-vpc接続)
  - [VPCエンドポイントの概要](#vpcエンドポイントの概要)
  - [エンドポイントアーキテクチャ](#エンドポイントアーキテクチャ)
  - [推奨VPCエンドポイント構成](#推奨vpcエンドポイント構成)
  - [VPCエンドポイントの作成](#vpcエンドポイントの作成)
    - [S3 Tables Interface Endpointの作成](#s3-tables-interface-endpointの作成)
    - [S3 Gateway Endpointの作成](#s3-gateway-endpointの作成)
  - [エンドポイント固有のDNS名](#エンドポイント固有のdns名)
    - [Regional DNS名](#regional-dns名)
    - [Zonal DNS名](#zonal-dns名)
  - [Private DNS](#private-dns)
  - [AWS CLIでのVPCエンドポイント使用](#aws-cliでのvpcエンドポイント使用)
    - [Table Bucketsの一覧表示](#table-bucketsの一覧表示)
    - [Tablesの一覧表示](#tablesの一覧表示)
  - [クエリエンジンでのVPC設定](#クエリエンジンでのvpc設定)
    - [Amazon EMRでの設定手順](#amazon-emrでの設定手順)
  - [Dual-Stack（IPv6）サポート](#dual-stackipv6サポート)
    - [Dual-Stackエンドポイント形式](#dual-stackエンドポイント形式)
    - [IPv6アクセスの前提条件](#ipv6アクセスの前提条件)
    - [Dual-Stack VPCエンドポイントの作成](#dual-stack-vpcエンドポイントの作成)
  - [VPCエンドポイントポリシー](#vpcエンドポイントポリシー)
    - [特定のTable Bucketへのアクセスを制限](#特定のtable-bucketへのアクセスを制限)
    - [VPC内からのアクセスのみを許可](#vpc内からのアクセスのみを許可)
- [検出: CloudTrailログ](#検出-cloudtrailログ)
  - [CloudTrail統合の概要](#cloudtrail統合の概要)
  - [管理イベント（Management Events）](#管理イベントmanagement-events)
    - [特徴](#特徴)
    - [記録される管理イベントAPI（26個）](#記録される管理イベントapi26個)
  - [メンテナンスイベント](#メンテナンスイベント)
    - [メンテナンスイベントの識別方法](#メンテナンスイベントの識別方法)
    - [メンテナンスイベントログ例](#メンテナンスイベントログ例)
  - [データイベント（Data Events）](#データイベントdata-events)
    - [特徴](#特徴)
    - [記録されるデータイベントAPI（7個）](#記録されるデータイベントapi7個)
    - [データイベントの有効化](#データイベントの有効化)
  - [CloudTrailログの分析](#cloudtrailログの分析)
    - [CloudWatch Logs Insightsでのクエリ例](#cloudwatch-logs-insightsでのクエリ例)
  - [CloudTrail監視のベストプラクティス](#cloudtrail監視のベストプラクティス)
    - [1. CloudWatch Logsへの統合](#1-cloudwatch-logsへの統合)
    - [2. メトリクスフィルターの作成](#2-メトリクスフィルターの作成)
    - [3. CloudWatch Alarmsの設定](#3-cloudwatch-alarmsの設定)
    - [4. 定期的なログ分析](#4-定期的なログ分析)
- [属性ベースアクセス制御（ABAC）](#属性ベースアクセス制御abac)
  - [ABACの概要](#abacの概要)
  - [ABACの利点](#abacの利点)
  - [タグベースのCondition Keys](#タグベースのcondition-keys)
  - [ABACポリシー例](#abacポリシー例)
    - [部門タグに基づくアクセス制御](#部門タグに基づくアクセス制御)
    - [プロジェクトタグに基づくTable作成制限](#プロジェクトタグに基づくtable作成制限)
    - [環境タグに基づくアクセス制御](#環境タグに基づくアクセス制御)
  - [タグの管理](#タグの管理)
    - [Table Bucketへのタグ付け](#table-bucketへのタグ付け)
    - [Tableへのタグ付け](#tableへのタグ付け)
  - [Lake Formationとの統合時の注意事項](#lake-formationとの統合時の注意事項)
- [セキュリティベストプラクティス](#セキュリティベストプラクティス)
  - [1. 最小権限の原則](#1-最小権限の原則)
    - [推奨事項](#推奨事項)
    - [実装例](#実装例)
  - [2. 多層防御](#2-多層防御)
    - [推奨事項](#推奨事項)
    - [実装例](#実装例)
  - [3. 暗号化の強制](#3-暗号化の強制)
    - [推奨事項](#推奨事項)
    - [実装例](#実装例)
  - [4. ネットワーク分離](#4-ネットワーク分離)
    - [推奨事項](#推奨事項)
    - [実装例](#実装例)
  - [5. 監査とモニタリング](#5-監査とモニタリング)
    - [推奨事項](#推奨事項)
    - [実装例](#実装例)
  - [6. タグ戦略](#6-タグ戦略)
    - [推奨事項](#推奨事項)
    - [実装例](#実装例)
  - [7. クロスアカウントアクセス](#7-クロスアカウントアクセス)
    - [推奨事項](#推奨事項)
    - [実装例](#実装例)
  - [8. インシデント対応](#8-インシデント対応)
    - [推奨事項](#推奨事項)
    - [実装例](#実装例)
  - [9. Firehose統合時のセキュリティ](#9-firehose統合時のセキュリティ)
    - [推奨事項](#推奨事項)
    - [実装例](#実装例)
    - [監視とアラート](#監視とアラート)
- [まとめ](#まとめ)
  - [セキュリティチェックリスト](#セキュリティチェックリスト)
- [出典](#出典)

## 概要

Amazon S3 Tablesは、Apache Icebergテーブルを安全に管理するための包括的なセキュリティ機能を提供します。本ドキュメントでは、S3 Tablesのセキュリティ機能とベストプラクティスを詳細に解説します。

### セキュリティの5つの柱

S3 Tablesのセキュリティは、AWS Well-Architected Frameworkのセキュリティの柱に基づいて設計されています：

1. **アイデンティティとアクセス管理**: IAMポリシー、リソースベースポリシー、ABAC
2. **検出**: CloudTrailログ、監査証跡
3. **インフラストラクチャ保護**: VPCエンドポイント、ネットワーク分離
4. **データ保護**: 暗号化（転送中・保管中）
5. **インシデント対応**: ログ分析、アラート

## アイデンティティとアクセス管理（IAM）

### リソースとARN形式

S3 Tablesのリソースは、`s3tables`ネームスペースを使用した独自のARN形式を持ちます。

#### Table Bucket ARN
```text
arn:aws:s3tables:region:account-id:bucket/table-bucket-name
```

#### Table ARN
```text
arn:aws:s3tables:region:account-id:bucket/table-bucket-name/table/table-id
```

### IAMアクション一覧

S3 Tablesは、すべてのアクションが`s3tables`ネームスペースに属します。以下は、サポートされているすべてのIAMアクションの詳細です。

#### Table Bucketレベルのアクション

| アクション | 説明 | アクセスレベル | クロスアカウント対応 |
|-----------|------|--------------|-------------------|
| `s3tables:CreateTableBucket` | Table Bucketを作成する権限 | Write | No |
| `s3tables:GetTableBucket` | Table BucketのARN、名前、作成日を取得する権限 | Read | Yes |
| `s3tables:ListTableBuckets` | アカウント内のすべてのTable Bucketを一覧表示する権限 | Read | No |
| `s3tables:DeleteTableBucket` | Table Bucketを削除する権限 | Write | Yes |
| `s3tables:PutTableBucketPolicy` | Table Bucket Policyを追加または置換する権限 | Permissions Management | No |
| `s3tables:GetTableBucketPolicy` | Table Bucket Policyを取得する権限 | Read | No |
| `s3tables:DeleteTableBucketPolicy` | Table Bucket Policyを削除する権限 | Permissions Management | No |
| `s3tables:PutTableBucketEncryption` | Table Bucketの暗号化設定を追加または置換する権限 | Write | No |
| `s3tables:GetTableBucketEncryption` | Table Bucketの暗号化設定を取得する権限 | Read | No |
| `s3tables:DeleteTableBucketEncryption` | Table Bucketの暗号化設定を削除する権限 | Write | No |
| `s3tables:PutTableBucketMaintenanceConfiguration` | Table Bucketのメンテナンス設定を追加または置換する権限 | Write | Yes |
| `s3tables:GetTableBucketMaintenanceConfiguration` | Table Bucketのメンテナンス設定を取得する権限 | Read | Yes |

#### Namespaceレベルのアクション

| アクション | 説明 | アクセスレベル | クロスアカウント対応 |
|-----------|------|--------------|-------------------|
| `s3tables:CreateNamespace` | Table Bucket内にNamespaceを作成する権限 | Write | Yes |
| `s3tables:GetNamespace` | Namespaceの詳細を取得する権限 | Read | Yes |
| `s3tables:ListNamespaces` | Table Bucket内のすべてのNamespaceを一覧表示する権限 | Read | Yes |
| `s3tables:DeleteNamespace` | Table Bucket内のNamespaceを削除する権限 | Write | Yes |

#### Tableレベルのアクション

| アクション | 説明 | アクセスレベル | クロスアカウント対応 |
|-----------|------|--------------|-------------------|
| `s3tables:CreateTable` | Table Bucket内にTableを作成する権限 | Write | Yes |
| `s3tables:GetTable` | Tableの情報を取得する権限 | Read | Yes |
| `s3tables:ListTables` | Table Bucket内のすべてのTableを一覧表示する権限 | Read | Yes |
| `s3tables:RenameTable` | Tableの名前を変更する権限 | Write | Yes |
| `s3tables:DeleteTable` | Table BucketからTableを削除する権限 | Write | Yes |
| `s3tables:GetTableMetadataLocation` | Tableのルートポインタ（メタデータファイル）を取得する権限 | Read | Yes |
| `s3tables:UpdateTableMetadataLocation` | Tableのルートポインタ（メタデータファイル）を更新する権限 | Write | Yes |
| `s3tables:GetTableData` | Tableのメタデータとデータオブジェクトを読み取る権限 | Read | Yes |
| `s3tables:PutTableData` | Tableのメタデータとデータオブジェクトを書き込む権限 | Write | Yes |
| `s3tables:PutTablePolicy` | Table Policyを追加または置換する権限 | Permissions Management | No |
| `s3tables:GetTablePolicy` | Table Policyを取得する権限 | Read | No |
| `s3tables:DeleteTablePolicy` | Table Policyを削除する権限 | Permissions Management | No |
| `s3tables:PutTableEncryption` | Tableに暗号化を追加する権限 | Write | No |
| `s3tables:GetTableEncryption` | Tableの暗号化設定を取得する権限 | Read | No |
| `s3tables:PutTableMaintenanceConfiguration` | Tableのメンテナンス設定を追加または置換する権限 | Write | Yes |
| `s3tables:GetTableMaintenanceConfiguration` | Tableのメンテナンス設定を取得する権限 | Read | Yes |
| `s3tables:GetTableMaintenanceJobStatus` | Tableのメンテナンスジョブステータスを取得する権限 | Read | Yes |

#### データアクセスとS3 APIの関係

`s3tables:GetTableData`および`s3tables:PutTableData`アクションは、複数のS3 APIオペレーションへのアクセスを制御します。

**s3tables:GetTableData**が含むS3 APIオペレーション：
- `GetObject`: データファイルとメタデータファイルの読み取り
- `HeadObject`: オブジェクトメタデータの取得
- `ListParts`: マルチパートアップロードのパート一覧取得

**s3tables:PutTableData**が含むS3 APIオペレーション：
- `PutObject`: データファイルとメタデータファイルの書き込み
- `CreateMultipartUpload`: マルチパートアップロードの開始
- `UploadPart`: マルチパートアップロードのパートアップロード
- `CompleteMultipartUpload`: マルチパートアップロードの完了
- `AbortMultipartUpload`: マルチパートアップロードの中止

### IAM Condition Keys

S3 Tablesは、AWS グローバルCondition Keysに加えて、以下のサービス固有のCondition Keysをサポートしています。

#### s3tables:tableName

**用途**: Table名によるアクセスフィルタリング

**タイプ**: String

**説明**: Table Bucket内のTableの名前に基づいてアクセスをフィルタリングします。ワイルドカード（`*`）を使用したパターンマッチングが可能です。

**使用例**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3tables:GetTable",
        "s3tables:GetTableData"
      ],
      "Resource": "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket/table/*",
      "Condition": {
        "StringLike": {
          "s3tables:tableName": "department-*"
        }
      }
    }
  ]
}
```

**注意事項**: Table名を変更すると、このCondition Keyを使用したポリシーに影響を与える可能性があります。

#### s3tables:namespace

**用途**: Namespaceによるアクセスフィルタリング

**タイプ**: String

**説明**: Table Bucket内のNamespaceに基づいてアクセスをフィルタリングします。特定のNamespaceに属するTableへのアクセスを制限できます。

**使用例**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3tables:CreateTable",
        "s3tables:GetTable",
        "s3tables:ListTables"
      ],
      "Resource": "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket/*",
      "Condition": {
        "StringEquals": {
          "s3tables:namespace": "hr"
        }
      }
    }
  ]
}
```

**注意事項**: Namespaceを変更すると、このCondition Keyを使用したポリシーに影響を与える可能性があります。

#### s3tables:SSEAlgorithm

**用途**: 暗号化アルゴリズムによるアクセスフィルタリング

**タイプ**: String

**説明**: Tableの暗号化に使用されるサーバーサイド暗号化アルゴリズムに基づいてアクセスをフィルタリングします。

**有効な値**:
- `AES256`: SSE-S3（Amazon S3管理キー）
- `aws:kms`: SSE-KMS（AWS KMS管理キー）

**使用例**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3tables:GetTableData",
        "s3tables:PutTableData"
      ],
      "Resource": "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket/table/*",
      "Condition": {
        "StringEquals": {
          "s3tables:SSEAlgorithm": "aws:kms"
        }
      }
    }
  ]
}
```

**注意事項**: 暗号化設定を変更すると、このCondition Keyを使用したポリシーに影響を与える可能性があります。

#### s3tables:KMSKeyArn

**用途**: KMSキーによるアクセスフィルタリング

**タイプ**: ARN

**説明**: Tableの暗号化に使用されるAWS KMSキーのARNに基づいてアクセスをフィルタリングします。特定のKMSキーで暗号化されたTableへのアクセスを制限できます。

**使用例**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3tables:GetTableData",
        "s3tables:PutTableData"
      ],
      "Resource": "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket/table/*",
      "Condition": {
        "StringEquals": {
          "s3tables:KMSKeyArn": "arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012"
        }
      }
    }
  ]
}
```

**注意事項**: KMSキーを変更すると、このCondition Keyを使用したポリシーに影響を与える可能性があります。


### Identity-Based Policies（アイデンティティベースポリシー）

Identity-Based Policiesは、IAMユーザー、グループ、またはロールにアタッチされます。デフォルトでは、ユーザーとロールはTable BucketやTableを作成・変更する権限を持ちません。

#### 読み取り専用アクセスポリシー例

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadOnlyAccessToTables",
      "Effect": "Allow",
      "Action": [
        "s3tables:GetTableBucket",
        "s3tables:ListTableBuckets",
        "s3tables:GetNamespace",
        "s3tables:ListNamespaces",
        "s3tables:GetTable",
        "s3tables:ListTables",
        "s3tables:GetTableData",
        "s3tables:GetTableMetadataLocation"
      ],
      "Resource": [
        "arn:aws:s3tables:us-east-1:123456789012:bucket/*"
      ]
    }
  ]
}
```

#### フルアクセスポリシー例

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "FullAccessToTables",
      "Effect": "Allow",
      "Action": [
        "s3tables:*"
      ],
      "Resource": [
        "arn:aws:s3tables:us-east-1:123456789012:bucket/*"
      ]
    }
  ]
}
```

#### Namespace別アクセス制御ポリシー例

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AccessToHRNamespace",
      "Effect": "Allow",
      "Action": [
        "s3tables:CreateTable",
        "s3tables:GetTable",
        "s3tables:ListTables",
        "s3tables:GetTableData",
        "s3tables:PutTableData"
      ],
      "Resource": "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket/*",
      "Condition": {
        "StringEquals": {
          "s3tables:namespace": "hr"
        }
      }
    }
  ]
}
```

### Resource-Based Policies（リソースベースポリシー）

Resource-Based Policiesは、リソース（Table BucketまたはTable）にアタッチされます。

#### Table Bucket Policy

Table Bucket Policyは、Table Bucket全体およびNamespaceレベルのAPIアクセス権限を制御します。また、Bucket内の複数のTableに対するTableレベルのAPI権限も制御できます。

**Table Bucket Policy例**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCrossAccountAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:root"
      },
      "Action": [
        "s3tables:GetTableBucket",
        "s3tables:ListNamespaces",
        "s3tables:ListTables",
        "s3tables:GetTable"
      ],
      "Resource": [
        "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket",
        "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket/*"
      ]
    }
  ]
}
```

#### Table Policy

Table Policyは、個別のTableに対するTableレベルのAPIアクセス権限を付与します。

**Table Policy例**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSpecificUserAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:user/data-analyst"
      },
      "Action": [
        "s3tables:GetTable",
        "s3tables:GetTableData"
      ],
      "Resource": "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket/table/my-table-id"
    }
  ]
}
```

### AWS Organizations Service Control Policies (SCPs)

S3 TablesはAWS OrganizationsのService Control Policies (SCPs)をサポートしています。すべてのTableおよびBucketレベルのアクションは、`s3tables`ネームスペースとして参照されます。

**SCP例: 特定リージョンでのTable Bucket作成を拒否**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyTableBucketCreationInSpecificRegions",
      "Effect": "Deny",
      "Action": "s3tables:CreateTableBucket",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "us-east-1",
            "us-west-2"
          ]
        }
      }
    }
  ]
}
```

## データ保護: 暗号化

### 転送中のデータ保護

S3 Tablesは、すべてのデータ転送でTransport Layer Security (TLS) 1.2以上を使用してHTTPS経由で保護します。HTTP接続は**サポートされていません**。

### 保管中のデータ保護

S3 Tablesは、保管中のデータを保護するために2つのサーバーサイド暗号化オプションを提供します。

#### SSE-S3（Amazon S3管理キー）

**特徴**:
- すべてのTable Bucketでデフォルトで有効
- 追加コストなし
- 各オブジェクトは一意のキーで暗号化
- キー自体は定期的にローテーションされるルートキーで暗号化
- AES-256暗号化を使用

**設定**:
デフォルトで有効のため、追加設定は不要です。

**ユースケース**:
- 標準的なセキュリティ要件
- コスト最適化が重要な場合
- キー管理の複雑さを避けたい場合

#### SSE-KMS（AWS KMS管理キー）

**特徴**:
- Customer Managed Keysのみサポート（AWS Managed Keysは非サポート）
- KMSキーの作成、表示、編集、監視、有効化/無効化、ローテーション、削除スケジュール設定が可能
- キーの使用方法と使用者を定義するポリシーを設定可能
- CloudTrailでキー使用状況を追跡可能
- コンプライアンス要件に対応

**設定方法**:

Table Bucketレベルでの設定:
```bash
aws s3tables put-table-bucket-encryption \
  --table-bucket-arn arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket \
  --encryption-configuration '{
    "SSEKMSConfiguration": {
      "KMSKeyArn": "arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012"
    }
  }'
```

Tableレベルでの設定:
```bash
aws s3tables put-table-encryption \
  --table-bucket-arn arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket \
  --namespace my-namespace \
  --name my-table \
  --encryption-configuration '{
    "SSEKMSConfiguration": {
      "KMSKeyArn": "arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012"
    }
  }'
```

**ユースケース**:
- 厳格なコンプライアンス要件がある場合
- キーの使用状況を詳細に監査する必要がある場合
- キーのローテーションポリシーをカスタマイズしたい場合
- クロスアカウントアクセスでキーレベルの制御が必要な場合

### SSE-KMS権限要件

SSE-KMSを使用する場合、複数のプリンシパルに適切な権限を付与する必要があります。

#### 必須権限

すべてのSSE-KMS暗号化Tableにアクセスするには、以下のKMS権限が必要です：
- `kms:GenerateDataKey`
- `kms:Decrypt`

#### メンテナンスサービスプリンシパルへの権限付与

**重要**: SSE-KMS暗号化Tableを作成するには、S3 Tablesメンテナンスサービスプリンシパル（`maintenance.s3tables.amazonaws.com`）にKMSキーへのアクセス権限を付与する必要があります。

**KMSキーポリシー例**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnableMaintenanceServiceAccess",
      "Effect": "Allow",
      "Principal": {
        "Service": "maintenance.s3tables.amazonaws.com"
      },
      "Action": [
        "kms:GenerateDataKey",
        "kms:Decrypt"
      ],
      "Resource": "arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012",
      "Condition": {
        "StringLike": {
          "kms:EncryptionContext:aws:s3:arn": "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket/*"
        }
      }
    }
  ]
}
```

**注意**: SSE-KMS暗号化Tableを作成するリクエストを行うと、S3 Tablesは`maintenance.s3tables.amazonaws.com`プリンシパルがKMSキーにアクセスできることを確認します。この確認のため、Table Bucket内に一時的にゼロバイトオブジェクトが作成され、未参照ファイル削除メンテナンス操作によって自動的に削除されます。KMSキーにメンテナンスアクセスがない場合、CreateTable操作は失敗します。

#### AWS分析サービス統合での権限付与

AWS分析サービス（Athena、EMR、Redshift等）でSSE-KMS暗号化Tableを使用する場合、統合ロールにKMSキーへのアクセス権限が必要です。

**S3TablesRoleForLakeFormationロールへの権限付与例**:
```json
{
  "Sid": "AllowAnalyticsServiceAccess",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:role/service-role/S3TablesRoleForLakeFormation"
  },
  "Action": [
    "kms:GenerateDataKey",
    "kms:Decrypt"
  ],
  "Resource": "arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012"
}
```

#### 直接アクセスでの権限付与

Iceberg REST APIやS3 Tables Catalog for Apache Icebergを使用して直接アクセスする場合、使用するIAMロールにKMSキーへのアクセス権限が必要です。

**IAMポリシー例**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowKMSKeyUsage",
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012"
    }
  ]
}
```

**KMSキーポリシー例**:
```json
{
  "Sid": "AllowDirectAccess",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:role/my-iceberg-client-role"
  },
  "Action": [
    "kms:Decrypt",
    "kms:GenerateDataKey"
  ],
  "Resource": "*"
}
```

#### S3 Metadataサービスプリンシパルへの権限付与

S3 MetadataテーブルでSSE-KMS暗号化を使用する場合、S3 Metadataサービスプリンシパル（`metadata.s3.amazonaws.com`）にKMSキーへのアクセス権限が必要です。

**KMSキーポリシー例**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnableMetadataServiceAccess",
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "maintenance.s3tables.amazonaws.com",
          "metadata.s3.amazonaws.com"
        ]
      },
      "Action": [
        "kms:GenerateDataKey",
        "kms:Decrypt"
      ],
      "Resource": "arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012",
      "Condition": {
        "StringLike": {
          "kms:EncryptionContext:aws:s3:arn": "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket/*"
        }
      }
    }
  ]
}
```

#### クロスアカウントKMSキーの権限

クロスアカウントでKMSキーを使用する場合、IAMロールにはキーアクセス権限とキーポリシーでの明示的な承認の両方が必要です。

### 暗号化の強制

#### Table BucketレベルでSSE-KMSを強制

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RequireSSEKMS",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3tables:CreateTable",
      "Resource": "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket/*",
      "Condition": {
        "StringNotEquals": {
          "s3tables:SSEAlgorithm": "aws:kms"
        }
      }
    }
  ]
}
```

#### 特定のKMSキーの使用を強制

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RequireSpecificKMSKey",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3tables:CreateTable",
      "Resource": "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket/*",
      "Condition": {
        "StringNotEquals": {
          "s3tables:KMSKeyArn": "arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012"
        }
      }
    }
  ]
}
```


## インフラストラクチャ保護: VPC接続

### VPCエンドポイントの概要

S3 Tablesは、AWS PrivateLinkを使用した2種類のVPCエンドポイントをサポートしています：

1. **Gateway Endpoint**: ルートテーブルで指定するゲートウェイ。AWSネットワーク経由でS3にアクセス
2. **Interface Endpoint**: プライベートIPアドレスを使用してリクエストをルーティング。VPC内、オンプレミス、または他のリージョンのVPCからアクセス可能

### エンドポイントアーキテクチャ

S3 TablesのTableは、2種類のS3オブジェクトで構成されています：
- **データファイル**: データを格納
- **メタデータファイル**: データファイルの情報を時系列で追跡

これらのオブジェクトは、異なるエンドポイントを経由してアクセスされます：

| 操作タイプ | エンドポイント | 例 |
|-----------|--------------|-----|
| Table Bucket、Namespace、Table操作 | S3 Tablesエンドポイント | `s3tables.region.amazonaws.com` |
| オブジェクトレベル操作（データ/メタデータファイル） | S3サービスエンドポイント | `s3.region.amazonaws.com` |

### 推奨VPCエンドポイント構成

S3 TablesにVPCからアクセスするには、**2つのVPCエンドポイント**を作成することを推奨します：

1. **S3用エンドポイント**: オブジェクトレベル操作用（Gateway EndpointまたはInterface Endpoint）
2. **S3 Tables用エンドポイント**: Table Bucket/Table操作用（Interface Endpoint）

### VPCエンドポイントの作成

#### S3 Tables Interface Endpointの作成

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --service-name com.amazonaws.us-east-1.s3tables \
  --subnet-ids subnet-12345678 subnet-87654321 \
  --security-group-ids sg-12345678 \
  --vpc-endpoint-type Interface \
  --private-dns-enabled
```

#### S3 Gateway Endpointの作成

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-12345678 \
  --vpc-endpoint-type Gateway
```

### エンドポイント固有のDNS名

VPCエンドポイントを作成すると、S3 Tablesは2種類のエンドポイント固有のDNS名を生成します。

#### Regional DNS名

**形式**: `VPCendpointID.s3tables.AWSregion.vpce.amazonaws.com`

**例**: `vpce-1a2b3c4d-5e6f.s3tables.us-east-1.vpce.amazonaws.com`

**用途**: リージョン全体でのアクセス

#### Zonal DNS名

**形式**: `VPCendpointID-AvailabilityZone.s3tables.AWSregion.vpce.amazonaws.com`

**例**: `vpce-1a2b3c4d-5e6f-us-east-1a.s3tables.us-east-1.vpce.amazonaws.com`

**用途**: Availability Zoneを分離するアーキテクチャで使用

### Private DNS

Private DNSオプションを使用すると、S3 Tablesのパブリックエンドポイント（`s3tables.region.amazonaws.com`）をVPC内のプライベートIPにマッピングできます。これにより、クライアントを更新せずにVPCエンドポイント経由でトラフィックをルーティングできます。

**有効化方法**:
```bash
aws ec2 modify-vpc-endpoint \
  --vpc-endpoint-id vpce-1a2b3c4d \
  --private-dns-enabled
```

### AWS CLIでのVPCエンドポイント使用

#### Table Bucketsの一覧表示

```bash
aws s3tables list-table-buckets \
  --endpoint-url https://vpce-1a2b3c4d-5e6f.s3tables.us-east-1.vpce.amazonaws.com \
  --region us-east-1
```

#### Tablesの一覧表示

```bash
aws s3tables list-tables \
  --table-bucket-arn arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket \
  --endpoint-url https://vpce-1a2b3c4d-5e6f.s3tables.us-east-1.vpce.amazonaws.com \
  --region us-east-1
```

### クエリエンジンでのVPC設定

#### Amazon EMRでの設定手順

1. VPCの作成または更新
2. S3 Tables用のInterface Endpointを作成
3. S3用のGateway EndpointまたはInterface Endpointを作成
4. EMRクラスターを起動
5. Sparkアプリケーションで追加設定を指定：

```bash
spark-submit \
  --conf spark.sql.catalog.ice_catalog.s3tables.endpoint=https://vpce-1a2b3c4d-5e6f.s3tables.us-east-1.vpce.amazonaws.com \
  my-spark-app.py
```

### Dual-Stack（IPv6）サポート

S3 TablesはAWS PrivateLinkのDual-Stack接続をサポートしています。IPv6とIPv4の両方のプロトコルを使用してTable Bucketsにアクセスできます。

#### Dual-Stackエンドポイント形式

```
s3tables.<region>.api.aws
```

#### IPv6アクセスの前提条件

1. **クライアントのDual-Stack有効化**: TableアクセスクライアントとS3クライアントの両方でDual-Stackを有効化
2. **セキュリティグループ設定**: IPv6インバウンドはデフォルトで無効。HTTPS（TCPポート443）を許可するルールを追加
3. **VPCのIPv6 CIDR**: VPCにIPv6 CIDRブロックが割り当てられていない場合は手動で追加
4. **IAMポリシー更新**: IPアドレスフィルタリングIAMポリシーをIPv6アドレスに対応するよう更新

#### Dual-Stack VPCエンドポイントの作成

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --service-name com.amazonaws.us-east-1.s3tables \
  --subnet-ids subnet-12345678 \
  --security-group-ids sg-12345678 \
  --vpc-endpoint-type Interface \
  --ip-address-type dualstack
```

### VPCエンドポイントポリシー

VPCエンドポイントポリシーを使用して、VPCエンドポイント経由でアクセスできるリソースを制限できます。

#### 特定のTable Bucketへのアクセスを制限

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RestrictToSpecificTableBucket",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3tables:*",
      "Resource": [
        "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket",
        "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket/*"
      ]
    }
  ]
}
```

#### VPC内からのアクセスのみを許可

Table Bucket Policyで、特定のVPCエンドポイントからのアクセスのみを許可できます。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RestrictToVPCEndpoint",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3tables:*",
      "Resource": [
        "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket",
        "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:SourceVpce": "vpce-1a2b3c4d"
        }
      }
    }
  ]
}
```

## 検出: CloudTrailログ

### CloudTrail統合の概要

S3 TablesはAWS CloudTrailと統合されており、すべてのAPI呼び出しをイベントとしてキャプチャします。CloudTrailを使用して、以下の情報を取得できます：
- 実行されたリクエスト
- リクエスト元のIPアドレス
- リクエストの実行者
- リクエストの実行時刻
- その他の詳細情報

### 管理イベント（Management Events）

管理イベントは、AWSアカウント内のリソースに対して実行される管理操作に関する情報を提供します。

#### 特徴
- **デフォルトで有効**: AWSアカウント設定時に自動的に有効化
- **イベントソース**: `s3tables.amazonaws.com`
- **追加コスト**: なし（最初のコピーは無料）

#### 記録される管理イベントAPI（26個）

**Table Bucket操作**:
- `CreateTableBucket`
- `GetTableBucket`
- `ListTableBuckets`
- `DeleteTableBucket`
- `PutTableBucketPolicy`
- `GetTableBucketPolicy`
- `DeleteTableBucketPolicy`
- `PutTableBucketEncryption`
- `GetTableBucketEncryption`
- `DeleteTableBucketEncryption`
- `PutTableBucketMaintenanceConfiguration`
- `GetTableBucketMaintenanceConfiguration`

**Namespace操作**:
- `CreateNamespace`
- `GetNamespace`
- `ListNamespaces`
- `DeleteNamespace`

**Table操作**:
- `CreateTable`
- `GetTable`
- `ListTables`
- `RenameTable`
- `DeleteTable`
- `GetTableMetadataLocation`
- `UpdateTableMetadataLocation`
- `PutTablePolicy`
- `GetTablePolicy`
- `DeleteTablePolicy`
- `PutTableMaintenanceConfiguration`
- `GetTableMaintenanceConfiguration`
- `GetTableMaintenanceJobStatus`

### メンテナンスイベント

S3 Tablesは、自動メンテナンス操作を`TablesMaintenanceEvent`管理イベントとしてCloudTrailに記録します。

#### メンテナンスイベントの識別方法

以下の属性値でメンテナンスイベントを識別できます：

| 属性 | 値 |
|------|-----|
| `eventSource` | `s3tables.amazonaws.com` |
| `eventType` | `AwsServiceEvent` |
| `eventName` | `TablesMaintenanceEvent` |
| `userAgent` | `maintenance.s3tables.amazonaws.com` |
| `activityType` | `IcebergCompaction`（コンパクション）<br>`IcebergSnapshotManagement`（スナップショット管理） |

#### メンテナンスイベントログ例

```json
{
  "eventVersion": "1.08",
  "userIdentity": {
    "type": "AWSService",
    "invokedBy": "s3tables.amazonaws.com"
  },
  "eventTime": "2024-12-01T10:00:00Z",
  "eventSource": "s3tables.amazonaws.com",
  "eventName": "TablesMaintenanceEvent",
  "awsRegion": "us-east-1",
  "sourceIPAddress": "s3tables.amazonaws.com",
  "userAgent": "maintenance.s3tables.amazonaws.com",
  "requestParameters": {
    "tableBucketArn": "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket",
    "namespace": "my-namespace",
    "tableName": "my-table",
    "activityType": "IcebergCompaction"
  },
  "responseElements": {
    "status": "Successful"
  },
  "eventType": "AwsServiceEvent",
  "recipientAccountId": "123456789012"
}
```

### データイベント（Data Events）

データイベントは、リソース内またはリソース上で実行されるリソース操作に関する情報を提供します。

#### 特徴
- **デフォルトで無効**: CloudTrail trailで明示的に有効化が必要
- **リソースタイプ**: `AWS::S3Tables::Table`、`AWS::S3Tables::TableBucket`
- **追加コスト**: あり（データイベントの記録には料金が発生）

#### 記録されるデータイベントAPI（7個）

- `AbortMultipartUpload`
- `CompleteMultipartUpload`
- `CreateMultipartUpload`
- `GetObject`
- `HeadObject`
- `ListParts`
- `PutObject`

#### データイベントの有効化

**AWS CLI**:
```bash
aws cloudtrail put-event-selectors \
  --trail-name my-trail \
  --event-selectors '[
    {
      "ReadWriteType": "All",
      "IncludeManagementEvents": true,
      "DataResources": [
        {
          "Type": "AWS::S3Tables::Table",
          "Values": ["arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket/table/*"]
        }
      ]
    }
  ]'
```

**CloudFormation**:
```yaml
MyTrail:
  Type: AWS::CloudTrail::Trail
  Properties:
    TrailName: my-s3-tables-trail
    S3BucketName: my-cloudtrail-bucket
    EventSelectors:
      - ReadWriteType: All
        IncludeManagementEvents: true
        DataResources:
          - Type: AWS::S3Tables::Table
            Values:
              - "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket/table/*"
```

### CloudTrailログの分析

#### CloudWatch Logs Insightsでのクエリ例

**特定のTableへのアクセスを検索**:
```
fields @timestamp, eventName, userIdentity.principalId, sourceIPAddress
| filter eventSource = "s3tables.amazonaws.com"
| filter requestParameters.tableName = "my-table"
| sort @timestamp desc
| limit 100
```

**メンテナンスイベントの失敗を検索**:
```
fields @timestamp, requestParameters.activityType, responseElements.status
| filter eventName = "TablesMaintenanceEvent"
| filter responseElements.status = "Failed"
| sort @timestamp desc
```

**特定ユーザーのアクションを検索**:
```
fields @timestamp, eventName, requestParameters
| filter userIdentity.principalId = "AIDAI23HXX2LMI5EXAMPLE"
| filter eventSource = "s3tables.amazonaws.com"
| sort @timestamp desc
```

### CloudTrail監視のベストプラクティス

#### 1. CloudWatch Logsへの統合

CloudTrailログをCloudWatch Logsに送信し、リアルタイム監視とアラートを設定します。

```bash
aws cloudtrail update-trail \
  --name my-trail \
  --cloud-watch-logs-log-group-arn arn:aws:logs:us-east-1:123456789012:log-group:cloudtrail-logs \
  --cloud-watch-logs-role-arn arn:aws:iam::123456789012:role/CloudTrailRole
```

#### 2. メトリクスフィルターの作成

重要なイベントに対してメトリクスフィルターを作成します。

**メンテナンスジョブ失敗のメトリクスフィルター**:
```bash
aws logs put-metric-filter \
  --log-group-name cloudtrail-logs \
  --filter-name MaintenanceJobFailures \
  --filter-pattern '{ $.eventName = "TablesMaintenanceEvent" && $.responseElements.status = "Failed" }' \
  --metric-transformations \
    metricName=MaintenanceJobFailures,\
metricNamespace=S3Tables,\
metricValue=1
```

#### 3. CloudWatch Alarmsの設定

メトリクスフィルターに基づいてアラームを設定します。

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name s3-tables-maintenance-failures \
  --alarm-description "Alert on S3 Tables maintenance job failures" \
  --metric-name MaintenanceJobFailures \
  --namespace S3Tables \
  --statistic Sum \
  --period 300 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:my-sns-topic
```

#### 4. 定期的なログ分析

CloudWatch Logs Insightsを使用して、定期的にログを分析し、異常なアクセスパターンを検出します。


## 属性ベースアクセス制御（ABAC）

### ABACの概要

属性ベースアクセス制御（ABAC）は、タグを使用してリソースへのアクセスを制御する認可戦略です。S3 Tablesは、Table BucketとTableの両方でタグをサポートしています（2024年11月6日追加）。

### ABACの利点

1. **スケーラビリティ**: 新しいリソースを追加する際にポリシーを更新する必要がない
2. **柔軟性**: 複数の属性を組み合わせてアクセス制御が可能
3. **監査性**: タグベースでアクセスパターンを追跡可能
4. **コスト配分**: タグを使用してコストを部門やプロジェクトに配分可能

### タグベースのCondition Keys

S3 TablesでABACを実装するために、以下のCondition Keysを使用できます：

| Condition Key | 説明 | 用途 |
|--------------|------|------|
| `aws:ResourceTag/key-name` | リソースにアタッチされたタグに基づいてアクセスを制御 | 既存リソースへのアクセス制御 |
| `aws:RequestTag/key-name` | リクエストで渡されたタグに基づいてアクセスを制御 | リソース作成時のタグ要件 |
| `aws:TagKeys` | リクエストで使用されるタグキーに基づいてアクセスを制御 | 許可されるタグキーの制限 |
| `s3tables:TableBucketTag/tag-key` | Table Bucketのタグに基づいてアクセスを制御 | Table Bucket固有のアクセス制御 |

### ABACポリシー例

#### 部門タグに基づくアクセス制御

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAccessBasedOnDepartmentTag",
      "Effect": "Allow",
      "Action": [
        "s3tables:GetTable",
        "s3tables:GetTableData",
        "s3tables:PutTableData"
      ],
      "Resource": "arn:aws:s3tables:us-east-1:123456789012:bucket/*/table/*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Department": "${aws:PrincipalTag/Department}"
        }
      }
    }
  ]
}
```

この例では、ユーザーのDepartmentタグとTableのDepartmentタグが一致する場合のみアクセスを許可します。

#### プロジェクトタグに基づくTable作成制限

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RequireProjectTagOnTableCreation",
      "Effect": "Allow",
      "Action": "s3tables:CreateTable",
      "Resource": "arn:aws:s3tables:us-east-1:123456789012:bucket/*/table/*",
      "Condition": {
        "StringEquals": {
          "aws:RequestTag/Project": "${aws:PrincipalTag/Project}"
        }
      }
    },
    {
      "Sid": "DenyTableCreationWithoutProjectTag",
      "Effect": "Deny",
      "Action": "s3tables:CreateTable",
      "Resource": "arn:aws:s3tables:us-east-1:123456789012:bucket/*/table/*",
      "Condition": {
        "Null": {
          "aws:RequestTag/Project": "true"
        }
      }
    }
  ]
}
```

#### 環境タグに基づくアクセス制御

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowProductionAccessToProductionTables",
      "Effect": "Allow",
      "Action": [
        "s3tables:GetTable",
        "s3tables:GetTableData"
      ],
      "Resource": "arn:aws:s3tables:us-east-1:123456789012:bucket/*/table/*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Environment": "production",
          "aws:PrincipalTag/Environment": "production"
        }
      }
    },
    {
      "Sid": "AllowDevelopmentAccessToAllTables",
      "Effect": "Allow",
      "Action": [
        "s3tables:GetTable",
        "s3tables:GetTableData",
        "s3tables:PutTableData"
      ],
      "Resource": "arn:aws:s3tables:us-east-1:123456789012:bucket/*/table/*",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalTag/Environment": "development"
        }
      }
    }
  ]
}
```

### タグの管理

#### Table Bucketへのタグ付け

```bash
aws s3tables put-table-bucket-tags \
  --table-bucket-arn arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket \
  --tags Department=Finance,Project=DataLake,Environment=production
```

#### Tableへのタグ付け

```bash
aws s3tables put-table-tags \
  --table-bucket-arn arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket \
  --namespace my-namespace \
  --name my-table \
  --tags Department=Finance,Project=DataLake,Sensitivity=High
```

### Lake Formationとの統合時の注意事項

Table BucketをAmazon SageMaker Lakehouseと統合する場合、タグベースのアクセス制御に関する追加の考慮事項があります。Lake Formationは独自のタグベースアクセス制御（LF-TBAC）を提供しており、S3 TablesのABACと併用する場合は、両方のアクセス制御メカニズムを考慮する必要があります。

## セキュリティベストプラクティス

### 1. 最小権限の原則

#### 推奨事項
- 必要最小限の権限のみを付与
- 定期的に権限を見直し、不要な権限を削除
- ワイルドカード（`*`）の使用を最小限に抑える

#### 実装例

**悪い例**:
```json
{
  "Effect": "Allow",
  "Action": "s3tables:*",
  "Resource": "*"
}
```

**良い例**:
```json
{
  "Effect": "Allow",
  "Action": [
    "s3tables:GetTable",
    "s3tables:GetTableData"
  ],
  "Resource": "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket/table/*",
  "Condition": {
    "StringEquals": {
      "s3tables:namespace": "analytics"
    }
  }
}
```

### 2. 多層防御

#### 推奨事項
- Identity-Based PolicyとResource-Based Policyを組み合わせる
- VPCエンドポイントポリシーで追加の制限を実施
- SCPsで組織レベルのガードレールを設定

#### 実装例

**IAMポリシー（Identity-Based）**:
```json
{
  "Effect": "Allow",
  "Action": [
    "s3tables:GetTable",
    "s3tables:GetTableData"
  ],
  "Resource": "arn:aws:s3tables:us-east-1:123456789012:bucket/*/table/*"
}
```

**Table Bucket Policy（Resource-Based）**:
```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3tables:*",
  "Resource": "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket/*",
  "Condition": {
    "StringNotEquals": {
      "aws:SourceVpce": "vpce-1a2b3c4d"
    }
  }
}
```

### 3. 暗号化の強制

#### 推奨事項
- すべてのTable BucketでSSE-KMSを使用
- Customer Managed Keysを使用してキーのライフサイクルを管理
- キーのローテーションを有効化
- クロスアカウントアクセスでは、キーポリシーで明示的に承認

#### 実装例

**SSE-KMSの強制**:
```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3tables:CreateTable",
  "Resource": "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket/*",
  "Condition": {
    "StringNotEquals": {
      "s3tables:SSEAlgorithm": "aws:kms"
    }
  }
}
```

**KMSキーの自動ローテーション**:
```bash
aws kms enable-key-rotation \
  --key-id 12345678-1234-1234-1234-123456789012
```

### 4. ネットワーク分離

#### 推奨事項
- VPCエンドポイントを使用してインターネット経由のアクセスを回避
- Private DNSを有効化してエンドポイント管理を簡素化
- セキュリティグループで必要なトラフィックのみを許可
- NACLsで追加のネットワークレベル制御を実施

#### 実装例

**セキュリティグループ設定**:
```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-12345678 \
  --protocol tcp \
  --port 443 \
  --cidr 10.0.0.0/16
```

**VPCエンドポイントからのアクセスのみを許可**:
```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3tables:*",
  "Resource": "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket/*",
  "Condition": {
    "StringNotEquals": {
      "aws:SourceVpce": [
        "vpce-1a2b3c4d",
        "vpce-5e6f7g8h"
      ]
    }
  }
}
```

### 5. 監査とモニタリング

#### 推奨事項
- CloudTrailで管理イベントとデータイベントの両方を有効化
- CloudWatch Logsに統合してリアルタイム分析を実施
- 重要なイベントに対してアラームを設定
- 定期的にアクセスパターンを分析

#### 実装例

**データイベントの有効化**:
```bash
aws cloudtrail put-event-selectors \
  --trail-name my-trail \
  --event-selectors '[
    {
      "ReadWriteType": "All",
      "IncludeManagementEvents": true,
      "DataResources": [
        {
          "Type": "AWS::S3Tables::Table",
          "Values": ["arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket/table/*"]
        }
      ]
    }
  ]'
```

**異常なアクセスパターンの検出**:
```text
fields @timestamp, userIdentity.principalId, eventName, sourceIPAddress
| filter eventSource = "s3tables.amazonaws.com"
| stats count() by userIdentity.principalId, sourceIPAddress
| filter count > 1000
```

### 6. タグ戦略

#### 推奨事項
- 一貫したタグ付け戦略を定義
- 必須タグを強制（Department、Project、Environment等）
- ABACを活用してスケーラブルなアクセス制御を実装
- コスト配分タグを使用してコストを追跡

#### 実装例

**必須タグの強制**:
```json
{
  "Effect": "Deny",
  "Action": [
    "s3tables:CreateTableBucket",
    "s3tables:CreateTable"
  ],
  "Resource": "*",
  "Condition": {
    "Null": {
      "aws:RequestTag/Department": "true"
    }
  }
}
```

### 7. クロスアカウントアクセス

#### 推奨事項
- 外部アカウントへのアクセスは明示的に承認
- `aws:SourceAccount`および`aws:SourceArn`条件を使用
- 定期的にクロスアカウントアクセスを監査
- 不要になったアクセスは即座に削除

#### 実装例

**安全なクロスアカウントアクセス**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCrossAccountAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:root"
      },
      "Action": [
        "s3tables:GetTable",
        "s3tables:GetTableData"
      ],
      "Resource": "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket/*",
      "Condition": {
        "StringEquals": {
          "aws:SourceAccount": "111111111111"
        }
      }
    }
  ]
}
```

### 8. インシデント対応

#### 推奨事項
- インシデント対応プレイブックを作成
- CloudTrailログを長期保存（最低90日、推奨1年以上）
- 自動化されたインシデント検出メカニズムを実装
- 定期的にインシデント対応訓練を実施

#### 実装例

**疑わしいアクティビティの自動検出**:
```bash
# Lambda関数でCloudTrailログを分析
# 異常なアクセスパターンを検出したらSNS通知を送信
aws lambda create-function \
  --function-name DetectAnomalousS3TablesAccess \
  --runtime python3.11 \
  --role arn:aws:iam::123456789012:role/LambdaExecutionRole \
  --handler index.handler \
  --zip-file fileb://function.zip
```

### 9. Firehose統合時のセキュリティ

Amazon Data Firehoseを使用してS3 Tablesにデータを配信する際のセキュリティ考慮事項です。

#### 推奨事項

**1. 暗号化設定の重要な制限**
- **制限**: Firehose側のKMS設定は使用されません
- **要件**: S3 Tables側でKMS暗号化を設定する必要があります
- **理由**: S3 Tablesの最適化されたストレージアーキテクチャとの統合のため

**2. IAMロールの最小権限設定**
- Firehoseが使用するIAMロールに必要最小限の権限を付与
- 特定のテーブルバケットとテーブルにアクセスを制限
- KMSキーへのアクセスを明示的に許可

**3. エラーバケットのセキュリティ**
- エラーバケットも暗号化を有効化
- エラーバケットへのアクセスを制限
- エラーログの定期的な監視

**4. Lambda変換関数のセキュリティ**
- Lambda関数に最小権限のIAMロールを付与
- 環境変数に機密情報を保存しない（Secrets Managerを使用）
- Lambda関数のVPC配置を検討

#### 実装例

**S3 Tables側での暗号化設定**:
```bash
# ✅ 正しい設定: S3 Tables側でKMS設定
aws s3tables create-table-bucket \
  --name my-table-bucket \
  --encryption-configuration \
    KmsKeyArn=arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012

# ❌ 誤った設定: Firehose側のKMS設定は無視される
aws firehose create-delivery-stream \
  --delivery-stream-name my-stream \
  --delivery-stream-encryption-configuration-input \
    KeyType=CUSTOMER_MANAGED_CMK,KeyARN=arn:aws:kms:...
```

**Firehose IAMロールポリシー**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3tables:GetTableMetadataLocation",
        "s3tables:UpdateTableMetadataLocation",
        "s3tables:GetTableData",
        "s3tables:PutTableData"
      ],
      "Resource": [
        "arn:aws:s3tables:us-east-1:123456789012:bucket/my-table-bucket/table/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "glue:GetTable",
        "glue:GetDatabase",
        "glue:UpdateTable"
      ],
      "Resource": [
        "arn:aws:glue:us-east-1:123456789012:catalog/s3tablescatalog/my-table-bucket",
        "arn:aws:glue:us-east-1:123456789012:database/s3tablescatalog/my-table-bucket/*",
        "arn:aws:glue:us-east-1:123456789012:table/s3tablescatalog/my-table-bucket/*/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "s3tables.us-east-1.amazonaws.com"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-error-bucket/*"
    }
  ]
}
```

**Lambda変換関数のIAMロール**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:123456789012:secret:my-secret-*"
    }
  ]
}
```

#### 監視とアラート

**CloudWatch Metricsの監視**:
- `DeliveryToS3.Success`: 成功率の監視
- `DeliveryToS3.DataFreshness`: データ鮮度の監視
- `IncomingRecords`: 入力レコード数の監視
- `DeliveryToS3.Records`: 配信レコード数の監視

**CloudWatch Alarmsの設定**:
```bash
# 配信失敗率のアラーム
aws cloudwatch put-metric-alarm \
  --alarm-name firehose-delivery-failure \
  --alarm-description "Firehose delivery failure rate is high" \
  --metric-name DeliveryToS3.Success \
  --namespace AWS/Firehose \
  --statistic Average \
  --period 300 \
  --threshold 95 \
  --comparison-operator LessThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:my-sns-topic
```

**詳細**: [Firehose統合](../04-integrations/02-firehose-integration.md)を参照してください。

## まとめ

S3 Tablesのセキュリティは、複数の層で構成されています：

1. **アイデンティティとアクセス管理**: IAMポリシー、リソースベースポリシー、ABAC、SCPs
2. **データ保護**: SSE-S3/SSE-KMS暗号化、TLS 1.2+
3. **インフラストラクチャ保護**: VPCエンドポイント、Private DNS、Dual-Stack
4. **検出**: CloudTrail管理イベント、データイベント、メンテナンスイベント
5. **インシデント対応**: ログ分析、アラート、自動化

これらのセキュリティ機能を適切に組み合わせることで、S3 Tablesを安全に運用し、AWS Well-Architected Frameworkのセキュリティの柱に準拠したアーキテクチャを実現できます。

### セキュリティチェックリスト

実装時に以下のチェックリストを使用して、セキュリティベストプラクティスが適用されていることを確認してください：

- [ ] 最小権限の原則に基づいてIAMポリシーを設計
- [ ] すべてのTable BucketでSSE-KMSを有効化
- [ ] メンテナンスサービスプリンシパルにKMS権限を付与
- [ ] VPCエンドポイントを使用してネットワークを分離
- [ ] CloudTrail管理イベントが有効（デフォルト）
- [ ] 機密データを含むTableでCloudTrailデータイベントを有効化
- [ ] CloudWatch Logsに統合してリアルタイム監視を実施
- [ ] 重要なイベントに対してCloudWatch Alarmsを設定
- [ ] 一貫したタグ付け戦略を実装
- [ ] ABACを使用してスケーラブルなアクセス制御を実装
- [ ] クロスアカウントアクセスを最小限に抑え、明示的に承認
- [ ] 定期的にアクセスパターンを監査
- [ ] インシデント対応プレイブックを作成
- [ ] CloudTrailログを長期保存（最低90日）
- [ ] Firehose統合時はS3 Tables側でKMS暗号化を設定
- [ ] Firehose IAMロールに最小権限を付与
- [ ] エラーバケットも暗号化を有効化
- [ ] Lambda変換関数でSecrets Managerを使用

## 出典

本ドキュメントは以下のAWS公式ドキュメントを基に作成されました：

1. **Security for S3 Tables**
   - URL: https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-security-overview.html
   - アクセス日: 2025-11-15

2. **Access management for S3 Tables**
   - URL: https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-setting-up.html
   - アクセス日: 2025-11-15

3. **Protecting S3 table data with encryption**
   - URL: https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-encryption.html
   - アクセス日: 2025-11-15

4. **Permission requirements for S3 Tables SSE-KMS encryption**
   - URL: https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-kms-permissions.html
   - アクセス日: 2025-11-15

5. **Logging with AWS CloudTrail for S3 Tables**
   - URL: https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-logging.html
   - アクセス日: 2025-11-15

6. **VPC connectivity for S3 Tables**
   - URL: https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-VPC.html
   - アクセス日: 2025-11-15

7. **Security considerations and limitations for S3 Tables**
   - URL: https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-restrictions.html
   - アクセス日: 2025-11-15

---

**最終更新**: 2025-11-15
**バージョン**: 1.0
