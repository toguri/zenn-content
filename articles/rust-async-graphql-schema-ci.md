---
title: "Rust async-graphql のスキーマを CI で守る話"
emoji: "🛡️"
type: "tech"
topics: ["rust", "graphql", "asyncgraphql", "ci", "githubactions"]
published: false
---

## TL;DR

- Rust の [async-graphql](https://github.com/async-graphql/async-graphql) は **code-first**（Rust の型 → GraphQL schema 自動生成）
- 便利だけど、frontend 側にチェックインされた `schema.graphqls` が **手書きで更新漏れ**すると気づけない
- 解決策: `Schema::sdl()` を stdout に吐くだけの **小さな `print_schema` バイナリ**を追加し、CI で `diff` を取って drift を fail させる
- ポイントは **DB 接続不要**でスキーマを export できるようにすること（CI でコンテナ立ち上げたくない）

## 背景 — code-first のスキーマ、どこに置く？

async-graphql は code-first。Rust の struct や impl に `#[Object]` を付ければ、スキーマは自動で生成される。

```rust
#[Object]
impl Query {
    /// 全てのトレードニュースを取得します
    async fn trade_news(&self, ctx: &Context<'_>) -> Result<Vec<TradeNews>> {
        // ...
    }
}
```

SSOT（Single Source of Truth）が Rust 型に寄るのが気持ちいい。リファクタ耐性も高い。

ただ、**frontend はそのままだと SDL が欲しい**。codegen（Apollo Kotlin、graphql-codegen など）を使うときの入力は SDL ファイルだし、レビュー時にスキーマ差分が見えるのも嬉しい。

で、よくあるのが **frontend リポに `schema.graphqls` を手書きで置く**パターン。これが地獄の始まり。

iso-flow でも実際に起きてた:

- backend では `tradeNews` という Query 名で実装
- frontend の `schema.graphqls` には `newsItems` という **過去の名前**が残ってた
- しかも frontend は Ktor で手書きクエリしてたので、ビルド時に検知されない

つまり SDL が **置物**になってて、嘘をついていた。

## 比較検討 — どうやって SSOT を守るか

3 つの選択肢を並べた。

### 1. SDL を完全に捨てる（schema-first を諦める）

frontend の型安全を諦めるなら可能。codegen も動的生成にする手もある。けど個人開発とはいえ、いずれ Apollo Kotlin で型安全にしたい予定があるので却下。

### 2. backend の build.rs で SDL を生成して `include_str!`

やれなくはない。ただし、frontend にファイルを渡す方法が別途必要で、**結局ファイルのコミットか配信が要る**。生成の自動化だけでは drift は防げない。

### 3. SDL を commit し続ける + CI で drift 検知（採用）

SSOT は Rust 型のまま。SDL は派生物として commit するが、**手で書かせない**。CI が `diff` で差分を fail させる。

採用理由:
- 既存の `schema.graphqls` 運用を大きく変えなくていい
- PR のレビュー時にスキーマ差分が diff で見える（レビュアーに優しい）
- codegen（次のフェーズで Apollo Kotlin を予定）が同じファイルを入力にできる

## 採用した設計

構成はシンプル。3 ファイルだけ。

### パーツ 1: `print_schema` バイナリ

```rust
// backend/src/bin/print_schema.rs
use async_graphql::{EmptySubscription, Schema};
use nba_trade_scraper::graphql::{Mutation, Query};

fn main() {
    let schema = Schema::build(Query, Mutation, EmptySubscription).finish();
    print!("{}", schema.sdl());
}
```

13 行。これだけ。やってるのは:

1. `Schema::build` に Query / Mutation の型を渡してインスタンス化
2. `schema.sdl()` で SDL 文字列を取り出して stdout に吐く

**重要なのは DB 接続が一切要らないこと。** async-graphql は型情報だけからスキーマを組み立てられるので、`PgPool` も `SqlitePool` も持たずに動く。これが CI で地味に効く（後述）。

### パーツ 2: commit された SDL

```graphql
# frontend/src/jsMain/graphql/schema.graphqls（抜粋）
type Query {
  """
  全てのトレードニュースを取得します（データベースから）

  最新100件のニュースを返します
  """
  tradeNews: [TradeNews!]!
  tradeNewsByCategory(category: String!): [TradeNews!]!
  # ...
}
```

このファイルは手で書かない。必ず `cargo run --bin print_schema` の出力で上書きする。Rust のドキュメントコメント（`///`）もそのまま GraphQL の description に流れるのが嬉しい。

### パーツ 3: CI workflow

```yaml
# .github/workflows/schema-check.yml
name: GraphQL Schema Drift Check

on:
  pull_request:
    paths:
      - 'backend/src/graphql.rs'
      - 'backend/src/bin/print_schema.rs'
      - 'backend/Cargo.toml'
      - 'backend/Cargo.lock'
      - 'frontend/src/jsMain/graphql/schema.graphqls'
      - '.github/workflows/schema-check.yml'

env:
  SQLX_OFFLINE: true

jobs:
  schema-drift:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
        with:
          workspaces: backend

      - name: Regenerate SDL from backend
        working-directory: backend
        run: cargo run --bin print_schema --quiet > /tmp/schema.graphqls

      - name: Diff against committed schema.graphqls
        run: |
          if ! diff -u frontend/src/jsMain/graphql/schema.graphqls /tmp/schema.graphqls; then
            echo "❌ schema.graphqls が backend とズレてる"
            echo "   cd backend && cargo run --bin print_schema > ../frontend/src/jsMain/graphql/schema.graphqls"
            exit 1
          fi
          echo "✅ SDL is in sync"
```

ポイントは 3 つ。

#### `paths` フィルタで無駄を省く

SDL に関係しない PR では走らせない。backend 全体の CI とは別枠に切って、build を cache しつつ必要なときだけ動く形にした。

#### `SQLX_OFFLINE: true`

iso-flow は sqlx の compile-time check を使っているので、DB なしで `cargo build` すると死ぬ。`SQLX_OFFLINE=true` で query cache を見に行く形にして、DB コンテナを立てずに済ませる。

これが効いてるのは **print_schema バイナリ自体は DB を touch しない**から。CI でこのバイナリだけビルドすれば、Postgres も MySQL もいらない。

#### エラーメッセージに **修正コマンドを埋め込む**

`diff -u` で差分を見せるだけだと、PR を出した人は「で、どうすればいいの？」となる。修正コマンドを echo で吐いておくと、Actions のログからコピペできる。これは地味に大事。

## 結果・学び

### 嬉しかったこと

- `schema.graphqls` が **真実**になった。「frontend のコードは古いけど schema は新しい」みたいな不整合が消えた
- レビュアーとしても、GraphQL の変更は SDL diff を見れば一発で分かる（description の変更も diff に出る）
- 後続の Apollo Kotlin 導入（次回予告）で、codegen の入力を疑う必要がなくなった

### 注意点

- `schema.sdl()` の出力は **表示順が実装順に依存する**ので、Rust 側の順番を動かすと diff が出てしまう。最初は面食らうかも。諦めて commit するか、ソートを噛ませるかの判断が要る（iso-flow は諦め派）
- `paths` フィルタに schema.graphqls 自身を入れるのを忘れない。手で schema を戻したいだけの PR で CI が走らないと、drift の検知をすり抜ける
- DB が要る別のバイナリを `src/bin/` に増やすと、CI で全 bin をビルドしようとして事故る。`cargo run --bin print_schema` と **バイナリ名を明示**する

### 次回予告

この SDL を入力に、**Kotlin/JS + Apollo Kotlin** で型安全 GraphQL クライアントを入れた話を書く予定。Apollo Kotlin 4 が Kotlin 2.0 必須だったり、`apollo-mockserver` が死んでたりで、地味にハマりどころがある。

## サンプルリポジトリ

👉 **[toguri/rust-async-graphql-schema-ci](https://github.com/toguri/rust-async-graphql-schema-ci)**

この記事の構成を最小サンプルとして切り出したリポ。`cargo run --bin print_schema > schema.graphqls` と `schema-check.yml` をコピーして使えるよう整えてある。

## 参考

- [async-graphql book — SDL export](https://async-graphql.github.io/async-graphql/en/sdl_export.html)
- [本家 async-graphql リポジトリ](https://github.com/async-graphql/async-graphql)
- 元ネタの PR: [iso-flow#137](https://github.com/toguri/iso-flow/pull/137)
