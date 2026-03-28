---
title: "Figma MCP × Claude Code で個人開発サイトを一気にリデザインした話"
emoji: "🏀"
type: "tech"
topics: ["figma", "claudecode", "webdev", "compose", "design"]
published: false
---

## TL;DR

- **Figma MCP Server** を使うと、Claude Code から直接 Figma にデザインを書き出せる
- 既存の画面をFigmaに再現 → 2026年トレンドのリデザイン案を生成 → コード修正まで**1セッションで完結**した
- ダークモード・Glassmorphism・Bento Grid など7つのトレンドを「ダークモードを軸」に統合して破綻なく適用できた

## 背景

個人で運営している NBA ニュースサイト **[NBA ISO FLOW](https://www.nba-iso-flow.com/)** のフロントエンドが、2020年代前半のフラットデザインのまま放置されていた。

技術スタックは Kotlin Compose for Web + Apollo GraphQL Client。バックエンドは Rust（Axum + async-graphql）で、RSS フィードから NBA の最新ニュースを収集し、日本語に翻訳して配信している。

UIの改善をしたいが、デザインツールとコードを行き来するのが面倒で後回しにしていた。そこで **Figma MCP Server × Claude Code** の組み合わせを試してみた。

## やったこと：3ステップのワークフロー

### Step 1: 現状の画面を Figma に書き出す

Claude Code から Figma の `use_figma` ツールを使って、コードベースのスタイル定義を読み取りながら Figma 上にフレームを構築した。

```
Claude Code → ソースコード読み取り → use_figma でFigmaに描画
```

ヘッダー、カテゴリフィルター、ニュースカードグリッド、フッターをセクションごとに作成。各セクションを作るたびに `get_screenshot` で確認しながら進めた。

### Step 2: 2026年トレンドでリデザイン

現状の画面をFigmaに落とした後、「2026年のトレンドで見直すならどこか？」を検討し、以下の7つを適用した。

| トレンド | 適用 | 控えめ度 |
|---------|------|---------|
| ダークモード | 全体のベース | ガッツリ |
| Bento Grid | カードレイアウト | ガッツリ |
| Pill タブ | カテゴリフィルター | ガッツリ |
| グラデーション | ヘッダーのみ | 控えめ |
| Glassmorphism | カード背景のみ | 控えめ |
| 角丸拡大 | 全体に薄く | 控えめ |
| マイクロインタラクション | ホバーだけ | 最小限 |

ポイントは **「全盛りでもダークモードを軸にすれば破綻しない」** という設計判断。「ガッツリ3 + 控えめ4」のバランスを意識した。

### Step 3: Figma → コード修正

Figma上で確定したデザインをもとに、Kotlin Compose for Web のスタイル定義を修正。変更は CSS（StyleSheet DSL）がメインで、ロジック変更はカテゴリフィルターの `<select>` → `<button>` のみ。

## 実際のコード変更

### ダークモード + グラデーションヘッダー

```kotlin
// App.kt - AppStylesheet
"body" style {
    backgroundColor(Color("#0f172a"))  // slate-900
    color(Color("#f1f5f9"))            // slate-100
}

".app-header" style {
    property("background", "linear-gradient(135deg, #1e40af 0%, #4f46e5 50%, #8b5cf6 100%)")
}

".app-header h1" style {
    fontSize(3.5.em)
    fontWeight(900)
    property("letter-spacing", "-0.03em")  // タイトルをキュッと詰める
}
```

### Glassmorphism カード

```kotlin
// NewsCard.kt - NewsCardStyles
".news-card" style {
    backgroundColor(rgba(30, 41, 59, 0.6))           // 半透明ダーク
    borderRadius(20.px)                                // 角丸拡大
    property("border", "1px solid rgba(255,255,255,0.08)")  // 微細な白枠
    property("backdrop-filter", "blur(16px)")           // すりガラス
    property("box-shadow", "0 4px 24px rgba(0,0,0,0.25)")
}

".news-card:hover" style {
    property("transform", "translateY(-2px)")  // マイクロインタラクション
    property("box-shadow", "0 8px 32px rgba(0,0,0,0.35)")
}
```

### Pill タブフィルター

旧デザインの `<select>` ドロップダウンを、Pill 型のボタンタブに変更した。

```kotlin
// CategoryFilter.kt
@Composable
fun CategoryFilter(
    selectedCategory: String?,
    onCategorySelected: (String?) -> Unit
) {
    val categories = listOf(
        null to "すべて",
        "Trade" to "Trade",
        "Signing" to "Signing",
        "Other" to "Other"
    )

    Div(attrs = { classes("category-filter") }) {
        categories.forEach { (value, label) ->
            Button(attrs = {
                classes("category-pill")
                if (selectedCategory == value) {
                    classes("category-pill-active")
                }
                onClick { onCategorySelected(value) }
            }) {
                Text(label)
            }
        }
    }
}
```

アクティブ状態は blue → purple のグラデーション:

```kotlin
".category-pill-active" style {
    property("background", "linear-gradient(135deg, #3b82f6, #8b5cf6)")
    fontWeight(600)
    opacity(1)
}
```

### Bento Grid

CSS Grid の `:first-child` に `grid-column: span 2` を指定するだけで、先頭のニュースカードが2列幅の「ヒーローカード」になる。

```kotlin
// NewsList.kt
".news-grid" style {
    display(DisplayStyle.Grid)
    property("grid-template-columns", "repeat(3, 1fr)")
    gap(1.25.em)
}

".news-grid > *:first-child" style {
    property("grid-column", "span 2")  // 先頭カードをヒーロー表示
}
```

## Figma MCP の使い勝手

### 良かった点

- **コードとデザインの往復がゼロ**。Claude Code 内で完結する
- `use_figma` で Plugin API を直接叩けるので、Auto Layout やフレーム構造も正確に作れる
- `get_screenshot` でセクションごとに確認できるため、手戻りが少ない

### 注意点

- `fills` の `color` に `alpha` を直接指定できない（`opacity` プロパティを使う）
- `layoutSizingHorizontal = "FILL"` は `appendChild` の**後**に設定しないとエラーになる
- 1回の `use_figma` で大きすぎるスクリプトを書くと失敗しやすい。セクション単位で分割するのがコツ

## まとめ

Figma MCP × Claude Code の組み合わせで、**デザイン検討 → Figma プロトタイプ → コード修正** を1セッションで完結できた。

個人開発では「デザインは後でやろう」と先延ばしにしがちだが、このワークフローなら思い立った時にサクッと改善できる。デザインツールを開く手間すらない。

---

🏀 **NBA ISO FLOW**: [https://www.nba-iso-flow.com/](https://www.nba-iso-flow.com/)
NBAの最新トレード・FA・ニュースを日本語でリアルタイム配信中。Kotlin Compose for Web + Rust で構築した個人開発プロジェクト。
