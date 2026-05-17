---
title: "私が code-first を選んだ理由 — Rust + Kotlin/JS で動かして見えた『ツールチェーン視点』"
emoji: "🧭"
type: "tech"
topics: ["graphql", "rust", "kotlin", "asyncgraphql", "apollo"]
published: false
---

## TL;DR

- GraphQL の code-first / schema-first は宗教戦争になりやすいが、本当に効くのは **「ツールチェーンの中で SDL がどこに置かれるか」** の判断
- 評価軸を **リファクタ耐性 / SSOT の位置 / 言語間整合 / レビュー体験** の 4 つに分解すると、自プロジェクトでどっちを取るかが落ち着く
- iso-flow (Rust async-graphql + Kotlin/JS Apollo) では code-first を採用。Rust 型を SSOT にし、SDL を CI で派生物として真実化することで **frontend は schema-first 的に振る舞える**ハイブリッドが取れた
- 「code-first か schema-first か」は二択ではなく、**どこに SSOT を置いて、どこに SDL を派生させるか**のグラデーション。多言語 backend / クエリ設計を先行させたい場面では schema-first を取る、というのも普通に正解

## 背景 — シリーズ #1 #2 で起きていたこと

これは GraphQL スキーマ管理ブログシリーズの最終回 (シリーズ #3 / 3 本構成)。前 2 本でやったことを 1 段引いて整理する回って話。

- [#1: Rust async-graphql のスキーマを CI で守る話](https://zenn.dev/toguri/articles/rust-async-graphql-schema-ci) — backend の `Schema::sdl()` を `print_schema` バイナリで吐き、frontend にチェックインした `schema.graphqls` と CI で `diff` を取って drift を fail させた
- [#2: Kotlin/JS に Apollo Kotlin 入れて型安全 GraphQL に倒した話](https://zenn.dev/toguri/articles/kotlin-js-apollo-kotlin-graphql) — その SDL を Apollo Kotlin に食わせ、frontend のクエリと型をビルド時にスキーマと照合させた

この 2 本を続けて回すと、**「code-first なのに frontend は schema-first っぽく振る舞ってる」** という、ちょっと面白い状態に落ち着いた。これを言語化したい。

「code-first か schema-first か」の宗教戦争は割と古くからあって、ググると一般論レベルの比較表は山ほど出てくる。けど、その比較表だけ見て自分のプロジェクトに当てはめると判断を外す。**実際に効くのはツールチェーン視点での観点**で、それを 4 軸に分解しておくと、自プロジェクトでどっちを取るかが落ち着く。

## 比較検討 — code-first / schema-first をツールチェーン視点で整理する

### 定義のおさらい

GraphQL におけるよくある説明:

- **code-first**: 実装言語の型 / impl から GraphQL schema を生成する。SSOT は **コード**。`schema.graphqls` は派生物
- **schema-first**: SDL (`.graphql` ファイル) を先に書いて、コードはそれに従う形で実装 / 生成する。SSOT は **SDL ファイル**

iso-flow の backend (async-graphql) は code-first。Apollo Kotlin の世界観は schema-first 寄り (= SDL からクライアントを生成する)。だから両方使うとどこかで「コード ↔ SDL」の橋を渡さないといけない、というのが今回のシリーズで扱っていた話。

### 一般的なメリデメ表で議論が止まる理由

よくある比較:

| 観点 | code-first | schema-first |
|---|---|---|
| 書き心地 | 実装言語のシンタックス・型補完が活きる | SDL という独立した DSL を覚える |
| SSOT | コード | SDL |
| 多言語クライアント | SDL を export すれば対応可 | SDL がそのまま元になる |
| スキーマ設計の主体 | バックエンド開発者 | API 設計者 / フロントエンド |

これだけ見て決めようとすると、「**結局どっちでも生成すれば SDL は手に入るじゃん**」となって判断が止まる。実際に効くのはもう 1 段下の **ツールチェーン視点での評価軸**。

### ツールチェーン視点の 4 評価軸

iso-flow を回しながら効いてきた軸はこの 4 つ:

#### 1. **リファクタ耐性** — スキーマ変更の引き金がどこから来るか

- **code-first**: コードを変えるとスキーマが変わる。リファクタ (フィールド rename, 型変更) は IDE のリファクタ機能でほぼ完結
- **schema-first**: SDL を変えるのが起点。コード側はその後に追従する。SDL リネームを実装に反映するには別途 codegen / 手書きの修正が必要

iso-flow の backend は Rust。Rust のリファクタ耐性は強力で、`rust-analyzer` での rename が確実に効く。これを SSOT に置けるのは **code-first の純粋な効用**って話。

#### 2. **SSOT の位置** — レビューで「ここを見ろ」と言える場所がどこか

- **code-first**: PR レビュー時、まず実装コードを読む。SDL は派生物として diff で確認
- **schema-first**: PR レビュー時、まず SDL を読む。「API 契約の変更」が独立した diff になる

iso-flow ではどう転んでも frontend にも変更が出るので、SDL diff も結局 PR の中に出る (= シリーズ #1 の CI drift 対応の効用)。**code-first なのに、レビュー時の「契約変更を見る場所」としては SDL が機能している**。これがハイブリッドの効果。

#### 3. **言語間整合** — backend と frontend で言語が違うとき、どこで型整合を取るか

- **code-first**: backend の型 → SDL export → 各言語の codegen、と中継地点が多い
- **schema-first**: SDL → 各言語の codegen、で同一の起点を全クライアントが共有

iso-flow は Rust (backend) + Kotlin/JS (frontend) で言語跨ぎ。`schema.graphqls` が **両方の言語の codegen の入力になる中継地点**として機能している。code-first の SDL export と schema-first の SDL 駆動 codegen が、`schema.graphqls` という 1 ファイルでちょうど合流する形。

#### 4. **レビュー体験** — PR の diff を見て何が変わるか分かりやすいか

- **code-first (素のまま)**: Rust の impl diff だけ見ても、GraphQL 的に何が変わったかは読み取りにくい
- **code-first + SDL commit**: Rust の diff と SDL の diff が **両方 PR に出る**。SDL diff の方が「API として何が増えた / 消えた」を素早く読めるので、レビュアーに優しい
- **schema-first**: SDL の diff だけ見れば API 変更が分かる。実装は別 commit / 別 PR の場合もある

シリーズ #1 で SDL を commit しに行ったのは、まさにこの「レビュー時に SDL diff を見たい」というモチベーション。code-first の **メリットを残したまま、schema-first 的なレビュー体験を獲得**できたところに価値がある。

## 採用した設計 — iso-flow がたどり着いた code-first ハイブリッド

iso-flow で取った構成を、4 評価軸で評価すると次のようになる。

| 軸 | iso-flow の選択 | 評価 |
|---|---|---|
| リファクタ耐性 | Rust 型を SSOT に置く (code-first) | ◎ — rust-analyzer の rename が API 契約を一気に動かせる |
| SSOT の位置 | コード (Rust) が真実、SDL は派生物 | ○ — ただし派生物の真実化を CI で保証 |
| 言語間整合 | SDL を中継ファイル化 (CI drift check) | ◎ — Kotlin 側 Apollo はこの SDL を入力にできる |
| レビュー体験 | SDL も PR に出るので diff が読みやすい | ◎ — code-first の弱みを SDL commit で補える |

つまり iso-flow は「code-first か schema-first か」と聞かれたら **code-first** と答えるんだけど、フロント側の experience は限りなく schema-first に近い。これは:

- **SSOT は Rust** (リファクタ耐性のため)
- **派生物 (SDL) を CI で真実化** (drift しないように)
- **フロント側はその SDL を起点に codegen** (言語間整合のため)

という 3 段構えで成り立っている。シリーズ #1 が中段 (CI drift check)、シリーズ #2 が下段 (Apollo Kotlin) を担当していた、と言ってもいい。

### なぜ schema-first から始めなかったのか

ありえた選択肢は: SDL ファイルを先に書いて、Rust 側は手書き or codegen で実装する。これを取らなかった理由は 3 つ。

1. **Rust の型システムが強い** — async-graphql の `#[Object]` マクロ + Rust の `Result<T, E>` / `Option<T>` の表現力で、SDL では書きにくい契約 (例: 必ず非 null) を自然に書ける
2. **個人開発の小規模 backend** — API 設計を SDL 駆動で始める文化を確立するメリットがチーム規模に対して合わない
3. **frontend は 1 つだけ** — Kotlin/JS の Apollo 用に SDL を渡す必要はあるが、複数の言語に同時配布する想定がない

逆に **次のどれかが該当するなら schema-first を取るのが普通に正解** ってのも書いておく:

- **多言語のクライアント** が事前にいる (iOS / Android / Web / バックオフィス…)
- **API 設計をフロントエンド主導で先行**させたい文化 (= SDL レビューを契約として使いたい)
- **複数のバックエンドサービス** が同じ API を実装する (= SDL を契約として配る)
- **GraphQL Federation** を使う

iso-flow はどれにも当てはまらなかったので code-first を取った、それだけって話メーン。

## 結果・学び

### シリーズ全体の振り返り

3 本を通して言いたかったことを 1 行で:

> **「code-first か schema-first か」は二択ではなく、SSOT をどこに置いて、SDL をどこに派生させるかのグラデーション。**

#1 (CI drift) と #2 (Apollo Kotlin) はそれぞれ単発でも使える話だが、組み合わせるとこのグラデーションを設計できる、というのがシリーズの主旨だった。

それぞれの章で得たもの:

- **#1 (CI drift)**: code-first を取りつつ、SDL を「派生物だけど真実」にする方法。CI が嘘をついた SDL を許さない構造を作れた
- **#2 (Apollo Kotlin)**: その「真実の SDL」をフロント側に持ち込んで、クエリも型もコンパイラに守らせた。code-first の弱点 (フロントの型安全) を schema-first 的なツールで補完
- **#3 (この記事)**: 振り返って、その判断を「ツールチェーン視点の 4 軸」で言語化

### 取り急ぎ言いたい 1 つ

「code-first / schema-first どっちにしようかな」と迷ったら、まず **一般論の比較表を捨てて、自分のプロジェクトの 4 軸を採点する**のがいい。

- リファクタ耐性が強い言語で書いていれば code-first が刺さる
- 多言語クライアントがいれば schema-first の方が摩擦が少ない
- レビュー体験は code-first でも「SDL commit + CI drift check」で取り戻せる
- 言語間整合は SDL を中継ファイルとして扱えばどっちでも取れる

「どっちが優れているか」じゃなくて、「**自分のプロジェクトのどこに摩擦があるか**」で選ぶ、って話ナーミー。

### 個人開発の限界と注意点

最後にいくつか正直なところを書いておく:

- この記事の構成は **個人開発 + 1 frontend** という規模感を前提にしている。チームが増えて複数 frontend が来た瞬間、判断はまた揺れる
- **CI drift check が常に効く前提**で SDL commit している。CI を切ったり paths フィルタを過剰に絞ったりすると、シリーズ #1 の真実化が壊れて drift が再発する
- 「code-first を選んだから schema-first の知識は要らない」ではない。**Apollo Kotlin との接点では schema-first 的な発想を借りる**ので、両方の語彙が要る

つまり、この記事は「code-first を布教する」記事ではなく、「**code-first を選ぶときに、どこを補強しておくと心地よく回るか**」のレシピって位置づけ。

## シリーズリンク

GraphQL スキーマ管理ブログシリーズ (全 3 本):

1. [Rust async-graphql のスキーマを CI で守る話](https://zenn.dev/toguri/articles/rust-async-graphql-schema-ci) — code-first での SDL drift 対策
2. [Kotlin/JS に Apollo Kotlin 入れて型安全 GraphQL に倒した話](https://zenn.dev/toguri/articles/kotlin-js-apollo-kotlin-graphql) — その SDL を frontend に持ち込む
3. **この記事** — 振り返り + ツールチェーン視点での評価軸

## 参考

- [async-graphql 公式 — code-first の哲学](https://async-graphql.github.io/async-graphql/en/quickstart.html)
- [Apollo Kotlin 公式](https://www.apollographql.com/docs/kotlin/) — schema-first 駆動の codegen
- [GraphQL 公式仕様](https://spec.graphql.org/) — そもそも GraphQL は実装方法を規定していない
- 本番運用中サイト: [nba-iso-flow.com](https://nba-iso-flow.com)
- iso-flow リポジトリ: [toguri/iso-flow](https://github.com/toguri/iso-flow)
