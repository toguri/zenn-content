---
title: "Chrome DevTools MCP × Playwright MCP — Claude Code で二刀流フロントエンド品質チェック"
emoji: "🏀"
type: "tech"
topics: ["claudecode", "mcp", "playwright", "devtools", "frontend"]
published: false
---

## TL;DR

- Claude Code から **Playwright MCP（操作）** と **Chrome DevTools MCP（分析）** を同時に使うと、フロントエンドの品質チェックが一発で終わる
- Playwright でユーザー操作をシミュレートしながら、Chrome DevTools でパフォーマンス計測・Lighthouse 監査・ネットワーク監視を並行実行できる
- 個人開発の NBA ニュースサイトで実践した結果、LCP 339ms / CLS 0.09 / Best Practices 100 を確認できた

## 背景

個人開発で NBA のトレードニュースを配信する Web アプリ [NBA ISO FLOW](https://www.nba-iso-flow.com/) を運用している。フロントエンドは **Kotlin Compose for Web**、バックエンドは **Rust（Axum + async-graphql）** という構成だ。

フロントエンドのコードレビューやテスト補強を進める中で、「ブラウザ操作」と「パフォーマンス分析」を別々にやるのが面倒になってきた。そこで Claude Code の MCP（Model Context Protocol）を活用して、2つの MCP サーバーを **同時に** 叩くワークフローを試してみた。

## 比較検討

### 選択肢A: Playwright MCP 単体

Playwright MCP だけでもブラウザ操作・スクリーンショット・アクセシビリティスナップショットが取れる。ただし：

- パフォーマンスメトリクス（LCP, CLS, TTFB）の計測ができない
- Lighthouse 監査が実行できない
- ネットワークリクエストの詳細監視ができない

### 選択肢B: Chrome DevTools MCP 単体

Chrome DevTools MCP はパフォーマンストレース、Lighthouse、ネットワーク/コンソール監視が強力。ただし：

- ユーザー操作のシミュレートが弱い（クリック等はできるが、Playwright ほどリッチではない）
- アクセシビリティスナップショット（DOM ツリーの意味構造）が取れない

### 採用: 二刀流（Playwright + Chrome DevTools）

両方を同時に使えば、それぞれの弱点を補完できる。

| 役割 | MCP | 得意分野 |
|------|-----|---------|
| **Driving（操作）** | Playwright MCP | ナビゲーション、クリック、フォーム入力、スクリーンショット、アクセシビリティスナップショット |
| **Debugging（分析）** | Chrome DevTools MCP | Performance Trace、Lighthouse、ネットワーク監視、コンソール監視 |

## 採用した設計

### ワークフロー

```
1. Playwright → サイトにナビゲート、ページ構造を把握
2. Chrome DevTools → 同じサイトを開いてパフォーマンストレース開始
3. Playwright → カテゴリフィルターをクリック、スクリーンショット撮影
4. Chrome DevTools → ネットワークリクエスト監視、コンソールエラー確認
5. Chrome DevTools → Lighthouse 監査（モバイル）
6. 結果を統合してレポート出力
```

ポイントは **Step 2〜4 で両方の MCP を並行して叩ける** こと。Claude Code はツールの並列呼び出しをサポートしているので、Playwright でクリックしながら Chrome DevTools でネットワークを監視する、といったことが1ターンでできる。

### 実際のやりとり例

まず Playwright でページを開いてスナップショットを取得する：

```
Playwright → browser_navigate("https://nba-iso-flow.com")
Playwright → browser_snapshot() → ボタン一覧が見える
  - button "すべて" [ref=e9]
  - button "Trade" [ref=e10]
  - button "Signing" [ref=e11]
```

次に Chrome DevTools でパフォーマンストレースを開始しつつ、Playwright でフィルターボタンをクリックする。**この2つは同時に実行される**：

```
Chrome DevTools → performance_start_trace(reload: true)
Playwright → browser_click(ref=e10)  // Trade ボタン
```

Chrome DevTools からは即座にメトリクスが返ってくる：

```
LCP: 339ms
  - TTFB: 61ms (17.9%)
  - Render Delay: 279ms (82.1%)
CLS: 0.09
```

さらに Lighthouse 監査とスクリーンショット撮影も同時実行：

```
Chrome DevTools → lighthouse_audit(device: "mobile")
Playwright → browser_take_screenshot()
```

### 対象アプリのコード

レビュー対象の Kotlin Compose for Web フロントエンドはこのような構成だ：

```kotlin
@Composable
fun App(apiClient: TradeNewsApiClient? = null) {
    var selectedCategory by remember { mutableStateOf<String?>(null) }
    val client = apiClient ?: remember { GraphQLClient() }

    DisposableEffect(client) {
        onDispose { client.close() }
    }

    // ヘルスチェック、カテゴリフィルター、ニュース一覧...
}
```

日付表示では `Intl.DateTimeFormat` で JST 固定表示を実装している：

```kotlin
fun formatDate(dateString: String): String {
    return try {
        val date = Date(dateString)
        val formatter = js("""new Intl.DateTimeFormat('ja-JP', {
            timeZone: 'Asia/Tokyo',
            year: 'numeric', month: '2-digit', day: '2-digit'
        })""")
        val formatted = (formatter.format(date) as String).replace("/", ".")
        "$formatted JST"
    } catch (e: dynamic) {
        dateString
    }
}
```

これらのコードが本番環境で正しく動作しているかを、二刀流で一気に検証できた。

## 結果・学び

### 取得できたデータ

| 項目 | 結果 |
|------|------|
| LCP | 339ms ✅ |
| CLS | 0.09 ✅ |
| TTFB | 61ms ✅ |
| Best Practices | 100/100 ✅ |
| Accessibility | 83/100 |
| SEO | 82/100 |
| コンソールエラー | 0件 ✅ |
| GraphQL レスポンス | 200 ✅ |

### 発見した改善点（Lighthouse Failed）

| 指摘 | 深刻度 |
|------|--------|
| コントラスト比不足 | Medium |
| 見出しレベル飛び（h1→h3） | Low |
| `<main>` ランドマーク未設定 | Low |
| meta description 未設定 | Medium |
| robots.txt 無効 | Low |

パフォーマンスは優秀だが、アクセシビリティと SEO に改善余地があることがわかった。これは従来のユニットテストや mutation testing では検出できない領域で、**ブラウザベースの MCP だからこそ見つかる指摘** だ。

### 二刀流の使い分け

実際に使ってみて感じた使い分けの勘所：

- **Playwright が得意**: DOM 構造の把握、ユーザー操作のシミュレート、視覚的なスクリーンショット
- **Chrome DevTools が得意**: Core Web Vitals 計測、Lighthouse 監査、ネットワーク/コンソールの低レベル監視
- **両方同時が効く場面**: 「フィルターをクリックした直後のネットワークリクエスト」のように、操作と観測を同時にしたいとき

## 参考

- [Chrome DevTools MCP](https://github.com/nichochar/chrome-devtools-mcp) — Chrome DevTools Protocol を MCP で公開するサーバー
- [Playwright MCP](https://github.com/nichochar/playwright-mcp) — Playwright をMCP で公開するサーバー
- [Claude Code MCP 設定ドキュメント](https://docs.anthropic.com/en/docs/claude-code/mcp)

---
🏀 NBA Trade Tracker: https://www.nba-iso-flow.com/
