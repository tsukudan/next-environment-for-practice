---
name: aws-well-architected-review
description: '現在のワークロードの IaC とアーキテクチャに対して AWS Well-Architected Framework レビューを実施し、改善点を GitHub Issue として作成する。'
---

# AWS Well-Architected レビュー

このワークフローは、ワークロードの IaC ファイルおよびデプロイ済みインフラに対して、体系的な AWS Well-Architected Framework (WAF) レビューを実施します。6 つの WAF 柱全体にわたるリスクを特定し、改善追跡のための GitHub Issue を作成します。

## 前提条件
- AWS CLI が設定済みで認証されていること
- リポジトリに IaC ファイルが存在すること（Terraform、CloudFormation、CDK、または SAM）
- GitHub MCP サーバーが設定済みで認証されていること

## ワークフローの手順

### ステップ 1: Well-Architected Framework リファレンスの読み込み
現在の AWS WAF ベストプラクティスを取得する:
- `https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html`
- ワークロードの種類（サーバーレス、SaaS など）に関連する柱ごとのレンズ

### ステップ 2: IaC とアーキテクチャの探索
リポジトリ内の IaC ファイルをスキャンする:
- Terraform: `**/*.tf`
- CloudFormation/SAM: `**/*.yaml`, `**/*.json`（CFn テンプレート）
- CDK: `lib/**/*.ts`, `bin/**/*.ts`, `cdk.json`

使用中の主要な AWS サービス（コンピューティング、データ、ネットワーク、セキュリティ、オブザーバビリティ）を特定し、Mermaid アーキテクチャ図を生成する。

### ステップ 3: 柱ごとのレビュー

#### 柱 1: 運用上の優秀性
- [ ] すべてのインフラが IaC で定義されている（コンソールでの手動変更なし）
- [ ] 一貫したタグ付け戦略がすべてのリソースに適用されている
- [ ] 主要メトリクスに対して CloudWatch アラームが定義されている
- [ ] 自動デプロイパイプラインが存在する（手動デプロイなし）
- [ ] 監査ログのために CloudTrail が有効になっている
- [ ] ランブックまたは運用ドキュメントが存在する

#### 柱 2: セキュリティ
- [ ] IAM ロールが最小権限ポリシーを使用している（正当な理由なく `*` アクションを使用しない）
- [ ] IaC またはコードにハードコードされた認証情報がない
- [ ] シークレットが Secrets Manager または SSM Parameter Store で管理されている
- [ ] S3 バケットのパブリックアクセスがブロックされ、サーバーサイド暗号化が有効になっている
- [ ] 機密性の高いリソースがプライベートサブネットに配置されている
- [ ] セキュリティグループの受信が最小限必要なポート/CIDR に制限されている
- [ ] 機密データストアに KMS 暗号化が有効になっている（RDS、EBS、S3、SQS、DynamoDB）
- [ ] すべてのエンドポイントで SSL/TLS が強制されている（`enforceSSL: true`）
- [ ] GuardDuty が有効になっている（`aws guardduty list-detectors`）
- [ ] パブリック向け API および CloudFront ディストリビューションに AWS WAF が設定されている
- [ ] 重要な S3 バケットで MFA 削除が有効になっている

#### 柱 3: 信頼性
- [ ] 本番データベースがマルチ AZ デプロイされている（RDS Multi-AZ、DynamoDB Global Tables）
- [ ] EC2/ECS に適切なポリシーで Auto Scaling が設定されている
- [ ] S3 のバージョニングとライフサイクルポリシーが設定されている
- [ ] 適切な保持期間で RDS の自動バックアップが有効になっている
- [ ] DynamoDB のポイントインタイムリカバリ（PITR）が有効になっている
- [ ] Lambda、SQS、SNS に Dead Letter Queue（DLQ）が設定されている
- [ ] DNS フェイルオーバーのために Route 53 ヘルスチェックが設定されている
- [ ] ノイジーネイバーによるスロットリングを防ぐために Lambda の予約同時実行数が設定されている

#### 柱 4: パフォーマンス効率
- [ ] インスタンスタイプが適切なサイズになっている（Lambda メモリ、EC2 タイプ、RDS クラス）
- [ ] 利用可能な場合に Graviton/ARM インスタンスが使用されている（Lambda `arm64`、EC2 Graviton）
- [ ] キャッシュが実装されている（ElastiCache、DAX、CloudFront、API Gateway キャッシュ）
- [ ] グローバルな静的コンテンツ配信に CloudFront が使用されている
- [ ] 変動する負荷パターンに Aurora Serverless または DynamoDB On-Demand が使用されている
- [ ] レイテンシーが重要な同期パスに Lambda Provisioned Concurrency が使用されている

