---
title: "Compose for Web × GraphQLで Apollo を「使わない」設計判断 — Ktor Client で十分だった話"
emoji: "🏀"
type: "tech"
topics: ["kotlin", "graphql", "composeweb", "ktor", "個人開発"]
published: false
---

## TL;DR

- NBAトレード速報サイトのフロントエンドに Kotlin Compose for Web を採用
- GraphQL通信に Apollo Kotlin ではなく **Ktor Client による手動実装** を選択した
- 結果: 依存が軽く、テストしやすく、Compose for Web との相性が良い構成になった

## 背景

個人開発で [NBA Trade Tracker](https://www.nba-iso-flow.com/) というサイトを作っている。NBAのトレード・FA情報をリアルタイムで配信するWebアプリで、構成はこうだ:

- **フロントエンド**: Kotlin Compose for Web
- **バックエンド**: Rust（axum + async-graphql）
- **通信**: GraphQL

当初の設計ドキュメントには「Apollo GraphQL Client」と書いていた。Kotlin × GraphQL なら Apollo Kotlin が第一候補だろう、と。

しかし実装を進める中で、**Apollo を導入しない**判断をした。この記事ではその比較検討プロセスと結果を書く。

## 選択肢の比較

### 選択肢A: Apollo Kotlin

GraphQL界隈で最もメジャーなクライアントライブラリ。Android開発では事実上のスタンダード。

**メリット:**
- `.graphql` ファイルからKotlinコードを自動生成（型安全）
- 正規化キャッシュ（同一エンティティの重複排除）
- サブスクリプション（WebSocket）対応
- 巨大なエコシステムとコミュニティ

**デメリット:**
- Compose for Web（Kotlin/JS IR）との互換性が不安定
  - Apollo Kotlin は Android / iOS / JVM がメインターゲット
  - Kotlin/JS IR バックエンドでのビルド問題が報告されている
- Gradle プラグインによるコード生成が必要（ビルド時間増加）
- 依存ツリーが深い（apollo-runtime, apollo-api, apollo-compiler, ...）
- 個人開発の小規模アプリに正規化キャッシュは過剰

### 選択肢B: Ktor Client 手動実装

Ktor は JetBrains 公式の HTTP クライアント。Compose for Web と同じ JetBrains エコシステム。

**メリット:**
- Compose for Web（Kotlin/JS）で確実に動作する `ktor-client-js` エンジン
- 依存が最小限（ktor-client-core + content-negotiation + serialization）
- GraphQL は結局 HTTP POST なので、専用ライブラリなしでも実装できる
- ビルドが速い（コード生成なし）
- kotlinx-serialization でリクエスト/レスポンスを型定義すれば十分な型安全性

**デメリット:**
- スキーマからのコード自動生成がない（手動でデータクラスを書く）
- キャッシュは自前で実装する必要がある
- サブスクリプション対応は自力（今は不要）

### 判断基準

| 基準 | 重み | Apollo | Ktor 手動 |
|------|------|--------|----------|
| Compose for Web で確実に動く | ★★★ | △ | ◎ |
| 依存の軽さ・ビルド速度 | ★★ | △ | ◎ |
| 型安全性 | ★★ | ◎ | ○ |
| 学習コスト | ★ | △ | ◎ |
| 将来の拡張性 | ★ | ◎ | ○ |

**結論: Ktor 手動実装を採用。**

最も重視したのは「Compose for Web で確実に動くこと」。個人開発でビルドが通らない問題のデバッグに時間を取られるのは致命的だった。

## 採用した設計

### インターフェースで抽象化

まずAPIクライアントをインターフェースで抽象化した。これがこの設計の肝になる。

```kotlin
interface TradeNewsApiClient {
    suspend fun fetchNewsItems(category: String? = null): List<NewsItem>
    suspend fun fetchHealthStatus(): BackendHealthResponse
}
```

実装は `GraphQLClient` クラスが担当するが、UIコンポーネントはこのインターフェースしか知らない。将来Apolloに移行しても、REST APIに変えても、UIは一切変更不要。

### GraphQL通信は「ただのHTTP POST」

冷静に考えると、GraphQL クエリの実行は以下の3ステップでしかない:

1. JSON で `{ "query": "...", "variables": {...} }` を POST
2. レスポンスの `data` フィールドをデシリアライズ
3. `errors` フィールドがあればハンドリング

```kotlin
@Serializable
data class GraphQLRequest(
    val query: String,
    val variables: Map<String, String?> = emptyMap()
)

@Serializable
data class GraphQLResponse<T>(
    val data: T? = null,
    val errors: List<GraphQLError>? = null
)
```

この2つのデータクラスだけで、GraphQL通信の型定義は完結する。Apollo の巨大な依存ツリーなしで。

### Ktor × kotlinx-serialization の組み合わせ

```kotlin
companion object {
    private fun createDefaultClient(): HttpClient {
        return HttpClient(Js) {
            install(ContentNegotiation) {
                json(Json {
                    prettyPrint = true
                    isLenient = true
                    ignoreUnknownKeys = true
                })
            }
        }
    }
}
```

`HttpClient(Js)` はブラウザの Fetch API をラップしたエンジン。JetBrains が Kotlin/JS 向けに公式サポートしているので、Compose for Web との相性は当然良い。

`ignoreUnknownKeys = true` は地味だが重要。バックエンドのRust側でスキーマにフィールドを追加しても、フロントが即座にクラッシュしない。Apollo のコード生成なしでスキーマ進化に対応する、防御的な設計判断だ。

### テスタビリティ

この設計の副産物として、テストが非常に書きやすくなった。

```kotlin
// Ktor の MockEngine でHTTPレベルのモック
val mockEngine = MockEngine { request ->
    respond(
        content = """{"data": {"tradeNews": [...]}}""",
        headers = headersOf(HttpHeaders.ContentType, "application/json")
    )
}

val client = GraphQLClient(
    client = HttpClient(mockEngine) { /* ... */ },
    endpoint = "https://test.example.com/graphql"
)
```

Apollo を使っていたら、Apollo 固有のテストユーティリティを覚える必要があった。Ktor の `MockEngine` は HTTP の知識だけで書ける。

## 結果・学び

### この設計で得られたもの

- **ビルド時間**: コード生成がないため、フロントエンドのビルドが高速
- **依存の安定性**: Ktor + kotlinx-serialization は JetBrains 公式。Compose for Web と同じ屋根の下
- **テストカバレッジ**: MockEngine ベースのテストで、GraphQL通信のHappy Path / Error / Transport Failure を網羅
- **変更容易性**: `TradeNewsApiClient` インターフェースにより、通信手段の変更がUIに波及しない

### この設計のトレードオフ

- GraphQL のクエリを文字列リテラルで管理しているので、**クエリのタイポはランタイムまで検出できない**
- スキーマが大きくなった場合、手動でデータクラスを書き続けるのは辛くなる可能性がある
- キャッシュ戦略が必要になったら、自前実装か Apollo 導入を再検討する

### いつ Apollo に移行するか

以下の条件のいずれかを満たしたら、Apollo 導入を検討する:

1. GraphQL のスキーマが20クエリを超えた
2. リアルタイム更新（Subscription）が必要になった
3. 正規化キャッシュがパフォーマンス上必要になった
4. Apollo Kotlin の Kotlin/JS IR サポートが安定した

現時点（2026年3月）では、どれも該当しない。**必要になるまで導入しない** — YAGNI原則そのものだ。

## 参考

- [Ktor Client - Kotlin Multiplatform](https://ktor.io/docs/client-supported-platforms.html)
- [kotlinx-serialization](https://github.com/Kotlin/kotlinx.serialization)
- [Apollo Kotlin - Supported Platforms](https://www.apollographql.com/docs/kotlin/advanced/kotlin-native)
- [Compose for Web - JetBrains](https://www.jetbrains.com/lp/compose-multiplatform/)

---
🏀 NBA Trade Tracker: https://www.nba-iso-flow.com/
