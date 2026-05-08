---
title: "Core Web Vitals（LCP・INP・CLS）をコードで改善する実践ガイド"
emoji: "⚡"
type: "tech"
topics: ["performance", "javascript", "css", "html", "seo"]
published: false
---

ページ速度改善とは、Webページの表示速度を最適化してユーザー体験とコンバージョン率を向上させるCRO施策です。Googleの調査によると、ページの読み込みが1秒から3秒に増えると直帰率が32%増加し、5秒では90%増加します。

ページ速度の改善はフロントエンドのコード最適化が中心で、すべてコードで対応できます。この記事では、Core Web Vitalsの3指標を改善する具体的な手法を解説します。

---

## Core Web Vitalsの3指標と目標値

| 指標 | 正式名称 | 良好 | 改善が必要 | 不良 |
|---|---|---|---|---|
| LCP | Largest Contentful Paint | 2.5秒以内 | 2.5〜4.0秒 | 4.0秒超 |
| INP | Interaction to Next Paint | 200ms以内 | 200〜500ms | 500ms超 |
| CLS | Cumulative Layout Shift | 0.1以下 | 0.1〜0.25 | 0.25超 |

参考: [Web Vitals - web.dev](https://web.dev/articles/vitals)

---

## LCP（最大コンテンツ描画）の改善

### 画像の最適化

LCPの対象は多くの場合ファーストビューのメイン画像です。WebP形式への変換とサイズの最適化で大幅に改善できます。

```html
<picture>
  <source srcset="/img/hero.avif" type="image/avif">
  <source srcset="/img/hero.webp" type="image/webp">
  <img src="/img/hero.jpg" alt="メインビジュアル"
       width="1200" height="630"
       loading="eager" fetchpriority="high" decoding="async">
</picture>
```

### Critical CSSのインライン化

ファーストビューの描画に必要なCSSだけを`<style>`タグでインライン化し、残りは非同期で読み込みます。

```html
<!-- Critical CSS: インライン -->
<style>/* ファーストビュー用の最小限CSS */</style>

<!-- Non-critical CSS: 非同期読み込み -->
<link rel="preload" href="/css/main.css" as="style"
      onload="this.onload=null;this.rel='stylesheet'">
<noscript><link rel="stylesheet" href="/css/main.css"></noscript>
```

参考: [クリティカルCSSの抽出 - web.dev](https://web.dev/articles/extract-critical-css)

### リソースヒント

```html
<!-- 外部ドメインへの事前接続 -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>

<!-- LCP画像のプリロード -->
<link rel="preload" href="/img/hero.webp" as="image" type="image/webp">
```

参考: [リソースヒント - web.dev](https://web.dev/articles/preconnect-and-dns-prefetch)

---

## INP（応答性）の改善

INPはユーザーの操作（クリック・タップ・キー入力）から画面が更新されるまでの時間です。JavaScriptのメインスレッドブロックが主な原因です。

```html
<!-- 非クリティカルなJSを遅延読み込み -->
<script src="/js/analytics.js" defer></script>
<script src="/js/chat-widget.js" defer></script>

<!-- サードパーティスクリプトを非同期化 -->
<script src="https://example.com/widget.js" async></script>
```

- `defer`: HTML解析完了後に実行（実行順序を保証）
- `async`: ダウンロード完了次第実行（順序不定）
- ファーストビューに不要なJSは`defer`、独立したスクリプトは`async`を使う

---

## CLS（レイアウトシフト）の改善

CLSは、ページ読み込み中にレイアウトがずれる現象です。画像・広告・Webフォントが主な原因です。

| 原因 | 対策 | 実装方法 |
|---|---|---|
| 画像のサイズ未指定 | width/height属性を必ず指定 | HTML属性またはCSS aspect-ratio |
| Webフォントの読み込み | font-display: swapを設定 | CSS @font-face |
| 動的コンテンツの挿入 | 表示領域を事前に確保 | CSS min-heightで固定 |
| 広告枠 | 固定サイズのコンテナを用意 | CSS width/height固定 |

```css
/* 画像のアスペクト比を事前確保 */
img {
  aspect-ratio: attr(width) / attr(height);
  width: 100%;
  height: auto;
}

/* Webフォントの読み込み中にレイアウトシフトを防止 */
@font-face {
  font-family: 'Noto Sans JP';
  src: url('/fonts/NotoSansJP.woff2') format('woff2');
  font-display: swap;
}
```

参考: [Cumulative Layout Shift (CLS) - web.dev](https://web.dev/articles/cls)

---

## まとめ

この記事は [CodeQuest.work](https://codequest.work/pagespeed-guide/) に掲載した記事を技術者向けに再構成したものです。

CROシリーズの他の記事:
- [CRO完全ガイド — コードで実装する8つの手法](https://codequest.work/cro-guide/)
- [ABテストのやり方 — コード実装方法](https://codequest.work/ab-test-guide/)
- [EFO（フォーム最適化）実装ガイド](https://codequest.work/efo-guide/)
- [離脱防止ポップアップの実装方法](https://codequest.work/exit-popup-guide/)
- [マイクロコピー改善でCVRを上げる方法](https://codequest.work/microcopy-guide/)
- [ファーストビュー改善の方法](https://codequest.work/firstview-guide/)
- [Core Web Vitalsをコードで最適化](https://codequest.work/pagespeed-guide/)
- [ソーシャルプルーフの実装パターン](https://codequest.work/social-proof-guide/)
- [Webパーソナライズの実装方法](https://codequest.work/personalize-guide/)

