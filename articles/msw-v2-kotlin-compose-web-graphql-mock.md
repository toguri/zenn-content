---
title: "Kotlin Compose for Web に MSW v2 を導入して GraphQL モックを実現した設計判断"
emoji: "🏀"
type: "tech"
topics: ["kotlin", "msw", "graphql", "compose", "frontend"]
published: false
---

## TL;DR

- Kotlin Compose for Web のフロントエンドに **MSW (Mock Service Worker) v2** を導入し、バックエンド不要で GraphQL レスポンスをモックできる開発環境を構築した
- Kotlin/JS から JavaScript ライブラリを呼ぶための **external 宣言 + dynamic 型**のブリッジパターンを確立
- `?msw=true` の URL パラメータで **オプトイン方式**にし、本番環境への影響をゼロにした
- Kotlin データクラスと JavaScript オブジェクト間の **シリアライゼーション・ラウンドトリップ**で相互運用を実現

## 背景

### プロジェクト構成

NBA トレード速報を配信する Web アプリ「ISO Flow」を開発している。技術スタックは以下の通り。

| レイヤー | 技術 |
|---------|------|
| フロントエンド | Kotlin Compose for Web |
| バックエンド | Rust (Axum + async-graphql) |
| 通信 | GraphQL (Ktor HttpClient) |

### 課題

フロントエンド開発中、以下の問題が頻発していた。

1. **バックエンドの起動が必要** — Rust サーバーをローカルで立ち上げないと UI 確認ができない
2. **データ準備のコスト** — RSS フィード取得やスクレイピングが必要で、テストデータの用意が面倒
3. **API 未実装時のブロック** — バックエンドの新機能が未完成だとフロントエンド開発が止まる

フロントエンド単体で動作する仕組みが欲しかった。

## 比較検討

### 候補

| 手法 | 概要 | メリット | デメリット |
|------|------|---------|-----------|
| **条件分岐** | Kotlin 内で `if (isDev)` でモックデータを返す | シンプル | プロダクションコードが汚れる、HTTP レイヤーをテストできない |
| **Ktor MockEngine** | Ktor HttpClient のエンジンを差し替え | Kotlin 純正 | DI やテスト設定が複雑、Service Worker レベルの透過性がない |
| **ローカルプロキシ** | mitmproxy 等でレスポンスを差し替え | 既存ツール | 環境構築コストが高い、チームメンバーへの展開が難しい |
| **MSW v2** | Service Worker でネットワークリクエストを透過的にインターセプト | HTTP レイヤー完全再現、アプリコード変更不要 | Kotlin/JS からの呼び出しに工夫が必要 |

### 決定: MSW v2

**最大の決め手は「アプリコードの変更が不要」という点。**

MSW は Service Worker レベルで HTTP リクエストをインターセプトするため、Ktor HttpClient は通常通りリクエストを送信し、MSW がレスポンスを返す。GraphQL クライアントのコードに一切手を入れる必要がない。

Kotlin/JS から JavaScript ライブラリを呼ぶ壁はあるが、`dynamic` 型を活用すれば十分に乗り越えられると判断した。

## 採用した設計

### アーキテクチャ全体像

```
Main.kt (エントリポイント)
  ↓
setupMsw() — [MswSetup.kt]
  ↓ require() で npm パッケージをロード
MSW npm package (v2.7.0)
  ├─ msw (コアモジュール)
  ├─ msw/browser (ブラウザ API)
  └─ mockServiceWorker.js (Service Worker)
  ↓
GraphQLClient.kt (変更なし)
  ↓
Ktor HttpClient (JS エンジン)
  ↓ fetch API
Service Worker がインターセプト → モックレスポンス返却
```

### 有効化の二重ガード

MSW が意図せず動くことを防ぐため、2 つの条件を AND で判定する。

```kotlin
fun isMswEnabled(): Boolean {
    // ガード 1: localhost 系のホストでのみ有効
    if (!isDevelopmentHost()) return false
    // ガード 2: URL パラメータ ?msw=true が必須
    val searchParams = js("new URLSearchParams(window.location.search)")
    return (searchParams.get("msw") as? String) == "true"
}
```

本番ドメインでは絶対に起動しない。開発中でも明示的にパラメータを付けない限り実バックエンドに接続する。

### GraphQL ハンドラの設計

```kotlin
val graphqlHandler = msw.http.post("/graphql", fun(info: dynamic): dynamic {
    return info.request.json().then(fun(body: dynamic): dynamic {
        val query = (body.query as? String) ?: ""

        if (query.contains("GetTradeNewsByCategory")) {
            val variables = body.variables
            val category = (variables?.category as? String) ?: "ALL"
            // カテゴリでフィルタリングしてレスポンス返却
            return@then msw.HttpResponse.json(buildFilteredResponse(category))
        }

        // デフォルト: 全件返却
        return@then msw.HttpResponse.json(buildAllNewsResponse())
    })
})
```

