# SSE-KMS暗号化によるテーブルデータ保護

## 概要

Amazon S3 Tablesでは、デフォルトでSSE-S3（Amazon S3管理キー）による暗号化が適用されますが、より高度な暗号化制御が必要な場合は、AWS Key Management Service (AWS KMS)を使用したサーバー側暗号化（SSE-KMS）を設定できます。

SSE-KMSを使用することで、以下のメリットが得られます：

- **キーローテーションの管理**: 暗号化キーの自動ローテーション設定
- **アクセスポリシーの詳細制御**: IAMポリシーとKMSキーポリシーによる細かいアクセス制御
- **監査とコンプライアンス**: CloudTrailによる暗号化操作の完全な監査証跡
- **コスト最適化**: テーブルレベルキーによるKMSリクエスト数の削減

## S3 TablesにおけるSSE-KMSの特徴

S3 Tablesの暗号化は、通常のS3バケットとは以下の点で異なります：

### 1. 2つの暗号化レベル

**テーブルバケットレベル暗号化**
- テーブルバケット作成時にデフォルト暗号化設定を指定
- バケット内に作成される全テーブルが自動的に継承
- 後から設定変更可能（新規テーブルのみ適用）

**個別テーブルレベル暗号化**
- テーブル作成時に個別のKMSキーを指定可能
- バケットのデフォルト設定を上書き
- 既存テーブルの暗号化変更は新テーブル作成+データコピーが必要

### 2. カスタマー管理キーのみ対応

S3 Tablesでは、**カスタマー管理キー（Customer Managed Keys）のみ**がサポートされます。AWS管理キーは使用できません。

これにより、以下が可能になります：
- キーポリシーの完全な制御
- キーの有効化/無効化
- キーの削除スケジュール設定
- クロスアカウントアクセスの設定

### 3. テーブルレベルキーによるコスト最適化

S3 Tablesは、各テーブルに対して一意の**テーブルレベルデータキー**を自動生成します。このキーは限られた期間のみ使用され、暗号化操作中のKMSリクエスト数を最小化します。

この仕組みは、S3 Bucket Keysと同様の原理で動作し、以下のメリットがあります：
- KMS APIコールの削減（コスト削減）
- 暗号化操作のパフォーマンス向上
- 自動管理（手動設定不要）

## 必要な権限設定

SSE-KMSを使用する場合、以下の4つのプリンシパルにKMSキーへのアクセス権限を付与する必要があります：

### 1. S3 Maintenance Principal
**目的**: 暗号化されたテーブルのメンテナンス操作（コンパクション、スナップショット管理など）

**プリンシパル**: `s3tables.amazonaws.com`

### 2. S3 Tables Integration Role
**目的**: AWS分析サービス（Athena、Redshift、EMRなど）との連携

**設定方法**: テーブルバケット統合時に作成したIAMロールに権限を付与

### 3. Client Access Role
**目的**: Apache Icebergクライアントからの直接アクセス

**設定方法**: クライアントアプリケーションが使用するIAMロールに権限を付与

### 4. S3 Metadata Principal
**目的**: S3メタデータテーブルの更新

**プリンシパル**: `s3.amazonaws.com`

## 設定方法

### テーブルバケットレベルの暗号化設定

```bash
# テーブルバケット作成時にSSE-KMSを指定
aws s3tables create-table-bucket \
  --name my-encrypted-table-bucket \
  --encryption-configuration '{
    "type": "aws:kms",
    "kmsKeyArn": "arn:aws:kms:us-east-1:111122223333:key/1234abcd-12ab-34cd-56ef-1234567890ab"
  }'

# 既存テーブルバケットの暗号化設定を変更
aws s3tables put-table-bucket-encryption \
  --table-bucket-arn arn:aws:s3tables:us-east-1:111122223333:bucket/my-table-bucket \
  --encryption-configuration '{
    "type": "aws:kms",
    "kmsKeyArn": "arn:aws:kms:us-east-1:111122223333:key/1234abcd-12ab-34cd-56ef-1234567890ab"
  }'

# 暗号化設定の確認
aws s3tables get-table-bucket-encryption \
  --table-bucket-arn arn:aws:s3tables:us-east-1:111122223333:bucket/my-table-bucket
```

### 個別テーブルレベルの暗号化設定

```bash
# テーブル作成時に個別のKMSキーを指定
aws s3tables create-table \
  --table-bucket-arn arn:aws:s3tables:us-east-1:111122223333:bucket/my-table-bucket \
  --namespace my-namespace \
  --name my-encrypted-table \
  --format ICEBERG \
  --encryption-configuration '{
    "type": "aws:kms",
    "kmsKeyArn": "arn:aws:kms:us-east-1:111122223333:key/abcd1234-56ef-78gh-90ij-1234567890ab"
  }'

# テーブルの暗号化設定を確認
aws s3tables get-table-encryption \
  --table-bucket-arn arn:aws:s3tables:us-east-1:111122223333:bucket/my-table-bucket \
  --namespace my-namespace \
  --name my-encrypted-table
```

