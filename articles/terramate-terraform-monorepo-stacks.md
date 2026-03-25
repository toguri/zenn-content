---
title: "Terramate導入でTerraformモノレポの苦しみから解放された話 — スタック分割・変更検知・DRY化の実践"
emoji: "🏗️"
type: "tech"
topics: ["terraform", "terramate", "aws", "devops", "iac"]
published: false
---

## TL;DR

- Terraformモノレポが肥大化して `plan` に10分以上、レビューも困難になっていた
- Terramateを導入してスタック分割 + Git変更検知で **差分があるスタックだけ** plan/applyする運用に移行
- `globals` と `generate_hcl` でbackend/provider設定をDRY化し、環境ごとのコピペ地獄から脱出した

## 背景

あるプロジェクトでAWSインフラをTerraformで管理していた。最初は1つの `main.tf` から始まり、リソースが増えるたびにファイルを分けて…という典型的な成長パターンだった。

半年もすると以下の問題が顕在化してきた:

- **stateファイルの肥大化**: `terraform plan` が10分超え。CIのジョブタイムアウトに引っかかる
- **blast radiusの拡大**: VPCの設定変更がなぜかLambdaのdiffに影響する（state内の依存関係）
- **環境間のコピペ**: `dev/` `stg/` `prod/` で同じ `backend.tf` と `provider.tf` を3回書く。片方だけ更新し忘れる事故が定期的に発生
- **レビューの困難さ**: PRで `terraform plan` の出力が数百行。何が変わるのか把握できない

「ディレクトリ分割すれば済む」と思ってやってみたが、分割しただけではコピペ問題は解決しなかった。

## 比較検討

### 選択肢A: Terragrunt

Terraform界隈で最も有名なラッパーツール。DRY化のために `terragrunt.hcl` で設定を共通化し、`include` で継承する仕組み。

**メリット:**
- 枯れていて情報が多い
- `include` / `dependency` で依存関係を表現できる
- `generate` ブロックでbackend設定を自動生成

**デメリット:**
- HCLの上にさらにHCLを被せる設計で、デバッグ時に「どの層の問題か」がわかりにくい
- Terraform本体のアップデートとの互換性問題が時々発生する
- 変更検知の仕組みが組み込まれていない（CI側で自前実装が必要）

### 選択肢B: Terramate

比較的新しいIaCオーケストレーションツール。スタックという概念でTerraformコードを分割し、Gitベースの変更検知で差分デプロイを実現する。

**メリット:**
- Gitの差分をベースにした変更検知が組み込み（ `terramate list --changed` ）
- `generate_hcl` でTerraformネイティブなコードを生成するため、生成結果が `.tf` ファイルとしてそのまま読める
- Terraform本体に依存しない薄いオーケストレーション層
- `globals` による変数の階層的な継承

**デメリット:**
- Terragruntと比べるとまだ情報が少ない
- Terramate Cloud（SaaS）と連携すると強力だが、CLIだけだとダッシュボード機能は使えない

### 判断基準

最終的に以下の理由でTerramateを選んだ:

1. **変更検知がビルトイン**: CI側の自前実装が不要。これが一番大きかった
2. **生成コードの透明性**: `generate_hcl` の出力が普通の `.tf` ファイルなので、Terramate抜きでも `terraform plan` が動く
3. **学習コストの低さ**: Terraform HCLを知っていればすぐ使える。独自のDSLを覚える必要がほぼない

## 採用した設計

### ディレクトリ構成

```
infra/
├── terramate.tm.hcl          # ルート設定
├── globals.tm.hcl             # 共通globals
├── stacks/
│   ├── network/
│   │   ├── stack.tm.hcl       # スタック定義
│   │   ├── generate.tm.hcl   # backend/provider生成
│   │   ├── main.tf
│   │   └── variables.tf
│   ├── compute/
│   │   ├── stack.tm.hcl
│   │   ├── generate.tm.hcl
│   │   ├── main.tf
│   │   └── variables.tf
│   └── monitoring/
│       ├── stack.tm.hcl
│       ├── generate.tm.hcl
│       ├── main.tf
│       └── variables.tf
└── envs/
    ├── dev/
    │   └── globals.tm.hcl     # dev固有のglobals
    ├── stg/
    │   └── globals.tm.hcl
    └── prod/
        └── globals.tm.hcl
```

### globalsによる階層的な変数管理

ルートの `globals.tm.hcl` で共通値を定義し、環境ごとにオーバーライドする:

```hcl
# infra/globals.tm.hcl（ルート）
globals {
  project_name = "myproject"
  aws_region   = "ap-northeast-1"

  # タグの共通設定
  common_tags = {
    ManagedBy = "terraform"
    Project   = global.project_name
  }
}
```

```hcl
# infra/envs/dev/globals.tm.hcl
globals {
  environment    = "dev"
  account_id     = "123456789012"
  instance_type  = "t3.small"
}
```

```hcl
# infra/envs/prod/globals.tm.hcl
globals {
  environment    = "prod"
  account_id     = "987654321098"
  instance_type  = "m5.large"
}
```

### generate_hclでbackend/provider設定をDRY化

各スタックの `generate.tm.hcl` で、globalsを参照してbackendとproviderを自動生成する:

```hcl
# stacks/network/generate.tm.hcl
generate_hcl "_backend.tf" {
  content {
    terraform {
      backend "s3" {
        bucket         = "tfstate-${global.account_id}"
        key            = "stacks/${terramate.stack.path.relative}/terraform.tfstate"
        region         = global.aws_region
        dynamodb_table = "terraform-lock"
        encrypt        = true
      }
    }
  }
}

generate_hcl "_provider.tf" {
  content {
    provider "aws" {
      region = global.aws_region

      default_tags {
        tags = merge(global.common_tags, {
          Environment = global.environment
          Stack       = terramate.stack.name
        })
      }
    }
  }
}
```

`terramate generate` を実行すると、各スタックに `_backend.tf` と `_provider.tf` が生成される。生成ファイルは `.gitignore` に入れず、コミットしてレビュー対象にした。「何が生成されたか」が明示的にわかるようにするためだ。

### CIでの変更検知

GitHub Actionsでの運用例:

```yaml
# .github/workflows/terraform-plan.yml
jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # 全履歴が必要（Git diffのため）

      - name: Install Terramate
        uses: terramate-io/terramate-action@v2

      - name: List changed stacks
        id: changed
        run: |
          terramate list --changed > changed_stacks.txt
          echo "stacks=$(cat changed_stacks.txt | tr '\n' ',')" >> $GITHUB_OUTPUT

      - name: Run terraform plan on changed stacks
        if: steps.changed.outputs.stacks != ''
        run: |
          terramate run --changed -- terraform init -no-color
          terramate run --changed -- terraform plan -no-color
```

`--changed` フラグがポイントで、mainブランチとの差分があるスタックだけを対象にしてくれる。50個スタックがあっても、変更があった2〜3個だけplanが走る。CIの実行時間が劇的に短くなった。

## 結果・学び

### Before → After

| 指標 | Before | After |
|------|--------|-------|
| `terraform plan` 時間（CI） | 10〜15分 | 1〜3分（変更スタックのみ） |
| PRのplan出力行数 | 300〜500行 | 30〜50行 |
| backend.tf のコピペミス | 月1〜2回 | 0回 |
| 新環境追加の工数 | 半日〜1日 | 30分 |

### 学び

**1. スタックの粒度は「チームの認知単位」に合わせる**

最初はリソースタイプごと（VPC、EC2、RDS…）に細かく分けすぎた。結果、依存関係が複雑になりすぎて `terraform apply` の順序管理が面倒になった。最終的には「ネットワーク」「コンピュート」「監視」のような、チームメンバーが直感的に理解できる単位に落ち着いた。

**2. 生成ファイルはコミットする派**

`.gitignore` に入れてCI上でだけ生成する運用もあるが、PRで差分が見えないのが不安だった。生成ファイルをコミットすることで「Terramateの設定変更が何を生むか」がレビューで可視化できる。

**3. Terragruntからの移行は段階的にやる**

もしTerragruntを使っている場合、一気に全部移行する必要はない。1つのスタックから試して、チームが慣れてから横展開するのがおすすめ。Terramateは既存のTerraformコードをそのまま使えるので、移行のハードルは低い。

## 参考

- [Terramate公式ドキュメント](https://terramate.io/docs/)
- [Terramate GitHub](https://github.com/terramate-io/terramate)
- [Terramate vs Terragrunt: A 2026 Comparison](https://terramate.io/rethinking-iac/terramate-vs-terragrunt-a-2026-comparison/)
- [Terraform Monorepo: Structure, Benefits & Best Practices](https://spacelift.io/blog/terraform-monorepo)
