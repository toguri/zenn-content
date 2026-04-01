---
title: "個人開発でAWS MWAAをやめたら年間$4,200浮いた — EventBridge Schedulerへの移行記録"
emoji: "💸"
type: "tech"
topics: ["aws", "terraform", "airflow", "個人開発", "コスト最適化"]
published: false
---

## TL;DR

- 個人開発のNBAニュースサイトでApache Airflow（AWS MWAA）を使っていたが、月額$350+のコストが重すぎた
- やっていたのは「定期的にECSタスクを起動する」だけだったので、EventBridge Schedulerに置き換えた
- **月額$350+ → 実質$0**（無料枠内）で年間約$4,200の削減に成功

## 背景

[NBA ISO FLOW](https://www.nba-iso-flow.com/) というNBAの最新ニュースを集約するWebアプリを個人開発しています。バックエンドはRust（Axum + async-graphql）、フロントエンドはKotlin Compose for Web、インフラはAWS上にTerraformで構築しています。

このアプリでは、複数のRSSフィードからNBAニュースを定期的にスクレイピングし、日本語に翻訳して配信しています。このスケジューリング基盤として**AWS MWAA（Managed Workflows for Apache Airflow）**を採用していました。

### なぜAirflowを選んだのか

当初の設計では、以下のような複雑なワークフローを想定していました：

- ヘルスチェック → スクレイピング → 翻訳 → 通知という依存関係のあるDAG
- 失敗時のリトライ・分岐ロジック
- DAGの実行状況モニタリング

しかし実際に運用してみると、**やっていることは2つだけ**でした：

1. **毎時**: ECSタスクを起動してRSSフィードをスクレイピング
2. **毎時15分**: GraphQL mutationを叩いて翻訳を実行

Airflowのフルスペックは完全にオーバーキルだったのです。

## 比較検討

### 選択肢A: AWS MWAA（現状維持）

```
MWAA (mw1.small) = ~$0.49/時間 × 24h × 30日 = 約$353/月
+ Worker費用、S3、CloudWatch Logs
= 合計 月$350〜400+
```

**メリット:**
- 複雑なDAGの依存関係を定義可能
- Airflow UIでの実行モニタリング
- 豊富なOperator/Provider

**デメリット:**
- 最小構成でも月$350+（個人開発には致命的）
- 環境の起動/削除に20〜50分かかる
- 2つのシンプルなジョブには過剰

### 選択肢B: Amazon EventBridge Scheduler

```
EventBridge Scheduler = 無料枠 月14,000,000回
実際の呼び出し: ~1,440回/月（無料枠の0.01%以下）
= 合計 実質$0/月
```

**メリット:**
- 無料枠が巨大すぎて課金されることがほぼない
- ECSタスクを直接起動できる（`ecs_parameters`対応）
- セットアップが圧倒的にシンプル
- Terraformで10分で構築可能

**デメリット:**
- 複雑なワークフロー依存関係は表現できない
- モニタリングはCloudWatchに委ねる
- UIダッシュボードはない

## 採用した設計

EventBridge Schedulerを採用しました。Terraformモジュールはこのように書いています：

```hcl
# RSSスクレイピング: 毎時ECSタスクを起動
resource "aws_scheduler_schedule" "rss_scraping" {
  name                = "${local.name}-rss-scraping"
  schedule_expression = "rate(1 hour)"

  target {
    arn      = var.ecs_cluster_arn
    role_arn = aws_iam_role.scheduler.arn

    ecs_parameters {
      task_definition_arn = var.ecs_task_definition_arn
      launch_type         = "FARGATE"

      network_configuration {
        subnets          = var.private_subnets
        security_groups  = [var.ecs_security_group_id]
        assign_public_ip = false
      }
    }

    input = jsonencode({
      containerOverrides = [{
        name    = "backend"
        command = ["/usr/local/bin/scrape"]
      }]
    })

    retry_policy {
      maximum_retry_attempts       = 3
      maximum_event_age_in_seconds = 86400
    }
  }

  state = var.enable_scheduler ? "ENABLED" : "DISABLED"
}
```

```hcl
# 翻訳: 毎時15分にGraphQL mutationを実行
resource "aws_scheduler_schedule" "translation" {
  name                = "${local.name}-translation"
  schedule_expression = "cron(15 * * * ? *)"

  target {
    arn      = var.ecs_cluster_arn
    role_arn = aws_iam_role.scheduler.arn

    ecs_parameters {
      task_definition_arn = var.ecs_task_definition_arn
      launch_type         = "FARGATE"

      network_configuration {
        subnets          = var.private_subnets
        security_groups  = [var.ecs_security_group_id]
        assign_public_ip = false
      }
    }

    input = jsonencode({
      containerOverrides = [{
        name    = "backend"
        command = ["sh", "-c", "curl -sf -X POST ..."]
      }]
    })
  }
}
```

### ポイント

- `ecs_parameters` でFargateタスクを直接起動できるのが強い
- `containerOverrides` でコマンドを上書きし、同じタスク定義でスクレイピングと翻訳を使い分け
- リトライポリシーも組み込みで対応（Airflowのretry相当）

## 移行手順

Terramate（Terraform state分割ツール）で管理していたので、移行は明確でした：

1. `scheduler` スタックの `terraform plan` でEventBridge Schedulerが稼働中であることを確認
2. `mwaa` スタックに対して `terraform destroy` を実行（MWAA環境の削除に**54分**かかった...）
3. MWAAモジュール、Airflow DAGs、関連ドキュメントをリポジトリから一括削除
4. PRをマージ

```bash
# 削除したもの
git rm -r terraform/stacks/prod/mwaa/    # MWAAスタック
git rm -r terraform/modules/mwaa/         # MWAAモジュール（488行）
git rm -r airflow/                        # DAGs、config、plugins
git rm -r docs/airflow/                   # Airflowドキュメント
```

## 結果・学び

### Before / After

| | MWAA | EventBridge Scheduler |
|---|---|---|
| 月額コスト | ~$350+ | ~$0 |
| 年間コスト | ~$4,200+ | ~$0 |
| Terraformコード | 488行（モジュール） | 175行（モジュール） |
| 環境構築時間 | 20〜30分 | 数秒 |
| 環境削除時間 | 54分 | 数秒 |
| 削除したコード | 2,309行 | - |

### 学び

**「本当に必要な複雑さ」を見極めろ。** Airflowは素晴らしいツールだが、「定期的にHTTPリクエストを投げる」だけの用途にはオーバースペック。個人開発では特に、コストと複雑さのトレードオフを定期的に見直すべき。

「将来複雑になるかも」で高コストなツールを維持するより、**今の要件に合ったシンプルな解で十分**。本当に複雑なオーケストレーションが必要になったら、そのときにStep Functionsなり何なり検討すればいい。

## 参考

- [Amazon EventBridge Scheduler](https://aws.amazon.com/eventbridge/scheduler/)
- [Amazon MWAA 料金](https://aws.amazon.com/managed-workflows-for-apache-airflow/pricing/)
- [Terramate — Terraform state分割](https://terramate.io/)

---
🏀 NBA ISO FLOW: https://www.nba-iso-flow.com/ — NBAの最新ニュースをリアルタイムでお届け