ポイントは `request.json()` が **Promise を返す**こと。Kotlin の `suspend` 関数ではなく、JavaScript の `.then()` チェーンで処理する必要がある。

### データブリッジ: Kotlin → JavaScript

Kotlin データクラスを JavaScript のハンドラから参照するために、シリアライゼーション・ラウンドトリップを使う。

```kotlin
fun publishToWindow() {
    // Step 1: Kotlin データクラスを JSON 文字列に変換
    val jsonString = json.encodeToString(
        ListSerializer(NewsItem.serializer()), newsItems
    )
    // Step 2: JSON 文字列を JavaScript オブジェクトに再パース
    val parsed = kotlin.js.JSON.parse<dynamic>(jsonString)
    // Step 3: window オブジェクトに公開
    kotlinx.browser.window.asDynamic().__mswMockData = parsed
}
```

**なぜ直接渡せないのか:** Kotlin データクラスのランタイム表現は JavaScript オブジェクトとは異なる。`kotlinx.serialization` で JSON 化 → `JSON.parse()` で再構築することで、JavaScript 側から自然にプロパティアクセスできるオブジェクトになる。

### アプリ起動シーケンス

```kotlin
fun main() {
    val promise: dynamic = try {
        setupMsw()
    } catch (error: dynamic) {
        renderApp()
        return
    }

    if (promise != null) {
        // MSW 有効: 起動完了を待ってからレンダリング
        promise.then {
            window.asDynamic().__mswStarted = true
            renderApp()
        }.catch { error: dynamic ->
            // MSW 起動失敗: フォールバックでレンダリング
            renderApp()
        }
    } else {
        // MSW 無効: 即座にレンダリング
        renderApp()
    }
}
```

MSW の起動に失敗しても、アプリは実バックエンドで正常動作する。**グレースフルデグラデーション**を徹底した。

### Webpack 設定

MSW v2 は `package.json` の `exports` フィールドでサブパス解決を行う。Kotlin/JS のデフォルト Webpack 設定ではこれを解決できないため、明示的に `conditionNames` を追加する。

```javascript
// webpack.config.d/msw-resolve.js
const existingConditionNames = config.resolve.conditionNames || [];
config.resolve.conditionNames = [
    'browser', 'import', 'require', 'default',
    ...existingConditionNames
];
```

### UI インジケーター

MSW が有効な状態を見逃さないよう、2 つの視覚要素を表示する。

1. **ページ上部のバナー** — `"MSW Mode — API responses are mocked"` のアンバー警告
2. **右下の固定バッジ** — 常時表示の `"MSW"` パープルバッジ

```kotlin
val mswStarted = js("window.__mswStarted === true") as Boolean
if (mswStarted) {
    Div(attrs = { classes("msw-banner") }) {
        Text("MSW Mode — API responses are mocked")
    }
}
```

## 結果・学び

### 得られた成果

- **フロントエンド開発が独立** — バックエンドなしで UI 開発・確認が可能に
- **GraphQL クライアントのコード変更ゼロ** — Ktor HttpClient はそのまま
- **安全な有効化** — 二重ガード（ホスト + URL パラメータ）で事故防止
- **Kotlin/JS interop パターンの確立** — 他の JavaScript ライブラリ導入にも応用可能

### Kotlin/JS × MSW で学んだ 3 つの教訓

1. **`js()` は try/catch で囲めない** — IR コンパイラの lowering 順序の制約。`js()` による `require()` はブロック外で実行し、その後の処理を try/catch で保護する
2. **Promise は `.then()` で処理する** — MSW ハンドラ内では Kotlin の `suspend` が使えない。JavaScript の Promise チェーンに合わせる
3. **データ共有はシリアライゼーション経由** — Kotlin データクラスを `window` に置く場合、JSON ラウンドトリップが最も安全

### 今後の展望

- モックデータの外部ファイル化（JSON で管理）
- MSW のハンドラをテストコードと共有
- エラーレスポンスのシミュレーション（500, timeout 等）

## サンプルリポジトリ

👉 **[toguri/example-msw-kotlin-compose](https://github.com/toguri/example-msw-kotlin-compose)**

## 参考

- [MSW (Mock Service Worker) 公式ドキュメント](https://mswjs.io/)
- [Kotlin/JS — JavaScript からの呼び出し](https://kotlinlang.org/docs/js-interop.html)
- [Kotlin/JS — dynamic 型](https://kotlinlang.org/docs/dynamic-type.html)
- [kotlinx.serialization](https://github.com/Kotlin/kotlinx.serialization)
