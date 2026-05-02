---
title: "Kotlin/JS に Apollo Kotlin 入れて型安全 GraphQL に倒した話"
emoji: "🧬"
type: "tech"
topics: ["kotlin", "kotlinjs", "graphql", "apollo", "composeforweb"]
published: false
---

## TL;DR

- Compose for Web (Kotlin/JS) のフロントエンドで GraphQL を **Ktor の手書き POST** で叩いていたのを、**Apollo Kotlin** の生成コードに置き換えた
- スキーマを SSOT にする話（前記事）の続編。**`.graphql` ファイル + Apollo 生成クラス** で「クエリ・型・実行」の 3 つをコンパイラに守らせる構成にした
- バージョン選定で **Apollo Kotlin 4 ではなく 3.8.5** を採用。Kotlin/JS で 4 系が Kotlin 2.0 必須 ＆ 既存リポは Kotlin 1.9.x なので、まずは 3.8 系で安定運用
- **GraphQL=Apollo / health=Ktor の二重 HTTP スタック**は意図して許容。「型安全が効く境界」と「素のままでいい境界」を分けた割り切り
- 周辺ハマりどころ（`apollo-mockserver` の死、`ApolloResponse.exception` 不在、生成クラス命名の不統一、endpoint 末尾 `/` バグ）は別途 Qiita で実装ログとして書く

## 背景 — Ktor 手書き POST のしんどさ