#### 柱 5: コスト最適化
- [ ] 安定した稼働ワークロードに EC2 リザーブドインスタンスまたは Savings Plans が使用されている
- [ ] より安価なストレージ階層にデータを移動する S3 ライフサイクルポリシーが設定されている
- [ ] Lambda `arm64` アーキテクチャが採用されている（コスト 20% 削減）
- [ ] NAT Gateway 料金を避けるために S3/DynamoDB の VPC エンドポイントが設定されている
- [ ] gp2 EBS ボリュームが gp3 に移行されている（同等のパフォーマンスで 20% 安価）
- [ ] 開発/テスト環境に自動シャットダウンスケジュールが設定されている
- [ ] AWS Budgets とコスト異常検出が設定されている
- [ ] アタッチされていない EBS ボリュームやアイドル状態の EC2 インスタンスが特定されている

#### 柱 6: 持続可能性
- [ ] 利用可能な場合に Graviton/ARM インスタンスが選択されている
- [ ] 常時稼働の EC2 よりサーバーレス/マネージドサービスが優先されている
- [ ] S3 ライフサイクルポリシーで不要な長期データストレージが削減されている
- [ ] 過剰プロビジョニングを避けるために Auto Scaling が設定されている
- [ ] リージョン選択で AWS の再生可能エネルギーへのコミットメントが考慮されている

### ステップ 4: リスク分類
各発見事項について以下のように分類する:
- **高リスク**: セキュリティ脆弱性、単一障害点、バックアップ/リカバリなし
- **中リスク**: 信頼性の低下、コスト非効率、パフォーマンス上の懸念
- **低リスク**: ベストプラクティスからの逸脱、軽微な最適化の機会

### ステップ 5: ユーザー確認

```
🏗️ AWS Well-Architected レビューサマリー

📊 レビュー結果:
• 分析した IaC ファイル数: X
• 特定した AWS サービス数: Y
• 発見事項の合計: Z
  • 高リスク: A（即時対応が必要）
  • 中リスク: B（早急に対処すべき）
  • 低リスク: C（あると望ましい）

🔴 高リスク発見事項のトップ:
1. [柱]: [発見事項] — [重要な理由]
2. [柱]: [発見事項] — [重要な理由]

💡 Z 件の個別 GitHub Issue + 1 件の EPIC Issue を作成します。

❓ GitHub Issue の作成を進めますか？ (y/n)
```

### ステップ 6: 個別の発見事項 Issue を作成する
"well-architected" および柱名（例: "security"、"reliability"）でラベルを付ける。

**タイトル**: `[WAF-<PILLAR>] [発見事項の概要] — [リスクレベル]`

**本文**:
```markdown
## 🏗️ Well-Architected 発見事項: [タイトル概要]

**柱**: [名前] | **リスクレベル**: [高/中/低] | **対応工数**: [低/中/高]

### 📋 説明
[発見事項の明確な説明とその重要性]

### 🔧 改善策

**IaC による修正**（推奨）:
```hcl
# Terraform の例
resource "aws_s3_bucket_server_side_encryption_configuration" "example" {
  bucket = aws_s3_bucket.example.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
  }
}
```

**AWS CLI による代替手段**:
```bash
aws s3api put-bucket-encryption --bucket <name> \
  --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"aws:kms"}}]}'
```

### 📚 AWS リファレンス
- [WAF ベストプラクティスリンク]
- [AWS ドキュメントリンク]

### ✅ 検証
- [ ] IaC に変更を実装してデプロイ済み
- [ ] AWS Config ルールが通過している（該当する場合）
- [ ] Security Hub の発見事項が解決されている（該当する場合）

**Well-Architected 質問**: [この発見事項に対応する WAF の質問]
```

### ステップ 7: EPIC トラッキング Issue を作成する
"well-architected" および "epic" でラベルを付ける。

**タイトル**: `[EPIC] AWS Well-Architected レビュー — 6 つの柱にわたる X 件の発見事項`

**本文**: 柱ごとの内訳テーブル（柱別・リスクレベル別の発見事項数）を含むエグゼクティブサマリー、Mermaid アーキテクチャ図、すべての個別 Issue へのリンクを含む優先度付きチェックリスト（高 → 中 → 低）、および成功基準:
- すべての高リスク発見事項が解決されている
- 中リスクの発見事項に承認済みの緩和計画がある
- 既存の CloudWatch アラームや Config ルールに回帰がない

## エラー処理
- **IaC ファイルが見つからない場合**: AWS CLI でのライブリソース探索に限定し、ギャップを記録する
- **AWS 権限が不十分な場合**: レビューに必要な読み取り専用権限の一覧を表示する
- **GitHub Issue 作成に失敗した場合**: すべての発見事項をフォーマット済みマークダウンとしてコンソールに出力する

## 成功基準
- ✅ IaC およびライブインフラに対してすべての 6 つの WAF 柱がレビューされている
- ✅ すべての発見事項がリスクレベルと柱で分類されている
- ✅ 各発見事項に IaC 例を含む実行可能な改善手順がある
- ✅ チームの追跡のために GitHub Issue が作成されている
- ✅ EPIC のコンテキスト用にアーキテクチャ図が生成されている
- ✅ AWS ドキュメントのリファレンスが含まれている
