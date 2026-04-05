---
title: "SDD駆動で1日でECS Blue/Green CD基盤を構築した設計判断の全記録"
emoji: "🚀"
type: "tech"
topics: ["aws", "githubactions", "terraform", "個人開発", "ecs"]
published: false
---

## TL;DR

- 個人開発のNBAニュースサイトに、ECS Native Deployment Strategy（CANARY/LINEAR）+ GitHub Actions の CD 基盤を1日で構築した
- SDD（Spec-Driven Development）で要件定義→設計→レビュー→実装の全フローを回し、設計判断を全て文書化した
- 「PipeCDの機能をECS Nativeで再現できるか」を検証し、Blue/Greenデプロイ + 3層ロールバックを実現した

## 背景

[NBA ISO FLOW](https://www.nba-iso-flow.com/) というNBAニュースサイトを個人開発しています。バックエンドはRust（Axum）、フロントエンドはKotlin Compose for Web、インフラはAWS + Terraformです。

このプロジェクトには **CDパイプラインが存在しなかった**。デプロイは `aws ecs update-service --force-new-deployment` を手動で叩くだけ。Terraformの適用もローカルから手動。

別プロジェクト（van）で PipeCD → GitHub Actions への CD 移行を計画しており、その検証環境としてiso-flowにCD基盤をゼロから構築することにした。

## 比較検討：ECS のデプロイ戦略

### 選択肢A: CodeDeploy Blue/Green

AWS CodeDeploy を使った伝統的な Blue/Green デプロイ。

- **メリット**: 実績豊富、Terraform サポート成熟
- **デメリット**: CodeDeploy という追加レイヤーが必要、月$300+のコスト増、AppSpec定義が必要

### 選択肢B: ECS Native Deployment Strategy

2025年7月にGAした ECS 標準の CANARY / LINEAR デプロイ。

- **メリット**: CodeDeploy 不要、ECS + CloudWatch だけで完結、月$3程度
- **デメリット**: GA後まだ日が浅い、Terraform Provider の対応状況に不安

### 判断：ECS Native を採用

**決定打は「Blue/Green のやり方の違い」に気づいたこと。**

ECS Native の CANARY/LINEAR は、内部的に新旧2つのタスクセットを同時に走らせてトラフィックを段階移行する。これは Blue/Green そのもの。CodeDeploy は Blue/Green の「実現手段」であり、ECS Native はより軽量な別の「実現手段」だった。

## 採用した設計

### アーキテクチャ全体像

```
Developer → merge to main
  ↓
CI (deploy.yml): Build → ECR Push
  ↓ workflow_run
CD (cd-ecs.yml): Register TD → update-service (CANARY or LINEAR)
  ↓
ECS Native: 10% traffic → bake 5min → 100% → cleanup
  ↕
CloudWatch Alarm: 5xx > 1% or P99 > 3s → 自動ロールバック
Circuit Breaker: タスク起動失敗 → 自動ロールバック
  ↕
Manual: cd-rollback.yml → 手動ロールバック
```

### 設計判断のハイライト

#### 1. CANARY の bake time で承認ゲートを代替

別プロジェクト（van）では PipeCD の `WAIT_APPROVAL` で人間の承認を挟んでいた。しかし ECS Native は外部からデプロイを途中停止する API がない。

**解決策**: CANARY の bake time（waitDuration）を承認ウィンドウとして利用。問題があれば bake 中に手動ロールバックを実行する。

ただし、iso-flow は private repo のため GitHub Environment の approval gate が使えないことが判明。public repo や Team プランなら `production` Environment の Required Reviewers で PipeCD 完全互換の承認フローを実現できる。

#### 2. デプロイ戦略の動的切り替え

ECS Service は1つの deployment strategy しか持てない。Terraform で CANARY をデフォルト定義し、CD ワークフローが `update-service` の `deploymentConfiguration` パラメータで動的に CANARY/LINEAR を切り替える設計にした。

```yaml
# cd-ecs.yml — 戦略に応じてJSONを生成
- name: Resolve deployment strategy
  run: |
    STRATEGY="${INPUT_STRATEGY:-canary}"
    if [ "$STRATEGY" = "linear" ]; then
      cat <<'EOJSON' > /tmp/deploy-config.json
      { "deploymentStrategy": { "type": "LINEAR", "linear": { "linearStepSize": 25, "waitDuration": 180 } } }
      EOJSON
    else
      cat <<'EOJSON' > /tmp/deploy-config.json
      { "deploymentStrategy": { "type": "CANARY", "canary": { "steps": [{"type":"CANARY","stepSize":10},{"type":"WAIT","duration":300}] } } }
      EOJSON
    fi
```

#### 3. Terraform CD の自動化

Terraform の apply もワークフローで自動化した。PR マージ → `cd-terraform.yml` → Terramate で変更スタック検出 → plan → apply。これで「terraform apply を手動で叩く」が完全になくなった。

### セキュリティレビューで見つかった問題

Devil's Advocate レビュー（意図的に批判的な視点でのコードレビュー）で **シェルインジェクション脆弱性** が3箇所見つかった。

```yaml
# NG: ${{ inputs.image_tag }} が run: 内で直接展開される
TAG="${{ inputs.image_tag }}"

# OK: env: 経由で環境変数として渡す
env:
  INPUT_IMAGE_TAG: ${{ inputs.image_tag }}
run: |
  TAG="$INPUT_IMAGE_TAG"
```

`workflow_dispatch` のフリーテキスト入力を `${{ }}` で直接展開すると、`"; malicious_command #` のような値で任意コマンドが実行される。GitHub 公式も `env:` 経由を強く推奨している。

## 結果・学び

### Before / After

| | Before | After |
|---|---|---|
| ECS デプロイ | 手動 `force-new-deployment` | CANARY/LINEAR Blue/Green |
| ロールバック | 手動 `update-service` | 3層防御（Circuit Breaker + CW Alarm + 手動） |
| Terraform | ローカルから手動 apply | PR マージで自動 apply |
| フロントエンド | 手動 S3 sync | push で自動ビルド&デプロイ |

### SDD で得たもの

SDD（Spec-Driven Development）のフローで進めたことで、**設計判断が全て文書化された**。

```
docs/designs/cd-stack/
├── requirements.md   # 要件書（EARS形式、8要件37AC）
├── design.md         # 概念設計（4つの設計判断）
├── detail.md         # 詳細設計（ワークフローYAML仕様、Terraform HCL）
├── research.md       # 調査ログ（PipeCD構成調査、ECS API仕様確認）
└── tasks.md          # タスク分解（6 Story、11タスク）
```

1日で終わった理由は「SDDで設計を先に固めたから実装でブレなかった」に尽きる。要件定義時点で Devil's Advocate レビューを入れたことで、「ECS Native は途中停止できない」「GitHub Environment は private repo で使えない」といった制約を実装前に発見できた。

### 学び

**「別プロジェクトの検証環境」としての個人開発は最強。** 失敗しても影響がない環境で新技術を検証し、設計判断と知見を文書化しておけば、本番プロジェクトへのフィードバックが圧倒的にスムーズになる。

## サンプルリポジトリ

この記事で解説した CD 基盤の実装コード（Terraform + GitHub Actions）をサンプルリポジトリとして公開しています。

👉 **[toguri/example-ecs-native-cd](https://github.com/toguri/example-ecs-native-cd)**

## 参考

- [Amazon ECS Native Deployment Strategy](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-ecs.html)
- [GitHub Actions — Keeping your GitHub Actions and workflows secure](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions)
- [Terramate](https://terramate.io/)

---
🏀 NBA ISO FLOW: https://www.nba-iso-flow.com/ — NBAの最新ニュースをリアルタイムでお届け
