# AWS ECS デプロイパイプライン 構築手順書

Next.js（静的エクスポート）フロントエンドをAWS ECS Fargateにデプロイし、  
GitHub Actionsで自動化するまでの手順と解説をまとめたドキュメントです。

---

## 目次

- [AWS ECS デプロイパイプライン 構築手順書](#aws-ecs-デプロイパイプライン-構築手順書)
  - [目次](#目次)
  - [1. 全体アーキテクチャ](#1-全体アーキテクチャ)
  - [2. 前提条件](#2-前提条件)
    - [AWS CLI SSO 認証設定](#aws-cli-sso-認証設定)
  - [3. Phase 1: AWSリソースの手動構築](#3-phase-1-awsリソースの手動構築)
    - [3-1. ECRリポジトリの作成](#3-1-ecrリポジトリの作成)
      - [ライフサイクルポリシー（コスト管理）](#ライフサイクルポリシーコスト管理)
    - [3-2. IAMロールの作成（タスク実行ロール）](#3-2-iamロールの作成タスク実行ロール)
    - [3-3. VPCとサブネットの作成](#3-3-vpcとサブネットの作成)
      - [サブネット設計](#サブネット設計)
    - [3-4. セキュリティグループの作成](#3-4-セキュリティグループの作成)
      - [sg-alb（ALB用）](#sg-albalb用)
      - [sg-ecs（ECSタスク用）](#sg-ecsecsタスク用)
      - [sg-vpce（VPCエンドポイント用）](#sg-vpcevpcエンドポイント用)
    - [3-5. VPCエンドポイントの作成](#3-5-vpcエンドポイントの作成)
    - [3-6. ALBとターゲットグループの作成](#3-6-albとターゲットグループの作成)
      - [ターゲットグループ](#ターゲットグループ)
      - [ALB](#alb)
    - [3-7. ECSクラスターの作成](#3-7-ecsクラスターの作成)
    - [3-8. ECRへのDockerイメージプッシュ](#3-8-ecrへのdockerイメージプッシュ)
    - [3-9. ECSタスク定義の作成](#3-9-ecsタスク定義の作成)
      - [基本設定](#基本設定)
      - [コンテナ設定](#コンテナ設定)
    - [3-10. ECSサービスの作成](#3-10-ecsサービスの作成)
      - [基本設定](#基本設定-1)
      - [ネットワーキング](#ネットワーキング)
      - [ロードバランシング](#ロードバランシング)
  - [4. Phase 2: GitHub Actionsパイプラインの構築](#4-phase-2-github-actionsパイプラインの構築)
    - [4-1. IAM OIDCプロバイダーの登録](#4-1-iam-oidcプロバイダーの登録)
    - [4-2. GitHub Actions用IAMロールの作成](#4-2-github-actions用iamロールの作成)
      - [信頼されたエンティティの設定](#信頼されたエンティティの設定)
      - [許可ポリシー](#許可ポリシー)
      - [ロール名](#ロール名)
    - [4-3. ワークフローファイルの作成](#4-3-ワークフローファイルの作成)
      - [ワークフローの各ステップの役割](#ワークフローの各ステップの役割)
  - [5. トラブルシューティング](#5-トラブルシューティング)
    - [ECSタスクが起動しない](#ecsタスクが起動しない)
    - [GitHub Actions で `Not authorized to perform sts:AssumeRoleWithWebIdentity`](#github-actions-で-not-authorized-to-perform-stsassumerolewithwebidentity)
    - [ECSクラスター作成時に `Unable to assume the service linked role` エラー](#ecsクラスター作成時に-unable-to-assume-the-service-linked-role-エラー)
    - [ALBのDNS名にアクセスできない](#albのdns名にアクセスできない)

---

## 1. 全体アーキテクチャ

```
GitHub Push / 手動実行
        │
        ▼
┌───────────────────────────────────────┐
│         GitHub Actions                │
│  1. docker build                      │
│  2. docker push → ECR                 │
│  3. ECSタスク定義更新 → サービス更新  │
└───────────────────────────────────────┘
                │
                ▼
┌──────────────────────────────────────────────────────┐
│  AWS (ap-northeast-1)                                 │
│                                                       │
│  ECR: next-practice/frontend                         │
│    └─ コミットSHAタグでイメージを管理                 │
│                                                       │
│  VPC: next-practice-vpc (10.0.0.0/16)                │
│  ├─ Public Subnet (10.0.1.0/24, 10.0.2.0/24)        │
│  │    └─ ALB (インターネット向け)                     │
│  └─ Private Subnet (10.0.11.0/24, 10.0.12.0/24)     │
│       └─ ECS Fargate Task                            │
│            └─ nginx コンテナ (Next.js静的ファイル)   │
│                                                       │
│  VPCエンドポイント経由でECR・CloudWatch Logsに接続   │
│  （NATゲートウェイ不要）                              │
└──────────────────────────────────────────────────────┘
```

---

## 2. 前提条件

- AWSアカウントを持っていること
- GitHubリポジトリにコードがpushされていること
- ローカルにDocker・AWS CLIがインストールされていること（初回イメージプッシュ時のみ必要）
- AWS CLIの認証設定済み（AWS SSO推奨）

### AWS CLI SSO 認証設定

```bash
aws configure sso
# SSO session name: next-practice
# SSO start URL: <IAM Identity CenterのアクセスポータルURL>
# SSO region: ap-northeast-1

# ログイン
aws sso login --profile next-practice

# 確認
aws sts get-caller-identity --profile next-practice
```

---

## 3. Phase 1: AWSリソースの手動構築

### 3-1. ECRリポジトリの作成

**目的**: DockerイメージをAWS内のプライベートレジストリに保存する場所を作る。

| 項目 | 値 |
|---|---|
| リポジトリ名 | `next-practice/frontend` |
| 可視性 | プライベート |
| タグのイミュータビリティ | **有効（Immutable）** |

> **タグのイミュータビリティを有効にする理由**  
> 同じタグ名のイメージを上書きできないようにする。CI/CDではコミットSHAをタグに使うため、同じSHAで異なるイメージが作られる事故を防ぐ。

#### ライフサイクルポリシー（コスト管理）

デプロイのたびに新しいイメージが蓄積されるため、古いイメージを自動削除するポリシーを設定する。

ECR → `next-practice/frontend` → 「ライフサイクルポリシー」→ JSONで以下を設定：

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep only the 10 most recent images",
      "selection": {
        "tagStatus": "any",
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

---

### 3-2. IAMロールの作成（タスク実行ロール）

**目的**: ECSがECRからイメージをPullし、CloudWatch Logsにログを書き込むための権限を付与する。

> ECSには2種類のIAMロールが存在する。
> - **タスク実行ロール**: ECS制御プレーンが使う（ECR Pull、Logsへの書き込み）
> - **タスクロール**: コンテナ内アプリが使う（S3・Secrets Managerなどへのアクセス）
>
> 今回は静的ファイル配信のみなのでタスクロールは不要。

**作成手順**:

- IAM → ロール → 「ロールを作成」
- エンティティ: AWSのサービス → ユースケース: `Elastic Container Service Task`
- アタッチするポリシー: `AmazonECSTaskExecutionRolePolicy`
- ロール名: `ecsTaskExecutionRole`

**`AmazonECSTaskExecutionRolePolicy`が含む主な権限**:
- `ecr:GetAuthorizationToken` — ECRへのログイン
- `ecr:BatchGetImage` — イメージのPull
- `logs:CreateLogStream` / `logs:PutLogEvents` — CloudWatch Logsへの書き込み

---

### 3-3. VPCとサブネットの作成

**目的**: AWSリソースを配置するネットワーク基盤を作る。

#### サブネット設計

```
VPC: next-practice-vpc (10.0.0.0/16)
├── Public Subnet (AZ-a)  10.0.1.0/24  ← ALBを置く
├── Public Subnet (AZ-c)  10.0.2.0/24  ← ALBの冗長化用（ALBは2AZ必須）
├── Private Subnet (AZ-a) 10.0.11.0/24 ← ECS Fargateを置く
└── Private Subnet (AZ-c) 10.0.12.0/24 ← ECS Fargateの冗長化用
```

> **PublicとPrivateを分ける理由**  
> ECSコンテナを直接インターネットに露出させないため。外部リクエストは必ずALBを経由させる。

**VPCウィザードで一括作成**:

VPC → 「VPCを作成」→「VPCなど」を選択

| 項目 | 値 |
|---|---|
| 名前タグ | `next-practice` |
| IPv4 CIDR | `10.0.0.0/16` |
| AZ数 | 2 |
| パブリックサブネット数 | 2 |
| プライベートサブネット数 | 2 |
| NATゲートウェイ | **なし** |
| VPCエンドポイント | なし |

> **NATゲートウェイを使わない理由**  
> 月額$30〜の費用がかかるため。代わりにVPCエンドポイントを使い、ECR・CloudWatch Logsへの通信をAWS内部で完結させる。

**VPCの追加設定（必須）**:

作成後、VPCの設定で以下を有効化する:
- **DNS解決を有効化**: VPC内でDNSクエリを使えるようにする
- **DNSホスト名を有効化**: VPCエンドポイントのDNS名をプライベートIPに解決できるようにする

> これが無効だとVPCエンドポイントのDNS名がプライベートIPに解決されず、ECSがECRに到達できない。

---

### 3-4. セキュリティグループの作成

**目的**: リソース単位で「誰が誰と通信できるか」を制御する。

```
インターネット
    │ 80/443
    ▼
[ sg-alb ]   ← ALB用：外部HTTPを受け付ける
    │ 80
    ▼
[ sg-ecs ]   ← ECSタスク用：ALBからのみ受け付ける
    │ 443
    ▼
[ sg-vpce ]  ← VPCエンドポイント用：ECSからのHTTPSを受け付ける
    │
    ▼
  ECR / CloudWatch Logs（AWS内部）
```

#### sg-alb（ALB用）

| 項目 | 値 |
|---|---|
| 名前 | `next-practice-sg-alb` |
| VPC | `next-practice-vpc` |
| インバウンド | TCP 80, ソース: `0.0.0.0/0` |
| アウトバウンド | デフォルト（全許可） |
| 説明 | `Allow HTTP(80) inbound from the internet for ALB` |

#### sg-ecs（ECSタスク用）

| 項目 | 値 |
|---|---|
| 名前 | `next-practice-sg-ecs` |
| VPC | `next-practice-vpc` |
| インバウンド | TCP 80, ソース: **`next-practice-sg-alb`** |
| アウトバウンド | デフォルト（全許可） |
| 説明 | `Allow HTTP(80) inbound from ALB only for ECS tasks` |

> **ソースにIPではなく別のSGを指定する理由**  
> 「ALBのSGを持つリソースからのみ許可」という意味になり、ALBのIPが変わっても自動追従する。

#### sg-vpce（VPCエンドポイント用）

| 項目 | 値 |
|---|---|
| 名前 | `next-practice-sg-vpce` |
| VPC | `next-practice-vpc` |
| インバウンド | TCP 443, ソース: **`next-practice-sg-ecs`** |
| アウトバウンド | デフォルト（全許可） |
| 説明 | `Allow HTTPS(443) inbound from ECS tasks for VPC endpoints` |

---

### 3-5. VPCエンドポイントの作成

**目的**: Private SubnetのECSがNATゲートウェイなしでECR・CloudWatch Logsにアクセスできるようにする。

> ECSがECRからイメージをPullする際の通信経路:
> 1. `ecr.api` — ECRへの認証・API操作（AWSのコントロールプレーン）
> 2. `ecr.dkr` — Dockerイメージのマニフェスト取得（Docker標準仕様）
> 3. `s3` — イメージレイヤーの実体データ取得（ECRはレイヤーをS3に保存している）
> 4. `logs` — CloudWatch Logsへのログ送信

| エンドポイント名 | サービス | 種別 | 費用 |
|---|---|---|---|
| `next-practice-vpce-ecr-api` | `com.amazonaws.ap-northeast-1.ecr.api` | Interface | 有料 |
| `next-practice-vpce-ecr-dkr` | `com.amazonaws.ap-northeast-1.ecr.dkr` | Interface | 有料 |
| `next-practice-vpce-logs` | `com.amazonaws.ap-northeast-1.logs` | Interface | 有料 |
| `next-practice-vpce-s3` | `com.amazonaws.ap-northeast-1.s3` | **Gateway** | **無料** |

**Interface型の共通設定**:
- サブネット: プライベートサブネット2つ
- セキュリティグループ: `next-practice-sg-vpce`

**Gateway型（S3）の設定**:
- ルートテーブル: プライベートサブネットのルートテーブル2つを選択
- セキュリティグループ: 不要

> **InterfaceとGatewayの違い**  
> - Interface: VPC内にENI（仮想NIC）を作成し、プライベートIPで接続
> - Gateway: ルートテーブルにエントリを追加して転送。無料

---

### 3-6. ALBとターゲットグループの作成

**目的**: インターネットからのHTTPリクエストを受け取り、Private SubnetのECSタスクに転送する。

> ALBがないとECSタスクのIPが再起動のたびに変わり、固定のURLでアクセスできない。ALBが固定のDNS名を提供し、複数タスクへの負荷分散も担う。

#### ターゲットグループ

EC2 → ターゲットグループ → 「ターゲットグループの作成」

| 項目 | 値 |
|---|---|
| ターゲットタイプ | **IPアドレス** |
| 名前 | `next-practice-tg` |
| プロトコル | HTTP / ポート80 |
| プロトコルバージョン | HTTP1 |
| VPC | `next-practice-vpc` |
| ヘルスチェックパス | `/` |

> **ターゲットタイプを「IPアドレス」にする理由**  
> Fargateタスクはインスタンス上に乗らず、直接ENIにIPが割り当てられる。「インスタンス」タイプではFargateのタスクを登録できない。
>
> ターゲットは手動登録不要。ECSサービスがタスクの起動・停止に合わせて自動で登録・解除する。

#### ALB

EC2 → ロードバランサー → 「ロードバランサーの作成」→ Application Load Balancer

| 項目 | 値 |
|---|---|
| 名前 | `next-practice-alb` |
| スキーム | **インターネット向け** |
| VPC | `next-practice-vpc` |
| サブネット | パブリックサブネット2つ |
| セキュリティグループ | `next-practice-sg-alb` |
| リスナー | HTTP:80 → `next-practice-tg` に転送 |

---

### 3-7. ECSクラスターの作成

**目的**: ECSタスク・サービスをまとめる管理単位を作る。

**初回のみ**: ECSサービスリンクロール (`AWSServiceRoleForECS`) が存在しない場合、クラスター作成が失敗する。IAMでロールを確認し、存在しない場合は作成する。CloudFormationスタックが残存している場合は先に削除が必要。

| 項目 | 値 |
|---|---|
| クラスター名 | `next-practice-cluster` |
| インフラストラクチャ | **Fargateのみ** |
| Container Insights | **オン** |

---

### 3-8. ECRへのDockerイメージプッシュ

初回のみローカルから手動プッシュして動作確認する。

```bash
# ECRにDockerを認証
aws ecr get-login-password --region ap-northeast-1 --profile next-practice \
  | docker login --username AWS --password-stdin \
    156428223749.dkr.ecr.ap-northeast-1.amazonaws.com

# イメージをビルド（プロジェクトルートから実行）
docker build \
  -f docker/nginx/Dockerfile \
  -t next-practice/frontend:latest \
  .

# ECR用タグを付ける
docker tag next-practice/frontend:latest \
  156428223749.dkr.ecr.ap-northeast-1.amazonaws.com/next-practice/frontend:latest

# ECRにプッシュ
docker push \
  156428223749.dkr.ecr.ap-northeast-1.amazonaws.com/next-practice/frontend:latest
```

> **`docker build` をプロジェクトルートから実行する理由**  
> Dockerfileの中で `COPY . .` や `COPY docker/nginx/nginx.conf` のようにプロジェクトルート基準のパスを参照しているため。

> **プッシュ時に `-` タグのイメージが作られる理由**  
> Docker BuildKitが自動生成するアテステーション（SBOM・Provenance）。ECSの動作に影響はない。抑制したい場合は `--provenance=false --sbom=false` を付ける。

---

### 3-9. ECSタスク定義の作成

**目的**: ECSに「どのイメージをどのスペックで動かすか」を定義する設計書を作る。

ECS → タスク定義 → 「新しいタスク定義の作成」

#### 基本設定

| 項目 | 値 |
|---|---|
| タスク定義ファミリー名 | `next-practice-frontend` |
| 起動タイプ | Fargate |
| OS/アーキテクチャ | Linux/X86_64 |
| CPU | `0.25 vCPU` |
| メモリ | `0.5 GB` |
| タスクロール | なし |
| タスク実行ロール | `ecsTaskExecutionRole` |

#### コンテナ設定

| 項目 | 値 |
|---|---|
| 名前 | `frontend` |
| イメージURI | `156428223749.dkr.ecr.ap-northeast-1.amazonaws.com/next-practice/frontend:latest` |
| コンテナポート | 80 (TCP, HTTP) |

ログ収集は「使用する」にチェックを入れる（CloudWatch Logsへの自動転送）。

---

### 3-10. ECSサービスの作成

**目的**: タスク定義を元にコンテナを起動し、常時稼働させる。ALBと連携してトラフィックを受け付ける。

> **タスクとサービスの違い**
> - タスク: コンテナの1回の実行インスタンス
> - サービス: 「タスクをN個常時維持する」管理者。タスクが落ちたら自動再起動し、ALBへの登録・解除も担う

ECS → `next-practice-cluster` → 「サービス」→「作成」

#### 基本設定

| 項目 | 値 |
|---|---|
| コンピューティング | キャパシティプロバイダー戦略（Fargate 100%） |
| タスク定義 | `next-practice-frontend` (最新リビジョン) |
| サービス名 | `next-practice-frontend-svc` |
| 必要なタスク数 | `1` |

#### ネットワーキング

| 項目 | 値 |
|---|---|
| VPC | `next-practice-vpc` |
| サブネット | プライベートサブネット2つ |
| セキュリティグループ | `next-practice-sg-ecs` |
| パブリックIP | **オフ** |

#### ロードバランシング

| 項目 | 値 |
|---|---|
| ロードバランサー | `next-practice-alb` |
| コンテナ | `frontend 80:80` |
| リスナー | 既存: ポート80 |
| ターゲットグループ | 既存: `next-practice-tg` |

**動作確認**: `http://next-practice-alb-xxxx.ap-northeast-1.elb.amazonaws.com` にアクセスしてNext.jsのページが表示されること。
（`https://` ではなく `http://` を明示すること）

---

## 4. Phase 2: GitHub Actionsパイプラインの構築

### 4-1. IAM OIDCプロバイダーの登録

**目的**: AWSがGitHubからの認証リクエストを信頼できるよう、GitHubのOIDCプロバイダー情報をAWSに登録する。

IAM → 「IDプロバイダー」→「プロバイダーを追加」

| 項目 | 値 |
|---|---|
| プロバイダーのタイプ | OpenID Connect |
| プロバイダーのURL | `https://token.actions.githubusercontent.com` |
| 対象者（Audience） | `sts.amazonaws.com` |

「サムプリントを取得」をクリックしてから「プロバイダーを追加」。

> **OIDCの認証フロー**
> ```
> GitHub Actions
>   → GitHub OIDCエンドポイントにトークンをリクエスト
>     （「sts.amazonaws.com向けのトークンをください」）
>   → JWTトークンを取得
>     { "aud": "sts.amazonaws.com", "sub": "repo:org/repo:ref:refs/heads/main" }
>   → AWS STSにAssumeRoleWithWebIdentityを呼び出し
>   → 条件が一致すれば一時的な認証情報を発行
> ```

---

### 4-2. GitHub Actions用IAMロールの作成

**目的**: GitHub Actionsがロールを引き受けることでAWSの一時的な認証情報を取得できるロールを作る。

IAM → ロール → 「ロールを作成」

#### 信頼されたエンティティの設定

| 項目 | 値 |
|---|---|
| エンティティタイプ | **ウェブアイデンティティ** |
| IDプロバイダー | `token.actions.githubusercontent.com` |
| Audience | `sts.amazonaws.com` |
| GitHubリポジトリ | `next-environment-for-practice` |

信頼ポリシー（生成されるJSON）:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::156428223749:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:tsukudan@99854994/next-environment-for-practice@1311799371:*"
        }
      }
    }
  ]
}
```

> **`sub` にユーザーID・リポジトリIDが含まれる点に注意**  
> GitHubは `sub` クレームに数値IDを付加した形式を使う:  
> `repo:USERNAME@USERID/REPONAME@REPOID:ref:refs/heads/BRANCH`  
> CloudTrailの `userIdentity.userName` で実際の値を確認して `StringLike` に設定すること。

#### 許可ポリシー

| ポリシー名 | 用途 |
|---|---|
| `AmazonEC2ContainerRegistryFullAccess` | ECRへのイメージプッシュ |
| `AmazonECS_FullAccess` | ECSサービスの更新 |

#### ロール名

`GitHubActions-next-practice-deploy`

---

### 4-3. ワークフローファイルの作成

ファイルパス: `.github/workflows/ecs-deploy-frontend.yml`

```yaml
name: Deploy to ECS

on:
  workflow_dispatch:  # GitHubのActionsタブから手動実行

permissions:
  id-token: write  # OIDCトークンを発行する権限
  contents: read   # リポジトリのコードをチェックアウトする権限

env:
  AWS_REGION: ap-northeast-1
  ECR_REPOSITORY: next-practice/frontend
  ECS_CLUSTER: next-practice-cluster
  ECS_SERVICE: next-practice-frontend-svc
  TASK_DEFINITION_FAMILY: next-practice-frontend
  CONTAINER_NAME: frontend

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::156428223749:role/GitHubActions-next-practice-deploy
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, and push image to ECR
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build \
            --provenance=false \
            --sbom=false \
            -f docker/nginx/Dockerfile \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
            .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT

      - name: Download current task definition
        run: |
          aws ecs describe-task-definition \
            --task-definition ${{ env.TASK_DEFINITION_FAMILY }} \
            --query taskDefinition \
            > task-definition.json

      - name: Render new task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-definition.json
          container-name: ${{ env.CONTAINER_NAME }}
          image: ${{ steps.build-image.outputs.image }}

      - name: Deploy to ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v2
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
```

#### ワークフローの各ステップの役割

| ステップ | 役割 |
|---|---|
| Checkout | リポジトリのコードをGitHub Actionsのランナーに取得 |
| Configure AWS credentials | OIDCでAWSの一時的な認証情報を取得（アクセスキー不要） |
| Login to Amazon ECR | DockerをECRに認証 |
| Build, tag, and push | コミットSHAタグでイメージをビルド・プッシュ |
| Download task definition | 現在のタスク定義をAWSから取得（ローカルにハードコードしない） |
| Render task definition | タスク定義のイメージURIを新しいコミットSHAに書き換え |
| Deploy to ECS | 新しいタスク定義を登録してサービスを更新。完了まで待機 |

> **コミットSHAタグを使う理由**  
> `latest` タグを上書きし続けると「今ECSで動いているのはどのコードか」が追跡できなくなる。コミットSHAをタグにすることでGitHubのコミット履歴とECRのイメージを1対1で対応づけられる。

---

## 5. トラブルシューティング

### ECSタスクが起動しない

1. ECS → クラスター → サービス → 「タスク」タブでタスクのステータスを確認
2. 停止しているタスクをクリック → 「ログ」タブでエラー内容を確認
3. よくある原因:
   - ECRへの接続失敗 → VPCエンドポイントの設定を確認
   - イメージが見つからない → タスク定義のイメージURIを確認
   - ヘルスチェック失敗 → ターゲットグループのヘルスチェック設定を確認

### GitHub Actions で `Not authorized to perform sts:AssumeRoleWithWebIdentity`

1. CloudTrail → イベント履歴 → `AssumeRoleWithWebIdentity` イベントを確認
2. `userIdentity.userName` で実際の `sub` クレームの値を確認
3. IAMロールの信頼ポリシーの `StringLike` 条件と一致しているか確認
4. GitHubの `sub` クレームには数値IDが含まれる形式に注意:
   ```
   repo:USERNAME@USERID/REPONAME@REPOID:ref:refs/heads/BRANCH
   ```

### ECSクラスター作成時に `Unable to assume the service linked role` エラー

IAM → ロール で `AWSServiceRoleForECS` の存在を確認。  
存在しない場合: IAM → ロール → 「ロールを作成」→ AWSのサービス → `Elastic Container Service` を選択して作成。  
CloudFormationスタックが残存している場合はCloudFormationコンソールから削除してから再試行。

### ALBのDNS名にアクセスできない

`http://` を明示してアクセスする。多くのブラウザは自動的に `https://` に変換するため、HTTP(80番ポート)のみのALBにはアクセスできない場合がある。
