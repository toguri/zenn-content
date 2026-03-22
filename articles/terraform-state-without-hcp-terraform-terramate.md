---
title: "脱HCP Terraform — 個人開発のTerraform運用をTerramate + S3 native lockingに移行した話"
emoji: "🏀"
type: "tech"
topics: ["terraform", "terramate", "aws", "個人開発", "iac"]
published: false
---

## TL;DR

- 個人開発（NBAトレード速報サイト）で156リソースのモノリスTerraform stateをTerramateで8スタックに分割した
- HCP Terraform（旧Terraform Cloud）を使わず、S3 backend + native locking（Terraform 1.10+）で十分だった
- DynamoDBロックテーブルも不要になり、インフラ管理のランニングコストがゼロになった

## 背景

NBAのトレード速報やニュースを配信する個人Webアプリ [NBA ISO Flow](https://www.nba-iso-flow.com/) を開発している。フロントエンドはKotlin Compose for Web、バックエンドはRust（Axum + async-graphql）、インフラはAWS上にECS Fargate / Aurora PostgreSQL / CloudFront / MWAAなどを構築している。

Terraform管理のリソースが増えるにつれ、1つのstateに全156リソースが詰まった状態になっていた。`terraform plan` に時間がかかり、CloudFrontの変更がRDSに影響する心配をしながら作業する日々。

ここで選択肢が2つあった。

1. **HCP Terraform（旧Terraform Cloud）** に移行してstate管理を委託する
2. **Terramate** でスタック分割し、S3 backendのまま運用する

## 比較検討

### 選択肢A: HCP Terraform（旧Terraform Cloud）

HashiCorpが提供するマネージドサービス。2023年にTerraform CloudからHCP Terraformに改称された。

**メリット**
- state管理・ロック・実行環境がフルマネージド
- Remote Execution（plan/applyをクラウド上で実行）
- チーム向けのアクセス制御・監査ログ
- VCS連携でPRからplan結果をコメント

**デメリット**
- Free tierは500リソースまで（超えると有料）
- ロックインが強い（backendをHCP Terraformに変更する必要がある）
- 個人開発にはオーバースペック（チーム機能・ガバナンス機能は不要）
- plan/applyの実行速度がローカルより遅い場合がある
- 2024年のHashiCorp IBM買収後のライセンス変更リスク

### 選択肢B: Terramate + S3 native locking

Terramateはオープンソースのスタックオーケストレーションツール。Terraform本体はそのまま使い、スタック分割・コード生成・変更検知を担当する。

**メリット**
- OSSなのでコストゼロ
- S3 backendのままでよい（移行コスト低）
- `use_lockfile = true`（Terraform 1.10+）でDynamoDBロックテーブルが不要に
- スタック間の依存をコード生成で自動管理
- `terramate run`で変更があったスタックだけplan/applyできる
- Terraformのバージョン制約以外のロックインがない

**デメリット**
- スタック分割の設計は自分でやる必要がある
- state移行スクリプトを書く必要がある（`terraform state mv`）
- Remote Executionはない（CI/CDで代替）

## 採用した設計

Terramate + S3 native lockingを採用した。個人開発でHCP Terraformの月額コストを払う理由がなかったのが決め手。

### スタック構成

156リソースを以下の8スタックに分割した:

```
terraform/stacks/prod/
├── network/      # VPC, サブネット, NAT Gateway
├── dns/          # Route53, ACM証明書
├── database/     # Aurora PostgreSQL, Secrets
├── ecs/          # ECS Fargate, ALB, ECR, IAM
├── cdn/          # CloudFront, S3 (frontend)
├── scheduler/    # EventBridge Scheduler
├── mwaa/         # Apache Airflow (MWAA)
└── monitoring/   # CloudWatch Alarms, SNS
```

### Terramate設定

Terramate側の設定は3ファイルだけで済む。

`globals.tm.hcl` でS3 backendの共通設定を定義:

```hcl
globals {
  s3_bucket       = "iso-flow-terraform-state-prod"
  s3_region       = "ap-northeast-1"
  s3_use_lockfile = true
}
```

`generate.tm.hcl` で `_backend.tf` と `_remote.tf` を自動生成:

```hcl
generate_hcl "_backend.tf" {
  content {
    terraform {
      backend "s3" {
        bucket       = global.s3_bucket
        key          = global.s3_key
        region       = global.s3_region
        encrypt      = true
        use_lockfile = global.s3_use_lockfile
      }
    }
  }
}
```

各スタックの `stack.tm.hcl` で、自スタックのstate keyと依存先を宣言:

```hcl
# terraform/stacks/prod/ecs/stack.tm.hcl
stack {
  name        = "iso-flow-prod-ecs"
  description = "Production ECS Fargate, ALB, ECR, IAM"
  tags        = ["iso-flow", "prod", "ecs"]
}

globals {
  s3_key = "prod/ecs/terraform.tfstate"
  remote_states = {
    network  = "prod/network/terraform.tfstate"
    database = "prod/database/terraform.tfstate"
    dns      = "prod/dns/terraform.tfstate"
  }
}
```

これだけで `terramate generate` を実行すると、各スタックに `_backend.tf` と `_remote.tf` が自動生成される。スタック間参照（`terraform_remote_state`）のボイラープレートを手書きしなくていい。

### S3 native locking: DynamoDBが要らなくなった

Terraform 1.10で追加された `use_lockfile = true` が地味に革命的だった。これまでS3 backendでstate lockingするには、DynamoDBテーブルが必須だった:

```hcl
# Before: DynamoDB必須
backend "s3" {
  bucket         = "my-bucket"
  key            = "terraform.tfstate"
  dynamodb_table = "my-lock-table"  # これがないとlock取れない
}
```

```hcl
# After: use_lockfile = true だけでOK
backend "s3" {
  bucket       = "my-bucket"
  key          = "terraform.tfstate"
  use_lockfile = true  # S3のConditional Writesでロック
}
```

DynamoDBの管理が不要になり、個人開発のランニングコストからDynamoDB分がまるごと消えた。

### マイグレーション

既存の1つのstateから8つのスタックへのリソース移動は `terraform state mv` で行った。489行のシェルスクリプトを書いて、Phase 0（バックアップ）〜 Phase 9（クリーンアップ）まで段階的に実行できるようにした。

ポイントは**バックアップを先に取ること**と、**各Phase後に `terraform plan` でzero diffを確認すること**。1回の `state mv` ミスで本番リソースが消えるリスクがあるので、ここは慎重にやった。

### CIの対応

GitHub Actionsの `code-quality.yml` をTerramate対応に更新:

```yaml
- name: Setup Terramate
  uses: terramate-io/terramate-action@v2

- name: Terraform Validate
  run: |
    cd terraform
    terramate generate
    terramate run -- terraform init -backend=false
    terramate run -- terraform validate
```

`terramate run` で全スタックに対してコマンドを一括実行できる。変更があったスタックだけを対象にしたい場合は `terramate run --changed` が使える。

## 結果・学び

**Before**
- 1 state / 156リソース / `terraform plan` に30秒超
- DynamoDBロックテーブルの管理が必要
- CloudFront変更時にRDSのdiffが出て怖い

**After**
- 8 stacks / 最大でも30リソース程度 / `plan` が5秒以下
- DynamoDB不要（S3 native locking）
- スタックごとに独立して変更可能、blast radiusが小さい

**コスト面**
- HCP Terraform: Free tierでも将来的に500リソース制限あり
- S3 native locking: DynamoDBのコストがゼロに（もともと微小だが、心理的に嬉しい）

個人開発のTerraform運用は**Terramate + S3 native lockingで十分**だった。HCP Terraformの機能が必要になるのは、チームでの運用やガバナンスが求められるフェーズ。個人開発では「使わない選択」がベストだと感じた。

## 参考

- [Terramate 公式ドキュメント](https://terramate.io/docs)
- [Terraform 1.10 リリースノート — S3 native state locking](https://www.hashicorp.com/blog/terraform-1-10-improves-handling-of-ephemeral-values)
- [HCP Terraform 料金体系](https://www.hashicorp.com/products/terraform/pricing)
- [Terraform S3 Backend — use_lockfile](https://developer.hashicorp.com/terraform/language/backend/s3)

---
🏀 NBA Trade Tracker: https://www.nba-iso-flow.com/