前記事 [Rust async-graphql のスキーマを CI で守る話](https://zenn.dev/toguri/articles/rust-async-graphql-schema-ci) で、backend の `Schema::sdl()` を `print_schema` バイナリで出力し、frontend にチェックインした `schema.graphqls` と CI で diff を取る構成を入れた。

これで **「真実の SDL」** がリポに居る状態にはなったけど、その隣で frontend のクエリは Ktor を使った手書き POST のままだった。

```kotlin
// Before: Ktor で GraphQL を素朴に叩く
val response: GraphQLResponse = httpClient.post(endpoint) {
    contentType(ContentType.Application.Json)
    setBody(GraphQLRequest(query = """
        query {
          tradeNews { id title description ... }
        }
    """.trimIndent()))
}.body()
```

これだと:

1. **クエリ文字列がコンパイラに守られない** — フィールド名を typo しても気づくのは実行時
2. **レスポンスの型を自前で書き続ける** — backend 側の SDL が真実なのに、frontend はその影絵を `data class` で再宣言している
3. **スキーマと実コードがズレた瞬間に気づけない** — 前記事の CI drift チェックは backend と SDL の整合は守るが、SDL とこの **手書き `data class`** の整合までは守らない

CI で SDL を真実化したのに、その真実をクエリ実行コードが食ってない、って話メーン。

## 比較検討 — どこで型安全を作るか

選択肢として大きく 3 つあった。

| 案 | 概要 | 型安全 | 学習コスト | バンドルサイズ |
|---|---|---|---|---|
| A. Ktor 手書きのまま、レスポンス型だけ codegen | スキーマから DTO だけ自動生成。POST は今のコードを継承 | 中（クエリ文字列は手書きのまま） | 小 | 小 |
| B. Apollo Kotlin に全面移行 | `.graphql` を Apollo に食わせて Query/Data クラスを生成 | 大 | 中 | 中（ランタイム分） |
| C. graphql-codegen + 軽量クライアント | Node 側のツールで TS 風に生成しつつ Kotlin 用ラッパーを自作 | 大 | 大（自作量） | 小〜中 |

**B（Apollo Kotlin）を採用**した。理由は 3 つ。

1. **「クエリも型もまとめて」生成できる** — A 案だと結局クエリ文字列の typo を実行時まで持ち越す。B はクエリそのものを `.graphql` ファイルに分離し、Apollo の Gradle plugin がスキーマと照合してビルド時に弾く
2. **Compose for Web (Kotlin/JS) で動く実績がある** — 公式の `apollo-runtime` が Kotlin Multiplatform 対応で、`apollo-runtime-js` が JS ターゲット用に自動解決される
3. **C 案は自作量が多い** — graphql-codegen の生成物を Kotlin から触る薄いラッパーをこちらで保守するのが地獄。今回の規模感に対して overkill

## 採用した設計

### Apollo Kotlin **3.8.5** を選んだ理由

最初は Apollo Kotlin 4 系に行こうとした。けど、

- Apollo Kotlin 4 は **Kotlin/JS で Kotlin 2.0 を必須**にしている
- このリポは現状 Kotlin 1.9.22 + Compose 1.5.12 で安定稼働している
- Kotlin 2.0 に上げるなら Compose 側のバージョン整合も同時に動かす必要があり、PR のスコープが膨れる

ので、**今回は 3.8.5 を採用**して、Kotlin 2.0 へのアップグレードを別タイミングで切り出すことにした。

```kotlin
// frontend/build.gradle.kts
plugins {
    kotlin("multiplatform") version "1.9.22"
    id("org.jetbrains.compose") version "1.5.12"
    id("com.apollographql.apollo3") version "3.8.5"
}

dependencies {
    implementation("com.apollographql.apollo3:apollo-runtime:3.8.5")
}

apollo {
    service("tradenews") {
        packageName.set("graphql")
        srcDir("src/jsMain/graphql")
        // async-graphql 側は DateTime を RFC3339 文字列でやり取りする
        // → 生成側も String に倒して、UI 層は今までどおり String を受け取る
        mapScalarToKotlinString("DateTime")
    }
}
```

「最新を入れない」のも 1 つの設計判断っていう話。新しい依存を入れるたびに、既存スタックとの整合に必要な作業量を見極めて、スコープが膨らみそうなら **今のスタックで動く一番新しい安定版**を取る。

### `.graphql` ファイルにクエリを分離

Ktor 時代はクエリ文字列が `GraphQLClient.kt` の中にインライン文字列として埋まってた。Apollo に寄せたタイミングで分離した。

```graphql
# frontend/src/jsMain/graphql/GetAllTradeNews.graphql
query GetAllTradeNews {
    tradeNews {
        id
        title
        description
        link
        source
        publishedAt
        category
        titleJa
        descriptionJa
        translationStatus
        translatedAt
    }
}
```

```graphql
# frontend/src/jsMain/graphql/GetTradeNewsByCategory.graphql
query GetTradeNewsByCategory($category: String!) {
    tradeNewsByCategory(category: $category) {
        id
        title
        # ...同じフィールド
    }
}
```

これで **クエリ自体がスキーマに対して照合される**ようになる。`schema.graphqls` (前記事で真実化したもの) を Apollo plugin が読み込んで、`./gradlew build` の段階で型不一致や typo を弾く。

### `GraphQLClient` は「生成コード + 薄いアダプタ層」に

Apollo の生成コードを UI 層がそのまま触ると、UI 側の型が Apollo の生成型に依存して結合度が上がる。なので **既存の `NewsItem` (UI 用 model) はそのまま残し**、`GraphQLClient` の中で生成型 → `NewsItem` のマッピングを薄く挟むようにした。

```kotlin
class GraphQLClient(
    private val apolloClient: ApolloClient,
    private val healthClient: HttpClient,
    private val endpoint: String,
) : TradeNewsApiClient {

    override suspend fun fetchNewsItems(category: String?): List<NewsItem> {
        return if (category == null) {
            executeQuery(GetAllTradeNewsQuery(), context = "trade news") { data ->
                data.tradeNews.map { it.toNewsItem() }
            }
        } else {
            executeQuery(GetTradeNewsByCategoryQuery(category), context = "category news") { data ->
                data.tradeNewsByCategory.map { it.toNewsItem() }
            }
        }
    }
    // ...
}

private fun GetAllTradeNewsQuery.TradeNew.toNewsItem() = NewsItem(
    id = id,
    title = title,
    // ... フィールドコピー
)
```

責務:
- **`.graphql` + 生成コード** — backend スキーマとの整合
- **`executeQuery`** — Apollo のレスポンスエラー / transport エラーを `GraphQLClientException` にラップして UI に上げる
- **`toNewsItem()`** — 生成型 → UI model の境界

UI 側は `NewsItem` だけを扱うので、もし将来 Apollo を別ライブラリに差し替えても `GraphQLClient` の中だけで完結する。

### HTTP スタック二重化 (Apollo + Ktor) は割り切る

GraphQL は Apollo に寄せたけど、`/health` エンドポイント (REST) は Ktor のクライアントを残した。理由:

- health は **GraphQL ではない素の JSON** — Apollo の管轄外
- 1 エンドポイントのために fetch 系の自作ラッパーを書くのも、Ktor 全捨てするのも overkill
- バンドルサイズ的に Ktor + Apollo の二重ランタイムは確かに重い (production bundle 1.56 MiB) けど、機能影響なしの webpack 警告レベル

ということで「**型安全が効く境界 (GraphQL) は Apollo、効かない境界 (health REST) は Ktor のまま**」という線引きを取った。bundle size がボトルネックになったタイミングで、health 側だけ fetch API の軽量ラッパーに置換する余地は残してある (= Ktor を捨てるのは別 PR の future work)。

「全部統一する」より「**境界ごとに最適なツールを選ぶ**」方が、複雑度に対してのメリットが大きい場面はある。今回はそれって話。

### endpoint 末尾 `/` の正規化を 1 箇所に寄せる

これは PR merge 後の Copilot レビューで見つけてもらった話だが、`.../graphql/` のように末尾 `/` 付きの endpoint を Apollo に渡すと、backend が `/graphql` だけ受ける構成だと **404 になる**。一方で health 側は `trimEnd('/')` で吸収してるので、health だけ通って GraphQL が落ちる、という地味に怖い不整合が起きうる状態だった。

```kotlin
companion object {
    operator fun invoke(endpoint: String = ApiConfig.apiEndpoint): GraphQLClient {
        val normalized = normalizeGraphQLEndpoint(endpoint)
        return GraphQLClient(
            apolloClient = ApolloClient.Builder().serverUrl(normalized).build(),
            healthClient = createDefaultHealthClient(),
            endpoint = normalized,
        )
    }
}

internal fun normalizeGraphQLEndpoint(raw: String): String = raw.trimEnd('/')
```

正規化を `companion object operator fun invoke` に寄せて、Apollo 側 (`serverUrl`) と internal の `endpoint` (health 側で `buildHealthEndpoint` に渡す) の **両方に同じ正規化済み値**を渡すようにした。境界値テストも `internal` で公開して 3 件追加。

## 結果・学び

### 良かったこと

- **クエリの typo がビルド時 fail に降りた** — 前記事の CI drift と合わせて、「backend SDL → frontend クエリ」の整合がコンパイラに守られる二段構えになった
- **UI 層は無傷** — 既存の `NewsItem` を `data class` のまま残せたので、Compose 側の DOM コードは触らずに済んだ
- **`GraphQLClient` の行数がほぼ変わらないのに、型安全性は大幅に上がった** — 生成コードに重さを押し付けて、自分のコードは薄く保てる

### 学び・トレードオフ

- **「最新の major」より「今のスタックで動く最新の minor」** — Apollo Kotlin 4 を待たずに 3.8.5 を取った判断は、PR スコープを絞るうえで効いた。Kotlin 2.0 アップグレードはそれ自体で別 PR にする
- **境界をまたぐ統一は値段次第** — GraphQL=Apollo / health=Ktor の二重スタックは、追加で 1 ライブラリ抱える代わりに「それぞれの境界に合った薄さ」を取れた
- **生成コードにも"癖"がある** — Apollo の生成クラス命名は `tradeNews → TradeNew` (末尾 s 剥がす) と `tradeNewsByCategory → TradeNewsByCategory` (capitalize のみ) で不統一。生成コード側がどう命名するかは事前に把握しておくのが安全

### サンプルリポジトリ

👉 **[toguri/apollo-kotlin-js-sample](https://github.com/toguri/apollo-kotlin-js-sample)**

`StubNetworkTransport` を含むテストも入っているので、ローカルで `./gradlew jsBrowserTest` を叩けばすぐ動く。

## 次回予告

シリーズ最終回 #3 は「**code-first を選んだ理由 — Rust async-graphql + Kotlin/JS Apollo の組合せで気づいたこと**」。code-first / schema-first の宗教戦争に巻き込まれず、自分のプロジェクトでどっちを選ぶかを判断するための観点を整理する話、って話 🎙️

## 参考

- [前記事: Rust async-graphql のスキーマを CI で守る話](https://zenn.dev/toguri/articles/rust-async-graphql-schema-ci)
- [Apollo Kotlin 公式](https://www.apollographql.com/docs/kotlin/)
- [Apollo Kotlin 4 migration ガイド (Kotlin/JS の Kotlin 2.0 要件含む)](https://www.apollographql.com/docs/kotlin/migration/4.0)
- iso-flow リポ: [toguri/iso-flow](https://github.com/toguri/iso-flow)
  - PR #138 (Apollo Kotlin 導入)
  - PR #139 (Copilot follow-up)