### KMSキーポリシーの設定例

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Allow S3 Tables maintenance",
      "Effect": "Allow",
      "Principal": {
        "Service": "s3tables.amazonaws.com"
      },
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:SourceAccount": "111122223333"
        }
      }
    },
    {
      "Sid": "Allow S3 metadata updates",
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:SourceAccount": "111122223333"
        }
      }
    }
  ]
}
```

## CloudTrailによる監査

SSE-KMSを使用すると、以下のAPIイベントがCloudTrailに記録されます：

### S3 Tables暗号化設定イベント
- `s3tables:PutTableBucketEncryption` - テーブルバケット暗号化設定
- `s3tables:GetTableBucketEncryption` - テーブルバケット暗号化設定取得
- `s3tables:DeleteTableBucketEncryption` - テーブルバケット暗号化設定削除
- `s3tables:GetTableEncryption` - テーブル暗号化設定取得
- `s3tables:CreateTable` - テーブル作成（暗号化設定含む）
- `s3tables:CreateTableBucket` - テーブルバケット作成（暗号化設定含む）

### KMS暗号化操作イベント
- `kms:GenerateDataKey` - データキー生成
- `kms:Decrypt` - データ復号化

これらのイベントを監視することで、以下が可能になります：
- 暗号化設定の変更履歴追跡
- 不正なアクセス試行の検出
- コンプライアンス要件の証明

## 制限事項と注意点

### 1. AWS管理キー非対応
S3 Tablesでは、カスタマー管理キーのみがサポートされます。`aws/s3`などのAWS管理キーは使用できません。

### 2. 既存テーブルの暗号化変更
既存テーブルの暗号化設定を変更する場合は、以下の手順が必要です：
1. 新しい暗号化設定でテーブルを作成
2. 既存テーブルから新テーブルにデータをコピー
3. 既存テーブルを削除

### 3. クロスリージョン制約
KMSキーとテーブルバケットは同じリージョンに存在する必要があります。

### 4. 権限設定の複雑性
4つのプリンシパルすべてに適切な権限を付与しないと、メンテナンス操作やデータアクセスが失敗する可能性があります。

### 5. コスト考慮
SSE-KMSを使用すると、以下のコストが発生します：
- KMS APIリクエストコスト（テーブルレベルキーにより最小化）
- KMSキーの月額料金

## ベストプラクティス

### 1. キーポリシーの最小権限原則
必要最小限の権限のみを付与し、条件キーを使用してアクセスを制限します。

### 2. キーローテーションの有効化
カスタマー管理キーの自動ローテーションを有効にして、セキュリティを強化します。

```bash
aws kms enable-key-rotation \
  --key-id arn:aws:kms:us-east-1:111122223333:key/1234abcd-12ab-34cd-56ef-1234567890ab
```

### 3. CloudTrailログの監視
暗号化関連のAPIイベントを定期的に監視し、異常なアクセスパターンを検出します。

### 4. キーエイリアスの使用
KMSキーIDの代わりにエイリアスを使用すると、キーの管理が容易になります。

```bash
aws kms create-alias \
  --alias-name alias/s3-tables-encryption \
  --target-key-id 1234abcd-12ab-34cd-56ef-1234567890ab
```

### 5. テーブルバケットレベル暗号化の推奨
個別テーブルごとに異なるキーが必要な場合を除き、テーブルバケットレベルでの暗号化設定を推奨します。

## トラブルシューティング

### 問題: テーブル作成時に暗号化エラーが発生
**原因**: KMSキーへのアクセス権限不足

**解決策**:
1. KMSキーポリシーを確認
2. 4つのプリンシパルすべてに権限が付与されているか確認
3. IAMロールの信頼ポリシーを確認

### 問題: メンテナンス操作が失敗
**原因**: S3 Maintenance Principalへの権限不足

**解決策**:
KMSキーポリシーに以下を追加：
```json
{
  "Effect": "Allow",
  "Principal": {
    "Service": "s3tables.amazonaws.com"
  },
  "Action": [
    "kms:Decrypt",
    "kms:GenerateDataKey"
  ],
  "Resource": "*"
}
```

### 問題: クエリ実行時にアクセス拒否エラー
**原因**: Client Access RoleまたはIntegration Roleへの権限不足

**解決策**:
使用しているIAMロールに以下のKMS権限を追加：
- `kms:Decrypt`
- `kms:DescribeKey`

## 参考リンク

- [AWS公式ドキュメント: Using server-side encryption with AWS KMS keys (SSE-KMS) in table buckets](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-kms-encryption.html)
- [AWS公式ドキュメント: Specifying server-side encryption with AWS KMS keys (SSE-KMS) in table buckets](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-kms-specify.html)
- [AWS公式ドキュメント: Permission requirements for S3 Tables SSE-KMS encryption](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-kms-permissions.html)
- [AWS公式ドキュメント: Enforcing and scoping SSE-KMS use for tables and table buckets](https://docs.aws.amazon.com/AmazonS3/latest/userguide/tables-require-kms.html)
- [AWS KMS開発者ガイド](https://docs.aws.amazon.com/kms/latest/developerguide/)
- [セキュリティベストプラクティス](./01-security-best-practices.md)
- [S3 Tables概要](../01-getting-started/01-overview.md)
